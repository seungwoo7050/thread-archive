# Thread: stdout 성공이 protocol ACK의 commit 조건이 되기까지

Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 04

## 개요

server가 byte를 decoder state에 반영했다는 사실과 그 byte가 stdout에 보였다는 사실은 같지 않다. 이전 구현처럼 `write`의 반환값을 무시한 채 ACK를 보내면 client는 실제로 출력되지 않은 data를 성공으로 확정하고 다음 bit로 넘어간다.

이 Thread는 output을 protocol state transition의 외부 부수 효과가 아니라 **ACK보다 먼저 성공해야 하는 commit 단계**로 바꾸는 과정을 다룬다. payload byte, terminating newline, abandoned session을 닫는 recovery newline, startup PID line이 모두 all-or-failure write를 사용한다.

### 최종 불변 조건

- `mt_write_all`의 성공은 요청한 모든 byte가 descriptor에 전달됐음을 뜻한다.
- positive short write는 progress이며 남은 suffix를 계속 쓴다.
- `EINTR`은 같은 위치에서 재시도하고, zero progress는 `EIO`로 실패한다.
- 완성 byte 또는 delimiter write가 실패하면 그 bit의 ACK를 보내지 않는다.
- output failure 뒤 server는 decoder를 계속 사용하지 않고 event loop를 중단한다.
- dead-owner recovery newline이 실패하면 stale state를 지우거나 replacement에 READY를 보내지 않는다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `826dd34c378f` | fix(io): 중단·부분 쓰기를 끝까지 처리 | A | `OUTPUT_COMMIT, PRACTICAL, RISK` | `mt_write_all`이 EINTR를 retry하고 short write만큼 offset을 전진하며 zero progress를 `EIO`로 처리합니다. |
| 2 | `db2004556d8b` | fix(server): stdout 실패 뒤 ACK 전송 차단 | S | `CORE, OUTPUT_COMMIT, RISK` | PID, payload, terminator newline, recovery newline을 all-or-failure로 쓰고 failure 시 triggering ACK를 보내지 않습니다. |
| 3 | `9aa80e047514` | test(server): 부분 쓰기와 출력 실패 검증 | A | `TEST, OUTPUT_COMMIT, RISK` | low-level write를 deterministic test implementation으로 바꿔 EINTR, one-byte short write, zero write, selected payload/newline EPIPE를 주입합니다. |
| 4 | `081a882d7fa3` | test(server): 회수 줄바꿈 출력 실패 검증 | A | `TEST, OUTPUT_COMMIT, SESSION` | dead owner가 visible partial line을 남긴 상태에서 replacement acquisition이 recovery newline을 쓰는 순간 failure를 주입합니다. |

## `826dd34c378f` — fix(io): 중단·부분 쓰기를 끝까지 처리

**중요도** `A` · **태그** `OUTPUT_COMMIT, PRACTICAL, RISK`

### single `write`에서 completion loop로

`write(fd, buffer, size)`는 성공 시에도 `size`보다 작은 양수를 반환할 수 있고, signal에 의해 `EINTR`로 중단될 수 있다. 새 helper는 이미 전송한 prefix와 아직 남은 suffix를 명시적으로 추적한다.

```c
int mt_write_all(int fd, const void *buffer, size_t size)
{
    const unsigned char *bytes;
    size_t              offset;
    ssize_t             written;

    bytes = (const unsigned char *)buffer;
    offset = 0;
    while (offset < size)
    {
        written = write(fd, bytes + offset, size - offset);
        if (written == -1 && errno == EINTR)
            continue ;
        if (written == -1)
            return (-1);
        if (written == 0)
        {
            errno = EIO;
            return (-1);
        }
        offset += (size_t)written;
    }
    return (0);
}
```

loop invariant는 `buffer[0..offset)`이 이미 전달됐고, 다음 호출은 `buffer[offset..size)`만 요청한다는 것이다. `size == 0`이면 write를 호출하지 않고 성공한다. 남은 byte가 있는데 `write`가 0을 반환하면 progress 없이 무한 반복할 수 있으므로 `EIO`로 바꾼다.

`mt_putstr_fd`와 `mt_putnbr_fd`도 내부 write를 이 helper로 바꾼다. 다만 두 함수의 반환형은 여전히 `void`이므로 caller가 diagnostic 출력 실패를 관찰하는 API로 바뀐 것은 아니다. protocol-critical output은 다음 commit에서 `mt_write_all`의 반환값을 직접 사용한다.

## `db2004556d8b` — fix(server): stdout 실패 뒤 ACK 전송 차단

**중요도** `S` · **태그** `CORE, OUTPUT_COMMIT, RISK`

### 기존 실패 순서

self-pipe 전환 직후의 `process_bit`은 8번째 bit에서 `flush_byte`를 호출했지만, `flush_byte`는 `write` 결과를 무시했다. 이후 decoder를 reset하고 sequence를 증가시켜 ACK를 전송했다.

```text
state에 8번째 bit 반영
  → write 시도(성공 여부 무시)
  → decoder reset
  → sequence 증가
  → ACK 전송
```

stdout이 닫혀 있거나 부분 쓰기 뒤 실패해도 client는 ACK를 받아 “이 byte가 commit됐다”고 판단할 수 있었다.

### output-before-ACK 순서

`reset_session`과 `flush_byte`가 `int`를 반환하도록 바뀐다.

```c
static int flush_byte(unsigned char output)
{
    if (output == '\0')
    {
        if (mt_write_all(STDOUT_FILENO, "\n", 1) == -1)
            return (-1);
        return (reset_session(0));
    }
    if (mt_write_all(STDOUT_FILENO, &output, 1) == -1)
        return (-1);
    g_line_started = 1;
    return (0);
}
```

`process_bit`은 write 성공 뒤에만 decoder reset, sequence 증가, ACK 전송으로 진행한다.

```c
if (g_received_bits == 8)
{
    output = g_current_byte;
    if (flush_byte(output) == -1)
        return (-1);
    g_current_byte = 0;
    g_received_bits = 0;
}
if (g_client_pid != 0)
    g_sequence++;
if (send_response(event->sender, MT_RESPONSE_ACK, sequence,
        MT_RESPONSE_OK) == -1)
{
    /* session recovery */
}
```

8번째 bit의 output이 실패하면 `process_bit`이 즉시 `-1`을 반환한다. sequence는 증가하지 않고 ACK도 생성되지 않으며, event loop가 실패해 server가 종료된다. 손상된 partial state를 가지고 다음 event를 계속 처리하지 않는다.

### delimiter도 payload와 같은 commit이다

두 종류의 newline이 같은 원칙을 따른다.

- message NUL을 받은 뒤 쓰는 terminating newline
- dead owner가 이미 visible byte를 남겼을 때 replacement 전에 쓰는 recovery newline

```c
static int reset_session(int close_partial_line)
{
    if (close_partial_line && g_line_started
        && mt_write_all(STDOUT_FILENO, "\n", 1) == -1)
        return (-1);
    g_current_byte = 0;
    g_received_bits = 0;
    g_client_pid = 0;
    g_line_started = 0;
    g_sequence = 0;
    return (0);
}
```

recovery newline이 실패하면 state reset 전에 반환한다. 즉 output boundary를 만들지 못했는데도 새 owner를 READY/OK로 승인하는 상태를 피한다.

### startup publication과 `SIGPIPE`

server PID와 newline을 하나의 buffer로 조립해 `mt_write_all`로 publish한다. 실패하면 event loop에 들어가지 않고 `server: failed to publish pid` 경로로 끝난다. response endpoint는 이미 생성됐더라도 `atexit` cleanup이 제거한다.

또한 server는 `SIGPIPE`를 ignore한다. closed pipe/stdout에 쓸 때 process가 signal default action으로 즉시 사라지는 대신 `write`가 `-1/EPIPE`를 반환해 위 failure path가 실행되도록 한다.

### 이 commit이 뜻하는 commit의 범위

`mt_write_all` 성공은 byte가 kernel descriptor interface에 전달됐다는 local guarantee다. terminal이 실제로 화면에 그렸거나 파일이 durable storage에 sync됐다는 뜻은 아니다. output 뒤 ACK datagram 자체가 유실될 수도 있으므로 exactly-once delivery도 보장하지 않는다. 이 경우 output은 존재하지만 client는 timeout할 수 있다.

## `9aa80e047514` — test(server): 부분 쓰기와 출력 실패 검증

**중요도** `A` · **태그** `TEST, OUTPUT_COMMIT, RISK`

### production path를 유지한 write seam

fault build는 `src/write_utils.c`의 low-level call만 macro로 바꾼다.

```c
#ifndef MT_WRITE_CALL
# define MT_WRITE_CALL write
#endif

written = MT_WRITE_CALL(fd, bytes + offset, size - offset);
```

production build에서는 raw `write`와 같고, test build에서는 환경 변수에 따라 `mt_test_write`가 selected behavior를 만든다.

| fault | production branch | test가 요구하는 결과 |
| --- | --- | --- |
| startup에서 0 반환 | `written == 0` → `EIO` | PID output 없음, server 실패, endpoint 없음 |
| first `EINTR` + max 1-byte write | retry + offset progress | 전체 message와 newline이 정확히 출력됨 |
| payload byte `X`에서 `EPIPE` | `flush_byte` 실패 | client와 server 모두 실패, 해당 byte ACK 없음 |
| 두 번째 newline에서 `EPIPE` | terminating newline 실패 | empty message를 성공으로 ACK하지 않음 |

첫 newline은 startup PID line에 포함되므로 `MT_TEST_FAIL_NEWLINE_NUMBER=2`는 message terminator를 정확히 겨냥한다. payload와 delimiter가 같은 output contract를 통과하는지 분리해 확인한다.

이 commit의 test code는 실제 OS가 우연히 short write를 만들기를 기다리지 않는다. 같은 production loop에 deterministic 반환값을 넣어 각 branch를 재현한다.

## `081a882d7fa3` — test(server): 회수 줄바꿈 출력 실패 검증

**중요도** `A` · **태그** `TEST, OUTPUT_COMMIT, SESSION`

앞 test의 newline failure는 정상 message 종료를 겨냥했다. 이 commit은 **abandoned session 회수** 중 newline을 겨냥한다.

```text
session_sender partial
  → server stdout에 'X'는 보였지만 newline은 없음
  → sender 종료
replacement client ACQUIRE
  → server가 old owner unavailable 판단
  → reset_session(1)에서 recovery newline write
  → 두 번째 newline에 EPIPE 주입
```

script는 replacement client가 성공하지 않아야 하고 server도 nonzero로 끝나야 한다고 요구한다. server stdout의 complete line 수는 PID line 하나뿐이어야 하며, server diagnostic은 event channel failure, client diagnostic은 acknowledgement timeout이어야 한다.

이 test가 고정하는 핵심은 “dead owner를 발견했다”만으로 recovery가 완료되지 않는다는 점이다. visible partial line을 닫는 output commit까지 성공해야 state를 지우고 replacement session을 열 수 있다.

## 최종 commit 순서

```text
bit 1..7
  → decoder state 반영
  → ACK

bit 8
  → decoder state 반영
  → 완성 byte 판정
  → mt_write_all(payload 또는 newline)
       ├─ 실패: event loop 중단, ACK 없음
       └─ 성공: decoder/session state 정리
  → sequence transition
  → ACK

stale owner replacement
  → owner unavailable 판정
  → visible partial line이면 mt_write_all("\n")
       ├─ 실패: owner state 유지, READY 없음, server 중단
       └─ 성공: decoder/session reset, 새 owner 예약
```

## 이 Thread의 경계

이 Thread는 **어떤 stdout write가 성공해야 protocol transition을 ACK할 수 있는가**를 다룬다.

- `mt_write_all`은 remote ACK delivery나 exactly-once output을 보장하지 않는다.
- owner availability 판단 자체는 `02-session-ownership-and-recovery.md`에서 다룬다.
- signal event loss의 fail-stop은 `03-self-pipe-event-loop.md`의 주제다.
- endpoint path와 descriptor cleanup은 `05-endpoint-ownership-and-bounded-polling.md`에서 다룬다.
- response identity와 timeout validation은 `06-bounded-response-correlation.md`에서 다룬다.

## 조사 범위

각 설명은 exact SHA의 `src/write_utils.c`, `src/server.c`, fault build와 shell script diff를 기준으로 작성했다. 테스트는 실행하지 않았으므로 “통과했다”가 아니라 fixture가 요구하는 status·output·diagnostic을 서술했다.
