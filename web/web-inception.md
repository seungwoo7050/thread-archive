===== BEGIN FILE: 01-readiness-aware-three-tier-stack.md =====
# Thread 1 — From custom services to a readiness-aware three-tier stack

## Thread 목표

개별 MariaDB, WordPress, Nginx 서비스가 Docker Compose 안에서 하나의 상태를 갖는 시스템으로 결합되는 과정을 복원합니다. 핵심은 이미지 세 개의 존재가 아니라 외부 transport, 애플리케이션 실행, 영속 상태의 책임을 분리하고 readiness와 named volume으로 연결한 결정입니다.

**Source significance**

> The thread progresses from individually runnable containers to one stateful application system. The decisive step is not the existence of three images but the Compose responsibility boundary: Nginx owns external transport, WordPress owns application execution, and MariaDB owns durable relational state. Health-gated dependencies and mounted volumes then make the topology operationally meaningful rather than merely connected.

## 이 Thread를 이해하기 위한 핵심 질문

- 각 서비스는 어떤 책임을 독점하며, 어떤 책임을 갖지 않습니까?
- 외부 HTTPS 요청은 어떤 설정 계약을 거쳐 PHP-FPM과 MariaDB까지 도달합니까?
- 이미지 계층의 상태와 named volume의 권위 있는 상태는 어디에서 분리됩니까?
- 컨테이너 생성 순서와 실제 서비스 준비 상태는 어떻게 구분됩니까?
- 초기 idempotent entrypoint가 제공한 보장과 이후 interruption-safe bootstrap이 추가로 해결한 문제는 무엇입니까?

## 완료 기준

- 세 서비스의 build/runtime/volume/network 책임을 해당 SHA의 설정과 entrypoint로 설명할 수 있습니다.
- Nginx → FastCGI → WordPress → MariaDB 경로를 실제 directive와 service name으로 추적했습니다.
- 초기화 조건, volume 재사용 조건, health check 조건을 서로 혼동하지 않고 기록했습니다.
- 초기 설계가 interruption-safe하지 않았던 지점을 후속 bootstrap thread와 연결했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `f8ec9621725c` | feat(mariadb): Debian 서버 이미지 추가 | **B** | `STACK`<br>`PERSISTENCE` | Established the project-owned MariaDB runtime and persistent-data ownership. |
| 2 | `e13b0357a21b` | feat(mariadb): DB와 애플리케이션 계정 초기화 | **A** | `BOOTSTRAP`<br>`SECRETS`<br>`CORE` | Added the first idempotent database and account bootstrap. |
| 3 | `d764d066167b` | feat(wordpress): 사이트와 사용자 계정 초기화 | **A** | `BOOTSTRAP`<br>`PERSISTENCE`<br>`CORE` | Added WordPress filesystem, site, and user convergence. |
| 4 | `99c03f54399a` | feat(nginx): PHP 요청을 WordPress로 전달 | **A** | `STACK`<br>`INTEGRATION`<br>`CORE` | Defined the HTTPS-to-FastCGI request boundary. |
| 5 | `a8b9f693c614` | feat(compose): 세 서비스 토폴로지 구성 | **S** | `ARCH`<br>`STACK`<br>`CORE` | Assembled the three service responsibilities into the core Compose topology. |
| 6 | `75590dedfb3a` | feat(compose): 준비 상태에 따라 영속 서비스 연결 | **A** | `PERSISTENCE`<br>`INTEGRATION`<br>`OPERATIONS` | Connected named volumes, health checks, and dependency readiness. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `f8ec9621725c` — feat(mariadb): Debian 서버 이미지 추가

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `STACK`, `PERSISTENCE` |
| Source-defined role | Established the project-owned MariaDB runtime and persistent-data ownership. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `e13b0357a21b` |

#### 원문이 확정한 범위

- **Summary:** Creates the custom Debian MariaDB image and foreground daemon lifecycle.
- **Classification reason:** The image is required for the project-owned database service, yet it mainly establishes expected container packaging and ownership conventions.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `f8ec9621725c`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/mariadb/Dockerfile`의 `FROM / RUN / CMD`에서 이미지가 DB 실행 파일과 빈 데이터 경로만 제공하고 실제 데이터는 실행 시 경로가 소유하도록 준비합니다.
- `srcs/requirements/mariadb/Dockerfile`의 `mariadbd foreground command`에서 PID 1과 DB daemon lifecycle이 분리되지 않으며 컨테이너 종료가 daemon 종료로 이어집니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| f8ec9621725c | srcs/requirements/mariadb/Dockerfile | FROM / RUN / CMD | Debian 기반에 MariaDB server/client, CA, gosu를 설치하고 배포판의 초기 DB 내용을 제거한 뒤 runtime/data directory를 `mysql` 소유로 다시 만듭니다. | 이미지가 DB 실행 파일과 빈 데이터 경로만 제공하고 실제 데이터는 실행 시 경로가 소유하도록 준비합니다. |
| f8ec9621725c | srcs/requirements/mariadb/Dockerfile | mariadbd foreground command | 최종 명령은 `mariadbd`를 `mysql` 사용자로 foreground에서 실행합니다. | PID 1과 DB daemon lifecycle이 분리되지 않으며 컨테이너 종료가 daemon 종료로 이어집니다. |

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Established the project-owned MariaDB runtime and persistent-data ownership. |
| 핵심 input / output / state | 이미지는 실행 파일과 초기 디렉터리 권한을 소유하지만, `/var/lib/mysql`에 실제 데이터가 생긴 뒤의 내용과 수명은 runtime 또는 후속 volume mount가 소유합니다. |
| 변경된 directive / helper / command | `srcs/requirements/mariadb/Dockerfile`의 `FROM / RUN / CMD`; `srcs/requirements/mariadb/Dockerfile`의 `mariadbd foreground command` |
| immediate failure 또는 boundary | 이 SHA에는 first-run database/account bootstrap이 없습니다. 빈 경로에서 daemon이 어떤 스키마와 계정을 가져야 하는지는 아직 정의되지 않았습니다. |
| 다음 commit에 넘긴 한계 | 초기 스키마, root hardening, application database/user, interruption recovery, named-volume persistence는 보장하지 않습니다. `e13b0357a21b`가 같은 이미지에 idempotent first-run entrypoint를 추가하고, `75590dedfb3a`가 data directory를 named volume에 연결합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 초기 스키마, root hardening, application database/user, interruption recovery, named-volume persistence는 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `e13b0357a21b`가 같은 이미지에 idempotent first-run entrypoint를 추가하고, `75590dedfb3a`가 data directory를 named volume에 연결합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 프로젝트가 직접 빌드한 MariaDB가 foreground process로 실행되고, 데이터 경로가 `mysql` 사용자에게 쓰기 가능하다는 점을 보장합니다.

### 2. `e13b0357a21b` — feat(mariadb): DB와 애플리케이션 계정 초기화

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `BOOTSTRAP`, `SECRETS`, `CORE` |
| Source-defined role | Added the first idempotent database and account bootstrap. |
| 이전 Thread commit | `f8ec9621725c` |
| 다음 Thread commit | `d764d066167b` |

#### 원문이 확정한 범위

- **Summary:** Adds first-run MariaDB initialization, account hardening, database creation, and idempotent volume reuse.
- **Classification reason:** This is the first substantial state-creation mechanism and establishes least-privilege database ownership, even though the later staged bootstrap redesign supersedes parts of it.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `e13b0357a21b`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `secret input / identifier validation`에서 초기화 SQL에 들어가는 입력을 entrypoint가 먼저 정규화합니다.
- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `first-run branch / temporary server`에서 외부 TCP 요청을 받기 전에 bootstrap SQL을 실행합니다.
- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `hardening SQL / final exec`에서 초기화 process와 장기 실행 daemon 사이의 handoff가 명시됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| e13b0357a21b | srcs/requirements/mariadb/tools/docker-entrypoint.sh | secret input / identifier validation | 직접 값과 `_FILE` 입력의 동시 사용을 거부하고, 필수 값 누락·잘못된 DB/user 식별자·SQL literal을 검사하거나 escape합니다. | 초기화 SQL에 들어가는 입력을 entrypoint가 먼저 정규화합니다. |
| e13b0357a21b | srcs/requirements/mariadb/tools/docker-entrypoint.sh | first-run branch / temporary server | system database directory가 없을 때만 `mariadb-install-db`를 실행하고, networking을 끈 socket-only 임시 서버를 띄워 readiness를 기다립니다. | 외부 TCP 요청을 받기 전에 bootstrap SQL을 실행합니다. |
| e13b0357a21b | srcs/requirements/mariadb/tools/docker-entrypoint.sh | hardening SQL / final exec | anonymous user·remote root·test DB를 제거하고 application DB/user/grant를 만든 뒤 임시 서버를 종료하고 최종 `mariadbd`로 `exec`합니다. | 초기화 process와 장기 실행 daemon 사이의 handoff가 명시됩니다. |

#### 비교 기준

- exact commit diff: `git diff e13b0357a21b^ e13b0357a21b -- <path>`
- 이전 Thread 상태와 비교: `git diff f8ec9621725c e13b0357a21b -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | `f8ec9621725c`의 이미지는 빈 DB 경로와 daemon만 제공했으므로 WordPress가 인증할 database/account가 없었습니다. |
| 선택한 boundary / decision | entrypoint가 빈 volume과 이미 채워진 volume을 구분하고, 빈 경우에만 socket-only 임시 서버를 이용해 DB와 계정을 생성하도록 했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `secret input / identifier validation`; `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `first-run branch / temporary server`; `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `hardening SQL / final exec` |
| state / ownership / lifecycle 변화 | first run에서는 entrypoint가 데이터 디렉터리와 temporary server를 관리합니다. 재시작에서는 system directory 존재를 근거로 bootstrap을 건너뛰고 기존 DB state를 권위자로 취급합니다. |
| 주요 failure branch | 임시 서버 start/readiness/SQL/shutdown 중 오류가 나면 entrypoint는 실패하지만, 데이터 디렉터리에 이미 쓰인 일부 상태가 남을 수 있습니다. 단순 system-directory 존재 검사는이 partial state를 완료로 오인할 수 있습니다. |
| 이 commit의 보장 | 정상 종료된 첫 실행 뒤에는 hardened root account, application database/user/grant가 존재하고 같은 volume 재사용 시 중복 생성하지 않습니다. |
| 한계와 다음 관련 commit | SIGKILL 등 cleanup trap을 실행하지 못하는 중단 뒤의 수렴, completion marker, staging publication은 보장하지 않습니다. `dc9601f5e670`이 partial persistent state 문제를 staging directory와 verified marker로 교정합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: SIGKILL 등 cleanup trap을 실행하지 못하는 중단 뒤의 수렴, completion marker, staging publication은 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `dc9601f5e670`이 partial persistent state 문제를 staging directory와 verified marker로 교정합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 정상 종료된 첫 실행 뒤에는 hardened root account, application database/user/grant가 존재하고 같은 volume 재사용 시 중복 생성하지 않습니다.

### 3. `d764d066167b` — feat(wordpress): 사이트와 사용자 계정 초기화

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `BOOTSTRAP`, `PERSISTENCE`, `CORE` |
| Source-defined role | Added WordPress filesystem, site, and user convergence. |
| 이전 Thread commit | `e13b0357a21b` |
| 다음 Thread commit | `99c03f54399a` |

#### 원문이 확정한 범위

- **Summary:** Adds idempotent WordPress core, configuration, site, and user initialization.
- **Classification reason:** It introduces the application half of persistent first-run convergence and separates filesystem, database, and account idempotency boundaries, although later commits make the process interruption-safe.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `d764d066167b`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `password/metadata validation`에서 WordPress 초기화 입력의 실패를 PHP-FPM start 전에 차단합니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `DB wait / core-config-site-user branches`에서 파일 존재, config 존재, DB의 site 설치, user 존재를 하나의 조건으로 뭉개지 않습니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `URL update / chown / exec php-fpm`에서 entrypoint 완료 뒤 장기 실행 책임이 PHP-FPM으로 넘어갑니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| d764d066167b | srcs/requirements/wordpress/tools/docker-entrypoint.sh | password/metadata validation | DB, admin, author password의 직접 값과 `_FILE` 입력을 해석하고 URL, title, email, user name 등 필수 metadata를 검증합니다. | WordPress 초기화 입력의 실패를 PHP-FPM start 전에 차단합니다. |
| d764d066167b | srcs/requirements/wordpress/tools/docker-entrypoint.sh | DB wait / core-config-site-user branches | MariaDB 인증이 될 때까지 bounded retry한 뒤 core files, `wp-config.php`, site 설치, author 존재 여부를 각각 별도 query로 판단하고 필요한 단계만 실행합니다. | 파일 존재, config 존재, DB의 site 설치, user 존재를 하나의 조건으로 뭉개지 않습니다. |
| d764d066167b | srcs/requirements/wordpress/tools/docker-entrypoint.sh | URL update / chown / exec php-fpm | canonical HTTPS home/site URL을 맞추고 ownership을 정규화한 뒤 PHP-FPM을 foreground로 `exec`합니다. | entrypoint 완료 뒤 장기 실행 책임이 PHP-FPM으로 넘어갑니다. |

#### 비교 기준

- exact commit diff: `git diff d764d066167b^ d764d066167b -- <path>`
- 이전 Thread 상태와 비교: `git diff e13b0357a21b d764d066167b -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | MariaDB 계정은 생겼지만 WordPress core, private config, site rows, admin/author 계정이 존재하지 않았습니다. |
| 선택한 boundary / decision | filesystem, configuration, site, user를 서로 다른 idempotency 조건으로 검사하고 누락된 항목만 생성하는 application bootstrap을 도입했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `password/metadata validation`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `DB wait / core-config-site-user branches`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `URL update / chown / exec php-fpm` |
| state / ownership / lifecycle 변화 | WordPress entrypoint가 writable web tree와 DB application state를 생성합니다. PHP-FPM은 초기화가 끝난 뒤 serving만 담당합니다. |
| 주요 failure branch | DB wait timeout, WP-CLI 실패, 중간 파일 생성 뒤 종료가 발생하면 일부 filesystem/DB state가 남습니다. 각 단계의 존재 검사는 정상 완료를 완전히 증명하지 않습니다. |
| 이 commit의 보장 | 정상 첫 실행 뒤 core/config/site/admin/author가 준비되고 재시작은 이미 존재하는 항목을 재생성하지 않습니다. |
| 한계와 다음 관련 commit | 중단 시점별 durable completion, config 분리 volume, secret-free long-running service는 아직 보장하지 않습니다. `99c03f54399a`가 이 PHP-FPM에 외부 HTTPS 요청을 연결하고, `dc9601f5e670`이 bootstrap lifecycle을 one-off convergence로 바꿉니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 중단 시점별 durable completion, config 분리 volume, secret-free long-running service는 아직 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `99c03f54399a`가 이 PHP-FPM에 외부 HTTPS 요청을 연결하고, `dc9601f5e670`이 bootstrap lifecycle을 one-off convergence로 바꿉니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 정상 첫 실행 뒤 core/config/site/admin/author가 준비되고 재시작은 이미 존재하는 항목을 재생성하지 않습니다.

### 4. `99c03f54399a` — feat(nginx): PHP 요청을 WordPress로 전달

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `STACK`, `INTEGRATION`, `CORE` |
| Source-defined role | Defined the HTTPS-to-FastCGI request boundary. |
| 이전 Thread commit | `d764d066167b` |
| 다음 Thread commit | `a8b9f693c614` |

#### 원문이 확정한 범위

- **Summary:** Adds TLS policy, static delivery, WordPress front-controller routing, FastCGI forwarding, and a health endpoint.
- **Classification reason:** This defines the actual external request path and the Nginx-to-PHP responsibility boundary, making it significant to understanding how the stack serves WordPress.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `99c03f54399a`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/nginx/conf/nginx.conf`의 `server / listen / ssl_protocols`에서 외부 transport와 TLS termination을 Nginx가 독점합니다.
- `srcs/requirements/nginx/conf/nginx.conf`의 `root / try_files / location ~ \.php$`에서 Nginx는 PHP를 실행하지 않고 service DNS를 통해 PHP-FPM에 요청을 넘깁니다.
- `srcs/requirements/nginx/conf/nginx.conf`의 `/healthz / nginx foreground`에서 transport process의 liveness를 독립적으로 검사할 수 있습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 99c03f54399a | srcs/requirements/nginx/conf/nginx.conf | server / listen / ssl_protocols | IPv4·IPv6 HTTPS listener와 TLS 1.2/1.3, certificate/key path를 정의합니다. | 외부 transport와 TLS termination을 Nginx가 독점합니다. |
| 99c03f54399a | srcs/requirements/nginx/conf/nginx.conf | root / try_files / location ~ \.php$ | shared WordPress document root에서 static file을 찾고, 없으면 front controller로 보내며 PHP request는 `wordpress:9000` FastCGI로 전달합니다. | Nginx는 PHP를 실행하지 않고 service DNS를 통해 PHP-FPM에 요청을 넘깁니다. |
| 99c03f54399a | srcs/requirements/nginx/conf/nginx.conf | /healthz / nginx foreground | application과 분리된 간단한 health endpoint가 있고 Nginx는 daemon-off foreground로 실행됩니다. | transport process의 liveness를 독립적으로 검사할 수 있습니다. |

#### 비교 기준

- exact commit diff: `git diff 99c03f54399a^ 99c03f54399a -- <path>`
- 이전 Thread 상태와 비교: `git diff d764d066167b 99c03f54399a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | WordPress는 PHP-FPM으로 실행 가능했지만 host-facing TLS listener와 static/front-controller/FastCGI routing이 없었습니다. |
| 선택한 boundary / decision | Nginx가 HTTPS·static delivery·routing만 소유하고, PHP execution은 `wordpress:9000`에 위임하도록 경계를 고정했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/requirements/nginx/conf/nginx.conf`의 `server / listen / ssl_protocols`; `srcs/requirements/nginx/conf/nginx.conf`의 `root / try_files / location ~ \.php$`; `srcs/requirements/nginx/conf/nginx.conf`의 `/healthz / nginx foreground` |
| state / ownership / lifecycle 변화 | certificate와 listener는 Nginx image/runtime가 소유합니다. WordPress web files는 공유 경로에서 읽히지만 application write responsibility는 WordPress에 남습니다. |
| 주요 failure branch | FastCGI service가 준비되지 않았거나 파일 path가 일치하지 않으면 5xx가 발생합니다. 이 SHA 자체는 startup dependency나 shared volume을 아직 결합하지 않습니다. |
| 이 commit의 보장 | 외부 HTTPS request가 static 또는 WordPress front controller로 분기되고 PHP는 별도 service에서 실행된다는 책임 분리를 보장합니다. |
| 한계와 다음 관련 commit | Compose DNS, mounted shared files, health-gated startup, persistent DB/data는 보장하지 않습니다. `a8b9f693c614`이 세 서비스를 하나의 topology로 묶고 `75590dedfb3a`가 mount와 health gate를 완성합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: Compose DNS, mounted shared files, health-gated startup, persistent DB/data는 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `a8b9f693c614`이 세 서비스를 하나의 topology로 묶고 `75590dedfb3a`가 mount와 health gate를 완성합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 외부 HTTPS request가 static 또는 WordPress front controller로 분기되고 PHP는 별도 service에서 실행된다는 책임 분리를 보장합니다.

### 5. `a8b9f693c614` — feat(compose): 세 서비스 토폴로지 구성

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **S** |
| Tags | `ARCH`, `STACK`, `CORE` |
| Source-defined role | Assembled the three service responsibilities into the core Compose topology. |
| 이전 Thread commit | `99c03f54399a` |
| 다음 Thread commit | `75590dedfb3a` |

#### 원문이 확정한 범위

- **Summary:** Introduces the three custom services, shared network, sole HTTPS publication, and named persistent resources in Compose.
- **Classification reason:** This is the foundational system topology. Removing it would leave a major gap in explaining the separation of transport, application execution, and durable state.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `a8b9f693c614`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `services: nginx, wordpress, mariadb`에서 개별 이미지를 하나의 project namespace와 network 안에 배치합니다.
- `srcs/docker-compose.yml`의 `network / volumes declarations`에서 영속 자원 이름의 소유자가 Compose project로 이동합니다.
- `srcs/docker-compose.yml`의 `initial topology limitation`에서 토폴로지의 존재와 operational readiness를 소급해 같은 것으로 취급하면 안 됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| a8b9f693c614 | srcs/docker-compose.yml | services: nginx, wordpress, mariadb | 세 custom build context와 service DNS, restart policy, Nginx host port, service dependency를 한 Compose model에 선언합니다. | 개별 이미지를 하나의 project namespace와 network 안에 배치합니다. |
| a8b9f693c614 | srcs/docker-compose.yml | network / volumes declarations | 공유 network와 MariaDB·WordPress용 named volume 이름을 선언합니다. | 영속 자원 이름의 소유자가 Compose project로 이동합니다. |
| a8b9f693c614 | srcs/docker-compose.yml | initial topology limitation | 이 commit에서는 volume 이름을 선언했지만 이후 commit에서 추가되는 실제 mount/environment/health 연결은 아직 없습니다. | 토폴로지의 존재와 operational readiness를 소급해 같은 것으로 취급하면 안 됩니다. |

#### 비교 기준

- exact commit diff: `git diff a8b9f693c614^ a8b9f693c614 -- <path>`
- 이전 Thread 상태와 비교: `git diff 99c03f54399a a8b9f693c614 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### S-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 commit 직전 상태 | 세 이미지와 설정은 존재했지만 build/run/network/port/resource naming을 한 번에 소유하는 system-level model이 없었습니다. |
| 해결하려던 문제 | 서비스가 생성되는 순서는 표현할 수 있지만 실제 데이터 mount와 health-based readiness가 없어 “연결됐다”가 “준비됐다”를 뜻하지 않습니다. |
| 기존 설계가 충분하지 않았던 이유 | 세 이미지와 설정은 존재했지만 build/run/network/port/resource naming을 한 번에 소유하는 system-level model이 없었습니다. 서비스가 생성되는 순서는 표현할 수 있지만 실제 데이터 mount와 health-based readiness가 없어 “연결됐다”가 “준비됐다”를 뜻하지 않습니다. |
| 핵심 결정 | Compose가 Nginx, WordPress, MariaDB의 build context, service DNS, restart, dependency, host port, resource namespace를 통합하도록 했습니다. |
| 주요 caller → callee / producer → consumer | `srcs/docker-compose.yml`의 `services: nginx, wordpress, mariadb`; `srcs/docker-compose.yml`의 `network / volumes declarations`; `srcs/docker-compose.yml`의 `initial topology limitation` |
| authoritative state와 publication boundary | Nginx는 host-facing service, WordPress는 application service, MariaDB는 DB service로 배치됩니다. Compose project가 container/network/volume naming을 소유합니다. 세 서비스의 책임과 호출 방향을 하나의 배포 topology로 고정하고 Nginx만 host port를 갖는 핵심 architecture를 도입합니다. |
| ownership / lifetime / responsibility 변화 | Nginx는 host-facing service, WordPress는 application service, MariaDB는 DB service로 배치됩니다. Compose project가 container/network/volume naming을 소유합니다. |
| failure scenario와 recovery path | 서비스가 생성되는 순서는 표현할 수 있지만 실제 데이터 mount와 health-based readiness가 없어 “연결됐다”가 “준비됐다”를 뜻하지 않습니다. |
| 이 commit이 보장하는 것 | 세 서비스의 책임과 호출 방향을 하나의 배포 topology로 고정하고 Nginx만 host port를 갖는 핵심 architecture를 도입합니다. |
| 아직 보장하지 않는 것 | 이 SHA만으로 persistent mount, application environment, health gate, restart 뒤 데이터 보존을 보장하지 않습니다. |
| 후속 fix / test와 연결 | `75590dedfb3a`가 선언된 named volume을 실제 경로에 mount하고 service-specific health를 dependency 조건으로 연결합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 이 SHA만으로 persistent mount, application environment, health gate, restart 뒤 데이터 보존을 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `75590dedfb3a`가 선언된 named volume을 실제 경로에 mount하고 service-specific health를 dependency 조건으로 연결합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 세 서비스의 책임과 호출 방향을 하나의 배포 topology로 고정하고 Nginx만 host port를 갖는 핵심 architecture를 도입합니다.

### 6. `75590dedfb3a` — feat(compose): 준비 상태에 따라 영속 서비스 연결

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `PERSISTENCE`, `INTEGRATION`, `OPERATIONS` |
| Source-defined role | Connected named volumes, health checks, and dependency readiness. |
| 이전 Thread commit | `a8b9f693c614` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Mounts durable data, adds service health checks, and gates startup on dependency health.
- **Classification reason:** It turns the service list into a stateful, readiness-aware stack and establishes important persistence and lifecycle integration across all three containers.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `75590dedfb3a`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `volumes mounts`에서 container filesystem이 아니라 named volume이 authoritative state가 됩니다.
- `srcs/docker-compose.yml`의 `healthcheck blocks`에서 단순 container-created 상태와 process/application readiness를 구분합니다.
- `srcs/docker-compose.yml`의 `depends_on.condition: service_healthy`에서 request path의 consumer가 producer 준비 전에 시작되는 race를 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 75590dedfb3a | srcs/docker-compose.yml | volumes mounts | MariaDB data directory, WordPress writable web tree를 named volume에 mount하고 Nginx는 WordPress document를 읽는 쪽으로 연결합니다. | container filesystem이 아니라 named volume이 authoritative state가 됩니다. |
| 75590dedfb3a | srcs/docker-compose.yml | healthcheck blocks | MariaDB, WordPress/PHP-FPM, Nginx에 각 service-specific command와 interval/retry/start-period를 설정합니다. | 단순 container-created 상태와 process/application readiness를 구분합니다. |
| 75590dedfb3a | srcs/docker-compose.yml | depends_on.condition: service_healthy | WordPress는 healthy MariaDB 뒤에, Nginx는 healthy WordPress 뒤에 시작하도록 gate를 둡니다. | request path의 consumer가 producer 준비 전에 시작되는 race를 줄입니다. |

#### 비교 기준

- exact commit diff: `git diff 75590dedfb3a^ 75590dedfb3a -- <path>`
- 이전 Thread 상태와 비교: `git diff a8b9f693c614 75590dedfb3a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | `a8b9f693c614`은 topology와 resource names만 만들었고 실제 state mount와 readiness semantics가 불완전했습니다. |
| 선택한 boundary / decision | named volume mount와 health-gated dependency를 추가해 topology를 stateful application system으로 바꿨습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/docker-compose.yml`의 `volumes mounts`; `srcs/docker-compose.yml`의 `healthcheck blocks`; `srcs/docker-compose.yml`의 `depends_on.condition: service_healthy` |
| state / ownership / lifecycle 변화 | MariaDB volume은 relational state, WordPress volume은 application files를 소유합니다. 컨테이너는 교체 가능하고 health command가 다음 service start 허용 여부를 결정합니다. |
| 주요 failure branch | health command가 현재 process에 응답한다는 것은 first-run initialization이 interruption-safe하다는 뜻이 아닙니다. 당시 health는 후속 completion marker 수준까지 강하지 않습니다. |
| 이 commit의 보장 | 서비스별 state가 named volume에 남고, dependency는 container creation이 아니라 health success를 기다립니다. |
| 한계와 다음 관련 commit | abrupt bootstrap interruption 수렴, volume identity의 실제 유지, end-to-end data path는 후속 tests 없이는 증명하지 않습니다. `fb1a689cf969`이 restart/recreate 뒤 volume identity와 값 보존을 검증하고 `dc9601f5e670`이 health에 durable marker를 결합합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: abrupt bootstrap interruption 수렴, volume identity의 실제 유지, end-to-end data path는 후속 tests 없이는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `fb1a689cf969`이 restart/recreate 뒤 volume identity와 값 보존을 검증하고 `dc9601f5e670`이 health에 durable marker를 결합합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 서비스별 state가 named volume에 남고, dependency는 container creation이 아니라 health success를 기다립니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 호스트 포트를 게시하는 서비스는 Nginx뿐입니다. | a8b9f693c614 | 75590dedfb3a | 8c9b5b9adef2 | Compose의 `ports`는 Nginx service에만 있고 runtime e2e는 해당 HTTPS endpoint를 사용합니다. |
| MariaDB 영속 상태는 이미지 계층이 아니라 mounted data volume이 권위자입니다. | f8ec9621725c | a8b9f693c614, 75590dedfb3a | fb1a689cf969 | Dockerfile은 빈 data path만 준비하고 Compose가 named volume을 mount하며 persistence scenario가 동일 volume ID와 row를 재검사합니다. |
| WordPress는 애플리케이션 상태를 쓸 수 있고 Nginx는 공유 문서를 읽는 경계에 놓입니다. | d764d066167b | 75590dedfb3a | 8c9b5b9adef2, fb1a689cf969 | WordPress entrypoint가 files/site/users를 만들고 Nginx는 동일 document root에서 static/FastCGI routing만 수행합니다. |
| 준비 상태는 단순 container creation이 아니라 service-specific health로 판단합니다. | 75590dedfb3a | dc9601f5e670에서 marker까지 강화 | 2bf6d3f11337 | Compose healthchecks와 `condition: service_healthy`, 이후 marker+process probe가 start gate를 구성합니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: `a8b9f693c614` 전에는 서비스별 이미지가 있었지만 host-facing service, shared network, named-volume mount가 하나의 시스템으로 결합되지 않았고, `75590dedfb3a` 전에는 container creation과 service readiness가 구분되지 않았습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: Compose의 sole port publication, service DNS, read/write mount mode, service-specific health check와 `service_healthy` dependency가 invariant를 관측 가능한 설정으로 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `8c9b5b9adef2`의 전체 request/data path와 `fb1a689cf969`의 restart/recreate persistence가 후속 회귀를 검출합니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| 개별 컨테이너만 존재하고 시스템 경계가 없음 | a8b9f693c614가 세 service topology와 유일한 host-facing Nginx를 확정 | 8c9b5b9adef2가 HTTPS→WordPress→MariaDB data path를 검증 | root cause는 buildable image와 integrated system을 같은 것으로 본 데 있습니다. |
| entrypoint의 existence check가 abrupt interruption 뒤 partial state를 완료로 오인 | dc9601f5e670가 staging, verified marker, one-off bootstrap으로 교정 | 2bf6d3f11337가 durable stage마다 SIGKILL 후 재실행 수렴을 검증 | graceful cleanup에 의존하지 않고 durable publication order로 완료를 판정합니다. |
| 컨테이너 생성 순서를 readiness로 오인 | 75590dedfb3a가 service health gate를 추가 | 후속 runtime harness가 live health와 integrated request path를 검사 | dependency는 process/application probe가 성공해야 해제됩니다. |

### 직접 재구성할 chain

```text
기존 가정: 개별 container가 존재하면 stack이 구성됐다고 볼 수 있다는 가정
  → 실제 failure 또는 위험: host publication, service routing, durable mount, readiness dependency가 분리되어 system-level 보장이 없었습니다.
  → root cause: 개별 Dockerfile과 entrypoint는 다른 서비스의 network·volume·startup state를 소유하지 않습니다.
  → 수정된 invariant / decision: Nginx만 host-facing transport를 소유하고 WordPress와 MariaDB는 service DNS와 named volume으로 연결되며 dependency는 health를 기준으로 합니다.
  → 해당 SHA의 실제 수정 코드: `a8b9f693c614`의 Compose topology와 `75590dedfb3a`의 volume/health/dependency blocks
  → failure injection 또는 regression test: `8c9b5b9adef2`, `fb1a689cf969` runtime scenarios
  → 증명된 보장 / 남은 비보장: HTTPS→FastCGI→WordPress→MariaDB 통합 경로와 container 교체 뒤 volume state 보존은 검증하지만 abrupt bootstrap convergence는 Thread 2가 보강합니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Nginx | 독립 이미지/프로세스 | 유일한 host-facing TLS 및 static/FastCGI routing 책임 | 99c03f54399a 설정과 a8b9f693c614/75590dedfb3a Compose port | PHP 실행이나 DB state를 소유하지 않습니다. |
| WordPress | 독립 PHP-FPM 이미지와 초기 entrypoint | application execution과 writable application state | d764d066167b entrypoint, PHP-FPM pool, WordPress volume | site/files/users를 쓰고 FastCGI request를 처리합니다. |
| MariaDB | 독립 DB image | 관계형 durable state와 DB/account 소유 | e13b0357a21b bootstrap, 75590dedfb3a data mount | application query에 필요한 DB와 grant를 유지합니다. |
| Compose | 서비스별 수동 실행 | network, service DNS, port, volume namespace, health dependency 통합 소유 | a8b9f693c614 및 75590dedfb3a의 `docker-compose.yml` | 컨테이너 교체와 resource naming을 project 단위로 관리합니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** The thread progresses from individually runnable containers to one stateful application system. The decisive step is not the existence of three images but the Compose responsibility boundary: Nginx owns external transport, WordPress owns application execution, and MariaDB owns durable relational state. Health-gated dependencies and mounted volumes then make the topology operationally meaningful rather than merely connected.
- 최종 authoritative state와 owner: MariaDB named volume이 relational state, WordPress named volume이 application files를 소유하며 Nginx는 state owner가 아닙니다.
- 정상 실행의 entry point와 완료 조건: Compose 또는 후속 `start_stack.py`가 서비스를 시작하고, MariaDB→WordPress→Nginx health gate가 모두 성공하면 정상 완료입니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: 이 Thread의 초기 entrypoint만으로는 abrupt interruption 수렴이 충분하지 않으며 Thread 2의 marker/staging bootstrap이 재시도 조건을 정의합니다.
- 이 Thread가 다른 Thread에 제공하는 전제: Thread 2의 bootstrap, Thread 3의 runtime/persistence test, Thread 4~6의 management transaction이 사용할 기본 topology와 state ownership을 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: 백업의 atomicity, restore rollback, credential compensation, supply-chain identity는이 Thread만으로 증명하지 않습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | 호스트 HTTPS 요청 | 99c03f54399a `nginx.conf` listener | Nginx가 TLS를 종료하고 server block을 선택합니다. | listener/certificate 오류면 Nginx health가 실패하고 downstream request가 열리지 않습니다. |
| 2 | static/front controller 분기 | 99c03f54399a `try_files` | 실제 file은 static으로, 나머지는 `index.php`로 이동합니다. | 잘못된 root/path는 404 또는 FastCGI 오류로 드러납니다. |
| 3 | FastCGI 전달 | 99c03f54399a `fastcgi_pass wordpress:9000` | service DNS가 WordPress PHP-FPM에 request를 전달합니다. | WordPress health가 실패하면 75590dedfb3a의 gate가 Nginx start를 막습니다. |
| 4 | WordPress DB 접근 | d764d066167b config 생성과 MariaDB service name | application credential로 MariaDB service에 연결합니다. | DB auth/readiness failure는 bounded wait 또는 WP-CLI failure로 종료합니다. |
| 5 | 상태 저장 | 75590dedfb3a volume mounts | DB rows와 WordPress files가 named volume에 남습니다. | 컨테이너 교체는 volume을 제거하지 않는 한 state owner를 바꾸지 않습니다. |
| 6 | readiness gate | 75590dedfb3a healthcheck/depends_on | 각 producer health가 성공해야 다음 consumer가 시작됩니다. | timeout/retry 소진은 startup failure이며 후속 bootstrap design이 partial state를 판정합니다. |

### 학습자의 최종 설명

> 이미지 세 개를 만든 것만으로 계층형 시스템은 완성되지 않았습니다. MariaDB image가 durable DB 경로를 준비하고 first-run account를 만들며, WordPress가 filesystem·site·users를 수렴시키고, Nginx가 TLS와 FastCGI routing만 맡은 뒤에야 책임이 나뉩니다. `a8b9f693c614`은 이 책임을 하나의 Compose namespace에 모았지만 mount와 readiness는 부족했습니다. `75590dedfb3a`에서 named volume과 service-specific health gate가 추가되어 컨테이너 교체 가능한 실행 단위와 권위 있는 persistent state가 분리됐습니다. 다만 초기 entrypoint는 abrupt interruption 뒤 partial state를 완전하게 판정하지 못하므로, Thread 2의 staged one-off bootstrap이 이 architecture의 실제 재시도 안전성을 완성합니다.

## 학습 완료 자가 점검

- [x] 세 이미지의 존재와 세 계층 architecture의 성립을 같은 것으로 설명하지 않았습니까?
- [x] Nginx가 PHP를 실행한다고 잘못 설명하지 않았습니까?
- [x] WordPress data volume과 MariaDB data volume의 상태 종류를 구분했습니까?
- [x] 초기 health check가 completion marker까지 포함한다고 소급해서 쓰지 않았습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 01-readiness-aware-three-tier-stack.md =====

===== BEGIN FILE: 02-convergent-one-off-bootstrap.md =====
# Thread 2 — From runtime secret mounts to convergent one-off bootstrap

## Thread 목표

Compose secret mount 기반의 초기 모델이 host-side secret validation, per-project lock, one-off bootstrap, completion marker를 이용한 수렴형 초기화로 바뀌는 핵심 lifecycle 교정을 추적합니다.

**Source significance**

> The earlier `_FILE` and Compose-secret model reduced direct environment exposure but still attached credential material to service startup. The later architecture resolves secrets on the host while holding the project lock, sends only required values to short-lived bootstrap containers, and lets long-running services start from verified persistent state. The final SIGKILL scenario is important because it validates the design's intended convergence after process death, not only after controlled errors.

## 이 Thread를 이해하기 위한 핵심 질문

- 초기 `_FILE` 모델은 environment 노출을 줄였지만 어떤 steady-state 노출과 partial-state 위험을 남겼습니까?
- host secret path가 신뢰 가능한 입력이 되기 위해 어떤 descriptor/stat/permission 검사가 필요합니까?
- 왜 management lock의 granularity가 Compose project name입니까?
- MariaDB staging publication과 WordPress completion marker는 어떤 순서 보장을 만듭니까?
- process readiness와 durable initialization completion은 health check에서 어떻게 결합됩니까?
- SIGKILL 테스트는 graceful trap 기반 테스트와 무엇을 다르게 증명합니까?

## 완료 기준

- 기존 runtime secret mount 구조와 최종 bootstrap-only secret 전달 구조를 비교했습니다.
- `project_operation_lock`의 private path, ownership, no-follow, non-blocking flock 계약을 실제 코드로 확인했습니다.
- MariaDB와 WordPress의 staging/marker/publish 순서를 각 SHA의 entrypoint와 orchestrator로 복원했습니다.
- static contract와 SIGKILL runtime regression이 각각 증명하는 범위를 분리했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `916391b9f8db` | feat(secrets): 비밀번호를 비밀 파일에서 로드 | **B** | `SECRETS`<br>`RISK` | Moved passwords out of ordinary environment values into Compose secret files. |
| 2 | `486ffb5c65aa` | refactor(secrets): 비밀 파일 로딩 경계 공통화 | **A** | `SECRETS`<br>`RISK`<br>`ARCH` | Centralized hardened host secret-file resolution and reading. |
| 3 | `e77c6f151b07` | refactor(runtime): 프로젝트 관리 작업 잠금 공통화 | **A** | `RECOVERY`<br>`OPERATIONS`<br>`RISK` | Established per-project management-operation serialization. |
| 4 | `dc9601f5e670` | fix(init): 중단된 단계별 초기화를 수렴 | **S** | `ARCH`<br>`BOOTSTRAP`<br>`RECOVERY` | Replaced steady-state secret mounts and one-shot initialization with staged one-off bootstrap. |
| 5 | `3beebbfc4723` | test(init): 단계별 초기화 계약 검사 | **B** | `TEST`<br>`BOOTSTRAP` | Added a source contract for completion markers and staged recovery. |
| 6 | `2bf6d3f11337` | test(init): 안정 단계별 초기화 중단 복구 검증 | **A** | `TEST`<br>`BOOTSTRAP`<br>`RECOVERY` | Killed bootstrap containers at every durable stage and proved rerun convergence. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `916391b9f8db` — feat(secrets): 비밀번호를 비밀 파일에서 로드

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `SECRETS`, `RISK` |
| Source-defined role | Moved passwords out of ordinary environment values into Compose secret files. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `486ffb5c65aa` |

#### 원문이 확정한 범위

- **Summary:** Replaces password environment values with Compose secret files and `_FILE` inputs.
- **Classification reason:** This is a meaningful intermediate security improvement, but the later one-off bootstrap architecture removes steady-state secret mounts and becomes the durable project boundary.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `916391b9f8db`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `.env.example`의 `*_PASSWORD_FILE variables`에서 operator가 값 대신 file source를 구성합니다.
- `srcs/docker-compose.yml`의 `secrets / service attachments`에서 credential material이 일반 environment value가 아니라 file mount로 전달됩니다.
- `srcs/docker-compose.yml`의 `*_PASSWORD_FILE=/run/secrets/... / healthcheck`에서 장기 실행 service도 steady state에서 secret mount를 계속 보유합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 916391b9f8db | .env.example | *_PASSWORD_FILE variables | 공개 예제 환경에서 password literal을 제거하고 host secret file path를 받도록 바꿉니다. | operator가 값 대신 file source를 구성합니다. |
| 916391b9f8db | srcs/docker-compose.yml | secrets / service attachments | 네 secret source를 선언하고 MariaDB와 WordPress가 필요한 subset을 `/run/secrets`로 mount합니다. | credential material이 일반 environment value가 아니라 file mount로 전달됩니다. |
| 916391b9f8db | srcs/docker-compose.yml | *_PASSWORD_FILE=/run/secrets/... / healthcheck | entrypoint `_FILE` 변수와 MariaDB health command가 mounted root secret을 읽습니다. | 장기 실행 service도 steady state에서 secret mount를 계속 보유합니다. |

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Moved passwords out of ordinary environment values into Compose secret files. |
| 핵심 input / output / state | host source file은 Compose가 mount하고 long-running MariaDB/WordPress container가 실행 내내 `/run/secrets`를 읽을 수 있습니다. |
| 변경된 directive / helper / command | `.env.example`의 `*_PASSWORD_FILE variables`; `srcs/docker-compose.yml`의 `secrets / service attachments`; `srcs/docker-compose.yml`의 `*_PASSWORD_FILE=/run/secrets/... / healthcheck` |
| immediate failure 또는 boundary | environment 노출은 줄지만 runtime container compromise나 진단 명령이 mounted secret에 접근할 수 있고, bootstrap과 serving lifecycle이 여전히 결합됩니다. |
| 다음 commit에 넘긴 한계 | host file의 owner/mode/symlink 안전성, project operation serialization, secret-free steady-state container는 보장하지 않습니다. `486ffb5c65aa`가 host secret read를 hardened shared boundary로 만들고 `dc9601f5e670`이 runtime mount 자체를 제거합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: host file의 owner/mode/symlink 안전성, project operation serialization, secret-free steady-state container는 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `486ffb5c65aa`가 host secret read를 hardened shared boundary로 만들고 `dc9601f5e670`이 runtime mount 자체를 제거합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: password literal이 일반 environment/config에 직접 놓이지 않고 `_FILE` contract로 소비된다는 중간 보장을 제공합니다.

### 2. `486ffb5c65aa` — refactor(secrets): 비밀 파일 로딩 경계 공통화

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `SECRETS`, `RISK`, `ARCH` |
| Source-defined role | Centralized hardened host secret-file resolution and reading. |
| 이전 Thread commit | `916391b9f8db` |
| 다음 Thread commit | `e77c6f151b07` |

#### 원문이 확정한 범위

- **Summary:** Adds hardened secret-file reading, rendered secret-path resolution, environment extraction, and stdin payload construction.
- **Classification reason:** This centralizes a critical trust boundary used by startup, backup, restore, rotation, and diagnostics, but it supports rather than alone defines the project-wide lifecycle architecture.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `486ffb5c65aa`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_runtime.py`의 `secret_source_paths`에서 secret path 해석을 startup/backup/restore/rotation/diagnostics가 공유할 수 있게 합니다.
- `tools/stack_runtime.py`의 `read_private_secret`에서 pathname 검사 뒤 교체되는 TOCTOU window를 descriptor 기반 검사로 줄입니다.
- `tools/stack_runtime.py`의 `load_secret_values / secret_payload / service_environment`에서 credential 전달의 producer/consumer 형식이 공통화됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 486ffb5c65aa | tools/stack_runtime.py | secret_source_paths | rendered Compose의 `x-secret-files` metadata에서 source path를 resolve하고 canonical path가 서로 겹치지 않는지 검사합니다. | secret path 해석을 startup/backup/restore/rotation/diagnostics가 공유할 수 있게 합니다. |
| 486ffb5c65aa | tools/stack_runtime.py | read_private_secret | `O_NOFOLLOW`로 열고 descriptor `fstat`으로 regular file, single link, current owner, `0600`, parent directory 안전성을 검사한 뒤 bounded single-line value를 읽습니다. | pathname 검사 뒤 교체되는 TOCTOU window를 descriptor 기반 검사로 줄입니다. |
| 486ffb5c65aa | tools/stack_runtime.py | load_secret_values / secret_payload / service_environment | 검증된 네 값을 mapping으로 만들고 one-off command가 stdin으로 받을 payload와 non-secret environment를 분리합니다. | credential 전달의 producer/consumer 형식이 공통화됩니다. |

#### 비교 기준

- exact commit diff: `git diff 486ffb5c65aa^ 486ffb5c65aa -- <path>`
- 이전 Thread 상태와 비교: `git diff 916391b9f8db 486ffb5c65aa -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | `916391b9f8db`은 file mount를 사용했지만 host source가 symlink, 잘못된 owner/mode, hard link, multiline인지 공통으로 검증하지 않았습니다. |
| 선택한 boundary / decision | rendered Compose metadata를 판단 기준로 삼고, path resolution부터 descriptor inspection과 content policy까지 한 module에 모았습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/stack_runtime.py`의 `secret_source_paths`; `tools/stack_runtime.py`의 `read_private_secret`; `tools/stack_runtime.py`의 `load_secret_values / secret_payload / service_environment` |
| state / ownership / lifecycle 변화 | host management process가 secret file을 읽고 즉시 memory mapping을 소유합니다. 이후 caller는 raw path를 다시 열지 않고 검증된 mapping/payload를 사용합니다. |
| 주요 failure branch | unsafe type, owner, mode, link count, parent permission, duplicate canonical path, size/multiline/password-shape 위반은 mutation 전에 실패합니다. |
| 이 commit의 보장 | 동일한 hardened secret-input contract를 여러 management operation이 재사용할 수 있습니다. |
| 한계와 다음 관련 commit | 동시 operation이 같은 project state와 secret generation을 바꾸는 race는 아직 막지 않습니다. `e77c6f151b07`가 project-scoped lock을 추가하고 `dc9601f5e670`의 startup이 lock 안에서 이 helper를 호출합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 동시 operation이 같은 project state와 secret generation을 바꾸는 race는 아직 막지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `e77c6f151b07`가 project-scoped lock을 추가하고 `dc9601f5e670`의 startup이 lock 안에서 이 helper를 호출합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 동일한 hardened secret-input contract를 여러 management operation이 재사용할 수 있습니다.

### 3. `e77c6f151b07` — refactor(runtime): 프로젝트 관리 작업 잠금 공통화

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `RECOVERY`, `OPERATIONS`, `RISK` |
| Source-defined role | Established per-project management-operation serialization. |
| 이전 Thread commit | `486ffb5c65aa` |
| 다음 Thread commit | `dc9601f5e670` |

#### 원문이 확정한 범위

- **Summary:** Adds a per-user, per-project non-blocking advisory lock in a private fixed directory.
- **Classification reason:** Serializing management operations is a critical concurrency invariant across later startup, backup, restore, and rotation flows, though the change is a focused mechanism rather than the whole project architecture.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `e77c6f151b07`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_runtime.py`의 `project_operation_lock`에서 TMPDIR 변경과 무관한 project identity를 사용합니다.
- `tools/stack_runtime.py`의 `os.open / flock LOCK_EX|LOCK_NB`에서 같은 project의 동시 management operation은 대기하지 않고 명시적으로 충돌 실패합니다.
- `tools/stack_runtime.py`의 `context-manager cleanup`에서 lock lifetime이 with-block의 management transaction과 일치합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| e77c6f151b07 | tools/stack_runtime.py | project_operation_lock | current user 전용 fixed private directory를 검사하고 project name을 opaque filename으로 변환해 lock file을 엽니다. | TMPDIR 변경과 무관한 project identity를 사용합니다. |
| e77c6f151b07 | tools/stack_runtime.py | os.open / flock LOCK_EX\|LOCK_NB | no-follow·owner/mode/type 검사를 거친 descriptor에 non-blocking exclusive advisory lock을 잡습니다. | 같은 project의 동시 management operation은 대기하지 않고 명시적으로 충돌 실패합니다. |
| e77c6f151b07 | tools/stack_runtime.py | context-manager cleanup | 예외 여부와 관계없이 flock release와 descriptor close가 수행됩니다. | lock lifetime이 with-block의 management transaction과 일치합니다. |

#### 비교 기준

- exact commit diff: `git diff e77c6f151b07^ e77c6f151b07 -- <path>`
- 이전 Thread 상태와 비교: `git diff 486ffb5c65aa e77c6f151b07 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | secret read helper가 안전해도 두 startup/backup/restore/rotation이 같은 Compose project를 동시에 변경하면 각자의 사전 조건이 무효화될 수 있었습니다. |
| 선택한 boundary / decision | lock identity를 project name에 맞추고 management operation 전체를 non-blocking exclusive lock으로 감쌌습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/stack_runtime.py`의 `project_operation_lock`; `tools/stack_runtime.py`의 `os.open / flock LOCK_EX\|LOCK_NB`; `tools/stack_runtime.py`의 `context-manager cleanup` |
| state / ownership / lifecycle 변화 | lock descriptor를 보유한 host process가 transaction 동안 project mutation 권한을 소유합니다. 다른 project name은 별도 lock이므로 병렬 실행 가능합니다. |
| 주요 failure branch | 같은 project lock contention은 즉시 domain error가 됩니다. process death 시 OS가 descriptor를 닫아 lock을 회수합니다. |
| 이 commit의 보장 | 동일 project를 변경하는 cooperating management code가 겹치지 않는다는 concurrency invariant를 제공합니다. |
| 한계와 다음 관련 commit | Docker 외부에서 lock을 무시하는 수동 명령이나 non-cooperating process는 막지 못합니다. `dc9601f5e670`이 startup orchestration에 lock을 적용하고 이후 backup/restore/rotation이 같은 mechanism을 공유합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: Docker 외부에서 lock을 무시하는 수동 명령이나 non-cooperating process는 막지 못합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `dc9601f5e670`이 startup orchestration에 lock을 적용하고 이후 backup/restore/rotation이 같은 mechanism을 공유합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 동일 project를 변경하는 cooperating management code가 겹치지 않는다는 concurrency invariant를 제공합니다.

### 4. `dc9601f5e670` — fix(init): 중단된 단계별 초기화를 수렴

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **S** |
| Tags | `ARCH`, `BOOTSTRAP`, `RECOVERY` |
| Source-defined role | Replaced steady-state secret mounts and one-shot initialization with staged one-off bootstrap. |
| 이전 Thread commit | `e77c6f151b07` |
| 다음 Thread commit | `3beebbfc4723` |

#### 원문이 확정한 범위

- **Summary:** Replaces in-container first-run setup with locked, staged one-off bootstrap orchestration, completion markers, and convergent restart behavior.
- **Classification reason:** This is the decisive lifecycle redesign: it removes runtime secret mounts, separates configuration state, survives interrupted initialization, and determines how persistent services are safely brought to readiness.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `dc9601f5e670`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/start_stack.py`의 `run_action / staged startup`에서 startup 전체가 한 project transaction으로 직렬화됩니다.
- `srcs/docker-compose.yml`의 `one-off bootstrap commands / runtime service blocks`에서 secret lifetime이 short-lived bootstrap process로 제한됩니다.
- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `staging data directory / marker / rename`에서 최종 DB path는 verified complete state만 보이도록 게시됩니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core/config/site/users stages / marker`에서 partial stage는 marker 부재로 다음 실행에서 다시 수렴합니다.
- `srcs/docker-compose.yml`의 `healthcheck marker + process probe`에서 durable completion과 live readiness가 동시에 충족돼야 healthy입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| dc9601f5e670 | tools/start_stack.py | run_action / staged startup | Compose path와 project를 검증하고 `project_operation_lock` 안에서 host secret을 읽은 뒤 MariaDB bootstrap→DB service health→WordPress bootstrap→application/Nginx start 순서로 실행합니다. | startup 전체가 한 project transaction으로 직렬화됩니다. |
| dc9601f5e670 | srcs/docker-compose.yml | one-off bootstrap commands / runtime service blocks | credential은 bootstrap container stdin으로만 전달되고 long-running service block에서는 password environment와 `/run/secrets` mount가 제거됩니다. | secret lifetime이 short-lived bootstrap process로 제한됩니다. |
| dc9601f5e670 | srcs/requirements/mariadb/tools/docker-entrypoint.sh | staging data directory / marker / rename | private staging에서 system tables와 accounts를 만들고 인증 검증·sync·completion marker를 끝낸 뒤 최종 data directory로 rename합니다. | 최종 DB path는 verified complete state만 보이도록 게시됩니다. |
| dc9601f5e670 | srcs/requirements/wordpress/tools/docker-entrypoint.sh | core/config/site/users stages / marker | core files, private config volume, site, users와 password를 단계별로 수렴·검증하고 마지막에 marker를 atomic replace합니다. | partial stage는 marker 부재로 다음 실행에서 다시 수렴합니다. |
| dc9601f5e670 | srcs/docker-compose.yml | healthcheck marker + process probe | MariaDB는 marker/socket/PID, WordPress는 marker/FastCGI ping을 함께 요구합니다. | durable completion과 live readiness가 동시에 충족돼야 healthy입니다. |

#### 비교 기준

- exact commit diff: `git diff dc9601f5e670^ dc9601f5e670 -- <path>`
- 이전 Thread 상태와 비교: `git diff e77c6f151b07 dc9601f5e670 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | directory/file existence를 정상 초기화 완료로 간주했습니다. |
| 실제 failure 또는 위험 | abrupt process death가 cleanup 없이 partial persistent state를 남기며 다음 start가 이를 재사용할 수 있습니다. |
| root cause | initialization과 long-running serving이 같은 entrypoint/lifetime에 결합되고 완료 publication boundary가 없었습니다. |
| 수정된 invariant / decision | short-lived one-off bootstrap이 verified marker 또는 atomic data-directory rename을 게시한 뒤에만 runtime service를 시작합니다. |
| 실제 수정 코드 | `tools/start_stack.py`의 `run_action / staged startup`; `srcs/docker-compose.yml`의 `one-off bootstrap commands / runtime service blocks`; `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `staging data directory / marker / rename`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core/config/site/users stages / marker`; `srcs/docker-compose.yml`의 `healthcheck marker + process probe` |
| 변경된 ordering / ownership / lifecycle | host code가 secret source와 operation order를 소유합니다. bootstrap container는 persistent volume을 한 번 수렴시키고 종료합니다. long-running service는 verified marker가 있는 state를 serving만 합니다. |
| 이 fix가 보장하는 것 | 장기 service의 secret-free boundary, same-project serialization, MariaDB atomic data publication, WordPress verified marker, marker+process readiness를 보장합니다. |
| 아직 보장하지 않는 것 | filesystem/DB 자체의 하드웨어 crash durability나 외부 수동 mutation까지 원자화하지는 않습니다. 실제 SIGKILL 수렴은 test commit이 별도로 증명해야 합니다. |
| 연결되는 regression test | static ordering contract와 durable-stage SIGKILL regression이 이 corrected invariant를 고정합니다. `3beebbfc4723`이 source contract를 고정하고 `2bf6d3f11337`이 durable stage마다 bootstrap process를 SIGKILL해 재실행 수렴을 검증합니다. |

#### S-level state transition 기록

| 단계 | 학습자 기록 |
| --- | --- |
| correction 전 authoritative state | 초기 entrypoint는 long-running service가 secret을 mount한 채 빈/존재 조건으로 즉석 초기화했습니다. SIGKILL 뒤 partial directory가 남으면 다음 start가 완료로 오인될 수 있었습니다. |
| partial / ambiguous state 종류 | abrupt process death가 cleanup 없이 partial persistent state를 남기며 다음 start가 이를 재사용할 수 있습니다. |
| publication 또는 commit boundary | host orchestrator, project lock, stdin-only one-off bootstrap, staging publication, completion marker, private config volume로 startup lifecycle을 재설계했습니다. |
| rollback / compensation 진입 조건 | 어느 stage에서 종료돼도 최종 marker/rename 전 state는 완료로 공개되지 않습니다. 재실행은 existing verified 부분을 검사하고 누락된 stage를 다시 수행합니다. one-off 실패는 long-running service start를 허용하지 않습니다. |
| recovery 중 보호되는 invariant | host code가 secret source와 operation order를 소유합니다. bootstrap container는 persistent volume을 한 번 수렴시키고 종료합니다. long-running service는 verified marker가 있는 state를 serving만 합니다. |
| 성공 endpoint | 장기 service의 secret-free boundary, same-project serialization, MariaDB atomic data publication, WordPress verified marker, marker+process readiness를 보장합니다. |
| 실패 endpoint | filesystem/DB 자체의 하드웨어 crash durability나 외부 수동 mutation까지 원자화하지는 않습니다. 실제 SIGKILL 수렴은 test commit이 별도로 증명해야 합니다. |
| 후속 regression evidence | `3beebbfc4723`이 source contract를 고정하고 `2bf6d3f11337`이 durable stage마다 bootstrap process를 SIGKILL해 재실행 수렴을 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: filesystem/DB 자체의 하드웨어 crash durability나 외부 수동 mutation까지 원자화하지는 않습니다. 실제 SIGKILL 수렴은 test commit이 별도로 증명해야 합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `3beebbfc4723`이 source contract를 고정하고 `2bf6d3f11337`이 durable stage마다 bootstrap process를 SIGKILL해 재실행 수렴을 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 장기 service의 secret-free boundary, same-project serialization, MariaDB atomic data publication, WordPress verified marker, marker+process readiness를 보장합니다.

### 5. `3beebbfc4723` — test(init): 단계별 초기화 계약 검사

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `TEST`, `BOOTSTRAP` |
| Source-defined role | Added a source contract for completion markers and staged recovery. |
| 이전 Thread commit | `dc9601f5e670` |
| 다음 Thread commit | `2bf6d3f11337` |

#### 원문이 확정한 범위

- **Summary:** Adds static assertions for staged MariaDB and WordPress bootstrap markers and recovery structure.
- **Classification reason:** The checks protect the new design at a source-pattern level, but they do not yet prove real interruption recovery.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `3beebbfc4723`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `bootstrap source-order validation`에서 후속 refactor가 marker를 너무 일찍 게시하는 회귀를 정적으로 막습니다.
- `tests/validate_stack.py`의 `runtime secret boundary checks`에서 one-off bootstrap-only secret boundary를 source contract로 고정합니다.
- `tests/validate_stack.py`의 `health marker patterns`에서 readiness semantics가 단순 liveness로 약화되지 않게 합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3beebbfc4723 | tests/validate_stack.py | bootstrap source-order validation | MariaDB staging/marker/rename과 WordPress files/config/site/users/password verification/marker의 상대적 source order를 검사합니다. | 후속 refactor가 marker를 너무 일찍 게시하는 회귀를 정적으로 막습니다. |
| 3beebbfc4723 | tests/validate_stack.py | runtime secret boundary checks | runtime service block의 `/run/secrets` mount와 password-bearing environment를 거부합니다. | one-off bootstrap-only secret boundary를 source contract로 고정합니다. |
| 3beebbfc4723 | tests/validate_stack.py | health marker patterns | healthcheck가 completion marker와 live socket/FastCGI probe를 함께 요구하는지 검사합니다. | readiness semantics가 단순 liveness로 약화되지 않게 합니다. |

#### 비교 기준

- exact commit diff: `git diff 3beebbfc4723^ 3beebbfc4723 -- <path>`
- 이전 Thread 상태와 비교: `git diff dc9601f5e670 3beebbfc4723 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | completion marker는 모든 필수 persistent state 검증 뒤에만 게시되고 runtime service는 secret을 mount하지 않습니다. |
| 재현하는 failure / boundary | 후속 source 변경이 marker를 앞당기거나 `/run/secrets`를 runtime service에 재도입하는 경계입니다. |
| test technique | static source contract와 Compose block pattern/order 검사 |
| fixture와 failure injection | repository의 Dockerfile/entrypoint/Compose source 자체가 fixture이며 별도 runtime failure injection은 없습니다. |
| 실제 통과하는 production path | `tests/validate_stack.py`가 source files를 읽어 marker·rename·health·secret pattern을 검사합니다. |
| 핵심 assertion | 필수 pattern의 존재, 금지 pattern의 부재, publication order의 단조 증가를 확인합니다. |
| 이 테스트가 증명하는 것 | 해당 architecture가 source에 표현되어 있고 명백한 순서 회귀가 없음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | shell control flow의 모든 branch, Docker runtime behavior, fsync 효과, SIGKILL 수렴은 증명하지 않습니다. |
| 성격 | source contract regression test |
| 막는 후속 regression | marker-before-verification, runtime secret mount, marker 없는 healthcheck 회귀를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: SIGKILL 뒤 실제 volume이 수렴하거나 Docker가 health gate를 적용한다는 runtime 사실은 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `2bf6d3f11337`이 동일 invariant를 live Docker와 SIGKILL로 보강합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: architecture를 구성하는 marker order, secret-free runtime, health contract가 source에 존재함을 빠르게 증명합니다.

### 6. `2bf6d3f11337` — test(init): 안정 단계별 초기화 중단 복구 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `BOOTSTRAP`, `RECOVERY` |
| Source-defined role | Killed bootstrap containers at every durable stage and proved rerun convergence. |
| 이전 Thread commit | `3beebbfc4723` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Kills MariaDB and WordPress bootstrap containers at every durable stage, reruns startup, and verifies state, credentials, markers, and temporary-file cleanup.
- **Classification reason:** This is unusually strong evidence for the staged-convergence invariant and demonstrates that the core initialization design survives abrupt process death rather than only graceful errors.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `2bf6d3f11337`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `verify_bootstrap / pause-ready protocol`에서 sleep 추측이 아니라 production test hook의 명시적 handoff를 사용합니다.
- `tests/runtime_stack.py`의 `docker kill --signal KILL`에서 graceful exception cleanup이 아니라 abrupt process death를 재현합니다.
- `tests/runtime_stack.py`의 `rerun start / state and boundary assertions`에서 모든 durable stage가 재실행 수렴한다는 end-to-end evidence를 만듭니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2bf6d3f11337 | tests/runtime_stack.py | verify_bootstrap / pause-ready protocol | 각 durable bootstrap stage에 `--pause-after`와 private ready file을 설정해 process가 정확한 stage를 통과했음을 동기화합니다. | sleep 추측이 아니라 production test hook의 명시적 handoff를 사용합니다. |
| 2bf6d3f11337 | tests/runtime_stack.py | docker kill --signal KILL | ready marker 확인 직후 해당 one-off bootstrap container를 SIGKILL하고, shell trap이 실행되지 않는 상태를 만듭니다. | graceful exception cleanup이 아니라 abrupt process death를 재현합니다. |
| 2bf6d3f11337 | tests/runtime_stack.py | rerun start / state and boundary assertions | 같은 project를 다시 시작해 DB/site/users/passwords/markers/health를 확인하고 long-running container에 secret mount/env가 없는지 재검사합니다. | 모든 durable stage가 재실행 수렴한다는 end-to-end evidence를 만듭니다. |

#### 비교 기준

- exact commit diff: `git diff 2bf6d3f11337^ 2bf6d3f11337 -- <path>`
- 이전 Thread 상태와 비교: `git diff 3beebbfc4723 2bf6d3f11337 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | verified marker/rename 전 partial state는 완료로 취급되지 않으며 같은 project startup은 다시 수렴합니다. |
| 재현하는 failure / boundary | MariaDB와 WordPress의 각 durable stage 직후 one-off bootstrap process가 SIGKILL되는 경계입니다. |
| test technique | live integration + deterministic pause-ready handshake + SIGKILL |
| fixture와 failure injection | 고유 project/port/secrets를 만들고 production `start_stack.py`에 pause stage를 전달한 뒤 ready file에서 동기화해 target container를 KILL합니다. |
| 실제 통과하는 production path | host orchestrator→one-off bootstrap→persistent volumes→health-gated long-running services 전체 경로를 통과합니다. |
| 핵심 assertion | 재실행 성공, marker/health, DB/site/users/passwords, runtime secret mount/env 부재, project cleanup을 확인합니다. |
| 이 테스트가 증명하는 것 | graceful handler 없이 process가 죽어도 durable stage별 재시도가 complete state로 수렴함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 하드웨어 crash consistency, uninstrumented 임의 지점, 외부 concurrent mutation은 증명하지 않습니다. |
| 성격 | deterministic runtime regression |
| 막는 후속 regression | partial directory/marker 조기 publication, SIGKILL 뒤 permanent broken volume, steady-state secret 재도입을 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 전원 손실/스토리지 write cache, 임의의 모든 instruction, 다른 Docker/OS 조합을 포괄하지는 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: Thread 3 이후 runtime harness의 기반이 되며 bootstrap lifecycle의 핵심 regression evidence입니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 각 durable MariaDB/WordPress stage에서 abrupt death가 발생해도 재실행이 one complete stack과 secret-free runtime으로 수렴함을 증명합니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 장기 실행 컨테이너는 host secret mount나 password-bearing environment를 유지하지 않습니다. | 916391b9f8db의 중간 단계 | dc9601f5e670에서 최종 경계 확립 | 2bf6d3f11337 | Compose runtime service block과 live container inspect 모두 `/run/secrets`/credential env 부재를 요구합니다. |
| 동일 Compose project를 변경하는 management operation은 직렬화됩니다. | e77c6f151b07 | dc9601f5e670 startup 적용 | 후속 backup/rotation runtime tests | `project_operation_lock`은 project identity별 non-blocking flock을 startup/management transaction 전체에 유지합니다. |
| completion marker는 필수 data/config/account/credential 검증 뒤에만 게시됩니다. | dc9601f5e670 | 3beebbfc4723 source contract | 2bf6d3f11337 | MariaDB staging rename과 WordPress marker replace가 verification 뒤에 있고 SIGKILL 재실행이 이를 실제로 확인합니다. |
| service readiness는 durable marker와 live process readiness를 함께 요구합니다. | dc9601f5e670 | 3beebbfc4723 | 2bf6d3f11337 | MariaDB marker+socket+PID, WordPress marker+FastCGI ping이 health condition입니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: `916391b9f8db`의 runtime secret mounts와 최초 entrypoint idempotency는 process death 뒤 partial persistent state를 완료 상태와 구분하지 못했습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: host descriptor validation, per-project lock, MariaDB staging rename, WordPress durable marker, marker-aware health와 one-off bootstrap ordering이 수렴 조건을 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `3beebbfc4723` static contract와 `2bf6d3f11337`의 durable-stage SIGKILL matrix가 source 구조와 실제 재실행 수렴을 각각 보호합니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| steady-state service가 secret mount를 보유하고 ordinary entrypoint가 초기화를 수행 | dc9601f5e670가 host-read/stdin/one-off bootstrap으로 교정 | 2bf6d3f11337가 runtime secret boundary와 rerun convergence를 검증 | credential lifetime과 state convergence를 serving lifecycle에서 분리했습니다. |
| partial volume existence가 completed state로 오인됨 | staging directory + verified marker + atomic publication | 3beebbfc4723 static contract와 2bf6d3f11337 SIGKILL regression | 완료 판정은 단순 경로 존재가 아니라 검증 뒤 publication입니다. |
| 동시 management operation이 같은 resource assumptions를 변경 | e77c6f151b07의 per-project non-blocking lock | 후속 cross-TMPDIR contention과 management tests | lock identity는 project name이며 다른 project는 병렬 가능합니다. |

### 직접 재구성할 chain

```text
기존 가정: 존재 여부 기반 idempotent entrypoint와 graceful cleanup이면 first run 재시도가 안전하다는 가정
  → 실제 failure 또는 위험: SIGKILL이 system directory, WordPress files 또는 DB rows 일부만 남기면 다음 실행이 이를 완료로 오인할 수 있었습니다.
  → root cause: credential material이 long-running service에 붙어 있었고 durable completion을 나타내는 verified publication point가 없었습니다.
  → 수정된 invariant / decision: host lock 아래 short-lived bootstrap만 secrets를 받고 MariaDB는 staging directory를 rename하며 WordPress는 검증 뒤 marker를 원자 게시합니다.
  → 해당 SHA의 실제 수정 코드: `dc9601f5e670`의 `start_stack.py`와 두 bootstrap entrypoint의 staging/marker/publish 경로
  → failure injection 또는 regression test: `2bf6d3f11337`가 각 pause stage에서 bootstrap container를 SIGKILL하고 같은 startup을 재실행합니다.
  → 증명된 보장 / 남은 비보장: 중간 state는 재실행으로 verified endpoint에 수렴하고 steady-state containers에는 secret mounts가 없지만 host/process 전체 crash의 모든 storage failure까지 증명하지는 않습니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Host management code | Compose가 secret file을 service에 mount | secret source 검증, project lock, bootstrap orchestration 소유 | 486ffb5c65aa/e77c6f151b07/dc9601f5e670 | credential과 operation order를 container 밖에서 통제합니다. |
| One-off bootstrap container | 장기 service entrypoint와 초기화 결합 | stdin credential을 잠시 받아 persistent state convergence만 수행 | dc9601f5e670 Compose run/labels/stdin path | 종료 뒤 credential-bearing process가 남지 않습니다. |
| Long-running MariaDB/WordPress | startup 때 secret path 접근 | verified persistent state를 열고 serving만 수행 | runtime mounts/env absence와 marker-gated command | steady state에서 host credential file을 보유하지 않습니다. |
| Completion marker | directory/file existence의 암묵적 판단 | 검증이 끝난 durable state의 publication boundary | write/fsync/rename/health evidence | marker 부재는 재수렴 필요를 뜻합니다. |
| WordPress configuration | public web tree 안 config | private config volume이 authoritative이고 web tree는 controlled symlink | dc9601f5e670 mount/symlink/marker path | Nginx가 private config volume을 읽지 않습니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** The earlier `_FILE` and Compose-secret model reduced direct environment exposure but still attached credential material to service startup. The later architecture resolves secrets on the host while holding the project lock, sends only required values to short-lived bootstrap containers, and lets long-running services start from verified persistent state. The final SIGKILL scenario is important because it validates the design's intended convergence after process death, not only after controlled errors.
- 최종 authoritative state와 owner: MariaDB final data directory/marker와 WordPress data/config marker가 persistent authoritative state이며 host management code가 secret source와 project lock을 소유합니다.
- 정상 실행의 entry point와 완료 조건: `start_stack.py`가 lock 안에서 secret을 읽고 DB bootstrap, DB health, WordPress bootstrap, application/Nginx start를 끝내며 모든 marker+process health가 성공하면 완료입니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: 실패·SIGKILL 뒤 marker가 없는 state는 다음 실행에서 다시 검사·수렴하며, complete marker가 없으면 long-running consumer를 healthy로 열지 않습니다.
- 이 Thread가 다른 Thread에 제공하는 전제: backup, restore, rotation이 같은 project lock과 hardened secret-input boundary를 재사용할 수 있는 전제를 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: 하드웨어 crash durability, non-cooperating manual Docker mutation, 모든 가능한 instruction-level kill point를 단독으로 증명하지 않습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | project lock 획득 | e77c6f151b07 `project_operation_lock` | 같은 project의 management mutation을 직렬화합니다. | contention이면 mutation 없이 즉시 실패합니다. |
| 2 | secret path 해석/읽기 | 486ffb5c65aa `secret_source_paths`, `read_private_secret` | rendered metadata와 descriptor 검사로 네 값을 memory에 올립니다. | unsafe path/mode/content면 bootstrap 전 실패합니다. |
| 3 | MariaDB one-off bootstrap | dc9601f5e670 `start_stack.py` + MariaDB entrypoint | stdin credential로 staging DB와 accounts를 만듭니다. | 중단 시 final directory/marker가 없으므로 다음 실행이 다시 수행합니다. |
| 4 | DB state publish | dc9601f5e670 marker/fsync/rename | 검증된 staging을 final path로 게시합니다. | publication 전 failure는 incomplete state를 final로 보이지 않습니다. |
| 5 | WordPress bootstrap | dc9601f5e670 WordPress entrypoint | private config, core, site, users/passwords를 검증·수렴합니다. | 어느 stage 실패든 marker를 게시하지 않고 재실행 대상이 됩니다. |
| 6 | runtime service start | dc9601f5e670 Compose health/start stages | marker+live probe 성공 뒤 WordPress와 Nginx를 엽니다. | health timeout은 startup failure이며 secret mount는 runtime에 없습니다. |
| 7 | SIGKILL regression | 2bf6d3f11337 `verify_bootstrap` | 각 durable stage kill 뒤 같은 project가 complete state로 수렴합니다. | test harness가 project-scoped teardown을 시도하며 실행 환경에서는 이번에 재실행하지 않았습니다. |

### 학습자의 최종 설명

> 초기 `_FILE` 방식은 password literal을 environment에서 제거했지만 long-running container에 secret mount를 남기고 초기화와 serving을 한 entrypoint에 묶었습니다. `486ffb5c65aa`와 `e77c6f151b07`은 host secret trust boundary와 same-project serialization을 만들었고, `dc9601f5e670`은 이 기반 위에서 startup을 short-lived one-off bootstrap transaction으로 바꿨습니다. MariaDB는 private staging을 검증한 뒤 final directory를 게시하고, WordPress는 config/site/users/password 검증 뒤 completion marker를 게시합니다. long-running services는 이 verified state만 열며 secret을 mount하지 않습니다. 정적 contract는 source order를 고정하고, SIGKILL test는 cleanup trap 없이 죽은 실제 process 뒤에도 같은 project가 수렴한다는 별도 runtime evidence를 제공합니다.

## 학습 완료 자가 점검

- [x] Compose secret 자체를 최종 steady-state secret 경계라고 설명하지 않았습니까?
- [x] marker 생성 시점과 data directory publication 시점을 실제 코드 순서로 확인했습니까?
- [x] SIGKILL 뒤 shell cleanup trap이 실행된다고 가정하지 않았습니까?
- [x] 같은 project만 직렬화되고 다른 project는 병렬 가능하다는 granularity를 설명했습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 02-convergent-one-off-bootstrap.md =====

===== BEGIN FILE: 03-isolated-runtime-and-persistence.md =====
# Thread 3 — Isolated runtime evidence and persistent-state verification

## Thread 목표

고정 identity를 제거해 독립 project를 만들고, isolated Docker harness로 request path와 persistent volume 보장을 실제 runtime에서 검증하는 흐름을 추적합니다.

**Source significance**

> Parameterization made independent test projects possible; the harness then turned those parameters into controlled Docker resources and private credentials. End-to-end and persistence scenarios prove distinct properties: one shows that the integrated request/data path works, while the other shows that container replacement does not replace authoritative volume state.

## 이 Thread를 이해하기 위한 핵심 질문

- fixed container/image/port/URL identity가 여러 test project를 막는 방식은 무엇입니까?
- test harness가 developer default project를 건드리지 않는다는 증거는 무엇입니까?
- source-level Compose validation과 live container inspection은 각각 무엇을 놓칠 수 있습니까?
- end-to-end request/data-path test와 restart/recreate persistence test가 증명하는 속성은 어떻게 다릅니까?
- port conflict recovery가 임의의 startup failure를 숨기지 않도록 어떤 조건으로 제한됩니까?

## 완료 기준

- project/image/port/URL parameter가 Compose resource naming과 WordPress canonical URL에 미치는 영향을 확인했습니다.
- harness의 private env/secret creation, timeout, diagnostics, teardown 경계를 코드로 추적했습니다.
- HTTPS → FastCGI → WordPress → MariaDB의 동일 데이터 round trip을 test assertion으로 복원했습니다.
- restart와 container recreation 뒤에도 같은 volume set이 유지되는지 기록했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `9d75a34e290f` | feat(runtime): 프로젝트·이미지·포트·URL 격리 | **A** | `ARCH`<br>`STACK`<br>`OPERATIONS` | Removed fixed project, image, port, and URL identities. |
| 2 | `2c436f574712` | test(bootstrap): 격리된 런타임 하네스 추가 | **A** | `TEST`<br>`ARCH`<br>`OPERATIONS` | Created the isolated Docker runtime harness and secret-boundary inspection. |
| 3 | `8c9b5b9adef2` | test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증 | **A** | `TEST`<br>`INTEGRATION`<br>`STACK` | Verified the complete HTTPS, FastCGI, WordPress, and MariaDB data path. |
| 4 | `fb1a689cf969` | test(persistence): 재시작·재생성 뒤 상태 보존 검증 | **A** | `TEST`<br>`PERSISTENCE`<br>`RISK` | Verified database, option, upload, and volume identity across restart and recreation. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `9d75a34e290f` — feat(runtime): 프로젝트·이미지·포트·URL 격리

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `ARCH`, `STACK`, `OPERATIONS` |
| Source-defined role | Removed fixed project, image, port, and URL identities. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `2c436f574712` |

#### 원문이 확정한 범위

- **Summary:** Parameterizes project names, image tags, HTTPS binding, port, and canonical WordPress URL while removing fixed container names.
- **Classification reason:** This enables multiple isolated stacks and makes later runtime testing and fresh-project restore possible; it is a significant deployment-boundary improvement.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `9d75a34e290f`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `container_name removal / image variables`에서 Compose project namespace가 container/resource names를 소유하고 test별 image tag가 충돌하지 않습니다.
- `srcs/docker-compose.yml`의 `HTTPS_BIND_ADDRESS / HTTPS_PORT`에서 여러 stack이 서로 다른 host port에서 동시에 실행될 수 있습니다.
- `.env.example / WordPress environment`의 `WORDPRESS_URL`에서 runtime endpoint와 WordPress home/site URL이 같은 test identity를 사용합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 9d75a34e290f | srcs/docker-compose.yml | container_name removal / image variables | 고정 `container_name`을 제거하고 `STACK_IMAGE_PREFIX`와 `STACK_IMAGE_TAG`로 local image identity를 parameterize합니다. | Compose project namespace가 container/resource names를 소유하고 test별 image tag가 충돌하지 않습니다. |
| 9d75a34e290f | srcs/docker-compose.yml | HTTPS_BIND_ADDRESS / HTTPS_PORT | host publish address와 port를 environment parameter로 만들고 loopback/non-default port를 허용합니다. | 여러 stack이 서로 다른 host port에서 동시에 실행될 수 있습니다. |
| 9d75a34e290f | .env.example / WordPress environment | WORDPRESS_URL | canonical WordPress URL을 명시적으로 요구해 domain과 non-default HTTPS port를 site state에 반영합니다. | runtime endpoint와 WordPress home/site URL이 같은 test identity를 사용합니다. |

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 고정 project/container/image/443 port/canonical URL은 두 test stack이 같은 Docker names와 host socket을 차지하게 했습니다. |
| 선택한 boundary / decision | Compose project name과 image prefix/tag, bind address/port, canonical URL을 모두 외부 parameter로 바꾸고 fixed container name을 제거했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/docker-compose.yml`의 `container_name removal / image variables`; `srcs/docker-compose.yml`의 `HTTPS_BIND_ADDRESS / HTTPS_PORT`; `.env.example / WordPress environment`의 `WORDPRESS_URL` |
| state / ownership / lifecycle 변화 | caller가 project identity를 선택하고 Compose가 그 이름으로 containers/networks/volumes를 namespace합니다. WordPress DB state에는 caller가 제공한 URL이 저장됩니다. |
| 주요 failure branch | 잘못 조합된 domain/port/URL은 stack이 뜨더라도 redirect와 test request가 어긋날 수 있습니다. parameterization 자체는 isolation을 실제로 사용했는지 증명하지 않습니다. |
| 이 commit의 보장 | 독립된 project/image/port/URL 조합을 만들 수 있고 default stack과 test stack이 이름을 공유하지 않게 합니다. |
| 한계와 다음 관련 commit | 실제 secret isolation, port reservation race, project-scoped teardown, request/data path correctness는 보장하지 않습니다. `2c436f574712`가 이 parameter를 사용해 private isolated harness를 만들고 후속 e2e/persistence scenario가 runtime 보장을 검사합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 실제 secret isolation, port reservation race, project-scoped teardown, request/data path correctness는 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `2c436f574712`가 이 parameter를 사용해 private isolated harness를 만들고 후속 e2e/persistence scenario가 runtime 보장을 검사합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 독립된 project/image/port/URL 조합을 만들 수 있고 default stack과 test stack이 이름을 공유하지 않게 합니다.

### 2. `2c436f574712` — test(bootstrap): 격리된 런타임 하네스 추가

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `ARCH`, `OPERATIONS` |
| Source-defined role | Created the isolated Docker runtime harness and secret-boundary inspection. |
| 이전 Thread commit | `9d75a34e290f` |
| 다음 Thread commit | `8c9b5b9adef2` |

#### 원문이 확정한 범위

- **Summary:** Adds an isolated Docker runtime harness with private credentials, random project names, dynamic ports, cleanup, and secret-boundary inspection.
- **Classification reason:** The harness becomes the foundation for the branch's later behavioral evidence and materially changes the project from source-validated configuration to reproducible runtime verification.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `2c436f574712`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `RuntimeStack.__init__ / _prepare_environment`에서 fixture의 credentials와 Docker identity가 developer defaults와 분리됩니다.
- `tests/runtime_stack.py`의 `reserve_port / run_compose / _run_start`에서 hang과 fixed-port collision을 test process의 명시적 failure로 바꿉니다.
- `tests/runtime_stack.py`의 `assert_runtime_secret_boundary / inspect_service`에서 source string만이 아니라 effective container configuration을 검사합니다.
- `tests/runtime_stack.py`의 `close / project-scoped down`에서 default project나 unrelated Docker resource를 teardown 대상으로 사용하지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2c436f574712 | tests/runtime_stack.py | RuntimeStack.__init__ / _prepare_environment | PID와 random token으로 unique project/image prefix를 만들고 temporary directory `0700`, secret files `0600`, private env file, loopback port를 준비합니다. | fixture의 credentials와 Docker identity가 developer defaults와 분리됩니다. |
| 2c436f574712 | tests/runtime_stack.py | reserve_port / run_compose / _run_start | loopback socket으로 candidate port를 찾고 모든 subprocess/Compose command에 bounded timeout을 적용합니다. | hang과 fixed-port collision을 test process의 명시적 failure로 바꿉니다. |
| 2c436f574712 | tests/runtime_stack.py | assert_runtime_secret_boundary / inspect_service | live container inspect와 rendered Compose를 사용해 runtime service의 password env와 `/run/secrets` mount가 없는지 확인합니다. | source string만이 아니라 effective container configuration을 검사합니다. |
| 2c436f574712 | tests/runtime_stack.py | close / project-scoped down | scenario가 만든 project name으로 `down --volumes --remove-orphans`하고 private temporary files를 제거합니다. | default project나 unrelated Docker resource를 teardown 대상으로 사용하지 않습니다. |

#### 비교 기준

- exact commit diff: `git diff 2c436f574712^ 2c436f574712 -- <path>`
- 이전 Thread 상태와 비교: `git diff 9d75a34e290f 2c436f574712 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | test scenario는 developer/default stack과 resource identity를 공유하지 않고 runtime service는 bootstrap secret을 보유하지 않습니다. |
| 재현하는 failure / boundary | fixed project/image/port/credential 또는 source/effective config 불일치 경계입니다. |
| test technique | live Docker harness + rendered config + container inspect |
| fixture와 failure injection | private temp directory에서 random project/image/secret과 loopback port를 만들고 production staged start를 실행합니다. |
| 실제 통과하는 production path | RuntimeStack 준비→`start_stack.py`→Compose services→inspect/health→project-scoped teardown 경로입니다. |
| 핵심 assertion | unique names/paths/modes, command timeouts, runtime secret env/mount 부재를 확인합니다. |
| 이 테스트가 증명하는 것 | isolated runtime fixture와 secret-boundary inspection이 실제 Docker objects에 적용됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | WordPress content round trip, named-volume persistence, all cleanup failure modes는 증명하지 않습니다. |
| 성격 | integration harness foundation |
| 막는 후속 regression | default namespace 사용, world-readable secret fixture, timeout 없는 subprocess, runtime secret 재도입을 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 특정 application request/data path나 container recreation 뒤 state 보존은이 harness 도입만으로 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `8c9b5b9adef2`와 `fb1a689cf969`이 같은 harness에 서로 다른 runtime properties를 추가합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 각 scenario가 고유 project와 credentials로 실제 stack을 만들고 effective secret boundary를 검사할 수 있게 합니다.

### 3. `8c9b5b9adef2` — test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `INTEGRATION`, `STACK` |
| Source-defined role | Verified the complete HTTPS, FastCGI, WordPress, and MariaDB data path. |
| 이전 Thread commit | `2c436f574712` |
| 다음 Thread commit | `fb1a689cf969` |

#### 원문이 확정한 범위

- **Summary:** Extends the harness to test HTTPS health, WordPress post creation and rendering, MariaDB persistence, port-conflict recovery, and legacy configuration migration.
- **Classification reason:** It verifies the complete browser-to-database path and catches integration failures that static checks cannot, making it significant but not an architectural implementation commit.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `8c9b5b9adef2`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `verify_e2e / blocked-port fixture`에서 임의 startup error를 port retry로 숨기지 않습니다.
- `tests/runtime_stack.py`의 `WP-CLI post creation`에서 fixture가 다른 run의 data와 혼동되지 않습니다.
- `tests/runtime_stack.py`의 `HTTPS fetch / MariaDB query`에서 Nginx→FastCGI→WordPress→MariaDB의 동일 data round trip을 연결합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 8c9b5b9adef2 | tests/runtime_stack.py | verify_e2e / blocked-port fixture | candidate port를 실제 listener로 점유한 상태에서 start를 시도해 genuine bind conflict만 분류하고 bounded 횟수 안에서 새 port로 재시도합니다. | 임의 startup error를 port retry로 숨기지 않습니다. |
| 8c9b5b9adef2 | tests/runtime_stack.py | WP-CLI post creation | unique token이 포함된 published post를 WordPress application interface로 생성하고 post ID/title/content를 기록합니다. | fixture가 다른 run의 data와 혼동되지 않습니다. |
| 8c9b5b9adef2 | tests/runtime_stack.py | HTTPS fetch / MariaDB query | HTTPS response에서 token을 확인하고 MariaDB에서 동일 post ID의 row와 content를 query해 비교합니다. | Nginx→FastCGI→WordPress→MariaDB의 동일 data round trip을 연결합니다. |

#### 비교 기준

- exact commit diff: `git diff 8c9b5b9adef2^ 8c9b5b9adef2 -- <path>`
- 이전 Thread 상태와 비교: `git diff 2c436f574712 8c9b5b9adef2 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | 외부 HTTPS request가 Nginx와 FastCGI를 거쳐 WordPress에서 실행되고 결과가 MariaDB row와 일치합니다. |
| 재현하는 failure / boundary | host port가 실제로 점유된 경우의 bounded recovery와 integrated data-path 단절입니다. |
| test technique | live end-to-end integration + uniquely identifiable fixture + DB differential assertion |
| fixture와 failure injection | 점유 listener로 첫 port를 막고, 새 port에서 stack을 시작한 뒤 unique post를 WP-CLI로 생성합니다. |
| 실제 통과하는 production path | HTTPS listener→Nginx routing→`wordpress:9000`→WordPress mutation/read→MariaDB query를 통과합니다. |
| 핵심 assertion | port 변경, health, HTTPS body token, DB post ID/title/content 일치를 확인합니다. |
| 이 테스트가 증명하는 것 | public response와 authoritative relational row가 동일 application operation을 반영함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | restart/recreate persistence, high concurrency, browser semantics 전체는 증명하지 않습니다. |
| 성격 | broad integration with bounded edge regression |
| 막는 후속 regression | routing/service-name/URL mismatch, false port-error retry, DB write와 public response 분리를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 컨테이너 restart/recreation 뒤에도 data가 남는지, backup/restore를 거치는지는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `fb1a689cf969`이 같은 stack에서 restart와 recreation을 별도 persistence property로 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 실제 integrated request/data path와 canonical non-default URL/port가 하나의 test project 안에서 일치함을 증명합니다.

### 4. `fb1a689cf969` — test(persistence): 재시작·재생성 뒤 상태 보존 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `PERSISTENCE`, `RISK` |
| Source-defined role | Verified database, option, upload, and volume identity across restart and recreation. |
| 이전 Thread commit | `8c9b5b9adef2` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Verifies posts, options, uploads, and all three named volumes across container restart and recreation.
- **Classification reason:** The test locks down a central durable-state invariant and distinguishes container lifecycle from volume lifecycle, providing strong evidence for a core project guarantee.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `fb1a689cf969`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `verify_persistence / project_volumes`에서 state 값뿐 아니라 authoritative volume identity를 비교할 기준을 만듭니다.
- `tests/runtime_stack.py`의 `persistent fixtures`에서 서로 다른 persistence class를 한 scenario에서 검사합니다.
- `tests/runtime_stack.py`의 `restart then down/up recreation`에서 process restart와 container replacement를 구분해 증명합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| fb1a689cf969 | tests/runtime_stack.py | verify_persistence / project_volumes | 초기 MariaDB/WordPress data/config named volume set과 concrete names를 기록합니다. | state 값뿐 아니라 authoritative volume identity를 비교할 기준을 만듭니다. |
| fb1a689cf969 | tests/runtime_stack.py | persistent fixtures | unique post, WordPress option, upload file을 각각 relational DB, application option, filesystem state로 만듭니다. | 서로 다른 persistence class를 한 scenario에서 검사합니다. |
| fb1a689cf969 | tests/runtime_stack.py | restart then down/up recreation | 먼저 service restart 후 값을 검사하고, 이어 `down`(volume 미삭제)과 `up`으로 container를 재생성한 뒤 exact volume set과 모든 값을 다시 확인합니다. | process restart와 container replacement를 구분해 증명합니다. |

#### 비교 기준

- exact commit diff: `git diff fb1a689cf969^ fb1a689cf969 -- <path>`
- 이전 Thread 상태와 비교: `git diff 8c9b5b9adef2 fb1a689cf969 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | container lifecycle과 persistent volume lifecycle은 분리되어 container replacement가 authoritative state를 교체하지 않습니다. |
| 재현하는 failure / boundary | restart 또는 `down`/`up` recreation 뒤 새 empty volume이나 누락된 application/filesystem state를 받는 경계입니다. |
| test technique | live persistence integration + identity/value comparison |
| fixture와 failure injection | unique post, option, upload를 만들고 initial volume set을 기록한 뒤 restart와 container recreation을 수행합니다. |
| 실제 통과하는 production path | WordPress/MariaDB writes→named volumes→restart/recreate→WP-CLI/DB/filesystem reads를 통과합니다. |
| 핵심 assertion | exact volume set, post row, option value, upload checksum/content를 전후 비교합니다. |
| 이 테스트가 증명하는 것 | container replacement와 process restart 뒤 동일 named-volume state가 유지됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | explicit volume deletion, host failure, backup/restore correctness는 증명하지 않습니다. |
| 성격 | deterministic persistence regression |
| 막는 후속 regression | anonymous/new volume mount, down 시 volume 삭제, state class 일부만 보존되는 회귀를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: host disk loss, explicit volume deletion, backup consistency, migration across Docker hosts는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: Thread 1에서 도입된 named-volume ownership을 runtime evidence로 고정하고 backup/restore Thread의 기준 state를 제공합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: process restart와 container recreation이 authoritative named-volume identity와 세 종류의 state를 바꾸지 않음을 증명합니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 각 runtime scenario는 고유 Compose project, image prefix, port, credential을 사용합니다. | 9d75a34e290f | 2c436f574712 | 8c9b5b9adef2, fb1a689cf969 | parameterized Compose와 RuntimeStack random identity/private fixture가 실제 scenario 전 과정에 사용됩니다. |
| loopback HTTPS bind와 explicit WordPress URL은 non-default port에서도 일치합니다. | 9d75a34e290f | 2c436f574712 | 8c9b5b9adef2 | env file의 bind/port/URL이 public fetch와 WordPress canonical state에서 같은 값을 사용합니다. |
| 통합 request path 성공과 persistence는 별도 속성입니다. | 8c9b5b9adef2 | fb1a689cf969가 durable evidence 추가 | fb1a689cf969 | e2e는 순간 round trip, persistence는 restart/recreate 전후 volume ID와 값 비교를 수행합니다. |
| container replacement는 authoritative named volume identity를 바꾸지 않습니다. | 75590dedfb3a에서 구조 도입 | fb1a689cf969 | fb1a689cf969 | `project_volumes()`의 exact set과 post/option/upload 값을 recreation 전후 비교합니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: fixed project/container/image/port/URL identity는 동시에 두 stack을 만들 수 없고 developer default resources를 test fixture와 분리하지 못했습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: parameterized Compose identity와 harness의 private credentials, random project name, loopback port, bounded command, exact teardown이 isolation을 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `8c9b5b9adef2`가 전체 request/data path를, `fb1a689cf969`가 restart와 `down`/`up` 뒤 동일 volume identity와 values를 검증합니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| fixed names/ports/images로 test stack 간 충돌 | 9d75a34e290f가 identity parameterization | 2c436f574712가 unique private harness로 사용 | 가능성을 선언한 것과 실제 isolation을 사용한 것을 분리했습니다. |
| healthy process만으로 application data path를 추정 | 8c9b5b9adef2가 unique post를 public HTTPS와 DB row로 연결 | 동일 scenario의 HTTPS/DB assertions | service health와 business data round trip은 다른 evidence입니다. |
| container recreation 뒤 새 empty volume을 받아도 이전 e2e는 통과 | fb1a689cf969가 volume names와 세 state class를 기록 | restart/down-up 뒤 exact identity/value assertions | container와 authoritative state의 lifecycle을 분리합니다. |

### 직접 재구성할 chain

```text
기존 가정: 고정 이름과 기본 포트로도 runtime test를 반복할 수 있다는 가정
  → 실제 failure 또는 위험: 병렬·연속 실행에서 resource/port 충돌과 developer stack 오염 위험이 발생했습니다.
  → root cause: Compose resource identity와 test fixture ownership이 parameterized project namespace에 묶여 있지 않았습니다.
  → 수정된 invariant / decision: 매 실행마다 project/image prefix/port/URL/secrets를 분리하고 genuine bind conflict만 제한적으로 재시도합니다.
  → 해당 SHA의 실제 수정 코드: `9d75a34e290f` parameterization과 `2c436f574712` isolated harness
  → failure injection 또는 regression test: `8c9b5b9adef2` e2e 및 `fb1a689cf969` persistence scenarios
  → 증명된 보장 / 남은 비보장: 선택된 project의 통합 경로와 volume lifecycle은 검증하지만 Docker daemon crash나 physical storage durability는 증명하지 않습니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Compose project namespace | fixed names의 암묵적 공유 | scenario별 container/network/volume identity 소유 | 9d75a34e290f project parameter와 rendered names | default project와 test project가 이름을 공유하지 않습니다. |
| Harness temporary directory | developer environment에 의존 | env, secrets, diagnostics, control files의 private owner | 2c436f574712 mode/creation/cleanup | host-side test material은 0700/0600 범위에 남습니다. |
| Runtime data | health로 간접 추정 | post/option/upload와 volume identity로 명시적 검증 | 8c9b5b9adef2/fb1a689cf969 assertions | 관계형·application·filesystem state를 분리해 확인합니다. |
| Port selection | fixed host port | loopback candidate와 genuine bind-conflict-only retry | 8c9b5b9adef2 listener/error classification | 다른 startup error는 재시도로 숨기지 않습니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** Parameterization made independent test projects possible; the harness then turned those parameters into controlled Docker resources and private credentials. End-to-end and persistence scenarios prove distinct properties: one shows that the integrated request/data path works, while the other shows that container replacement does not replace authoritative volume state.
- 최종 authoritative state와 owner: Compose project가 Docker resource identity를, private RuntimeStack fixture가 test env/secrets/port/expected values를, named volumes가 persistent state를 소유합니다.
- 정상 실행의 entry point와 완료 조건: scenario별 production staged start가 healthy해지고 해당 e2e 또는 persistence assertion을 모두 통과하면 정상 완료입니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: start failure는 genuine bind conflict일 때만 bounded port retry하며 다른 failure는 즉시 보고합니다. 종료 시 project-scoped teardown을 수행합니다.
- 이 Thread가 다른 Thread에 제공하는 전제: 후속 backup/restore/rotation/operations scenarios가 default stack과 충돌하지 않고 live evidence를 만들 수 있는 harness를 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: Docker가 없는 현재 환경에서는 실제 scenario result를 새로 증명하지 않았으며 코드에 표현된 test mechanism만 확인했습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | private fixture 생성 | 2c436f574712 RuntimeStack preparation | 0700 temp dir, 0600 secrets, unique env를 만듭니다. | unsafe permission/file creation 실패면 Docker mutation 전 종료합니다. |
| 2 | project/image/port 선택 | 9d75a34e290f parameters + 2c436f574712 random identity | Compose resources와 host endpoint를 scenario별 분리합니다. | 8c9b5b9adef2가 genuine bind conflict만 새 port로 재시도합니다. |
| 3 | production startup | 2c436f574712 `_run_start` | `start_stack.py`를 bounded timeout으로 실행합니다. | timeout/start failure는 StackError로 전파됩니다. |
| 4 | runtime secret/marker inspect | 2c436f574712 live inspect | effective containers가 bootstrap boundary를 유지하는지 확인합니다. | secret env/mount나 marker/health mismatch면 실패합니다. |
| 5 | e2e round trip | 8c9b5b9adef2 `verify_e2e` | unique post가 HTTPS와 MariaDB에서 일치합니다. | 어느 layer든 token/ID가 다르면 path failure입니다. |
| 6 | persistence round trip | fb1a689cf969 `verify_persistence` | restart/recreate 뒤 같은 volumes와 values를 확인합니다. | identity/value mismatch는 persistence regression입니다. |
| 7 | teardown | RuntimeStack.close | scenario project와 private files만 제거합니다. | 후속 Thread 8에서는 cleanup failure도 scenario result에 합칩니다. |

### 학습자의 최종 설명

> 고정 이름과 포트를 없앤 것만으로 isolation이 증명되지는 않습니다. `RuntimeStack`은 random project/image identity, loopback port, private env/secrets, bounded subprocess, project-scoped teardown을 실제 fixture로 만들고 production startup을 호출합니다. e2e scenario는 unique post를 WordPress로 만든 뒤 public HTTPS와 MariaDB row에서 같은 값을 확인해 전체 request/data path를 증명합니다. persistence scenario는 별도로 initial volume set과 DB option/upload state를 기록하고 process restart와 container recreation 뒤 다시 비교합니다. 따라서 “현재 요청이 동작한다”와 “교체 뒤에도 권위 있는 state가 남는다”는 서로 다른 evidence로 유지됩니다.

## 학습 완료 자가 점검

- [x] e2e test가 persistence까지 자동 증명한다고 합쳤습니까?
- [x] port retry가 모든 startup error에 적용된다고 잘못 기록하지 않았습니까?
- [x] volume 이름의 동일성과 volume 안 값의 동일성을 모두 확인했습니까?
- [x] test harness가 default Compose namespace를 사용할 가능성을 코드로 배제했습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 03-isolated-runtime-and-persistence.md =====

===== BEGIN FILE: 04-atomic-backup-publication.md =====
# Thread 4 — Atomic backup publication under failure and cancellation

## Thread 목표

MariaDB transactional dump와 WordPress filesystem archive를 하나의 신뢰 가능한 backup set으로 결합하고, failure·signal에도 partial set을 게시하지 않으며 source stack을 복구하는 transaction을 추적합니다.

**Source significance**

> The implementation deliberately separates data capture from publication. Private streaming files, an exact output reservation, a manifest, and directory replacement ensure that only a complete set becomes visible. Signal-aware recovery and negative runtime tests establish the equally important converse: cancelled or failed work must not leave a plausible backup or a degraded source stack.

## 이 Thread를 이해하기 위한 핵심 질문

- 두 artifact가 생성됐다는 사실만으로 하나의 일관된 backup이라고 할 수 없는 이유는 무엇입니까?
- private file creation, fsync, directory fsync, checksum은 각각 어떤 failure window를 줄입니까?
- signal을 exception으로 전환하고 synchronized pause stage를 둔 이유는 무엇입니까?
- destination path reservation에서 pathname 비교가 아니라 device/inode identity가 필요한 이유는 무엇입니까?
- Nginx와 WordPress를 멈추되 MariaDB는 transactional dump를 위해 유지하는 ordering은 어디에 구현됩니까?
- published backup과 failed attempt의 observable end state는 각각 무엇입니까?

## 완료 기준

- data capture와 publication을 별도 단계로 나누고 각 durability boundary를 코드로 확인했습니다.
- backup directory가 정확히 DB dump, WordPress archive, manifest의 완전한 set으로만 보이는 과정을 추적했습니다.
- failure와 SIGINT/SIGTERM이 동일 cleanup/recovery 경로로 수렴하는지 확인했습니다.
- negative test가 final output, temporary sibling, ready marker, lock, service health를 어떻게 검사하는지 기록했습니다.
- large fixture와 signal-race test가 small 정상 처리보다 추가로 증명하는 내용을 구분했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `fdd55605ba74` | feat(backup): 백업 무결성과 비공개 파일 I/O 정의 | **B** | `PERSISTENCE`<br>`OPERATIONS` | Defined private output, synchronization, and checksum primitives. |
| 2 | `d26c885c5cd5` | feat(backup): 관리 작업 신호와 테스트 중단 경계 추가 | **A** | `RECOVERY`<br>`TEST`<br>`HARD` | Created deterministic signal and failure-test boundaries. |
| 3 | `3a0995ff0d4f` | feat(backup): 프로젝트별 백업 작업 잠금 적용 | **B** | `RECOVERY`<br>`OPERATIONS`<br>`PERSISTENCE` | Serialized backup with other operations on the same project. |
| 4 | `b478b5243c5a` | feat(backup): DB 덤프와 WordPress 볼륨 수집 | **A** | `PERSISTENCE`<br>`CORE`<br>`INTEGRATION` | Captured transactional MariaDB and WordPress volume streams. |
| 5 | `0540ff1b5a4b` | feat(backup): 백업 출력 경로를 안전하게 예약 | **A** | `PERSISTENCE`<br>`RISK`<br>`EDGE` | Reserved and identity-checked the destination path. |
| 6 | `6999190ffd34` | feat(backup): 백업 세트를 원자적으로 게시 | **S** | `PERSISTENCE`<br>`RECOVERY`<br>`HARD` | Published a complete checksummed backup set atomically and recovered services. |
| 7 | `b6920a0c918c` | test(backup): 게시 실패와 중단 정리 검증 | **A** | `TEST`<br>`RECOVERY`<br>`PERSISTENCE` | Verified non-publication, cleanup, recovery, and shared-lock behavior on failure. |
| 8 | `030e7310c665` | test(backup): 자원 충돌과 시그널 경계 검증 | **A** | `TEST`<br>`PERSISTENCE`<br>`EDGE` | Extended evidence to signal races, large data, and collision boundaries. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `fdd55605ba74` — feat(backup): 백업 무결성과 비공개 파일 I/O 정의

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `PERSISTENCE`, `OPERATIONS` |
| Source-defined role | Defined private output, synchronization, and checksum primitives. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `d26c885c5cd5` |

#### 원문이 확정한 범위

- **Summary:** Introduces SHA-256 helpers, directory synchronization, and exclusive private-file output primitives for backup work.
- **Classification reason:** These are necessary low-level safety utilities, but they are supporting pieces whose project significance depends on later backup publication and restore orchestration.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `fdd55605ba74`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `sha256_stream`에서 manifest digest 계산이 caller의 후속 stream 소비 위치를 망가뜨리지 않습니다.
- `tools/stack_backup.py`의 `private output helper`에서 artifact가 생성되는 첫 순간부터 다른 user에게 공개되지 않습니다.
- `tools/stack_backup.py`의 `flush/fsync / fsync_directory`에서 후속 rename 전에 file contents와 directory metadata의 durability precondition을 만듭니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| fdd55605ba74 | tools/stack_backup.py | sha256_stream | stream의 현재 position을 저장하고 처음부터 chunk로 SHA-256을 계산한 뒤 원래 position으로 되돌립니다. | manifest digest 계산이 caller의 후속 stream 소비 위치를 망가뜨리지 않습니다. |
| fdd55605ba74 | tools/stack_backup.py | private output helper | `O_CREAT\|O_EXCL`과 mode `0600`으로 output을 만들고 기존 path를 덮어쓰지 않습니다. | artifact가 생성되는 첫 순간부터 다른 user에게 공개되지 않습니다. |
| fdd55605ba74 | tools/stack_backup.py | flush/fsync / fsync_directory | file stream을 flush·fsync하고 parent directory descriptor도 sync하며 OS 오류를 backup-domain error로 변환합니다. | 후속 rename 전에 file contents와 directory metadata의 durability precondition을 만듭니다. |

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Defined private output, synchronization, and checksum primitives. |
| 핵심 input / output / state | helper가 열린 stream과 file descriptor의 lifetime을 소유하며 caller는 성공한 durable private file만 다음 단계로 넘깁니다. |
| 변경된 directive / helper / command | `tools/stack_backup.py`의 `sha256_stream`; `tools/stack_backup.py`의 `private output helper`; `tools/stack_backup.py`의 `flush/fsync / fsync_directory` |
| immediate failure 또는 boundary | existing path, short write/flush/fsync, non-seekable digest input 등 지원하지 않는 조건은 명시적 failure가 됩니다. |
| 다음 commit에 넘긴 한계 | DB dump와 WordPress archive의 cross-artifact consistency, atomic directory publication, service recovery는 보장하지 않습니다. `b478b5243c5a`가 streaming capture에 사용하고 `6999190ffd34`가 manifest와 atomic directory publication으로 완전한 set을 만듭니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: DB dump와 WordPress archive의 cross-artifact consistency, atomic directory publication, service recovery는 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `b478b5243c5a`가 streaming capture에 사용하고 `6999190ffd34`가 manifest와 atomic directory publication으로 완전한 set을 만듭니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 개별 artifact가 private하고 existing path를 덮어쓰지 않으며 checksum과 sync를 계산할 수 있음을 보장합니다.

### 2. `d26c885c5cd5` — feat(backup): 관리 작업 신호와 테스트 중단 경계 추가

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `RECOVERY`, `TEST`, `HARD` |
| Source-defined role | Created deterministic signal and failure-test boundaries. |
| 이전 Thread commit | `fdd55605ba74` |
| 다음 Thread commit | `3a0995ff0d4f` |

#### 원문이 확정한 범위

- **Summary:** Adds controlled signal handling plus deterministic failure and pause stages for management-operation tests.
- **Classification reason:** It creates a reliable way to exercise asynchronous cancellation through the same cleanup paths as ordinary errors, a significant failure-path engineering boundary used throughout backup and rotation testing.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `d26c885c5cd5`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `operation_signal_handlers`에서 signal과 ordinary exception을 같은 cleanup path로 보냅니다.
- `tools/stack_backup.py`의 `pause/failure stage hook`에서 failure timing을 sleep에 의존하지 않고 production control flow에 동기화합니다.
- `tools/stack_backup.py`의 `signal masking around ready publication`에서 test가 관측한 ready 상태와 실제 pause state가 어긋나는 window를 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| d26c885c5cd5 | tools/stack_backup.py | operation_signal_handlers | SIGINT/SIGTERM handler가 즉시 임의 지점에서 종료하는 대신 management-domain exception을 발생시키고 원래 handler를 finally에서 복원합니다. | signal과 ordinary exception을 같은 cleanup path로 보냅니다. |
| d26c885c5cd5 | tools/stack_backup.py | pause/failure stage hook | 명명된 stage에서 private ready file을 게시한 뒤 test가 진행을 허용할 때까지 기다리거나 configured failure를 발생시킵니다. | failure timing을 sleep에 의존하지 않고 production control flow에 동기화합니다. |
| d26c885c5cd5 | tools/stack_backup.py | signal masking around ready publication | ready marker 생성·sync와 handler transition의 작은 구간에서 signal race를 제어합니다. | test가 관측한 ready 상태와 실제 pause state가 어긋나는 window를 줄입니다. |

#### 비교 기준

- exact commit diff: `git diff d26c885c5cd5^ d26c885c5cd5 -- <path>`
- 이전 Thread 상태와 비교: `git diff fdd55605ba74 d26c885c5cd5 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 비동기 signal은 cleanup code 어느 지점에서든 process를 끝내 test 재현성과 resource recovery를 불명확하게 만들 수 있었습니다. |
| 선택한 boundary / decision | operator signal을 normal 실패 처리로 변환하고, named stage/ready-file protocol로 deterministic interruption point를 만들었습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/stack_backup.py`의 `operation_signal_handlers`; `tools/stack_backup.py`의 `pause/failure stage hook`; `tools/stack_backup.py`의 `signal masking around ready publication` |
| state / ownership / lifecycle 변화 | management operation이 handler 설치부터 복원까지 signal state를 소유합니다. test ready marker는 해당 operation의 temporary control state입니다. |
| 주요 failure branch | 첫 signal은 cancellation exception이 되고 finally가 service recovery와 temp cleanup을 수행합니다. ready-file publish 실패도 operation failure입니다. |
| 이 commit의 보장 | signal cancellation이 ordinary error와 같은 recovery path를 지나며 test가 정확한 stage에서 signal을 보낼 수 있습니다. |
| 한계와 다음 관련 commit | SIGKILL처럼 handler를 우회하는 종료나 하드웨어 장애는 처리하지 않습니다. `b6920a0c918c`와 `030e7310c665`가 publication stage와 signal race에서 이 mechanism을 사용합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: SIGKILL처럼 handler를 우회하는 종료나 하드웨어 장애는 처리하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `b6920a0c918c`와 `030e7310c665`가 publication stage와 signal race에서 이 mechanism을 사용합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: signal cancellation이 ordinary error와 같은 recovery path를 지나며 test가 정확한 stage에서 signal을 보낼 수 있습니다.

### 3. `3a0995ff0d4f` — feat(backup): 프로젝트별 백업 작업 잠금 적용

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `RECOVERY`, `OPERATIONS`, `PERSISTENCE` |
| Source-defined role | Serialized backup with other operations on the same project. |
| 이전 Thread commit | `d26c885c5cd5` |
| 다음 Thread commit | `b478b5243c5a` |

#### 원문이 확정한 범위

- **Summary:** Applies the per-project advisory lock model to backup operations.
- **Classification reason:** The lock is important, but this commit mainly extends an existing serialization decision to another management path rather than introducing a new project-wide mechanism.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `3a0995ff0d4f`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `project_operation_lock(project)`에서 사전 조건 검사와 mutation 사이에 다른 cooperating operation이 끼지 않습니다.
- `tools/stack_runtime.py`의 `shared lock identity`에서 operation 종류가 달라도 동일 project mutation은 하나의 serialization domain입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3a0995ff0d4f | tools/stack_backup.py | project_operation_lock(project) | backup entry path가 destination/secret/service state를 검사하고 writer를 멈추기 전부터 project lock을 보유합니다. | 사전 조건 검사와 mutation 사이에 다른 cooperating operation이 끼지 않습니다. |
| 3a0995ff0d4f | tools/stack_runtime.py | shared lock identity | startup, backup, restore, rotation이 같은 project-name-derived lock을 공유합니다. | operation 종류가 달라도 동일 project mutation은 하나의 serialization domain입니다. |

#### 비교 기준

- exact commit diff: `git diff 3a0995ff0d4f^ 3a0995ff0d4f -- <path>`
- 이전 Thread 상태와 비교: `git diff d26c885c5cd5 3a0995ff0d4f -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Serialized backup with other operations on the same project. |
| 핵심 input / output / state | backup host process가 capture/publication/recovery 동안 project mutation 권한을 소유합니다. |
| 변경된 directive / helper / command | `tools/stack_backup.py`의 `project_operation_lock(project)`; `tools/stack_runtime.py`의 `shared lock identity` |
| immediate failure 또는 boundary | lock contention은 source stack이나 output path를 건드리기 전에 failure가 됩니다. |
| 다음 commit에 넘긴 한계 | 수동 Docker/DB command처럼 lock을 사용하지 않는 actor는 막지 않습니다. `6999190ffd34`가 lock 안에서 전체 atomic backup을 구성하고 `b6920a0c918c`가 다른 TMPDIR에서도 same-project contention을 확인합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 수동 Docker/DB command처럼 lock을 사용하지 않는 actor는 막지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `6999190ffd34`가 lock 안에서 전체 atomic backup을 구성하고 `b6920a0c918c`가 다른 TMPDIR에서도 same-project contention을 확인합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: cooperating management operation과 같은 project backup이 겹치지 않음을 보장합니다.

### 4. `b478b5243c5a` — feat(backup): DB 덤프와 WordPress 볼륨 수집

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `PERSISTENCE`, `CORE`, `INTEGRATION` |
| Source-defined role | Captured transactional MariaDB and WordPress volume streams. |
| 이전 Thread commit | `3a0995ff0d4f` |
| 다음 Thread commit | `0540ff1b5a4b` |

#### 원문이 확정한 범위

- **Summary:** Streams a transactional MariaDB dump and a WordPress data/config archive into private files.
- **Classification reason:** This implements the substantive data-capture path spanning database and filesystem state, a major component of backup functionality but not yet its atomic publication guarantee.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `b478b5243c5a`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `database dump helper`에서 DB server는 running 상태에서 consistent transactional view를 제공합니다.
- `tools/stack_backup.py`의 `WordPress archive helper`에서 host가 volume implementation path를 직접 가정하지 않습니다.
- `tools/stack_backup.py`의 `archive content policy`에서 restore 시 public symlink와 private config source를 중복/충돌시키지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| b478b5243c5a | tools/stack_backup.py | database dump helper | `mariadb-dump`를 `--single-transaction`과 stream-oriented option으로 실행하고 stdout을 private file로 직접 전달합니다. | DB server는 running 상태에서 consistent transactional view를 제공합니다. |
| b478b5243c5a | tools/stack_backup.py | WordPress archive helper | one-off tar process가 WordPress data/config volume을 read-only로 mount해 archive stream을 private file로 보냅니다. | host가 volume implementation path를 직접 가정하지 않습니다. |
| b478b5243c5a | tools/stack_backup.py | archive content policy | public web tree의 config symlink를 archive에서 제외하고 실제 private config volume을 별도 root로 수집하며 archive member를 검증합니다. | restore 시 public symlink와 private config source를 중복/충돌시키지 않습니다. |

#### 비교 기준

- exact commit diff: `git diff b478b5243c5a^ b478b5243c5a -- <path>`
- 이전 Thread 상태와 비교: `git diff 3a0995ff0d4f b478b5243c5a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | private file primitive는 있었지만 DB와 WordPress의 authoritative sources를 어떤 command와 consistency mode로 capture할지 정의되지 않았습니다. |
| 선택한 boundary / decision | DB는 transactional dump stream, WordPress는 mounted volume tar stream으로 수집하고 application writers는 후속 orchestration에서 quiesce할 수 있게 했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/stack_backup.py`의 `database dump helper`; `tools/stack_backup.py`의 `WordPress archive helper`; `tools/stack_backup.py`의 `archive content policy` |
| state / ownership / lifecycle 변화 | MariaDB server가 transaction snapshot을 소유하고 dump subprocess가 output stream을 생산합니다. archive container가 volume read view를 소유하고 host helper가 files를 받습니다. |
| 주요 failure branch | subprocess timeout/nonzero, stream write, archive member validation failure는 artifact를 invalid로 처리합니다. 이 commit만으로 두 stream의 publication timing은 묶이지 않습니다. |
| 이 commit의 보장 | 큰 data도 host memory 전체에 올리지 않고 DB dump와 WordPress volume archive를 capture할 수 있습니다. |
| 한계와 다음 관련 commit | 두 artifact가 같은 application cut에 해당하거나 partial output이 final destination에 보이지 않는다는 보장은 아직 없습니다. `6999190ffd34`가 writers stop과 manifest/rename으로 두 stream을 한 backup set으로 묶고 `030e7310c665`가 large fixture를 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 두 artifact가 같은 application cut에 해당하거나 partial output이 final destination에 보이지 않는다는 보장은 아직 없습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `6999190ffd34`가 writers stop과 manifest/rename으로 두 stream을 한 backup set으로 묶고 `030e7310c665`가 large fixture를 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 큰 data도 host memory 전체에 올리지 않고 DB dump와 WordPress volume archive를 capture할 수 있습니다.

### 5. `0540ff1b5a4b` — feat(backup): 백업 출력 경로를 안전하게 예약

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `PERSISTENCE`, `RISK`, `EDGE` |
| Source-defined role | Reserved and identity-checked the destination path. |
| 이전 Thread commit | `b478b5243c5a` |
| 다음 Thread commit | `6999190ffd34` |

#### 원문이 확정한 범위

- **Summary:** Normalizes and reserves a new backup output directory while tracking its exact inode identity.
- **Classification reason:** The small interface prevents overwrite, symlink, and path-substitution races at the publication boundary, protecting the integrity of a high-risk destructive and archival workflow.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `0540ff1b5a4b`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `destination parent resolution`에서 publication이 예상한 parent filesystem 안에서만 일어납니다.
- `tools/stack_backup.py`의 `exclusive reservation / stat identity`에서 사용자가 지정한 pathname을 다른 object로 바꾸는 공격/경쟁을 식별할 기준이 생깁니다.
- `tools/stack_backup.py`의 `pre-publication identity recheck`에서 문자열 path 일치만으로 TOCTOU replacement를 놓치지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 0540ff1b5a4b | tools/stack_backup.py | destination parent resolution | destination parent를 먼저 canonicalize하고 symlink/unsafe parent와 존재하는 non-reservation target을 거부합니다. | publication이 예상한 parent filesystem 안에서만 일어납니다. |
| 0540ff1b5a4b | tools/stack_backup.py | exclusive reservation / stat identity | 최종 path에 private empty reservation object를 exclusive create하고 device/inode identity를 보존합니다. | 사용자가 지정한 pathname을 다른 object로 바꾸는 공격/경쟁을 식별할 기준이 생깁니다. |
| 0540ff1b5a4b | tools/stack_backup.py | pre-publication identity recheck | rename 직전 path를 다시 `stat`해 최초 reservation과 `(st_dev, st_ino)`가 같은지 확인합니다. | 문자열 path 일치만으로 TOCTOU replacement를 놓치지 않습니다. |

#### 비교 기준

- exact commit diff: `git diff 0540ff1b5a4b^ 0540ff1b5a4b -- <path>`
- 이전 Thread 상태와 비교: `git diff b478b5243c5a 0540ff1b5a4b -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | destination이 검증 뒤 symlink나 다른 directory/file로 바뀌면 complete backup을 공격자가 선택한 위치에 게시하거나 기존 data를 덮어쓸 수 있었습니다. |
| 선택한 boundary / decision | parent를 고정하고 final pathname에 exclusive private reservation을 만든 뒤 object identity를 publication 직전 재검사했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/stack_backup.py`의 `destination parent resolution`; `tools/stack_backup.py`의 `exclusive reservation / stat identity`; `tools/stack_backup.py`의 `pre-publication identity recheck` |
| state / ownership / lifecycle 변화 | operation이 reservation inode와 sibling temporary directory를 소유합니다. caller가 제공한 path string은 더 이상 충분한 authority가 아닙니다. |
| 주요 failure branch | existing output, symlink, unsafe parent, reservation identity mismatch, cross-filesystem rename 조건은 publication 전에 실패합니다. |
| 이 commit의 보장 | final destination을 기존 object 위에 덮어쓰지 않고 정확히 자신이 예약한 slot에만 게시함을 보장합니다. |
| 한계와 다음 관련 commit | reservation만으로 artifact completeness/checksum/service recovery를 보장하지 않습니다. `6999190ffd34`가 verified temporary directory를 이 reservation에 atomic replace합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: reservation만으로 artifact completeness/checksum/service recovery를 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `6999190ffd34`가 verified temporary directory를 이 reservation에 atomic replace합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: final destination을 기존 object 위에 덮어쓰지 않고 정확히 자신이 예약한 slot에만 게시함을 보장합니다.

### 6. `6999190ffd34` — feat(backup): 백업 세트를 원자적으로 게시

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **S** |
| Tags | `PERSISTENCE`, `RECOVERY`, `HARD` |
| Source-defined role | Published a complete checksummed backup set atomically and recovered services. |
| 이전 Thread commit | `0540ff1b5a4b` |
| 다음 Thread commit | `b6920a0c918c` |

#### 원문이 확정한 범위

- **Summary:** Stops application writers, captures database and WordPress state, writes a checksummed manifest, atomically publishes the set, and recovers services on failure.
- **Classification reason:** This is the defining backup transaction. It establishes the all-or-nothing publication and service-recovery guarantees needed to treat a directory as a valid backup.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `6999190ffd34`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `backup orchestration`에서 public/application writers를 quiesce하되 transactional dump를 위해 MariaDB는 계속 실행합니다.
- `tools/stack_backup.py`의 `private sibling capture`에서 불완전 artifact는 final pathname 아래에 보이지 않습니다.
- `tools/stack_backup.py`의 `validate/checksum/manifest/sync`에서 manifest가 정확한 artifact identity를 하나의 set으로 묶습니다.
- `tools/stack_backup.py`의 `reservation identity + atomic replace`에서 관측 가능한 final path는 incomplete reservation에서 complete directory로 한 번에 바뀝니다.
- `tools/stack_backup.py`의 `finally service recovery / cleanup`에서 failed backup이 source stack을 degraded 상태로 남기지 않는 반대 보장을 시도합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 6999190ffd34 | tools/stack_backup.py | backup orchestration | project lock과 signal handler 아래 source services, secrets, destination reservation을 검사하고 Nginx와 WordPress를 중지합니다. | public/application writers를 quiesce하되 transactional dump를 위해 MariaDB는 계속 실행합니다. |
| 6999190ffd34 | tools/stack_backup.py | private sibling capture | DB dump와 WordPress archive를 final path와 같은 parent의 private temporary directory에 streaming capture합니다. | 불완전 artifact는 final pathname 아래에 보이지 않습니다. |
| 6999190ffd34 | tools/stack_backup.py | validate/checksum/manifest/sync | archive 구조를 재검증하고 두 artifact의 size/digest를 manifest에 기록한 뒤 files와 temporary directory를 sync합니다. | manifest가 정확한 artifact identity를 하나의 set으로 묶습니다. |
| 6999190ffd34 | tools/stack_backup.py | reservation identity + atomic replace | reserved inode를 재확인한 뒤 temporary directory를 final destination으로 replace하고 parent를 fsync합니다. | 관측 가능한 final path는 incomplete reservation에서 complete directory로 한 번에 바뀝니다. |
| 6999190ffd34 | tools/stack_backup.py | finally service recovery / cleanup | 성공·실패·signal 모두에서 Nginx/WordPress를 다시 시작하고 temporary/reservation/ready state를 정리하며 recovery failure를 숨기지 않습니다. | failed backup이 source stack을 degraded 상태로 남기지 않는 반대 보장을 시도합니다. |

#### 비교 기준

- exact commit diff: `git diff 6999190ffd34^ 6999190ffd34 -- <path>`
- 이전 Thread 상태와 비교: `git diff 0540ff1b5a4b 6999190ffd34 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### S-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 commit 직전 상태 | 개별 stream capture만으로는 DB와 filesystem 사이 write가 끼거나 한 artifact만 final에 보이는 partial set, cancellation 뒤 stopped services가 생길 수 있었습니다. |
| 해결하려던 문제 | capture/validation/sync/identity/rename 어느 단계 실패든 final complete directory를 게시하지 않습니다. finally는 services를 복구하며 복구 실패는 primary error에 추가됩니다. |
| 기존 설계가 충분하지 않았던 이유 | 개별 stream capture만으로는 DB와 filesystem 사이 write가 끼거나 한 artifact만 final에 보이는 partial set, cancellation 뒤 stopped services가 생길 수 있었습니다. capture/validation/sync/identity/rename 어느 단계 실패든 final complete directory를 게시하지 않습니다. finally는 services를 복구하며 복구 실패는 primary error에 추가됩니다. |
| 핵심 결정 | writers stop, private sibling capture, manifest/checksum/sync, exact reservation, atomic directory replace, unconditional service recovery를 하나의 locked transaction으로 연결했습니다. |
| 주요 caller → callee / producer → consumer | `tools/stack_backup.py`의 `backup orchestration`; `tools/stack_backup.py`의 `private sibling capture`; `tools/stack_backup.py`의 `validate/checksum/manifest/sync`; `tools/stack_backup.py`의 `reservation identity + atomic replace`; `tools/stack_backup.py`의 `finally service recovery / cleanup` |
| authoritative state와 publication boundary | backup operation이 일시적으로 application writer lifecycle과 unpublished artifacts를 소유합니다. publication 후에는 final directory와 manifest가 authoritative backup unit입니다. 성공 시 세 파일의 complete checksummed set만 final path에 보이고, 실패/취소 시 plausible final set이 없으며 source application services가 복구됩니다. |
| ownership / lifetime / responsibility 변화 | backup operation이 일시적으로 application writer lifecycle과 unpublished artifacts를 소유합니다. publication 후에는 final directory와 manifest가 authoritative backup unit입니다. |
| failure scenario와 recovery path | capture/validation/sync/identity/rename 어느 단계 실패든 final complete directory를 게시하지 않습니다. finally는 services를 복구하며 복구 실패는 primary error에 추가됩니다. |
| 이 commit이 보장하는 것 | 성공 시 세 파일의 complete checksummed set만 final path에 보이고, 실패/취소 시 plausible final set이 없으며 source application services가 복구됩니다. |
| 아직 보장하지 않는 것 | MariaDB transaction 밖의 storage-level crash semantics, lock을 무시하는 외부 writer, remote filesystem rename semantics는 보장하지 않습니다. |
| 후속 fix / test와 연결 | `b6920a0c918c`이 failure/signal/lock cleanup을, `030e7310c665`가 signal race와 large stream을 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: MariaDB transaction 밖의 storage-level crash semantics, lock을 무시하는 외부 writer, remote filesystem rename semantics는 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `b6920a0c918c`이 failure/signal/lock cleanup을, `030e7310c665`가 signal race와 large stream을 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 성공 시 세 파일의 complete checksummed set만 final path에 보이고, 실패/취소 시 plausible final set이 없으며 source application services가 복구됩니다.

### 7. `b6920a0c918c` — test(backup): 게시 실패와 중단 정리 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `RECOVERY`, `PERSISTENCE` |
| Source-defined role | Verified non-publication, cleanup, recovery, and shared-lock behavior on failure. |
| 이전 Thread commit | `6999190ffd34` |
| 다음 Thread commit | `030e7310c665` |

#### 원문이 확정한 범위

- **Summary:** Adds runtime checks for failed backup publication, signal cancellation, service recovery, temporary cleanup, and cross-`TMPDIR` lock contention.
- **Classification reason:** It materially validates the negative guarantees of atomic backup: failure must publish nothing, restore the live stack, and release scoped synchronization resources.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `b6920a0c918c`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `backup failure-stage matrix`에서 production backup control flow의 여러 durable boundary를 결정적으로 통과합니다.
- `tests/runtime_stack.py`의 `negative filesystem assertions`에서 failed attempt가 plausible backup 흔적을 남기지 않음을 확인합니다.
- `tests/runtime_stack.py`의 `service health / lock contention`에서 recovery와 fixed project lock identity를 함께 검증합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| b6920a0c918c | tests/runtime_stack.py | backup failure-stage matrix | DB dump, archive, manifest, sync, publication 전후의 named pause/failure stage에서 command failure 또는 signal을 주입합니다. | production backup control flow의 여러 durable boundary를 결정적으로 통과합니다. |
| b6920a0c918c | tests/runtime_stack.py | negative filesystem assertions | final output, sibling temporary, reservation/ready marker가 남지 않고 기존 destination은 변경되지 않았는지 검사합니다. | failed attempt가 plausible backup 흔적을 남기지 않음을 확인합니다. |
| b6920a0c918c | tests/runtime_stack.py | service health / lock contention | 실패 뒤 Nginx/WordPress가 healthy인지 확인하고 다른 TMPDIR process가 같은 project lock을 얻지 못하는지 검사합니다. | recovery와 fixed project lock identity를 함께 검증합니다. |

#### 비교 기준

- exact commit diff: `git diff b6920a0c918c^ b6920a0c918c -- <path>`
- 이전 Thread 상태와 비교: `git diff 6999190ffd34 b6920a0c918c -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | 실패하거나 취소된 backup은 complete final set을 게시하지 않고 temporary/control state를 지우며 source services를 복구합니다. |
| 재현하는 failure / boundary | capture·manifest·sync·publish 경계의 injected failure/SIGINT/SIGTERM과 cross-TMPDIR lock contention입니다. |
| test technique | deterministic pause/failure injection + live Docker negative integration |
| fixture와 failure injection | healthy source stack과 private destination parent를 만든 뒤 named stage ready file에서 backup subprocess를 실패시키거나 signal합니다. |
| 실제 통과하는 production path | Make/CLI→project lock→writer stop→stream capture→publication/cleanup→service restart 경로를 통과합니다. |
| 핵심 assertion | final/temp/reservation/ready/lock 부재, existing output 보존, service health와 retry 가능성을 확인합니다. |
| 이 테스트가 증명하는 것 | handler를 통과하는 failure/signal에서 all-or-nothing publication과 source recovery가 적용됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | SIGKILL·storage failure·uncooperative external writer는 증명하지 않습니다. |
| 성격 | deterministic negative runtime regression |
| 막는 후속 regression | partial backup 노출, stale lock/control file, cancellation 뒤 application service 중단을 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: SIGKILL, 실제 disk power loss, 모든 filesystem 구현의 durability는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `030e7310c665`가 repeated signal race와 large input/collision edge를 더합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: ordinary failure와 controlled signal이 non-publication, cleanup, source service recovery로 수렴하고 same-project lock이 공유됨을 증명합니다.

### 8. `030e7310c665` — test(backup): 자원 충돌과 시그널 경계 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `PERSISTENCE`, `EDGE` |
| Source-defined role | Extended evidence to signal races, large data, and collision boundaries. |
| 이전 Thread commit | `b6920a0c918c` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Adds signal-race checks, labelled and name-only restore-collision refusal, large filesystem and database fixtures, checksums, and stricter secondary cleanup reporting.
- **Classification reason:** It tests boundary conditions that small normal-path fixtures and simple cancellation cannot cover, protecting the integrity and lifecycle guarantees of backup and restore.

- **이 Thread의 재검토 관점:** 이 문서에서는 signal handoff, streaming size, backup cleanup 관점을 우선 기록하고 restore collision은 연결 정보로만 남깁니다.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `030e7310c665`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `large backup fixtures`에서 streaming path가 작은 fixture를 메모리에 우연히 맞춰 처리한 것이 아님을 확인합니다.
- `tests/runtime_stack.py`의 `repeated pause/signal race`에서 signal mask/ready protocol의 timing contract를 압박합니다.
- `tests/runtime_stack.py`의 `collision fixtures`에서 backup/restore resource identity 검사의 coverage를 넓힙니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 030e7310c665 | tests/runtime_stack.py | large backup fixtures | WordPress에 약 32 MiB file과 MariaDB에 약 4 MiB value를 만들고 backup/restore artifact의 length와 SHA-256을 비교합니다. | streaming path가 작은 fixture를 메모리에 우연히 맞춰 처리한 것이 아님을 확인합니다. |
| 030e7310c665 | tests/runtime_stack.py | repeated pause/signal race | ready publication과 signal 전달 경계를 반복 실행해 marker가 보였는데 process가 아직 pause하지 않았거나 cleanup이 누락되는 race를 탐지합니다. | signal mask/ready protocol의 timing contract를 압박합니다. |
| 030e7310c665 | tests/runtime_stack.py | collision fixtures | stopped·unlabelled container/volume/network와 destination collision을 만들어 label-only 또는 running-only 검사가 놓치는 edge를 검사합니다. | backup/restore resource identity 검사의 coverage를 넓힙니다. |

#### 비교 기준

- exact commit diff: `git diff 030e7310c665^ 030e7310c665 -- <path>`
- 이전 Thread 상태와 비교: `git diff b6920a0c918c 030e7310c665 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | backup/restore stream은 큰 input에서도 truncate하지 않고 signal/collision 경계에서 안전하게 실패합니다. |
| 재현하는 failure / boundary | large artifact, repeated ready/signal race, stopped 또는 unlabelled pre-existing resource입니다. |
| test technique | boundary/large-fixture runtime regression + repeated deterministic signal injection |
| fixture와 failure injection | 32 MiB filesystem file, 4 MiB DB value, collision Docker objects와 반복 signal run을 만듭니다. |
| 실제 통과하는 production path | 실제 backup capture/publication 및 restore validation/injection/resource discovery 경로를 통과합니다. |
| 핵심 assertion | source/restored length·digest, failure non-publication, pre-existing object 보존, cleanup outcome을 확인합니다. |
| 이 테스트가 증명하는 것 | stream-oriented implementation과 broadened collision/signal contract가 작은 정상 처리를 넘어 유지됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 data distribution·filesystem·scheduler interleaving은 증명하지 않습니다. |
| 성격 | large boundary and race regression |
| 막는 후속 regression | whole-buffer 구현, truncation, label-only collision lookup, ready/signal marker race 회귀를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 무한 크기, 모든 signal interleaving, remote filesystem/object store는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: Thread 5의 restore large/collision evidence에도 같은 commit이 사용됩니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: streaming correctness와 pause/signal synchronization이 현실적인 크기와 반복 timing에서도 유지되고 resource collision detection이 label에만 의존하지 않음을 증명합니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| backup output file은 처음부터 private하고 기존 path를 덮어쓰지 않습니다. | fdd55605ba74 | 0540ff1b5a4b | b6920a0c918c | exclusive 0600 file와 destination reservation/identity check, existing-output negative assertion이 연결됩니다. |
| 같은 project의 backup은 다른 mutating management operation과 겹치지 않습니다. | 3a0995ff0d4f | 6999190ffd34 | b6920a0c918c cross-TMPDIR contention | project-name-derived lock을 transaction 전체에서 보유하고 TMPDIR 변경으로 우회되지 않습니다. |
| published backup set은 DB dump, WordPress archive, matching manifest의 완전한 단위입니다. | b478b5243c5a, 0540ff1b5a4b | 6999190ffd34 | b6920a0c918c, 030e7310c665 | private capture→checksum manifest→sync→atomic directory replace와 digest/length assertions가 연결됩니다. |
| 실패하거나 취소된 backup은 plausible final set을 남기지 않고 application services를 복구합니다. | 6999190ffd34 | b6920a0c918c | 030e7310c665 | finally recovery와 negative final/temp/ready/service assertions가 반대 상태를 검증합니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: DB dump와 WordPress archive를 순차적으로 final path에 쓰면 두 artifact 중 하나만 보이는 plausible partial backup과 writer 중단 뒤 degraded source stack이 남을 수 있었습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: private O_EXCL files, fsync/checksum/manifest, inode-checked reservation, application-writer quiescence, directory replacement와 service recovery가 publication boundary를 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `b6920a0c918c` negative scenarios와 `030e7310c665` signal-race/large-fixture checks가 non-publication, cleanup, recovery와 streaming boundary를 보호합니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| DB dump와 filesystem archive 사이 write 또는 partial files 노출 | 6999190ffd34의 writers stop + sibling temp + manifest + atomic replace | b6920a0c918c publication failure/signal non-publication | capture와 publication을 분리하고 final path를 complete set에만 부여합니다. |
| destination validation 뒤 symlink/object replacement | 0540ff1b5a4b parent resolution과 device/inode reservation | unsafe destination/collision runtime cases | pathname equality 대신 actual reserved object identity를 재확인합니다. |
| asynchronous signal로 cleanup timing과 ready marker 불일치 | d26c885c5cd5 signal-to-exception과 masked ready publication | 030e7310c665 repeated pause/signal race | test와 production이 같은 named stage handoff를 사용합니다. |
| small fixture에서만 streaming이 우연히 동작 | b478b5243c5a stream capture | 030e7310c665 32 MiB/4 MiB checksum·length | whole-buffer assumption과 truncation을 별도 boundary test로 막습니다. |

### 직접 재구성할 chain

```text
기존 가정: 각 artifact command가 성공하면 backup directory를 유효하다고 볼 수 있다는 가정
  → 실제 failure 또는 위험: 중간 failure·signal에서 일부 file, temporary sibling, stopped services가 남고 final path가 완전한 set처럼 보일 수 있었습니다.
  → root cause: data capture와 publication이 같은 pathname lifecycle에 섞였고 cancellation이 ordinary failure cleanup과 일치하지 않았습니다.
  → 수정된 invariant / decision: 모든 artifact를 private staging에 capture·sync·checksum한 뒤 manifest를 포함한 directory 전체만 inode-checked destination에 원자 게시합니다.
  → 해당 SHA의 실제 수정 코드: `6999190ffd34`의 stop→capture→manifest→rename→recover transaction
  → failure injection 또는 regression test: `b6920a0c918c` injected failure/signal matrix와 `030e7310c665` race/large fixtures
  → 증명된 보장 / 남은 비보장: success에는 정확한 세 파일만 보이고 failure에는 published set과 temporary state가 없으며 source health가 회복되지만 cross-filesystem rename은 허용하지 않습니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Source stack services | 모두 running | backup이 writer quiescence와 recovery를 일시 소유 | 6999190ffd34 stop/start/finally | MariaDB는 dump 동안 running, Nginx/WordPress는 capture cut을 위해 중지됩니다. |
| Temporary sibling directory | 없음 | unpublished dump/archive/manifest를 독점 보유 | exclusive creation, sync, cleanup | 실패 시 제거되고 final consumer는 볼 수 없습니다. |
| Reserved destination | user path 문자열 | empty private inode가 publication slot 역할 | 0540ff1b5a4b dev/inode check | 정확히 예약한 object만 replace 대상입니다. |
| Published backup set | 개별 artifact | manifest가 artifact identity/checksum을 묶는 authoritative unit | 6999190ffd34 manifest/schema/digest | 세 파일 전체가 restore input 단위입니다. |
| Signal/pause state | OS의 비동기 종료 | normal cleanup path와 deterministic test handoff | d26c885c5cd5 handler/ready lifecycle | signal을 ordinary failure semantics로 수렴시킵니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** The implementation deliberately separates data capture from publication. Private streaming files, an exact output reservation, a manifest, and directory replacement ensure that only a complete set becomes visible. Signal-aware recovery and negative runtime tests establish the equally important converse: cancelled or failed work must not leave a plausible backup or a degraded source stack.
- 최종 authoritative state와 owner: final backup directory와 manifest가 complete set의 authoritative state이며 source MariaDB/WordPress volumes는 원본 state owner로 남습니다.
- 정상 실행의 entry point와 완료 조건: locked backup이 source validation, writer stop, two-stream capture, validation/checksum/sync, atomic publication, service recovery를 끝내면 완료입니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: failure/signal 시 temporary/reservation/control state를 제거하고 source services를 재시작합니다. service recovery failure는 primary failure와 함께 보고합니다.
- 이 Thread가 다른 Thread에 제공하는 전제: Thread 5 restore가 stable descriptor로 열고 검증할 private checksummed input을 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: Docker 미설치 환경에서는 runtime failure matrix를 실행하지 않았으며 코드에 구현된 mechanism만 확인했습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | lock/signal 설정 | 3a0995ff0d4f + d26c885c5cd5 | same-project serialization과 cancellation exception을 설정합니다. | contention/signal이면 mutation 전 또는 common cleanup path로 실패합니다. |
| 2 | source/destination 검증 | 0540ff1b5a4b + 6999190ffd34 | services/secrets와 exact destination reservation을 확인합니다. | unsafe/existing/mismatched path면 writer를 멈추지 않고 거부합니다. |
| 3 | writers quiesce | 6999190ffd34 backup orchestration | Nginx와 WordPress를 중지하고 MariaDB는 transaction dump를 위해 유지합니다. | stop failure면 capture를 진행하지 않고 recovery를 시도합니다. |
| 4 | stream capture | b478b5243c5a helpers | DB dump와 WordPress archive를 private sibling files에 씁니다. | subprocess/stream/archive failure면 temp를 제거합니다. |
| 5 | manifest/durability | fdd55605ba74 + 6999190ffd34 | digest/size/manifest와 file/directory fsync를 완료합니다. | 검증/sync 실패면 final path에 complete set을 게시하지 않습니다. |
| 6 | atomic publication | 0540ff1b5a4b + 6999190ffd34 | reservation identity 후 sibling directory를 final path로 replace합니다. | identity mismatch/rename 실패는 non-publication입니다. |
| 7 | service recovery/cleanup | 6999190ffd34 finally | 성공·실패 모두에서 application services를 healthy로 되돌립니다. | recovery failure는 숨기지 않고 operation result를 실패로 유지합니다. |

### 학습자의 최종 설명

> backup은 파일 두 개를 만드는 작업이 아니라 source writer를 일시 정지하고 complete set을 한 번에 공개하는 transaction입니다. DB는 running MariaDB의 single-transaction stream으로, WordPress는 read-only volume archive stream으로 private sibling에 수집됩니다. output pathname은 미리 private inode로 예약되고 publication 직전 device/inode가 다시 확인됩니다. archive 검증과 checksum manifest, file/directory sync가 끝난 directory만 final path로 atomic replace됩니다. ordinary failure와 SIGINT/SIGTERM은 같은 finally로 들어가 temp/control state를 제거하고 Nginx/WordPress를 복구합니다. negative tests는 “성공한 backup이 맞다”뿐 아니라 “실패한 시도는 plausible set과 degraded source를 남기지 않는다”는 반대 invariant를 검사합니다.

## 학습 완료 자가 점검

- [x] 두 artifact의 생성 성공을 atomic publication과 같은 의미로 썼습니까?
- [x] file fsync와 containing directory fsync의 역할을 구분했습니까?
- [x] MariaDB까지 멈춘다고 잘못 설명하지 않았습니까?
- [x] failure 뒤 source services recovery 실패가 별도 오류로 surfaced되는지 확인했습니까?
- [x] signal test의 ready marker가 sleep 기반 동기화가 아니라는 점을 코드로 확인했습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 04-atomic-backup-publication.md =====

===== BEGIN FILE: 05-fresh-project-restore.md =====
# Thread 5 — Verified fresh-project restore with cleanup rollback

## Thread 목표

backup을 기존 state 위에 덮는 작업이 아니라 완전히 fresh한 Compose project를 생성하는 transaction으로 정의하고, verified input·streaming injection·failure rollback을 통해 all-or-nothing restore를 복원합니다.

**Source significance**

> Restore is treated as creation of a new project, not as an in-place overwrite. That constraint makes rollback tractable: verified input is applied only after collision checks, and any failure removes the resources created by the attempt. The later tests show that refusal preserves pre-existing objects and that the streaming implementation remains correct beyond small fixtures.

## 이 Thread를 이해하기 위한 핵심 질문

- restore target이 반드시 empty namespace여야 rollback ownership이 단순해지는 이유는 무엇입니까?
- Compose label만으로 collision을 찾지 못하는 경우와 exact rendered name만으로 부족한 경우는 무엇입니까?
- backup path를 반복 resolve하지 않고 descriptor-anchored object로 유지하는 이유는 무엇입니까?
- archive validation과 empty-volume precondition은 각각 어떤 write/merge 위험을 막습니까?
- `compose down --volumes` 실행 성공만으로 rollback complete라고 할 수 없는 이유는 무엇입니까?
- restore failure와 cleanup failure가 동시에 발생할 때 오류 context는 어떻게 보존됩니까?

## 완료 기준

- fresh-target detection이 label, exact names, rendered volume/network names를 모두 사용하는 이유를 확인했습니다.
- `VerifiedBackup`이 directory descriptor와 opened streams를 restore 종료까지 유지하는 경계를 추적했습니다.
- SQL import와 WordPress extraction이 empty new volumes에 streaming으로 적용되는 경로를 확인했습니다.
- failure 이후 Compose cleanup과 independent resource enumeration이 모두 통과해야 하는 조건을 기록했습니다.
- malformed input, signal, injected failure, pre-existing collision, successful restore, second restore refusal을 분리했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `e5cb60c7d743` | feat(restore): Compose 리소스 이름과 기존 객체 조회 | **B** | `PERSISTENCE`<br>`OPERATIONS` | Mapped rendered and conventionally named Docker resources. |
| 2 | `851dc1708881` | feat(restore): 대상 프로젝트 자원 충돌 사전 차단 | **A** | `PERSISTENCE`<br>`RISK`<br>`EDGE` | Made an empty target project a restore precondition. |
| 3 | `953a0f6bd571` | feat(restore): 백업 입력의 형식과 체크섬 검증 | **A** | `PERSISTENCE`<br>`RISK`<br>`EDGE` | Established the private, locked, checksummed backup input boundary. |
| 4 | `1250fcf7c006` | feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입 | **B** | `PERSISTENCE`<br>`INTEGRATION` | Injected SQL and WordPress streams into empty new volumes. |
| 5 | `9ca04b1c30cd` | feat(restore): 실패한 복원 자원을 정리하고 롤백 | **S** | `PERSISTENCE`<br>`RECOVERY`<br>`HARD` | Orchestrated startup and removed every partial resource after failure. |
| 6 | `3a37a491ecea` | feat(restore): 복원 CLI와 Make 타깃 연결 | **B** | `PERSISTENCE`<br>`OPERATIONS` | Exposed restore through the CLI and Makefile. |
| 7 | `4f8eb9aff842` | test(restore): 거부·롤백·복원 상태 검증 | **A** | `TEST`<br>`RECOVERY`<br>`RISK` | Verified malformed input refusal, failure cleanup, interruption, and successful state. |
| 8 | `030e7310c665` | test(backup): 자원 충돌과 시그널 경계 검증 | **A** | `TEST`<br>`PERSISTENCE`<br>`EDGE` | Added stopped and unlabelled collision cases plus large restored fixtures. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `e5cb60c7d743` — feat(restore): Compose 리소스 이름과 기존 객체 조회

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `PERSISTENCE`, `OPERATIONS` |
| Source-defined role | Mapped rendered and conventionally named Docker resources. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `851dc1708881` |

#### 원문이 확정한 범위

- **Summary:** Adds discovery of rendered Compose resource names and existing labelled or conventionally named objects.
- **Classification reason:** This is necessary restore plumbing, but it mainly inventories resources within the restore architecture developed by subsequent commits.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `e5cb60c7d743`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `rendered resource-name helpers`에서 restore가 실제 Docker object 이름을 mutation 전에 계산합니다.
- `tools/stack_backup.py`의 `expected container-name candidates`에서 label이 없거나 stopped인 object도 exact name으로 찾을 후보를 확보합니다.
- `tools/stack_backup.py`의 `list labelled/exact resources`에서 한 discovery 방식의 누락을 다른 방식으로 보완합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| e5cb60c7d743 | tools/stack_backup.py | rendered resource-name helpers | rendered Compose JSON에서 volume/network의 concrete name을 읽고 explicit `name`과 project-prefixed default를 구분합니다. | restore가 실제 Docker object 이름을 mutation 전에 계산합니다. |
| e5cb60c7d743 | tools/stack_backup.py | expected container-name candidates | service와 one-off bootstrap container의 current/legacy Compose naming form을 project/service/index 조합으로 만듭니다. | label이 없거나 stopped인 object도 exact name으로 찾을 후보를 확보합니다. |
| e5cb60c7d743 | tools/stack_backup.py | list labelled/exact resources | containers, volumes, networks를 project label과 exact expected name 양쪽으로 query합니다. | 한 discovery 방식의 누락을 다른 방식으로 보완합니다. |

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Mapped rendered and conventionally named Docker resources. |
| 핵심 input / output / state | host restore process가 target project의 expected resource identity와 discovery 결과를 소유합니다. |
| 변경된 directive / helper / command | `tools/stack_backup.py`의 `rendered resource-name helpers`; `tools/stack_backup.py`의 `expected container-name candidates`; `tools/stack_backup.py`의 `list labelled/exact resources` |
| immediate failure 또는 boundary | Compose config rendering 실패, Docker query timeout/nonzero, ambiguous/malformed output은 mutation 전에 restore-domain error가 됩니다. |
| 다음 commit에 넘긴 한계 | 열거 결과를 fresh-target precondition으로 강제하거나 실패 시 cleanup하는 orchestration은 아직 없습니다. `851dc1708881`이 이 inventory를 restore 시작 전 emptiness gate로 사용합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 열거 결과를 fresh-target precondition으로 강제하거나 실패 시 cleanup하는 orchestration은 아직 없습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `851dc1708881`이 이 inventory를 restore 시작 전 emptiness gate로 사용합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: target project에 관련될 수 있는 concrete resource names와 existing objects를 체계적으로 열거할 수 있습니다.

### 2. `851dc1708881` — feat(restore): 대상 프로젝트 자원 충돌 사전 차단

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `PERSISTENCE`, `RISK`, `EDGE` |
| Source-defined role | Made an empty target project a restore precondition. |
| 이전 Thread commit | `e5cb60c7d743` |
| 다음 Thread commit | `953a0f6bd571` |

#### 원문이 확정한 범위

- **Summary:** Rejects restore targets that already contain matching containers, volumes, or networks.
- **Classification reason:** Fresh-project enforcement prevents restore from overwriting or mixing with live state and establishes a significant safety precondition for all later restore steps.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `851dc1708881`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `ensure_fresh_project`에서 stopped·unlabelled·partially created object까지 preflight 범위에 넣습니다.
- `tools/stack_backup.py`의 `collision report`에서 기존 object를 지우거나 덮어쓰는 대신 operator에게 소유권 충돌을 보여줍니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 851dc1708881 | tools/stack_backup.py | ensure_fresh_project | project label로 containers/volumes/networks를 찾고 rendered exact volume/network names와 expected current/legacy container names를 추가로 확인합니다. | stopped·unlabelled·partially created object까지 preflight 범위에 넣습니다. |
| 851dc1708881 | tools/stack_backup.py | collision report | 발견한 kind/name을 모아 하나라도 있으면 restore mutation 전에 명시적인 refusal error를 만듭니다. | 기존 object를 지우거나 덮어쓰는 대신 operator에게 소유권 충돌을 보여줍니다. |

#### 비교 기준

- exact commit diff: `git diff 851dc1708881^ 851dc1708881 -- <path>`
- 이전 Thread 상태와 비교: `git diff e5cb60c7d743 851dc1708881 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 기존 project에 restore하면 unrelated data와 merge/overwrite되고 실패 rollback 때 어떤 object가 원래 있었는지 구분하기 어렵습니다. |
| 선택한 boundary / decision | target namespace가 완전히 비어 있다는 것을 restore transaction의 필수 사전 조건으로 만들었습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/stack_backup.py`의 `ensure_fresh_project`; `tools/stack_backup.py`의 `collision report` |
| state / ownership / lifecycle 변화 | preflight 성공 뒤 restore attempt가 target project에서 새로 생기는 모든 object를 독점 소유한다고 추론할 수 있습니다. |
| 주요 failure branch | labelled/exact/rendered 이름 중 하나라도 존재하면 input을 쓰기 전에 실패합니다. 발견된 기존 object는 수정하거나 제거하지 않습니다. |
| 이 commit의 보장 | restore는 fresh project 생성으로만 시작하며 rollback ownership을 새 attempt가 만든 resource로 한정합니다. |
| 한계와 다음 관련 commit | preflight 직후 외부 actor가 같은 name을 만드는 race를 완전히 막지는 않으며 actual Docker create error도 처리해야 합니다. `9ca04b1c30cd`이 project lock과 cleanup rollback 안에서 이 precondition을 재사용하고 tests가 collision 보존을 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: preflight 직후 외부 actor가 같은 name을 만드는 race를 완전히 막지는 않으며 actual Docker create error도 처리해야 합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `9ca04b1c30cd`이 project lock과 cleanup rollback 안에서 이 precondition을 재사용하고 tests가 collision 보존을 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: restore는 fresh project 생성으로만 시작하며 rollback ownership을 새 attempt가 만든 resource로 한정합니다.

### 3. `953a0f6bd571` — feat(restore): 백업 입력의 형식과 체크섬 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `PERSISTENCE`, `RISK`, `EDGE` |
| Source-defined role | Established the private, locked, checksummed backup input boundary. |
| 이전 Thread commit | `851dc1708881` |
| 다음 Thread commit | `1250fcf7c006` |

#### 원문이 확정한 범위

- **Summary:** Opens a private backup set with no-follow and locking checks, validates its exact files, manifest format, checksums, and archive structure.
- **Classification reason:** This creates the restore trust boundary. It ensures restoration consumes one stable, owner-controlled, internally consistent backup rather than mutable or substituted input.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `953a0f6bd571`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `VerifiedBackup`에서 user pathname을 한 번 검증한 stable directory object로 고정합니다.
- `tools/stack_backup.py`의 `openat-style artifact opens / shared locks`에서 검증 후 pathname substitution과 concurrent writer 변경을 줄입니다.
- `tools/stack_backup.py`의 `manifest/schema/checksum/archive validation`에서 restore mutation 전에 input completeness와 구조를 증명합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 953a0f6bd571 | tools/stack_backup.py | VerifiedBackup | backup directory를 no-follow descriptor로 열고 owner/mode/type/link를 검사한 뒤 expected artifact names만 허용합니다. | user pathname을 한 번 검증한 stable directory object로 고정합니다. |
| 953a0f6bd571 | tools/stack_backup.py | openat-style artifact opens / shared locks | directory descriptor를 기준으로 DB dump, WordPress archive, manifest를 다시 no-follow로 열고 regular/single-link/private 조건과 shared lock을 유지합니다. | 검증 후 pathname substitution과 concurrent writer 변경을 줄입니다. |
| 953a0f6bd571 | tools/stack_backup.py | manifest/schema/checksum/archive validation | manifest의 exact file set, size, SHA-256을 열린 stream과 비교하고 tar member path/type policy를 검사합니다. | restore mutation 전에 input completeness와 구조를 증명합니다. |

#### 비교 기준

- exact commit diff: `git diff 953a0f6bd571^ 953a0f6bd571 -- <path>`
- 이전 Thread 상태와 비교: `git diff 851dc1708881 953a0f6bd571 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | backup pathname을 단계마다 다시 열면 검증된 object와 실제 주입 object가 달라질 수 있고 malformed archive/checksum이 target을 일부 변경할 수 있었습니다. |
| 선택한 boundary / decision | directory와 artifact descriptors를 restore lifetime 동안 유지하는 `VerifiedBackup` boundary를 만들고 모든 형식·checksum 검증을 mutation보다 앞에 배치했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/stack_backup.py`의 `VerifiedBackup`; `tools/stack_backup.py`의 `openat-style artifact opens / shared locks`; `tools/stack_backup.py`의 `manifest/schema/checksum/archive validation` |
| state / ownership / lifecycle 변화 | VerifiedBackup object가 directory/file descriptors, shared locks, stream positions를 소유하고 restore orchestration은이 stable handles만 소비합니다. |
| 주요 failure branch | symlink, wrong owner/mode/link/type, extra/missing file, malformed manifest, size/digest mismatch, unsafe archive member는 target resource 생성 전에 실패합니다. |
| 이 commit의 보장 | restore input이 private owner-controlled exact files의 checksummed structurally valid set이며 검증한 object와 사용하는 object가 같은 descriptor에 anchored됨을 보장합니다. |
| 한계와 다음 관련 commit | backup이 application-consistent하게 생성됐는지는 manifest만으로 다시 증명하지 않으며 그 속성은 Thread 4 publication 과정에 의존합니다. `1250fcf7c006`이 retained streams를 fresh volumes에 직접 주입하고 `4f8eb9aff842`이 malformed/symlink input refusal을 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: backup이 application-consistent하게 생성됐는지는 manifest만으로 다시 증명하지 않으며 그 속성은 Thread 4 publication 과정에 의존합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `1250fcf7c006`이 retained streams를 fresh volumes에 직접 주입하고 `4f8eb9aff842`이 malformed/symlink input refusal을 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: restore input이 private owner-controlled exact files의 checksummed structurally valid set이며 검증한 object와 사용하는 object가 같은 descriptor에 anchored됨을 보장합니다.

### 4. `1250fcf7c006` — feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `PERSISTENCE`, `INTEGRATION` |
| Source-defined role | Injected SQL and WordPress streams into empty new volumes. |
| 이전 Thread commit | `953a0f6bd571` |
| 다음 Thread commit | `9ca04b1c30cd` |

#### 원문이 확정한 범위

- **Summary:** Imports the SQL stream into MariaDB and extracts the WordPress archive only into empty data and config volumes.
- **Classification reason:** It is core restore work, but the implementation follows the already-defined verified-input and fresh-target contracts; rollback and lifecycle safety arrive later.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `1250fcf7c006`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `database restore helper`에서 large dump를 host memory에 전부 올리지 않고 DB service interface로 주입합니다.
- `tools/stack_backup.py`의 `WordPress extraction helper`에서 host가 Docker volume path를 직접 탐색하지 않습니다.
- `tools/stack_backup.py`의 `empty mount checks`에서 기존 state와 archive를 합치는 in-place overwrite를 막습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 1250fcf7c006 | tools/stack_backup.py | database restore helper | fresh MariaDB bootstrap을 완료한 뒤 retained SQL stream을 client stdin으로 전달해 import합니다. | large dump를 host memory에 전부 올리지 않고 DB service interface로 주입합니다. |
| 1250fcf7c006 | tools/stack_backup.py | WordPress extraction helper | new WordPress data/config volumes를 one-off container에 mount하고 validated archive stream을 tar stdin으로 전달합니다. | host가 Docker volume path를 직접 탐색하지 않습니다. |
| 1250fcf7c006 | tools/stack_backup.py | empty mount checks | extraction 전에 target mount roots가 비어 있는지 검사하고 expected root mapping 외 merge를 거부합니다. | 기존 state와 archive를 합치는 in-place overwrite를 막습니다. |

#### 비교 기준

- exact commit diff: `git diff 1250fcf7c006^ 1250fcf7c006 -- <path>`
- 이전 Thread 상태와 비교: `git diff 953a0f6bd571 1250fcf7c006 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Injected SQL and WordPress streams into empty new volumes. |
| 핵심 input / output / state | MariaDB client가 relational import를, one-off tar container가 filesystem extraction을 소유합니다. VerifiedBackup은 source stream descriptors를 끝까지 유지합니다. |
| 변경된 directive / helper / command | `tools/stack_backup.py`의 `database restore helper`; `tools/stack_backup.py`의 `WordPress extraction helper`; `tools/stack_backup.py`의 `empty mount checks` |
| immediate failure 또는 boundary | empty precondition 위반, client/tar timeout/nonzero, broken pipe, extraction policy failure는 partial restore error가 됩니다. |
| 다음 commit에 넘긴 한계 | import 중간 실패 뒤 partial volumes를 제거하거나 full application health까지 수렴하는 rollback은 아직 보장하지 않습니다. `9ca04b1c30cd`이 primitives를 transaction으로 묶고 실패 시 created resources를 제거합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: import 중간 실패 뒤 partial volumes를 제거하거나 full application health까지 수렴하는 rollback은 아직 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `9ca04b1c30cd`이 primitives를 transaction으로 묶고 실패 시 created resources를 제거합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: validated backup streams를 새 DB/WordPress volumes에 bounded-memory 방식으로 주입할 수 있습니다.

### 5. `9ca04b1c30cd` — feat(restore): 실패한 복원 자원을 정리하고 롤백

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **S** |
| Tags | `PERSISTENCE`, `RECOVERY`, `HARD` |
| Source-defined role | Orchestrated startup and removed every partial resource after failure. |
| 이전 Thread commit | `1250fcf7c006` |
| 다음 Thread commit | `3a37a491ecea` |

#### 원문이 확정한 범위

- **Summary:** Orchestrates fresh database bootstrap, SQL import, WordPress extraction, application startup, and complete resource cleanup on any restore failure.
- **Classification reason:** This is the defining restore mechanism and its critical failure invariant. Without it, partial restoration could leave plausible but unusable project resources and make retries unsafe.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `9ca04b1c30cd`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `restore orchestration`에서 input/freshness 검증이 모든 target mutation보다 앞섭니다.
- `tools/stack_backup.py`의 `cleanup_failed_restore`에서 Compose가 아는 created resources의 일차 rollback을 수행합니다.
- `tools/stack_backup.py`의 `independent residual enumeration`에서 cleanup command 성공을 cleanup complete로 오인하지 않습니다.
- `tools/stack_backup.py`의 `error chaining / service convergence`에서 성공은 healthy stack, 실패는 zero-owned-resource state라는 양쪽 endpoint를 정의합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 9ca04b1c30cd | tools/stack_backup.py | restore orchestration | project lock과 signal handler 아래 VerifiedBackup 검증과 fresh-target check를 마친 뒤 MariaDB bootstrap, SQL import, WordPress extraction, normal application bootstrap/start를 순서대로 실행합니다. | input/freshness 검증이 모든 target mutation보다 앞섭니다. |
| 9ca04b1c30cd | tools/stack_backup.py | cleanup_failed_restore | 예외 또는 cancellation 시 `compose down --volumes --remove-orphans`를 실행하고 결과를 검사합니다. | Compose가 아는 created resources의 일차 rollback을 수행합니다. |
| 9ca04b1c30cd | tools/stack_backup.py | independent residual enumeration | down 성공 여부와 별도로 project label, rendered names, exact expected names를 다시 query해 containers/volumes/networks가 0인지 확인합니다. | cleanup command 성공을 cleanup complete로 오인하지 않습니다. |
| 9ca04b1c30cd | tools/stack_backup.py | error chaining / service convergence | primary restore error와 cleanup failure를 모두 보존하고 성공 path는 application services/health까지 완료합니다. | 성공은 healthy stack, 실패는 zero-owned-resource state라는 양쪽 endpoint를 정의합니다. |

#### 비교 기준

- exact commit diff: `git diff 9ca04b1c30cd^ 9ca04b1c30cd -- <path>`
- 이전 Thread 상태와 비교: `git diff 1250fcf7c006 9ca04b1c30cd -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### S-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 commit 직전 상태 | stream import/extraction 중 실패하면 새 DB row, partial files, containers/volumes/networks가 남아 다음 restore를 막거나 incomplete project처럼 보일 수 있었습니다. |
| 해결하려던 문제 | 어느 mutation 이후든 exception/signal은 cleanup path로 갑니다. down failure 또는 residual object는 별도 cleanup error로 보고되며 primary failure를 덮지 않습니다. |
| 기존 설계가 충분하지 않았던 이유 | stream import/extraction 중 실패하면 새 DB row, partial files, containers/volumes/networks가 남아 다음 restore를 막거나 incomplete project처럼 보일 수 있었습니다. 어느 mutation 이후든 exception/signal은 cleanup path로 갑니다. down failure 또는 residual object는 별도 cleanup error로 보고되며 primary failure를 덮지 않습니다. |
| 핵심 결정 | fresh namespace, verified input, ordered streaming injection, normal bootstrap reuse, unconditional compose cleanup와 independent zero-resource verification을 하나의 restore transaction으로 연결했습니다. |
| 주요 caller → callee / producer → consumer | `tools/stack_backup.py`의 `restore orchestration`; `tools/stack_backup.py`의 `cleanup_failed_restore`; `tools/stack_backup.py`의 `independent residual enumeration`; `tools/stack_backup.py`의 `error chaining / service convergence` |
| authoritative state와 publication boundary | preflight 뒤 target namespace의 새 resources는 restore attempt가 독점 소유합니다. 성공하면 새 project가 state owner가 되고 실패하면 owner가 만든 모든 object를 제거해야 합니다. 성공 시 backup state를 가진 healthy fresh project, 실패 시 attempt-owned Docker resource가 하나도 없는 상태를 목표로 하고 검증합니다. |
| ownership / lifetime / responsibility 변화 | preflight 뒤 target namespace의 새 resources는 restore attempt가 독점 소유합니다. 성공하면 새 project가 state owner가 되고 실패하면 owner가 만든 모든 object를 제거해야 합니다. |
| failure scenario와 recovery path | 어느 mutation 이후든 exception/signal은 cleanup path로 갑니다. down failure 또는 residual object는 별도 cleanup error로 보고되며 primary failure를 덮지 않습니다. |
| 이 commit이 보장하는 것 | 성공 시 backup state를 가진 healthy fresh project, 실패 시 attempt-owned Docker resource가 하나도 없는 상태를 목표로 하고 검증합니다. |
| 아직 보장하지 않는 것 | non-cooperating 외부 actor가 동시에 만든 exact object, Docker daemon crash 중 cleanup, physical storage remnants는 완전하게 보장하지 않습니다. |
| 후속 fix / test와 연결 | `4f8eb9aff842`이 malformed/refusal/injected failure/SIGINT/success/second-restore를 검증하고 `030e7310c665`이 collision/large cases를 확장합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: non-cooperating 외부 actor가 동시에 만든 exact object, Docker daemon crash 중 cleanup, physical storage remnants는 완전하게 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `4f8eb9aff842`이 malformed/refusal/injected failure/SIGINT/success/second-restore를 검증하고 `030e7310c665`이 collision/large cases를 확장합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 성공 시 backup state를 가진 healthy fresh project, 실패 시 attempt-owned Docker resource가 하나도 없는 상태를 목표로 하고 검증합니다.

### 6. `3a37a491ecea` — feat(restore): 복원 CLI와 Make 타깃 연결

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `PERSISTENCE`, `OPERATIONS` |
| Source-defined role | Exposed restore through the CLI and Makefile. |
| 이전 Thread commit | `9ca04b1c30cd` |
| 다음 Thread commit | `4f8eb9aff842` |

#### 원문이 확정한 범위

- **Summary:** Adds `restore` CLI dispatch and a guarded Make target.
- **Classification reason:** It exposes the completed restore path without materially changing its correctness or recovery model.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `3a37a491ecea`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `CLI restore subcommand`에서 operator-facing entry point가 internal helper와 같은 transaction을 사용합니다.
- `Makefile`의 `restore target`에서 manual command 조립을 줄이고 documented management path를 제공합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3a37a491ecea | tools/stack_backup.py | CLI restore subcommand | backup path, project, env/Compose inputs를 parse하고 production restore orchestration을 호출하며 domain error를 nonzero exit로 변환합니다. | operator-facing entry point가 internal helper와 같은 transaction을 사용합니다. |
| 3a37a491ecea | Makefile | restore target | 명시적 backup source와 project/env variables를 CLI에 전달하는 target을 추가합니다. | manual command 조립을 줄이고 documented management path를 제공합니다. |

#### 비교 기준

- exact commit diff: `git diff 3a37a491ecea^ 3a37a491ecea -- <path>`
- 이전 Thread 상태와 비교: `git diff 9ca04b1c30cd 3a37a491ecea -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Exposed restore through the CLI and Makefile. |
| 핵심 input / output / state | CLI가 argument validation과 process exit status를 소유하고 transaction 자체의 resource ownership은 restore orchestration에 남습니다. |
| 변경된 directive / helper / command | `tools/stack_backup.py`의 `CLI restore subcommand`; `Makefile`의 `restore target` |
| immediate failure 또는 boundary | 필수 path/variable 누락과 RestoreError는 mutation 여부에 맞는 nonzero status와 stderr로 surfaced됩니다. |
| 다음 commit에 넘긴 한계 | CLI 연결 자체는 restore correctness나 cleanup을 추가로 증명하지 않습니다. `4f8eb9aff842`이 실제 command path로 failure와 success를 실행합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: CLI 연결 자체는 restore correctness나 cleanup을 추가로 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `4f8eb9aff842`이 실제 command path로 failure와 success를 실행합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: documented command path가 동일 lock/input/freshness/rollback semantics를 사용함을 보장합니다.

### 7. `4f8eb9aff842` — test(restore): 거부·롤백·복원 상태 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `RECOVERY`, `RISK` |
| Source-defined role | Verified malformed input refusal, failure cleanup, interruption, and successful state. |
| 이전 Thread commit | `3a37a491ecea` |
| 다음 Thread commit | `030e7310c665` |

#### 원문이 확정한 범위

- **Summary:** Tests symlinked backup rejection, injected and signalled restore failure cleanup, successful data recovery, and refusal to restore twice.
- **Classification reason:** These scenarios validate the restore security and rollback contracts against real Docker resources, significantly increasing confidence in a high-risk mechanism.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `4f8eb9aff842`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `malformed/unsafe backup cases`에서 VerifiedBackup trust boundary의 negative evidence입니다.
- `tests/runtime_stack.py`의 `restore failure injection / SIGINT`에서 partial target resources가 생길 수 있는 실제 rollback path를 통과합니다.
- `tests/runtime_stack.py`의 `zero-resource assertions`에서 `down` 호출 여부가 아니라 rollback endpoint를 확인합니다.
- `tests/runtime_stack.py`의 `success and second restore refusal`에서 fresh-project invariant의 positive/negative 양쪽을 검증합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 4f8eb9aff842 | tests/runtime_stack.py | malformed/unsafe backup cases | manifest checksum/shape를 깨거나 symlink artifact를 넣고 restore가 target mutation 전에 거부되는지 확인합니다. | VerifiedBackup trust boundary의 negative evidence입니다. |
| 4f8eb9aff842 | tests/runtime_stack.py | restore failure injection / SIGINT | DB import 이후, archive extraction 이후, bootstrap 단계 등 mutation 뒤 named failure 또는 signal을 주입합니다. | partial target resources가 생길 수 있는 실제 rollback path를 통과합니다. |
| 4f8eb9aff842 | tests/runtime_stack.py | zero-resource assertions | containers, volumes, networks를 labels와 expected names로 독립 조회해 모두 없어야 한다고 검사합니다. | `down` 호출 여부가 아니라 rollback endpoint를 확인합니다. |
| 4f8eb9aff842 | tests/runtime_stack.py | success and second restore refusal | 복원된 post/option/upload/users/health를 검사하고 같은 target에 두 번째 restore가 기존 state를 바꾸지 않고 거부되는지 확인합니다. | fresh-project invariant의 positive/negative 양쪽을 검증합니다. |

#### 비교 기준

- exact commit diff: `git diff 4f8eb9aff842^ 4f8eb9aff842 -- <path>`
- 이전 Thread 상태와 비교: `git diff 3a37a491ecea 4f8eb9aff842 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | verified input만 fresh target에 적용되며 failure는 zero-owned-resource state, success는 healthy restored stack으로 끝납니다. |
| 재현하는 failure / boundary | malformed/symlink input, DB/archive/bootstrap 이후 injected failure, SIGINT, second restore입니다. |
| test technique | live negative/positive integration + deterministic failure/signal injection |
| fixture와 failure injection | valid backup과 여러 corrupted copies, fresh secondary project를 만들고 mutation stage별 failure를 주입합니다. |
| 실제 통과하는 production path | CLI/Make restore→VerifiedBackup→freshness→stream import/extraction→bootstrap/health 또는 cleanup_failed_restore를 통과합니다. |
| 핵심 assertion | mutation 전 refusal, residual object 0, restored values/health, second restore refusal와 existing-state 보존을 확인합니다. |
| 이 테스트가 증명하는 것 | 주요 failure/cancellation이 all-or-nothing resource endpoint로 수렴하고 success가 실제 application state를 복원함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | daemon crash·hardware storage·모든 concurrent actor를 증명하지 않습니다. |
| 성격 | deterministic rollback and broad restore integration |
| 막는 후속 regression | partial target leak, malformed input after-mutation rejection, in-place second restore, cleanup command 성공만 신뢰하는 회귀를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: Docker daemon crash, physical volume garbage, 모든 malformed tar format, uncooperative concurrent actor는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `030e7310c665`이 stopped/unlabelled collision과 large streaming restore를 보강합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: restore의 주요 all-or-nothing endpoint와 pre-existing state preservation을 실제 Docker resource와 application data로 증명합니다.

### 8. `030e7310c665` — test(backup): 자원 충돌과 시그널 경계 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `PERSISTENCE`, `EDGE` |
| Source-defined role | Added stopped and unlabelled collision cases plus large restored fixtures. |
| 이전 Thread commit | `4f8eb9aff842` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Adds signal-race checks, labelled and name-only restore-collision refusal, large filesystem and database fixtures, checksums, and stricter secondary cleanup reporting.
- **Classification reason:** It tests boundary conditions that small normal-path fixtures and simple cancellation cannot cover, protecting the integrity and lifecycle guarantees of backup and restore.

- **이 Thread의 재검토 관점:** 이 문서에서는 stopped/unlabelled collision refusal, large restored fixtures, secondary cleanup 관점을 우선 기록합니다.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `030e7310c665`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `restore resource collision fixtures`에서 freshness 검사가 label-only 또는 running-only로 축소되는 회귀를 탐지합니다.
- `tests/runtime_stack.py`의 `large restore fixture`에서 SQL/tar streaming injection의 large-input correctness를 검증합니다.
- `tests/runtime_stack.py`의 `secondary close/error propagation`에서 test가 만든 secondary resources의 cleanup도 evidence lifecycle에 포함합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 030e7310c665 | tests/runtime_stack.py | restore resource collision fixtures | project label은 없지만 expected exact name을 가진 stopped container/volume/network와 labelled custom-name object를 만듭니다. | freshness 검사가 label-only 또는 running-only로 축소되는 회귀를 탐지합니다. |
| 030e7310c665 | tests/runtime_stack.py | large restore fixture | backup의 32 MiB file과 4 MiB DB value를 fresh target에 restore한 뒤 source/restored length와 digest를 비교합니다. | SQL/tar streaming injection의 large-input correctness를 검증합니다. |
| 030e7310c665 | tests/runtime_stack.py | secondary close/error propagation | restore target teardown이 실패하면 primary scenario가 성공했더라도 nonzero로 처리되는 branch를 검사합니다. | test가 만든 secondary resources의 cleanup도 evidence lifecycle에 포함합니다. |

#### 비교 기준

- exact commit diff: `git diff 030e7310c665^ 030e7310c665 -- <path>`
- 이전 Thread 상태와 비교: `git diff 4f8eb9aff842 030e7310c665 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | freshness check는 labelled/running object뿐 아니라 stopped/unlabelled exact-name object도 거부하고 large streams를 정확히 복원합니다. |
| 재현하는 failure / boundary | stopped/unlabelled collisions, 32 MiB file, 4 MiB DB value, secondary cleanup failure입니다. |
| test technique | edge-boundary runtime integration + large fixture checksum comparison |
| fixture와 failure injection | expected resource names으로 collision objects를 만들고 large state backup을 fresh target에 적용합니다. |
| 실제 통과하는 production path | resource discovery→freshness refusal 또는 VerifiedBackup→stream injection→application verification을 통과합니다. |
| 핵심 assertion | pre-existing object 보존, mutation 0, source/restored digest·length, cleanup-result precedence를 확인합니다. |
| 이 테스트가 증명하는 것 | label/name discovery가 보완적이고 streaming restore가 small fixture에 국한되지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 새로운 미래 Compose naming form이나 모든 volume driver를 증명하지 않습니다. |
| 성격 | collision and large-stream regression |
| 막는 후속 regression | label-only discovery, running-only lookup, whole-buffer/truncation, secondary cleanup 무시를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 모든 Compose version naming convention과 무한 크기 input을 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: Thread 4와 공유되는 commit이지만 여기서는 restore collision/large-state 관점만 사용합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: fresh-target inventory의 다중 discovery 방식과 bounded-memory restore가 realistic large fixture에서 동작함을 증명합니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| restore는 완전히 비어 있는 target project에만 시작합니다. | e5cb60c7d743, 851dc1708881 | 9ca04b1c30cd | 4f8eb9aff842, 030e7310c665 | labels와 rendered/exact names preflight가 mutation보다 앞서고 collision fixtures가 기존 object 보존을 확인합니다. |
| input backup은 private, owner-controlled, non-symlink, exact-file, checksummed, structurally valid object입니다. | 953a0f6bd571 | 9ca04b1c30cd | 4f8eb9aff842 | VerifiedBackup descriptor/lock/schema/checksum과 malformed/symlink refusal가 연결됩니다. |
| WordPress archive와 SQL은 empty newly created volumes에만 주입됩니다. | 1250fcf7c006 | 9ca04b1c30cd | 4f8eb9aff842 | empty mount checks와 fresh target, second restore refusal가 merge/overwrite를 막습니다. |
| restore 성공은 healthy complete stack, 실패는 owned Docker resource가 하나도 없는 상태입니다. | 9ca04b1c30cd | 4f8eb9aff842 | 030e7310c665 | success data/health assertion과 failure labels/names zero enumeration 및 cleanup-result propagation이 양 endpoint를 고정합니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: 기존 project에 restore하면 pre-existing state와 attempt-created state를 구분할 수 없어 failure rollback이 안전하게 삭제할 범위를 결정할 수 없었습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: label·exact/rendered name collision checks, descriptor-anchored `VerifiedBackup`, empty-volume streaming injection과 independent zero-resource cleanup verification이 fresh-create semantics를 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `4f8eb9aff842`의 malformed/signal/success/second-refusal scenarios와 `030e7310c665`의 stopped/unlabelled collision 및 large fixtures가 보호합니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| 기존 target에 restore하면 unrelated state와 merge/overwrite되고 rollback ownership 불명확 | 851dc1708881 fresh-project precondition | 4f8eb9aff842 second restore refusal, 030e7310c665 collision cases | new attempt가 만든 resources만 독점 소유하도록 namespace를 비웁니다. |
| 검증 뒤 input path가 symlink/modified file로 바뀜 | 953a0f6bd571 descriptor-anchored no-follow open과 retained locks/streams | 4f8eb9aff842 symlink artifact rejection | pathname 재해석 대신 stable open object를 끝까지 소비합니다. |
| DB import 뒤 bootstrap 실패로 partial resources가 남음 | 9ca04b1c30cd compose cleanup + independent enumeration | 4f8eb9aff842 injected failure/SIGINT cleanup | cleanup command 실행과 zero-resource 검증을 분리합니다. |
| small archive/dump에서만 streaming이 통과 | 1250fcf7c006 stdin streaming primitives | 030e7310c665 large restored checksum/length | bounded-memory path와 byte completeness를 실제 large fixture로 확인합니다. |

### 직접 재구성할 chain

```text
기존 가정: restore를 기존 project volume에 덮거나 merge해도 실패 시 되돌릴 수 있다는 가정
  → 실제 failure 또는 위험: partial SQL/archive 적용 뒤 어떤 object가 기존 것인지 판별할 수 없고 retry가 mixed state를 재사용할 수 있었습니다.
  → root cause: fresh-target precondition과 attempt-owned resource set이 없으면 cleanup ownership이 정의되지 않습니다.
  → 수정된 invariant / decision: 검증된 backup을 빈 namespace와 빈 volumes에만 적용하고 어느 failure든 attempt-created containers/volumes/networks를 0으로 되돌립니다.
  → 해당 SHA의 실제 수정 코드: `851dc1708881`, `953a0f6bd571`, `1250fcf7c006`, `9ca04b1c30cd`의 ordered restore path
  → failure injection 또는 regression test: `4f8eb9aff842`와 `030e7310c665` restore scenarios
  → 증명된 보장 / 남은 비보장: success는 healthy restored project, failure는 owned resources 0이며 기존 collision object는 보존하지만 in-place restore는 제공하지 않습니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Verified backup input | user path 문자열 | opened directory/file descriptors와 shared lock이 stable input 소유 | 953a0f6bd571 no-follow/fstat/lock | restore 종료까지 path를 다시 신뢰하지 않습니다. |
| Target namespace | 존재 여부 불명 | freshness 이후 restore attempt가 새 resources 독점 소유 | 851dc1708881 discovery와 9ca04b1c30cd transaction | 실패 시 제거 가능한 ownership이 명확합니다. |
| MariaDB volume | 없음 | fresh bootstrap 후 SQL stream의 authoritative relational state | 1250fcf7c006 import order | 기존 DB와 merge하지 않습니다. |
| WordPress data/config volumes | 없음 | validated archive가 empty mounts에만 extraction | 1250fcf7c006 roots/emptiness/tar | public data와 private config를 분리 복원합니다. |
| Rollback | 단일 down command 시도 가능 | Compose cleanup + independent zero-resource verification | 9ca04b1c30cd cleanup_failed_restore | cleanup failure도 transaction failure입니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** Restore is treated as creation of a new project, not as an in-place overwrite. That constraint makes rollback tractable: verified input is applied only after collision checks, and any failure removes the resources created by the attempt. The later tests show that refusal preserves pre-existing objects and that the streaming implementation remains correct beyond small fixtures.
- 최종 authoritative state와 owner: VerifiedBackup descriptors가 stable source input을, successful fresh Compose project의 named volumes가 restored authoritative state를 소유합니다.
- 정상 실행의 entry point와 완료 조건: lock 아래 input/freshness 검증, DB bootstrap/import, WordPress extraction, normal bootstrap/start, health가 모두 성공하면 완료입니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: 예외/signal은 down --volumes와 independent labels/names enumeration으로 zero-owned-resource state를 요구하며 cleanup failure는 primary error와 함께 보고합니다.
- 이 Thread가 다른 Thread에 제공하는 전제: credential rotation과 operations tests가 사용할 완전한 fresh project 생성 semantics를 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: daemon crash와 physical storage leak, non-cooperating external actor는이 Thread 단독으로 완전 증명하지 않습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | input 검증 | 953a0f6bd571 VerifiedBackup | descriptor/lock/schema/checksum/archive validation을 mutation 전에 끝냅니다. | unsafe input은 target object 0 상태로 거부합니다. |
| 2 | target freshness | 851dc1708881 ensure_fresh_project | labels와 exact/rendered names가 모두 비었는지 확인합니다. | collision이면 기존 object를 보존하고 실패합니다. |
| 3 | MariaDB bootstrap/import | 1250fcf7c006 + 9ca04b1c30cd | fresh DB resources를 만들고 SQL stream을 주입합니다. | import failure는 rollback으로 갑니다. |
| 4 | WordPress extraction | 1250fcf7c006 extraction helper | empty data/config mounts에 validated tar stream을 풉니다. | non-empty/path/tar failure는 rollback으로 갑니다. |
| 5 | application convergence | 9ca04b1c30cd normal startup reuse | WordPress bootstrap와 health-gated services를 시작합니다. | bootstrap/health failure는 created resources를 제거합니다. |
| 6 | rollback | 9ca04b1c30cd cleanup_failed_restore | Compose cleanup 후 independent enumeration으로 0을 증명합니다. | residual object나 cleanup command failure는 별도 오류입니다. |
| 7 | success/second refusal | 4f8eb9aff842 runtime scenario | restored values/health를 확인하고 같은 target 재실행을 거부합니다. | second restore는 existing state를 변경하지 않습니다. |

### 학습자의 최종 설명

> restore를 기존 volume에 덮는 작업으로 보면 failure rollback의 소유권이 불명확해집니다. 이 구현은 target project가 labels와 exact/rendered names 모두에서 비어 있어야만 시작합니다. backup은 directory/file descriptor에 anchored된 `VerifiedBackup`으로 열어 checksum과 archive 구조를 mutation 전에 끝내고, SQL과 tar stream은 새 empty volumes에만 주입합니다. 이후 normal bootstrap와 health를 재사용합니다. 어느 지점에서 실패하거나 signal이 오면 Compose cleanup을 실행한 뒤 별도로 containers/volumes/networks를 다시 열거해 0을 확인합니다. 성공은 healthy restored stack, 실패는 attempt-owned resources 0이라는 두 endpoint로 정의되며, 같은 target의 두 번째 restore는 in-place merge 대신 거부됩니다.

## 학습 완료 자가 점검

- [x] restore를 in-place overwrite로 설명하지 않았습니까?
- [x] labelled resource만 collision으로 본다고 잘못 기록하지 않았습니까?
- [x] backup 검증 뒤 파일을 pathname으로 다시 열어도 안전하다고 가정하지 않았습니까?
- [x] cleanup을 시도한 것과 cleanup 완료를 검증한 것을 구분했습니까?
- [x] successful restore 뒤 같은 target에 재실행이 허용된다고 쓰지 않았습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 05-fresh-project-restore.md =====

===== BEGIN FILE: 06-credential-rotation-and-compensation.md =====
# Thread 6 — Coordinated credential rotation and compensation

## Thread 목표

host secret files, MariaDB accounts, WordPress users, `wp-config.php`에 분산된 credential set을 ordered transition으로 교체하고, post-write failure와 signal interruption에도 이전 verified state로 보상하는 multi-store transaction을 추적합니다.

**Source significance**

> Credentials are represented in four host files, two MariaDB accounts, two WordPress users, and WordPress configuration. The thread therefore evolves from individual mutation primitives to a verified state transition and then to compensation for commands that may change state before failing. Deferring further termination while rollback is active is the key correction that prevents recovery itself from being interrupted.

## 이 Thread를 이해하기 위한 핵심 질문

- 하나의 logical credential generation이 실제로 어느 저장소와 consumer에 분산됩니까?
- command failure가 no mutation을 뜻하지 않는 post-write ambiguity는 어떤 helper/test hook으로 재현됩니까?
- application password를 root password보다 먼저 바꾸는 이유와 rollback 순서는 무엇입니까?
- 새 credential의 성공뿐 아니라 이전 credential의 실패를 확인해야 하는 이유는 무엇입니까?
- host file publication이 per-file atomic이어도 전체 네 파일이 transaction이 아닌 이유는 무엇입니까?
- 첫 signal과 rollback 중 추가 signal을 다르게 처리하는 상태 전이는 어디에 구현됩니까?

## 완료 기준

- 네 host files, MariaDB root/application accounts, WordPress admin/author users, private config의 관계를 그렸습니다.
- 각 mutation primitive의 stdin/no-argument secret boundary와 atomic file publication을 확인했습니다.
- forward rotation ordering과 compensation ordering을 actual call sequence로 비교했습니다.
- positive/negative authentication probes가 actual state를 authoritative하게 판단하는 방식을 기록했습니다.
- 각 failure injection과 double-signal scenario가 어떤 mixed state를 재현하는지 구분했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `a2d20b8c2c03` | feat(secrets): 교체 비밀 파일을 안전하게 읽고 게시 | **A** | `SECRETS`<br>`RISK`<br>`OPERATIONS` | Established safe replacement input and atomic host-file publication. |
| 2 | `832d182743ea` | feat(secrets): MariaDB 계정 비밀번호 원자 교체 | **A** | `SECRETS`<br>`RISK`<br>`INTEGRATION` | Implemented MariaDB application and root credential changes. |
| 3 | `0aa998fdd344` | feat(secrets): WordPress 설정과 사용자 비밀번호 교체 | **A** | `SECRETS`<br>`RISK`<br>`INTEGRATION` | Implemented WordPress configuration and user credential changes. |
| 4 | `64844c583211` | feat(secrets): 신규 자격증명 수용과 기존 값 거부 검증 | **A** | `TEST`<br>`SECRETS`<br>`RISK` | Required replacement credentials to work and previous values to fail. |
| 5 | `c68486d55f30` | feat(secrets): 회전 실패 시 기존 자격증명 복구 | **A** | `SECRETS`<br>`RECOVERY`<br>`HARD` | Added cross-store rollback to the prior verified state. |
| 6 | `9934b478c79a` | feat(secrets): 스택 자격증명 회전 절차 연결 | **S** | `SECRETS`<br>`RECOVERY`<br>`CORE` | Connected the ordered, locked, verified rotation transaction. |
| 7 | `2e6649a7706d` | fix(secrets): 회전 중단과 불명확한 상태를 보상 | **S** | `SECRETS`<br>`RECOVERY`<br>`HARD` | Compensated ambiguous post-write failures and deferred signals during rollback. |
| 8 | `0da35c72add5` | test(secrets): 회전 롤백과 재시도 검증 | **A** | `TEST`<br>`SECRETS`<br>`RECOVERY` | Exercised successful rotation, injected failures, interruption, rollback, leak checks, and retry. |
| 9 | `2557079c2d19` | test(secrets): 회전 후 런타임 비밀 경계 고정 | **B** | `TEST`<br>`SECRETS` | Prevented tests from weakening the steady-state secret boundary. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `a2d20b8c2c03` — feat(secrets): 교체 비밀 파일을 안전하게 읽고 게시

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `SECRETS`, `RISK`, `OPERATIONS` |
| Source-defined role | Established safe replacement input and atomic host-file publication. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `832d182743ea` |

#### 원문이 확정한 범위

- **Summary:** Adds hardened replacement-secret reads and per-file atomic, durable host-secret publication.
- **Classification reason:** This establishes the host filesystem side of credential rotation and prevents partial individual files or unsafe input types from entering a multi-system state transition.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `a2d20b8c2c03`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `read replacement/active secret`에서 rotation source와 current generation의 host-side trust boundary가 같습니다.
- `tools/rotate_secrets.py`의 `publish_secret_file`에서 개별 file은 torn write 없이 한 번에 새 content로 보입니다.
- `tools/rotate_secrets.py`의 `temporary cleanup`에서 per-file publication 실패가 stray credential file을 남기지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| a2d20b8c2c03 | tools/rotate_secrets.py | read replacement/active secret | incoming과 active secret을 no-follow descriptor로 열고 regular file, single link, current owner, `0600`, bounded single-line/password-shape를 검사합니다. | rotation source와 current generation의 host-side trust boundary가 같습니다. |
| a2d20b8c2c03 | tools/rotate_secrets.py | publish_secret_file | target과 같은 directory에 private temporary file을 만들고 write·flush·fsync한 뒤 `os.replace`하고 parent directory를 fsync합니다. | 개별 file은 torn write 없이 한 번에 새 content로 보입니다. |
| a2d20b8c2c03 | tools/rotate_secrets.py | temporary cleanup | write/sync/replace 실패마다 아직 남은 temporary file을 unlink하고 original target을 보존합니다. | per-file publication 실패가 stray credential file을 남기지 않습니다. |

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | replacement value와 active host file을 일반 pathname/read/write로 다루면 symlink·permission·partial write·existing temp 위험이 있었습니다. |
| 선택한 boundary / decision | input과 current file 모두 hardened read를 사용하고 각 target은 same-directory atomic durable replace로 게시했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/rotate_secrets.py`의 `read replacement/active secret`; `tools/rotate_secrets.py`의 `publish_secret_file`; `tools/rotate_secrets.py`의 `temporary cleanup` |
| state / ownership / lifecycle 변화 | helper가 한 secret file의 temp/descriptor/replace lifetime을 소유합니다. orchestrator는 네 file의 logical generation ordering을 별도로 소유해야 합니다. |
| 주요 failure branch | unsafe input/target, write/fsync/replace/parent-sync failure는 해당 file publication을 실패시키고 temp를 정리합니다. |
| 이 commit의 보장 | 각 host secret file이 private regular single-link이며 개별 replacement가 atomic/durable하다는 것을 보장합니다. |
| 한계와 다음 관련 commit | 네 files와 DB/WordPress stores 전체가 한 번에 바뀌는 global transaction은 제공하지 않습니다. `9934b478c79a`가 per-file primitive를 ordered multi-store transition에 넣고 `2e6649a7706d`가 post-write ambiguity를 보상합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 네 files와 DB/WordPress stores 전체가 한 번에 바뀌는 global transaction은 제공하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `9934b478c79a`가 per-file primitive를 ordered multi-store transition에 넣고 `2e6649a7706d`가 post-write ambiguity를 보상합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 각 host secret file이 private regular single-link이며 개별 replacement가 atomic/durable하다는 것을 보장합니다.

### 2. `832d182743ea` — feat(secrets): MariaDB 계정 비밀번호 원자 교체

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `SECRETS`, `RISK`, `INTEGRATION` |
| Source-defined role | Implemented MariaDB application and root credential changes. |
| 이전 Thread commit | `a2d20b8c2c03` |
| 다음 Thread commit | `0aa998fdd344` |

#### 원문이 확정한 범위

- **Summary:** Adds root-authenticated MariaDB SQL execution and coordinated application/root password changes through private option files.
- **Classification reason:** This implements a high-risk part of credential rotation while keeping credentials out of process arguments and preserving SQL literal correctness.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `832d182743ea`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `change_application_db_password`에서 public TCP/argv에 새 password를 노출하지 않고 DB-owned account state를 변경합니다.
- `tools/rotate_secrets.py`의 `change_root_db_password`에서 root credential 변경 시점과 application 변경 시점을 orchestrator가 분리할 수 있습니다.
- `tools/rotate_secrets.py`의 `bounded subprocess / cleanup`에서 mutation command lifecycle과 secret-bearing temporary state를 제한합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 832d182743ea | tools/rotate_secrets.py | change_application_db_password | local MariaDB socket에서 current privileged credential로 application account의 password를 바꾸고 identifier/literal을 안전하게 구성합니다. | public TCP/argv에 새 password를 노출하지 않고 DB-owned account state를 변경합니다. |
| 832d182743ea | tools/rotate_secrets.py | change_root_db_password | root account 변경을 별도 primitive로 두고 temporary private client options/stdin을 사용해 password가 process argument에 나타나지 않게 합니다. | root credential 변경 시점과 application 변경 시점을 orchestrator가 분리할 수 있습니다. |
| 832d182743ea | tools/rotate_secrets.py | bounded subprocess / cleanup | DB command timeout/nonzero와 temporary option file cleanup을 domain error로 처리합니다. | mutation command lifecycle과 secret-bearing temporary state를 제한합니다. |

#### 비교 기준

- exact commit diff: `git diff 832d182743ea^ 832d182743ea -- <path>`
- 이전 Thread 상태와 비교: `git diff a2d20b8c2c03 832d182743ea -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | host files만 바꾸면 MariaDB가 저장한 application/root password와 consumer file이 불일치합니다. |
| 선택한 boundary / decision | application account와 root account를 별도 local-socket mutation primitive로 만들고 secret을 argv가 아닌 private input으로 전달했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/rotate_secrets.py`의 `change_application_db_password`; `tools/rotate_secrets.py`의 `change_root_db_password`; `tools/rotate_secrets.py`의 `bounded subprocess / cleanup` |
| state / ownership / lifecycle 변화 | MariaDB가 실제 authentication state를 소유하며 host helper는 command 실행과 temporary credential material을 잠시 소유합니다. |
| 주요 failure branch | client nonzero/timeout이 mutation 전인지 후인지이 commit만으로 구분되지 않습니다. command가 write 후 실패 status를 낼 수 있습니다. |
| 이 commit의 보장 | DB application/root credentials를 순서 제어 가능한 primitive로 교체하고 process arguments에 password를 넣지 않습니다. |
| 한계와 다음 관련 commit | WordPress config/users, host files, new/old authentication verification, ambiguous command result compensation은 보장하지 않습니다. `64844c583211`이 positive/negative probes를, `9934b478c79a`가 application-first/root-last ordering을 적용합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: WordPress config/users, host files, new/old authentication verification, ambiguous command result compensation은 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `64844c583211`이 positive/negative probes를, `9934b478c79a`가 application-first/root-last ordering을 적용합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: DB application/root credentials를 순서 제어 가능한 primitive로 교체하고 process arguments에 password를 넣지 않습니다.

### 3. `0aa998fdd344` — feat(secrets): WordPress 설정과 사용자 비밀번호 교체

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `SECRETS`, `RISK`, `INTEGRATION` |
| Source-defined role | Implemented WordPress configuration and user credential changes. |
| 이전 Thread commit | `832d182743ea` |
| 다음 Thread commit | `64844c583211` |

#### 원문이 확정한 범위

- **Summary:** Adds atomic `wp-config.php` DB-password replacement and WordPress administrator/author password changes.
- **Classification reason:** The commit coordinates filesystem configuration and application database state, establishing the WordPress side of the cross-subsystem rotation problem.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `0aa998fdd344`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `replace_wp_config_password`에서 WordPress DB consumer config를 partial write 없이 갱신합니다.
- `tools/rotate_secrets.py`의 `set_wordpress_user_password`에서 해시를 직접 조작하지 않고 WordPress가 user password state를 소유합니다.
- `tools/rotate_secrets.py`의 `admin/author primitives`에서 두 user 중 일부만 바뀌는 stage를 orchestrator와 tests가 식별할 수 있습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 0aa998fdd344 | tools/rotate_secrets.py | replace_wp_config_password | private config file이 regular/private이며 DB password define이 정확히 한 번 존재하는지 검사하고 same-filesystem temporary+replace로 값을 교체합니다. | WordPress DB consumer config를 partial write 없이 갱신합니다. |
| 0aa998fdd344 | tools/rotate_secrets.py | set_wordpress_user_password | WordPress runtime에서 application API `wp_set_password`를 호출하는 PHP/WP command에 user/password payload를 stdin으로 전달합니다. | 해시를 직접 조작하지 않고 WordPress가 user password state를 소유합니다. |
| 0aa998fdd344 | tools/rotate_secrets.py | admin/author primitives | admin과 author account를 독립 mutation/verification 대상으로 유지합니다. | 두 user 중 일부만 바뀌는 stage를 orchestrator와 tests가 식별할 수 있습니다. |

#### 비교 기준

- exact commit diff: `git diff 0aa998fdd344^ 0aa998fdd344 -- <path>`
- 이전 Thread 상태와 비교: `git diff 832d182743ea 0aa998fdd344 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | DB account가 바뀌어도 `wp-config.php`와 WordPress admin/author credential이 old generation이면 stack 전체의 logical credential set은 일치하지 않습니다. |
| 선택한 boundary / decision | private config와 두 application users를 WordPress-owned interface와 atomic file replacement를 통해 바꾸는 primitive를 추가했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/rotate_secrets.py`의 `replace_wp_config_password`; `tools/rotate_secrets.py`의 `set_wordpress_user_password`; `tools/rotate_secrets.py`의 `admin/author primitives` |
| state / ownership / lifecycle 변화 | WordPress DB가 user hashes를, private config volume이 DB consumer credential을 소유합니다. helper는 mutation command와 config temp file만 일시 소유합니다. |
| 주요 failure branch | config format/duplicate define, WordPress command timeout/nonzero, temporary publication failure가 발생할 수 있고 command nonzero가 no mutation을 뜻하지는 않습니다. |
| 이 commit의 보장 | WordPress config, admin, author credential을 process argument 노출 없이 별도 단계로 교체할 수 있습니다. |
| 한계와 다음 관련 commit | DB accounts/host files와의 global ordering·rollback·old value rejection은 보장하지 않습니다. `64844c583211`이 actual consumer probes를 만들고 `9934b478c79a`가 전체 ordering에 연결합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: DB accounts/host files와의 global ordering·rollback·old value rejection은 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `64844c583211`이 actual consumer probes를 만들고 `9934b478c79a`가 전체 ordering에 연결합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: WordPress config, admin, author credential을 process argument 노출 없이 별도 단계로 교체할 수 있습니다.

### 4. `64844c583211` — feat(secrets): 신규 자격증명 수용과 기존 값 거부 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `SECRETS`, `RISK` |
| Source-defined role | Required replacement credentials to work and previous values to fail. |
| 이전 Thread commit | `0aa998fdd344` |
| 다음 Thread commit | `c68486d55f30` |

#### 원문이 확정한 범위

- **Summary:** Verifies new credentials work, old credentials fail, configuration matches, and no accepted or rejected value leaks into runtime metadata.
- **Classification reason:** Successful rotation requires both positive and negative authentication evidence; this commit makes that state transition verifiable rather than inferred from command success.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `64844c583211`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `MariaDB auth probes`에서 command exit status가 아니라 실제 DB consumer interface가 authoritative state를 판단합니다.
- `tools/rotate_secrets.py`의 `WordPress user probes`에서 새 값만 성공하는 one-generation state를 요구합니다.
- `tools/rotate_secrets.py`의 `config/file equality checks`에서 runtime store와 host source의 generation 일치를 함께 봅니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 64844c583211 | tools/rotate_secrets.py | MariaDB auth probes | root/application 각각 new credential로 local/service authentication이 성공하고 old credential은 실패해야 한다고 검사합니다. | command exit status가 아니라 실제 DB consumer interface가 authoritative state를 판단합니다. |
| 64844c583211 | tools/rotate_secrets.py | WordPress user probes | admin/author의 new password가 WordPress authentication API에서 맞고 old password가 틀린지 양방향으로 확인합니다. | 새 값만 성공하는 one-generation state를 요구합니다. |
| 64844c583211 | tools/rotate_secrets.py | config/file equality checks | private config와 active host secret files가 expected generation의 exact values인지 확인합니다. | runtime store와 host source의 generation 일치를 함께 봅니다. |

#### 비교 기준

- exact commit diff: `git diff 64844c583211^ 64844c583211 -- <path>`
- 이전 Thread 상태와 비교: `git diff 0aa998fdd344 64844c583211 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | mutation command가 성공해도 old credential이 alias/다른 account scope로 계속 작동하거나 일부 consumer가 old file을 사용할 수 있었습니다. |
| 선택한 boundary / decision | 각 store를 실제 인증/consumer interface로 positive(new works)와 negative(old fails) 양쪽에서 확인했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/rotate_secrets.py`의 `MariaDB auth probes`; `tools/rotate_secrets.py`의 `WordPress user probes`; `tools/rotate_secrets.py`의 `config/file equality checks` |
| state / ownership / lifecycle 변화 | verification layer가 actual state를 authoritative하게 판정하고 orchestrator는 성공 결정을 내리기 전에 모든 probe를 통과해야 합니다. |
| 주요 failure branch | new failure 또는 old success 하나라도 mixed/ambiguous generation으로 간주해 rotation을 실패시킵니다. |
| 이 commit의 보장 | 성공 판정은 replacement가 작동한다는 것뿐 아니라 previous generation이 거부되고 files/config가 일치한다는 것까지 포함합니다. |
| 한계와 다음 관련 commit | 실패 시 prior state로 되돌리는 compensation은 아직 없습니다. `c68486d55f30`이 이 probes를 rollback 완료 판단에도 사용합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 실패 시 prior state로 되돌리는 compensation은 아직 없습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `c68486d55f30`이 이 probes를 rollback 완료 판단에도 사용합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 성공 판정은 replacement가 작동한다는 것뿐 아니라 previous generation이 거부되고 files/config가 일치한다는 것까지 포함합니다.

### 5. `c68486d55f30` — feat(secrets): 회전 실패 시 기존 자격증명 복구

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `SECRETS`, `RECOVERY`, `HARD` |
| Source-defined role | Added cross-store rollback to the prior verified state. |
| 이전 Thread commit | `64844c583211` |
| 다음 Thread commit | `9934b478c79a` |

#### 원문이 확정한 범위

- **Summary:** Adds compensation that restores database accounts, WordPress configuration and users, host files, and a verified running stack after rotation failure.
- **Classification reason:** This is significant multi-store rollback engineering, though the following correction handles additional ambiguous command and signal states not yet covered here.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `c68486d55f30`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `compensation helpers`에서 partial new generation을 old verified generation으로 수렴시키는 경로가 생깁니다.
- `tools/rotate_secrets.py`의 `rollback verification`에서 rollback command 호출만으로 완료를 간주하지 않습니다.
- `tools/rotate_secrets.py`의 `rollback error aggregation`에서 첫 rollback 오류가 이후 repair 시도를 중단하거나 원인을 덮지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| c68486d55f30 | tools/rotate_secrets.py | compensation helpers | forward stage별 mutation을 추적하고 failure 뒤 WordPress users/config, MariaDB accounts, host files를 previous values로 되돌리는 reverse operations를 정의합니다. | partial new generation을 old verified generation으로 수렴시키는 경로가 생깁니다. |
| c68486d55f30 | tools/rotate_secrets.py | rollback verification | old values가 다시 작동하고 new values가 거부되며 files/config가 old generation과 같은지 probes로 확인합니다. | rollback command 호출만으로 완료를 간주하지 않습니다. |
| c68486d55f30 | tools/rotate_secrets.py | rollback error aggregation | 여러 compensation 실패를 모아 primary rotation failure와 함께 보고합니다. | 첫 rollback 오류가 이후 repair 시도를 중단하거나 원인을 덮지 않습니다. |

#### 비교 기준

- exact commit diff: `git diff c68486d55f30^ c68486d55f30 -- <path>`
- 이전 Thread 상태와 비교: `git diff 64844c583211 c68486d55f30 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 일부 store가 new generation으로 바뀐 뒤 다음 step이 실패하면 host files, DB, WordPress가 mixed state에 남았습니다. |
| 선택한 boundary / decision | forward mutation을 추적하고 reverse compensation과 prior-generation verification을 추가했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/rotate_secrets.py`의 `compensation helpers`; `tools/rotate_secrets.py`의 `rollback verification`; `tools/rotate_secrets.py`의 `rollback error aggregation` |
| state / ownership / lifecycle 변화 | orchestrator가 logical generation 전환 책임을 가지며 각 subsystem helper는 reversible mutation만 수행합니다. |
| 주요 failure branch | rollback step 자체가 실패할 수 있으므로 all-or-nothing을 선언하지 않고 incomplete compensation을 명시적으로 보고합니다. |
| 이 commit의 보장 | ordinary failure 뒤 가능한 한 previous verified generation으로 되돌리고 실제 old/new probes로 결과를 판정합니다. |
| 한계와 다음 관련 commit | post-write command failure의 실제 mutation 여부와 rollback 중 signal interruption은 아직 충분히 처리하지 않습니다. `2e6649a7706d`이 ambiguous post-write failure와 rollback-active signal state를 교정합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: post-write command failure의 실제 mutation 여부와 rollback 중 signal interruption은 아직 충분히 처리하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `2e6649a7706d`이 ambiguous post-write failure와 rollback-active signal state를 교정합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: ordinary failure 뒤 가능한 한 previous verified generation으로 되돌리고 실제 old/new probes로 결과를 판정합니다.

### 6. `9934b478c79a` — feat(secrets): 스택 자격증명 회전 절차 연결

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **S** |
| Tags | `SECRETS`, `RECOVERY`, `CORE` |
| Source-defined role | Connected the ordered, locked, verified rotation transaction. |
| 이전 Thread commit | `c68486d55f30` |
| 다음 Thread commit | `2e6649a7706d` |

#### 원문이 확정한 범위

- **Summary:** Coordinates the complete credential rotation sequence, serializes it, recreates services, verifies new values and rejection of old ones, and invokes compensation on failure.
- **Classification reason:** Credential rotation is a defining management mechanism spanning four host files, MariaDB accounts, WordPress users, and application configuration; this commit establishes that transaction.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `9934b478c79a`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `rotate / project_operation_lock`에서 startup/backup/restore와 rotation이 같은 serialization domain을 공유합니다.
- `tools/rotate_secrets.py`의 `forward order`에서 repair에 필요한 privileged/current path를 너무 일찍 끊지 않도록 root와 host publication을 뒤에 둡니다.
- `tools/rotate_secrets.py`의 `force recreate / end verification`에서 새 process가 new generation을 실제로 소비하는지 검증합니다.
- `tools/rotate_secrets.py`의 `compensating failure path`에서 multi-store transition의 실패 endpoint를 정의합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 9934b478c79a | tools/rotate_secrets.py | rotate / project_operation_lock | replacement/active secrets와 current runtime state를 검증하고 project lock을 잡은 채 전체 transition을 실행합니다. | startup/backup/restore와 rotation이 같은 serialization domain을 공유합니다. |
| 9934b478c79a | tools/rotate_secrets.py | forward order | public request를 닫기 위해 Nginx를 멈춘 뒤 WordPress users와 private config, MariaDB application password, MariaDB root password, host files 순으로 전환합니다. | repair에 필요한 privileged/current path를 너무 일찍 끊지 않도록 root와 host publication을 뒤에 둡니다. |
| 9934b478c79a | tools/rotate_secrets.py | force recreate / end verification | active files 게시 뒤 services를 force-recreate하고 new works/old fails/files match/runtime boundary/no leak를 확인합니다. | 새 process가 new generation을 실제로 소비하는지 검증합니다. |
| 9934b478c79a | tools/rotate_secrets.py | compensating failure path | 어느 stage 실패든 previous generation compensation과 service recovery를 시도하고 incomplete rollback을 surfaced합니다. | multi-store transition의 실패 endpoint를 정의합니다. |

#### 비교 기준

- exact commit diff: `git diff 9934b478c79a^ 9934b478c79a -- <path>`
- 이전 Thread 상태와 비교: `git diff c68486d55f30 9934b478c79a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### S-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 commit 직전 상태 | 개별 host/DB/WordPress primitives가 있어도 caller별 ordering이 다르면 root access가 먼저 끊기거나 public requests가 mixed generation을 관측할 수 있었습니다. |
| 해결하려던 문제 | stage failure는 compensation으로 이동합니다. 그러나 subprocess가 state를 바꾼 뒤 nonzero를 반환하면 tracked stage만으로 actual mutation을 놓칠 수 있고 signal이 rollback을 중단할 수 있습니다. |
| 기존 설계가 충분하지 않았던 이유 | 개별 host/DB/WordPress primitives가 있어도 caller별 ordering이 다르면 root access가 먼저 끊기거나 public requests가 mixed generation을 관측할 수 있었습니다. stage failure는 compensation으로 이동합니다. 그러나 subprocess가 state를 바꾼 뒤 nonzero를 반환하면 tracked stage만으로 actual mutation을 놓칠 수 있고 signal이 rollback을 중단할 수 있습니다. |
| 핵심 결정 | same-project lock, pre-verification, Nginx quiescence, repair-friendly forward order, host publication, recreate, global probes, compensation을 하나의 procedure로 고정했습니다. |
| 주요 caller → callee / producer → consumer | `tools/rotate_secrets.py`의 `rotate / project_operation_lock`; `tools/rotate_secrets.py`의 `forward order`; `tools/rotate_secrets.py`의 `force recreate / end verification`; `tools/rotate_secrets.py`의 `compensating failure path` |
| authoritative state와 publication boundary | rotation orchestrator가 logical credential generation과 service lifecycle을 소유합니다. 각 store는 자신의 representation을 소유하되 transaction success/failure 판정은 orchestrator가 합니다. ordinary 성공 path에서 네 host files, 두 DB accounts, 두 WP users, private config가 한 new generation으로 전환되고 services가 이를 소비하며 old values가 거부됩니다. |
| ownership / lifetime / responsibility 변화 | rotation orchestrator가 logical credential generation과 service lifecycle을 소유합니다. 각 store는 자신의 representation을 소유하되 transaction success/failure 판정은 orchestrator가 합니다. |
| failure scenario와 recovery path | stage failure는 compensation으로 이동합니다. 그러나 subprocess가 state를 바꾼 뒤 nonzero를 반환하면 tracked stage만으로 actual mutation을 놓칠 수 있고 signal이 rollback을 중단할 수 있습니다. |
| 이 commit이 보장하는 것 | ordinary 성공 path에서 네 host files, 두 DB accounts, 두 WP users, private config가 한 new generation으로 전환되고 services가 이를 소비하며 old values가 거부됩니다. |
| 아직 보장하지 않는 것 | distributed stores 전체의 진정한 atomic commit은 아니며 ambiguous write/result와 rollback-interruption 위험이 남습니다. |
| 후속 fix / test와 연결 | `2e6649a7706d`이 이 두 핵심 failure assumption을 수정하고 `0da35c72add5`가 stage matrix/double signal/retry를 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: distributed stores 전체의 진정한 atomic commit은 아니며 ambiguous write/result와 rollback-interruption 위험이 남습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `2e6649a7706d`이 이 두 핵심 failure assumption을 수정하고 `0da35c72add5`가 stage matrix/double signal/retry를 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: ordinary 성공 path에서 네 host files, 두 DB accounts, 두 WP users, private config가 한 new generation으로 전환되고 services가 이를 소비하며 old values가 거부됩니다.

### 7. `2e6649a7706d` — fix(secrets): 회전 중단과 불명확한 상태를 보상

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **S** |
| Tags | `SECRETS`, `RECOVERY`, `HARD` |
| Source-defined role | Compensated ambiguous post-write failures and deferred signals during rollback. |
| 이전 Thread commit | `9934b478c79a` |
| 다음 Thread commit | `0da35c72add5` |

#### 원문이 확정한 범위

- **Summary:** Adds stage-level failure injection, interruption handling, ambiguous post-write compensation, and deferred signals during rollback.
- **Classification reason:** This corrects non-obvious partial-state hazards in the rotation transaction. It is essential to explaining how the project prevents operator cancellation or uncertain command outcomes from interrupting recovery itself.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `2e6649a7706d`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `post-write failure hooks`에서 “nonzero/exception이면 mutation 없음”이라는 잘못된 가정을 노출합니다.
- `tools/rotate_secrets.py`의 `actual-state probes before compensation`에서 보상 대상을 observed state에서 계산합니다.
- `tools/rotate_secrets.py`의 `rotation signal state machine`에서 recovery 자체가 두 번째 operator signal로 끊기는 것을 막습니다.
- `tools/rotate_secrets.py`의 `deferred signal/result reporting`에서 복구 완료와 process exit semantics를 분리합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2e6649a7706d | tools/rotate_secrets.py | post-write failure hooks | 각 persistent mutation 직후 test hook이 failure를 발생시켜 command가 state를 바꿨지만 caller가 failure를 받은 상황을 재현합니다. | “nonzero/exception이면 mutation 없음”이라는 잘못된 가정을 노출합니다. |
| 2e6649a7706d | tools/rotate_secrets.py | actual-state probes before compensation | tracked stage가 아니라 DB/WP authentication, config/file values를 다시 probe해 old/new/ambiguous actual state를 판정합니다. | 보상 대상을 observed state에서 계산합니다. |
| 2e6649a7706d | tools/rotate_secrets.py | rotation signal state machine | 첫 SIGINT/SIGTERM은 forward transition을 중단시키지만 rollback-active가 된 뒤 추가 termination signal은 기록만 하고 compensation 완료까지 즉시 종료하지 않습니다. | recovery 자체가 두 번째 operator signal로 끊기는 것을 막습니다. |
| 2e6649a7706d | tools/rotate_secrets.py | deferred signal/result reporting | rollback verification을 끝낸 뒤 pending signal과 primary/rollback errors를 정해진 precedence로 보고합니다. | 복구 완료와 process exit semantics를 분리합니다. |

#### 비교 기준

- exact commit diff: `git diff 2e6649a7706d^ 2e6649a7706d -- <path>`
- 이전 Thread 상태와 비교: `git diff 9934b478c79a 2e6649a7706d -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | subprocess/step failure는 해당 persistent mutation이 일어나지 않았다고 가정했습니다. |
| 실제 failure 또는 위험 | store write 후 timeout/nonzero/exception이 가능해 tracked state와 actual credential generation이 다를 수 있고 rollback 중 signal이 복구를 끊을 수 있었습니다. |
| root cause | exit status 중심 stage tracking과 forward/rollback을 구분하지 않는 signal handling이 root cause였습니다. |
| 수정된 invariant / decision | actual authentication/file probes로 state를 재구성하고 rollback-active 동안 추가 signal을 defer한 뒤 prior generation을 보상합니다. |
| 실제 수정 코드 | `tools/rotate_secrets.py`의 `post-write failure hooks`; `tools/rotate_secrets.py`의 `actual-state probes before compensation`; `tools/rotate_secrets.py`의 `rotation signal state machine`; `tools/rotate_secrets.py`의 `deferred signal/result reporting` |
| 변경된 ordering / ownership / lifecycle | orchestrator가 forward/rollback/pending-signal state를 소유합니다. subsystem command result가 아니라 authoritative probes가 compensation order를 결정합니다. |
| 이 fix가 보장하는 것 | command exit와 actual mutation이 어긋나도 observed state에 맞춰 prior generation을 복구하며, recovery 중 추가 termination이 compensation을 중단하지 않습니다. |
| 아직 보장하지 않는 것 | process SIGKILL, host power loss, rollback primitive 자체가 모두 실패하는 경우 prior generation을 반드시 복원한다고 보장하지 않습니다. |
| 연결되는 regression test | post-write failure matrix와 double-signal runtime scenario가 corrected invariant와 retry 가능성을 고정합니다. `0da35c72add5`이 every-persistent-stage post-write failure와 SIGTERM→rollback-active SIGINT, retry를 live runtime에서 검증합니다. |

#### S-level state transition 기록

| 단계 | 학습자 기록 |
| --- | --- |
| correction 전 authoritative state | `9934b478c79a`는 command failure를 no mutation으로 해석할 수 있었고 첫 signal 뒤 rollback 중 추가 signal이 process를 끝내 mixed state를 고착시킬 수 있었습니다. |
| partial / ambiguous state 종류 | store write 후 timeout/nonzero/exception이 가능해 tracked state와 actual credential generation이 다를 수 있고 rollback 중 signal이 복구를 끊을 수 있었습니다. |
| publication 또는 commit boundary | 각 write 뒤 ambiguity를 failure injection으로 모델링하고 actual consumer probes로 state를 탐색하며 rollback-active 동안 signal을 지연하는 state machine을 도입했습니다. |
| rollback / compensation 진입 조건 | post-write exception, first signal, rollback step failure, rollback-active second signal을 모두 compensation/error aggregation 경로로 보냅니다. incomplete recovery는 성공으로 숨기지 않습니다. |
| recovery 중 보호되는 invariant | orchestrator가 forward/rollback/pending-signal state를 소유합니다. subsystem command result가 아니라 authoritative probes가 compensation order를 결정합니다. |
| 성공 endpoint | command exit와 actual mutation이 어긋나도 observed state에 맞춰 prior generation을 복구하며, recovery 중 추가 termination이 compensation을 중단하지 않습니다. |
| 실패 endpoint | process SIGKILL, host power loss, rollback primitive 자체가 모두 실패하는 경우 prior generation을 반드시 복원한다고 보장하지 않습니다. |
| 후속 regression evidence | `0da35c72add5`이 every-persistent-stage post-write failure와 SIGTERM→rollback-active SIGINT, retry를 live runtime에서 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: process SIGKILL, host power loss, rollback primitive 자체가 모두 실패하는 경우 prior generation을 반드시 복원한다고 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `0da35c72add5`이 every-persistent-stage post-write failure와 SIGTERM→rollback-active SIGINT, retry를 live runtime에서 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: command exit와 actual mutation이 어긋나도 observed state에 맞춰 prior generation을 복구하며, recovery 중 추가 termination이 compensation을 중단하지 않습니다.

### 8. `0da35c72add5` — test(secrets): 회전 롤백과 재시도 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `SECRETS`, `RECOVERY` |
| Source-defined role | Exercised successful rotation, injected failures, interruption, rollback, leak checks, and retry. |
| 이전 Thread commit | `2e6649a7706d` |
| 다음 Thread commit | `2557079c2d19` |

#### 원문이 확정한 범위

- **Summary:** Exercises successful rotation, multiple post-write failures, signal interruption during host-file publication, rollback, leak checks, and retry with the same inputs.
- **Classification reason:** The scenario provides strong real-system evidence for one of the project's hardest state transitions and protects against regressions in compensation ordering.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `0da35c72add5`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `rotation success scenario`에서 complete forward path를 실제 consumer interface로 검증합니다.
- `tests/runtime_stack.py`의 `failure-stage matrix`에서 ambiguous state compensation coverage를 만듭니다.
- `tests/runtime_stack.py`의 `double-signal scenario`에서 rollback signal deferral의 실제 process behavior를 검증합니다.
- `tests/runtime_stack.py`의 `retry/leak assertions`에서 복구가 단순 old state가 아니라 재시도 가능한 clean state임을 확인합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 0da35c72add5 | tests/runtime_stack.py | rotation success scenario | replacement generation을 만든 뒤 CLI로 rotation하고 DB root/app, WP admin/author의 new success/old failure, config/files, service health를 확인합니다. | complete forward path를 실제 consumer interface로 검증합니다. |
| 0da35c72add5 | tests/runtime_stack.py | failure-stage matrix | 각 host/DB/WP persistent publication 전후, 특히 post-write failure hook에서 exception을 주입하고 prior generation 복구를 검사합니다. | ambiguous state compensation coverage를 만듭니다. |
| 0da35c72add5 | tests/runtime_stack.py | double-signal scenario | SIGTERM으로 forward를 중단하고 rollback-active ready marker 뒤 SIGINT를 추가로 보내 compensation이 계속되는지 확인합니다. | rollback signal deferral의 실제 process behavior를 검증합니다. |
| 0da35c72add5 | tests/runtime_stack.py | retry/leak assertions | 실패/rollback 뒤 같은 replacement로 다시 rotation해 성공하고 temp files, secret args/env/log leaks, Docker resources가 없는지 검사합니다. | 복구가 단순 old state가 아니라 재시도 가능한 clean state임을 확인합니다. |

#### 비교 기준

- exact commit diff: `git diff 0da35c72add5^ 0da35c72add5 -- <path>`
- 이전 Thread 상태와 비교: `git diff 2e6649a7706d 0da35c72add5 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | rotation은 success 시 new-only generation, failure/signal 시 verified old generation 또는 explicit incomplete rollback로 끝나며 retry 가능합니다. |
| 재현하는 failure / boundary | 각 persistent write의 pre/post failure, post-write ambiguity, SIGTERM과 rollback-active SIGINT입니다. |
| test technique | live integration + deterministic failure-stage hooks + double-signal synchronization |
| fixture와 failure injection | old/new 네-file generations와 healthy stack을 만들고 production rotation에 named failure/pause hooks를 전달합니다. |
| 실제 통과하는 production path | lock→quiesce→WP/config/DB/files publication→recreate/probes 또는 actual-state compensation→retry를 통과합니다. |
| 핵심 assertion | new works/old fails 또는 old works/new fails, files/config 일치, service health, no temp/argv/env/log secret, retry success를 확인합니다. |
| 이 테스트가 증명하는 것 | multi-store compensation과 rollback signal deferral이 실제 consumer state에서 작동함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | SIGKILL/power loss와 모든 external mutation은 증명하지 않습니다. |
| 성격 | deterministic distributed-transaction regression |
| 막는 후속 regression | partial generation, command-status-only rollback, rollback 중 second signal abort, stale temp/leak, non-retryable state를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: SIGKILL·hardware loss·모든 DB/WordPress internal failure를 포괄하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `2557079c2d19`이 test code 자체가 obsolete runtime secret mount를 허용해 property를 약화하지 못하도록 static guard를 추가합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: ordinary/post-write/signal failures가 prior verified generation으로 보상되고 clean retry가 가능하며 success는 new-only generation으로 끝남을 증명합니다.

### 9. `2557079c2d19` — test(secrets): 회전 후 런타임 비밀 경계 고정

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `TEST`, `SECRETS` |
| Source-defined role | Prevented tests from weakening the steady-state secret boundary. |
| 이전 Thread commit | `0da35c72add5` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Statically forbids rotation tests from depending on obsolete runtime secret mounts and requires post-rotation cleanup checks.
- **Classification reason:** It preserves the intended secret architecture, but the change is a focused regression guard rather than a new security mechanism.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `2557079c2d19`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `forbidden `/run/secrets` assumptions`에서 test 편의를 위해 production invariant를 약화하는 회귀를 막습니다.
- `tests/validate_stack.py`의 `post-rotation boundary requirements`에서 성공/rollback 후에도 bootstrap-only secret boundary가 유지됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2557079c2d19 | tests/validate_stack.py | forbidden `/run/secrets` assumptions | rotation/runtime test source에서 mounted secret comparison helper와 obsolete `/run/secrets` path pattern을 거부합니다. | test 편의를 위해 production invariant를 약화하는 회귀를 막습니다. |
| 2557079c2d19 | tests/validate_stack.py | post-rotation boundary requirements | rotation scenario가 full runtime secret env/mount absence를 다시 검사하고 private config temp cleanup을 요구하는지 확인합니다. | 성공/rollback 후에도 bootstrap-only secret boundary가 유지됩니다. |

#### 비교 기준

- exact commit diff: `git diff 2557079c2d19^ 2557079c2d19 -- <path>`
- 이전 Thread 상태와 비교: `git diff 0da35c72add5 2557079c2d19 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | rotation 이후에도 long-running services는 secret mount/password environment를 보유하지 않고 temp credential files가 남지 않습니다. |
| 재현하는 failure / boundary | test code가 obsolete `/run/secrets` helper를 사용해 weaker architecture를 암묵적으로 허용하는 경계입니다. |
| test technique | static source contract |
| fixture와 failure injection | tests와 production source의 forbidden/required patterns가 fixture입니다. |
| 실제 통과하는 production path | `tests/validate_stack.py`가 rotation/runtime test source와 Compose blocks를 읽습니다. |
| 핵심 assertion | 금지 mounted-secret pattern 부재, full post-rotation boundary assertion과 temp cleanup pattern 존재를 확인합니다. |
| 이 테스트가 증명하는 것 | verification code가 production secret boundary를 약화하지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 실제 container inspect, authentication, signal timing은 증명하지 않습니다. |
| 성격 | static architecture guard |
| 막는 후속 regression | test-only runtime secret mount, incomplete post-rotation inspect, config temp leak assertion 제거를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: runtime authentication/rollback correctness를 직접 실행해 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: Thread 2의 secret-free runtime invariant를 credential rotation 이후에도 source contract로 연결합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: test suite 자체가 steady-state secret mount를 재도입하거나 production보다 약한 test fixture를 사용하지 않게 합니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| replacement 및 active secret files는 private regular single-link input이며 individual publication은 atomic/durable합니다. | a2d20b8c2c03 | 9934b478c79a | 0da35c72add5 | descriptor validation과 same-directory fsync/replace, temp/leak runtime assertions이 연결됩니다. |
| rotation 성공 시 모든 replacement credential은 작동하고 모든 previous credential은 거부됩니다. | 64844c583211 | 9934b478c79a | 0da35c72add5 | DB/WP positive+negative probes와 files/config exact generation 검사가 success endpoint입니다. |
| rotation 실패 시 verified prior generation으로 compensation하거나 incomplete rollback을 명시적으로 보고합니다. | c68486d55f30 | 2e6649a7706d에서 ambiguous state/signal 보강 | 0da35c72add5 | actual-state probes, reverse compensation, error aggregation과 stage matrix가 연결됩니다. |
| rollback 활성화 뒤 추가 termination signal은 recovery를 중단하지 않습니다. | 2e6649a7706d | 2e6649a7706d | 0da35c72add5 | rollback-active state가 second signal을 defer하고 double-signal scenario가 completion을 확인합니다. |
| 회전 뒤에도 long-running runtime secret exposure boundary가 유지됩니다. | 기존 bootstrap architecture | 2557079c2d19 static guard | 0da35c72add5 | runtime inspect와 obsolete test-pattern ban이 success/rollback 후에도 secret-free serving을 요구합니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: 개별 command의 nonzero를 no mutation으로 해석하고 ordinary exception만 rollback하면 post-write failure와 signal이 mixed credential generation을 남길 수 있었습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: project lock, Nginx quiescence, application-first/root-last ordering, actual positive/negative probes, reverse compensation과 rollback-active signal deferral이 transition endpoint를 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `0da35c72add5`의 per-stage/post-write/double-signal/retry matrix와 `2557079c2d19` secret-boundary static guard가 regression을 막습니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| DB/WordPress/host files 중 일부만 new generation | 9934b478c79a ordered locked transaction + c68486d55f30 compensation | 0da35c72add5 publication-stage failure matrix | per-store atomicity를 global atomicity로 과장하지 않고 observed state를 보상합니다. |
| subprocess가 write 후 nonzero를 반환해 state ambiguous | 2e6649a7706d actual probes와 post-write compensation | post-write failure injection scenarios | exit code 대신 actual authentication/file behavior가 authoritative합니다. |
| operator signal이 host files 교체 뒤 또는 rollback 중 도착 | 2e6649a7706d first signal failure conversion + rollback signal defer | 0da35c72add5 SIGTERM 후 rollback-active SIGINT | recovery를 완료한 뒤 pending termination을 처리합니다. |
| 새 값은 작동하지만 old 값도 인증됨 | 64844c583211 positive+negative verification | 0da35c72add5 end-to-end authentication | success는 new acceptance뿐 아니라 old revocation까지 포함합니다. |
| test가 obsolete `/run/secrets`를 허용 | 2557079c2d19 static guard | static validator assertions | test convenience가 production boundary를 바꾸지 못하게 합니다. |

### 직접 재구성할 chain

```text
기존 가정: 각 mutation command의 exit status가 실제 credential state를 정확히 나타내고 signal은 일반 예외와 같다는 가정
  → 실제 failure 또는 위험: write 뒤 nonzero 또는 host-file publication 중 signal이 일부 stores만 new generation으로 남길 수 있었고 rollback 중 추가 signal이 복구 자체를 끊을 수 있었습니다.
  → root cause: credential state가 네 host files, DB accounts, WordPress users와 private config에 분산되어 command status만으로 authoritative state를 알 수 없습니다.
  → 수정된 invariant / decision: 각 write 뒤 actual authentication/file probes로 state를 재구성하고 reverse compensation 동안 추가 termination을 지연합니다.
  → 해당 SHA의 실제 수정 코드: `2e6649a7706d`의 post-write hooks, probe-driven compensation와 rollback signal state
  → failure injection 또는 regression test: `0da35c72add5` successful/failure/signal/retry scenarios
  → 증명된 보장 / 남은 비보장: success는 new works/old fails, failure는 verified old generation 또는 explicit incomplete rollback이며 SIGKILL/power loss의 모든 시점은 보장하지 않습니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Host secret files | old generation 개별 files | same-directory atomic replacement와 parent sync | a2d20b8c2c03 | 개별 file은 atomic하지만 네 files 전체는 orchestrator가 순서를 관리합니다. |
| MariaDB accounts | root/application current passwords | local-socket mutation; application 먼저, root 마지막 | 832d182743ea + 9934b478c79a order | root repair path를 forward 후반까지 유지합니다. |
| WordPress users | WordPress DB의 hashed passwords | `wp_set_password` application-owned mutation | 0aa998fdd344 | admin/author를 별도 stages로 추적합니다. |
| Private wp-config.php | DB credential consumer | exact define를 same-filesystem rename으로 교체 | 0aa998fdd344 | web tree가 아닌 private config volume이 authority입니다. |
| Rotation orchestrator | 개별 helper 결과 의존 | actual probes와 project lock으로 generation 전환/보상 소유 | 9934b478c79a, 2e6649a7706d | distributed stores의 success/failure endpoint를 판정합니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** Credentials are represented in four host files, two MariaDB accounts, two WordPress users, and WordPress configuration. The thread therefore evolves from individual mutation primitives to a verified state transition and then to compensation for commands that may change state before failing. Deferring further termination while rollback is active is the key correction that prevents recovery itself from being interrupted.
- 최종 authoritative state와 owner: 성공 시 네 host files, DB root/app accounts, WP admin/author users, private config가 같은 new generation을 소유하고 old generation은 거부됩니다.
- 정상 실행의 entry point와 완료 조건: project lock 아래 pre-verification, Nginx quiesce, ordered mutations, host publication, force-recreate, new/old probes가 모두 성공하면 완료입니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: failure/first signal 시 actual state를 probe해 old generation으로 reverse compensation하고, rollback 중 추가 signal은 완료까지 지연합니다. incomplete rollback은 명시적 failure입니다.
- 이 Thread가 다른 Thread에 제공하는 전제: Thread 2의 secret-free runtime과 Thread 8의 operations/diagnostics가 credential generation 변화 뒤에도 유지될 전제를 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: 여러 stores를 한 storage transaction처럼 원자 commit하거나 SIGKILL/power loss에서 반드시 복구한다고 보장하지 않습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | replacement/current 검증 | a2d20b8c2c03 readers | 네 old/new files를 private stable input으로 읽습니다. | unsafe file/content면 mutation 전 실패합니다. |
| 2 | lock/old-state probes | 9934b478c79a rotate | same-project lock과 current generation/runtime boundary를 확인합니다. | mixed pre-state면 rotation을 시작하지 않습니다. |
| 3 | public path quiesce | 9934b478c79a Nginx stop | 외부 request가 transition 중 mixed generation을 관측하지 않게 합니다. | stop failure면 forward mutation 전에 compensation/recovery로 갑니다. |
| 4 | WP/config 변경 | 0aa998fdd344 primitives | admin/author와 private config를 new values로 바꿉니다. | post-write failure는 actual probes 대상입니다. |
| 5 | DB app/root 변경 | 832d182743ea + ordered orchestrator | application password 후 root password를 바꿉니다. | root는 repair capability 때문에 forward 후반에 변경됩니다. |
| 6 | host files/recreate | a2d20b8c2c03 publication + 9934b478c79a recreate | active source files를 new generation으로 게시하고 services를 새로 띄웁니다. | publication/recreate failure는 observed state compensation으로 갑니다. |
| 7 | global verification | 64844c583211 probes | new works, old fails, files/config match, runtime no-secret를 확인합니다. | 하나라도 어긋나면 success로 확정하지 않습니다. |
| 8 | compensation/signal defer | 2e6649a7706d | actual state를 탐색해 old generation으로 되돌리고 rollback-active signal을 지연합니다. | 복구 불완전과 pending signal을 모두 최종 result에 보존합니다. |

### 학습자의 최종 설명

> credential generation은 한 DB row가 아니라 네 host files, MariaDB 두 accounts, WordPress 두 users, private config에 분산됩니다. 각 file과 subsystem primitive는 개별적으로 안전하지만 전체는 atomic transaction이 아닙니다. orchestrator는 project lock과 Nginx quiescence 아래 WordPress users/config, DB application, DB root, host files 순으로 전환하고 services를 recreate한 뒤 new success와 old rejection을 모두 확인합니다. 초기 compensation은 command failure를 no mutation으로 볼 위험이 있었으나 `2e6649a7706d`에서 각 write 뒤 actual probes로 state를 재구성하고 rollback 중 추가 signal을 defer하도록 수정됐습니다. 따라서 success는 new-only generation, failure는 verified old generation 또는 명시적 incomplete rollback이라는 endpoint로 관리됩니다.

## 학습 완료 자가 점검

- [x] rotation을 하나의 DB transaction처럼 원자적이라고 표현하지 않았습니까?
- [x] subprocess exit code만으로 state mutation 여부를 결정하지 않았습니까?
- [x] root credential을 너무 일찍 바꿔 후속 repair path를 끊는 ordering을 놓치지 않았습니까?
- [x] rollback 중 두 번째 signal이 즉시 종료시킨다고 잘못 설명하지 않았습니까?
- [x] 새 값 성공과 옛 값 거부를 모두 실제 consumer interface로 확인했습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 06-credential-rotation-and-compensation.md =====

===== BEGIN FILE: 07-immutable-build-inputs.md =====
# Thread 7 — Immutable build inputs and runtime supply-chain evidence

## Thread 목표

moving Debian/WordPress inputs를 immutable digest·snapshot·checksum으로 고정하고, source string뿐 아니라 실제 실행 중 package/application version까지 검증하는 maintained supply-chain contract를 추적합니다.

**Source significance**

> Reproducibility is treated as a maintained contract rather than a one-time freeze. The first commits make upstream identities explicit; the later update demonstrates how supported versions advance; and runtime inspection closes the gap between strings in Dockerfiles and the software actually executing inside containers.

## 이 Thread를 이해하기 위한 핵심 질문

- base image digest와 dated APT snapshot은 서로 다른 어떤 input을 고정합니까?
- snapshot metadata의 validity date를 비활성화한 trade-off는 무엇입니까?
- WordPress core를 image-controlled, `wp-content`를 volume-controlled로 나눈 기준은 무엇입니까?
- bootstrap runtime download를 제거하면 interruption recovery와 reproducibility가 어떻게 연결됩니까?
- source pin checks만으로 stale image cache나 unintended package resolution을 잡지 못하는 이유는 무엇입니까?
- pin update commit이 reproducibility contract를 깨지 않고 maintenance를 수행했다는 증거는 무엇입니까?

## 완료 기준

- 각 Dockerfile의 Debian digest와 snapshot source 설정을 해당 SHA에서 확인했습니다.
- WP-CLI/WordPress version, SHA-256, image source directory, core manifest 생성/검증 경로를 추적했습니다.
- core reconciliation과 `wp-content` preservation policy를 파일별로 비교했습니다.
- static pin test, live WordPress/WP-CLI identity, dpkg minimum, PHP/MariaDB compatibility 검증을 구분했습니다.
- 보안 지원 pin update에서 함께 변경되어야 한 source/test 값을 기록했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `3e29fbd34389` | build(images): Debian 이미지와 패키지 입력 고정 | **A** | `SUPPLY_CHAIN`<br>`RISK`<br>`ARCH` | Pinned Debian base images and package repositories to immutable inputs. |
| 2 | `f60ac8061c01` | build(wordpress): WordPress 산출물을 고정해 게시 | **A** | `SUPPLY_CHAIN`<br>`BOOTSTRAP`<br>`RISK` | Pinned WordPress and WP-CLI artifacts and moved core publication into bootstrap reconciliation. |
| 3 | `7b28cccaec1d` | test(supply-chain): 불변 image 입력 검증 | **A** | `TEST`<br>`SUPPLY_CHAIN`<br>`RISK` | Locked the source pins and running application versions in tests. |
| 4 | `cd5982c8ea42` | fix(supply-chain): 보안 지원 runtime pin 갱신 | **B** | `SUPPLY_CHAIN`<br>`RISK` | Advanced the reviewed immutable runtime set without returning to moving inputs. |
| 5 | `127a70f6e4b2` | test(supply-chain): 검토된 runtime 최소 버전 검증 | **A** | `TEST`<br>`SUPPLY_CHAIN`<br>`RISK` | Verified installed package minimums and live PHP/MariaDB compatibility floors. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `3e29fbd34389` — build(images): Debian 이미지와 패키지 입력 고정

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `SUPPLY_CHAIN`, `RISK`, `ARCH` |
| Source-defined role | Pinned Debian base images and package repositories to immutable inputs. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `f60ac8061c01` |

#### 원문이 확정한 범위

- **Summary:** Pins all service base images by digest and redirects Debian packages to an immutable dated snapshot.
- **Classification reason:** This changes the build trust model from moving upstream inputs to reviewed immutable inputs, a significant reproducibility and supply-chain decision despite not changing application behavior.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `3e29fbd34389`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/*/Dockerfile`의 `FROM Debian dated tag@sha256`에서 base filesystem bytes가 moving tag가 아니라 content digest로 고정됩니다.
- `srcs/requirements/*/Dockerfile`의 `snapshot.debian.org sources`에서 APT package universe와 metadata가 build date에 따라 움직이지 않습니다.
- `srcs/requirements/wordpress/Dockerfile`의 `dependency cleanup`에서 pinning decision과 별개로 reviewed package set을 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3e29fbd34389 | srcs/requirements/*/Dockerfile | FROM Debian dated tag@sha256 | Nginx, MariaDB, WordPress 세 Dockerfile이 동일한 reviewed Debian dated slim image digest를 사용합니다. | base filesystem bytes가 moving tag가 아니라 content digest로 고정됩니다. |
| 3e29fbd34389 | srcs/requirements/*/Dockerfile | snapshot.debian.org sources | main, updates, security package sources를 explicit timestamp snapshot으로 바꾸고 snapshot 사용을 위해 Valid-Until 검사를 비활성화합니다. | APT package universe와 metadata가 build date에 따라 움직이지 않습니다. |
| 3e29fbd34389 | srcs/requirements/wordpress/Dockerfile | dependency cleanup | 더 이상 사용하지 않는 `unzip`을 제거합니다. | pinning decision과 별개로 reviewed package set을 줄입니다. |

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | moving `debian:bookworm-slim`과 live APT mirror는 동일 source를 다른 날 build할 때 base layer와 package versions가 달라질 수 있었습니다. |
| 선택한 boundary / decision | base image는 digest, package repositories는 dated snapshot timestamp로 각각 고정했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/requirements/*/Dockerfile`의 `FROM Debian dated tag@sha256`; `srcs/requirements/*/Dockerfile`의 `snapshot.debian.org sources`; `srcs/requirements/wordpress/Dockerfile`의 `dependency cleanup` |
| state / ownership / lifecycle 변화 | Dockerfile이 reviewed upstream identities를 소유하고 build는 해당 digest/snapshot만 소비합니다. security updates는 자동이 아니라 explicit pin maintenance가 소유합니다. |
| 주요 failure branch | snapshot unavailable, digest mismatch, package index/install failure는 build failure가 됩니다. Valid-Until 비활성화는 오래된 metadata를 의도적으로 허용하는 trade-off입니다. |
| 이 commit의 보장 | 동일 source와 reachable snapshot으로 같은 base/package input을 선택하는 reproducibility boundary를 제공합니다. |
| 한계와 다음 관련 commit | WordPress/WP-CLI artifact identity, actual installed package versions, image cache가 올바른지까지는 보장하지 않습니다. `f60ac8061c01`이 application artifacts를 고정하고 `7b28cccaec1d`/`127a70f6e4b2`가 source와 live runtime identity를 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: WordPress/WP-CLI artifact identity, actual installed package versions, image cache가 올바른지까지는 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `f60ac8061c01`이 application artifacts를 고정하고 `7b28cccaec1d`/`127a70f6e4b2`가 source와 live runtime identity를 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 동일 source와 reachable snapshot으로 같은 base/package input을 선택하는 reproducibility boundary를 제공합니다.

### 2. `f60ac8061c01` — build(wordpress): WordPress 산출물을 고정해 게시

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `SUPPLY_CHAIN`, `BOOTSTRAP`, `RISK` |
| Source-defined role | Pinned WordPress and WP-CLI artifacts and moved core publication into bootstrap reconciliation. |
| 이전 Thread commit | `3e29fbd34389` |
| 다음 Thread commit | `7b28cccaec1d` |

#### 원문이 확정한 범위

- **Summary:** Pins WP-CLI and WordPress archives with checksums, stages WordPress core in the image, atomically reconciles files at bootstrap, and disables automatic core updates.
- **Classification reason:** It removes runtime downloads from initialization and makes the application artifact an immutable, verified build input, significantly strengthening both reproducibility and recovery semantics.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `f60ac8061c01`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/wordpress/Dockerfile`의 `WP_CLI_VERSION / SHA-256`에서 runtime network가 다른 WP-CLI bytes를 제공하지 못합니다.
- `srcs/requirements/wordpress/Dockerfile`의 `WORDPRESS_VERSION / SHA-256 / /usr/src/wordpress`에서 reviewed image artifact가 core file authority가 됩니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core reconciliation`에서 bootstrap interruption recovery와 reproducibility가 같은 image artifact에 의존합니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 ``wp-content` preservation branch`에서 image upgrade가 application-owned content를 덮어쓰지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| f60ac8061c01 | srcs/requirements/wordpress/Dockerfile | WP_CLI_VERSION / SHA-256 | explicit WP-CLI version과 checksum으로 phar를 build 시 download·검증한 뒤 image에 설치합니다. | runtime network가 다른 WP-CLI bytes를 제공하지 못합니다. |
| f60ac8061c01 | srcs/requirements/wordpress/Dockerfile | WORDPRESS_VERSION / SHA-256 / /usr/src/wordpress | WordPress archive도 explicit version/checksum으로 검증하고 image-owned source directory에 풀며 sorted core manifest를 만듭니다. | reviewed image artifact가 core file authority가 됩니다. |
| f60ac8061c01 | srcs/requirements/wordpress/tools/docker-entrypoint.sh | core reconciliation | runtime `wp core download`를 제거하고 image source/manifest를 검증한 뒤 persistent web volume의 core files를 temporary+rename 방식으로 맞춥니다. | bootstrap interruption recovery와 reproducibility가 같은 image artifact에 의존합니다. |
| f60ac8061c01 | srcs/requirements/wordpress/tools/docker-entrypoint.sh | `wp-content` preservation branch | existing `wp-content`는 volume-owned user/plugin/upload state로 보존하고 core reconciliation 대상에서 분리합니다. | image upgrade가 application-owned content를 덮어쓰지 않습니다. |

#### 비교 기준

- exact commit diff: `git diff f60ac8061c01^ f60ac8061c01 -- <path>`
- 이전 Thread 상태와 비교: `git diff 3e29fbd34389 f60ac8061c01 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | WordPress bootstrap이 runtime network에서 moving artifact를 download하면 image review와 실제 persistent core bytes가 분리되고 중단 시 partially downloaded state가 남을 수 있었습니다. |
| 선택한 boundary / decision | WP-CLI와 WordPress를 build-time version/checksum으로 고정하고 image-controlled core source/manifest를 bootstrap이 persistent volume에 수렴시키도록 했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/requirements/wordpress/Dockerfile`의 `WP_CLI_VERSION / SHA-256`; `srcs/requirements/wordpress/Dockerfile`의 `WORDPRESS_VERSION / SHA-256 / /usr/src/wordpress`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core reconciliation`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 ``wp-content` preservation branch` |
| state / ownership / lifecycle 변화 | image가 WordPress core identity를, named volume이 `wp-content`와 runtime state를 소유합니다. bootstrap은 둘 사이 reconciliation/publish lifecycle을 소유합니다. |
| 주요 failure branch | download/checksum/unpack/manifest build failure는 image build를 중단합니다. runtime manifest/source 검증이나 atomic file publication 실패는 completion marker 전에 bootstrap을 실패시킵니다. |
| 이 commit의 보장 | runtime network download 없이 reviewed core bytes와 WP-CLI를 사용하고, existing `wp-content`를 보존하면서 core를 manifest에 맞출 수 있습니다. |
| 한계와 다음 관련 commit | source pins가 실제 running container에 적용됐는지와 package security floor는 별도 runtime evidence가 필요합니다. `7b28cccaec1d`이 no-runtime-download/source pins/live versions를 검사하고 `cd5982c8ea42`이 maintained update 절차를 보여줍니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: source pins가 실제 running container에 적용됐는지와 package security floor는 별도 runtime evidence가 필요합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `7b28cccaec1d`이 no-runtime-download/source pins/live versions를 검사하고 `cd5982c8ea42`이 maintained update 절차를 보여줍니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: runtime network download 없이 reviewed core bytes와 WP-CLI를 사용하고, existing `wp-content`를 보존하면서 core를 manifest에 맞출 수 있습니다.

### 3. `7b28cccaec1d` — test(supply-chain): 불변 image 입력 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `SUPPLY_CHAIN`, `RISK` |
| Source-defined role | Locked the source pins and running application versions in tests. |
| 이전 Thread commit | `f60ac8061c01` |
| 다음 Thread commit | `cd5982c8ea42` |

#### 원문이 확정한 범위

- **Summary:** Checks immutable Debian and WordPress pins statically and verifies the running WordPress and WP-CLI versions.
- **Classification reason:** The commit protects the newly established supply-chain contract from silent reversion to moving inputs or runtime downloads.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `7b28cccaec1d`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `Dockerfile pin assertions`에서 moving tag/live mirror/checksum 제거를 정적으로 막습니다.
- `tests/validate_stack.py`의 `no runtime download / reconciliation patterns`에서 build-owned core와 runtime reconciliation architecture를 고정합니다.
- `tests/runtime_stack.py`의 `live WordPress/WP-CLI versions`에서 Dockerfile 문자열과 실제 runtime identity 사이의 gap을 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 7b28cccaec1d | tests/validate_stack.py | Dockerfile pin assertions | 세 base digest/snapshot timestamp와 WP-CLI/WordPress version/checksum을 exact source values로 검사합니다. | moving tag/live mirror/checksum 제거를 정적으로 막습니다. |
| 7b28cccaec1d | tests/validate_stack.py | no runtime download / reconciliation patterns | WordPress entrypoint의 `wp core download`를 금지하고 image artifact/manifest/atomic publication pattern을 요구합니다. | build-owned core와 runtime reconciliation architecture를 고정합니다. |
| 7b28cccaec1d | tests/runtime_stack.py | live WordPress/WP-CLI versions | running WordPress container에서 WP core version과 WP-CLI version을 실행해 pinned expected values와 비교합니다. | Dockerfile 문자열과 실제 runtime identity 사이의 gap을 줄입니다. |

#### 비교 기준

- exact commit diff: `git diff 7b28cccaec1d^ 7b28cccaec1d -- <path>`
- 이전 Thread 상태와 비교: `git diff f60ac8061c01 7b28cccaec1d -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | immutable source inputs과 running WordPress/WP-CLI identity가 같은 reviewed set을 가리킵니다. |
| 재현하는 failure / boundary | moving tag/mirror, checksum 제거, runtime core download, stale/wrong application image입니다. |
| test technique | static source pin contract + live version inspection |
| fixture와 failure injection | Dockerfiles/entrypoint source와 isolated running stack이 fixture입니다. |
| 실제 통과하는 production path | validator가 source를 검사하고 runtime harness가 WordPress container에서 version commands를 실행합니다. |
| 핵심 assertion | exact digest/snapshot/version/checksum, no runtime download, live WP/WP-CLI version 일치를 확인합니다. |
| 이 테스트가 증명하는 것 | source policy와 실제 application-level runtime identity의 연결을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 installed package version, artifact signer, vulnerability 상태는 증명하지 않습니다. |
| 성격 | mixed static/runtime supply-chain regression |
| 막는 후속 regression | moving input 재도입, checksum 제거, runtime download, wrong cached application artifact를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: OS package full inventory, cryptographic provenance, vulnerability absence, cache content 전체는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `127a70f6e4b2`가 installed package minimum과 PHP/MariaDB compatibility evidence를 더합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: reviewed source pins와 running WordPress/WP-CLI identity가 함께 일치함을 증명합니다.

### 4. `cd5982c8ea42` — fix(supply-chain): 보안 지원 runtime pin 갱신

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `SUPPLY_CHAIN`, `RISK` |
| Source-defined role | Advanced the reviewed immutable runtime set without returning to moving inputs. |
| 이전 Thread commit | `7b28cccaec1d` |
| 다음 Thread commit | `127a70f6e4b2` |

#### 원문이 확정한 범위

- **Summary:** Advances the reviewed Debian digest, package snapshot, WordPress version, checksum, and matching assertions.
- **Classification reason:** The update is security- and support-relevant, but it follows the immutable-input mechanism already established rather than introducing a new trust model.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `cd5982c8ea42`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/*/Dockerfile`의 `coordinated Debian digest/snapshot update`에서 서비스별 package universe가 서로 다른 review generation으로 갈라지지 않습니다.
- `srcs/requirements/wordpress/Dockerfile`의 `WordPress version/checksum update`에서 application artifact도 explicit immutable identity를 유지한 채 advance합니다.
- `tests/validate_stack.py / tests/runtime_stack.py`의 `expected pin/version updates`에서 test가 old pin을 무조건 고정하는 것이 아니라 reviewed set maintenance를 추적합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| cd5982c8ea42 | srcs/requirements/*/Dockerfile | coordinated Debian digest/snapshot update | 세 service가 같은 새 dated Debian base digest와 새 snapshot timestamp로 함께 이동합니다. | 서비스별 package universe가 서로 다른 review generation으로 갈라지지 않습니다. |
| cd5982c8ea42 | srcs/requirements/wordpress/Dockerfile | WordPress version/checksum update | WordPress pin과 checksum을 새 supported patch release로 갱신합니다. | application artifact도 explicit immutable identity를 유지한 채 advance합니다. |
| cd5982c8ea42 | tests/validate_stack.py / tests/runtime_stack.py | expected pin/version updates | source/static/runtime expected values를 production pin과 같은 commit에서 갱신합니다. | test가 old pin을 무조건 고정하는 것이 아니라 reviewed set maintenance를 추적합니다. |

#### 비교 기준

- exact commit diff: `git diff cd5982c8ea42^ cd5982c8ea42 -- <path>`
- 이전 Thread 상태와 비교: `git diff 7b28cccaec1d cd5982c8ea42 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | immutable pin을 영구 동결하면 upstream security/support floor가 내려가 reproducible하지만 오래된 runtime이 됩니다. |
| 실제 failure 또는 위험 | 새 digest/checksum/compatibility가 틀리면 build/static/runtime test가 실패합니다. |
| root cause | immutable pin을 영구 동결하면 upstream security/support floor가 내려가 reproducible하지만 오래된 runtime이 됩니다. |
| 수정된 invariant / decision | digest/snapshot/application version/checksum과 corresponding tests를 coordinated review unit으로 전진시켰습니다. |
| 실제 수정 코드 | `srcs/requirements/*/Dockerfile`의 `coordinated Debian digest/snapshot update`; `srcs/requirements/wordpress/Dockerfile`의 `WordPress version/checksum update`; `tests/validate_stack.py / tests/runtime_stack.py`의 `expected pin/version updates` |
| 변경된 ordering / ownership / lifecycle | repository commit이 새 reviewed generation의 identity를 소유하며 automatic moving update는 계속 허용하지 않습니다. |
| 이 fix가 보장하는 것 | maintenance가 moving input으로 회귀하지 않고 새 immutable reviewed set으로 수행될 수 있음을 보여줍니다. |
| 아직 보장하지 않는 것 | 새 set의 모든 vulnerability가 제거됐다는 보장은 아니며 explicit support/security criteria가 계속 필요합니다. |
| 연결되는 regression test | `127a70f6e4b2`가 새 runtime의 package minimum과 platform compatibility를 live inspect합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 새 set의 모든 vulnerability가 제거됐다는 보장은 아니며 explicit support/security criteria가 계속 필요합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `127a70f6e4b2`가 새 runtime의 package minimum과 platform compatibility를 live inspect합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: maintenance가 moving input으로 회귀하지 않고 새 immutable reviewed set으로 수행될 수 있음을 보여줍니다.

### 5. `127a70f6e4b2` — test(supply-chain): 검토된 runtime 최소 버전 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `SUPPLY_CHAIN`, `RISK` |
| Source-defined role | Verified installed package minimums and live PHP/MariaDB compatibility floors. |
| 이전 Thread commit | `cd5982c8ea42` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Verifies installed package minimums plus the live PHP and MariaDB compatibility floors inside the built stack.
- **Classification reason:** This closes the gap between source pins and actual runtime contents, catching stale caches or unexpected package resolution in a security-sensitive build path.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `127a70f6e4b2`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `DEBIAN_PACKAGE_MINIMUMS / dpkg comparison`에서 source snapshot string뿐 아니라 실제 package selection을 검사합니다.
- `tests/runtime_stack.py`의 `live PHP version compatibility`에서 application runtime engine compatibility를 live process에서 확인합니다.
- `tests/runtime_stack.py`의 `live MariaDB server version compatibility`에서 DB service가 expected platform generation을 실제 실행 중인지 확인합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 127a70f6e4b2 | tests/runtime_stack.py | DEBIAN_PACKAGE_MINIMUMS / dpkg comparison | Nginx/OpenSSL/PHP/MariaDB 등 reviewed minimum을 service container의 installed package version과 `dpkg --compare-versions`로 비교합니다. | source snapshot string뿐 아니라 실제 package selection을 검사합니다. |
| 127a70f6e4b2 | tests/runtime_stack.py | live PHP version compatibility | running PHP version을 parse하고 WordPress가 요구하는 minimum과 reviewed floor 이상인지 확인합니다. | application runtime engine compatibility를 live process에서 확인합니다. |
| 127a70f6e4b2 | tests/runtime_stack.py | live MariaDB server version compatibility | server version string을 query/parse해 WordPress/MySQL compatibility minimum과 reviewed MariaDB floor를 검사합니다. | DB service가 expected platform generation을 실제 실행 중인지 확인합니다. |

#### 비교 기준

- exact commit diff: `git diff 127a70f6e4b2^ 127a70f6e4b2 -- <path>`
- 이전 Thread 상태와 비교: `git diff cd5982c8ea42 127a70f6e4b2 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | reviewed snapshot에서 실제 설치된 packages와 live PHP/MariaDB가 정한 minimum/compatibility floor 이상입니다. |
| 재현하는 failure / boundary | stale cache, alternate package resolution, unintended downgrade, incompatible live engine/server입니다. |
| test technique | live installed-package and runtime-version boundary test |
| fixture와 failure injection | isolated built stack의 Nginx/WordPress/MariaDB containers와 reviewed minimum mapping을 사용합니다. |
| 실제 통과하는 production path | container exec→dpkg query/compare→PHP version parse→MariaDB server version query/parse를 통과합니다. |
| 핵심 assertion | 필수 package 존재와 minimum 이상, live PHP/DB application floor 이상을 확인합니다. |
| 이 테스트가 증명하는 것 | source pin이 실제 package/runtime identity로 이어졌음을 강화합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 dependency, behavior compatibility, 보안 취약점 부재는 증명하지 않습니다. |
| 성격 | runtime supply-chain boundary regression |
| 막는 후속 regression | wrong cache/image, package downgrade, unsupported PHP/DB runtime이 source-only checks를 통과하는 회귀를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 모든 transitive library, CVE absence, functional compatibility 전체를 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: source text와 running software 사이 verification chain을 완성합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: reviewed minimum packages와 live platform compatibility가 실제 built/running containers에 적용됨을 증명합니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 세 service image는 동일한 reviewed Debian base digest와 dated package snapshot을 사용합니다. | 3e29fbd34389 | cd5982c8ea42 coordinated advance | 7b28cccaec1d, 127a70f6e4b2 | 세 Dockerfile exact pins와 live installed-package checks가 같은 reviewed generation을 연결합니다. |
| WordPress와 WP-CLI artifact는 explicit version과 checksum으로 build-time 검증됩니다. | f60ac8061c01 | cd5982c8ea42 WordPress pin advance | 7b28cccaec1d | Dockerfile checksum verification, no runtime download, live version assertions이 연결됩니다. |
| WordPress core는 image-controlled manifest에 수렴하고 `wp-content`는 existing volume state를 보존합니다. | f60ac8061c01 | f60ac8061c01 | runtime e2e/version checks | image source/manifest reconciliation은 core만 다루고 content branch는 existing volume data를 유지합니다. |
| source pin과 실제 installed/runtime identity가 함께 충족되어야 supply-chain contract가 성립합니다. | 7b28cccaec1d | 127a70f6e4b2 package/platform evidence 강화 | 127a70f6e4b2 | static exact strings와 live WP/WP-CLI/package/PHP/MariaDB version을 다른 layer로 검사합니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: moving base tags, live APT mirrors와 runtime WordPress download는 같은 source에서 시간에 따라 다른 image와 interrupted bootstrap 결과를 만들 수 있었습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: Debian digest, dated snapshots, WordPress/WP-CLI version+checksum, image-owned core manifest와 bootstrap reconciliation이 reviewed input set을 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `7b28cccaec1d` source/live identity checks와 `127a70f6e4b2` installed package/runtime compatibility floors가 stale cache와 unexpected resolution을 검출합니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| moving base tag/live mirror로 같은 source가 다른 bytes를 build | 3e29fbd34389 immutable digest/snapshot | 7b28cccaec1d static source checks | base filesystem과 package universe를 별도 immutable inputs로 고정합니다. |
| startup 때 WordPress runtime download로 image review와 state 분리 | f60ac8061c01 verified build artifact와 core reconciliation | 7b28cccaec1d no-download/live version | bootstrap은 network가 아니라 image source/manifest를 사용합니다. |
| immutable pin이 오래되어 support/security floor 아래로 감 | cd5982c8ea42 explicit coordinated advance | 127a70f6e4b2 installed minimum/compatibility checks | immutability를 유지하면서 review generation을 갱신합니다. |
| Dockerfile 문자열은 맞지만 stale cache/alternate path 실행 | runtime identity/minimum verification | 7b28cccaec1d, 127a70f6e4b2 | source contract와 effective runtime evidence를 결합합니다. |

### 직접 재구성할 chain

```text
기존 가정: version tag와 package name만 적으면 반복 가능한 build가 된다는 가정
  → 실제 failure 또는 위험: base image, repository metadata, WordPress archive가 이동해 동일 commit의 산출물이 달라지고 bootstrap이 network 상태에 의존했습니다.
  → root cause: upstream identity와 checksum이 source-controlled input으로 고정되지 않았습니다.
  → 수정된 invariant / decision: 모든 external artifact를 immutable identity로 검증해 image에 포함하고 runtime은 manifest로 core를 reconcile합니다.
  → 해당 SHA의 실제 수정 코드: `3e29fbd34389`, `f60ac8061c01` Dockerfile/entrypoint changes와 `cd5982c8ea42` coordinated pin update
  → failure injection 또는 regression test: `7b28cccaec1d`, `127a70f6e4b2` static/live tests
  → 증명된 보장 / 남은 비보장: reviewed build input과 실제 runtime minimum을 검증하지만 새 보안 release 반영은 자동이 아니라 명시적 maintenance가 필요합니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Dockerfile/base image | moving bookworm-slim identity | reviewed digest가 base filesystem identity 소유 | 3e29fbd34389 FROM digest | tag 이름만 신뢰하지 않습니다. |
| APT repositories | live mirror resolution | dated snapshot timestamp가 package universe 소유 | snapshot source/Valid-Until config | 업데이트는 explicit commit이 필요합니다. |
| WordPress core | runtime download/volume drift 가능 | image source + checksum manifest가 authority | f60ac8061c01 download/checksum/manifest/reconcile | persistent core는 image-reviewed bytes에 수렴합니다. |
| wp-content | core와 동일 overwrite 위험 | volume-controlled user/application data | f60ac8061c01 preservation branch | image update가 existing content를 덮지 않습니다. |
| Verification | source text 중심 | live versions와 installed package floors까지 evidence 확장 | 7b28cccaec1d, 127a70f6e4b2 | source와 runtime을 별도 layer로 비교합니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** Reproducibility is treated as a maintained contract rather than a one-time freeze. The first commits make upstream identities explicit; the later update demonstrates how supported versions advance; and runtime inspection closes the gap between strings in Dockerfiles and the software actually executing inside containers.
- 최종 authoritative state와 owner: repository pins가 reviewed base/package/application identities를, image manifest가 WordPress core를, named volume이 `wp-content`를 소유합니다.
- 정상 실행의 entry point와 완료 조건: build-time digest/snapshot/checksum 검증과 bootstrap manifest reconciliation, static pins, live versions/minimums가 모두 통과하면 contract가 충족됩니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: pin mismatch/build failure/runtime version mismatch는 silent fallback 없이 실패하며 update는 coordinated reviewed commit으로 수행합니다.
- 이 Thread가 다른 Thread에 제공하는 전제: Thread 2 bootstrap이 network-independent reviewed artifact로 수렴하고 Thread 8 CI가 동일 checks를 자동 실행할 전제를 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: 취약점 부재나 byte-for-byte 모든 transitive toolchain reproducibility를 단독으로 증명하지 않습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | base/package 선택 | 3e29fbd34389 Dockerfiles | digest base와 dated snapshots만 사용합니다. | unreachable/mismatch/package failure면 build 중단입니다. |
| 2 | application artifacts 검증 | f60ac8061c01 Dockerfile | WP-CLI/WordPress version+SHA-256을 확인합니다. | checksum mismatch면 image가 생성되지 않습니다. |
| 3 | core source/manifest 생성 | f60ac8061c01 `/usr/src/wordpress` | reviewed core bytes와 sorted manifest를 image에 저장합니다. | manifest/source mismatch는 bootstrap 실패입니다. |
| 4 | persistent core 수렴 | f60ac8061c01 entrypoint reconciliation | core files를 image manifest에 맞추고 atomic publish합니다. | marker 전 failure는 다음 bootstrap에서 다시 수렴합니다. |
| 5 | content 보존 | f60ac8061c01 wp-content branch | existing user/plugin/upload data를 유지합니다. | core update가 content tree를 overwrite하지 않습니다. |
| 6 | source/live verification | 7b28cccaec1d, 127a70f6e4b2 tests | pins와 running app/package/platform versions를 비교합니다. | 문자열 또는 live identity mismatch는 regression입니다. |
| 7 | reviewed update | cd5982c8ea42 | digest/snapshot/version/checksum/tests를 함께 advance합니다. | 부분 update는 static/runtime mismatch로 실패합니다. |

### 학습자의 최종 설명

> reproducibility는 한번 freeze하고 잊는 상태가 아닙니다. base filesystem은 digest, Debian package universe는 dated snapshot, WP-CLI와 WordPress는 version+checksum으로 각각 다른 upstream input을 고정합니다. WordPress core는 image source와 manifest가 authority가 되어 bootstrap이 persistent volume을 수렴시키고, `wp-content`는 volume-owned state로 보존됩니다. static tests는 moving input과 runtime download 회귀를 막지만 stale cache나 alternate build path까지 보지 못하므로 live WP/WP-CLI, installed package minimum, PHP/MariaDB compatibility를 별도로 확인합니다. 후속 pin update는 모든 identities와 tests를 함께 바꿔 immutable contract를 유지하면서 supported generation으로 전진합니다.

## 학습 완료 자가 점검

- [x] immutable pin을 영구히 업데이트하지 않는다는 의미로 오해하지 않았습니까?
- [x] base image digest와 package snapshot이 같은 것을 고정한다고 합쳤습니까?
- [x] WordPress core와 `wp-content`의 ownership policy를 반대로 설명하지 않았습니까?
- [x] source strings만 확인하고 실제 container version evidence를 생략하지 않았습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 07-immutable-build-inputs.md =====

===== BEGIN FILE: 08-operational-hardening-and-automation.md =====
# Thread 8 — Operational hardening, private diagnostics, and bounded automation

## Thread 목표

network least privilege, resource/shutdown policy, destructive-operation guard, private diagnostics, owned-resource cleanup, serial local verification, least-privilege CI를 하나의 bounded operational lifecycle로 연결합니다.

**Source significance**

> This progression turns operational policy into executable evidence. Runtime limits and network boundaries are inspected on live containers; destructive commands and diagnostics fail safely; and both local and CI runners account for every project resource they create. The cleanup tooling deliberately avoids global Docker pruning, preserving the same ownership discipline used by the product management paths.

## 이 Thread를 이해하기 위한 핵심 질문

- frontend/backend network 분리 뒤 각 service의 exact membership과 WordPress dual-homing 이유는 무엇입니까?
- Compose에 limit/stop/log policy를 선언한 것과 Docker가 실제 적용한 것을 어떻게 구분해 검증합니까?
- `fclean` confirmation이 generic yes/no가 아니라 selected project name과 같아야 하는 이유는 무엇입니까?
- diagnostics가 secret을 안전하게 읽지 못하면 왜 일부 자료라도 게시하지 않고 fail closed해야 합니까?
- test cleanup failure가 primary scenario success를 무효화해야 하는 이유는 무엇입니까?
- crash recovery cleanup이 label/tag ownership만 사용하고 global prune을 금지하는 이유는 무엇입니까?
- local `verify`와 CI workflow가 result precedence, timeout, diagnostics, cleanup을 어떻게 보존합니까?

## 완료 기준

- live Docker inspect로 network, limits, stop, log, security policy를 source declaration과 대조했습니다.
- destructive Make target의 exact project-name guard와 거부 후 stack health를 확인했습니다.
- diagnostic redaction input, longer-first masking, structural masking, private publication, rescan을 추적했습니다.
- normal cleanup, crash recovery cleanup, `--keep`, leak report exit status의 ownership 차이를 정리했습니다.
- serial verify와 CI의 timeout/cleanup/diagnostic result precedence를 실제 control flow로 확인했습니다.
- workflow 자체를 검증하는 text/AST/mock layers와 금지 pattern을 구분했습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `27a3dca01d3b` | feat(network): DB 트래픽을 내부 backend로 격리 | **A** | `STACK`<br>`RISK`<br>`ARCH` | Separated public request traffic from the internal database network. |
| 2 | `911544133fb4` | feat(runtime): 서비스 자원과 종료 한계 적용 | **B** | `OPERATIONS`<br>`RISK`<br>`STACK` | Applied resource, stop, privilege, and log-rotation policy. |
| 3 | `74c285925325` | fix(make): 볼륨 삭제 전에 확인을 요구 | **A** | `OPERATIONS`<br>`RISK`<br>`EDGE` | Guarded destructive volume deletion with exact project confirmation. |
| 4 | `ef74ad47ea81` | feat(diagnostics): Compose 비밀값과 민감 항목 마스킹 | **A** | `OPERATIONS`<br>`SECRETS`<br>`RISK` | Established fail-closed diagnostic redaction. |
| 5 | `27a083d91c87` | feat(diagnostics): 비공개 진단 세트와 CLI 연결 | **A** | `OPERATIONS`<br>`SECRETS`<br>`RISK` | Published exclusive private diagnostic sets. |
| 6 | `7fbd41fe5af4` | test(operations): 자원·격리·삭제 보호·진단 검증 | **A** | `TEST`<br>`OPERATIONS`<br>`RISK` | Verified runtime limits, network membership, deletion refusal, and diagnostic safety. |
| 7 | `98e4af62e884` | test(runtime): 프로세스·비밀값·정리 제어 흐름 강화 | **A** | `TEST`<br>`RECOVERY`<br>`OPERATIONS` | Made scenario cleanup failures affect verification results. |
| 8 | `2b35aa3d2217` | test(cleanup): 테스트 프로젝트 소유 자원만 정리 | **A** | `OPERATIONS`<br>`RECOVERY`<br>`RISK` | Tracked exact project ownership and added scoped leak recovery. |
| 9 | `43ccded05e4f` | test(verify): 전체 스택 검증을 직렬 실행 | **A** | `TEST`<br>`OPERATIONS`<br>`RECOVERY` | Serialized the complete local verification lifecycle. |
| 10 | `18508c25eef0` | ci(stack): 정적·런타임·복구 검증 자동화 | **A** | `TEST`<br>`OPERATIONS`<br>`SUPPLY_CHAIN` | Automated all scenarios under least-privilege, pinned CI actions. |
| 11 | `8a6c07988160` | test(ci): workflow 검증 계약 추가 | **A** | `TEST`<br>`OPERATIONS`<br>`RISK` | Validated the workflow, tool timeouts, secret boundaries, cleanup, and artifact allowlist. |

> Commit 순서는 source의 Development Thread 정의를 그대로 따릅니다. 같은 SHA가 다른 Thread에도 있으면 이 문서의 관점으로 다시 확인합니다.

## Commit별 학습 기록

### 1. `27a3dca01d3b` — feat(network): DB 트래픽을 내부 backend로 격리

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `STACK`, `RISK`, `ARCH` |
| Source-defined role | Separated public request traffic from the internal database network. |
| 이전 Thread commit | 없음 |
| 다음 Thread commit | `911544133fb4` |

#### 원문이 확정한 범위

- **Summary:** Splits frontend and backend networks, attaching MariaDB only to an internal backend.
- **Classification reason:** This materially narrows the database communication boundary and makes WordPress the sole bridge between request-serving and persistence networks.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `27a3dca01d3b`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `networks: frontend / backend`에서 backend는 Docker 외부 route를 갖지 않는 database-only communication domain이 됩니다.
- `srcs/docker-compose.yml`의 `service memberships`에서 WordPress만 request-serving과 persistence networks를 연결하는 application bridge입니다.
- `srcs/docker-compose.yml`의 `service-name routing`에서 필요한 통신만 유지하면서 Nginx→MariaDB 직접 addressability를 제거합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 27a3dca01d3b | srcs/docker-compose.yml | networks: frontend / backend | frontend bridge와 `internal: true` backend를 분리합니다. | backend는 Docker 외부 route를 갖지 않는 database-only communication domain이 됩니다. |
| 27a3dca01d3b | srcs/docker-compose.yml | service memberships | Nginx는 frontend만, MariaDB는 backend만, WordPress는 frontend와 backend 모두 join합니다. | WordPress만 request-serving과 persistence networks를 연결하는 application bridge입니다. |
| 27a3dca01d3b | srcs/docker-compose.yml | service-name routing | `fastcgi_pass wordpress:9000`과 WordPress의 MariaDB service name은 network split 뒤에도 각 shared network에서 resolve됩니다. | 필요한 통신만 유지하면서 Nginx→MariaDB 직접 addressability를 제거합니다. |

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 세 services가 하나의 shared network에 있으면 Nginx compromise/error가 MariaDB addressability까지 얻습니다. |
| 선택한 boundary / decision | public request path와 internal DB path를 frontend/backend로 분리하고 exact service membership을 최소화했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `srcs/docker-compose.yml`의 `networks: frontend / backend`; `srcs/docker-compose.yml`의 `service memberships`; `srcs/docker-compose.yml`의 `service-name routing` |
| state / ownership / lifecycle 변화 | Nginx는 frontend, MariaDB는 backend, WordPress는 두 network의 application endpoint를 소유합니다. |
| 주요 failure branch | network membership을 잘못 바꾸면 FastCGI 또는 DB DNS가 끊깁니다. source declaration만으로 effective runtime membership을 증명하지 않습니다. |
| 이 commit의 보장 | Nginx와 MariaDB가 network를 공유하지 않고 WordPress만 양쪽과 통신하도록 reachability를 좁힙니다. |
| 한계와 다음 관련 commit | container 내부 application exploit이 WordPress를 통해 DB에 접근하는 것을 제거하지는 않습니다. `7fbd41fe5af4`이 live Docker inspect로 exact membership과 Nginx→DB isolation을 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: container 내부 application exploit이 WordPress를 통해 DB에 접근하는 것을 제거하지는 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 live Docker inspect로 exact membership과 Nginx→DB isolation을 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: Nginx와 MariaDB가 network를 공유하지 않고 WordPress만 양쪽과 통신하도록 reachability를 좁힙니다.

### 2. `911544133fb4` — feat(runtime): 서비스 자원과 종료 한계 적용

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **B** |
| Tags | `OPERATIONS`, `RISK`, `STACK` |
| Source-defined role | Applied resource, stop, privilege, and log-rotation policy. |
| 이전 Thread commit | `27a3dca01d3b` |
| 다음 Thread commit | `74c285925325` |

#### 원문이 확정한 범위

- **Summary:** Applies CPU, memory, PID, file-descriptor, stop-signal, privilege, and log-rotation limits to all services.
- **Classification reason:** The policy is broad and useful, but it applies standard operational hardening to the already-defined runtime rather than changing core state or data flow.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `911544133fb4`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `cpus / mem_limit / pids_limit / ulimits`에서 한 service의 runaway resource use가 host 전체를 무제한 점유하지 못하게 합니다.
- `srcs/docker-compose.yml`의 `stop_signal / stop_grace_period`에서 service-specific graceful shutdown 시간을 Compose lifecycle에 반영합니다.
- `srcs/docker-compose.yml`의 `security_opt / logging`에서 privilege escalation과 unbounded local log growth를 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 911544133fb4 | srcs/docker-compose.yml | cpus / mem_limit / pids_limit / ulimits | Nginx, WordPress, MariaDB에 service별 CPU, memory, PID, nofile limits를 선언합니다. | 한 service의 runaway resource use가 host 전체를 무제한 점유하지 못하게 합니다. |
| 911544133fb4 | srcs/docker-compose.yml | stop_signal / stop_grace_period | Nginx/WordPress는 SIGQUIT와 짧은 grace, MariaDB는 SIGTERM과 더 긴 grace를 사용합니다. | service-specific graceful shutdown 시간을 Compose lifecycle에 반영합니다. |
| 911544133fb4 | srcs/docker-compose.yml | security_opt / logging | `no-new-privileges:true`와 json-file `max-size`/`max-file` rotation을 적용합니다. | privilege escalation과 unbounded local log growth를 줄입니다. |

#### 비교 기준

- exact commit diff: `git diff 911544133fb4^ 911544133fb4 -- <path>`
- 이전 Thread 상태와 비교: `git diff 27a3dca01d3b 911544133fb4 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### B-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| Thread에서 맡은 구현 역할 | Applied resource, stop, privilege, and log-rotation policy. |
| 핵심 input / output / state | Compose/Docker가 cgroup/rlimit/logging/shutdown policy를 소유하고 container process는 그 한계 안에서 실행됩니다. |
| 변경된 directive / helper / command | `srcs/docker-compose.yml`의 `cpus / mem_limit / pids_limit / ulimits`; `srcs/docker-compose.yml`의 `stop_signal / stop_grace_period`; `srcs/docker-compose.yml`의 `security_opt / logging` |
| immediate failure 또는 boundary | host 또는 Docker가 unsupported field를 무시하거나 다르게 적용할 수 있어 source declaration만으로 effective state를 확정할 수 없습니다. |
| 다음 commit에 넘긴 한계 | 실제 runtime 적용 여부와 workload 적정성은 보장하지 않습니다. `7fbd41fe5af4`이 live inspect와 container commands로 effective limits를 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 실제 runtime 적용 여부와 workload 적정성은 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 live inspect와 container commands로 effective limits를 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 운영 resource/shutdown/log/security policy를 version-controlled Compose contract로 만듭니다.

### 3. `74c285925325` — fix(make): 볼륨 삭제 전에 확인을 요구

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `OPERATIONS`, `RISK`, `EDGE` |
| Source-defined role | Guarded destructive volume deletion with exact project confirmation. |
| 이전 Thread commit | `911544133fb4` |
| 다음 Thread commit | `ef74ad47ea81` |

#### 원문이 확정한 범위

- **Summary:** Requires an exact project-name confirmation before `fclean` deletes volumes and local images.
- **Classification reason:** A very small diff protects the project's most destructive operator action and restores an important ownership and data-loss boundary.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `74c285925325`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `Makefile`의 `fclean guard`에서 generic yes/force가 아니라 삭제 대상 namespace를 operator가 다시 입력해야 합니다.
- `Makefile`의 `refusal branch`에서 wrong-project 변수나 copy/paste에서 fail closed합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 74c285925325 | Makefile | fclean guard | `DESTROY_CONFIRM`가 selected `PROJECT_NAME`과 exact string equality일 때만 `down --volumes`를 실행합니다. | generic yes/force가 아니라 삭제 대상 namespace를 operator가 다시 입력해야 합니다. |
| 74c285925325 | Makefile | refusal branch | 누락 또는 mismatch면 설명을 출력하고 nonzero로 종료하며 destructive Docker command를 호출하지 않습니다. | wrong-project 변수나 copy/paste에서 fail closed합니다. |

#### 비교 기준

- exact commit diff: `git diff 74c285925325^ 74c285925325 -- <path>`
- 이전 Thread 상태와 비교: `git diff 911544133fb4 74c285925325 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | fclean을 일반 cleanup과 비슷한 low-risk operation으로 취급했습니다. |
| 실제 failure 또는 위험 | project variable이 잘못되면 unrelated persistent volumes까지 즉시 삭제됩니다. |
| root cause | destructive scope를 operator가 command 실행 시 다시 확인하는 guard가 없었습니다. |
| 수정된 invariant / decision | selected project name과 exact equality인 confirmation만 volume deletion을 허용합니다. |
| 실제 수정 코드 | `Makefile`의 `fclean guard`; `Makefile`의 `refusal branch` |
| 변경된 ordering / ownership / lifecycle | operator/Make variables가 target project identity를 소유하고 destructive command는 exact confirmation 뒤에만 resource lifecycle을 종료합니다. |
| 이 fix가 보장하는 것 | generic 확인보다 project identity를 재확인하는 bounded destructive operation을 제공합니다. |
| 아직 보장하지 않는 것 | 악의적/부주의한 operator가 exact name을 입력한 뒤 잘못 삭제하는 것까지 막지 못합니다. |
| 연결되는 regression test | operations runtime test가 mismatch refusal과 running stack health/state 보존을 확인합니다. `7fbd41fe5af4`이 refusal 뒤 stack/volumes/health가 유지되는지 runtime에서 확인합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 악의적/부주의한 operator가 exact name을 입력한 뒤 잘못 삭제하는 것까지 막지 못합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 refusal 뒤 stack/volumes/health가 유지되는지 runtime에서 확인합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: generic 확인보다 project identity를 재확인하는 bounded destructive operation을 제공합니다.

### 4. `ef74ad47ea81` — feat(diagnostics): Compose 비밀값과 민감 항목 마스킹

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `OPERATIONS`, `SECRETS`, `RISK` |
| Source-defined role | Established fail-closed diagnostic redaction. |
| 이전 Thread commit | `74c285925325` |
| 다음 Thread commit | `27a083d91c87` |

#### 원문이 확정한 범위

- **Summary:** Derives secret paths and values from rendered Compose configuration and defines fail-closed redaction of credentials and sensitive assignments.
- **Classification reason:** Diagnostics can themselves become a leakage channel; this commit establishes the critical rule that collection stops when required secrets cannot be read and redacted.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `ef74ad47ea81`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/diagnose_stack.py`의 `build_redaction_values`에서 path와 value가 logs/config에 나타나는 여러 representation을 masking set에 넣습니다.
- `tools/diagnose_stack.py`의 `longer-first literal redaction`에서 literal secret/path leakage를 deterministic하게 제거합니다.
- `tools/diagnose_stack.py`의 `structural sensitive-field masking`에서 알려진 exact value 외의 민감 출력도 줄입니다.
- `tools/diagnose_stack.py`의 `fail-closed input boundary`에서 부분 sanitize된 bundle을 신뢰 가능한 것으로 게시하지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| ef74ad47ea81 | tools/diagnose_stack.py | build_redaction_values | rendered Compose에서 secret source paths를 해석하고 raw path, resolved path, secret contents를 hardened reader로 모두 수집합니다. | path와 value가 logs/config에 나타나는 여러 representation을 masking set에 넣습니다. |
| ef74ad47ea81 | tools/diagnose_stack.py | longer-first literal redaction | masking values를 길이 내림차순으로 치환해 긴 credential 안의 짧은 substring이 먼저 바뀌어 원문 일부가 남는 문제를 피합니다. | literal secret/path leakage를 deterministic하게 제거합니다. |
| ef74ad47ea81 | tools/diagnose_stack.py | structural sensitive-field masking | password/token/secret/key 계열 assignment와 JSON/YAML-like sensitive fields를 pattern으로 추가 마스킹합니다. | 알려진 exact value 외의 민감 출력도 줄입니다. |
| ef74ad47ea81 | tools/diagnose_stack.py | fail-closed input boundary | secret file 하나라도 안전하게 읽지 못하면 masking set을 불완전하게 만든 채 진행하지 않고 diagnostics 전체를 실패시킵니다. | 부분 sanitize된 bundle을 신뢰 가능한 것으로 게시하지 않습니다. |

#### 비교 기준

- exact commit diff: `git diff ef74ad47ea81^ ef74ad47ea81 -- <path>`
- 이전 Thread 상태와 비교: `git diff 74c285925325 ef74ad47ea81 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | Compose config, inspect, logs에는 credential value뿐 아니라 host secret path나 sensitive assignment가 포함될 수 있어 원문 수집이 정보 유출을 만들었습니다. |
| 선택한 boundary / decision | hardened secret reader로 complete masking set을 만들고 literal longer-first와 structural masking을 결합했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/diagnose_stack.py`의 `build_redaction_values`; `tools/diagnose_stack.py`의 `longer-first literal redaction`; `tools/diagnose_stack.py`의 `structural sensitive-field masking`; `tools/diagnose_stack.py`의 `fail-closed input boundary` |
| state / ownership / lifecycle 변화 | diagnostics process가 raw capture와 masking values를 memory에서 일시 소유하며 sanitized text만 publication layer로 넘깁니다. |
| 주요 failure branch | secret source를 안전하게 읽지 못하거나 sanitize가 실패하면 아무 diagnostic output도 신뢰하지 않고 failure로 처리합니다. |
| 이 commit의 보장 | complete known secret/path set과 sensitive field patterns을 모두 마스킹한 text만 다음 단계로 보낼 수 있습니다. |
| 한계와 다음 관련 commit | unknown semantic secret, encoded/encrypted/compressed representation, side channel까지 모두 탐지하지는 않습니다. `27a083d91c87`이 private exclusive output set과 final rescan/cleanup을 연결하고 `7fbd41fe5af4`이 실제 secret log를 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: unknown semantic secret, encoded/encrypted/compressed representation, side channel까지 모두 탐지하지는 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `27a083d91c87`이 private exclusive output set과 final rescan/cleanup을 연결하고 `7fbd41fe5af4`이 실제 secret log를 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: complete known secret/path set과 sensitive field patterns을 모두 마스킹한 text만 다음 단계로 보낼 수 있습니다.

### 5. `27a083d91c87` — feat(diagnostics): 비공개 진단 세트와 CLI 연결

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `OPERATIONS`, `SECRETS`, `RISK` |
| Source-defined role | Published exclusive private diagnostic sets. |
| 이전 Thread commit | `ef74ad47ea81` |
| 다음 Thread commit | `7fbd41fe5af4` |

#### 원문이 확정한 범위

- **Summary:** Publishes an exclusive private diagnostic directory with allowlisted, redacted Compose, log, version, and container-state files.
- **Classification reason:** This completes a safe observability mechanism: failure evidence becomes actionable without overwriting existing output or exposing credential material.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `27a083d91c87`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/diagnose_stack.py`의 `create_output_directory`에서 diagnostic set이 기존 directory를 덮어쓰거나 공격자 path를 따라가지 않습니다.
- `tools/diagnose_stack.py`의 `allowlisted private files`에서 bundle contents와 confidentiality가 좁은 allowlist로 고정됩니다.
- `tools/diagnose_stack.py`의 `post-write rescan / cleanup on error`에서 partial 또는 unsanitized bundle을 남기지 않는 publication endpoint입니다.
- `Makefile / CLI`의 `diagnostics command`에서 operator path와 test path가 같은 safety logic을 사용합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 27a083d91c87 | tools/diagnose_stack.py | create_output_directory | output parent를 검증하고 target directory를 `0700`으로 exclusive create하며 existing path나 symlink를 거부합니다. | diagnostic set이 기존 directory를 덮어쓰거나 공격자 path를 따라가지 않습니다. |
| 27a083d91c87 | tools/diagnose_stack.py | allowlisted private files | 정해진 compose config/ps/inspect/log/metadata files만 `0600` O_EXCL로 작성합니다. | bundle contents와 confidentiality가 좁은 allowlist로 고정됩니다. |
| 27a083d91c87 | tools/diagnose_stack.py | post-write rescan / cleanup on error | 모든 output을 다시 secret/path/pattern으로 scan하고 하나라도 발견되거나 command/write가 실패하면 output directory 전체를 제거합니다. | partial 또는 unsanitized bundle을 남기지 않는 publication endpoint입니다. |
| 27a083d91c87 | Makefile / CLI | diagnostics command | project/env/output을 명시해 같은 production collector를 호출하는 documented entry point를 제공합니다. | operator path와 test path가 같은 safety logic을 사용합니다. |

#### 비교 기준

- exact commit diff: `git diff 27a083d91c87^ 27a083d91c87 -- <path>`
- 이전 Thread 상태와 비교: `git diff ef74ad47ea81 27a083d91c87 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | redaction function만 있어도 output path overwrite, broad permissions, partial file set, sanitize 후 leakage를 막지 못했습니다. |
| 선택한 boundary / decision | 새 private directory와 allowlisted exclusive files에만 sanitized data를 쓰고 전체 rescan 후 성공으로 게시하도록 했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/diagnose_stack.py`의 `create_output_directory`; `tools/diagnose_stack.py`의 `allowlisted private files`; `tools/diagnose_stack.py`의 `post-write rescan / cleanup on error`; `Makefile / CLI`의 `diagnostics command` |
| state / ownership / lifecycle 변화 | diagnostics command가 output directory lifecycle 전체를 소유하며 성공한 complete private set만 caller에게 넘깁니다. |
| 주요 failure branch | existing/symlink target, any capture/redaction/write/rescan failure는 directory를 삭제하고 nonzero로 종료합니다. |
| 이 commit의 보장 | diagnostic set은 private, non-overwriting, allowlisted, fully redacted이며 sanitize 불가 시 아무것도 게시하지 않습니다. |
| 한계와 다음 관련 commit | masking model이 모르는 새로운 encoding의 secret은 자동 보장하지 않습니다. `7fbd41fe5af4`이 unreadable secret, overwrite/symlink, real log secret, permissions/file set을 runtime 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: masking model이 모르는 새로운 encoding의 secret은 자동 보장하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 unreadable secret, overwrite/symlink, real log secret, permissions/file set을 runtime 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: diagnostic set은 private, non-overwriting, allowlisted, fully redacted이며 sanitize 불가 시 아무것도 게시하지 않습니다.

### 6. `7fbd41fe5af4` — test(operations): 자원·격리·삭제 보호·진단 검증

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `OPERATIONS`, `RISK` |
| Source-defined role | Verified runtime limits, network membership, deletion refusal, and diagnostic safety. |
| 이전 Thread commit | `27a083d91c87` |
| 다음 Thread commit | `98e4af62e884` |

#### 원문이 확정한 범위

- **Summary:** Verifies runtime limits, network membership, destructive-action refusal, fail-closed redaction, file permissions, overwrite refusal, and symlink-output rejection.
- **Classification reason:** The scenario materially validates several operational and security boundaries that configuration inspection alone cannot prove.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `7fbd41fe5af4`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `operations inspect scenario`에서 Compose declaration과 effective runtime state를 대조합니다.
- `tests/runtime_stack.py`의 `fclean refusal`에서 guard refusal이 실제 mutation 0임을 증명합니다.
- `tests/runtime_stack.py`의 `diagnostic redaction fixtures`에서 literal/structural masking과 fail-closed publication을 live data로 통과합니다.
- `tests/runtime_stack.py`의 `diagnostic file/perms/rescan assertions`에서 private publication endpoint를 검증합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 7fbd41fe5af4 | tests/runtime_stack.py | operations inspect scenario | live containers의 networks, memory/CPU/PID/nofile, stop signal/grace, no-new-privileges, log driver/options를 Docker inspect와 in-container probes로 확인합니다. | Compose declaration과 effective runtime state를 대조합니다. |
| 7fbd41fe5af4 | tests/runtime_stack.py | fclean refusal | 잘못된/누락 confirmation으로 destructive target을 실행하고 volumes와 application health/state가 그대로인지 확인합니다. | guard refusal이 실제 mutation 0임을 증명합니다. |
| 7fbd41fe5af4 | tests/runtime_stack.py | diagnostic redaction fixtures | 실제 secret을 request/log에 넣고 unreadable secret, existing directory, dangling symlink cases를 만든 뒤 success/failure output을 검사합니다. | literal/structural masking과 fail-closed publication을 live data로 통과합니다. |
| 7fbd41fe5af4 | tests/runtime_stack.py | diagnostic file/perms/rescan assertions | 성공 set의 exact allowlist, 0700/0600 mode, raw/resolved secret path와 content 부재를 확인합니다. | private publication endpoint를 검증합니다. |

#### 비교 기준

- exact commit diff: `git diff 7fbd41fe5af4^ 7fbd41fe5af4 -- <path>`
- 이전 Thread 상태와 비교: `git diff 27a083d91c87 7fbd41fe5af4 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | declared operational policy가 effective containers에 적용되고 destructive/diagnostic commands는 unsafe case에서 mutation/output 없이 실패합니다. |
| 재현하는 failure / boundary | network/limit mismatch, wrong fclean confirmation, real secret log, unreadable secret, existing/symlink output입니다. |
| test technique | live Docker inspection + negative filesystem/command integration |
| fixture와 failure injection | healthy isolated stack, secret-bearing request/log, permission/collision output fixtures를 만듭니다. |
| 실제 통과하는 production path | Compose containers→Docker inspect/in-container probes→Make fclean→diagnose_stack capture/redact/publish를 통과합니다. |
| 핵심 assertion | exact networks/limits/stop/log/security, unchanged health/volumes, exact private redacted bundle 또는 output 부재를 확인합니다. |
| 이 테스트가 증명하는 것 | 운영 policy의 source-to-effective-state 연결과 fail-closed destructive/diagnostic behavior를 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 platform과 unknown leakage channel을 증명하지 않습니다. |
| 성격 | broad operational integration + negative regression |
| 막는 후속 regression | network widening, ignored limits, generic destructive confirmation, partial/unsanitized diagnostics를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: 모든 host kernel/Docker version, unknown secret encoding, production load 적정성은 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `98e4af62e884`과 후속 commits가 scenario cleanup 자체를 verified lifecycle로 강화합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: network/limit/stop/log/security effective state, exact-project deletion guard, fail-closed private diagnostics가 실제 stack에서 동작함을 증명합니다.

### 7. `98e4af62e884` — test(runtime): 프로세스·비밀값·정리 제어 흐름 강화

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `RECOVERY`, `OPERATIONS` |
| Source-defined role | Made scenario cleanup failures affect verification results. |
| 이전 Thread commit | `7fbd41fe5af4` |
| 다음 Thread commit | `2b35aa3d2217` |

#### 원문이 확정한 범위

- **Summary:** Makes private fixture replacement durable, separates start command construction, improves timeout diagnostics, and treats cleanup failure as test failure.
- **Classification reason:** It strengthens the verification control plane so successful scenarios cannot hide leaked resources or incomplete teardown, a significant reliability property for the extensive runtime suite.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `98e4af62e884`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `explicit subprocess wait timeouts`에서 hung child가 verification process를 무기한 점유하지 않습니다.
- `tests/runtime_stack.py`의 `RuntimeStack.close error accumulation`에서 scenario body success가 cleanup failure를 덮지 않습니다.
- `tests/runtime_stack.py`의 `main result precedence`에서 evidence lifecycle 전체가 성공해야 process success입니다.
- `tools/rotate_secrets.py / related private writes`의 `private temp fsync/cleanup hardening`에서 verification이 관측하는 resource state와 host file durability를 맞춥니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 98e4af62e884 | tests/runtime_stack.py | explicit subprocess wait timeouts | Popen communicate/wait/terminate/kill paths에 bounded timeout과 escalation을 추가합니다. | hung child가 verification process를 무기한 점유하지 않습니다. |
| 98e4af62e884 | tests/runtime_stack.py | RuntimeStack.close error accumulation | teardown, image removal, temp cleanup의 failure를 수집하고 primary scenario outcome과 합칩니다. | scenario body success가 cleanup failure를 덮지 않습니다. |
| 98e4af62e884 | tests/runtime_stack.py | main result precedence | scenario exception, diagnostics failure, cleanup failure, unexpected exception의 exit status와 report order를 명시합니다. | evidence lifecycle 전체가 성공해야 process success입니다. |
| 98e4af62e884 | tools/rotate_secrets.py / related private writes | private temp fsync/cleanup hardening | test가 의존하는 secret/config temporary publication의 sync/cleanup contract도 강화합니다. | verification이 관측하는 resource state와 host file durability를 맞춥니다. |

#### 비교 기준

- exact commit diff: `git diff 98e4af62e884^ 98e4af62e884 -- <path>`
- 이전 Thread 상태와 비교: `git diff 7fbd41fe5af4 98e4af62e884 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | test body가 pass해도 teardown subprocess가 hang/fail하거나 diagnostics가 실패해 Docker resources/secrets가 남으면 전체 verification은 신뢰할 수 없습니다. |
| 선택한 boundary / decision | 모든 child wait를 bounded하고 cleanup errors를 primary result에 포함하는 explicit precedence를 만들었습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tests/runtime_stack.py`의 `explicit subprocess wait timeouts`; `tests/runtime_stack.py`의 `RuntimeStack.close error accumulation`; `tests/runtime_stack.py`의 `main result precedence`; `tools/rotate_secrets.py / related private writes`의 `private temp fsync/cleanup hardening` |
| state / ownership / lifecycle 변화 | RuntimeStack과 top-level main이 child processes, project resources, diagnostics, temporary files의 full lifecycle을 소유합니다. |
| 주요 failure branch | scenario success + cleanup failure는 nonzero입니다. primary failure와 cleanup failure가 함께 있으면 둘 다 보고하며 unexpected exception도 finally cleanup을 거칩니다. |
| 이 commit의 보장 | test outcome이 scenario assertions뿐 아니라 bounded process termination과 successful cleanup까지 포함합니다. |
| 한계와 다음 관련 commit | process crash 전에 ownership record가 남지 않은 resources를 모두 찾는 기능은 아직 제한됩니다. `2b35aa3d2217`이 exact ownership records와 crash recovery utility를 추가하고 `43ccded05e4f`이 full serial lifecycle을 구성합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: process crash 전에 ownership record가 남지 않은 resources를 모두 찾는 기능은 아직 제한됩니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `2b35aa3d2217`이 exact ownership records와 crash recovery utility를 추가하고 `43ccded05e4f`이 full serial lifecycle을 구성합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: test outcome이 scenario assertions뿐 아니라 bounded process termination과 successful cleanup까지 포함합니다.

### 8. `2b35aa3d2217` — test(cleanup): 테스트 프로젝트 소유 자원만 정리

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `OPERATIONS`, `RECOVERY`, `RISK` |
| Source-defined role | Tracked exact project ownership and added scoped leak recovery. |
| 이전 Thread commit | `98e4af62e884` |
| 다음 Thread commit | `43ccded05e4f` |

#### 원문이 확정한 범위

- **Summary:** Records exact test project ownership, removes only owned Compose resources and image tags, and adds a scoped crash-recovery cleanup tool with private reports.
- **Classification reason:** This solves a high-risk verification-lifecycle problem without broad Docker pruning, ensuring failed tests cannot damage unrelated developer or CI resources.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `2b35aa3d2217`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `ownership record publication`에서 process crash 뒤에도 어떤 project/image identities가 test-owned인지 남깁니다.
- `tools/cleanup_test_resources.py`의 `record validation / discovery`에서 malformed/untrusted record로 unrelated resource를 삭제하지 않습니다.
- `tools/cleanup_test_resources.py`의 `scoped removal and leak report`에서 crash recovery scope가 explicit ownership에 제한됩니다.
- `tests/runtime_stack.py`의 `normal close record lifecycle`에서 normal과 crash cleanup이 같은 identity source를 공유합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2b35aa3d2217 | tests/runtime_stack.py | ownership record publication | test project name과 image prefix를 private record directory `0700`의 strict `0600` files에 기록합니다. | process crash 뒤에도 어떤 project/image identities가 test-owned인지 남깁니다. |
| 2b35aa3d2217 | tools/cleanup_test_resources.py | record validation / discovery | record owner/mode/type/content를 검증하고 exact project labels/names와 image repository/tag prefix만 query합니다. | malformed/untrusted record로 unrelated resource를 삭제하지 않습니다. |
| 2b35aa3d2217 | tools/cleanup_test_resources.py | scoped removal and leak report | recorded projects를 down/remove하고 exact images만 지운 뒤 residuals를 보고하며 global Docker prune을 사용하지 않습니다. | crash recovery scope가 explicit ownership에 제한됩니다. |
| 2b35aa3d2217 | tests/runtime_stack.py | normal close record lifecycle | 정상 teardown이 끝나면 corresponding ownership record를 제거하고 cleanup failure면 report를 보존합니다. | normal과 crash cleanup이 같은 identity source를 공유합니다. |

#### 비교 기준

- exact commit diff: `git diff 2b35aa3d2217^ 2b35aa3d2217 -- <path>`
- 이전 Thread 상태와 비교: `git diff 98e4af62e884 2b35aa3d2217 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | test process가 crash하면 in-memory project list가 사라져 orphaned containers/volumes/networks/images를 안전하게 찾기 어렵고 broad prune은 unrelated resources를 삭제합니다. |
| 선택한 boundary / decision | private strict ownership records와 label/tag-scoped recovery tool을 도입했습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tests/runtime_stack.py`의 `ownership record publication`; `tools/cleanup_test_resources.py`의 `record validation / discovery`; `tools/cleanup_test_resources.py`의 `scoped removal and leak report`; `tests/runtime_stack.py`의 `normal close record lifecycle` |
| state / ownership / lifecycle 변화 | test harness가 record publication/removal을, recovery utility가 validated records에 해당하는 Docker resources만 소유합니다. |
| 주요 failure branch | unsafe/malformed records는 삭제 작업 전에 거부됩니다. 일부 removal failure나 residual leak는 report와 nonzero로 surfaced됩니다. |
| 이 commit의 보장 | crash 뒤에도 test가 명시적으로 소유한 project/image scope만 정리하고 unrelated Docker objects를 보존합니다. |
| 한계와 다음 관련 commit | record publication 전 crash하거나 external actor가 ownership label/tag를 위조한 경우까지 완전 해결하지는 않습니다. `43ccded05e4f`과 CI가 preparation/finally에서 recovery utility를 호출합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: record publication 전 crash하거나 external actor가 ownership label/tag를 위조한 경우까지 완전 해결하지는 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `43ccded05e4f`과 CI가 preparation/finally에서 recovery utility를 호출합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: crash 뒤에도 test가 명시적으로 소유한 project/image scope만 정리하고 unrelated Docker objects를 보존합니다.

### 9. `43ccded05e4f` — test(verify): 전체 스택 검증을 직렬 실행

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `OPERATIONS`, `RECOVERY` |
| Source-defined role | Serialized the complete local verification lifecycle. |
| 이전 Thread commit | `2b35aa3d2217` |
| 다음 Thread commit | `18508c25eef0` |

#### 원문이 확정한 범위

- **Summary:** Runs static checks and all runtime scenarios serially with per-scenario timeouts, shared project records, and mandatory final leak recovery.
- **Classification reason:** It defines the complete local verification transaction and makes resource cleanliness part of success, significantly improving confidence and failure attribution.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `43ccded05e4f`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/verify_stack.py`의 `serial scenario plan`에서 공유 host Docker capacity와 logs/records가 concurrent scenarios로 섞이지 않습니다.
- `tools/verify_stack.py`의 `per-step timeout/result handling`에서 hang/failure가 unbounded pipeline이나 cleanup skip으로 이어지지 않습니다.
- `tools/verify_stack.py`의 `pre/post crash recovery`에서 이전 crash와 현재 run leak를 같은 bounded lifecycle에서 처리합니다.
- `tools/verify_stack.py`의 `outcome precedence`에서 green result가 resource leak를 숨기지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 43ccded05e4f | tools/verify_stack.py | serial scenario plan | static validation과 config render 뒤 bootstrap, e2e, persistence, backup/restore, rotation, operations scenarios를 정해진 순서로 한 번에 하나씩 실행합니다. | 공유 host Docker capacity와 logs/records가 concurrent scenarios로 섞이지 않습니다. |
| 43ccded05e4f | tools/verify_stack.py | per-step timeout/result handling | 각 command에 explicit timeout을 주고 failure 시 diagnostics를 수집하되 remaining/final cleanup semantics를 보존합니다. | hang/failure가 unbounded pipeline이나 cleanup skip으로 이어지지 않습니다. |
| 43ccded05e4f | tools/verify_stack.py | pre/post crash recovery | 시작 전 stale ownership records를 정리하고 finally에서 cleanup utility와 residual report를 실행합니다. | 이전 crash와 현재 run leak를 같은 bounded lifecycle에서 처리합니다. |
| 43ccded05e4f | tools/verify_stack.py | outcome precedence | scenario failure, diagnostic failure, cleanup leak 중 하나라도 final nonzero를 만들며 cleanup report를 primary result와 함께 보존합니다. | green result가 resource leak를 숨기지 않습니다. |

#### 비교 기준

- exact commit diff: `git diff 43ccded05e4f^ 43ccded05e4f -- <path>`
- 이전 Thread 상태와 비교: `git diff 2b35aa3d2217 43ccded05e4f -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 개별 test commands는 있었지만 operator가 일부만 실행하거나 parallel/interrupted run에서 stale resources와 result precedence를 일관되게 관리하기 어려웠습니다. |
| 선택한 boundary / decision | 전체 verification을 serial ordered plan, bounded subprocess, diagnostics, pre/final cleanup으로 묶었습니다. |
| 핵심 caller/callee 또는 configuration consumer | `tools/verify_stack.py`의 `serial scenario plan`; `tools/verify_stack.py`의 `per-step timeout/result handling`; `tools/verify_stack.py`의 `pre/post crash recovery`; `tools/verify_stack.py`의 `outcome precedence` |
| state / ownership / lifecycle 변화 | verify orchestrator가 local evidence lifecycle과 command order를 소유하고 각 scenario는 자신의 project resources만 소유합니다. |
| 주요 failure branch | 어느 단계 failure/timeout/unexpected exception에서도 finally cleanup을 실행하고 cleanup failure도 nonzero에 포함합니다. |
| 이 commit의 보장 | 하나의 local command가 static부터 모든 runtime scenarios, diagnostics, leak recovery를 serial bounded lifecycle로 실행합니다. |
| 한계와 다음 관련 commit | machine crash로 finally가 실행되지 않는 경우는 다음 run의 ownership-record recovery에 의존합니다. `18508c25eef0`이 같은 lifecycle을 least-privilege CI에 자동화하고 `8a6c07988160`이 orchestrator/workflow 자체를 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: machine crash로 finally가 실행되지 않는 경우는 다음 run의 ownership-record recovery에 의존합니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `18508c25eef0`이 같은 lifecycle을 least-privilege CI에 자동화하고 `8a6c07988160`이 orchestrator/workflow 자체를 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: 하나의 local command가 static부터 모든 runtime scenarios, diagnostics, leak recovery를 serial bounded lifecycle로 실행합니다.

### 10. `18508c25eef0` — ci(stack): 정적·런타임·복구 검증 자동화

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `OPERATIONS`, `SUPPLY_CHAIN` |
| Source-defined role | Automated all scenarios under least-privilege, pinned CI actions. |
| 이전 Thread commit | `43ccded05e4f` |
| 다음 Thread commit | `8a6c07988160` |

#### 원문이 확정한 범위

- **Summary:** Adds a least-privilege, pinned-action GitHub Actions workflow running all static and runtime scenarios, scoped cleanup, and allowlisted failure diagnostics.
- **Classification reason:** This is significant integration of the project's verification, supply-chain, and resource-ownership policies into automation, though it does not alter product runtime behavior.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `18508c25eef0`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `.github/workflows/container-stack.yml`의 `runner/timeout/permissions/concurrency`에서 workflow token과 execution duration을 최소/한정합니다.
- `.github/workflows/container-stack.yml`의 `pinned actions / full checkout`에서 CI dependency와 historical verification input을 명시합니다.
- `.github/workflows/container-stack.yml`의 `serial verification / always cleanup`에서 local lifecycle의 cleanup/result semantics를 CI에도 유지합니다.
- `.github/workflows/container-stack.yml`의 `artifact allowlist`에서 CI artifact가 새로운 disclosure channel이 되지 않게 합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 18508c25eef0 | .github/workflows/container-stack.yml | runner/timeout/permissions/concurrency | Ubuntu 24.04 runner, 전체 210분 timeout, `contents: read`, concurrency cancel policy를 선언합니다. | workflow token과 execution duration을 최소/한정합니다. |
| 18508c25eef0 | .github/workflows/container-stack.yml | pinned actions / full checkout | checkout와 artifact upload를 immutable commit SHA로 고정하고 full history를 받아 commit-range/source checks를 가능하게 합니다. | CI dependency와 historical verification input을 명시합니다. |
| 18508c25eef0 | .github/workflows/container-stack.yml | serial verification / always cleanup | static/config/runtime scenarios를 serial 실행하고 failure와 무관하게 diagnostics/ownership cleanup을 `always()` path에서 수행합니다. | local lifecycle의 cleanup/result semantics를 CI에도 유지합니다. |
| 18508c25eef0 | .github/workflows/container-stack.yml | artifact allowlist | 실패 evidence는 redacted diagnostics와 cleanup reports의 좁은 path만 업로드하고 secret/env dump를 포함하지 않습니다. | CI artifact가 새로운 disclosure channel이 되지 않게 합니다. |

#### 비교 기준

- exact commit diff: `git diff 18508c25eef0^ 18508c25eef0 -- <path>`
- 이전 Thread 상태와 비교: `git diff 43ccded05e4f 18508c25eef0 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### A-level 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | local verify만으로는 모든 변경에 자동 실행되지 않고 CI가 broad token/action tags/unbounded runtime/unsafe artifact를 사용하면 verification 자체가 위험해질 수 있었습니다. |
| 선택한 boundary / decision | least privilege, pinned actions, bounded timeout, serial scenarios, always cleanup, allowlisted failure evidence를 workflow policy로 만들었습니다. |
| 핵심 caller/callee 또는 configuration consumer | `.github/workflows/container-stack.yml`의 `runner/timeout/permissions/concurrency`; `.github/workflows/container-stack.yml`의 `pinned actions / full checkout`; `.github/workflows/container-stack.yml`의 `serial verification / always cleanup`; `.github/workflows/container-stack.yml`의 `artifact allowlist` |
| state / ownership / lifecycle 변화 | CI job이 ephemeral runner의 checkout/Docker resources/artifacts lifecycle을 소유하며 workflow token은 read-only contents 범위만 가집니다. |
| 주요 failure branch | scenario failure, timeout, cancellation에서도 cleanup steps가 실행되며 diagnostics/cleanup failure는 job status/evidence에 반영됩니다. |
| 이 commit의 보장 | repository change마다 동일 static/runtime/recovery suite를 bounded least-privilege environment에서 자동 실행합니다. |
| 한계와 다음 관련 commit | workflow text가 미래 편집으로 약화되지 않는다는 보장은 별도 self-test 없이는 없습니다. `8a6c07988160`이 workflow, tools, secret/timeout/cleanup contract를 static/AST/mock layers로 검증합니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: workflow text가 미래 편집으로 약화되지 않는다는 보장은 별도 self-test 없이는 없습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: `8a6c07988160`이 workflow, tools, secret/timeout/cleanup contract를 static/AST/mock layers로 검증합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: repository change마다 동일 static/runtime/recovery suite를 bounded least-privilege environment에서 자동 실행합니다.

### 11. `8a6c07988160` — test(ci): workflow 검증 계약 추가

| 항목 | 원문 확정값 |
| --- | --- |
| Importance | **A** |
| Tags | `TEST`, `OPERATIONS`, `RISK` |
| Source-defined role | Validated the workflow, tool timeouts, secret boundaries, cleanup, and artifact allowlist. |
| 이전 Thread commit | `18508c25eef0` |
| 다음 Thread commit | 없음 |

#### 원문이 확정한 범위

- **Summary:** Expands static and AST-based checks to enforce workflow permissions, action pins, scenario ordering, timeouts, secret boundaries, cleanup semantics, and safe subprocess use.
- **Classification reason:** The commit protects the verification system itself from subtle weakening and provides layered evidence for security and lifecycle properties across many tools.

#### 해당 SHA에서 확인할 코드

> 기준 commit은 반드시 `8a6c07988160`입니다. final HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `workflow text contract`에서 CI policy의 필수 요소를 source regression으로 고정합니다.
- `tests/validate_stack.py`의 `forbidden workflow patterns`에서 검증 infrastructure의 high-risk shortcuts를 fail closed합니다.
- `tests/validate_stack.py`의 `Compose service-block parser`에서 global substring보다 service responsibility를 정확히 검증합니다.
- `tests/validate_stack.py`의 `Python AST visitors`에서 text 검색이 놓칠 management-tool semantic weakening을 탐지합니다.
- `tests/validate_stack.py`의 `mocked main-path probes`에서 Docker 없이도 orchestrator failure semantics 일부를 deterministic하게 확인합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / command | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 8a6c07988160 | tests/validate_stack.py | workflow text contract | runner, timeout, permissions, concurrency, action SHAs, full checkout, serial commands, diagnostics, always cleanup, artifact allowlist를 exact pattern으로 검사합니다. | CI policy의 필수 요소를 source regression으로 고정합니다. |
| 8a6c07988160 | tests/validate_stack.py | forbidden workflow patterns | `pull_request_target`, secret contexts, shell tracing, environment dumping, broad Docker prune, unpinned action references를 거부합니다. | 검증 infrastructure의 high-risk shortcuts를 fail closed합니다. |
| 8a6c07988160 | tests/validate_stack.py | Compose service-block parser | runtime service별 exact mounts/env/networks를 parse해 password env, `/run/secrets`, Nginx private config mount 등 forbidden exposure를 검사합니다. | global substring보다 service responsibility를 정확히 검증합니다. |
| 8a6c07988160 | tests/validate_stack.py | Python AST visitors | subprocess wait/communicate의 explicit timeout과 startup secret read가 project lock lexical/control-flow 안에 있는지 AST로 검사합니다. | text 검색이 놓칠 management-tool semantic weakening을 탐지합니다. |
| 8a6c07988160 | tests/validate_stack.py | mocked main-path probes | preparation timeout, scenario failure, cleanup failure, unexpected exception을 mock해 exit status와 cleanup invocation/result precedence를 검사합니다. | Docker 없이도 orchestrator failure semantics 일부를 deterministic하게 확인합니다. |

#### 비교 기준

- exact commit diff: `git diff 8a6c07988160^ 8a6c07988160 -- <path>`
- 이전 Thread 상태와 비교: `git diff 18508c25eef0 8a6c07988160 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | CI와 management/test tools 자체가 least-privilege, pinned, bounded, secret-safe, scoped-cleanup/result-precedence contract를 유지합니다. |
| 재현하는 failure / boundary | workflow/tool source가 green appearance를 유지한 채 permissions/timeouts/secret/cleanup semantics를 약화하는 경계입니다. |
| test technique | static text + service-block parse + Python AST + mocked unit-style control-flow probes |
| fixture와 failure injection | workflow YAML, Compose, Python tools를 읽고 selected main paths를 mock subprocess/cleanup behavior로 호출합니다. |
| 실제 통과하는 production path | validator source parsing과 imported verify/runtime main error branches를 통과하며 Docker production path는 실행하지 않습니다. |
| 핵심 assertion | 필수/금지 workflow pattern, action pins, timeouts, lock/secret order, cleanup invocation, exit/result precedence를 확인합니다. |
| 이 테스트가 증명하는 것 | verification infrastructure의 명시적 policy와 일부 failure control flow가 약화되지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 실제 CI service enforcement, live Docker/container behavior, 모든 dynamic Python path는 증명하지 않습니다. |
| 성격 | verification-system source/control-flow regression |
| 막는 후속 regression | broad permissions, unpinned actions, unsafe event/secrets, timeout 제거, global prune, cleanup failure 무시, lock 밖 secret read를 막습니다. |
| 직접 실행 command와 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository checkout이 없습니다. 해당 SHA의 test code와 command wiring만 검사했습니다. |

#### 다음 연결

- 이 commit 뒤에도 남아 있는 불충분한 보장: GitHub-hosted runner가 실제로 모든 policy를 적용하는지, Docker runtime behavior 전체, AST가 모델링하지 않은 dynamic path는 증명하지 않습니다.
- 다음 관련 commit이 바꾸거나 검증하는 지점: Thread 전체의 “verification of verification” endpoint이며 runtime tests와 상호 보완합니다.
- 이 commit을 제거했을 때 Thread 설명에서 생기는 공백: verification infrastructure 자체가 least-privilege/bounded/secret-safe/scoped-cleanup contract를 source와 selected control-flow 수준에서 유지함을 증명합니다.

## Invariant ledger

| Source에서 연결된 invariant | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| MariaDB는 internal backend에만 있고 Nginx는 DB network에 접근하지 않습니다. | 27a3dca01d3b | 27a3dca01d3b | 7fbd41fe5af4 | Compose exact membership과 live Docker network inspect가 Nginx/frontend, MariaDB/backend, WordPress dual-homed 상태를 확인합니다. |
| destructive cleanup은 exact selected project confirmation과 owned-resource scope를 요구합니다. | 74c285925325 | 2b35aa3d2217 | 7fbd41fe5af4, 43ccded05e4f, 18508c25eef0 | Make exact-name guard와 records/labels/tags scoped cleanup이 generic deletion/global prune를 대신합니다. |
| diagnostics는 private, redacted, non-overwriting이며 sanitization 불가 시 아무것도 게시하지 않습니다. | ef74ad47ea81 | 27a083d91c87 | 7fbd41fe5af4 | complete masking set, 0700/0600 O_EXCL output, rescan, error-directory removal과 unsafe fixtures가 연결됩니다. |
| runtime test cleanup 실패와 residual leak는 verification failure입니다. | 98e4af62e884 | 2b35aa3d2217, 43ccded05e4f | 18508c25eef0, 8a6c07988160 | close error accumulation, ownership records, final leak recovery, CI/self-test result precedence가 연결됩니다. |
| cleanup과 recovery는 global Docker prune 없이 exact project labels/names/image tags만 제거합니다. | 2b35aa3d2217 | 43ccded05e4f | 18508c25eef0, 8a6c07988160 | private records와 exact discovery/removal, forbidden prune pattern이 ownership discipline을 고정합니다. |
| automation command는 bounded timeout, least privilege, pinned dependencies, allowlisted evidence를 유지합니다. | 43ccded05e4f | 18508c25eef0 | 8a6c07988160 | serial timeouts, read-only token, SHA-pinned actions, always cleanup, artifact allowlist와 text/AST/mock tests가 연결됩니다. |

### Ledger 보완 기록

- source에 명시되지 않은 새 invariant를 확정 사실로 추가하지 않습니다.
- invariant가 실제로 부족했음을 드러낸 commit 또는 failure stage: single network, unguarded destructive cleanup, best-effort diagnostics/teardown와 독립 test commands는 failure 반경과 leak를 verification success 밖에 남겼습니다.
- marker, rename, lock, health, authentication, cleanup 등 invariant를 고정하는 concrete mechanism: internal backend, exact project confirmation, fail-closed private diagnostics, owned-resource records, serial bounded orchestration와 least-privilege pinned workflow가 operational ownership을 고정합니다.
- 후속 commit이 invariant를 약화하지 못하게 하는 regression evidence: `7fbd41fe5af4`, `98e4af62e884`, `43ccded05e4f`, `18508c25eef0`, `8a6c07988160`이 live policy, result precedence, cleanup과 workflow 자체를 계층적으로 검증합니다.
## Failure → Fix → Test 연결

| failure / 위험 | fix 또는 mechanism | test / evidence | 학습자 연결 기록 |
| --- | --- | --- | --- |
| single shared network가 frontend에 DB reachability 부여 | 27a3dca01d3b internal backend/exact memberships | 7fbd41fe5af4 live network inspect | WordPress만 request와 persistence networks를 연결합니다. |
| generic confirmation으로 wrong project volume 삭제 | 74c285925325 exact PROJECT_NAME confirmation | 7fbd41fe5af4 refusal+health/state | destructive scope를 operation 시점에 다시 명시합니다. |
| diagnostic bundle이 credential/path 누출 또는 partial output | ef74ad47ea81 redaction + 27a083d91c87 private fail-closed publication | 7fbd41fe5af4 real log/unreadable/overwrite/symlink cases | sanitize input이 불완전하면 일부 결과도 게시하지 않습니다. |
| scenario pass지만 teardown failure로 resources 누수 | 98e4af62e884 cleanup error result propagation | 2b35aa3d2217 ownership records + 43ccded05e4f final recovery | cleanup은 test 외 부수 작업이 아니라 evidence success 조건입니다. |
| crash cleanup이 broad prune로 unrelated resources 삭제 | 2b35aa3d2217 label/tag-scoped recovery | 43ccded05e4f/CI final cleanup과 8a6c07988160 no-prune guard | explicit ownership 이외에는 삭제하지 않습니다. |
| CI green이지만 permissions/action pins/timeout/secret/cleanup 약화 | 18508c25eef0 policy-rich workflow | 8a6c07988160 text/AST/mock self-validation | verification system도 versioned executable contract로 검증합니다. |

### 직접 재구성할 chain

```text
기존 가정: Compose declaration과 scenario body success만으로 운영 policy와 test success를 판단할 수 있다는 가정
  → 실제 failure 또는 위험: effective runtime mismatch, wrong-project deletion, secret-bearing partial diagnostics, cleanup leak와 weakened CI가 green result 뒤에 숨을 수 있었습니다.
  → root cause: policy 적용 상태, diagnostics publication, resource ownership과 verification control flow가 success condition에 포함되지 않았습니다.
  → 수정된 invariant / decision: live inspect와 negative checks를 수행하고 diagnostics/cleanup/leak 결과를 primary outcome에 합치며 exact owned resources만 제거합니다.
  → 해당 SHA의 실제 수정 코드: `27a3dca01d3b`~`2b35aa3d2217` operational mechanisms와 `43ccded05e4f`/`18508c25eef0` orchestrators
  → failure injection 또는 regression test: `7fbd41fe5af4`, `98e4af62e884`, `8a6c07988160` runtime/static/AST/mock evidence
  → 증명된 보장 / 남은 비보장: network/limits/guard/redaction/cleanup/CI policy weakening을 검출하지만 이번 환경에서는 Docker와 Actions를 직접 실행하지 않았습니다.
```

## Ownership / state / responsibility 변화

| 대상 | 이전 상태 | 이후 책임/authoritative state | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Networks | single shared bridge | frontend/backend 분리; WordPress만 dual-homed | 27a3dca01d3b Compose + 7fbd41fe5af4 inspect | Nginx는 DB addressability를 갖지 않습니다. |
| Service resource lifecycle | Docker defaults | service-specific limits, stop signal/timeout, log rotation | 911544133fb4 + live inspect | Docker effective state까지 확인 대상입니다. |
| Diagnostics output | arbitrary existing path/broad inspect 가능 | new private directory와 allowlisted redacted files | ef74ad47ea81, 27a083d91c87 | collector가 complete publication lifecycle을 소유합니다. |
| Scenario cleanup | best-effort teardown | test result 일부이며 exact project/image ownership만 제거 | 98e4af62e884, 2b35aa3d2217 | cleanup failure는 success를 무효화합니다. |
| Leak recovery utility | broad manual cleanup 위험 | strict private records와 exact labels/tags 기반 recovery | 2b35aa3d2217 | global prune를 사용하지 않습니다. |
| Local/CI orchestrator | 개별 commands | serial bounded lifecycle, diagnostics, final cleanup, outcome precedence 소유 | 43ccded05e4f, 18508c25eef0, 8a6c07988160 | verification infrastructure 자체도 contract로 검사합니다. |

## Thread 최종 상태

- **Source-confirmed endpoint:** This progression turns operational policy into executable evidence. Runtime limits and network boundaries are inspected on live containers; destructive commands and diagnostics fail safely; and both local and CI runners account for every project resource they create. The cleanup tooling deliberately avoids global Docker pruning, preserving the same ownership discipline used by the product management paths.
- 최종 authoritative state와 owner: Compose가 network/resource/shutdown/log policy를, private diagnostics directory가 redacted evidence를, ownership records가 test resource scope를 소유합니다.
- 정상 실행의 entry point와 완료 조건: local/CI orchestrator가 static/config/runtime scenarios를 serial bounded execution하고 diagnostics와 final cleanup까지 성공해야 정상 완료입니다.
- failure 또는 interruption 뒤 retry/rollback/compensation 조건: scenario/diagnostic/cleanup/timeout/unexpected failure는 모두 nonzero에 반영되며 records 기반 exact cleanup을 시도하고 residual leak를 보고합니다.
- 이 Thread가 다른 Thread에 제공하는 전제: 앞선 모든 Threads의 architecture와 recovery properties를 live inspect하고 지속적으로 자동 검증하는 운영 lifecycle을 제공합니다.
- 이 Thread 단독으로는 증명하지 않는 것: Docker가 없는 현재 환경에서는 live operations/CI workflow를 실행하지 않았으며 source/commit inspection으로 mechanism만 확인했습니다.

## 최종 architecture 또는 execution flow 정리

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | Compose policy 선언 | 27a3dca01d3b, 911544133fb4 | network membership과 runtime limits/stop/log/security를 정의합니다. | misconfiguration은 render/start 또는 live test failure입니다. |
| 2 | effective operations test | 7fbd41fe5af4 | Docker inspect와 negative commands로 실제 적용을 확인합니다. | policy/guard/diagnostic mismatch면 scenario 실패입니다. |
| 3 | redaction set 생성 | ef74ad47ea81 diagnose_stack | rendered secret paths와 values를 안전하게 읽어 longer-first/structural masking합니다. | 하나라도 읽지 못하면 bundle을 만들지 않습니다. |
| 4 | private diagnostic publish | 27a083d91c87 | 0700 directory/0600 allowlisted files를 O_EXCL로 쓰고 rescan합니다. | failure/secret residue/existing path면 directory 전체를 제거합니다. |
| 5 | scenario result+cleanup | 98e4af62e884 | body, diagnostics, teardown errors를 한 final status로 합칩니다. | cleanup failure는 primary success를 무효화합니다. |
| 6 | ownership/crash recovery | 2b35aa3d2217 | private records와 exact labels/tags만 제거합니다. | unsafe record나 residual leak는 nonzero이며 global prune는 금지됩니다. |
| 7 | local serial verify | 43ccded05e4f | all scenarios를 timeout/diagnostics/final cleanup과 순서대로 실행합니다. | 각 failure에도 finally cleanup과 result precedence를 유지합니다. |
| 8 | CI automation | 18508c25eef0 | least-privilege/pinned actions/allowlisted artifacts로 같은 lifecycle을 실행합니다. | timeout/cancel/failure에도 always cleanup을 시도합니다. |
| 9 | self-validation | 8a6c07988160 | workflow text, Compose blocks, Python AST, mocked main paths를 검사합니다. | verification policy weakening을 static failure로 바꿉니다. |

### 학습자의 최종 설명

> 운영 강화는 옵션을 많이 추가하는 것이 아니라 실패 반경과 소유권을 줄이는 과정입니다. frontend/backend를 분리해 WordPress만 dual-homed bridge로 남기고, service-specific limits와 shutdown/log policy를 선언한 뒤 live inspect로 effective state를 확인합니다. destructive volume deletion은 selected project name의 exact confirmation을 요구합니다. diagnostics는 complete secret/path masking set을 안전하게 만들 수 있을 때만 새 private directory에 allowlisted files를 쓰고 final rescan하며, 어느 단계 실패든 output 전체를 제거합니다. runtime tests는 cleanup failure도 test failure로 취급하고 private ownership records를 남겨 crash recovery가 exact projects/images만 제거하게 합니다. local verify와 CI는 모든 scenarios를 serial timeout으로 실행하고 diagnostics/cleanup/result precedence를 보존하며, 마지막 static/AST/mock tests가 workflow와 tools 자체의 policy weakening까지 검사합니다.

## 학습 완료 자가 점검

- [x] Nginx와 MariaDB가 같은 network에 남아 있다고 설명하지 않았습니까?
- [x] Compose declaration만 보고 effective runtime limits를 확인했다고 간주하지 않았습니까?
- [x] diagnostic redaction 실패 시 partial bundle을 남긴다고 잘못 기록하지 않았습니까?
- [x] cleanup을 product 외 부수 작업으로 취급해 scenario success와 분리하지 않았습니까?
- [x] recovery utility가 Docker prune을 사용한다고 쓰지 않았습니까?
- [x] CI workflow만 보고 verification tools의 control flow 계약을 생략하지 않았습니까?
- [x] 모든 code snippet에 SHA와 path/symbol을 기록했습니다.
- [x] final HEAD의 field/helper/test를 이전 SHA에 소급하지 않았습니다.
- [x] source가 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 Thread를 commit 순서대로 구두 설명할 수 있습니다.
===== END FILE: 08-operational-hardening-and-automation.md =====

===== BEGIN FILE: README.md =====
# Inception Development Thread 학습 골격

## 목적

이 문서 세트는 `web/inception`의 실제 commit history와 각 SHA 시점의 코드를 직접 읽으며 설계, 구현, 실패 처리, 수정, 검증의 발전 과정을 복원하기 위한 기록 골격입니다.

문서에 미리 작성된 SHA, subject, importance, tags, Thread 순서, Source-defined role과 분류 근거는 제공된 두 source 문서의 확정값입니다. 실제 함수 동작, 변경 전후 코드, ownership/lifetime, 실패 처리, 테스트 실행 결과, 최종 설명은 학습자가 해당 SHA의 코드를 확인해 작성해야 합니다.

## 권장 학습 순서

1. [`01-readiness-aware-three-tier-stack.md`](01-readiness-aware-three-tier-stack.md)
2. [`02-convergent-one-off-bootstrap.md`](02-convergent-one-off-bootstrap.md)
3. [`03-isolated-runtime-and-persistence.md`](03-isolated-runtime-and-persistence.md)
4. [`04-atomic-backup-publication.md`](04-atomic-backup-publication.md)
5. [`05-fresh-project-restore.md`](05-fresh-project-restore.md)
6. [`06-credential-rotation-and-compensation.md`](06-credential-rotation-and-compensation.md)
7. [`07-immutable-build-inputs.md`](07-immutable-build-inputs.md)
8. [`08-operational-hardening-and-automation.md`](08-operational-hardening-and-automation.md)

이 순서는 source의 Development Threads 순서와 같습니다. 동일 SHA가 여러 Thread에 나타나면 제거하지 말고 각 문서의 관점으로 다시 확인합니다.

## Thread 문서 사용법

1. 먼저 Commit map으로 Thread의 상태 변화와 중요도 분포를 확인합니다.
2. 각 commit에서 `Source-defined role`, `Summary`, `Classification reason`을 읽고 source가 확정한 범위를 고정합니다.
3. `해당 SHA에서 확인할 코드`에 적힌 항목을 실제 tree와 diff에서 찾습니다.
4. 코드 근거 표에는 경로, symbol/directive, 핵심 line, caller/callee 또는 producer/consumer 관계를 함께 기록합니다.
5. Invariant ledger에서 도입, 강화, 부족함 노출, fix, regression test의 연결을 채웁니다.
6. Thread 마지막에는 최종 architecture 또는 execution flow를 자신의 코드 근거만으로 설명합니다.

## 해당 SHA 코드 확인 원칙

항상 commit 당시의 tree를 기준으로 확인합니다.

```bash
git show --stat --summary <sha>
git diff <sha>^ <sha> -- <path>
git show <sha>:<path>
```

Thread의 이전 관련 commit과 비교할 때는 다음처럼 별도 diff를 사용합니다.

```bash
git diff <previous-thread-sha> <current-sha> -- <path>
```

파일명이나 symbol이 source에 명시되지 않은 경우 먼저 `git show --name-status <sha>`로 변경 파일을 식별한 뒤 기록합니다. 최종 HEAD의 동일 파일을 열어 과거 commit의 동작을 추정하지 않습니다.

## final HEAD 소급 사용 금지

- 후속 refactor, fix, test에서 추가된 field, helper, marker, volume, network, timeout을 이전 SHA의 설계로 기록하지 않습니다.
- 현재 HEAD에서 사라진 초기 구현도 해당 SHA에서 직접 확인합니다.
- 후속 commit의 code는 “다음 변화” 또는 비교 대상으로만 사용하고 현재 commit의 근거로 대체하지 않습니다.
- source가 확정하지 않은 invariant를 새 사실처럼 추가하지 않습니다.

## Importance별 학습 깊이

### S

프로젝트의 defining architecture 또는 core state transaction으로 다룹니다. 문제, 직전 상태, failure 가능성, 핵심 결정, actual code, ownership/lifecycle/state transition, rollback/compensation, 보장과 비보장, 후속 fix/test까지 추적합니다.

### A

주요 subsystem, security/persistence/lifecycle boundary, integration point, non-trivial 실패 처리를 이해할 수 있을 정도로 actual code와 설계 판단을 확인합니다. Test commit은 production invariant, injected failure, technique, traversed path, 증명 범위를 분리합니다.

### B

Thread 흐름에서 맡은 구현 역할과 필요한 상태 변화를 확인합니다. 핵심 파일, directive/helper, input/output, immediate failure branch, 다음 commit에 넘기는 한계를 기록합니다.

### C

Thread 이해에 필요한 맥락만 확인합니다. 동일한 깊이의 분석란을 억지로 확장하지 않습니다. 현재 Development Threads에는 C commit이 없지만 분류 원칙은 유지합니다.

## 실제 코드 삽입 기준

- 설명을 대신하는 대량 복사는 피하고 invariant, state mutation order, ownership transfer, failure branch를 증명하는 최소 범위만 삽입합니다.
- snippet마다 반드시 SHA, path, symbol/directive, line range 또는 인접 문맥을 기록합니다.
- shell/Compose/Python 설정은 caller와 consumer를 함께 적습니다. 예를 들어 environment mapping만 넣지 말고 이를 읽는 entrypoint/helper도 연결합니다.
- test code는 fixture setup, failure injection, production command/path, assertion을 한 세트로 기록합니다.
- code excerpt만으로 의미가 불분명하면 앞뒤 상태와 실패 시 결과를 자신의 문장으로 설명합니다.

## Test commit 학습 방법

각 Test commit에서 다음을 반드시 분리합니다.

- 대상으로 하는 production invariant
- 재현하는 failure 또는 boundary
- 사용하는 technique: static source contract, rendered configuration, live integration, deterministic pause/signal, SIGKILL, AST/control-flow probe 등
- 실제로 통과하는 production code path
- test가 증명하는 것
- test가 증명하지 않는 것
- broad integration인지 deterministic regression인지
- 후속 변경에서 막는 regression
- 직접 실행한 command와 실제 결과

테스트가 성공했다는 사실만 기록하지 말고 실패 주입 위치와 assertion이 production invariant에 연결되는 과정을 남깁니다.

## 문서 완료 기준

- 8개 Development Thread 문서의 모든 commit을 source 순서대로 검토했습니다.
- 모든 SHA, subject, importance, tags를 변경하지 않았습니다.
- 동일 commit이 여러 Thread에 있는 경우 각 관점의 기록을 모두 작성했습니다.
- 중요한 commit마다 해당 SHA의 실제 파일과 symbol/directive 근거가 있습니다.
- S/A/B/C 깊이 차이가 기록량과 질문 범위에 반영되어 있습니다.
- fix는 기존 가정 → failure/risk → root cause → corrected invariant → code → regression test로 연결했습니다.
- test는 production invariant, failure technique, traversed path, 증명/비증명 범위를 구분했습니다.
- 각 Thread의 Invariant ledger, Failure → Fix → Test, ownership/state 변화, final flow, 자가 점검을 완료했습니다.
- final HEAD의 구현을 과거 SHA에 소급한 설명이 없습니다.
- 최종적으로 commit history에 근거해 설계 → 구현 → 실패 → 수정 → 검증의 발전을 다시 설명할 수 있습니다.
===== END FILE: README.md =====

