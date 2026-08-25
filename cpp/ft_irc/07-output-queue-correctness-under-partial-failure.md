# Thread 07. 부분 실패에서의 송신 대기열 정확성

## 핵심 불변 조건

non-blocking socket의 output queue는 “보낼 문자열 목록”이 아니라 이미 전송한 prefix와 아직 전송하지 못한 suffix를 구분하는 상태 machine입니다. capacity check 역시 현재 storage 크기가 아니라 남은 pending byte를 기준으로 해야 합니다.

이 Thread가 최종적으로 보장하는 식은 다음과 같습니다.

```text
pending = writeBuffer.size - writeOffset
0 <= pending <= maxPendingBytes

성공한 send(count)에서만:
writeOffset' = writeOffset + count,  0 < count <= pending

EINTR / EAGAIN / EWOULDBLOCK / 0 / hard error에서:
전송 성공 byte가 없으면 offset은 변하지 않음
```

capacity 판단은 overflow 가능한 `pending + append <= limit` 대신 다음 조건을 사용합니다.

```text
pending <= limit && append <= limit - pending
```

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a10fe961e2b1` | `feat(connection): 부분 송신 대기열 처리` | S | `CORE, EVENT_IO, LIFECYCLE` | buffer+offset으로 unsent suffix를 표현하고 non-blocking send 결과 처리 |
| 2 | `d7d85e518177` | `feat(buffer): 송신 대기열 크기 제한` | A | `EVENT_IO, RESILIENCE, RISK` | 연결별 hard cap과 enqueue 실패 반환 도입 |
| 3 | `881e59734a9a` | `fix(connection): 송신 대기열 계산과 재시도 상태 보호` | S | `DEBUG, EVENT_IO, RISK` | overflow-safe capacity와 비정상 send 결과 방어, injectable send seam 도입 |
| 4 | `f34ab135c546` | `test(connection): 부분 송신과 대기열 경계 검증` | A | `VERIFICATION, EVENT_IO, RISK` | scripted syscall로 partial/retry/error/limit 상태를 직접 검증 |

## 1. `a10fe961e2b1` — queue를 “남은 suffix”로 정의하다

`Connection`은 `writeBuffer_`와 `writeOffset_`을 함께 보관합니다.

```text
writeBuffer_ = "abcdefghijkl"
writeOffset_ = 5
pending      = "fghijkl" (7 bytes)
```

`flushPending`은 매 반복마다 offset 이후 pointer와 남은 size를 syscall에 전달합니다.

```cpp
const char* data = writeBuffer_.data() + writeOffset_;
const std::size_t size = writeBuffer_.size() - writeOffset_;
const ssize_t count = ::send(fd_, data, size, sendFlags());
```

결과 처리는 다음 의미를 갖습니다.

| 결과 | offset | queue 상태 | 반환 의미 |
| --- | --- | --- | --- |
| `count > 0` | 정확히 `count` 증가 | 일부 또는 전체 소비 | loop 계속 |
| `count == 0` | 변화 없음 | 그대로 보존 | 현재 cycle은 block된 것으로 종료 |
| `-1, EINTR` | 변화 없음 | 그대로 보존 | 같은 suffix 즉시 재시도 |
| `-1, EAGAIN/EWOULDBLOCK` | 변화 없음 | 그대로 보존 | 다음 Write-ready까지 대기 |
| 기타 오류 | 변화 없음 | close 전까지 남아 있음 | error/close request |

전부 전송하면 buffer와 offset을 모두 초기화합니다. 일부 prefix가 오래 남아 storage를 차지할 때는 일정 크기 이후 unsent suffix를 앞으로 compact하고 offset을 0으로 되돌립니다. compact는 logical pending bytes를 바꾸지 않습니다.

`queueLine`은 전달된 line 끝의 CR/LF를 제거한 뒤 정확히 하나의 `\r\n`을 붙입니다. 따라서 caller가 terminator를 포함했는지와 무관하게 wire framing이 한 번만 적용됩니다.

### 이 시점의 보장과 빈틈

보장:

- partial send 뒤 이미 보낸 prefix를 다시 보내지 않습니다.
- `EINTR`과 would-block이 byte consumption으로 오해되지 않습니다.
- pending output이 남으면 Server가 Write interest를 유지할 수 있습니다.

아직 test seam이 없으므로 “첫 send는 2 byte, 다음은 EAGAIN” 같은 syscall sequence를 결정적으로 재현하기 어렵습니다. 또한 후속 cap과 결합했을 때 정수 overflow 및 비정상 반환값 방어가 없습니다.

## 2. `d7d85e518177` — hard cap을 append 전 검사하다

느린 client가 output을 읽지 않으면 계속되는 fan-out이 connection의 memory를 무한히 늘릴 수 있습니다. 이 commit은 `maxPendingBytes_`를 추가하고 enqueue API를 `void`에서 `bool`로 바꿉니다.

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

거절된 byte는 buffer에 일부도 추가되지 않습니다. 기존 pending data는 그대로 유지되고 close request만 설정됩니다. Server는 반환값을 application에 전달하고, interest를 갱신해 기존 queue를 drain하거나 비어 있으면 연결을 정리합니다.

line의 cap에는 normalized payload와 새 CRLF 2 byte가 포함됩니다. 예를 들어 limit 5의 빈 queue에 `"abc\n"`를 넣으면 실제 저장될 `"abc\r\n"` 5 byte가 exact limit을 채웁니다.

### 최초 capacity 식의 root cause

초기 helper는 다음과 같습니다.

```cpp
return pendingBytes() + byteCount <= maxPendingBytes_;
```

`std::size_t` 덧셈은 modulo arithmetic입니다. `pending + byteCount`가 `SIZE_MAX`를 넘으면 작은 수로 wrap하여 오히려 limit 이하로 보일 수 있습니다.

```text
pending = 1
byteCount = SIZE_MAX
pending + byteCount = 0  (wrap)
0 <= SIZE_MAX           → 잘못된 허용
```

실제 `std::string::append`가 그 크기를 할당하기 전에 예외를 던질 가능성이 크더라도, queue contract 자체가 잘못된 판단을 한다는 사실은 변하지 않습니다.

## 3. `881e59734a9a` — 산술과 syscall 결과를 모두 방어하다

이 S급 fix는 서로 연결된 두 문제를 수정하고, 둘을 deterministic하게 시험할 seam을 함께 만듭니다.

### overflow-safe capacity predicate

`ConnectionLimits.hpp`에 작은 pure helper가 추가됩니다.

```cpp
inline bool canAppendPending(std::size_t pending,
                             std::size_t byteCount,
                             std::size_t limit) noexcept {
    return pending <= limit && byteCount <= limit - pending;
}
```

첫 조건이 거짓이면 subtraction을 수행하지 않는 short-circuit 덕분에 `limit - pending` underflow도 없습니다. 둘째 비교는 addition 없이 append 가능한 여유를 판단합니다.

`queueLine`은 payload `end`와 terminator 2를 단계적으로 검사합니다.

```cpp
const std::size_t pending = pendingBytes();
if (!canAppendPending(pending, end, maxPendingBytes_) ||
    !canAppendPending(pending + end, 2, maxPendingBytes_)) {
    requestClose("outbound queue limit exceeded");
    return false;
}
```

첫 검사가 성공했기 때문에 `pending + end <= limit <= SIZE_MAX`이고, 두 번째 식의 `pending + end`는 안전합니다. 이렇게 하면 `end + 2` 자체도 overflow하지 않습니다.

### `send`를 주입 가능한 operation으로 분리

`Connection` constructor가 optional `SendOperation`을 받습니다.

```cpp
using SendOperation =
    std::function<ssize_t(int, const void*, std::size_t, int)>;
```

production에서는 default wrapper가 `::send`를 호출하고, test는 scripted callable을 넣습니다. move construction/assignment도 이 operation을 함께 이동합니다. seam은 queue owner를 바꾸지 않고 syscall result만 통제합니다.

### 요청보다 큰 양수 반환을 거부

정상적인 `send`는 요청한 size보다 큰 수를 반환할 수 없습니다. 하지만 injected seam, wrapper bug, 잘못된 platform adapter가 그런 값을 돌려도 offset을 넘어가게 두면 다음 `pendingBytes()`가 underflow합니다.

```cpp
if (count > 0) {
    const std::size_t sent = static_cast<std::size_t>(count);
    if (sent > size) {
        result.hasError = true;
        result.error = "send returned more bytes than requested";
        requestClose(result.error);
        break;
    }
    writeOffset_ += sent;
    continue;
}
```

검사 실패에서는 offset을 전혀 바꾸지 않습니다. 이 defensive branch는 kernel behavior를 의심한다기보다, state transition의 precondition을 코드에 명시합니다.

## 4. `f34ab135c546` — syscall sequence를 상태 machine으로 검증하다

`tests/connection_test.cpp`는 실제 network timing 대신 `ScriptedSender`에 반환 sequence를 넣습니다. 각 step은 양수 byte 수, errno error, 0 중 하나입니다.

### maximum arithmetic

```cpp
const std::size_t maximum = std::numeric_limits<std::size_t>::max();

canAppendPending(0, maximum, maximum)       == true
canAppendPending(1, maximum, maximum)       == false
canAppendPending(maximum, 1, maximum)       == false
canAppendPending(maximum, 0, maximum - 1)   == false
```

이 사례들은 실제 거대한 string을 할당하지 않고 predicate의 boundary만 직접 검사합니다.

### exact line limit과 rejection atomicity

limit 5에서 `queueLine("abc\n")`는 `abc\r\n` 5 byte를 수락합니다. 뒤의 `queueRaw("x")`는 거절되고 다음을 확인합니다.

- close request가 설정됨
- pending byte는 5 그대로
- 거절된 `x`가 buffer에 섞이지 않음
- 기존 exact-limit queue는 여전히 정상 flush됨

### EINTR → partial → EAGAIN → append → partial → EWOULDBLOCK → complete

script는 다음 순서입니다.

```text
EINTR
2 bytes
EAGAIN
3 bytes
EWOULDBLOCK
7 bytes
```

초기 payload는 `abcdef`입니다.

1. EINTR은 아무것도 소비하지 않음
2. 2 byte 성공 후 `ab`만 기록, pending `cdef`
3. EAGAIN에서 반환, pending 4
4. 새 `ghijkl` 6 byte append → pending exact 10
5. 한 byte 추가는 cap 때문에 거절, pending 10 유지
6. 다음 flush에서 3 byte 성공 후 EWOULDBLOCK → pending 7
7. 마지막 flush에서 7 byte 성공 → queue empty

최종 sink byte가 정확히 `abcdefghijkl`인지 비교해 duplication과 loss를 동시에 잡습니다.

### 0-byte send와 terminal error

0 반환은 queue를 소비하지 않고 would-block으로 취급하며 다음 호출에서 같은 3 byte가 전송됩니다. `EPIPE`는 error와 close request를 만들지만 pending bytes는 그대로 남습니다. “실패했으니 보낸 것으로 간주”하지 않는지 확인하는 사례입니다.

### impossible positive count

3 byte를 요청했는데 scripted sender가 4를 반환합니다. test는 error/close request, pending 3 유지, offset 미변경을 요구합니다.

## 최종 transition 표

| 사건 | capacity/offset 변화 | connection 상태 |
| --- | --- | --- |
| append가 cap 안 | pending 증가 | 정상 |
| append가 cap 초과 | 변화 없음 | close requested |
| partial send `n` | offset `+n`, pending `-n` | 계속 write |
| complete send | buffer clear, offset 0 | Write interest 제거 가능 |
| EINTR | 변화 없음 | 같은 cycle 재시도 |
| EAGAIN/EWOULDBLOCK | 변화 없음 | 다음 readiness 대기 |
| send 0 | 변화 없음 | 다음 readiness 대기 |
| hard error | 변화 없음 | close requested |
| `count > requested` | 변화 없음 | invariant violation error, close requested |

## 이 Thread의 경계

- queue failure가 `Server::refreshInterest`에서 즉시 disconnect를 일으키고 callback stack으로 재진입하는 문제는 Thread 08입니다.
- 한 느린 수신자가 다른 159개 연결의 progress를 막지 않는다는 network-level evidence는 Thread 09의 event fairness test입니다.
- cap 값 자체를 CLI에서 overflow 없이 읽는 문제는 Thread 05입니다.

> 검사 범위: exact production diff와 scripted unit test source를 확인했습니다. `tests/connection_test` binary를 실제로 build/run하지 않았습니다.
