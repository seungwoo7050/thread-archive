# Thread 4 — 실패와 중단에도 완전한 세트만 게시하는 백업

> 원문 제목: **Atomic backup publication under failure and cancellation**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

MariaDB dump 파일과 WordPress volume archive가 각각 존재한다는 사실만으로는 신뢰할 수 있는 백업이 아닙니다. 둘 중 하나가 오래됐거나, manifest가 아직 없거나, 작업 중 signal을 받아 일부 파일만 최종 경로에 남았다면 사용자는 그것을 완전한 backup set으로 오인할 수 있습니다.

이 Thread는 **작업 종료 후** 다음 두 terminal state만 허용합니다. 작업 중에는 이름을 독점하기 위한 빈 예약 directory가 보일 수 있지만, 일부 artifact가 들어 있는 그럴듯한 backup set은 final pathname에 게시되지 않습니다.

```text
성공: final directory에 database.sql + wordpress.tar.gz + manifest.json이 모두 존재
실패: final directory와 임시 sibling이 모두 존재하지 않고 source stack은 다시 healthy
```

핵심 S급 커밋 `6999190ffd34`는 capture와 publication을 분리합니다. private sibling directory에서 모든 파일을 만들고 검증·동기화한 뒤, 마지막 directory replacement 한 번으로만 final set을 보이게 합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`fdd55605ba74`](https://github.com/seungwoo7050/42-archive/commit/fdd55605ba74) | feat(backup): 백업 무결성과 비공개 파일 I/O 정의 | B | `PERSISTENCE`, `OPERATIONS` | private output, checksum, file/directory sync primitive를 정의합니다. |
| 2 | [`d26c885c5cd5`](https://github.com/seungwoo7050/42-archive/commit/d26c885c5cd5) | feat(backup): 관리 작업 신호와 테스트 중단 경계 추가 | A | `RECOVERY`, `TEST`, `HARD` | signal과 ordinary error를 같은 정리 경로로 보내고 정확한 중단 지점을 만듭니다. |
| 3 | [`3a0995ff0d4f`](https://github.com/seungwoo7050/42-archive/commit/3a0995ff0d4f) | feat(backup): 프로젝트별 백업 작업 잠금 적용 | B | `RECOVERY`, `OPERATIONS`, `PERSISTENCE` | 같은 project의 다른 관리 작업과 backup을 직렬화합니다. |
| 4 | [`b478b5243c5a`](https://github.com/seungwoo7050/42-archive/commit/b478b5243c5a) | feat(backup): DB 덤프와 WordPress 볼륨 수집 | A | `PERSISTENCE`, `CORE`, `INTEGRATION` | transaction DB dump와 WordPress archive를 stream으로 수집합니다. |
| 5 | [`0540ff1b5a4b`](https://github.com/seungwoo7050/42-archive/commit/0540ff1b5a4b) | feat(backup): 백업 출력 경로를 안전하게 예약 | A | `PERSISTENCE`, `RISK`, `EDGE` | 최종 경로를 미리 독점하고 device/inode로 동일 객체인지 확인합니다. |
| 6 | [`6999190ffd34`](https://github.com/seungwoo7050/42-archive/commit/6999190ffd34) | feat(backup): 백업 세트를 원자적으로 게시 | S | `PERSISTENCE`, `RECOVERY`, `HARD` | 완성·검증·동기화된 세트만 directory replacement로 게시하고 서비스를 복구합니다. |
| 7 | [`b6920a0c918c`](https://github.com/seungwoo7050/42-archive/commit/b6920a0c918c) | test(backup): 게시 실패와 중단 정리 검증 | A | `TEST`, `RECOVERY`, `PERSISTENCE` | non-publication, 임시 파일 정리, 서비스 복구, shared lock을 검사합니다. |
| 8 | [`030e7310c665`](https://github.com/seungwoo7050/42-archive/commit/030e7310c665) | test(backup): 자원 충돌과 시그널 경계 검증 | A | `TEST`, `PERSISTENCE`, `EDGE` | signal race, 큰 데이터 stream, 경로 충돌 범위까지 증거를 넓힙니다. |

## 1. 작은 primitive가 만드는 publication 전제

### `fdd55605ba74` — private file과 durability primitive

백업 파일은 생성되는 첫 순간부터 다른 사용자가 읽을 수 없어야 하고, 기존 파일을 덮어쓰면 안 됩니다.

```python
descriptor = os.open(
    path,
    os.O_WRONLY | os.O_CREAT | os.O_EXCL,
    0o600,
)
```

`private_output`은 caller가 stream을 쓰고 난 뒤 `flush`와 `fsync`를 수행합니다. `sha256_stream`은 stream position을 저장·복원해 checksum 계산 뒤 같은 stream을 다시 사용할 수 있게 하고, `fsync_directory`는 rename으로 바뀐 directory entry도 동기화합니다.

이 primitive만으로 두 artifact가 같은 시점의 세트가 되지는 않습니다. “개별 파일이 private하고 durable하다”는 전제만 만듭니다.

### `d26c885c5cd5` — signal을 cleanup 가능한 오류로 바꿈

SIGINT와 SIGTERM handler는 즉시 `os._exit`하지 않고 backup domain exception을 발생시킵니다. 그러면 normal exception과 signal cancellation이 같은 `finally`를 통과합니다.

테스트용 pause는 named stage에 도달했음을 mode `0600` ready file로 게시하고 잠듭니다. ready file 생성과 signal handler transition 사이의 작은 race를 줄이기 위해 SIGINT/SIGTERM을 잠깐 block한 뒤 ready file을 `fsync`하고 mask를 복원합니다.

이 방식이 다루지 못하는 신호도 명확합니다. SIGKILL과 host power loss는 Python handler를 실행하지 않으므로 이 Thread의 cleanup path를 보장하지 않습니다.

### `3a0995ff0d4f` — project operation lock

backup은 source stack의 Nginx와 WordPress를 멈추고 DB와 volume을 읽습니다. 같은 시간에 secret rotation이나 다른 backup이 실행되면 관찰 시점이 섞입니다. project name 기반 non-blocking exclusive lock은 시작부터 복구까지 전체 작업을 감쌉니다.

중요도가 B인 이유는 lock 자체가 backup consistency를 완성하지 않기 때문입니다. 하지만 lock이 없으면 이후 transaction의 전제가 성립하지 않습니다.

## 2. 두 artifact를 어떻게 수집하는가

### `b478b5243c5a` — MariaDB는 살아 있고 writer는 멈춘 상태

DB dump는 root password를 process argument에 넣지 않습니다. 표준 입력으로 전달해 컨테이너 내부의 mode `0600` 임시 client option file에 쓰고 종료 trap으로 지웁니다.

```text
mariadb-dump
  --single-transaction
  --routines --events --triggers
  --hex-blob
  --add-drop-database
  --databases "$MYSQL_DATABASE"
```

`--single-transaction`은 InnoDB 데이터의 논리적 snapshot을 얻는 선택입니다. MariaDB daemon은 dump 동안 실행되어야 합니다.

WordPress 쪽은 일회성 `tar` 컨테이너가 `/var/www`의 `html`과 `config`를 gzip stream으로 내보냅니다. web volume의 `html/wp-config.php`는 config volume을 가리키는 symlink이므로 archive에서 제외하고, 실제 private config directory는 별도로 포함합니다.

```text
tar -C /var/www -czf -
    --exclude=html/wp-config.php
    html config
```

capture 함수는 output을 메모리에 한 번에 올리지 않고 `output_stream`으로 private file에 흘립니다.

### 서비스 정지 순서

실제 stop ordering은 capture helper가 아니라 `6999190ffd34`의 orchestrator에 들어 있습니다.

- Nginx와 WordPress를 먼저 정지해 애플리케이션 writer와 외부 요청을 막습니다.
- MariaDB는 transactional dump를 위해 계속 실행합니다.
- capture와 publication이 끝나거나 실패하면 전체 stack을 `up --wait`로 복구합니다.

## 3. `0540ff1b5a4b` — final pathname을 미리 독점

출력 경로는 단순히 `exists()`를 한 번 확인하지 않습니다.

1. 상위 디렉터리를 strict resolve하고 실제 directory인지 확인합니다.
2. final output directory를 mode `0700`으로 생성해 이름을 예약합니다.
3. `lstat`의 device와 inode를 저장합니다.
4. publication 직전 같은 pathname이 동일 directory object이고 비어 있는지 다시 검사합니다.

```python
def same_directory(path, expected):
    actual = os.lstat(path)
    return (
        stat.S_ISDIR(actual.st_mode)
        and actual.st_dev == expected.st_dev
        and actual.st_ino == expected.st_ino
    )
```

pathname 문자열만 비교하면 다른 프로세스가 예약 directory를 지우고 같은 이름으로 새 directory나 symlink를 만들 수 있습니다. device/inode 확인은 처음 소유한 object가 여전히 그 자리에 있는지를 묻습니다.

## 4. `6999190ffd34` — capture와 publication을 분리한 transaction

### 4.1 성공 경로

실제 `_create_backup`의 상태 변화는 다음 순서입니다.

```text
세 서비스 running 확인
  → final output directory 0700 예약 + parent fsync
  → 같은 parent에 private temporary sibling 생성
  → Nginx·WordPress 정지
  → database.sql stream 생성·형식 확인
  → wordpress.tar.gz stream 생성·archive 확인
  → 두 checksum을 가진 manifest.json 0600 생성
  → temporary directory fsync
  → 예약 directory의 device/inode·empty 상태 재확인
  → os.replace(temporary, output)
  → parent directory fsync
  → compose up --detach --wait
```

핵심 publication 코드는 짧습니다.

```python
if not same_directory(output, reservation) or any(output.iterdir()):
    raise BackupError("백업 출력 예약 경로가 변경되었습니다")
os.replace(temporary, output)
published = True
fsync_directory(output.parent)
```

이 순간 전에는 final path가 빈 예약 directory이므로 partial artifact를 완전한 백업으로 읽을 수 없습니다. replacement 뒤에는 세 파일이 들어 있는 완전한 sibling directory가 한 번에 보입니다.

manifest는 세트의 의미를 고정합니다.

```json
{
  "format": 1,
  "created_at": "...",
  "project": "...",
  "sha256": {
    "database.sql": "...",
    "wordpress.tar.gz": "..."
  }
}
```

### 4.2 실패·signal 경로

`finally`는 현재 상태를 `stopped`, `published`, `temporary`, 예약 directory identity로 판단합니다.

- 서비스가 정지된 상태면 `compose up --wait`를 시도합니다.
- 남은 temporary sibling을 지웁니다.
- 게시되지 않았고 예약 object가 그대로라면 빈 final reservation을 지웁니다.
- directory entry 제거 뒤 parent를 다시 `fsync`합니다.
- 원래 오류와 서비스 복구 오류가 함께 있으면 복구 실패를 원래 오류에 연결해 보고합니다.

```python
if temporary is not None and temporary.exists():
    shutil.rmtree(temporary)
if not published and same_directory(output, reservation):
    output.rmdir()
if recovery_error:
    raise BackupError("백업 작업 뒤 서비스를 복구하지 못했습니다: ...") \
        from original_error
```

final pathname을 다른 객체로 바꾼 공격/경쟁 상태에서는 자신이 소유하지 않는 새 객체를 지우지 않습니다. cleanup 범위도 처음 예약한 inode와 만든 temporary directory에 한정됩니다.

### 최종 종료 상태

| 경로 | final output | temporary sibling | source stack |
| --- | --- | --- | --- |
| 성공 | 정확히 세 파일의 검증된 set | 없음 | healthy 복구 |
| dump/archive/manifest 실패 | 없음 | 없음 | 복구 시도 후 결과 보고 |
| SIGINT/SIGTERM | 없음 | 없음 | 같은 failure cleanup으로 복구 |
| 예약 경로 교체/오염 | 소유하지 않은 객체는 보존, 작업 실패 | 자신의 temp만 제거 | 복구 |
| publication 후 service recovery 실패 | 완성된 backup은 존재 | 없음 | 복구 실패를 명시적으로 반환 |

마지막 행은 중요한 trade-off입니다. 이미 atomic publication이 끝난 뒤 stack recovery가 실패하면 backup set을 다시 지우지 않습니다. 완성된 backup과 source stack의 운영 상태는 별개의 사실로 보고합니다.

## 5. 테스트가 겨냥하는 failure window

### `b6920a0c918c`

테스트 source는 다음 경로를 분리해 검사합니다.

- 기존 output path가 있으면 시작 전에 거부하고 변경하지 않음
- DB dump 뒤 failure injection 시 final/temp/ready marker가 남지 않음
- Nginx·WordPress 정지 직후 SIGINT/SIGTERM을 보내도 source stack이 다시 healthy
- 성공 시 final directory에 예상 세 파일만 있고 mode/checksum이 맞음
- 같은 project에서 다른 관리 작업이 lock을 잡고 있으면 backup이 시작되지 않음

종료 코드만 확인하지 않고 filesystem과 service health를 함께 관찰하도록 구성됐습니다.

### `030e7310c665`

이 커밋은 작은 happy path가 놓치는 두 영역을 넓힙니다.

- ready file이 게시되는 경계에서 SIGINT와 SIGTERM을 번갈아 여러 번 보내 race window를 반복합니다.
- 큰 WordPress 파일과 큰 DB 값을 넣어 stream copy, checksum, archive가 작은 fixture에만 우연히 맞는 구현이 아닌지 확인합니다.

같은 SHA에는 restore 관련 collision/large-data assertion도 섞여 있지만, 이 Thread에서는 backup capture와 signal publication 경계에 해당하는 부분만 포함합니다.

## 무엇까지 보장하는가

- complete set만 final pathname에 게시됩니다.
- individual files와 output directory는 private mode로 생성됩니다.
- checksum과 archive validation이 publication 전에 끝납니다.
- SIGINT/SIGTERM과 ordinary error는 같은 cleanup/recovery path를 통과합니다.
- 같은 project의 관리 작업은 겹치지 않습니다.
- 실패한 attempt가 소유한 reservation과 temporary sibling은 제거됩니다.

다음은 이 Thread 밖입니다.

- SIGKILL, host crash, storage hardware가 `fsync`를 지키지 않는 경우
- MyISAM처럼 `--single-transaction` snapshot에 포함되지 않는 storage engine semantics
- backup retention, encryption, remote upload
- 이미 게시된 set을 새 project에 적용하고 실패 자원을 지우는 restore transaction

## 검토 메모

exact SHA의 source와 테스트 코드를 확인했습니다. 이 세션에서는 실제 DB dump, signal scenario, 큰 fixture 테스트를 실행하지 않았습니다.
