# Thread 2 — 런타임 비밀 마운트에서 수렴형 일회성 초기화로

> 원문 제목: **From runtime secret mounts to convergent one-off bootstrap**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

초기 모델은 비밀번호 값을 일반 환경 변수 대신 Compose secret file로 옮겼습니다. 노출을 줄이는 개선이지만 MariaDB와 WordPress의 장기 실행 컨테이너가 `/run/secrets`를 계속 mount하고, 초기화가 중간에 끊겼을 때 “완료된 상태”와 “일부 파일만 생긴 상태”를 명확히 구분하지 못했습니다.

이 Thread의 최종 구조는 다음 세 결정을 결합합니다.

1. host가 비밀 파일의 형식·소유자·권한·파일 종류를 검증합니다.
2. 같은 Compose project의 관리 작업은 하나의 잠금으로 직렬화합니다.
3. 비밀값은 짧게 살아 있는 bootstrap 컨테이너에 표준 입력으로만 전달하고, 장기 실행 서비스는 검증된 volume state만 읽습니다.

핵심 S급 커밋은 `dc9601f5e670`입니다. 단순 재시도 가능한 entrypoint를 completion marker와 staging publication을 갖는 수렴형 초기화로 교체합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`916391b9f8db`](https://github.com/seungwoo7050/42-archive/commit/916391b9f8db) | feat(secrets): 비밀번호를 비밀 파일에서 로드 | B | `SECRETS`, `RISK` | 비밀번호 literal을 Compose secret file과 `_FILE` 입력으로 옮깁니다. |
| 2 | [`486ffb5c65aa`](https://github.com/seungwoo7050/42-archive/commit/486ffb5c65aa) | refactor(secrets): 비밀 파일 로딩 경계 공통화 | A | `SECRETS`, `RISK`, `ARCH` | host 비밀 파일을 안전하게 찾고 읽는 공통 코드를 만듭니다. |
| 3 | [`e77c6f151b07`](https://github.com/seungwoo7050/42-archive/commit/e77c6f151b07) | refactor(runtime): 프로젝트 관리 작업 잠금 공통화 | A | `RECOVERY`, `OPERATIONS`, `RISK` | 같은 project의 시작·백업·복원·회전 작업을 직렬화할 잠금을 만듭니다. |
| 4 | [`dc9601f5e670`](https://github.com/seungwoo7050/42-archive/commit/dc9601f5e670) | fix(init): 중단된 단계별 초기화를 수렴 | S | `ARCH`, `BOOTSTRAP`, `RECOVERY` | 장기 실행 secret mount와 불완전 초기화를 staged one-off bootstrap으로 교체합니다. |
| 5 | [`3beebbfc4723`](https://github.com/seungwoo7050/42-archive/commit/3beebbfc4723) | test(init): 단계별 초기화 계약 검사 | B | `TEST`, `BOOTSTRAP` | marker·staging·실행 모드가 source에 남는지 정적으로 검사합니다. |
| 6 | [`2bf6d3f11337`](https://github.com/seungwoo7050/42-archive/commit/2bf6d3f11337) | test(init): 안정 단계별 초기화 중단 복구 검증 | A | `TEST`, `BOOTSTRAP`, `RECOVERY` | 각 durable stage에서 bootstrap을 강제 종료하고 재실행 수렴을 검사합니다. |

## 1. 중간 단계: Compose secret mount

`916391b9f8db`은 `.env`에서 비밀번호 값을 제거하고 host의 파일 경로를 받도록 바꿉니다. Compose는 네 secret source를 선언하고 MariaDB와 WordPress가 필요한 subset을 `/run/secrets`에서 읽습니다.

```text
host secret file
    → Compose secret declaration
    → /run/secrets/<name> mount
    → *_PASSWORD_FILE environment
    → long-running service entrypoint
```

이 변화는 process environment와 Compose 설정에 평문 비밀번호를 직접 적는 문제를 줄입니다. 하지만 다음 상태는 남습니다.

- 서비스가 실행되는 동안 secret mount가 계속 존재합니다.
- MariaDB healthcheck도 root secret을 읽습니다.
- 초기화와 steady-state 실행이 같은 컨테이너 lifecycle에 묶여 있습니다.
- 파일 경로가 symlink인지, 현재 사용자 소유인지, 권한이 안전한지 host에서 통일해 검증하지 않습니다.

따라서 B급의 의미 있는 중간 단계이지만 최종 비밀 경계는 아닙니다.

## 2. host 입력을 신뢰하기 위한 검사: `486ffb5c65aa`

공통 `stack_runtime.py`는 Compose의 `x-secret-files`를 읽어 실제 source path를 계산하고, 각 파일을 pathname만 믿지 않고 descriptor로 확인합니다.

검사 범위는 다음과 같습니다.

- `O_NOFOLLOW`로 마지막 symlink를 거부합니다.
- `fstat`으로 regular file인지 확인합니다.
- 현재 사용자 소유인지 확인합니다.
- group/other 권한이 없는지 확인합니다.
- 너무 큰 파일과 여러 줄 입력을 거부합니다.
- 비밀번호 문자·길이 형식을 확인합니다.
- 서로 다른 secret 이름이 같은 canonical path를 가리키는 경우를 거부합니다.

이 커밋은 secret을 어디에 전달할지 바꾸지 않습니다. 대신 **host file을 읽는 순간**을 하나의 검증된 입력 경계로 만듭니다. 이후 시작·백업·회전 도구가 같은 규칙을 재사용할 수 있습니다.

## 3. 같은 project의 관리 작업을 하나로 제한

`e77c6f151b07`은 project name을 해시한 lock file을 사용합니다.

```text
/tmp/container-stack-operation-locks-<uid>/
    <sha256(project-name)>.lock
```

잠금 디렉터리는 현재 사용자만 접근 가능한 `0700`, 파일은 `0600` regular file이어야 하며 symlink를 따라가지 않습니다. `flock(LOCK_EX | LOCK_NB)`이 실패하면 기다리며 섞이지 않고 즉시 “다른 관리 작업이 실행 중”으로 실패합니다.

잠금 단위가 서비스가 아니라 project인 이유는 초기화, 백업, 복원, 자격증명 회전이 여러 서비스를 동시에 읽거나 바꾸기 때문입니다. MariaDB만 잠그고 WordPress를 별도로 잠그면 두 작업이 서로 다른 순서로 잠금을 잡아 mixed state를 만들 수 있습니다.

## 4. `dc9601f5e670` — 초기화와 steady state를 분리

### 4.1 host orchestrator가 secret을 잠금 안에서 읽음

`run_action`은 project lock을 획득한 뒤 secret을 읽고, database와 application 단계를 순서대로 호출합니다.

```python
lock = project_operation_lock(project.project) if acquire_lock else nullcontext()
with lock:
    resolved_secrets = (
        secrets if secrets is not None else load_secret_values(project)
    )
    if action in {"start", "database"}:
        start_database(...)
    if action in {"start", "application"}:
        start_application(...)
```

이 순서가 중요합니다. 파일 검증과 실제 사용 사이에 다른 회전 작업이 끼어드는 창을 줄이고, 선택한 secret generation으로 bootstrap 전체를 실행합니다.

### 4.2 secret은 일회성 컨테이너의 stdin으로만 전달

`run_bootstrap`은 stale bootstrap 컨테이너의 label ownership을 확인해 제거한 뒤, 같은 project 이름 아래 짧게 살아 있는 컨테이너를 실행합니다.

```python
project.run(
    "run",
    "--rm",
    "--no-deps",
    "--no-TTY",
    "--name", name,
    "--label", f"{BOOTSTRAP_LABEL}={service}",
    service,
    "bootstrap",
    input_data=payload,
)
```

비밀번호는 command argument나 장기 실행 환경 변수가 아니라 `input_data`로 전달됩니다. bootstrap 종료와 함께 입력을 소비한 프로세스도 사라집니다. long-running MariaDB/WordPress service는 `/run/secrets`를 mount하지 않고, volume 안의 검증된 state만 사용합니다.

### 4.3 MariaDB: staging directory 전체를 한 번에 게시

초기화 전 상태는 다음처럼 분리됩니다.

```sh
volume_dir="/var/lib/mysql-volume"
data_dir="${volume_dir}/data"
staging_dir="${volume_dir}/.container-stack-bootstrap"
marker="${data_dir}/.container-stack-initialized"
```

처음부터 `data_dir`에 system table을 쓰지 않고 `staging_dir`에서 전 과정을 수행합니다.

```sh
mariadb-install-db --user=mysql --datadir="$staging_dir" --skip-test-db
start_temporary_server "$staging_dir"
# DB·계정 생성 및 실제 인증 검증
(umask 077; : >"${staging_dir}/.container-stack-initialized")
sync -f "$staging_dir"
mv -- "$staging_dir" "$data_dir"
sync -f "$volume_dir"
```

완료 전 죽으면 authoritative `data_dir`는 생기지 않습니다. 다음 bootstrap은 이전 staging directory를 지우고 다시 시작합니다. 이미 `data_dir`가 있다면 marker와 system table을 모두 요구하고 실제 root/application 인증을 다시 확인합니다. 데이터가 있으면서 marker가 없으면 자동으로 덮지 않고 실패합니다.

이 결정이 고치는 root cause는 “초기 파일 하나가 생겼는지를 전체 완료로 오인한 것”입니다. 완료 상태는 marker와 검증된 인증을 포함하고, publish는 staging directory rename 뒤에만 일어납니다.

### 4.4 WordPress: 각 상태를 수렴시킨 뒤 마지막 marker 게시

WordPress는 MariaDB처럼 전체 volume을 새 디렉터리로 교체할 수 없습니다. 재실행 시 유지해야 할 content와 DB 상태가 함께 있기 때문입니다. 대신 단계별 검증과 마지막 completion marker를 사용합니다.

```sh
install_core_files
pause_after core-files
converge_wordpress_config
pause_after wordpress-config
install_wordpress
pause_after wordpress-core
ensure_author
pause_after wordpress-users
# URL·계정·비밀번호를 다시 검증
mv -f -- "$marker_tmp" "$marker"
sync -f "$wordpress_dir"
```

runtime 모드는 다음 세 가지를 모두 요구합니다.

```sh
[ -f "${wordpress_dir}/wp-includes/version.php" ] || fail ...
[ -L "$config_link" ] && [ -f "$config_path" ] || fail ...
[ -f "$marker" ] && [ ! -L "$marker" ] || fail ...
```

따라서 WordPress 파일 몇 개가 존재한다는 이유만으로 PHP-FPM을 시작하지 않습니다. bootstrap이 core, config, site, 사용자, 비밀번호 검증을 끝내고 marker를 게시해야 steady-state service가 실행됩니다.

### 4.5 상태 검사도 completion marker를 포함

Compose healthcheck는 단순 socket/process 검사에 marker를 결합합니다.

- MariaDB: completion marker + socket + PID 1 생존
- WordPress: completion marker + PHP-FPM `/ping`

이제 “프로세스가 떠 있음”과 “영속 초기화가 끝남”을 같은 healthy 조건으로 관찰합니다.

## 5. static contract와 강제 종료 검증의 차이

### `3beebbfc4723` — source shape를 고정

정적 검사는 entrypoint와 orchestrator source에 다음 요소가 남아 있는지 확인합니다.

- `bootstrap`/`runtime` mode
- completion marker
- staging directory
- host-side secret 전달
- runtime secret mount 제거

이 테스트는 빠르고 구조 회귀를 잡지만 실제 filesystem durability나 SIGKILL 뒤 상태를 관찰하지 않습니다.

### `2bf6d3f11337` — durable stage마다 SIGKILL

런타임 harness는 bootstrap 컨테이너가 특정 단계의 ready file을 게시할 때까지 기다린 뒤 `SIGKILL`합니다.

MariaDB 대상 단계:

- `system-tables`
- `temporary-server`
- `database-state`
- `database-marker`
- `database-publish`

WordPress 대상 단계:

- `core-files`
- `wordpress-config`
- `wordpress-core`
- `wordpress-users`
- `wordpress-marker`

각 중단 뒤 stale bootstrap container를 정리하고 같은 start action을 다시 실행한 다음 다음을 검사하도록 작성됐습니다.

- 완료 marker 존재
- MariaDB staging directory 제거
- WordPress 임시 config/marker 파일 제거
- root/application DB 인증 성공
- WordPress admin/author 비밀번호 성공
- steady-state service에 `/run/secrets` mount와 비밀번호 환경 변수가 없음

SIGKILL은 shell trap이나 Python exception handler를 실행하지 않습니다. 따라서 이 테스트가 겨냥하는 것은 graceful cleanup이 아니라 **disk에 남은 상태만 보고 재실행이 수렴하는가**입니다.

## 최종 불변 조건

| 불변 조건 | 구현 근거 | 검증 근거 |
| --- | --- | --- |
| host secret은 검증된 private regular file에서만 읽음 | `486ffb5c65aa` | static/runtime secret boundary 검사 |
| 같은 project의 관리 작업은 겹치지 않음 | `e77c6f151b07` project lock | lock collision scenario |
| long-running service는 secret file을 mount하지 않음 | `dc9601f5e670` Compose/orchestrator | runtime inspect |
| MariaDB는 완성된 staging directory만 authoritative data로 게시 | `dc9601f5e670` marker + rename | stage별 SIGKILL 후 rerun |
| WordPress는 전체 상태 검증 뒤 marker를 게시 | `dc9601f5e670` ordered convergence | stage별 SIGKILL 후 rerun |
| healthy는 process readiness와 durable completion을 함께 뜻함 | marker-aware healthcheck | Compose `--wait`/runtime checks |

## Thread 경계

이 Thread는 초기화와 secret 전달 lifecycle만 다룹니다. 격리된 project에서 HTTPS·DB 경로와 volume persistence를 실제로 검증하는 문제는 Thread 3, backup/restore/rotation은 각각 별도 Thread입니다.

## 검토 메모

exact SHA의 diff와 source 및 테스트 harness를 읽었습니다. Docker scenario를 이 환경에서 직접 실행하지 않았으므로 위 테스트 항목은 source가 구성한 assertion 범위입니다.
