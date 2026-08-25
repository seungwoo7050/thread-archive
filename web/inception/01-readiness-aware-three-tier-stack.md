# Thread 1 — 준비 상태로 연결되는 3계층 스택

> 원문 제목: **From custom services to a readiness-aware three-tier stack**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

이 Thread는 MariaDB, WordPress, Nginx 이미지를 각각 만들었다는 사실보다, 세 프로세스가 하나의 상태를 갖는 서비스로 결합되는 과정을 다룹니다.

- Nginx는 외부 HTTPS 연결을 받고 PHP 요청을 WordPress로 전달합니다.
- WordPress는 PHP-FPM과 애플리케이션 상태를 담당합니다.
- MariaDB는 관계형 데이터를 보관합니다.
- Compose는 세 책임을 서비스 이름, 볼륨, 네트워크, 상태 검사로 연결합니다.

핵심 변화는 `a8b9f693c614`에서 세 서비스가 한 Compose 모델 안에 처음 등장한 것과, `75590dedfb3a`에서 named volume과 상태 기반 의존성이 실제로 연결된 것을 구분하는 데 있습니다. 전자는 토폴로지를 선언하지만, 후자 전에는 데이터 경로와 준비 상태가 아직 운영 가능한 의미를 갖지 않습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`f8ec9621725c`](https://github.com/seungwoo7050/42-archive/commit/f8ec9621725c) | feat(mariadb): Debian 서버 이미지 추가 | B | `STACK`, `PERSISTENCE` | 프로젝트가 직접 소유하는 MariaDB 실행 이미지와 데이터 경로의 기본 소유권을 만듭니다. |
| 2 | [`e13b0357a21b`](https://github.com/seungwoo7050/42-archive/commit/e13b0357a21b) | feat(mariadb): DB와 애플리케이션 계정 초기화 | A | `BOOTSTRAP`, `SECRETS`, `CORE` | 첫 실행에서 DB와 계정을 만드는 초기 idempotent 초기화를 추가합니다. |
| 3 | [`d764d066167b`](https://github.com/seungwoo7050/42-archive/commit/d764d066167b) | feat(wordpress): 사이트와 사용자 계정 초기화 | A | `BOOTSTRAP`, `PERSISTENCE`, `CORE` | WordPress 파일·사이트·사용자 상태를 필요한 경우에만 생성합니다. |
| 4 | [`99c03f54399a`](https://github.com/seungwoo7050/42-archive/commit/99c03f54399a) | feat(nginx): PHP 요청을 WordPress로 전달 | A | `STACK`, `INTEGRATION`, `CORE` | HTTPS 요청을 정적 파일 또는 WordPress FastCGI로 보내는 외부 요청 경계를 정의합니다. |
| 5 | [`a8b9f693c614`](https://github.com/seungwoo7050/42-archive/commit/a8b9f693c614) | feat(compose): 세 서비스 토폴로지 구성 | S | `ARCH`, `STACK`, `CORE` | 세 서비스의 책임을 하나의 Compose 토폴로지로 조립합니다. |
| 6 | [`75590dedfb3a`](https://github.com/seungwoo7050/42-archive/commit/75590dedfb3a) | feat(compose): 준비 상태에 따라 영속 서비스 연결 | A | `PERSISTENCE`, `INTEGRATION`, `OPERATIONS` | 실제 볼륨 mount, 상태 검사, `service_healthy` 의존성을 연결합니다. |

## 1. MariaDB 이미지와 첫 초기화 모델

### `f8ec9621725c` — 실행 환경만 만든 단계

첫 커밋은 Debian 이미지에 MariaDB server/client와 `gosu`를 설치하고, 배포판 이미지에 들어 있던 초기 데이터 디렉터리를 지운 뒤 `mysql` 사용자가 실행할 경로를 준비합니다. 최종 프로세스는 MariaDB daemon을 foreground로 실행하므로 컨테이너 수명과 DB 프로세스 수명이 분리되지 않습니다.

이 시점의 책임은 명확하지만 좁습니다.

- 이미지가 실행 파일과 디렉터리 소유권을 제공합니다.
- 어떤 DB와 계정을 만들지는 아직 정하지 않습니다.
- 실제 영속성은 아직 named volume과 연결되지 않았습니다.

따라서 중요도 B가 적절합니다. 프로젝트가 직접 만든 DB 이미지를 여는 기반이지만, 애플리케이션 상태를 정의하지는 않습니다.

### `e13b0357a21b` — “데이터 디렉터리가 없으면 초기화”

이 커밋은 MariaDB entrypoint에 비밀번호 입력, 이름 검증, 임시 socket 서버, root hardening, 애플리케이션 DB와 계정 생성을 추가합니다. 직접 값과 `_FILE` 입력은 동시에 허용하지 않습니다.

```sh
if [ -n "$current" ] && [ -n "$file_path" ]; then
    echo "$var and $file_var are mutually exclusive" >&2
    exit 1
fi
```

초기화 여부는 다음 한 줄로 결정됩니다.

```sh
if [ ! -d /var/lib/mysql/mysql ]; then
    mariadb-install-db --user=mysql --datadir=/var/lib/mysql --skip-test-db >/dev/null
    # 임시 socket 서버 시작 → 계정/DB 생성 → 종료
fi
```

정상 경로에서는 충분히 idempotent합니다. 이미 system table 디렉터리가 있으면 초기 SQL을 반복하지 않고 daemon을 실행합니다. 그러나 완료 여부를 `/var/lib/mysql/mysql`의 존재 하나로 판단하기 때문에, system table 생성 뒤 애플리케이션 DB나 계정 생성 전에 프로세스가 죽으면 다음 실행은 초기화를 건너뛸 수 있습니다.

즉 이 커밋이 보장하는 것은 **정상 완료 뒤 재실행 시 중복 초기화를 피한다**는 것이지, **중간 종료 뒤 완성 상태로 수렴한다**는 것은 아닙니다. 후자는 Thread 2의 staged bootstrap에서 해결됩니다.

## 2. WordPress 파일·사이트·계정의 조건부 생성

`d764d066167b`도 비슷한 방식으로 각 단계의 존재 여부를 조건으로 삼습니다.

```sh
if [ ! -f /var/www/html/wp-includes/version.php ]; then
    wp core download --allow-root --path=/var/www/html
fi

if [ ! -f /var/www/html/wp-config.php ]; then
    wp config create ...
fi

if ! wp core is-installed --allow-root --path=/var/www/html >/dev/null 2>&1; then
    wp core install ...
fi

if ! wp user get "$WORDPRESS_USER" --allow-root --path=/var/www/html >/dev/null 2>&1; then
    wp user create ... --role=author ...
fi
```

이 분기들은 다음 네 상태를 각각 따로 수렴시킵니다.

1. WordPress core 파일
2. `wp-config.php`
3. 설치된 사이트와 관리자
4. 일반 작성자 계정

다만 비밀번호가 command argument로 전달되고, core를 컨테이너 시작 중 네트워크에서 내려받으며, 여러 상태를 모두 끝냈다는 별도 completion marker는 없습니다. 어느 한 단계 뒤에 종료되면 다음 실행이 남은 단계를 이어갈 수는 있지만, 부분 상태가 정말 일관적인지 검증한 뒤 계속하는 구조는 아닙니다.

이 커밋의 의미는 “한 번만 실행하는 스크립트”에서 “이미 존재하는 상태를 확인하며 필요한 부분만 만든다”로 이동한 데 있습니다. interruption-safe transaction은 아직 아닙니다.

## 3. 외부 요청의 소유자: `99c03f54399a`

Nginx 커밋은 단순히 443 포트를 여는 것이 아니라, 요청 종류별 책임을 분리합니다.

- TLS 1.2/1.3으로 HTTPS를 종료합니다.
- `/healthz`는 Nginx 자체 상태를 확인하는 고정 응답입니다.
- 일반 경로는 정적 파일을 먼저 찾고 WordPress front controller로 보냅니다.
- PHP 요청은 Compose 서비스 이름 `wordpress:9000`으로 전달합니다.
- `SCRIPT_FILENAME` 등 FastCGI 매개변수로 WordPress 컨테이너의 파일 경로를 전달합니다.

요청 경로는 다음과 같습니다.

```text
client HTTPS
  → nginx:443
  → 정적 파일 또는 fastcgi_pass wordpress:9000
  → PHP-FPM / WordPress
  → mariadb 서비스 이름으로 SQL 요청
```

Nginx는 DB 자격증명을 알 필요가 없고 MariaDB와 직접 통신할 이유도 없습니다. 이 책임 분리는 이후 frontend/backend 네트워크 분리의 전제가 됩니다.

## 4. `a8b9f693c614` — 세 이미지를 하나의 시스템으로 조립

이 S급 커밋은 Nginx, WordPress, MariaDB 서비스를 한 Compose 파일에 배치하고 하나의 네트워크와 named volume 이름을 선언합니다. 서비스의 역할은 다음처럼 고정됩니다.

| 서비스 | 입력 | 주된 실행 책임 | 지속되어야 할 상태 |
| --- | --- | --- | --- |
| Nginx | host 443 | TLS 종료, 정적 파일, FastCGI 전달 | 없음 |
| WordPress | FastCGI | PHP 애플리케이션 실행 | WordPress 파일과 설정 |
| MariaDB | WordPress SQL | 관계형 데이터 처리 | DB data directory |

중요한 제한도 있습니다. 이 SHA에서 volume 이름은 선언되지만 서비스의 `volumes:`에 실제 mount가 아직 연결되지 않았고, healthcheck와 상태 기반 `depends_on`도 없습니다. 따라서 이 커밋만으로는 다음을 보장하지 않습니다.

- 컨테이너 재생성 뒤 DB·WordPress 상태가 유지됨
- Nginx가 WordPress와 동일한 파일 tree를 읽음
- WordPress가 단순히 컨테이너가 생성된 시점이 아니라 MariaDB가 실제 준비된 뒤 시작됨

즉 `a8b9f693c614`의 핵심은 **책임과 주소를 한 모델에 배치한 것**입니다. **영속 상태와 준비 상태를 연결한 것**은 다음 커밋입니다.

## 5. `75590dedfb3a` — 이름뿐이던 volume과 dependency를 실행 규칙으로 바꿈

이 커밋은 Compose 서비스에 실제 mount를 추가합니다.

```yaml
mariadb:
  volumes:
    - mariadb_data:/var/lib/mysql

wordpress:
  volumes:
    - wordpress_data:/var/www/html

nginx:
  volumes:
    - wordpress_data:/var/www/html:ro
```

그 결과 WordPress가 쓴 core·upload 파일을 Nginx가 read-only로 제공하고, MariaDB 데이터는 컨테이너 writable layer가 아니라 named volume에 남습니다.

의존성도 단순 생성 순서가 아니라 상태 조건으로 바뀝니다.

```yaml
wordpress:
  depends_on:
    mariadb:
      condition: service_healthy

nginx:
  depends_on:
    wordpress:
      condition: service_healthy
```

여기서 healthcheck는 “프로세스가 하나 존재한다”보다 강한 조건을 사용합니다. MariaDB는 DB daemon에 접근할 수 있어야 하고, WordPress는 PHP-FPM ping이 응답해야 하며, Nginx는 HTTPS `/healthz`가 성공해야 합니다. 이 구조가 만드는 순서는 다음과 같습니다.

```text
MariaDB healthy
    ↓
WordPress start / PHP-FPM healthy
    ↓
Nginx start / HTTPS health healthy
```

### 보장과 남은 문제

최종 상태에서 세 서비스는 영속 volume과 준비 상태로 연결됩니다. 그러나 초기 entrypoint는 여전히 여러 durable write 사이에서 죽을 수 있고, long-running service가 초기화용 비밀값을 보유할 수 있습니다. 따라서 이 Thread의 최종 토폴로지는 운영 가능한 3계층 구조이지만, 초기화 lifecycle 자체는 Thread 2에서 다시 설계됩니다.

## 최종 상태

| 관심사 | 최종 소유자 | 관찰 가능한 조건 |
| --- | --- | --- |
| 외부 HTTPS와 FastCGI 전달 | Nginx | `/healthz` 성공, `wordpress:9000` 연결 |
| PHP 애플리케이션·사이트 파일 | WordPress | PHP-FPM ping, `wordpress_data` mount |
| 관계형 데이터 | MariaDB | DB health, `mariadb_data` mount |
| 시작 순서 | Compose 상태 기반 의존성 | MariaDB → WordPress → Nginx |
| 컨테이너 교체 뒤 상태 | named volumes | 컨테이너 writable layer와 독립 |

## Thread 경계

이 문서는 세 서비스의 책임과 기본 Compose 연결만 다룹니다. 다음 항목은 다른 Thread의 문제입니다.

- 중간 종료 뒤 초기화 상태를 완성하거나 거부하는 staged bootstrap
- steady-state 컨테이너에서 비밀 파일을 제거하는 방식
- 실제 restart/recreate 뒤 데이터가 남는지 확인하는 런타임 테스트
- frontend/backend 네트워크 분리와 자원 한계

## 검토 메모

각 설명은 표시된 exact SHA의 diff와 source를 기준으로 작성했습니다. 이 환경에서는 이미지를 build하거나 Compose 스택을 실행하지 않았습니다.
