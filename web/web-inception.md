===== BEGIN FILE: 01-readiness-aware-three-tier-stack.md =====
# 개발 흐름 1 — 개별 서비스에서 준비 상태를 확인하는 3계층 스택까지

## 개발 흐름의 목표

개별 MariaDB, WordPress, Nginx 서비스가 Docker Compose 안에서 하나의 상태를 갖는 시스템으로 결합되는 과정을 복원합니다. 핵심은 이미지 세 개의 존재가 아니라 외부 transport, 애플리케이션 실행, 영속 상태의 책임을 분리하고 준비 상태와 명명된 볼륨으로 연결한 결정입니다.

### 원문에서 정한 의의

개별 실행 가능한 컨테이너를 하나의 상태 있는 애플리케이션 시스템으로 결합합니다. Nginx는 외부 TLS와 요청 전달, WordPress는 애플리케이션 실행, MariaDB는 영속 관계형 데이터를 맡으며, 명명된 볼륨과 준비 상태 기반 의존성으로 단순 연결을 실제 운영 가능한 토폴로지로 바꿉니다.

<details>
<summary>영문 원문</summary>

> The thread progresses from individually runnable containers to one stateful application system. The decisive step is not the existence of three images but the Compose responsibility boundary: Nginx owns external transport, WordPress owns application execution, and MariaDB owns durable relational state. Health-gated dependencies and mounted volumes then make the topology operationally meaningful rather than merely connected.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- 각 서비스는 어떤 책임을 독점하며, 어떤 책임을 갖지 않습니까?
- 외부 HTTPS 요청은 어떤 설정 계약을 거쳐 PHP-FPM과 MariaDB까지 도달합니까?
- 이미지 계층의 상태와 명명된 볼륨의 권위 있는 상태는 어디에서 분리됩니까?
- 컨테이너 생성 순서와 실제 서비스 준비 상태는 어떻게 구분됩니까?
- 초기 idempotent 엔트리포인트가 제공한 보장과 이후 interruption-safe 초기화가 추가로 해결한 문제는 무엇입니까?

## 완료 기준

- 세 서비스의 빌드/런타임/볼륨/네트워크 책임을 해당 SHA의 설정과 엔트리포인트로 설명할 수 있습니다.
- Nginx → FastCGI → WordPress → MariaDB 경로를 실제 directive와 서비스 name으로 추적했습니다.
- 초기화 조건, 볼륨 재사용 조건, 헬스 체크 조건을 서로 혼동하지 않고 기록했습니다.
- 초기 설계가 interruption-safe하지 않았던 지점을 후속 초기화 thread와 연결했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `f8ec9621725c` | feat(mariadb): Debian 서버 이미지 추가 | **B** | `STACK`<br>`PERSISTENCE` | 프로젝트가 직접 관리하는 MariaDB 실행 환경과 영속 데이터 소유권을 정했습니다. |
| 2 | `e13b0357a21b` | feat(mariadb): DB와 애플리케이션 계정 초기화 | **A** | `BOOTSTRAP`<br>`SECRETS`<br>`CORE` | 최초의 재실행 가능한 DB·계정 초기화를 추가했습니다. |
| 3 | `d764d066167b` | feat(wordpress): 사이트와 사용자 계정 초기화 | **A** | `BOOTSTRAP`<br>`PERSISTENCE`<br>`CORE` | WordPress 파일·사이트·사용자 상태를 재실행 시 수렴하도록 추가했습니다. |
| 4 | `99c03f54399a` | feat(nginx): PHP 요청을 WordPress로 전달 | **A** | `STACK`<br>`INTEGRATION`<br>`CORE` | HTTPS 요청을 FastCGI로 넘기는 경계를 정의했습니다. |
| 5 | `a8b9f693c614` | feat(compose): 세 서비스 토폴로지 구성 | **S** | `ARCH`<br>`STACK`<br>`CORE` | 세 서비스의 역할을 핵심 Compose 토폴로지로 결합했습니다. |
| 6 | `75590dedfb3a` | feat(compose): 준비 상태에 따라 영속 서비스 연결 | **A** | `PERSISTENCE`<br>`INTEGRATION`<br>`OPERATIONS` | 명명된 볼륨, 헬스 체크, 준비 상태 기반 의존성을 연결했습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `f8ec9621725c` — feat(mariadb): Debian 서버 이미지 추가

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `STACK`, `PERSISTENCE` |
| 원문에서 정한 역할 | 프로젝트가 직접 관리하는 MariaDB 실행 환경과 영속 데이터 소유권을 정했습니다. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `e13b0357a21b` |

#### 원문이 확정한 범위

<!-- 원문 요약: Creates the custom Debian MariaDB image and foreground daemon lifecycle. -->
<!-- 원문 판단 근거: The image is required for the project-owned database service, yet it mainly establishes expected container packaging and ownership conventions. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `f8ec9621725c`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/mariadb/Dockerfile`의 `FROM / RUN / CMD`에서 이미지가 DB 실행 파일과 빈 데이터 경로만 제공하고 실제 데이터는 실행 시 경로가 소유하도록 준비합니다.
- `srcs/requirements/mariadb/Dockerfile`의 `mariadbd foreground command`에서 PID 1과 DB 데몬 lifecycle이 분리되지 않으며 컨테이너 종료가 데몬 종료로 이어집니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| f8ec9621725c | srcs/requirements/mariadb/Dockerfile | FROM / RUN / CMD | Debian 기반에 MariaDB server/클라이언트, CA, gosu를 설치하고 배포판의 초기 DB 내용을 제거한 뒤 런타임/data directory를 `mysql` 소유로 다시 만듭니다. | 이미지가 DB 실행 파일과 빈 데이터 경로만 제공하고 실제 데이터는 실행 시 경로가 소유하도록 준비합니다. |
| f8ec9621725c | srcs/requirements/mariadb/Dockerfile | mariadbd foreground 명령 | 최종 명령은 `mariadbd`를 `mysql` 사용자로 foreground에서 실행합니다. | PID 1과 DB 데몬 lifecycle이 분리되지 않으며 컨테이너 종료가 데몬 종료로 이어집니다. |

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 프로젝트가 직접 관리하는 MariaDB 실행 환경과 영속 데이터 소유권을 정했습니다. |
| 핵심 입력 / 출력 / 상태 | 이미지는 실행 파일과 초기 디렉터리 권한을 소유하지만, `/var/lib/mysql`에 실제 데이터가 생긴 뒤의 내용과 수명은 런타임 또는 후속 볼륨 마운트가 소유합니다. |
| 변경된 directive / helper / 명령 | `srcs/requirements/mariadb/Dockerfile`의 `FROM / RUN / CMD`; `srcs/requirements/mariadb/Dockerfile`의 `mariadbd foreground command` |
| immediate 실패 또는 경계 | 이 SHA에는 최초 실행 database/account 초기화가 없습니다. 빈 경로에서 데몬이 어떤 스키마와 계정을 가져야 하는지는 아직 정의되지 않았습니다. |
| 다음 커밋에 넘긴 한계 | 초기 스키마, root hardening, 애플리케이션 database/user, interruption recovery, named-볼륨 persistence는 보장하지 않습니다. `e13b0357a21b`가 같은 이미지에 idempotent 최초 실행 엔트리포인트를 추가하고, `75590dedfb3a`가 data directory를 명명된 볼륨에 연결합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 초기 스키마, root hardening, 애플리케이션 database/user, interruption recovery, named-볼륨 persistence는 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `e13b0357a21b`가 같은 이미지에 idempotent 최초 실행 엔트리포인트를 추가하고, `75590dedfb3a`가 data directory를 명명된 볼륨에 연결합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 프로젝트가 직접 빌드한 MariaDB가 foreground 프로세스로 실행되고, 데이터 경로가 `mysql` 사용자에게 쓰기 가능하다는 점을 보장합니다.

### 2. `e13b0357a21b` — feat(mariadb): DB와 애플리케이션 계정 초기화

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `BOOTSTRAP`, `SECRETS`, `CORE` |
| 원문에서 정한 역할 | 최초의 재실행 가능한 DB·계정 초기화를 추가했습니다. |
| 이전 커밋 | `f8ec9621725c` |
| 다음 커밋 | `d764d066167b` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds first-run MariaDB initialization, account hardening, database creation, and idempotent volume reuse. -->
<!-- 원문 판단 근거: This is the first substantial state-creation mechanism and establishes least-privilege database ownership, even though the later staged bootstrap redesign supersedes parts of it. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `e13b0357a21b`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `secret input / identifier validation`에서 초기화 SQL에 들어가는 입력을 엔트리포인트가 먼저 정규화합니다.
- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `first-run branch / temporary server`에서 외부 TCP 요청을 받기 전에 초기화 SQL을 실행합니다.
- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `hardening SQL / final exec`에서 초기화 프로세스와 장기 실행 데몬 사이의 handoff가 명시됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| e13b0357a21b | srcs/requirements/mariadb/tools/docker-entrypoint.sh | 비밀값 입력 / identifier validation | 직접 값과 `_FILE` 입력의 동시 사용을 거부하고, 필수 값 누락·잘못된 DB/user 식별자·SQL literal을 검사하거나 escape합니다. | 초기화 SQL에 들어가는 입력을 엔트리포인트가 먼저 정규화합니다. |
| e13b0357a21b | srcs/requirements/mariadb/tools/docker-entrypoint.sh | 최초 실행 브랜치 / 임시 객체 server | system database directory가 없을 때만 `mariadb-install-db`를 실행하고, networking을 끈 소켓-only 임시 서버를 띄워 준비 상태를 기다립니다. | 외부 TCP 요청을 받기 전에 초기화 SQL을 실행합니다. |
| e13b0357a21b | srcs/requirements/mariadb/tools/docker-entrypoint.sh | hardening SQL / final exec | anonymous user·remote root·테스트 DB를 제거하고 애플리케이션 DB/user/grant를 만든 뒤 임시 서버를 종료하고 최종 `mariadbd`로 `exec`합니다. | 초기화 프로세스와 장기 실행 데몬 사이의 handoff가 명시됩니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff e13b0357a21b^ e13b0357a21b -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff f8ec9621725c e13b0357a21b -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | `f8ec9621725c`의 이미지는 빈 DB 경로와 데몬만 제공했으므로 WordPress가 인증할 database/account가 없었습니다. |
| 선택한 경계 / 결정 | 엔트리포인트가 빈 볼륨과 이미 채워진 볼륨을 구분하고, 빈 경우에만 소켓-only 임시 서버를 이용해 DB와 계정을 생성하도록 했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `secret input / identifier validation`; `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `first-run branch / temporary server`; `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `hardening SQL / final exec` |
| 상태·소유권·수명 변화 | first run에서는 엔트리포인트가 데이터 디렉터리와 임시 객체 server를 관리합니다. 재시작에서는 system directory 존재를 근거로 초기화를 건너뛰고 기존 DB 상태를 권위자로 취급합니다. |
| 주요 실패 브랜치 | 임시 서버 start/준비 상태/SQL/shutdown 중 오류가 나면 엔트리포인트는 실패하지만, 데이터 디렉터리에 이미 쓰인 일부 상태가 남을 수 있습니다. 단순한 시스템 디렉터리 존재 검사는 이 부분 상태를 완료로 오인할 수 있습니다. |
| 이 커밋의 보장 | 정상 종료된 첫 실행 뒤에는 hardened root account, 애플리케이션 database/user/grant가 존재하고 같은 볼륨 재사용 시 중복 생성하지 않습니다. |
| 한계와 다음 관련 커밋 | SIGKILL 등 정리 trap을 실행하지 못하는 중단 뒤의 수렴, completion marker, 준비 영역 반영은 보장하지 않습니다. `dc9601f5e670`이 partial 영속 상태 문제를 준비 영역 directory와 verified marker로 교정합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: SIGKILL 등 정리 trap을 실행하지 못하는 중단 뒤의 수렴, completion marker, 준비 영역 반영은 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `dc9601f5e670`이 partial 영속 상태 문제를 준비 영역 directory와 verified marker로 교정합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 정상 종료된 첫 실행 뒤에는 hardened root account, 애플리케이션 database/user/grant가 존재하고 같은 볼륨 재사용 시 중복 생성하지 않습니다.

### 3. `d764d066167b` — feat(wordpress): 사이트와 사용자 계정 초기화

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `BOOTSTRAP`, `PERSISTENCE`, `CORE` |
| 원문에서 정한 역할 | WordPress 파일·사이트·사용자 상태를 재실행 시 수렴하도록 추가했습니다. |
| 이전 커밋 | `e13b0357a21b` |
| 다음 커밋 | `99c03f54399a` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds idempotent WordPress core, configuration, site, and user initialization. -->
<!-- 원문 판단 근거: It introduces the application half of persistent first-run convergence and separates filesystem, database, and account idempotency boundaries, although later commits make the process interruption-safe. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `d764d066167b`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `password/metadata validation`에서 WordPress 초기화 입력의 실패를 PHP-FPM start 전에 차단합니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `DB wait / core-config-site-user branches`에서 파일 존재, config 존재, DB의 site 설치, user 존재를 하나의 조건으로 뭉개지 않습니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `URL update / chown / exec php-fpm`에서 엔트리포인트 완료 뒤 장기 실행 책임이 PHP-FPM으로 넘어갑니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| d764d066167b | srcs/requirements/wordpress/tools/docker-entrypoint.sh | password/메타데이터 validation | DB, admin, author password의 직접 값과 `_FILE` 입력을 해석하고 URL, title, email, user name 등 필수 메타데이터를 검증합니다. | WordPress 초기화 입력의 실패를 PHP-FPM start 전에 차단합니다. |
| d764d066167b | srcs/requirements/wordpress/tools/docker-entrypoint.sh | DB wait / core-config-site-user branches | MariaDB 인증이 될 때까지 bounded retry한 뒤 core files, `wp-config.php`, site 설치, author 존재 여부를 각각 별도 query로 판단하고 필요한 단계만 실행합니다. | 파일 존재, config 존재, DB의 site 설치, user 존재를 하나의 조건으로 뭉개지 않습니다. |
| d764d066167b | srcs/requirements/wordpress/tools/docker-entrypoint.sh | URL update / chown / exec php-fpm | canonical HTTPS home/site URL을 맞추고 소유권을 정규화한 뒤 PHP-FPM을 foreground로 `exec`합니다. | 엔트리포인트 완료 뒤 장기 실행 책임이 PHP-FPM으로 넘어갑니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff d764d066167b^ d764d066167b -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff e13b0357a21b d764d066167b -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | MariaDB 계정은 생겼지만 WordPress core, 비공개 config, site rows, admin/author 계정이 존재하지 않았습니다. |
| 선택한 경계 / 결정 | filesystem, configuration, site, user를 서로 다른 idempotency 조건으로 검사하고 누락된 항목만 생성하는 애플리케이션 초기화를 도입했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `password/metadata validation`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `DB wait / core-config-site-user branches`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `URL update / chown / exec php-fpm` |
| 상태·소유권·수명 변화 | WordPress 엔트리포인트가 writable web tree와 DB 애플리케이션 상태를 생성합니다. PHP-FPM은 초기화가 끝난 뒤 serving만 담당합니다. |
| 주요 실패 브랜치 | DB wait timeout, WP-CLI 실패, 중간 파일 생성 뒤 종료가 발생하면 일부 filesystem/DB 상태가 남습니다. 각 단계의 존재 검사는 정상 완료를 완전히 증명하지 않습니다. |
| 이 커밋의 보장 | 정상 첫 실행 뒤 core/config/site/admin/author가 준비되고 재시작은 이미 존재하는 항목을 재생성하지 않습니다. |
| 한계와 다음 관련 커밋 | 중단 시점별 영속 completion, config 분리 볼륨, 비밀값-free 장기 실행 서비스는 아직 보장하지 않습니다. `99c03f54399a`가 이 PHP-FPM에 외부 HTTPS 요청을 연결하고, `dc9601f5e670`이 초기화 lifecycle을 one-off convergence로 바꿉니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 중단 시점별 영속 completion, config 분리 볼륨, 비밀값-free 장기 실행 서비스는 아직 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `99c03f54399a`가 이 PHP-FPM에 외부 HTTPS 요청을 연결하고, `dc9601f5e670`이 초기화 lifecycle을 one-off convergence로 바꿉니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 정상 첫 실행 뒤 core/config/site/admin/author가 준비되고 재시작은 이미 존재하는 항목을 재생성하지 않습니다.

### 4. `99c03f54399a` — feat(nginx): PHP 요청을 WordPress로 전달

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `STACK`, `INTEGRATION`, `CORE` |
| 원문에서 정한 역할 | HTTPS 요청을 FastCGI로 넘기는 경계를 정의했습니다. |
| 이전 커밋 | `d764d066167b` |
| 다음 커밋 | `a8b9f693c614` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds TLS policy, static delivery, WordPress front-controller routing, FastCGI forwarding, and a health endpoint. -->
<!-- 원문 판단 근거: This defines the actual external request path and the Nginx-to-PHP responsibility boundary, making it significant to understanding how the stack serves WordPress. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `99c03f54399a`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/nginx/conf/nginx.conf`의 `server / listen / ssl_protocols`에서 외부 transport와 TLS termination을 Nginx가 독점합니다.
- `srcs/requirements/nginx/conf/nginx.conf`의 `root / try_files / location ~ \.php$`에서 Nginx는 PHP를 실행하지 않고 서비스 DNS를 통해 PHP-FPM에 요청을 넘깁니다.
- `srcs/requirements/nginx/conf/nginx.conf`의 `/healthz / nginx foreground`에서 transport 프로세스의 liveness를 독립적으로 검사할 수 있습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 99c03f54399a | srcs/requirements/nginx/conf/nginx.conf | server / listen / ssl_protocols | IPv4·IPv6 HTTPS listener와 TLS 1.2/1.3, certificate/key path를 정의합니다. | 외부 transport와 TLS termination을 Nginx가 독점합니다. |
| 99c03f54399a | srcs/requirements/nginx/conf/nginx.conf | root / try_files / location ~ \.php$ | shared WordPress document root에서 정적 file을 찾고, 없으면 front controller로 보내며 PHP 요청은 `wordpress:9000` FastCGI로 전달합니다. | Nginx는 PHP를 실행하지 않고 서비스 DNS를 통해 PHP-FPM에 요청을 넘깁니다. |
| 99c03f54399a | srcs/requirements/nginx/conf/nginx.conf | /healthz / nginx foreground | 애플리케이션과 분리된 간단한 헬스 상태 endpoint가 있고 Nginx는 데몬-off foreground로 실행됩니다. | transport 프로세스의 liveness를 독립적으로 검사할 수 있습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 99c03f54399a^ 99c03f54399a -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff d764d066167b 99c03f54399a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | WordPress는 PHP-FPM으로 실행 가능했지만 host-facing TLS listener와 정적/front-controller/FastCGI routing이 없었습니다. |
| 선택한 경계 / 결정 | Nginx가 HTTPS·정적 delivery·routing만 소유하고, PHP execution은 `wordpress:9000`에 위임하도록 경계를 고정했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/requirements/nginx/conf/nginx.conf`의 `server / listen / ssl_protocols`; `srcs/requirements/nginx/conf/nginx.conf`의 `root / try_files / location ~ \.php$`; `srcs/requirements/nginx/conf/nginx.conf`의 `/healthz / nginx foreground` |
| 상태·소유권·수명 변화 | certificate와 listener는 Nginx 이미지/런타임이 소유합니다. WordPress web files는 공유 경로에서 읽히지만 애플리케이션 write responsibility는 WordPress에 남습니다. |
| 주요 실패 브랜치 | FastCGI 서비스가 준비되지 않았거나 파일 path가 일치하지 않으면 5xx가 발생합니다. 이 SHA 자체는 startup dependency나 shared 볼륨을 아직 결합하지 않습니다. |
| 이 커밋의 보장 | 외부 HTTPS 요청이 정적 또는 WordPress front controller로 분기되고 PHP는 별도 서비스에서 실행된다는 책임 분리를 보장합니다. |
| 한계와 다음 관련 커밋 | Compose DNS, mounted shared files, 헬스 상태-gated startup, 영속 DB/data는 보장하지 않습니다. `a8b9f693c614`이 세 서비스를 하나의 topology로 묶고 `75590dedfb3a`가 마운트와 헬스 상태 gate를 완성합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: Compose DNS, mounted shared files, 헬스 상태-gated startup, 영속 DB/data는 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `a8b9f693c614`이 세 서비스를 하나의 topology로 묶고 `75590dedfb3a`가 마운트와 헬스 상태 gate를 완성합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 외부 HTTPS 요청이 정적 또는 WordPress front controller로 분기되고 PHP는 별도 서비스에서 실행된다는 책임 분리를 보장합니다.

### 5. `a8b9f693c614` — feat(compose): 세 서비스 토폴로지 구성

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **S** |
| 태그 | `ARCH`, `STACK`, `CORE` |
| 원문에서 정한 역할 | 세 서비스의 역할을 핵심 Compose 토폴로지로 결합했습니다. |
| 이전 커밋 | `99c03f54399a` |
| 다음 커밋 | `75590dedfb3a` |

#### 원문이 확정한 범위

<!-- 원문 요약: Introduces the three custom services, shared network, sole HTTPS publication, and named persistent resources in Compose. -->
<!-- 원문 판단 근거: This is the foundational system topology. Removing it would leave a major gap in explaining the separation of transport, application execution, and durable state. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `a8b9f693c614`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `services: nginx, wordpress, mariadb`에서 개별 이미지를 하나의 프로젝트 namespace와 네트워크 안에 배치합니다.
- `srcs/docker-compose.yml`의 `network / volumes declarations`에서 영속 자원 이름의 소유자가 Compose 프로젝트로 이동합니다.
- `srcs/docker-compose.yml`의 `initial topology limitation`에서 토폴로지의 존재와 operational 준비 상태를 소급해 같은 것으로 취급하면 안 됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| a8b9f693c614 | srcs/docker-compose.yml | services: nginx, wordpress, mariadb | 세 custom 빌드 context와 서비스 DNS, restart policy, Nginx host port, 서비스 dependency를 한 Compose model에 선언합니다. | 개별 이미지를 하나의 프로젝트 namespace와 네트워크 안에 배치합니다. |
| a8b9f693c614 | srcs/docker-compose.yml | 네트워크 / volumes declarations | 공유 네트워크와 MariaDB·WordPress용 명명된 볼륨 이름을 선언합니다. | 영속 자원 이름의 소유자가 Compose 프로젝트로 이동합니다. |
| a8b9f693c614 | srcs/docker-compose.yml | initial topology limitation | 이 커밋에서는 볼륨 이름을 선언했지만 이후 커밋에서 추가되는 실제 마운트/환경/헬스 상태 연결은 아직 없습니다. | 토폴로지의 존재와 operational 준비 상태를 소급해 같은 것으로 취급하면 안 됩니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff a8b9f693c614^ a8b9f693c614 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 99c03f54399a a8b9f693c614 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 S 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 커밋 직전 상태 | 세 이미지와 설정은 존재했지만 빌드/run/네트워크/port/자원 naming을 한 번에 소유하는 system-level model이 없었습니다. |
| 해결하려던 문제 | 서비스가 생성되는 순서는 표현할 수 있지만 실제 데이터 마운트와 헬스 상태-based 준비 상태가 없어 “연결됐다”가 “준비됐다”를 뜻하지 않습니다. |
| 기존 설계가 충분하지 않았던 이유 | 세 이미지와 설정은 존재했지만 빌드/run/네트워크/port/자원 naming을 한 번에 소유하는 system-level model이 없었습니다. 서비스가 생성되는 순서는 표현할 수 있지만 실제 데이터 마운트와 헬스 상태-based 준비 상태가 없어 “연결됐다”가 “준비됐다”를 뜻하지 않습니다. |
| 핵심 결정 | Compose가 Nginx, WordPress, MariaDB의 빌드 context, 서비스 DNS, restart, dependency, host port, 자원 namespace를 통합하도록 했습니다. |
| 주요 caller → callee / producer → consumer | `srcs/docker-compose.yml`의 `services: nginx, wordpress, mariadb`; `srcs/docker-compose.yml`의 `network / volumes declarations`; `srcs/docker-compose.yml`의 `initial topology limitation` |
| authoritative 상태와 반영 경계 | Nginx는 host-facing 서비스, WordPress는 애플리케이션 서비스, MariaDB는 DB 서비스로 배치됩니다. Compose 프로젝트가 컨테이너/네트워크/볼륨 naming을 소유합니다. 세 서비스의 책임과 호출 방향을 하나의 배포 topology로 고정하고 Nginx만 host port를 갖는 핵심 architecture를 도입합니다. |
| 소유권 / 수명 / responsibility 변화 | Nginx는 host-facing 서비스, WordPress는 애플리케이션 서비스, MariaDB는 DB 서비스로 배치됩니다. Compose 프로젝트가 컨테이너/네트워크/볼륨 naming을 소유합니다. |
| 실패 상황과 recovery path | 서비스가 생성되는 순서는 표현할 수 있지만 실제 데이터 마운트와 헬스 상태-based 준비 상태가 없어 “연결됐다”가 “준비됐다”를 뜻하지 않습니다. |
| 이 커밋이 보장하는 것 | 세 서비스의 책임과 호출 방향을 하나의 배포 topology로 고정하고 Nginx만 host port를 갖는 핵심 architecture를 도입합니다. |
| 아직 보장하지 않는 것 | 이 SHA만으로 영속 마운트, 애플리케이션 환경, 헬스 상태 gate, restart 뒤 데이터 보존을 보장하지 않습니다. |
| 후속 fix / 테스트와 연결 | `75590dedfb3a`가 선언된 명명된 볼륨을 실제 경로에 마운트하고 서비스-specific 헬스 상태를 dependency 조건으로 연결합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 이 SHA만으로 영속 마운트, 애플리케이션 환경, 헬스 상태 gate, restart 뒤 데이터 보존을 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `75590dedfb3a`가 선언된 명명된 볼륨을 실제 경로에 마운트하고 서비스-specific 헬스 상태를 dependency 조건으로 연결합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 세 서비스의 책임과 호출 방향을 하나의 배포 topology로 고정하고 Nginx만 host port를 갖는 핵심 architecture를 도입합니다.

### 6. `75590dedfb3a` — feat(compose): 준비 상태에 따라 영속 서비스 연결

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `PERSISTENCE`, `INTEGRATION`, `OPERATIONS` |
| 원문에서 정한 역할 | 명명된 볼륨, 헬스 체크, 준비 상태 기반 의존성을 연결했습니다. |
| 이전 커밋 | `a8b9f693c614` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Mounts durable data, adds service health checks, and gates startup on dependency health. -->
<!-- 원문 판단 근거: It turns the service list into a stateful, readiness-aware stack and establishes important persistence and lifecycle integration across all three containers. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `75590dedfb3a`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `volumes mounts`에서 컨테이너 filesystem이 아니라 명명된 볼륨이 authoritative 상태가 됩니다.
- `srcs/docker-compose.yml`의 `healthcheck blocks`에서 단순 컨테이너-created 상태와 프로세스/애플리케이션 준비 상태를 구분합니다.
- `srcs/docker-compose.yml`의 `depends_on.condition: service_healthy`에서 요청 path의 consumer가 producer 준비 전에 시작되는 race를 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 75590dedfb3a | srcs/docker-compose.yml | volumes mounts | MariaDB data directory, WordPress writable web tree를 명명된 볼륨에 마운트하고 Nginx는 WordPress document를 읽는 쪽으로 연결합니다. | 컨테이너 filesystem이 아니라 명명된 볼륨이 authoritative 상태가 됩니다. |
| 75590dedfb3a | srcs/docker-compose.yml | healthcheck blocks | MariaDB, WordPress/PHP-FPM, Nginx에 각 서비스-specific 명령과 interval/retry/start-period를 설정합니다. | 단순 컨테이너-created 상태와 프로세스/애플리케이션 준비 상태를 구분합니다. |
| 75590dedfb3a | srcs/docker-compose.yml | depends_on.condition: service_healthy | WordPress는 healthy MariaDB 뒤에, Nginx는 healthy WordPress 뒤에 시작하도록 gate를 둡니다. | 요청 path의 consumer가 producer 준비 전에 시작되는 race를 줄입니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 75590dedfb3a^ 75590dedfb3a -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff a8b9f693c614 75590dedfb3a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | `a8b9f693c614`은 topology와 자원 names만 만들었고 실제 상태 마운트와 준비 상태 semantics가 불완전했습니다. |
| 선택한 경계 / 결정 | 명명된 볼륨 마운트와 헬스 상태-gated dependency를 추가해 topology를 stateful 애플리케이션 system으로 바꿨습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/docker-compose.yml`의 `volumes mounts`; `srcs/docker-compose.yml`의 `healthcheck blocks`; `srcs/docker-compose.yml`의 `depends_on.condition: service_healthy` |
| 상태·소유권·수명 변화 | MariaDB 볼륨은 relational 상태, WordPress 볼륨은 애플리케이션 files를 소유합니다. 컨테이너는 교체 가능하고 헬스 상태 명령이 다음 서비스 start 허용 여부를 결정합니다. |
| 주요 실패 브랜치 | 헬스 상태 명령이 현재 프로세스에 응답한다는 것은 최초 실행 initialization이 interruption-safe하다는 뜻이 아닙니다. 당시 헬스 상태는 후속 completion marker 수준까지 강하지 않습니다. |
| 이 커밋의 보장 | 서비스별 상태가 명명된 볼륨에 남고, dependency는 컨테이너 creation이 아니라 헬스 상태 success를 기다립니다. |
| 한계와 다음 관련 커밋 | abrupt 초기화 interruption 수렴, 볼륨 identity의 실제 유지, end-to-end data path는 후속 테스트 없이는 증명하지 않습니다. `fb1a689cf969`이 restart/recreate 뒤 볼륨 identity와 값 보존을 검증하고 `dc9601f5e670`이 헬스 상태에 영속 marker를 결합합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: abrupt 초기화 interruption 수렴, 볼륨 identity의 실제 유지, end-to-end data path는 후속 테스트 없이는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `fb1a689cf969`이 restart/recreate 뒤 볼륨 identity와 값 보존을 검증하고 `dc9601f5e670`이 헬스 상태에 영속 marker를 결합합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 서비스별 상태가 명명된 볼륨에 남고, dependency는 컨테이너 creation이 아니라 헬스 상태 success를 기다립니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 호스트 포트를 게시하는 서비스는 Nginx뿐입니다. | a8b9f693c614 | 75590dedfb3a | 8c9b5b9adef2 | Compose의 `ports`는 Nginx 서비스에만 있고 런타임 e2e는 해당 HTTPS endpoint를 사용합니다. |
| MariaDB 영속 상태는 이미지 계층이 아니라 mounted data 볼륨이 권위자입니다. | f8ec9621725c | a8b9f693c614, 75590dedfb3a | fb1a689cf969 | Dockerfile은 빈 data path만 준비하고 Compose가 명명된 볼륨을 마운트하며 persistence scenario가 동일 볼륨 ID와 row를 재검사합니다. |
| WordPress는 애플리케이션 상태를 쓸 수 있고 Nginx는 공유 문서를 읽는 경계에 놓입니다. | d764d066167b | 75590dedfb3a | 8c9b5b9adef2, fb1a689cf969 | WordPress 엔트리포인트가 files/site/users를 만들고 Nginx는 동일 document root에서 정적/FastCGI routing만 수행합니다. |
| 준비 상태는 단순 컨테이너 creation이 아니라 서비스-specific 헬스 상태로 판단합니다. | 75590dedfb3a | dc9601f5e670에서 marker까지 강화 | 2bf6d3f11337 | Compose healthchecks와 `condition: service_healthy`, 이후 marker+프로세스 probe가 start gate를 구성합니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: `a8b9f693c614` 전에는 서비스별 이미지가 있었지만 host-facing 서비스, shared 네트워크, named-볼륨 마운트가 하나의 시스템으로 결합되지 않았고, `75590dedfb3a` 전에는 컨테이너 creation과 서비스 준비 상태가 구분되지 않았습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: Compose의 sole port 반영, 서비스 DNS, read/write 마운트 mode, 서비스-specific 헬스 체크와 `service_healthy` dependency가 불변식을 관측 가능한 설정으로 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `8c9b5b9adef2`의 전체 요청/data path와 `fb1a689cf969`의 restart/recreate persistence가 후속 회귀를 검출합니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| 개별 컨테이너만 존재하고 시스템 경계가 없음 | a8b9f693c614가 세 서비스 topology와 유일한 host-facing Nginx를 확정 | 8c9b5b9adef2가 HTTPS→WordPress→MariaDB data path를 검증 | root cause는 buildable 이미지와 integrated system을 같은 것으로 본 데 있습니다. |
| 엔트리포인트의 existence check가 abrupt interruption 뒤 부분 상태를 완료로 오인 | dc9601f5e670가 준비 영역, verified marker, one-off 초기화로 교정 | 2bf6d3f11337가 영속 stage마다 SIGKILL 후 재실행 수렴을 검증 | graceful 정리에 의존하지 않고 영속 반영 order로 완료를 판정합니다. |
| 컨테이너 생성 순서를 준비 상태로 오인 | 75590dedfb3a가 서비스 헬스 상태 gate를 추가 | 후속 런타임 harness가 live 헬스 상태와 integrated 요청 path를 검사 | dependency는 프로세스/애플리케이션 probe가 성공해야 해제됩니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Nginx | 독립 이미지/프로세스 | 유일한 host-facing TLS 및 정적/FastCGI routing 책임 | 99c03f54399a 설정과 a8b9f693c614/75590dedfb3a Compose port | PHP 실행이나 DB 상태를 소유하지 않습니다. |
| WordPress | 독립 PHP-FPM 이미지와 초기 엔트리포인트 | 애플리케이션 execution과 writable 애플리케이션 상태 | d764d066167b 엔트리포인트, PHP-FPM pool, WordPress 볼륨 | site/files/users를 쓰고 FastCGI 요청을 처리합니다. |
| MariaDB | 독립 DB 이미지 | 관계형 영속 상태와 DB/account 소유 | e13b0357a21b 초기화, 75590dedfb3a data 마운트 | 애플리케이션 query에 필요한 DB와 grant를 유지합니다. |
| Compose | 서비스별 수동 실행 | 네트워크, 서비스 DNS, port, 볼륨 namespace, 헬스 상태 dependency 통합 소유 | a8b9f693c614 및 75590dedfb3a의 `docker-compose.yml` | 컨테이너 교체와 자원 naming을 프로젝트 단위로 관리합니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: The thread progresses from individually runnable 컨테이너 to one stateful application system. The decisive step is not the existence of three images but the Compose responsibility boundary: Nginx owns external transport, WordPress owns application execution, and MariaDB owns durable relational 상태. Health-gated dependencies and mounted volumes then make the topology operationally meaningful rather than merely connected. -->
- 최종 authoritative 상태와 소유자: MariaDB 명명된 볼륨이 relational 상태, WordPress 명명된 볼륨이 애플리케이션 files를 소유하며 Nginx는 상태 소유자가 아닙니다.
- 정상 실행의 entry point와 완료 조건: Compose 또는 후속 `start_stack.py`가 서비스를 시작하고, MariaDB→WordPress→Nginx 헬스 상태 gate가 모두 성공하면 정상 완료입니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: 이 개발 흐름의 초기 엔트리포인트만으로는 abrupt interruption 수렴이 충분하지 않으며 개발 흐름 2의 marker/준비 영역 초기화가 재시도 조건을 정의합니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: 개발 흐름 2의 초기화, 개발 흐름 3의 런타임/persistence 테스트, 개발 흐름 4~6의 management transaction이 사용할 기본 topology와 상태 소유권을 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: 백업의 atomicity, restore 되돌리기, credential compensation, supply-chain identity는 이 개발 흐름만으로 증명하지 않습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | 호스트 HTTPS 요청 | 99c03f54399a `nginx.conf` listener | Nginx가 TLS를 종료하고 server block을 선택합니다. | listener/certificate 오류면 Nginx 헬스 상태가 실패하고 downstream 요청이 열리지 않습니다. |
| 2 | 정적/front controller 분기 | 99c03f54399a `try_files` | 실제 file은 정적으로, 나머지는 `index.php`로 이동합니다. | 잘못된 root/path는 404 또는 FastCGI 오류로 드러납니다. |
| 3 | FastCGI 전달 | 99c03f54399a `fastcgi_pass wordpress:9000` | 서비스 DNS가 WordPress PHP-FPM에 요청을 전달합니다. | WordPress 헬스 상태가 실패하면 75590dedfb3a의 gate가 Nginx start를 막습니다. |
| 4 | WordPress DB 접근 | d764d066167b config 생성과 MariaDB 서비스 name | 애플리케이션 credential로 MariaDB 서비스에 연결합니다. | DB auth/준비 상태 실패는 bounded wait 또는 WP-CLI 실패로 종료합니다. |
| 5 | 상태 저장 | 75590dedfb3a 볼륨 mounts | DB rows와 WordPress files가 명명된 볼륨에 남습니다. | 컨테이너 교체는 볼륨을 제거하지 않는 한 상태 소유자를 바꾸지 않습니다. |
| 6 | 준비 상태 gate | 75590dedfb3a healthcheck/depends_on | 각 producer 헬스 상태가 성공해야 다음 consumer가 시작됩니다. | timeout/retry 소진은 startup 실패이며 후속 초기화 design이 부분 상태를 판정합니다. |

### 학습자의 최종 설명

> 이미지 세 개를 만든 것만으로 계층형 시스템은 완성되지 않았습니다. MariaDB 이미지가 영속 DB 경로를 준비하고 최초 실행 account를 만들며, WordPress가 filesystem·site·users를 수렴시키고, Nginx가 TLS와 FastCGI routing만 맡은 뒤에야 책임이 나뉩니다. `a8b9f693c614`은 이 책임을 하나의 Compose namespace에 모았지만 마운트와 준비 상태는 부족했습니다. `75590dedfb3a`에서 명명된 볼륨과 서비스-specific 헬스 상태 gate가 추가되어 컨테이너 교체 가능한 실행 단위와 권위 있는 영속 상태가 분리됐습니다. 다만 초기 엔트리포인트는 abrupt interruption 뒤 부분 상태를 완전하게 판정하지 못하므로, 개발 흐름 2의 staged one-off 초기화가 이 architecture의 실제 재시도 안전성을 완성합니다.

## 학습 완료 자가 점검

- [x] 세 이미지의 존재와 세 계층 architecture의 성립을 같은 것으로 설명하지 않았습니까?
- [x] Nginx가 PHP를 실행한다고 잘못 설명하지 않았습니까?
- [x] WordPress data 볼륨과 MariaDB data 볼륨의 상태 종류를 구분했습니까?
- [x] 초기 헬스 체크가 completion marker까지 포함한다고 소급해서 쓰지 않았습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 01-readiness-aware-three-tier-stack.md =====

===== BEGIN FILE: 02-convergent-one-off-bootstrap.md =====
# 개발 흐름 2 — 실행 중 비밀 파일 마운트에서 수렴형 일회성 초기화까지

## 개발 흐름의 목표

Compose 비밀값 마운트 기반의 초기 모델이 host-side 비밀값 validation, per-프로젝트 잠금, one-off 초기화, completion marker를 이용한 수렴형 초기화로 바뀌는 핵심 lifecycle 교정을 추적합니다.

### 원문에서 정한 의의

초기 `_FILE`·Compose 비밀 파일 방식은 일반 환경 변수 노출을 줄였지만 장기 실행 서비스 시작 과정에 자격 증명을 계속 연결했습니다. 이후 호스트가 프로젝트 잠금 안에서 비밀값을 검증하고 단기 초기화 컨테이너에 필요한 값만 전달하며, 검증된 영속 상태가 완성된 뒤 장기 서비스를 시작하도록 바꿉니다. SIGKILL 테스트는 통제된 오류가 아닌 프로세스 강제 종료 뒤에도 재실행이 수렴하는지 확인합니다.

<details>
<summary>영문 원문</summary>

> The earlier `_FILE` and Compose-secret model reduced direct environment exposure but still attached credential material to service startup. The later architecture resolves secrets on the host while holding the project lock, sends only required values to short-lived bootstrap containers, and lets long-running services start from verified persistent state. The final SIGKILL scenario is important because it validates the design's intended convergence after process death, not only after controlled errors.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- 초기 `_FILE` 모델은 환경 노출을 줄였지만 어떤 steady-상태 노출과 partial-상태 위험을 남겼습니까?
- host 비밀값 path가 신뢰 가능한 입력이 되기 위해 어떤 파일 디스크립터/stat/permission 검사가 필요합니까?
- 왜 management 잠금의 granularity가 Compose 프로젝트 name입니까?
- MariaDB 준비 영역 반영과 WordPress completion marker는 어떤 순서 보장을 만듭니까?
- 프로세스 준비 상태와 영속 initialization completion은 헬스 체크에서 어떻게 결합됩니까?
- SIGKILL 테스트는 graceful trap 기반 테스트와 무엇을 다르게 증명합니까?

## 완료 기준

- 기존 런타임 비밀값 마운트 구조와 최종 초기화-only 비밀값 전달 구조를 비교했습니다.
- `project_operation_lock`의 비공개 path, 소유권, no-follow, non-blocking flock 계약을 실제 코드로 확인했습니다.
- MariaDB와 WordPress의 준비 영역/marker/publish 순서를 각 SHA의 엔트리포인트와 orchestrator로 복원했습니다.
- 정적 규약과 SIGKILL 런타임 회귀가 각각 증명하는 범위를 분리했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `916391b9f8db` | feat(secrets): 비밀번호를 비밀 파일에서 로드 | **B** | `SECRETS`<br>`RISK` | 비밀번호를 일반 환경 변수에서 Compose 비밀 파일로 옮겼습니다. |
| 2 | `486ffb5c65aa` | refactor(secrets): 비밀 파일 로딩 경계 공통화 | **A** | `SECRETS`<br>`RISK`<br>`ARCH` | 호스트 비밀 파일 경로 해석과 안전한 읽기를 한곳에 모았습니다. |
| 3 | `e77c6f151b07` | refactor(runtime): 프로젝트 관리 작업 잠금 공통화 | **A** | `RECOVERY`<br>`OPERATIONS`<br>`RISK` | 프로젝트별 관리 작업 직렬화를 확립했습니다. |
| 4 | `dc9601f5e670` | fix(init): 중단된 단계별 초기화를 수렴 | **S** | `ARCH`<br>`BOOTSTRAP`<br>`RECOVERY` | 장기 실행 서비스의 비밀 파일 마운트와 일회성 초기화를 단계별 일회성 초기화로 교체했습니다. |
| 5 | `3beebbfc4723` | test(init): 단계별 초기화 계약 검사 | **B** | `TEST`<br>`BOOTSTRAP` | 완료 표식과 단계별 복구 규약을 소스 검사로 추가했습니다. |
| 6 | `2bf6d3f11337` | test(init): 안정 단계별 초기화 중단 복구 검증 | **A** | `TEST`<br>`BOOTSTRAP`<br>`RECOVERY` | 영속 단계마다 초기화 컨테이너를 강제 종료하고 재실행 수렴을 검증했습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `916391b9f8db` — feat(secrets): 비밀번호를 비밀 파일에서 로드

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `SECRETS`, `RISK` |
| 원문에서 정한 역할 | 비밀번호를 일반 환경 변수에서 Compose 비밀 파일로 옮겼습니다. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `486ffb5c65aa` |

#### 원문이 확정한 범위

<!-- 원문 요약: Replaces password environment values with Compose secret files and `_FILE` inputs. -->
<!-- 원문 판단 근거: This is a meaningful intermediate security improvement, but the later one-off bootstrap architecture removes steady-state secret mounts and becomes the durable project boundary. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `916391b9f8db`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `.env.example`의 `*_PASSWORD_FILE variables`에서 operator가 값 대신 file 원본을 구성합니다.
- `srcs/docker-compose.yml`의 `secrets / service attachments`에서 credential 재질이 일반 환경 value가 아니라 file 마운트로 전달됩니다.
- `srcs/docker-compose.yml`의 `*_PASSWORD_FILE=/run/secrets/... / healthcheck`에서 장기 실행 서비스도 steady 상태에서 비밀값 마운트를 계속 보유합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 916391b9f8db | .env.example | *_PASSWORD_FILE variables | 공개 예제 환경에서 password literal을 제거하고 host 비밀값 file path를 받도록 바꿉니다. | operator가 값 대신 file 원본을 구성합니다. |
| 916391b9f8db | srcs/docker-compose.yml | secrets / 서비스 attachments | 네 비밀값 원본을 선언하고 MariaDB와 WordPress가 필요한 subset을 `/run/secrets`로 마운트합니다. | credential 재질이 일반 환경 value가 아니라 file 마운트로 전달됩니다. |
| 916391b9f8db | srcs/docker-compose.yml | *_PASSWORD_FILE=/run/secrets/... / healthcheck | 엔트리포인트 `_FILE` 변수와 MariaDB 헬스 상태 명령이 mounted root 비밀값을 읽습니다. | 장기 실행 서비스도 steady 상태에서 비밀값 마운트를 계속 보유합니다. |

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 비밀번호를 일반 환경 변수에서 Compose 비밀 파일로 옮겼습니다. |
| 핵심 입력 / 출력 / 상태 | host 소스 파일은 Compose가 마운트하고 장기 실행 MariaDB/WordPress 컨테이너가 실행 내내 `/run/secrets`를 읽을 수 있습니다. |
| 변경된 directive / helper / 명령 | `.env.example`의 `*_PASSWORD_FILE variables`; `srcs/docker-compose.yml`의 `secrets / service attachments`; `srcs/docker-compose.yml`의 `*_PASSWORD_FILE=/run/secrets/... / healthcheck` |
| immediate 실패 또는 경계 | 환경 노출은 줄지만 런타임 컨테이너 compromise나 진단 명령이 mounted 비밀값에 접근할 수 있고, 초기화와 serving lifecycle이 여전히 결합됩니다. |
| 다음 커밋에 넘긴 한계 | host file의 소유자/mode/symlink 안전성, 프로젝트 작업 serialization, 비밀값-free steady-상태 컨테이너는 보장하지 않습니다. `486ffb5c65aa`가 host 비밀값 read를 hardened shared 경계로 만들고 `dc9601f5e670`이 런타임 마운트 자체를 제거합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: host file의 소유자/mode/symlink 안전성, 프로젝트 작업 serialization, 비밀값-free steady-상태 컨테이너는 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `486ffb5c65aa`가 host 비밀값 read를 hardened shared 경계로 만들고 `dc9601f5e670`이 런타임 마운트 자체를 제거합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: password literal이 일반 환경/config에 직접 놓이지 않고 `_FILE` 규약으로 소비된다는 중간 보장을 제공합니다.

### 2. `486ffb5c65aa` — refactor(secrets): 비밀 파일 로딩 경계 공통화

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `SECRETS`, `RISK`, `ARCH` |
| 원문에서 정한 역할 | 호스트 비밀 파일 경로 해석과 안전한 읽기를 한곳에 모았습니다. |
| 이전 커밋 | `916391b9f8db` |
| 다음 커밋 | `e77c6f151b07` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds hardened secret-file reading, rendered secret-path resolution, environment extraction, and stdin payload construction. -->
<!-- 원문 판단 근거: This centralizes a critical trust boundary used by startup, backup, restore, rotation, and diagnostics, but it supports rather than alone defines the project-wide lifecycle architecture. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `486ffb5c65aa`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_runtime.py`의 `secret_source_paths`에서 비밀값 path 해석을 startup/backup/restore/rotation/diagnostics가 공유할 수 있게 합니다.
- `tools/stack_runtime.py`의 `read_private_secret`에서 pathname 검사 뒤 교체되는 TOCTOU window를 파일 디스크립터 기반 검사로 줄입니다.
- `tools/stack_runtime.py`의 `load_secret_values / secret_payload / service_environment`에서 credential 전달의 producer/consumer 형식이 공통화됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 486ffb5c65aa | tools/stack_runtime.py | secret_source_paths | rendered Compose의 `x-secret-files` 메타데이터에서 원본 path를 resolve하고 canonical path가 서로 겹치지 않는지 검사합니다. | 비밀값 path 해석을 startup/backup/restore/rotation/diagnostics가 공유할 수 있게 합니다. |
| 486ffb5c65aa | tools/stack_runtime.py | read_private_secret | `O_NOFOLLOW`로 열고 파일 디스크립터 `fstat`으로 regular file, single link, current 소유자, `0600`, parent directory 안전성을 검사한 뒤 bounded single-line value를 읽습니다. | pathname 검사 뒤 교체되는 TOCTOU window를 파일 디스크립터 기반 검사로 줄입니다. |
| 486ffb5c65aa | tools/stack_runtime.py | load_secret_values / secret_payload / service_environment | 검증된 네 값을 mapping으로 만들고 one-off 명령이 stdin으로 받을 전달값과 non-비밀값 환경을 분리합니다. | credential 전달의 producer/consumer 형식이 공통화됩니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 486ffb5c65aa^ 486ffb5c65aa -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 916391b9f8db 486ffb5c65aa -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | `916391b9f8db`은 file 마운트를 사용했지만 host 원본이 symlink, 잘못된 소유자/mode, hard link, multiline인지 공통으로 검증하지 않았습니다. |
| 선택한 경계 / 결정 | rendered Compose 메타데이터를 판단 기준으로 삼고, path resolution부터 파일 디스크립터 inspection과 content policy까지 한 module에 모았습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/stack_runtime.py`의 `secret_source_paths`; `tools/stack_runtime.py`의 `read_private_secret`; `tools/stack_runtime.py`의 `load_secret_values / secret_payload / service_environment` |
| 상태·소유권·수명 변화 | host management 프로세스가 비밀값 file을 읽고 즉시 memory mapping을 소유합니다. 이후 caller는 raw path를 다시 열지 않고 검증된 mapping/전달값을 사용합니다. |
| 주요 실패 브랜치 | unsafe type, 소유자, mode, link count, parent permission, duplicate canonical path, size/multiline/password-도형 위반은 변경 전에 실패합니다. |
| 이 커밋의 보장 | 동일한 hardened 비밀값-입력 규약을 여러 management 작업이 재사용할 수 있습니다. |
| 한계와 다음 관련 커밋 | 동시 작업이 같은 프로젝트 상태와 비밀값 generation을 바꾸는 race는 아직 막지 않습니다. `e77c6f151b07`가 프로젝트-scoped 잠금을 추가하고 `dc9601f5e670`의 startup이 잠금 안에서 이 helper를 호출합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 동시 작업이 같은 프로젝트 상태와 비밀값 generation을 바꾸는 race는 아직 막지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `e77c6f151b07`가 프로젝트-scoped 잠금을 추가하고 `dc9601f5e670`의 startup이 잠금 안에서 이 helper를 호출합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 동일한 hardened 비밀값-입력 규약을 여러 management 작업이 재사용할 수 있습니다.

### 3. `e77c6f151b07` — refactor(runtime): 프로젝트 관리 작업 잠금 공통화

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `RECOVERY`, `OPERATIONS`, `RISK` |
| 원문에서 정한 역할 | 프로젝트별 관리 작업 직렬화를 확립했습니다. |
| 이전 커밋 | `486ffb5c65aa` |
| 다음 커밋 | `dc9601f5e670` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds a per-user, per-project non-blocking advisory lock in a private fixed directory. -->
<!-- 원문 판단 근거: Serializing management operations is a critical concurrency invariant across later startup, backup, restore, and rotation flows, though the change is a focused mechanism rather than the whole project architecture. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `e77c6f151b07`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_runtime.py`의 `project_operation_lock`에서 TMPDIR 변경과 무관한 프로젝트 identity를 사용합니다.
- `tools/stack_runtime.py`의 `os.open / flock LOCK_EX|LOCK_NB`에서 같은 프로젝트의 동시 management 작업은 대기하지 않고 명시적으로 충돌 실패합니다.
- `tools/stack_runtime.py`의 `context-manager cleanup`에서 잠금 수명이 with-block의 management transaction과 일치합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| e77c6f151b07 | tools/stack_runtime.py | project_operation_lock | current user 전용 fixed 비공개 directory를 검사하고 프로젝트 name을 opaque filename으로 변환해 잠금 file을 엽니다. | TMPDIR 변경과 무관한 프로젝트 identity를 사용합니다. |
| e77c6f151b07 | tools/stack_runtime.py | os.open / flock LOCK_EX\|LOCK_NB | no-follow·소유자/mode/type 검사를 거친 파일 디스크립터에 non-blocking exclusive advisory 잠금을 잡습니다. | 같은 프로젝트의 동시 management 작업은 대기하지 않고 명시적으로 충돌 실패합니다. |
| e77c6f151b07 | tools/stack_runtime.py | context-manager 정리 | 예외 여부와 관계없이 flock release와 파일 디스크립터 close가 수행됩니다. | 잠금 수명이 with-block의 management transaction과 일치합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff e77c6f151b07^ e77c6f151b07 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 486ffb5c65aa e77c6f151b07 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 비밀값 read helper가 안전해도 두 startup/backup/restore/rotation이 같은 Compose 프로젝트를 동시에 변경하면 각자의 사전 조건이 무효화될 수 있었습니다. |
| 선택한 경계 / 결정 | 잠금 identity를 프로젝트 name에 맞추고 management 작업 전체를 non-blocking exclusive 잠금으로 감쌌습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/stack_runtime.py`의 `project_operation_lock`; `tools/stack_runtime.py`의 `os.open / flock LOCK_EX\|LOCK_NB`; `tools/stack_runtime.py`의 `context-manager cleanup` |
| 상태·소유권·수명 변화 | 잠금 파일 디스크립터를 보유한 host 프로세스가 transaction 동안 프로젝트 변경 권한을 소유합니다. 다른 프로젝트 name은 별도 잠금이므로 병렬 실행 가능합니다. |
| 주요 실패 브랜치 | 같은 프로젝트 잠금 contention은 즉시 domain error가 됩니다. 프로세스 death 시 OS가 파일 디스크립터를 닫아 잠금을 회수합니다. |
| 이 커밋의 보장 | 동일 프로젝트를 변경하는 cooperating management 코드가 겹치지 않는다는 concurrency 불변식을 제공합니다. |
| 한계와 다음 관련 커밋 | Docker 외부에서 잠금을 무시하는 수동 명령이나 non-cooperating 프로세스는 막지 못합니다. `dc9601f5e670`이 startup orchestration에 잠금을 적용하고 이후 backup/restore/rotation이 같은 mechanism을 공유합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: Docker 외부에서 잠금을 무시하는 수동 명령이나 non-cooperating 프로세스는 막지 못합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `dc9601f5e670`이 startup orchestration에 잠금을 적용하고 이후 backup/restore/rotation이 같은 mechanism을 공유합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 동일 프로젝트를 변경하는 cooperating management 코드가 겹치지 않는다는 concurrency 불변식을 제공합니다.

### 4. `dc9601f5e670` — fix(init): 중단된 단계별 초기화를 수렴

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **S** |
| 태그 | `ARCH`, `BOOTSTRAP`, `RECOVERY` |
| 원문에서 정한 역할 | 장기 실행 서비스의 비밀 파일 마운트와 일회성 초기화를 단계별 일회성 초기화로 교체했습니다. |
| 이전 커밋 | `e77c6f151b07` |
| 다음 커밋 | `3beebbfc4723` |

#### 원문이 확정한 범위

<!-- 원문 요약: Replaces in-container first-run setup with locked, staged one-off bootstrap orchestration, completion markers, and convergent restart behavior. -->
<!-- 원문 판단 근거: This is the decisive lifecycle redesign: it removes runtime secret mounts, separates configuration state, survives interrupted initialization, and determines how persistent services are safely brought to readiness. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `dc9601f5e670`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/start_stack.py`의 `run_action / staged startup`에서 startup 전체가 한 프로젝트 transaction으로 직렬화됩니다.
- `srcs/docker-compose.yml`의 `one-off bootstrap commands / runtime service blocks`에서 비밀값 수명이 단기 실행 초기화 프로세스로 제한됩니다.
- `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `staging data directory / marker / rename`에서 최종 DB path는 verified complete 상태만 보이도록 게시됩니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core/config/site/users stages / marker`에서 partial stage는 marker 부재로 다음 실행에서 다시 수렴합니다.
- `srcs/docker-compose.yml`의 `healthcheck marker + process probe`에서 영속 completion과 live 준비 상태가 동시에 충족돼야 healthy입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| dc9601f5e670 | tools/start_stack.py | run_action / staged startup | Compose path와 프로젝트를 검증하고 `project_operation_lock` 안에서 host 비밀값을 읽은 뒤 MariaDB 초기화→DB 서비스 헬스 상태→WordPress 초기화→애플리케이션/Nginx start 순서로 실행합니다. | startup 전체가 한 프로젝트 transaction으로 직렬화됩니다. |
| dc9601f5e670 | srcs/docker-compose.yml | one-off 초기화 commands / 런타임 서비스 blocks | credential은 초기화 컨테이너 stdin으로만 전달되고 장기 실행 서비스 block에서는 password 환경과 `/run/secrets` 마운트가 제거됩니다. | 비밀값 수명이 단기 실행 초기화 프로세스로 제한됩니다. |
| dc9601f5e670 | srcs/requirements/mariadb/tools/docker-entrypoint.sh | 준비 영역 data directory / marker / rename | 비공개 준비 영역에서 system tables와 accounts를 만들고 인증 검증·sync·completion marker를 끝낸 뒤 최종 data directory로 rename합니다. | 최종 DB path는 verified complete 상태만 보이도록 게시됩니다. |
| dc9601f5e670 | srcs/requirements/wordpress/tools/docker-entrypoint.sh | core/config/site/users stages / marker | core files, 비공개 config 볼륨, site, users와 password를 단계별로 수렴·검증하고 마지막에 marker를 atomic replace합니다. | partial stage는 marker 부재로 다음 실행에서 다시 수렴합니다. |
| dc9601f5e670 | srcs/docker-compose.yml | healthcheck marker + 프로세스 probe | MariaDB는 marker/소켓/PID, WordPress는 marker/FastCGI ping을 함께 요구합니다. | 영속 completion과 live 준비 상태가 동시에 충족돼야 healthy입니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff dc9601f5e670^ dc9601f5e670 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff e77c6f151b07 dc9601f5e670 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | directory/file existence를 정상 초기화 완료로 간주했습니다. |
| 실제 실패 또는 위험 | abrupt 프로세스 death가 정리 없이 partial 영속 상태를 남기며 다음 start가 이를 재사용할 수 있습니다. |
| root cause | initialization과 장기 실행 serving이 같은 엔트리포인트/수명에 결합되고 완료 반영 경계가 없었습니다. |
| 수정된 불변식 / 결정 | 단기 실행 one-off 초기화가 verified marker 또는 atomic data-directory rename을 게시한 뒤에만 런타임 서비스를 시작합니다. |
| 실제 수정 코드 | `tools/start_stack.py`의 `run_action / staged startup`; `srcs/docker-compose.yml`의 `one-off bootstrap commands / runtime service blocks`; `srcs/requirements/mariadb/tools/docker-entrypoint.sh`의 `staging data directory / marker / rename`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core/config/site/users stages / marker`; `srcs/docker-compose.yml`의 `healthcheck marker + process probe` |
| 변경된 ordering / 소유권 / lifecycle | host 코드가 비밀값 원본과 작업 order를 소유합니다. 초기화 컨테이너는 영속 볼륨을 한 번 수렴시키고 종료합니다. 장기 실행 서비스는 verified marker가 있는 상태를 serving만 합니다. |
| 이 fix가 보장하는 것 | 장기 서비스의 비밀값-free 경계, same-프로젝트 serialization, MariaDB atomic data 반영, WordPress verified marker, marker+프로세스 준비 상태를 보장합니다. |
| 아직 보장하지 않는 것 | filesystem/DB 자체의 하드웨어 crash durability나 외부 수동 변경까지 원자화하지는 않습니다. 실제 SIGKILL 수렴은 테스트 커밋이 별도로 증명해야 합니다. |
| 연결되는 회귀 테스트 | 정적 ordering 규약과 영속-stage SIGKILL 회귀가 이 corrected 불변식을 고정합니다. `3beebbfc4723`이 원본 규약을 고정하고 `2bf6d3f11337`이 영속 stage마다 초기화 프로세스를 SIGKILL해 재실행 수렴을 검증합니다. |

#### 중요도 S 상태 변화 기록

| 단계 | 학습자 기록 |
| --- | --- |
| correction 전 authoritative 상태 | 초기 엔트리포인트는 장기 실행 서비스가 비밀값을 마운트한 채 빈/존재 조건으로 즉석 초기화했습니다. SIGKILL 뒤 partial directory가 남으면 다음 start가 완료로 오인될 수 있었습니다. |
| partial / ambiguous 상태 종류 | abrupt 프로세스 death가 정리 없이 partial 영속 상태를 남기며 다음 start가 이를 재사용할 수 있습니다. |
| 반영 또는 커밋 경계 | host orchestrator, 프로젝트 잠금, stdin-only one-off 초기화, 준비 영역 반영, completion marker, 비공개 config 볼륨으로 startup lifecycle을 재설계했습니다. |
| 되돌리기 / compensation 진입 조건 | 어느 stage에서 종료돼도 최종 marker/rename 전 상태는 완료로 공개되지 않습니다. 재실행은 existing verified 부분을 검사하고 누락된 stage를 다시 수행합니다. one-off 실패는 장기 실행 서비스 start를 허용하지 않습니다. |
| recovery 중 보호되는 불변식 | host 코드가 비밀값 원본과 작업 order를 소유합니다. 초기화 컨테이너는 영속 볼륨을 한 번 수렴시키고 종료합니다. 장기 실행 서비스는 verified marker가 있는 상태를 serving만 합니다. |
| 성공 endpoint | 장기 서비스의 비밀값-free 경계, same-프로젝트 serialization, MariaDB atomic data 반영, WordPress verified marker, marker+프로세스 준비 상태를 보장합니다. |
| 실패 endpoint | filesystem/DB 자체의 하드웨어 crash durability나 외부 수동 변경까지 원자화하지는 않습니다. 실제 SIGKILL 수렴은 테스트 커밋이 별도로 증명해야 합니다. |
| 후속 회귀 근거 | `3beebbfc4723`이 원본 규약을 고정하고 `2bf6d3f11337`이 영속 stage마다 초기화 프로세스를 SIGKILL해 재실행 수렴을 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: filesystem/DB 자체의 하드웨어 crash durability나 외부 수동 변경까지 원자화하지는 않습니다. 실제 SIGKILL 수렴은 테스트 커밋이 별도로 증명해야 합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `3beebbfc4723`이 원본 규약을 고정하고 `2bf6d3f11337`이 영속 stage마다 초기화 프로세스를 SIGKILL해 재실행 수렴을 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 장기 서비스의 비밀값-free 경계, same-프로젝트 serialization, MariaDB atomic data 반영, WordPress verified marker, marker+프로세스 준비 상태를 보장합니다.

### 5. `3beebbfc4723` — test(init): 단계별 초기화 계약 검사

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `TEST`, `BOOTSTRAP` |
| 원문에서 정한 역할 | 완료 표식과 단계별 복구 규약을 소스 검사로 추가했습니다. |
| 이전 커밋 | `dc9601f5e670` |
| 다음 커밋 | `2bf6d3f11337` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds static assertions for staged MariaDB and WordPress bootstrap markers and recovery structure. -->
<!-- 원문 판단 근거: The checks protect the new design at a source-pattern level, but they do not yet prove real interruption recovery. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `3beebbfc4723`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `bootstrap source-order validation`에서 후속 refactor가 marker를 너무 일찍 게시하는 회귀를 정적으로 막습니다.
- `tests/validate_stack.py`의 `runtime secret boundary checks`에서 one-off 초기화-only 비밀값 경계를 원본 규약으로 고정합니다.
- `tests/validate_stack.py`의 `health marker patterns`에서 준비 상태 semantics가 단순 liveness로 약화되지 않게 합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3beebbfc4723 | tests/validate_stack.py | 초기화 원본-order validation | MariaDB 준비 영역/marker/rename과 WordPress files/config/site/users/password verification/marker의 상대적 원본 order를 검사합니다. | 후속 refactor가 marker를 너무 일찍 게시하는 회귀를 정적으로 막습니다. |
| 3beebbfc4723 | tests/validate_stack.py | 런타임 비밀값 경계 checks | 런타임 서비스 block의 `/run/secrets` 마운트와 password-bearing 환경을 거부합니다. | one-off 초기화-only 비밀값 경계를 원본 규약으로 고정합니다. |
| 3beebbfc4723 | tests/validate_stack.py | 헬스 상태 marker patterns | healthcheck가 completion marker와 live 소켓/FastCGI probe를 함께 요구하는지 검사합니다. | 준비 상태 semantics가 단순 liveness로 약화되지 않게 합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 3beebbfc4723^ 3beebbfc4723 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff dc9601f5e670 3beebbfc4723 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | completion marker는 모든 필수 영속 상태 검증 뒤에만 게시되고 런타임 서비스는 비밀값을 마운트하지 않습니다. |
| 재현하는 실패 / 경계 | 후속 원본 변경이 marker를 앞당기거나 `/run/secrets`를 런타임 서비스에 재도입하는 경계입니다. |
| 테스트 방식 | 정적 원본 규약과 Compose block pattern/order 검사 |
| 테스트 준비 코드와 실패 주입 | repository의 Dockerfile/엔트리포인트/Compose 원본 자체가 테스트 준비 코드이며 별도 런타임 실패 주입은 없습니다. |
| 실제 통과하는 실제 실행 경로 | `tests/validate_stack.py`가 원본 files를 읽어 marker·rename·헬스 상태·비밀값 pattern을 검사합니다. |
| 핵심 검사문 | 필수 pattern의 존재, 금지 pattern의 부재, 반영 order의 단조 증가를 확인합니다. |
| 이 테스트가 증명하는 것 | 해당 architecture가 원본에 표현되어 있고 명백한 순서 회귀가 없음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | shell control flow의 모든 브랜치, Docker 런타임 동작, fsync 효과, SIGKILL 수렴은 증명하지 않습니다. |
| 성격 | 원본 규약 회귀 테스트 |
| 막는 후속 회귀 | marker-before-verification, 런타임 비밀값 마운트, marker 없는 healthcheck 회귀를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: SIGKILL 뒤 실제 볼륨이 수렴하거나 Docker가 헬스 상태 gate를 적용한다는 런타임 사실은 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `2bf6d3f11337`이 동일 불변식을 live Docker와 SIGKILL로 보강합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: architecture를 구성하는 marker order, 비밀값-free 런타임, 헬스 상태 규약이 원본에 존재함을 빠르게 증명합니다.

### 6. `2bf6d3f11337` — test(init): 안정 단계별 초기화 중단 복구 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `BOOTSTRAP`, `RECOVERY` |
| 원문에서 정한 역할 | 영속 단계마다 초기화 컨테이너를 강제 종료하고 재실행 수렴을 검증했습니다. |
| 이전 커밋 | `3beebbfc4723` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Kills MariaDB and WordPress bootstrap containers at every durable stage, reruns startup, and verifies state, credentials, markers, and temporary-file cleanup. -->
<!-- 원문 판단 근거: This is unusually strong evidence for the staged-convergence invariant and demonstrates that the core initialization design survives abrupt process death rather than only graceful errors. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `2bf6d3f11337`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `verify_bootstrap / pause-ready protocol`에서 sleep 추측이 아니라 production 테스트 hook의 명시적 handoff를 사용합니다.
- `tests/runtime_stack.py`의 `docker kill --signal KILL`에서 graceful exception 정리가 아니라 abrupt 프로세스 death를 재현합니다.
- `tests/runtime_stack.py`의 `rerun start / state and boundary assertions`에서 모든 영속 stage가 재실행 수렴한다는 end-to-end 근거를 만듭니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2bf6d3f11337 | tests/runtime_stack.py | verify_bootstrap / pause-ready protocol | 각 영속 초기화 stage에 `--pause-after`와 비공개 ready file을 설정해 프로세스가 정확한 stage를 통과했음을 동기화합니다. | sleep 추측이 아니라 production 테스트 hook의 명시적 handoff를 사용합니다. |
| 2bf6d3f11337 | tests/runtime_stack.py | docker kill --signal KILL | ready marker 확인 직후 해당 one-off 초기화 컨테이너를 SIGKILL하고, shell trap이 실행되지 않는 상태를 만듭니다. | graceful exception 정리가 아니라 abrupt 프로세스 death를 재현합니다. |
| 2bf6d3f11337 | tests/runtime_stack.py | rerun start / 상태 and 경계 검사문 | 같은 프로젝트를 다시 시작해 DB/site/users/passwords/markers/헬스 상태를 확인하고 장기 실행 컨테이너에 비밀값 마운트/env가 없는지 재검사합니다. | 모든 영속 stage가 재실행 수렴한다는 end-to-end 근거를 만듭니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 2bf6d3f11337^ 2bf6d3f11337 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 3beebbfc4723 2bf6d3f11337 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | verified marker/rename 전 부분 상태는 완료로 취급되지 않으며 같은 프로젝트 startup은 다시 수렴합니다. |
| 재현하는 실패 / 경계 | MariaDB와 WordPress의 각 영속 stage 직후 one-off 초기화 프로세스가 SIGKILL되는 경계입니다. |
| 테스트 방식 | live 통합 + 결정적 pause-ready handshake + SIGKILL |
| 테스트 준비 코드와 실패 주입 | 고유 프로젝트/port/secrets를 만들고 production `start_stack.py`에 pause stage를 전달한 뒤 ready file에서 동기화해 대상 컨테이너를 KILL합니다. |
| 실제 통과하는 실제 실행 경로 | host orchestrator→one-off 초기화→영속 volumes→헬스 상태-gated 장기 실행 services 전체 경로를 통과합니다. |
| 핵심 검사문 | 재실행 성공, marker/헬스 상태, DB/site/users/passwords, 런타임 비밀값 마운트/env 부재, 프로젝트 정리를 확인합니다. |
| 이 테스트가 증명하는 것 | graceful handler 없이 프로세스가 죽어도 영속 stage별 재시도가 complete 상태로 수렴함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 하드웨어 crash consistency, uninstrumented 임의 지점, 외부 concurrent 변경은 증명하지 않습니다. |
| 성격 | 결정적 런타임 회귀 |
| 막는 후속 회귀 | partial directory/marker 조기 반영, SIGKILL 뒤 permanent broken 볼륨, steady-상태 비밀값 재도입을 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 전원 손실/스토리지 write cache, 임의의 모든 instruction, 다른 Docker/OS 조합을 포괄하지는 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: 개발 흐름 3 이후 런타임 harness의 기반이 되며 초기화 lifecycle의 핵심 회귀 근거입니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 각 영속 MariaDB/WordPress stage에서 abrupt death가 발생해도 재실행이 one complete stack과 비밀값-free 런타임으로 수렴함을 증명합니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 장기 실행 컨테이너는 host 비밀값 마운트나 password-bearing 환경을 유지하지 않습니다. | 916391b9f8db의 중간 단계 | dc9601f5e670에서 최종 경계 확립 | 2bf6d3f11337 | Compose 런타임 서비스 block과 live 컨테이너 inspect 모두 `/run/secrets`/credential env 부재를 요구합니다. |
| 동일 Compose 프로젝트를 변경하는 management 작업은 직렬화됩니다. | e77c6f151b07 | dc9601f5e670 startup 적용 | 후속 backup/rotation 런타임 테스트 | `project_operation_lock`은 프로젝트 identity별 non-blocking flock을 startup/management transaction 전체에 유지합니다. |
| completion marker는 필수 data/config/account/credential 검증 뒤에만 게시됩니다. | dc9601f5e670 | 3beebbfc4723 원본 규약 | 2bf6d3f11337 | MariaDB 준비 영역 rename과 WordPress marker replace가 verification 뒤에 있고 SIGKILL 재실행이 이를 실제로 확인합니다. |
| 서비스 준비 상태는 영속 marker와 live 프로세스 준비 상태를 함께 요구합니다. | dc9601f5e670 | 3beebbfc4723 | 2bf6d3f11337 | MariaDB marker+소켓+PID, WordPress marker+FastCGI ping이 헬스 상태 condition입니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: `916391b9f8db`의 런타임 비밀값 mounts와 최초 엔트리포인트 idempotency는 프로세스 death 뒤 partial 영속 상태를 완료 상태와 구분하지 못했습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: host 파일 디스크립터 validation, per-프로젝트 잠금, MariaDB 준비 영역 rename, WordPress 영속 marker, marker-aware 헬스 상태와 one-off 초기화 ordering이 수렴 조건을 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `3beebbfc4723` 정적 규약과 `2bf6d3f11337`의 영속-stage SIGKILL matrix가 원본 구조와 실제 재실행 수렴을 각각 보호합니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| steady-상태 서비스가 비밀값 마운트를 보유하고 ordinary 엔트리포인트가 초기화를 수행 | dc9601f5e670가 host-read/stdin/one-off 초기화로 교정 | 2bf6d3f11337가 런타임 비밀값 경계와 rerun convergence를 검증 | credential 수명과 상태 convergence를 serving lifecycle에서 분리했습니다. |
| partial 볼륨 existence가 completed 상태로 오인됨 | 준비 영역 directory + verified marker + atomic 반영 | 3beebbfc4723 정적 규약과 2bf6d3f11337 SIGKILL 회귀 | 완료 판정은 단순 경로 존재가 아니라 검증 뒤 반영입니다. |
| 동시 management 작업이 같은 자원 assumptions를 변경 | e77c6f151b07의 per-프로젝트 non-blocking 잠금 | 후속 cross-TMPDIR contention과 management 테스트 | 잠금 identity는 프로젝트 name이며 다른 프로젝트는 병렬 가능합니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Host management 코드 | Compose가 비밀값 file을 서비스에 마운트 | 비밀값 원본 검증, 프로젝트 잠금, 초기화 orchestration 소유 | 486ffb5c65aa/e77c6f151b07/dc9601f5e670 | credential과 작업 order를 컨테이너 밖에서 통제합니다. |
| One-off 초기화 컨테이너 | 장기 서비스 엔트리포인트와 초기화 결합 | stdin credential을 잠시 받아 영속 상태 convergence만 수행 | dc9601f5e670 Compose run/labels/stdin path | 종료 뒤 credential-bearing 프로세스가 남지 않습니다. |
| Long-running MariaDB/WordPress | startup 때 비밀값 path 접근 | verified 영속 상태를 열고 serving만 수행 | 런타임 mounts/env absence와 marker-gated 명령 | steady 상태에서 host credential file을 보유하지 않습니다. |
| Completion marker | directory/file existence의 암묵적 판단 | 검증이 끝난 영속 상태의 반영 경계 | write/fsync/rename/헬스 상태 근거 | marker 부재는 재수렴 필요를 뜻합니다. |
| WordPress configuration | 공개 web tree 안 config | 비공개 config 볼륨이 authoritative이고 web tree는 controlled symlink | dc9601f5e670 마운트/symlink/marker path | Nginx가 비공개 config 볼륨을 읽지 않습니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: The earlier `_FILE` and Compose-secret model reduced direct environment exposure but still attached credential material to service startup. The later architecture resolves secrets on the host while holding the project lock, sends only required values to short-lived 초기화 컨테이너, and lets long-running services start from verified persistent 상태. The final SIGKILL scenario is important because it validates the design's intended convergence after process death, not only after controlled errors. -->
- 최종 authoritative 상태와 소유자: MariaDB final data directory/marker와 WordPress data/config marker가 영속 authoritative 상태이며 host management 코드가 비밀값 원본과 프로젝트 잠금을 소유합니다.
- 정상 실행의 entry point와 완료 조건: `start_stack.py`가 잠금 안에서 비밀값을 읽고 DB 초기화, DB 헬스 상태, WordPress 초기화, 애플리케이션/Nginx start를 끝내며 모든 marker+프로세스 헬스 상태가 성공하면 완료입니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: 실패·SIGKILL 뒤 marker가 없는 상태는 다음 실행에서 다시 검사·수렴하며, complete marker가 없으면 장기 실행 consumer를 healthy로 열지 않습니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: backup, restore, rotation이 같은 프로젝트 잠금과 hardened 비밀값-입력 경계를 재사용할 수 있는 전제를 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: 하드웨어 crash durability, non-cooperating manual Docker 변경, 모든 가능한 instruction-level kill point를 단독으로 증명하지 않습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | 프로젝트 잠금 획득 | e77c6f151b07 `project_operation_lock` | 같은 프로젝트의 management 변경을 직렬화합니다. | contention이면 변경 없이 즉시 실패합니다. |
| 2 | 비밀값 path 해석/읽기 | 486ffb5c65aa `secret_source_paths`, `read_private_secret` | rendered 메타데이터와 파일 디스크립터 검사로 네 값을 memory에 올립니다. | unsafe path/mode/content면 초기화 전 실패합니다. |
| 3 | MariaDB one-off 초기화 | dc9601f5e670 `start_stack.py` + MariaDB 엔트리포인트 | stdin credential로 준비 영역 DB와 accounts를 만듭니다. | 중단 시 final directory/marker가 없으므로 다음 실행이 다시 수행합니다. |
| 4 | DB 상태 publish | dc9601f5e670 marker/fsync/rename | 검증된 준비 영역을 final path로 게시합니다. | 반영 전 실패는 incomplete 상태를 final로 보이지 않습니다. |
| 5 | WordPress 초기화 | dc9601f5e670 WordPress 엔트리포인트 | 비공개 config, core, site, users/passwords를 검증·수렴합니다. | 어느 stage 실패든 marker를 게시하지 않고 재실행 대상이 됩니다. |
| 6 | 런타임 서비스 start | dc9601f5e670 Compose 헬스 상태/start stages | marker+live probe 성공 뒤 WordPress와 Nginx를 엽니다. | 헬스 상태 timeout은 startup 실패이며 비밀값 마운트는 런타임에 없습니다. |
| 7 | SIGKILL 회귀 | 2bf6d3f11337 `verify_bootstrap` | 각 영속 stage kill 뒤 같은 프로젝트가 complete 상태로 수렴합니다. | 테스트 harness가 프로젝트-scoped teardown을 시도하며 실행 환경에서는 이번에 재실행하지 않았습니다. |

### 학습자의 최종 설명

> 초기 `_FILE` 방식은 password literal을 환경에서 제거했지만 장기 실행 컨테이너에 비밀값 마운트를 남기고 초기화와 serving을 한 엔트리포인트에 묶었습니다. `486ffb5c65aa`와 `e77c6f151b07`은 host 비밀값 trust 경계와 same-프로젝트 serialization을 만들었고, `dc9601f5e670`은 이 기반 위에서 startup을 단기 실행 one-off 초기화 transaction으로 바꿨습니다. MariaDB는 비공개 준비 영역을 검증한 뒤 final directory를 게시하고, WordPress는 config/site/users/password 검증 뒤 completion marker를 게시합니다. 장기 실행 services는 이 verified 상태만 열며 비밀값을 마운트하지 않습니다. 정적 규약은 원본 order를 고정하고, SIGKILL 테스트는 정리 trap 없이 죽은 실제 프로세스 뒤에도 같은 프로젝트가 수렴한다는 별도 런타임 근거를 제공합니다.

## 학습 완료 자가 점검

- [x] Compose 비밀값 자체를 최종 steady-상태 비밀값 경계라고 설명하지 않았습니까?
- [x] marker 생성 시점과 data directory 반영 시점을 실제 코드 순서로 확인했습니까?
- [x] SIGKILL 뒤 shell 정리 trap이 실행된다고 가정하지 않았습니까?
- [x] 같은 프로젝트만 직렬화되고 다른 프로젝트는 병렬 가능하다는 granularity를 설명했습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 02-convergent-one-off-bootstrap.md =====

===== BEGIN FILE: 03-isolated-runtime-and-persistence.md =====
# 개발 흐름 3 — 격리된 실행 검증과 영속 상태 확인

## 개발 흐름의 목표

고정 identity를 제거해 독립 프로젝트를 만들고, isolated Docker harness로 요청 path와 영속 볼륨 보장을 실제 런타임에서 검증하는 흐름을 추적합니다.

### 원문에서 정한 의의

프로젝트명·이미지·포트·URL을 매개변수화해 서로 독립된 테스트 스택을 만들고, 격리된 Docker 하네스가 자원과 비밀값을 직접 관리하도록 합니다. 종단 간 테스트는 HTTPS부터 MariaDB까지의 요청·데이터 경로를, 영속성 테스트는 컨테이너를 교체해도 권위 있는 볼륨 상태가 유지되는지를 각각 증명합니다.

<details>
<summary>영문 원문</summary>

> Parameterization made independent test projects possible; the harness then turned those parameters into controlled Docker resources and private credentials. End-to-end and persistence scenarios prove distinct properties: one shows that the integrated request/data path works, while the other shows that container replacement does not replace authoritative volume state.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- fixed 컨테이너/이미지/port/URL identity가 여러 테스트 프로젝트를 막는 방식은 무엇입니까?
- 테스트 harness가 developer default 프로젝트를 건드리지 않는다는 증거는 무엇입니까?
- 코드 수준 Compose validation과 live 컨테이너 inspection은 각각 무엇을 놓칠 수 있습니까?
- end-to-end 요청/data-path 테스트와 restart/recreate persistence 테스트가 증명하는 속성은 어떻게 다릅니까?
- port conflict recovery가 임의의 startup 실패를 숨기지 않도록 어떤 조건으로 제한됩니까?

## 완료 기준

- 프로젝트/이미지/port/URL parameter가 Compose 자원 naming과 WordPress canonical URL에 미치는 영향을 확인했습니다.
- harness의 비공개 env/비밀값 creation, timeout, diagnostics, teardown 경계를 코드로 추적했습니다.
- HTTPS → FastCGI → WordPress → MariaDB의 동일 데이터 round trip을 테스트 검사문으로 복원했습니다.
- restart와 컨테이너 recreation 뒤에도 같은 볼륨 set이 유지되는지 기록했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `9d75a34e290f` | feat(runtime): 프로젝트·이미지·포트·URL 격리 | **A** | `ARCH`<br>`STACK`<br>`OPERATIONS` | 고정된 프로젝트·이미지·포트·URL 식별자를 제거했습니다. |
| 2 | `2c436f574712` | test(bootstrap): 격리된 런타임 하네스 추가 | **A** | `TEST`<br>`ARCH`<br>`OPERATIONS` | 격리된 Docker 실행 테스트 도구와 비밀값 경계 검사를 만들었습니다. |
| 3 | `8c9b5b9adef2` | test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증 | **A** | `TEST`<br>`INTEGRATION`<br>`STACK` | HTTPS, FastCGI, WordPress, MariaDB 전체 데이터 경로를 검증했습니다. |
| 4 | `fb1a689cf969` | test(persistence): 재시작·재생성 뒤 상태 보존 검증 | **A** | `TEST`<br>`PERSISTENCE`<br>`RISK` | 재시작·재생성 뒤 DB·옵션·업로드·볼륨 식별 정보가 유지되는지 검증했습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `9d75a34e290f` — feat(runtime): 프로젝트·이미지·포트·URL 격리

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `ARCH`, `STACK`, `OPERATIONS` |
| 원문에서 정한 역할 | 고정된 프로젝트·이미지·포트·URL 식별자를 제거했습니다. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `2c436f574712` |

#### 원문이 확정한 범위

<!-- 원문 요약: Parameterizes project names, image tags, HTTPS binding, port, and canonical WordPress URL while removing fixed container names. -->
<!-- 원문 판단 근거: This enables multiple isolated stacks and makes later runtime testing and fresh-project restore possible; it is a significant deployment-boundary improvement. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `9d75a34e290f`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `container_name removal / image variables`에서 Compose 프로젝트 namespace가 컨테이너/자원 names를 소유하고 테스트별 이미지 tag가 충돌하지 않습니다.
- `srcs/docker-compose.yml`의 `HTTPS_BIND_ADDRESS / HTTPS_PORT`에서 여러 stack이 서로 다른 host port에서 동시에 실행될 수 있습니다.
- `.env.example / WordPress environment`의 `WORDPRESS_URL`에서 런타임 endpoint와 WordPress home/site URL이 같은 테스트 identity를 사용합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 9d75a34e290f | srcs/docker-compose.yml | container_name removal / 이미지 variables | 고정 `container_name`을 제거하고 `STACK_IMAGE_PREFIX`와 `STACK_IMAGE_TAG`로 local 이미지 identity를 parameterize합니다. | Compose 프로젝트 namespace가 컨테이너/자원 names를 소유하고 테스트별 이미지 tag가 충돌하지 않습니다. |
| 9d75a34e290f | srcs/docker-compose.yml | HTTPS_BIND_ADDRESS / HTTPS_PORT | host publish address와 port를 환경 parameter로 만들고 loopback/non-default port를 허용합니다. | 여러 stack이 서로 다른 host port에서 동시에 실행될 수 있습니다. |
| 9d75a34e290f | .env.example / WordPress 환경 | WORDPRESS_URL | canonical WordPress URL을 명시적으로 요구해 domain과 non-default HTTPS port를 site 상태에 반영합니다. | 런타임 endpoint와 WordPress home/site URL이 같은 테스트 identity를 사용합니다. |

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 고정 프로젝트/컨테이너/이미지/443 port/canonical URL은 두 테스트 stack이 같은 Docker names와 host 소켓을 차지하게 했습니다. |
| 선택한 경계 / 결정 | Compose 프로젝트 name과 이미지 앞부분/tag, bind address/port, canonical URL을 모두 외부 parameter로 바꾸고 fixed 컨테이너 name을 제거했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/docker-compose.yml`의 `container_name removal / image variables`; `srcs/docker-compose.yml`의 `HTTPS_BIND_ADDRESS / HTTPS_PORT`; `.env.example / WordPress environment`의 `WORDPRESS_URL` |
| 상태·소유권·수명 변화 | caller가 프로젝트 identity를 선택하고 Compose가 그 이름으로 컨테이너/networks/volumes를 namespace합니다. WordPress DB 상태에는 caller가 제공한 URL이 저장됩니다. |
| 주요 실패 브랜치 | 잘못 조합된 domain/port/URL은 stack이 뜨더라도 redirect와 테스트 요청이 어긋날 수 있습니다. parameterization 자체는 isolation을 실제로 사용했는지 증명하지 않습니다. |
| 이 커밋의 보장 | 독립된 프로젝트/이미지/port/URL 조합을 만들 수 있고 default stack과 테스트 stack이 이름을 공유하지 않게 합니다. |
| 한계와 다음 관련 커밋 | 실제 비밀값 isolation, port reservation race, 프로젝트-scoped teardown, 요청/data path correctness는 보장하지 않습니다. `2c436f574712`가 이 parameter를 사용해 비공개 isolated harness를 만들고 후속 e2e/persistence scenario가 런타임 보장을 검사합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 실제 비밀값 isolation, port reservation race, 프로젝트-scoped teardown, 요청/data path correctness는 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `2c436f574712`가 이 parameter를 사용해 비공개 isolated harness를 만들고 후속 e2e/persistence scenario가 런타임 보장을 검사합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 독립된 프로젝트/이미지/port/URL 조합을 만들 수 있고 default stack과 테스트 stack이 이름을 공유하지 않게 합니다.

### 2. `2c436f574712` — test(bootstrap): 격리된 런타임 하네스 추가

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `ARCH`, `OPERATIONS` |
| 원문에서 정한 역할 | 격리된 Docker 실행 테스트 도구와 비밀값 경계 검사를 만들었습니다. |
| 이전 커밋 | `9d75a34e290f` |
| 다음 커밋 | `8c9b5b9adef2` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds an isolated Docker runtime harness with private credentials, random project names, dynamic ports, cleanup, and secret-boundary inspection. -->
<!-- 원문 판단 근거: The harness becomes the foundation for the branch's later behavioral evidence and materially changes the project from source-validated configuration to reproducible runtime verification. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `2c436f574712`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `RuntimeStack.__init__ / _prepare_environment`에서 테스트 준비 코드의 credentials와 Docker identity가 developer defaults와 분리됩니다.
- `tests/runtime_stack.py`의 `reserve_port / run_compose / _run_start`에서 hang과 fixed-port collision을 테스트 프로세스의 명시적 실패로 바꿉니다.
- `tests/runtime_stack.py`의 `assert_runtime_secret_boundary / inspect_service`에서 원본 string만이 아니라 effective 컨테이너 configuration을 검사합니다.
- `tests/runtime_stack.py`의 `close / project-scoped down`에서 default 프로젝트나 unrelated Docker 자원을 teardown 대상으로 사용하지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2c436f574712 | tests/runtime_stack.py | RuntimeStack.__init__ / _prepare_environment | PID와 random token으로 unique 프로젝트/이미지 앞부분을 만들고 임시 객체 directory `0700`, 비밀값 files `0600`, 비공개 env file, loopback port를 준비합니다. | 테스트 준비 코드의 credentials와 Docker identity가 developer defaults와 분리됩니다. |
| 2c436f574712 | tests/runtime_stack.py | reserve_port / run_compose / _run_start | loopback 소켓으로 후보 port를 찾고 모든 subprocess/Compose 명령에 bounded timeout을 적용합니다. | hang과 fixed-port collision을 테스트 프로세스의 명시적 실패로 바꿉니다. |
| 2c436f574712 | tests/runtime_stack.py | assert_runtime_secret_boundary / inspect_service | live 컨테이너 inspect와 rendered Compose를 사용해 런타임 서비스의 password env와 `/run/secrets` 마운트가 없는지 확인합니다. | 원본 string만이 아니라 effective 컨테이너 configuration을 검사합니다. |
| 2c436f574712 | tests/runtime_stack.py | close / 프로젝트-scoped down | scenario가 만든 프로젝트 name으로 `down --volumes --remove-orphans`하고 비공개 임시 객체 files를 제거합니다. | default 프로젝트나 unrelated Docker 자원을 teardown 대상으로 사용하지 않습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 2c436f574712^ 2c436f574712 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 9d75a34e290f 2c436f574712 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 테스트 scenario는 developer/default stack과 자원 identity를 공유하지 않고 런타임 서비스는 초기화 비밀값을 보유하지 않습니다. |
| 재현하는 실패 / 경계 | fixed 프로젝트/이미지/port/credential 또는 원본/effective config 불일치 경계입니다. |
| 테스트 방식 | live Docker harness + rendered config + 컨테이너 inspect |
| 테스트 준비 코드와 실패 주입 | 비공개 temp directory에서 random 프로젝트/이미지/비밀값과 loopback port를 만들고 production staged start를 실행합니다. |
| 실제 통과하는 실제 실행 경로 | RuntimeStack 준비→`start_stack.py`→Compose services→inspect/헬스 상태→프로젝트-scoped teardown 경로입니다. |
| 핵심 검사문 | unique names/paths/modes, 명령 timeouts, 런타임 비밀값 env/마운트 부재를 확인합니다. |
| 이 테스트가 증명하는 것 | isolated 런타임 테스트 준비 코드와 비밀값-경계 inspection이 실제 Docker objects에 적용됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | WordPress content round trip, named-볼륨 persistence, all 정리 실패 modes는 증명하지 않습니다. |
| 성격 | 통합 harness foundation |
| 막는 후속 회귀 | default namespace 사용, world-readable 비밀값 테스트 준비 코드, timeout 없는 subprocess, 런타임 비밀값 재도입을 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 특정 애플리케이션 요청/data path나 컨테이너 recreation 뒤 상태 보존은 이 harness 도입만으로 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `8c9b5b9adef2`와 `fb1a689cf969`이 같은 harness에 서로 다른 런타임 properties를 추가합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 각 scenario가 고유 프로젝트와 credentials로 실제 stack을 만들고 effective 비밀값 경계를 검사할 수 있게 합니다.

### 3. `8c9b5b9adef2` — test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `INTEGRATION`, `STACK` |
| 원문에서 정한 역할 | HTTPS, FastCGI, WordPress, MariaDB 전체 데이터 경로를 검증했습니다. |
| 이전 커밋 | `2c436f574712` |
| 다음 커밋 | `fb1a689cf969` |

#### 원문이 확정한 범위

<!-- 원문 요약: Extends the harness to test HTTPS health, WordPress post creation and rendering, MariaDB persistence, port-conflict recovery, and legacy configuration migration. -->
<!-- 원문 판단 근거: It verifies the complete browser-to-database path and catches integration failures that static checks cannot, making it significant but not an architectural implementation commit. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `8c9b5b9adef2`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `verify_e2e / blocked-port fixture`에서 임의 startup error를 port retry로 숨기지 않습니다.
- `tests/runtime_stack.py`의 `WP-CLI post creation`에서 테스트 준비 코드가 다른 run의 data와 혼동되지 않습니다.
- `tests/runtime_stack.py`의 `HTTPS fetch / MariaDB query`에서 Nginx→FastCGI→WordPress→MariaDB의 동일 data round trip을 연결합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 8c9b5b9adef2 | tests/runtime_stack.py | verify_e2e / blocked-port 테스트 준비 코드 | 후보 port를 실제 listener로 점유한 상태에서 start를 시도해 genuine bind conflict만 분류하고 bounded 횟수 안에서 새 port로 재시도합니다. | 임의 startup error를 port retry로 숨기지 않습니다. |
| 8c9b5b9adef2 | tests/runtime_stack.py | WP-CLI post creation | unique token이 포함된 published post를 WordPress 애플리케이션 인터페이스로 생성하고 post ID/title/content를 기록합니다. | 테스트 준비 코드가 다른 run의 data와 혼동되지 않습니다. |
| 8c9b5b9adef2 | tests/runtime_stack.py | HTTPS fetch / MariaDB query | HTTPS response에서 token을 확인하고 MariaDB에서 동일 post ID의 row와 content를 query해 비교합니다. | Nginx→FastCGI→WordPress→MariaDB의 동일 data round trip을 연결합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 8c9b5b9adef2^ 8c9b5b9adef2 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 2c436f574712 8c9b5b9adef2 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 외부 HTTPS 요청이 Nginx와 FastCGI를 거쳐 WordPress에서 실행되고 결과가 MariaDB row와 일치합니다. |
| 재현하는 실패 / 경계 | host port가 실제로 점유된 경우의 bounded recovery와 integrated data-path 단절입니다. |
| 테스트 방식 | live end-to-end 통합 + uniquely identifiable 테스트 준비 코드 + DB differential 검사문 |
| 테스트 준비 코드와 실패 주입 | 점유 listener로 첫 port를 막고, 새 port에서 stack을 시작한 뒤 unique post를 WP-CLI로 생성합니다. |
| 실제 통과하는 실제 실행 경로 | HTTPS listener→Nginx routing→`wordpress:9000`→WordPress 변경/read→MariaDB query를 통과합니다. |
| 핵심 검사문 | port 변경, 헬스 상태, HTTPS body token, DB post ID/title/content 일치를 확인합니다. |
| 이 테스트가 증명하는 것 | 공개 response와 authoritative relational row가 동일 애플리케이션 작업을 반영함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | restart/recreate persistence, high concurrency, browser semantics 전체는 증명하지 않습니다. |
| 성격 | broad 통합 with bounded edge 회귀 |
| 막는 후속 회귀 | routing/서비스-name/URL mismatch, false port-error retry, DB write와 공개 response 분리를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 컨테이너 restart/recreation 뒤에도 data가 남는지, backup/restore를 거치는지는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `fb1a689cf969`이 같은 stack에서 restart와 recreation을 별도 persistence property로 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 실제 integrated 요청/data path와 canonical non-default URL/port가 하나의 테스트 프로젝트 안에서 일치함을 증명합니다.

### 4. `fb1a689cf969` — test(persistence): 재시작·재생성 뒤 상태 보존 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `PERSISTENCE`, `RISK` |
| 원문에서 정한 역할 | 재시작·재생성 뒤 DB·옵션·업로드·볼륨 식별 정보가 유지되는지 검증했습니다. |
| 이전 커밋 | `8c9b5b9adef2` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Verifies posts, options, uploads, and all three named volumes across container restart and recreation. -->
<!-- 원문 판단 근거: The test locks down a central durable-state invariant and distinguishes container lifecycle from volume lifecycle, providing strong evidence for a core project guarantee. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `fb1a689cf969`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `verify_persistence / project_volumes`에서 상태 값뿐 아니라 authoritative 볼륨 identity를 비교할 기준을 만듭니다.
- `tests/runtime_stack.py`의 `persistent fixtures`에서 서로 다른 persistence class를 한 scenario에서 검사합니다.
- `tests/runtime_stack.py`의 `restart then down/up recreation`에서 프로세스 restart와 컨테이너 replacement를 구분해 증명합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| fb1a689cf969 | tests/runtime_stack.py | verify_persistence / project_volumes | 초기 MariaDB/WordPress data/config 명명된 볼륨 set과 concrete names를 기록합니다. | 상태 값뿐 아니라 authoritative 볼륨 identity를 비교할 기준을 만듭니다. |
| fb1a689cf969 | tests/runtime_stack.py | 영속 테스트 준비 코드 | unique post, WordPress option, upload file을 각각 relational DB, 애플리케이션 option, filesystem 상태로 만듭니다. | 서로 다른 persistence class를 한 scenario에서 검사합니다. |
| fb1a689cf969 | tests/runtime_stack.py | restart then down/up recreation | 먼저 서비스 restart 후 값을 검사하고, 이어 `down`(볼륨 미삭제)과 `up`으로 컨테이너를 재생성한 뒤 exact 볼륨 set과 모든 값을 다시 확인합니다. | 프로세스 restart와 컨테이너 replacement를 구분해 증명합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff fb1a689cf969^ fb1a689cf969 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 8c9b5b9adef2 fb1a689cf969 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 컨테이너 lifecycle과 영속 볼륨 lifecycle은 분리되어 컨테이너 replacement가 authoritative 상태를 교체하지 않습니다. |
| 재현하는 실패 / 경계 | restart 또는 `down`/`up` recreation 뒤 새 empty 볼륨이나 누락된 애플리케이션/filesystem 상태를 받는 경계입니다. |
| 테스트 방식 | live persistence 통합 + identity/value comparison |
| 테스트 준비 코드와 실패 주입 | unique post, option, upload를 만들고 initial 볼륨 set을 기록한 뒤 restart와 컨테이너 recreation을 수행합니다. |
| 실제 통과하는 실제 실행 경로 | WordPress/MariaDB writes→named volumes→restart/recreate→WP-CLI/DB/filesystem reads를 통과합니다. |
| 핵심 검사문 | exact 볼륨 set, post row, option value, upload 체크섬/content를 전후 비교합니다. |
| 이 테스트가 증명하는 것 | 컨테이너 replacement와 프로세스 restart 뒤 동일 named-볼륨 상태가 유지됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | explicit 볼륨 deletion, host 실패, backup/restore correctness는 증명하지 않습니다. |
| 성격 | 결정적 persistence 회귀 |
| 막는 후속 회귀 | anonymous/new 볼륨 마운트, down 시 볼륨 삭제, 상태 class 일부만 보존되는 회귀를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: host disk loss, explicit 볼륨 deletion, backup consistency, migration across Docker hosts는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: 개발 흐름 1에서 도입된 named-볼륨 소유권을 런타임 근거로 고정하고 backup/restore 개발 흐름의 기준 상태를 제공합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 프로세스 restart와 컨테이너 recreation이 authoritative named-볼륨 identity와 세 종류의 상태를 바꾸지 않음을 증명합니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 각 런타임 scenario는 고유 Compose 프로젝트, 이미지 앞부분, port, credential을 사용합니다. | 9d75a34e290f | 2c436f574712 | 8c9b5b9adef2, fb1a689cf969 | parameterized Compose와 RuntimeStack random identity/비공개 테스트 준비 코드가 실제 scenario 전 과정에 사용됩니다. |
| loopback HTTPS bind와 explicit WordPress URL은 non-default port에서도 일치합니다. | 9d75a34e290f | 2c436f574712 | 8c9b5b9adef2 | env file의 bind/port/URL이 공개 fetch와 WordPress canonical 상태에서 같은 값을 사용합니다. |
| 통합 요청 path 성공과 persistence는 별도 속성입니다. | 8c9b5b9adef2 | fb1a689cf969가 영속 근거 추가 | fb1a689cf969 | e2e는 순간 round trip, persistence는 restart/recreate 전후 볼륨 ID와 값 비교를 수행합니다. |
| 컨테이너 replacement는 authoritative 명명된 볼륨 identity를 바꾸지 않습니다. | 75590dedfb3a에서 구조 도입 | fb1a689cf969 | fb1a689cf969 | `project_volumes()`의 exact set과 post/option/upload 값을 recreation 전후 비교합니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: fixed 프로젝트/컨테이너/이미지/port/URL identity는 동시에 두 stack을 만들 수 없고 developer default 자원을 테스트 테스트 준비 코드와 분리하지 못했습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: parameterized Compose identity와 harness의 비공개 credentials, random 프로젝트 name, loopback port, bounded 명령, exact teardown이 isolation을 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `8c9b5b9adef2`가 전체 요청/data path를, `fb1a689cf969`가 restart와 `down`/`up` 뒤 동일 볼륨 identity와 values를 검증합니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| fixed names/ports/images로 테스트 stack 간 충돌 | 9d75a34e290f가 identity parameterization | 2c436f574712가 unique 비공개 harness로 사용 | 가능성을 선언한 것과 실제 isolation을 사용한 것을 분리했습니다. |
| healthy 프로세스만으로 애플리케이션 data path를 추정 | 8c9b5b9adef2가 unique post를 공개 HTTPS와 DB row로 연결 | 동일 scenario의 HTTPS/DB 검사문 | 서비스 헬스 상태와 business data round trip은 다른 근거입니다. |
| 컨테이너 recreation 뒤 새 empty 볼륨을 받아도 이전 e2e는 통과 | fb1a689cf969가 볼륨 names와 세 상태 class를 기록 | restart/down-up 뒤 exact identity/value 검사문 | 컨테이너와 authoritative 상태의 lifecycle을 분리합니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Compose 프로젝트 namespace | fixed names의 암묵적 공유 | scenario별 컨테이너/네트워크/볼륨 identity 소유 | 9d75a34e290f 프로젝트 parameter와 rendered names | default 프로젝트와 테스트 프로젝트가 이름을 공유하지 않습니다. |
| Harness 임시 객체 directory | developer 환경에 의존 | env, secrets, diagnostics, control files의 비공개 소유자 | 2c436f574712 mode/creation/정리 | host-side 테스트 재질은 0700/0600 범위에 남습니다. |
| Runtime data | 헬스 상태로 간접 추정 | post/option/upload와 볼륨 identity로 명시적 검증 | 8c9b5b9adef2/fb1a689cf969 검사문 | 관계형·애플리케이션·filesystem 상태를 분리해 확인합니다. |
| Port selection | fixed host port | loopback 후보와 genuine bind-conflict-only retry | 8c9b5b9adef2 listener/error classification | 다른 startup error는 재시도로 숨기지 않습니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: Parameterization made independent 테스트 projects possible; the harness then turned those parameters into controlled Docker 자원 and private credentials. End-to-end and persistence scenarios prove distinct properties: one shows that the integrated request/data path works, while the other shows that 컨테이너 replacement does not replace authoritative 볼륨 상태. -->
- 최종 authoritative 상태와 소유자: Compose 프로젝트가 Docker 자원 identity를, 비공개 RuntimeStack 테스트 준비 코드가 테스트 env/secrets/port/expected values를, named volumes가 영속 상태를 소유합니다.
- 정상 실행의 entry point와 완료 조건: scenario별 production staged start가 healthy해지고 해당 e2e 또는 persistence 검사문을 모두 통과하면 정상 완료입니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: start 실패는 genuine bind conflict일 때만 bounded port retry하며 다른 실패는 즉시 보고합니다. 종료 시 프로젝트-scoped teardown을 수행합니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: 후속 backup/restore/rotation/operations scenarios가 default stack과 충돌하지 않고 live 근거를 만들 수 있는 harness를 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: Docker가 없는 현재 환경에서는 실제 scenario result를 새로 증명하지 않았으며 코드에 표현된 테스트 mechanism만 확인했습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | 비공개 테스트 준비 코드 생성 | 2c436f574712 RuntimeStack preparation | 0700 temp dir, 0600 secrets, unique env를 만듭니다. | unsafe permission/file creation 실패면 Docker 변경 전 종료합니다. |
| 2 | 프로젝트/이미지/port 선택 | 9d75a34e290f parameters + 2c436f574712 random identity | Compose 자원과 host endpoint를 scenario별 분리합니다. | 8c9b5b9adef2가 genuine bind conflict만 새 port로 재시도합니다. |
| 3 | production startup | 2c436f574712 `_run_start` | `start_stack.py`를 bounded timeout으로 실행합니다. | timeout/start 실패는 StackError로 전파됩니다. |
| 4 | 런타임 비밀값/marker inspect | 2c436f574712 live inspect | effective 컨테이너가 초기화 경계를 유지하는지 확인합니다. | 비밀값 env/마운트나 marker/헬스 상태 mismatch면 실패합니다. |
| 5 | e2e round trip | 8c9b5b9adef2 `verify_e2e` | unique post가 HTTPS와 MariaDB에서 일치합니다. | 어느 layer든 token/ID가 다르면 path 실패입니다. |
| 6 | persistence round trip | fb1a689cf969 `verify_persistence` | restart/recreate 뒤 같은 volumes와 values를 확인합니다. | identity/value mismatch는 persistence 회귀입니다. |
| 7 | teardown | RuntimeStack.close | scenario 프로젝트와 비공개 files만 제거합니다. | 후속 개발 흐름 8에서는 정리 실패도 scenario result에 합칩니다. |

### 학습자의 최종 설명

> 고정 이름과 포트를 없앤 것만으로 isolation이 증명되지는 않습니다. `RuntimeStack`은 random 프로젝트/이미지 identity, loopback port, 비공개 env/secrets, bounded subprocess, 프로젝트-scoped teardown을 실제 테스트 준비 코드로 만들고 production startup을 호출합니다. e2e scenario는 unique post를 WordPress로 만든 뒤 공개 HTTPS와 MariaDB row에서 같은 값을 확인해 전체 요청/data path를 증명합니다. persistence scenario는 별도로 initial 볼륨 set과 DB option/upload 상태를 기록하고 프로세스 restart와 컨테이너 recreation 뒤 다시 비교합니다. 따라서 “현재 요청이 동작한다”와 “교체 뒤에도 권위 있는 상태가 남는다”는 서로 다른 근거로 유지됩니다.

## 학습 완료 자가 점검

- [x] e2e 테스트가 persistence까지 자동 증명한다고 합쳤습니까?
- [x] port retry가 모든 startup error에 적용된다고 잘못 기록하지 않았습니까?
- [x] 볼륨 이름의 동일성과 볼륨 안 값의 동일성을 모두 확인했습니까?
- [x] 테스트 harness가 default Compose namespace를 사용할 가능성을 코드로 배제했습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 03-isolated-runtime-and-persistence.md =====

===== BEGIN FILE: 04-atomic-backup-publication.md =====
# 개발 흐름 4 — 실패·취소 상황의 원자적 백업 반영

## 개발 흐름의 목표

MariaDB transactional dump와 WordPress filesystem archive를 하나의 신뢰 가능한 backup set으로 결합하고, 실패·signal에도 partial set을 게시하지 않으며 원본 stack을 복구하는 transaction을 추적합니다.

### 원문에서 정한 의의

데이터 수집과 백업 반영을 분리합니다. 비공개 임시 스트림 파일, 출력 경로 선점, 매니페스트, 디렉터리 교체를 사용해 완전한 백업 집합만 보이게 하고, 오류나 신호 취소 시 그럴듯한 미완성 백업을 남기거나 원본 스택을 저하시키지 않도록 합니다.

<details>
<summary>영문 원문</summary>

> The implementation deliberately separates data capture from publication. Private streaming files, an exact output reservation, a manifest, and directory replacement ensure that only a complete set becomes visible. Signal-aware recovery and negative runtime tests establish the equally important converse: cancelled or failed work must not leave a plausible backup or a degraded source stack.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- 두 artifact가 생성됐다는 사실만으로 하나의 일관된 backup이라고 할 수 없는 이유는 무엇입니까?
- 비공개 file creation, fsync, directory fsync, 체크섬은 각각 어떤 실패 window를 줄입니까?
- signal을 exception으로 전환하고 synchronized pause stage를 둔 이유는 무엇입니까?
- destination path reservation에서 pathname 비교가 아니라 device/inode identity가 필요한 이유는 무엇입니까?
- Nginx와 WordPress를 멈추되 MariaDB는 transactional dump를 위해 유지하는 ordering은 어디에 구현됩니까?
- published backup과 failed attempt의 observable end 상태는 각각 무엇입니까?

## 완료 기준

- data capture와 반영을 별도 단계로 나누고 각 durability 경계를 코드로 확인했습니다.
- backup directory가 정확히 DB dump, WordPress archive, manifest의 완전한 set으로만 보이는 과정을 추적했습니다.
- 실패와 SIGINT/SIGTERM이 동일 정리/recovery 경로로 수렴하는지 확인했습니다.
- negative 테스트가 final 출력, 임시 객체 sibling, ready marker, 잠금, 서비스 헬스 상태를 어떻게 검사하는지 기록했습니다.
- large 테스트 준비 코드와 signal-race 테스트가 small 정상 처리보다 추가로 증명하는 내용을 구분했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `fdd55605ba74` | feat(backup): 백업 무결성과 비공개 파일 I/O 정의 | **B** | `PERSISTENCE`<br>`OPERATIONS` | 비공개 출력, 동기화, 체크섬의 공용 도구를 정의했습니다. |
| 2 | `d26c885c5cd5` | feat(backup): 관리 작업 신호와 테스트 중단 경계 추가 | **A** | `RECOVERY`<br>`TEST`<br>`HARD` | 결정적인 시그널·실패 테스트 경계를 만들었습니다. |
| 3 | `3a0995ff0d4f` | feat(backup): 프로젝트별 백업 작업 잠금 적용 | **B** | `RECOVERY`<br>`OPERATIONS`<br>`PERSISTENCE` | Serialized backup with other operations on the same 프로젝트. |
| 4 | `b478b5243c5a` | feat(backup): DB 덤프와 WordPress 볼륨 수집 | **A** | `PERSISTENCE`<br>`CORE`<br>`INTEGRATION` | MariaDB 트랜잭션 덤프와 WordPress 볼륨 스트림을 수집했습니다. |
| 5 | `0540ff1b5a4b` | feat(backup): 백업 출력 경로를 안전하게 예약 | **A** | `PERSISTENCE`<br>`RISK`<br>`EDGE` | Reserved and identity-checked the destination path. |
| 6 | `6999190ffd34` | feat(backup): 백업 세트를 원자적으로 게시 | **S** | `PERSISTENCE`<br>`RECOVERY`<br>`HARD` | 체크섬이 포함된 완전한 백업 세트를 원자적으로 반영하고 서비스를 복구했습니다. |
| 7 | `b6920a0c918c` | test(backup): 게시 실패와 중단 정리 검증 | **A** | `TEST`<br>`RECOVERY`<br>`PERSISTENCE` | 실패 시 미반영, 정리, 서비스 복구, 공유 잠금 동작을 검증했습니다. |
| 8 | `030e7310c665` | test(backup): 자원 충돌과 시그널 경계 검증 | **A** | `TEST`<br>`PERSISTENCE`<br>`EDGE` | 시그널 경합, 대용량 데이터, 자원 충돌 경계까지 검증 범위를 넓혔습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `fdd55605ba74` — feat(backup): 백업 무결성과 비공개 파일 I/O 정의

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `PERSISTENCE`, `OPERATIONS` |
| 원문에서 정한 역할 | 비공개 출력, 동기화, 체크섬의 공용 도구를 정의했습니다. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `d26c885c5cd5` |

#### 원문이 확정한 범위

<!-- 원문 요약: Introduces SHA-256 helpers, directory synchronization, and exclusive private-file output primitives for backup work. -->
<!-- 원문 판단 근거: These are necessary low-level safety utilities, but they are supporting pieces whose project significance depends on later backup publication and restore orchestration. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `fdd55605ba74`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `sha256_stream`에서 manifest digest 계산이 caller의 후속 스트림 소비 위치를 망가뜨리지 않습니다.
- `tools/stack_backup.py`의 `private output helper`에서 artifact가 생성되는 첫 순간부터 다른 user에게 공개되지 않습니다.
- `tools/stack_backup.py`의 `flush/fsync / fsync_directory`에서 후속 rename 전에 file contents와 directory 메타데이터의 durability precondition을 만듭니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| fdd55605ba74 | tools/stack_backup.py | sha256_stream | 스트림의 현재 위치를 저장하고 처음부터 chunk로 SHA-256을 계산한 뒤 원래 위치로 되돌립니다. | manifest digest 계산이 caller의 후속 스트림 소비 위치를 망가뜨리지 않습니다. |
| fdd55605ba74 | tools/stack_backup.py | 비공개 출력 helper | `O_CREAT\|O_EXCL`과 mode `0600`으로 출력을 만들고 기존 path를 덮어쓰지 않습니다. | artifact가 생성되는 첫 순간부터 다른 user에게 공개되지 않습니다. |
| fdd55605ba74 | tools/stack_backup.py | flush/fsync / fsync_directory | file 스트림을 flush·fsync하고 parent directory 파일 디스크립터도 sync하며 OS 오류를 backup-domain error로 변환합니다. | 후속 rename 전에 file contents와 directory 메타데이터의 durability precondition을 만듭니다. |

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 비공개 출력, 동기화, 체크섬의 공용 도구를 정의했습니다. |
| 핵심 입력 / 출력 / 상태 | helper가 열린 스트림과 file 파일 디스크립터의 수명을 소유하며 caller는 성공한 영속 비공개 file만 다음 단계로 넘깁니다. |
| 변경된 directive / helper / 명령 | `tools/stack_backup.py`의 `sha256_stream`; `tools/stack_backup.py`의 `private output helper`; `tools/stack_backup.py`의 `flush/fsync / fsync_directory` |
| immediate 실패 또는 경계 | existing path, short write/flush/fsync, non-seekable digest 입력 등 지원하지 않는 조건은 명시적 실패가 됩니다. |
| 다음 커밋에 넘긴 한계 | DB dump와 WordPress archive의 cross-artifact consistency, atomic directory 반영, 서비스 recovery는 보장하지 않습니다. `b478b5243c5a`가 streaming capture에 사용하고 `6999190ffd34`가 manifest와 atomic directory 반영으로 완전한 set을 만듭니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: DB dump와 WordPress archive의 cross-artifact consistency, atomic directory 반영, 서비스 recovery는 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `b478b5243c5a`가 streaming capture에 사용하고 `6999190ffd34`가 manifest와 atomic directory 반영으로 완전한 set을 만듭니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 개별 artifact가 비공개하고 existing path를 덮어쓰지 않으며 체크섬과 sync를 계산할 수 있음을 보장합니다.

### 2. `d26c885c5cd5` — feat(backup): 관리 작업 신호와 테스트 중단 경계 추가

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `RECOVERY`, `TEST`, `HARD` |
| 원문에서 정한 역할 | 결정적인 시그널·실패 테스트 경계를 만들었습니다. |
| 이전 커밋 | `fdd55605ba74` |
| 다음 커밋 | `3a0995ff0d4f` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds controlled signal handling plus deterministic failure and pause stages for management-operation tests. -->
<!-- 원문 판단 근거: It creates a reliable way to exercise asynchronous cancellation through the same cleanup paths as ordinary errors, a significant failure-path engineering boundary used throughout backup and rotation testing. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `d26c885c5cd5`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `operation_signal_handlers`에서 signal과 ordinary exception을 같은 정리 path로 보냅니다.
- `tools/stack_backup.py`의 `pause/failure stage hook`에서 실패 timing을 sleep에 의존하지 않고 production control flow에 동기화합니다.
- `tools/stack_backup.py`의 `signal masking around ready publication`에서 테스트가 관측한 ready 상태와 실제 pause 상태가 어긋나는 window를 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| d26c885c5cd5 | tools/stack_backup.py | operation_signal_handlers | SIGINT/SIGTERM handler가 즉시 임의 지점에서 종료하는 대신 management-domain exception을 발생시키고 원래 handler를 finally에서 복원합니다. | signal과 ordinary exception을 같은 정리 path로 보냅니다. |
| d26c885c5cd5 | tools/stack_backup.py | pause/실패 stage hook | 명명된 stage에서 비공개 ready file을 게시한 뒤 테스트가 진행을 허용할 때까지 기다리거나 configured 실패를 발생시킵니다. | 실패 timing을 sleep에 의존하지 않고 production control flow에 동기화합니다. |
| d26c885c5cd5 | tools/stack_backup.py | signal masking around ready 반영 | ready marker 생성·sync와 handler transition의 작은 구간에서 signal race를 제어합니다. | 테스트가 관측한 ready 상태와 실제 pause 상태가 어긋나는 window를 줄입니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff d26c885c5cd5^ d26c885c5cd5 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff fdd55605ba74 d26c885c5cd5 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 비동기 signal은 정리 코드 어느 지점에서든 프로세스를 끝내 테스트 재현성과 자원 recovery를 불명확하게 만들 수 있었습니다. |
| 선택한 경계 / 결정 | operator signal을 normal 실패 처리로 변환하고, named stage/ready-file protocol로 결정적 interruption point를 만들었습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/stack_backup.py`의 `operation_signal_handlers`; `tools/stack_backup.py`의 `pause/failure stage hook`; `tools/stack_backup.py`의 `signal masking around ready publication` |
| 상태·소유권·수명 변화 | management 작업이 handler 설치부터 복원까지 signal 상태를 소유합니다. 테스트 ready marker는 해당 작업의 임시 객체 control 상태입니다. |
| 주요 실패 브랜치 | 첫 signal은 cancellation exception이 되고 finally가 서비스 recovery와 temp 정리를 수행합니다. ready-file publish 실패도 작업 실패입니다. |
| 이 커밋의 보장 | signal cancellation이 ordinary error와 같은 recovery path를 지나며 테스트가 정확한 stage에서 signal을 보낼 수 있습니다. |
| 한계와 다음 관련 커밋 | SIGKILL처럼 handler를 우회하는 종료나 하드웨어 장애는 처리하지 않습니다. `b6920a0c918c`와 `030e7310c665`가 반영 stage와 signal race에서 이 mechanism을 사용합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: SIGKILL처럼 handler를 우회하는 종료나 하드웨어 장애는 처리하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `b6920a0c918c`와 `030e7310c665`가 반영 stage와 signal race에서 이 mechanism을 사용합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: signal cancellation이 ordinary error와 같은 recovery path를 지나며 테스트가 정확한 stage에서 signal을 보낼 수 있습니다.

### 3. `3a0995ff0d4f` — feat(backup): 프로젝트별 백업 작업 잠금 적용

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `RECOVERY`, `OPERATIONS`, `PERSISTENCE` |
| 원문에서 정한 역할 | Serialized backup with other operations on the same 프로젝트. |
| 이전 커밋 | `d26c885c5cd5` |
| 다음 커밋 | `b478b5243c5a` |

#### 원문이 확정한 범위

<!-- 원문 요약: Applies the per-project advisory lock model to backup operations. -->
<!-- 원문 판단 근거: The lock is important, but this commit mainly extends an existing serialization decision to another management path rather than introducing a new project-wide mechanism. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `3a0995ff0d4f`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `project_operation_lock(project)`에서 사전 조건 검사와 변경 사이에 다른 cooperating 작업이 끼지 않습니다.
- `tools/stack_runtime.py`의 `shared lock identity`에서 작업 종류가 달라도 동일 프로젝트 변경은 하나의 serialization domain입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3a0995ff0d4f | tools/stack_backup.py | project_operation_lock(프로젝트) | backup entry path가 destination/비밀값/서비스 상태를 검사하고 writer를 멈추기 전부터 프로젝트 잠금을 보유합니다. | 사전 조건 검사와 변경 사이에 다른 cooperating 작업이 끼지 않습니다. |
| 3a0995ff0d4f | tools/stack_runtime.py | shared 잠금 identity | startup, backup, restore, rotation이 같은 프로젝트-name-derived 잠금을 공유합니다. | 작업 종류가 달라도 동일 프로젝트 변경은 하나의 serialization domain입니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 3a0995ff0d4f^ 3a0995ff0d4f -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff d26c885c5cd5 3a0995ff0d4f -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | Serialized backup with other operations on the same 프로젝트. |
| 핵심 입력 / 출력 / 상태 | backup host 프로세스가 capture/반영/recovery 동안 프로젝트 변경 권한을 소유합니다. |
| 변경된 directive / helper / 명령 | `tools/stack_backup.py`의 `project_operation_lock(project)`; `tools/stack_runtime.py`의 `shared lock identity` |
| immediate 실패 또는 경계 | 잠금 contention은 원본 stack이나 출력 path를 건드리기 전에 실패가 됩니다. |
| 다음 커밋에 넘긴 한계 | 수동 Docker/DB 명령처럼 잠금을 사용하지 않는 actor는 막지 않습니다. `6999190ffd34`가 잠금 안에서 전체 atomic backup을 구성하고 `b6920a0c918c`가 다른 TMPDIR에서도 same-프로젝트 contention을 확인합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 수동 Docker/DB 명령처럼 잠금을 사용하지 않는 actor는 막지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `6999190ffd34`가 잠금 안에서 전체 atomic backup을 구성하고 `b6920a0c918c`가 다른 TMPDIR에서도 same-프로젝트 contention을 확인합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: cooperating management 작업과 같은 프로젝트 backup이 겹치지 않음을 보장합니다.

### 4. `b478b5243c5a` — feat(backup): DB 덤프와 WordPress 볼륨 수집

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `PERSISTENCE`, `CORE`, `INTEGRATION` |
| 원문에서 정한 역할 | Captured transactional MariaDB and WordPress 볼륨 streams. |
| 이전 커밋 | `3a0995ff0d4f` |
| 다음 커밋 | `0540ff1b5a4b` |

#### 원문이 확정한 범위

<!-- 원문 요약: Streams a transactional MariaDB dump and a WordPress data/config archive into private files. -->
<!-- 원문 판단 근거: This implements the substantive data-capture path spanning database and filesystem state, a major component of backup functionality but not yet its atomic publication guarantee. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `b478b5243c5a`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `database dump helper`에서 DB server는 running 상태에서 consistent transactional view를 제공합니다.
- `tools/stack_backup.py`의 `WordPress archive helper`에서 host가 볼륨 구현 path를 직접 가정하지 않습니다.
- `tools/stack_backup.py`의 `archive content policy`에서 restore 시 공개 symlink와 비공개 config 원본을 중복/충돌시키지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| b478b5243c5a | tools/stack_backup.py | database dump helper | `mariadb-dump`를 `--single-transaction`과 스트림-oriented option으로 실행하고 stdout을 비공개 file로 직접 전달합니다. | DB server는 running 상태에서 consistent transactional view를 제공합니다. |
| b478b5243c5a | tools/stack_backup.py | WordPress archive helper | one-off tar 프로세스가 WordPress data/config 볼륨을 read-only로 마운트해 archive 스트림을 비공개 file로 보냅니다. | host가 볼륨 구현 path를 직접 가정하지 않습니다. |
| b478b5243c5a | tools/stack_backup.py | archive content policy | 공개 web tree의 config symlink를 archive에서 제외하고 실제 비공개 config 볼륨을 별도 root로 수집하며 archive member를 검증합니다. | restore 시 공개 symlink와 비공개 config 원본을 중복/충돌시키지 않습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff b478b5243c5a^ b478b5243c5a -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 3a0995ff0d4f b478b5243c5a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 비공개 file primitive는 있었지만 DB와 WordPress의 authoritative sources를 어떤 명령과 consistency mode로 capture할지 정의되지 않았습니다. |
| 선택한 경계 / 결정 | DB는 transactional dump 스트림, WordPress는 mounted 볼륨 tar 스트림으로 수집하고 애플리케이션 writers는 후속 orchestration에서 quiesce할 수 있게 했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/stack_backup.py`의 `database dump helper`; `tools/stack_backup.py`의 `WordPress archive helper`; `tools/stack_backup.py`의 `archive content policy` |
| 상태·소유권·수명 변화 | MariaDB server가 transaction 스냅샷을 소유하고 dump subprocess가 출력 스트림을 생산합니다. archive 컨테이너가 볼륨 read view를 소유하고 host helper가 files를 받습니다. |
| 주요 실패 브랜치 | subprocess timeout/nonzero, 스트림 write, archive member validation 실패는 artifact를 invalid로 처리합니다. 이 커밋만으로 두 스트림의 반영 timing은 묶이지 않습니다. |
| 이 커밋의 보장 | 큰 data도 host memory 전체에 올리지 않고 DB dump와 WordPress 볼륨 archive를 capture할 수 있습니다. |
| 한계와 다음 관련 커밋 | 두 artifact가 같은 애플리케이션 cut에 해당하거나 partial 출력이 final destination에 보이지 않는다는 보장은 아직 없습니다. `6999190ffd34`가 writers stop과 manifest/rename으로 두 스트림을 한 backup set으로 묶고 `030e7310c665`가 large 테스트 준비 코드를 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 두 artifact가 같은 애플리케이션 cut에 해당하거나 partial 출력이 final destination에 보이지 않는다는 보장은 아직 없습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `6999190ffd34`가 writers stop과 manifest/rename으로 두 스트림을 한 backup set으로 묶고 `030e7310c665`가 large 테스트 준비 코드를 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 큰 data도 host memory 전체에 올리지 않고 DB dump와 WordPress 볼륨 archive를 capture할 수 있습니다.

### 5. `0540ff1b5a4b` — feat(backup): 백업 출력 경로를 안전하게 예약

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `PERSISTENCE`, `RISK`, `EDGE` |
| 원문에서 정한 역할 | Reserved and identity-checked the destination path. |
| 이전 커밋 | `b478b5243c5a` |
| 다음 커밋 | `6999190ffd34` |

#### 원문이 확정한 범위

<!-- 원문 요약: Normalizes and reserves a new backup output directory while tracking its exact inode identity. -->
<!-- 원문 판단 근거: The small interface prevents overwrite, symlink, and path-substitution races at the publication boundary, protecting the integrity of a high-risk destructive and archival workflow. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `0540ff1b5a4b`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `destination parent resolution`에서 반영이 예상한 parent filesystem 안에서만 일어납니다.
- `tools/stack_backup.py`의 `exclusive reservation / stat identity`에서 사용자가 지정한 pathname을 다른 object로 바꾸는 공격/경쟁을 식별할 기준이 생깁니다.
- `tools/stack_backup.py`의 `pre-publication identity recheck`에서 문자열 path 일치만으로 TOCTOU replacement를 놓치지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 0540ff1b5a4b | tools/stack_backup.py | destination parent resolution | destination parent를 먼저 canonicalize하고 symlink/unsafe parent와 존재하는 non-reservation 대상을 거부합니다. | 반영이 예상한 parent filesystem 안에서만 일어납니다. |
| 0540ff1b5a4b | tools/stack_backup.py | exclusive reservation / stat identity | 최종 path에 비공개 empty reservation object를 exclusive create하고 device/inode identity를 보존합니다. | 사용자가 지정한 pathname을 다른 object로 바꾸는 공격/경쟁을 식별할 기준이 생깁니다. |
| 0540ff1b5a4b | tools/stack_backup.py | pre-반영 identity recheck | rename 직전 path를 다시 `stat`해 최초 reservation과 `(st_dev, st_ino)`가 같은지 확인합니다. | 문자열 path 일치만으로 TOCTOU replacement를 놓치지 않습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 0540ff1b5a4b^ 0540ff1b5a4b -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff b478b5243c5a 0540ff1b5a4b -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | destination이 검증 뒤 symlink나 다른 directory/file로 바뀌면 complete backup을 공격자가 선택한 위치에 게시하거나 기존 data를 덮어쓸 수 있었습니다. |
| 선택한 경계 / 결정 | parent를 고정하고 final pathname에 exclusive 비공개 reservation을 만든 뒤 object identity를 반영 직전 재검사했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/stack_backup.py`의 `destination parent resolution`; `tools/stack_backup.py`의 `exclusive reservation / stat identity`; `tools/stack_backup.py`의 `pre-publication identity recheck` |
| 상태·소유권·수명 변화 | 작업이 reservation inode와 sibling 임시 객체 directory를 소유합니다. caller가 제공한 path string은 더 이상 충분한 authority가 아닙니다. |
| 주요 실패 브랜치 | existing 출력, symlink, unsafe parent, reservation identity mismatch, cross-filesystem rename 조건은 반영 전에 실패합니다. |
| 이 커밋의 보장 | final destination을 기존 object 위에 덮어쓰지 않고 정확히 자신이 예약한 slot에만 게시함을 보장합니다. |
| 한계와 다음 관련 커밋 | reservation만으로 artifact completeness/체크섬/서비스 recovery를 보장하지 않습니다. `6999190ffd34`가 verified 임시 객체 directory를 이 reservation에 atomic replace합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: reservation만으로 artifact completeness/체크섬/서비스 recovery를 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `6999190ffd34`가 verified 임시 객체 directory를 이 reservation에 atomic replace합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: final destination을 기존 object 위에 덮어쓰지 않고 정확히 자신이 예약한 slot에만 게시함을 보장합니다.

### 6. `6999190ffd34` — feat(backup): 백업 세트를 원자적으로 게시

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **S** |
| 태그 | `PERSISTENCE`, `RECOVERY`, `HARD` |
| 원문에서 정한 역할 | 체크섬이 포함된 완전한 백업 세트를 원자적으로 반영하고 서비스를 복구했습니다. |
| 이전 커밋 | `0540ff1b5a4b` |
| 다음 커밋 | `b6920a0c918c` |

#### 원문이 확정한 범위

<!-- 원문 요약: Stops application writers, captures database and WordPress state, writes a checksummed manifest, atomically publishes the set, and recovers services on failure. -->
<!-- 원문 판단 근거: This is the defining backup transaction. It establishes the all-or-nothing publication and service-recovery guarantees needed to treat a directory as a valid backup. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `6999190ffd34`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `backup orchestration`에서 공개/애플리케이션 writers를 quiesce하되 transactional dump를 위해 MariaDB는 계속 실행합니다.
- `tools/stack_backup.py`의 `private sibling capture`에서 불완전 artifact는 final pathname 아래에 보이지 않습니다.
- `tools/stack_backup.py`의 `validate/checksum/manifest/sync`에서 manifest가 정확한 artifact identity를 하나의 set으로 묶습니다.
- `tools/stack_backup.py`의 `reservation identity + atomic replace`에서 관측 가능한 final path는 incomplete reservation에서 complete directory로 한 번에 바뀝니다.
- `tools/stack_backup.py`의 `finally service recovery / cleanup`에서 failed backup이 원본 stack을 degraded 상태로 남기지 않는 반대 보장을 시도합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 6999190ffd34 | tools/stack_backup.py | backup orchestration | 프로젝트 잠금과 signal handler 아래 원본 services, secrets, destination reservation을 검사하고 Nginx와 WordPress를 중지합니다. | 공개/애플리케이션 writers를 quiesce하되 transactional dump를 위해 MariaDB는 계속 실행합니다. |
| 6999190ffd34 | tools/stack_backup.py | 비공개 sibling capture | DB dump와 WordPress archive를 final path와 같은 parent의 비공개 임시 객체 directory에 streaming capture합니다. | 불완전 artifact는 final pathname 아래에 보이지 않습니다. |
| 6999190ffd34 | tools/stack_backup.py | validate/체크섬/manifest/sync | archive 구조를 재검증하고 두 artifact의 size/digest를 manifest에 기록한 뒤 files와 임시 객체 directory를 sync합니다. | manifest가 정확한 artifact identity를 하나의 set으로 묶습니다. |
| 6999190ffd34 | tools/stack_backup.py | reservation identity + atomic replace | reserved inode를 재확인한 뒤 임시 객체 directory를 final destination으로 replace하고 parent를 fsync합니다. | 관측 가능한 final path는 incomplete reservation에서 complete directory로 한 번에 바뀝니다. |
| 6999190ffd34 | tools/stack_backup.py | finally 서비스 recovery / 정리 | 성공·실패·signal 모두에서 Nginx/WordPress를 다시 시작하고 임시 객체/reservation/ready 상태를 정리하며 recovery 실패를 숨기지 않습니다. | failed backup이 원본 stack을 degraded 상태로 남기지 않는 반대 보장을 시도합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 6999190ffd34^ 6999190ffd34 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 0540ff1b5a4b 6999190ffd34 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 S 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 커밋 직전 상태 | 개별 스트림 capture만으로는 DB와 filesystem 사이 write가 끼거나 한 artifact만 final에 보이는 partial set, cancellation 뒤 stopped services가 생길 수 있었습니다. |
| 해결하려던 문제 | capture/validation/sync/identity/rename 어느 단계 실패든 final complete directory를 게시하지 않습니다. finally는 services를 복구하며 복구 실패는 primary error에 추가됩니다. |
| 기존 설계가 충분하지 않았던 이유 | 개별 스트림 capture만으로는 DB와 filesystem 사이 write가 끼거나 한 artifact만 final에 보이는 partial set, cancellation 뒤 stopped services가 생길 수 있었습니다. capture/validation/sync/identity/rename 어느 단계 실패든 final complete directory를 게시하지 않습니다. finally는 services를 복구하며 복구 실패는 primary error에 추가됩니다. |
| 핵심 결정 | writers stop, 비공개 sibling capture, manifest/체크섬/sync, exact reservation, atomic directory replace, unconditional 서비스 recovery를 하나의 locked transaction으로 연결했습니다. |
| 주요 caller → callee / producer → consumer | `tools/stack_backup.py`의 `backup orchestration`; `tools/stack_backup.py`의 `private sibling capture`; `tools/stack_backup.py`의 `validate/checksum/manifest/sync`; `tools/stack_backup.py`의 `reservation identity + atomic replace`; `tools/stack_backup.py`의 `finally service recovery / cleanup` |
| authoritative 상태와 반영 경계 | backup 작업이 일시적으로 애플리케이션 writer lifecycle과 unpublished artifacts를 소유합니다. 반영 후에는 final directory와 manifest가 authoritative backup unit입니다. 성공 시 세 파일의 complete checksummed set만 final path에 보이고, 실패/취소 시 plausible final set이 없으며 원본 애플리케이션 services가 복구됩니다. |
| 소유권 / 수명 / responsibility 변화 | backup 작업이 일시적으로 애플리케이션 writer lifecycle과 unpublished artifacts를 소유합니다. 반영 후에는 final directory와 manifest가 authoritative backup unit입니다. |
| 실패 상황과 recovery path | capture/validation/sync/identity/rename 어느 단계 실패든 final complete directory를 게시하지 않습니다. finally는 services를 복구하며 복구 실패는 primary error에 추가됩니다. |
| 이 커밋이 보장하는 것 | 성공 시 세 파일의 complete checksummed set만 final path에 보이고, 실패/취소 시 plausible final set이 없으며 원본 애플리케이션 services가 복구됩니다. |
| 아직 보장하지 않는 것 | MariaDB transaction 밖의 storage-level crash semantics, 잠금을 무시하는 외부 writer, remote filesystem rename semantics는 보장하지 않습니다. |
| 후속 fix / 테스트와 연결 | `b6920a0c918c`이 실패/signal/잠금 정리를, `030e7310c665`가 signal race와 large 스트림을 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: MariaDB transaction 밖의 storage-level crash semantics, 잠금을 무시하는 외부 writer, remote filesystem rename semantics는 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `b6920a0c918c`이 실패/signal/잠금 정리를, `030e7310c665`가 signal race와 large 스트림을 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 성공 시 세 파일의 complete checksummed set만 final path에 보이고, 실패/취소 시 plausible final set이 없으며 원본 애플리케이션 services가 복구됩니다.

### 7. `b6920a0c918c` — test(backup): 게시 실패와 중단 정리 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `RECOVERY`, `PERSISTENCE` |
| 원문에서 정한 역할 | 실패 시 미반영, 정리, 서비스 복구, 공유 잠금 동작을 검증했습니다. |
| 이전 커밋 | `6999190ffd34` |
| 다음 커밋 | `030e7310c665` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds runtime checks for failed backup publication, signal cancellation, service recovery, temporary cleanup, and cross-`TMPDIR` lock contention. -->
<!-- 원문 판단 근거: It materially validates the negative guarantees of atomic backup: failure must publish nothing, restore the live stack, and release scoped synchronization resources. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `b6920a0c918c`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `backup failure-stage matrix`에서 production backup control flow의 여러 영속 경계를 결정적으로 통과합니다.
- `tests/runtime_stack.py`의 `negative filesystem assertions`에서 failed attempt가 plausible backup 흔적을 남기지 않음을 확인합니다.
- `tests/runtime_stack.py`의 `service health / lock contention`에서 recovery와 fixed 프로젝트 잠금 identity를 함께 검증합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| b6920a0c918c | tests/runtime_stack.py | backup 실패-stage matrix | DB dump, archive, manifest, sync, 반영 전후의 named pause/실패 stage에서 명령 실패 또는 signal을 주입합니다. | production backup control flow의 여러 영속 경계를 결정적으로 통과합니다. |
| b6920a0c918c | tests/runtime_stack.py | negative filesystem 검사문 | final 출력, sibling 임시 객체, reservation/ready marker가 남지 않고 기존 destination은 변경되지 않았는지 검사합니다. | failed attempt가 plausible backup 흔적을 남기지 않음을 확인합니다. |
| b6920a0c918c | tests/runtime_stack.py | 서비스 헬스 상태 / 잠금 contention | 실패 뒤 Nginx/WordPress가 healthy인지 확인하고 다른 TMPDIR 프로세스가 같은 프로젝트 잠금을 얻지 못하는지 검사합니다. | recovery와 fixed 프로젝트 잠금 identity를 함께 검증합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff b6920a0c918c^ b6920a0c918c -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 6999190ffd34 b6920a0c918c -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | 실패하거나 취소된 backup은 complete final set을 게시하지 않고 임시 객체/control 상태를 지우며 원본 services를 복구합니다. |
| 재현하는 실패 / 경계 | capture·manifest·sync·publish 경계의 injected 실패/SIGINT/SIGTERM과 cross-TMPDIR 잠금 contention입니다. |
| 테스트 방식 | 결정적 pause/실패 주입 + live Docker negative 통합 |
| 테스트 준비 코드와 실패 주입 | healthy 원본 stack과 비공개 destination parent를 만든 뒤 named stage ready file에서 backup subprocess를 실패시키거나 signal합니다. |
| 실제 통과하는 실제 실행 경로 | Make/CLI→프로젝트 잠금→writer stop→스트림 capture→반영/정리→서비스 restart 경로를 통과합니다. |
| 핵심 검사문 | final/temp/reservation/ready/잠금 부재, existing 출력 보존, 서비스 헬스 상태와 retry 가능성을 확인합니다. |
| 이 테스트가 증명하는 것 | handler를 통과하는 실패/signal에서 all-or-nothing 반영과 원본 recovery가 적용됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | SIGKILL·storage 실패·uncooperative external writer는 증명하지 않습니다. |
| 성격 | 결정적 negative 런타임 회귀 |
| 막는 후속 회귀 | partial backup 노출, stale 잠금/control file, cancellation 뒤 애플리케이션 서비스 중단을 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: SIGKILL, 실제 disk power loss, 모든 filesystem 구현의 durability는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `030e7310c665`가 repeated signal race와 large 입력/collision edge를 더합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: ordinary 실패와 controlled signal이 non-반영, 정리, 원본 서비스 recovery로 수렴하고 same-프로젝트 잠금이 공유됨을 증명합니다.

### 8. `030e7310c665` — test(backup): 자원 충돌과 시그널 경계 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `PERSISTENCE`, `EDGE` |
| 원문에서 정한 역할 | 시그널 경합, 대용량 데이터, 자원 충돌 경계까지 검증 범위를 넓혔습니다. |
| 이전 커밋 | `b6920a0c918c` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds signal-race checks, labelled and name-only restore-collision refusal, large filesystem and database fixtures, checksums, and stricter secondary cleanup reporting. -->
<!-- 원문 판단 근거: It tests boundary conditions that small normal-path fixtures and simple cancellation cannot cover, protecting the integrity and lifecycle guarantees of backup and restore. -->

- **이 개발 흐름의 재검토 관점:** 이 문서에서는 signal handoff, streaming size, backup 정리 관점을 우선 기록하고 restore collision은 연결 정보로만 남깁니다.

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `030e7310c665`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `large backup fixtures`에서 streaming path가 작은 테스트 준비 코드를 메모리에 우연히 맞춰 처리한 것이 아님을 확인합니다.
- `tests/runtime_stack.py`의 `repeated pause/signal race`에서 signal mask/ready protocol의 timing 규약을 압박합니다.
- `tests/runtime_stack.py`의 `collision fixtures`에서 backup/restore 자원 identity 검사의 coverage를 넓힙니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 030e7310c665 | tests/runtime_stack.py | large backup 테스트 준비 코드 | WordPress에 약 32 MiB file과 MariaDB에 약 4 MiB value를 만들고 backup/restore artifact의 length와 SHA-256을 비교합니다. | streaming path가 작은 테스트 준비 코드를 메모리에 우연히 맞춰 처리한 것이 아님을 확인합니다. |
| 030e7310c665 | tests/runtime_stack.py | repeated pause/signal race | ready 반영과 signal 전달 경계를 반복 실행해 marker가 보였는데 프로세스가 아직 pause하지 않았거나 정리가 누락되는 race를 탐지합니다. | signal mask/ready protocol의 timing 규약을 압박합니다. |
| 030e7310c665 | tests/runtime_stack.py | collision 테스트 준비 코드 | stopped·unlabelled 컨테이너/볼륨/네트워크와 destination collision을 만들어 label-only 또는 running-only 검사가 놓치는 edge를 검사합니다. | backup/restore 자원 identity 검사의 coverage를 넓힙니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 030e7310c665^ 030e7310c665 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff b6920a0c918c 030e7310c665 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | backup/restore 스트림은 큰 입력에서도 truncate하지 않고 signal/collision 경계에서 안전하게 실패합니다. |
| 재현하는 실패 / 경계 | large artifact, repeated ready/signal race, stopped 또는 unlabelled pre-existing 자원입니다. |
| 테스트 방식 | 경계/large-테스트 준비 코드 런타임 회귀 + repeated 결정적 signal injection |
| 테스트 준비 코드와 실패 주입 | 32 MiB filesystem file, 4 MiB DB value, collision Docker objects와 반복 signal run을 만듭니다. |
| 실제 통과하는 실제 실행 경로 | 실제 backup capture/반영 및 restore validation/injection/자원 discovery 경로를 통과합니다. |
| 핵심 검사문 | 원본/restored length·digest, 실패 non-반영, pre-existing object 보존, 정리 outcome을 확인합니다. |
| 이 테스트가 증명하는 것 | 스트림-oriented 구현과 broadened collision/signal 규약이 작은 정상 처리를 넘어 유지됨을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 data distribution·filesystem·scheduler interleaving은 증명하지 않습니다. |
| 성격 | large 경계 and race 회귀 |
| 막는 후속 회귀 | whole-buffer 구현, truncation, label-only collision lookup, ready/signal marker race 회귀를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 무한 크기, 모든 signal interleaving, remote filesystem/object store는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: 개발 흐름 5의 restore large/collision 근거에도 같은 커밋이 사용됩니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: streaming correctness와 pause/signal synchronization이 현실적인 크기와 반복 timing에서도 유지되고 자원 collision detection이 label에만 의존하지 않음을 증명합니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| backup 출력 file은 처음부터 비공개하고 기존 path를 덮어쓰지 않습니다. | fdd55605ba74 | 0540ff1b5a4b | b6920a0c918c | exclusive 0600 file와 destination reservation/identity check, existing-출력 negative 검사문이 연결됩니다. |
| 같은 프로젝트의 backup은 다른 mutating management 작업과 겹치지 않습니다. | 3a0995ff0d4f | 6999190ffd34 | b6920a0c918c cross-TMPDIR contention | 프로젝트-name-derived 잠금을 transaction 전체에서 보유하고 TMPDIR 변경으로 우회되지 않습니다. |
| published backup set은 DB dump, WordPress archive, matching manifest의 완전한 단위입니다. | b478b5243c5a, 0540ff1b5a4b | 6999190ffd34 | b6920a0c918c, 030e7310c665 | 비공개 capture→체크섬 manifest→sync→atomic directory replace와 digest/length 검사문이 연결됩니다. |
| 실패하거나 취소된 backup은 plausible final set을 남기지 않고 애플리케이션 services를 복구합니다. | 6999190ffd34 | b6920a0c918c | 030e7310c665 | finally recovery와 negative final/temp/ready/서비스 검사문이 반대 상태를 검증합니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: DB dump와 WordPress archive를 순차적으로 final path에 쓰면 두 artifact 중 하나만 보이는 plausible partial backup과 writer 중단 뒤 degraded 원본 stack이 남을 수 있었습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: 비공개 O_EXCL files, fsync/체크섬/manifest, inode-checked reservation, 애플리케이션-writer quiescence, directory replacement와 서비스 recovery가 반영 경계를 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `b6920a0c918c` negative scenarios와 `030e7310c665` signal-race/large-테스트 준비 코드 checks가 non-반영, 정리, recovery와 streaming 경계를 보호합니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| DB dump와 filesystem archive 사이 write 또는 partial files 노출 | 6999190ffd34의 writers stop + sibling temp + manifest + atomic replace | b6920a0c918c 반영 실패/signal non-반영 | capture와 반영을 분리하고 final path를 complete set에만 부여합니다. |
| destination validation 뒤 symlink/object replacement | 0540ff1b5a4b parent resolution과 device/inode reservation | unsafe destination/collision 런타임 cases | pathname equality 대신 actual reserved object identity를 재확인합니다. |
| asynchronous signal로 정리 timing과 ready marker 불일치 | d26c885c5cd5 signal-to-exception과 masked ready 반영 | 030e7310c665 repeated pause/signal race | 테스트와 production이 같은 named stage handoff를 사용합니다. |
| small 테스트 준비 코드에서만 streaming이 우연히 동작 | b478b5243c5a 스트림 capture | 030e7310c665 32 MiB/4 MiB 체크섬·length | whole-buffer assumption과 truncation을 별도 경계 테스트로 막습니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Source stack services | 모두 running | backup이 writer quiescence와 recovery를 일시 소유 | 6999190ffd34 stop/start/finally | MariaDB는 dump 동안 running, Nginx/WordPress는 capture cut을 위해 중지됩니다. |
| Temporary sibling directory | 없음 | unpublished dump/archive/manifest를 독점 보유 | exclusive creation, sync, 정리 | 실패 시 제거되고 final consumer는 볼 수 없습니다. |
| Reserved destination | user path 문자열 | empty 비공개 inode가 반영 slot 역할 | 0540ff1b5a4b dev/inode check | 정확히 예약한 object만 replace 대상입니다. |
| Published backup set | 개별 artifact | manifest가 artifact identity/체크섬을 묶는 authoritative unit | 6999190ffd34 manifest/schema/digest | 세 파일 전체가 restore 입력 단위입니다. |
| Signal/pause 상태 | OS의 비동기 종료 | normal 정리 path와 결정적 테스트 handoff | d26c885c5cd5 handler/ready lifecycle | signal을 ordinary 실패 semantics로 수렴시킵니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: The implementation deliberately separates data capture from 반영. Private streaming files, an exact output reservation, a manifest, and directory replacement ensure that only a complete set becomes visible. Signal-aware recovery and negative 런타임 테스트 establish the equally important converse: cancelled or failed work must not leave a plausible backup or a degraded 원본 stack. -->
- 최종 authoritative 상태와 소유자: final backup directory와 manifest가 complete set의 authoritative 상태이며 원본 MariaDB/WordPress volumes는 원본 상태 소유자로 남습니다.
- 정상 실행의 entry point와 완료 조건: locked backup이 원본 validation, writer stop, two-스트림 capture, validation/체크섬/sync, atomic 반영, 서비스 recovery를 끝내면 완료입니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: 실패/signal 시 임시 객체/reservation/control 상태를 제거하고 원본 services를 재시작합니다. 서비스 recovery 실패는 primary 실패와 함께 보고합니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: 개발 흐름 5 restore가 stable 파일 디스크립터로 열고 검증할 비공개 checksummed 입력을 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: Docker 미설치 환경에서는 런타임 실패 matrix를 실행하지 않았으며 코드에 구현된 mechanism만 확인했습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | 잠금/signal 설정 | 3a0995ff0d4f + d26c885c5cd5 | same-프로젝트 serialization과 cancellation exception을 설정합니다. | contention/signal이면 변경 전 또는 common 정리 path로 실패합니다. |
| 2 | 원본/destination 검증 | 0540ff1b5a4b + 6999190ffd34 | services/secrets와 exact destination reservation을 확인합니다. | unsafe/existing/mismatched path면 writer를 멈추지 않고 거부합니다. |
| 3 | writers quiesce | 6999190ffd34 backup orchestration | Nginx와 WordPress를 중지하고 MariaDB는 transaction dump를 위해 유지합니다. | stop 실패면 capture를 진행하지 않고 recovery를 시도합니다. |
| 4 | 스트림 capture | b478b5243c5a helpers | DB dump와 WordPress archive를 비공개 sibling files에 씁니다. | subprocess/스트림/archive 실패면 temp를 제거합니다. |
| 5 | manifest/durability | fdd55605ba74 + 6999190ffd34 | digest/size/manifest와 file/directory fsync를 완료합니다. | 검증/sync 실패면 final path에 complete set을 게시하지 않습니다. |
| 6 | atomic 반영 | 0540ff1b5a4b + 6999190ffd34 | reservation identity 후 sibling directory를 final path로 replace합니다. | identity mismatch/rename 실패는 non-반영입니다. |
| 7 | 서비스 recovery/정리 | 6999190ffd34 finally | 성공·실패 모두에서 애플리케이션 services를 healthy로 되돌립니다. | recovery 실패는 숨기지 않고 작업 result를 실패로 유지합니다. |

### 학습자의 최종 설명

> backup은 파일 두 개를 만드는 작업이 아니라 원본 writer를 일시 정지하고 complete set을 한 번에 공개하는 transaction입니다. DB는 running MariaDB의 single-transaction 스트림으로, WordPress는 read-only 볼륨 archive 스트림으로 비공개 sibling에 수집됩니다. 출력 pathname은 미리 비공개 inode로 예약되고 반영 직전 device/inode가 다시 확인됩니다. archive 검증과 체크섬 manifest, file/directory sync가 끝난 directory만 final path로 atomic replace됩니다. ordinary 실패와 SIGINT/SIGTERM은 같은 finally로 들어가 temp/control 상태를 제거하고 Nginx/WordPress를 복구합니다. negative 테스트는 “성공한 backup이 맞다”뿐 아니라 “실패한 시도는 plausible set과 degraded 원본을 남기지 않는다”는 반대 불변식을 검사합니다.

## 학습 완료 자가 점검

- [x] 두 artifact의 생성 성공을 atomic 반영과 같은 의미로 썼습니까?
- [x] file fsync와 containing directory fsync의 역할을 구분했습니까?
- [x] MariaDB까지 멈춘다고 잘못 설명하지 않았습니까?
- [x] 실패 뒤 원본 services recovery 실패가 별도 오류로 surfaced되는지 확인했습니까?
- [x] signal 테스트의 ready marker가 sleep 기반 동기화가 아니라는 점을 코드로 확인했습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 04-atomic-backup-publication.md =====

===== BEGIN FILE: 05-fresh-project-restore.md =====
# 개발 흐름 5 — 새 프로젝트로의 검증된 복원과 실패 시 정리

## 개발 흐름의 목표

backup을 기존 상태 위에 덮는 작업이 아니라 완전히 fresh한 Compose 프로젝트를 생성하는 transaction으로 정의하고, verified 입력·streaming injection·실패 되돌리기를 통해 all-or-nothing restore를 복원합니다.

### 원문에서 정한 의의

복원을 기존 프로젝트 덮어쓰기가 아니라 새 프로젝트 생성으로 취급합니다. 입력을 검증하고 이름 충돌을 확인한 뒤에만 자원을 만들며, 실패하면 이번 시도에서 만든 자원을 모두 제거합니다. 기존 객체 보존과 큰 스트리밍 입력에서도 같은 동작을 테스트합니다.

<details>
<summary>영문 원문</summary>

> Restore is treated as creation of a new project, not as an in-place overwrite. That constraint makes rollback tractable: verified input is applied only after collision checks, and any failure removes the resources created by the attempt. The later tests show that refusal preserves pre-existing objects and that the streaming implementation remains correct beyond small fixtures.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- restore 대상이 반드시 empty namespace여야 되돌리기 소유권이 단순해지는 이유는 무엇입니까?
- Compose label만으로 collision을 찾지 못하는 경우와 exact rendered name만으로 부족한 경우는 무엇입니까?
- backup path를 반복 resolve하지 않고 파일 디스크립터-anchored object로 유지하는 이유는 무엇입니까?
- archive validation과 empty-볼륨 precondition은 각각 어떤 write/merge 위험을 막습니까?
- `compose down --volumes` 실행 성공만으로 되돌리기 complete라고 할 수 없는 이유는 무엇입니까?
- restore 실패와 정리 실패가 동시에 발생할 때 오류 context는 어떻게 보존됩니까?

## 완료 기준

- fresh-대상 detection이 label, exact names, rendered 볼륨/네트워크 names를 모두 사용하는 이유를 확인했습니다.
- `VerifiedBackup`이 directory 파일 디스크립터와 opened streams를 restore 종료까지 유지하는 경계를 추적했습니다.
- SQL import와 WordPress extraction이 empty new volumes에 streaming으로 적용되는 경로를 확인했습니다.
- 실패 이후 Compose 정리와 independent 자원 enumeration이 모두 통과해야 하는 조건을 기록했습니다.
- malformed 입력, signal, injected 실패, pre-existing collision, successful restore, second restore refusal을 분리했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `e5cb60c7d743` | feat(restore): Compose 리소스 이름과 기존 객체 조회 | **B** | `PERSISTENCE`<br>`OPERATIONS` | 렌더링된 이름과 관례적 이름의 Docker 자원을 찾도록 했습니다. |
| 2 | `851dc1708881` | feat(restore): 대상 프로젝트 자원 충돌 사전 차단 | **A** | `PERSISTENCE`<br>`RISK`<br>`EDGE` | 대상 프로젝트가 비어 있어야 복원할 수 있도록 사전 조건을 정했습니다. |
| 3 | `953a0f6bd571` | feat(restore): 백업 입력의 형식과 체크섬 검증 | **A** | `PERSISTENCE`<br>`RISK`<br>`EDGE` | 비공개·잠금·체크섬 검증을 거친 백업 입력 경계를 정했습니다. |
| 4 | `1250fcf7c006` | feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입 | **B** | `PERSISTENCE`<br>`INTEGRATION` | Injected SQL and WordPress streams into empty new volumes. |
| 5 | `9ca04b1c30cd` | feat(restore): 실패한 복원 자원을 정리하고 롤백 | **S** | `PERSISTENCE`<br>`RECOVERY`<br>`HARD` | Orchestrated startup and removed every partial 자원 after 실패. |
| 6 | `3a37a491ecea` | feat(restore): 복원 CLI와 Make 타깃 연결 | **B** | `PERSISTENCE`<br>`OPERATIONS` | Exposed restore through the CLI and Makefile. |
| 7 | `4f8eb9aff842` | test(restore): 거부·롤백·복원 상태 검증 | **A** | `TEST`<br>`RECOVERY`<br>`RISK` | 잘못된 입력 거부, 실패 정리, 중단, 정상 복원 상태를 검증했습니다. |
| 8 | `030e7310c665` | test(backup): 자원 충돌과 시그널 경계 검증 | **A** | `TEST`<br>`PERSISTENCE`<br>`EDGE` | 중지된 자원, 라벨 없는 충돌, 대용량 복원 입력을 추가로 검증했습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `e5cb60c7d743` — feat(restore): Compose 리소스 이름과 기존 객체 조회

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `PERSISTENCE`, `OPERATIONS` |
| 원문에서 정한 역할 | 렌더링된 이름과 관례적 이름의 Docker 자원을 찾도록 했습니다. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `851dc1708881` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds discovery of rendered Compose resource names and existing labelled or conventionally named objects. -->
<!-- 원문 판단 근거: This is necessary restore plumbing, but it mainly inventories resources within the restore architecture developed by subsequent commits. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `e5cb60c7d743`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `rendered resource-name helpers`에서 restore가 실제 Docker object 이름을 변경 전에 계산합니다.
- `tools/stack_backup.py`의 `expected container-name candidates`에서 label이 없거나 stopped인 object도 exact name으로 찾을 후보를 확보합니다.
- `tools/stack_backup.py`의 `list labelled/exact resources`에서 한 discovery 방식의 누락을 다른 방식으로 보완합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| e5cb60c7d743 | tools/stack_backup.py | rendered 자원-name helpers | rendered Compose JSON에서 볼륨/네트워크의 concrete name을 읽고 explicit `name`과 프로젝트-prefixed default를 구분합니다. | restore가 실제 Docker object 이름을 변경 전에 계산합니다. |
| e5cb60c7d743 | tools/stack_backup.py | expected 컨테이너-name candidates | 서비스와 one-off 초기화 컨테이너의 current/legacy Compose naming form을 프로젝트/서비스/index 조합으로 만듭니다. | label이 없거나 stopped인 object도 exact name으로 찾을 후보를 확보합니다. |
| e5cb60c7d743 | tools/stack_backup.py | list labelled/exact 자원 | 컨테이너, volumes, networks를 프로젝트 label과 exact expected name 양쪽으로 query합니다. | 한 discovery 방식의 누락을 다른 방식으로 보완합니다. |

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | 렌더링된 이름과 관례적 이름의 Docker 자원을 찾도록 했습니다. |
| 핵심 입력 / 출력 / 상태 | host restore 프로세스가 대상 프로젝트의 expected 자원 identity와 discovery 결과를 소유합니다. |
| 변경된 directive / helper / 명령 | `tools/stack_backup.py`의 `rendered resource-name helpers`; `tools/stack_backup.py`의 `expected container-name candidates`; `tools/stack_backup.py`의 `list labelled/exact resources` |
| immediate 실패 또는 경계 | Compose config rendering 실패, Docker query timeout/nonzero, ambiguous/malformed 출력은 변경 전에 restore-domain error가 됩니다. |
| 다음 커밋에 넘긴 한계 | 열거 결과를 fresh-대상 precondition으로 강제하거나 실패 시 정리하는 orchestration은 아직 없습니다. `851dc1708881`이 이 inventory를 restore 시작 전 emptiness gate로 사용합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 열거 결과를 fresh-대상 precondition으로 강제하거나 실패 시 정리하는 orchestration은 아직 없습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `851dc1708881`이 이 inventory를 restore 시작 전 emptiness gate로 사용합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 대상 프로젝트에 관련될 수 있는 concrete 자원 names와 existing objects를 체계적으로 열거할 수 있습니다.

### 2. `851dc1708881` — feat(restore): 대상 프로젝트 자원 충돌 사전 차단

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `PERSISTENCE`, `RISK`, `EDGE` |
| 원문에서 정한 역할 | 대상 프로젝트가 비어 있어야 복원할 수 있도록 사전 조건을 정했습니다. |
| 이전 커밋 | `e5cb60c7d743` |
| 다음 커밋 | `953a0f6bd571` |

#### 원문이 확정한 범위

<!-- 원문 요약: Rejects restore targets that already contain matching containers, volumes, or networks. -->
<!-- 원문 판단 근거: Fresh-project enforcement prevents restore from overwriting or mixing with live state and establishes a significant safety precondition for all later restore steps. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `851dc1708881`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `ensure_fresh_project`에서 stopped·unlabelled·partially created object까지 preflight 범위에 넣습니다.
- `tools/stack_backup.py`의 `collision report`에서 기존 object를 지우거나 덮어쓰는 대신 operator에게 소유권 충돌을 보여줍니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 851dc1708881 | tools/stack_backup.py | ensure_fresh_project | 프로젝트 label로 컨테이너/volumes/networks를 찾고 rendered exact 볼륨/네트워크 names와 expected current/legacy 컨테이너 names를 추가로 확인합니다. | stopped·unlabelled·partially created object까지 preflight 범위에 넣습니다. |
| 851dc1708881 | tools/stack_backup.py | collision report | 발견한 kind/name을 모아 하나라도 있으면 restore 변경 전에 명시적인 refusal error를 만듭니다. | 기존 object를 지우거나 덮어쓰는 대신 operator에게 소유권 충돌을 보여줍니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 851dc1708881^ 851dc1708881 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff e5cb60c7d743 851dc1708881 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 기존 프로젝트에 restore하면 unrelated data와 merge/overwrite되고 실패 되돌리기 때 어떤 object가 원래 있었는지 구분하기 어렵습니다. |
| 선택한 경계 / 결정 | 대상 namespace가 완전히 비어 있다는 것을 restore transaction의 필수 사전 조건으로 만들었습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/stack_backup.py`의 `ensure_fresh_project`; `tools/stack_backup.py`의 `collision report` |
| 상태·소유권·수명 변화 | preflight 성공 뒤 restore attempt가 대상 프로젝트에서 새로 생기는 모든 object를 독점 소유한다고 추론할 수 있습니다. |
| 주요 실패 브랜치 | labelled/exact/rendered 이름 중 하나라도 존재하면 입력을 쓰기 전에 실패합니다. 발견된 기존 object는 수정하거나 제거하지 않습니다. |
| 이 커밋의 보장 | restore는 fresh 프로젝트 생성으로만 시작하며 되돌리기 소유권을 새 attempt가 만든 자원으로 한정합니다. |
| 한계와 다음 관련 커밋 | preflight 직후 외부 actor가 같은 name을 만드는 race를 완전히 막지는 않으며 actual Docker create error도 처리해야 합니다. `9ca04b1c30cd`이 프로젝트 잠금과 정리 되돌리기 안에서 이 precondition을 재사용하고 테스트가 collision 보존을 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: preflight 직후 외부 actor가 같은 name을 만드는 race를 완전히 막지는 않으며 actual Docker create error도 처리해야 합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `9ca04b1c30cd`이 프로젝트 잠금과 정리 되돌리기 안에서 이 precondition을 재사용하고 테스트가 collision 보존을 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: restore는 fresh 프로젝트 생성으로만 시작하며 되돌리기 소유권을 새 attempt가 만든 자원으로 한정합니다.

### 3. `953a0f6bd571` — feat(restore): 백업 입력의 형식과 체크섬 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `PERSISTENCE`, `RISK`, `EDGE` |
| 원문에서 정한 역할 | 비공개·잠금·체크섬 검증을 거친 백업 입력 경계를 정했습니다. |
| 이전 커밋 | `851dc1708881` |
| 다음 커밋 | `1250fcf7c006` |

#### 원문이 확정한 범위

<!-- 원문 요약: Opens a private backup set with no-follow and locking checks, validates its exact files, manifest format, checksums, and archive structure. -->
<!-- 원문 판단 근거: This creates the restore trust boundary. It ensures restoration consumes one stable, owner-controlled, internally consistent backup rather than mutable or substituted input. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `953a0f6bd571`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `VerifiedBackup`에서 user pathname을 한 번 검증한 stable directory object로 고정합니다.
- `tools/stack_backup.py`의 `openat-style artifact opens / shared locks`에서 검증 후 pathname substitution과 concurrent writer 변경을 줄입니다.
- `tools/stack_backup.py`의 `manifest/schema/checksum/archive validation`에서 restore 변경 전에 입력 completeness와 구조를 증명합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 953a0f6bd571 | tools/stack_backup.py | VerifiedBackup | backup directory를 no-follow 파일 디스크립터로 열고 소유자/mode/type/link를 검사한 뒤 expected artifact names만 허용합니다. | user pathname을 한 번 검증한 stable directory object로 고정합니다. |
| 953a0f6bd571 | tools/stack_backup.py | openat-style artifact opens / shared locks | directory 파일 디스크립터를 기준으로 DB dump, WordPress archive, manifest를 다시 no-follow로 열고 regular/single-link/비공개 조건과 shared 잠금을 유지합니다. | 검증 후 pathname substitution과 concurrent writer 변경을 줄입니다. |
| 953a0f6bd571 | tools/stack_backup.py | manifest/schema/체크섬/archive validation | manifest의 exact file set, size, SHA-256을 열린 스트림과 비교하고 tar member path/type policy를 검사합니다. | restore 변경 전에 입력 completeness와 구조를 증명합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 953a0f6bd571^ 953a0f6bd571 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 851dc1708881 953a0f6bd571 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | backup pathname을 단계마다 다시 열면 검증된 object와 실제 주입 object가 달라질 수 있고 malformed archive/체크섬이 대상을 일부 변경할 수 있었습니다. |
| 선택한 경계 / 결정 | directory와 artifact descriptors를 restore 수명 동안 유지하는 `VerifiedBackup` 경계를 만들고 모든 형식·체크섬 검증을 변경보다 앞에 배치했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/stack_backup.py`의 `VerifiedBackup`; `tools/stack_backup.py`의 `openat-style artifact opens / shared locks`; `tools/stack_backup.py`의 `manifest/schema/checksum/archive validation` |
| 상태·소유권·수명 변화 | VerifiedBackup object가 directory/file descriptors, shared locks, 스트림 positions를 소유하고 restore orchestration은 이 stable handles만 소비합니다. |
| 주요 실패 브랜치 | symlink, wrong 소유자/mode/link/type, extra/missing file, malformed manifest, size/digest mismatch, unsafe archive member는 대상 자원 생성 전에 실패합니다. |
| 이 커밋의 보장 | restore 입력이 비공개 소유자-controlled exact files의 checksummed structurally valid set이며 검증한 object와 사용하는 object가 같은 파일 디스크립터에 anchored됨을 보장합니다. |
| 한계와 다음 관련 커밋 | backup이 애플리케이션-consistent하게 생성됐는지는 manifest만으로 다시 증명하지 않으며 그 속성은 개발 흐름 4 반영 과정에 의존합니다. `1250fcf7c006`이 retained streams를 fresh volumes에 직접 주입하고 `4f8eb9aff842`이 malformed/symlink 입력 refusal을 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: backup이 애플리케이션-consistent하게 생성됐는지는 manifest만으로 다시 증명하지 않으며 그 속성은 개발 흐름 4 반영 과정에 의존합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `1250fcf7c006`이 retained streams를 fresh volumes에 직접 주입하고 `4f8eb9aff842`이 malformed/symlink 입력 refusal을 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: restore 입력이 비공개 소유자-controlled exact files의 checksummed structurally valid set이며 검증한 object와 사용하는 object가 같은 파일 디스크립터에 anchored됨을 보장합니다.

### 4. `1250fcf7c006` — feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `PERSISTENCE`, `INTEGRATION` |
| 원문에서 정한 역할 | Injected SQL and WordPress streams into empty new volumes. |
| 이전 커밋 | `953a0f6bd571` |
| 다음 커밋 | `9ca04b1c30cd` |

#### 원문이 확정한 범위

<!-- 원문 요약: Imports the SQL stream into MariaDB and extracts the WordPress archive only into empty data and config volumes. -->
<!-- 원문 판단 근거: It is core restore work, but the implementation follows the already-defined verified-input and fresh-target contracts; rollback and lifecycle safety arrive later. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `1250fcf7c006`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `database restore helper`에서 large dump를 host memory에 전부 올리지 않고 DB 서비스 인터페이스로 주입합니다.
- `tools/stack_backup.py`의 `WordPress extraction helper`에서 host가 Docker 볼륨 path를 직접 탐색하지 않습니다.
- `tools/stack_backup.py`의 `empty mount checks`에서 기존 상태와 archive를 합치는 제자리 overwrite를 막습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 1250fcf7c006 | tools/stack_backup.py | database restore helper | fresh MariaDB 초기화를 완료한 뒤 retained SQL 스트림을 클라이언트 stdin으로 전달해 import합니다. | large dump를 host memory에 전부 올리지 않고 DB 서비스 인터페이스로 주입합니다. |
| 1250fcf7c006 | tools/stack_backup.py | WordPress extraction helper | new WordPress data/config volumes를 one-off 컨테이너에 마운트하고 validated archive 스트림을 tar stdin으로 전달합니다. | host가 Docker 볼륨 path를 직접 탐색하지 않습니다. |
| 1250fcf7c006 | tools/stack_backup.py | empty 마운트 checks | extraction 전에 대상 마운트 roots가 비어 있는지 검사하고 expected root mapping 외 merge를 거부합니다. | 기존 상태와 archive를 합치는 제자리 overwrite를 막습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 1250fcf7c006^ 1250fcf7c006 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 953a0f6bd571 1250fcf7c006 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | Injected SQL and WordPress streams into empty new volumes. |
| 핵심 입력 / 출력 / 상태 | MariaDB 클라이언트가 relational import를, one-off tar 컨테이너가 filesystem extraction을 소유합니다. VerifiedBackup은 원본 스트림 descriptors를 끝까지 유지합니다. |
| 변경된 directive / helper / 명령 | `tools/stack_backup.py`의 `database restore helper`; `tools/stack_backup.py`의 `WordPress extraction helper`; `tools/stack_backup.py`의 `empty mount checks` |
| immediate 실패 또는 경계 | empty precondition 위반, 클라이언트/tar timeout/nonzero, broken pipe, extraction policy 실패는 partial restore error가 됩니다. |
| 다음 커밋에 넘긴 한계 | import 중간 실패 뒤 partial volumes를 제거하거나 full 애플리케이션 헬스 상태까지 수렴하는 되돌리기는 아직 보장하지 않습니다. `9ca04b1c30cd`이 primitives를 transaction으로 묶고 실패 시 created 자원을 제거합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: import 중간 실패 뒤 partial volumes를 제거하거나 full 애플리케이션 헬스 상태까지 수렴하는 되돌리기는 아직 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `9ca04b1c30cd`이 primitives를 transaction으로 묶고 실패 시 created 자원을 제거합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: validated backup streams를 새 DB/WordPress volumes에 bounded-memory 방식으로 주입할 수 있습니다.

### 5. `9ca04b1c30cd` — feat(restore): 실패한 복원 자원을 정리하고 롤백

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **S** |
| 태그 | `PERSISTENCE`, `RECOVERY`, `HARD` |
| 원문에서 정한 역할 | Orchestrated startup and removed every partial 자원 after 실패. |
| 이전 커밋 | `1250fcf7c006` |
| 다음 커밋 | `3a37a491ecea` |

#### 원문이 확정한 범위

<!-- 원문 요약: Orchestrates fresh database bootstrap, SQL import, WordPress extraction, application startup, and complete resource cleanup on any restore failure. -->
<!-- 원문 판단 근거: This is the defining restore mechanism and its critical failure invariant. Without it, partial restoration could leave plausible but unusable project resources and make retries unsafe. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `9ca04b1c30cd`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `restore orchestration`에서 입력/freshness 검증이 모든 대상 변경보다 앞섭니다.
- `tools/stack_backup.py`의 `cleanup_failed_restore`에서 Compose가 아는 created 자원의 일차 되돌리기를 수행합니다.
- `tools/stack_backup.py`의 `independent residual enumeration`에서 정리 명령 성공을 정리 complete로 오인하지 않습니다.
- `tools/stack_backup.py`의 `error chaining / service convergence`에서 성공은 healthy stack, 실패는 zero-owned-자원 상태라는 양쪽 endpoint를 정의합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 9ca04b1c30cd | tools/stack_backup.py | restore orchestration | 프로젝트 잠금과 signal handler 아래 VerifiedBackup 검증과 fresh-대상 check를 마친 뒤 MariaDB 초기화, SQL import, WordPress extraction, normal 애플리케이션 초기화/start를 순서대로 실행합니다. | 입력/freshness 검증이 모든 대상 변경보다 앞섭니다. |
| 9ca04b1c30cd | tools/stack_backup.py | cleanup_failed_restore | 예외 또는 cancellation 시 `compose down --volumes --remove-orphans`를 실행하고 결과를 검사합니다. | Compose가 아는 created 자원의 일차 되돌리기를 수행합니다. |
| 9ca04b1c30cd | tools/stack_backup.py | independent residual enumeration | down 성공 여부와 별도로 프로젝트 label, rendered names, exact expected names를 다시 query해 컨테이너/volumes/networks가 0인지 확인합니다. | 정리 명령 성공을 정리 complete로 오인하지 않습니다. |
| 9ca04b1c30cd | tools/stack_backup.py | error chaining / 서비스 convergence | primary restore error와 정리 실패를 모두 보존하고 성공 path는 애플리케이션 services/헬스 상태까지 완료합니다. | 성공은 healthy stack, 실패는 zero-owned-자원 상태라는 양쪽 endpoint를 정의합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 9ca04b1c30cd^ 9ca04b1c30cd -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 1250fcf7c006 9ca04b1c30cd -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 S 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 커밋 직전 상태 | 스트림 import/extraction 중 실패하면 새 DB row, partial files, 컨테이너/volumes/networks가 남아 다음 restore를 막거나 incomplete 프로젝트처럼 보일 수 있었습니다. |
| 해결하려던 문제 | 어느 변경 이후든 exception/signal은 정리 path로 갑니다. down 실패 또는 residual object는 별도 정리 error로 보고되며 primary 실패를 덮지 않습니다. |
| 기존 설계가 충분하지 않았던 이유 | 스트림 import/extraction 중 실패하면 새 DB row, partial files, 컨테이너/volumes/networks가 남아 다음 restore를 막거나 incomplete 프로젝트처럼 보일 수 있었습니다. 어느 변경 이후든 exception/signal은 정리 path로 갑니다. down 실패 또는 residual object는 별도 정리 error로 보고되며 primary 실패를 덮지 않습니다. |
| 핵심 결정 | fresh namespace, verified 입력, ordered streaming injection, normal 초기화 reuse, unconditional compose 정리와 independent zero-자원 verification을 하나의 restore transaction으로 연결했습니다. |
| 주요 caller → callee / producer → consumer | `tools/stack_backup.py`의 `restore orchestration`; `tools/stack_backup.py`의 `cleanup_failed_restore`; `tools/stack_backup.py`의 `independent residual enumeration`; `tools/stack_backup.py`의 `error chaining / service convergence` |
| authoritative 상태와 반영 경계 | preflight 뒤 대상 namespace의 새 자원은 restore attempt가 독점 소유합니다. 성공하면 새 프로젝트가 상태 소유자가 되고 실패하면 소유자가 만든 모든 object를 제거해야 합니다. 성공 시 backup 상태를 가진 healthy fresh 프로젝트, 실패 시 attempt-owned Docker 자원이 하나도 없는 상태를 목표로 하고 검증합니다. |
| 소유권 / 수명 / responsibility 변화 | preflight 뒤 대상 namespace의 새 자원은 restore attempt가 독점 소유합니다. 성공하면 새 프로젝트가 상태 소유자가 되고 실패하면 소유자가 만든 모든 object를 제거해야 합니다. |
| 실패 상황과 recovery path | 어느 변경 이후든 exception/signal은 정리 path로 갑니다. down 실패 또는 residual object는 별도 정리 error로 보고되며 primary 실패를 덮지 않습니다. |
| 이 커밋이 보장하는 것 | 성공 시 backup 상태를 가진 healthy fresh 프로젝트, 실패 시 attempt-owned Docker 자원이 하나도 없는 상태를 목표로 하고 검증합니다. |
| 아직 보장하지 않는 것 | non-cooperating 외부 actor가 동시에 만든 exact object, Docker 데몬 crash 중 정리, physical storage remnants는 완전하게 보장하지 않습니다. |
| 후속 fix / 테스트와 연결 | `4f8eb9aff842`이 malformed/refusal/injected 실패/SIGINT/success/second-restore를 검증하고 `030e7310c665`이 collision/large cases를 확장합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: non-cooperating 외부 actor가 동시에 만든 exact object, Docker 데몬 crash 중 정리, physical storage remnants는 완전하게 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `4f8eb9aff842`이 malformed/refusal/injected 실패/SIGINT/success/second-restore를 검증하고 `030e7310c665`이 collision/large cases를 확장합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 성공 시 backup 상태를 가진 healthy fresh 프로젝트, 실패 시 attempt-owned Docker 자원이 하나도 없는 상태를 목표로 하고 검증합니다.

### 6. `3a37a491ecea` — feat(restore): 복원 CLI와 Make 타깃 연결

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `PERSISTENCE`, `OPERATIONS` |
| 원문에서 정한 역할 | Exposed restore through the CLI and Makefile. |
| 이전 커밋 | `9ca04b1c30cd` |
| 다음 커밋 | `4f8eb9aff842` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds `restore` CLI dispatch and a guarded Make target. -->
<!-- 원문 판단 근거: It exposes the completed restore path without materially changing its correctness or recovery model. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `3a37a491ecea`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/stack_backup.py`의 `CLI restore subcommand`에서 operator-facing entry point가 internal helper와 같은 transaction을 사용합니다.
- `Makefile`의 `restore target`에서 manual 명령 조립을 줄이고 documented management path를 제공합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3a37a491ecea | tools/stack_backup.py | CLI restore subcommand | backup path, 프로젝트, env/Compose inputs를 parse하고 production restore orchestration을 호출하며 domain error를 nonzero exit로 변환합니다. | operator-facing entry point가 internal helper와 같은 transaction을 사용합니다. |
| 3a37a491ecea | Makefile | restore 대상 | 명시적 backup 원본과 프로젝트/env variables를 CLI에 전달하는 대상을 추가합니다. | manual 명령 조립을 줄이고 documented management path를 제공합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 3a37a491ecea^ 3a37a491ecea -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 9ca04b1c30cd 3a37a491ecea -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | Exposed restore through the CLI and Makefile. |
| 핵심 입력 / 출력 / 상태 | CLI가 argument validation과 프로세스 exit status를 소유하고 transaction 자체의 자원 소유권은 restore orchestration에 남습니다. |
| 변경된 directive / helper / 명령 | `tools/stack_backup.py`의 `CLI restore subcommand`; `Makefile`의 `restore target` |
| immediate 실패 또는 경계 | 필수 path/variable 누락과 RestoreError는 변경 여부에 맞는 nonzero status와 stderr로 surfaced됩니다. |
| 다음 커밋에 넘긴 한계 | CLI 연결 자체는 restore correctness나 정리를 추가로 증명하지 않습니다. `4f8eb9aff842`이 실제 명령 path로 실패와 success를 실행합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: CLI 연결 자체는 restore correctness나 정리를 추가로 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `4f8eb9aff842`이 실제 명령 path로 실패와 success를 실행합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: documented 명령 path가 동일 잠금/입력/freshness/되돌리기 semantics를 사용함을 보장합니다.

### 7. `4f8eb9aff842` — test(restore): 거부·롤백·복원 상태 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `RECOVERY`, `RISK` |
| 원문에서 정한 역할 | 잘못된 입력 거부, 실패 정리, 중단, 정상 복원 상태를 검증했습니다. |
| 이전 커밋 | `3a37a491ecea` |
| 다음 커밋 | `030e7310c665` |

#### 원문이 확정한 범위

<!-- 원문 요약: Tests symlinked backup rejection, injected and signalled restore failure cleanup, successful data recovery, and refusal to restore twice. -->
<!-- 원문 판단 근거: These scenarios validate the restore security and rollback contracts against real Docker resources, significantly increasing confidence in a high-risk mechanism. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `4f8eb9aff842`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `malformed/unsafe backup cases`에서 `VerifiedBackup` 신뢰 경계가 잘못된 입력을 거부한다는 근거를 제공합니다.
- `tests/runtime_stack.py`의 `restore failure injection / SIGINT`에서 partial 대상 자원이 생길 수 있는 실제 되돌리기 path를 통과합니다.
- `tests/runtime_stack.py`의 `zero-resource assertions`에서 `down` 호출 여부가 아니라 되돌리기 endpoint를 확인합니다.
- `tests/runtime_stack.py`의 `success and second restore refusal`에서 fresh-프로젝트 불변식의 positive/negative 양쪽을 검증합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 4f8eb9aff842 | tests/runtime_stack.py | malformed/unsafe backup cases | manifest 체크섬/도형을 깨거나 symlink artifact를 넣고 restore가 대상 변경 전에 거부되는지 확인합니다. | `VerifiedBackup` 신뢰 경계가 잘못된 입력을 거부한다는 근거를 제공합니다. |
| 4f8eb9aff842 | tests/runtime_stack.py | restore 실패 주입 / SIGINT | DB import 이후, archive extraction 이후, 초기화 단계 등 변경 뒤 named 실패 또는 signal을 주입합니다. | partial 대상 자원이 생길 수 있는 실제 되돌리기 path를 통과합니다. |
| 4f8eb9aff842 | tests/runtime_stack.py | zero-자원 검사문 | 컨테이너, volumes, networks를 labels와 expected names로 독립 조회해 모두 없어야 한다고 검사합니다. | `down` 호출 여부가 아니라 되돌리기 endpoint를 확인합니다. |
| 4f8eb9aff842 | tests/runtime_stack.py | success and second restore refusal | 복원된 post/option/upload/users/헬스 상태를 검사하고 같은 대상에 두 번째 restore가 기존 상태를 바꾸지 않고 거부되는지 확인합니다. | fresh-프로젝트 불변식의 positive/negative 양쪽을 검증합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 4f8eb9aff842^ 4f8eb9aff842 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 3a37a491ecea 4f8eb9aff842 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | verified 입력만 fresh 대상에 적용되며 실패는 zero-owned-자원 상태, success는 healthy restored stack으로 끝납니다. |
| 재현하는 실패 / 경계 | malformed/symlink 입력, DB/archive/초기화 이후 injected 실패, SIGINT, second restore입니다. |
| 테스트 방식 | live negative/positive 통합 + 결정적 실패/signal injection |
| 테스트 준비 코드와 실패 주입 | valid backup과 여러 corrupted copies, fresh secondary 프로젝트를 만들고 변경 stage별 실패를 주입합니다. |
| 실제 통과하는 실제 실행 경로 | CLI/Make restore→VerifiedBackup→freshness→스트림 import/extraction→초기화/헬스 상태 또는 cleanup_failed_restore를 통과합니다. |
| 핵심 검사문 | 변경 전 refusal, residual object 0, restored values/헬스 상태, second restore refusal와 existing-상태 보존을 확인합니다. |
| 이 테스트가 증명하는 것 | 주요 실패/cancellation이 all-or-nothing 자원 endpoint로 수렴하고 success가 실제 애플리케이션 상태를 복원함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 데몬 crash·hardware storage·모든 concurrent actor를 증명하지 않습니다. |
| 성격 | 결정적 되돌리기 and broad restore 통합 |
| 막는 후속 회귀 | partial 대상 leak, malformed 입력 after-변경 rejection, 제자리 second restore, 정리 명령 성공만 신뢰하는 회귀를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: Docker 데몬 crash, physical 볼륨 garbage, 모든 malformed tar format, uncooperative concurrent actor는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `030e7310c665`이 stopped/unlabelled collision과 large streaming restore를 보강합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: restore의 주요 all-or-nothing endpoint와 pre-existing 상태 보존을 실제 Docker 자원과 애플리케이션 data로 증명합니다.

### 8. `030e7310c665` — test(backup): 자원 충돌과 시그널 경계 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `PERSISTENCE`, `EDGE` |
| 원문에서 정한 역할 | 중지된 자원, 라벨 없는 충돌, 대용량 복원 입력을 추가로 검증했습니다. |
| 이전 커밋 | `4f8eb9aff842` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds signal-race checks, labelled and name-only restore-collision refusal, large filesystem and database fixtures, checksums, and stricter secondary cleanup reporting. -->
<!-- 원문 판단 근거: It tests boundary conditions that small normal-path fixtures and simple cancellation cannot cover, protecting the integrity and lifecycle guarantees of backup and restore. -->

- **이 개발 흐름의 재검토 관점:** 이 문서에서는 stopped/unlabelled collision refusal, large restored 테스트 준비 코드, secondary 정리 관점을 우선 기록합니다.

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `030e7310c665`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `restore resource collision fixtures`에서 freshness 검사가 label-only 또는 running-only로 축소되는 회귀를 탐지합니다.
- `tests/runtime_stack.py`의 `large restore fixture`에서 SQL/tar streaming injection의 large-입력 correctness를 검증합니다.
- `tests/runtime_stack.py`의 `secondary close/error propagation`에서 테스트가 만든 secondary 자원의 정리도 근거 lifecycle에 포함합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 030e7310c665 | tests/runtime_stack.py | restore 자원 collision 테스트 준비 코드 | 프로젝트 label은 없지만 expected exact name을 가진 stopped 컨테이너/볼륨/네트워크와 labelled custom-name object를 만듭니다. | freshness 검사가 label-only 또는 running-only로 축소되는 회귀를 탐지합니다. |
| 030e7310c665 | tests/runtime_stack.py | large restore 테스트 준비 코드 | backup의 32 MiB file과 4 MiB DB value를 fresh 대상에 restore한 뒤 원본/restored length와 digest를 비교합니다. | SQL/tar streaming injection의 large-입력 correctness를 검증합니다. |
| 030e7310c665 | tests/runtime_stack.py | secondary close/error propagation | restore 대상 teardown이 실패하면 primary scenario가 성공했더라도 nonzero로 처리되는 브랜치를 검사합니다. | 테스트가 만든 secondary 자원의 정리도 근거 lifecycle에 포함합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 030e7310c665^ 030e7310c665 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 4f8eb9aff842 030e7310c665 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | freshness check는 labelled/running object뿐 아니라 stopped/unlabelled exact-name object도 거부하고 large streams를 정확히 복원합니다. |
| 재현하는 실패 / 경계 | stopped/unlabelled collisions, 32 MiB file, 4 MiB DB value, secondary 정리 실패입니다. |
| 테스트 방식 | edge-경계 런타임 통합 + large 테스트 준비 코드 체크섬 comparison |
| 테스트 준비 코드와 실패 주입 | expected 자원 names으로 collision objects를 만들고 large 상태 backup을 fresh 대상에 적용합니다. |
| 실제 통과하는 실제 실행 경로 | 자원 discovery→freshness refusal 또는 VerifiedBackup→스트림 injection→애플리케이션 verification을 통과합니다. |
| 핵심 검사문 | pre-existing object 보존, 변경 0, 원본/restored digest·length, 정리-result precedence를 확인합니다. |
| 이 테스트가 증명하는 것 | label/name discovery가 보완적이고 streaming restore가 small 테스트 준비 코드에 국한되지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 새로운 미래 Compose naming form이나 모든 볼륨 driver를 증명하지 않습니다. |
| 성격 | collision and large-스트림 회귀 |
| 막는 후속 회귀 | label-only discovery, running-only lookup, whole-buffer/truncation, secondary 정리 무시를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 모든 Compose version naming convention과 무한 크기 입력을 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: 개발 흐름 4와 공유되는 커밋이지만 여기서는 restore collision/large-상태 관점만 사용합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: fresh-대상 inventory의 다중 discovery 방식과 bounded-memory restore가 realistic large 테스트 준비 코드에서 동작함을 증명합니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| restore는 완전히 비어 있는 대상 프로젝트에만 시작합니다. | e5cb60c7d743, 851dc1708881 | 9ca04b1c30cd | 4f8eb9aff842, 030e7310c665 | labels와 rendered/exact names preflight가 변경보다 앞서고 collision 테스트 준비 코드가 기존 object 보존을 확인합니다. |
| 입력 backup은 비공개, 소유자-controlled, non-symlink, exact-file, checksummed, structurally valid object입니다. | 953a0f6bd571 | 9ca04b1c30cd | 4f8eb9aff842 | `VerifiedBackup`의 디스크립터·잠금·스키마·체크섬 검증과 잘못된 형식·심볼릭 링크 거부를 연결합니다. |
| WordPress archive와 SQL은 empty newly created volumes에만 주입됩니다. | 1250fcf7c006 | 9ca04b1c30cd | 4f8eb9aff842 | empty 마운트 checks와 fresh 대상, second restore refusal가 merge/overwrite를 막습니다. |
| restore 성공은 healthy complete stack, 실패는 owned Docker 자원이 하나도 없는 상태입니다. | 9ca04b1c30cd | 4f8eb9aff842 | 030e7310c665 | success data/헬스 상태 검사문과 실패 labels/names zero enumeration 및 정리-result propagation이 양 endpoint를 고정합니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: 기존 프로젝트에 restore하면 pre-existing 상태와 attempt-created 상태를 구분할 수 없어 실패 되돌리기가 안전하게 삭제할 범위를 결정할 수 없었습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: label·exact/rendered name collision checks, 파일 디스크립터-anchored `VerifiedBackup`, empty-볼륨 streaming injection과 independent zero-자원 정리 verification이 fresh-create semantics를 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `4f8eb9aff842`의 malformed/signal/success/second-refusal scenarios와 `030e7310c665`의 stopped/unlabelled collision 및 large 테스트 준비 코드가 보호합니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| 기존 대상에 restore하면 unrelated 상태와 merge/overwrite되고 되돌리기 소유권 불명확 | 851dc1708881 fresh-프로젝트 precondition | 4f8eb9aff842 second restore refusal, 030e7310c665 collision cases | new attempt가 만든 자원만 독점 소유하도록 namespace를 비웁니다. |
| 검증 뒤 입력 path가 symlink/modified file로 바뀜 | 953a0f6bd571 파일 디스크립터-anchored no-follow open과 retained locks/streams | 4f8eb9aff842 symlink artifact rejection | pathname 재해석 대신 stable open object를 끝까지 소비합니다. |
| DB import 뒤 초기화 실패로 partial 자원이 남음 | 9ca04b1c30cd compose 정리 + independent enumeration | 4f8eb9aff842 injected 실패/SIGINT 정리 | 정리 명령 실행과 zero-자원 검증을 분리합니다. |
| small archive/dump에서만 streaming이 통과 | 1250fcf7c006 stdin streaming primitives | 030e7310c665 large restored 체크섬/length | bounded-memory path와 byte completeness를 실제 large 테스트 준비 코드로 확인합니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Verified backup 입력 | user path 문자열 | opened directory/file descriptors와 shared 잠금이 stable 입력 소유 | 953a0f6bd571 no-follow/fstat/잠금 | restore 종료까지 path를 다시 신뢰하지 않습니다. |
| Target namespace | 존재 여부 불명 | freshness 이후 restore attempt가 새 자원 독점 소유 | 851dc1708881 discovery와 9ca04b1c30cd transaction | 실패 시 제거 가능한 소유권이 명확합니다. |
| MariaDB 볼륨 | 없음 | fresh 초기화 후 SQL 스트림의 authoritative relational 상태 | 1250fcf7c006 import order | 기존 DB와 merge하지 않습니다. |
| WordPress data/config volumes | 없음 | validated archive가 empty mounts에만 extraction | 1250fcf7c006 roots/emptiness/tar | 공개 data와 비공개 config를 분리 복원합니다. |
| Rollback | 단일 down 명령 시도 가능 | Compose 정리 + independent zero-자원 verification | 9ca04b1c30cd cleanup_failed_restore | 정리 실패도 transaction 실패입니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: Restore is treated as creation of a new project, not as an in-place overwrite. That constraint makes rollback tractable: verified input is applied only after collision checks, and any 실패 removes the 자원 created by the attempt. The later 테스트 show that refusal preserves pre-existing objects and that the streaming implementation remains correct beyond small fixtures. -->
- 최종 authoritative 상태와 소유자: VerifiedBackup descriptors가 stable 원본 입력을, successful fresh Compose 프로젝트의 named volumes가 restored authoritative 상태를 소유합니다.
- 정상 실행의 entry point와 완료 조건: 잠금 아래 입력/freshness 검증, DB 초기화/import, WordPress extraction, normal 초기화/start, 헬스 상태가 모두 성공하면 완료입니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: 예외/signal은 down --volumes와 independent labels/names enumeration으로 zero-owned-자원 상태를 요구하며 정리 실패는 primary error와 함께 보고합니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: credential rotation과 operations 테스트가 사용할 완전한 fresh 프로젝트 생성 semantics를 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: 데몬 crash와 physical storage leak, non-cooperating external actor는 이 개발 흐름 단독으로 완전 증명하지 않습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | 입력 검증 | 953a0f6bd571 VerifiedBackup | 파일 디스크립터/잠금/schema/체크섬/archive validation을 변경 전에 끝냅니다. | unsafe 입력은 대상 object 0 상태로 거부합니다. |
| 2 | 대상 freshness | 851dc1708881 ensure_fresh_project | labels와 exact/rendered names가 모두 비었는지 확인합니다. | collision이면 기존 object를 보존하고 실패합니다. |
| 3 | MariaDB 초기화/import | 1250fcf7c006 + 9ca04b1c30cd | fresh DB 자원을 만들고 SQL 스트림을 주입합니다. | import 실패는 되돌리기로 갑니다. |
| 4 | WordPress extraction | 1250fcf7c006 extraction helper | empty data/config mounts에 validated tar 스트림을 풉니다. | non-empty/path/tar 실패는 되돌리기로 갑니다. |
| 5 | 애플리케이션 convergence | 9ca04b1c30cd normal startup reuse | WordPress 초기화와 헬스 상태-gated services를 시작합니다. | 초기화/헬스 상태 실패는 created 자원을 제거합니다. |
| 6 | 되돌리기 | 9ca04b1c30cd cleanup_failed_restore | Compose 정리 후 independent enumeration으로 0을 증명합니다. | residual object나 정리 명령 실패는 별도 오류입니다. |
| 7 | success/second refusal | 4f8eb9aff842 런타임 scenario | restored values/헬스 상태를 확인하고 같은 대상 재실행을 거부합니다. | second restore는 existing 상태를 변경하지 않습니다. |

### 학습자의 최종 설명

> restore를 기존 볼륨에 덮는 작업으로 보면 실패 되돌리기의 소유권이 불명확해집니다. 이 구현은 대상 프로젝트가 labels와 exact/rendered names 모두에서 비어 있어야만 시작합니다. backup은 directory/file 파일 디스크립터에 anchored된 `VerifiedBackup`으로 열어 체크섬과 archive 구조를 변경 전에 끝내고, SQL과 tar 스트림은 새 empty volumes에만 주입합니다. 이후 normal 초기화와 헬스 상태를 재사용합니다. 어느 지점에서 실패하거나 signal이 오면 Compose 정리를 실행한 뒤 별도로 컨테이너/volumes/networks를 다시 열거해 0을 확인합니다. 성공은 healthy restored stack, 실패는 attempt-owned 자원 0이라는 두 endpoint로 정의되며, 같은 대상의 두 번째 restore는 제자리 merge 대신 거부됩니다.

## 학습 완료 자가 점검

- [x] restore를 제자리 overwrite로 설명하지 않았습니까?
- [x] labelled 자원만 collision으로 본다고 잘못 기록하지 않았습니까?
- [x] backup 검증 뒤 파일을 pathname으로 다시 열어도 안전하다고 가정하지 않았습니까?
- [x] 정리를 시도한 것과 정리 완료를 검증한 것을 구분했습니까?
- [x] successful restore 뒤 같은 대상에 재실행이 허용된다고 쓰지 않았습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 05-fresh-project-restore.md =====

===== BEGIN FILE: 06-credential-rotation-and-compensation.md =====
# 개발 흐름 6 — 자격 증명 동시 교체와 실패 보상

## 개발 흐름의 목표

host 비밀값 files, MariaDB accounts, WordPress users, `wp-config.php`에 분산된 credential set을 ordered transition으로 교체하고, post-write 실패와 signal interruption에도 이전 verified 상태로 보상하는 multi-store transaction을 추적합니다.

### 원문에서 정한 의의

자격 증명은 호스트 비밀 파일 네 개, MariaDB 계정 두 개, WordPress 사용자 두 명과 설정에 걸쳐 존재합니다. 개별 변경 기능에서 출발해 전체 상태 전이를 검증하고, 일부 명령이 실패 전에 상태를 바꿀 수 있으므로 보상 절차를 추가합니다. 복구 중에는 추가 종료 신호 처리를 미뤄 보상 자체가 중단되지 않게 합니다.

<details>
<summary>영문 원문</summary>

> Credentials are represented in four host files, two MariaDB accounts, two WordPress users, and WordPress configuration. The thread therefore evolves from individual mutation primitives to a verified state transition and then to compensation for commands that may change state before failing. Deferring further termination while rollback is active is the key correction that prevents recovery itself from being interrupted.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- 하나의 logical credential generation이 실제로 어느 저장소와 consumer에 분산됩니까?
- 명령 실패가 no 변경을 뜻하지 않는 post-write ambiguity는 어떤 helper/테스트 hook으로 재현됩니까?
- 애플리케이션 password를 root password보다 먼저 바꾸는 이유와 되돌리기 순서는 무엇입니까?
- 새 credential의 성공뿐 아니라 이전 credential의 실패를 확인해야 하는 이유는 무엇입니까?
- host file 반영이 per-file atomic이어도 전체 네 파일이 transaction이 아닌 이유는 무엇입니까?
- 첫 signal과 되돌리기 중 추가 signal을 다르게 처리하는 상태 전이는 어디에 구현됩니까?

## 완료 기준

- 네 host files, MariaDB root/애플리케이션 accounts, WordPress admin/author users, 비공개 config의 관계를 그렸습니다.
- 각 변경 primitive의 stdin/no-argument 비밀값 경계와 atomic file 반영을 확인했습니다.
- forward rotation ordering과 compensation ordering을 actual call 순서로 비교했습니다.
- positive/negative authentication probes가 actual 상태를 authoritative하게 판단하는 방식을 기록했습니다.
- 각 실패 주입과 double-signal scenario가 어떤 mixed 상태를 재현하는지 구분했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `a2d20b8c2c03` | feat(secrets): 교체 비밀 파일을 안전하게 읽고 게시 | **A** | `SECRETS`<br>`RISK`<br>`OPERATIONS` | 안전한 교체 입력과 호스트 파일의 원자적 반영을 확립했습니다. |
| 2 | `832d182743ea` | feat(secrets): MariaDB 계정 비밀번호 원자 교체 | **A** | `SECRETS`<br>`RISK`<br>`INTEGRATION` | MariaDB 애플리케이션·root 자격 증명 교체를 구현했습니다. |
| 3 | `0aa998fdd344` | feat(secrets): WordPress 설정과 사용자 비밀번호 교체 | **A** | `SECRETS`<br>`RISK`<br>`INTEGRATION` | WordPress 설정·사용자 자격 증명 교체를 구현했습니다. |
| 4 | `64844c583211` | feat(secrets): 신규 자격증명 수용과 기존 값 거부 검증 | **A** | `TEST`<br>`SECRETS`<br>`RISK` | 새 자격 증명은 성공하고 기존 값은 실패해야 한다는 검증 조건을 추가했습니다. |
| 5 | `c68486d55f30` | feat(secrets): 회전 실패 시 기존 자격증명 복구 | **A** | `SECRETS`<br>`RECOVERY`<br>`HARD` | 여러 저장소의 상태를 이전 검증값으로 되돌리는 기능을 추가했습니다. |
| 6 | `9934b478c79a` | feat(secrets): 스택 자격증명 회전 절차 연결 | **S** | `SECRETS`<br>`RECOVERY`<br>`CORE` | 순서·잠금·검증이 정해진 자격 증명 교체 작업을 연결했습니다. |
| 7 | `2e6649a7706d` | fix(secrets): 회전 중단과 불명확한 상태를 보상 | **S** | `SECRETS`<br>`RECOVERY`<br>`HARD` | 쓰기 뒤 결과가 불명확한 실패를 보상하고 되돌리기 중 시그널 처리를 미뤘습니다. |
| 8 | `0da35c72add5` | test(secrets): 회전 롤백과 재시도 검증 | **A** | `TEST`<br>`SECRETS`<br>`RECOVERY` | 정상 교체, 실패 주입, 중단, 되돌리기, 누수, 재시도를 검증했습니다. |
| 9 | `2557079c2d19` | test(secrets): 회전 후 런타임 비밀 경계 고정 | **B** | `TEST`<br>`SECRETS` | 테스트가 정상 실행 중의 비밀값 경계를 약화하지 못하게 했습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `a2d20b8c2c03` — feat(secrets): 교체 비밀 파일을 안전하게 읽고 게시

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `SECRETS`, `RISK`, `OPERATIONS` |
| 원문에서 정한 역할 | 안전한 교체 입력과 호스트 파일의 원자적 반영을 확립했습니다. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `832d182743ea` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds hardened replacement-secret reads and per-file atomic, durable host-secret publication. -->
<!-- 원문 판단 근거: This establishes the host filesystem side of credential rotation and prevents partial individual files or unsafe input types from entering a multi-system state transition. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `a2d20b8c2c03`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `read replacement/active secret`에서 rotation 원본과 current generation의 host-side trust 경계가 같습니다.
- `tools/rotate_secrets.py`의 `publish_secret_file`에서 개별 file은 torn write 없이 한 번에 새 content로 보입니다.
- `tools/rotate_secrets.py`의 `temporary cleanup`에서 per-file 반영 실패가 stray credential file을 남기지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| a2d20b8c2c03 | tools/rotate_secrets.py | read replacement/active 비밀값 | incoming과 active 비밀값을 no-follow 파일 디스크립터로 열고 regular file, single link, current 소유자, `0600`, bounded single-line/password-도형을 검사합니다. | rotation 원본과 current generation의 host-side trust 경계가 같습니다. |
| a2d20b8c2c03 | tools/rotate_secrets.py | publish_secret_file | 대상과 같은 directory에 비공개 임시 객체 file을 만들고 write·flush·fsync한 뒤 `os.replace`하고 parent directory를 fsync합니다. | 개별 file은 torn write 없이 한 번에 새 content로 보입니다. |
| a2d20b8c2c03 | tools/rotate_secrets.py | 임시 객체 정리 | write/sync/replace 실패마다 아직 남은 임시 객체 file을 unlink하고 original 대상을 보존합니다. | per-file 반영 실패가 stray credential file을 남기지 않습니다. |

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | replacement value와 active host file을 일반 pathname/read/write로 다루면 symlink·permission·partial write·existing temp 위험이 있었습니다. |
| 선택한 경계 / 결정 | 입력과 current file 모두 hardened read를 사용하고 각 대상은 same-directory atomic 영속 replace로 게시했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/rotate_secrets.py`의 `read replacement/active secret`; `tools/rotate_secrets.py`의 `publish_secret_file`; `tools/rotate_secrets.py`의 `temporary cleanup` |
| 상태·소유권·수명 변화 | helper가 한 비밀값 file의 temp/파일 디스크립터/replace 수명을 소유합니다. orchestrator는 네 file의 logical generation ordering을 별도로 소유해야 합니다. |
| 주요 실패 브랜치 | unsafe 입력/대상, write/fsync/replace/parent-sync 실패는 해당 file 반영을 실패시키고 temp를 정리합니다. |
| 이 커밋의 보장 | 각 host 비밀값 file이 비공개 regular single-link이며 개별 replacement가 atomic/영속하다는 것을 보장합니다. |
| 한계와 다음 관련 커밋 | 네 files와 DB/WordPress stores 전체가 한 번에 바뀌는 global transaction은 제공하지 않습니다. `9934b478c79a`가 per-file primitive를 ordered multi-store transition에 넣고 `2e6649a7706d`가 post-write ambiguity를 보상합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 네 files와 DB/WordPress stores 전체가 한 번에 바뀌는 global transaction은 제공하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `9934b478c79a`가 per-file primitive를 ordered multi-store transition에 넣고 `2e6649a7706d`가 post-write ambiguity를 보상합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 각 host 비밀값 file이 비공개 regular single-link이며 개별 replacement가 atomic/영속하다는 것을 보장합니다.

### 2. `832d182743ea` — feat(secrets): MariaDB 계정 비밀번호 원자 교체

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `SECRETS`, `RISK`, `INTEGRATION` |
| 원문에서 정한 역할 | MariaDB 애플리케이션·root 자격 증명 교체를 구현했습니다. |
| 이전 커밋 | `a2d20b8c2c03` |
| 다음 커밋 | `0aa998fdd344` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds root-authenticated MariaDB SQL execution and coordinated application/root password changes through private option files. -->
<!-- 원문 판단 근거: This implements a high-risk part of credential rotation while keeping credentials out of process arguments and preserving SQL literal correctness. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `832d182743ea`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `change_application_db_password`에서 공개 TCP/argv에 새 password를 노출하지 않고 DB-owned account 상태를 변경합니다.
- `tools/rotate_secrets.py`의 `change_root_db_password`에서 root credential 변경 시점과 애플리케이션 변경 시점을 orchestrator가 분리할 수 있습니다.
- `tools/rotate_secrets.py`의 `bounded subprocess / cleanup`에서 변경 명령 lifecycle과 비밀값-bearing 임시 객체 상태를 제한합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 832d182743ea | tools/rotate_secrets.py | change_application_db_password | local MariaDB 소켓에서 current privileged credential로 애플리케이션 account의 password를 바꾸고 identifier/literal을 안전하게 구성합니다. | 공개 TCP/argv에 새 password를 노출하지 않고 DB-owned account 상태를 변경합니다. |
| 832d182743ea | tools/rotate_secrets.py | change_root_db_password | root account 변경을 별도 primitive로 두고 임시 객체 비공개 클라이언트 options/stdin을 사용해 password가 프로세스 argument에 나타나지 않게 합니다. | root credential 변경 시점과 애플리케이션 변경 시점을 orchestrator가 분리할 수 있습니다. |
| 832d182743ea | tools/rotate_secrets.py | bounded subprocess / 정리 | DB 명령 timeout/nonzero와 임시 객체 option file 정리를 domain error로 처리합니다. | 변경 명령 lifecycle과 비밀값-bearing 임시 객체 상태를 제한합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 832d182743ea^ 832d182743ea -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff a2d20b8c2c03 832d182743ea -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | host files만 바꾸면 MariaDB가 저장한 애플리케이션/root password와 consumer file이 불일치합니다. |
| 선택한 경계 / 결정 | 애플리케이션 account와 root account를 별도 local-소켓 변경 primitive로 만들고 비밀값을 argv가 아닌 비공개 입력으로 전달했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/rotate_secrets.py`의 `change_application_db_password`; `tools/rotate_secrets.py`의 `change_root_db_password`; `tools/rotate_secrets.py`의 `bounded subprocess / cleanup` |
| 상태·소유권·수명 변화 | MariaDB가 실제 authentication 상태를 소유하며 host helper는 명령 실행과 임시 객체 credential 재질을 잠시 소유합니다. |
| 주요 실패 브랜치 | 클라이언트 nonzero/timeout이 변경 전인지 후인지이 커밋만으로 구분되지 않습니다. 명령이 write 후 실패 status를 낼 수 있습니다. |
| 이 커밋의 보장 | DB 애플리케이션/root credentials를 순서 제어 가능한 primitive로 교체하고 프로세스 arguments에 password를 넣지 않습니다. |
| 한계와 다음 관련 커밋 | WordPress config/users, host files, new/old authentication verification, ambiguous 명령 result compensation은 보장하지 않습니다. `64844c583211`이 positive/negative probes를, `9934b478c79a`가 애플리케이션-first/root-last ordering을 적용합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: WordPress config/users, host files, new/old authentication verification, ambiguous 명령 result compensation은 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `64844c583211`이 positive/negative probes를, `9934b478c79a`가 애플리케이션-first/root-last ordering을 적용합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: DB 애플리케이션/root credentials를 순서 제어 가능한 primitive로 교체하고 프로세스 arguments에 password를 넣지 않습니다.

### 3. `0aa998fdd344` — feat(secrets): WordPress 설정과 사용자 비밀번호 교체

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `SECRETS`, `RISK`, `INTEGRATION` |
| 원문에서 정한 역할 | WordPress 설정·사용자 자격 증명 교체를 구현했습니다. |
| 이전 커밋 | `832d182743ea` |
| 다음 커밋 | `64844c583211` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds atomic `wp-config.php` DB-password replacement and WordPress administrator/author password changes. -->
<!-- 원문 판단 근거: The commit coordinates filesystem configuration and application database state, establishing the WordPress side of the cross-subsystem rotation problem. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `0aa998fdd344`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `replace_wp_config_password`에서 WordPress DB consumer config를 partial write 없이 갱신합니다.
- `tools/rotate_secrets.py`의 `set_wordpress_user_password`에서 해시를 직접 조작하지 않고 WordPress가 user password 상태를 소유합니다.
- `tools/rotate_secrets.py`의 `admin/author primitives`에서 두 user 중 일부만 바뀌는 stage를 orchestrator와 테스트가 식별할 수 있습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 0aa998fdd344 | tools/rotate_secrets.py | replace_wp_config_password | 비공개 config file이 regular/비공개이며 DB password define이 정확히 한 번 존재하는지 검사하고 same-filesystem 임시 객체+replace로 값을 교체합니다. | WordPress DB consumer config를 partial write 없이 갱신합니다. |
| 0aa998fdd344 | tools/rotate_secrets.py | set_wordpress_user_password | WordPress 런타임에서 애플리케이션 API `wp_set_password`를 호출하는 PHP/WP 명령에 user/password 전달값을 stdin으로 전달합니다. | 해시를 직접 조작하지 않고 WordPress가 user password 상태를 소유합니다. |
| 0aa998fdd344 | tools/rotate_secrets.py | admin/author primitives | admin과 author account를 독립 변경/verification 대상으로 유지합니다. | 두 user 중 일부만 바뀌는 stage를 orchestrator와 테스트가 식별할 수 있습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 0aa998fdd344^ 0aa998fdd344 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 832d182743ea 0aa998fdd344 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | DB account가 바뀌어도 `wp-config.php`와 WordPress admin/author credential이 old generation이면 stack 전체의 logical credential set은 일치하지 않습니다. |
| 선택한 경계 / 결정 | 비공개 config와 두 애플리케이션 users를 WordPress-owned 인터페이스와 atomic file replacement를 통해 바꾸는 primitive를 추가했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/rotate_secrets.py`의 `replace_wp_config_password`; `tools/rotate_secrets.py`의 `set_wordpress_user_password`; `tools/rotate_secrets.py`의 `admin/author primitives` |
| 상태·소유권·수명 변화 | WordPress DB가 user hashes를, 비공개 config 볼륨이 DB consumer credential을 소유합니다. helper는 변경 명령과 config temp file만 일시 소유합니다. |
| 주요 실패 브랜치 | config format/duplicate define, WordPress 명령 timeout/nonzero, 임시 객체 반영 실패가 발생할 수 있고 명령 nonzero가 no 변경을 뜻하지는 않습니다. |
| 이 커밋의 보장 | WordPress config, admin, author credential을 프로세스 argument 노출 없이 별도 단계로 교체할 수 있습니다. |
| 한계와 다음 관련 커밋 | DB accounts/host files와의 global ordering·되돌리기·old value rejection은 보장하지 않습니다. `64844c583211`이 actual consumer probes를 만들고 `9934b478c79a`가 전체 ordering에 연결합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: DB accounts/host files와의 global ordering·되돌리기·old value rejection은 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `64844c583211`이 actual consumer probes를 만들고 `9934b478c79a`가 전체 ordering에 연결합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: WordPress config, admin, author credential을 프로세스 argument 노출 없이 별도 단계로 교체할 수 있습니다.

### 4. `64844c583211` — feat(secrets): 신규 자격증명 수용과 기존 값 거부 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `SECRETS`, `RISK` |
| 원문에서 정한 역할 | 새 자격 증명은 성공하고 기존 값은 실패해야 한다는 검증 조건을 추가했습니다. |
| 이전 커밋 | `0aa998fdd344` |
| 다음 커밋 | `c68486d55f30` |

#### 원문이 확정한 범위

<!-- 원문 요약: Verifies new credentials work, old credentials fail, configuration matches, and no accepted or rejected value leaks into runtime metadata. -->
<!-- 원문 판단 근거: Successful rotation requires both positive and negative authentication evidence; this commit makes that state transition verifiable rather than inferred from command success. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `64844c583211`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `MariaDB auth probes`에서 명령 exit status가 아니라 실제 DB consumer 인터페이스가 authoritative 상태를 판단합니다.
- `tools/rotate_secrets.py`의 `WordPress user probes`에서 새 값만 성공하는 one-generation 상태를 요구합니다.
- `tools/rotate_secrets.py`의 `config/file equality checks`에서 런타임 store와 host 원본의 generation 일치를 함께 봅니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 64844c583211 | tools/rotate_secrets.py | MariaDB auth probes | root/애플리케이션 각각 new credential로 local/서비스 authentication이 성공하고 old credential은 실패해야 한다고 검사합니다. | 명령 exit status가 아니라 실제 DB consumer 인터페이스가 authoritative 상태를 판단합니다. |
| 64844c583211 | tools/rotate_secrets.py | WordPress user probes | admin/author의 new password가 WordPress authentication API에서 맞고 old password가 틀린지 양방향으로 확인합니다. | 새 값만 성공하는 one-generation 상태를 요구합니다. |
| 64844c583211 | tools/rotate_secrets.py | config/file equality checks | 비공개 config와 active host 비밀값 files가 expected generation의 exact values인지 확인합니다. | 런타임 store와 host 원본의 generation 일치를 함께 봅니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 64844c583211^ 64844c583211 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 0aa998fdd344 64844c583211 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 변경 명령이 성공해도 old credential이 alias/다른 account scope로 계속 작동하거나 일부 consumer가 old file을 사용할 수 있었습니다. |
| 선택한 경계 / 결정 | 각 store를 실제 인증/consumer 인터페이스로 positive(new works)와 negative(old fails) 양쪽에서 확인했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/rotate_secrets.py`의 `MariaDB auth probes`; `tools/rotate_secrets.py`의 `WordPress user probes`; `tools/rotate_secrets.py`의 `config/file equality checks` |
| 상태·소유권·수명 변화 | verification layer가 actual 상태를 authoritative하게 판정하고 orchestrator는 성공 결정을 내리기 전에 모든 probe를 통과해야 합니다. |
| 주요 실패 브랜치 | new 실패 또는 old success 하나라도 mixed/ambiguous generation으로 간주해 rotation을 실패시킵니다. |
| 이 커밋의 보장 | 성공 판정은 replacement가 작동한다는 것뿐 아니라 previous generation이 거부되고 files/config가 일치한다는 것까지 포함합니다. |
| 한계와 다음 관련 커밋 | 실패 시 prior 상태로 되돌리는 compensation은 아직 없습니다. `c68486d55f30`이 이 probes를 되돌리기 완료 판단에도 사용합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 실패 시 prior 상태로 되돌리는 compensation은 아직 없습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `c68486d55f30`이 이 probes를 되돌리기 완료 판단에도 사용합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 성공 판정은 replacement가 작동한다는 것뿐 아니라 previous generation이 거부되고 files/config가 일치한다는 것까지 포함합니다.

### 5. `c68486d55f30` — feat(secrets): 회전 실패 시 기존 자격증명 복구

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `SECRETS`, `RECOVERY`, `HARD` |
| 원문에서 정한 역할 | 여러 저장소의 상태를 이전 검증값으로 되돌리는 기능을 추가했습니다. |
| 이전 커밋 | `64844c583211` |
| 다음 커밋 | `9934b478c79a` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds compensation that restores database accounts, WordPress configuration and users, host files, and a verified running stack after rotation failure. -->
<!-- 원문 판단 근거: This is significant multi-store rollback engineering, though the following correction handles additional ambiguous command and signal states not yet covered here. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `c68486d55f30`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `compensation helpers`에서 partial new generation을 old verified generation으로 수렴시키는 경로가 생깁니다.
- `tools/rotate_secrets.py`의 `rollback verification`에서 되돌리기 명령 호출만으로 완료를 간주하지 않습니다.
- `tools/rotate_secrets.py`의 `rollback error aggregation`에서 첫 되돌리기 오류가 이후 repair 시도를 중단하거나 원인을 덮지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| c68486d55f30 | tools/rotate_secrets.py | compensation helpers | forward stage별 변경을 추적하고 실패 뒤 WordPress users/config, MariaDB accounts, host files를 previous values로 되돌리는 reverse operations를 정의합니다. | partial new generation을 old verified generation으로 수렴시키는 경로가 생깁니다. |
| c68486d55f30 | tools/rotate_secrets.py | 되돌리기 verification | old values가 다시 작동하고 new values가 거부되며 files/config가 old generation과 같은지 probes로 확인합니다. | 되돌리기 명령 호출만으로 완료를 간주하지 않습니다. |
| c68486d55f30 | tools/rotate_secrets.py | 되돌리기 error aggregation | 여러 compensation 실패를 모아 primary rotation 실패와 함께 보고합니다. | 첫 되돌리기 오류가 이후 repair 시도를 중단하거나 원인을 덮지 않습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff c68486d55f30^ c68486d55f30 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 64844c583211 c68486d55f30 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 일부 store가 new generation으로 바뀐 뒤 다음 step이 실패하면 host files, DB, WordPress가 mixed 상태에 남았습니다. |
| 선택한 경계 / 결정 | forward 변경을 추적하고 reverse compensation과 prior-generation verification을 추가했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/rotate_secrets.py`의 `compensation helpers`; `tools/rotate_secrets.py`의 `rollback verification`; `tools/rotate_secrets.py`의 `rollback error aggregation` |
| 상태·소유권·수명 변화 | orchestrator가 logical generation 전환 책임을 가지며 각 구성 요소 helper는 reversible 변경만 수행합니다. |
| 주요 실패 브랜치 | 되돌리기 step 자체가 실패할 수 있으므로 all-or-nothing을 선언하지 않고 incomplete compensation을 명시적으로 보고합니다. |
| 이 커밋의 보장 | ordinary 실패 뒤 가능한 한 previous verified generation으로 되돌리고 실제 old/new probes로 결과를 판정합니다. |
| 한계와 다음 관련 커밋 | post-write 명령 실패의 실제 변경 여부와 되돌리기 중 signal interruption은 아직 충분히 처리하지 않습니다. `2e6649a7706d`이 ambiguous post-write 실패와 되돌리기-active signal 상태를 교정합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: post-write 명령 실패의 실제 변경 여부와 되돌리기 중 signal interruption은 아직 충분히 처리하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `2e6649a7706d`이 ambiguous post-write 실패와 되돌리기-active signal 상태를 교정합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: ordinary 실패 뒤 가능한 한 previous verified generation으로 되돌리고 실제 old/new probes로 결과를 판정합니다.

### 6. `9934b478c79a` — feat(secrets): 스택 자격증명 회전 절차 연결

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **S** |
| 태그 | `SECRETS`, `RECOVERY`, `CORE` |
| 원문에서 정한 역할 | 순서·잠금·검증이 정해진 자격 증명 교체 작업을 연결했습니다. |
| 이전 커밋 | `c68486d55f30` |
| 다음 커밋 | `2e6649a7706d` |

#### 원문이 확정한 범위

<!-- 원문 요약: Coordinates the complete credential rotation sequence, serializes it, recreates services, verifies new values and rejection of old ones, and invokes compensation on failure. -->
<!-- 원문 판단 근거: Credential rotation is a defining management mechanism spanning four host files, MariaDB accounts, WordPress users, and application configuration; this commit establishes that transaction. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `9934b478c79a`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `rotate / project_operation_lock`에서 startup/backup/restore와 rotation이 같은 serialization domain을 공유합니다.
- `tools/rotate_secrets.py`의 `forward order`에서 repair에 필요한 privileged/current path를 너무 일찍 끊지 않도록 root와 host 반영을 뒤에 둡니다.
- `tools/rotate_secrets.py`의 `force recreate / end verification`에서 새 프로세스가 new generation을 실제로 소비하는지 검증합니다.
- `tools/rotate_secrets.py`의 `compensating failure path`에서 multi-store transition의 실패 endpoint를 정의합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 9934b478c79a | tools/rotate_secrets.py | rotate / project_operation_lock | replacement/active secrets와 current 런타임 상태를 검증하고 프로젝트 잠금을 잡은 채 전체 transition을 실행합니다. | startup/backup/restore와 rotation이 같은 serialization domain을 공유합니다. |
| 9934b478c79a | tools/rotate_secrets.py | forward order | 공개 요청을 닫기 위해 Nginx를 멈춘 뒤 WordPress users와 비공개 config, MariaDB 애플리케이션 password, MariaDB root password, host files 순으로 전환합니다. | repair에 필요한 privileged/current path를 너무 일찍 끊지 않도록 root와 host 반영을 뒤에 둡니다. |
| 9934b478c79a | tools/rotate_secrets.py | force recreate / end verification | active files 게시 뒤 services를 force-recreate하고 new works/old fails/files match/런타임 경계/no leak를 확인합니다. | 새 프로세스가 new generation을 실제로 소비하는지 검증합니다. |
| 9934b478c79a | tools/rotate_secrets.py | compensating 실패 경로 | 어느 stage 실패든 previous generation compensation과 서비스 recovery를 시도하고 incomplete 되돌리기를 surfaced합니다. | multi-store transition의 실패 endpoint를 정의합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 9934b478c79a^ 9934b478c79a -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff c68486d55f30 9934b478c79a -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 S 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 이 커밋 직전 상태 | 개별 host/DB/WordPress primitives가 있어도 caller별 ordering이 다르면 root access가 먼저 끊기거나 공개 requests가 mixed generation을 관측할 수 있었습니다. |
| 해결하려던 문제 | stage 실패는 compensation으로 이동합니다. 그러나 subprocess가 상태를 바꾼 뒤 nonzero를 반환하면 tracked stage만으로 actual 변경을 놓칠 수 있고 signal이 되돌리기를 중단할 수 있습니다. |
| 기존 설계가 충분하지 않았던 이유 | 개별 host/DB/WordPress primitives가 있어도 caller별 ordering이 다르면 root access가 먼저 끊기거나 공개 requests가 mixed generation을 관측할 수 있었습니다. stage 실패는 compensation으로 이동합니다. 그러나 subprocess가 상태를 바꾼 뒤 nonzero를 반환하면 tracked stage만으로 actual 변경을 놓칠 수 있고 signal이 되돌리기를 중단할 수 있습니다. |
| 핵심 결정 | same-프로젝트 잠금, pre-verification, Nginx quiescence, repair-friendly forward order, host 반영, recreate, global probes, compensation을 하나의 procedure로 고정했습니다. |
| 주요 caller → callee / producer → consumer | `tools/rotate_secrets.py`의 `rotate / project_operation_lock`; `tools/rotate_secrets.py`의 `forward order`; `tools/rotate_secrets.py`의 `force recreate / end verification`; `tools/rotate_secrets.py`의 `compensating failure path` |
| authoritative 상태와 반영 경계 | rotation orchestrator가 logical credential generation과 서비스 lifecycle을 소유합니다. 각 store는 자신의 표현을 소유하되 transaction success/실패 판정은 orchestrator가 합니다. ordinary 성공 path에서 네 host files, 두 DB accounts, 두 WP users, 비공개 config가 한 new generation으로 전환되고 services가 이를 소비하며 old values가 거부됩니다. |
| 소유권 / 수명 / responsibility 변화 | rotation orchestrator가 logical credential generation과 서비스 lifecycle을 소유합니다. 각 store는 자신의 표현을 소유하되 transaction success/실패 판정은 orchestrator가 합니다. |
| 실패 상황과 recovery path | stage 실패는 compensation으로 이동합니다. 그러나 subprocess가 상태를 바꾼 뒤 nonzero를 반환하면 tracked stage만으로 actual 변경을 놓칠 수 있고 signal이 되돌리기를 중단할 수 있습니다. |
| 이 커밋이 보장하는 것 | ordinary 성공 path에서 네 host files, 두 DB accounts, 두 WP users, 비공개 config가 한 new generation으로 전환되고 services가 이를 소비하며 old values가 거부됩니다. |
| 아직 보장하지 않는 것 | distributed stores 전체의 진정한 atomic 커밋은 아니며 ambiguous write/result와 되돌리기-interruption 위험이 남습니다. |
| 후속 fix / 테스트와 연결 | `2e6649a7706d`이 이 두 핵심 실패 assumption을 수정하고 `0da35c72add5`가 stage matrix/double signal/retry를 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: distributed stores 전체의 진정한 atomic 커밋은 아니며 ambiguous write/result와 되돌리기-interruption 위험이 남습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `2e6649a7706d`이 이 두 핵심 실패 assumption을 수정하고 `0da35c72add5`가 stage matrix/double signal/retry를 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: ordinary 성공 path에서 네 host files, 두 DB accounts, 두 WP users, 비공개 config가 한 new generation으로 전환되고 services가 이를 소비하며 old values가 거부됩니다.

### 7. `2e6649a7706d` — fix(secrets): 회전 중단과 불명확한 상태를 보상

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **S** |
| 태그 | `SECRETS`, `RECOVERY`, `HARD` |
| 원문에서 정한 역할 | 쓰기 뒤 결과가 불명확한 실패를 보상하고 되돌리기 중 시그널 처리를 미뤘습니다. |
| 이전 커밋 | `9934b478c79a` |
| 다음 커밋 | `0da35c72add5` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds stage-level failure injection, interruption handling, ambiguous post-write compensation, and deferred signals during rollback. -->
<!-- 원문 판단 근거: This corrects non-obvious partial-state hazards in the rotation transaction. It is essential to explaining how the project prevents operator cancellation or uncertain command outcomes from interrupting recovery itself. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `2e6649a7706d`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/rotate_secrets.py`의 `post-write failure hooks`에서 “nonzero/exception이면 변경 없음”이라는 잘못된 가정을 노출합니다.
- `tools/rotate_secrets.py`의 `actual-state probes before compensation`에서 보상 대상을 observed 상태에서 계산합니다.
- `tools/rotate_secrets.py`의 `rotation signal state machine`에서 recovery 자체가 두 번째 operator signal로 끊기는 것을 막습니다.
- `tools/rotate_secrets.py`의 `deferred signal/result reporting`에서 복구 완료와 프로세스 exit semantics를 분리합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2e6649a7706d | tools/rotate_secrets.py | post-write 실패 hooks | 각 영속 변경 직후 테스트 hook이 실패를 발생시켜 명령이 상태를 바꿨지만 caller가 실패를 받은 상황을 재현합니다. | “nonzero/exception이면 변경 없음”이라는 잘못된 가정을 노출합니다. |
| 2e6649a7706d | tools/rotate_secrets.py | actual-상태 probes before compensation | tracked stage가 아니라 DB/WP authentication, config/file values를 다시 probe해 old/new/ambiguous actual 상태를 판정합니다. | 보상 대상을 observed 상태에서 계산합니다. |
| 2e6649a7706d | tools/rotate_secrets.py | rotation signal 상태 처리 | 첫 SIGINT/SIGTERM은 forward transition을 중단시키지만 되돌리기-active가 된 뒤 추가 termination signal은 기록만 하고 compensation 완료까지 즉시 종료하지 않습니다. | recovery 자체가 두 번째 operator signal로 끊기는 것을 막습니다. |
| 2e6649a7706d | tools/rotate_secrets.py | deferred signal/result reporting | 되돌리기 verification을 끝낸 뒤 대기 중인 signal과 primary/되돌리기 errors를 정해진 precedence로 보고합니다. | 복구 완료와 프로세스 exit semantics를 분리합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 2e6649a7706d^ 2e6649a7706d -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 9934b478c79a 2e6649a7706d -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | subprocess/step 실패는 해당 영속 변경이 일어나지 않았다고 가정했습니다. |
| 실제 실패 또는 위험 | store write 후 timeout/nonzero/exception이 가능해 tracked 상태와 actual credential generation이 다를 수 있고 되돌리기 중 signal이 복구를 끊을 수 있었습니다. |
| root cause | exit status 중심 stage tracking과 forward/되돌리기를 구분하지 않는 signal handling이 root cause였습니다. |
| 수정된 불변식 / 결정 | actual authentication/file probes로 상태를 재구성하고 되돌리기-active 동안 추가 signal을 defer한 뒤 prior generation을 보상합니다. |
| 실제 수정 코드 | `tools/rotate_secrets.py`의 `post-write failure hooks`; `tools/rotate_secrets.py`의 `actual-state probes before compensation`; `tools/rotate_secrets.py`의 `rotation signal state machine`; `tools/rotate_secrets.py`의 `deferred signal/result reporting` |
| 변경된 ordering / 소유권 / lifecycle | orchestrator가 forward/되돌리기/대기 중인-signal 상태를 소유합니다. 구성 요소 명령 result가 아니라 authoritative probes가 compensation order를 결정합니다. |
| 이 fix가 보장하는 것 | 명령 exit와 actual 변경이 어긋나도 observed 상태에 맞춰 prior generation을 복구하며, recovery 중 추가 termination이 compensation을 중단하지 않습니다. |
| 아직 보장하지 않는 것 | 프로세스 SIGKILL, host power loss, 되돌리기 primitive 자체가 모두 실패하는 경우 prior generation을 반드시 복원한다고 보장하지 않습니다. |
| 연결되는 회귀 테스트 | post-write 실패 matrix와 double-signal 런타임 scenario가 corrected 불변식과 retry 가능성을 고정합니다. `0da35c72add5`이 every-영속-stage post-write 실패와 SIGTERM→되돌리기-active SIGINT, retry를 live 런타임에서 검증합니다. |

#### 중요도 S 상태 변화 기록

| 단계 | 학습자 기록 |
| --- | --- |
| correction 전 authoritative 상태 | `9934b478c79a`는 명령 실패를 no 변경으로 해석할 수 있었고 첫 signal 뒤 되돌리기 중 추가 signal이 프로세스를 끝내 mixed 상태를 고착시킬 수 있었습니다. |
| partial / ambiguous 상태 종류 | store write 후 timeout/nonzero/exception이 가능해 tracked 상태와 actual credential generation이 다를 수 있고 되돌리기 중 signal이 복구를 끊을 수 있었습니다. |
| 반영 또는 커밋 경계 | 각 write 뒤 ambiguity를 실패 주입으로 모델링하고 actual consumer probes로 상태를 탐색하며 되돌리기-active 동안 signal을 지연하는 상태 처리를 도입했습니다. |
| 되돌리기 / compensation 진입 조건 | post-write exception, first signal, 되돌리기 step 실패, 되돌리기-active second signal을 모두 compensation/error aggregation 경로로 보냅니다. incomplete recovery는 성공으로 숨기지 않습니다. |
| recovery 중 보호되는 불변식 | orchestrator가 forward/되돌리기/대기 중인-signal 상태를 소유합니다. 구성 요소 명령 result가 아니라 authoritative probes가 compensation order를 결정합니다. |
| 성공 endpoint | 명령 exit와 actual 변경이 어긋나도 observed 상태에 맞춰 prior generation을 복구하며, recovery 중 추가 termination이 compensation을 중단하지 않습니다. |
| 실패 endpoint | 프로세스 SIGKILL, host power loss, 되돌리기 primitive 자체가 모두 실패하는 경우 prior generation을 반드시 복원한다고 보장하지 않습니다. |
| 후속 회귀 근거 | `0da35c72add5`이 every-영속-stage post-write 실패와 SIGTERM→되돌리기-active SIGINT, retry를 live 런타임에서 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 프로세스 SIGKILL, host power loss, 되돌리기 primitive 자체가 모두 실패하는 경우 prior generation을 반드시 복원한다고 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `0da35c72add5`이 every-영속-stage post-write 실패와 SIGTERM→되돌리기-active SIGINT, retry를 live 런타임에서 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 명령 exit와 actual 변경이 어긋나도 observed 상태에 맞춰 prior generation을 복구하며, recovery 중 추가 termination이 compensation을 중단하지 않습니다.

### 8. `0da35c72add5` — test(secrets): 회전 롤백과 재시도 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `SECRETS`, `RECOVERY` |
| 원문에서 정한 역할 | 정상 교체, 실패 주입, 중단, 되돌리기, 누수, 재시도를 검증했습니다. |
| 이전 커밋 | `2e6649a7706d` |
| 다음 커밋 | `2557079c2d19` |

#### 원문이 확정한 범위

<!-- 원문 요약: Exercises successful rotation, multiple post-write failures, signal interruption during host-file publication, rollback, leak checks, and retry with the same inputs. -->
<!-- 원문 판단 근거: The scenario provides strong real-system evidence for one of the project's hardest state transitions and protects against regressions in compensation ordering. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `0da35c72add5`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `rotation success scenario`에서 complete forward path를 실제 consumer 인터페이스로 검증합니다.
- `tests/runtime_stack.py`의 `failure-stage matrix`에서 ambiguous 상태 compensation coverage를 만듭니다.
- `tests/runtime_stack.py`의 `double-signal scenario`에서 되돌리기 signal deferral의 실제 프로세스 동작을 검증합니다.
- `tests/runtime_stack.py`의 `retry/leak assertions`에서 복구가 단순 old 상태가 아니라 재시도 가능한 clean 상태임을 확인합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 0da35c72add5 | tests/runtime_stack.py | rotation success scenario | replacement generation을 만든 뒤 CLI로 rotation하고 DB root/app, WP admin/author의 new success/old 실패, config/files, 서비스 헬스 상태를 확인합니다. | complete forward path를 실제 consumer 인터페이스로 검증합니다. |
| 0da35c72add5 | tests/runtime_stack.py | 실패-stage matrix | 각 host/DB/WP 영속 반영 전후, 특히 post-write 실패 hook에서 exception을 주입하고 prior generation 복구를 검사합니다. | ambiguous 상태 compensation coverage를 만듭니다. |
| 0da35c72add5 | tests/runtime_stack.py | double-signal scenario | SIGTERM으로 forward를 중단하고 되돌리기-active ready marker 뒤 SIGINT를 추가로 보내 compensation이 계속되는지 확인합니다. | 되돌리기 signal deferral의 실제 프로세스 동작을 검증합니다. |
| 0da35c72add5 | tests/runtime_stack.py | retry/leak 검사문 | 실패/되돌리기 뒤 같은 replacement로 다시 rotation해 성공하고 temp files, 비밀값 args/env/log leaks, Docker 자원이 없는지 검사합니다. | 복구가 단순 old 상태가 아니라 재시도 가능한 clean 상태임을 확인합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 0da35c72add5^ 0da35c72add5 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 2e6649a7706d 0da35c72add5 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | rotation은 success 시 new-only generation, 실패/signal 시 verified old generation 또는 explicit incomplete 되돌리기로 끝나며 retry 가능합니다. |
| 재현하는 실패 / 경계 | 각 영속 write의 pre/post 실패, post-write ambiguity, SIGTERM과 되돌리기-active SIGINT입니다. |
| 테스트 방식 | live 통합 + 결정적 실패-stage hooks + double-signal synchronization |
| 테스트 준비 코드와 실패 주입 | old/new 네-file generations와 healthy stack을 만들고 production rotation에 named 실패/pause hooks를 전달합니다. |
| 실제 통과하는 실제 실행 경로 | 잠금→quiesce→WP/config/DB/files 반영→recreate/probes 또는 actual-상태 compensation→retry를 통과합니다. |
| 핵심 검사문 | new works/old fails 또는 old works/new fails, files/config 일치, 서비스 헬스 상태, no temp/argv/env/log 비밀값, retry success를 확인합니다. |
| 이 테스트가 증명하는 것 | multi-store compensation과 되돌리기 signal deferral이 실제 consumer 상태에서 작동함을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | SIGKILL/power loss와 모든 external 변경은 증명하지 않습니다. |
| 성격 | 결정적 distributed-transaction 회귀 |
| 막는 후속 회귀 | partial generation, 명령-status-only 되돌리기, 되돌리기 중 second signal abort, stale temp/leak, non-retryable 상태를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: SIGKILL·hardware loss·모든 DB/WordPress internal 실패를 포괄하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `2557079c2d19`이 테스트 코드 자체가 obsolete 런타임 비밀값 마운트를 허용해 property를 약화하지 못하도록 정적 guard를 추가합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: ordinary/post-write/signal failures가 prior verified generation으로 보상되고 clean retry가 가능하며 success는 new-only generation으로 끝남을 증명합니다.

### 9. `2557079c2d19` — test(secrets): 회전 후 런타임 비밀 경계 고정

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `TEST`, `SECRETS` |
| 원문에서 정한 역할 | 테스트가 정상 실행 중의 비밀값 경계를 약화하지 못하게 했습니다. |
| 이전 커밋 | `0da35c72add5` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Statically forbids rotation tests from depending on obsolete runtime secret mounts and requires post-rotation cleanup checks. -->
<!-- 원문 판단 근거: It preserves the intended secret architecture, but the change is a focused regression guard rather than a new security mechanism. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `2557079c2d19`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `forbidden `/run/secrets` assumptions`에서 테스트 편의를 위해 실제 코드의 불변식를 약화하는 회귀를 막습니다.
- `tests/validate_stack.py`의 `post-rotation boundary requirements`에서 성공/되돌리기 후에도 초기화-only 비밀값 경계가 유지됩니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2557079c2d19 | tests/validate_stack.py | forbidden `/run/secrets` assumptions | rotation/런타임 테스트 소스에서 mounted 비밀값 comparison helper와 obsolete `/run/secrets` path pattern을 거부합니다. | 테스트 편의를 위해 실제 코드의 불변식를 약화하는 회귀를 막습니다. |
| 2557079c2d19 | tests/validate_stack.py | post-rotation 경계 requirements | rotation scenario가 full 런타임 비밀값 env/마운트 absence를 다시 검사하고 비공개 config temp 정리를 요구하는지 확인합니다. | 성공/되돌리기 후에도 초기화-only 비밀값 경계가 유지됩니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 2557079c2d19^ 2557079c2d19 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 0da35c72add5 2557079c2d19 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | rotation 이후에도 장기 실행 services는 비밀값 마운트/password 환경을 보유하지 않고 temp credential files가 남지 않습니다. |
| 재현하는 실패 / 경계 | 테스트 코드가 obsolete `/run/secrets` helper를 사용해 weaker architecture를 암묵적으로 허용하는 경계입니다. |
| 테스트 방식 | 정적 원본 규약 |
| 테스트 준비 코드와 실패 주입 | 테스트와 production 원본의 forbidden/required patterns가 테스트 준비 코드입니다. |
| 실제 통과하는 실제 실행 경로 | `tests/validate_stack.py`가 rotation/런타임 테스트 소스와 Compose blocks를 읽습니다. |
| 핵심 검사문 | 금지 mounted-비밀값 pattern 부재, full post-rotation 경계 검사문과 temp 정리 pattern 존재를 확인합니다. |
| 이 테스트가 증명하는 것 | verification 코드가 production 비밀값 경계를 약화하지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 실제 컨테이너 inspect, authentication, signal timing은 증명하지 않습니다. |
| 성격 | 정적 architecture guard |
| 막는 후속 회귀 | 테스트-only 런타임 비밀값 마운트, incomplete post-rotation inspect, config temp leak 검사문 제거를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 런타임 authentication/되돌리기 correctness를 직접 실행해 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: 개발 흐름 2의 비밀값-free 런타임 불변식을 credential rotation 이후에도 원본 규약으로 연결합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 테스트 suite 자체가 steady-상태 비밀값 마운트를 재도입하거나 production보다 약한 테스트 테스트 준비 코드를 사용하지 않게 합니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| replacement 및 active 비밀값 files는 비공개 regular single-link 입력이며 individual 반영은 atomic/영속합니다. | a2d20b8c2c03 | 9934b478c79a | 0da35c72add5 | 파일 디스크립터 validation과 same-directory fsync/replace, temp/leak 런타임 검사문이 연결됩니다. |
| rotation 성공 시 모든 replacement credential은 작동하고 모든 previous credential은 거부됩니다. | 64844c583211 | 9934b478c79a | 0da35c72add5 | DB/WP positive+negative probes와 files/config exact generation 검사가 success endpoint입니다. |
| rotation 실패 시 verified prior generation으로 compensation하거나 incomplete 되돌리기를 명시적으로 보고합니다. | c68486d55f30 | 2e6649a7706d에서 ambiguous 상태/signal 보강 | 0da35c72add5 | actual-상태 probes, reverse compensation, error aggregation과 stage matrix가 연결됩니다. |
| 되돌리기 활성화 뒤 추가 termination signal은 recovery를 중단하지 않습니다. | 2e6649a7706d | 2e6649a7706d | 0da35c72add5 | 되돌리기-active 상태가 second signal을 defer하고 double-signal scenario가 completion을 확인합니다. |
| 회전 뒤에도 장기 실행 런타임 비밀값 exposure 경계가 유지됩니다. | 기존 초기화 architecture | 2557079c2d19 정적 guard | 0da35c72add5 | 런타임 inspect와 obsolete 테스트-pattern ban이 success/되돌리기 후에도 비밀값-free serving을 요구합니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: 개별 명령의 nonzero를 no 변경으로 해석하고 ordinary exception만 되돌리기하면 post-write 실패와 signal이 mixed credential generation을 남길 수 있었습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: 프로젝트 잠금, Nginx quiescence, 애플리케이션-first/root-last ordering, actual positive/negative probes, reverse compensation과 되돌리기-active signal deferral이 transition endpoint를 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `0da35c72add5`의 per-stage/post-write/double-signal/retry matrix와 `2557079c2d19` 비밀값-경계 정적 guard가 회귀를 막습니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| DB/WordPress/host files 중 일부만 new generation | 9934b478c79a ordered locked transaction + c68486d55f30 compensation | 0da35c72add5 반영-stage 실패 matrix | per-store atomicity를 global atomicity로 과장하지 않고 observed 상태를 보상합니다. |
| subprocess가 write 후 nonzero를 반환해 상태 ambiguous | 2e6649a7706d actual probes와 post-write compensation | post-write 실패 주입 scenarios | exit 코드 대신 actual authentication/file 동작이 authoritative합니다. |
| operator signal이 host files 교체 뒤 또는 되돌리기 중 도착 | 2e6649a7706d first signal 실패 conversion + 되돌리기 signal defer | 0da35c72add5 SIGTERM 후 되돌리기-active SIGINT | recovery를 완료한 뒤 대기 중인 termination을 처리합니다. |
| 새 값은 작동하지만 old 값도 인증됨 | 64844c583211 positive+negative verification | 0da35c72add5 end-to-end authentication | success는 new acceptance뿐 아니라 old revocation까지 포함합니다. |
| 테스트가 obsolete `/run/secrets`를 허용 | 2557079c2d19 정적 guard | 정적 validator 검사문 | 테스트 convenience가 production 경계를 바꾸지 못하게 합니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Host 비밀값 files | old generation 개별 files | same-directory atomic replacement와 parent sync | a2d20b8c2c03 | 개별 file은 atomic하지만 네 files 전체는 orchestrator가 순서를 관리합니다. |
| MariaDB accounts | root/애플리케이션 current passwords | local-소켓 변경; 애플리케이션 먼저, root 마지막 | 832d182743ea + 9934b478c79a order | root repair path를 forward 후반까지 유지합니다. |
| WordPress users | WordPress DB의 hashed passwords | `wp_set_password` 애플리케이션-owned 변경 | 0aa998fdd344 | admin/author를 별도 stages로 추적합니다. |
| Private wp-config.php | DB credential consumer | exact define를 same-filesystem rename으로 교체 | 0aa998fdd344 | web tree가 아닌 비공개 config 볼륨이 authority입니다. |
| Rotation orchestrator | 개별 helper 결과 의존 | actual probes와 프로젝트 잠금으로 generation 전환/보상 소유 | 9934b478c79a, 2e6649a7706d | distributed stores의 success/실패 endpoint를 판정합니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: Credentials are represented in four host files, two MariaDB accounts, two WordPress users, and WordPress configuration. The thread therefore evolves from individual 변경 primitives to a verified 상태 변화 and then to compensation for commands that may change 상태 before failing. Deferring further termination while rollback is active is the key correction that prevents recovery itself from being interrupted. -->
- 최종 authoritative 상태와 소유자: 성공 시 네 host files, DB root/app accounts, WP admin/author users, 비공개 config가 같은 new generation을 소유하고 old generation은 거부됩니다.
- 정상 실행의 entry point와 완료 조건: 프로젝트 잠금 아래 pre-verification, Nginx quiesce, ordered mutations, host 반영, force-recreate, new/old probes가 모두 성공하면 완료입니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: 실패/first signal 시 actual 상태를 probe해 old generation으로 reverse compensation하고, 되돌리기 중 추가 signal은 완료까지 지연합니다. incomplete 되돌리기는 명시적 실패입니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: 개발 흐름 2의 비밀값-free 런타임과 개발 흐름 8의 operations/diagnostics가 credential generation 변화 뒤에도 유지될 전제를 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: 여러 stores를 한 storage transaction처럼 원자 커밋하거나 SIGKILL/power loss에서 반드시 복구한다고 보장하지 않습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | replacement/current 검증 | a2d20b8c2c03 readers | 네 old/new files를 비공개 stable 입력으로 읽습니다. | unsafe file/content면 변경 전 실패합니다. |
| 2 | 잠금/old-상태 probes | 9934b478c79a rotate | same-프로젝트 잠금과 current generation/런타임 경계를 확인합니다. | mixed pre-상태면 rotation을 시작하지 않습니다. |
| 3 | 공개 path quiesce | 9934b478c79a Nginx stop | 외부 요청이 transition 중 mixed generation을 관측하지 않게 합니다. | stop 실패면 forward 변경 전에 compensation/recovery로 갑니다. |
| 4 | WP/config 변경 | 0aa998fdd344 primitives | admin/author와 비공개 config를 new values로 바꿉니다. | post-write 실패는 actual probes 대상입니다. |
| 5 | DB app/root 변경 | 832d182743ea + ordered orchestrator | 애플리케이션 password 후 root password를 바꿉니다. | root는 repair capability 때문에 forward 후반에 변경됩니다. |
| 6 | host files/recreate | a2d20b8c2c03 반영 + 9934b478c79a recreate | active 원본 files를 new generation으로 게시하고 services를 새로 띄웁니다. | 반영/recreate 실패는 observed 상태 compensation으로 갑니다. |
| 7 | global verification | 64844c583211 probes | new works, old fails, files/config match, 런타임 no-비밀값을 확인합니다. | 하나라도 어긋나면 success로 확정하지 않습니다. |
| 8 | compensation/signal defer | 2e6649a7706d | actual 상태를 탐색해 old generation으로 되돌리고 되돌리기-active signal을 지연합니다. | 복구 불완전과 대기 중인 signal을 모두 최종 result에 보존합니다. |

### 학습자의 최종 설명

> credential generation은 한 DB row가 아니라 네 host files, MariaDB 두 accounts, WordPress 두 users, 비공개 config에 분산됩니다. 각 file과 구성 요소 primitive는 개별적으로 안전하지만 전체는 atomic transaction이 아닙니다. orchestrator는 프로젝트 잠금과 Nginx quiescence 아래 WordPress users/config, DB 애플리케이션, DB root, host files 순으로 전환하고 services를 recreate한 뒤 new success와 old rejection을 모두 확인합니다. 초기 compensation은 명령 실패를 no 변경으로 볼 위험이 있었으나 `2e6649a7706d`에서 각 write 뒤 actual probes로 상태를 재구성하고 되돌리기 중 추가 signal을 defer하도록 수정됐습니다. 따라서 success는 new-only generation, 실패는 verified old generation 또는 명시적 incomplete 되돌리기가라는 endpoint로 관리됩니다.

## 학습 완료 자가 점검

- [x] rotation을 하나의 DB transaction처럼 원자적이라고 표현하지 않았습니까?
- [x] subprocess exit 코드만으로 상태 변경 여부를 결정하지 않았습니까?
- [x] root credential을 너무 일찍 바꿔 후속 repair path를 끊는 ordering을 놓치지 않았습니까?
- [x] 되돌리기 중 두 번째 signal이 즉시 종료시킨다고 잘못 설명하지 않았습니까?
- [x] 새 값 성공과 옛 값 거부를 모두 실제 consumer 인터페이스로 확인했습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 06-credential-rotation-and-compensation.md =====

===== BEGIN FILE: 07-immutable-build-inputs.md =====
# 개발 흐름 7 — 변경 불가능한 빌드 입력과 실행 공급망 검증

## 개발 흐름의 목표

moving Debian/WordPress inputs를 immutable digest·스냅샷·체크섬으로 고정하고, 원본 string뿐 아니라 실제 실행 중 package/애플리케이션 version까지 검증하는 maintained supply-chain 규약을 추적합니다.

### 원문에서 정한 의의

재현 가능성을 한 번 고정하고 끝내는 값이 아니라 계속 관리하는 규약으로 다룹니다. 상위 이미지와 패키지 식별자를 명시하고 지원 버전 갱신 절차를 기록하며, Dockerfile 문자열뿐 아니라 실제 컨테이너에서 실행 중인 소프트웨어까지 검사합니다.

<details>
<summary>영문 원문</summary>

> Reproducibility is treated as a maintained contract rather than a one-time freeze. The first commits make upstream identities explicit; the later update demonstrates how supported versions advance; and runtime inspection closes the gap between strings in Dockerfiles and the software actually executing inside containers.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- base 이미지 digest와 dated APT 스냅샷은 서로 다른 어떤 입력을 고정합니까?
- 스냅샷 메타데이터의 validity date를 비활성화한 trade-off는 무엇입니까?
- WordPress core를 이미지-controlled, `wp-content`를 볼륨-controlled로 나눈 기준은 무엇입니까?
- 초기화 런타임 download를 제거하면 interruption recovery와 reproducibility가 어떻게 연결됩니까?
- 원본 pin checks만으로 stale 이미지 cache나 unintended package resolution을 잡지 못하는 이유는 무엇입니까?
- pin update 커밋이 reproducibility 규약을 깨지 않고 maintenance를 수행했다는 증거는 무엇입니까?

## 완료 기준

- 각 Dockerfile의 Debian digest와 스냅샷 원본 설정을 해당 SHA에서 확인했습니다.
- WP-CLI/WordPress version, SHA-256, 이미지 원본 directory, core manifest 생성/검증 경로를 추적했습니다.
- core reconciliation과 `wp-content` 보존 policy를 파일별로 비교했습니다.
- 정적 pin 테스트, live WordPress/WP-CLI identity, dpkg minimum, PHP/MariaDB compatibility 검증을 구분했습니다.
- 보안 지원 pin update에서 함께 변경되어야 한 원본/테스트 값을 기록했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `3e29fbd34389` | build(images): Debian 이미지와 패키지 입력 고정 | **A** | `SUPPLY_CHAIN`<br>`RISK`<br>`ARCH` | Debian 기반 이미지와 패키지 저장소를 변경 불가능한 입력으로 고정했습니다. |
| 2 | `f60ac8061c01` | build(wordpress): WordPress 산출물을 고정해 게시 | **A** | `SUPPLY_CHAIN`<br>`BOOTSTRAP`<br>`RISK` | WordPress·WP-CLI 산출물을 고정하고 코어 파일 반영을 초기화 수렴 과정으로 옮겼습니다. |
| 3 | `7b28cccaec1d` | test(supply-chain): 불변 image 입력 검증 | **A** | `TEST`<br>`SUPPLY_CHAIN`<br>`RISK` | Locked the 원본 pins and running 애플리케이션 versions in 테스트. |
| 4 | `cd5982c8ea42` | fix(supply-chain): 보안 지원 runtime pin 갱신 | **B** | `SUPPLY_CHAIN`<br>`RISK` | 검토된 불변 런타임 버전 집합을 갱신하되 다시 이동하는 입력으로 돌아가지 않았습니다. |
| 5 | `127a70f6e4b2` | test(supply-chain): 검토된 runtime 최소 버전 검증 | **A** | `TEST`<br>`SUPPLY_CHAIN`<br>`RISK` | 설치 패키지 최소 버전과 실행 중 PHP/MariaDB 호환성 하한을 검증했습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `3e29fbd34389` — build(images): Debian 이미지와 패키지 입력 고정

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `SUPPLY_CHAIN`, `RISK`, `ARCH` |
| 원문에서 정한 역할 | Debian 기반 이미지와 패키지 저장소를 변경 불가능한 입력으로 고정했습니다. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `f60ac8061c01` |

#### 원문이 확정한 범위

<!-- 원문 요약: Pins all service base images by digest and redirects Debian packages to an immutable dated snapshot. -->
<!-- 원문 판단 근거: This changes the build trust model from moving upstream inputs to reviewed immutable inputs, a significant reproducibility and supply-chain decision despite not changing application behavior. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `3e29fbd34389`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/*/Dockerfile`의 `FROM Debian dated tag@sha256`에서 base filesystem bytes가 moving tag가 아니라 content digest로 고정됩니다.
- `srcs/requirements/*/Dockerfile`의 `snapshot.debian.org sources`에서 APT package universe와 메타데이터가 빌드 date에 따라 움직이지 않습니다.
- `srcs/requirements/wordpress/Dockerfile`의 `dependency cleanup`에서 pinning 결정과 별개로 reviewed package set을 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 3e29fbd34389 | srcs/requirements/*/Dockerfile | FROM Debian dated tag@sha256 | Nginx, MariaDB, WordPress 세 Dockerfile이 동일한 reviewed Debian dated slim 이미지 digest를 사용합니다. | base filesystem bytes가 moving tag가 아니라 content digest로 고정됩니다. |
| 3e29fbd34389 | srcs/requirements/*/Dockerfile | 스냅샷.debian.org sources | main, updates, security package sources를 explicit timestamp 스냅샷으로 바꾸고 스냅샷 사용을 위해 Valid-Until 검사를 비활성화합니다. | APT package universe와 메타데이터가 빌드 date에 따라 움직이지 않습니다. |
| 3e29fbd34389 | srcs/requirements/wordpress/Dockerfile | dependency 정리 | 더 이상 사용하지 않는 `unzip`을 제거합니다. | pinning 결정과 별개로 reviewed package set을 줄입니다. |

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | moving `debian:bookworm-slim`과 live APT mirror는 동일 원본을 다른 날 빌드할 때 base layer와 package versions가 달라질 수 있었습니다. |
| 선택한 경계 / 결정 | base 이미지는 digest, package repositories는 dated 스냅샷 timestamp로 각각 고정했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/requirements/*/Dockerfile`의 `FROM Debian dated tag@sha256`; `srcs/requirements/*/Dockerfile`의 `snapshot.debian.org sources`; `srcs/requirements/wordpress/Dockerfile`의 `dependency cleanup` |
| 상태·소유권·수명 변화 | Dockerfile이 reviewed upstream identities를 소유하고 빌드는 해당 digest/스냅샷만 소비합니다. security updates는 자동이 아니라 explicit pin maintenance가 소유합니다. |
| 주요 실패 브랜치 | 스냅샷 unavailable, digest mismatch, package index/install 실패는 빌드 실패가 됩니다. Valid-Until 비활성화는 오래된 메타데이터를 의도적으로 허용하는 trade-off입니다. |
| 이 커밋의 보장 | 동일 원본과 reachable 스냅샷으로 같은 base/package 입력을 선택하는 reproducibility 경계를 제공합니다. |
| 한계와 다음 관련 커밋 | WordPress/WP-CLI artifact identity, actual installed package versions, 이미지 cache가 올바른지까지는 보장하지 않습니다. `f60ac8061c01`이 애플리케이션 artifacts를 고정하고 `7b28cccaec1d`/`127a70f6e4b2`가 원본과 live 런타임 identity를 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: WordPress/WP-CLI artifact identity, actual installed package versions, 이미지 cache가 올바른지까지는 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `f60ac8061c01`이 애플리케이션 artifacts를 고정하고 `7b28cccaec1d`/`127a70f6e4b2`가 원본과 live 런타임 identity를 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 동일 원본과 reachable 스냅샷으로 같은 base/package 입력을 선택하는 reproducibility 경계를 제공합니다.

### 2. `f60ac8061c01` — build(wordpress): WordPress 산출물을 고정해 게시

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `SUPPLY_CHAIN`, `BOOTSTRAP`, `RISK` |
| 원문에서 정한 역할 | WordPress·WP-CLI 산출물을 고정하고 코어 파일 반영을 초기화 수렴 과정으로 옮겼습니다. |
| 이전 커밋 | `3e29fbd34389` |
| 다음 커밋 | `7b28cccaec1d` |

#### 원문이 확정한 범위

<!-- 원문 요약: Pins WP-CLI and WordPress archives with checksums, stages WordPress core in the image, atomically reconciles files at bootstrap, and disables automatic core updates. -->
<!-- 원문 판단 근거: It removes runtime downloads from initialization and makes the application artifact an immutable, verified build input, significantly strengthening both reproducibility and recovery semantics. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `f60ac8061c01`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/wordpress/Dockerfile`의 `WP_CLI_VERSION / SHA-256`에서 런타임 네트워크가 다른 WP-CLI bytes를 제공하지 못합니다.
- `srcs/requirements/wordpress/Dockerfile`의 `WORDPRESS_VERSION / SHA-256 / /usr/src/wordpress`에서 reviewed 이미지 artifact가 core file authority가 됩니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core reconciliation`에서 초기화 interruption recovery와 reproducibility가 같은 이미지 artifact에 의존합니다.
- `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 ``wp-content` preservation branch`에서 이미지 upgrade가 애플리케이션-owned content를 덮어쓰지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| f60ac8061c01 | srcs/requirements/wordpress/Dockerfile | WP_CLI_VERSION / SHA-256 | explicit WP-CLI version과 체크섬으로 phar를 빌드 시 download·검증한 뒤 이미지에 설치합니다. | 런타임 네트워크가 다른 WP-CLI bytes를 제공하지 못합니다. |
| f60ac8061c01 | srcs/requirements/wordpress/Dockerfile | WORDPRESS_VERSION / SHA-256 / /usr/src/wordpress | WordPress archive도 explicit version/체크섬으로 검증하고 이미지-owned 원본 directory에 풀며 sorted core manifest를 만듭니다. | reviewed 이미지 artifact가 core file authority가 됩니다. |
| f60ac8061c01 | srcs/requirements/wordpress/tools/docker-entrypoint.sh | core reconciliation | 런타임 `wp core download`를 제거하고 이미지 원본/manifest를 검증한 뒤 영속 web 볼륨의 core files를 임시 객체+rename 방식으로 맞춥니다. | 초기화 interruption recovery와 reproducibility가 같은 이미지 artifact에 의존합니다. |
| f60ac8061c01 | srcs/requirements/wordpress/tools/docker-entrypoint.sh | `wp-content` 보존 브랜치 | existing `wp-content`는 볼륨-owned user/plugin/upload 상태로 보존하고 core reconciliation 대상에서 분리합니다. | 이미지 upgrade가 애플리케이션-owned content를 덮어쓰지 않습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff f60ac8061c01^ f60ac8061c01 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 3e29fbd34389 f60ac8061c01 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | WordPress 초기화가 런타임 네트워크에서 moving artifact를 download하면 이미지 review와 실제 영속 core bytes가 분리되고 중단 시 partially downloaded 상태가 남을 수 있었습니다. |
| 선택한 경계 / 결정 | WP-CLI와 WordPress를 빌드-time version/체크섬으로 고정하고 이미지-controlled core 원본/manifest를 초기화가 영속 볼륨에 수렴시키도록 했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/requirements/wordpress/Dockerfile`의 `WP_CLI_VERSION / SHA-256`; `srcs/requirements/wordpress/Dockerfile`의 `WORDPRESS_VERSION / SHA-256 / /usr/src/wordpress`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 `core reconciliation`; `srcs/requirements/wordpress/tools/docker-entrypoint.sh`의 ``wp-content` preservation branch` |
| 상태·소유권·수명 변화 | 이미지가 WordPress core identity를, 명명된 볼륨이 `wp-content`와 런타임 상태를 소유합니다. 초기화는 둘 사이 reconciliation/publish lifecycle을 소유합니다. |
| 주요 실패 브랜치 | download/체크섬/unpack/manifest 빌드 실패는 이미지 빌드를 중단합니다. 런타임 manifest/원본 검증이나 atomic file 반영 실패는 completion marker 전에 초기화를 실패시킵니다. |
| 이 커밋의 보장 | 런타임 네트워크 download 없이 reviewed core bytes와 WP-CLI를 사용하고, existing `wp-content`를 보존하면서 core를 manifest에 맞출 수 있습니다. |
| 한계와 다음 관련 커밋 | 원본 pins가 실제 running 컨테이너에 적용됐는지와 package security floor는 별도 런타임 근거가 필요합니다. `7b28cccaec1d`이 no-런타임-download/원본 pins/live versions를 검사하고 `cd5982c8ea42`이 maintained update 절차를 보여줍니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 원본 pins가 실제 running 컨테이너에 적용됐는지와 package security floor는 별도 런타임 근거가 필요합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `7b28cccaec1d`이 no-런타임-download/원본 pins/live versions를 검사하고 `cd5982c8ea42`이 maintained update 절차를 보여줍니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 런타임 네트워크 download 없이 reviewed core bytes와 WP-CLI를 사용하고, existing `wp-content`를 보존하면서 core를 manifest에 맞출 수 있습니다.

### 3. `7b28cccaec1d` — test(supply-chain): 불변 image 입력 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `SUPPLY_CHAIN`, `RISK` |
| 원문에서 정한 역할 | Locked the 원본 pins and running 애플리케이션 versions in 테스트. |
| 이전 커밋 | `f60ac8061c01` |
| 다음 커밋 | `cd5982c8ea42` |

#### 원문이 확정한 범위

<!-- 원문 요약: Checks immutable Debian and WordPress pins statically and verifies the running WordPress and WP-CLI versions. -->
<!-- 원문 판단 근거: The commit protects the newly established supply-chain contract from silent reversion to moving inputs or runtime downloads. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `7b28cccaec1d`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `Dockerfile pin assertions`에서 moving tag/live mirror/체크섬 제거를 정적으로 막습니다.
- `tests/validate_stack.py`의 `no runtime download / reconciliation patterns`에서 빌드-owned core와 런타임 reconciliation architecture를 고정합니다.
- `tests/runtime_stack.py`의 `live WordPress/WP-CLI versions`에서 Dockerfile 문자열과 실제 런타임 identity 사이의 gap을 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 7b28cccaec1d | tests/validate_stack.py | Dockerfile pin 검사문 | 세 base digest/스냅샷 timestamp와 WP-CLI/WordPress version/체크섬을 exact 원본 values로 검사합니다. | moving tag/live mirror/체크섬 제거를 정적으로 막습니다. |
| 7b28cccaec1d | tests/validate_stack.py | no 런타임 download / reconciliation patterns | WordPress 엔트리포인트의 `wp core download`를 금지하고 이미지 artifact/manifest/atomic 반영 pattern을 요구합니다. | 빌드-owned core와 런타임 reconciliation architecture를 고정합니다. |
| 7b28cccaec1d | tests/runtime_stack.py | live WordPress/WP-CLI versions | running WordPress 컨테이너에서 WP core version과 WP-CLI version을 실행해 pinned expected values와 비교합니다. | Dockerfile 문자열과 실제 런타임 identity 사이의 gap을 줄입니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 7b28cccaec1d^ 7b28cccaec1d -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff f60ac8061c01 7b28cccaec1d -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | immutable 원본 inputs과 running WordPress/WP-CLI identity가 같은 reviewed set을 가리킵니다. |
| 재현하는 실패 / 경계 | moving tag/mirror, 체크섬 제거, 런타임 core download, stale/wrong 애플리케이션 이미지입니다. |
| 테스트 방식 | 정적 원본 pin 규약 + live version inspection |
| 테스트 준비 코드와 실패 주입 | Dockerfiles/엔트리포인트 원본과 isolated running stack이 테스트 준비 코드입니다. |
| 실제 통과하는 실제 실행 경로 | validator가 원본을 검사하고 런타임 harness가 WordPress 컨테이너에서 version commands를 실행합니다. |
| 핵심 검사문 | exact digest/스냅샷/version/체크섬, no 런타임 download, live WP/WP-CLI version 일치를 확인합니다. |
| 이 테스트가 증명하는 것 | 원본 policy와 실제 애플리케이션-level 런타임 identity의 연결을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 installed package version, artifact signer, vulnerability 상태는 증명하지 않습니다. |
| 성격 | mixed 정적/런타임 supply-chain 회귀 |
| 막는 후속 회귀 | moving 입력 재도입, 체크섬 제거, 런타임 download, wrong cached 애플리케이션 artifact를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: OS package full inventory, cryptographic provenance, vulnerability absence, cache content 전체는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `127a70f6e4b2`가 installed package minimum과 PHP/MariaDB compatibility 근거를 더합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: reviewed 원본 pins와 running WordPress/WP-CLI identity가 함께 일치함을 증명합니다.

### 4. `cd5982c8ea42` — fix(supply-chain): 보안 지원 runtime pin 갱신

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `SUPPLY_CHAIN`, `RISK` |
| 원문에서 정한 역할 | Advanced the reviewed immutable 런타임 set without returning to moving inputs. |
| 이전 커밋 | `7b28cccaec1d` |
| 다음 커밋 | `127a70f6e4b2` |

#### 원문이 확정한 범위

<!-- 원문 요약: Advances the reviewed Debian digest, package snapshot, WordPress version, checksum, and matching assertions. -->
<!-- 원문 판단 근거: The update is security- and support-relevant, but it follows the immutable-input mechanism already established rather than introducing a new trust model. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `cd5982c8ea42`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/requirements/*/Dockerfile`의 `coordinated Debian digest/snapshot update`에서 서비스별 package universe가 서로 다른 review generation으로 갈라지지 않습니다.
- `srcs/requirements/wordpress/Dockerfile`의 `WordPress version/checksum update`에서 애플리케이션 artifact도 explicit immutable identity를 유지한 채 advance합니다.
- `tests/validate_stack.py / tests/runtime_stack.py`의 `expected pin/version updates`에서 테스트가 old pin을 무조건 고정하는 것이 아니라 reviewed set maintenance를 추적합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| cd5982c8ea42 | srcs/requirements/*/Dockerfile | coordinated Debian digest/스냅샷 update | 세 서비스가 같은 새 dated Debian base digest와 새 스냅샷 timestamp로 함께 이동합니다. | 서비스별 package universe가 서로 다른 review generation으로 갈라지지 않습니다. |
| cd5982c8ea42 | srcs/requirements/wordpress/Dockerfile | WordPress version/체크섬 update | WordPress pin과 체크섬을 새 supported patch release로 갱신합니다. | 애플리케이션 artifact도 explicit immutable identity를 유지한 채 advance합니다. |
| cd5982c8ea42 | tests/validate_stack.py / tests/runtime_stack.py | expected pin/version updates | 원본/정적/런타임 expected values를 production pin과 같은 커밋에서 갱신합니다. | 테스트가 old pin을 무조건 고정하는 것이 아니라 reviewed set maintenance를 추적합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff cd5982c8ea42^ cd5982c8ea42 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 7b28cccaec1d cd5982c8ea42 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | immutable pin을 영구 동결하면 upstream security/support floor가 내려가 reproducible하지만 오래된 런타임이 됩니다. |
| 실제 실패 또는 위험 | 새 digest/체크섬/compatibility가 틀리면 빌드/정적/런타임 테스트가 실패합니다. |
| root cause | immutable pin을 영구 동결하면 upstream security/support floor가 내려가 reproducible하지만 오래된 런타임이 됩니다. |
| 수정된 불변식 / 결정 | digest/스냅샷/애플리케이션 version/체크섬과 corresponding 테스트를 coordinated review unit으로 전진시켰습니다. |
| 실제 수정 코드 | `srcs/requirements/*/Dockerfile`의 `coordinated Debian digest/snapshot update`; `srcs/requirements/wordpress/Dockerfile`의 `WordPress version/checksum update`; `tests/validate_stack.py / tests/runtime_stack.py`의 `expected pin/version updates` |
| 변경된 ordering / 소유권 / lifecycle | repository 커밋이 새 reviewed generation의 identity를 소유하며 automatic moving update는 계속 허용하지 않습니다. |
| 이 fix가 보장하는 것 | maintenance가 moving 입력으로 회귀하지 않고 새 immutable reviewed set으로 수행될 수 있음을 보여줍니다. |
| 아직 보장하지 않는 것 | 새 set의 모든 vulnerability가 제거됐다는 보장은 아니며 explicit support/security criteria가 계속 필요합니다. |
| 연결되는 회귀 테스트 | `127a70f6e4b2`가 새 런타임의 package minimum과 platform compatibility를 live inspect합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 새 set의 모든 vulnerability가 제거됐다는 보장은 아니며 explicit support/security criteria가 계속 필요합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `127a70f6e4b2`가 새 런타임의 package minimum과 platform compatibility를 live inspect합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: maintenance가 moving 입력으로 회귀하지 않고 새 immutable reviewed set으로 수행될 수 있음을 보여줍니다.

### 5. `127a70f6e4b2` — test(supply-chain): 검토된 runtime 최소 버전 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `SUPPLY_CHAIN`, `RISK` |
| 원문에서 정한 역할 | 설치 패키지 최소 버전과 실행 중 PHP/MariaDB 호환성 하한을 검증했습니다. |
| 이전 커밋 | `cd5982c8ea42` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Verifies installed package minimums plus the live PHP and MariaDB compatibility floors inside the built stack. -->
<!-- 원문 판단 근거: This closes the gap between source pins and actual runtime contents, catching stale caches or unexpected package resolution in a security-sensitive build path. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `127a70f6e4b2`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `DEBIAN_PACKAGE_MINIMUMS / dpkg comparison`에서 원본 스냅샷 string뿐 아니라 실제 package selection을 검사합니다.
- `tests/runtime_stack.py`의 `live PHP version compatibility`에서 애플리케이션 런타임 engine compatibility를 live 프로세스에서 확인합니다.
- `tests/runtime_stack.py`의 `live MariaDB server version compatibility`에서 DB 서비스가 expected platform generation을 실제 실행 중인지 확인합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 127a70f6e4b2 | tests/runtime_stack.py | DEBIAN_PACKAGE_MINIMUMS / dpkg comparison | Nginx/OpenSSL/PHP/MariaDB 등 reviewed minimum을 서비스 컨테이너의 installed package version과 `dpkg --compare-versions`로 비교합니다. | 원본 스냅샷 string뿐 아니라 실제 package selection을 검사합니다. |
| 127a70f6e4b2 | tests/runtime_stack.py | live PHP version compatibility | running PHP version을 parse하고 WordPress가 요구하는 minimum과 reviewed floor 이상인지 확인합니다. | 애플리케이션 런타임 engine compatibility를 live 프로세스에서 확인합니다. |
| 127a70f6e4b2 | tests/runtime_stack.py | live MariaDB server version compatibility | server version string을 query/parse해 WordPress/MySQL compatibility minimum과 reviewed MariaDB floor를 검사합니다. | DB 서비스가 expected platform generation을 실제 실행 중인지 확인합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 127a70f6e4b2^ 127a70f6e4b2 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff cd5982c8ea42 127a70f6e4b2 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | reviewed 스냅샷에서 실제 설치된 packages와 live PHP/MariaDB가 정한 minimum/compatibility floor 이상입니다. |
| 재현하는 실패 / 경계 | stale cache, alternate package resolution, unintended downgrade, incompatible live engine/server입니다. |
| 테스트 방식 | live installed-package and 런타임-version 경계 테스트 |
| 테스트 준비 코드와 실패 주입 | isolated built stack의 Nginx/WordPress/MariaDB 컨테이너와 reviewed minimum mapping을 사용합니다. |
| 실제 통과하는 실제 실행 경로 | 컨테이너 exec→dpkg query/compare→PHP version parse→MariaDB server version query/parse를 통과합니다. |
| 핵심 검사문 | 필수 package 존재와 minimum 이상, live PHP/DB 애플리케이션 floor 이상을 확인합니다. |
| 이 테스트가 증명하는 것 | 원본 pin이 실제 package/런타임 identity로 이어졌음을 강화합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 dependency, 동작 compatibility, 보안 취약점 부재는 증명하지 않습니다. |
| 성격 | 런타임 supply-chain 경계 회귀 |
| 막는 후속 회귀 | wrong cache/이미지, package downgrade, unsupported PHP/DB 런타임이 원본-only checks를 통과하는 회귀를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 모든 transitive library, CVE absence, functional compatibility 전체를 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: 원본 text와 running software 사이 verification chain을 완성합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: reviewed minimum packages와 live platform compatibility가 실제 built/running 컨테이너에 적용됨을 증명합니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| 세 서비스 이미지는 동일한 reviewed Debian base digest와 dated package 스냅샷을 사용합니다. | 3e29fbd34389 | cd5982c8ea42 coordinated advance | 7b28cccaec1d, 127a70f6e4b2 | 세 Dockerfile exact pins와 live installed-package checks가 같은 reviewed generation을 연결합니다. |
| WordPress와 WP-CLI artifact는 explicit version과 체크섬으로 빌드-time 검증됩니다. | f60ac8061c01 | cd5982c8ea42 WordPress pin advance | 7b28cccaec1d | Dockerfile 체크섬 verification, no 런타임 download, live version 검사문이 연결됩니다. |
| WordPress core는 이미지-controlled manifest에 수렴하고 `wp-content`는 existing 볼륨 상태를 보존합니다. | f60ac8061c01 | f60ac8061c01 | 런타임 e2e/version checks | 이미지 원본/manifest reconciliation은 core만 다루고 content 브랜치는 existing 볼륨 data를 유지합니다. |
| 원본 pin과 실제 installed/런타임 identity가 함께 충족되어야 supply-chain 규약이 성립합니다. | 7b28cccaec1d | 127a70f6e4b2 package/platform 근거 강화 | 127a70f6e4b2 | 정적 exact strings와 live WP/WP-CLI/package/PHP/MariaDB version을 다른 layer로 검사합니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: moving base tags, live APT mirrors와 런타임 WordPress download는 같은 원본에서 시간에 따라 다른 이미지와 interrupted 초기화 결과를 만들 수 있었습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: Debian digest, dated snapshots, WordPress/WP-CLI version+체크섬, 이미지-owned core manifest와 초기화 reconciliation이 reviewed 입력 set을 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `7b28cccaec1d` 원본/live identity checks와 `127a70f6e4b2` installed package/런타임 compatibility floors가 stale cache와 unexpected resolution을 검출합니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| moving base tag/live mirror로 같은 원본이 다른 bytes를 빌드 | 3e29fbd34389 immutable digest/스냅샷 | 7b28cccaec1d 정적 원본 checks | base filesystem과 package universe를 별도 immutable inputs로 고정합니다. |
| startup 때 WordPress 런타임 download로 이미지 review와 상태 분리 | f60ac8061c01 verified 빌드 artifact와 core reconciliation | 7b28cccaec1d no-download/live version | 초기화는 네트워크가 아니라 이미지 원본/manifest를 사용합니다. |
| immutable pin이 오래되어 support/security floor 아래로 감 | cd5982c8ea42 explicit coordinated advance | 127a70f6e4b2 installed minimum/compatibility checks | immutability를 유지하면서 review generation을 갱신합니다. |
| Dockerfile 문자열은 맞지만 stale cache/alternate path 실행 | 런타임 identity/minimum verification | 7b28cccaec1d, 127a70f6e4b2 | 원본 규약과 effective 런타임 근거를 결합합니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Dockerfile/base 이미지 | moving bookworm-slim identity | reviewed digest가 base filesystem identity 소유 | 3e29fbd34389 FROM digest | tag 이름만 신뢰하지 않습니다. |
| APT repositories | live mirror resolution | dated 스냅샷 timestamp가 package universe 소유 | 스냅샷 원본/Valid-Until config | 업데이트는 explicit 커밋이 필요합니다. |
| WordPress core | 런타임 download/볼륨 drift 가능 | 이미지 원본 + 체크섬 manifest가 authority | f60ac8061c01 download/체크섬/manifest/reconcile | 영속 core는 이미지-reviewed bytes에 수렴합니다. |
| wp-content | core와 동일 overwrite 위험 | 볼륨-controlled user/애플리케이션 data | f60ac8061c01 보존 브랜치 | 이미지 update가 existing content를 덮지 않습니다. |
| Verification | 원본 text 중심 | live versions와 installed package floors까지 근거 확장 | 7b28cccaec1d, 127a70f6e4b2 | 원본과 런타임을 별도 layer로 비교합니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: Reproducibility is treated as a maintained 규약 rather than a one-time freeze. The first 커밋 make upstream identities explicit; the later update demonstrates how supported versions advance; and 런타임 inspection closes the gap between strings in Dockerfiles and the software actually executing inside 컨테이너. -->
- 최종 authoritative 상태와 소유자: repository pins가 reviewed base/package/애플리케이션 identities를, 이미지 manifest가 WordPress core를, 명명된 볼륨이 `wp-content`를 소유합니다.
- 정상 실행의 entry point와 완료 조건: 빌드-time digest/스냅샷/체크섬 검증과 초기화 manifest reconciliation, 정적 pins, live versions/minimums가 모두 통과하면 규약이 충족됩니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: pin mismatch/빌드 실패/런타임 version mismatch는 silent fallback 없이 실패하며 update는 coordinated reviewed 커밋으로 수행합니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: 개발 흐름 2 초기화가 네트워크-independent reviewed artifact로 수렴하고 개발 흐름 8 CI가 동일 checks를 자동 실행할 전제를 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: 취약점 부재나 byte-for-byte 모든 transitive toolchain reproducibility를 단독으로 증명하지 않습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | base/package 선택 | 3e29fbd34389 Dockerfiles | digest base와 dated snapshots만 사용합니다. | unreachable/mismatch/package 실패면 빌드 중단입니다. |
| 2 | 애플리케이션 artifacts 검증 | f60ac8061c01 Dockerfile | WP-CLI/WordPress version+SHA-256을 확인합니다. | 체크섬 mismatch면 이미지가 생성되지 않습니다. |
| 3 | core 원본/manifest 생성 | f60ac8061c01 `/usr/src/wordpress` | reviewed core bytes와 sorted manifest를 이미지에 저장합니다. | manifest/원본 mismatch는 초기화 실패입니다. |
| 4 | 영속 core 수렴 | f60ac8061c01 엔트리포인트 reconciliation | core files를 이미지 manifest에 맞추고 atomic publish합니다. | marker 전 실패는 다음 초기화에서 다시 수렴합니다. |
| 5 | content 보존 | f60ac8061c01 wp-content 브랜치 | existing user/plugin/upload data를 유지합니다. | core update가 content tree를 overwrite하지 않습니다. |
| 6 | 원본/live verification | 7b28cccaec1d, 127a70f6e4b2 테스트 | pins와 running app/package/platform versions를 비교합니다. | 문자열 또는 live identity mismatch는 회귀입니다. |
| 7 | reviewed update | cd5982c8ea42 | digest/스냅샷/version/체크섬/테스트를 함께 advance합니다. | 부분 update는 정적/런타임 mismatch로 실패합니다. |

### 학습자의 최종 설명

> reproducibility는 한번 freeze하고 잊는 상태가 아닙니다. base filesystem은 digest, Debian package universe는 dated 스냅샷, WP-CLI와 WordPress는 version+체크섬으로 각각 다른 upstream 입력을 고정합니다. WordPress core는 이미지 원본과 manifest가 authority가 되어 초기화가 영속 볼륨을 수렴시키고, `wp-content`는 볼륨-owned 상태로 보존됩니다. 정적 테스트는 moving 입력과 런타임 download 회귀를 막지만 stale cache나 alternate 빌드 path까지 보지 못하므로 live WP/WP-CLI, installed package minimum, PHP/MariaDB compatibility를 별도로 확인합니다. 후속 pin update는 모든 identities와 테스트를 함께 바꿔 immutable 규약을 유지하면서 supported generation으로 전진합니다.

## 학습 완료 자가 점검

- [x] immutable pin을 영구히 업데이트하지 않는다는 의미로 오해하지 않았습니까?
- [x] base 이미지 digest와 package 스냅샷이 같은 것을 고정한다고 합쳤습니까?
- [x] WordPress core와 `wp-content`의 소유권 policy를 반대로 설명하지 않았습니까?
- [x] 원본 strings만 확인하고 실제 컨테이너 version 근거를 생략하지 않았습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 07-immutable-build-inputs.md =====

===== BEGIN FILE: 08-operational-hardening-and-automation.md =====
# 개발 흐름 8 — 운영 강화, 비공개 진단, 범위가 제한된 자동화

## 개발 흐름의 목표

네트워크 least privilege, 자원/shutdown policy, destructive-작업 guard, 비공개 diagnostics, owned-자원 정리, 직렬 local verification, least-privilege CI를 하나의 bounded operational lifecycle로 연결합니다.

### 원문에서 정한 의의

운영 방침을 실행 가능한 근거로 바꿉니다. 실제 컨테이너에서 자원 제한과 네트워크 경계를 검사하고, 파괴적 명령과 진단 기능이 안전하게 실패하도록 하며, 로컬·CI 실행기가 만든 모든 프로젝트 자원을 추적해 정리합니다. 전역 Docker 정리는 사용하지 않고 해당 프로젝트가 소유한 자원만 제거합니다.

<details>
<summary>영문 원문</summary>

> This progression turns operational policy into executable evidence. Runtime limits and network boundaries are inspected on live containers; destructive commands and diagnostics fail safely; and both local and CI runners account for every project resource they create. The cleanup tooling deliberately avoids global Docker pruning, preserving the same ownership discipline used by the product management paths.

</details>

## 이 개발 흐름을 이해하기 위한 핵심 질문

- frontend/backend 네트워크 분리 뒤 각 서비스의 exact membership과 WordPress dual-homing 이유는 무엇입니까?
- Compose에 limit/stop/log policy를 선언한 것과 Docker가 실제 적용한 것을 어떻게 구분해 검증합니까?
- `fclean` confirmation이 generic yes/no가 아니라 selected 프로젝트 name과 같아야 하는 이유는 무엇입니까?
- diagnostics가 비밀값을 안전하게 읽지 못하면 왜 일부 자료라도 게시하지 않고 fail closed해야 합니까?
- 테스트 정리 실패가 primary scenario success를 무효화해야 하는 이유는 무엇입니까?
- crash recovery 정리가 label/tag 소유권만 사용하고 global prune을 금지하는 이유는 무엇입니까?
- local `verify`와 CI workflow가 result precedence, timeout, diagnostics, 정리를 어떻게 보존합니까?

## 완료 기준

- live Docker inspect로 네트워크, limits, stop, log, security policy를 원본 declaration과 대조했습니다.
- destructive Make 대상의 exact 프로젝트-name guard와 거부 후 stack 헬스 상태를 확인했습니다.
- diagnostic redaction 입력, longer-first masking, structural masking, 비공개 반영, rescan을 추적했습니다.
- normal 정리, crash recovery 정리, `--keep`, leak report exit status의 소유권 차이를 정리했습니다.
- 직렬 verify와 CI의 timeout/정리/diagnostic result precedence를 실제 control flow로 확인했습니다.
- workflow 자체를 검증하는 text/AST/mock layers와 금지 pattern을 구분했습니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `27a3dca01d3b` | feat(network): DB 트래픽을 내부 backend로 격리 | **A** | `STACK`<br>`RISK`<br>`ARCH` | Separated 공개 요청 traffic from the internal database 네트워크. |
| 2 | `911544133fb4` | feat(runtime): 서비스 자원과 종료 한계 적용 | **B** | `OPERATIONS`<br>`RISK`<br>`STACK` | Applied 자원, stop, privilege, and log-rotation policy. |
| 3 | `74c285925325` | fix(make): 볼륨 삭제 전에 확인을 요구 | **A** | `OPERATIONS`<br>`RISK`<br>`EDGE` | Guarded destructive 볼륨 deletion with exact 프로젝트 confirmation. |
| 4 | `ef74ad47ea81` | feat(diagnostics): Compose 비밀값과 민감 항목 마스킹 | **A** | `OPERATIONS`<br>`SECRETS`<br>`RISK` | 진단 정보 마스킹이 불완전하면 실패하도록 만들었습니다. |
| 5 | `27a083d91c87` | feat(diagnostics): 비공개 진단 세트와 CLI 연결 | **A** | `OPERATIONS`<br>`SECRETS`<br>`RISK` | 새 비공개 진단 세트를 독점적으로 반영했습니다. |
| 6 | `7fbd41fe5af4` | test(operations): 자원·격리·삭제 보호·진단 검증 | **A** | `TEST`<br>`OPERATIONS`<br>`RISK` | 실행 제한, 네트워크 소속, 삭제 거부, 진단 안전성을 검증했습니다. |
| 7 | `98e4af62e884` | test(runtime): 프로세스·비밀값·정리 제어 흐름 강화 | **A** | `TEST`<br>`RECOVERY`<br>`OPERATIONS` | 테스트 정리 실패도 전체 검증 실패로 처리했습니다. |
| 8 | `2b35aa3d2217` | test(cleanup): 테스트 프로젝트 소유 자원만 정리 | **A** | `OPERATIONS`<br>`RECOVERY`<br>`RISK` | Tracked exact 프로젝트 소유권 and added scoped leak recovery. |
| 9 | `43ccded05e4f` | test(verify): 전체 스택 검증을 직렬 실행 | **A** | `TEST`<br>`OPERATIONS`<br>`RECOVERY` | Serialized the complete local verification lifecycle. |
| 10 | `18508c25eef0` | ci(stack): 정적·런타임·복구 검증 자동화 | **A** | `TEST`<br>`OPERATIONS`<br>`SUPPLY_CHAIN` | Automated all scenarios under least-privilege, pinned CI actions. |
| 11 | `8a6c07988160` | test(ci): workflow 검증 계약 추가 | **A** | `TEST`<br>`OPERATIONS`<br>`RISK` | 워크플로, 도구 제한 시간, 비밀값 경계, 정리, 산출물 허용 목록을 검증했습니다. |

> 커밋 순서는 원문의 개발 흐름 정의를 그대로 따릅니다. 같은 SHA가 다른 개발 흐름에도 포함되면 이 문서의 관점에서 다시 확인합니다.

## 커밋별 학습 기록

### 1. `27a3dca01d3b` — feat(network): DB 트래픽을 내부 backend로 격리

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `STACK`, `RISK`, `ARCH` |
| 원문에서 정한 역할 | Separated 공개 요청 traffic from the internal database 네트워크. |
| 이전 커밋 | 없음 |
| 다음 커밋 | `911544133fb4` |

#### 원문이 확정한 범위

<!-- 원문 요약: Splits frontend and backend networks, attaching MariaDB only to an internal backend. -->
<!-- 원문 판단 근거: This materially narrows the database communication boundary and makes WordPress the sole bridge between request-serving and persistence networks. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `27a3dca01d3b`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `networks: frontend / backend`에서 backend는 Docker 외부 route를 갖지 않는 database-only communication domain이 됩니다.
- `srcs/docker-compose.yml`의 `service memberships`에서 WordPress만 요청-serving과 persistence networks를 연결하는 애플리케이션 bridge입니다.
- `srcs/docker-compose.yml`의 `service-name routing`에서 필요한 통신만 유지하면서 Nginx→MariaDB 직접 addressability를 제거합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 27a3dca01d3b | srcs/docker-compose.yml | networks: frontend / backend | frontend bridge와 `internal: true` backend를 분리합니다. | backend는 Docker 외부 route를 갖지 않는 database-only communication domain이 됩니다. |
| 27a3dca01d3b | srcs/docker-compose.yml | 서비스 memberships | Nginx는 frontend만, MariaDB는 backend만, WordPress는 frontend와 backend 모두 join합니다. | WordPress만 요청-serving과 persistence networks를 연결하는 애플리케이션 bridge입니다. |
| 27a3dca01d3b | srcs/docker-compose.yml | 서비스-name routing | `fastcgi_pass wordpress:9000`과 WordPress의 MariaDB 서비스 name은 네트워크 split 뒤에도 각 shared 네트워크에서 resolve됩니다. | 필요한 통신만 유지하면서 Nginx→MariaDB 직접 addressability를 제거합니다. |

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 세 services가 하나의 shared 네트워크에 있으면 Nginx compromise/error가 MariaDB addressability까지 얻습니다. |
| 선택한 경계 / 결정 | 공개 요청 path와 internal DB path를 frontend/backend로 분리하고 exact 서비스 membership을 최소화했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `srcs/docker-compose.yml`의 `networks: frontend / backend`; `srcs/docker-compose.yml`의 `service memberships`; `srcs/docker-compose.yml`의 `service-name routing` |
| 상태·소유권·수명 변화 | Nginx는 frontend, MariaDB는 backend, WordPress는 두 네트워크의 애플리케이션 endpoint를 소유합니다. |
| 주요 실패 브랜치 | 네트워크 membership을 잘못 바꾸면 FastCGI 또는 DB DNS가 끊깁니다. 원본 declaration만으로 effective 런타임 membership을 증명하지 않습니다. |
| 이 커밋의 보장 | Nginx와 MariaDB가 네트워크를 공유하지 않고 WordPress만 양쪽과 통신하도록 reachability를 좁힙니다. |
| 한계와 다음 관련 커밋 | 컨테이너 내부 애플리케이션 exploit이 WordPress를 통해 DB에 접근하는 것을 제거하지는 않습니다. `7fbd41fe5af4`이 live Docker inspect로 exact membership과 Nginx→DB isolation을 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 컨테이너 내부 애플리케이션 exploit이 WordPress를 통해 DB에 접근하는 것을 제거하지는 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 live Docker inspect로 exact membership과 Nginx→DB isolation을 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: Nginx와 MariaDB가 네트워크를 공유하지 않고 WordPress만 양쪽과 통신하도록 reachability를 좁힙니다.

### 2. `911544133fb4` — feat(runtime): 서비스 자원과 종료 한계 적용

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **B** |
| 태그 | `OPERATIONS`, `RISK`, `STACK` |
| 원문에서 정한 역할 | Applied 자원, stop, privilege, and log-rotation policy. |
| 이전 커밋 | `27a3dca01d3b` |
| 다음 커밋 | `74c285925325` |

#### 원문이 확정한 범위

<!-- 원문 요약: Applies CPU, memory, PID, file-descriptor, stop-signal, privilege, and log-rotation limits to all services. -->
<!-- 원문 판단 근거: The policy is broad and useful, but it applies standard operational hardening to the already-defined runtime rather than changing core state or data flow. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `911544133fb4`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `srcs/docker-compose.yml`의 `cpus / mem_limit / pids_limit / ulimits`에서 한 서비스의 runaway 자원 use가 host 전체를 무제한 점유하지 못하게 합니다.
- `srcs/docker-compose.yml`의 `stop_signal / stop_grace_period`에서 서비스-specific graceful shutdown 시간을 Compose lifecycle에 반영합니다.
- `srcs/docker-compose.yml`의 `security_opt / logging`에서 privilege escalation과 unbounded local log growth를 줄입니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 911544133fb4 | srcs/docker-compose.yml | cpus / mem_limit / pids_limit / ulimits | Nginx, WordPress, MariaDB에 서비스별 CPU, memory, PID, nofile limits를 선언합니다. | 한 서비스의 runaway 자원 use가 host 전체를 무제한 점유하지 못하게 합니다. |
| 911544133fb4 | srcs/docker-compose.yml | stop_signal / stop_grace_period | Nginx/WordPress는 SIGQUIT와 짧은 grace, MariaDB는 SIGTERM과 더 긴 grace를 사용합니다. | 서비스-specific graceful shutdown 시간을 Compose lifecycle에 반영합니다. |
| 911544133fb4 | srcs/docker-compose.yml | security_opt / logging | `no-new-privileges:true`와 json-file `max-size`/`max-file` rotation을 적용합니다. | privilege escalation과 unbounded local log growth를 줄입니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 911544133fb4^ 911544133fb4 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 27a3dca01d3b 911544133fb4 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 B 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 개발 흐름에서 맡은 역할 | Applied 자원, stop, privilege, and log-rotation policy. |
| 핵심 입력 / 출력 / 상태 | Compose/Docker가 cgroup/rlimit/logging/shutdown policy를 소유하고 컨테이너 프로세스는 그 한계 안에서 실행됩니다. |
| 변경된 directive / helper / 명령 | `srcs/docker-compose.yml`의 `cpus / mem_limit / pids_limit / ulimits`; `srcs/docker-compose.yml`의 `stop_signal / stop_grace_period`; `srcs/docker-compose.yml`의 `security_opt / logging` |
| immediate 실패 또는 경계 | host 또는 Docker가 unsupported field를 무시하거나 다르게 적용할 수 있어 원본 declaration만으로 effective 상태를 확정할 수 없습니다. |
| 다음 커밋에 넘긴 한계 | 실제 런타임 적용 여부와 workload 적정성은 보장하지 않습니다. `7fbd41fe5af4`이 live inspect와 컨테이너 commands로 effective limits를 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 실제 런타임 적용 여부와 workload 적정성은 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 live inspect와 컨테이너 commands로 effective limits를 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 운영 자원/shutdown/log/security policy를 version-controlled Compose 규약으로 만듭니다.

### 3. `74c285925325` — fix(make): 볼륨 삭제 전에 확인을 요구

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `OPERATIONS`, `RISK`, `EDGE` |
| 원문에서 정한 역할 | Guarded destructive 볼륨 deletion with exact 프로젝트 confirmation. |
| 이전 커밋 | `911544133fb4` |
| 다음 커밋 | `ef74ad47ea81` |

#### 원문이 확정한 범위

<!-- 원문 요약: Requires an exact project-name confirmation before `fclean` deletes volumes and local images. -->
<!-- 원문 판단 근거: A very small diff protects the project's most destructive operator action and restores an important ownership and data-loss boundary. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `74c285925325`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `Makefile`의 `fclean guard`에서 generic yes/force가 아니라 삭제 대상 namespace를 operator가 다시 입력해야 합니다.
- `Makefile`의 `refusal branch`에서 wrong-프로젝트 변수나 copy/paste에서 fail closed합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 74c285925325 | Makefile | fclean guard | `DESTROY_CONFIRM`가 selected `PROJECT_NAME`과 exact string equality일 때만 `down --volumes`를 실행합니다. | generic yes/force가 아니라 삭제 대상 namespace를 operator가 다시 입력해야 합니다. |
| 74c285925325 | Makefile | refusal 브랜치 | 누락 또는 mismatch면 설명을 출력하고 nonzero로 종료하며 destructive Docker 명령을 호출하지 않습니다. | wrong-프로젝트 변수나 copy/paste에서 fail closed합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 74c285925325^ 74c285925325 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 911544133fb4 74c285925325 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### Fix chain 기록

| 단계 | 학습자 기록 |
| --- | --- |
| 기존 가정 | fclean을 일반 정리와 비슷한 low-risk 작업으로 취급했습니다. |
| 실제 실패 또는 위험 | 프로젝트 variable이 잘못되면 unrelated 영속 volumes까지 즉시 삭제됩니다. |
| root cause | destructive scope를 operator가 명령 실행 시 다시 확인하는 guard가 없었습니다. |
| 수정된 불변식 / 결정 | selected 프로젝트 name과 exact equality인 confirmation만 볼륨 deletion을 허용합니다. |
| 실제 수정 코드 | `Makefile`의 `fclean guard`; `Makefile`의 `refusal branch` |
| 변경된 ordering / 소유권 / lifecycle | operator/Make variables가 대상 프로젝트 identity를 소유하고 destructive 명령은 exact confirmation 뒤에만 자원 lifecycle을 종료합니다. |
| 이 fix가 보장하는 것 | generic 확인보다 프로젝트 identity를 재확인하는 bounded destructive 작업을 제공합니다. |
| 아직 보장하지 않는 것 | 악의적/부주의한 operator가 exact name을 입력한 뒤 잘못 삭제하는 것까지 막지 못합니다. |
| 연결되는 회귀 테스트 | operations 런타임 테스트가 mismatch refusal과 running stack 헬스 상태/상태 보존을 확인합니다. `7fbd41fe5af4`이 refusal 뒤 stack/volumes/헬스 상태가 유지되는지 런타임에서 확인합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 악의적/부주의한 operator가 exact name을 입력한 뒤 잘못 삭제하는 것까지 막지 못합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 refusal 뒤 stack/volumes/헬스 상태가 유지되는지 런타임에서 확인합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: generic 확인보다 프로젝트 identity를 재확인하는 bounded destructive 작업을 제공합니다.

### 4. `ef74ad47ea81` — feat(diagnostics): Compose 비밀값과 민감 항목 마스킹

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `OPERATIONS`, `SECRETS`, `RISK` |
| 원문에서 정한 역할 | 진단 정보 마스킹이 불완전하면 실패하도록 만들었습니다. |
| 이전 커밋 | `74c285925325` |
| 다음 커밋 | `27a083d91c87` |

#### 원문이 확정한 범위

<!-- 원문 요약: Derives secret paths and values from rendered Compose configuration and defines fail-closed redaction of credentials and sensitive assignments. -->
<!-- 원문 판단 근거: Diagnostics can themselves become a leakage channel; this commit establishes the critical rule that collection stops when required secrets cannot be read and redacted. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `ef74ad47ea81`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/diagnose_stack.py`의 `build_redaction_values`에서 path와 value가 logs/config에 나타나는 여러 표현을 masking set에 넣습니다.
- `tools/diagnose_stack.py`의 `longer-first literal redaction`에서 literal 비밀값/path leakage를 결정적하게 제거합니다.
- `tools/diagnose_stack.py`의 `structural sensitive-field masking`에서 알려진 exact value 외의 민감 출력도 줄입니다.
- `tools/diagnose_stack.py`의 `fail-closed input boundary`에서 부분 sanitize된 bundle을 신뢰 가능한 것으로 게시하지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| ef74ad47ea81 | tools/diagnose_stack.py | build_redaction_values | rendered Compose에서 비밀값 원본 paths를 해석하고 raw path, resolved path, 비밀값 contents를 hardened reader로 모두 수집합니다. | path와 value가 logs/config에 나타나는 여러 표현을 masking set에 넣습니다. |
| ef74ad47ea81 | tools/diagnose_stack.py | longer-first literal redaction | masking values를 길이 내림차순으로 치환해 긴 credential 안의 짧은 substring이 먼저 바뀌어 원문 일부가 남는 문제를 피합니다. | literal 비밀값/path leakage를 결정적하게 제거합니다. |
| ef74ad47ea81 | tools/diagnose_stack.py | structural sensitive-field masking | password/token/비밀값/key 계열 대입과 JSON/YAML-like sensitive fields를 pattern으로 추가 마스킹합니다. | 알려진 exact value 외의 민감 출력도 줄입니다. |
| ef74ad47ea81 | tools/diagnose_stack.py | fail-closed 입력 경계 | 비밀값 file 하나라도 안전하게 읽지 못하면 masking set을 불완전하게 만든 채 진행하지 않고 diagnostics 전체를 실패시킵니다. | 부분 sanitize된 bundle을 신뢰 가능한 것으로 게시하지 않습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff ef74ad47ea81^ ef74ad47ea81 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 74c285925325 ef74ad47ea81 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | Compose config, inspect, logs에는 credential value뿐 아니라 host 비밀값 path나 sensitive 대입이 포함될 수 있어 원문 수집이 정보 유출을 만들었습니다. |
| 선택한 경계 / 결정 | hardened 비밀값 reader로 complete masking set을 만들고 literal longer-first와 structural masking을 결합했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/diagnose_stack.py`의 `build_redaction_values`; `tools/diagnose_stack.py`의 `longer-first literal redaction`; `tools/diagnose_stack.py`의 `structural sensitive-field masking`; `tools/diagnose_stack.py`의 `fail-closed input boundary` |
| 상태·소유권·수명 변화 | diagnostics 프로세스가 raw capture와 masking values를 memory에서 일시 소유하며 sanitized text만 반영 layer로 넘깁니다. |
| 주요 실패 브랜치 | 비밀값 원본을 안전하게 읽지 못하거나 sanitize가 실패하면 아무 diagnostic 출력도 신뢰하지 않고 실패로 처리합니다. |
| 이 커밋의 보장 | complete known 비밀값/path set과 sensitive field patterns을 모두 마스킹한 text만 다음 단계로 보낼 수 있습니다. |
| 한계와 다음 관련 커밋 | unknown semantic 비밀값, encoded/encrypted/compressed 표현, side 채널까지 모두 탐지하지는 않습니다. `27a083d91c87`이 비공개 exclusive 출력 set과 final rescan/정리를 연결하고 `7fbd41fe5af4`이 실제 비밀값 log를 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: unknown semantic 비밀값, encoded/encrypted/compressed 표현, side 채널까지 모두 탐지하지는 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `27a083d91c87`이 비공개 exclusive 출력 set과 final rescan/정리를 연결하고 `7fbd41fe5af4`이 실제 비밀값 log를 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: complete known 비밀값/path set과 sensitive field patterns을 모두 마스킹한 text만 다음 단계로 보낼 수 있습니다.

### 5. `27a083d91c87` — feat(diagnostics): 비공개 진단 세트와 CLI 연결

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `OPERATIONS`, `SECRETS`, `RISK` |
| 원문에서 정한 역할 | 새 비공개 진단 세트를 독점적으로 반영했습니다. |
| 이전 커밋 | `ef74ad47ea81` |
| 다음 커밋 | `7fbd41fe5af4` |

#### 원문이 확정한 범위

<!-- 원문 요약: Publishes an exclusive private diagnostic directory with allowlisted, redacted Compose, log, version, and container-state files. -->
<!-- 원문 판단 근거: This completes a safe observability mechanism: failure evidence becomes actionable without overwriting existing output or exposing credential material. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `27a083d91c87`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/diagnose_stack.py`의 `create_output_directory`에서 diagnostic set이 기존 directory를 덮어쓰거나 공격자 path를 따라가지 않습니다.
- `tools/diagnose_stack.py`의 `allowlisted private files`에서 bundle contents와 confidentiality가 좁은 allowlist로 고정됩니다.
- `tools/diagnose_stack.py`의 `post-write rescan / cleanup on error`에서 partial 또는 unsanitized bundle을 남기지 않는 반영 endpoint입니다.
- `Makefile / CLI`의 `diagnostics command`에서 operator path와 테스트 path가 같은 safety logic을 사용합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 27a083d91c87 | tools/diagnose_stack.py | create_output_directory | 출력 parent를 검증하고 대상 directory를 `0700`으로 exclusive create하며 existing path나 symlink를 거부합니다. | diagnostic set이 기존 directory를 덮어쓰거나 공격자 path를 따라가지 않습니다. |
| 27a083d91c87 | tools/diagnose_stack.py | allowlisted 비공개 files | 정해진 compose config/ps/inspect/log/메타데이터 files만 `0600` O_EXCL로 작성합니다. | bundle contents와 confidentiality가 좁은 allowlist로 고정됩니다. |
| 27a083d91c87 | tools/diagnose_stack.py | post-write rescan / 정리 on error | 모든 출력을 다시 비밀값/path/pattern으로 scan하고 하나라도 발견되거나 명령/write가 실패하면 출력 directory 전체를 제거합니다. | partial 또는 unsanitized bundle을 남기지 않는 반영 endpoint입니다. |
| 27a083d91c87 | Makefile / CLI | diagnostics 명령 | 프로젝트/env/출력을 명시해 같은 production collector를 호출하는 documented entry point를 제공합니다. | operator path와 테스트 path가 같은 safety logic을 사용합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 27a083d91c87^ 27a083d91c87 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff ef74ad47ea81 27a083d91c87 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | redaction function만 있어도 출력 path overwrite, broad permissions, partial file set, sanitize 후 leakage를 막지 못했습니다. |
| 선택한 경계 / 결정 | 새 비공개 directory와 allowlisted exclusive files에만 sanitized data를 쓰고 전체 rescan 후 성공으로 게시하도록 했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/diagnose_stack.py`의 `create_output_directory`; `tools/diagnose_stack.py`의 `allowlisted private files`; `tools/diagnose_stack.py`의 `post-write rescan / cleanup on error`; `Makefile / CLI`의 `diagnostics command` |
| 상태·소유권·수명 변화 | diagnostics 명령이 출력 directory lifecycle 전체를 소유하며 성공한 complete 비공개 set만 caller에게 넘깁니다. |
| 주요 실패 브랜치 | existing/symlink 대상, any capture/redaction/write/rescan 실패는 directory를 삭제하고 nonzero로 종료합니다. |
| 이 커밋의 보장 | diagnostic set은 비공개, non-overwriting, allowlisted, fully redacted이며 sanitize 불가 시 아무것도 게시하지 않습니다. |
| 한계와 다음 관련 커밋 | masking model이 모르는 새로운 encoding의 비밀값은 자동 보장하지 않습니다. `7fbd41fe5af4`이 unreadable 비밀값, overwrite/symlink, real log 비밀값, permissions/file set을 런타임 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: masking model이 모르는 새로운 encoding의 비밀값은 자동 보장하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `7fbd41fe5af4`이 unreadable 비밀값, overwrite/symlink, real log 비밀값, permissions/file set을 런타임 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: diagnostic set은 비공개, non-overwriting, allowlisted, fully redacted이며 sanitize 불가 시 아무것도 게시하지 않습니다.

### 6. `7fbd41fe5af4` — test(operations): 자원·격리·삭제 보호·진단 검증

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `OPERATIONS`, `RISK` |
| 원문에서 정한 역할 | 실행 제한, 네트워크 소속, 삭제 거부, 진단 안전성을 검증했습니다. |
| 이전 커밋 | `27a083d91c87` |
| 다음 커밋 | `98e4af62e884` |

#### 원문이 확정한 범위

<!-- 원문 요약: Verifies runtime limits, network membership, destructive-action refusal, fail-closed redaction, file permissions, overwrite refusal, and symlink-output rejection. -->
<!-- 원문 판단 근거: The scenario materially validates several operational and security boundaries that configuration inspection alone cannot prove. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `7fbd41fe5af4`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `operations inspect scenario`에서 Compose declaration과 effective 런타임 상태를 대조합니다.
- `tests/runtime_stack.py`의 `fclean refusal`에서 guard refusal이 실제 변경 0임을 증명합니다.
- `tests/runtime_stack.py`의 `diagnostic redaction fixtures`에서 literal/structural masking과 fail-closed 반영을 live data로 통과합니다.
- `tests/runtime_stack.py`의 `diagnostic file/perms/rescan assertions`에서 비공개 반영 endpoint를 검증합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 7fbd41fe5af4 | tests/runtime_stack.py | operations inspect scenario | live 컨테이너의 networks, memory/CPU/PID/nofile, stop signal/grace, no-new-privileges, log driver/options를 Docker inspect와 in-컨테이너 probes로 확인합니다. | Compose declaration과 effective 런타임 상태를 대조합니다. |
| 7fbd41fe5af4 | tests/runtime_stack.py | fclean refusal | 잘못된/누락 confirmation으로 destructive 대상을 실행하고 volumes와 애플리케이션 헬스 상태/상태가 그대로인지 확인합니다. | guard refusal이 실제 변경 0임을 증명합니다. |
| 7fbd41fe5af4 | tests/runtime_stack.py | diagnostic redaction 테스트 준비 코드 | 실제 비밀값을 요청/log에 넣고 unreadable 비밀값, existing directory, dangling symlink cases를 만든 뒤 success/실패 출력을 검사합니다. | literal/structural masking과 fail-closed 반영을 live data로 통과합니다. |
| 7fbd41fe5af4 | tests/runtime_stack.py | diagnostic file/perms/rescan 검사문 | 성공 set의 exact allowlist, 0700/0600 mode, raw/resolved 비밀값 path와 content 부재를 확인합니다. | 비공개 반영 endpoint를 검증합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 7fbd41fe5af4^ 7fbd41fe5af4 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 27a083d91c87 7fbd41fe5af4 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | declared operational policy가 effective 컨테이너에 적용되고 destructive/diagnostic commands는 unsafe case에서 변경/출력 없이 실패합니다. |
| 재현하는 실패 / 경계 | 네트워크/limit mismatch, wrong fclean confirmation, real 비밀값 log, unreadable 비밀값, existing/symlink 출력입니다. |
| 테스트 방식 | live Docker inspection + negative filesystem/명령 통합 |
| 테스트 준비 코드와 실패 주입 | healthy isolated stack, 비밀값-bearing 요청/log, permission/collision 출력 테스트 준비 코드를 만듭니다. |
| 실제 통과하는 실제 실행 경로 | Compose 컨테이너→Docker inspect/in-컨테이너 probes→Make fclean→diagnose_stack capture/redact/publish를 통과합니다. |
| 핵심 검사문 | exact networks/limits/stop/log/security, unchanged 헬스 상태/volumes, exact 비공개 redacted bundle 또는 출력 부재를 확인합니다. |
| 이 테스트가 증명하는 것 | 운영 policy의 원본-to-effective-상태 연결과 fail-closed destructive/diagnostic 동작을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 모든 platform과 unknown leakage 채널을 증명하지 않습니다. |
| 성격 | broad operational 통합 + negative 회귀 |
| 막는 후속 회귀 | 네트워크 widening, ignored limits, generic destructive confirmation, partial/unsanitized diagnostics를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 모든 host kernel/Docker version, unknown 비밀값 encoding, production load 적정성은 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `98e4af62e884`과 후속 커밋이 scenario 정리 자체를 verified lifecycle로 강화합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 네트워크/limit/stop/log/security effective 상태, exact-프로젝트 deletion guard, fail-closed 비공개 diagnostics가 실제 stack에서 동작함을 증명합니다.

### 7. `98e4af62e884` — test(runtime): 프로세스·비밀값·정리 제어 흐름 강화

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `RECOVERY`, `OPERATIONS` |
| 원문에서 정한 역할 | 테스트 정리 실패도 전체 검증 실패로 처리했습니다. |
| 이전 커밋 | `7fbd41fe5af4` |
| 다음 커밋 | `2b35aa3d2217` |

#### 원문이 확정한 범위

<!-- 원문 요약: Makes private fixture replacement durable, separates start command construction, improves timeout diagnostics, and treats cleanup failure as test failure. -->
<!-- 원문 판단 근거: It strengthens the verification control plane so successful scenarios cannot hide leaked resources or incomplete teardown, a significant reliability property for the extensive runtime suite. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `98e4af62e884`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `explicit subprocess wait timeouts`에서 hung child가 verification 프로세스를 무기한 점유하지 않습니다.
- `tests/runtime_stack.py`의 `RuntimeStack.close error accumulation`에서 scenario body success가 정리 실패를 덮지 않습니다.
- `tests/runtime_stack.py`의 `main result precedence`에서 근거 lifecycle 전체가 성공해야 프로세스 success입니다.
- `tools/rotate_secrets.py / related private writes`의 `private temp fsync/cleanup hardening`에서 verification이 관측하는 자원 상태와 host file durability를 맞춥니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 98e4af62e884 | tests/runtime_stack.py | explicit subprocess wait timeouts | Popen communicate/wait/terminate/kill paths에 bounded timeout과 escalation을 추가합니다. | hung child가 verification 프로세스를 무기한 점유하지 않습니다. |
| 98e4af62e884 | tests/runtime_stack.py | RuntimeStack.close error accumulation | teardown, 이미지 removal, temp 정리의 실패를 수집하고 primary scenario outcome과 합칩니다. | scenario body success가 정리 실패를 덮지 않습니다. |
| 98e4af62e884 | tests/runtime_stack.py | main result precedence | scenario exception, diagnostics 실패, 정리 실패, unexpected exception의 exit status와 report order를 명시합니다. | 근거 lifecycle 전체가 성공해야 프로세스 success입니다. |
| 98e4af62e884 | tools/rotate_secrets.py / related 비공개 writes | 비공개 temp fsync/정리 hardening | 테스트가 의존하는 비밀값/config 임시 객체 반영의 sync/정리 규약도 강화합니다. | verification이 관측하는 자원 상태와 host file durability를 맞춥니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 98e4af62e884^ 98e4af62e884 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 7fbd41fe5af4 98e4af62e884 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 테스트 body가 pass해도 teardown subprocess가 hang/fail하거나 diagnostics가 실패해 Docker 자원/secrets가 남으면 전체 verification은 신뢰할 수 없습니다. |
| 선택한 경계 / 결정 | 모든 child wait를 bounded하고 정리 errors를 primary result에 포함하는 explicit precedence를 만들었습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tests/runtime_stack.py`의 `explicit subprocess wait timeouts`; `tests/runtime_stack.py`의 `RuntimeStack.close error accumulation`; `tests/runtime_stack.py`의 `main result precedence`; `tools/rotate_secrets.py / related private writes`의 `private temp fsync/cleanup hardening` |
| 상태·소유권·수명 변화 | RuntimeStack과 top-level main이 child processes, 프로젝트 자원, diagnostics, 임시 객체 files의 full lifecycle을 소유합니다. |
| 주요 실패 브랜치 | scenario success + 정리 실패는 nonzero입니다. primary 실패와 정리 실패가 함께 있으면 둘 다 보고하며 unexpected exception도 finally 정리를 거칩니다. |
| 이 커밋의 보장 | 테스트 outcome이 scenario 검사문뿐 아니라 bounded 프로세스 termination과 successful 정리까지 포함합니다. |
| 한계와 다음 관련 커밋 | 프로세스 crash 전에 소유권 record가 남지 않은 자원을 모두 찾는 기능은 아직 제한됩니다. `2b35aa3d2217`이 exact 소유권 records와 crash recovery utility를 추가하고 `43ccded05e4f`이 full 직렬 lifecycle을 구성합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: 프로세스 crash 전에 소유권 record가 남지 않은 자원을 모두 찾는 기능은 아직 제한됩니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `2b35aa3d2217`이 exact 소유권 records와 crash recovery utility를 추가하고 `43ccded05e4f`이 full 직렬 lifecycle을 구성합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 테스트 outcome이 scenario 검사문뿐 아니라 bounded 프로세스 termination과 successful 정리까지 포함합니다.

### 8. `2b35aa3d2217` — test(cleanup): 테스트 프로젝트 소유 자원만 정리

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `OPERATIONS`, `RECOVERY`, `RISK` |
| 원문에서 정한 역할 | Tracked exact 프로젝트 소유권 and added scoped leak recovery. |
| 이전 커밋 | `98e4af62e884` |
| 다음 커밋 | `43ccded05e4f` |

#### 원문이 확정한 범위

<!-- 원문 요약: Records exact test project ownership, removes only owned Compose resources and image tags, and adds a scoped crash-recovery cleanup tool with private reports. -->
<!-- 원문 판단 근거: This solves a high-risk verification-lifecycle problem without broad Docker pruning, ensuring failed tests cannot damage unrelated developer or CI resources. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `2b35aa3d2217`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/runtime_stack.py`의 `ownership record publication`에서 프로세스 crash 뒤에도 어떤 프로젝트/이미지 identities가 테스트-owned인지 남깁니다.
- `tools/cleanup_test_resources.py`의 `record validation / discovery`에서 malformed/untrusted record로 unrelated 자원을 삭제하지 않습니다.
- `tools/cleanup_test_resources.py`의 `scoped removal and leak report`에서 crash recovery scope가 explicit 소유권에 제한됩니다.
- `tests/runtime_stack.py`의 `normal close record lifecycle`에서 normal과 crash 정리가 같은 identity 원본을 공유합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 2b35aa3d2217 | tests/runtime_stack.py | 소유권 record 반영 | 테스트 프로젝트 name과 이미지 앞부분을 비공개 record directory `0700`의 strict `0600` files에 기록합니다. | 프로세스 crash 뒤에도 어떤 프로젝트/이미지 identities가 테스트-owned인지 남깁니다. |
| 2b35aa3d2217 | tools/cleanup_test_resources.py | record validation / discovery | record 소유자/mode/type/content를 검증하고 exact 프로젝트 labels/names와 이미지 repository/tag 앞부분만 query합니다. | malformed/untrusted record로 unrelated 자원을 삭제하지 않습니다. |
| 2b35aa3d2217 | tools/cleanup_test_resources.py | scoped removal and leak report | recorded projects를 down/remove하고 exact images만 지운 뒤 residuals를 보고하며 global Docker prune을 사용하지 않습니다. | crash recovery scope가 explicit 소유권에 제한됩니다. |
| 2b35aa3d2217 | tests/runtime_stack.py | normal close record lifecycle | 정상 teardown이 끝나면 corresponding 소유권 record를 제거하고 정리 실패면 report를 보존합니다. | normal과 crash 정리가 같은 identity 원본을 공유합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 2b35aa3d2217^ 2b35aa3d2217 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 98e4af62e884 2b35aa3d2217 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 테스트 프로세스가 crash하면 in-memory 프로젝트 list가 사라져 orphaned 컨테이너/volumes/networks/images를 안전하게 찾기 어렵고 broad prune은 unrelated 자원을 삭제합니다. |
| 선택한 경계 / 결정 | 비공개 strict 소유권 records와 label/tag-scoped recovery tool을 도입했습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tests/runtime_stack.py`의 `ownership record publication`; `tools/cleanup_test_resources.py`의 `record validation / discovery`; `tools/cleanup_test_resources.py`의 `scoped removal and leak report`; `tests/runtime_stack.py`의 `normal close record lifecycle` |
| 상태·소유권·수명 변화 | 테스트 harness가 record 반영/removal을, recovery utility가 validated records에 해당하는 Docker 자원만 소유합니다. |
| 주요 실패 브랜치 | unsafe/malformed records는 삭제 작업 전에 거부됩니다. 일부 removal 실패나 residual leak는 report와 nonzero로 surfaced됩니다. |
| 이 커밋의 보장 | crash 뒤에도 테스트가 명시적으로 소유한 프로젝트/이미지 scope만 정리하고 unrelated Docker objects를 보존합니다. |
| 한계와 다음 관련 커밋 | record 반영 전 crash하거나 external actor가 소유권 label/tag를 위조한 경우까지 완전 해결하지는 않습니다. `43ccded05e4f`과 CI가 preparation/finally에서 recovery utility를 호출합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: record 반영 전 crash하거나 external actor가 소유권 label/tag를 위조한 경우까지 완전 해결하지는 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `43ccded05e4f`과 CI가 preparation/finally에서 recovery utility를 호출합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: crash 뒤에도 테스트가 명시적으로 소유한 프로젝트/이미지 scope만 정리하고 unrelated Docker objects를 보존합니다.

### 9. `43ccded05e4f` — test(verify): 전체 스택 검증을 직렬 실행

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `OPERATIONS`, `RECOVERY` |
| 원문에서 정한 역할 | Serialized the complete local verification lifecycle. |
| 이전 커밋 | `2b35aa3d2217` |
| 다음 커밋 | `18508c25eef0` |

#### 원문이 확정한 범위

<!-- 원문 요약: Runs static checks and all runtime scenarios serially with per-scenario timeouts, shared project records, and mandatory final leak recovery. -->
<!-- 원문 판단 근거: It defines the complete local verification transaction and makes resource cleanliness part of success, significantly improving confidence and failure attribution. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `43ccded05e4f`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tools/verify_stack.py`의 `serial scenario plan`에서 공유 host Docker capacity와 logs/records가 concurrent scenarios로 섞이지 않습니다.
- `tools/verify_stack.py`의 `per-step timeout/result handling`에서 hang/실패가 unbounded 파이프라인이나 정리 skip으로 이어지지 않습니다.
- `tools/verify_stack.py`의 `pre/post crash recovery`에서 이전 crash와 현재 run leak를 같은 bounded lifecycle에서 처리합니다.
- `tools/verify_stack.py`의 `outcome precedence`에서 green result가 자원 leak를 숨기지 않습니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 43ccded05e4f | tools/verify_stack.py | 직렬 scenario plan | 정적 validation과 config 렌더링 뒤 초기화, e2e, persistence, backup/restore, rotation, operations scenarios를 정해진 순서로 한 번에 하나씩 실행합니다. | 공유 host Docker capacity와 logs/records가 concurrent scenarios로 섞이지 않습니다. |
| 43ccded05e4f | tools/verify_stack.py | per-step timeout/result handling | 각 명령에 explicit timeout을 주고 실패 시 diagnostics를 수집하되 remaining/final 정리 semantics를 보존합니다. | hang/실패가 unbounded 파이프라인이나 정리 skip으로 이어지지 않습니다. |
| 43ccded05e4f | tools/verify_stack.py | pre/post crash recovery | 시작 전 stale 소유권 records를 정리하고 finally에서 정리 utility와 residual report를 실행합니다. | 이전 crash와 현재 run leak를 같은 bounded lifecycle에서 처리합니다. |
| 43ccded05e4f | tools/verify_stack.py | outcome precedence | scenario 실패, diagnostic 실패, 정리 leak 중 하나라도 final nonzero를 만들며 정리 report를 primary result와 함께 보존합니다. | green result가 자원 leak를 숨기지 않습니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 43ccded05e4f^ 43ccded05e4f -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 2b35aa3d2217 43ccded05e4f -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | 개별 테스트 commands는 있었지만 operator가 일부만 실행하거나 parallel/interrupted run에서 stale 자원과 result precedence를 일관되게 관리하기 어려웠습니다. |
| 선택한 경계 / 결정 | 전체 verification을 직렬 ordered plan, bounded subprocess, diagnostics, pre/final 정리로 묶었습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `tools/verify_stack.py`의 `serial scenario plan`; `tools/verify_stack.py`의 `per-step timeout/result handling`; `tools/verify_stack.py`의 `pre/post crash recovery`; `tools/verify_stack.py`의 `outcome precedence` |
| 상태·소유권·수명 변화 | verify orchestrator가 local 근거 lifecycle과 명령 order를 소유하고 각 scenario는 자신의 프로젝트 자원만 소유합니다. |
| 주요 실패 브랜치 | 어느 단계 실패/timeout/unexpected exception에서도 finally 정리를 실행하고 정리 실패도 nonzero에 포함합니다. |
| 이 커밋의 보장 | 하나의 local 명령이 정적부터 모든 런타임 scenarios, diagnostics, leak recovery를 직렬 bounded lifecycle로 실행합니다. |
| 한계와 다음 관련 커밋 | machine crash로 finally가 실행되지 않는 경우는 다음 run의 소유권-record recovery에 의존합니다. `18508c25eef0`이 같은 lifecycle을 least-privilege CI에 자동화하고 `8a6c07988160`이 orchestrator/workflow 자체를 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: machine crash로 finally가 실행되지 않는 경우는 다음 run의 소유권-record recovery에 의존합니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `18508c25eef0`이 같은 lifecycle을 least-privilege CI에 자동화하고 `8a6c07988160`이 orchestrator/workflow 자체를 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: 하나의 local 명령이 정적부터 모든 런타임 scenarios, diagnostics, leak recovery를 직렬 bounded lifecycle로 실행합니다.

### 10. `18508c25eef0` — ci(stack): 정적·런타임·복구 검증 자동화

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `OPERATIONS`, `SUPPLY_CHAIN` |
| 원문에서 정한 역할 | Automated all scenarios under least-privilege, pinned CI actions. |
| 이전 커밋 | `43ccded05e4f` |
| 다음 커밋 | `8a6c07988160` |

#### 원문이 확정한 범위

<!-- 원문 요약: Adds a least-privilege, pinned-action GitHub Actions workflow running all static and runtime scenarios, scoped cleanup, and allowlisted failure diagnostics. -->
<!-- 원문 판단 근거: This is significant integration of the project's verification, supply-chain, and resource-ownership policies into automation, though it does not alter product runtime behavior. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `18508c25eef0`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `.github/workflows/container-stack.yml`의 `runner/timeout/permissions/concurrency`에서 workflow token과 execution duration을 최소/한정합니다.
- `.github/workflows/container-stack.yml`의 `pinned actions / full checkout`에서 CI dependency와 historical verification 입력을 명시합니다.
- `.github/workflows/container-stack.yml`의 `serial verification / always cleanup`에서 local lifecycle의 정리/result semantics를 CI에도 유지합니다.
- `.github/workflows/container-stack.yml`의 `artifact allowlist`에서 CI artifact가 새로운 disclosure 채널이 되지 않게 합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 18508c25eef0 | .github/workflows/container-stack.yml | runner/timeout/permissions/concurrency | Ubuntu 24.04 runner, 전체 210분 timeout, `contents: read`, concurrency cancel policy를 선언합니다. | workflow token과 execution duration을 최소/한정합니다. |
| 18508c25eef0 | .github/workflows/container-stack.yml | pinned actions / full 체크아웃 | 체크아웃과 artifact upload를 immutable 커밋 SHA로 고정하고 full history를 받아 커밋-range/원본 checks를 가능하게 합니다. | CI dependency와 historical verification 입력을 명시합니다. |
| 18508c25eef0 | .github/workflows/container-stack.yml | 직렬 verification / always 정리 | 정적/config/런타임 scenarios를 직렬 실행하고 실패와 무관하게 diagnostics/소유권 정리를 `always()` path에서 수행합니다. | local lifecycle의 정리/result semantics를 CI에도 유지합니다. |
| 18508c25eef0 | .github/workflows/container-stack.yml | artifact allowlist | 실패 근거는 redacted diagnostics와 정리 reports의 좁은 path만 업로드하고 비밀값/env dump를 포함하지 않습니다. | CI artifact가 새로운 disclosure 채널이 되지 않게 합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 18508c25eef0^ 18508c25eef0 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 43ccded05e4f 18508c25eef0 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 중요도 A 학습 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 직전 관련 상태와 문제 | local verify만으로는 모든 변경에 자동 실행되지 않고 CI가 broad token/action tags/unbounded 런타임/unsafe artifact를 사용하면 verification 자체가 위험해질 수 있었습니다. |
| 선택한 경계 / 결정 | least privilege, pinned actions, bounded timeout, 직렬 scenarios, always 정리, allowlisted 실패 근거를 workflow policy로 만들었습니다. |
| 핵심 호출자/피호출자 또는 configuration consumer | `.github/workflows/container-stack.yml`의 `runner/timeout/permissions/concurrency`; `.github/workflows/container-stack.yml`의 `pinned actions / full checkout`; `.github/workflows/container-stack.yml`의 `serial verification / always cleanup`; `.github/workflows/container-stack.yml`의 `artifact allowlist` |
| 상태·소유권·수명 변화 | CI job이 ephemeral runner의 체크아웃/Docker 자원/artifacts lifecycle을 소유하며 workflow token은 read-only contents 범위만 가집니다. |
| 주요 실패 브랜치 | scenario 실패, timeout, cancellation에서도 정리 steps가 실행되며 diagnostics/정리 실패는 job status/근거에 반영됩니다. |
| 이 커밋의 보장 | repository change마다 동일 정적/런타임/recovery suite를 bounded least-privilege 환경에서 자동 실행합니다. |
| 한계와 다음 관련 커밋 | workflow text가 미래 편집으로 약화되지 않는다는 보장은 별도 self-테스트 없이는 없습니다. `8a6c07988160`이 workflow, tools, 비밀값/timeout/정리 규약을 정적/AST/mock layers로 검증합니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: workflow text가 미래 편집으로 약화되지 않는다는 보장은 별도 self-테스트 없이는 없습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: `8a6c07988160`이 workflow, tools, 비밀값/timeout/정리 규약을 정적/AST/mock layers로 검증합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: repository change마다 동일 정적/런타임/recovery suite를 bounded least-privilege 환경에서 자동 실행합니다.

### 11. `8a6c07988160` — test(ci): workflow 검증 계약 추가

| 항목 | 원문 확정값 |
| --- | --- |
| 중요도 | **A** |
| 태그 | `TEST`, `OPERATIONS`, `RISK` |
| 원문에서 정한 역할 | 워크플로, 도구 제한 시간, 비밀값 경계, 정리, 산출물 허용 목록을 검증했습니다. |
| 이전 커밋 | `18508c25eef0` |
| 다음 커밋 | 없음 |

#### 원문이 확정한 범위

<!-- 원문 요약: Expands static and AST-based checks to enforce workflow permissions, action pins, scenario ordering, timeouts, secret boundaries, cleanup semantics, and safe subprocess use. -->
<!-- 원문 판단 근거: The commit protects the verification system itself from subtle weakening and provides layered evidence for security and lifecycle properties across many tools. -->

#### 해당 SHA에서 확인할 코드

> 기준 커밋은 반드시 `8a6c07988160`입니다. 최종 HEAD의 동일 파일을 근거로 대체하지 않습니다.

- `tests/validate_stack.py`의 `workflow text contract`에서 CI policy의 필수 요소를 원본 회귀로 고정합니다.
- `tests/validate_stack.py`의 `forbidden workflow patterns`에서 검증 infrastructure의 high-risk shortcuts를 fail closed합니다.
- `tests/validate_stack.py`의 `Compose service-block parser`에서 global substring보다 서비스 responsibility를 정확히 검증합니다.
- `tests/validate_stack.py`의 `Python AST visitors`에서 text 검색이 놓칠 management-tool semantic weakening을 탐지합니다.
- `tests/validate_stack.py`의 `mocked main-path probes`에서 Docker 없이도 orchestrator 실패 semantics 일부를 결정적하게 확인합니다.

#### 코드 근거

| SHA | 경로 | symbol / directive / 명령 | 확인한 line 또는 최소 코드 | 이 코드가 증명하는 사실 |
| --- | --- | --- | --- | --- |
| 8a6c07988160 | tests/validate_stack.py | workflow text 규약 | runner, timeout, permissions, concurrency, action SHAs, full 체크아웃, 직렬 commands, diagnostics, always 정리, artifact allowlist를 exact pattern으로 검사합니다. | CI policy의 필수 요소를 원본 회귀로 고정합니다. |
| 8a6c07988160 | tests/validate_stack.py | forbidden workflow patterns | `pull_request_target`, 비밀값 contexts, shell tracing, 환경 dumping, broad Docker prune, unpinned action references를 거부합니다. | 검증 infrastructure의 high-risk shortcuts를 fail closed합니다. |
| 8a6c07988160 | tests/validate_stack.py | Compose 서비스-block parser | 런타임 서비스별 exact mounts/env/networks를 parse해 password env, `/run/secrets`, Nginx 비공개 config 마운트 등 forbidden exposure를 검사합니다. | global substring보다 서비스 responsibility를 정확히 검증합니다. |
| 8a6c07988160 | tests/validate_stack.py | Python AST visitors | subprocess wait/communicate의 explicit timeout과 startup 비밀값 read가 프로젝트 잠금 lexical/control-flow 안에 있는지 AST로 검사합니다. | text 검색이 놓칠 management-tool semantic weakening을 탐지합니다. |
| 8a6c07988160 | tests/validate_stack.py | mocked main-path probes | preparation timeout, scenario 실패, 정리 실패, unexpected exception을 mock해 exit status와 정리 invocation/result precedence를 검사합니다. | Docker 없이도 orchestrator 실패 semantics 일부를 결정적하게 확인합니다. |

#### 비교 기준

- 해당 커밋의 변경 내역: `git diff 8a6c07988160^ 8a6c07988160 -- <path>`
- 이전 개발 흐름 상태와 비교: `git diff 18508c25eef0 8a6c07988160 -- <path>`
- 두 diff가 보여주는 범위가 다르면 각각 따로 기록합니다.

#### 테스트 학습 기록

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 실제 코드의 불변식 | CI와 management/테스트 tools 자체가 least-privilege, pinned, bounded, 비밀값-safe, scoped-정리/result-precedence 규약을 유지합니다. |
| 재현하는 실패 / 경계 | workflow/tool 원본이 green appearance를 유지한 채 permissions/timeouts/비밀값/정리 semantics를 약화하는 경계입니다. |
| 테스트 방식 | 정적 text + 서비스-block parse + Python AST + mocked unit-style control-flow probes |
| 테스트 준비 코드와 실패 주입 | workflow YAML, Compose, Python tools를 읽고 selected main paths를 mock subprocess/정리 동작으로 호출합니다. |
| 실제 통과하는 실제 실행 경로 | validator 원본 parsing과 imported verify/런타임 main error branches를 통과하며 Docker 실제 실행 경로는 실행하지 않습니다. |
| 핵심 검사문 | 필수/금지 workflow pattern, action pins, timeouts, 잠금/비밀값 order, 정리 invocation, exit/result precedence를 확인합니다. |
| 이 테스트가 증명하는 것 | verification infrastructure의 명시적 policy와 일부 실패 control flow가 약화되지 않음을 증명합니다. |
| 이 테스트가 증명하지 않는 것 | 실제 CI 서비스 enforcement, live Docker/컨테이너 동작, 모든 동적 Python path는 증명하지 않습니다. |
| 성격 | verification-system 원본/control-flow 회귀 |
| 막는 후속 회귀 | broad permissions, unpinned actions, unsafe event/secrets, timeout 제거, global prune, 정리 실패 무시, 잠금 밖 비밀값 read를 막습니다. |
| 직접 실행 명령과 결과 | 실행하지 않았습니다. 현재 환경에는 Docker와 로컬 repository 체크아웃이 없습니다. 해당 SHA의 테스트 코드와 명령 wiring만 검사했습니다. |

#### 다음 연결

- 이 커밋 뒤에도 남아 있는 불충분한 보장: GitHub-hosted runner가 실제로 모든 policy를 적용하는지, Docker 런타임 동작 전체, AST가 모델링하지 않은 동적 path는 증명하지 않습니다.
- 다음 관련 커밋이 바꾸거나 검증하는 지점: 개발 흐름 전체의 “verification of verification” endpoint이며 런타임 테스트와 상호 보완합니다.
- 이 커밋을 제거했을 때 개발 흐름 설명에서 생기는 공백: verification infrastructure 자체가 least-privilege/bounded/비밀값-safe/scoped-정리 규약을 원본과 selected control-flow 수준에서 유지함을 증명합니다.

## 불변식 변화 기록

| Source에서 연결된 불변식 | 처음/초기 단계 | 강화·교정 단계 | 검증 단계 | 학습자가 확인한 실제 근거 |
| --- | --- | --- | --- | --- |
| MariaDB는 internal backend에만 있고 Nginx는 DB 네트워크에 접근하지 않습니다. | 27a3dca01d3b | 27a3dca01d3b | 7fbd41fe5af4 | Compose exact membership과 live Docker 네트워크 inspect가 Nginx/frontend, MariaDB/backend, WordPress dual-homed 상태를 확인합니다. |
| destructive 정리는 exact selected 프로젝트 confirmation과 owned-자원 scope를 요구합니다. | 74c285925325 | 2b35aa3d2217 | 7fbd41fe5af4, 43ccded05e4f, 18508c25eef0 | Make exact-name guard와 records/labels/tags scoped 정리가 generic deletion/global prune를 대신합니다. |
| diagnostics는 비공개, redacted, non-overwriting이며 sanitization 불가 시 아무것도 게시하지 않습니다. | ef74ad47ea81 | 27a083d91c87 | 7fbd41fe5af4 | complete masking set, 0700/0600 O_EXCL 출력, rescan, error-directory removal과 unsafe 테스트 준비 코드가 연결됩니다. |
| 런타임 테스트 정리 실패와 residual leak는 verification 실패입니다. | 98e4af62e884 | 2b35aa3d2217, 43ccded05e4f | 18508c25eef0, 8a6c07988160 | close error accumulation, 소유권 records, final leak recovery, CI/self-테스트 result precedence가 연결됩니다. |
| 정리와 recovery는 global Docker prune 없이 exact 프로젝트 labels/names/이미지 tags만 제거합니다. | 2b35aa3d2217 | 43ccded05e4f | 18508c25eef0, 8a6c07988160 | 비공개 records와 exact discovery/removal, forbidden prune pattern이 소유권 discipline을 고정합니다. |
| automation 명령은 bounded timeout, least privilege, pinned dependencies, allowlisted 근거를 유지합니다. | 43ccded05e4f | 18508c25eef0 | 8a6c07988160 | 직렬 timeouts, read-only token, SHA-pinned actions, always 정리, artifact allowlist와 text/AST/mock 테스트가 연결됩니다. |

### Ledger 보완 기록

- 원문에 명시되지 않은 새 불변식을 확정 사실로 추가하지 않습니다.
- 불변식이 실제로 부족했음을 드러낸 커밋 또는 실패 stage: single 네트워크, unguarded destructive 정리, best-effort diagnostics/teardown와 독립 테스트 commands는 실패 반경과 leak를 verification success 밖에 남겼습니다.
- marker, rename, 잠금, 헬스 상태, authentication, 정리 등 불변식을 고정하는 concrete mechanism: internal backend, exact 프로젝트 confirmation, fail-closed 비공개 diagnostics, owned-자원 records, 직렬 bounded orchestration와 least-privilege pinned workflow가 operational 소유권을 고정합니다.
- 후속 커밋이 불변식을 약화하지 못하게 하는 회귀 근거: `7fbd41fe5af4`, `98e4af62e884`, `43ccded05e4f`, `18508c25eef0`, `8a6c07988160`이 live policy, result precedence, 정리와 workflow 자체를 계층적으로 검증합니다.
## 문제 → 수정 → 검증 연결

| 실패 / 위험 | fix 또는 mechanism | 테스트 / 근거 | 학습자 연결 기록 |
| --- | --- | --- | --- |
| single shared 네트워크가 frontend에 DB reachability 부여 | 27a3dca01d3b internal backend/exact memberships | 7fbd41fe5af4 live 네트워크 inspect | WordPress만 요청과 persistence networks를 연결합니다. |
| generic confirmation으로 wrong 프로젝트 볼륨 삭제 | 74c285925325 exact PROJECT_NAME confirmation | 7fbd41fe5af4 refusal+헬스 상태/상태 | destructive scope를 작업 시점에 다시 명시합니다. |
| diagnostic bundle이 credential/path 누출 또는 partial 출력 | ef74ad47ea81 redaction + 27a083d91c87 비공개 fail-closed 반영 | 7fbd41fe5af4 real log/unreadable/overwrite/symlink cases | sanitize 입력이 불완전하면 일부 결과도 게시하지 않습니다. |
| scenario pass지만 teardown 실패로 자원 누수 | 98e4af62e884 정리 error result propagation | 2b35aa3d2217 소유권 records + 43ccded05e4f final recovery | 정리는 테스트 외 부수 작업이 아니라 근거 success 조건입니다. |
| crash 정리가 broad prune로 unrelated 자원 삭제 | 2b35aa3d2217 label/tag-scoped recovery | 43ccded05e4f/CI final 정리와 8a6c07988160 no-prune guard | explicit 소유권 이외에는 삭제하지 않습니다. |
| CI green이지만 permissions/action pins/timeout/비밀값/정리 약화 | 18508c25eef0 policy-rich workflow | 8a6c07988160 text/AST/mock self-validation | verification system도 versioned executable 규약으로 검증합니다. |

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

## 소유권·상태·담당 변화

| 대상 | 이전 상태 | 이후 책임/authoritative 상태 | 확인할 근거 | 학습자 결론 |
| --- | --- | --- | --- | --- |
| Networks | single shared bridge | frontend/backend 분리; WordPress만 dual-homed | 27a3dca01d3b Compose + 7fbd41fe5af4 inspect | Nginx는 DB addressability를 갖지 않습니다. |
| Service 자원 lifecycle | Docker defaults | 서비스-specific limits, stop signal/timeout, log rotation | 911544133fb4 + live inspect | Docker effective 상태까지 확인 대상입니다. |
| Diagnostics 출력 | arbitrary existing path/broad inspect 가능 | new 비공개 directory와 allowlisted redacted files | ef74ad47ea81, 27a083d91c87 | collector가 complete 반영 lifecycle을 소유합니다. |
| Scenario 정리 | best-effort teardown | 테스트 result 일부이며 exact 프로젝트/이미지 소유권만 제거 | 98e4af62e884, 2b35aa3d2217 | 정리 실패는 success를 무효화합니다. |
| Leak recovery utility | broad manual 정리 위험 | strict 비공개 records와 exact labels/tags 기반 recovery | 2b35aa3d2217 | global prune를 사용하지 않습니다. |
| Local/CI orchestrator | 개별 commands | 직렬 bounded lifecycle, diagnostics, final 정리, outcome precedence 소유 | 43ccded05e4f, 18508c25eef0, 8a6c07988160 | verification infrastructure 자체도 규약으로 검사합니다. |

## 개발 흐름의 최종 상태

<!-- 원문에서 정한 최종 상태: This progression turns operational policy into executable evidence. Runtime limits and network boundaries are inspected on live 컨테이너; destructive commands and diagnostics fail safely; and both local and CI runners account for every project 자원 they create. The cleanup tooling deliberately avoids global Docker pruning, preserving the same 소유권 discipline used by the product management paths. -->
- 최종 authoritative 상태와 소유자: Compose가 네트워크/자원/shutdown/log policy를, 비공개 diagnostics directory가 redacted 근거를, 소유권 records가 테스트 자원 scope를 소유합니다.
- 정상 실행의 entry point와 완료 조건: local/CI orchestrator가 정적/config/런타임 scenarios를 직렬 bounded execution하고 diagnostics와 final 정리까지 성공해야 정상 완료입니다.
- 실패 또는 interruption 뒤 retry/되돌리기/compensation 조건: scenario/diagnostic/정리/timeout/unexpected 실패는 모두 nonzero에 반영되며 records 기반 exact 정리를 시도하고 residual leak를 보고합니다.
- 이 개발 흐름이 다른 개발 흐름에 제공하는 전제: 앞선 모든 개발 흐름의 architecture와 recovery properties를 live inspect하고 지속적으로 자동 검증하는 운영 lifecycle을 제공합니다.
- 이 개발 흐름 단독으로는 증명하지 않는 것: Docker가 없는 현재 환경에서는 live operations/CI workflow를 실행하지 않았으며 원본/커밋 inspection으로 mechanism만 확인했습니다.

## 최종 설계와 실행 흐름

| 단계 | 확인할 흐름 | 실제 코드 근거 | 정상 전이 | 실패·정리·재시도 |
| --- | --- | --- | --- | --- |
| 1 | Compose policy 선언 | 27a3dca01d3b, 911544133fb4 | 네트워크 membership과 런타임 limits/stop/log/security를 정의합니다. | misconfiguration은 렌더링/start 또는 live 테스트 실패입니다. |
| 2 | effective operations 테스트 | 7fbd41fe5af4 | Docker inspect와 negative commands로 실제 적용을 확인합니다. | policy/guard/diagnostic mismatch면 scenario 실패입니다. |
| 3 | redaction set 생성 | ef74ad47ea81 diagnose_stack | rendered 비밀값 paths와 values를 안전하게 읽어 longer-first/structural masking합니다. | 하나라도 읽지 못하면 bundle을 만들지 않습니다. |
| 4 | 비공개 diagnostic publish | 27a083d91c87 | 0700 directory/0600 allowlisted files를 O_EXCL로 쓰고 rescan합니다. | 실패/비밀값 residue/existing path면 directory 전체를 제거합니다. |
| 5 | scenario result+정리 | 98e4af62e884 | body, diagnostics, teardown errors를 한 final status로 합칩니다. | 정리 실패는 primary success를 무효화합니다. |
| 6 | 소유권/crash recovery | 2b35aa3d2217 | 비공개 records와 exact labels/tags만 제거합니다. | unsafe record나 residual leak는 nonzero이며 global prune는 금지됩니다. |
| 7 | local 직렬 verify | 43ccded05e4f | all scenarios를 timeout/diagnostics/final 정리와 순서대로 실행합니다. | 각 실패에도 finally 정리와 result precedence를 유지합니다. |
| 8 | CI automation | 18508c25eef0 | least-privilege/pinned actions/allowlisted artifacts로 같은 lifecycle을 실행합니다. | timeout/cancel/실패에도 always 정리를 시도합니다. |
| 9 | self-validation | 8a6c07988160 | workflow text, Compose blocks, Python AST, mocked main paths를 검사합니다. | verification policy weakening을 정적 실패로 바꿉니다. |

### 학습자의 최종 설명

> 운영 강화는 옵션을 많이 추가하는 것이 아니라 실패 반경과 소유권을 줄이는 과정입니다. frontend/backend를 분리해 WordPress만 dual-homed bridge로 남기고, 서비스-specific limits와 shutdown/log policy를 선언한 뒤 live inspect로 effective 상태를 확인합니다. destructive 볼륨 deletion은 selected 프로젝트 name의 exact confirmation을 요구합니다. diagnostics는 complete 비밀값/path masking set을 안전하게 만들 수 있을 때만 새 비공개 directory에 allowlisted files를 쓰고 final rescan하며, 어느 단계 실패든 출력 전체를 제거합니다. 런타임 테스트는 정리 실패도 테스트 실패로 취급하고 비공개 소유권 records를 남겨 crash recovery가 exact projects/images만 제거하게 합니다. local verify와 CI는 모든 scenarios를 직렬 timeout으로 실행하고 diagnostics/정리/result precedence를 보존하며, 마지막 정적/AST/mock 테스트가 workflow와 tools 자체의 policy weakening까지 검사합니다.

## 학습 완료 자가 점검

- [x] Nginx와 MariaDB가 같은 네트워크에 남아 있다고 설명하지 않았습니까?
- [x] Compose declaration만 보고 effective 런타임 limits를 확인했다고 간주하지 않았습니까?
- [x] diagnostic redaction 실패 시 partial bundle을 남긴다고 잘못 기록하지 않았습니까?
- [x] 정리를 product 외 부수 작업으로 취급해 scenario success와 분리하지 않았습니까?
- [x] recovery utility가 Docker prune을 사용한다고 쓰지 않았습니까?
- [x] CI workflow만 보고 verification tools의 control flow 계약을 생략하지 않았습니까?
- [x] 모든 코드 snippet에 SHA와 경로/심볼을 기록했습니다.
- [x] 최종 HEAD의 field/helper/테스트를 이전 SHA에 소급하지 않았습니다.
- [x] 원문이 확정하지 않은 사실을 추정으로 채우지 않았습니다.
- [x] 테스트가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] 이 개발 흐름을 커밋 순서대로 구두 설명할 수 있습니다.
===== END FILE: 08-operational-hardening-and-automation.md =====

===== BEGIN FILE: README.md =====
# Inception 개발 흐름 학습 안내

## 목적

이 문서 세트는 `web/inception`의 실제 커밋 history와 각 SHA 시점의 코드를 직접 읽으며 설계, 구현, 실패 처리, 수정, 검증의 발전 과정을 복원하기 위한 기록 골격입니다.

문서에 미리 작성된 SHA, subject, 중요도, tags, 개발 흐름 순서, 원문에서 정한 역할과 분류 근거는 제공된 두 원본 문서의 확정값입니다. 실제 함수 동작, 변경 전후 코드, 소유권/수명, 실패 처리, 테스트 실행 결과, 최종 설명은 학습자가 해당 SHA의 코드를 확인해 작성해야 합니다.

## 권장 학습 순서

1. [`01-readiness-aware-three-tier-stack.md`](01-readiness-aware-three-tier-stack.md)
2. [`02-convergent-one-off-bootstrap.md`](02-convergent-one-off-bootstrap.md)
3. [`03-isolated-runtime-and-persistence.md`](03-isolated-runtime-and-persistence.md)
4. [`04-atomic-backup-publication.md`](04-atomic-backup-publication.md)
5. [`05-fresh-project-restore.md`](05-fresh-project-restore.md)
6. [`06-credential-rotation-and-compensation.md`](06-credential-rotation-and-compensation.md)
7. [`07-immutable-build-inputs.md`](07-immutable-build-inputs.md)
8. [`08-operational-hardening-and-automation.md`](08-operational-hardening-and-automation.md)

이 순서는 원본의 개발 흐름 순서와 같습니다. 동일 SHA가 여러 개발 흐름에 나타나면 제거하지 말고 각 문서의 관점으로 다시 확인합니다.

## 개발 흐름 문서 사용법

1. 먼저 커밋 목록으로 개발 흐름의 상태 변화와 중요도 분포를 확인합니다.
2. 각 커밋에서 `원문에서 정한 역할`, `Summary`, `Classification reason`을 읽고 원문이 확정한 범위를 고정합니다.
3. `해당 SHA에서 확인할 코드`에 적힌 항목을 실제 tree와 diff에서 찾습니다.
4. 코드 근거 표에는 경로, symbol/directive, 핵심 line, 호출자/피호출자 또는 producer/consumer 관계를 함께 기록합니다.
5. Invariant ledger에서 도입, 강화, 부족함 노출, fix, 회귀 테스트의 연결을 채웁니다.
6. 개발 흐름 마지막에는 최종 architecture 또는 execution flow를 자신의 코드 근거만으로 설명합니다.

## 해당 SHA 코드 확인 원칙

항상 커밋 당시의 tree를 기준으로 확인합니다.

```bash
git show --stat --summary <sha>
git diff <sha>^ <sha> -- <path>
git show <sha>:<path>
```

개발 흐름의 이전 관련 커밋과 비교할 때는 다음처럼 별도 diff를 사용합니다.

```bash
git diff <previous-thread-sha> <current-sha> -- <path>
```

파일명이나 symbol이 원문에 명시되지 않은 경우 먼저 `git show --name-status <sha>`로 변경 파일을 식별한 뒤 기록합니다. 최종 HEAD의 동일 파일을 열어 과거 커밋의 동작을 추정하지 않습니다.

## 최종 HEAD 소급 사용 금지

- 후속 refactor, fix, 테스트에서 추가된 field, helper, marker, 볼륨, 네트워크, timeout을 이전 SHA의 설계로 기록하지 않습니다.
- 현재 HEAD에서 사라진 초기 구현도 해당 SHA에서 직접 확인합니다.
- 후속 커밋의 코드는 “다음 변화” 또는 비교 대상으로만 사용하고 현재 커밋의 근거로 대체하지 않습니다.
- 원문이 확정하지 않은 불변식을 새 사실처럼 추가하지 않습니다.

## 중요도별 학습 깊이

### S

프로젝트의 defining architecture 또는 core 상태 transaction으로 다룹니다. 문제, 직전 상태, 실패 가능성, 핵심 결정, actual 코드, 소유권/lifecycle/상태 변화, 되돌리기/compensation, 보장과 비보장, 후속 fix/테스트까지 추적합니다.

### A

주요 구성 요소, security/persistence/lifecycle 경계, 통합 point, non-trivial 실패 처리를 이해할 수 있을 정도로 actual 코드와 설계 판단을 확인합니다. 테스트 커밋은 실제 코드의 불변식, injected 실패, 검증 방식, traversed path, 증명 범위를 분리합니다.

### B

개발 흐름에서 맡은 구현 역할과 필요한 상태 변화를 확인합니다. 핵심 파일, directive/helper, 입력/출력, immediate 실패 브랜치, 다음 커밋에 넘기는 한계를 기록합니다.

### C

개발 흐름 이해에 필요한 맥락만 확인합니다. 동일한 깊이의 분석란을 억지로 확장하지 않습니다. 현재 개발 흐름에는 C 커밋이 없지만 분류 원칙은 유지합니다.

## 실제 코드 삽입 기준

- 설명을 대신하는 대량 복사는 피하고 불변식, 상태 변경 order, 소유권 transfer, 실패 브랜치를 증명하는 최소 범위만 삽입합니다.
- snippet마다 반드시 SHA, path, symbol/directive, line range 또는 인접 문맥을 기록합니다.
- shell/Compose/Python 설정은 caller와 consumer를 함께 적습니다. 예를 들어 환경 mapping만 넣지 말고 이를 읽는 엔트리포인트/helper도 연결합니다.
- 테스트 코드는 테스트 준비 코드 setup, 실패 주입, production 명령/path, 검사문을 한 세트로 기록합니다.
- 코드 excerpt만으로 의미가 불분명하면 앞뒤 상태와 실패 시 결과를 자신의 문장으로 설명합니다.

## 테스트 커밋 학습 방법

각 테스트 커밋에서 다음을 반드시 분리합니다.

- 대상으로 하는 실제 코드의 불변식
- 재현하는 실패 또는 경계
- 사용하는 검증 방식: 정적 원본 규약, rendered configuration, live 통합, 결정적 pause/signal, SIGKILL, AST/control-flow probe 등
- 실제로 통과하는 production 코드 path
- 테스트가 증명하는 것
- 테스트가 증명하지 않는 것
- broad 통합인지 결정적 회귀 테스트인지
- 후속 변경에서 막는 회귀
- 직접 실행한 명령과 실제 결과

테스트가 성공했다는 사실만 기록하지 말고 실패 주입 위치와 검사문이 실제 코드의 불변식에 연결되는 과정을 남깁니다.

## 문서 완료 기준

- 8개 개발 흐름 문서의 모든 커밋을 원문 순서대로 검토했습니다.
- 모든 SHA, subject, 중요도, tags를 변경하지 않았습니다.
- 동일 커밋이 여러 개발 흐름에 있는 경우 각 관점의 기록을 모두 작성했습니다.
- 중요한 커밋마다 해당 SHA의 실제 파일과 symbol/directive 근거가 있습니다.
- S/A/B/C 깊이 차이가 기록량과 질문 범위에 반영되어 있습니다.
- fix는 기존 가정 → 실패/risk → root cause → corrected 불변식 → 코드 → 회귀 테스트로 연결했습니다.
- 테스트는 실제 코드의 불변식, 실패 검증 방식, traversed path, 증명/비증명 범위를 구분했습니다.
- 각 개발 흐름의 Invariant ledger, Failure → Fix → Test, 소유권/상태 변화, final flow, 자가 점검을 완료했습니다.
- 최종 HEAD의 구현을 과거 SHA에 소급한 설명이 없습니다.
- 최종적으로 커밋 history에 근거해 설계 → 구현 → 실패 → 수정 → 검증의 발전을 다시 설명할 수 있습니다.
===== END FILE: README.md =====

