# Thread 5 — 새 프로젝트에만 적용되는 검증된 복원과 정리 롤백

> 원문 제목: **Verified fresh-project restore with cleanup rollback**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

이 복원 기능은 기존 project의 volume 위에 데이터를 덮어쓰지 않습니다. target namespace가 완전히 비어 있을 때만 새 Compose project를 만들고, 실패하면 이번 시도가 만든 container·volume·network를 모두 제거합니다.

이 제약은 단순한 보수적 선택이 아니라 rollback 가능성의 기반입니다.

- 기존 객체에 merge하면 어느 파일·row가 복원 전부터 있었는지 구분하기 어렵습니다.
- 새 project만 허용하면 첫 자원 생성 이후의 모든 project 자원을 이번 시도의 소유물로 취급할 수 있습니다.
- 따라서 성공은 완전한 새 stack, 실패는 target namespace가 다시 빈 상태라는 두 결과로 수렴할 수 있습니다.

핵심 S급 커밋은 `9ca04b1c30cd`입니다. 검증된 backup stream을 새 volume에 적용하고, 어느 단계에서든 실패하면 Compose cleanup과 독립 resource re-scan을 모두 통과해야 rollback 완료로 인정합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`e5cb60c7d743`](https://github.com/seungwoo7050/42-archive/commit/e5cb60c7d743) | feat(restore): Compose 리소스 이름과 기존 객체 조회 | B | `PERSISTENCE`, `OPERATIONS` | 렌더링된 이름과 Compose 관례 이름으로 관련 Docker 객체를 찾습니다. |
| 2 | [`851dc1708881`](https://github.com/seungwoo7050/42-archive/commit/851dc1708881) | feat(restore): 대상 프로젝트 자원 충돌 사전 차단 | A | `PERSISTENCE`, `RISK`, `EDGE` | 빈 target project를 restore의 mutation 전제조건으로 만듭니다. |
| 3 | [`953a0f6bd571`](https://github.com/seungwoo7050/42-archive/commit/953a0f6bd571) | feat(restore): 백업 입력의 형식과 체크섬 검증 | A | `PERSISTENCE`, `RISK`, `EDGE` | private, locked, checksummed descriptor-anchored input을 만듭니다. |
| 4 | [`1250fcf7c006`](https://github.com/seungwoo7050/42-archive/commit/1250fcf7c006) | feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입 | B | `PERSISTENCE`, `INTEGRATION` | SQL과 archive stream을 비어 있는 새 volume에 주입합니다. |
| 5 | [`9ca04b1c30cd`](https://github.com/seungwoo7050/42-archive/commit/9ca04b1c30cd) | feat(restore): 실패한 복원 자원을 정리하고 롤백 | S | `PERSISTENCE`, `RECOVERY`, `HARD` | restore orchestration과 partial project cleanup을 all-or-nothing으로 연결합니다. |
| 6 | [`3a37a491ecea`](https://github.com/seungwoo7050/42-archive/commit/3a37a491ecea) | feat(restore): 복원 CLI와 Make 타깃 연결 | B | `PERSISTENCE`, `OPERATIONS` | restore 입력을 CLI와 Make target으로 노출합니다. |
| 7 | [`4f8eb9aff842`](https://github.com/seungwoo7050/42-archive/commit/4f8eb9aff842) | test(restore): 거부·롤백·복원 상태 검증 | A | `TEST`, `RECOVERY`, `RISK` | malformed input, injected failure, signal, success, second restore refusal을 검사합니다. |
| 8 | [`030e7310c665`](https://github.com/seungwoo7050/42-archive/commit/030e7310c665) | test(backup): 자원 충돌과 시그널 경계 검증 | A | `TEST`, `PERSISTENCE`, `EDGE` | stopped/unlabelled collision과 큰 복원 fixture를 추가합니다. |

## 1. “비어 있음”을 어떻게 판정하는가

### `e5cb60c7d743` — 한 가지 조회 방식으로는 부족함

Compose project에 속한 객체는 보통 `com.docker.compose.project=<name>` label로 찾을 수 있습니다. 그러나 다음 경우가 있습니다.

- 중지된 컨테이너
- label을 잃었지만 예상 이름을 사용하는 객체
- 현재 Compose와 예전 Compose가 사용하는 `project-service-1` / `project_service_1` 이름 차이
- one-off bootstrap 이름 `project-service-bootstrap`
- explicit `name:` 또는 project prefix로 렌더링된 volume/network

따라서 helper는 세 종류의 identity를 모읍니다.

```text
1. project label로 찾은 container / volume / network
2. current·legacy convention과 bootstrap을 포함한 exact container names
3. compose config --format json에서 얻은 concrete volume / network names
```

이 커밋은 inventory만 만듭니다. 발견한 객체가 있으면 restore를 거부하는 결정은 다음 커밋입니다.

### `851dc1708881` — 첫 mutation 전에 모두 합쳐 판정

`ensure_fresh_project`는 세 조회 결과를 union하고 하나라도 있으면 실패합니다.

```python
for kind in ("container", "volume", "network"):
    identifiers = project.labelled_resources(kind)
    ...
named_containers = existing_named_containers(expected_container_names(project))
...
rendered = rendered_resource_names(project)
...
if found:
    raise BackupError("복원 대상 프로젝트가 비어 있지 않습니다: ...")
```

이 precondition은 “running service가 없다”보다 강합니다. 멈춘 container, 남은 volume, 내부 network, 이름만 충돌하는 무라벨 객체도 mutation 전에 막습니다.

거부 경로에서는 발견한 객체를 지우지 않습니다. fresh 여부를 확인하는 도구가 기존 객체의 소유자를 알 수 없으므로, 안전한 행동은 보존하고 실패하는 것입니다.

## 2. `953a0f6bd571` — pathname이 아니라 열린 객체를 restore 끝까지 사용

backup directory를 처음 검증한 뒤 나중에 pathname으로 다시 열면, 그 사이 다른 process가 directory나 파일을 바꿀 수 있습니다. `VerifiedBackup`은 directory descriptor와 두 열린 stream을 restore 종료까지 유지합니다.

### directory와 exact set 검증

```python
directory_descriptor = os.open(
    source,
    os.O_RDONLY | O_DIRECTORY | O_NOFOLLOW,
)
present = set(os.listdir(directory_descriptor))
if present != {"database.sql", "wordpress.tar.gz", "manifest.json"}:
    raise BackupError(...)
```

입력 directory는 현재 사용자 소유의 private real directory여야 합니다. symlink와 group/other access를 거부합니다.

### 각 파일을 directory descriptor 기준으로 열기

```python
descriptor = os.open(
    filename,
    os.O_RDONLY | O_NOFOLLOW | O_NONBLOCK,
    dir_fd=directory_descriptor,
)
```

`fstat`으로 다음을 확인합니다.

- regular file
- hard link count 1
- 현재 사용자 소유
- group/other permission 없음
- non-blocking shared `flock` 획득

manifest는 64 KiB 이하이고 `format == 1`이어야 하며 두 artifact의 SHA-256과 정확히 일치해야 합니다. archive는 비어 있지 않고 absolute path, `..`, duplicate path, symlink/device 같은 비정상 entry를 포함하지 않아야 합니다.

검증 뒤에도 `database`와 `wordpress` stream, directory descriptor를 닫지 않습니다.

```python
return VerifiedBackup(
    directory_descriptor,
    database,
    wordpress,
    manifest,
)
```

이렇게 하면 restore는 검증한 바로 그 inode의 stream을 사용합니다. source pathname이 나중에 교체돼도 이미 연 객체가 바뀌지 않습니다.

## 3. `1250fcf7c006` — 새 volume에 stream으로 주입

### MariaDB

root password 한 줄과 SQL dump를 `TemporaryFile`에 이어 붙인 뒤 container stdin으로 전달합니다. container 안에서는 비밀번호만 먼저 읽어 private option file로 만들고 남은 bytes를 MariaDB client가 SQL로 소비합니다.

```text
TemporaryFile payload
 ├─ first line: root password
 └─ remaining bytes: database.sql
      → container stdin
      → private client option file + mariadb import
```

비밀번호는 process argument에 들어가지 않고, 큰 SQL을 Python memory 한 번에 올리지 않습니다.

### WordPress

one-off WordPress container는 `/var/www/html`과 `/var/www/config`가 비어 있는지 먼저 확인한 뒤 archive를 표준 입력에서 추출합니다.

```sh
test -z "$(find /var/www/html -mindepth 1 -print -quit)"
test -z "$(find /var/www/config -mindepth 1 -print -quit)"
exec tar -xzf - -C /var/www
```

empty-volume check는 기존 파일과의 merge를 막습니다. archive validation과 결합되어 “검증된 regular file/directory만 빈 새 volume에 쓴다”는 조건이 됩니다.

이 커밋 자체는 실패 뒤 project 자원을 지우지 않습니다. injection primitive일 뿐이므로 B입니다.

## 4. `9ca04b1c30cd` — restore transaction과 rollback ownership

### 4.1 순서

실제 `restore_backup`은 signal handler와 project lock 안에서 다음 순서로 실행됩니다.

```text
backup directory/streams open + checksum/archive verification
  → ensure_fresh_project
  → current host secrets load
  → MariaDB build/bootstrap/start
  → database.sql import
  → WordPress image build
  → empty volumes에 archive extract
  → WordPress bootstrap/start
  → Nginx start / wait
```

첫 Docker 자원을 만들기 직전에 `restoration_started = True`가 됩니다. 이후 어떤 `BaseException`도 cleanup 대상으로 들어갑니다.

```python
try:
    restoration_started = True
    start_database(...)
    restore_database(...)
    ...
    restore_wordpress(...)
    start_application(...)
except BaseException as original_error:
    if restoration_started:
        cleanup_failed_restore(project)
    raise
```

### 4.2 `compose down --volumes` 성공만 믿지 않음

rollback은 우선 Compose에게 project 자원을 제거하도록 요청합니다.

```text
docker compose down
  --volumes
  --remove-orphans
  --timeout 20
```

그 뒤 독립적으로 다시 조회합니다.

- project label container/volume/network
- current·legacy·bootstrap exact container names
- rendered volume/network exact names

```python
remaining = {
    kind: project.labelled_resources(kind)
    for kind in ("container", "volume", "network")
}
remaining["container"].update(
    existing_named_containers(expected_container_names(project))
)
...
if result.returncode != 0 or remaining:
    raise BackupError("실패한 복원 자원을 정리하지 못했습니다: ...")
```

Compose command가 0을 반환해도 orphan 또는 무라벨 객체가 남으면 rollback 완료가 아닙니다. 반대로 command stderr가 비어 있어도 re-scan이 residue를 찾으면 실패입니다.

### 4.3 원래 오류와 cleanup 오류를 둘 다 보존

restore와 cleanup이 함께 실패하면 cleanup error만 덮어쓰지 않습니다.

```python
except Exception as cleanup_error:
    raise BackupError(
        f"복원과 실패 자원 정리가 모두 실패했습니다: {cleanup_error}"
    ) from original_error
```

operator는 최초 실패 원인과 target namespace가 깨끗하지 않다는 더 위험한 상태를 함께 알 수 있습니다.

### 종료 상태

| 경로 | 입력 backup | target Docker 자원 | 기존 충돌 객체 |
| --- | --- | --- | --- |
| 입력 검증 실패 | read-only, 닫힘 | 생성 안 함 | 변경 없음 |
| freshness 거부 | read-only, 닫힘 | 생성 안 함 | 보존 |
| DB/WordPress/start 중 실패 | read-only, 닫힘 | 전부 제거되어야 성공적인 rollback | 해당 없음 |
| cleanup도 실패 | read-only, 닫힘 | residue를 구체적으로 보고 | 해당 없음 |
| 성공 | read-only, 닫힘 | 새 healthy project 유지 | 처음부터 없어야 함 |

## 5. CLI 연결은 얇게 유지: `3a37a491ecea`

이 커밋은 backup/restore operation, input/output 인자, test-only failure/pause stage 조합을 CLI와 Make target에 연결합니다. 안전성 판단을 Make recipe에 복제하지 않고 `stack_backup.py`의 동일 함수를 호출합니다.

중요도 B인 이유는 새로운 restore invariant를 만들지 않기 때문입니다. operator entry point만 제공합니다.

## 6. failure·collision·large-data evidence

### `4f8eb9aff842`

테스트 source는 다음 case를 분리합니다.

- backup item이 symlink면 target resource가 하나도 생기기 전에 거부
- checksum/manifest/archive가 잘못되면 mutation 전 거부
- `database-restore` 뒤 failure injection이면 생성된 project 자원이 모두 제거됨
- restore pause 지점에서 SIGINT를 보내도 cleanup 뒤 target이 비어 있음
- 성공하면 원래 post/option/upload/DB 값을 새 project에서 확인
- 같은 target에 두 번째 restore를 시도하면 fresh-project gate가 거부

마지막 case는 idempotent overwrite를 기대하지 않는 설계를 명확히 합니다. 성공한 restore를 다시 실행하는 것은 “이미 원하는 상태”가 아니라 “기존 project에 덮어쓰기”이므로 실패가 맞습니다.

### `030e7310c665`

이 커밋은 label만으로는 놓칠 수 있는 실제 collision을 추가합니다.

- stopped but labelled container
- expected name을 쓰는 unlabelled container
- rendered name을 쓰는 unlabelled volume/network
- current·legacy naming form

거부 뒤 fixture가 그대로 남아 있는지 확인해 freshness check가 남의 객체를 정리하지 않도록 합니다. 또한 큰 WordPress 파일과 큰 DB 값이 backup→restore 뒤 checksum/length를 유지하는지 검사해 streaming path의 범위를 넓힙니다.

같은 SHA의 signal-race backup 부분은 Thread 4에서 설명합니다.

## 최종 불변 조건

1. target project는 mutation 전에 label과 exact name 양쪽에서 비어 있어야 합니다.
2. restore는 검증 시 연 directory/file descriptor를 종료까지 사용합니다.
3. artifact checksum과 archive entry는 write 전에 검증됩니다.
4. WordPress archive는 empty new volumes에만 풀립니다.
5. 첫 자원 생성 뒤 실패하면 이번 project가 만든 모든 Docker 자원을 제거합니다.
6. cleanup command와 independent residue scan이 모두 성공해야 rollback 완료입니다.
7. 기존 collision object는 거부만 하고 변경하지 않습니다.

## Thread 경계

이 Thread는 동일 host의 새 Compose project 생성과 rollback을 다룹니다. in-place migration, version upgrade, cross-host storage transport, backup encryption, 기존 project merge는 보장하지 않습니다.

## 검토 메모

exact SHA source와 테스트 assertion을 확인했습니다. 이 작업 환경에서는 실제 restore와 failure injection 시나리오를 실행하지 않았습니다.
