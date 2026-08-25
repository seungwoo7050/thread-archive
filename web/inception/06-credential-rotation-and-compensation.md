# Thread 6 — 여러 저장소에 걸친 자격증명 회전과 보상

> 원문 제목: **Coordinated credential rotation and compensation**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

이 stack의 한 자격증명 세대는 파일 네 개를 바꾸는 것으로 끝나지 않습니다.

| 논리 값 | 실제 저장·검증 지점 |
| --- | --- |
| DB root password | host secret file, MariaDB root account |
| DB application password | host secret file, MariaDB application account, `wp-config.php` |
| WordPress admin password | host secret file, WordPress admin user row |
| WordPress author password | host secret file, WordPress author user row |

따라서 회전 도중 한 단계만 성공하면 old/new가 섞인 세대가 됩니다. 더 어려운 경우는 command가 state를 바꾼 뒤 실패를 반환하는 상황입니다. 반환 코드만 보고 “변경되지 않았다”고 가정할 수 없습니다.

이 Thread의 최종 불변 조건은 둘 중 하나입니다.

```text
성공: 모든 저장소가 replacement generation이고 old generation은 인증에 실패
실패: 모든 저장소가 verified previous generation이고 replacement는 인증에 실패
```

`9934b478c79a`가 ordered transaction을 조립하고, `2e6649a7706d`가 post-write ambiguity와 rollback 중 두 번째 signal이라는 핵심 결함을 고칩니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`a2d20b8c2c03`](https://github.com/seungwoo7050/42-archive/commit/a2d20b8c2c03) | feat(secrets): 교체 비밀 파일을 안전하게 읽고 게시 | A | `SECRETS`, `RISK`, `OPERATIONS` | replacement input 검증과 host file의 atomic/durable publication을 만듭니다. |
| 2 | [`832d182743ea`](https://github.com/seungwoo7050/42-archive/commit/832d182743ea) | feat(secrets): MariaDB 계정 비밀번호 원자 교체 | A | `SECRETS`, `RISK`, `INTEGRATION` | application/root account password를 stdin 기반 SQL로 교체합니다. |
| 3 | [`0aa998fdd344`](https://github.com/seungwoo7050/42-archive/commit/0aa998fdd344) | feat(secrets): WordPress 설정과 사용자 비밀번호 교체 | A | `SECRETS`, `RISK`, `INTEGRATION` | `wp-config.php`와 두 WordPress user password를 교체합니다. |
| 4 | [`64844c583211`](https://github.com/seungwoo7050/42-archive/commit/64844c583211) | feat(secrets): 신규 자격증명 수용과 기존 값 거부 검증 | A | `TEST`, `SECRETS`, `RISK` | 새 값의 성공과 이전 값의 실패를 모두 요구합니다. |
| 5 | [`c68486d55f30`](https://github.com/seungwoo7050/42-archive/commit/c68486d55f30) | feat(secrets): 회전 실패 시 기존 자격증명 복구 | A | `SECRETS`, `RECOVERY`, `HARD` | 여러 저장소를 이전 verified generation으로 되돌리는 compensation을 추가합니다. |
| 6 | [`9934b478c79a`](https://github.com/seungwoo7050/42-archive/commit/9934b478c79a) | feat(secrets): 스택 자격증명 회전 절차 연결 | S | `SECRETS`, `RECOVERY`, `CORE` | 잠금·순서·재기동·positive/negative verification을 하나의 rotation transaction으로 연결합니다. |
| 7 | [`2e6649a7706d`](https://github.com/seungwoo7050/42-archive/commit/2e6649a7706d) | fix(secrets): 회전 중단과 불명확한 상태를 보상 | S | `SECRETS`, `RECOVERY`, `HARD` | state 변경 뒤 실패와 rollback 중 추가 signal을 보상합니다. |
| 8 | [`0da35c72add5`](https://github.com/seungwoo7050/42-archive/commit/0da35c72add5) | test(secrets): 회전 롤백과 재시도 검증 | A | `TEST`, `SECRETS`, `RECOVERY` | 성공, 모든 failure stage, signal, rollback, retry, leak를 runtime으로 검사합니다. |
| 9 | [`2557079c2d19`](https://github.com/seungwoo7050/42-archive/commit/2557079c2d19) | test(secrets): 회전 후 런타임 비밀 경계 고정 | B | `TEST`, `SECRETS` | 테스트가 steady-state secret boundary를 약화시키지 못하게 정적으로 고정합니다. |

## 1. 각 저장소를 바꾸는 primitive

### `a2d20b8c2c03` — host file 하나를 안전하게 교체

current secret과 replacement secret 모두 같은 hardened read를 통과합니다.

- no-follow descriptor로 엽니다.
- regular file, hard link count 1, 현재 사용자 소유를 요구합니다.
- mode `0600`을 요구합니다.
- 크기·한 줄·password character/length를 검사합니다.
- replacement directory도 private real directory여야 합니다.

개별 host file publication은 target과 같은 directory에 임시 파일을 만들고 `fsync` 후 rename합니다.

```text
same-directory temporary 0600
  → write
  → flush + fsync(file)
  → os.replace(temp, target)
  → fsync(parent directory)
```

이 방식은 한 파일이 torn content로 보이는 것을 막습니다. 그러나 네 파일을 순서대로 교체하므로 첫 파일 뒤 process가 죽으면 group 전체는 여전히 mixed generation입니다. per-file atomicity와 multi-file transaction은 다릅니다.

### `832d182743ea` — MariaDB account 변경

DB password와 SQL은 process argument에 넣지 않습니다. root password를 stdin으로 받아 container 내부 private client option file을 만들고 SQL을 실행합니다. identifier와 SQL literal을 별도로 escape합니다.

helper는 application account와 root account를 선택적으로 바꿀 수 있습니다. 후속 fix에서 중요한 test hook도 이 primitive에 붙습니다.

```sql
ALTER USER '<app>'@'%' IDENTIFIED BY '<new-app>';
ALTER USER 'root'@'localhost' IDENTIFIED BY '<new-root>';
FLUSH PRIVILEGES;
```

이 SQL command가 실패했다고 해서 앞의 `ALTER USER`가 취소됐다고 가정할 수 없습니다. MariaDB account mutation은 일반 파일 rename처럼 caller가 commit 여부를 반환 코드 하나로 알 수 있는 primitive가 아닙니다.

### `0aa998fdd344` — WordPress config와 user row

`wp-config.php`는 PHP helper가 stdin JSON에서 새 DB password를 읽고, 정확히 하나의 `DB_PASSWORD` definition을 바꿉니다. target과 같은 filesystem에 temp file을 만든 뒤 mode·owner·group을 맞추고 `fsync`한 다음 rename합니다.

WordPress admin/author password는 WordPress API를 통해 user row를 갱신합니다. 이 역시 “row를 바꾼 뒤 helper가 실패”할 수 있는 test hook을 갖습니다.

이 커밋이 제공하는 것은 세 개의 mutation primitive입니다.

- WordPress admin password 변경
- WordPress author password 변경
- private config의 DB password 변경

아직 이들을 DB account와 host file의 순서 안에 넣지는 않습니다.

## 2. `64844c583211` — 성공 확인만으로는 부족함

`verify_rotation`은 현재 generation의 각 값이 실제 consumer에서 동작하는지 확인합니다.

```text
root password       → root SQL SELECT 1
application password→ WordPress container에서 MariaDB SELECT 1
DB config password  → wp-config.php parsed value 비교
admin/user password → wp_check_password
```

replacement를 검증할 때는 old generation도 함께 넘기고, old 값이 **실패해야** 성공으로 봅니다.

- old root가 여전히 동작하면 root account가 바뀌지 않은 것입니다.
- old app password가 동작하면 DB grant credential이 혼재합니다.
- old WordPress user password가 동작하면 user row가 교체되지 않았습니다.

positive probe만으로는 “새 값도 추가로 허용됨”을 잡지 못합니다. positive와 negative probe를 함께 사용해야 generation 교체를 확인할 수 있습니다.

`find_root_password`는 uncertain state에서 current와 replacement 후보를 실제 `SELECT 1`로 시험해, rollback이 어느 root credential로 DB에 들어가야 하는지 판단합니다. 저장된 진행 flag보다 authoritative system의 인증 결과를 믿습니다.

## 3. `c68486d55f30` — 이전 상태로 되돌리는 compensation

rollback은 역순 `undo` 몇 줄이 아닙니다. 어떤 단계까지 반영됐는지 알 수 없기 때문에 각 저장소에 old 값을 다시 **설정**하고 마지막에 검증합니다.

```text
MariaDB가 접근 가능한 상태인지 확보
  → replacement/current 중 동작하는 root credential 탐색
  → application DB password를 old로 설정
  → wp-config.php DB password를 old로 설정
  → admin/author password를 old로 설정
  → root DB password를 old로 설정
  → host secret files 네 개를 old로 게시
  → services force-recreate
  → old success + replacement rejection + host files 비교
```

각 compensation step의 오류는 모아 두고 가능한 다음 step을 계속 시도합니다. 예를 들어 WordPress user 복구가 실패해도 DB root와 host file 복구를 포기하지 않습니다. 마지막 verification이 통과해야 `recovered=True`입니다.

이 커밋은 rollback primitive를 만들지만, forward mutation과 어떤 순서로 연결하고 언제 호출할지는 `9934b478c79a`가 결정합니다.

## 4. `9934b478c79a` — 여러 저장소를 하나의 ordered transition으로 연결

### 시작 전 전제

project operation lock 안에서 다음을 확인합니다.

- current host files가 안전하고 서로 다른 path임
- replacement directory와 네 파일이 안전함
- 모든 replacement 값이 current와 다름
- MariaDB application user 이름이 안전함
- WordPress admin/author가 존재하고 서로 다름
- current generation이 실제 모든 consumer에서 동작함
- replacement generation은 아직 거부됨

### forward 순서

실제 `_rotate`의 순서는 다음과 같습니다.

```text
1. Nginx stop                         # 외부 요청 차단
2. WordPress admin password 변경
3. WordPress author password 변경
4. wp-config.php의 DB password 변경
5. MariaDB application password 변경
6. MariaDB root password 변경
7. host secret files 네 개 게시
8. services force-recreate + wait
9. replacement success / current rejection / host file 내용 검증
```

왜 config를 먼저 바꾸고 DB application password를 나중에 바꾸는가라는 질문에는 “중간 상태가 잠깐 존재한다”는 답이 먼저입니다. 이 작업은 DB와 filesystem에 걸친 진짜 atomic commit을 제공할 수 없습니다. 대신 Nginx를 멈춰 외부 traffic을 차단하고, 실패 시 어느 단계에서든 old generation을 다시 설정하는 compensation을 사용합니다.

root password를 host files보다 먼저 바꾸기 때문에 그 사이 process가 죽으면 host의 current root file은 실제 DB와 다릅니다. 그래서 rollback은 file 값을 맹신하지 않고 current/replacement 두 후보를 실제 인증해 usable root를 찾습니다.

### 실패 처리

forward block에서 어떤 exception이 나도 `rollback_rotation`을 호출합니다. rollback이 old generation을 검증하면 Nginx 차단 상태를 풀고, 원래 오류에 “롤백 완료”를 붙여 실패를 반환합니다. rollback도 실패하면 “롤백 불완전”과 수집한 각 저장소의 오류를 보고합니다.

이 commit의 핵심은 성공 반환보다 실패 후 상태를 정의한 것입니다.

## 5. `2e6649a7706d` — 반환 코드와 실제 state가 다른 경우를 보상

### 5.1 post-write ambiguity를 의도적으로 재현

MariaDB helper는 password를 바꾼 뒤 다음 statement로 command를 실패시킬 수 있습니다.

```sql
ALTER USER ... IDENTIFIED BY ...;
FLUSH PRIVILEGES;
SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT='injected rotation failure';
```

WordPress PHP helper도 config rename 또는 user password update 뒤 예외를 발생시킬 수 있습니다. caller가 nonzero만 보고 “변경 전”이라고 판단하면 rollback 대상에서 빠져 mixed state가 남습니다.

fix는 progress flag로 어느 저장소가 바뀌었다고 추측하지 않습니다. 실패 지점과 관계없이 rollback이 모든 저장소에 old 값을 다시 적용하고 positive/negative probe로 최종 상태를 확인합니다.

### 5.2 rollback 중 두 번째 signal을 지연

첫 SIGINT/SIGTERM은 forward 회전을 중단시키는 exception이 됩니다. 그러나 rollback이 진행 중일 때 같은 방식으로 두 번째 signal을 raise하면 recovery 자체가 중간에서 끊깁니다.

```python
def interrupt(signum, _frame):
    if signal_state["rollback_active"]:
        signal_state["deferred"] = True
        return
    raise RotationError(f"{signal_name} 신호로 회전이 중단되었습니다")
```

failure handler는 compensation 전에 다음 state를 설정합니다.

```python
signal_state["rollback_active"] = True
rollback_errors, recovered = rollback_rotation(...)
```

두 번째 signal은 사라지지 않습니다. `deferred=True`로 기록되어 최종 오류 메시지에 “롤백 중 추가 종료 신호 지연 처리”가 포함됩니다. 다만 rollback control flow를 즉시 끊지는 않습니다.

### 5.3 최종 제어 흐름

```python
try:
    # ordered forward mutations
except BaseException as original_error:
    signal_state["rollback_active"] = True
    rollback_errors, recovered = rollback_rotation(...)
    ...
    raise RotationError(
        f"회전 실패 ({original_error}); {detail}"
    ) from original_error
finally:
    if blocked:
        project.run("up", "--detach", check=False)
```

`blocked`는 외부 service를 멈춘 뒤 아직 verified success/rollback에 도달하지 못한 상태를 나타냅니다. 최종 fallback `up`은 best effort이며, rollback verification을 대신하지 않습니다.

## 6. forward와 compensation 비교

| 저장소 | forward | compensation | authoritative 검증 |
| --- | --- | --- | --- |
| WP admin/author | replacement user password 설정 | current user password 다시 설정 | `wp_check_password`; 반대 세대 거부 |
| `wp-config.php` | replacement DB password atomic rename | current DB password atomic rename | parsed config value |
| MariaDB app | replacement로 `ALTER USER` | usable root로 current를 재설정 | app `SELECT 1`; 반대 세대 거부 |
| MariaDB root | replacement로 `ALTER USER` | current/replacement 중 usable root를 찾아 current로 재설정 | root `SELECT 1`; 반대 세대 거부 |
| host files | replacement 네 파일 순차 게시 | current 네 파일 순차 게시 | hardened read와 exact value 비교 |
| services | force-recreate | force-recreate | healthy + runtime secret boundary |

## 7. 테스트 evidence

### `0da35c72add5`

runtime matrix는 다음 failure를 각각 주입하도록 작성됐습니다.

- users/config/app-password/root-password 단계 직후
- admin user, config, app DB, root DB가 state를 바꾼 뒤 command failure
- 첫 host file만 게시된 시점
- 네 host files 게시 뒤
- WordPress container를 제거한 recreation failure
- host files 이후 SIGTERM
- rollback ready marker 뒤 두 번째 SIGINT

각 실패 뒤 old generation의 네 종류 인증이 모두 성공하고 replacement가 모두 실패하는지, host files가 old로 돌아왔는지, temporary credential file이 남지 않았는지 검사합니다. 그 뒤 같은 replacement로 다시 회전해 retry 가능성도 확인합니다.

성공 case에서는 반대로 replacement가 모두 성공하고 old가 모두 실패하며, runtime mount/env/argument/log에 current와 replacement 비밀값이 남지 않는지 확인합니다.

### `2557079c2d19`

이 B급 정적 검사는 runtime test가 편의를 위해 `/run/secrets`를 다시 mount하거나 config temporary cleanup과 secret-boundary assertion을 제거하는 회귀를 막습니다. production invariant를 새로 만드는 commit은 아니므로 supporting evidence로만 다룹니다.

## 최종 상태와 한계

보장하는 것:

- 한 project에서 rotation은 다른 관리 작업과 겹치지 않습니다.
- secret은 process argument 대신 stdin/private file을 통해 전달됩니다.
- success는 replacement acceptance와 old rejection을 모두 요구합니다.
- failure는 모든 저장소에 old 값을 다시 설정하고 old acceptance/replacement rejection을 검증합니다.
- state 변경 뒤 실패를 반환하는 command도 보상 대상입니다.
- rollback 중 추가 SIGINT/SIGTERM은 recovery를 중단하지 않고 지연 기록됩니다.

보장하지 않는 것:

- host crash·SIGKILL·storage failure가 정확히 multi-store 중간에서 발생한 뒤 자동 resume하는 journal
- 외부 secret manager와 distributed transaction
- 네 host file을 하나의 filesystem atomic operation으로 교체하는 기능
- rollback이 불가능한 경우 자동으로 어떤 generation을 선택해야 하는지

rollback이 불완전하면 도구는 mixed state 가능성을 숨기지 않고 각 단계 오류를 보고합니다.

## 검토 메모

각 exact SHA의 diff/source와 runtime test matrix를 확인했습니다. 이 세션에서 실제 회전·failure injection·double-signal scenario를 실행하지 않았습니다.
