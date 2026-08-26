# Thread: endpoint 경로는 bind한 process만 정리하고 polling 표현 범위를 넘지 않는다
> Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 05

## 개요

Unix datagram endpoint를 만든다는 것은 descriptor 하나를 여는 작업보다 크다. process는 predictable filesystem path를 계산하고, 기존 entry가 교체 가능한 stale socket인지 확인하고, bind에 성공한 경우에만 그 path의 cleanup owner가 된다. “경로를 안다”, “기존 entry를 지울 수 있다”, “현재 endpoint를 내가 소유한다”는 서로 다른 상태다.

이 Thread의 앞부분은 per-UID namespace와 client/server endpoint lifecycle을 만든 뒤, bind 실패 후 unowned path를 지우던 client cleanup을 고친다. 뒷부분은 valid descriptor라도 `fd_set`이 표현할 수 없는 번호라면 `FD_SET` 전에 startup을 중단해야 한다는 별도 runtime boundary를 확립하고, real inherited descriptor table로 이를 검증한다.

### 최종 ownership 상태

| 단계 | process가 알고 있거나 소유한 것 | cleanup 책임 |
| --- | --- | --- |
| path 계산 완료 | expected pathname | 없음 |
| same-UID stale socket 제거 | 비어 있는 pathname | 없음 |
| socket 생성·flag 설정 | descriptor | descriptor close |
| bind 성공 | descriptor + filesystem socket entry | close + own path unlink |
| bind 전 실패 | 계산한 path와 descriptor 일부 | descriptor만 close, 기존 path는 unlink하지 않음 |
| descriptor `>= FD_SETSIZE` | polling에 넣을 수 없는 descriptor | 초기화 실패로 close/unlink rollback |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `2c37cb592d05` | feat(runtime): 안전한 응답 endpoint 경로 생성 | A | `ARCH, ENDPOINT, RISK` | current UID 전용 runtime directory와 role/PID 기반 Unix socket path를 만든다. |
| 2 | `25780b881ee8` | feat(client): datagram 응답 endpoint 수명주기 관리 | B | `ENDPOINT, PROCESS_LIFECYCLE` | client가 nonblocking·close-on-exec datagram socket을 PID path에 bind하고 invocation 종료 시 정리한다. |
| 3 | `32390dcdfc1b` | feat(server): datagram 응답 endpoint 수명주기 관리 | B | `ENDPOINT, PROCESS_LIFECYCLE` | server가 long-lived response socket을 bind하고 rollback과 normal exit에서 정리한다. |
| 4 | `622d80020fb2` | fix(client): bind한 응답 경로만 정리 | A | `ENDPOINT, RISK` | client cleanup이 actual bind ownership을 별도 flag로 확인하게 한다. |
| 5 | `ffd3647a1518` | test(runtime): stale 응답 endpoint 처리 검증 | A | `TEST, ENDPOINT, RISK` | real stale sockets, regular files와 unrelated process로 replacement·preservation policy를 검증한다. |
| 6 | `4e1c84bfacfc` | fix(runtime): select 범위를 벗어난 descriptor 거부 | A | `EDGE, PRACTICAL, RISK` | `FD_SET` 대상 descriptor가 `FD_SETSIZE` 이상이면 initialization에서 거부한다. |
| 7 | `1de95310195d` | test(runtime): 높은 descriptor 번호의 안전한 실패 검증 | A | `TEST, EDGE, RISK` | `/dev/null`을 반복 open한 뒤 real client/server를 `exec`해 high-FD failure를 검증한다. |

---

**역할군: private namespace와 endpoint lifecycle을 만들기**

## 2c37cb592d05 — feat(runtime): 안전한 응답 endpoint 경로 생성

**중요도** `A` · **태그** `ARCH, ENDPOINT, RISK`

### 무엇을 만들었는가 (diff)

```diff
+static int validate_runtime_dir(const char *path)
+{
+    struct stat info;
+
+    if (lstat(path, &info) == -1)
+        return (-1);
+    if (!S_ISDIR(info.st_mode) || info.st_uid != getuid()
+        || (info.st_mode & 077) != 0)
+    {
+        errno = EACCES;
+        return (-1);
+    }
+    return (0);
+}
@@
+    length = snprintf(buffer, size, "/tmp/signal-message-bus-%lu",
+            (unsigned long)getuid());
+    if (length < 0 || (size_t)length >= size)
+    {
+        errno = ENAMETOOLONG;
+        return (-1);
+    }
+    if (mkdir(buffer, 0700) == -1 && errno != EEXIST)
+        return (-1);
+    return (validate_runtime_dir(buffer));
```

existing runtime path는 directory인지, current UID 소유인지, group/other permission bit가 모두 꺼져 있는지 확인한다. 생성 mode는 0700이지만 validation은 owner permission 자체를 정확히 7로 고정하지 않고 외부 접근 가능성만 거부한다.

role/PID helper는 `server`, `client`, test의 `forger`만 허용하고 `pid <= 1`, NULL input과 formatted path truncation을 거부한다.

```diff
+    if (buffer == NULL || role == NULL || pid <= 1
+        || mt_runtime_dir(directory, sizeof(directory)) == -1)
+        return (-1);
+    if (strcmp(role, "server") != 0 && strcmp(role, "client") != 0
+        && strcmp(role, "forger") != 0)
+    {
+        errno = EINVAL;
+        return (-1);
+    }
+    length = snprintf(buffer, size, "%s/%s-%ld.sock", directory, role,
+            (long)pid);
+    if (length < 0 || (size_t)length >= size)
+    {
+        errno = ENAMETOOLONG;
+        return (-1);
+    }
```

### 이 namespace가 보장하지 않는 것은 무엇인가

private per-UID directory는 다른 UID의 filesystem interference를 줄인다. 같은 UID의 다른 process를 cryptographically authenticate하지 않으며, predictable PID-derived path 자체가 peer identity proof는 아니다.

### 관련 커밋과 어떤 관계인가

`25780b881ee8`과 `32390dcdfc1b`가 이 naming policy를 실제 client/server socket acquisition과 cleanup에 연결한다. `ffd3647a1518`은 real filesystem object로 stale replacement와 protected entry preservation을 검증한다.

## 25780b881ee8 — feat(client): datagram 응답 endpoint 수명주기 관리

**중요도** `B` · **태그** `ENDPOINT, PROCESS_LIFECYCLE`

### 무엇이 바뀌었는가 (diff)

```diff
+static int g_response_socket = -1;
+static char g_client_path[MT_RESPONSE_PATH_SIZE];
+
+static void cleanup_response_socket(void)
+{
+    if (g_response_socket != -1)
+        close(g_response_socket);
+    g_response_socket = -1;
+    if (g_client_path[0] != '\0')
+        unlink(g_client_path);
+    g_client_path[0] = '\0';
+}
```

client setup 순서는 path 계산 → same-UID stale socket 확인/삭제 → `socket(AF_UNIX, SOCK_DGRAM)` → `O_NONBLOCK`/`FD_CLOEXEC` → sockaddr 구성 → `bind`다. 성공한 invocation은 `atexit(cleanup_response_socket)`로 descriptor와 filesystem entry를 정리한다.

### 무엇을 준비하지만 아직 불완전한가

`g_client_path`는 bind를 시도하기 전에 채워진다. 그런데 cleanup은 path 문자열이 비어 있지 않다는 사실만 보고 unlink한다. bind가 기존 regular file 때문에 실패하면 client가 그 file을 소유한 적이 없는데도 cleanup에서 삭제를 시도할 수 있다. path knowledge와 bind ownership이 아직 분리되지 않았다.

### 관련 커밋과 어떤 관계인가

`622d80020fb2`가 successful bind를 ownership publication point로 만들고 cleanup 조건을 고친다. `ffd3647a1518`의 regular-file fixture가 이 차이를 직접 관찰한다.

## 32390dcdfc1b — feat(server): datagram 응답 endpoint 수명주기 관리

**중요도** `B` · **태그** `ENDPOINT, PROCESS_LIFECYCLE`

### 무엇이 바뀌었는가 (diff)

```diff
+static int  g_response_socket = -1;
+static int  g_server_bound;
+static char g_server_path[MT_RESPONSE_PATH_SIZE];
+
+static void cleanup_server(void)
+{
+    if (g_response_socket != -1)
+        close(g_response_socket);
+    g_response_socket = -1;
+    if (g_server_bound && g_server_path[0] != '\0')
+        unlink(g_server_path);
+    g_server_bound = 0;
+    g_server_path[0] = '\0';
+}
@@
+    if (bind(g_response_socket, (struct sockaddr *)&address,
+            sizeof(address)) == -1)
+        return (-1);
+    g_server_bound = 1;
```

### 왜 가볍게 다루는가

client와 같은 create/flags/stale/bind lifecycle을 server에 적용하지만, server는 처음부터 `g_server_bound`를 actual bind success 뒤에 설정한다. 이 Thread의 핵심 ownership bug는 client path에만 남는다. server-specific 의미는 long-lived control endpoint가 startup failure와 normal exit에서 동일 cleanup 함수를 사용한다는 점이다.

### 관련 커밋과 어떤 관계인가

`caf2feec4971`과 이후 event loop는 이 bound server socket에서 session request를 받지만, 그 protocol 의미는 다른 Thread의 책임이다. 여기서는 `2c37cb592d05`의 path policy가 resource lifecycle로 이어지는 지점만 다룬다.

## 622d80020fb2 — fix(client): bind한 응답 경로만 정리

**중요도** `A` · **태그** `ENDPOINT, RISK`

### 무엇이 바뀌었는가 (diff)

```diff
 static int  g_response_socket = -1;
 static char g_client_path[MT_RESPONSE_PATH_SIZE];
+static int  g_client_bound;
@@
-    if (g_client_path[0] != '\0')
+    if (g_client_bound && g_client_path[0] != '\0')
         unlink(g_client_path);
+    g_client_bound = 0;
     g_client_path[0] = '\0';
@@
     if (bind(g_response_socket, (struct sockaddr *)&address,
             sizeof(address)) == -1)
         return (-1);
+    g_client_bound = 1;
```

### 왜 이렇게 작은가

문제는 cleanup code 양이 아니라 ownership representation 하나가 빠진 데 있다. path buffer가 채워졌다는 사실은 bind success를 뜻하지 않는다. `g_client_bound`는 exact success 지점에서만 1이 되고, cleanup은 이 flag가 있을 때만 unlink한다. descriptor close는 bind 여부와 무관하게 계속 수행한다.

### 관련 커밋과 어떤 관계인가

`ffd3647a1518`은 client PID path에 stale socket 또는 regular file을 미리 만들어, 전자는 교체·정리하고 후자는 보존하는지 검증한다.

## ffd3647a1518 — test(runtime): stale 응답 endpoint 처리 검증

**중요도** `A` · **태그** `TEST, ENDPOINT, RISK`

### 왜 다른 기법이 필요한가

cleanup ownership은 return code만으로 확인하기 어렵다. test는 target PID에서 실제로 계산되는 path에 filesystem object를 만들고 child를 `exec`해, 종료 뒤 object가 남았는지 직접 검사한다.

```diff
+static int create_stale_entry(const char *path, const char *mode)
+{
+    int fd;
+
+    unlink(path);
+    if (strcmp(mode, "socket") == 0)
+        return (create_stale_socket(path));
+    if (strcmp(mode, "file") != 0)
+        return (-1);
+    fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0600);
+    if (fd == -1)
+        return (-1);
+    return (close(fd));
+}
```

shell regression은 다음 차이를 요구한다.

```diff
+"$ROOT/tests/stale_exec" "$ROOT/client" "$SERVER_PID" stale socket
+[ ! -s "$TEST_TMP/stale.err" ]
+
+"$ROOT/tests/stale_exec" "$ROOT/client" "$SERVER_PID" blocked file \
+    2>"$TEST_TMP/file.err"
+grep -qx 'client: failed to create response channel' "$TEST_TMP/file.err"
```

stale same-UID socket은 교체할 수 있고 child 종료 뒤 없어져야 한다. regular file은 client setup을 실패시키지만 그대로 보존돼야 한다. 별도 helper는 unrelated live process의 PID path와 stale server entry도 구성한다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

selected same-UID socket/file fixtures에서 replacement authority와 cleanup ownership이 구분된다는 direct filesystem evidence를 제공한다. bind와 lstat 사이의 TOCTOU race, symlink 공격 전체, 다른 UID fixture는 다루지 않는다.

### 관련 커밋과 어떤 관계인가

`622d80020fb2`의 bound flag가 없다면 regular-file scenario의 cleanup이 unowned path에 unlink를 시도한다. `2c37cb592d05`의 directory mode와 object-type validation도 같은 suite에서 관찰된다.

---

**역할군: valid descriptor와 `fd_set`에 표현 가능한 descriptor를 구분하기**

## 4e1c84bfacfc — fix(runtime): select 범위를 벗어난 descriptor 거부

**중요도** `A` · **태그** `EDGE, PRACTICAL, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

`socket`이나 `pipe`가 nonnegative descriptor를 반환해도 `FD_SET(fd, &set)`이 안전하다는 뜻은 아니다. classic `fd_set`은 `0 <= fd < FD_SETSIZE`만 표현한다. 높은 번호를 넘기면 set storage 범위 밖을 접근할 수 있으므로, descriptor 생성 직후 controlled initialization failure로 바꾼다.

```diff
     g_response_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
     if (g_response_socket == -1
+        || g_response_socket >= FD_SETSIZE
         || set_nonblocking_close_on_exec(g_response_socket) == -1)
         return (-1);
@@
     if (pipe(g_event_pipe) == -1
+        || g_event_pipe[0] >= FD_SETSIZE
         || set_close_on_exec(g_event_pipe[0]) == -1
@@
     g_response_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
     if (g_response_socket == -1
+        || g_response_socket >= FD_SETSIZE
         || set_nonblocking_close_on_exec(g_response_socket) == -1)
         return (-1);
```

client response socket, server response socket와 self-pipe read end가 실제 `FD_SET` 대상이므로 세 곳을 검사한다. self-pipe write end는 handler가 write만 하고 `fd_set`에 넣지 않으므로 같은 guard 대상이 아니다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

`select` representation을 넘는 descriptor를 use-before-check하지 않고 setup failure로 바꾼다. polling backend를 `poll`/`epoll`로 확장하거나 `FD_SETSIZE` 자체를 늘리지는 않는다.

### 관련 커밋과 어떤 관계인가

`1de95310195d`는 fake integer를 주입하지 않고 inherited descriptor table을 실제로 채운 뒤 client/server가 이 guard에서 실패하는지 검증한다.

## 1de95310195d — test(runtime): 높은 descriptor 번호의 안전한 실패 검증

**중요도** `A` · **태그** `TEST, EDGE, RISK`

### 무엇을 검증하는가

helper는 `/dev/null`을 반복 open해 마지막 descriptor가 `FD_SETSIZE - 1` 이상이 된 뒤 target program을 `exec`한다. open descriptors는 close-on-exec가 아니므로 child가 새로 얻는 socket/pipe 번호가 polling 범위를 넘는다.

```diff
+int main(int argc, char **argv)
+{
+    int fd;
+
+    if (argc < 2)
+        return (2);
+    fd = -1;
+    while (fd < FD_SETSIZE - 1)
+    {
+        fd = open("/dev/null", O_RDONLY);
+        if (fd == -1)
+            return (1);
+    }
+    execv(argv[1], &argv[1]);
+    return (127);
+}
```

shell test는 high-FD client가 status 1과 `client: failed to create response channel`을 내되 기존 server는 살아 있어야 한다고 요구한다. high-FD server는 PID를 publish하지 않고 `server: failed to create response channel`로 status 1이어야 한다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

real descriptor allocation order를 통해 client/server initialization이 `FD_SET` 전에 멈춘다는 regression evidence다. 운영체제가 hard limit 때문에 `FD_SETSIZE`까지 open하지 못하는 환경, 다른 polling call site, descriptor exhaustion 자체의 정책을 일반화하지 않는다.

### 관련 커밋과 어떤 관계인가

`4e1c84bfacfc`의 세 guards가 모두 acquisition 직후에 있어야 이 test가 memory-unsafe polling 대신 controlled error로 끝난다. endpoint cleanup code도 실패 과정에서 이미 얻은 descriptors와 bound path를 회수해야 한다.

## 이 Thread의 경계

- server가 client endpoint의 존재를 session owner availability에 사용하는 의미는 `02-session-ownership-and-recovery.md`가 다룬다.
- self-pipe가 signal event를 전달하고 overflow를 처리하는 protocol 책임은 `03-self-pipe-event-loop.md`에 있다. 이 문서는 pipe descriptor의 acquisition과 polling 표현 범위만 다룬다.
- response source path와 record fields를 함께 검사하는 acceptance policy는 `06-bounded-response-correlation.md`가 다룬다.
- same-UID adversary에 대한 cryptographic authentication, filesystem TOCTOU hardening과 `select`를 다른 polling API로 교체하는 설계는 별개의 문제다.

> 검토 범위: `2c37cb592d05`, `25780b881ee8`, `32390dcdfc1b`, `622d80020fb2`, `ffd3647a1518`, `4e1c84bfacfc`, `1de95310195d`의 diff와 해당 SHA의 `src/response_channel.c`, client/server setup·cleanup, stale/high-FD test helpers를 확인했다. branch checkout, stale-entry suite와 high-FD suite 실행은 수행하지 않았으므로 runtime 통과를 주장하지 않는다.
