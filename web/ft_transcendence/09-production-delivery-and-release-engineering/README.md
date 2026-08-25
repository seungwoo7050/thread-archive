# 09 — Production delivery and release engineering

이 디렉터리는 application 기능 구현 뒤에 남는 **빌드 산출물, image, process lifecycle, CI, runtime version, production configuration** 문제를 개발 Thread 단위로 재구성한다.

문서는 시간순 changelog가 아니다. 각 파일은 하나의 delivery 문제와 그 문제를 만들거나 드러낸 feature, 실제 fix, regression contract, 남은 비보장 범위를 묶는다. 같은 시기에 진행된 Thread 사이에는 반드시 선형 선후관계가 있는 것은 아니다.

## 학습 순서

| 문서 | 다루는 문제 | 핵심 도달점 |
| --- | --- | --- |
| [01-runtime-composition-and-reverse-proxy-evolution.md](01-runtime-composition-and-reverse-proxy-evolution.md) | 개발용 Compose와 browser gateway | `/api`·`/ws`·Web route graph, Web build→start, public metrics 차단, Caddy image artifact |
| [02-production-build-and-package-artifacts.md](02-production-build-and-package-artifacts.md) | typecheck-only workspace를 emitted artifact graph로 전환 | shared/DB/API ESM dist, DB SQL asset, Next standalone, ordered root build와 artifact postcondition |
| [03-container-images-and-production-runtime-lifecycle.md](03-container-images-and-production-runtime-lifecycle.md) | artifact를 source 없이 실행하고 startup/stop lifecycle을 조립 | non-root images, one-shot migration, health-gated single gateway, required secrets, 60s drain보다 긴 stop grace |
| [04-ci-production-process-and-browser-delivery-verification.md](04-ci-production-process-and-browser-delivery-verification.md) | verification command를 실제 DB/process/browser 실행으로 확장 | unit·Testcontainers·compiled process smoke·registered/demo browser job과 static contracts |
| [05-runtime-version-and-dependency-security-contracts.md](05-runtime-version-and-dependency-security-contracts.md) | 분산된 runtime/dependency version drift | canonical Node comparison, coordinated patch update, manifest+lockfile+override release graph |
| [06-production-configuration-and-durable-storage-fail-closed.md](06-production-configuration-and-durable-storage-fail-closed.md) | production의 silent memory repository fallback | effective production mode에서 DB URL 누락을 repository 생성·port bind 전에 거부 |

## Thread 사이의 실제 연결

```text
01 route/process mode
       ↓
02 emitted artifact graph
       ↓
03 image + migration/readiness/stop lifecycle
       ↓
04 CI에서 compiled process와 browser flow 실행

05 toolchain/dependency version은 02~04 전체의 build/runtime 입력을 가로지름
06 production storage guard는 03의 production config와 04의 demo/registered mode 분리를 가로지름
```

이 그림은 학습을 위한 dependency 관계이지 모든 commit이 이 순서대로만 진행됐다는 뜻은 아니다.

특히 DB migration artifact에는 cross-Thread 연결이 있다.

- `430389943b34`는 SQL을 `dist/migrations`에 복사하지만 당시 compiled migrator의 lookup은 아직 package-root path만 사용한다.
- persistence category의 `30aac132e14e`가 `./migrations`(compiled artifact)와 `../migrations`(source execution) 후보를 도입한다.
- production Compose가 one-shot migration image를 구성하는 `2c44cb7cd71f` 시점에는 이 보완이 이미 ancestry에 존재한다.

이 commit을 category 09의 map에 중복 편입하지 않고, 실제 artifact/lifecycle 연결을 이해하는 데 필요한 cross-Thread dependency로 명시했다.

## 문서 작성 기준

- 기존 문서의 Thread 구분, SHA, 제목, importance, tags를 초기 metadata로 유지했다.
- 각 설명은 표시된 exact SHA 또는 그 parent diff를 기준으로 작성했다.
- 같은 commit에 섞인 무관한 변경은 해당 Thread 본문에서 제외했다.
- final HEAD의 구현을 과거 commit의 상태처럼 소급하지 않았다.
- repository와 기존 서술이 충돌하면 repository를 우선했다. 예를 들어:
  - `be15e937d718`의 API `start`는 당시 `node dist`가 아니라 `tsx src/index.ts`다.
  - `e2c12ded1d5f`의 Docker contract는 secret 누락 render case를 별도로 실행하지 않고 required interpolation text를 검사한다.
  - `48c2188eb42a`의 canonical Node test는 CI와 API/Web Dockerfile을 비교하며 `.nvmrc`·package engine 전체까지 직접 검사하지 않는다.
- 작은 commit은 역할과 경계를 짧게 설명하고, artifact graph·container lifecycle·CI process job·fail-closed configuration처럼 중요한 변경은 source flow와 비보장 범위까지 깊게 추적했다.

## 실행 증거의 범위

GitHub의 대상 branch `web/ft_transcendence`에서 exact commit diff와 필요한 historical source/config/test를 검사했다. 이 작업 환경에서는 branch checkout, package install, build, Docker image/Compose, GitHub Actions, PostgreSQL/Testcontainers, Playwright를 실행하지 않았다. 따라서 문서는 source가 정의하는 contract와 test가 **무엇을 실행하도록 작성됐는지**를 설명하며, 실행 성공을 소급해 주장하지 않는다.
