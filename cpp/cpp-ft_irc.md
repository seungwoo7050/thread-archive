===== BEGIN FILE: 01-portable-readiness-and-nonblocking-transport.md =====
# 개발 흐름 01 — 이식 가능한 입출력 준비 상태와 논블로킹 전송

부제: 이벤트 준비 상태와 논블로킹 전송

## 1. 개발 흐름의 목표

native 준비 상태 API의 차이를 공통 의미로 번역하고, move-only 연결 상태와 single-threaded event loop가 입력·출력·종료를 어떻게 진행시키는지 커밋 순서대로 복원합니다.

### 원문에서 정한 의의

플랫폼별 준비 상태 API를 공통 이벤트 의미로 바꾸고, 연결마다 파일 디스크립터·입력 조각·미전송 출력·종료 상태를 하나의 이동 전용 객체가 소유하도록 만듭니다. 부분 송신 진행 상태와 서버의 단일 정리 경로가 이후 신뢰성 수정의 전제가 됩니다.

<details>
<summary>영문 원문</summary>

> This progression builds the server from a portable event vocabulary into an operational single-threaded reactor. The decisive choices are not the platform calls themselves, but the separation of native readiness from server semantics, the move-only per-connection state machine, persistent partial-write progress, and the authoritative server cleanup path. Later reliability work depends directly on these ownership and I/O contracts.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- `epoll`과 `kqueue`의 등록/notification 차이를 어느 경계에서 공통 의미로 바꾸는가?
- 한 fd의 소켓 수명, 입력 fragment, 출력 뒷부분, close 상태를 누가 함께 소유하는가?
- 준비 상태 한 번이 여러 line, partial line, partial write, EOF, backpressure를 만들 때 각 상태는 어디에 남는가?
- accepted fd가 event registry와 연결 map에 들어가고 빠지는 순서는 무엇인가?
- 대기 중인 출력과 write 관심 상태, graceful close completion은 어떤 조건으로 동기화되는가?

### 원문에서 확인되는 불변식

- 수락한 파일 디스크립터마다 권위 있는 `Connection` 소유자가 하나만 있어야 하고, 파일 디스크립터는 정확히 한 번 닫혀야 합니다. 이벤트 등록 정보와 연결 맵의 상태가 어긋나서는 안 됩니다.
- 부분 송신은 실제로 전송된 바이트 수만큼만 진행해야 하며, 재시도할 때 미전송 구간의 순서를 보존해야 합니다.
- 입력 프레임은 점진적으로 분리하고 출력 줄은 CRLF로 정규화해야 합니다. 형식이 잘못되었거나 지나치게 긴 프레임을 이후 유효한 명령으로 처리해서는 안 됩니다.

### 원문에서 확인되는 구현 난점

- `epoll`과 `kqueue`의 서로 다른 등록/notification 모델을 server loop에 노출하지 않고 통합하는 문제.
- 한 번의 준비 이벤트에서 여러 줄, 미완성 줄, 부분 송수신, 인터럽트, 역압, EOF, 복구 불가능한 오류가 발생할 수 있는 바이트 스트림을 모델링하는 문제.
- 콜백 또는 관심 상태 update가 현재 처리 중인 연결을 제거할 수 있는 수명 문제의 기반.

## 3. 완료 기준

- 공통 `Event`에서 `Server::pollOnce()`와 클라이언트 read/write 처리까지 호출자/피호출자 흐름을 해당 SHA 코드로 설명할 수 있습니다.
- `Connection`의 fd·buffer·offset·close 상태 소유권과 move 이후 원본/destination 상태를 증거 코드로 제시할 수 있습니다.
- `EINTR`, `EAGAIN`/`EWOULDBLOCK`, EOF, hard error, hangup을 read/write/wait별로 구분할 수 있습니다.
- queue된 byte와 write 관심 상태가 어긋나지 않는 조건 및 disconnect의 erase-before-콜백 순서를 설명할 수 있습니다.
- 최종 HEAD가 아니라 각 커밋 SHA의 코드와 필요한 직전 관련 SHA만으로 변화 과정을 기록했습니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `8e8a3db87950` | `feat(event): 이벤트 준비 상태 계약 정의` | A | ARCH, EVENT_IO | 공통 입출력 준비 상태와 백엔드 규약을 정의합니다. |
| 2 | `d3f74e2857da` | `feat(event): kqueue 관심 상태 등록 구현` | B | EVENT_IO | 논리적 관심 상태 변경을 kqueue 필터에 반영합니다. |
| 3 | `769cd3094f71` | `feat(event): kqueue 준비 이벤트 변환 구현` | B | EVENT_IO | kqueue 알림을 공통 이벤트로 변환합니다. |
| 4 | `2284a0e0d8bb` | `feat(event): epoll 관심 상태 등록 구현` | B | EVENT_IO | 논리적 관심 상태 변경을 epoll 등록에 반영합니다. |
| 5 | `f1320f357bca` | `feat(event): epoll 준비 이벤트 변환 구현` | B | EVENT_IO | epoll 알림과 소켓 오류를 공통 이벤트로 변환합니다. |
| 6 | `8864f253ac15` | `feat(connection): 스트림 연결 상태 계약 정의` | A | ARCH, EVENT_IO, LIFECYCLE | 이동만 가능한 `Connection`과 입출력 결과 형식을 정의합니다. |
| 7 | `d71a2549c0eb` | `feat(connection): 소켓 소유권과 이동 수명 구현` | A | LIFECYCLE, RISK | 소켓 디스크립터 소유권을 정확히 한 번 이전·해제하도록 구현합니다. |
| 8 | `9b601b69de4f` | `feat(connection): IRC 입력 프레임 추출 구현` | A | EVENT_IO, IRC_PROTOCOL, RISK | IRC 입력을 점진적으로 프레임으로 분리하고 줄 길이 한계를 정합니다. |
| 9 | `b00589d4b1b1` | `feat(connection): 논블로킹 수신 상태 처리` | A | EVENT_IO, RISK | 논블로킹 수신 상태 처리를 구현합니다. |
| 10 | `a10fe961e2b1` | `feat(connection): 부분 송신 대기열 처리` | S | CORE, EVENT_IO, LIFECYCLE | 부분 송신 진행 상태를 보존하고 출력 대기열을 모두 비운 뒤 닫도록 구현합니다. |
| 11 | `e6492e27cc30` | `feat(server): 이벤트 서버 공개 계약 정의` | A | ARCH, EVENT_IO, LIFECYCLE | `Server`의 소유 대상과 전송 계층 콜백 경계를 정의합니다. |
| 12 | `4ad1227e5119` | `feat(server): 논블로킹 연결 수락과 등록 구현` | A | EVENT_IO, LIFECYCLE | 수락한 디스크립터를 연결 객체와 이벤트 등록 소유권에 편입합니다. |
| 13 | `378d5304828d` | `feat(server): 준비 이벤트 루프 구동` | S | CORE, ARCH, EVENT_IO | 이식 가능한 이벤트 루프를 실제로 동작하게 만듭니다. |
| 14 | `625ffc924de8` | `feat(server): 송신 큐와 쓰기 관심 상태 연결` | A | EVENT_IO, LIFECYCLE, INTEGRATION | 전송 대기 데이터, 쓰기 준비 이벤트, 종료 완료 조건을 연결합니다. |
| 15 | `7a6bc7e1276a` | `feat(server): 연결 해제와 오류 정리 구현` | A | LIFECYCLE, RISK | 전송 계층 연결 제거와 연결 종료 통지를 한 경로로 모읍니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `8e8a3db87950` — `feat(event): 이벤트 준비 상태 계약 정의`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | ARCH, EVENT_IO |
| 원문에서 정한 역할 | 공통 입출력 준비 상태와 백엔드 규약을 정의합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 공통 입출력 준비 상태와 백엔드 규약을 정의합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | portable 준비 상태 의미와 backend 수명 계약이 아직 존재하지 않았습니다. |
| 주요 문제와 설계 판단 | `EventInterest`를 Read/Write 조합형 비트마스크로 만들고, `Event`가 준비 상태·error·hangup·소켓 error를 공통 형태로 보유하도록 했습니다. `EventManager`는 add/update/remove/wait와 `createDefault()`를 제공하는 추상 경계가 되었습니다. |
| 변경 파일·symbol | `include/EventManager.hpp — EventInterest, Event, EventManager` |
| 상태 / 소유권 변화 | native flag는 공개 계약 밖에 남고 backend 인스턴스는 `std::unique_ptr<EventManager>`가 단독 소유합니다. |
| 실패 또는 경계 | 이 SHA는 실패를 공통 예외형으로 강제하지는 않으며 실제 kernel 오류 처리는 backend 구현에 맡깁니다. |
| 보장 / 비보장 | 보장: 상위 `Server`가 epoll/kqueue 상수 없이 읽기·쓰기 관심과 준비 결과를 표현할 수 있습니다.<br>비보장: 실제 등록·대기·오류 번역은 아직 구현되지 않았습니다. |
| 다음 관련 변화 | d3f74e2857da와 2284a0e0d8bb가 등록을, 769cd3094f71과 f1320f357bca가 notification 번역을 채웁니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `8e8a3db87950` | `include/EventManager.hpp` | `EventInterest, operator\|, hasInterest` | read와 write를 동시에 표현하는 공통 비트 연산 |
| `8e8a3db87950` | `include/EventManager.hpp` | `EventManager::createDefault` | backend 수명을 unique_ptr로 상위 계층에 이전 |

- 조사 방법: GitHub에서 `8e8a3db87950` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `d3f74e2857da` — `feat(event): kqueue 관심 상태 등록 구현`. 원문 역할은 “논리적 관심 상태 변경을 kqueue 필터에 반영합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `d3f74e2857da` — `feat(event): kqueue 관심 상태 등록 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | EVENT_IO |
| 원문에서 정한 역할 | 논리적 관심 상태 변경을 kqueue 필터에 반영합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 논리적 관심 상태 변경을 kqueue 필터에 반영합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 논리적 Read/Write를 `EVFILT_READ`/`EVFILT_WRITE`의 독립 filter로 비교·적용하고, kernel 변경이 성공한 뒤 그림자 map을 갱신했습니다. |
| 확인한 변경 파일·symbol | `src/KqueueEventManager.cpp — addFd, updateFd, removeFd, applyInterestChange, applyFilterChange` |
| 핵심 상태 / 브랜치 | fd별 설치 관심은 backend 그림자 map이 보유하며 kernel과 local 상태가 같은 순서로 전진합니다. |
| 실패 handling | 등록 변경 실패는 `std::system_error`로 전달합니다. remove 중 이미 사라진 대상의 `ENOENT`/`EBADF`는 정리의 멱등성을 위해 허용합니다. |
| 보장과 다음 연결 | read와 write filter의 추가·삭제가 이전/새 관심 차이에 맞게 적용됩니다. 769cd3094f71가 같은 backend의 wait 결과를 번역합니다. |
| 이 시점의 한계 | 준비 이벤트의 공통 `Event` 변환은 아직 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d3f74e2857da` | `src/KqueueEventManager.cpp` | `applyInterestChange` | 이전/새 관심을 비교해 filter별 변경을 생성 |
| `d3f74e2857da` | `src/KqueueEventManager.cpp` | `removeFd` | 허용 errno와 재전파 errno를 분리 |

- 조사 방법: GitHub에서 `d3f74e2857da` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `769cd3094f71` — `feat(event): kqueue 준비 이벤트 변환 구현`. 원문 역할은 “kqueue 알림을 공통 이벤트로 변환합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `769cd3094f71` — `feat(event): kqueue 준비 이벤트 변환 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | EVENT_IO |
| 원문에서 정한 역할 | kqueue 알림을 공통 이벤트로 변환합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> kqueue 알림을 공통 이벤트로 변환합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 최대 128개의 `kevent`를 받고 filter를 Read/Write로, `EV_ERROR`를 error와 코드로, `EV_EOF`를 hangup으로 변환했습니다. |
| 확인한 변경 파일·symbol | `src/KqueueEventManager.cpp — wait, createDefault` |
| 핵심 상태 / 브랜치 | 한 native notification은 fd와 공통 관심 상태/error/hangup 상태를 가진 `Event`로 복사됩니다. |
| 실패 handling | `kevent`가 `EINTR`로 끝나면 빈 결과로 돌아가 loop가 다시 진행하며 그 외 오류는 예외로 전달합니다. |
| 보장과 다음 연결 | kqueue-specific flag layout이 `Server`에 노출되지 않습니다. 2284a0e0d8bb와 f1320f357bca가 Linux backend에 같은 공통 의미를 구현합니다. |
| 이 시점의 한계 | 동일 fd의 여러 native record 병합 여부와 상위 처리 semantics는 server 단계에서 결정됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `769cd3094f71` | `src/KqueueEventManager.cpp` | `KqueueEventManager::wait` | EVFILT/EV_ERROR/EV_EOF를 공통 Event 필드로 번역 |

- 조사 방법: GitHub에서 `769cd3094f71` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `2284a0e0d8bb` — `feat(event): epoll 관심 상태 등록 구현`. 원문 역할은 “논리적 관심 상태 변경을 epoll 등록에 반영합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `2284a0e0d8bb` — `feat(event): epoll 관심 상태 등록 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | EVENT_IO |
| 원문에서 정한 역할 | 논리적 관심 상태 변경을 epoll 등록에 반영합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 논리적 관심 상태 변경을 epoll 등록에 반영합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `nativeEventsFor()`로 Read/Write 관심을 epoll mask로 만들고 `epoll_ctl` ADD/MOD/DEL을 backend 메서드에 배치했습니다. |
| 확인한 변경 파일·symbol | `src/EpollEventManager.cpp — nativeEventsFor, addFd, updateFd, removeFd` |
| 핵심 상태 / 브랜치 | fd 등록 상태는 kernel epoll set이 authoritative하며 공통 관심 상태가 호출 입력입니다. |
| 실패 handling | `epoll_ctl` 실패는 system error로 전달하고 제거 중 이미 닫히거나 사라진 fd는 멱등 정리 대상으로 취급합니다. |
| 보장과 다음 연결 | Linux에서도 동일한 add/update/remove 계약을 제공합니다. f1320f357bca가 준비 상태/error/hangup 변환을 완성합니다. |
| 이 시점의 한계 | `epoll_wait` 결과와 `SO_ERROR` 번역은 다음 커밋 전에는 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `2284a0e0d8bb` | `src/EpollEventManager.cpp` | `nativeEventsFor` | 공통 Read/Write를 EPOLLIN/EPOLLOUT로 변환 |
| `2284a0e0d8bb` | `src/EpollEventManager.cpp` | `addFd/updateFd/removeFd` | epoll_ctl 작업별 등록 수명 |

- 조사 방법: GitHub에서 `2284a0e0d8bb` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `f1320f357bca` — `feat(event): epoll 준비 이벤트 변환 구현`. 원문 역할은 “epoll 알림과 소켓 오류를 공통 이벤트로 변환합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `f1320f357bca` — `feat(event): epoll 준비 이벤트 변환 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | EVENT_IO |
| 원문에서 정한 역할 | epoll 알림과 소켓 오류를 공통 이벤트로 변환합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> epoll 알림과 소켓 오류를 공통 이벤트로 변환합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `epoll_wait` 결과의 IN/OUT/ERR/HUP/RDHUP를 공통 `Event`로 만들고, error 시 `getsockopt(SO_ERROR)`로 소켓 error를 보충했습니다. |
| 확인한 변경 파일·symbol | `src/EpollEventManager.cpp — wait, socketErrorFor, createDefault` |
| 핵심 상태 / 브랜치 | native event array에서 공통 Event vector로 값이 복사되며 backend 수명은 팩터리 반환 unique_ptr가 관리합니다. |
| 실패 handling | `epoll_wait`의 `EINTR`은 빈 준비 상태로 처리하고 다른 오류는 예외로 전달합니다. `SO_ERROR` 조회 실패도 오류 정보를 잃지 않도록 처리됩니다. |
| 보장과 다음 연결 | epoll과 kqueue가 상위 loop에 같은 필드 집합을 제공합니다. 8864f253ac15부터 공통 Event가 구동할 연결 상태를 정의합니다. |
| 이 시점의 한계 | 두 OS에서 실제로 동일 suite가 실행되는 보장은 416efc91e580의 CI에서 추가됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f1320f357bca` | `src/EpollEventManager.cpp` | `EpollEventManager::wait` | EPOLL* notification을 공통 Event로 변환 |
| `f1320f357bca` | `src/EpollEventManager.cpp` | `socketErrorFor` | EPOLLERR를 소켓-level error 코드와 연결 |

- 조사 방법: GitHub에서 `f1320f357bca` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `8864f253ac15` — `feat(connection): 스트림 연결 상태 계약 정의`. 원문 역할은 “이동만 가능한 `Connection`과 입출력 결과 형식을 정의합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `8864f253ac15` — `feat(connection): 스트림 연결 상태 계약 정의`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | ARCH, EVENT_IO, LIFECYCLE |
| 원문에서 정한 역할 | 이동만 가능한 `Connection`과 입출력 결과 형식을 정의합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 이동만 가능한 `Connection`과 입출력 결과 형식을 정의합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | fd별 byte-스트림 진행 상태와 소유권을 한 객체로 묶는 계약이 없었습니다. |
| 주요 문제와 설계 판단 | `Connection`을 copy 금지·move 허용 객체로 정의하고 fd, peer address, read/write buffer, `writeOffset_`, line limit, peer/close 상태와 read/write 결과를 한곳에 모았습니다. |
| 변경 파일·symbol | `include/Connection.hpp — Connection, ReadResult, WriteResult` |
| 상태 / 소유권 변화 | 한 `Connection`이 하나의 fd와 그 fd의 미완성 입력·미전송 출력·종료 상태를 함께 소유합니다. |
| 실패 또는 경계 | 결과 구조는 would-block, peer close, hard error를 분리해 server가 즉시 제거와 재시도를 구분할 수 있게 합니다. |
| 보장 / 비보장 | 보장: transport 상태의 authoritative 소유자와 move-only 경계가 정의됩니다.<br>비보장: 소멸자와 move의 exactly-once close 구현은 다음 커밋 전에는 선언뿐입니다. |
| 다음 관련 변화 | d71a2549c0eb이 fd 이동 수명을, 9b601b69de4f부터 실제 I/O 상태 처리를 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `8864f253ac15` | `include/Connection.hpp` | `Connection copy/move declarations` | copy를 막고 파일 디스크립터 소유권 이전만 허용 |
| `8864f253ac15` | `include/Connection.hpp` | `readBuffer_, writeBuffer_, writeOffset_` | framing과 partial 출력 진행 상태를 fd 소유자에 결합 |

- 조사 방법: GitHub에서 `8864f253ac15` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `d71a2549c0eb` — `feat(connection): 소켓 소유권과 이동 수명 구현`. 원문 역할은 “소켓 디스크립터 소유권을 정확히 한 번 이전·해제하도록 구현합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `d71a2549c0eb` — `feat(connection): 소켓 소유권과 이동 수명 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | LIFECYCLE, RISK |
| 원문에서 정한 역할 | 소켓 디스크립터 소유권을 정확히 한 번 이전·해제하도록 구현합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 소켓 디스크립터 소유권을 정확히 한 번 이전·해제하도록 구현합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | move-only 선언은 있었지만 fd를 정확히 한 번 닫는 구현이 없었습니다. |
| 주요 문제와 설계 판단 | 소멸자가 유효 fd를 닫고, move 생성자/대입이 모든 상태를 옮긴 뒤 원본 fd를 -1로 만들었습니다. move 대입은 destination의 기존 fd를 먼저 닫습니다. |
| 변경 파일·symbol | `src/Connection.cpp — destructor, move constructor, move assignment` |
| 상태 / 소유권 변화 | 소유권 이전 후 destination만 fd를 보유하고 moved-from 객체는 비소유 상태가 됩니다. |
| 실패 또는 경계 | 자기 대입을 피하고 기존 destination 자원을 먼저 해제해 overwrite 누수를 막습니다. |
| 보장 / 비보장 | 보장: 정상 소멸과 move chain에서 파일 디스크립터가 중복 close되거나 유실되지 않습니다.<br>비보장: event registry와 server map 등록 실패까지의 cross-object 되돌리기는 아직 다루지 않습니다. |
| 다음 관련 변화 | 4ad1227e5119가 이 move-only 객체를 server map에 넣고 5dcd882f0763가 등록 되돌리기를 보강합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d71a2549c0eb` | `src/Connection.cpp` | `Connection::~Connection` | 유효 fd의 최종 close 소유자 |
| `d71a2549c0eb` | `src/Connection.cpp` | `operator=(Connection&&)` | 기존 fd close 후 상태 이동, 원본 fd=-1 |

- 조사 방법: GitHub에서 `d71a2549c0eb` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `9b601b69de4f` — `feat(connection): IRC 입력 프레임 추출 구현`. 원문 역할은 “IRC 입력을 점진적으로 프레임으로 분리하고 줄 길이 한계를 정합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `9b601b69de4f` — `feat(connection): IRC 입력 프레임 추출 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | EVENT_IO, IRC_PROTOCOL, RISK |
| 원문에서 정한 역할 | IRC 입력을 점진적으로 프레임으로 분리하고 줄 길이 한계를 정합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> IRC 입력을 점진적으로 프레임으로 분리하고 줄 길이 한계를 정합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | read buffer는 있었지만 TCP fragment에서 IRC line을 증분 추출하는 규칙이 없었습니다. |
| 주요 문제와 설계 판단 | newline을 기준으로 complete 프레임을 반복 추출하고 직전 CR을 제거하며 incomplete 뒷부분은 buffer에 남겼습니다. 최대 line 길이를 newline 포함 기준으로 검사했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — extractLines` |
| 상태 / 소유권 변화 | read buffer는 아직 line이 되지 않은 뒷부분만 유지하고 결과 vector가 완성 line의 순서를 보유합니다. |
| 실패 또는 경계 | oversized complete 프레임 또는 delimiter 없이 limit를 넘은 fragment는 close/error 상태를 만들고 해당 bytes를 이후 valid 명령으로 남기지 않습니다. |
| 보장 / 비보장 | 보장: TCP packet 경계와 무관한 incremental IRC framing을 제공합니다.<br>비보장: 실제 recv drain, EOF, EINTR/would-block 처리는 다음 커밋에 없습니다. |
| 다음 관련 변화 | b00589d4b1b1가 소켓 bytes를 이 framing 함수로 공급합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `9b601b69de4f` | `src/Connection.cpp` | `Connection::extractLines` | newline 반복 추출, CR 제거, 뒷부분 보존과 size 경계 |

- 조사 방법: GitHub에서 `9b601b69de4f` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `b00589d4b1b1` — `feat(connection): 논블로킹 수신 상태 처리`. 원문 역할은 “논블로킹 수신 상태 처리를 구현합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `b00589d4b1b1` — `feat(connection): 논블로킹 수신 상태 처리`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | EVENT_IO, RISK |
| 원문에서 정한 역할 | 논블로킹 수신 상태 처리를 구현합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 논블로킹 수신 상태 처리를 구현합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 프레임 extractor는 있었지만 non-blocking 소켓을 끝까지 읽는 상태 처리가 없었습니다. |
| 주요 문제와 설계 판단 | `recv`를 반복해 positive bytes를 buffer에 붙이고 line을 추출하며, 0은 peer close, EINTR은 재시도, EAGAIN/EWOULDBLOCK은 현재 drain 완료, 나머지는 hard error로 분류했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — readAvailable` |
| 상태 / 소유권 변화 | 한 read 준비 상태에서 가능한 bytes와 lines를 모두 소비하고 incomplete 뒷부분은 다음 event까지 남습니다. |
| 실패 또는 경계 | EOF와 hard error는 close 경로로 전달되고 would-block은 연결 상태를 유지합니다. |
| 보장 / 비보장 | 보장: edge/level-trigger 차이와 무관하게 준비된 입력을 drain하며 retryable 상태를 오류로 오인하지 않습니다.<br>비보장: 애플리케이션 콜백 중 연결이 제거되는 reentrancy는 server 단계에서 아직 해결되지 않습니다. |
| 다음 관련 변화 | 378d5304828d가 read result를 line 콜백과 disconnect 판단에 연결합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `b00589d4b1b1` | `src/Connection.cpp` | `Connection::readAvailable` | recv 결과별 progress/retry/terminal 브랜치 |

- 조사 방법: GitHub에서 `b00589d4b1b1` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `a10fe961e2b1` — `feat(connection): 부분 송신 대기열 처리`. 원문 역할은 “부분 송신 진행 상태를 보존하고 출력 대기열을 모두 비운 뒤 닫도록 구현합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.10 `a10fe961e2b1` — `feat(connection): 부분 송신 대기열 처리`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | CORE, EVENT_IO, LIFECYCLE |
| 원문에서 정한 역할 | 부분 송신 진행 상태를 보존하고 출력 대기열을 모두 비운 뒤 닫도록 구현합니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> 부분 송신 진행 상태를 보존하고 출력 대기열을 모두 비운 뒤 닫도록 구현합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | 출력 bytes와 한 번의 send를 연결할 영속 progress 상태가 없었습니다. |
| 실패 / 경계 | EINTR은 같은 뒷부분을 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 핵심 결정 | `writeOffset_`로 sent 앞부분을 나타내고 `flushPending()`이 unsent 뒷부분만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — flushPending, queueRaw, queueLine, pendingBytes, wantsWrite` |
| 소유권 / lifecycle / 상태 변화 | `writeBuffer_[0:writeOffset_]`는 이미 전송된 앞부분, 나머지는 순서가 보존된 대기 중인 뒷부분입니다. 완료 시 buffer/offset을 reset하고 큰 consumed 앞부분만 compact합니다. |
| 이 커밋이 보장하는 것 | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 아직 보장하지 않는 것 | queue 크기는 무제한이며 `pending + append` 오버플로와 send가 요청보다 큰 값을 돌려주는 비정상 결과는 아직 방어하지 않습니다. |
| 후속 fix/테스트 연결 | 625ffc924de8이 write 관심 상태에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | 출력 bytes와 한 번의 send를 연결할 영속 progress 상태가 없었습니다. |
| 2. 실패 conditions / 경계 | EINTR은 같은 뒷부분을 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 3. 선택한 표현과 순서 | `writeOffset_`로 sent 앞부분을 나타내고 `flushPending()`이 unsent 뒷부분만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 4. authoritative 상태 | `writeBuffer_[0:writeOffset_]`는 이미 전송된 앞부분, 나머지는 순서가 보존된 대기 중인 뒷부분입니다. 완료 시 buffer/offset을 reset하고 큰 consumed 앞부분만 compact합니다. |
| 5. 결과 불변식 | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 6. 후속 보강 | 625ffc924de8이 write 관심 상태에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a10fe961e2b1` | `src/Connection.cpp` | `Connection::flushPending` | unsent 뒷부분 포인터/length와 return-count 기반 offset 전진 |
| `a10fe961e2b1` | `src/Connection.cpp` | `queueLine` | 기존 CR/LF 제거 후 CRLF 하나 append |
| `a10fe961e2b1` | `src/Connection.cpp` | `pendingBytes/wantsWrite` | logical unsent bytes가 server 관심 상태 판단의 입력 |

- 조사 방법: GitHub에서 `a10fe961e2b1` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `e6492e27cc30` — `feat(server): 이벤트 서버 공개 계약 정의`. 원문 역할은 “`Server`의 소유 대상과 전송 계층 콜백 경계를 정의합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.11 `e6492e27cc30` — `feat(server): 이벤트 서버 공개 계약 정의`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | ARCH, EVENT_IO, LIFECYCLE |
| 원문에서 정한 역할 | `Server`의 소유 대상과 전송 계층 콜백 경계를 정의합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> `Server`의 소유 대상과 전송 계층 콜백 경계를 정의합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection과 EventManager는 있었지만 listener·연결 map·콜백을 조율하는 소유자가 없었습니다. |
| 주요 문제와 설계 판단 | `Server` 공개 계약에 config, lifecycle, poll, send/disconnect API와 connect/line/disconnect/error 콜백을 정의했습니다. event manager와 fd→`unique_ptr<Connection>` map을 소유하게 했습니다. |
| 변경 파일·symbol | `include/Server.hpp — Server, Config, callback types` |
| 상태 / 소유권 변화 | Server가 listener, backend, 모든 live Connection의 authoritative transport 소유자가 됩니다. 애플리케이션은 콜백을 통해서만 protocol 상태를 연결합니다. |
| 실패 또는 경계 | 공개 API는 start/poll/send/disconnect 실패가 transport 정리로 수렴할 수 있는 경계를 제공합니다. |
| 보장 / 비보장 | 보장: event reactor와 protocol coordinator 사이의 소유·호출 경계가 정의됩니다.<br>비보장: accept, dispatch, 관심 상태 refresh, 정리는 아직 구현되지 않았습니다. |
| 다음 관련 변화 | 4ad1227e5119부터 listener와 accepted fd를 실제 소유자 graph에 편입합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e6492e27cc30` | `include/Server.hpp` | `connections_` | fd별 unique_ptr Connection 소유 |
| `e6492e27cc30` | `include/Server.hpp` | `ConnectHandler/LineHandler/DisconnectHandler` | transport와 애플리케이션의 콜백 경계 |

- 조사 방법: GitHub에서 `e6492e27cc30` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `4ad1227e5119` — `feat(server): 논블로킹 연결 수락과 등록 구현`. 원문 역할은 “수락한 디스크립터를 연결 객체와 이벤트 등록 소유권에 편입합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.12 `4ad1227e5119` — `feat(server): 논블로킹 연결 수락과 등록 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | EVENT_IO, LIFECYCLE |
| 원문에서 정한 역할 | 수락한 디스크립터를 연결 객체와 이벤트 등록 소유권에 편입합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 수락한 디스크립터를 연결 객체와 이벤트 등록 소유권에 편입합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Server 계약은 있었지만 accepted fd를 non-blocking 연결과 event registry에 넣지 못했습니다. |
| 주요 문제와 설계 판단 | accept를 would-block까지 반복하고 소켓을 설정한 뒤 Connection을 만들고 event backend에 Read 관심을 등록하고 map에 이동시킨 다음 connect 콜백을 호출했습니다. |
| 변경 파일·symbol | `src/Server.cpp — acceptReadyClients, socket configuration helpers` |
| 상태 / 소유권 변화 | accept된 raw fd는 Connection 생성 이후 그 객체로, map 삽입 이후 Server로 소유권이 이전됩니다. |
| 실패 또는 경계 | accept EINTR은 재시도하고 EAGAIN/EWOULDBLOCK은 drain 종료입니다. 설정/등록/삽입/콜백 예외는 catch 경로로 전달되지만이 버전은 event add와 map 삽입의 완전한 되돌리기 및 콜백 self-removal 안전성이 부족합니다. |
| 보장 / 비보장 | 보장: listener 준비 상태 하나에서 accept queue를 drain하고 live fd를 event loop에 등록합니다.<br>비보장: 등록 성공 뒤 raw 포인터를 콜백 이후 재사용할 수 있어 후속 reentrancy fix 대상입니다. |
| 다음 관련 변화 | 5dcd882f0763가 map-first 삽입, add 되돌리기, fd relookup으로 수명 결함을 수정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `4ad1227e5119` | `src/Server.cpp` | `Server::acceptReadyClients` | accept drain과 fd→Connection→event/map/콜백 순서 |

- 조사 방법: GitHub에서 `4ad1227e5119` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `378d5304828d` — `feat(server): 준비 이벤트 루프 구동`. 원문 역할은 “이식 가능한 이벤트 루프를 실제로 동작하게 만듭니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.13 `378d5304828d` — `feat(server): 준비 이벤트 루프 구동`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | CORE, ARCH, EVENT_IO |
| 원문에서 정한 역할 | 이식 가능한 이벤트 루프를 실제로 동작하게 만듭니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> 이식 가능한 이벤트 루프를 실제로 동작하게 만듭니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | listener와 연결 등록은 가능했지만 공통 준비 상태를 지속 처리하는 reactor가 없었습니다. |
| 실패 / 경계 | listener error/hangup은 server-level 실패이며 클라이언트 error는 해당 fd disconnect로 수렴합니다. stop 요청을 확인해 더 이상 dispatch하지 않는 경계가 있습니다. |
| 핵심 결정 | `start()`, `run()`, `pollOnce()`를 연결해 listener event는 accept로, 클라이언트 event는 read/write/error/hangup 처리로 분배했습니다. |
| 변경 파일·symbol | `src/Server.cpp — start, run, pollOnce, handleClientEvent` |
| 소유권 / lifecycle / 상태 변화 | single-threaded loop가 backend wait 결과를 순서대로 처리하고 콜백을 통해 complete lines를 애플리케이션에 넘깁니다. |
| 이 커밋이 보장하는 것 | portable backend 위에서 동작하는 실제 reactor와 연결-scoped 실패 isolation을 제공합니다. |
| 아직 보장하지 않는 것 | 콜백이 현재 연결을 삭제할 때 보유 포인터를 재사용하는 위험과 관심 상태 update 실패 되돌리기는 남아 있습니다. |
| 후속 fix/테스트 연결 | 625ffc924de8이 출력 준비 상태를, 7a6bc7e1276a가 authoritative 정리를 추가합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | listener와 연결 등록은 가능했지만 공통 준비 상태를 지속 처리하는 reactor가 없었습니다. |
| 2. 실패 conditions / 경계 | listener error/hangup은 server-level 실패이며 클라이언트 error는 해당 fd disconnect로 수렴합니다. stop 요청을 확인해 더 이상 dispatch하지 않는 경계가 있습니다. |
| 3. 선택한 표현과 순서 | `start()`, `run()`, `pollOnce()`를 연결해 listener event는 accept로, 클라이언트 event는 read/write/error/hangup 처리로 분배했습니다. |
| 4. authoritative 상태 | single-threaded loop가 backend wait 결과를 순서대로 처리하고 콜백을 통해 complete lines를 애플리케이션에 넘깁니다. |
| 5. 결과 불변식 | portable backend 위에서 동작하는 실제 reactor와 연결-scoped 실패 isolation을 제공합니다. |
| 6. 후속 보강 | 625ffc924de8이 출력 준비 상태를, 7a6bc7e1276a가 authoritative 정리를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `378d5304828d` | `src/Server.cpp` | `Server::pollOnce` | 공통 Event에서 listener/클라이언트 분기 |
| `378d5304828d` | `src/Server.cpp` | `Server::handleClientEvent` | read lines, write flush, error/hangup dispatch |

- 조사 방법: GitHub에서 `378d5304828d` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `625ffc924de8` — `feat(server): 송신 큐와 쓰기 관심 상태 연결`. 원문 역할은 “전송 대기 데이터, 쓰기 준비 이벤트, 종료 완료 조건을 연결합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.14 `625ffc924de8` — `feat(server): 송신 큐와 쓰기 관심 상태 연결`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | EVENT_IO, LIFECYCLE, INTEGRATION |
| 원문에서 정한 역할 | 전송 대기 데이터, 쓰기 준비 이벤트, 종료 완료 조건을 연결합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 전송 대기 데이터, 쓰기 준비 이벤트, 종료 완료 조건을 연결합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection은 대기 중인 출력을 보유했지만 server backend의 Write 관심과 자동 동기화되지 않았습니다. |
| 주요 문제와 설계 판단 | `sendTo()`/`queueRawTo()`가 queue 후 `refreshInterest()`를 호출하고, 대기 중인 bytes가 있을 때만 Write 관심을 추가했습니다. closeRequested 상태는 대기 중인이 비면 disconnect하도록 했습니다. |
| 변경 파일·symbol | `src/Server.cpp — sendTo, queueRawTo, refreshInterest, writable handling` |
| 상태 / 소유권 변화 | Connection의 `wantsWrite()`가 kernel write 관심 상태의 단일 입력이고 drain 완료가 graceful close completion 조건입니다. |
| 실패 또는 경계 | queue나 flush 오류는 close/disconnect로 수렴합니다. 이 버전의 update 실패 처리는 예외 후 stale 소유권을 남길 수 있어 후속 fix가 필요합니다. |
| 보장 / 비보장 | 보장: busy-loop 없이 대기 중인 출력이 있을 때만 writable notification을 요청하고, queued final response를 보낸 뒤 닫을 수 있습니다.<br>비보장: 관심 상태 update가 콜백/정리를 유발한 뒤 reference를 다시 쓰는 문제는 5dcd882f0763 이전에 남습니다. |
| 다음 관련 변화 | 7a6bc7e1276a가 disconnect 종착점을 만들고 5dcd882f0763가 refresh를 fd 기반으로 수정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `625ffc924de8` | `src/Server.cpp` | `Server::refreshInterest` | 대기 중인/close 상태를 Read/Write 관심 상태와 연결 |
| `625ffc924de8` | `src/Server.cpp` | `Server::sendTo/queueRawTo` | queue 결과와 backend 관심 갱신 연결 |

- 조사 방법: GitHub에서 `625ffc924de8` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `7a6bc7e1276a` — `feat(server): 연결 해제와 오류 정리 구현`. 원문 역할은 “전송 계층 연결 제거와 연결 종료 통지를 한 경로로 모읍니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.15 `7a6bc7e1276a` — `feat(server): 연결 해제와 오류 정리 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | LIFECYCLE, RISK |
| 원문에서 정한 역할 | 전송 계층 연결 제거와 연결 종료 통지를 한 경로로 모읍니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 전송 계층 연결 제거와 연결 종료 통지를 한 경로로 모읍니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 여러 event/error/stop 경로에서 연결 제거 순서가 하나의 함수로 고정되지 않았습니다. |
| 주요 문제와 설계 판단 | `disconnect()`가 backend remove를 시도하고 `unique_ptr`를 map 밖 local로 옮겨 entry를 먼저 erase한 뒤 disconnect 콜백을 호출했습니다. bulk close는 fd 스냅샷을 사용했습니다. |
| 변경 파일·symbol | `src/Server.cpp — disconnect, closeAllConnections, handleClientEvent` |
| 상태 / 소유권 변화 | 콜백 시점에는 server map에서 fd가 이미 사라졌지만 local unique_ptr가 콜백 동안 객체 수명을 유지하고 콜백 후 소멸자가 fd를 닫습니다. |
| 실패 또는 경계 | backend remove와 콜백 exception은 report되더라도 object 소멸을 막지 않습니다. hangup은 대기 중인 출력이 없을 때 terminal입니다. |
| 보장 / 비보장 | 보장: recursive lookup/repeated disconnect는 이미 제거된 상태를 보고, map iteration 중 erase로 반복자가 깨지지 않습니다.<br>비보장: 콜백 전에 잡은 raw 포인터를 콜백 뒤 다시 사용하는 call site와 event add/update 되돌리기는 아직 안전하지 않습니다. |
| 다음 관련 변화 | 5dcd882f0763/928594ec160c가 이 baseline의 reentrancy와 되돌리기 결함을 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::disconnect` | event remove → map 소유권 move/erase → 콜백 → 소멸 순서 |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::closeAllConnections` | fd 스냅샷으로 반복 erase 안전성 |

- 조사 방법: GitHub에서 `7a6bc7e1276a` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| 공통 준비 상태 의미 | `8e8a3db87950` | d3f74e2857da, 769cd3094f71, 2284a0e0d8bb, f1320f357bca | 각 backend의 native registration/notification을 같은 `Event` 의미로 번역합니다. | 별도 런타임 테스트는 이 개발 흐름에 없고 416efc91e580의 두 OS CI로 이어집니다. |
| fd 단일 소유권 | `8864f253ac15` | d71a2549c0eb, 4ad1227e5119, 7a6bc7e1276a | Connection move와 map/local unique_ptr 수명으로 close 소유자를 하나로 유지합니다. | 5dcd882f0763/928594ec160c가 등록 되돌리기와 콜백 수명을 보강합니다. |
| incremental 입력 framing | `9b601b69de4f` | b00589d4b1b1 | complete line만 반환하고 incomplete 뒷부분을 보존하며 oversize를 terminal로 만듭니다. | 6b4a7738a285의 fragmented TCP smoke가 통합 경로를 확인합니다. |
| 영속 partial 출력 | `a10fe961e2b1` | 625ffc924de8 | writeOffset와 Write 관심 상태를 결합해 unsent 뒷부분을 event 사이에 보존합니다. | 881e59734a9a/f34ab135c546가 오버플로와 모든 send outcome을 수정·검증합니다. |
| authoritative 정리 | `e6492e27cc30` | 378d5304828d, 7a6bc7e1276a | event remove → map erase → 콜백 → 파일 디스크립터 소멸의 종착점을 만듭니다. | 5dcd882f0763/928594ec160c가 reentrant call site와 되돌리기를 고정합니다. |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| 한 send가 전부 완료된다는 가정 | short send/backpressure에서 뒷부분 유실·중복 | a10fe961e2b1 → 881e59734a9a | 영속 offset과 오버플로-safe admission/return guard | f34ab135c546 |
| 콜백 뒤 기존 포인터가 유효하다는 가정 | self-disconnect 뒤 stale Connection 접근 | 7a6bc7e1276a → 5dcd882f0763 | erase-before-콜백과 fd relookup | 928594ec160c |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| native 준비 상태 registration/translation | 없음 | `EventManager` backend | 8e8a3db87950 → f1320f357bca | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 소켓·입력·출력·close progress | raw fd 중심 | move-only `Connection` | 8864f253ac15 → a10fe961e2b1 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| listener와 연결 orchestration | 구성 요소 분리 | `Server`/`pollOnce()` | e6492e27cc30 → 378d5304828d | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| write 준비 상태 정책 | Connection queue만 존재 | `Server::refreshInterest()` | 625ffc924de8 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 파일 디스크립터 removal/notification | 분산 가능성 | `Server::disconnect()` | 7a6bc7e1276a | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: portable 준비 상태 의미와 backend 수명 계약이 아직 존재하지 않았습니다.
- 마지막 커밋 `7a6bc7e1276a` 시점의 상태: recursive lookup/repeated disconnect는 이미 제거된 상태를 보고, map iteration 중 erase로 반복자가 깨지지 않습니다.
- 개발 흐름 안에서 강화된 핵심 불변식: 공통 준비 상태 의미, fd 단일 소유권, incremental 입력 framing, 영속 partial 출력, authoritative 정리.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: 콜백 전에 잡은 raw 포인터를 콜백 뒤 다시 사용하는 call site와 event add/update 되돌리기는 아직 안전하지 않습니다. 5dcd882f0763/928594ec160c가 이 baseline의 reentrancy와 되돌리기 결함을 수정·검증합니다.
- 최종 설명: `8e8a3db87950`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `7a6bc7e1276a`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| backend wait | 769cd3094f71 / f1320f357bca | `Kqueue/EpollEventManager::wait` | 공통 `Event` vector | EINTR은 빈 결과, 다른 wait 오류는 예외 |
| event dispatch | 378d5304828d | `Server::pollOnce` | listener는 accept, 클라이언트는 handleClientEvent | event error는 연결-scoped disconnect |
| read path | b00589d4b1b1 / 378d5304828d | `Server::handleClientEvent` | Connection::readAvailable → line 콜백 | EOF/hard error/close 요청은 disconnect |
| write path | a10fe961e2b1 / 625ffc924de8 | `Server writable branch` | Connection::flushPending → offset/관심 상태 갱신 | would-block은 뒷부분 보존, hard error는 disconnect |
| 정리 | 7a6bc7e1276a | `Server::disconnect` | backend remove → map erase/local 소유자 → 콜백 | 예외를 report해도 local 소유자 소멸자가 fd close |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 01-portable-readiness-and-nonblocking-transport.md =====

===== BEGIN FILE: 02-protocol-boundary-identity-and-registration.md =====
# 개발 흐름 02 — 프로토콜 구문 경계, 사용자 식별, 등록

부제: 프로토콜 경계, 식별자와 등록

## 1. 개발 흐름의 목표

wire line을 구조화 message로 바꾸는 문법 경계, 파일 디스크립터 기반 클라이언트 상태와 닉네임 index, `IrcApplication` 책임 분리, PASS/NICK/USER 등록 gate가 하나의 실행 경로로 결합되는 과정을 복원합니다.

### 원문에서 정한 의의

바이트 스트림 분리, IRC 문법 해석, 연결 식별 정보, 등록 권한 검사를 이벤트 루프에 섞지 않고 분리합니다. 애플리케이션 조정 계층과 등록 전 명령 차단이 핵심이며, 실제 TCP 스모크 테스트로 프레이밍·파싱·닉네임 색인·응답·수명 전이가 함께 동작하는지 확인합니다.

<details>
<summary>영문 원문</summary>

> The sequence separates syntax, connection identity, and registration authorization rather than mixing them into the event loop. The application boundary and registration gate are the durable decisions; the individual registration commands are normal implementations within those decisions. The first smoke suite then verifies that framing, parsing, identity indexes, replies, and lifecycle transitions compose correctly.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- transport framing과 IRC grammar parsing은 어디에서 분리되는가?
- display 닉네임과 canonical lookup key는 왜 별도로 유지되는가?
- `Server` 콜백에서 protocol 상태를 생성·조회·삭제하는 authoritative 애플리케이션 경계는 어디인가?
- syntax-valid 명령이 registration 상태 때문에 거부되는 지점은 어디인가?
- PASS/NICK/USER prerequisite와 001–003 welcome의 단일 transition은 어떻게 보장되는가?
- 첫 real-TCP smoke가 어떤 계층 연결을 증명하고 무엇은 아직 정확히 고정하지 않는가?

### 원문에서 확인되는 불변식

- IRC 등록은 비밀번호, 닉네임, USER 정보가 모두 준비된 뒤에만 완료되어야 합니다.
- 닉네임은 레지스트리의 표준화된 조회 규칙에 따라 유일해야 하며, 표시용 철자와 역방향 색인 키가 일치해야 합니다.
- 형식이 잘못되었거나 너무 긴 프레임은 이전 구조화 상태를 남기거나 이후 유효한 명령으로 처리되어서는 안 됩니다.

### 원문에서 확인되는 구현 난점

- 바이트 스트림 프레임 분리와 IRC 문법·권한 검사를 서로 다른 역할로 유지하는 문제.
- 콜백 수명과 파일 디스크립터 기준 프로토콜 상태를 연결하면서 유령 상태나 오래된 색인을 만들지 않는 문제.

## 3. 완료 기준

- 한 TCP line이 parser, normalized 명령, registration gate, handler, reply queue까지 이동하는 흐름을 해당 SHA 코드로 설명할 수 있습니다.
- 파일 디스크립터 map과 닉네임 reverse index의 consistency 조건 및 collision-before-변경 순서를 설명할 수 있습니다.
- transport 연결과 IRC registration을 같은 상태로 취급하지 않고 각 lifecycle 필드를 구분할 수 있습니다.
- registration prerequisites와 welcome reply ordering을 상태 변화로 정리했습니다.
- smoke 테스트의 실제 실행 경로, 테스트 방식, 증명/비증명 범위를 구분했습니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `a22bf6ddbd75` | `feat(parser): IRC 메시지 값과 직렬화 정의` | B | IRC_PROTOCOL | 구조화된 IRC 메시지 표현을 도입합니다. |
| 2 | `c31a32b6cb24` | `feat(parser): IRC 한 줄 구문 해석 구현` | A | IRC_PROTOCOL, RISK | 한 줄 문법과 매개변수 파싱 경계를 정의합니다. |
| 3 | `aeb1e9b709b9` | `feat(reply): IRC 서버 응답 생성` | B | IRC_PROTOCOL | 표준 서버 메시지·숫자 응답·호스트마스크 생성을 추가합니다. |
| 4 | `991b76b8d793` | `feat(client): 연결별 등록 상태 저장` | B | IDENTITY | 디스크립터별 등록 상태를 저장합니다. |
| 5 | `b47135c51cfc` | `feat(client): 닉네임 색인 관리` | A | IDENTITY, RISK | 대소문자를 구분하지 않는 닉네임 색인 불변식을 정합니다. |
| 6 | `0b5ae6aef328` | `feat(app): IRC 동작 조율 계약 정의` | S | CORE, ARCH, INTEGRATION | `Server` 위에서 프로토콜 상태와 명령을 조정하는 `IrcApplication`을 정의합니다. |
| 7 | `2ed9331124bb` | `feat(app): 연결 수명 콜백 조율` | A | LIFECYCLE, IRC_PROTOCOL, INTEGRATION | 전송 계층 수명 콜백을 클라이언트 상태·파싱·정리에 연결합니다. |
| 8 | `035e1137e0dd` | `feat(app): 등록 전 명령 분배 구현` | A | IRC_PROTOCOL, IDENTITY, RISK | 등록 전 명령 허용 규칙을 한곳에서 적용합니다. |
| 9 | `582317254e24` | `feat(registration): PASS 인증 상태 처리` | B | IDENTITY, IRC_PROTOCOL | 비밀번호 인증 상태를 추가합니다. |
| 10 | `80a639321bad` | `feat(registration): 닉네임 검증과 색인 갱신` | B | IDENTITY, IRC_PROTOCOL | 닉네임 검증과 충돌 처리를 추가합니다. |
| 11 | `d9e420b570a0` | `feat(registration): USER 정보와 환영 응답 연결` | B | IDENTITY, IRC_PROTOCOL | PASS/NICK/USER 등록과 환영 응답을 완성합니다. |
| 12 | `6b4a7738a285` | `test(smoke): 실제 TCP 등록과 채널 흐름 검증` | A | VERIFICATION, INTEGRATION, RISK | 실제 TCP에서 등록과 명령 처리 경로가 함께 동작함을 검증합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `a22bf6ddbd75` — `feat(parser): IRC 메시지 값과 직렬화 정의`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | IRC_PROTOCOL |
| 원문에서 정한 역할 | 구조화된 IRC 메시지 표현을 도입합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 구조화된 IRC 메시지 표현을 도입합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `IrcMessage`에 optional 앞부분, uppercase 명령, ordered params, raw 프레임을 분리하고 safe indexed access와 `toLine()`을 구현했습니다. |
| 확인한 변경 파일·symbol | `include/IrcMessage.hpp — IrcMessage`<br>`src/IrcMessage.cpp — normalization, param access, toLine` |
| 핵심 상태 / 브랜치 | parser/handler는 명령 casing을 따로 다루지 않고 message 값 하나를 공유합니다. |
| 실패 handling | out-of-range parameter access는 unchecked indexing 대신 fallback을 돌려 handler의 UB를 막습니다. |
| 보장과 다음 연결 | 마지막 parameter의 공백/colon/empty 조건에 trailing marker를 적용하고 CRLF 프레임을 만듭니다. c31a32b6cb24가 line parser를, aeb1e9b709b9가 server reply helper를 추가합니다. |
| 이 시점의 한계 | wire text를 구조화하는 parse grammar는 아직 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a22bf6ddbd75` | `src/IrcMessage.cpp` | `IrcMessage::toLine` | 앞부분/params/trailing 규칙과 CRLF 직렬화 |
| `a22bf6ddbd75` | `src/IrcMessage.cpp` | `command normalization/param` | uppercase dispatch key와 safe fallback |

- 조사 방법: GitHub에서 `a22bf6ddbd75` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `c31a32b6cb24` — `feat(parser): IRC 한 줄 구문 해석 구현`. 원문 역할은 “한 줄 문법과 매개변수 파싱 경계를 정의합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `c31a32b6cb24` — `feat(parser): IRC 한 줄 구문 해석 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | IRC_PROTOCOL, RISK |
| 원문에서 정한 역할 | 한 줄 문법과 매개변수 파싱 경계를 정의합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 한 줄 문법과 매개변수 파싱 경계를 정의합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 값 객체는 있었지만 raw IRC line의 앞부분/명령/parameter 문법을 일관되게 해석하지 못했습니다. |
| 주요 문제와 설계 판단 | 프레임 terminator를 제거하고 출력을 초기화한 뒤 optional 앞부분, 필수 명령, middle/trailing parameters를 순서대로 파싱했습니다. 명령을 uppercase로 정규화하고 raw를 보존했습니다. |
| 변경 파일·symbol | `src/IrcMessage.cpp — parseLine` |
| 상태 / 소유권 변화 | 성공 시 출력이 한 프레임의 완전한 structured 상태가 되고 실패 시 이전 상태가 남지 않습니다. |
| 실패 또는 경계 | empty line, 앞부분 뒤 명령 누락, 510-byte 초과 등 malformed 경계를 false로 반환합니다. |
| 보장 / 비보장 | 보장: trailing colon 이후 공백 포함 text를 한 parameter로 보존하고 packet framing과 grammar parsing을 분리합니다.<br>비보장: authorization·registration 상태에 따른 명령 허용은 애플리케이션 단계에 없습니다. |
| 다음 관련 변화 | 035e1137e0dd가 parse 성공 후 registration gate를 적용합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c31a32b6cb24` | `src/IrcMessage.cpp` | `parseLine` | 출력 reset, 앞부분/명령/middle/trailing parameter 순서와 length 검사 |

- 조사 방법: GitHub에서 `c31a32b6cb24` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `aeb1e9b709b9` — `feat(reply): IRC 서버 응답 생성`. 원문 역할은 “표준 서버 메시지·숫자 응답·호스트마스크 생성을 추가합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `aeb1e9b709b9` — `feat(reply): IRC 서버 응답 생성`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | IRC_PROTOCOL |
| 원문에서 정한 역할 | 표준 서버 메시지·숫자 응답·호스트마스크 생성을 추가합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 표준 서버 메시지·숫자 응답·호스트마스크 생성을 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `Replies`에 numeric 코드 formatting, generic message, ERROR, hostmask helper를 두고 대상이 없을 때 `*`를 사용하도록 했습니다. |
| 확인한 변경 파일·symbol | `include/Replies.hpp`<br>`src/Replies.cpp — numeric, formatMessage, error, hostmask` |
| 핵심 상태 / 브랜치 | server response formatting의 authoritative 위치가 reply helper로 이동합니다. |
| 실패 handling | 마지막 parameter만 trailing 처리해 중간 parameter의 공백이 wire grammar를 깨뜨리지 않도록 호출자가 구조화 값을 넘기게 합니다. |
| 보장과 다음 연결 | numeric width와 앞부분/대상/CRLF 형태가 handler마다 달라지지 않습니다. IrcApplication handlers가 이 helper를 통해 Connection 출력 queue로 응답합니다. |
| 이 시점의 한계 | 실제 send 실패와 lifecycle invalidation은 이 helper가 다루지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `aeb1e9b709b9` | `src/Replies.cpp` | `Replies::numeric/formatMessage` | server 앞부분, 코드, 대상, trailing serialization 중앙화 |

- 조사 방법: GitHub에서 `aeb1e9b709b9` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `991b76b8d793` — `feat(client): 연결별 등록 상태 저장`. 원문 역할은 “디스크립터별 등록 상태를 저장합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `991b76b8d793` — `feat(client): 연결별 등록 상태 저장`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | IDENTITY |
| 원문에서 정한 역할 | 디스크립터별 등록 상태를 저장합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 디스크립터별 등록 상태를 저장합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `ClientState`와 `ClientRegistry`를 만들고 fd-keyed map, create-on-demand `state()`, non-creating `find()`, contains/fds/erase를 제공했습니다. |
| 확인한 변경 파일·symbol | `src/ClientRegistry.hpp — ClientState, ClientRegistry`<br>`src/ClientRegistry.cpp` |
| 핵심 상태 / 브랜치 | 소켓 수명은 Server/Connection이, IRC identity/registration 수명은 ClientRegistry가 같은 fd를 key로 각각 관리합니다. |
| 실패 handling | `find()`는 phantom 상태를 만들지 않고 `erase()`는 없는 fd에도 안전합니다. |
| 보장과 다음 연결 | transport connected와 IRC registered를 다른 상태로 표현할 기반이 생깁니다. b47135c51cfc가 canonical 닉네임 index와 erase consistency를 추가합니다. |
| 이 시점의 한계 | 닉네임 uniqueness를 위한 reverse index는 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `991b76b8d793` | `src/ClientRegistry.cpp` | `state/find/erase/fds` | 생성 조회와 비생성 조회, 스냅샷, 멱등 erase 분리 |

- 조사 방법: GitHub에서 `991b76b8d793` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `b47135c51cfc` — `feat(client): 닉네임 색인 관리`. 원문 역할은 “대소문자를 구분하지 않는 닉네임 색인 불변식을 정합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `b47135c51cfc` — `feat(client): 닉네임 색인 관리`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | IDENTITY, RISK |
| 원문에서 정한 역할 | 대소문자를 구분하지 않는 닉네임 색인 불변식을 정합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 대소문자를 구분하지 않는 닉네임 색인 불변식을 정합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | fd→클라이언트 상태만 있어 닉네임 lookup이 linear하거나 대소문자 collision을 놓칠 수 있었습니다. |
| 주요 문제와 설계 판단 | canonical 닉네임→fd reverse map을 추가하고 `setNickname()`이 old canonical key를 먼저 제거한 뒤 new key를 삽입하도록 했습니다. 클라이언트 erase는 reverse key도 함께 제거합니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp/.cpp — canonical nickname index, setNickname, findByNick, erase`<br>`include/Channel.hpp — canonicalNick helper` |
| 상태 / 소유권 변화 | display spelling은 ClientState에, lookup key는 canonical map에 있으며 두 표현이 한 변경 API에서 함께 갱신됩니다. |
| 실패 또는 경계 | collision 여부를 변경 전에 확인할 수 있고 erase 후 stale 닉네임이 다른 fd를 가리키지 않습니다. |
| 보장 / 비보장 | 보장: case-insensitive uniqueness와 fd↔닉네임 index consistency를 유지할 수 있습니다.<br>비보장: 실제 NICK validation/collision numeric은 handler에 아직 없습니다. |
| 다음 관련 변화 | 80a639321bad가 validation과 collision-before-변경을 명령 경로에 적용합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `b47135c51cfc` | `src/ClientRegistry.cpp` | `ClientRegistry::setNickname` | old reverse key 제거 후 상태/new key 갱신 |
| `b47135c51cfc` | `src/ClientRegistry.cpp` | `findByNick/erase` | canonical lookup과 lifecycle 정리 |

- 조사 방법: GitHub에서 `b47135c51cfc` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `0b5ae6aef328` — `feat(app): IRC 동작 조율 계약 정의`. 원문 역할은 “`Server` 위에서 프로토콜 상태와 명령을 조정하는 `IrcApplication`을 정의합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `0b5ae6aef328` — `feat(app): IRC 동작 조율 계약 정의`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | CORE, ARCH, INTEGRATION |
| 원문에서 정한 역할 | `Server` 위에서 프로토콜 상태와 명령을 조정하는 `IrcApplication`을 정의합니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> `Server` 위에서 프로토콜 상태와 명령을 조정하는 `IrcApplication`을 정의합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | Server 콜백과 parser/registry/명령 handler를 묶는 protocol 소유자가 없었습니다. |
| 실패 / 경계 | 콜백 경계가 transport removal을 protocol 정리로 전달할 위치를 고정합니다. |
| 핵심 결정 | `IrcApplication`이 Server reference, password/런타임/server name, ClientRegistry와 명령 handlers를 소유하는 coordinator 계약을 정의했습니다. |
| 변경 파일·symbol | `src/IrcApplication.hpp — IrcApplication public callbacks and private handlers` |
| 소유권 / lifecycle / 상태 변화 | Server는 transport만 소유하고 애플리케이션은 IRC/domain 상태와 dispatch를 소유합니다. |
| 이 커밋이 보장하는 것 | event loop에 parser/authorization/채널 상태를 직접 섞지 않는 architecture 경계가 생깁니다. |
| 아직 보장하지 않는 것 | 콜백 구현과 registration gate는 아직 없습니다. |
| 후속 fix/테스트 연결 | 2ed9331124bb와 035e1137e0dd가 lifecycle와 dispatch를 구현합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | Server 콜백과 parser/registry/명령 handler를 묶는 protocol 소유자가 없었습니다. |
| 2. 실패 conditions / 경계 | 콜백 경계가 transport removal을 protocol 정리로 전달할 위치를 고정합니다. |
| 3. 선택한 표현과 순서 | `IrcApplication`이 Server reference, password/런타임/server name, ClientRegistry와 명령 handlers를 소유하는 coordinator 계약을 정의했습니다. |
| 4. authoritative 상태 | Server는 transport만 소유하고 애플리케이션은 IRC/domain 상태와 dispatch를 소유합니다. |
| 5. 결과 불변식 | event loop에 parser/authorization/채널 상태를 직접 섞지 않는 architecture 경계가 생깁니다. |
| 6. 후속 보강 | 2ed9331124bb와 035e1137e0dd가 lifecycle와 dispatch를 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0b5ae6aef328` | `src/IrcApplication.hpp` | `onConnect/onLine/onDisconnect/onTick` | transport 콜백에서 protocol coordinator로 진입하는 공개 경계 |
| `0b5ae6aef328` | `src/IrcApplication.hpp` | `_clients and handler declarations` | protocol 상태/dispatch 소유권 |

- 조사 방법: GitHub에서 `0b5ae6aef328` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `2ed9331124bb` — `feat(app): 연결 수명 콜백 조율`. 원문 역할은 “전송 계층 수명 콜백을 클라이언트 상태·파싱·정리에 연결합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `2ed9331124bb` — `feat(app): 연결 수명 콜백 조율`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | LIFECYCLE, IRC_PROTOCOL, INTEGRATION |
| 원문에서 정한 역할 | 전송 계층 수명 콜백을 클라이언트 상태·파싱·정리에 연결합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 전송 계층 수명 콜백을 클라이언트 상태·파싱·정리에 연결합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 애플리케이션 계약은 있었지만 Connection lifecycle와 ClientRegistry가 연결되지 않았습니다. |
| 주요 문제와 설계 판단 | onConnect가 ClientState를 만들고 host/fd/time을 기록하며, onLine이 필요 시 상태를 복구하고 parse 후 dispatch하며, onDisconnect가 애플리케이션 상태를 정리하도록 했습니다. |
| 변경 파일·symbol | `src/IrcApplication.cpp — onConnect, onLine, onDisconnect` |
| 상태 / 소유권 변화 | transport 콜백의 fd가 protocol registry key로 이어지고 disconnect 콜백이 상태 수명의 종착점입니다. |
| 실패 또는 경계 | parse 실패는 명령 handler로 넘어가지 않으며 없는 상태는 연결 시점 정보로 생성됩니다. |
| 보장 / 비보장 | 보장: 연결 lifecycle와 parser/registry 정리가 한 coordinator를 통과합니다.<br>비보장: 응답 send가 동기 disconnect를 일으킨 뒤 borrowed 상태를 재사용하는 문제는 후속 08 thread에 남습니다. |
| 다음 관련 변화 | 035e1137e0dd가 parsed 명령에 registration gate를 적용합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `2ed9331124bb` | `src/IrcApplication.cpp` | `onConnect/onLine/onDisconnect` | fd별 상태 생성 → parse/dispatch → 정리 lifecycle |

- 조사 방법: GitHub에서 `2ed9331124bb` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `035e1137e0dd` — `feat(app): 등록 전 명령 분배 구현`. 원문 역할은 “등록 전 명령 허용 규칙을 한곳에서 적용합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `035e1137e0dd` — `feat(app): 등록 전 명령 분배 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | IRC_PROTOCOL, IDENTITY, RISK |
| 원문에서 정한 역할 | 등록 전 명령 허용 규칙을 한곳에서 적용합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 등록 전 명령 허용 규칙을 한곳에서 적용합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | syntax-valid 명령이 registration 전후 동일하게 dispatch될 수 있었습니다. |
| 주요 문제와 설계 판단 | PASS/NICK/USER/PING/QUIT는 등록 전 허용하고, 그 외 명령은 unregistered면 451, registered인데 unsupported면 421로 분리하는 central gate를 구현했습니다. |
| 변경 파일·symbol | `src/IrcApplication.cpp — handleMessage` |
| 상태 / 소유권 변화 | ClientState.registered가 syntax parsing 뒤 authorization/dispatch 상태를 결정합니다. |
| 실패 또는 경계 | 등록 전 금지 명령은 handler side effect 없이 numeric으로 끝납니다. |
| 보장 / 비보장 | 보장: 각 handler가 registration check를 중복하지 않고 단일 정책을 적용합니다.<br>비보장: PASS/NICK/USER prerequisite 상태와 welcome transition 구현은 다음 커밋에 있습니다. |
| 다음 관련 변화 | 582317254e24, 80a639321bad, d9e420b570a0가 gate를 통과하는 등록 상태를 채웁니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `035e1137e0dd` | `src/IrcApplication.cpp` | `IrcApplication::handleMessage` | pre-registration allowlist와 451/421 분기 |

- 조사 방법: GitHub에서 `035e1137e0dd` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `582317254e24` — `feat(registration): PASS 인증 상태 처리`. 원문 역할은 “비밀번호 인증 상태를 추가합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `582317254e24` — `feat(registration): PASS 인증 상태 처리`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | IDENTITY, IRC_PROTOCOL |
| 원문에서 정한 역할 | 비밀번호 인증 상태를 추가합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 비밀번호 인증 상태를 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `handlePass()`가 재등록 462, parameter 부족 461, 잘못된 password 464+close를 처리하고 성공 시 `passOk`를 세운 뒤 `maybeRegister()`를 호출했습니다. |
| 확인한 변경 파일·symbol | `src/RegistrationCommands.cpp — handlePass` |
| 핵심 상태 / 브랜치 | ClientState.passOk가 registration prerequisite 중 하나가 됩니다. |
| 실패 handling | 잘못된 password는 상태를 승인하지 않고 protocol error를 queue한 뒤 연결 close를 요청합니다. |
| 보장과 다음 연결 | password가 정확히 확인되기 전 registered transition이 일어나지 않습니다. 80a639321bad와 d9e420b570a0가 나머지 prerequisites를 추가합니다. |
| 이 시점의 한계 | 닉네임/USER prerequisite가 아직 없으므로 PASS만으로 등록되지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `582317254e24` | `src/RegistrationCommands.cpp` | `IrcApplication::handlePass` | 462/461/464 브랜치와 passOk 변경-before-maybeRegister |

- 조사 방법: GitHub에서 `582317254e24` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `80a639321bad` — `feat(registration): 닉네임 검증과 색인 갱신`. 원문 역할은 “닉네임 검증과 충돌 처리를 추가합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.10 `80a639321bad` — `feat(registration): 닉네임 검증과 색인 갱신`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | IDENTITY, IRC_PROTOCOL |
| 원문에서 정한 역할 | 닉네임 검증과 충돌 처리를 추가합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 닉네임 검증과 충돌 처리를 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | NICK missing/invalid/collision을 431/432/433으로 거부하고, canonical collision 검사를 마친 뒤에만 registry 닉네임을 갱신했습니다. |
| 확인한 변경 파일·symbol | `src/RegistrationCommands.cpp — handleNick, nickname validation` |
| 핵심 상태 / 브랜치 | hasNick/display nick/reverse index가 성공 브랜치에서만 함께 전진합니다. |
| 실패 handling | collision 전에 old mapping을 제거하지 않으므로 실패가 기존 identity를 보존합니다. |
| 보장과 다음 연결 | case-insensitive uniqueness와 validation-before-변경 ordering을 명령 경로에서 지킵니다. d9e420b570a0가 USER와 final registration transition을 연결합니다. |
| 이 시점의 한계 | 이미 registered 클라이언트의 NICK fan-out/정리 semantics는 채널 thread에서 추가됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `80a639321bad` | `src/RegistrationCommands.cpp` | `IrcApplication::handleNick` | 431/432/433 검사 후 registry 변경 |

- 조사 방법: GitHub에서 `80a639321bad` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `d9e420b570a0` — `feat(registration): USER 정보와 환영 응답 연결`. 원문 역할은 “PASS/NICK/USER 등록과 환영 응답을 완성합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.11 `d9e420b570a0` — `feat(registration): USER 정보와 환영 응답 연결`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | IDENTITY, IRC_PROTOCOL |
| 원문에서 정한 역할 | PASS/NICK/USER 등록과 환영 응답을 완성합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> PASS/NICK/USER 등록과 환영 응답을 완성합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | USER fields를 저장하고 `maybeRegister()`가 `passOk && hasNick && hasUser`일 때만 `registered=true`로 만든 뒤 001, 002, 003을 순서대로 queue했습니다. |
| 확인한 변경 파일·symbol | `src/RegistrationCommands.cpp — handleUser, maybeRegister` |
| 핵심 상태 / 브랜치 | 등록은 세 prerequisite의 conjunction에서 false→true로 한 번 전이하며 welcome send 전에 flag를 먼저 세워 재진입 중 중복 등록을 막습니다. |
| 실패 handling | parameter 부족/이미 등록은 numeric으로 거부합니다. 다만 welcome enqueue가 연결을 제거해도이 버전은 이후 send/log를 계속할 수 있습니다. |
| 보장과 다음 연결 | PASS/NICK/USER가 모두 충족된 클라이언트만 registered가 되고 welcome 순서가 정해집니다. 6b4a7738a285가 실제 TCP에서 정상 등록을 통합 검증하고 08 thread가 실패 semantics를 보강합니다. |
| 이 시점의 한계 | response 실패 후 애플리케이션 상태 revalidation은 728aaabc4012/5edcafda8a4d 이전에 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d9e420b570a0` | `src/RegistrationCommands.cpp` | `IrcApplication::maybeRegister` | 세 prerequisite conjunction, registered 변경, 001→002→003 |

- 조사 방법: GitHub에서 `d9e420b570a0` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `6b4a7738a285` — `test(smoke): 실제 TCP 등록과 채널 흐름 검증`. 원문 역할은 “실제 TCP에서 등록과 명령 처리 경로가 함께 동작함을 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.12 `6b4a7738a285` — `test(smoke): 실제 TCP 등록과 채널 흐름 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, INTEGRATION, RISK |
| 원문에서 정한 역할 | 실제 TCP에서 등록과 명령 처리 경로가 함께 동작함을 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 실제 TCP에서 등록과 명령 처리 경로가 함께 동작함을 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | parser·registry·registration·채널 기능이 실제 프로세스/소켓 안에서 함께 동작한다는 통합 증거가 없었습니다. |
| 주요 문제와 설계 판단 | Makefile 테스트/smoke, shell runner, Python peer를 추가해 loopback server를 실행하고 다중 클라이언트 시나리오를 수행했습니다. |
| 변경 파일·symbol | `Makefile — test, smoke`<br>`tests/irc_smoke.sh`<br>`tools/irc_smoke_client.py` |
| 상태 / 소유권 변화 | 테스트 harness가 프로세스 PID, free port, temp log, peer sockets와 수신 line buffer를 소유하고 EXIT trap으로 정리합니다. |
| 실패 또는 경계 | wrong password, fragmented 입력, case-insensitive collision, 채널/mode/message/departure 오류를 실제 wire에서 관찰합니다. |
| 보장 / 비보장 | 보장: transport framing부터 애플리케이션/채널/출력 queue까지 주요 정상·일부 오류 경로가 한 프로세스에서 조합됩니다.<br>비보장: 검사문은 주로 substring 중심이어서 exact 전체 프레임/order/CRLF와 rare syscall 실패, capacity/fairness를 증명하지 않습니다. |
| 다음 관련 변화 | e5e6c57db80d가 exact 공개 규약으로 정밀도를 높이고 계층별 결정적 테스트가 후속됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `6b4a7738a285` | `tests/irc_smoke.sh` | `runner and cleanup trap` | 실제 server 프로세스 기동·준비 상태·로그·종료 관리 |
| `6b4a7738a285` | `tools/irc_smoke_client.py` | `IrcPeer and scenario` | fragmented write와 registration/채널/messaging 시나리오 |

- 조사 방법: GitHub에서 `6b4a7738a285` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | Connection framing, parser, registration, 닉네임 index, 채널 상태와 queued send가 실제 프로세스에서 결합됩니다. |
| 재현 실패 / 경계 | wrong password, fragmented PING, 닉네임 collision, JOIN/TOPIC/PRIVMSG/INVITE/MODE/KICK/PART/QUIT입니다. |
| 테스트 방식 | compiled server + real loopback TCP + shell 프로세스 harness + Python peer의 broad end-to-end smoke입니다. |
| 통과하는 실제 실행 경로 | 프로세스 startup → Server event loop → Connection framing → parser → IrcApplication → registry/채널 → 출력 queue입니다. |
| 이 테스트가 증명하는 것 | 대표 정상·오류 흐름이 실제 소켓에서 연결되어 진행됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | exact 모든 프레임, 모든 ordering, rare send/event 실패, formal capacity·fairness는 증명하지 않습니다. |
| 테스트 성격 | broad 통합 smoke |
| 막는 회귀 | packet 경계를 명령 경계로 오인하거나 identity/채널 fan-out 통합이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `6b4a7738a285`
- repository에 정의된 실행 명령: `make test` 또는 `make smoke`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| wire syntax value | `a22bf6ddbd75` | c31a32b6cb24, aeb1e9b709b9 | structured value, parser reset/grammar, reply serializer를 분리합니다. | 6b4a7738a285가 실제 fragmented wire와 response 조합을 확인합니다. |
| 파일 디스크립터-keyed registration 상태 | `991b76b8d793` | 2ed9331124bb | transport lifecycle와 별도 ClientState를 fd로 연결합니다. | 5edcafda8a4d가 response 실패 정리를 보강합니다. |
| canonical 닉네임 uniqueness | `b47135c51cfc` | 80a639321bad | collision-before-변경과 erase 시 reverse index 정리를 지킵니다. | 6b4a7738a285의 case-insensitive collision scenario |
| transport/protocol 책임 분리 | `0b5ae6aef328` | 2ed9331124bb, 035e1137e0dd | Server 콜백 위에서 parser·registry·dispatch를 애플리케이션이 소유합니다. | 728aaabc4012/5edcafda8a4d가 send reentrancy를 보강합니다. |
| registration monotonic transition | `582317254e24` | 80a639321bad, d9e420b570a0 | PASS/NICK/USER conjunction에서 registered를 한 번 세우고 001→002→003을 queue합니다. | 6b4a7738a285 정상 흐름, 5edcafda8a4d 실패 흐름 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| handler별 명령 casing/serialization | dispatch와 wire 도형 불일치 | a22bf6ddbd75 + c31a32b6cb24 + aeb1e9b709b9 | structured parser/reply 경계 | 6b4a7738a285 |
| display nick만 비교 | case variant collision·stale reverse mapping | b47135c51cfc + 80a639321bad | canonical collision-before-변경 | 6b4a7738a285 |
| welcome send는 실패하지 않음 | 등록 중 synchronous disconnect 후 stale 상태 | 728aaabc4012 | send bool 전파와 상태 relookup | 5edcafda8a4d |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| 프레임 text ↔ structured message | handler별 문자열 | `IrcMessage`/parser/reply helper | a22bf6ddbd75 → aeb1e9b709b9 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 연결별 IRC identity | transport와 혼합 가능 | `ClientRegistry` | 991b76b8d793 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 닉네임 lookup/uniqueness | linear/display-only 가능 | canonical reverse index | b47135c51cfc | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| protocol orchestration | Server에 혼합 가능 | `IrcApplication` | 0b5ae6aef328 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| registration authorization | handler별 검사 가능 | central gate + `maybeRegister()` | 035e1137e0dd → d9e420b570a0 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: IRC line을 구조화 값과 wire serialization으로 표현하는 공통 타입이 없었습니다.
- 마지막 커밋 `6b4a7738a285` 시점의 상태: transport framing부터 애플리케이션/채널/출력 queue까지 주요 정상·일부 오류 경로가 한 프로세스에서 조합됩니다.
- 개발 흐름 안에서 강화된 핵심 불변식: wire syntax value, 파일 디스크립터-keyed registration 상태, canonical 닉네임 uniqueness, transport/protocol 책임 분리, registration monotonic transition.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: 검사문은 주로 substring 중심이어서 exact 전체 프레임/order/CRLF와 rare syscall 실패, capacity/fairness를 증명하지 않습니다. e5e6c57db80d가 exact 공개 규약으로 정밀도를 높이고 계층별 결정적 테스트가 후속됩니다.
- 최종 설명: `a22bf6ddbd75`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `6b4a7738a285`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| complete line | 2ed9331124bb | `Server line callback` | IrcApplication::onLine | missing 상태는 생성, disconnect는 정리 |
| grammar | c31a32b6cb24 | `IrcApplication::onLine` | parseLine → IrcMessage | malformed/oversize는 dispatch하지 않음 |
| gate | 035e1137e0dd | `handleMessage` | ClientState.registered와 allowlist | 미등록 금지 451, 등록 후 미지원 421 |
| registration | 582317254e24 / 80a639321bad / d9e420b570a0 | `PASS/NICK/USER handlers` | maybeRegister conjunction → registered | invalid/collision/password 실패는 변경 없이 numeric/close |
| welcome 출력 | d9e420b570a0 | `maybeRegister` | 001 → 002 → 003 → Server::sendTo | send 실패 revalidation은 728aaabc4012에서 보강 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 02-protocol-boundary-identity-and-registration.md =====

===== BEGIN FILE: 03-channel-authority-fanout-and-cleanup.md =====
# 개발 흐름 03 — 채널 상태의 권위, 다중 전파, 정리

부제: 채널 권한, 팬아웃과 정리

## 1. 개발 흐름의 목표

Channel aggregate가 membership·operator·invitation·topic·mode를 authoritative하게 유지하고, 애플리케이션이 fan-out과 identity/departure 정리를 통해 클라이언트/index/채널 graph를 일관되게 만드는 과정을 복원합니다.

### 원문에서 정한 의의

채널 입장, 메시지 전파, 닉네임 변경, 명시적 퇴장, KICK, 전송 연결 종료가 모두 같은 멤버·운영자·초대 상태를 사용해야 합니다. 모든 종료 경로 뒤 클라이언트·닉네임·채널 상태가 수렴하고 오래된 파일 디스크립터나 중복 알림이 남지 않도록 정리 경로를 확립합니다.

<details>
<summary>영문 원문</summary>

> Channel functionality becomes reliable only when the same state model governs admission, fan-out, identity changes, explicit departures, kicks, and transport disconnects. The S-level cleanup commit is the decisive point: it makes client, nickname, and channel state converge after every termination path and prevents stale descriptors or duplicate peer notifications.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- operator가 항상 member라는 불변식은 aggregate 어느 연산에서 강제되는가?
- 채널 lookup과 create-on-demand는 어떤 명령에서 분리되는가?
- 여러 shared 채널을 가진 peer에게 NICK/QUIT를 한 번만 보내는 recipient 계산은 무엇인가?
- NICK, PART, QUIT, transport disconnect에서 old identity와 membership을 어느 시점까지 유지하는가?
- JOIN의 validation, invitation consumption, first-member operator, broadcast/topic/NAMES 순서는 무엇인가?
- MODE `+i`, `+t`, `+o`/`-o`가 authorization과 parameter cursor를 어떻게 유지하는가?

### 원문에서 확인되는 불변식

- 채널 멤버·운영자·초대·빈 채널 수명 상태는 클라이언트와 닉네임 상태에 맞아야 합니다.
- 연결이 끊긴 파일 디스크립터는 어떤 채널에도 남아서는 안 되며, 공통 상대는 식별 정보 변경이나 QUIT 전이를 최대 한 번만 받아야 합니다.
- 초대 전용 입장, 주제 보호, KICK, 초대, 운영자 모드는 멤버 여부와 운영자 권한을 강제해야 합니다.

### 원문에서 확인되는 구현 난점

- NICK·PART·KICK·QUIT·전송 실패·다중 전파·빈 채널 삭제를 하나의 공유 상태 관계에서 일관되게 처리하는 문제.
- 방송 중 수명 상태가 바뀌어 비소유 채널·클라이언트 참조가 사라질 수 있는 재진입 문제.

## 3. 완료 기준

- `Channel`의 모든 authoritative 상태와 non-owning 클라이언트 identifier 관계를 정리했습니다.
- member/operator/invitation/topic/mode 변경을 실제 aggregate method와 명령 handler caller로 연결했습니다.
- deduplicated common-peer fan-out과 disconnect 정리 순서를 스냅샷/erase 시점까지 설명합니다.
- JOIN과 MODE의 권한 및 변경-before-broadcast ordering을 해당 SHA 코드로 확인했습니다.
- send 실패와 reentrant 정리 한계는 개발 흐름 08의 후속 fix/테스트와 연결했습니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `786ed5d839d9` | `feat(channel): 채널 상태 계약 정의` | A | ARCH, CHANNEL_STATE | 채널 상태의 권위 있는 소유자인 `Channel`을 정의합니다. |
| 2 | `966f5663dbcc` | `feat(channel): 구성원과 운영자 상태 관리` | B | CHANNEL_STATE | 멤버와 운영자 상태를 구현합니다. |
| 3 | `0d0d850f007d` | `feat(channel): 주제·초대·모드와 이름 규칙 구현` | B | CHANNEL_STATE, IRC_PROTOCOL | 주제·초대·모드·채널 이름 규칙을 구현합니다. |
| 4 | `c1762e011fd6` | `feat(channel): 채널 탐색과 대상 해석 지원` | B | CHANNEL_STATE | 애플리케이션이 소유하는 채널 저장소와 명령 조회를 추가합니다. |
| 5 | `46e2b7785bee` | `feat(channel): 채널 방송 대상 팬아웃 지원` | A | CHANNEL_STATE, INTEGRATION | 중복을 제거한 채널·공통 상대 전파 기능을 재사용 가능하게 만듭니다. |
| 6 | `a147d6994d58` | `feat(channel): 구성원 정리와 식별자 변경 방송` | S | CORE, CHANNEL_STATE, LIFECYCLE | 닉네임 변경, PART, QUIT, 멤버 제거, 빈 채널 삭제를 일관되게 처리합니다. |
| 7 | `7ac793d3b695` | `feat(message): 채널 대상 메시지 방송` | B | IRC_PROTOCOL, CHANNEL_STATE | 멤버 권한을 확인한 뒤 채널 `PRIVMSG`를 전파합니다. |
| 8 | `22e5f82bc693` | `feat(channel): JOIN 채널 입장 처리` | A | CHANNEL_STATE, IRC_PROTOCOL, RISK | JOIN 상태 변경, 초대 전용 입장, 첫 멤버 운영자 지정을 구현합니다. |
| 9 | `0e601da7d2bc` | `feat(channel): 채널 모드 조회와 i·t 변경` | B | CHANNEL_STATE, IRC_PROTOCOL | 운영자 권한 아래 `+i`와 `+t`를 조회·변경합니다. |
| 10 | `bce6933f69ab` | `feat(channel): 채널 운영자 모드 변경` | B | CHANNEL_STATE, IRC_PROTOCOL | 채널 권한 관리를 `+o`와 `-o`로 확장합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `786ed5d839d9` — `feat(channel): 채널 상태 계약 정의`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | ARCH, CHANNEL_STATE |
| 원문에서 정한 역할 | 채널 상태의 권위 있는 소유자인 `Channel`을 정의합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 채널 상태의 권위 있는 소유자인 `Channel`을 정의합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 애플리케이션에 채널별 구성원·운영자·초대·주제·모드를 함께 보존하는 authoritative aggregate가 없었습니다. |
| 주요 문제와 설계 판단 | `Channel`이 이름, member fd set, operator fd set, canonical 닉네임 invitation set, topic 존재/문자열, `+i`·`+t` 상태를 비공개 field로 소유하고 변경/query API를 제공하도록 계약을 정의했습니다. |
| 변경 파일·symbol | `include/Channel.hpp — Channel state and public mutation/query API` |
| 상태 / 소유권 변화 | 채널은 클라이언트 fd를 비소유 식별자로만 보관합니다. 소켓과 `Connection`의 수명은 계속 `Server`가 소유하고 `Channel`은 membership graph만 소유합니다. |
| 실패 또는 경계 | 컨테이너를 handler에 직접 노출하지 않아 비회원 운영자, casing이 다른 invitation, topic presence와 text의 불일치를 aggregate 내부에서 막을 수 있게 했습니다. |
| 보장 / 비보장 | 보장: membership·authority·invitation·topic·mode를 한 객체에서 갱신할 수 있는 단일 상태 소유자가 생깁니다.<br>비보장: 각 변경의 실제 구현과 명령 authorization은 아직 없습니다. |
| 다음 관련 변화 | 966f5663dbcc가 member/operator 불변식을, 0d0d850f007d가 topic/invite/mode/name 규칙을 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `786ed5d839d9` | `include/Channel.hpp` | `Channel private fields` | member/operator/invitation/topic/mode의 authoritative 저장 위치 |
| `786ed5d839d9` | `include/Channel.hpp` | `member and mode API` | handler가 컨테이너를 직접 수정하지 않는 aggregate 경계 |

- 조사 방법: GitHub에서 `786ed5d839d9` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `966f5663dbcc` — `feat(channel): 구성원과 운영자 상태 관리`. 원문 역할은 “멤버와 운영자 상태를 구현합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `966f5663dbcc` — `feat(channel): 구성원과 운영자 상태 관리`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | CHANNEL_STATE |
| 원문에서 정한 역할 | 멤버와 운영자 상태를 구현합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 멤버와 운영자 상태를 구현합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | member 추가·삭제와 operator 승격·해제를 aggregate method로 구현했습니다. `removeMember()`는 같은 fd를 operator set에서도 지우고 `setOperator(true)`는 현재 member인 fd만 허용했습니다. |
| 확인한 변경 파일·symbol | `src/Channel.cpp — addMember, removeMember, hasMember, setOperator, isOperator, members` |
| 핵심 상태 / 브랜치 | operator set은 member set의 부분집합으로 유지되며 `members()`는 fan-out과 안전한 iteration에 사용할 값 스냅샷을 반환합니다. |
| 실패 handling | 비회원에게 운영자 권한을 부여하려는 요청은 상태를 변경하지 않으며, 퇴장한 fd가 operator로 남지 않습니다. |
| 보장과 다음 연결 | “operator는 항상 member”라는 핵심 채널 불변식이 모든 aggregate 변경에서 유지됩니다. 0d0d850f007d가 나머지 채널 상태를 구현하고 bce6933f69ab가 `+o/-o` 명령에 연결합니다. |
| 이 시점의 한계 | invitation·topic·mode와 실제 MODE authorization은 아직 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `966f5663dbcc` | `src/Channel.cpp` | `Channel::removeMember` | member와 operator 상태를 함께 제거 |
| `966f5663dbcc` | `src/Channel.cpp` | `Channel::setOperator` | 비회원 운영자 승격을 거부 |

- 조사 방법: GitHub에서 `966f5663dbcc` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `0d0d850f007d` — `feat(channel): 주제·초대·모드와 이름 규칙 구현`. 원문 역할은 “주제·초대·모드·채널 이름 규칙을 구현합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `0d0d850f007d` — `feat(channel): 주제·초대·모드와 이름 규칙 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | CHANNEL_STATE, IRC_PROTOCOL |
| 원문에서 정한 역할 | 주제·초대·모드·채널 이름 규칙을 구현합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 주제·초대·모드·채널 이름 규칙을 구현합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | topic presence/text, canonical 닉네임 invitation, invite-only와 topic-protected flag, 채널 이름 검증 및 닉네임 canonicalization을 구현했습니다. |
| 확인한 변경 파일·symbol | `src/Channel.cpp — topic, invitation, mode, name/canonical helpers` |
| 핵심 상태 / 브랜치 | 초대는 display spelling이 아니라 canonical key로 저장되고, topic은 text와 존재 여부를 함께 표현합니다. |
| 실패 handling | 잘못된 채널 name은 생성 전에 거부할 수 있고, clearInvite는 없는 초대에도 안전하며 mode query는 현재 aggregate 상태만 읽습니다. |
| 보장과 다음 연결 | 입장·TOPIC·MODE 명령이 의존할 authoritative 채널 policy 상태가 완성됩니다. c1762e011fd6가 애플리케이션-owned 채널 map을, 22e5f82bc693과 0e601da7d2bc가 명령 transition을 추가합니다. |
| 이 시점의 한계 | 애플리케이션 map과 명령 lookup/authorization은 아직 연결되지 않았습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0d0d850f007d` | `src/Channel.cpp` | `Channel::canonicalNick / invite / clearInvite` | case-insensitive invitation 상태 |
| `0d0d850f007d` | `src/Channel.cpp` | `topic and mode methods` | topic presence와 +i/+t 변경을 aggregate에 집중 |

- 조사 방법: GitHub에서 `0d0d850f007d` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `c1762e011fd6` — `feat(channel): 채널 탐색과 대상 해석 지원`. 원문 역할은 “애플리케이션이 소유하는 채널 저장소와 명령 조회를 추가합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `c1762e011fd6` — `feat(channel): 채널 탐색과 대상 해석 지원`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | CHANNEL_STATE |
| 원문에서 정한 역할 | 애플리케이션이 소유하는 채널 저장소와 명령 조회를 추가합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 애플리케이션이 소유하는 채널 저장소와 명령 조회를 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `IrcApplication`에 채널 map을 추가하고 create-on-demand `ensureChannel()`과 non-creating lookup/명령 helper를 분리했습니다. 명령 helper는 존재·membership 요구에 따라 403/442를 보냅니다. |
| 확인한 변경 파일·symbol | `src/IrcApplication.hpp — _channels`<br>`src/ApplicationSupport.cpp — ensureChannel, findChannelForCommand, eraseChannelIfEmpty` |
| 핵심 상태 / 브랜치 | 채널 object 수명의 소유자는 `IrcApplication::_channels`이고 handler가 보유한 포인터/reference는 해당 map entry가 존재하는 동안만 유효합니다. |
| 실패 handling | 조회 명령이 실수로 채널을 생성하지 않으며, 없는 채널과 비회원 접근이 변경 전에 종료됩니다. |
| 보장과 다음 연결 | 채널 생성과 조회의 의미가 명령별로 구분되고 empty-채널 erase 종착점이 생깁니다. 46e2b7785bee가 map의 member snapshots를 이용한 fan-out을 추가하고 728aaabc4012가 send 뒤 relookup 규칙을 보강합니다. |
| 이 시점의 한계 | broadcast 중 map entry가 지워지는 reentrancy는 아직 방어하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c1762e011fd6` | `src/ApplicationSupport.cpp` | `ensureChannel` | create-on-demand가 필요한 명령만 map entry 생성 |
| `c1762e011fd6` | `src/ApplicationSupport.cpp` | `findChannelForCommand` | 403/442를 변경 전 공통 검사 |

- 조사 방법: GitHub에서 `c1762e011fd6` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `46e2b7785bee` — `feat(channel): 채널 방송 대상 팬아웃 지원`. 원문 역할은 “중복을 제거한 채널·공통 상대 전파 기능을 재사용 가능하게 만듭니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `46e2b7785bee` — `feat(channel): 채널 방송 대상 팬아웃 지원`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | CHANNEL_STATE, INTEGRATION |
| 원문에서 정한 역할 | 중복을 제거한 채널·공통 상대 전파 기능을 재사용 가능하게 만듭니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 중복을 제거한 채널·공통 상대 전파 기능을 재사용 가능하게 만듭니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | handler마다 채널 member를 직접 순회해야 했고 여러 공통 채널을 가진 peer에게 동일 event가 중복될 수 있었습니다. |
| 주요 문제와 설계 판단 | 한 채널의 `members()` 스냅샷을 순회하는 broadcast와, sender가 속한 모든 채널의 peer fd를 `std::set<int>`에 합치는 common-peer fan-out을 구현했습니다. |
| 변경 파일·symbol | `src/ApplicationSupport.cpp — broadcastToChannel, broadcastToCommonPeers` |
| 상태 / 소유권 변화 | fan-out 도중 순회 기준은 live 컨테이너 반복자가 아니라 fd 스냅샷이며 recipient dedup set이 한 transition당 최대 한 번 전송을 보장합니다. |
| 실패 또는 경계 | 없는 클라이언트나 연결로 보내는 개별 실패는 server send 결과와 정리로 넘어가며 다른 recipient 계산은 스냅샷에 의해 계속 결정됩니다. |
| 보장 / 비보장 | 보장: NICK/QUIT 같은 identity transition이 shared 채널 수와 무관하게 같은 peer에게 한 번만 전달될 수 있습니다.<br>비보장: send가 sender나 채널을 동기적으로 지울 수 있다는 cross-layer invalidation은 아직 처리하지 않습니다. |
| 다음 관련 변화 | a147d6994d58가 identity/departure 정리를 이 fan-out 위에 구현하고 08 thread가 reentrancy를 수정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `46e2b7785bee` | `src/ApplicationSupport.cpp` | `broadcastToChannel` | member fd 스냅샷 기반 채널 fan-out |
| `46e2b7785bee` | `src/ApplicationSupport.cpp` | `broadcastToCommonPeers` | std::set recipient dedup으로 중복 notification 방지 |

- 조사 방법: GitHub에서 `46e2b7785bee` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `a147d6994d58` — `feat(channel): 구성원 정리와 식별자 변경 방송`. 원문 역할은 “닉네임 변경, PART, QUIT, 멤버 제거, 빈 채널 삭제를 일관되게 처리합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `a147d6994d58` — `feat(channel): 구성원 정리와 식별자 변경 방송`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | CORE, CHANNEL_STATE, LIFECYCLE |
| 원문에서 정한 역할 | 닉네임 변경, PART, QUIT, 멤버 제거, 빈 채널 삭제를 일관되게 처리합니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> 닉네임 변경, PART, QUIT, 멤버 제거, 빈 채널 삭제를 일관되게 처리합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | 채널 fan-out은 있었지만 NICK/PART/QUIT/transport disconnect가 클라이언트 index와 모든 membership을 하나의 순서로 정리하지 못했습니다. |
| 실패 / 경계 | 여러 shared 채널을 가진 peer는 dedup set 때문에 QUIT/NICK을 한 번만 받으며 transport disconnect도 explicit QUIT과 같은 정리 primitives를 재사용합니다. |
| 핵심 결정 | 공통 peer를 먼저 계산하고 old identity 앞부분으로 NICK/QUIT/PART를 방송한 뒤, 각 채널 membership을 제거하고 empty 채널을 erase하며 마지막에 registry 닉네임/클라이언트 상태를 갱신·삭제하도록 수명 전이를 조율했습니다. |
| 변경 파일·symbol | `src/ApplicationSupport.cpp — partAllChannels, partChannel, removeClientFromChannels, broadcastToCommonPeers`<br>`src/RegistrationCommands.cpp — registered NICK transition`<br>`src/IrcApplication.cpp — QUIT/disconnect cleanup` |
| 소유권 / lifecycle / 상태 변화 | old nick와 membership은 outbound event를 구성할 때까지 유지되고, recipient 스냅샷 이후 채널 graph와 닉네임 index/클라이언트 record가 수렴합니다. |
| 이 커밋이 보장하는 것 | 퇴장한 fd가 채널/operator set이나 닉네임 reverse index에 남지 않고 empty 채널이 map에 잔존하지 않습니다. |
| 아직 보장하지 않는 것 | broadcast 중 send 실패가 map/클라이언트를 지운 뒤 borrowed reference를 사용하는 위험은 728aaabc4012 이전에 남습니다. |
| 후속 fix/테스트 연결 | 22e5f82bc693 등 정상 명령이 같은 graph를 확장하고 08 thread의 fix/테스트가 reentrant 정리를 강화합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | 채널 fan-out은 있었지만 NICK/PART/QUIT/transport disconnect가 클라이언트 index와 모든 membership을 하나의 순서로 정리하지 못했습니다. |
| 2. 실패 conditions / 경계 | 여러 shared 채널을 가진 peer는 dedup set 때문에 QUIT/NICK을 한 번만 받으며 transport disconnect도 explicit QUIT과 같은 정리 primitives를 재사용합니다. |
| 3. 선택한 표현과 순서 | 공통 peer를 먼저 계산하고 old identity 앞부분으로 NICK/QUIT/PART를 방송한 뒤, 각 채널 membership을 제거하고 empty 채널을 erase하며 마지막에 registry 닉네임/클라이언트 상태를 갱신·삭제하도록 수명 전이를 조율했습니다. |
| 4. authoritative 상태 | old nick와 membership은 outbound event를 구성할 때까지 유지되고, recipient 스냅샷 이후 채널 graph와 닉네임 index/클라이언트 record가 수렴합니다. |
| 5. 결과 불변식 | 퇴장한 fd가 채널/operator set이나 닉네임 reverse index에 남지 않고 empty 채널이 map에 잔존하지 않습니다. |
| 6. 후속 보강 | 22e5f82bc693 등 정상 명령이 같은 graph를 확장하고 08 thread의 fix/테스트가 reentrant 정리를 강화합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a147d6994d58` | `src/ApplicationSupport.cpp` | `partAllChannels / removeClientFromChannels` | 채널 스냅샷 → membership 제거 → empty erase 순서 |
| `a147d6994d58` | `src/RegistrationCommands.cpp` | `registered NICK handling` | old 앞부분 fan-out 후 canonical index 갱신 |
| `a147d6994d58` | `src/IrcApplication.cpp` | `QUIT/onDisconnect cleanup` | explicit/transport 종료가 동일 상태 graph를 정리 |

- 조사 방법: GitHub에서 `a147d6994d58` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `7ac793d3b695` — `feat(message): 채널 대상 메시지 방송`. 원문 역할은 “멤버 권한을 확인한 뒤 채널 `PRIVMSG`를 전파합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `7ac793d3b695` — `feat(message): 채널 대상 메시지 방송`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | IRC_PROTOCOL, CHANNEL_STATE |
| 원문에서 정한 역할 | 멤버 권한을 확인한 뒤 채널 `PRIVMSG`를 전파합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 멤버 권한을 확인한 뒤 채널 `PRIVMSG`를 전파합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 채널 대상이면 채널 존재와 sender membership을 확인하고, 성공 시 sender를 제외한 member 스냅샷에 원본 hostmask의 PRIVMSG를 방송했습니다. |
| 확인한 변경 파일·symbol | `src/MessagingCommands.cpp — handlePrivmsg channel branch` |
| 핵심 상태 / 브랜치 | message text는 채널 aggregate를 변경하지 않고 현재 membership 스냅샷만 읽습니다. |
| 실패 handling | 없는 채널은 403, 비회원 발신은 404로 끝나며 fan-out side effect가 없습니다. |
| 보장과 다음 연결 | 채널 메시지는 authorized member에서 현재 member들로만 전달되고 sender echo는 제외됩니다. de1dd0fc30d0가 실제 slow receiver 아래 unrelated 연결 progress를 검증합니다. |
| 이 시점의 한계 | slow receiver와 send 실패 격리는 출력/server layer에 맡기며이 커밋 자체는 fairness를 보장하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `7ac793d3b695` | `src/MessagingCommands.cpp` | `IrcApplication::handlePrivmsg` | 채널 존재·membership 검사 후 sender 제외 fan-out |

- 조사 방법: GitHub에서 `7ac793d3b695` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `22e5f82bc693` — `feat(channel): JOIN 채널 입장 처리`. 원문 역할은 “JOIN 상태 변경, 초대 전용 입장, 첫 멤버 운영자 지정을 구현합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `22e5f82bc693` — `feat(channel): JOIN 채널 입장 처리`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | CHANNEL_STATE, IRC_PROTOCOL, RISK |
| 원문에서 정한 역할 | JOIN 상태 변경, 초대 전용 입장, 첫 멤버 운영자 지정을 구현합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> JOIN 상태 변경, 초대 전용 입장, 첫 멤버 운영자 지정을 구현합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 채널 aggregate와 helper는 있었지만 JOIN의 admission·membership·첫 운영자·응답 순서가 한 transition으로 결합되지 않았습니다. |
| 주요 문제와 설계 판단 | JOIN 0은 전체 PART로 처리하고, 쉼표 목록의 각 이름을 검증한 뒤 채널을 확보합니다. invite-only admission을 확인하고 첫 member만 operator로 추가하며 초대를 소비한 뒤 JOIN 방송, TOPIC reply, NAMES reply 순서로 전송했습니다. |
| 변경 파일·symbol | `src/ChannelCommands.cpp — handleJoin`<br>`src/ApplicationSupport.cpp — sendTopicReply, sendNames` |
| 상태 / 소유권 변화 | membership 변경은 JOIN fan-out 전에 완료되고 invitation은 성공 입장 시에만 제거됩니다. 첫 member 여부는 변경 전 `empty()` 결과로 결정됩니다. |
| 실패 또는 경계 | invalid name은 403, invite-only 미초대는 473이며 기존 member의 재JOIN은 상태를 중복 추가하지 않고 NAMES만 보냅니다. |
| 보장 / 비보장 | 보장: 입장 authorization과 first-member operator 불변식, 응답 순서가 하나의 handler에서 고정됩니다.<br>비보장: 각 send 뒤 actor/채널 수명 재검증은 728aaabc4012 이전에 부족합니다. |
| 다음 관련 변화 | 0e601da7d2bc와 bce6933f69ab가 입장 후 operator가 변경할 채널 mode를 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `22e5f82bc693` | `src/ChannelCommands.cpp` | `IrcApplication::handleJoin` | validation → admission → addMember(first) → invite clear → JOIN/TOPIC/NAMES 순서 |

- 조사 방법: GitHub에서 `22e5f82bc693` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `0e601da7d2bc` — `feat(channel): 채널 모드 조회와 i·t 변경`. 원문 역할은 “운영자 권한 아래 `+i`와 `+t`를 조회·변경합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `0e601da7d2bc` — `feat(channel): 채널 모드 조회와 i·t 변경`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | CHANNEL_STATE, IRC_PROTOCOL |
| 원문에서 정한 역할 | 운영자 권한 아래 `+i`와 `+t`를 조회·변경합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 운영자 권한 아래 `+i`와 `+t`를 조회·변경합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | MODE 조회는 324와 current mode string을 보내고, 변경은 membership과 operator를 확인한 뒤 mode 문자열을 순회해 `+i/-i`, `+t/-t`를 aggregate에 반영하고 방송했습니다. |
| 확인한 변경 파일·symbol | `src/ChannelCommands.cpp — handleChannelMode, broadcastMode` |
| 핵심 상태 / 브랜치 | adding/removing cursor가 mode string 전체에 적용되고 각 성공 변경 직후 현재 채널 members에게 mode event를 fan-out합니다. |
| 실패 handling | 비회원 442, 비운영자 482, 알 수 없는 mode 472로 변경 없이 거부합니다. |
| 보장과 다음 연결 | invite-only와 topic-protection policy를 채널 operator만 변경할 수 있습니다. bce6933f69ab가 argument를 소비하는 `o` mode를 추가하고 d48e1f1f8c04/aee5edebe294가 continuation semantics를 수정·검증합니다. |
| 이 시점의 한계 | compound mode 중 첫 broadcast 실패 뒤 다음 mode가 계속 적용되는 문제는 d48e1f1f8c04까지 남습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0e601da7d2bc` | `src/ChannelCommands.cpp` | `handleChannelMode +i/+t` | membership/operator 검사, mode cursor, 변경 후 broadcast |

- 조사 방법: GitHub에서 `0e601da7d2bc` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `bce6933f69ab` — `feat(channel): 채널 운영자 모드 변경`. 원문 역할은 “채널 권한 관리를 `+o`와 `-o`로 확장합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.10 `bce6933f69ab` — `feat(channel): 채널 운영자 모드 변경`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | CHANNEL_STATE, IRC_PROTOCOL |
| 원문에서 정한 역할 | 채널 권한 관리를 `+o`와 `-o`로 확장합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 채널 권한 관리를 `+o`와 `-o`로 확장합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | mode cursor와 별도의 parameter cursor를 사용해 `+o/-o` 대상 nick을 소비하고, 닉네임 lookup과 채널 membership을 확인한 뒤 `Channel::setOperator()`를 호출해 실제 display nick으로 방송했습니다. |
| 확인한 변경 파일·symbol | `src/ChannelCommands.cpp — handleChannelMode o branch` |
| 핵심 상태 / 브랜치 | operator 상태는 Channel aggregate만 변경하고 닉네임은 ClientRegistry에서 lookup한 현재 display value를 사용합니다. |
| 실패 handling | 인자 부족은 461, 존재하지 않거나 비회원인 대상은 441이며 operator set을 변경하지 않습니다. |
| 보장과 다음 연결 | 운영자 권한 변경도 “operator는 member” 불변식과 명령 parameter ordering을 따릅니다. d48e1f1f8c04와 aee5edebe294가 response 실패 후 더 이상의 mode transition을 막습니다. |
| 이 시점의 한계 | send 실패 뒤 compound mode continuation과 대상 수명 revalidation은 08 thread에서 보강됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `bce6933f69ab` | `src/ChannelCommands.cpp` | `handleChannelMode o branch` | argument cursor, nick lookup/member 검사, setOperator와 broadcast |

- 조사 방법: GitHub에서 `bce6933f69ab` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| authoritative Channel aggregate | `786ed5d839d9` | 966f5663dbcc, 0d0d850f007d | member/operator/invitation/topic/mode를 한 객체에서 관리합니다. | 명령 통합은 22e5f82bc693, 0e601da7d2bc, bce6933f69ab |
| operator ⊆ member | `966f5663dbcc` | bce6933f69ab | removeMember가 operator도 제거하고 비회원 승격을 거부합니다. | aee5edebe294가 실패 중 unrelated member/채널 보존을 확인 |
| deduplicated peer fan-out | `46e2b7785bee` | a147d6994d58 | member 스냅샷과 std::set recipient로 shared peer 중복을 제거합니다. | 6b4a7738a285의 multi-클라이언트 flow |
| identity/departure graph 정리 | `a147d6994d58` | PART/QUIT/NICK/transport disconnect | old identity로 notification 후 membership/index/클라이언트 상태를 수렴시킵니다. | 728aaabc4012/5edcafda8a4d가 send-실패 reentrancy 보강 |
| authorized admission/mode | `22e5f82bc693` | 0e601da7d2bc, bce6933f69ab | invite/member/operator 검사를 변경 전에 수행합니다. | d48e1f1f8c04/aee5edebe294가 compound 실패 continuation 고정 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| handler가 여러 컨테이너를 직접 수정 | 비회원 operator·stale invite/index | 786ed5d839d9 → 0d0d850f007d | aggregate method 뒤에 상태 변경 집중 | 6b4a7738a285 |
| shared 채널마다 NICK/QUIT 전송 | 같은 peer 중복 notification | 46e2b7785bee + a147d6994d58 | recipient fd set dedup | 6b4a7738a285 |
| broadcast가 local side effect뿐 | send 중 actor/채널 erase 후 dangling reference | 728aaabc4012 | stable value copy와 컨테이너 relookup | 5edcafda8a4d / aee5edebe294 |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| 채널 membership/authority | 분산 컨테이너 가능 | `Channel` aggregate | 786ed5d839d9 → 0d0d850f007d | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 채널 object 수명 | 없음 | `IrcApplication::_channels` map | c1762e011fd6 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| recipient computation | handler별 반복 | broadcast helper/스냅샷/dedup set | 46e2b7785bee | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| identity/departure 정리 | 명령별 분산 가능 | IrcApplication 정리 helpers | a147d6994d58 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 소켓 수명 | Channel이 소유하지 않음 | Server/Connection, Channel에는 fd만 저장 | 전체 개발 흐름 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: 애플리케이션에 채널별 구성원·운영자·초대·주제·모드를 함께 보존하는 authoritative aggregate가 없었습니다.
- 마지막 커밋 `bce6933f69ab` 시점의 상태: 운영자 권한 변경도 “operator는 member” 불변식과 명령 parameter ordering을 따릅니다.
- 개발 흐름 안에서 강화된 핵심 불변식: authoritative Channel aggregate, operator ⊆ member, deduplicated peer fan-out, identity/departure graph 정리, authorized admission/mode.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: send 실패 뒤 compound mode continuation과 대상 수명 revalidation은 08 thread에서 보강됩니다. d48e1f1f8c04와 aee5edebe294가 response 실패 후 더 이상의 mode transition을 막습니다.
- 최종 설명: `786ed5d839d9`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `bce6933f69ab`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| 채널 resolve | c1762e011fd6 | `command handler` | ensureChannel/findChannelForCommand | invalid/missing/nonmember는 403/442 |
| authorize | 22e5f82bc693 / 0e601da7d2bc | `JOIN/MODE handler` | invite/member/operator 상태 | 473/482 등에서 변경 없이 종료 |
| mutate | 966f5663dbcc / 0d0d850f007d | `handler` | Channel aggregate methods | 비회원 operator와 stale operator를 방지 |
| fan-out | 46e2b7785bee | `application helper` | member 스냅샷/common-peer set | 개별 send 실패는 Server 정리로 전달 |
| departure 정리 | a147d6994d58 | `PART/QUIT/onDisconnect` | old 앞부분 notification → member 제거 → empty erase → registry 정리 | send reentrancy는 728aaabc4012에서 relookup |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 03-channel-authority-fanout-and-cleanup.md =====

===== BEGIN FILE: 04-operational-protections-and-controlled-shutdown.md =====
# 개발 흐름 04 — 운영 보호 장치와 통제된 종료

부제: 운영 보호와 제어된 종료

## 1. 개발 흐름의 목표

등록·유휴·PING·명령률·대기 중인 출력·연결 수를 bounded policy로 만들고, 그 상태를 metrics/log로 관찰하며 signal 이후 final 출력을 제한된 시간 안에 drain하는 공개 프로세스 동작을 복원합니다.

### 원문에서 정한 의의

기능만 수행하던 서버를 자원 사용량과 연결 생존 시간이 제한된 운영 가능한 프로세스로 확장합니다. 등록·유휴·PING·명령률·출력 대기열·연결 수 제한, 관측 지표와 로그, 신호 수신 뒤 제한된 시간의 출력 비우기를 공개 프로세스 동작으로 연결합니다.

<details>
<summary>영문 원문</summary>

> The server evolves from functional protocol handling into an operable bounded process. Each protection addresses a distinct resource or liveness risk, while shutdown and the executable contract define how those protections appear at the public process boundary. The sequence also provides the failure signals later used to expose reentrant cleanup defects.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- 각 timeout/limit은 어떤 authoritative 상태와 transition에서 적용되는가?
- 보호 정책 위반은 server 전체가 아니라 한 연결의 queued response와 close로 어떻게 수렴하는가?
- 대기 중인-출력 실패가 왜 이후 reentrant 정리 defect를 드러내는 signal이 되는가?
- counter와 gauge는 어느 계층이 기록하며 live 컨테이너와 중복 판단 기준을 만들지 않는가?
- signal handler와 normal control flow의 책임은 어떻게 분리되는가?
- exact executable 규약이 CLI, wire, log, shutdown 순서를 어디까지 고정하는가?

### 원문에서 확인되는 불변식

- 등록 제한, 유휴 상태, PING, 속도 제한 판단은 클라이언트별로 일관된 상태를 사용해야 합니다.
- 전송 대기 출력, 연결 수, 등록 시간, 명령 속도, 하트비트 대기는 상한이나 명시적인 처리 규칙을 가져야 합니다.
- 종료 과정은 연결을 정리하기 전에 마지막 프로토콜 오류 전송을 시도하되, 읽지 않는 상대 때문에 무한히 기다려서는 안 됩니다.

### 원문에서 확인되는 구현 난점

- 서로 다른 자원·연결 생존 위험을 한 번의 이벤트 루프 처리와 연결별 정리 경로에 통합하는 문제.
- heartbeat를 wall-clock movement 및 unrelated PONG에서 안전하게 만드는 문제는 후속 개발 흐름에서 복구됩니다.
- 외부 프로세스 동작을 환경에 취약하지 않으면서 정확하게 검증하는 문제.

## 3. 완료 기준

- registration/idle/ping/rate/출력/연결 limit마다 상태 field, check timing, rejection response, 정리 path를 정리했습니다.
- metrics/log가 사실이 확정되는 authoritative transition에 붙어 있음을 확인했습니다.
- SIGTERM/SIGINT에서 flag 변경, 애플리케이션 shutdown, bounded poll drain, 콜백 detach, stop 순서를 설명합니다.
- exact 규약 suite의 공개-경계 증명/비증명 범위를 기록했습니다.
- heartbeat/config/출력 실패의 후속 fix 개발 흐름과 연결했습니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `c4df44554866` | `feat(registration): 등록 대기 시간 제한` | B | RESILIENCE, LIFECYCLE | 등록 제한 시간을 추가합니다. |
| 2 | `764361c52b2a` | `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` | A | RESILIENCE, LIFECYCLE, RISK | 유휴 하트비트와 PING 응답 제한 상태를 추가합니다. |
| 3 | `9e2b214f9227` | `feat(throttle): 클라이언트별 명령 호출 횟수 제한` | B | RESILIENCE | 클라이언트별 명령 속도 제한 구간을 추가합니다. |
| 4 | `d7d85e518177` | `feat(buffer): 송신 대기열 크기 제한` | A | EVENT_IO, RESILIENCE, RISK | 연결별 미전송 출력 크기를 제한하고 대기열 추가 실패를 전달합니다. |
| 5 | `adb49d9466e4` | `feat(server): 최대 연결 수 제한` | B | RESILIENCE, EVENT_IO | 동시 연결 수를 제한합니다. |
| 6 | `e05e35ca7da9` | `feat(metrics): 서버 실행 지표 조회 기능 추가` | B | OBSERVABILITY | 연결·명령·메시지·대기열 폐기·속도 제한 상태를 조회할 수 있게 합니다. |
| 7 | `c34aa18f89af` | `feat(log): 연결 상태와 실행 지표 기록` | B | OBSERVABILITY, LIFECYCLE | 수명 주기·보호 동작·집계 지표 이벤트를 구조화해 기록합니다. |
| 8 | `dd04279c47fd` | `feat(shutdown): 종료 전 송신 대기열 처리` | A | LIFECYCLE, RESILIENCE, RISK | 시그널 즉시 중단 대신 애플리케이션 종료 절차와 출력 비우기를 수행합니다. |
| 9 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | VERIFICATION, IRC_PROTOCOL, RISK | CLI, 통신 프레임, 제한 시간, 속도, 지표, 종료, 로그 순서의 공개 동작을 고정합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `c4df44554866` — `feat(registration): 등록 대기 시간 제한`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | RESILIENCE, LIFECYCLE |
| 원문에서 정한 역할 | 등록 제한 시간을 추가합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 등록 제한 시간을 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | connect 시 `connectedAt`을 저장하고 main event loop가 각 `pollOnce()` 뒤 `onTick()`을 호출하도록 했습니다. maintenance는 fd 스냅샷을 돌며 미등록 클라이언트의 elapsed time을 검사해 451을 queue하고 normal close path를 요청했습니다. |
| 확인한 변경 파일·symbol | `src/ClientRegistry.hpp — connectedAt`<br>`src/IrcApplication.cpp — onConnect, onTick, maintainClient`<br>`src/main.cpp — pollOnce/onTick order`<br>`src/RuntimeConfig.cpp — --registration-timeout` |
| 핵심 상태 / 브랜치 | registration deadline 상태는 ClientState가 보유하고 maintenance는 registry의 fd 스냅샷을 사용해 disconnect 중 반복자 invalidation을 피합니다. |
| 실패 handling | registered 클라이언트는 검사에서 제외되고 timeout 클라이언트만 response+close로 수렴합니다. option은 positive decimal과 1일 상한을 요구합니다. |
| 보장과 다음 연결 | 미등록 연결이 registry와 연결 map을 무기한 점유하지 못합니다. 764361c52b2a가 같은 maintenance loop에 heartbeat를 추가하고 3f2b3ae1d3f9가 monotonic clock으로 교정합니다. |
| 이 시점의 한계 | 시간 기준은 이 시점의 wall clock이며 엄격한 대상-width parsing은 b6c10bc51937 이전에 부족합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c4df44554866` | `src/IrcApplication.cpp` | `maintainClient registration branch` | 미등록 elapsed deadline → 451 → requestClose |
| `c4df44554866` | `src/main.cpp` | `pollOnce then onTick` | 준비 상태 처리와 policy tick 순서 |

- 조사 방법: GitHub에서 `c4df44554866` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `764361c52b2a` — `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`. 원문 역할은 “유휴 하트비트와 PING 응답 제한 상태를 추가합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `764361c52b2a` — `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | RESILIENCE, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 유휴 하트비트와 PING 응답 제한 상태를 추가합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 유휴 하트비트와 PING 응답 제한 상태를 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 등록 deadline 외에는 유휴 연결의 생존 여부를 확인하거나 제거하는 정책이 없었습니다. |
| 주요 문제와 설계 판단 | 모든 received line에서 `lastActivityAt`을 갱신하고, idle 클라이언트에 PING token을 queue한 뒤 `awaitingPong`과 `lastPingAt`을 기록했습니다. deadline을 넘으면 ERROR를 queue하고 close를 요청했으며 PONG handler가 wait를 해제했습니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp — awaitingPong, lastActivityAt, lastPingAt`<br>`src/IrcApplication.cpp — onLine, maintainClient, handlePong`<br>`src/RuntimeConfig.cpp — idle/ping options` |
| 상태 / 소유권 변화 | maintenance 우선순위는 registration timeout → registered 여부 → awaiting PONG timeout → idle probe입니다. |
| 실패 또는 경계 | 초기 구현은 `std::time` wall clock을 사용하고 outstanding token을 저장하지 않아 어떤 PONG도 wait를 해제합니다. |
| 보장 / 비보장 | 보장: idle probe와 timeout이라는 liveness 상태 처리의 최초 정책을 제공합니다.<br>비보장: clock correction과 forged/unrelated PONG에 안전하지 않습니다. |
| 다음 관련 변화 | d710f29f38a4/f313e707474f가 양성 흐름을 먼저 검증하고 3f2b3ae1d3f9/0c76aad19579가 불변식을 수정·고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `764361c52b2a` | `src/IrcApplication.cpp` | `maintainClient` | registration → ping timeout → idle PING 순서 |
| `764361c52b2a` | `src/IrcApplication.cpp` | `handlePong` | 초기 버전은 token 검사 없이 awaiting 상태 해제 |

- 조사 방법: GitHub에서 `764361c52b2a` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `9e2b214f9227` — `feat(throttle): 클라이언트별 명령 호출 횟수 제한`. 원문 역할은 “클라이언트별 명령 속도 제한 구간을 추가합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `9e2b214f9227` — `feat(throttle): 클라이언트별 명령 호출 횟수 제한`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | RESILIENCE |
| 원문에서 정한 역할 | 클라이언트별 명령 속도 제한 구간을 추가합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 클라이언트별 명령 속도 제한 구간을 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | ClientState에 명령 timestamp deque를 두고 parsed 명령마다 오래된 항목을 제거한 후 현재 시각을 추가했습니다. window 내 count가 설정값을 넘으면 439를 queue하고 close를 요청했습니다. |
| 확인한 변경 파일·symbol | `src/ClientRegistry.hpp — commandWindow`<br>`src/IrcApplication.cpp — recordCommand/onLine`<br>`src/RuntimeConfig.cpp — --rate-limit` |
| 핵심 상태 / 브랜치 | rate window는 클라이언트별로 독립이며 parser가 프레임을 명령으로 인정한 뒤에만 기록됩니다. |
| 실패 handling | 초과 클라이언트만 close path로 보내고 server 전체나 다른 클라이언트 상태는 유지합니다. |
| 보장과 다음 연결 | 명령 호출량이 구성된 count/window 아래에서 연결 단위로 governed됩니다. e05e35ca7da9/c34aa18f89af가 limit event를 지표와 로그로 관찰 가능하게 합니다. |
| 이 시점의 한계 | 초기 timestamp는 wall-clock 기반이고 rate-count parsing의 대상-width 오버플로는 b6c10bc51937 전까지 남습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `9e2b214f9227` | `src/IrcApplication.cpp` | `recordCommand` | old timestamp eviction → current push → count compare |
| `9e2b214f9227` | `src/IrcApplication.cpp` | `onLine rate branch` | parsed 명령 초과를 439와 close로 수렴 |

- 조사 방법: GitHub에서 `9e2b214f9227` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`. 원문 역할은 “연결별 미전송 출력 크기를 제한하고 대기열 추가 실패를 전달합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | EVENT_IO, RESILIENCE, RISK |
| 원문에서 정한 역할 | 연결별 미전송 출력 크기를 제한하고 대기열 추가 실패를 전달합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 연결별 미전송 출력 크기를 제한하고 대기열 추가 실패를 전달합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | partial 출력 queue가 무제한이라 느리거나 읽지 않는 peer가 memory를 계속 점유할 수 있었습니다. |
| 주요 문제와 설계 판단 | Connection에 `maxPendingBytes_`를 추가하고 `pendingBytes()` 기준으로 `queueRaw/queueLine` admission을 제한했습니다. queue 함수는 bool을 반환하고 초과 시 close를 요청하며 Server send API가 실패를 caller에 전달했습니다. |
| 변경 파일·symbol | `include/Connection.hpp — maxPendingBytes and bool queue API`<br>`src/Connection.cpp — pendingBytes, canAppendPending, queueRaw, queueLine`<br>`src/Server.cpp — sendTo/queueRawTo propagation` |
| 상태 / 소유권 변화 | limit은 backing buffer size가 아니라 `writeBuffer_.size() - writeOffset_`인 logical unsent bytes를 지배합니다. |
| 실패 또는 경계 | 초과 append는 기존 대기 중인 bytes를 변경하지 않고 `outbound queue limit exceeded` close 상태를 만듭니다. |
| 보장 / 비보장 | 보장: 한 연결의 unsent 출력에 hard bound와 observable 실패 result를 제공합니다.<br>비보장: `pending + byteCount <= limit` 덧셈은 size_t wrap 가능성이 있고 CRLF 추가의 두 단계 경계도 충분히 증명되지 않았습니다. |
| 다음 관련 변화 | 881e59734a9a가 subtraction predicate와 send guard로 수정하고 f34ab135c546가 결정적하게 검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d7d85e518177` | `src/Connection.cpp` | `pendingBytes/canAppendPending` | logical unsent byte 기준 admission |
| `d7d85e518177` | `src/Server.cpp` | `sendTo/queueRawTo` | queue 실패를 애플리케이션까지 bool로 전달 |

- 조사 방법: GitHub에서 `d7d85e518177` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `adb49d9466e4` — `feat(server): 최대 연결 수 제한`. 원문 역할은 “동시 연결 수를 제한합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `adb49d9466e4` — `feat(server): 최대 연결 수 제한`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | RESILIENCE, EVENT_IO |
| 원문에서 정한 역할 | 동시 연결 수를 제한합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 동시 연결 수를 제한합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | accept 직후 현재 연결 count와 configured maximum을 비교해 초과 fd를 event registration/map 소유권 이전에 거부하고 닫았습니다. |
| 확인한 변경 파일·symbol | `include/Server.hpp — maxConnections config/metrics`<br>`src/Server.cpp — acceptReadyClients, rejectReadyClient`<br>`src/RuntimeConfig.cpp — --max-connections` |
| 핵심 상태 / 브랜치 | 거부된 raw fd는 임시 accept scope가 닫고 Server 연결 map이나 애플리케이션 registry에 들어가지 않습니다. |
| 실패 handling | limit 도달은 server-level exception이 아니라 해당 incoming 연결만 drop/reject하는 정상 보호 브랜치입니다. |
| 보장과 다음 연결 | live transport 연결 수가 명시된 최대값을 넘지 않습니다. e05e35ca7da9가 live/accepted/rejected 수를 metrics로 노출하고 규약 테스트가 공개 option을 검증합니다. |
| 이 시점의 한계 | listen backlog와 OS fd limit, 순간 accept rate 자체의 보장은 아닙니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `adb49d9466e4` | `src/Server.cpp` | `acceptReadyClients/rejectReadyClient` | map 편입 전 연결-limit 판정과 raw fd close |

- 조사 방법: GitHub에서 `adb49d9466e4` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `e05e35ca7da9` — `feat(metrics): 서버 실행 지표 조회 기능 추가`. 원문 역할은 “연결·명령·메시지·대기열 폐기·속도 제한 상태를 조회할 수 있게 합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `e05e35ca7da9` — `feat(metrics): 서버 실행 지표 조회 기능 추가`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | OBSERVABILITY |
| 원문에서 정한 역할 | 연결·명령·메시지·대기열 폐기·속도 제한 상태를 조회할 수 있게 합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 연결·명령·메시지·대기열 폐기·속도 제한 상태를 조회할 수 있게 합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | Server transport metrics와 IrcApplication metrics를 각각 authoritative transition에서 누적하고 `METRICS` 명령이 fixed-order `key=value` 값을 반환하도록 했습니다. |
| 확인한 변경 파일·symbol | `include/Server.hpp — Metrics`<br>`src/IrcApplication.hpp — AppMetrics`<br>`src/IrcApplication.cpp/ApplicationSupport.cpp — counters and METRICS` |
| 핵심 상태 / 브랜치 | live 연결 같은 gauge는 server 컨테이너에서 읽고 event count는 실제 transition 직후 증가해 별도 mutable 판단 기준을 만들지 않습니다. |
| 실패 handling | queue rejection/rate limit/idle timeout도 성공 경로와 분리된 counter로 기록됩니다. |
| 보장과 다음 연결 | 운영 보호 결과와 명령/message activity를 query 가능한 stable field set으로 노출합니다. c34aa18f89af가 같은 authoritative transitions를 structured log로 남기고 e5e6c57db80d가 ordering/도형을 고정합니다. |
| 이 시점의 한계 | 지표는 in-프로세스 누적값이며 영속 monitoring/storage나 latency 분포를 제공하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e05e35ca7da9` | `include/Server.hpp` | `Server::Metrics` | transport transition counter 소유자 |
| `e05e35ca7da9` | `src/IrcApplication.cpp` | `METRICS handling` | server/app 수치를 고정 순서 response로 구성 |

- 조사 방법: GitHub에서 `e05e35ca7da9` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `c34aa18f89af` — `feat(log): 연결 상태와 실행 지표 기록`. 원문 역할은 “수명 주기·보호 동작·집계 지표 이벤트를 구조화해 기록합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `c34aa18f89af` — `feat(log): 연결 상태와 실행 지표 기록`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | OBSERVABILITY, LIFECYCLE |
| 원문에서 정한 역할 | 수명 주기·보호 동작·집계 지표 이벤트를 구조화해 기록합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 수명 주기·보호 동작·집계 지표 이벤트를 구조화해 기록합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 공백을 underscore로 정규화한 `key=value` structured log helper를 만들고 connect/register/disconnect/rate-limit/timeout/metrics/shutdown의 확정 시점에 event를 기록했습니다. |
| 확인한 변경 파일·symbol | `src/ApplicationSupport.cpp/IrcApplication.cpp — structured log helpers and call sites` |
| 핵심 상태 / 브랜치 | 로그는 상태를 소유하지 않고 authoritative 변경 직후 그 결과를 직렬화합니다. |
| 실패 handling | 사용자 제공 reason/nick의 공백은 한 field를 깨지 않도록 escape/normalize되고 missing 상태에서는 추측한 identity를 기록하지 않습니다. |
| 보장과 다음 연결 | lifecycle 및 protection event의 machine-readable ordering을 관찰할 수 있습니다. dd04279c47fd의 shutdown ordering과 e5e6c57db80d 규약 테스트가 final log 순서를 검증합니다. |
| 이 시점의 한계 | logger 실패 격리나 durability/rotation은 이 프로젝트 범위에서 보장하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c34aa18f89af` | `src/ApplicationSupport.cpp` | `log event/value sanitization` | 공백 없는 key=value record 생성 |
| `c34aa18f89af` | `src/IrcApplication.cpp` | `lifecycle/protection log call sites` | 확정 변경 뒤 event 기록 |

- 조사 방법: GitHub에서 `c34aa18f89af` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `dd04279c47fd` — `feat(shutdown): 종료 전 송신 대기열 처리`. 원문 역할은 “시그널 즉시 중단 대신 애플리케이션 종료 절차와 출력 비우기를 수행합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `dd04279c47fd` — `feat(shutdown): 종료 전 송신 대기열 처리`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | LIFECYCLE, RESILIENCE, RISK |
| 원문에서 정한 역할 | 시그널 즉시 중단 대신 애플리케이션 종료 절차와 출력 비우기를 수행합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 시그널 즉시 중단 대신 애플리케이션 종료 절차와 출력 비우기를 수행합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | SIGINT/SIGTERM이 server를 즉시 멈춰 queued protocol 출력과 disconnect 정리를 건너뛸 수 있었습니다. |
| 주요 문제와 설계 판단 | signal handler는 async-signal-safe flag만 변경하고 normal loop가 `IrcApplication::shutdown()`으로 각 클라이언트에 final ERROR를 queue·close 요청한 뒤 최대 8회, 각 50ms poll로 대기 중인 출력을 drain하도록 했습니다. 그 후 콜백을 detach하고 server를 stop했습니다. |
| 변경 파일·symbol | `src/main.cpp — signal flag, main loop shutdown/drain order`<br>`src/IrcApplication.cpp — shutdown` |
| 상태 / 소유권 변화 | shutdown transition은 signal context가 아니라 main control flow가 소유하며 연결은 final queue가 비거나 bounded drain budget가 끝날 때까지 Server map에 남습니다. |
| 실패 또는 경계 | 읽지 않는 peer가 있어도 8×50ms 이후 teardown으로 진행해 무한 대기를 막습니다. queue 실패는 해당 연결 정리로 즉시 수렴할 수 있습니다. |
| 보장 / 비보장 | 보장: 종료 전에 final protocol 출력을 시도하면서 프로세스 termination latency를 제한합니다.<br>비보장: 모든 peer가 ERROR를 실제 수신한다는 보장은 없고 drain budget 이후 unsent bytes는 teardown됩니다. |
| 다음 관련 변화 | e5e6c57db80d가 signal→ERROR→close/log ordering을 공개 프로세스 경계에서 검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `dd04279c47fd` | `src/main.cpp` | `signal handler and shutdown sequence` | signal flag → 애플리케이션 shutdown → bounded poll drain → 콜백 detach/stop |
| `dd04279c47fd` | `src/IrcApplication.cpp` | `IrcApplication::shutdown` | 클라이언트 스냅샷에 ERROR queue와 close 요청 |

- 조사 방법: GitHub에서 `dd04279c47fd` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`. 원문 역할은 “CLI, 통신 프레임, 제한 시간, 속도, 지표, 종료, 로그 순서의 공개 동작을 고정합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, IRC_PROTOCOL, RISK |
| 원문에서 정한 역할 | CLI, 통신 프레임, 제한 시간, 속도, 지표, 종료, 로그 순서의 공개 동작을 고정합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> CLI, 통신 프레임, 제한 시간, 속도, 지표, 종료, 로그 순서의 공개 동작을 고정합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | broad smoke는 주요 기능 조합을 보여 주었지만 CLI 실패, exact 프레임/CRLF/numeric order, timeout/rate/metrics/shutdown/log ordering을 정밀하게 고정하지 못했습니다. |
| 주요 문제와 설계 판단 | `tests/irc_contract.py` 중심의 executable 규약 suite를 추가해 프로세스 startup 오류, 런타임 options, exact/regex wire frames, protection responses, metrics field order, graceful shutdown 및 log order를 manifest와 검사문으로 검사했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py — CLI and wire contract checks`<br>`Makefile — contract/test integration` |
| 상태 / 소유권 변화 | 테스트가 server 프로세스, sockets, manifest, logs와 expected frames를 소유하며 동적 fd/token 같은 값만 regex로 허용합니다. |
| 실패 또는 경계 | invalid/missing option과 런타임 timeout/rate/queue/shutdown 경계를 subprocess return 코드, exact line, close 및 log event로 관찰합니다. |
| 보장 / 비보장 | 보장: 공개 CLI·wire·shutdown·observability 규약의 성공/실패 도형과 ordering을 broad substring 수준보다 정확히 고정합니다.<br>비보장: rare syscall return, event add/update 실패, 포인터 수명, formal fairness는 직접 주입하지 않습니다. |
| 다음 관련 변화 | b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level 결정적 실패를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e5e6c57db80d` | `tests/irc_contract.py` | `CLI checks and record_exact/record_regex` | 정적 프레임은 exact, 동적 값만 제한된 regex로 검증 |
| `e5e6c57db80d` | `tests/irc_contract.py` | `wire/shutdown/log scenarios` | timeout·rate·metrics·shutdown 공개 ordering |

- 조사 방법: GitHub에서 `e5e6c57db80d` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 문서화된 CLI syntax, IRC 프레임/CRLF/numeric ordering, protection response, metrics와 shutdown/log ordering이 공개 경계와 일치합니다. |
| 재현 실패 / 경계 | missing/invalid args, registration/heartbeat/rate/출력 limits, exact numerics, signal shutdown과 final ERROR입니다. |
| 테스트 방식 | compiled server subprocess + real loopback TCP + exact/limited-regex 검사문 + captured logs/manifest입니다. |
| 통과하는 실제 실행 경로 | argv parser → Server/IrcApplication 런타임 policy → wire 출력 → signal flag → shutdown/drain → logs입니다. |
| 이 테스트가 증명하는 것 | 외부 사용자가 관찰하는 성공·실패 프레임과 ordering을 정확히 고정합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 syscall/event 실패, memory safety, formal latency·fairness, 내부 수명 안전성은 증명하지 않습니다. |
| 테스트 성격 | exact 프로세스 and wire 규약 통합 |
| 막는 회귀 | CLI가 sign/whitespace를 허용하거나 numeric/CRLF/order/shutdown log 규약이 변하는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `e5e6c57db80d`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 해당 Make 대상
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| registration deadline | `c4df44554866` | onTick 스냅샷 maintenance | 미등록 클라이언트만 deadline 뒤 451+close로 수렴합니다. | e5e6c57db80d 공개 규약 |
| heartbeat liveness | `764361c52b2a` | initial idle PING/PONG | 초기에는 wall-clock/any-PONG이라는 부족함이 있습니다. | 3f2b3ae1d3f9/0c76aad19579가 수정·검증 |
| 명령-rate bound | `9e2b214f9227` | per-클라이언트 timestamp window | 초과 actor만 439+close 처리합니다. | e05e35ca7da9/c34aa18f89af 관찰, e5e6c57db80d 규약 |
| 출력/연결 bounds | `d7d85e518177` | adb49d9466e4 | logical 대기 중인 bytes와 live 연결 수를 제한합니다. | 881e59734a9a/f34ab135c546 arithmetic fix/테스트 |
| observable controlled shutdown | `e05e35ca7da9` | c34aa18f89af, dd04279c47fd | metrics/log와 bounded final-출력 drain을 연결합니다. | e5e6c57db80d exact 프로세스 규약 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| functional server는 무제한 자원을 감당 | 미등록·slow reader·연결 flood가 상태/memory/fd 점유 | c4df44554866 / d7d85e518177 / adb49d9466e4 | 각 자원을 연결-scoped bound로 제한 | e5e6c57db80d |
| wall-clock 및 any PONG이면 충분 | clock 이동·forged PONG으로 liveness 오판 | 3f2b3ae1d3f9 | steady_clock + exact token | 0c76aad19579 |
| signal에서 즉시 stop | queued final 출력과 정리 유실 | dd04279c47fd | flag-only handler와 bounded normal-flow drain | e5e6c57db80d |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| registration/rate/heartbeat 상태 | 없음 | ClientState + IrcApplication maintenance | c4df44554866 → 9e2b214f9227 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| unsent-출력 bound | 무제한 Connection queue | Connection maxPendingBytes | d7d85e518177 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| live-연결 bound | listener가 모두 수락 | Server accept policy | adb49d9466e4 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| transport/app metrics | 관찰 불가 | Server::Metrics / AppMetrics | e05e35ca7da9 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| shutdown orchestration | signal-time immediate stop | main normal control flow + IrcApplication::shutdown | dd04279c47fd | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: transport를 받아 놓고 등록을 끝내지 않는 클라이언트를 제거할 deadline이 없었습니다.
- 마지막 커밋 `e5e6c57db80d` 시점의 상태: 공개 CLI·wire·shutdown·observability 규약의 성공/실패 도형과 ordering을 broad substring 수준보다 정확히 고정합니다.
- 개발 흐름 안에서 강화된 핵심 불변식: registration deadline, heartbeat liveness, 명령-rate bound, 출력/연결 bounds, observable controlled shutdown.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: rare syscall return, event add/update 실패, 포인터 수명, formal fairness는 직접 주입하지 않습니다. b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level 결정적 실패를 추가합니다.
- 최종 설명: `c4df44554866`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `e5e6c57db80d`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| 연결/명령 event | c4df44554866 / 9e2b214f9227 | `Server callbacks/IrcApplication` | ClientState timestamps/window 갱신 | deadline/rate 초과는 response+requestClose |
| 출력 admission | d7d85e518177 | `IrcApplication send helper` | Server.sendTo → Connection queue | limit 초과는 false+연결 close |
| accept admission | adb49d9466e4 | `Server::acceptReadyClients` | count 확인 후 map/backend 편입 | limit 도달 raw fd는 편입 전 close |
| observability | e05e35ca7da9 / c34aa18f89af | `authoritative transition` | counter/gauge와 key=value log | 실패 브랜치도 별도 metric/event |
| signal shutdown | dd04279c47fd | `signal flag/main loop` | ERROR queue → close 요청 → 8×50ms drain → detach/stop | budget 만료 후 unsent bytes는 teardown |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 04-operational-protections-and-controlled-shutdown.md =====

===== BEGIN FILE: 05-strict-runtime-configuration-boundaries.md =====
# 개발 흐름 05 — 엄격한 실행 설정 검증

부제: 엄격한 런타임 구성 경계

## 1. 개발 흐름의 목표

CLI 문자열을 실제 destination type과 operational bound에 정확히 맞는 값으로 바꾸는 경계를 복원하고, permissive C conversion과 narrowing의 결함이 digit-by-digit parser 및 cross-width 회귀로 어떻게 수정되는지 학습합니다.

### 원문에서 정한 의의

초기 설정 파싱은 문서화된 문법과 최종 정수 형식의 범위를 정확히 일치시키지 못했습니다. 숫자를 한 자리씩 누적하며 대상 형식의 상한과 직접 비교하도록 수정하고, 호스트 정수 폭이 달라도 같은 경계를 검증하도록 테스트를 구성합니다.

<details>
<summary>영문 원문</summary>

> The public configuration surface initially relied on conversion functions whose accepted syntax and intermediate width did not exactly match the documented contract. The fix moves range checking into digit-by-digit accumulation against the destination type, and the follow-up tests preserve that cross-platform boundary without treating the test itself as a second architectural decision.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- 초기 `RuntimeConfig`와 `Server::Config`의 책임은 어떻게 구분되는가?
- `strtol`/`strtoul`의 accepted syntax와 intermediate width가 공개 규약과 어디서 어긋나는가?
- 곱셈·덧셈 전에 오버플로를 증명하는 비교식은 무엇인가?
- port, timeout, `std::size_t` option이 각 destination maximum을 직접 사용하는가?
- 32/64-bit 모두에서 first-unrepresentable size를 테스트가 어떻게 계산하는가?

### 원문에서 확인되는 불변식

- 허용한 숫자 옵션은 실제 소켓·버퍼·연결·시간 제한에 사용하는 대상 정수 형식으로 정확히 표현할 수 있어야 합니다.
- 공개 CLI syntax는 non-empty ASCII decimal이라는 확정 계약과 일치해야 하며 sign/whitespace를 암묵적으로 허용해서는 안 됩니다.

### 원문에서 확인되는 구현 난점

- 외부 입력을 중간 C 정수 변환에 맡기지 않고 최종 대상 정수 폭에서 오버플로 없이 검증하는 문제.
- 호스트 정수 폭이 달라도 같은 표현 가능 범위를 검증할 수 있는 테스트 경계를 설계하는 문제.

## 3. 완료 기준

- 초기 parser가 허용/거부하는 입력을 해당 SHA 코드로 정리했습니다.
- old conversion+narrowing과 new bounded accumulation을 숫자 예제로 비교했습니다.
- sign, whitespace, exact maximum, maximum+1, extremely long decimal의 control flow를 설명합니다.
- CLI 규약 테스트와 경계 회귀의 증명 범위를 구분했습니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `52b6f1ce8f0f` | `feat(config): 기본 실행 인자 해석 모듈 구성` | B | BUILD, RESILIENCE | 초기 `RuntimeConfig` 파싱 경계를 정합니다. |
| 2 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | VERIFICATION, IRC_PROTOCOL, RISK | CLI의 정확한 성공·실패 동작을 실행 가능한 테스트로 고정합니다. |
| 3 | `b6c10bc51937` | `fix(config): 서버 크기 옵션을 오버플로 없이 해석` | A | DEBUG, RESILIENCE, RISK | 관대한 C 변환 대신 대상 정수 폭과 오버플로를 검사하는 10진수 파싱을 사용합니다. |
| 4 | `5d1286620994` | `test(config): 크기 옵션 경계와 오류 입력 검증` | B | VERIFICATION, RESILIENCE, RISK | 부호·공백·포트 범위 축소·제한 시간 상한·`size_t` 오버플로·속도 제한 횟수 오버플로를 검증합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `52b6f1ce8f0f` — `feat(config): 기본 실행 인자 해석 모듈 구성`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | BUILD, RESILIENCE |
| 원문에서 정한 역할 | 초기 `RuntimeConfig` 파싱 경계를 정합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 초기 `RuntimeConfig` 파싱 경계를 정합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | `RuntimeConfig`에 rate, idle/ping, registration timeout 기본값을 두고 `<port> <password>`를 필수 인자로 해석했습니다. 초기 port parser는 `strtol`, end 포인터, 1..65535 range를 확인하고 unknown extra argument를 usage error로 처리했습니다. |
| 확인한 변경 파일·symbol | `src/RuntimeConfig.hpp — RuntimeConfig fields/defaults`<br>`src/RuntimeConfig.cpp — parseRuntimeConfig, initial port parser`<br>`src/main.cpp — Server::Config mapping` |
| 핵심 상태 / 브랜치 | 애플리케이션 policy는 RuntimeConfig가, bind address/port와 transport limit은 Server::Config가 소유하도록 책임을 분리합니다. |
| 실패 handling | empty/trailing garbage, zero/negative/out-of-port-range와 잘못된 argument count는 server 시작 전에 diagnostic과 실패 return으로 끝납니다. |
| 보장과 다음 연결 | 공개 argv를 typed 런타임 상태로 바꾸는 단일 parsing 경계와 최소 CLI 규약이 생깁니다. e5e6c57db80d가 exact CLI 동작을 기록하고 b6c10bc51937가 destination-width decimal parser로 교체합니다. |
| 이 시점의 한계 | `strtol`은 leading whitespace와 sign을 허용하는 C lexical semantics를 가지며 optional flags는 이 scaffold 단계에서 아직 실제 config를 모두 변경하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `52b6f1ce8f0f` | `src/RuntimeConfig.cpp` | `port parsing and parseRuntimeConfig` | complete consumption/range 검사와 required argv 도형 |
| `52b6f1ce8f0f` | `src/RuntimeConfig.hpp` | `RuntimeConfig` | 애플리케이션 policy와 transport config의 책임 분리 |

- 조사 방법: GitHub에서 `52b6f1ce8f0f` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`. 원문 역할은 “CLI의 정확한 성공·실패 동작을 실행 가능한 테스트로 고정합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, IRC_PROTOCOL, RISK |
| 원문에서 정한 역할 | CLI의 정확한 성공·실패 동작을 실행 가능한 테스트로 고정합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> CLI의 정확한 성공·실패 동작을 실행 가능한 테스트로 고정합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | broad smoke는 주요 기능 조합을 보여 주었지만 CLI 실패, exact 프레임/CRLF/numeric order, timeout/rate/metrics/shutdown/log ordering을 정밀하게 고정하지 못했습니다. |
| 주요 문제와 설계 판단 | `tests/irc_contract.py` 중심의 executable 규약 suite를 추가해 프로세스 startup 오류, 런타임 options, exact/regex wire frames, protection responses, metrics field order, graceful shutdown 및 log order를 manifest와 검사문으로 검사했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py — CLI and wire contract checks`<br>`Makefile — contract/test integration` |
| 상태 / 소유권 변화 | 테스트가 server 프로세스, sockets, manifest, logs와 expected frames를 소유하며 동적 fd/token 같은 값만 regex로 허용합니다. |
| 실패 또는 경계 | invalid/missing option과 런타임 timeout/rate/queue/shutdown 경계를 subprocess return 코드, exact line, close 및 log event로 관찰합니다. |
| 보장 / 비보장 | 보장: 공개 CLI·wire·shutdown·observability 규약의 성공/실패 도형과 ordering을 broad substring 수준보다 정확히 고정합니다.<br>비보장: rare syscall return, event add/update 실패, 포인터 수명, formal fairness는 직접 주입하지 않습니다. |
| 다음 관련 변화 | b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level 결정적 실패를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e5e6c57db80d` | `tests/irc_contract.py` | `CLI checks and record_exact/record_regex` | 정적 프레임은 exact, 동적 값만 제한된 regex로 검증 |
| `e5e6c57db80d` | `tests/irc_contract.py` | `wire/shutdown/log scenarios` | timeout·rate·metrics·shutdown 공개 ordering |

- 조사 방법: GitHub에서 `e5e6c57db80d` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 문서화된 CLI syntax, IRC 프레임/CRLF/numeric ordering, protection response, metrics와 shutdown/log ordering이 공개 경계와 일치합니다. |
| 재현 실패 / 경계 | missing/invalid args, registration/heartbeat/rate/출력 limits, exact numerics, signal shutdown과 final ERROR입니다. |
| 테스트 방식 | compiled server subprocess + real loopback TCP + exact/limited-regex 검사문 + captured logs/manifest입니다. |
| 통과하는 실제 실행 경로 | argv parser → Server/IrcApplication 런타임 policy → wire 출력 → signal flag → shutdown/drain → logs입니다. |
| 이 테스트가 증명하는 것 | 외부 사용자가 관찰하는 성공·실패 프레임과 ordering을 정확히 고정합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 syscall/event 실패, memory safety, formal latency·fairness, 내부 수명 안전성은 증명하지 않습니다. |
| 테스트 성격 | exact 프로세스 and wire 규약 통합 |
| 막는 회귀 | CLI가 sign/whitespace를 허용하거나 numeric/CRLF/order/shutdown log 규약이 변하는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `e5e6c57db80d`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 해당 Make 대상
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `b6c10bc51937` — `fix(config): 서버 크기 옵션을 오버플로 없이 해석`. 원문 역할은 “관대한 C 변환 대신 대상 정수 폭과 오버플로를 검사하는 10진수 파싱을 사용합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `b6c10bc51937` — `fix(config): 서버 크기 옵션을 오버플로 없이 해석`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | DEBUG, RESILIENCE, RISK |
| 원문에서 정한 역할 | 관대한 C 변환 대신 대상 정수 폭과 오버플로를 검사하는 10진수 파싱을 사용합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 관대한 C 변환 대신 대상 정수 폭과 오버플로를 검사하는 10진수 파싱을 사용합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | numeric options가 `strtol/strtoul`의 permissive syntax와 host intermediate width를 거친 뒤 port·timeout·size_t로 narrowing되어 destination 규약보다 넓은 입력을 받아들일 수 있었습니다. |
| 주요 문제와 설계 판단 | ASCII digit만 한 자리씩 누적하는 `parseUnsignedDecimal<Unsigned>()`를 도입하고 각 digit 전에 `value > max/10 \|\| (value == max/10 && digit > max%10)`을 검사했습니다. 각 option은 실제 destination type의 maximum을 template argument/limit로 사용했습니다. |
| 변경 파일·symbol | `src/RuntimeConfig.cpp — parseUnsignedDecimal and option parsers` |
| 상태 / 소유권 변화 | 외부 문자열은 intermediate signed/unsigned long이 아니라 최종 대상-width에서 직접 representability를 검증받습니다. |
| 실패 또는 경계 | empty, `+/-`, whitespace, non-digit, exact maximum 초과, 매우 긴 decimal은 곱셈·덧셈 전에 거부되어 wrap이나 narrowing이 일어나지 않습니다. |
| 보장 / 비보장 | 보장: accepted option은 non-empty ASCII decimal이고 실제 port/timeout/size_t/rate-count destination에 표현 가능합니다.<br>비보장: semantic relation인 idle/ping 조합 등은 별도 policy 검증이며 이 helper는 decimal representability만 보장합니다. |
| 다음 관련 변화 | 5d1286620994가 플랫폼 width를 계산한 first-unrepresentable values와 lexical 오류를 회귀로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `b6c10bc51937` | `src/RuntimeConfig.cpp` | `parseUnsignedDecimal<Unsigned>` | `maximum / 10`과 `maximum % 10` pre-check로 오버플로 전 거부 |
| `b6c10bc51937` | `src/RuntimeConfig.cpp` | `port/size/rate option parsing` | 각 destination maximum을 직접 전달 |

- 조사 방법: GitHub에서 `b6c10bc51937` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | C conversion이 complete consumption과 errno를 확인하면 공개 decimal 규약 및 모든 destination width에도 충분하다는 가정이었습니다. |
| 실제 실패 / risk | leading sign/whitespace가 허용되고 wider intermediate를 좁은 port·count·size_t로 cast하면 플랫폼에 따라 wrap/narrowing할 수 있었습니다. |
| root cause | lexical 규약과 range check가 최종 destination type이 아니라 C library intermediate semantics에 묶여 있었습니다. |
| 수정된 불변식 / 결정 | ASCII digit accumulation을 최종 Unsigned maximum에 대해 매 step pre-check하고 성공한 값만 config에 저장합니다. |
| 회귀 근거 | 5d1286620994가 sign/whitespace, 65536, huge timeout, size_t max+1, rate-count 오버플로를 실행 경계로 고정합니다. |

#### 다음 연결

- 다음 커밋: `5d1286620994` — `test(config): 크기 옵션 경계와 오류 입력 검증`. 원문 역할은 “부호·공백·포트 범위 축소·제한 시간 상한·`size_t` 오버플로·속도 제한 횟수 오버플로를 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `5d1286620994` — `test(config): 크기 옵션 경계와 오류 입력 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | VERIFICATION, RESILIENCE, RISK |
| 원문에서 정한 역할 | 부호·공백·포트 범위 축소·제한 시간 상한·`size_t` 오버플로·속도 제한 횟수 오버플로를 검증합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 부호·공백·포트 범위 축소·제한 시간 상한·`size_t` 오버플로·속도 제한 횟수 오버플로를 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 규약 테스트에 sign/whitespace, port 65536, timeout upper bound 초과, 매우 긴 decimal, 런타임에서 계산한 `std::size_t` maximum의 다음 값, rate-count destination 오버플로 cases를 추가했습니다. |
| 확인한 변경 파일·symbol | `tests/irc_contract.py — numeric boundary and invalid CLI cases` |
| 핵심 상태 / 브랜치 | 테스트는 Python integer로 host/프로세스가 노출한 size width에 맞는 first-unrepresentable decimal을 구성하고 각 subprocess 실패를 manifest에 기록합니다. |
| 실패 handling | 각 invalid 입력은 server가 listen하기 전에 nonzero exit와 정해진 diagnostic으로 거부되어야 합니다. |
| 보장과 다음 연결 | 32/64-bit 차이와 무관하게 destination representability 및 strict ASCII-decimal syntax가 유지됩니다. 416efc91e580의 Linux/macOS matrix가 서로 다른 platform 환경에서 같은 규약 suite를 반복합니다. |
| 이 시점의 한계 | parser 내부의 모든 template instantiation을 unit level에서 직접 호출하는 테스트는 아니며 프로세스 CLI 경계만 통과합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5d1286620994` | `tests/irc_contract.py` | `configuration boundary cases` | sign/whitespace/port/timeout/rate/size 오버플로를 프로세스 실패로 검증 |

- 조사 방법: GitHub에서 `5d1286620994` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | accepted numeric CLI value가 strict ASCII decimal이고 실제 destination type에 표현 가능해야 합니다. |
| 재현 실패 / 경계 | `+1`, `-1`, leading/trailing whitespace, port 65536, timeout 상한 초과, size_t max+1, rate-count 오버플로입니다. |
| 테스트 방식 | real executable CLI subprocess와 Python arbitrary-precision integer로 platform first-unrepresentable 값을 생성하는 경계 회귀입니다. |
| 통과하는 실제 실행 경로 | argv → `parseUnsignedDecimal` → option-specific bound → RuntimeConfig/Server::Config 또는 startup rejection입니다. |
| 이 테스트가 증명하는 것 | lexical strictness와 대상-width 오버플로 rejection이 공개 CLI에서 유지됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 internal caller나 non-CLI serialization 경로, semantic option 조합 전체는 증명하지 않습니다. |
| 테스트 성격 | cross-width 경계 회귀 |
| 막는 회귀 | C conversion으로 회귀하거나 narrowing이 32/64-bit 중 한쪽에서만 통과하는 문제를 막습니다. |

실행 기록:

- 실행 SHA: `5d1286620994`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| 런타임 parsing 경계 | `52b6f1ce8f0f` | e5e6c57db80d | RuntimeConfig와 Server::Config 책임 및 공개 CLI 동작을 고정합니다. | 초기 C conversion의 lexical/width 한계가 남음 |
| 대상-width representability | `b6c10bc51937` | digit-by-digit accumulation | destination maximum에 대해 변경 전 오버플로를 거부합니다. | 5d1286620994 cross-width 회귀 |
| strict decimal syntax | `b6c10bc51937` | all numeric option parsers | sign/whitespace/non-digit를 허용하지 않습니다. | 5d1286620994 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| strtol/strtoul + cast면 충분 | sign/whitespace 허용과 intermediate-width narrowing | b6c10bc51937 | ASCII digit + destination max pre-check | 5d1286620994 |
| 한 platform 경계만 고정 | 32/64-bit 중 한쪽에서만 오버플로 재현 | 5d1286620994 | 런타임 first-unrepresentable decimal 계산 | 416efc91e580 OS matrix |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| 애플리케이션 런타임 policy | 없음 | RuntimeConfig | 52b6f1ce8f0f | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| transport bind/size policy | 문자열 argv | Server::Config typed fields | 52b6f1ce8f0f → b6c10bc51937 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| numeric representability check | C library conversion | parseUnsignedDecimal<Unsigned> | b6c10bc51937 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: 실행 인자에서 transport와 애플리케이션 런타임 policy를 분리해 보관·검증하는 모듈이 없었습니다.
- 마지막 커밋 `5d1286620994` 시점의 상태: 32/64-bit 차이와 무관하게 destination representability 및 strict ASCII-decimal syntax가 유지됩니다.
- 개발 흐름 안에서 강화된 핵심 불변식: 런타임 parsing 경계, 대상-width representability, strict decimal syntax.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: parser 내부의 모든 template instantiation을 unit level에서 직접 호출하는 테스트는 아니며 프로세스 CLI 경계만 통과합니다. 416efc91e580의 Linux/macOS matrix가 서로 다른 platform 환경에서 같은 규약 suite를 반복합니다.
- 최종 설명: `52b6f1ce8f0f`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `5d1286620994`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| argv 도형 | 52b6f1ce8f0f | `main/parser` | required port/password와 option 분리 | missing/unknown은 usage 실패 |
| decimal scan | b6c10bc51937 | `option parser` | ASCII digit를 대상 Unsigned에 누적 | digit 전 max/10·max%10 검사에서 reject |
| typed 대입 | b6c10bc51937 | `parseRuntimeConfig` | RuntimeConfig/Server::Config에 성공 값 저장 | 실패 값은 config를 변경하지 않음 |
| 회귀 | 5d1286620994 | `real CLI subprocess` | exact max/first-unrepresentable 및 lexical cases | startup 전에 diagnostic/nonzero exit |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 05-strict-runtime-configuration-boundaries.md =====

===== BEGIN FILE: 06-heartbeat-liveness-correctness.md =====
# 개발 흐름 06 — 하트비트 기반 연결 생존 판정

부제: 하트비트 생존성 정확성

## 1. 개발 흐름의 목표

초기 idle PING/PONG 상태 처리의 실제 보장과 부족함을 분리하고, monotonic clock과 exact outstanding token correlation로 liveness 불변식을 복구한 뒤 forged-response 회귀로 고정합니다.

### 원문에서 정한 의의

초기 하트비트는 시스템 시각을 사용하고 어떤 PONG도 현재 도전에 대한 응답으로 인정했습니다. 제한 시간은 단조 증가 시계로 계산하고, 서버가 보낸 현재 토큰과 일치하는 PONG만 연결 생존 근거로 인정하도록 수정합니다.

<details>
<summary>영문 원문</summary>

> The initial feature supplied liveness policy but relied on wall-clock time and accepted any PONG as evidence for the outstanding heartbeat. The correction restores the actual invariant: deadline calculation must be monotonic, and only the response corresponding to the server's current challenge can preserve the connection. The regression test makes the forged-response failure deterministic.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- idle, awaitingPong, pingSentAt, token 상태는 어떤 전이로 움직이는가?
- registration timeout, heartbeat deadline, idle probe의 maintenance 우선순위는 무엇인가?
- ordinary activity와 outstanding challenge completion은 왜 다른 상태인가?
- wall-clock 대신 `steady_clock`이 필요한 실패 상황은 무엇인가?
- matching PONG과 unrelated PONG을 테스트가 어떻게 결정적하게 구분하는가?

### 원문에서 확인되는 불변식

- 하트비트 제한 시간은 단조 증가 시계의 경과 시간으로 계산해야 합니다.
- idle 클라이언트는 outstanding probe가 없거나 하나의 식별 가능한 probe와 send time을 가져야 합니다.
- 현재 outstanding token과 정확히 일치하는 PONG만 wait 상태를 해제해야 합니다.

### 원문에서 확인되는 구현 난점

- 시스템 시각 보정이나 일반 프로토콜 트래픽을 연결 생존 근거로 잘못 취급하지 않는 상태 처리를 설계하는 문제.
- 실제 시간·소켓 통합 테스트와 위조 토큰을 정확히 검사하는 회귀 테스트의 역할을 분리하는 문제.

## 3. 완료 기준

- 764361c52b2a의 상태 처리와 any-PONG/wall-clock 한계를 코드로 기록했습니다.
- 테스트 peer 자동응답이 broad smoke 안정화와 dedicated 실패 상황을 모두 지원하는 구조를 설명합니다.
- 3f2b3ae1d3f9의 clock type, token generation/storage/validation/clear 순서를 복원했습니다.
- 0c76aad19579가 matching response와 forged response를 각각 검증하는 이유를 설명합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `764361c52b2a` | `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` | A | RESILIENCE, LIFECYCLE, RISK | 하트비트 PING, PONG 대기, 제한 시간 상태를 도입합니다. |
| 2 | `d710f29f38a4` | `test(client): 서버 PING에 응답하는 검사 클라이언트 구현` | B | VERIFICATION, IRC_PROTOCOL | 테스트 클라이언트에 자동 PONG 응답을 추가합니다. |
| 3 | `f313e707474f` | `test(client): 유휴 연결의 PING·PONG 흐름 검증` | B | VERIFICATION, RESILIENCE | 정상 PING/PONG 흐름을 검증합니다. |
| 4 | `3f2b3ae1d3f9` | `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리` | A | DEBUG, RESILIENCE, RISK | 시간을 `steady_clock`으로 측정하고 PONG을 정확한 미응답 토큰과 연결합니다. |
| 5 | `0c76aad19579` | `test(heartbeat): PONG 토큰과 시간 경계 검증` | A | VERIFICATION, RESILIENCE, RISK | 일치하는 PONG만 제한 시간을 해제하며 위조 PONG은 해제하지 못함을 검증합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `764361c52b2a` — `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | RESILIENCE, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 하트비트 PING, PONG 대기, 제한 시간 상태를 도입합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 하트비트 PING, PONG 대기, 제한 시간 상태를 도입합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 등록 deadline 외에는 유휴 연결의 생존 여부를 확인하거나 제거하는 정책이 없었습니다. |
| 주요 문제와 설계 판단 | 모든 received line에서 `lastActivityAt`을 갱신하고, idle 클라이언트에 PING token을 queue한 뒤 `awaitingPong`과 `lastPingAt`을 기록했습니다. deadline을 넘으면 ERROR를 queue하고 close를 요청했으며 PONG handler가 wait를 해제했습니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp — awaitingPong, lastActivityAt, lastPingAt`<br>`src/IrcApplication.cpp — onLine, maintainClient, handlePong`<br>`src/RuntimeConfig.cpp — idle/ping options` |
| 상태 / 소유권 변화 | maintenance 우선순위는 registration timeout → registered 여부 → awaiting PONG timeout → idle probe입니다. |
| 실패 또는 경계 | 초기 구현은 `std::time` wall clock을 사용하고 outstanding token을 저장하지 않아 어떤 PONG도 wait를 해제합니다. |
| 보장 / 비보장 | 보장: idle probe와 timeout이라는 liveness 상태 처리의 최초 정책을 제공합니다.<br>비보장: clock correction과 forged/unrelated PONG에 안전하지 않습니다. |
| 다음 관련 변화 | d710f29f38a4/f313e707474f가 양성 흐름을 먼저 검증하고 3f2b3ae1d3f9/0c76aad19579가 불변식을 수정·고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `764361c52b2a` | `src/IrcApplication.cpp` | `maintainClient` | registration → ping timeout → idle PING 순서 |
| `764361c52b2a` | `src/IrcApplication.cpp` | `handlePong` | 초기 버전은 token 검사 없이 awaiting 상태 해제 |

- 조사 방법: GitHub에서 `764361c52b2a` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `d710f29f38a4` — `test(client): 서버 PING에 응답하는 검사 클라이언트 구현`. 원문 역할은 “테스트 클라이언트에 자동 PONG 응답을 추가합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `d710f29f38a4` — `test(client): 서버 PING에 응답하는 검사 클라이언트 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | VERIFICATION, IRC_PROTOCOL |
| 원문에서 정한 역할 | 테스트 클라이언트에 자동 PONG 응답을 추가합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 테스트 클라이언트에 자동 PONG 응답을 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 테스트 peer에 `auto_pong` option을 추가하고 server `PING`의 parameter를 그대로 echo한 `PONG`을 자동 전송하도록 했습니다. |
| 확인한 변경 파일·symbol | `tools/irc_smoke_client.py — IrcPeer auto_pong and receive loop` |
| 핵심 상태 / 브랜치 | 테스트 peer가 수신 buffer를 parse하는 시점에 PING을 감지해 같은 소켓으로 PONG을 보내며 scenario는 필요할 때 자동응답을 끌 수 있습니다. |
| 실패 handling | PING에 parameter가 없거나 malformed인 경우 임의 token을 만들지 않고 기존 검사문/timeout 경로가 실패를 드러냅니다. |
| 보장과 다음 연결 | broad 테스트 시나리오가 heartbeat policy 때문에 우연히 종료되지 않고 production PING→PONG 경로를 통과합니다. f313e707474f가 짧은 timeout으로 양성 흐름을 전용 시나리오로 확인하고 0c76aad19579는 auto_pong을 꺼 forged case를 만듭니다. |
| 이 시점의 한계 | 자동응답은 matching PONG 양성 경로만 만들며 forged token이나 deadline correctness를 검증하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d710f29f38a4` | `tools/irc_smoke_client.py` | `IrcPeer auto_pong receive handling` | server PING token을 같은 parameter의 PONG으로 자동 응답 |

- 조사 방법: GitHub에서 `d710f29f38a4` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 검사 클라이언트가 server heartbeat challenge에 protocol-compatible response를 보낼 수 있어야 합니다. |
| 재현 실패 / 경계 | server PING 수신과 token echo입니다. |
| 테스트 방식 | real TCP peer helper의 automatic response seam입니다. |
| 통과하는 실제 실행 경로 | 소켓 receive buffer → PING line parse → PONG sendall → server PONG handler입니다. |
| 이 테스트가 증명하는 것 | 테스트 harness가 양성 heartbeat path를 자동으로 유지할 수 있음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | server가 exact token만 인정하거나 deadline을 올바르게 계산하는지는 증명하지 않습니다. |
| 테스트 성격 | 테스트-harness capability |
| 막는 회귀 | heartbeat 도입 후 unrelated smoke가 idle timeout으로 불안정해지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `d710f29f38a4`
- repository에 정의된 실행 명령: 이 helper를 사용하는 smoke/규약 scenario
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `f313e707474f` — `test(client): 유휴 연결의 PING·PONG 흐름 검증`. 원문 역할은 “정상 PING/PONG 흐름을 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `f313e707474f` — `test(client): 유휴 연결의 PING·PONG 흐름 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | B |
| 태그 | VERIFICATION, RESILIENCE |
| 원문에서 정한 역할 | 정상 PING/PONG 흐름을 검증합니다. |
| 학습 깊이 | 개발 흐름에서 맡는 구체적 구현 역할과 핵심 상태/브랜치만 기록합니다. |

#### 원문에서 정한 역할

> 정상 PING/PONG 흐름을 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 테스트 scenario가 짧은 heartbeat 옵션으로 server를 실행하고 idle peer가 PING을 받은 뒤 자동 PONG하며 이후 명령/PING response가 계속 진행되는지 확인했습니다. |
| 확인한 변경 파일·symbol | `tools/irc_smoke_client.py or tests scenario — idle heartbeat positive flow` |
| 핵심 상태 / 브랜치 | real monotonic 테스트 deadline과 소켓 timeout이 scenario 수명을 제한하고 peer 연결은 PONG 뒤에도 유지됩니다. |
| 실패 handling | expected server PING 또는 후속 response가 deadline 안에 없거나 연결이 닫히면 테스트가 실패합니다. |
| 보장과 다음 연결 | 초기 heartbeat의 positive 통합 path가 listener/event loop/parser/애플리케이션/출력 queue를 거쳐 동작합니다. 3f2b3ae1d3f9가 실제 liveness 불변식을 고치고 0c76aad19579가 matching/forged response를 분리합니다. |
| 이 시점의 한계 | 초기 구현의 wall-clock 사용과 any-PONG acceptance는 이 양성 테스트로 드러나지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f313e707474f` | `tools/irc_smoke_client.py` | `idle heartbeat scenario` | 짧은 idle timeout에서 PING→automatic PONG→연결 progress |

- 조사 방법: GitHub에서 `f313e707474f` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | idle probe에 정상 PONG을 보낸 연결은 ping deadline으로 제거되지 않아야 합니다. |
| 재현 실패 / 경계 | 짧은 idle interval과 positive PING/PONG round trip입니다. |
| 테스트 방식 | real 프로세스/소켓 timing 통합 with automatic PONG입니다. |
| 통과하는 실제 실행 경로 | onTick idle 브랜치 → send PING → 테스트 peer auto-PONG → handlePong → later progress입니다. |
| 이 테스트가 증명하는 것 | 정상 heartbeat round trip과 연결 유지가 통합 환경에서 동작합니다. |
| 이 테스트가 증명하지 않는 것 | forged token rejection, wall-clock 되돌리기, exact deadline ordering은 증명하지 않습니다. |
| 테스트 성격 | positive liveness 통합 |
| 막는 회귀 | heartbeat feature가 정상 응답 클라이언트도 timeout시키는 기본 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `f313e707474f`
- repository에 정의된 실행 명령: 해당 smoke/규약 heartbeat scenario
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `3f2b3ae1d3f9` — `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리`. 원문 역할은 “시간을 `steady_clock`으로 측정하고 PONG을 정확한 미응답 토큰과 연결합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `3f2b3ae1d3f9` — `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | DEBUG, RESILIENCE, RISK |
| 원문에서 정한 역할 | 시간을 `steady_clock`으로 측정하고 PONG을 정확한 미응답 토큰과 연결합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 시간을 `steady_clock`으로 측정하고 PONG을 정확한 미응답 토큰과 연결합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | heartbeat와 rate/registration timing이 wall clock을 사용하고, `awaitingPong`만 있어 unrelated PONG도 outstanding challenge를 완료할 수 있었습니다. |
| 주요 문제와 설계 판단 | 클라이언트 timing을 `std::chrono::steady_clock` 기반 `MonotonicTime`으로 바꾸고, 증가하는 heartbeat 순서로 token을 만들었습니다. PING 전 `awaitingPong`, `pendingPongToken`, `lastPingAt`을 저장하고 PONG parameter가 exact token과 일치할 때만 wait 상태/token을 지웠습니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp — MonotonicTime, pendingPongToken`<br>`src/IrcApplication.hpp/.cpp — heartbeat sequence, maintainClient, handlePong` |
| 상태 / 소유권 변화 | 한 클라이언트는 outstanding probe가 없거나 정확히 하나의 token과 send time을 가지며 ordinary activity는 그 challenge를 완료하지 않습니다. |
| 실패 또는 경계 | system clock correction은 elapsed deadline에 영향을 주지 않고, parameter 누락·추가·불일치 PONG은 liveness proof가 아니므로 상태를 그대로 둡니다. |
| 보장 / 비보장 | 보장: deadline이 monotonic elapsed time을 사용하고 현재 challenge와 정확히 상관된 response만 연결을 보존합니다.<br>비보장: 네트워크 delay/clock scheduling 자체의 upper bound와 cryptographic token unpredictability는 보장하지 않습니다. |
| 다음 관련 변화 | 0c76aad19579가 matching response와 forged response를 결정적한 real-소켓 시나리오로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `3f2b3ae1d3f9` | `src/IrcApplication.cpp` | `maintainClient heartbeat branch` | token 생성과 awaiting/token/time 변경을 send 전에 설정 |
| `3f2b3ae1d3f9` | `src/IrcApplication.cpp` | `handlePong` | 정확히 하나의 parameter가 대기 중인 token과 같을 때만 clear |
| `3f2b3ae1d3f9` | `src/ClientRegistry.hpp` | `steady_clock-based fields` | wall-clock과 분리된 elapsed deadline |

- 조사 방법: GitHub에서 `3f2b3ae1d3f9` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | 최근 activity나 아무 PONG이면 현재 peer가 server heartbeat에 응답했다고 간주해도 된다는 가정이었습니다. |
| 실제 실패 / risk | clock 되돌리기는 timeout을 지연시키고, forged/unrelated PONG은 실제 challenge를 보지 않은 클라이언트도 wait 상태를 해제했습니다. |
| root cause | deadline clock과 protocol correlation이 각각 wall-clock 및 boolean 상태에만 의존했습니다. |
| 수정된 불변식 / 결정 | steady_clock elapsed time과 outstanding exact token을 함께 authoritative liveness 상태로 사용합니다. |
| 회귀 근거 | 0c76aad19579가 matching PONG은 유지하고 forged PONG은 timeout ERROR/close로 끝나는 두 경로를 분리합니다. |

#### 다음 연결

- 다음 커밋: `0c76aad19579` — `test(heartbeat): PONG 토큰과 시간 경계 검증`. 원문 역할은 “일치하는 PONG만 제한 시간을 해제하며 위조 PONG은 해제하지 못함을 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `0c76aad19579` — `test(heartbeat): PONG 토큰과 시간 경계 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, RESILIENCE, RISK |
| 원문에서 정한 역할 | 일치하는 PONG만 제한 시간을 해제하며 위조 PONG은 해제하지 못함을 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 일치하는 PONG만 제한 시간을 해제하며 위조 PONG은 해제하지 못함을 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | steady-clock/exact-token fix는 있었지만 양성·위조 response가 서로 다른 production 브랜치를 통과한다는 회귀 증거가 없었습니다. |
| 주요 문제와 설계 판단 | 테스트 peer의 auto-PONG을 끈 채 server PING token을 직접 읽어 exact PONG을 보내는 생존 case와, 다른 token을 보낸 뒤 exact `ERROR :Ping timeout` 및 close를 기다리는 forged case를 추가했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py or tools/irc_smoke_client.py — heartbeat token/deadline scenarios` |
| 상태 / 소유권 변화 | 테스트가 server가 발급한 token을 capture하고 scenario별 소켓/deadline을 독립 관리합니다. |
| 실패 또는 경계 | forged PONG은 awaiting 상태를 바꾸지 않아 original `lastPingAt` 기준 deadline에서 timeout되고 연결이 닫혀야 합니다. |
| 보장 / 비보장 | 보장: matching response만 liveness deadline을 해제하며 unrelated response는 연결을 보존하지 못합니다.<br>비보장: OS wall-clock을 실제로 뒤로 움직이는 테스트는 아니며 steady_clock type과 코드 검토가 그 부분의 근거입니다. |
| 다음 관련 변화 | 416efc91e580가 이 규약을 Linux/macOS와 sanitizer 빌드에서 반복하도록 합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0c76aad19579` | `tests/irc_contract.py` | `matching PONG scenario` | captured exact token response 후 연결 progress |
| `0c76aad19579` | `tests/irc_contract.py` | `forged PONG scenario` | 불일치 token 뒤 exact Ping timeout ERROR와 close |

- 조사 방법: GitHub에서 `0c76aad19579` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 현재 outstanding heartbeat token과 정확히 일치하는 PONG만 wait 상태를 해제해야 합니다. |
| 재현 실패 / 경계 | matching token과 forged/unrelated token, ping deadline입니다. |
| 테스트 방식 | real 프로세스/소켓 + auto-PONG disabled + server token capture + exact timeout 프레임 검사문입니다. |
| 통과하는 실제 실행 경로 | maintainClient PING 상태 → wire token → handlePong exact comparison → clear 또는 unchanged → timeout/close입니다. |
| 이 테스트가 증명하는 것 | token correlation의 positive/negative 브랜치와 deadline outcome을 결정적하게 구분합니다. |
| 이 테스트가 증명하지 않는 것 | 실제 system-clock jump나 모든 scheduler delay, token 보안성은 증명하지 않습니다. |
| 테스트 성격 | 결정적 liveness 회귀 |
| 막는 회귀 | any-PONG acceptance와 forged response로 idle 연결을 무기한 유지하는 문제를 막습니다. |

실행 기록:

- 실행 SHA: `0c76aad19579`
- repository에 정의된 실행 명령: `make test`의 heartbeat 규약 scenario
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| idle heartbeat policy | `764361c52b2a` | PING/awaiting/timeout 상태 | registration timeout 뒤 idle probe와 deadline을 적용합니다. | 초기 wall-clock/any-PONG이 부족함 |
| 테스트 peer response seam | `d710f29f38a4` | f313e707474f | matching PONG 자동응답과 positive real-소켓 흐름을 제공합니다. | negative token 브랜치는 아직 없음 |
| monotonic exact challenge | `3f2b3ae1d3f9` | steady_clock + pendingPongToken | ordinary activity와 unrelated PONG은 challenge를 완료하지 않습니다. | 0c76aad19579 matching/forged 회귀 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| wall-clock elapsed time | clock 되돌리기/adjustment으로 deadline 왜곡 | 3f2b3ae1d3f9 | steady_clock timestamp | 0c76aad19579은 코드 path, 실제 clock jump는 inspection 근거 |
| 아무 PONG이면 현재 challenge 응답 | forged/unrelated token으로 timeout 회피 | 3f2b3ae1d3f9 | exact 대기 중인 token comparison | 0c76aad19579 |
| 양성 PING/PONG만 검증 | 실제 결함 브랜치가 드러나지 않음 | 0c76aad19579 | auto-PONG off + forged response | exact ERROR/close 검사문 |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| activity/deadline timestamps | wall-clock time_t | ClientState steady_clock time points | 764361c52b2a → 3f2b3ae1d3f9 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| outstanding challenge identity | awaiting bool만 존재 | ClientState pendingPongToken | 3f2b3ae1d3f9 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| token 순서 | fd/time 조합 | IrcApplication monotonic counter | 3f2b3ae1d3f9 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 테스트 response policy | 항상 일반 line | IrcPeer auto_pong flag | d710f29f38a4 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: 등록 deadline 외에는 유휴 연결의 생존 여부를 확인하거나 제거하는 정책이 없었습니다.
- 마지막 커밋 `0c76aad19579` 시점의 상태: matching response만 liveness deadline을 해제하며 unrelated response는 연결을 보존하지 못합니다.
- 개발 흐름 안에서 강화된 핵심 불변식: idle heartbeat policy, 테스트 peer response seam, monotonic exact challenge.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: OS wall-clock을 실제로 뒤로 움직이는 테스트는 아니며 steady_clock type과 코드 검토가 그 부분의 근거입니다. 416efc91e580가 이 규약을 Linux/macOS와 sanitizer 빌드에서 반복하도록 합니다.
- 최종 설명: `764361c52b2a`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `0c76aad19579`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| activity receive | 764361c52b2a / 3f2b3ae1d3f9 | `IrcApplication::onLine` | lastActivityAt 갱신 | ordinary activity는 outstanding token을 clear하지 않음 |
| idle probe | 3f2b3ae1d3f9 | `maintainClient` | token 생성 → awaiting/token/time 저장 → PING send | send 실패면 lifecycle 종료 후 상태 재사용 안 함 |
| response | 3f2b3ae1d3f9 | `handlePong` | exact one-param token이면 awaiting/token clear | 불일치/누락은 unchanged |
| deadline | 3f2b3ae1d3f9 | `maintainClient` | steady elapsed가 ping timeout 미만이면 대기 | 초과 시 ERROR + requestClose |
| 회귀 | 0c76aad19579 | `real socket scenario` | matching PONG은 progress | forged PONG은 exact Ping timeout ERROR/close |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 06-heartbeat-liveness-correctness.md =====

===== BEGIN FILE: 07-output-queue-correctness-under-partial-failure.md =====
# 개발 흐름 07 — 부분 실패 상황의 출력 대기열 정확성

부제: 부분 실패에서의 송신 대기열 정확성

## 1. 개발 흐름의 목표

영속 partial-send 상태가 자원 limit 도입 뒤 어떤 arithmetic·retry 경계에서 깨질 수 있었는지 추적하고, 오버플로-safe admission과 injected send 스크립트가 exact-once ordered delivery 불변식을 어떻게 복구·검증하는지 학습합니다.

### 원문에서 정한 의의

부분 송신 자체는 구현되어 있었지만 출력 상한을 도입하자 크기 계산 오버플로, 비정상 시스템 호출 결과, 재시도 상태 손상이 드러났습니다. 미전송 바이트 수를 넘침 없이 검사하고 송신 함수를 주입해 드문 결과도 재현 가능한 단위 테스트로 고정합니다.

<details>
<summary>영문 원문</summary>

> This thread shows a foundational mechanism being revisited after resource limits make its boundary conditions observable. The later fix is not a feature extension: it protects the exact-once output invariant against arithmetic wraparound, impossible syscall results, and retry-state corruption. Injecting the send operation turns those rare conditions into reproducible unit tests.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- sent 앞부분과 unsent 뒷부분을 나타내는 최소 상태는 무엇인가?
- 대기 중인-byte limit은 backing buffer가 아니라 logical unsent bytes를 어떻게 측정하는가?
- `pending + byteCount` 비교가 어떤 값에서 wrap될 수 있으며 subtraction predicate는 왜 안전한가?
- zero, EINTR, EAGAIN, EWOULDBLOCK, EPIPE, impossible oversize return에서 offset과 queue는 어떻게 남는가?
- scripted sender가 real 소켓으로 만들기 어려운 순서를 어떻게 재현하는가?

### 원문에서 확인되는 불변식

- partial send는 kernel이 보고한 exact byte 수만큼만 offset을 전진해야 합니다.
- retryable/zero/terminal outcome에서도 아직 전송되지 않은 뒷부분은 순서와 내용이 보존되어야 합니다.
- 대기열 상한 계산은 정수 범위를 돌아서는 안 되며, 거부된 바이트 때문에 기존 전송 대기 상태가 바뀌어서는 안 됩니다.

### 원문에서 확인되는 구현 난점

- 드문 시스템 호출 결과와 크기 계산 실패를 타이밍에 의존하는 네트워크 우연 없이 재현하는 문제.
- 느린 수신자의 역압을 해당 연결의 전송 대기 상태에만 가두면서 정확한 바이트 수를 유지하는 문제.

## 3. 완료 기준

- a10fe961e2b1의 buffer/offset 불변식과 graceful drain을 byte 순서로 설명합니다.
- d7d85e518177의 exact admission 기준과 rejection 변경 순서를 기록했습니다.
- 881e59734a9a의 arithmetic proof와 impossible send-count guard를 parent change로 설명합니다.
- f34ab135c546의 스크립트 steps와 expected 대기 중인/출력 상태를 재구성했습니다.
- 각 테스트가 증명하는 exact-order/경계와 증명하지 않는 kernel scheduling 범위를 구분했습니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `a10fe961e2b1` | `feat(connection): 부분 송신 대기열 처리` | S | CORE, EVENT_IO, LIFECYCLE | 부분 송신 상태를 이벤트 사이에 보존하고 바이트 순서를 정확히 유지합니다. |
| 2 | `d7d85e518177` | `feat(buffer): 송신 대기열 크기 제한` | A | EVENT_IO, RESILIENCE, RISK | 미전송 출력에 강제 상한을 두고 대기열 추가 실패를 호출자에게 알립니다. |
| 3 | `881e59734a9a` | `fix(connection): 송신 대기열 계산과 재시도 상태 보호` | S | DEBUG, EVENT_IO, RISK | 크기 계산의 오버플로를 막고 모든 `send` 결과에서 오프셋 상태를 보호합니다. |
| 4 | `f34ab135c546` | `test(connection): 부분 송신과 대기열 경계 검증` | A | VERIFICATION, EVENT_IO, RISK | 부분 송신, 인터럽트, 역압, 0바이트 송신, 종료 오류, 잘못된 반환값을 결정적으로 재현합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `a10fe961e2b1` — `feat(connection): 부분 송신 대기열 처리`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | CORE, EVENT_IO, LIFECYCLE |
| 원문에서 정한 역할 | 부분 송신 상태를 이벤트 사이에 보존하고 바이트 순서를 정확히 유지합니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> 부분 송신 상태를 이벤트 사이에 보존하고 바이트 순서를 정확히 유지합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | 출력 bytes와 한 번의 send를 연결할 영속 progress 상태가 없었습니다. |
| 실패 / 경계 | EINTR은 같은 뒷부분을 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 핵심 결정 | `writeOffset_`로 sent 앞부분을 나타내고 `flushPending()`이 unsent 뒷부분만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — flushPending, queueRaw, queueLine, pendingBytes, wantsWrite` |
| 소유권 / lifecycle / 상태 변화 | `writeBuffer_[0:writeOffset_]`는 이미 전송된 앞부분, 나머지는 순서가 보존된 대기 중인 뒷부분입니다. 완료 시 buffer/offset을 reset하고 큰 consumed 앞부분만 compact합니다. |
| 이 커밋이 보장하는 것 | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 아직 보장하지 않는 것 | queue 크기는 무제한이며 `pending + append` 오버플로와 send가 요청보다 큰 값을 돌려주는 비정상 결과는 아직 방어하지 않습니다. |
| 후속 fix/테스트 연결 | 625ffc924de8이 write 관심 상태에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | 출력 bytes와 한 번의 send를 연결할 영속 progress 상태가 없었습니다. |
| 2. 실패 conditions / 경계 | EINTR은 같은 뒷부분을 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 3. 선택한 표현과 순서 | `writeOffset_`로 sent 앞부분을 나타내고 `flushPending()`이 unsent 뒷부분만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 4. authoritative 상태 | `writeBuffer_[0:writeOffset_]`는 이미 전송된 앞부분, 나머지는 순서가 보존된 대기 중인 뒷부분입니다. 완료 시 buffer/offset을 reset하고 큰 consumed 앞부분만 compact합니다. |
| 5. 결과 불변식 | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 6. 후속 보강 | 625ffc924de8이 write 관심 상태에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a10fe961e2b1` | `src/Connection.cpp` | `Connection::flushPending` | unsent 뒷부분 포인터/length와 return-count 기반 offset 전진 |
| `a10fe961e2b1` | `src/Connection.cpp` | `queueLine` | 기존 CR/LF 제거 후 CRLF 하나 append |
| `a10fe961e2b1` | `src/Connection.cpp` | `pendingBytes/wantsWrite` | logical unsent bytes가 server 관심 상태 판단의 입력 |

- 조사 방법: GitHub에서 `a10fe961e2b1` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`. 원문 역할은 “미전송 출력에 강제 상한을 두고 대기열 추가 실패를 호출자에게 알립니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | EVENT_IO, RESILIENCE, RISK |
| 원문에서 정한 역할 | 미전송 출력에 강제 상한을 두고 대기열 추가 실패를 호출자에게 알립니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 미전송 출력에 강제 상한을 두고 대기열 추가 실패를 호출자에게 알립니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | partial 출력 queue가 무제한이라 느리거나 읽지 않는 peer가 memory를 계속 점유할 수 있었습니다. |
| 주요 문제와 설계 판단 | Connection에 `maxPendingBytes_`를 추가하고 `pendingBytes()` 기준으로 `queueRaw/queueLine` admission을 제한했습니다. queue 함수는 bool을 반환하고 초과 시 close를 요청하며 Server send API가 실패를 caller에 전달했습니다. |
| 변경 파일·symbol | `include/Connection.hpp — maxPendingBytes and bool queue API`<br>`src/Connection.cpp — pendingBytes, canAppendPending, queueRaw, queueLine`<br>`src/Server.cpp — sendTo/queueRawTo propagation` |
| 상태 / 소유권 변화 | limit은 backing buffer size가 아니라 `writeBuffer_.size() - writeOffset_`인 logical unsent bytes를 지배합니다. |
| 실패 또는 경계 | 초과 append는 기존 대기 중인 bytes를 변경하지 않고 `outbound queue limit exceeded` close 상태를 만듭니다. |
| 보장 / 비보장 | 보장: 한 연결의 unsent 출력에 hard bound와 observable 실패 result를 제공합니다.<br>비보장: `pending + byteCount <= limit` 덧셈은 size_t wrap 가능성이 있고 CRLF 추가의 두 단계 경계도 충분히 증명되지 않았습니다. |
| 다음 관련 변화 | 881e59734a9a가 subtraction predicate와 send guard로 수정하고 f34ab135c546가 결정적하게 검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d7d85e518177` | `src/Connection.cpp` | `pendingBytes/canAppendPending` | logical unsent byte 기준 admission |
| `d7d85e518177` | `src/Server.cpp` | `sendTo/queueRawTo` | queue 실패를 애플리케이션까지 bool로 전달 |

- 조사 방법: GitHub에서 `d7d85e518177` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `881e59734a9a` — `fix(connection): 송신 대기열 계산과 재시도 상태 보호`. 원문 역할은 “크기 계산의 오버플로를 막고 모든 `send` 결과에서 오프셋 상태를 보호합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `881e59734a9a` — `fix(connection): 송신 대기열 계산과 재시도 상태 보호`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | DEBUG, EVENT_IO, RISK |
| 원문에서 정한 역할 | 크기 계산의 오버플로를 막고 모든 `send` 결과에서 오프셋 상태를 보호합니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> 크기 계산의 오버플로를 막고 모든 `send` 결과에서 오프셋 상태를 보호합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | 대기 중인 limit은 `pending + append <= limit` 덧셈을 사용했고 send return count를 요청 길이보다 작거나 같다고 암묵적으로 가정했습니다. rare retry 순서를 직접 주입할 seam도 없었습니다. |
| 실패 / 경계 | size_t wrap, 전달값+CRLF 경계, impossible oversize return은 기존 대기 중인 bytes/offset을 보존한 채 rejection 또는 close로 수렴합니다. EINTR/would-block/zero/hard error의 기존 뒷부분 보존 semantics도 주입 가능한 작업을 통과합니다. |
| 핵심 결정 | `ConnectionLimits::canAppendPending()`을 `pending <= limit && byteCount <= limit - pending`으로 바꾸고 line 전달값과 CRLF를 별도 safe admission으로 검사했습니다. `SendOperation`을 생성자에 주입하고 move 상태에 포함했으며, positive send count가 요청 size를 넘으면 offset을 전진시키지 않고 hard error/close로 처리했습니다. |
| 변경 파일·symbol | `include/Connection.hpp — SendOperation constructor seam`<br>`src/Connection.cpp — send wrapper, move, flushPending, queueLine`<br>`src/ConnectionLimits.hpp — canAppendPending` |
| 소유권 / lifecycle / 상태 변화 | send 작업도 Connection의 transport 상태와 함께 move되고, write offset은 검증된 실제 successful count에 의해서만 전진합니다. |
| 이 커밋이 보장하는 것 | queue admission 산술이 오버플로할 수 없고 모든 send outcome에서 unsent 뒷부분과 exact-order offset 불변식이 보호됩니다. |
| 아직 보장하지 않는 것 | 주입 seam은 send syscall 결과를 모델링하지만 실제 kernel scheduling, TCP peer 동작, write 준비 상태 fairness 자체는 보장하지 않습니다. |
| 후속 fix/테스트 연결 | f34ab135c546가 scripted sender로 산술·retry·partial·zero·terminal·invalid-count 브랜치를 모두 고정합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | 대기 중인 limit은 `pending + append <= limit` 덧셈을 사용했고 send return count를 요청 길이보다 작거나 같다고 암묵적으로 가정했습니다. rare retry 순서를 직접 주입할 seam도 없었습니다. |
| 2. 실패 conditions / 경계 | size_t wrap, 전달값+CRLF 경계, impossible oversize return은 기존 대기 중인 bytes/offset을 보존한 채 rejection 또는 close로 수렴합니다. EINTR/would-block/zero/hard error의 기존 뒷부분 보존 semantics도 주입 가능한 작업을 통과합니다. |
| 3. 선택한 표현과 순서 | `ConnectionLimits::canAppendPending()`을 `pending <= limit && byteCount <= limit - pending`으로 바꾸고 line 전달값과 CRLF를 별도 safe admission으로 검사했습니다. `SendOperation`을 생성자에 주입하고 move 상태에 포함했으며, positive send count가 요청 size를 넘으면 offset을 전진시키지 않고 hard error/close로 처리했습니다. |
| 4. authoritative 상태 | send 작업도 Connection의 transport 상태와 함께 move되고, write offset은 검증된 실제 successful count에 의해서만 전진합니다. |
| 5. 결과 불변식 | queue admission 산술이 오버플로할 수 없고 모든 send outcome에서 unsent 뒷부분과 exact-order offset 불변식이 보호됩니다. |
| 6. 후속 보강 | f34ab135c546가 scripted sender로 산술·retry·partial·zero·terminal·invalid-count 브랜치를 모두 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `881e59734a9a` | `src/ConnectionLimits.hpp` | `detail::canAppendPending` | subtraction predicate로 size_t wrap 방지 |
| `881e59734a9a` | `src/Connection.cpp` | `Connection::flushPending` | send count > requested guard가 offset 변경 전에 실행 |
| `881e59734a9a` | `src/Connection.cpp` | `queueLine` | 전달값과 CRLF를 각각 오버플로-safe하게 admission |
| `881e59734a9a` | `include/Connection.hpp` | `SendOperation` | rare syscall result를 실제 실행 경로에 주입 |

- 조사 방법: GitHub에서 `881e59734a9a` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | unsigned 덧셈이 limit 비교 전에 wrap하지 않고 send가 요청 길이 이하를 반환한다는 가정이었습니다. |
| 실제 실패 / risk | wrap된 합은 oversize queue를 허용하고 impossible return은 offset을 buffer 끝 너머로 이동시켜 대기 중인 계산과 retry 상태를 깨뜨릴 수 있었습니다. |
| root cause | admission과 progress 변경 전에 arithmetic/syscall postcondition을 명시적으로 검증하지 않았습니다. |
| 수정된 불변식 / 결정 | limit에서 대기 중인을 빼는 representable 비교와 return-count upper bound 검사를 변경 전에 수행합니다. |
| 회귀 근거 | f34ab135c546가 size_t maximum, exact CRLF limit, EINTR/partial/EAGAIN/EWOULDBLOCK/zero/EPIPE/oversize return을 스크립트로 검증합니다. |

#### 다음 연결

- 다음 커밋: `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`. 원문 역할은 “부분 송신, 인터럽트, 역압, 0바이트 송신, 종료 오류, 잘못된 반환값을 결정적으로 재현합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, EVENT_IO, RISK |
| 원문에서 정한 역할 | 부분 송신, 인터럽트, 역압, 0바이트 송신, 종료 오류, 잘못된 반환값을 결정적으로 재현합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 부분 송신, 인터럽트, 역압, 0바이트 송신, 종료 오류, 잘못된 반환값을 결정적으로 재현합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection fix의 rare syscall/오버플로 브랜치는 real 소켓 timing에 맡기면 반복적으로 재현하기 어려웠습니다. |
| 주요 문제와 설계 판단 | `SendStep`과 `ScriptedSender`가 Bytes/Error/Zero 순서를 반환하는 unit 실행 파일을 추가하고 production `Connection::flushPending()`에 주입했습니다. Makefile에 `connection-test`를 연결했습니다. |
| 변경 파일·symbol | `tests/connection_test.cpp — SendStep, ScriptedSender and five test groups`<br>`Makefile — connection-test` |
| 상태 / 소유권 변화 | 스크립트 index, captured bytes와 Connection 대기 중인/offset/close 상태를 각 단계 후 동시에 검사문합니다. |
| 실패 또는 경계 | EINTR→partial→EAGAIN→partial→EWOULDBLOCK→final 순서, zero then progress, EPIPE, requested size보다 큰 return, size_t max arithmetic와 exact CRLF limit을 결정적으로 재현합니다. |
| 보장 / 비보장 | 보장: successful bytes는 `abcdefghijkl` 순서로 정확히 한 번 전달되고 retryable/terminal/invalid outcome은 아직 안 보낸 bytes를 소비하지 않으며 rejected append는 queue를 변경하지 않습니다.<br>비보장: fake sender는 kernel buffer, actual 준비 상태 notification, TCP segmentation이나 peer scheduling을 검증하지 않습니다. |
| 다음 관련 변화 | de1dd0fc30d0가 real 프로세스/slow reader에서 연결-local queue isolation을 보완하고 416efc91e580가 sanitizer로 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f34ab135c546` | `tests/connection_test.cpp` | `testLimitArithmetic/testHardLimitAndLineTerminator` | size_t max와 CRLF 포함 exact-limit 변경 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInterruptedAndPartialSendState` | scripted retry마다 대기 중인 offset과 exact 출력 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testZeroAndTerminalErrorPreservePendingBytes` | zero/EPIPE가 뒷부분을 소비하지 않음 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInvalidSendCountCannotAdvanceOffset` | oversize return이 offset을 손상시키지 않음 |

- 조사 방법: GitHub에서 `f34ab135c546` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | logical 대기 중인 limit은 오버플로하지 않고, offset은 검증된 successful byte만큼만 전진하며 unsent 뒷부분이 exact order로 보존됩니다. |
| 재현 실패 / 경계 | size_t max, CRLF exact limit, EINTR, partial, EAGAIN/EWOULDBLOCK, zero, EPIPE, impossible oversize return입니다. |
| 테스트 방식 | injected send 작업을 사용하는 결정적 scripted unit 테스트입니다. |
| 통과하는 실제 실행 경로 | queueRaw/queueLine → 오버플로-safe admission → flushPending → injected send result → offset/close/대기 중인 query입니다. |
| 이 테스트가 증명하는 것 | rare outcome 순서에서 exact byte accounting과 변경-free rejection을 직접 증명합니다. |
| 이 테스트가 증명하지 않는 것 | real kernel 준비 상태, TCP delivery acknowledgement, end-to-end fairness는 증명하지 않습니다. |
| 테스트 성격 | 결정적 transport 경계 회귀 |
| 막는 회귀 | partial retry 중 byte 중복·유실, unsigned wrap, invalid return으로 offset이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `f34ab135c546`
- repository에 정의된 실행 명령: `make connection-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| 영속 sent-앞부분 상태 | `a10fe961e2b1` | writeOffset/대기 중인 뒷부분 | positive return만큼만 전진하고 retry 사이에 offset을 보존합니다. | 881e59734a9a invalid-count guard |
| hard logical 대기 중인 bound | `d7d85e518177` | pendingBytes + bool admission | limit 초과 append는 기존 queue를 변경하지 않습니다. | 초기 addition wrap이 부족함 |
| 오버플로-safe exact-order 불변식 | `881e59734a9a` | subtraction predicate + injected sender | all outcome에서 offset/뒷부분을 변경 전에 검증합니다. | f34ab135c546 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| 한 send가 전부 또는 정상 범위 반환 | partial/retry/invalid count에서 byte 중복·유실·offset 손상 | a10fe961e2b1 → 881e59734a9a | 영속 offset + count upper bound | f34ab135c546 |
| 대기 중인 + append 비교는 안전 | size_t wrap 후 oversize admission | 881e59734a9a | append <= limit - 대기 중인 | f34ab135c546 |
| rare syscall 순서는 real 소켓으로 충분 | timing에 따라 브랜치 미재현 | 881e59734a9a SendOperation seam | scripted 결정적 outcomes | f34ab135c546 |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| 출력 backing bytes/offset | 없음 | Connection | a10fe961e2b1 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| logical 대기 중인 limit | 무제한 | Connection maxPendingBytes | d7d85e518177 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| send 작업 | 직접 syscall 고정 | Connection-owned SendOperation | 881e59734a9a | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 실패 스크립트/captured 출력 | 없음 | connection_test ScriptedSender | f34ab135c546 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: 출력 bytes와 한 번의 send를 연결할 영속 progress 상태가 없었습니다.
- 마지막 커밋 `f34ab135c546` 시점의 상태: successful bytes는 `abcdefghijkl` 순서로 정확히 한 번 전달되고 retryable/terminal/invalid outcome은 아직 안 보낸 bytes를 소비하지 않으며 rejected append는 queue를 변경하지 않습니다.
- 개발 흐름 안에서 강화된 핵심 불변식: 영속 sent-앞부분 상태, hard logical 대기 중인 bound, 오버플로-safe exact-order 불변식.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: fake sender는 kernel buffer, actual 준비 상태 notification, TCP segmentation이나 peer scheduling을 검증하지 않습니다. de1dd0fc30d0가 real 프로세스/slow reader에서 연결-local queue isolation을 보완하고 416efc91e580가 sanitizer로 반복합니다.
- 최종 설명: `a10fe961e2b1`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `f34ab135c546`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| admission | 881e59734a9a | `queueRaw/queueLine` | 대기 중인 <= limit && append <= limit-대기 중인 | reject 시 existing queue/offset unchanged |
| writable call | a10fe961e2b1 / 881e59734a9a | `flushPending` | unsent 뒷부분 포인터/length로 SendOperation 호출 | count > requested는 terminal before offset 변경 |
| retry outcomes | 881e59734a9a | `flushPending` | EINTR retry, partial positive offset advance | EAGAIN/EWOULDBLOCK/zero는 뒷부분 보존, EPIPE는 close |
| completion | a10fe961e2b1 | `flushPending` | offset가 size에 도달하면 buffer/offset reset | closeRequested라면 Server가 drain 후 disconnect |
| 회귀 | f34ab135c546 | `ScriptedSender` | exact `abcdefghijkl` captured once | every 실패 step에서 대기 중인/출력 검사문 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 07-output-queue-correctness-under-partial-failure.md =====

===== BEGIN FILE: 08-reentrant-server-and-application-cleanup.md =====
# 개발 흐름 08 — 재진입 가능한 서버·애플리케이션 정리

부제: 재진입 가능한 서버와 애플리케이션 정리

## 1. 개발 흐름의 목표

콜백, 관심 상태 update, response enqueue가 현재 처리 중인 연결과 애플리케이션 aggregate를 동기적으로 제거할 수 있다는 사실을 중심으로, 파일 디스크립터 relookup·되돌리기·실패 propagation·명령 continuation 중단 불변식을 fix/테스트 쌍으로 복원합니다.

### 원문에서 정한 의의

송신이나 콜백처럼 지역적으로 보이는 호출이 현재 연결, 클라이언트 기록, 채널 멤버십과 채널 자체를 동기적으로 삭제할 수 있습니다. 서버는 파일 디스크립터와 이벤트 등록 소유권을 복구하고, 애플리케이션은 호출 뒤 상태를 다시 조회하며 행위자 수명이 끝나면 즉시 명령 처리를 중단하도록 일반화합니다.

<details>
<summary>영문 원문</summary>

> The difficult integration problem is that a seemingly local send or callback can synchronously remove the connection, client record, memberships, and possibly the channel currently referenced by the caller. The server fix restores descriptor and event-registry ownership; the application fix generalizes the rule to domain state. The follow-up commit and test refine the chosen semantics: earlier mutations are not rolled back transactionally, but command processing must stop immediately after the actor's lifecycle ends.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- 초기 erase-before-콜백 정리는 무엇을 보장하고 무엇을 아직 보장하지 않는가?
- 콜백 self-disconnect 뒤 raw 포인터/reference가 왜 즉시 위험해지는가?
- accept/listener registration 실패에서 map, event backend, fd 수명을 어떻게 되돌리기하는가?
- `sendTo()` 실패가 애플리케이션 클라이언트/index/membership/채널 erase로 이어지는 synchronous call stack은 무엇인가?
- send 뒤 언제 return하고 언제 stable id로 re-resolve해야 하는가?
- compound MODE의 부분 완료를 되돌리기하지 않으면서 후속 transition을 막는 semantics는 무엇인가?

### 원문에서 확인되는 불변식

- 콜백, 이벤트 관심 상태 갱신, 응답 대기열 추가 뒤에는 이전 `Connection`, 클라이언트, 채널, 포인터, 참조, 반복자를 재사용하지 말고 필요한 상태를 다시 조회해야 합니다.
- 연결 맵, 이벤트 관리자 등록, 파일 디스크립터 수명은 함께 추가되거나 함께 정리되어야 합니다.
- 출력 실패로 행위자의 수명이 끝났다면 이미 완료한 변경을 되돌리지 않더라도 이후 명령 상태 변경은 실행하지 않아야 합니다.

### 원문에서 확인되는 구현 난점

- 단일 스레드 코드에서도 외부 콜백이 현재 소유 객체를 동기적으로 삭제할 수 있는 재진입 수명 문제.
- 단순한 송신처럼 보이는 지역 동작이 연결 종료 콜백과 채널 삭제까지 일으키는 계층 간 무효화 문제.
- 드문 이벤트 등록 추가·갱신 실패와 정확한 중단 지점을 가짜 백엔드·제한값으로 결정적으로 재현하는 문제.

## 3. 완료 기준

- 7a6bc7e1276a의 정리 ordering을 baseline으로 기록했습니다.
- 5dcd882f0763의 콜백/registration/update 실패 경계와 fd-based relookup/되돌리기를 설명합니다.
- 928594ec160c의 five 실패 contracts와 fake backend injection path를 구분했습니다.
- 728aaabc4012의 cross-layer send-실패 invalidation rule을 stable copy/relookup으로 복원했습니다.
- d48e1f1f8c04/aee5edebe294의 non-transactional stop-after-실패 semantics를 실제 상태 검사문으로 설명합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `7a6bc7e1276a` | `feat(server): 연결 해제와 오류 정리 구현` | A | LIFECYCLE, RISK | 연결 종료 콜백 전에 맵에서 지우는 초기 정리 경로를 정합니다. |
| 2 | `5dcd882f0763` | `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장` | S | DEBUG, LIFECYCLE, RISK | 콜백이 자신의 연결을 제거해도 안전하게 처리하고 이벤트 등록 실패를 되돌립니다. |
| 3 | `928594ec160c` | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` | A | VERIFICATION, LIFECYCLE, RISK | 등록 추가·갱신 실패와 콜백 제거를 주입해 서버 객체 수명을 검증합니다. |
| 4 | `728aaabc4012` | `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인` | S | DEBUG, LIFECYCLE, RISK | 출력 실패를 상위로 전달하고 연결을 끊을 수 있는 송신 뒤에는 애플리케이션 상태를 다시 조회하게 합니다. |
| 5 | `5edcafda8a4d` | `test(app): 작은 송신 한도에서 상태 정리 검증` | A | VERIFICATION, LIFECYCLE, RISK | 등록 응답 실패를 강제로 발생시켜 애플리케이션 정리와 로그 순서를 검증합니다. |
| 6 | `d48e1f1f8c04` | `fix(app): 응답 실패 뒤 명령 처리를 중단` | A | DEBUG, LIFECYCLE, RISK | 응답 전송 실패 뒤 LIST, NAMES, 복합 MODE 처리를 중단합니다. |
| 7 | `aee5edebe294` | `test(app): 연결 정리 뒤 모드 변경 중단 검증` | A | VERIFICATION, LIFECYCLE, CHANNEL_STATE | 복합 MODE는 이미 완료한 변경만 남기고 발신자 제거 뒤 후속 변경을 수행하지 않음을 검증합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `7a6bc7e1276a` — `feat(server): 연결 해제와 오류 정리 구현`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | LIFECYCLE, RISK |
| 원문에서 정한 역할 | 연결 종료 콜백 전에 맵에서 지우는 초기 정리 경로를 정합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 연결 종료 콜백 전에 맵에서 지우는 초기 정리 경로를 정합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 여러 event/error/stop 경로에서 연결 제거 순서가 하나의 함수로 고정되지 않았습니다. |
| 주요 문제와 설계 판단 | `disconnect()`가 backend remove를 시도하고 `unique_ptr`를 map 밖 local로 옮겨 entry를 먼저 erase한 뒤 disconnect 콜백을 호출했습니다. bulk close는 fd 스냅샷을 사용했습니다. |
| 변경 파일·symbol | `src/Server.cpp — disconnect, closeAllConnections, handleClientEvent` |
| 상태 / 소유권 변화 | 콜백 시점에는 server map에서 fd가 이미 사라졌지만 local unique_ptr가 콜백 동안 객체 수명을 유지하고 콜백 후 소멸자가 fd를 닫습니다. |
| 실패 또는 경계 | backend remove와 콜백 exception은 report되더라도 object 소멸을 막지 않습니다. hangup은 대기 중인 출력이 없을 때 terminal입니다. |
| 보장 / 비보장 | 보장: recursive lookup/repeated disconnect는 이미 제거된 상태를 보고, map iteration 중 erase로 반복자가 깨지지 않습니다.<br>비보장: 콜백 전에 잡은 raw 포인터를 콜백 뒤 다시 사용하는 call site와 event add/update 되돌리기는 아직 안전하지 않습니다. |
| 다음 관련 변화 | 5dcd882f0763/928594ec160c가 이 baseline의 reentrancy와 되돌리기 결함을 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::disconnect` | event remove → map 소유권 move/erase → 콜백 → 소멸 순서 |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::closeAllConnections` | fd 스냅샷으로 반복 erase 안전성 |

- 조사 방법: GitHub에서 `7a6bc7e1276a` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### 다음 연결

- 다음 커밋: `5dcd882f0763` — `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장`. 원문 역할은 “콜백이 자신의 연결을 제거해도 안전하게 처리하고 이벤트 등록 실패를 되돌립니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `5dcd882f0763` — `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | DEBUG, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 콜백이 자신의 연결을 제거해도 안전하게 처리하고 이벤트 등록 실패를 되돌립니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> 콜백이 자신의 연결을 제거해도 안전하게 처리하고 이벤트 등록 실패를 되돌립니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | Server는 콜백 전에 map에서 연결을 지우는 disconnect 경로는 가졌지만, connect/line 콜백이 자신을 제거한 뒤 기존 raw 포인터를 다시 쓰고 event add/update 실패 시 map·backend·fd 상태가 갈라질 수 있었습니다. |
| 실패 / 경계 | listener add 실패는 listen fd를 닫고 backend 소유자를 reset합니다. 클라이언트 add 실패는 inserted map entry를 erase해 Connection 소멸자가 fd를 닫습니다. connect/line 콜백 self-disconnect 뒤 lookup이 null이면 즉시 중단하고 update 실패는 backend/map 모두 제거합니다. |
| 핵심 결정 | 테스트용 EventManager 주입 생성자를 추가하고 listener add를 되돌리기 가능하게 했습니다. accept에서는 map에 먼저 `emplace`한 뒤 backend add가 실패하면 entry를 erase했습니다. 콜백과 read/write 단계 후에는 fd로 `findConnection()`을 다시 호출하고, `refreshInterest(int fd)`는 update exception을 잡아 disconnect한 뒤 bool을 반환하도록 바꿨습니다. error 콜백 예외는 `noexcept` report path에서 흡수했습니다. |
| 변경 파일·symbol | `include/Server.hpp — injected EventManager constructor, refreshInterest(int)`<br>`src/Server.cpp — start rollback, acceptReadyClients, handleClientEvent, refreshInterest, reportError` |
| 소유권 / lifecycle / 상태 변화 | stable identity는 fd이고 `Connection*`는 다음 external 콜백/관심 상태 작업 전까지만 유효합니다. 연결 map, backend registration, 파일 디스크립터 수명이 성공 시 함께 전진하고 실패 시 authoritative disconnect/erase로 함께 teardown됩니다. |
| 이 커밋이 보장하는 것 | single-threaded 콜백 reentrancy와 event registration/update 실패 뒤 stale 포인터, leaked 연결, ghost backend entry가 남지 않습니다. |
| 아직 보장하지 않는 것 | 애플리케이션 handler가 response send 후 borrowed ClientState/Channel/반복자를 재사용하는 cross-layer 문제는 아직 남습니다. |
| 후속 fix/테스트 연결 | 928594ec160c가 fake backend로 five server 실패 contracts를 검증하고 728aaabc4012가 동일 규칙을 애플리케이션 상태까지 확장합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | Server는 콜백 전에 map에서 연결을 지우는 disconnect 경로는 가졌지만, connect/line 콜백이 자신을 제거한 뒤 기존 raw 포인터를 다시 쓰고 event add/update 실패 시 map·backend·fd 상태가 갈라질 수 있었습니다. |
| 2. 실패 conditions / 경계 | listener add 실패는 listen fd를 닫고 backend 소유자를 reset합니다. 클라이언트 add 실패는 inserted map entry를 erase해 Connection 소멸자가 fd를 닫습니다. connect/line 콜백 self-disconnect 뒤 lookup이 null이면 즉시 중단하고 update 실패는 backend/map 모두 제거합니다. |
| 3. 선택한 표현과 순서 | 테스트용 EventManager 주입 생성자를 추가하고 listener add를 되돌리기 가능하게 했습니다. accept에서는 map에 먼저 `emplace`한 뒤 backend add가 실패하면 entry를 erase했습니다. 콜백과 read/write 단계 후에는 fd로 `findConnection()`을 다시 호출하고, `refreshInterest(int fd)`는 update exception을 잡아 disconnect한 뒤 bool을 반환하도록 바꿨습니다. error 콜백 예외는 `noexcept` report path에서 흡수했습니다. |
| 4. authoritative 상태 | stable identity는 fd이고 `Connection*`는 다음 external 콜백/관심 상태 작업 전까지만 유효합니다. 연결 map, backend registration, 파일 디스크립터 수명이 성공 시 함께 전진하고 실패 시 authoritative disconnect/erase로 함께 teardown됩니다. |
| 5. 결과 불변식 | single-threaded 콜백 reentrancy와 event registration/update 실패 뒤 stale 포인터, leaked 연결, ghost backend entry가 남지 않습니다. |
| 6. 후속 보강 | 928594ec160c가 fake backend로 five server 실패 contracts를 검증하고 728aaabc4012가 동일 규칙을 애플리케이션 상태까지 확장합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5dcd882f0763` | `src/Server.cpp` | `Server::start` | listener add 예외 시 listen fd close와 backend reset |
| `5dcd882f0763` | `src/Server.cpp` | `acceptReadyClients` | map emplace → backend add → 콜백, add 실패 시 erase 되돌리기 |
| `5dcd882f0763` | `src/Server.cpp` | `handleClientEvent` | 콜백/read/write 후 fd relookup과 null이면 즉시 return |
| `5dcd882f0763` | `src/Server.cpp` | `refreshInterest(int)` | update 실패 report → disconnect → false |
| `5dcd882f0763` | `src/Server.cpp` | `reportError noexcept` | user error 콜백 예외가 정리를 중단하지 않음 |

- 조사 방법: GitHub에서 `5dcd882f0763` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | single-threaded 콜백과 event update가 반환되면 이전 Connection 포인터/reference가 계속 유효하고 등록 단계가 부분 실패하지 않는다는 가정이었습니다. |
| 실제 실패 / risk | 콜백 self-disconnect는 map object를 파괴해 use-after-free를 만들고 add/update exception은 map과 kernel registry 중 한쪽만 남길 수 있었습니다. |
| root cause | 콜백/foreign 작업을 lifecycle 변경 경계로 취급하지 않았고 map/backend/fd 변화가 transaction-like ordering을 갖지 않았습니다. |
| 수정된 불변식 / 결정 | fd만 stable key로 보존하고 각 경계 뒤 relookup하며, 등록은 map 소유자를 먼저 세운 뒤 backend 실패를 erase로 되돌리기하고 update 실패는 disconnect로 수렴합니다. |
| 회귀 근거 | 928594ec160c가 add/update injection 및 connect/line 콜백 removal을 production Server 경로에서 검증합니다. |

#### 다음 연결

- 다음 커밋: `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`. 원문 역할은 “등록 추가·갱신 실패와 콜백 제거를 주입해 서버 객체 수명을 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 등록 추가·갱신 실패와 콜백 제거를 주입해 서버 객체 수명을 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 등록 추가·갱신 실패와 콜백 제거를 주입해 서버 객체 수명을 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | server 수명 fix는 있었지만 add/update 실패와 콜백 self-removal을 real backend/timing에 의존하지 않고 재현하는 회귀 테스트 모음이 없었습니다. |
| 주요 문제와 설계 판단 | `FakeEventManager`에 관심 상태 map, queued events, fail-next-add/update seam을 두고 loopback ClientSocket과 `Server::pollOnce()`를 결합한 `server_lifetime_test`를 추가했습니다. |
| 변경 파일·symbol | `tests/server_lifetime_test.cpp — FakeEventManager, ClientSocket, five tests`<br>`Makefile — unit/server lifetime target` |
| 상태 / 소유권 변화 | fake backend가 등록 fd와 호출 횟수를 관찰하고 테스트가 real accepted 소켓과 Server 연결 count를 함께 검사문합니다. |
| 실패 또는 경계 | (1) 클라이언트 add 실패 되돌리기, (2) connect 콜백 self-disconnect+throw, (3) line 콜백 self-disconnect+throw, (4) write 관심 상태 update 실패, (5) outbound queue rejection close를 각각 결정적으로 주입합니다. |
| 보장 / 비보장 | 보장: 각 case 뒤 연결 map과 fake backend에 클라이언트 fd가 남지 않고 콜백 예외가 정리를 무효화하지 않으며 queue/update 실패가 false를 반환합니다.<br>비보장: fake backend는 epoll/kqueue native semantics 자체를 검증하지 않고 콜백 이후 애플리케이션 채널/클라이언트 graph도 직접 검사하지 않습니다. |
| 다음 관련 변화 | 728aaabc4012/5edcafda8a4d가 send 실패가 애플리케이션 상태와 log까지 정리되는 경로를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `registrationRollbackTest` | injected add 실패 뒤 map/backend가 모두 비어 있음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `connectCallbackLifetimeTest/lineCallbackLifetimeTest` | 콜백 self-disconnect와 throw 뒤 stale access 없음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `interestUpdateRollbackTest` | update 실패가 send false와 map/backend removal로 수렴 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `queueLimitCloseTest` | queue rejection close가 event/map entry를 남기지 않음 |

- 조사 방법: GitHub에서 `928594ec160c` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 연결 map, event registration, 파일 디스크립터 수명이 함께 전진·정리되고 콜백 뒤 stale 포인터를 재사용하지 않아야 합니다. |
| 재현 실패 / 경계 | event add/update exception, connect/line 콜백 self-disconnect+throw, outbound queue rejection입니다. |
| 테스트 방식 | injected FakeEventManager + real loopback accepted 소켓 + direct `pollOnce()` 결정적 수명 unit/통합입니다. |
| 통과하는 실제 실행 경로 | accept/read/send → Server map/backend → 콜백/refreshInterest → disconnect/erase/소멸입니다. |
| 이 테스트가 증명하는 것 | server-level 소유권 되돌리기와 콜백 relookup rule을 timing accident 없이 검증합니다. |
| 이 테스트가 증명하지 않는 것 | native epoll/kqueue kernel 동작, 애플리케이션 aggregate 정리, 모든 OS 자원 실패는 증명하지 않습니다. |
| 테스트 성격 | 결정적 server 수명 회귀 |
| 막는 회귀 | self-disconnect use-after-free와 partial event registration/관심 상태 update leak을 막습니다. |

실행 기록:

- 실행 SHA: `928594ec160c`
- repository에 정의된 실행 명령: `make unit` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `728aaabc4012` — `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인`. 원문 역할은 “출력 실패를 상위로 전달하고 연결을 끊을 수 있는 송신 뒤에는 애플리케이션 상태를 다시 조회하게 합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `728aaabc4012` — `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인`

| 항목 | 값 |
| --- | --- |
| 중요도 | S |
| 태그 | DEBUG, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 출력 실패를 상위로 전달하고 연결을 끊을 수 있는 송신 뒤에는 애플리케이션 상태를 다시 조회하게 합니다. |
| 학습 깊이 | 프로젝트 핵심 architecture/불변식입니다. 이전 상태, 실패 순서, 핵심 결정, 소유권/lifecycle, 남은 한계와 후속 fix/테스트를 모두 복원합니다. |

#### 원문에서 정한 역할

> 출력 실패를 상위로 전달하고 연결을 끊을 수 있는 송신 뒤에는 애플리케이션 상태를 다시 조회하게 합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | Server.sendTo가 queue/관심 상태 실패 시 synchronous disconnect와 애플리케이션 disconnect 콜백을 실행할 수 있었지만 handlers는 send 뒤 기존 ClientState, Channel reference, 반복자를 계속 사용했습니다. |
| 실패 / 경계 | queue limit 또는 관심 상태 update 실패가 `Server::disconnect()` → 애플리케이션 `onDisconnect()` → registry membership/채널 erase를 동기 실행할 수 있으므로 false 또는 missing relookup에서 명령을 종료합니다. |
| 핵심 결정 | `sendRaw`, `sendNumeric`, `sendNumericRaw`, `sendTopicReply`, `sendNames`, `broadcastMode`가 bool을 반환하도록 전파했습니다. handler는 send 전에 채널 name/nick/앞부분 같은 stable value를 복사하고, send/broadcast 뒤 `_clients.contains/find`와 `_channels.find`로 다시 resolve하거나 실패 즉시 return하도록 바꿨습니다. |
| 변경 파일·symbol | `src/IrcApplication.hpp/.cpp — bool send helpers and support methods`<br>`src/ChannelCommands.cpp — JOIN/PART/KICK/INVITE/MODE revalidation`<br>`src/RegistrationCommands.cpp — registration response checks`<br>`src/ApplicationSupport.cpp — fan-out/relookup` |
| 소유권 / lifecycle / 상태 변화 | send는 단순 출력 append가 아니라 actor lifecycle을 끝낼 수 있는 변경 경계입니다. 포인터/reference/반복자 대신 fd와 copied 채널 name/nick이 경계를 넘는 stable identity가 됩니다. |
| 이 커밋이 보장하는 것 | 응답 실패 뒤 삭제된 클라이언트/채널 상태에 접근하지 않고 후속 response/변경을 기존 borrowed object로 수행하지 않습니다. |
| 아직 보장하지 않는 것 | LIST/NAMES 반복과 compound MODE 일부 브랜치는 아직 send 실패 뒤 loop를 계속할 수 있어 d48e1f1f8c04에서 추가 수정됩니다. 이미 성공한 변경을 transaction처럼 되돌리기하지는 않습니다. |
| 후속 fix/테스트 연결 | 5edcafda8a4d가 one-byte queue로 registration 실패 정리/log ordering을 검증하고 d48e1f1f8c04/aee5edebe294가 stop-after-실패 semantics를 완성합니다. |

#### 중요도 S 불변식 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | Server.sendTo가 queue/관심 상태 실패 시 synchronous disconnect와 애플리케이션 disconnect 콜백을 실행할 수 있었지만 handlers는 send 뒤 기존 ClientState, Channel reference, 반복자를 계속 사용했습니다. |
| 2. 실패 conditions / 경계 | queue limit 또는 관심 상태 update 실패가 `Server::disconnect()` → 애플리케이션 `onDisconnect()` → registry membership/채널 erase를 동기 실행할 수 있으므로 false 또는 missing relookup에서 명령을 종료합니다. |
| 3. 선택한 표현과 순서 | `sendRaw`, `sendNumeric`, `sendNumericRaw`, `sendTopicReply`, `sendNames`, `broadcastMode`가 bool을 반환하도록 전파했습니다. handler는 send 전에 채널 name/nick/앞부분 같은 stable value를 복사하고, send/broadcast 뒤 `_clients.contains/find`와 `_channels.find`로 다시 resolve하거나 실패 즉시 return하도록 바꿨습니다. |
| 4. authoritative 상태 | send는 단순 출력 append가 아니라 actor lifecycle을 끝낼 수 있는 변경 경계입니다. 포인터/reference/반복자 대신 fd와 copied 채널 name/nick이 경계를 넘는 stable identity가 됩니다. |
| 5. 결과 불변식 | 응답 실패 뒤 삭제된 클라이언트/채널 상태에 접근하지 않고 후속 response/변경을 기존 borrowed object로 수행하지 않습니다. |
| 6. 후속 보강 | 5edcafda8a4d가 one-byte queue로 registration 실패 정리/log ordering을 검증하고 d48e1f1f8c04/aee5edebe294가 stop-after-실패 semantics를 완성합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `728aaabc4012` | `src/IrcApplication.cpp` | `sendRaw/sendNumeric/sendNumericRaw` | Server send 결과를 애플리케이션 caller에 bool로 전파 |
| `728aaabc4012` | `src/ChannelCommands.cpp` | `handleJoin/part/kick/invite` | stable string copy와 send 뒤 클라이언트/채널 relookup |
| `728aaabc4012` | `src/IrcApplication.cpp` | `maintainClient` | heartbeat 상태를 send 전에 설정하고 실패 시 removed 상태를 재사용하지 않음 |

- 조사 방법: GitHub에서 `728aaabc4012` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | response enqueue는 local 작업이라 caller가 보유한 클라이언트/채널 reference를 무효화하지 않는다는 가정이었습니다. |
| 실제 실패 / risk | queue/관심 상태 실패는 synchronous disconnect 콜백을 통해 registry, memberships, empty 채널을 지워 handler의 포인터/반복자를 dangling으로 만들었습니다. |
| root cause | cross-layer call stack의 lifecycle side effect가 send API의 void signature와 handler continuation에 숨겨져 있었습니다. |
| 수정된 불변식 / 결정 | send 실패를 bool로 끝까지 전파하고 stable id/value만 경계를 넘기며 모든 이후 상태는 authoritative 컨테이너에서 다시 resolve합니다. |
| 회귀 근거 | 5edcafda8a4d가 registration response 실패를, aee5edebe294가 compound MODE sender removal을 재현합니다. |

#### 다음 연결

- 다음 커밋: `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`. 원문 역할은 “등록 응답 실패를 강제로 발생시켜 애플리케이션 정리와 로그 순서를 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 등록 응답 실패를 강제로 발생시켜 애플리케이션 정리와 로그 순서를 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 등록 응답 실패를 강제로 발생시켜 애플리케이션 정리와 로그 순서를 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 애플리케이션 revalidation fix가 welcome 응답 실패로 연결이 제거되는 실제 cross-layer stack에서 상태/log를 올바르게 정리한다는 테스트가 없었습니다. |
| 주요 문제와 설계 판단 | FakeEventManager와 captured stderr를 사용하는 `application_lifetime_test`를 추가하고 Server `maxPendingBytes=1`로 설정해 NICK/USER 등록 중 첫 welcome 프레임 admission을 강제로 실패시켰습니다. |
| 변경 파일·symbol | `tests/application_lifetime_test.cpp — CapturedStderr, FakeEventManager, registrationQueueFailureTest`<br>`Makefile — application-test` |
| 상태 / 소유권 변화 | real accepted 소켓과 애플리케이션 callbacks를 사용하되 tiny queue limit가 결정적 disconnect seam이며 테스트가 server 연결 count, running 상태와 log text를 관찰합니다. |
| 실패 또는 경계 | welcome queue 실패는 Server disconnect와 애플리케이션 onDisconnect 정리를 동기 실행합니다. |
| 보장 / 비보장 | 보장: 실패 뒤 연결 count가 0이고 server는 계속 실행되며 제거된 클라이언트를 `event=client_registered`로 기록하지 않습니다.<br>비보장: 이 최초 애플리케이션 테스트는 채널/mode의 multi-step continuation을 검사하지 않습니다. |
| 다음 관련 변화 | d48e1f1f8c04/aee5edebe294가 LIST/NAMES/MODE continuation과 부분 완료 semantics를 추가로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5edcafda8a4d` | `tests/application_lifetime_test.cpp` | `registrationQueueFailureTest` | 1-byte limit → welcome send 실패 → 연결 정리, server running, registered log 부재 |

- 조사 방법: GitHub에서 `5edcafda8a4d` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | registration response 실패로 actor가 제거되면 애플리케이션 상태와 lifecycle log가 제거 결과와 일치해야 합니다. |
| 재현 실패 / 경계 | `maxPendingBytes=1`에서 001 welcome 프레임 queue rejection입니다. |
| 테스트 방식 | tiny real Server queue limit + fake event backend + real loopback 연결 + captured stderr입니다. |
| 통과하는 실제 실행 경로 | NICK/USER → maybeRegister → sendNumeric → Server.sendTo → queue rejection/refresh/disconnect → onDisconnect 정리입니다. |
| 이 테스트가 증명하는 것 | cross-layer synchronous removal 뒤 phantom registered 상태/log가 없고 server가 프로세스-wide 실패로 멈추지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 handler, compound iteration, native backend 실패는 증명하지 않습니다. |
| 테스트 성격 | 결정적 애플리케이션 수명 회귀 |
| 막는 회귀 | welcome 실패 후 removed 클라이언트를 계속 등록 처리·로그하는 use-after-lifecycle 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `5edcafda8a4d`
- repository에 정의된 실행 명령: `make application-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `d48e1f1f8c04` — `fix(app): 응답 실패 뒤 명령 처리를 중단`. 원문 역할은 “응답 전송 실패 뒤 LIST, NAMES, 복합 MODE 처리를 중단합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `d48e1f1f8c04` — `fix(app): 응답 실패 뒤 명령 처리를 중단`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | DEBUG, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 응답 전송 실패 뒤 LIST, NAMES, 복합 MODE 처리를 중단합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 응답 전송 실패 뒤 LIST, NAMES, 복합 MODE 처리를 중단합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 주요 handlers는 send 결과를 확인했지만 LIST/NAMES 반복과 compound MODE는 중간 response/broadcast 실패 뒤 다음 iteration/mode 변경을 계속할 수 있었습니다. |
| 주요 문제와 설계 판단 | LIST header/item, NAMES 채널/item, MODE i/t/o/unknown response의 bool 결과를 즉시 검사해 false면 return하도록 했습니다. 채널 name과 대상 nick을 stable copy하고 broadcast 뒤 continuation 전에 actor/채널 존재를 확인했습니다. |
| 변경 파일·symbol | `src/ChannelCommands.cpp — handleList, handleNames, handleChannelMode` |
| 상태 / 소유권 변화 | compound 명령은 앞서 완료한 변경을 유지하지만 actor lifecycle이 끝난 정확한 경계 이후에는 cursor를 더 전진시키지 않습니다. |
| 실패 또는 경계 | 첫 mode 변경의 broadcast가 sender disconnect를 유발하면 뒤 mode 문자는 처리되지 않습니다. 반복 reply도 첫 실패에서 끝나 stale 반복자/reference를 재사용하지 않습니다. |
| 보장 / 비보장 | 보장: 출력 실패 이후 명령 continuation은 즉시 중단됩니다.<br>비보장: transactional 되돌리기를 제공하지 않으므로 실패 이전에 성공한 변경은 남습니다. 이것이 선택된 semantics입니다. |
| 다음 관련 변화 | aee5edebe294가 `MODE #room +it`에서 +i는 유지되고 +t는 적용되지 않는 상태 검사문으로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d48e1f1f8c04` | `src/ChannelCommands.cpp` | `handleList/handleNames` | 각 send 실패에서 반복 즉시 return |
| `d48e1f1f8c04` | `src/ChannelCommands.cpp` | `handleChannelMode` | mode별 broadcast/send false 뒤 cursor continuation 중단 |

- 조사 방법: GitHub에서 `d48e1f1f8c04` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | handler entry에서 revalidation하면 같은 명령 안의 이후 send/iteration도 안전하다는 가정이었습니다. |
| 실제 실패 / risk | 각 send가 actor를 제거할 수 있어 list loop나 mode cursor가 다음 transition을 삭제된 sender 권한으로 계속 수행했습니다. |
| root cause | 실패 propagation은 추가되었지만 모든 continuation point가 return 조건으로 연결되지 않았습니다. |
| 수정된 불변식 / 결정 | 각 출력 경계의 bool 결과를 그 자리에서 검사하고 compound 작업은 non-transactional partial completion 후 즉시 종료합니다. |
| 회귀 근거 | aee5edebe294가 첫 mode만 남고 later mode는 적용되지 않는 exact 상태를 검증합니다. |

#### 다음 연결

- 다음 커밋: `aee5edebe294` — `test(app): 연결 정리 뒤 모드 변경 중단 검증`. 원문 역할은 “복합 MODE는 이미 완료한 변경만 남기고 발신자 제거 뒤 후속 변경을 수행하지 않음을 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `aee5edebe294` — `test(app): 연결 정리 뒤 모드 변경 중단 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, LIFECYCLE, CHANNEL_STATE |
| 원문에서 정한 역할 | 복합 MODE는 이미 완료한 변경만 남기고 발신자 제거 뒤 후속 변경을 수행하지 않음을 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 복합 MODE는 이미 완료한 변경만 남기고 발신자 제거 뒤 후속 변경을 수행하지 않음을 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | stop-after-실패 semantics는 코드에 반영됐지만 compound MODE의 partial completion과 unrelated peer 보존을 상태 level에서 확인하지 않았습니다. |
| 주요 문제와 설계 판단 | white-box 애플리케이션 테스트 준비 코드와 fd-specific update 실패를 추가했습니다. 두 registered 클라이언트가 있는 `#room`에서 sender를 operator로 두고 topic protection을 끈 뒤 `MODE #room +it`를 실행하며 첫 `+i` broadcast 관심 상태 update에서 sender를 제거했습니다. |
| 변경 파일·symbol | `tests/application_lifetime_test.cpp — FakeEventManager::failNextUpdateFor, modeStopsAfterSenderCleanupTest` |
| 상태 / 소유권 변화 | 테스트는 `_clients`와 `_channels`를 직접 구성·관찰하고 실패 전/후 채널 flags와 두 클라이언트 존재를 각각 저장한 뒤 정리합니다. |
| 실패 또는 경계 | `+i` 변경은 broadcast 전에 이미 완료되지만 update 실패가 sender disconnect를 유발하므로 handler가 return해 다음 `+t` 변경은 실행하지 않습니다. |
| 보장 / 비보장 | 보장: 채널은 peer 때문에 유지되고 `inviteOnly=true`, `topicProtected=false`, sender removed, peer present라는 non-transactional stop semantics가 고정됩니다.<br>비보장: white-box setup은 공개 IRC parser/authorization 전체를 통과하지 않으며 한 compound mode pattern만 검사합니다. |
| 다음 관련 변화 | 416efc91e580가 애플리케이션 수명 suite를 두 OS와 sanitizer에서 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `aee5edebe294` | `tests/application_lifetime_test.cpp` | `modeStopsAfterSenderCleanupTest` | injected sender update 실패 뒤 +i 유지, +t 미적용, sender만 제거 |
| `aee5edebe294` | `tests/application_lifetime_test.cpp` | `FakeEventManager::failNextUpdateFor` | 정확한 fd의 다음 관심 상태 update만 실패 |

- 조사 방법: GitHub에서 `aee5edebe294` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | actor lifecycle이 출력 경계에서 끝나면 완료된 변경은 유지하되 같은 compound 명령의 이후 transition은 실행하지 않아야 합니다. |
| 재현 실패 / 경계 | `MODE #room +it`의 첫 `+i` broadcast에서 sender 관심 상태 update 실패입니다. |
| 테스트 방식 | white-box 애플리케이션 상태 + fd-specific fake backend 실패 주입입니다. |
| 통과하는 실제 실행 경로 | handleChannelMode → setInviteOnly → broadcastMode → Server.update 실패/disconnect → false → handler return입니다. |
| 이 테스트가 증명하는 것 | partial completion, sender-only 정리, peer/채널 보존과 later mode suppression을 exact 상태로 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 compound MODE 조합이나 공개 wire path, transactional 되돌리기는 증명하지 않습니다. |
| 테스트 성격 | 결정적 명령-continuation 회귀 |
| 막는 회귀 | sender 제거 뒤 남은 mode 문자를 실행하거나 unrelated peer/채널을 함께 지우는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `aee5edebe294`
- repository에 정의된 실행 명령: `make application-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| erase-before-콜백 baseline | `7a6bc7e1276a` | Server::disconnect | recursive lookup은 removed 상태를 보지만 caller의 기존 포인터 문제는 남습니다. | 5dcd882f0763/928594ec160c |
| map/backend/fd atomic lifecycle | `5dcd882f0763` | start/accept/update 되돌리기 | map 소유자를 먼저 세우고 backend 실패를 erase/disconnect로 되돌립니다. | 928594ec160c |
| 애플리케이션 send 경계 | `728aaabc4012` | bool propagation + stable value/relookup | send가 클라이언트/채널 graph를 지울 수 있음을 API에 반영합니다. | 5edcafda8a4d |
| stop-after-actor-removal | `d48e1f1f8c04` | LIST/NAMES/MODE continuation checks | completed 변경은 유지하고 later transition을 중단합니다. | aee5edebe294 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| single-thread 콜백은 포인터를 무효화하지 않음 | self-disconnect 뒤 use-after-free | 5dcd882f0763 | fd relookup after every 콜백/foreign 작업 | 928594ec160c |
| event add/update는 실패해도 local 상태와 일치 | map/backend/fd partial 소유권 | 5dcd882f0763 | map-first+erase 되돌리기, update 실패 disconnect | 928594ec160c |
| send는 local queue append | sync disconnect가 클라이언트/채널/반복자를 제거 | 728aaabc4012 | bool 전파+stable copy+relookup | 5edcafda8a4d |
| 한 번 revalidate하면 compound 명령 끝까지 안전 | 첫 response 실패 뒤 later mode 변경 | d48e1f1f8c04 | 각 출력 경계에서 즉시 return | aee5edebe294 |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| 연결 stable identity | raw 포인터/reference | fd + Server map relookup | 5dcd882f0763 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| event registration 되돌리기 | 분산 exception path | Server start/accept/refreshInterest | 5dcd882f0763 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| send 실패 signal | void/hidden lifecycle | bool through Server→IrcApplication helper | 728aaabc4012 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 채널/클라이언트 post-send 상태 | 비소유 포인터/반복자 | stable copied key + 컨테이너 relookup | 728aaabc4012 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| compound 실패 semantics | 계속 처리 가능 | partial completion + immediate stop | d48e1f1f8c04 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: 여러 event/error/stop 경로에서 연결 제거 순서가 하나의 함수로 고정되지 않았습니다.
- 마지막 커밋 `aee5edebe294` 시점의 상태: 채널은 peer 때문에 유지되고 `inviteOnly=true`, `topicProtected=false`, sender removed, peer present라는 non-transactional stop semantics가 고정됩니다.
- 개발 흐름 안에서 강화된 핵심 불변식: erase-before-콜백 baseline, map/backend/fd atomic lifecycle, 애플리케이션 send 경계, stop-after-actor-removal.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: white-box setup은 공개 IRC parser/authorization 전체를 통과하지 않으며 한 compound mode pattern만 검사합니다. 416efc91e580가 애플리케이션 수명 suite를 두 OS와 sanitizer에서 반복합니다.
- 최종 설명: `7a6bc7e1276a`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `aee5edebe294`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| server 콜백 경계 | 5dcd882f0763 | `accept/read handler` | 콜백 후 fd로 map relookup | missing이면 즉시 return |
| event registration/update | 5dcd882f0763 | `start/accept/refreshInterest` | map/backend/fd 함께 성공 | add 실패 erase 되돌리기, update 실패 disconnect |
| 애플리케이션 send | 728aaabc4012 | `sendNumeric/broadcast helper` | Server.sendTo true면 authoritative 상태 relookup | false 또는 missing 상태면 handler return |
| sync 정리 stack | 728aaabc4012 | `Connection queue/update failure` | Server::disconnect → onDisconnect → registry/채널 정리 | caller의 pre-send 포인터/반복자는 무효 |
| compound 명령 | d48e1f1f8c04 / aee5edebe294 | `MODE +it` | +i 완료 후 실패 없으면 +t | +i broadcast에서 sender 제거되면 +t 미실행 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 08-reentrant-server-and-application-cleanup.md =====

===== BEGIN FILE: 09-verification-maturation-and-portability-enforcement.md =====
# 개발 흐름 09 — 검증 체계 고도화와 이식성 강제

부제: 검증 성숙과 이식성 강제

## 1. 개발 흐름의 목표

broad live-TCP smoke에서 exact 공개 규약, injected transport/server/애플리케이션 실패 테스트, high-파일 디스크립터·slow-reader isolation, Linux/macOS·sanitizer CI로 verification architecture가 성숙하는 과정을 복원합니다.

### 원문에서 정한 의의

넓은 기능 확인 중심의 실제 TCP 테스트에서 출발해 정확한 공개 규약, 주입한 전송·이벤트 실패, 애플리케이션 수명, 많은 파일 디스크립터와 느린 수신자 압력까지 검증 범위를 넓힙니다. Linux와 macOS의 서로 다른 백엔드 및 ASan·UBSan 환경에서 같은 근거를 반복 실행하도록 CI를 구성합니다.

<details>
<summary>영문 원문</summary>

> Verification evolves from broad functional integration to exact public contracts, injected transport and event failures, application-lifetime tests, and real-process descriptor pressure. The CI commit then makes the same evidence repeatable for both readiness backends and under dynamic memory and undefined-behavior checks. No individual test proves absolute fairness or every failure class, but together they align the verification structure with the architecture's highest-risk boundaries.

</details>

원문에서 확정한 문장, 커밋 목록, SHA, 커밋 제목, 중요도, 태그와 역할은 바꾸지 않았습니다. 아래 학습 기록은 지정 브랜치의 각 SHA에서 확인한 코드 변경 내역을 기준으로 작성했습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- broad smoke와 exact 규약은 검사문 precision과 실패 localization이 어떻게 다른가?
- Connection, Server, Application 수명 테스트는 각각 어떤 실제 코드의 불변식와 injection seam을 사용하는가?
- real-프로세스 160-peer/slow-reader 테스트는 무엇을 증명하고 formal fairness·capacity는 왜 증명하지 않는가?
- Linux와 macOS matrix가 실제 epoll/kqueue backend를 같은 suite로 통과시키는가?
- ASan/UBSan job이 unit과 real-네트워크 실행 파일 모두를 같은 instrumentation으로 빌드하는가?

### 원문에서 확인되는 불변식

- 검증 구조는 부분 입출력, 이벤트 등록과 맵 소유권, 애플리케이션 재진입, 플랫폼별 백엔드처럼 설계상 위험이 큰 지점을 직접 다뤄야 합니다.
- 공개 규약 테스트는 CLI, 정확한 IRC 프레임, CRLF, 숫자 응답과 순서, 종료, 로그 순서를 명시적으로 고정해야 합니다.
- 드문 실패는 타이밍 우연이 아니라 주입한 연산·백엔드·제한값으로 결정적으로 재현해야 합니다.

### 원문에서 확인되는 구현 난점

- 드문 전송·이벤트 실패를 결정적으로 만들면서 실제 실행 경로를 우회하지 않는 테스트 지점을 설계하는 문제.
- 많은 fd와 slow receiver에서도 progress isolation을 보여 주되 벤치마크/fairness 보장을 과장하지 않는 검증 범위 설정.
- 두 event backend와 sanitizer suite를 지속적 자동화로 강제하는 문제.

## 3. 완료 기준

- 각 테스트 커밋을 실제 코드의 불변식, 경계, 검증 방식, 코드 path, proves/does-not-prove, 테스트 type, prevented 회귀로 분류했습니다.
- smoke → exact 규약 → 결정적 layer 테스트 → stress/isolation → CI의 confidence 증가를 설명합니다.
- 테스트 double과 real 소켓/프로세스를 선택한 이유를 실패 특성에 맞게 구분했습니다.
- CI matrix와 sanitizer flags가 Makefile 대상 전체에 전달되는지 확인했습니다.
- formal fairness, latency, maximum capacity, 모든 syscall 실패 class를 과장하지 않았습니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `6b4a7738a285` | `test(smoke): 실제 TCP 등록과 채널 흐름 검증` | A | VERIFICATION, INTEGRATION, RISK | 최초의 전체 실제 TCP 스모크 흐름을 추가합니다. |
| 2 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | VERIFICATION, IRC_PROTOCOL, RISK | 부분 문자열 확인 대신 CLI·프레임·종료의 정확한 규약을 검사합니다. |
| 3 | `f34ab135c546` | `test(connection): 부분 송신과 대기열 경계 검증` | A | VERIFICATION, EVENT_IO, RISK | `Connection` 실패 상태를 결정적으로 검증합니다. |
| 4 | `928594ec160c` | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` | A | VERIFICATION, LIFECYCLE, RISK | `Server` 소유권과 되돌리기를 결정적으로 검증합니다. |
| 5 | `5edcafda8a4d` | `test(app): 작은 송신 한도에서 상태 정리 검증` | A | VERIFICATION, LIFECYCLE, RISK | 애플리케이션 정리를 결정적으로 검증합니다. |
| 6 | `de1dd0fc30d0` | `test(event): 160개 연결과 느린 수신자 처리 공정성 검증` | A | VERIFICATION, EVENT_IO, RISK | 160개 클라이언트의 준비 상태 처리와 느린 수신자 격리를 검증합니다. |
| 7 | `416efc91e580` | `ci: Linux·macOS 회귀와 새니타이저 자동화` | A | BUILD, VERIFICATION, RISK | 전체 테스트를 Linux·macOS와 ASan·UBSan에서 실행합니다. |

커밋 순서는 원문에 정의된 순서를 유지했습니다.

## 5. 커밋별 학습 기록

각 기록은 해당 커밋의 변경 파일·심볼·상태와 필요할 때 직전 관련 SHA의 차이를 기준으로 작성했습니다. 후속 커밋에서 추가된 필드나 테스트 지점을 이전 커밋 설명에 소급하지 않았습니다.

### 5.1 `6b4a7738a285` — `test(smoke): 실제 TCP 등록과 채널 흐름 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, INTEGRATION, RISK |
| 원문에서 정한 역할 | 최초의 전체 실제 TCP 스모크 흐름을 추가합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 최초의 전체 실제 TCP 스모크 흐름을 추가합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | parser·registry·registration·채널 기능이 실제 프로세스/소켓 안에서 함께 동작한다는 통합 증거가 없었습니다. |
| 주요 문제와 설계 판단 | Makefile 테스트/smoke, shell runner, Python peer를 추가해 loopback server를 실행하고 다중 클라이언트 시나리오를 수행했습니다. |
| 변경 파일·symbol | `Makefile — test, smoke`<br>`tests/irc_smoke.sh`<br>`tools/irc_smoke_client.py` |
| 상태 / 소유권 변화 | 테스트 harness가 프로세스 PID, free port, temp log, peer sockets와 수신 line buffer를 소유하고 EXIT trap으로 정리합니다. |
| 실패 또는 경계 | wrong password, fragmented 입력, case-insensitive collision, 채널/mode/message/departure 오류를 실제 wire에서 관찰합니다. |
| 보장 / 비보장 | 보장: transport framing부터 애플리케이션/채널/출력 queue까지 주요 정상·일부 오류 경로가 한 프로세스에서 조합됩니다.<br>비보장: 검사문은 주로 substring 중심이어서 exact 전체 프레임/order/CRLF와 rare syscall 실패, capacity/fairness를 증명하지 않습니다. |
| 다음 관련 변화 | e5e6c57db80d가 exact 공개 규약으로 정밀도를 높이고 계층별 결정적 테스트가 후속됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `6b4a7738a285` | `tests/irc_smoke.sh` | `runner and cleanup trap` | 실제 server 프로세스 기동·준비 상태·로그·종료 관리 |
| `6b4a7738a285` | `tools/irc_smoke_client.py` | `IrcPeer and scenario` | fragmented write와 registration/채널/messaging 시나리오 |

- 조사 방법: GitHub에서 `6b4a7738a285` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | Connection framing, parser, registration, 닉네임 index, 채널 상태와 queued send가 실제 프로세스에서 결합됩니다. |
| 재현 실패 / 경계 | wrong password, fragmented PING, 닉네임 collision, JOIN/TOPIC/PRIVMSG/INVITE/MODE/KICK/PART/QUIT입니다. |
| 테스트 방식 | compiled server + real loopback TCP + shell 프로세스 harness + Python peer의 broad end-to-end smoke입니다. |
| 통과하는 실제 실행 경로 | 프로세스 startup → Server event loop → Connection framing → parser → IrcApplication → registry/채널 → 출력 queue입니다. |
| 이 테스트가 증명하는 것 | 대표 정상·오류 흐름이 실제 소켓에서 연결되어 진행됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | exact 모든 프레임, 모든 ordering, rare send/event 실패, formal capacity·fairness는 증명하지 않습니다. |
| 테스트 성격 | broad 통합 smoke |
| 막는 회귀 | packet 경계를 명령 경계로 오인하거나 identity/채널 fan-out 통합이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `6b4a7738a285`
- repository에 정의된 실행 명령: `make test` 또는 `make smoke`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`. 원문 역할은 “부분 문자열 확인 대신 CLI·프레임·종료의 정확한 규약을 검사합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, IRC_PROTOCOL, RISK |
| 원문에서 정한 역할 | 부분 문자열 확인 대신 CLI·프레임·종료의 정확한 규약을 검사합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 부분 문자열 확인 대신 CLI·프레임·종료의 정확한 규약을 검사합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | broad smoke는 주요 기능 조합을 보여 주었지만 CLI 실패, exact 프레임/CRLF/numeric order, timeout/rate/metrics/shutdown/log ordering을 정밀하게 고정하지 못했습니다. |
| 주요 문제와 설계 판단 | `tests/irc_contract.py` 중심의 executable 규약 suite를 추가해 프로세스 startup 오류, 런타임 options, exact/regex wire frames, protection responses, metrics field order, graceful shutdown 및 log order를 manifest와 검사문으로 검사했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py — CLI and wire contract checks`<br>`Makefile — contract/test integration` |
| 상태 / 소유권 변화 | 테스트가 server 프로세스, sockets, manifest, logs와 expected frames를 소유하며 동적 fd/token 같은 값만 regex로 허용합니다. |
| 실패 또는 경계 | invalid/missing option과 런타임 timeout/rate/queue/shutdown 경계를 subprocess return 코드, exact line, close 및 log event로 관찰합니다. |
| 보장 / 비보장 | 보장: 공개 CLI·wire·shutdown·observability 규약의 성공/실패 도형과 ordering을 broad substring 수준보다 정확히 고정합니다.<br>비보장: rare syscall return, event add/update 실패, 포인터 수명, formal fairness는 직접 주입하지 않습니다. |
| 다음 관련 변화 | b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level 결정적 실패를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e5e6c57db80d` | `tests/irc_contract.py` | `CLI checks and record_exact/record_regex` | 정적 프레임은 exact, 동적 값만 제한된 regex로 검증 |
| `e5e6c57db80d` | `tests/irc_contract.py` | `wire/shutdown/log scenarios` | timeout·rate·metrics·shutdown 공개 ordering |

- 조사 방법: GitHub에서 `e5e6c57db80d` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 문서화된 CLI syntax, IRC 프레임/CRLF/numeric ordering, protection response, metrics와 shutdown/log ordering이 공개 경계와 일치합니다. |
| 재현 실패 / 경계 | missing/invalid args, registration/heartbeat/rate/출력 limits, exact numerics, signal shutdown과 final ERROR입니다. |
| 테스트 방식 | compiled server subprocess + real loopback TCP + exact/limited-regex 검사문 + captured logs/manifest입니다. |
| 통과하는 실제 실행 경로 | argv parser → Server/IrcApplication 런타임 policy → wire 출력 → signal flag → shutdown/drain → logs입니다. |
| 이 테스트가 증명하는 것 | 외부 사용자가 관찰하는 성공·실패 프레임과 ordering을 정확히 고정합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 syscall/event 실패, memory safety, formal latency·fairness, 내부 수명 안전성은 증명하지 않습니다. |
| 테스트 성격 | exact 프로세스 and wire 규약 통합 |
| 막는 회귀 | CLI가 sign/whitespace를 허용하거나 numeric/CRLF/order/shutdown log 규약이 변하는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `e5e6c57db80d`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 해당 Make 대상
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`. 원문 역할은 “`Connection` 실패 상태를 결정적으로 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, EVENT_IO, RISK |
| 원문에서 정한 역할 | `Connection` 실패 상태를 결정적으로 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> `Connection` 실패 상태를 결정적으로 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection fix의 rare syscall/오버플로 브랜치는 real 소켓 timing에 맡기면 반복적으로 재현하기 어려웠습니다. |
| 주요 문제와 설계 판단 | `SendStep`과 `ScriptedSender`가 Bytes/Error/Zero 순서를 반환하는 unit 실행 파일을 추가하고 production `Connection::flushPending()`에 주입했습니다. Makefile에 `connection-test`를 연결했습니다. |
| 변경 파일·symbol | `tests/connection_test.cpp — SendStep, ScriptedSender and five test groups`<br>`Makefile — connection-test` |
| 상태 / 소유권 변화 | 스크립트 index, captured bytes와 Connection 대기 중인/offset/close 상태를 각 단계 후 동시에 검사문합니다. |
| 실패 또는 경계 | EINTR→partial→EAGAIN→partial→EWOULDBLOCK→final 순서, zero then progress, EPIPE, requested size보다 큰 return, size_t max arithmetic와 exact CRLF limit을 결정적으로 재현합니다. |
| 보장 / 비보장 | 보장: successful bytes는 `abcdefghijkl` 순서로 정확히 한 번 전달되고 retryable/terminal/invalid outcome은 아직 안 보낸 bytes를 소비하지 않으며 rejected append는 queue를 변경하지 않습니다.<br>비보장: fake sender는 kernel buffer, actual 준비 상태 notification, TCP segmentation이나 peer scheduling을 검증하지 않습니다. |
| 다음 관련 변화 | de1dd0fc30d0가 real 프로세스/slow reader에서 연결-local queue isolation을 보완하고 416efc91e580가 sanitizer로 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f34ab135c546` | `tests/connection_test.cpp` | `testLimitArithmetic/testHardLimitAndLineTerminator` | size_t max와 CRLF 포함 exact-limit 변경 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInterruptedAndPartialSendState` | scripted retry마다 대기 중인 offset과 exact 출력 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testZeroAndTerminalErrorPreservePendingBytes` | zero/EPIPE가 뒷부분을 소비하지 않음 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInvalidSendCountCannotAdvanceOffset` | oversize return이 offset을 손상시키지 않음 |

- 조사 방법: GitHub에서 `f34ab135c546` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | logical 대기 중인 limit은 오버플로하지 않고, offset은 검증된 successful byte만큼만 전진하며 unsent 뒷부분이 exact order로 보존됩니다. |
| 재현 실패 / 경계 | size_t max, CRLF exact limit, EINTR, partial, EAGAIN/EWOULDBLOCK, zero, EPIPE, impossible oversize return입니다. |
| 테스트 방식 | injected send 작업을 사용하는 결정적 scripted unit 테스트입니다. |
| 통과하는 실제 실행 경로 | queueRaw/queueLine → 오버플로-safe admission → flushPending → injected send result → offset/close/대기 중인 query입니다. |
| 이 테스트가 증명하는 것 | rare outcome 순서에서 exact byte accounting과 변경-free rejection을 직접 증명합니다. |
| 이 테스트가 증명하지 않는 것 | real kernel 준비 상태, TCP delivery acknowledgement, end-to-end fairness는 증명하지 않습니다. |
| 테스트 성격 | 결정적 transport 경계 회귀 |
| 막는 회귀 | partial retry 중 byte 중복·유실, unsigned wrap, invalid return으로 offset이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `f34ab135c546`
- repository에 정의된 실행 명령: `make connection-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`. 원문 역할은 “`Server` 소유권과 되돌리기를 결정적으로 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, LIFECYCLE, RISK |
| 원문에서 정한 역할 | `Server` 소유권과 되돌리기를 결정적으로 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> `Server` 소유권과 되돌리기를 결정적으로 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | server 수명 fix는 있었지만 add/update 실패와 콜백 self-removal을 real backend/timing에 의존하지 않고 재현하는 회귀 테스트 모음이 없었습니다. |
| 주요 문제와 설계 판단 | `FakeEventManager`에 관심 상태 map, queued events, fail-next-add/update seam을 두고 loopback ClientSocket과 `Server::pollOnce()`를 결합한 `server_lifetime_test`를 추가했습니다. |
| 변경 파일·symbol | `tests/server_lifetime_test.cpp — FakeEventManager, ClientSocket, five tests`<br>`Makefile — unit/server lifetime target` |
| 상태 / 소유권 변화 | fake backend가 등록 fd와 호출 횟수를 관찰하고 테스트가 real accepted 소켓과 Server 연결 count를 함께 검사문합니다. |
| 실패 또는 경계 | (1) 클라이언트 add 실패 되돌리기, (2) connect 콜백 self-disconnect+throw, (3) line 콜백 self-disconnect+throw, (4) write 관심 상태 update 실패, (5) outbound queue rejection close를 각각 결정적으로 주입합니다. |
| 보장 / 비보장 | 보장: 각 case 뒤 연결 map과 fake backend에 클라이언트 fd가 남지 않고 콜백 예외가 정리를 무효화하지 않으며 queue/update 실패가 false를 반환합니다.<br>비보장: fake backend는 epoll/kqueue native semantics 자체를 검증하지 않고 콜백 이후 애플리케이션 채널/클라이언트 graph도 직접 검사하지 않습니다. |
| 다음 관련 변화 | 728aaabc4012/5edcafda8a4d가 send 실패가 애플리케이션 상태와 log까지 정리되는 경로를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `registrationRollbackTest` | injected add 실패 뒤 map/backend가 모두 비어 있음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `connectCallbackLifetimeTest/lineCallbackLifetimeTest` | 콜백 self-disconnect와 throw 뒤 stale access 없음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `interestUpdateRollbackTest` | update 실패가 send false와 map/backend removal로 수렴 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `queueLimitCloseTest` | queue rejection close가 event/map entry를 남기지 않음 |

- 조사 방법: GitHub에서 `928594ec160c` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 연결 map, event registration, 파일 디스크립터 수명이 함께 전진·정리되고 콜백 뒤 stale 포인터를 재사용하지 않아야 합니다. |
| 재현 실패 / 경계 | event add/update exception, connect/line 콜백 self-disconnect+throw, outbound queue rejection입니다. |
| 테스트 방식 | injected FakeEventManager + real loopback accepted 소켓 + direct `pollOnce()` 결정적 수명 unit/통합입니다. |
| 통과하는 실제 실행 경로 | accept/read/send → Server map/backend → 콜백/refreshInterest → disconnect/erase/소멸입니다. |
| 이 테스트가 증명하는 것 | server-level 소유권 되돌리기와 콜백 relookup rule을 timing accident 없이 검증합니다. |
| 이 테스트가 증명하지 않는 것 | native epoll/kqueue kernel 동작, 애플리케이션 aggregate 정리, 모든 OS 자원 실패는 증명하지 않습니다. |
| 테스트 성격 | 결정적 server 수명 회귀 |
| 막는 회귀 | self-disconnect use-after-free와 partial event registration/관심 상태 update leak을 막습니다. |

실행 기록:

- 실행 SHA: `928594ec160c`
- repository에 정의된 실행 명령: `make unit` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`. 원문 역할은 “애플리케이션 정리를 결정적으로 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, LIFECYCLE, RISK |
| 원문에서 정한 역할 | 애플리케이션 정리를 결정적으로 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 애플리케이션 정리를 결정적으로 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 애플리케이션 revalidation fix가 welcome 응답 실패로 연결이 제거되는 실제 cross-layer stack에서 상태/log를 올바르게 정리한다는 테스트가 없었습니다. |
| 주요 문제와 설계 판단 | FakeEventManager와 captured stderr를 사용하는 `application_lifetime_test`를 추가하고 Server `maxPendingBytes=1`로 설정해 NICK/USER 등록 중 첫 welcome 프레임 admission을 강제로 실패시켰습니다. |
| 변경 파일·symbol | `tests/application_lifetime_test.cpp — CapturedStderr, FakeEventManager, registrationQueueFailureTest`<br>`Makefile — application-test` |
| 상태 / 소유권 변화 | real accepted 소켓과 애플리케이션 callbacks를 사용하되 tiny queue limit가 결정적 disconnect seam이며 테스트가 server 연결 count, running 상태와 log text를 관찰합니다. |
| 실패 또는 경계 | welcome queue 실패는 Server disconnect와 애플리케이션 onDisconnect 정리를 동기 실행합니다. |
| 보장 / 비보장 | 보장: 실패 뒤 연결 count가 0이고 server는 계속 실행되며 제거된 클라이언트를 `event=client_registered`로 기록하지 않습니다.<br>비보장: 이 최초 애플리케이션 테스트는 채널/mode의 multi-step continuation을 검사하지 않습니다. |
| 다음 관련 변화 | d48e1f1f8c04/aee5edebe294가 LIST/NAMES/MODE continuation과 부분 완료 semantics를 추가로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5edcafda8a4d` | `tests/application_lifetime_test.cpp` | `registrationQueueFailureTest` | 1-byte limit → welcome send 실패 → 연결 정리, server running, registered log 부재 |

- 조사 방법: GitHub에서 `5edcafda8a4d` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | registration response 실패로 actor가 제거되면 애플리케이션 상태와 lifecycle log가 제거 결과와 일치해야 합니다. |
| 재현 실패 / 경계 | `maxPendingBytes=1`에서 001 welcome 프레임 queue rejection입니다. |
| 테스트 방식 | tiny real Server queue limit + fake event backend + real loopback 연결 + captured stderr입니다. |
| 통과하는 실제 실행 경로 | NICK/USER → maybeRegister → sendNumeric → Server.sendTo → queue rejection/refresh/disconnect → onDisconnect 정리입니다. |
| 이 테스트가 증명하는 것 | cross-layer synchronous removal 뒤 phantom registered 상태/log가 없고 server가 프로세스-wide 실패로 멈추지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 handler, compound iteration, native backend 실패는 증명하지 않습니다. |
| 테스트 성격 | 결정적 애플리케이션 수명 회귀 |
| 막는 회귀 | welcome 실패 후 removed 클라이언트를 계속 등록 처리·로그하는 use-after-lifecycle 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `5edcafda8a4d`
- repository에 정의된 실행 명령: `make application-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `de1dd0fc30d0` — `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`. 원문 역할은 “160개 클라이언트의 준비 상태 처리와 느린 수신자 격리를 검증합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `de1dd0fc30d0` — `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | VERIFICATION, EVENT_IO, RISK |
| 원문에서 정한 역할 | 160개 클라이언트의 준비 상태 처리와 느린 수신자 격리를 검증합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 160개 클라이언트의 준비 상태 처리와 느린 수신자 격리를 검증합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 결정적 unit/수명 테스트는 rare 브랜치를 고정했지만 많은 live fd와 실제 읽지 않는 recipient 아래 다른 연결 progress를 보여 주는 프로세스-level 증거가 없었습니다. |
| 주요 문제와 설계 판단 | `irc_event_fairness.py`와 Make `event-test`를 추가했습니다. 첫 scenario는 160개 non-blocking peer가 각기 다른 PING을 보내 exact PONG을 공통 deadline 안에 받게 하고, 둘째는 receive buffer 1024인 slow peer에게 4096개의 400-byte PRIVMSG를 보내는 동안 unrelated probe의 PING/PONG이 15초 안에 진행되는지 확인했습니다. |
| 변경 파일·symbol | `tests/irc_event_fairness.py — check_many_connections, check_slow_receiver_isolation, process harness`<br>`Makefile — event-test/test integration` |
| 상태 / 소유권 변화 | Python selector가 160 peer 소켓과 expected PONG/buffer를 관리하고 separate slow/sender/probe sockets가 연결-local pressure를 만듭니다. |
| 실패 또는 경계 | 어느 peer든 PONG을 못 받거나 slow receiver 때문에 probe가 멈추거나 server가 SIGTERM 후 정상 종료하지 못하면 captured server 출력과 함께 실패합니다. |
| 보장 / 비보장 | 보장: 160 live descriptors에서 준비 상태 fan-out이 진행되고 한 unread recipient의 대기 중인 pressure가 unrelated 연결의 기본 progress를 막지 않는다는 관찰 증거를 제공합니다.<br>비보장: formal fairness, latency SLO, 최대 capacity, 모든 fd limit과 scheduler adversary를 증명하지 않으며 한 구성·한 workload의 회귀입니다. |
| 다음 관련 변화 | 416efc91e580가 이 real-프로세스 suite를 Linux epoll과 macOS kqueue 모두에서 자동 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `de1dd0fc30d0` | `tests/irc_event_fairness.py` | `check_many_connections` | 160 exact PONG을 selector와 monotonic deadline으로 수집 |
| `de1dd0fc30d0` | `tests/irc_event_fairness.py` | `check_slow_receiver_isolation` | slow recipient flood 중 unrelated probe progress |
| `de1dd0fc30d0` | `tests/irc_event_fairness.py` | `stop_server` | SIGTERM bounded normal exit 규약 |

- 조사 방법: GitHub에서 `de1dd0fc30d0` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 높은 fd 수와 연결-local backpressure에서도 unrelated 연결의 event-loop progress가 유지되어야 합니다. |
| 재현 실패 / 경계 | 160 simultaneous peers와 한 unread receiver에 대한 4096×400-byte message flood입니다. |
| 테스트 방식 | compiled real server 프로세스 + loopback sockets + Python selectors + small receive buffer + monotonic deadlines입니다. |
| 통과하는 실제 실행 경로 | many accept registrations → epoll/kqueue wait → per-연결 read/queue/write → slow 대기 중인 queue while probe PING/PONG progresses입니다. |
| 이 테스트가 증명하는 것 | 해당 workload에서 fd>select 전통 한계 수준과 slow-reader isolation을 실제 프로세스로 보여 줍니다. |
| 이 테스트가 증명하지 않는 것 | formal fairness, worst-case latency, maximum 연결 capacity, 모든 backpressure pattern은 증명하지 않습니다. |
| 테스트 성격 | real-프로세스 stress and progress-isolation 회귀 |
| 막는 회귀 | 고정 배열/fd 한계, 한 느린 recipient가 event loop 전체를 정지시키는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `de1dd0fc30d0`
- repository에 정의된 실행 명령: `make event-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 커밋: `416efc91e580` — `ci: Linux·macOS 회귀와 새니타이저 자동화`. 원문 역할은 “전체 테스트를 Linux·macOS와 ASan·UBSan에서 실행합니다.”입니다. 현재 커밋의 상태/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `416efc91e580` — `ci: Linux·macOS 회귀와 새니타이저 자동화`

| 항목 | 값 |
| --- | --- |
| 중요도 | A |
| 태그 | BUILD, VERIFICATION, RISK |
| 원문에서 정한 역할 | 전체 테스트를 Linux·macOS와 ASan·UBSan에서 실행합니다. |
| 학습 깊이 | 주요 구성 요소·경계·실패 처리·통합 point를 path와 symbol 기준으로 복원합니다. |

#### 원문에서 정한 역할

> 전체 테스트를 Linux·macOS와 ASan·UBSan에서 실행합니다.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 모든 테스트 대상이 로컬에서 실행 가능했지만 epoll/kqueue 양쪽과 sanitizer instrumentation 아래 반복된다는 repository-level enforcement가 없었습니다. |
| 주요 문제와 설계 판단 | GitHub Actions workflow를 추가해 `ubuntu-latest`와 `macos-latest` matrix에서 `make -j2`와 `make test`를 실행하고, 별도 Ubuntu job에서 ASan+UBSan·프레임 포인터·leak detection·halt-on-error flags로 동일 실행 파일/테스트를 빌드/run하도록 했습니다. |
| 변경 파일·symbol | `.github/workflows/ci.yml — platform-regression and linux-sanitizers jobs` |
| 상태 / 소유권 변화 | matrix OS가 default EventManager 팩터리를 통해 각각 epoll/kqueue backend를 선택하고 sanitizer CXXFLAGS가 production 실행 파일과 unit/네트워크 테스트 builds에 동일하게 전달됩니다. |
| 실패 또는 경계 | 어느 OS 빌드·테스트 또는 sanitizer report라도 job을 실패시키며 matrix는 fail-fast=false로 다른 platform 결과도 수집합니다. |
| 보장 / 비보장 | 보장: complete Make 테스트 graph를 Linux/macOS 및 Linux ASan/UBSan에서 push/PR마다 반복하도록 자동화합니다.<br>비보장: CI runner가 지원하지 않는 OS/compiler, TSan, exhaustive race/fuzz/formal verification은 포함하지 않습니다. workflow 정의의 존재는 현재 run 성공 이력 자체와 동일하지 않습니다. |
| 다음 관련 변화 | 개발 흐름 09의 broad→exact→injected→stress 근거를 두 준비 상태 backend와 동적 checks에 연결하는 최종 enforcement입니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `416efc91e580` | `.github/workflows/ci.yml` | `platform-regression matrix` | ubuntu/macos에서 빌드와 complete make 테스트 실행 |
| `416efc91e580` | `.github/workflows/ci.yml` | `linux-sanitizers job` | ASan+UBSan/leak options를 빌드와 테스트 전체에 전달 |

- 조사 방법: GitHub에서 `416efc91e580` 커밋의 변경 내역과 변경 파일을 확인했습니다.
- 런타임 결과로 표시한 내용은 없으며 위 표는 코드 검토으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 동일 회귀 graph가 Linux epoll, macOS kqueue, ASan/UBSan instrumentation에서 반복 가능해야 합니다. |
| 재현 실패 / 경계 | cross-platform compilation/warnings, complete unit/네트워크 테스트, address/undefined/leak failures입니다. |
| 테스트 방식 | GitHub Actions OS matrix와 separate sanitizer CI job입니다. |
| 통과하는 실제 실행 경로 | 체크아웃 → make production/테스트 → make 테스트 → backend 팩터리/네트워크 프로세스/unit binaries입니다. |
| 이 테스트가 증명하는 것 | workflow가 두 OS와 sanitizer에서 같은 suite를 실행하도록 구성되었음을 코드로 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 이 작업 환경에서 실제 CI run이 성공했다는 사실, TSan/Windows/BSD와 모든 UB class는 증명하지 않습니다. |
| 테스트 성격 | continuous cross-platform and 동적-analysis enforcement |
| 막는 회귀 | 한 backend에서만 컴파일/테스트되거나 memory/UB defect가 일반 빌드에서 숨는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `416efc91e580`
- repository에 정의된 실행 명령: workflow의 `make -j2`, `make test`, sanitizer CXXFLAGS 빌드·테스트
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 브랜치의 local 체크아웃을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/테스트 diff와 테스트 mechanism은 확인했지만, 실행 파일/테스트 명령의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 커밋은 이 개발 흐름의 마지막 항목입니다. 아래 Invariant ledger와 개발 흐름 최종 상태에서 전체 변화를 연결합니다.

## 6. 불변식 변화 기록

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| broad live 통합 | `6b4a7738a285` | real 프로세스/TCP smoke | 여러 구성 요소의 조합을 넓게 확인하지만 검사문 precision은 낮습니다. | e5e6c57db80d |
| exact 공개 규약 | `e5e6c57db80d` | CLI/wire/shutdown/log exact 검사문 | 외부 동작과 ordering을 정밀하게 고정합니다. | low-level rare 실패는 별도 seam 필요 |
| 결정적 layer failures | `f34ab135c546` | 928594ec160c, 5edcafda8a4d | send 작업, event backend, queue limit로 rare 브랜치를 재현합니다. | 각 architecture risk와 직접 대응 |
| real-프로세스 pressure isolation | `de1dd0fc30d0` | 160 peers + slow receiver | 실제 파일 디스크립터 수와 backpressure에서 progress 근거를 추가합니다. | formal fairness/capacity는 아님 |
| cross-platform/동적 enforcement | `416efc91e580` | Linux/macOS matrix + ASan/UBSan | complete make 테스트 graph를 두 backend와 sanitizer에 반복합니다. | 현재 run 성공 자체는 workflow inspection과 구분 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. 문제 → 수정 → 검증 연결

| 기존 가정 | 실제 실패 / risk | Fix 또는 기반 변화 | 수정된 결정 / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| broad substring smoke만 충분 | exact 프레임/order와 rare 브랜치가 숨음 | e5e6c57db80d + layer 테스트 | 규약과 결정적 injection 분리 | f34ab/928/5ed |
| real 소켓에서 rare syscall/event 실패를 기다림 | timing-dependent·불안정 재현 | SendOperation/FakeEventManager/tiny queue | 실제 실행 경로에 controlled 실패 주입 | f34ab/928/5ed |
| 단일 OS local success면 portable | epoll/kqueue 또는 sanitizer에서만 나타나는 결함 | 416efc91e580 | OS matrix와 same-suite instrumentation | CI jobs |

## 8. 소유권·상태·담당 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 커밋 | 확인 결과 |
| --- | --- | --- | --- | --- |
| 프로세스 lifecycle 테스트 준비 코드 | 없음 | shell/Python smoke harness | 6b4a7738a285 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 공개 규약 oracle | substring expectation | exact/regex manifest | e5e6c57db80d | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| transport 실패 스크립트 | kernel timing | ScriptedSender | f34ab135c546 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| server backend 실패 control | native backend | FakeEventManager | 928594ec160c | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| 애플리케이션 실패 trigger | 우연한 send pressure | maxPendingBytes=1 + captured log | 5edcafda8a4d | 경로/심볼은 위 커밋 증거표에 연결했습니다. |
| portable enforcement | developer local 환경 | GitHub Actions matrix/sanitizer | 416efc91e580 | 경로/심볼은 위 커밋 증거표에 연결했습니다. |

## 9. 개발 흐름의 최종 상태

- 시작 직전 상태: parser·registry·registration·채널 기능이 실제 프로세스/소켓 안에서 함께 동작한다는 통합 증거가 없었습니다.
- 마지막 커밋 `416efc91e580` 시점의 상태: complete Make 테스트 graph를 Linux/macOS 및 Linux ASan/UBSan에서 push/PR마다 반복하도록 자동화합니다.
- 개발 흐름 안에서 강화된 핵심 불변식: broad live 통합, exact 공개 규약, 결정적 layer failures, real-프로세스 pressure isolation, cross-platform/동적 enforcement.
- 남은 한계 또는 후속 개발 흐름에서 보강되는 부분: CI runner가 지원하지 않는 OS/compiler, TSan, exhaustive race/fuzz/formal verification은 포함하지 않습니다. workflow 정의의 존재는 현재 run 성공 이력 자체와 동일하지 않습니다. 개발 흐름 09의 broad→exact→injected→stress 근거를 두 준비 상태 backend와 동적 checks에 연결하는 최종 enforcement입니다.
- 최종 설명: `6b4a7738a285`에서 시작한 책임은 커밋 map 순서대로 상태 소유자, 실패 브랜치와 정리 ordering을 추가했고 `416efc91e580`에서 이 개발 흐름이 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/테스트 SHA를 별도로 연결했으며 최종 HEAD 상태를 과거 커밋에 소급하지 않았습니다.

## 10. 최종 설계와 실행 흐름

| 단계 | SHA | Caller / callee / 상태 소유자 | 정상 transition | 실패 / 정리 transition |
| --- | --- | --- | --- | --- |
| broad 통합 | 6b4a7738a285 | `make smoke/test` | real server + peers + substring 검사문 | 실패 시 프로세스/log 정리 |
| exact 규약 | e5e6c57db80d | `irc_contract.py` | exact 프레임/CLI/shutdown/log 검사문 | 동적 value만 limited regex |
| 결정적 units | f34ab135c546 / 928594ec160c / 5edcafda8a4d | `injected sender/backend/limit` | production Connection/Server/App paths | rare 브랜치를 fixed outcome으로 재현 |
| stress isolation | de1dd0fc30d0 | `irc_event_fairness.py` | 160 fd exact PONG + slow-reader/probe | deadline miss 또는 abnormal exit는 실패 |
| CI enforcement | 416efc91e580 | `GitHub Actions` | ubuntu/macos make 테스트 + Linux ASan/UBSan | 한 job 실패가 회귀 signal |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA, subject, 중요도, tags를 원본과 대조했습니다.
- [x] 모든 커밋의 해당 SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] 최종 HEAD의 함수나 field를 과거 커밋 설명에 소급하지 않았습니다.
- [x] S 커밋은 architecture/불변식, 실패, 소유권/lifecycle, 후속 fix/테스트까지 기록했습니다.
- [x] A 커밋은 주요 구성 요소/경계/실패 처리와 설계 판단을 기록했습니다.
- [x] B 커밋은 개발 흐름에서 맡는 구현 역할과 상태 변화를 기록했습니다.
- [x] fix 커밋의 기존 가정, root cause, 수정 불변식, 회귀 테스트를 연결했습니다.
- [x] 테스트 커밋의 실제 실행 경로, 검증 방식, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 경로/심볼 증거에 연결했습니다.
- [ ] production 빌드·테스트 명령를 이 작업 환경에서 직접 실행했습니다. local 체크아웃을 만들 수 없어 코드/테스트 inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 09-verification-maturation-and-portability-enforcement.md =====

===== BEGIN FILE: README.md =====
# ft_irc 개발 흐름 학습 안내

## 목적

이 문서 세트는 완성형 프로젝트 해설서가 아닙니다. 학습자가 실제 커밋 history와 각 SHA 시점의 코드를 직접 읽고 증거를 기록하면서 설계, 구현, 실패 처리, 수정, 검증의 발전 과정을 복원하기 위한 골격입니다.

문서 구조와 커밋 관계는 `commit-importance.md`의 개발 흐름을 따르며, 커밋의 구현 의도와 실패 handling 확인 항목은 `commit-bodies.md`를 기준으로 작성되었습니다.

## 권장 학습 순서

| 순서 | 개발 흐름 문서 | 학습 초점 |
| --- | --- | --- |
| 1 | [Portable 준비 상태 and non-blocking transport](01-portable-readiness-and-nonblocking-transport.md) | 이벤트 준비 상태와 논블로킹 전송 |
| 2 | [Protocol 경계, identity, and registration](02-protocol-boundary-identity-and-registration.md) | 프로토콜 경계, 식별자와 등록 |
| 3 | [Channel authority, fan-out, and 정리](03-channel-authority-fanout-and-cleanup.md) | 채널 권한, 팬아웃과 정리 |
| 4 | [Operational protections and controlled shutdown](04-operational-protections-and-controlled-shutdown.md) | 운영 보호와 제어된 종료 |
| 5 | [Strict 런타임 configuration boundaries](05-strict-runtime-configuration-boundaries.md) | 엄격한 런타임 구성 경계 |
| 6 | [Heartbeat liveness correctness](06-heartbeat-liveness-correctness.md) | 하트비트 생존성 정확성 |
| 7 | [Output-queue correctness under partial 실패](07-output-queue-correctness-under-partial-failure.md) | 부분 실패에서의 송신 대기열 정확성 |
| 8 | [Reentrant server and 애플리케이션 정리](08-reentrant-server-and-application-cleanup.md) | 재진입 가능한 서버와 애플리케이션 정리 |
| 9 | [Verification maturation and portability enforcement](09-verification-maturation-and-portability-enforcement.md) | 검증 성숙과 이식성 강제 |

위 순서는 원본에 정의된 개발 흐름 순서입니다. 같은 커밋이 여러 문서에 등장해도 제거하지 않습니다. 각 문서에서 서로 다른 불변식과 학습 관점으로 다시 확인합니다.

## 개발 흐름 문서 사용법

1. 문서의 커밋 목록에서 현재 확인할 SHA와 중요도를 확인합니다.
2. repository를 해당 SHA로 이동한 뒤 그 시점의 변경 파일과 실제 symbol을 찾습니다.
3. 문서에 미리 적힌 원문에서 확정한 role과 구현 anchor를 기준으로 코드 증거를 수집합니다.
4. 학습 기록란에는 path, symbol, 호출자/피호출자, 상태 변경 순서, 실패 브랜치, 정리 경로를 직접 채웁니다.
5. fix 커밋은 기존 가정 → 실제 위험 → root cause → 수정 불변식 → 수정 코드 → 회귀 테스트 순서로 연결합니다.
6. 테스트 커밋은 실제 코드의 불변식, 재현 경계, 검증 방식, 통과하는 실제 실행 경로, 증명/비증명 범위를 분리합니다.
7. 개발 흐름 마지막에 불변식 ledger, 실패-fix-테스트, responsibility 변화, execution flow를 자신의 코드 증거로 완성합니다.

## 해당 SHA 코드 확인 원칙

- 반드시 현재 학습 중인 커밋 SHA의 tree를 확인합니다.
- 기본 비교는 `<SHA>^`와 `<SHA>`입니다. 개발 흐름에서 직전 관련 SHA가 따로 제시되면 두 시점도 함께 비교합니다.
- 최종 HEAD의 코드를 과거 커밋 설명에 소급 적용하지 않습니다.
- 후속 fix에서 생긴 함수, field, 테스트 seam을 이전 커밋에 존재했던 것처럼 기록하지 않습니다.
- 원문이 확정하지 않은 불변식을 새 사실처럼 추가하지 않습니다. 코드에서 직접 확인한 해석은 path와 symbol 증거를 붙여 학습자 결론으로 구분합니다.
- 커밋 subject, SHA, 중요도, tags, 개발 흐름 순서는 변경하지 않습니다.

권장 확인 명령 예시는 다음과 같습니다. 실제 repository 상태와 작업 방식에 맞게 사용하되, 확인 대상 SHA는 바꾸지 않습니다.

```sh
git switch --detach <SHA>
git show --stat --oneline <SHA>
git diff <SHA>^ <SHA> -- <path>
git show <SHA>:<path>
```

## S/A/B/C별 학습 깊이

| 중요도 | 학습 깊이 |
| --- | --- |
| S | 프로젝트 핵심 architecture/불변식으로 취급합니다. 직전 상태, 실패 가능성, 핵심 결정, 실제 코드, 소유권/lifecycle/상태 변화, 후속 fix/테스트까지 깊게 기록합니다. |
| A | 주요 구성 요소·경계·실패 처리·통합 point를 설명할 수 있도록 핵심 코드와 설계 판단을 기록합니다. |
| B | thread 흐름에서 맡는 구현 역할과 필요한 코드·상태 변화를 확인합니다. S/A와 같은 분량을 강제하지 않습니다. |
| C | thread 이해에 필요한 맥락만 기록합니다. 이 프로젝트의 개발 흐름에는 C 커밋이 포함되지 않습니다. |

## 실제 코드 삽입 기준

- 문서에는 해당 SHA에서 직접 확인한 최소 코드 조각만 삽입합니다.
- 코드 조각마다 SHA, file path, symbol 또는 함수 범위를 함께 적습니다.
- 구현 전체를 복사하지 않고 불변식, ordering, 소유권 transfer, 실패 브랜치를 증명하는 부분만 남깁니다.
- 변경 전/후 비교가 핵심이면 parent와 current SHA의 대응 조각을 나란히 기록합니다.
- line number는 tree나 포매터에 따라 변할 수 있으므로 path와 symbol을 기본 식별자로 사용합니다.
- 실제 코드를 확인하지 못한 항목은 추측으로 채우지 않고 “확인하지 못한 경로/심볼과 이유”를 기록합니다.

## 테스트 커밋 학습 방법

각 테스트 커밋에서 다음을 분리해 기록합니다.

- 대상 실제 코드의 불변식
- 재현하는 실패 또는 경계
- 테스트 방식: real 프로세스/소켓, fake backend, injected 작업, white-box setup, sanitizer 등
- 실제 통과하는 production 코드 path
- 테스트가 증명하는 것
- 테스트가 증명하지 않는 것
- broad 통합인지 결정적 회귀 테스트인지
- 후속 변경에서 막는 회귀
- 실행 환경, 명령, 결과와 실패 transcript/log

테스트가 통과했다는 사실만 적지 않습니다. 어떤 production 브랜치를 어떤 입력과 seam으로 통과시켰는지 설명할 수 있어야 완료입니다.

## 문서 완료 기준

- 모든 개발 흐름의 커밋을 원문 순서대로 확인했습니다.
- 모든 기록이 해당 SHA 코드에 근거하며 최종 HEAD를 소급 사용하지 않았습니다.
- S/A/B/C 중요도에 맞게 학습 깊이를 구분했습니다.
- 각 중요한 커밋에서 실제 path, symbol, 상태, 호출자/피호출자, 실패/정리 근거를 남겼습니다.
- fix와 회귀 테스트를 하나의 원인-수정-검증 흐름으로 연결했습니다.
- 불변식 ledger에 도입, 강화, 부족함 발견, fix, 회귀 테스트를 구분했습니다.
- 개발 흐름 최종 architecture 또는 execution flow를 자신의 코드 증거로 설명할 수 있습니다.
- 별도 프로젝트 재학습 없이 커밋 history에 근거해 설계 → 구현 → 실패 → 수정 → 검증 과정을 다시 설명할 수 있습니다.
===== END FILE: README.md =====

