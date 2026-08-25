# Thread 3 — 격리된 런타임 증거와 영속 상태 검증

> 원문 제목: **Isolated runtime evidence and persistent-state verification**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

소스에 올바른 Compose 설정이 있다는 사실과, 실제 Docker가 그 설정대로 컨테이너·volume·port를 만들었다는 사실은 다릅니다. 이 Thread는 고정된 project/container/image/port identity를 제거한 뒤, 독립된 임시 project를 만들고 두 종류의 런타임 증거를 수집합니다.

- 통합 경로 검증: HTTPS 요청이 Nginx, PHP-FPM, WordPress, MariaDB를 통과해 같은 데이터를 읽는가?
- 영속성 검증: 프로세스 restart와 컨테이너 recreation 뒤에도 authoritative volume과 데이터가 그대로인가?

둘은 서로 대체할 수 없습니다. 페이지가 한 번 보인다고 컨테이너 교체 뒤 데이터 보존이 증명되지는 않고, volume이 남는다고 외부 request path가 올바르다는 뜻도 아닙니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`9d75a34e290f`](https://github.com/seungwoo7050/42-archive/commit/9d75a34e290f) | feat(runtime): 프로젝트·이미지·포트·URL 격리 | A | `ARCH`, `STACK`, `OPERATIONS` | 고정된 project, image, port, canonical URL identity를 제거합니다. |
| 2 | [`2c436f574712`](https://github.com/seungwoo7050/42-archive/commit/2c436f574712) | test(bootstrap): 격리된 런타임 하네스 추가 | A | `TEST`, `ARCH`, `OPERATIONS` | 임시 비밀값과 독립 Docker 자원을 갖는 runtime harness를 만듭니다. |
| 3 | [`8c9b5b9adef2`](https://github.com/seungwoo7050/42-archive/commit/8c9b5b9adef2) | test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증 | A | `TEST`, `INTEGRATION`, `STACK` | 외부 HTTPS부터 DB까지 이어지는 동일 데이터 경로를 확인합니다. |
| 4 | [`fb1a689cf969`](https://github.com/seungwoo7050/42-archive/commit/fb1a689cf969) | test(persistence): 재시작·재생성 뒤 상태 보존 검증 | A | `TEST`, `PERSISTENCE`, `RISK` | restart와 recreation 뒤 DB·option·upload·volume identity를 확인합니다. |

## 1. `9d75a34e290f` — 독립 project를 만들 수 있는 입력으로 바꿈

이 커밋 전에는 고정 `container_name`, 고정 이미지 이름, host 443, `https://${DOMAIN_NAME}` 조합이 하나의 stack만 전제했습니다. 동일 host에서 두 검증을 병렬 또는 연속 실행하면 이름과 socket이 충돌하고, 테스트 cleanup이 개발자의 기본 stack을 건드릴 위험이 있습니다.

변경 뒤에는 다음 값이 caller 소유가 됩니다.

```yaml
image: ${STACK_IMAGE_PREFIX:-container-stack}-nginx:${STACK_IMAGE_TAG:-local}
ports:
  - "${HTTPS_BIND_ADDRESS:-0.0.0.0}:${HTTPS_PORT:-443}:443"
environment:
  WORDPRESS_URL: ${WORDPRESS_URL:?set WORDPRESS_URL}
```

고정 `container_name`은 제거되고 Compose project name이 컨테이너·network·volume namespace를 결정합니다. 테스트는 임의 project name과 image prefix를 선택할 수 있고, loopback의 임의 port를 사용할 수 있습니다.

`WORDPRESS_URL`을 별도 입력으로 둔 이유는 host port가 443이 아닐 수 있기 때문입니다. 실제 endpoint가 `https://stack.test:49123`인데 DB의 `home`과 `siteurl`이 `https://stack.test`로 고정되면 redirect와 테스트 요청이 어긋납니다.

이 커밋은 격리를 **가능하게** 하지만 실제로 안전한 harness가 그 parameter를 사용한다는 증거는 다음 커밋이 제공합니다.

## 2. `2c436f574712` — 테스트가 만든 자원만 소유하는 harness

런타임 harness는 시나리오마다 다음 자원을 새로 만듭니다.

- mode `0700` 임시 디렉터리
- mode `0600`의 네 비밀 파일
- PID와 random token이 포함된 Compose project name
- project별 image prefix
- loopback에서 예약한 HTTPS port
- test용 `.env`

개념상 한 시나리오의 소유 범위는 다음과 같습니다.

```text
RuntimeStack
 ├─ temp directory / env / secret files
 ├─ project-name namespace
 │   ├─ containers
 │   ├─ networks
 │   └─ named volumes
 ├─ project-specific local image tags
 └─ reserved host HTTPS port
```

### port 충돌을 숨기지 않는 재시도

port 예약과 Docker bind 사이에는 다른 프로세스가 같은 port를 차지할 수 있는 시간이 있습니다. harness는 startup error를 모두 “port 문제”로 취급하지 않습니다. stderr에 `address already in use`, `bind: address already in use`처럼 알려진 marker가 있고 선택한 port와 관련된 경우에만 새 port를 골라 제한된 횟수로 재시도합니다. 이미지 build 실패나 bootstrap 오류까지 무조건 port 재시도로 덮지 않습니다.

### source validation보다 강한 runtime inspection

harness는 container inspect와 `/proc` 관찰을 사용해 다음을 확인하도록 작성됐습니다.

- steady-state container에 `/run/secrets` mount가 없음
- 비밀번호 이름이나 실제 값이 environment와 process argument에 없음
- WordPress만 `/var/www/config`를 mount함
- Nginx에서는 `wp-config.php` symlink target을 실제로 읽을 수 없음
- completion marker가 volume에 존재함
- 각 service가 running/healthy 상태임

Compose YAML에 secret mount가 없다는 정적 검사만으로는 이전 컨테이너가 남아 있거나 다른 파일로 실행된 상태를 찾지 못할 수 있습니다. 반대로 inspect만으로는 source가 다음 build에서 다시 secret을 mount하지 않도록 막지 못합니다. 두 검사가 서로 보완합니다.

## 3. `8c9b5b9adef2` — 한 데이터가 네 계층을 통과하는지 확인

이 테스트는 단순 `/healthz` 확인을 넘어 애플리케이션 데이터를 만듭니다.

1. WordPress CLI로 고유한 게시물을 생성합니다.
2. 실제 HTTPS endpoint에 `curl --resolve`를 사용해 요청합니다.
3. 응답 HTML에서 같은 게시물 내용이 보이는지 확인합니다.
4. WordPress DB client 경로로 MariaDB를 조회해 같은 row가 존재하는지 확인합니다.

```text
write: wp-cli inside WordPress
        ↓
MariaDB row
        ↓
read: curl HTTPS → Nginx → FastCGI → WordPress
        ↓
response contains the same unique value
```

이 구성은 서로 다른 실패를 구분합니다.

- CLI write가 실패하면 WordPress↔MariaDB 연결 문제입니다.
- DB에는 row가 있지만 HTTPS에 보이지 않으면 Nginx/FastCGI/WordPress rendering 경로 문제입니다.
- HTTPS는 보이지만 별도 DB query가 실패하면 DB 인증 또는 client 경로 문제입니다.

같은 시나리오에서 처음 선택한 port를 의도적으로 점유해 startup이 새 port를 선택하는지도 확인합니다. 이 검사는 parameterization과 제한된 port-conflict recovery가 실제 함께 작동하도록 만든 것입니다.

### 이 테스트가 증명하지 않는 것

- 모든 WordPress route와 plugin behavior
- 인증·인가 전반
- 장기간 부하와 동시성
- 컨테이너 recreation 뒤 상태 보존

마지막 항목은 다음 커밋이 별도로 검사합니다.

## 4. `fb1a689cf969` — process restart와 container recreation을 분리

영속성 시나리오는 서로 다른 종류의 상태를 만듭니다.

- MariaDB의 게시물 row
- WordPress option 값
- WordPress upload 파일과 checksum
- 현재 project에 속한 세 named volume의 ID/name set

그 뒤 두 단계로 상태를 흔듭니다.

### 4.1 같은 컨테이너의 process restart

```text
compose restart
    → 동일 컨테이너/volume 연결에서 프로세스 재시작
    → 게시물·option·upload 재검증
```

이 단계는 entrypoint/runtime 시작이 기존 marker와 데이터를 손상하지 않는지 확인합니다.

### 4.2 컨테이너 제거 후 recreation

```text
compose down --remove-orphans        # --volumes 없음
compose up --detach --wait
    → 새 컨테이너 생성
    → 이전과 동일한 volume set인지 비교
    → 게시물·option·upload 재검증
```

여기서 단순히 데이터가 보이는지만 검사하지 않고, recreation 전후 project volume set이 같은지도 비교합니다. 우연히 새 volume에 bootstrap된 기본 데이터가 보이는 것을 영속성으로 오인하지 않기 위해서입니다.

세 volume은 역할이 다릅니다.

| volume | 권위 있는 상태 |
| --- | --- |
| MariaDB data | DB schema, users, WordPress rows |
| WordPress data | core/content/upload 및 completion marker |
| WordPress config | private `wp-config.php` |

### 최종적으로 확인하는 불변 조건

컨테이너는 교체 가능한 실행 단위이고, authoritative state는 project의 named volumes에 남아야 합니다. `down`에 `--volumes`를 붙이지 않은 recreation은 그 소유권을 유지합니다.

## 테스트 증거의 범위

| 시나리오 | 직접 관찰하는 것 | 범위 밖 |
| --- | --- | --- |
| bootstrap harness | 독립 project, private secret, runtime mount/env/arg 상태 | 모든 host 보안 설정 |
| e2e | HTTPS→FastCGI→WordPress→MariaDB의 한 데이터 round trip | 전체 기능·부하·브라우저 behavior |
| persistence/restart | 기존 컨테이너 프로세스 재시작 뒤 상태 | host reboot, filesystem corruption |
| persistence/recreate | 새 컨테이너가 같은 volume을 다시 사용 | volume 삭제·migration·cross-host restore |

## Thread 경계

이 Thread는 검증용 project의 격리와 통합·영속성 evidence만 다룹니다. backup set publication, fresh-project restore, credential rotation, CI automation은 각자 실패 모델과 cleanup ownership이 다르므로 별도 Thread에서 다룹니다.

## 검토 메모

표시된 exact SHA의 source와 테스트 assertion을 확인했습니다. 이 환경에서 Docker 시나리오를 직접 실행하지 않았으므로 “검증”은 테스트 코드가 수행하도록 작성된 관찰을 뜻합니다.
