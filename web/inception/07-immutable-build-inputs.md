# Thread 7 — 불변 빌드 입력과 실제 런타임 공급망 증거

> 원문 제목: **Immutable build inputs and runtime supply-chain evidence**  
> Project: `container-stack` · Branch: `web/inception`

## 개요

`debian:bookworm-slim`이나 live APT mirror처럼 움직이는 입력은 같은 Dockerfile을 다른 날 build했을 때 다른 bytes를 만들 수 있습니다. WordPress core를 bootstrap 시점에 내려받는 구조도 image digest만으로 실제 애플리케이션을 설명할 수 없게 합니다.

이 Thread는 공급망 보장을 세 층으로 나눕니다.

1. base image와 package repository를 immutable identity로 고정합니다.
2. WordPress와 WP-CLI artifact를 version+SHA-256으로 image 안에 넣습니다.
3. source 문자열뿐 아니라 running container의 package/application version도 확인합니다.

고정은 “영원히 업데이트하지 않는다”는 뜻이 아닙니다. `cd5982c8ea42`는 지원되는 보안 버전으로 pin 전체를 명시적으로 이동하면서도 moving input으로 돌아가지 않는 maintenance 방식을 보여 줍니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | [`3e29fbd34389`](https://github.com/seungwoo7050/42-archive/commit/3e29fbd34389) | build(images): Debian 이미지와 패키지 입력 고정 | A | `SUPPLY_CHAIN`, `RISK`, `ARCH` | Debian base digest와 dated package snapshot을 고정합니다. |
| 2 | [`f60ac8061c01`](https://github.com/seungwoo7050/42-archive/commit/f60ac8061c01) | build(wordpress): WordPress 산출물을 고정해 게시 | A | `SUPPLY_CHAIN`, `BOOTSTRAP`, `RISK` | WordPress/WP-CLI artifact를 checksum 검증하고 bootstrap에서 core를 게시합니다. |
| 3 | [`7b28cccaec1d`](https://github.com/seungwoo7050/42-archive/commit/7b28cccaec1d) | test(supply-chain): 불변 image 입력 검증 | A | `TEST`, `SUPPLY_CHAIN`, `RISK` | source pin과 running WordPress/WP-CLI identity를 고정합니다. |
| 4 | [`cd5982c8ea42`](https://github.com/seungwoo7050/42-archive/commit/cd5982c8ea42) | fix(supply-chain): 보안 지원 runtime pin 갱신 | B | `SUPPLY_CHAIN`, `RISK` | 검토된 immutable set을 새 보안 지원 버전으로 이동합니다. |
| 5 | [`127a70f6e4b2`](https://github.com/seungwoo7050/42-archive/commit/127a70f6e4b2) | test(supply-chain): 검토된 runtime 최소 버전 검증 | A | `TEST`, `SUPPLY_CHAIN`, `RISK` | 설치 package 최소선과 live PHP/MariaDB compatibility floor를 검사합니다. |

## 1. `3e29fbd34389` — base layer와 package universe를 각각 고정

세 Dockerfile은 같은 Debian dated slim image를 digest까지 지정합니다.

```dockerfile
FROM debian:bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63
```

여기서 tag의 날짜는 사람이 읽는 release identity이고, 실제 bytes를 고정하는 것은 digest입니다. tag가 가리키는 내용이 바뀌더라도 digest mismatch로 build가 실패합니다.

APT source도 live mirror에서 `snapshot.debian.org`의 explicit timestamp로 바뀝니다.

```dockerfile
RUN printf '%s\n' \
    'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm main' \
    'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm-updates main' \
    'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20241214T000000Z bookworm-security main' \
    > /etc/apt/sources.list
```

```text
base image digest     → root filesystem layer 고정
dated APT snapshot    → package index와 package version 집합 고정
```

둘 중 하나만 고정하면 충분하지 않습니다.

- base digest만 고정하고 live APT를 쓰면 install 결과가 날짜에 따라 달라집니다.
- snapshot만 고정하고 moving base tag를 쓰면 package install 전 filesystem이 달라질 수 있습니다.

snapshot metadata를 오래 뒤에도 사용할 수 있도록 `Valid-Until` 검사를 비활성화합니다. 재현성을 얻는 대신 자동 freshness 판단을 포기하므로, security update 책임은 explicit pin maintenance로 이동합니다.

이 커밋은 WordPress/WP-CLI 다운로드까지 고정하지 않습니다.

## 2. `f60ac8061c01` — WordPress core를 runtime download에서 image artifact로 이동

WordPress Dockerfile은 다음 artifact를 version과 SHA-256으로 고정해 build 중 다운로드·검증합니다.

- WP-CLI `2.11.0` phar — SHA-256 `a39021ac809530ea607580dbf93afbc46ba02f86b6cffd03de4b126ca53079f6`
- WordPress `6.7.1` archive — SHA-256 `33529cd638c845007e8e0d26c91d60c9c16b822c849c8deead03d0c851a26deb`

```dockerfile
ENV WP_CLI_VERSION=2.11.0 \
    WORDPRESS_VERSION=6.7.1

RUN curl -fsSL "https://wordpress.org/wordpress-${WORDPRESS_VERSION}.tar.gz" \
        -o /tmp/wordpress.tar.gz \
    && echo '33529cd638c845007e8e0d26c91d60c9c16b822c849c8deead03d0c851a26deb  /tmp/wordpress.tar.gz' \
        | sha256sum -c - \
    && tar -xzf /tmp/wordpress.tar.gz --strip-components=1 -C /usr/src/wordpress \
    && find . -type f ! -path './wp-content/*' -print0 \
        | sort -z \
        | xargs -0 sha256sum > /usr/src/wordpress-core.sha256
```

검증된 core는 image의 `/usr/src` 아래에 보관되고 manifest/checksum과 함께 bootstrap이 소비합니다. runtime entrypoint의 `wp core download`는 제거됩니다.

### core와 content를 다르게 취급

WordPress volume에는 project가 관리하는 core와 사용자 상태인 `wp-content`가 함께 있습니다. bootstrap은 image artifact의 core 파일을 기준으로 누락·불일치를 수렴시키되 content는 보존합니다.

```text
image-controlled
  wp-admin/
  wp-includes/
  root core files

volume-controlled
  wp-content/
  uploads/plugins/themes and application content
```

파일 publication은 target과 같은 directory의 temporary file로 copy하고 sync한 뒤 rename합니다. core manifest를 다시 검사해 중간 종료 뒤 일부 core만 남은 경우 다음 bootstrap이 보완할 수 있게 합니다.

이 변화는 Thread 2의 interruption recovery와도 연결됩니다. bootstrap 중 외부 WordPress 서버에서 다시 다운로드하지 않으므로, 재실행은 동일 image artifact를 사용합니다.

## 3. `7b28cccaec1d` — source pin과 running identity를 둘 다 검사

정적 검사는 Dockerfile에 다음이 있는지 확인합니다.

- dated Debian tag와 exact digest
- snapshot timestamp
- WordPress/WP-CLI version과 checksum
- `sha256sum -c`
- runtime `wp core download` 부재
- verified image artifact를 temporary→rename으로 게시하는 코드

하지만 source 문자열만 맞아도 stale local image cache나 잘못된 build artifact가 실행될 수 있습니다. runtime e2e는 running WordPress에서 core version과 WP-CLI version을 직접 조회합니다.

```text
source contract: 무엇을 build하도록 선언했는가
runtime evidence: 실제 container에서 무엇이 실행 중인가
```

두 검사의 역할은 다릅니다. source contract는 다음 build의 회귀를 막고, runtime identity는 현재 실행 image가 기대와 맞는지 확인합니다.

## 4. `cd5982c8ea42` — immutable set도 유지보수 대상

이 B급 커밋은 architecture를 바꾸지 않고 다음 pin을 한 세트로 갱신합니다.

- Debian base를 `bookworm-20260803-slim@sha256:abd67ffcfa541b485a3dff59865ab629aa048a6c613e639d36e7456b0b229241`로 갱신
- APT snapshot을 `20260812T000000Z`로 갱신
- WordPress core를 `6.7.7`로 갱신
- WordPress archive checksum을 `dadac21d0fd6f54f7c0565d8935d3e4baea5e649486ac6f40fe89792da498350`으로 갱신
- static/runtime expected value를 같은 세트로 갱신

중요한 점은 “latest” URL이나 moving tag로 되돌아가지 않았다는 것입니다. 이전 immutable set을 새 immutable set으로 review하여 교체합니다.

```text
old reviewed set
  → version/digest/checksum을 함께 변경
  → source test expected values 함께 변경
  → running version 검증 기대값 함께 변경
  → new reviewed set
```

중요도 B인 이유는 신규 보장보다 정기 maintenance에 가깝기 때문입니다. 그러나 이 commit이 빠지면 reproducible하지만 보안 지원이 낡은 image가 될 수 있습니다.

## 5. `127a70f6e4b2` — pin 문자열보다 실제 설치 결과를 확인

runtime harness는 `dpkg-query`와 `dpkg --compare-versions`로 각 container의 주요 package가 검토된 최소선 이상인지 확인합니다.

| 서비스 | package 예시 | 최소선 |
| --- | --- | --- |
| Nginx | `nginx` | `1.22.1-9+deb12u9` |
| Nginx | `openssl`, `libssl3` | `3.0.20-1~deb12u2` |
| WordPress | `php8.2-fpm`, `php8.2-cli` | `8.2.33-1~deb12u1` |
| WordPress | `libssl3` | `3.0.20-1~deb12u2` |
| MariaDB | `mariadb-server` | `1:10.11.18-0+deb12u1` |
| MariaDB | `libssl3` | `3.0.20-1~deb12u2` |

추가로 WordPress PHP runtime과 DB server가 최소 compatibility floor를 넘는지 확인합니다.

```python
WORDPRESS_REQUIRED_PHP = (7, 2, 24)
WORDPRESS_REQUIRED_MYSQL = (5, 5, 5)
```

DB version 문자열에는 `MariaDB`가 포함돼야 합니다. 숫자만 높은 다른 DB 구현을 동일한 runtime으로 오인하지 않습니다.

최소선 검사는 exact equality가 아닙니다. Dockerfile의 immutable snapshot이 build 선택을 고정하고, runtime minimum은 실제 실행 artifact가 보안·호환성 하한보다 낮지 않다는 별도 guard입니다.

## 최종 공급망 계약

| 층 | 고정/검증 방법 | 잡는 회귀 | 잡지 못하는 것 |
| --- | --- | --- | --- |
| base image | dated tag + digest | moving base layer | registry availability, build engine bug |
| Debian packages | dated snapshot | live mirror 변화 | snapshot 서비스 장기 보존 실패 |
| WP-CLI/core | version + SHA-256 | runtime moving download | upstream key provenance 자체 |
| bootstrap publication | image artifact + manifest/checksum | partial core, 재실행 download | user content corruption |
| source tests | Dockerfile/entrypoint strings와 구조 | pin 삭제·moving URL 회귀 | stale running image |
| runtime tests | live app/package version 조회 | stale/wrong image 실행 | 전체 CVE 분석 |

## Thread 경계

이 Thread는 build input identity와 runtime version evidence를 다룹니다. image signing, SBOM, provenance attestation, vulnerability scanner, registry replication은 포함하지 않습니다. 또한 pin이 최신 보안 상태인지 판단하려면 별도의 정기 review가 필요합니다.

## 검토 메모

exact SHA의 Dockerfile, entrypoint, 정적 검사, runtime assertion을 확인했습니다. 이 환경에서 실제 image build와 package/version 검사를 실행하지 않았습니다.
