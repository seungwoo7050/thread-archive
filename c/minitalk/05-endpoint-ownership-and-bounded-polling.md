# Thread: Unix endpoint 소유권과 `fd_set` 표현 범위

Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 05

## 개요

Unix datagram socket은 descriptor 하나만 얻는 작업이 아니다. filesystem namespace에 predictable path를 만들고, 기존 entry를 검사하고, bind 성공 시 그 path를 삭제할 책임까지 얻는다. 따라서 “path 문자열을 계산했다”, “stale socket을 지울 권한이 있다”, “내 socket을 그 path에 bind했다”는 서로 다른 상태다.

또한 descriptor가 유효한 정수라는 사실만으로 `select`에 넣을 수 있는 것은 아니다. `fd_set`은 `FD_SETSIZE` 미만의 번호만 표현할 수 있으므로, 높은 descriptor를 `FD_SET`에 넘기기 전에 controlled startup failure로 바꿔야 한다.

### 최종 불변 조건

- runtime directory는 current UID 전용 경로이며 directory type, owner, group/other permission을 검사한다.
- endpoint path는 허용된 role과 PID에서만 생성하고 buffer truncation을 거부한다.
- same-UID stale Unix socket은 교체할 수 있지만 regular file이나 다른 종류의 entry는 삭제하지 않는다.
- cleanup은 **실제로 bind한 path만** unlink한다.
- socket과 self-pipe descriptors는 acquisition 실패와 normal exit 모두에서 close된다.
- `FD_SET`에 들어갈 client socket, server socket, event-pipe read end는 생성 직후 `FD_SETSIZE` 미만인지 검사한다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `2c37cb592d05` | feat(runtime): 안전한 응답 endpoint 경로 생성 | A | `ARCH, ENDPOINT, RISK` | per-UID private runtime directory와 role/PID-derived Unix socket path helper를 정의합니다. |
| 2 | `25780b881ee8` | feat(client): datagram 응답 endpoint 수명주기 관리 | B | `ENDPOINT, PROCESS_LIFECYCLE` | client가 nonblocking, close-on-exec datagram socket을 PID-derived path에 bind하고 invocation lifetime에 맞춰 cleanup합니다. |
| 3 | `32390dcdfc1b` | feat(server): datagram 응답 endpoint 수명주기 관리 | B | `ENDPOINT, PROCESS_LIFECYCLE` | server가 long-lived nonblocking, close-on-exec datagram endpoint를 server path에 bind하고 normal exit와 rollback에서 cleanup합니다. |
| 4 | `622d80020fb2` | fix(client): bind한 응답 경로만 정리 | A | `ENDPOINT, RISK` | client cleanup이 response path를 실제 `bind`한 경우에만 unlink하도록 bound ownership flag를 사용합니다. |
| 5 | `ffd3647a1518` | test(runtime): stale 응답 endpoint 처리 검증 | A | `TEST, ENDPOINT, RISK` | real PID-derived paths에 stale client/server sockets, regular files, unrelated live processes를 만들어 endpoint trust와 cleanup policy를 검증합니다. |
| 6 | `4e1c84bfacfc` | fix(runtime): select 범위를 벗어난 descriptor 거부 | A | `EDGE, PRACTICAL, RISK` | client response socket, server response socket, self-pipe read fd가 `FD_SETSIZE` 이상이면 initialization에서 거부합니다. |
| 7 | `1de95310195d` | test(runtime): 높은 descriptor 번호의 안전한 실패 검증 | A | `TEST, EDGE, RISK` | wrapper가 `/dev/null`을 반복 open해 inherited descriptor table을 `FD_SETSIZE` boundary까지 높인 뒤 real client/server를 `exec`합니다. |

## `2c37cb592d05` — feat(runtime): 안전한 응답 endpoint 경로 생성

**중요도** `A` · **태그** `ARCH, ENDPOINT, RISK`

### per-UID namespace

`mt_runtime_dir`는 다음 경로를 만든다.

```text
/tmp/signal-message-bus-<uid>
```

```c
length = snprintf(buffer, size, "/tmp/signal-message-bus-%lu",
        (unsigned long)getuid());
if (length < 0 || (size_t)length >= size)
{
    errno = ENAMETOOLONG;
    return (-1);
}
if (mkdir(buffer, 0700) == -1 && errno != EEXIST)
    return (-1);
return (validate_runtime_dir(buffer));
```

이미 존재하는 path는 이름만 믿지 않는다.

```c
if (lstat(path, &info) == -1)
    return (-1);
if (!S_ISDIR(info.st_mode) || info.st_uid != getuid()
    || (info.st_mode & 077) != 0)
{
    errno = EACCES;
    return (-1);
}
```

검사는 directory type, current UID owner, group/other permission 부재를 요구한다. 생성 시에는 0700을 사용하지만 validation이 owner permission을 정확히 0700으로 강제하는 것은 아니다. 핵심은 다른 UID 또는 group/other가 접근 가능한 namespace를 사용하지 않는다는 점이다.

### role/PID path

```c
if (buffer == NULL || role == NULL || pid <= 1
    || mt_runtime_dir(directory, sizeof(directory)) == -1)
    return (-1);
if (strcmp(role, "server") != 0 && strcmp(role, "client") != 0
    && strcmp(role, "forger") != 0)
{
    errno = EINVAL;
    return (-1);
}
length = snprintf(buffer, size, "%s/%s-%ld.sock",
        directory, role, (long)pid);
if (length < 0 || (size_t)length >= size)
{
    errno = ENAMETOOLONG;
    return (-1);
}
```

결과는 `server-<pid>.sock`, `client-<pid>.sock`이며 test response source를 위한 `forger` role도 whitelist에 포함된다. helper는 path를 계산할 뿐 파일을 만들거나 그 path의 소유권을 획득하지 않는다.

이 namespace는 cooperative same-UID local boundary다. path와 UID 검사는 다른 user의 entry를 배제하지만, 같은 UID의 악의적인 process까지 cryptographically 인증하지는 않는다.

## `25780b881ee8` — feat(client): datagram 응답 endpoint 수명주기 관리

**중요도** `B` · **태그** `ENDPOINT, PROCESS_LIFECYCLE`

client는 자신의 PID-derived path를 준비하고 datagram socket을 bind한다.

```text
client path 계산
  → 기존 entry 검사/제거
  → socket(AF_UNIX, SOCK_DGRAM)
  → O_NONBLOCK, FD_CLOEXEC
  → sockaddr_un 길이 확인
  → bind
  → atexit cleanup 등록
```

stale entry 제거는 제한적이다.

```c
if (lstat(path, &info) == -1)
{
    if (errno == ENOENT)
        return (0);
    return (-1);
}
if (!S_ISSOCK(info.st_mode) || info.st_uid != getuid())
{
    errno = EACCES;
    return (-1);
}
return (unlink(path));
```

same-UID Unix socket만 stale candidate로 지운다. regular file, symlink가 가리키는 target, 다른 UID entry는 거부한다. `lstat`을 사용하므로 symlink 자체는 socket으로 판정되지 않는다.

cleanup은 socket descriptor를 close하고 client path를 unlink하도록 작성됐다. 그러나 이 최초 구현은 path 문자열이 채워졌다는 사실만 보고 unlink한다. bind 전 failure에서도 `g_client_path`는 이미 계산돼 있으므로 ownership 판정이 불완전하다. 이 bug가 `622d80020fb2`의 직접 원인이다.

## `32390dcdfc1b` — feat(server): datagram 응답 endpoint 수명주기 관리

**중요도** `B` · **태그** `ENDPOINT, PROCESS_LIFECYCLE`

server도 같은 stale-entry 검사, nonblocking/CLOEXEC socket, path-length check를 거쳐 long-lived endpoint를 bind한다.

server 구현은 처음부터 `g_server_bound`를 둔다.

```c
if (bind(g_response_socket, (struct sockaddr *)&address,
        sizeof(address)) == -1)
    return (-1);
g_server_bound = 1;
```

cleanup은 이 flag가 true일 때만 unlink한다.

```c
if (g_server_bound && g_server_path[0] != '\0')
    unlink(g_server_path);
g_server_bound = 0;
g_server_path[0] = '\0';
```

따라서 path 계산 후 socket creation이나 bind가 실패하면 descriptor는 close하지만 unowned filesystem entry는 지우지 않는다. normal main return과 initialization rollback 모두 같은 cleanup function을 사용한다.

## `622d80020fb2` — fix(client): bind한 응답 경로만 정리

**중요도** `A` · **태그** `ENDPOINT, RISK`

### path knowledge와 ownership을 혼동한 failure

client path에 같은 UID의 regular file이 있다고 가정한다.

1. `mt_response_path`가 그 path를 `g_client_path`에 쓴다.
2. `remove_stale_socket`은 regular file을 보고 `EACCES`로 실패한다. 여기까지는 안전하다.
3. main의 error path가 `cleanup_response_socket()`을 호출한다.
4. old cleanup은 `g_client_path[0] != '\0'`만 보고 regular file을 unlink한다.

즉 guard가 stale removal 함수에는 있었지만 rollback cleanup이 이를 우회했다.

### bind success를 ownership publication으로 사용

```diff
 static int  g_response_socket = -1;
 static char g_client_path[MT_RESPONSE_PATH_SIZE];
+static int  g_client_bound;
```

```c
if (bind(g_response_socket, (struct sockaddr *)&address,
        sizeof(address)) == -1)
    return (-1);
g_client_bound = 1;
```

```c
if (g_client_bound && g_client_path[0] != '\0')
    unlink(g_client_path);
g_client_bound = 0;
g_client_path[0] = '\0';
```

`g_client_bound`는 단순 상태 flag가 아니라 filesystem cleanup authority가 생기는 commit point다. path를 계산하거나 stale socket을 지운 것만으로는 true가 되지 않는다.

## `ffd3647a1518` — test(runtime): stale 응답 endpoint 처리 검증

**중요도** `A` · **태그** `TEST, ENDPOINT, RISK`

이 commit은 mock path 대신 실제 future child PID에서 계산한 path에 entry를 만든 뒤 child를 exec한다. gate pipe로 child 실행 시점을 늦춰, parent가 정확한 PID-derived path를 먼저 준비한다.

주요 fixture는 다음 policy를 구분한다.

| 준비한 상태 | 기대 동작 |
| --- | --- |
| target PID는 살아 있지만 server endpoint 없음 | client가 invalid server PID로 거부 |
| same-UID stale client Unix socket | client가 stale socket을 제거하고 새 endpoint를 bind해 정상 전송 |
| client path에 regular file | client startup 실패, file은 남아 있음 |
| server path에 stale same-UID socket | server가 교체 가능 |
| server path에 protected non-socket entry | server startup 실패, entry를 임의 삭제하지 않음 |
| 새 runtime directory | mode가 700 |

`stale_exec`는 fork한 child의 PID path에 socket 또는 file을 만들고 child를 release해 real client를 exec한다. socket case에서는 child 성공 후 path가 사라져야 하고, file case에서는 child가 실패해도 path가 그대로 있어야 한다. 이는 `622d80020fb2`의 bound flag를 filesystem 관찰로 고정한다.

이 test는 same-UID namespace가 공격자에 안전하다는 것을 증명하지 않는다. 허용된 stale socket 교체와 보호해야 할 non-socket entry를 구분하는 local cleanup policy를 검증한다.

## `4e1c84bfacfc` — fix(runtime): select 범위를 벗어난 descriptor 거부

**중요도** `A` · **태그** `EDGE, PRACTICAL, RISK`

`FD_SET(fd, &set)`은 fd가 open되어 있는지만 확인하지 않는다. `fd >= FD_SETSIZE`이면 fixed bitmap 밖을 접근할 수 있다. 따라서 실제 `FD_SET` 대상이 되는 descriptor를 acquisition 직후 거부한다.

```c
/* client: wait_for_response에서 FD_SET되는 socket */
if (g_response_socket == -1
    || g_response_socket >= FD_SETSIZE
    || set_nonblocking_close_on_exec(g_response_socket) == -1)
    return (-1);
```

```c
/* server: event loop의 self-pipe read end */
if (pipe(g_event_pipe) == -1
    || g_event_pipe[0] >= FD_SETSIZE
    || set_close_on_exec(g_event_pipe[0]) == -1
    || set_nonblocking_close_on_exec(g_event_pipe[1]) == -1)
    return (-1);
```

```c
/* server: event loop의 datagram socket */
if (g_response_socket == -1
    || g_response_socket >= FD_SETSIZE
    || set_nonblocking_close_on_exec(g_response_socket) == -1)
    return (-1);
```

write-only event pipe end는 `FD_SET`에 들어가지 않으므로 이 commit에서 같은 guard 대상이 아니다. initialization이 실패하면 기존 cleanup이 열린 pipe ends/socket을 close하고 bound path만 제거한다.

이 fix는 descriptor를 낮은 번호로 복제하지 않는다. 현재 `select` representation으로 표현할 수 없는 runtime state를 명시적으로 거부한다.

## `1de95310195d` — test(runtime): 높은 descriptor 번호의 안전한 실패 검증

**중요도** `A` · **태그** `TEST, EDGE, RISK`

`tests/high_fd_exec`는 `/dev/null`을 반복 open한다.

```c
fd = -1;
while (fd < FD_SETSIZE - 1)
{
    fd = open("/dev/null", O_RDONLY);
    if (fd == -1)
        return (1);
}
execv(argv[1], &argv[1]);
```

wrapper가 연 descriptors에는 `FD_CLOEXEC`를 설정하지 않으므로 exec된 client/server가 같은 높은 table을 상속한다. 다음 `socket` 또는 `pipe`가 `FD_SETSIZE` 이상의 실제 kernel descriptor를 반환한다.

script가 요구하는 결과는 다음과 같다.

- high-FD client는 `client: failed to create response channel`과 status 1로 끝난다.
- 그 client가 대상으로 삼은 정상 server는 계속 살아 있고 stderr도 비어 있다.
- high-FD server는 PID를 publish하거나 event loop에 들어가지 않고 `server: failed to create response channel`로 status 1을 반환한다.

mock integer를 helper에 넘기는 unit test가 아니라 real inherited descriptor table을 사용하므로, guard 위치와 initialization cleanup을 함께 통과한다.

## 소유권 상태 요약

| 상태 | path | descriptor | cleanup 권한 |
| --- | --- | --- | --- |
| path 계산 전 | 없음 | 없음 | 없음 |
| path 계산 완료 | 알고 있음 | 없음 | 없음 |
| same-UID stale socket 제거 | 빈 namespace | 없음 | 아직 없음 |
| socket 생성/flags 설정 | bind 예정 path | open | descriptor close만 |
| bind 성공 | process가 만든 socket entry | open | close + unlink |
| bind 전 실패 | 기존 entry일 수 있음 | 일부 resource 가능 | close만, path unlink 금지 |
| normal exit | owned path | open | close + unlink 후 state clear |

## 이 Thread의 경계

이 Thread는 **endpoint namespace와 polling representation을 누가 언제 소유하는가**를 다룬다.

- endpoint가 session owner availability의 일부가 되는 이유는 `02-session-ownership-and-recovery.md`에서 다룬다.
- self-pipe가 전달하는 event semantics는 `03-self-pipe-event-loop.md`의 주제다.
- response source path와 record fields를 결합한 acceptance predicate는 `06-bounded-response-correlation.md`에서 다룬다.
- same-UID path는 cryptographic authentication이나 hostile local user isolation을 제공하지 않는다.

## 조사 범위

각 설명은 exact SHA의 `src/response_channel.c`, client/server setup·cleanup diff와 test helpers를 기준으로 작성했다. high-FD와 stale-entry scripts는 실행하지 않았으며, source에 적힌 fixture와 assertion을 실행 결과처럼 표현하지 않았다.
