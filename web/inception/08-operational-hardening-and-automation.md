# Thread 8 — 운영 방어선, 비공개 진단, 소유권이 제한된 자동화

> 원문 제목: **Operational hardening, private diagnostics, and bounded automation**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

이 Thread의 커밋은 하나의 기능을 시간순으로 확장한 단일 선형 작업이 아닙니다. 서로 다른 운영 실패를 막는 방어선이 병렬로 추가되고, 후반부에서 실제 Docker inspection·정리 ownership·local verifier·CI로 함께 검증됩니다.

다루는 문제는 다섯 축입니다.

1. Nginx와 MariaDB가 같은 network를 공유하지 않게 합니다.
2. container resource, 종료, 로그, 권한 정책을 제한합니다.
3. volume을 지우는 명령은 정확한 project name 확인 없이는 실행되지 않습니다.
4. 장애 진단 자료는 비밀값을 읽어 가릴 수 있을 때만 private set으로 게시합니다.
5. 테스트와 CI는 자신이 만든 project만 기록·정리하며 cleanup 실패를 성공으로 숨기지 않습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`27a3dca01d3b`](https://github.com/seungwoo7050/42-archive/commit/27a3dca01d3b) | feat(network): DB 트래픽을 내부 backend로 격리 | A | `STACK`, `RISK`, `ARCH` | public request network와 internal DB network를 분리합니다. |
| 2 | [`911544133fb4`](https://github.com/seungwoo7050/42-archive/commit/911544133fb4) | feat(runtime): 서비스 자원과 종료 한계 적용 | B | `OPERATIONS`, `RISK`, `STACK` | CPU/memory/PID/FD, stop, privilege, log rotation policy를 선언합니다. |
| 3 | [`74c285925325`](https://github.com/seungwoo7050/42-archive/commit/74c285925325) | fix(make): 볼륨 삭제 전에 확인을 요구 | A | `OPERATIONS`, `RISK`, `EDGE` | exact project confirmation 없이는 destructive `fclean`을 거부합니다. |
| 4 | [`ef74ad47ea81`](https://github.com/seungwoo7050/42-archive/commit/ef74ad47ea81) | feat(diagnostics): Compose 비밀값과 민감 항목 마스킹 | A | `OPERATIONS`, `SECRETS`, `RISK` | 진단에 들어갈 실제 secret/path를 찾아 fail-closed redaction을 만듭니다. |
| 5 | [`27a083d91c87`](https://github.com/seungwoo7050/42-archive/commit/27a083d91c87) | feat(diagnostics): 비공개 진단 세트와 CLI 연결 | A | `OPERATIONS`, `SECRETS`, `RISK` | exclusive private diagnostic directory와 CLI/Make target을 만듭니다. |
| 6 | [`7fbd41fe5af4`](https://github.com/seungwoo7050/42-archive/commit/7fbd41fe5af4) | test(operations): 자원·격리·삭제 보호·진단 검증 | A | `TEST`, `OPERATIONS`, `RISK` | live limits/network, deletion refusal, diagnostic redaction/publication을 검사합니다. |
| 7 | [`98e4af62e884`](https://github.com/seungwoo7050/42-archive/commit/98e4af62e884) | test(runtime): 프로세스·비밀값·정리 제어 흐름 강화 | A | `TEST`, `RECOVERY`, `OPERATIONS` | cleanup failure가 scenario 결과를 실패로 바꾸게 합니다. |
| 8 | [`2b35aa3d2217`](https://github.com/seungwoo7050/42-archive/commit/2b35aa3d2217) | test(cleanup): 테스트 프로젝트 소유 자원만 정리 | A | `OPERATIONS`, `RECOVERY`, `RISK` | exact project ownership record와 scoped leak cleanup을 추가합니다. |
| 9 | [`43ccded05e4f`](https://github.com/seungwoo7050/42-archive/commit/43ccded05e4f) | test(verify): 전체 스택 검증을 직렬 실행 | A | `TEST`, `OPERATIONS`, `RECOVERY` | 모든 local 검증을 timeout과 final cleanup 아래 직렬화합니다. |
| 10 | [`18508c25eef0`](https://github.com/seungwoo7050/42-archive/commit/18508c25eef0) | ci(stack): 정적·런타임·복구 검증 자동화 | A | `TEST`, `OPERATIONS`, `SUPPLY_CHAIN` | pinned action과 최소 권한으로 전체 scenario를 CI에 올립니다. |
| 11 | [`8a6c07988160`](https://github.com/seungwoo7050/42-archive/commit/8a6c07988160) | test(ci): workflow 검증 계약 추가 | A | `TEST`, `OPERATIONS`, `RISK` | workflow, timeout, secret boundary, cleanup, artifact allowlist 자체를 검사합니다. |

## 1. network reachability를 필요한 service pair로 제한

`27a3dca01d3b`은 하나의 `inception` network를 둘로 나눕니다.

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

service membership은 정확히 다음과 같습니다.

| 서비스 | frontend | backend | 이유 |
| --- | :---: | :---: | --- |
| Nginx | O | X | 외부 request와 WordPress FastCGI만 필요 |
| WordPress | O | O | Nginx 요청을 받고 MariaDB에 접근해야 함 |
| MariaDB | X | O | WordPress SQL만 수신 |

WordPress가 두 network를 잇지만 packet router 역할을 하는 것은 아닙니다. 두 network에 각각 application endpoint를 제공할 뿐입니다. Nginx는 MariaDB service DNS/addressability를 직접 얻지 않습니다.

`backend: internal: true`는 Docker network 밖으로의 route를 제한하지만 WordPress compromise가 DB credential을 사용하는 것까지 막지는 않습니다. network least privilege와 application authorization은 다른 문제입니다.

## 2. resource와 종료 정책: `911544133fb4`

각 service에 CPU, memory, PID, `nofile`, stop signal/grace period, `no-new-privileges`, json-file log rotation이 선언됩니다.

```yaml
cpus: ${NGINX_CPUS:-0.50}
mem_limit: ${NGINX_MEMORY:-128m}
pids_limit: ${NGINX_PIDS_LIMIT:-64}
ulimits:
  nofile:
    soft: ${NGINX_NOFILE_SOFT:-1024}
    hard: ${NGINX_NOFILE_HARD:-4096}
stop_signal: SIGQUIT
stop_grace_period: 15s
security_opt:
  - no-new-privileges:true
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

MariaDB와 WordPress는 더 큰 기본 memory/PID/FD 한계와 각 daemon에 맞는 stop signal/grace period를 갖습니다.

중요도 B인 이유는 선언 자체가 새 core behavior를 만들기보다 blast radius를 제한하는 운영 설정이기 때문입니다. 또한 YAML에 값이 있다고 Docker HostConfig에 실제 적용됐다는 증거는 아닙니다. `7fbd41fe5af4`가 live inspect로 이를 확인합니다.

## 3. `74c285925325` — destructive command는 target identity를 다시 입력해야 함

`make fclean`은 volume과 local image를 지우므로 일반 `down`과 성격이 다릅니다. generic `yes`가 아니라 selected `PROJECT_NAME`과 같은 값을 요구합니다.

```make
fclean:
	@test -n "$(PROJECT_NAME)" \
	  && test "$(DESTROY_CONFIRM)" = "$(PROJECT_NAME)" || { \
		echo "... DESTROY_CONFIRM=$(PROJECT_NAME)을 지정하십시오." >&2; \
		exit 2; \
	  }
	$(COMPOSE_RUN) down -v --rmi local --remove-orphans
```

operator가 어떤 project를 파괴하는지 확인 문자열에 포함시키므로, shell history에서 잘못 남은 `DESTROY_CONFIRM=yes`가 다른 project에도 재사용되지 않습니다. guard는 destructive command보다 먼저 실행되고 거부 시 exit 2를 반환합니다.

## 4. 진단은 비밀값을 읽을 수 없으면 아예 게시하지 않음

### `ef74ad47ea81` — redaction input 수집

진단 도구는 rendered Compose JSON의 `x-secret-files`를 읽고 다음 값을 모두 redaction set에 넣습니다.

- Compose에 적힌 raw secret path
- resolve한 absolute path
- hardened reader로 읽은 실제 secret value

한 secret file이라도 안전하게 읽지 못하면 “나머지만 가린 진단”을 만들지 않고 실패합니다. 무엇을 가려야 하는지 모르는 상태에서 partial diagnostic을 게시하는 것보다 자료를 만들지 않는 쪽을 선택합니다.

실제 redaction은 긴 값부터 바꿉니다.

```python
for value in sorted(secrets, key=len, reverse=True):
    redacted = redacted.replace(value, "<redacted>")
return SENSITIVE_ASSIGNMENT.sub(
    r"\1\2<redacted>",
    redacted,
)
```

긴 secret 안에 짧은 secret이 포함된 경우 짧은 값을 먼저 바꾸면 긴 값의 일부가 남을 수 있습니다. 추가 regex는 `password=...`, `secret: ...`, `token=...` 같은 구조적 assignment도 가립니다.

### `27a083d91c87` — private set으로만 publication

output directory는 이미 존재하면 거부하며 mode `0700`으로 새로 만듭니다. 각 파일은 `O_EXCL`, mode `0600`으로 생성됩니다.

수집 파일은 정확히 다음 다섯 개입니다.

- `versions.txt`
- `compose-ps.txt`
- `compose-logs.txt`
- `compose-model.txt`
- `container-state.txt`

각 command output을 redaction한 뒤 known secret/path가 남았는지 다시 scan하고 파일에 씁니다. 어느 단계에서든 실패하면 output directory 전체를 지웁니다.

```python
try:
    ...
    for filename, output in outputs.items():
        redacted = redact(output, secrets)
        if any(value in redacted for value in secrets):
            raise DiagnosticError(...)
        write_private(destination / filename, redacted)
except Exception:
    shutil.rmtree(destination, ignore_errors=True)
    raise
```

기존 output을 덮어쓰지 않으므로 이전 진단을 조용히 손상시키지 않고, dangling symlink도 “이미 존재하는 path”로 거부됩니다.

## 5. `7fbd41fe5af4` — 선언이 아니라 live Docker state를 검사

operations scenario는 세 service의 `docker inspect` 결과를 기대 정책과 exact 비교합니다.

- memory bytes, NanoCpus, PidsLimit
- stop signal과 timeout
- frontend/backend membership
- log driver와 rotation option
- `no-new-privileges:true`
- `nofile` soft/hard

network inspect로 `backend.Internal == true`와 exact member container ID set도 확인합니다.

### destructive refusal

confirmation 없이 `make fclean`을 실행해 exit 2와 `DESTROY_CONFIRM` 메시지를 요구하고, 거부 뒤 `/healthz`가 여전히 성공하는지 확인합니다. 단순 반환 코드뿐 아니라 guard가 destructive command 전에 멈췄는지 관찰합니다.

### 실제 로그에 secret을 넣어 redaction 확인

테스트는 request query에 실제 credential 값을 넣어 Nginx log에 노출되도록 한 뒤 diagnostic tool을 실행합니다. 결과 set에서 다음을 확인하도록 작성됐습니다.

- 파일 이름 exact set
- directory `0700`, files `0600`
- 실제 credential와 raw/resolved path 부재
- `<redacted>` 존재
- 일반 파일만 존재, symlink 없음
- 같은 output 재실행 거부와 기존 bytes 불변
- unreadable secret이면 exit 2, output directory 없음
- dangling symlink output 거부

이 방법은 “redact 함수 unit test”보다 강합니다. 실제 Compose log와 config/model output을 통과한 end-to-end diagnostic publication을 검사합니다.

## 6. 테스트 성공보다 cleanup까지 포함한 결과가 중요함

### `98e4af62e884`

이전 harness는 scenario가 성공하면 `finally`의 cleanup 오류를 출력만 하고 성공을 반환할 수 있었습니다. 변경 뒤 `close()`는 failure list를 반환하고, cleanup failure가 있으면 scenario result도 실패가 됩니다.

```python
cleanup_failures = stack.close(failed=failed)
if cleanup_failures:
    ...
    if result == 0:
        result = 1
```

정리 실패를 숨긴 테스트는 다음 시나리오에 port·volume·container를 남겨 false failure와 host 오염을 만듭니다. 테스트 결과의 일부로 cleanup을 취급하는 이유입니다.

이 commit은 private file replacement에도 `fsync`와 guaranteed temp cleanup을 추가하고, subprocess timeout 오류에 어떤 Compose operation이 멈췄는지 포함합니다.

### `2b35aa3d2217` — global prune 대신 exact ownership record

시나리오는 project 이름을 private record directory에 mode `0600` 파일로 기록합니다. record filename/content는 strict project pattern과 정확히 일치해야 하고 symlink·unsafe mode를 거부합니다.

normal `close()`는 다음만 제거합니다.

- 자신이 시작한 Compose project의 containers/volumes/networks
- 자신이 소유한 image prefix의 세 local tag
- 자신의 private temp directory와 secret files

crash 뒤 별도 `cleanup_test_resources.py`가 records를 읽어 project label과 exact image tag로 자원을 찾습니다. `docker system prune`, 모든 dangling volume 삭제, broad prefix search를 사용하지 않습니다. 다른 개발 stack을 지우지 않기 위해 cleanup 능력을 의도적으로 제한합니다.

cleanup report는 제거 후 다시 enumerate한 residue를 기록합니다. 도구 자체 오류와 실제 leak을 구분하는 exit status도 갖습니다.

## 7. local verifier와 CI가 ownership을 이어받음

### `43ccded05e4f` — 직렬 local verification

`tools/verify_stack.py`는 다음을 순서대로 실행합니다.

```text
make test
make config-strict ENV_FILE=.env.example
bootstrap
e2e
persistence
backup-restore
rotation
operations
```

각 command/scenario에 explicit timeout이 있고 첫 실패에서 다음 scenario를 중단합니다. 그러나 `finally`의 scoped cleanup은 항상 실행합니다.

결과 우선순위는 다음 의미를 갖습니다.

- scenario가 성공했는데 cleanup에서 leak를 찾으면 전체 실패
- cleanup tool 자체가 실행 불가능한 오류를 내면 기존 scenario 결과보다 우선
- scenario가 이미 실패했고 cleanup도 residue를 찾으면 failure state와 report를 보존
- cleanup까지 성공할 때만 temporary verification directory를 제거

### `18508c25eef0` — CI에서도 같은 lifecycle

GitHub Actions workflow는 다음 운영 결정을 명시합니다.

- Ubuntu 24.04 runner
- `contents: read` 최소 권한
- checkout/upload-artifact action을 commit SHA로 고정
- ref별 concurrency와 in-progress cancellation
- 전체 job timeout과 scenario별 timeout
- 정적 검사 뒤 여섯 runtime scenario
- `if: always()` scoped cleanup
- `if: failure()`에만 diagnostic artifact 업로드
- 업로드 path를 redacted 다섯 파일과 cleanup report allowlist로 제한
- 7일 retention, hidden file 제외

CI가 실패했을 때도 원본 `.env`나 secret directory 전체를 artifact로 올리지 않습니다. diagnostic tool이 만든 private/redacted 결과만 선택합니다.

### `8a6c07988160` — workflow와 test tooling 자체도 검증 대상

정적 validator는 단순 문자열 몇 개보다 넓은 계약을 검사합니다.

- Python AST로 `subprocess.run/check_output/check_call`과 `wait/communicate`에 explicit timeout이 있는지 확인
- credential이 process argument에 들어가는 금지 pattern 확인
- Compose service block이 정확히 세 개인지와 runtime secret mount 부재 확인
- Nginx가 private config volume을 mount하지 않고 WordPress만 mount하는지 확인
- start action의 secret read가 project lock `with` 안에 있는지 AST 위치로 확인
- workflow action pin, cleanup `always`, artifact allowlist, permissions/concurrency/timeout 존재 확인
- cleanup tool이 broad prune 대신 recorded project ownership을 사용하는지 확인

이 commit은 production runtime을 바꾸지 않습니다. 하지만 검증 체계가 timeout이나 secret boundary를 조용히 제거해 “통과하기 쉬운 테스트”가 되는 것을 막습니다.

## 운영 불변 조건 정리

| concern | 선언/구현 | 실제 evidence |
| --- | --- | --- |
| DB reachability | frontend/backend exact membership | live container/network inspect |
| resource exhaustion | CPU/memory/PID/FD limits | HostConfig exact comparison |
| graceful stop/log growth | signal/grace/log rotation | Config/HostConfig inspection |
| destructive deletion | exact project-name confirmation | refusal 뒤 stack health |
| diagnostics privacy | fail-closed redaction + private exclusive set | 실제 log secret 주입·rescan·mode 검사 |
| scenario cleanup | cleanup failures affect result | `close()` failure aggregation |
| crash cleanup | private project records + scoped labels/tags | post-cleanup re-enumeration/report |
| full verification | serial local runner + timeouts | cleanup result precedence |
| CI safety | least privilege, pinned actions, artifact allowlist | workflow contract validation |

## Thread 경계와 한계

- network split은 WordPress compromise가 DB credential을 사용하는 것을 막지 않습니다.
- Compose resource limits는 host 전체 capacity planning이나 autoscaling을 제공하지 않습니다.
- diagnostic redaction은 known secret values와 assignment pattern을 대상으로 하며 임의의 개인정보 분류기는 아닙니다.
- scoped cleanup은 record되지 않은 자원이나 외부 plugin이 만든 객체를 무조건 지우지 않습니다. 이는 누락 가능성과 타 project 보호 사이의 의도적 선택입니다.
- CI source가 존재한다는 것과 실제 workflow run이 성공했다는 것은 다릅니다.

## 검토 메모

각 exact SHA의 Compose, Python tool, Makefile, runtime test, workflow diff를 확인했습니다. 이 환경에서는 Docker inspection과 GitHub Actions workflow를 실행하지 않았습니다.
