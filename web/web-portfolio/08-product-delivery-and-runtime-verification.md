===== BEGIN FILE: 01-reproducible-toolchain-and-production-server-verification.md =====
# Thread: Reproducible toolchain and production-server verification

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 원본 Development Thread를 변경하지 않고, 같은 branch history에 product-delivery 관점을 추가한 확장 workbook입니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 `commit/commit-importance.md`의 branch-scoped 분류를 사용합니다.
- Phase 1 audit에서 category/thread grouping과 commit set을 실제 history에 대조한 뒤 이 문서를 freeze했습니다.
- Phase 2는 freeze된 구조와 fixed metadata를 바꾸지 않고 learner-facing 기록만 완성합니다.
- 다른 branch의 구현이나 final HEAD를 과거 SHA 설명에 소급하지 않습니다.
- 실행하지 않은 build/test/CI/Docker 결과는 exact-SHA source inspection과 구분합니다.

## 1. Thread 목표

개발 환경의 암묵적 버전 차이를 제거하고, 최적화된 production server를 실제 browser test 대상으로 만든 뒤 같은 검증을 CI의 기본 전달 gate로 승격하는 과정을 복원합니다.

### 계획된 핵심 invariant

- 지원 Node.js와 npm 버전은 repository metadata와 CI가 같은 값으로 해석합니다.
- 브라우저 검증은 development server가 아니라 production build를 시작한 server를 대상으로 수행할 수 있습니다.
- CI는 fresh install부터 정적 검사, content validation, production browser verification까지 하나의 재현 가능한 경로로 수행합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 버전 파일과 `package.json` 선언은 어떤 소비자에게 읽히며, 잘못된 local runtime을 실제로 차단하는가?
- `test:e2e:production`은 build, server 시작, readiness와 browser matrix를 어떤 순서로 소유하는가?
- CI가 local production E2E 경로를 재사용하는 지점과 아직 포함하지 않는 release 검증은 무엇인가?
- 각 단계의 timeout, cancellation, server reuse와 권한 경계는 failure를 어떻게 제한하는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 변경 file, function, config, script와 workflow step을 확인했습니다.
- Source, generated artifact, CI gate, container/runtime owner를 구분했습니다.
- Missing artifact, portability failure, threshold violation, startup failure와 cleanup branch를 기록했습니다.
- Test/CI command의 technique, production path, proves/does-not-prove와 실제 실행 여부를 구분했습니다.
- 최종 product-delivery 흐름과 cross-thread handoff를 코드 없이 설명할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 확장 thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f66b880a8f97` | chore(runtime): 지원 Node.js와 npm 버전 고정 | B | DEPLOY | 초기 전달 경계 — runtime/package-manager 버전을 repository contract로 고정합니다. |
| 2 | `f81691072413` | test(e2e): production server 검증 경로 추가 | A | VALIDATION, DEPLOY, TEST | 회귀·artifact 검증 — production build와 별도 server port를 사용하는 browser verification 경로를 만듭니다. |
| 3 | `9fd3541c11dc` | ci: 기본 배포 품질 검사 추가 | A | DEPLOY, TEST | 통합 gate — 고정 toolchain과 production E2E를 GitHub Actions의 기본 delivery gate로 연결합니다. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA의 tree와 parent diff를 기준으로 작성합니다. 다른 Thread의 later commit은 관계 설명에만 사용하고 과거 구현에 소급하지 않습니다.

### 1. `f66b880a8f97` — chore(runtime): 지원 Node.js와 npm 버전 고정

- **Full SHA:** `f66b880a8f975443974896891a71e1dfd70fbe32`
- **Importance:** B
- **Tags:** DEPLOY
- **확장 thread에서의 역할:** 초기 전달 경계 — runtime/package-manager 버전을 repository contract로 고정합니다.

#### 해당 SHA에서 확인할 실제 코드

- `f66b880a8f97^`와 비교해 `.node-version`, `.nvmrc`, `package.json`, `package-lock.json`에 추가된 정확한 Node/npm 값을 대조합니다.
- `packageManager`와 `engines`가 npm/version manager/CI에서 각각 어떤 방식으로 소비되는지 구분합니다.
- 이 SHA 자체에는 잘못된 runtime을 강제 종료하는 script나 CI가 있는지 확인하고, 선언과 enforcement를 혼동하지 않습니다.
- 후속 `9fd3541c11dc`가 `.nvmrc`와 npm pin을 실제 workflow 입력으로 소비하는 연결을 기록합니다.

확인 원칙:

- 먼저 `f66b880a8f97^`와 `f66b880a8f97`를 비교하고, 필요한 file은 `f66b880a8f97:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | `package.json`과 lockfile은 있었지만 repository가 지원 Node.js/npm의 단일 값을 선언하지 않았습니다. 따라서 개발자 shell과 자동화가 서로 다른 runtime/package-manager로 같은 lockfile을 해석할 여지가 있었습니다. |
| 실제 변경 file/symbol/command/artifact | `.node-version`과 `.nvmrc`에 `24.18.0`을 추가하고, `package.json`의 `packageManager`를 `npm@11.16.0`, `engines.node`를 `24.18.0`, `engines.npm`을 `11.16.0`으로 고정했습니다. `package-lock.json`의 root package metadata에도 같은 engines가 기록됩니다. |
| Build/runtime/resource owner와 lifetime | 버전 값의 owner는 repository metadata입니다. `.node-version`/`.nvmrc`는 version manager와 CI setup step이 읽고, `packageManager`/`engines`는 package tooling이 읽습니다. 이 commit은 process나 resource lifetime을 새로 만들지 않습니다. |
| Failure·missing output·cleanup 처리 | 이 SHA에는 version mismatch를 직접 검사해 실패시키는 command가 없습니다. npm의 engine 처리는 환경 설정에 따라 warning에 그칠 수 있고, 현재 shell이 실제로 선언값을 사용했는지도 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 네 곳의 runtime 선언과 lockfile root metadata가 같은 값으로 수렴합니다. 그러나 dependency 설치의 네트워크 가용성, OS 차이, 또는 잘못된 runtime의 강제 차단까지 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `f81691072413`이 production build/test command를 만들고, `9fd3541c11dc`가 `.nvmrc`로 Node를 설정한 뒤 npm 11.16.0을 설치해 이 선언을 실행 경로로 승격합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 `.node-version`과 `.nvmrc`가 없고 `package.json`에도 `packageManager`/`engines`가 없습니다.
- **해당 SHA 핵심 코드:** `f66b880a8f975443974896891a71e1dfd70fbe32` · `package.json 및 version files`

```text
.node-version / .nvmrc
24.18.0

package.json
"packageManager": "npm@11.16.0",
"engines": {
  "node": "24.18.0",
  "npm": "11.16.0"
}
```

- **관찰 근거의 성격:** Exact-SHA diff에서 직접 확인한 repository metadata입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `f81691072413`이 production build/test command를 만들고, `9fd3541c11dc`가 `.nvmrc`로 Node를 설정한 뒤 npm 11.16.0을 설치해 이 선언을 실행 경로로 승격합니다.

### 2. `f81691072413` — test(e2e): production server 검증 경로 추가

- **Full SHA:** `f81691072413c11707251491fdb5af0c67e716ec`
- **Importance:** A
- **Tags:** VALIDATION, DEPLOY, TEST
- **확장 thread에서의 역할:** 회귀·artifact 검증 — production build와 별도 server port를 사용하는 browser verification 경로를 만듭니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `start:e2e`와 `test:e2e:production`을 추적해 build가 server 시작보다 먼저 실패할 수 있는 경계를 확인합니다.
- `playwright.production.config.ts`의 `webServer.command`, `url`, `reuseExistingServer`, `timeout`, `workers`와 두 device project를 기록합니다.
- 기존 `tests/e2e`를 재사용한다는 사실과 production 전용 fixture를 새로 만드는 것이 아니라는 점을 구분합니다.
- 이 경로가 `next start`를 검증하지만 `.next/standalone/server.js`나 container image를 실행하지 않는다는 non-guarantee를 명시합니다.

확인 원칙:

- 먼저 `f81691072413^`와 `f81691072413`를 비교하고, 필요한 file은 `f81691072413:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | 기존 Playwright 경로는 있었지만 production build를 먼저 만들고 최적화된 `next start` process를 별도 port에서 검증하는 command/config가 없었습니다. 따라서 development compiler에서 통과한 route matrix가 production serving에서도 동작하는지는 별도 계약이 아니었습니다. |
| 실제 변경 file/symbol/command/artifact | `start:e2e`를 `next start -p 3200`으로 추가하고, `test:e2e:production`을 `npm run build && playwright test --config=playwright.production.config.ts`로 연결했습니다. 새 config는 `http://localhost:3200`, `reuseExistingServer: false`, server timeout 120초, worker 1, Desktop Chrome와 Pixel 7을 사용합니다. |
| Build/runtime/resource owner와 lifetime | package script가 build 선행 조건을 소유하고, Playwright `webServer`가 server process의 시작·readiness 대기·test 종료 시 정리를 소유합니다. E2E specification은 기존 `tests/e2e`가 그대로 소유합니다. |
| Failure·missing output·cleanup 처리 | `npm run build`가 실패하면 browser phase로 진행하지 않습니다. server가 120초 안에 URL을 제공하지 못하면 Playwright가 실패하며, 기존 server 재사용을 금지해 다른 process의 성공을 오인하지 않습니다. test assertion 실패도 command exit를 실패로 만듭니다. |
| 보장하는 것과 보장하지 않는 것 | 동일한 desktop/mobile E2E matrix가 optimized production server를 대상으로 실행될 수 있습니다. standalone artifact 구조, container, 성능 budget, 운영 환경 네트워크나 production content mode는 아직 검증하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `9fd3541c11dc`가 이 exact command를 CI의 마지막 검증 step으로 호출합니다. 뒤의 standalone/container Thread는 같은 build output을 더 좁은 artifact/runtime 경계에서 검증합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent의 `package.json`에는 `test:e2e`만 있고 production-specific build/start command와 `playwright.production.config.ts`가 없습니다.
- **해당 SHA 핵심 코드:** `f81691072413c11707251491fdb5af0c67e716ec` · `package.json, playwright.production.config.ts`

```text
"start:e2e": "next start -p 3200",
"test:e2e:production":
  "npm run build && playwright test --config=playwright.production.config.ts"

webServer: {
  command: "npm run start:e2e",
  url: "http://localhost:3200",
  reuseExistingServer: false,
  timeout: 120_000,
}
```

- **관찰 근거의 성격:** Exact-SHA package/config diff에서 직접 확인한 production E2E contract입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `9fd3541c11dc`가 이 exact command를 CI의 마지막 검증 step으로 호출합니다. 뒤의 standalone/container Thread는 같은 build output을 더 좁은 artifact/runtime 경계에서 검증합니다.

### 3. `9fd3541c11dc` — ci: 기본 배포 품질 검사 추가

- **Full SHA:** `9fd3541c11dc8fa86561c3cd0b7116449b8f0f03`
- **Importance:** A
- **Tags:** DEPLOY, TEST
- **확장 thread에서의 역할:** 통합 gate — 고정 toolchain과 production E2E를 GitHub Actions의 기본 delivery gate로 연결합니다.

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml`의 trigger, permissions, concurrency, timeout과 step 순서를 parent diff에서 확인합니다.
- `actions/setup-node`가 `.nvmrc`를 읽고, 별도 global install이 npm 11.16.0을 고정하는 이중 소비 관계를 추적합니다.
- `npm ci` 이후 lint → typecheck → content check → Chromium install → production E2E 순서와 fail-fast 성질을 기록합니다.
- 이 SHA의 workflow가 unit test, standalone verify, bundle/Lighthouse, Docker를 아직 실행하지 않는다는 범위를 명시합니다.

확인 원칙:

- 먼저 `9fd3541c11dc^`와 `9fd3541c11dc`를 비교하고, 필요한 file은 `9fd3541c11dc:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | production E2E command는 local opt-in 경로였고 push/pull request에서 동일한 toolchain과 command를 강제하는 workflow가 없었습니다. 검증을 수행했다는 사실이 integration 조건이 아니었습니다. |
| 실제 변경 file/symbol/command/artifact | read-only `CI` workflow를 추가했습니다. push와 pull request에서 `.nvmrc` 기반 Node setup, npm 11.16.0 global pin, version 출력, `npm ci`, lint, typecheck, content validation, Chromium 설치, `npm run test:e2e:production`을 순서대로 실행합니다. job은 30분 timeout과 ref별 concurrency cancellation을 갖습니다. |
| Build/runtime/resource owner와 lifetime | GitHub Actions `verify` job이 fresh runner와 step ordering을 소유합니다. repository는 command만 제공하고, Actions runner가 checkout·tool cache·process lifecycle을 관리합니다. `permissions: contents: read`로 workflow authority를 제한합니다. |
| Failure·missing output·cleanup 처리 | 각 shell step의 non-zero exit가 후속 step을 막습니다. 30분을 넘으면 job이 중단되고, 같은 ref의 이전 run은 새 run으로 취소됩니다. Chromium 설치나 dependency registry 접근 실패도 delivery gate 실패로 표면화됩니다. |
| 보장하는 것과 보장하지 않는 것 | push/PR마다 pinned toolchain에서 fresh install과 production browser path가 자동 요구됩니다. 이 시점에는 `npm test`, standalone file completeness, performance gate, Docker runtime 또는 외부 hosting 배포가 포함되지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | Thread 3의 `c5e73853a1b6`, Thread 4의 `abbd530368a0`, Thread 5의 `b94fa6dd0118`이 같은 workflow에 artifact, performance, container gate를 단계적으로 추가합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 `.github/workflows/ci.yml`이 없습니다.
- **해당 SHA 핵심 코드:** `9fd3541c11dc8fa86561c3cd0b7116449b8f0f03` · `.github/workflows/ci.yml`

```text
- uses: actions/setup-node@v4
  with:
    node-version-file: .nvmrc
    cache: npm
- run: npm install --global npm@11.16.0
- run: npm ci
- run: npm run lint
- run: npm run typecheck
- run: npm run content:check
- run: npx playwright install --with-deps chromium
- run: npm run test:e2e:production
```

- **관찰 근거의 성격:** Exact-SHA workflow diff에서 직접 확인한 CI step graph입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** Thread 3의 `c5e73853a1b6`, Thread 4의 `abbd530368a0`, Thread 5의 `b94fa6dd0118`이 같은 workflow에 artifact, performance, container gate를 단계적으로 추가합니다.

## 6. Invariant ledger

| Invariant | 이전 상태 | 도입·수정 | 검증·소비 | 남은 비보장 |
| --- | --- | --- | --- | --- |
| Runtime version discovery | 환경별 암묵값 | `f66b880a8f97`에서 Node 24.18.0/npm 11.16.0을 여러 metadata 경계에 선언 | `9fd3541c11dc`가 `.nvmrc`와 npm pin을 실제 CI 입력으로 사용 | local shell의 강제 일치 여부 |
| Browser target | development path만 존재 | `f81691072413`에서 build 후 `next start`를 별도 port로 실행 | `9fd3541c11dc`가 production E2E command를 push/PR gate로 실행 | standalone/container runtime |
| Delivery gate ownership | local 수행 여부에 의존 | `9fd3541c11dc`에서 read-only, bounded CI job이 순서를 소유 | 후속 artifact/performance/container gate로 확장 | 외부 배포와 운영 |

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Fix/decision | Test·gate evidence | 한계 |
| --- | --- | --- | --- |
| 서로 다른 Node/npm이 lockfile을 해석 | `f66b880a8f97`의 네 metadata 경계 | `9fd3541c11dc`의 version setup·출력·fresh install | 잘못된 local shell은 강제 차단하지 않음 |
| dev server 통과를 production 성공으로 오인 | `f81691072413`의 build-first production Playwright config | `9fd3541c11dc`가 exact command를 CI에서 실행 | standalone/container는 별도 Thread |
| 검증이 local 선택사항 | `9fd3541c11dc`의 push/PR workflow | step non-zero/timeout/cancellation이 integration을 차단 | unit test와 후속 release gate는 아직 없음 |

## 8. Ownership / state / responsibility 변화

| 대상 | 이전 owner/state | 중간 변화 | 최종 owner/state |
| --- | --- | --- | --- |
| Toolchain 선택 | 개발자/runner의 설치 상태 | repository version metadata | GitHub Actions setup + npm pin |
| Production process | production-specific owner 없음 | Playwright `webServer`가 `next start` lifecycle 소유 | CI가 같은 command를 호출 |
| Integration 판정 | 수동 local 결과 | local production E2E command | bounded `verify` job의 exit status |

## 9. Thread 최종 상태

브랜치는 지원 Node/npm을 명시하고, production build를 만든 뒤 별도 `next start` process에서 기존 desktop/mobile route matrix를 실행할 수 있습니다. CI는 fresh checkout에서 같은 버전과 command를 사용해 push/PR을 판정합니다. 다만 이 Thread의 마지막 SHA만으로는 standalone 파일 배치, 성능 budget, container image, production hosting을 검증하지 않습니다.

## 10. 최종 product-delivery flow 정리

version metadata → Actions가 Node/npm 선택 → `npm ci` → lint/type/content checks → `npm run build` → Playwright가 port 3200의 `next start`를 시작하고 기다림 → 기존 E2E matrix 실행 → command/job exit status가 integration 결과가 됩니다.

## 11. 학습 완료 자가 점검

- [x] 네 runtime metadata 위치와 같은 값이 반복되는 이유를 설명했습니다.
- [x] production E2E의 build/server/readiness/test lifecycle owner를 구분했습니다.
- [x] CI의 step 순서, 권한, timeout과 cancellation을 확인했습니다.
- [x] 이 Thread가 standalone, performance, container, hosting을 보장하지 않는다고 기록했습니다.
- [ ] Exact-SHA runtime command를 직접 실행해 결과를 기록했습니다. — 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
===== END FILE: 01-reproducible-toolchain-and-production-server-verification.md =====

===== BEGIN FILE: 02-self-contained-production-build-and-portability.md =====
# Thread: Self-contained production build and portability

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 원본 Development Thread를 변경하지 않고, 같은 branch history에 product-delivery 관점을 추가한 확장 workbook입니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 `commit/commit-importance.md`의 branch-scoped 분류를 사용합니다.
- Phase 1 audit에서 category/thread grouping과 commit set을 실제 history에 대조한 뒤 이 문서를 freeze했습니다.
- Phase 2는 freeze된 구조와 fixed metadata를 바꾸지 않고 learner-facing 기록만 완성합니다.
- 다른 branch의 구현이나 final HEAD를 과거 SHA 설명에 소급하지 않습니다.
- 실행하지 않은 build/test/CI/Docker 결과는 exact-SHA source inspection과 구분합니다.

## 1. Thread 목표

production build가 외부 font fetch와 암묵적인 compiler·framework native binary·CSS transform 상태에 기대지 않도록 만들고, fresh Linux/container 환경에서도 같은 입력과 설정으로 artifact를 만들 수 있는 경계를 복원합니다.

### 계획된 핵심 invariant

- 빌드에 필요한 font binary와 license/provenance 정보는 repository가 소유하며 `next/font/local`이 소비합니다.
- production build compiler는 generated manifest를 해석하는 downstream tooling과 합의된 webpack 경로로 고정됩니다.
- Next.js runtime, lint tooling과 platform-specific SWC package는 같은 patch line과 lockfile resolution을 사용합니다.
- Tailwind utility 변환은 dependency 존재 여부가 아니라 명시적인 root PostCSS configuration으로 활성화됩니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 외부 font provider 호출을 제거할 때 binary, license, source record와 CSS variable compatibility는 각각 누가 소유하는가?
- source-level TypeScript 성공과 실제 production CSS/font output 성공 사이에 어떤 build-only failure가 남는가?
- webpack compiler pin, Next patch update와 GNU/musl SWC lockfile metadata가 portability에 어떤 서로 다른 역할을 갖는가?
- 각 fix에 직접적인 regression test가 있는지, 아니면 broad build/visual verification만 존재하는지 구분할 수 있는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 변경 file, function, config, script와 workflow step을 확인했습니다.
- Source, generated artifact, CI gate, container/runtime owner를 구분했습니다.
- Missing artifact, portability failure, threshold violation, startup failure와 cleanup branch를 기록했습니다.
- Test/CI command의 technique, production path, proves/does-not-prove와 실제 실행 여부를 구분했습니다.
- 최종 product-delivery 흐름과 cross-thread handoff를 코드 없이 설명할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 확장 thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7872e1214de7` | fix(font): 빌드용 글꼴과 출처를 저장소에서 제공 | A | DEPLOY, DEBUG | 외부 build dependency 제거 — Google Font fetch를 repository-owned WOFF2와 `next/font/local`로 교체합니다. |
| 2 | `2f65f6a6fcb6` | test(font): 로컬 글꼴과 license 경계 검증 | A | VALIDATION, DEPLOY, TEST | 결정적 source/file regression — local font registration, WOFF2 file와 license notice를 검사합니다. |
| 3 | `404a220e5d40` | fix(build): production build에 webpack compiler 고정 | B | DEPLOY, DEBUG | compiler contract 고정 — production output을 downstream manifest tooling이 이해하는 webpack 형식으로 만듭니다. |
| 4 | `5d903132306a` | fix(deps): Next.js runtime 보안 패치 적용 | B | DEPLOY, DEBUG | framework/native portability maintenance — Next, ESLint config와 platform SWC resolution을 같은 patch line으로 갱신합니다. |
| 5 | `1de3d36e3a48` | fix(build): Tailwind utility CSS 변환 복원 | A | DEPLOY, DEBUG | production transform 복원 — installed Tailwind PostCSS plugin을 explicit root config로 등록합니다. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA의 tree와 parent diff를 기준으로 작성합니다. 다른 Thread의 later commit은 관계 설명에만 사용하고 과거 구현에 소급하지 않습니다.

### 1. `7872e1214de7` — fix(font): 빌드용 글꼴과 출처를 저장소에서 제공

- **Full SHA:** `7872e1214de7b5d58722358998c6d63cfbe9f279`
- **Importance:** A
- **Tags:** DEPLOY, DEBUG
- **확장 thread에서의 역할:** 외부 build dependency 제거 — Google Font fetch를 repository-owned WOFF2와 `next/font/local`로 교체합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`에서 `next/font/google` import와 세 font registration이 `next/font/local`로 어떻게 바뀌는지 비교합니다.
- `src/app/fonts/FONT_SOURCES.md`, 세 WOFF2, 두 OFL license file의 version·size·SHA-256 기록과 실제 file ownership을 확인합니다.
- `SourceHanSerifKR`를 사용하면서 기존 CSS variable `--font-noto-serif-kr`을 유지하는 compatibility decision을 추적합니다.
- `src/designs/editorial/editorial-route.module.css` 등 font-family consumer가 hard-coded name 대신 generated CSS variable을 사용하도록 바뀌는지 확인합니다.

확인 원칙:

- 먼저 `7872e1214de7^`와 `7872e1214de7`를 비교하고, 필요한 file은 `7872e1214de7:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | `src/app/layout.tsx`가 `next/font/google`의 Geist, Geist Mono, Noto Serif KR를 사용했습니다. 따라서 production build가 font provider에 접근할 수 없는 환경에서는 source가 유효해도 font download 단계에서 실패할 수 있었습니다. |
| 실제 변경 file/symbol/command/artifact | 세 원본 WOFF2와 OFL license를 `src/app/fonts/**`에 저장하고, 출처·version·size·SHA-256을 `FONT_SOURCES.md`에 기록했습니다. layout은 `next/font/local`로 variable font weight range와 `display: swap`을 선언합니다. Source Han Serif KR를 사용하지만 기존 style contract를 깨지 않도록 `--font-noto-serif-kr` 이름은 유지하고 CSS consumer를 variable 기반으로 바꿉니다. |
| Build/runtime/resource owner와 lifetime | font binary와 license lifetime은 repository가 소유하고 build가 bundle에 포함합니다. `src/app/layout.tsx`가 font registration과 HTML class binding을 소유하며, renderer CSS는 제공된 variable만 소비합니다. 외부 provider는 build path에서 제거됩니다. |
| Failure·missing output·cleanup 처리 | missing/corrupt local file은 build 또는 file read에서 실패할 수 있습니다. 이 commit 자체에는 file signature나 license 검사가 없으며, 문서의 SHA-256과 실제 binary가 일치하는지 자동 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 이 세 font를 구성하기 위해 build-time Google Fonts 요청이 필요하지 않습니다. npm registry/base image 등 다른 dependency network, 모든 CSS의 외부 URL 부재, glyph coverage와 실제 browser rendering 품질은 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `2f65f6a6fcb6`이 layout import, local path, WOFF2 magic과 license text를 검증합니다. 후속 performance work는 큰 CJK font의 route별 loading cost를 별도로 최적화하지만 이 SHA에 소급하지 않습니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** `src/app/layout.tsx`는 `next/font/google`에서 `Geist`, `Geist_Mono`, `Noto_Serif_KR`를 import했습니다.
- **해당 SHA 핵심 코드:** `7872e1214de7b5d58722358998c6d63cfbe9f279` · `src/app/layout.tsx`

```text
import localFont from "next/font/local";

const geistSans = localFont({
  display: "swap",
  src: "./fonts/Geist-Variable.woff2",
  variable: "--font-geist-sans",
  weight: "100 900",
});

const koreanSerif = localFont({
  src: "./fonts/SourceHanSerifKR-Variable.woff2",
  variable: "--font-noto-serif-kr",
  weight: "250 900",
});
```

- **관찰 근거의 성격:** Exact-SHA diff와 repository-owned font/source/license files에서 직접 확인했습니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `2f65f6a6fcb6`이 layout import, local path, WOFF2 magic과 license text를 검증합니다. 후속 performance work는 큰 CJK font의 route별 loading cost를 별도로 최적화하지만 이 SHA에 소급하지 않습니다.

### 2. `2f65f6a6fcb6` — test(font): 로컬 글꼴과 license 경계 검증

- **Full SHA:** `2f65f6a6fcb6e753497ded0f4a8948cfa0238c48`
- **Importance:** A
- **Tags:** VALIDATION, DEPLOY, TEST
- **확장 thread에서의 역할:** 결정적 source/file regression — local font registration, WOFF2 file와 license notice를 검사합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/app/local-fonts.test.ts`가 layout source를 어떻게 읽고 Google provider token을 어떤 정규식으로 금지하는지 확인합니다.
- `it.each`의 configured source path와 physical filename 쌍을 추적하고 `wOF2` magic 검사 범위를 기록합니다.
- license 검사가 두 file의 특정 문구만 확인하며 provenance hash나 전체 license equivalence를 검증하지 않는다는 점을 구분합니다.
- production build/browser를 실행하는 test가 아니라 source + repository file boundary test라는 technique를 분류합니다.

확인 원칙:

- 먼저 `2f65f6a6fcb6^`와 `2f65f6a6fcb6`를 비교하고, 필요한 file은 `2f65f6a6fcb6:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | local font file과 license가 repository에 들어왔지만, layout이 다시 Google import로 돌아가거나 file이 사라져도 이를 빠르게 잡는 focused regression이 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `src/app/local-fonts.test.ts`를 추가했습니다. layout source에 `next/font/local`이 있고 `next/font/google`, `fonts.googleapis.com`, `fonts.gstatic.com`이 없음을 확인합니다. 세 local path가 layout에 존재하고 각 file의 첫 네 byte가 `wOF2`인지 검사하며, 두 license file에 OFL 1.1 표기가 있는지 검사합니다. |
| Build/runtime/resource owner와 lifetime | Vitest test process가 source와 binary/license file을 read-only로 읽습니다. fixture를 복사하거나 mutation하지 않으며, 각 `readFileSync` 호출의 file descriptor lifetime은 Node가 호출 단위로 관리합니다. |
| Failure·missing output·cleanup 처리 | layout token mismatch, missing file, 잘못된 magic, license 문구 부재가 assertion 또는 file-read error로 실패합니다. binary 전체 corruption, recorded SHA-256, browser font load, glyph coverage와 build output은 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | local registration과 최소 file/license shape가 source regression으로 보호됩니다. 이 test 통과만으로 production build가 성공하거나 font가 시각적으로 올바르게 표시된다고 결론 내릴 수 없습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `7872e1214de7`의 local ownership invariant를 직접 보호합니다. compiler/CSS 변환은 뒤의 별도 fix와 cross-thread build/visual 검증이 맡습니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 `src/app/local-fonts.test.ts`가 없습니다.
- **해당 SHA 핵심 코드:** `2f65f6a6fcb6e753497ded0f4a8948cfa0238c48` · `src/app/local-fonts.test.ts`

```text
expect(layoutSource).toContain('from "next/font/local"');
expect(layoutSource).not.toMatch(
  /next\/font\/google|fonts\.googleapis\.com|fonts\.gstatic\.com/,
);

const font = readFileSync(resolve(projectRoot, "src/app/fonts", fileName));
expect(font.subarray(0, 4).toString("ascii")).toBe("wOF2");
```

- **관찰 근거의 성격:** Exact-SHA test implementation에서 직접 확인한 static repository contract입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `7872e1214de7`의 local ownership invariant를 직접 보호합니다. compiler/CSS 변환은 뒤의 별도 fix와 cross-thread build/visual 검증이 맡습니다.

### 3. `404a220e5d40` — fix(build): production build에 webpack compiler 고정

- **Full SHA:** `404a220e5d408d39e360a7fe2149042a6e2af3ee`
- **Importance:** B
- **Tags:** DEPLOY, DEBUG
- **확장 thread에서의 역할:** compiler contract 고정 — production output을 downstream manifest tooling이 이해하는 webpack 형식으로 만듭니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `build`가 `next build`에서 `next build --webpack`으로 바뀌는 단일 diff를 확인합니다.
- 같은 SHA의 `dev`가 이미 `next dev --webpack`이라는 점과 development/production compiler 정렬을 기록합니다.
- 이 commit에는 generated manifest parser나 regression test가 아직 없음을 확인하고 후속 `c24c350ce42c`와 연결합니다.
- compiler pin이 application semantics를 검증하는 것이 아니라 output format 선택을 소유한다는 점을 명시합니다.

확인 원칙:

- 먼저 `404a220e5d40^`와 `404a220e5d40`를 비교하고, 필요한 file은 `404a220e5d40:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | development command는 webpack을 명시했지만 production `build`는 compiler를 명시하지 않았습니다. framework default가 바뀌면 development와 production output 형식이 달라질 수 있었습니다. |
| 실제 변경 file/symbol/command/artifact | `package.json`의 `build`를 `next build --webpack`으로 바꿨습니다. 다른 script나 source file은 변경하지 않습니다. |
| Build/runtime/resource owner와 lifetime | compiler 선택의 owner가 framework default에서 repository package script로 이동합니다. 모든 downstream command가 `npm run build`를 호출할 때 동일한 compiler path를 사용합니다. |
| Failure·missing output·cleanup 처리 | webpack build 자체의 failure는 command non-zero로 나타나지만 이 SHA에는 선택값을 검사하는 test가 없습니다. webpack manifest format과 parser의 실제 호환성도 아직 구현되지 않았습니다. |
| 보장하는 것과 보장하지 않는 것 | repository의 canonical production build command가 webpack을 명시합니다. Next 내부 output format의 영구 안정성이나 다른 사람이 직접 `next build`를 호출하는 경우까지 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | Thread 4의 `c2fb8a7c238d` parser와 `c24c350ce42c` compiler contract test가 이 선택을 실제 measurement invariant로 사용합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** `"build": "next build"`였습니다.
- **해당 SHA 핵심 코드:** `404a220e5d408d39e360a7fe2149042a6e2af3ee` · `package.json`

```text
"build": "next build --webpack"
```

- **관찰 근거의 성격:** Exact-SHA one-line package script diff입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** Thread 4의 `c2fb8a7c238d` parser와 `c24c350ce42c` compiler contract test가 이 선택을 실제 measurement invariant로 사용합니다.

### 4. `5d903132306a` — fix(deps): Next.js runtime 보안 패치 적용

- **Full SHA:** `5d903132306a1ab6db0fe715415e1527f63ebb93`
- **Importance:** B
- **Tags:** DEPLOY, DEBUG
- **확장 thread에서의 역할:** framework/native portability maintenance — Next, ESLint config와 platform SWC resolution을 같은 patch line으로 갱신합니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`에서 `next`와 `eslint-config-next`가 16.2.4에서 16.2.11로 함께 이동하는지 확인합니다.
- `package-lock.json`에서 `@next/env`, lint plugin, 각 OS/CPU SWC package가 같은 patch line으로 해석되는지 추적합니다.
- Linux ARM64/x64 GNU와 musl package에 추가된 `libc` constraint가 container/native binary selection에 미치는 역할을 기록합니다.
- repository가 특정 CVE 번호, exploit 재현이나 security regression test를 제공하는지 확인하고 없는 사실을 명시합니다.

확인 원칙:

- 먼저 `5d903132306a^`와 `5d903132306a`를 비교하고, 필요한 file은 `5d903132306a:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | Next runtime과 matching ESLint/compiler package가 16.2.4 patch line에 고정돼 있었습니다. commit subject는 security maintenance 필요를 나타내지만 repository evidence에는 특정 CVE나 재현 scenario가 없습니다. |
| 실제 변경 file/symbol/command/artifact | `next`와 `eslint-config-next`를 16.2.11로 올리고 lockfile의 `@next/env`, `@next/eslint-plugin-next`, darwin/linux/windows SWC package를 같은 version으로 갱신했습니다. Linux GNU/musl native package에는 명시적 `libc` metadata가 기록됩니다. |
| Build/runtime/resource owner와 lifetime | runtime/compiler dependency resolution은 `package.json`과 lockfile이 소유합니다. install 시 npm이 OS·CPU·libc 조건에 맞는 optional SWC package를 선택합니다. application code ownership은 바뀌지 않습니다. |
| Failure·missing output·cleanup 처리 | 잘못된 native package 선택은 install/build failure로 드러날 수 있습니다. 이 commit에는 CVE-specific test, runtime exploit reproduction, application behavior regression test가 없습니다. |
| 보장하는 것과 보장하지 않는 것 | framework, lint plugin과 native compiler artifact가 16.2.11 patch line으로 정렬되고 libc 조건이 lockfile에 남습니다. 모든 보안 취약점 제거, future vulnerability 부재, multi-architecture runtime 실행 성공은 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | 앞의 webpack pin과 뒤의 build measurement/container install이 같은 patched dependency graph를 사용합니다. 이 commit은 font나 Tailwind failure를 직접 고친 것이 아니라 portability thread의 framework/native dependency 경계를 보강합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** `next`와 `eslint-config-next`가 16.2.4이고 lockfile의 matching packages도 16.2.4였습니다.
- **해당 SHA 핵심 코드:** `5d903132306a1ab6db0fe715415e1527f63ebb93` · `package.json`

```text
"dependencies": {
  "next": "16.2.11"
},
"devDependencies": {
  "eslint-config-next": "16.2.11"
}
```

- **관찰 근거의 성격:** Exact-SHA dependency/lockfile diff와 branch commit body에서 확인했습니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** 앞의 webpack pin과 뒤의 build measurement/container install이 같은 patched dependency graph를 사용합니다. 이 commit은 font나 Tailwind failure를 직접 고친 것이 아니라 portability thread의 framework/native dependency 경계를 보강합니다.

### 5. `1de3d36e3a48` — fix(build): Tailwind utility CSS 변환 복원

- **Full SHA:** `1de3d36e3a485830b0a459cbc9dc9748ca15d763`
- **Importance:** A
- **Tags:** DEPLOY, DEBUG
- **확장 thread에서의 역할:** production transform 복원 — installed Tailwind PostCSS plugin을 explicit root config로 등록합니다.

#### 해당 SHA에서 확인할 실제 코드

- parent에 `@tailwindcss/postcss` dependency는 있지만 root `postcss.config.mjs`가 없는지 구분합니다.
- 새 config의 export shape와 plugin key를 확인하고 Next production build가 conventional root config를 발견하는 경로를 추적합니다.
- source CSS가 parse되더라도 utility output이 없는 structurally valid/visually broken artifact 가능성을 기록합니다.
- 이 SHA에 direct regression test가 없고 후속 broad production visual checks가 간접 보호만 제공한다는 점을 명시합니다.

확인 원칙:

- 먼저 `1de3d36e3a48^`와 `1de3d36e3a48`를 비교하고, 필요한 file은 `1de3d36e3a48:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | `@tailwindcss/postcss` package는 설치돼 있었지만 root PostCSS configuration이 없었습니다. dependency 존재만으로 Next build가 plugin을 호출하지 않으므로 Tailwind import/utility가 production CSS로 확장되지 않을 수 있었습니다. |
| 실제 변경 file/symbol/command/artifact | root `postcss.config.mjs`를 추가해 `plugins: { "@tailwindcss/postcss": {} }`를 export합니다. application component나 stylesheet source는 변경하지 않습니다. |
| Build/runtime/resource owner와 lifetime | CSS transform activation의 owner가 implicit tooling assumption에서 repository root config로 이동합니다. Next development, production build, Lighthouse와 bundle measurement가 같은 config discovery path를 사용합니다. |
| Failure·missing output·cleanup 처리 | config가 없거나 plugin key가 틀리면 source/build가 일부 성공해도 utility layer가 누락된 visually broken artifact가 나올 수 있습니다. 이 commit 자체에는 config contract unit test나 snapshot이 없습니다. |
| 보장하는 것과 보장하지 않는 것 | canonical Next/PostCSS path가 Tailwind plugin을 명시적으로 등록합니다. 모든 utility가 사용 의도대로 생성되는지, visual snapshot이 통과하는지, browser별 rendering이 동일한지는 이 SHA만으로 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | 후속 production visual regression suite와 Lighthouse/CI가 broad artifact 결과를 검사하지만, PostCSS config 하나만을 격리한 direct test는 branch에 없습니다. Thread 4의 CI activation 전에 이 fix가 들어가므로 performance 수치가 styling이 빠진 artifact를 기준으로 확정되는 위험을 줄입니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent root에는 `postcss.config.mjs`가 없습니다.
- **해당 SHA 핵심 코드:** `1de3d36e3a485830b0a459cbc9dc9748ca15d763` · `postcss.config.mjs`

```text
const config = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};

export default config;
```

- **관찰 근거의 성격:** Exact-SHA에서 새로 추가된 root build configuration입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** 후속 production visual regression suite와 Lighthouse/CI가 broad artifact 결과를 검사하지만, PostCSS config 하나만을 격리한 direct test는 branch에 없습니다. Thread 4의 CI activation 전에 이 fix가 들어가므로 performance 수치가 styling이 빠진 artifact를 기준으로 확정되는 위험을 줄입니다.

## 6. Invariant ledger

| Invariant | 이전 상태 | 도입·수정 | 검증·소비 | 남은 비보장 |
| --- | --- | --- | --- | --- |
| Font acquisition | build-time provider fetch | `7872e1214de7`에서 binary/license/provenance를 repository로 이동 | `2f65f6a6fcb6`이 source/path/magic/license를 보호 | hash·glyph·browser rendering |
| Compiler output format | production compiler default | `404a220e5d40`에서 webpack을 package script에 고정 | Thread 4 parser/contract test가 소비 | Next internal format의 future 변화 |
| Framework/native resolution | 16.2.4 patch line | `5d903132306a`에서 16.2.11과 GNU/musl metadata로 정렬 | 후속 npm install/build/container가 같은 lockfile 사용 | CVE-specific behavior·multiarch 실행 |
| CSS transform | plugin 설치만 존재 | `1de3d36e3a48`에서 root PostCSS config가 plugin 호출을 소유 | 후속 broad production visual/performance path가 결과를 소비 | direct config regression test |

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Fix/decision | Test·gate evidence | 한계 |
| --- | --- | --- | --- |
| 외부 font endpoint가 차단된 build | local WOFF2 + `next/font/local` | `local-fonts.test.ts`의 source/file contract | 실제 build/browser는 별도 |
| framework default compiler drift | `next build --webpack` | Thread 4의 exact script/parser fixture test | 직접 `next build` 호출은 우회 가능 |
| OS/libc에 맞지 않는 SWC resolution | matching 16.2.11 lockfile + libc constraints | install/build downstream에서만 드러남 | dedicated native matrix 없음 |
| Tailwind plugin 미호출로 utility CSS 누락 | root PostCSS config | 후속 broad visual regression이 간접 검출 | 이 commit에 direct test 없음 |

## 8. Ownership / state / responsibility 변화

| 대상 | 이전 owner/state | 중간 변화 | 최종 owner/state |
| --- | --- | --- | --- |
| Font bytes/provenance | external provider | `src/app/fonts/**`와 `FONT_SOURCES.md` | layout registration + CSS variables |
| Compiler selection | Next production default | `package.json` canonical build script | performance parser/test가 계약 소비 |
| Native compiler package | 기존 lock resolution | patched package/lock graph와 libc metadata | npm platform selection |
| CSS transform activation | dependency가 암묵적으로 동작한다는 가정 | root `postcss.config.mjs` | 모든 Next build consumer |

## 9. Thread 최종 상태

font binary와 legal/source record는 repository 안에 있고, production build는 webpack과 patched Next/SWC graph를 명시하며, Tailwind transform은 root config로 활성화됩니다. 이는 fresh build portability를 크게 좁히지만 npm registry, base image, OS toolchain과 실제 browser rendering을 완전히 offline/self-contained하게 만들지는 않습니다.

## 10. 최종 product-delivery flow 정리

npm이 pinned framework/SWC graph를 platform 조건에 맞게 설치 → `npm run build`가 webpack을 선택 → Next가 root PostCSS config로 Tailwind를 변환 → `next/font/local`이 repository WOFF2를 build output에 포함 → downstream production/measurement/container 경로가 같은 artifact를 소비합니다.

## 11. 학습 완료 자가 점검

- [x] font binary, source record, license와 layout/CSS consumer ownership을 연결했습니다.
- [x] webpack pin과 manifest parser contract의 cross-thread 관계를 설명했습니다.
- [x] Next patch update의 security subject를 CVE-specific 주장으로 확대하지 않았습니다.
- [x] Tailwind fix의 direct regression test 부재와 broad verification 범위를 구분했습니다.
- [ ] Exact-SHA runtime command를 직접 실행해 결과를 기록했습니다. — 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
===== END FILE: 02-self-contained-production-build-and-portability.md =====

===== BEGIN FILE: 03-standalone-artifact-contract-and-ci-verification.md =====
# Thread: Standalone artifact contract and CI verification

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 원본 Development Thread를 변경하지 않고, 같은 branch history에 product-delivery 관점을 추가한 확장 workbook입니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 `commit/commit-importance.md`의 branch-scoped 분류를 사용합니다.
- Phase 1 audit에서 category/thread grouping과 commit set을 실제 history에 대조한 뒤 이 문서를 freeze했습니다.
- Phase 2는 freeze된 구조와 fixed metadata를 바꾸지 않고 learner-facing 기록만 완성합니다.
- 다른 branch의 구현이나 final HEAD를 과거 SHA 설명에 소급하지 않습니다.
- 실행하지 않은 build/test/CI/Docker 결과는 exact-SHA source inspection과 구분합니다.

## 1. Thread 목표

Next production build를 source tree나 전체 development dependency graph가 아닌 standalone server artifact로 전달할 수 있게 만들고, 그 artifact의 최소 file-layout contract를 local command와 CI가 동일하게 검증하도록 복원합니다.

### 계획된 핵심 invariant

- `next.config.ts`는 standalone output 생성을 명시합니다.
- post-build verification은 `.next/standalone/server.js`와 `.next/static`의 존재를 fail-closed로 요구합니다.
- CI는 production E2E가 만든 동일한 `.next` tree에 local `build:verify` command를 적용합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `output: "standalone"`이 생성하는 artifact와 별도로 복사해야 하는 static/public 자산은 무엇인가?
- file existence 검증은 어떤 missing state를 잡고, 실행 가능성·내용·public asset에 대해서는 무엇을 증명하지 못하는가?
- CI가 build를 두 번 수행하는지, 아니면 production E2E의 output을 재사용하는지 step ordering으로 확인할 수 있는가?
- 이 Thread의 artifact contract가 다음 Docker Thread에 어떻게 handoff되는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 변경 file, function, config, script와 workflow step을 확인했습니다.
- Source, generated artifact, CI gate, container/runtime owner를 구분했습니다.
- Missing artifact, portability failure, threshold violation, startup failure와 cleanup branch를 기록했습니다.
- Test/CI command의 technique, production path, proves/does-not-prove와 실제 실행 여부를 구분했습니다.
- 최종 product-delivery 흐름과 cross-thread handoff를 코드 없이 설명할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 확장 thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `29508f4668ea` | build: standalone server 산출물 생성 | B | DEPLOY | artifact 형식 선택 — Next.js가 traced runtime dependency를 포함한 standalone server bundle을 생성하도록 설정합니다. |
| 2 | `c0f7434467a0` | test(build): standalone 산출물 완전성 검증 | A | VALIDATION, DEPLOY, TEST | artifact shape regression — server entry와 generated static directory의 존재를 explicit command로 검사합니다. |
| 3 | `c5e73853a1b6` | ci: standalone 산출물 검증 추가 | A | VALIDATION, DEPLOY, TEST | CI promotion — production E2E build output에 standalone completeness command를 적용합니다. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA의 tree와 parent diff를 기준으로 작성합니다. 다른 Thread의 later commit은 관계 설명에만 사용하고 과거 구현에 소급하지 않습니다.

### 1. `29508f4668ea` — build: standalone server 산출물 생성

- **Full SHA:** `29508f4668eaed37c393c8c2ef2e80d0e6c8e2f2`
- **Importance:** B
- **Tags:** DEPLOY
- **확장 thread에서의 역할:** artifact 형식 선택 — Next.js가 traced runtime dependency를 포함한 standalone server bundle을 생성하도록 설정합니다.

#### 해당 SHA에서 확인할 실제 코드

- `next.config.ts`의 parent/resulting tree를 비교해 `output: "standalone"`이 유일한 behavior change인지 확인합니다.
- generated `.next/standalone`은 source control에 commit되지 않고 build가 소유하는 ephemeral output이라는 점을 기록합니다.
- standalone output만으로 `.next/static`과 `public`이 자동 포함되는지 후속 commits의 copy/verify logic으로 확인합니다.
- 이 SHA에는 artifact existence test나 runtime launch가 없다는 범위를 명시합니다.

확인 원칙:

- 먼저 `29508f4668ea^`와 `29508f4668ea`를 비교하고, 필요한 file은 `29508f4668ea:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | Next production build는 가능했지만 deployment용 traced standalone server bundle을 생성하라는 repository configuration이 없었습니다. 전달자가 전체 project/node_modules를 어떻게 배치할지 암묵적으로 결정해야 했습니다. |
| 실제 변경 file/symbol/command/artifact | `next.config.ts`의 `nextConfig`에 `output: "standalone"`을 추가했습니다. |
| Build/runtime/resource owner와 lifetime | artifact generation의 owner는 Next build configuration과 `npm run build`입니다. `.next/standalone`은 generated directory이며 repository source가 소유하지 않습니다. |
| Failure·missing output·cleanup 처리 | build가 실패하면 artifact가 생성되지 않지만, 이 SHA에는 missing/partial output을 별도로 검사하는 script가 없습니다. standalone server가 실제로 시작되는지도 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | canonical build가 standalone output mode를 요청합니다. generated file의 존재·완전성, static/public asset 포함, runtime response는 아직 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `c0f7434467a0`이 최소 required artifact를 검사하고, Thread 5의 Dockerfile이 standalone server와 static/public을 명시적으로 분리해 복사합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** `nextConfig`에는 `devIndicators: false`만 있고 `output` 설정이 없습니다.
- **해당 SHA 핵심 코드:** `29508f4668eaed37c393c8c2ef2e80d0e6c8e2f2` · `next.config.ts`

```text
const nextConfig: NextConfig = {
  devIndicators: false,
  output: "standalone",
};
```

- **관찰 근거의 성격:** Exact-SHA config diff에서 직접 확인한 generated artifact mode입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `c0f7434467a0`이 최소 required artifact를 검사하고, Thread 5의 Dockerfile이 standalone server와 static/public을 명시적으로 분리해 복사합니다.

### 2. `c0f7434467a0` — test(build): standalone 산출물 완전성 검증

- **Full SHA:** `c0f7434467a051f93273a7e850d7bf94cc97a215`
- **Importance:** A
- **Tags:** VALIDATION, DEPLOY, TEST
- **확장 thread에서의 역할:** artifact shape regression — server entry와 generated static directory의 존재를 explicit command로 검사합니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `build:verify`와 `scripts/verify-build-output.mjs`의 `requiredArtifacts` 배열을 확인합니다.
- `existsSync(resolve(...))`가 file/directory type이나 내용이 아니라 path existence만 확인한다는 technique를 분류합니다.
- missing path를 모두 수집해 한 error에 출력하는 failure shape와 success log를 기록합니다.
- `public` directory와 standalone server launch가 검사 목록에 없는 이유를 후속 Docker contract와 연결합니다.

확인 원칙:

- 먼저 `c0f7434467a0^`와 `c0f7434467a0`를 비교하고, 필요한 file은 `c0f7434467a0:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | standalone output mode는 설정됐지만 build가 부분적으로 끝나거나 expected layout이 바뀌어도 repository command가 이를 명시적으로 판정하지 않았습니다. |
| 실제 변경 file/symbol/command/artifact | `build:verify` script와 `scripts/verify-build-output.mjs`를 추가했습니다. `.next/standalone/server.js`와 `.next/static`을 `existsSync`로 확인하고, 누락된 모든 path를 나열해 exception을 던집니다. |
| Build/runtime/resource owner와 lifetime | verification script가 required path list와 pass/fail 결정을 소유합니다. existing build output을 read-only로 검사하며 artifact를 생성·수정·정리하지 않습니다. |
| Failure·missing output·cleanup 처리 | 둘 중 하나라도 없으면 `Standalone build output is incomplete` error와 missing list로 process가 실패합니다. path가 존재하기만 하면 통과하므로 file type, server syntax, static content, permissions, public assets와 runtime startup은 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 두 deployment-critical path의 부재는 deterministic post-build failure가 됩니다. artifact가 실제로 독립 실행되거나 올바른 response를 제공한다는 보장은 아닙니다. |
| 다음 delivery commit 또는 관련 test 연결 | `c5e73853a1b6`이 exact `npm run build:verify`를 CI에서 호출합니다. Docker builder도 이후 `npm run build && npm run build:verify`를 image 생성 전 조건으로 재사용합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 `build:verify` script와 `scripts/verify-build-output.mjs`가 없습니다.
- **해당 SHA 핵심 코드:** `c0f7434467a051f93273a7e850d7bf94cc97a215` · `scripts/verify-build-output.mjs`

```text
if (missing.length > 0) {
  throw new Error(
    `Standalone build output is incomplete:\n${missing
      .map((artifact) => `- ${artifact}`)
      .join("\n")}`,
  );
}

console.log(`verified ${requiredArtifacts.length} portfolio build artifacts`);
```

- **관찰 근거의 성격:** Exact-SHA script implementation에서 직접 확인한 path-existence test입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `c5e73853a1b6`이 exact `npm run build:verify`를 CI에서 호출합니다. Docker builder도 이후 `npm run build && npm run build:verify`를 image 생성 전 조건으로 재사용합니다.

### 3. `c5e73853a1b6` — ci: standalone 산출물 검증 추가

- **Full SHA:** `c5e73853a1b69e39561748435f4768109a368544`
- **Importance:** A
- **Tags:** VALIDATION, DEPLOY, TEST
- **확장 thread에서의 역할:** CI promotion — production E2E build output에 standalone completeness command를 적용합니다.

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml`에서 새 step이 `Build and run production E2E tests` 뒤에 위치하는지 확인합니다.
- 새 step이 rebuild하지 않고 앞 step이 남긴 `.next`를 `npm run build:verify`로 검사한다는 artifact handoff를 기록합니다.
- 앞 E2E 실패 시 verify step에 도달하지 않는 fail-fast ordering과, E2E 성공 뒤 artifact shape가 별도 실패할 수 있는 이유를 설명합니다.
- CI가 이 시점에 `public` copy나 standalone `server.js` 직접 실행을 아직 하지 않는다는 범위를 명시합니다.

확인 원칙:

- 먼저 `c5e73853a1b6^`와 `c5e73853a1b6`를 비교하고, 필요한 file은 `c5e73853a1b6:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | `build:verify`는 local opt-in command였고 CI production E2E가 성공하더라도 standalone path contract는 integration 조건이 아니었습니다. |
| 실제 변경 file/symbol/command/artifact | CI workflow에 `Verify standalone output` step을 추가해 production E2E 직후 `npm run build:verify`를 실행합니다. |
| Build/runtime/resource owner와 lifetime | 앞 production E2E step이 `.next`를 생성하고, 다음 verify step이 같은 runner workspace의 artifact를 소비합니다. workflow ordering이 producer/consumer lifetime을 소유하며 별도 persistence는 없습니다. |
| Failure·missing output·cleanup 처리 | E2E build/test가 실패하면 step에 도달하지 않습니다. E2E가 통과해도 required path가 없으면 verify command가 non-zero로 CI를 실패시킵니다. runner 종료 시 generated artifact는 폐기됩니다. |
| 보장하는 것과 보장하지 않는 것 | push/PR의 production path는 browser behavior와 standalone 최소 layout을 모두 요구합니다. standalone entry를 직접 실행하거나 public assets, image packaging, non-root user를 검증하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | Thread 5가 이 artifact contract를 Docker builder prerequisite로 사용하고, runtime script가 실제 HTTP response와 public assets를 검증합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent workflow는 production E2E step으로 끝나며 standalone verify step이 없습니다.
- **해당 SHA 핵심 코드:** `c5e73853a1b69e39561748435f4768109a368544` · `.github/workflows/ci.yml`

```text
- name: Build and run production E2E tests
  run: npm run test:e2e:production

- name: Verify standalone output
  run: npm run build:verify
```

- **관찰 근거의 성격:** Exact-SHA workflow diff에서 직접 확인한 artifact producer/consumer order입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** Thread 5가 이 artifact contract를 Docker builder prerequisite로 사용하고, runtime script가 실제 HTTP response와 public assets를 검증합니다.

## 6. Invariant ledger

| Invariant | 이전 상태 | 도입·수정 | 검증·소비 | 남은 비보장 |
| --- | --- | --- | --- | --- |
| Artifact generation | 일반 Next build output | `29508f4668ea`에서 standalone mode 선택 | production build가 `.next/standalone` 생성 | 생성 성공 여부는 후속 검사 |
| Minimum layout | implicit framework assumption | `c0f7434467a0`에서 server.js + static path를 explicit list로 정의 | `c5e73853a1b6`이 CI에서 소비 | public·content·runtime response |
| CI artifact lifetime | 검증 command local-only | production E2E step이 `.next` producer | 다음 step이 같은 workspace output을 검사 | runner 밖 artifact publish |

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Fix/decision | Test·gate evidence | 한계 |
| --- | --- | --- | --- |
| standalone mode 미설정 | `output: "standalone"` | post-build required path check | mode 설정만으로 runtime 성공은 아님 |
| partial/missing build layout | missing path aggregation + throw | CI의 `npm run build:verify` | path type/content 미검사 |
| browser test는 성공하지만 deployable layout 누락 | E2E 뒤 별도 artifact step | 동일 `.next` tree를 순차 검사 | public/container는 다음 Thread |

## 8. Ownership / state / responsibility 변화

| 대상 | 이전 owner/state | 중간 변화 | 최종 owner/state |
| --- | --- | --- | --- |
| Output format | framework default | `next.config.ts` | `npm run build` |
| Artifact completeness policy | 암묵적 | `requiredArtifacts` array | local/CI/Docker builder가 재사용 |
| Generated tree lifetime | local build directory | CI E2E step producer | verify step consumer 후 runner 폐기 |

## 9. Thread 최종 상태

production build는 standalone server mode를 요청하고, repository command와 CI는 `server.js` 및 generated static directory의 존재를 요구합니다. 이 계약은 artifact shape만 다루며 public asset copy, 직접 server startup, runtime user와 HTTP response는 다음 container Thread가 맡습니다.

## 10. 최종 product-delivery flow 정리

`next.config.ts`가 standalone mode 선택 → production E2E의 `npm run build`가 `.next` 생성 → existing browser suite가 `next start` 검증 → 같은 workspace에서 `build:verify`가 server entry/static path 확인 → CI exit status가 artifact completeness를 판정합니다.

## 11. 학습 완료 자가 점검

- [x] standalone mode 설정과 generated output ownership을 구분했습니다.
- [x] path-existence test가 증명하는 것과 증명하지 않는 것을 기록했습니다.
- [x] CI가 rebuild하지 않고 앞 step의 `.next`를 재사용함을 확인했습니다.
- [x] public assets와 actual standalone runtime이 다음 Thread 책임임을 연결했습니다.
- [ ] Exact-SHA runtime command를 직접 실행해 결과를 기록했습니다. — 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
===== END FILE: 03-standalone-artifact-contract-and-ci-verification.md =====

===== BEGIN FILE: 04-release-performance-gates.md =====
# Thread: Release performance gates

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 원본 Development Thread를 변경하지 않고, 같은 branch history에 product-delivery 관점을 추가한 확장 workbook입니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 `commit/commit-importance.md`의 branch-scoped 분류를 사용합니다.
- Phase 1 audit에서 category/thread grouping과 commit set을 실제 history에 대조한 뒤 이 문서를 freeze했습니다.
- Phase 2는 freeze된 구조와 fixed metadata를 바꾸지 않고 learner-facing 기록만 완성합니다.
- 다른 branch의 구현이나 final HEAD를 과거 SHA 설명에 소급하지 않습니다.
- 실행하지 않은 build/test/CI/Docker 결과는 exact-SHA source inspection과 구분합니다.

## 1. Thread 목표

webpack production output에서 route별 client JS/CSS를 계측하고 reviewable baseline에 대해 fail-closed budget을 적용한 뒤, production server의 desktop Lighthouse matrix와 함께 CI release gate로 승격하는 과정을 복원합니다.

### 계획된 핵심 invariant

- route asset measurement는 source 추정치가 아니라 webpack production manifests와 실제 asset file size를 사용합니다.
- routine `bundle:check`는 committed baseline을 읽기만 하며, baseline 갱신은 별도 explicit command입니다.
- baseline route 누락, 새 route의 baseline 부재, route별 JS/CSS 5% 초과는 모두 violation입니다.
- Lighthouse는 production server에서 5 designs × home/project-detail × 3 desktop runs를 median으로 평가합니다.
- CI는 production build output을 재사용해 standalone, bundle budget과 desktop Lighthouse threshold를 차례로 요구합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- generated client-reference manifest를 JavaScript로 실행하지 않고 JSON payload만 추출하는 parser boundary는 무엇인가?
- shared JS, route JS, non-inlined CSS와 duplicate asset을 어떤 규칙으로 byte accounting하는가?
- baseline creation과 routine check를 분리하지 않으면 어떤 self-approval failure가 생기는가?
- Lighthouse gate의 URL matrix, aggregation, threshold와 browser executable ownership은 어디에 고정되는가?
- committed lab result는 실제 CI gate인가, observation artifact인가, 또는 둘의 조합인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 변경 file, function, config, script와 workflow step을 확인했습니다.
- Source, generated artifact, CI gate, container/runtime owner를 구분했습니다.
- Missing artifact, portability failure, threshold violation, startup failure와 cleanup branch를 기록했습니다.
- Test/CI command의 technique, production path, proves/does-not-prove와 실제 실행 여부를 구분했습니다.
- 최종 product-delivery 흐름과 cross-thread handoff를 코드 없이 설명할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 확장 thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `c2fb8a7c238d` | fix(perf): webpack route manifest parser 보강 | A | ARCH, ROUTING, PERF | generated-output trust boundary — webpack client-reference wrapper에서 JSON payload만 안전하게 추출합니다. |
| 2 | `c24c350ce42c` | test(build): compiler와 manifest parser 계약 검증 | A | VALIDATION, DEPLOY, TEST | deterministic contract test — webpack build script와 generated wrapper parser fixture를 함께 고정합니다. |
| 3 | `605b64512edf` | build(perf): route별 client asset 측정 추가 | A | ARCH, ROUTING, PERF | production measurement — app paths와 client-reference manifests를 join해 route별 uncompressed JS/CSS bytes를 계산합니다. |
| 4 | `518ff5b51ec5` | build(perf): route bundle 성장 예산 평가 추가 | A | ARCH, ROUTING, PERF | pure policy evaluator — committed route coverage와 JS/CSS별 최대 5% 성장을 structured violations로 판정합니다. |
| 5 | `57a1b0876941` | build(perf): bundle budget CLI 연결 | B | PERF, DEPLOY | operational split — baseline write와 routine check를 별도 package commands로 노출합니다. |
| 6 | `6ac1ea4b5055` | chore(perf): route bundle 기준값 기록 | B | ROUTING, PERF | reviewable operational state — 여덟 public route pattern의 accepted uncompressed byte baseline을 commit합니다. |
| 7 | `1529ccf225c1` | build(perf): desktop Lighthouse 실행 경계 추가 | A | PERF, DEPLOY | production lab gate definition — 10 URL desktop matrix, median aggregation과 release thresholds를 설정합니다. |
| 8 | `f1c72dfdd16a` | build(perf): Lighthouse 결과 요약기 추가 | B | PERF, DEPLOY | measurement provenance — raw LHRs를 URL별 runs/median과 execution environment를 가진 JSON으로 요약합니다. |
| 9 | `4e8f95249481` | test(perf): 배포 성능 gate 규칙 검증 | A | VALIDATION, PERF, TEST | policy regression suite — compiler/parser assertions를 보존하며 Lighthouse matrix와 budget arithmetic을 검증합니다. |
| 10 | `abbd530368a0` | ci: 검증된 bundle과 Lighthouse gate 활성화 | A | VALIDATION, PERF, DEPLOY | release integration — production build output에 standalone, bundle와 Lighthouse checks를 순차 적용합니다. |
| 11 | `a39856cf734a` | chore(perf): 최종 lab 성능 측정 결과 기록 | C | PERF | generated evidence snapshot — desktop enforced matrix와 별도 mobile observation의 측정 결과를 환경과 함께 보존합니다. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA의 tree와 parent diff를 기준으로 작성합니다. 다른 Thread의 later commit은 관계 설명에만 사용하고 과거 구현에 소급하지 않습니다.

### 1. `c2fb8a7c238d` — fix(perf): webpack route manifest parser 보강

- **Full SHA:** `c2fb8a7c238d355c717a14359ef805fb9cc7f6f7`
- **Importance:** A
- **Tags:** ARCH, ROUTING, PERF
- **확장 thread에서의 역할:** generated-output trust boundary — webpack client-reference wrapper에서 JSON payload만 안전하게 추출합니다.

#### 해당 SHA에서 확인할 실제 코드

- parent에는 `scripts/route-budgets.mjs`가 없다는 사실을 확인해, subject의 `fix`를 이전 branch implementation 수정으로 오해하지 않습니다.
- `parseClientReferenceManifest`의 assignment regex, source slicing, semicolon requirement와 `JSON.parse` 호출 순서를 추적합니다.
- ordinary route와 `[...]` dynamic key가 regex에서 어떻게 처리되는지 후속 test fixture와 연결합니다.
- missing assignment/terminator와 malformed JSON이 filename을 포함한 error 또는 JSON parse failure로 닫히는지 확인합니다.

확인 원칙:

- 먼저 `c2fb8a7c238d^`와 `c2fb8a7c238d`를 비교하고, 필요한 file은 `c2fb8a7c238d:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | parent에는 route budget script나 manifest parser가 없습니다. commit body가 지적하는 위험은 generated JavaScript wrapper를 plain JSON으로 취급하거나 평가하는 접근이며, branch에서 그런 이전 implementation이 실제 commit돼 있었다고 볼 근거는 없습니다. |
| 실제 변경 file/symbol/command/artifact | `scripts/route-budgets.mjs`와 declaration file을 추가했습니다. parser는 `globalThis.__RSC_MANIFEST[...] =` assignment를 line break를 포함해 찾고, 뒤 serialized value를 slice한 뒤 terminating semicolon을 요구해 제거하고 `JSON.parse`합니다. |
| Build/runtime/resource owner와 lifetime | generated source 해석의 owner가 ad hoc caller가 아니라 pure parser function으로 모입니다. source string/filename은 caller가 소유하고 parser는 새 object를 반환하며 global state를 실행·변경하지 않습니다. |
| Failure·missing output·cleanup 처리 | assignment가 없거나 semicolon로 끝나지 않으면 filename을 포함한 custom error가 발생합니다. payload가 JSON이 아니면 `JSON.parse` error가 전파됩니다. regex가 future Next output wrapper 형식을 이해하지 못해도 measurement를 계속하지 않고 실패합니다. |
| 보장하는 것과 보장하지 않는 것 | expected webpack wrapper에서는 value만 JSON으로 해석하며 `eval`하지 않습니다. 모든 future manifest syntax, semantic schema validation, asset existence와 route accounting은 아직 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `c24c350ce42c`이 compact/dynamic fixtures를 고정하고, `605b64512edf`가 actual build manifest collector에서 이 parser를 호출합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 두 `route-budgets` files와 parser가 없습니다.
- **해당 SHA 핵심 코드:** `c2fb8a7c238d355c717a14359ef805fb9cc7f6f7` · `scripts/route-budgets.mjs — parseClientReferenceManifest`

```text
const assignment =
  /globalThis\.__RSC_MANIFEST\[[\s\S]+?\]\s*=\s*/.exec(source);
const serialized = assignment
  ? source.slice(assignment.index + assignment[0].length).trim()
  : "";

if (!assignment || !serialized.endsWith(";")) {
  throw new Error(`Cannot parse client reference manifest: ${filename}`);
}

return JSON.parse(serialized.slice(0, -1));
```

- **관찰 근거의 성격:** Exact-SHA new parser implementation과 parent absence를 직접 확인했습니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `c24c350ce42c`이 compact/dynamic fixtures를 고정하고, `605b64512edf`가 actual build manifest collector에서 이 parser를 호출합니다.

### 2. `c24c350ce42c` — test(build): compiler와 manifest parser 계약 검증

- **Full SHA:** `c24c350ce42cdbfe35eff3c6dd3bebc62ab132aa`
- **Importance:** A
- **Tags:** VALIDATION, DEPLOY, TEST
- **확장 thread에서의 역할:** deterministic contract test — webpack build script와 generated wrapper parser fixture를 함께 고정합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/performance/build-manifest-contract.test.ts`가 `package.json`을 실제로 require해 exact build string을 검사하는지 확인합니다.
- compact ordinary route fixture와 square bracket가 중첩된 dynamic route fixture를 parser에 직접 전달하는 test technique를 기록합니다.
- fixture가 actual `.next` file을 읽는 integration test가 아니라 pure parser/config contract라는 점을 구분합니다.
- parser가 malformed input, semicolon absence 또는 actual asset accounting을 이 SHA에서 test하는지 확인합니다.

확인 원칙:

- 먼저 `c24c350ce42c^`와 `c24c350ce42c`를 비교하고, 필요한 file은 `c24c350ce42c:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | webpack build command와 parser는 존재했지만 compiler flag 제거 또는 dynamic route key handling regression을 결정적으로 잡는 test가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `src/performance/build-manifest-contract.test.ts`를 추가해 `packageJson.scripts.build === "next build --webpack"`을 요구합니다. ordinary `/about/page`와 `/projects/[projectId]/page` wrapper string을 parser에 넣고 expected object를 비교합니다. |
| Build/runtime/resource owner와 lifetime | Vitest가 package config와 pure parser contract를 소유합니다. fixture string은 test가 소유하며 filesystem-generated manifests, build process와 assets는 사용하지 않습니다. |
| Failure·missing output·cleanup 처리 | compiler string drift, assignment regex가 compact wrapper 또는 dynamic brackets를 처리하지 못하면 assertion이 실패합니다. 실제 Next build format이 fixture와 함께 잘못 업데이트되는 경우, manifest file read/stat failure는 검출하지 못합니다. |
| 보장하는 것과 보장하지 않는 것 | repository build command와 두 대표 wrapper syntax의 parser behavior가 결정적으로 보호됩니다. production build 성공, 모든 route key, malformed input, bundle size accuracy는 증명하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `605b64512edf`의 collector가 parser를 실제 manifest path에 적용합니다. `4e8f95249481`은 이 test를 broader performance gate test로 이동하면서 같은 assertions를 보존합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 `src/performance/build-manifest-contract.test.ts`가 없습니다.
- **해당 SHA 핵심 코드:** `c24c350ce42cdbfe35eff3c6dd3bebc62ab132aa` · `src/performance/build-manifest-contract.test.ts`

```text
const source =
  'globalThis.__RSC_MANIFEST=(globalThis.__RSC_MANIFEST||{});globalThis.__RSC_MANIFEST["/projects/[projectId]/page"]={"entryJSFiles":{}};';

expect(parseClientReferenceManifest(source, "project.js")).toEqual({
  entryJSFiles: {},
});
```

- **관찰 근거의 성격:** Exact-SHA test file에서 직접 확인한 config/pure parser regression입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `605b64512edf`의 collector가 parser를 실제 manifest path에 적용합니다. `4e8f95249481`은 이 test를 broader performance gate test로 이동하면서 같은 assertions를 보존합니다.

### 3. `605b64512edf` — build(perf): route별 client asset 측정 추가

- **Full SHA:** `605b64512edfdf416335ee500355fb1008100168`
- **Importance:** A
- **Tags:** ARCH, ROUTING, PERF
- **확장 thread에서의 역할:** production measurement — app paths와 client-reference manifests를 join해 route별 uncompressed JS/CSS bytes를 계산합니다.

#### 해당 SHA에서 확인할 실제 코드

- `collectRouteBundleMeasurements`가 `server/app-paths-manifest.json`과 `build-manifest.json`을 어떤 순서로 읽는지 확인합니다.
- `/page` → `/`, `/.../page` suffix 제거, `/_` skip과 sorted key iteration 규칙을 기록합니다.
- root shared JS와 route `entryJSFiles`, non-inlined `entryCSSFiles`를 합치고 `Set`으로 route 내부 duplicate를 제거하는 byte accounting을 추적합니다.
- missing manifest/asset, invalid JSON와 parser failure가 catch되지 않고 caller를 실패시키는 fail-closed behavior를 확인합니다.

확인 원칙:

- 먼저 `605b64512edf^`와 `605b64512edf`를 비교하고, 필요한 file은 `605b64512edf:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | parser만 존재했고 실제 production routes, shared chunks와 emitted file sizes를 수집하는 measurement path가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `collectRouteBundleMeasurements`와 related types를 추가했습니다. app-paths manifest의 page entries를 public route pattern으로 바꾸고, 각 route의 client-reference manifest를 parse합니다. root shared JS + route JS와 non-inlined route CSS를 deduplicate한 뒤 `.next` 아래 실제 file의 `stat.size`를 합산합니다. |
| Build/runtime/resource owner와 lifetime | Next generated manifests가 route→artifact mapping의 source of truth이고 collector가 join/accounting을 소유합니다. `assetBytes`는 caller의 build directory를 read-only로 stat하며 output object가 route별 byte state를 소유합니다. |
| Failure·missing output·cleanup 처리 | manifest read/JSON parse/parser/stat 중 하나라도 실패하면 Promise가 reject합니다. framework-internal `/_`와 non-page entry는 의도적으로 제외됩니다. missing files를 0으로 처리하지 않아 false-green measurement를 만들지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 측정 대상 route별로 shared + entry JS와 별도 transfer되는 CSS의 uncompressed bytes를 계산합니다. gzip/Brotli, runtime cache, lazy chunk의 모든 future fetch, concrete dynamic IDs, network transfer overhead는 측정하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `518ff5b51ec5`가 이 measurement와 baseline을 비교하고, `57a1b0876941`이 production build directory를 대상으로 CLI에서 호출합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parser만 있고 filesystem/manifests를 연결하는 collector와 byte types가 없습니다.
- **해당 SHA 핵심 코드:** `605b64512edfdf416335ee500355fb1008100168` · `scripts/route-budgets.mjs — collectRouteBundleMeasurements`

```text
const sharedJavaScript = buildManifest.rootMainFiles ?? [];

const routeJavaScript = Object.values(
  manifest.entryJSFiles ?? {},
).flat();
const routeCss = Object.values(manifest.entryCSSFiles ?? {})
  .flat()
  .filter(({ inlined }) => !inlined)
  .map(({ path: assetPath }) => assetPath);

measurements[route] = {
  cssBytes: await assetBytes(buildDirectory, routeCss),
  jsBytes: await assetBytes(buildDirectory, [
    ...sharedJavaScript,
    ...routeJavaScript,
  ]),
};
```

- **관찰 근거의 성격:** Exact-SHA collector implementation에서 직접 확인한 build-output accounting입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `518ff5b51ec5`가 이 measurement와 baseline을 비교하고, `57a1b0876941`이 production build directory를 대상으로 CLI에서 호출합니다.

### 4. `518ff5b51ec5` — build(perf): route bundle 성장 예산 평가 추가

- **Full SHA:** `518ff5b51ec54c4eb4fcca9bab5ff6ba8e70a67b`
- **Importance:** A
- **Tags:** ARCH, ROUTING, PERF
- **확장 thread에서의 역할:** pure policy evaluator — committed route coverage와 JS/CSS별 최대 5% 성장을 structured violations로 판정합니다.

#### 해당 SHA에서 확인할 실제 코드

- `BUDGET_GROWTH_FACTOR = 1.05`와 `Math.floor(expected * factor)`의 exact pass boundary를 계산합니다.
- baseline route missing, asset overage와 measured new route without baseline이 서로 다른 `asset` category로 기록되는지 확인합니다.
- evaluator가 process exit, logging, file read/write를 하지 않는 pure function인지 확인합니다.
- type에 `schemaVersion`/`growthLimitPercent`가 있어도 evaluator 자체가 runtime value를 검증하는지 구분합니다.

확인 원칙:

- 먼저 `518ff5b51ec5^`와 `518ff5b51ec5`를 비교하고, 필요한 file은 `518ff5b51ec5:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | route별 bytes는 계산할 수 있었지만 accepted reference, 허용 성장률, route coverage mismatch를 판정하는 policy가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `BUDGET_GROWTH_FACTOR = 1.05`, baseline/violation types와 `evaluateRouteBudgets`를 추가했습니다. 각 baseline route의 CSS/JS allowed bytes를 floor해 초과를 기록하고, expected route missing과 baseline 없는 measured route도 violation으로 반환합니다. |
| Build/runtime/resource owner와 lifetime | pure evaluator가 policy calculation을 소유하고 caller가 measurement/baseline input과 violation 처리 lifecycle을 소유합니다. function은 input을 수정하거나 baseline을 update하지 않습니다. |
| Failure·missing output·cleanup 처리 | function 자체는 throw/exit하지 않고 violations를 반환합니다. missing expected route는 `route`, new route는 `baseline`, overage는 `css`/`js`로 구분합니다. schema/growth field validity는 이 function에서 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | exactly floor(105%)까지 허용하고 그 다음 byte부터 실패 대상으로 표시하며 route set drift도 false-green으로 두지 않습니다. baseline 값 자체의 적절성, absolute size quality, compression과 schema validation은 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `57a1b0876941` CLI가 violations를 stderr/exit code로 승격하고, `4e8f95249481`이 exact 5%와 first-byte-over tests를 추가합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Collector output을 비교하는 evaluator와 budget policy가 없습니다.
- **해당 SHA 핵심 코드:** `518ff5b51ec54c4eb4fcca9bab5ff6ba8e70a67b` · `scripts/route-budgets.mjs — evaluateRouteBudgets`

```text
const allowedBytes = Math.floor(
  expected[property] * BUDGET_GROWTH_FACTOR,
);
if (actual[property] > allowedBytes) {
  violations.push({
    actualBytes: actual[property],
    allowedBytes,
    asset,
    baselineBytes: expected[property],
    route,
  });
}
```

- **관찰 근거의 성격:** Exact-SHA pure evaluator에서 직접 확인한 arithmetic/fail categories입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `57a1b0876941` CLI가 violations를 stderr/exit code로 승격하고, `4e8f95249481`이 exact 5%와 first-byte-over tests를 추가합니다.

### 5. `57a1b0876941` — build(perf): bundle budget CLI 연결

- **Full SHA:** `57a1b0876941d2dc1e78f4439ef9dd4f4b9edb2a`
- **Importance:** B
- **Tags:** PERF, DEPLOY
- **확장 thread에서의 역할:** operational split — baseline write와 routine check를 별도 package commands로 노출합니다.

#### 해당 SHA에서 확인할 실제 코드

- `bundle:baseline`과 `bundle:check`의 argv 차이와 둘 다 existing `.next` measurement를 먼저 수행한다는 순서를 확인합니다.
- `--write-baseline` branch가 directory 생성, formatted JSON write 후 return하고 comparison을 수행하지 않는다는 점을 기록합니다.
- normal branch가 committed file을 읽고 `growthLimitPercent === 5`를 검사한 뒤 모든 violation을 출력하고 `process.exitCode = 1`로 끝나는지 확인합니다.
- `isMain` guard가 test import 시 filesystem/process side effect를 막는지 확인합니다.

확인 원칙:

- 먼저 `57a1b0876941^`와 `57a1b0876941`를 비교하고, 필요한 file은 `57a1b0876941:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | collector/evaluator는 import 가능한 functions였지만 operator가 baseline을 만들거나 routine validation을 실행할 canonical package command와 exit behavior가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `bundle:baseline`과 `bundle:check`를 추가했습니다. CLI는 measurement를 출력하고, explicit write mode에서는 schema/source/routes를 `performance/route-budgets.json`에 씁니다. normal mode에서는 committed baseline과 policy를 읽어 violations를 모두 출력하고 exit code 1을 설정합니다. |
| Build/runtime/resource owner와 lifetime | operator가 어느 command를 선택하는지 소유하고, CLI가 filesystem path, serialization, diagnostics와 process exit state를 소유합니다. `isMain` guard 덕분에 imported functions는 CLI lifecycle을 시작하지 않습니다. |
| Failure·missing output·cleanup 처리 | missing/invalid baseline JSON, growth limit가 5가 아님, measurement failure는 throw/reject합니다. violations는 전부 출력된 뒤 exit code 1입니다. write mode는 current numbers를 승인 여부 없이 overwrite할 수 있으므로 review process가 policy owner입니다. |
| 보장하는 것과 보장하지 않는 것 | routine check는 baseline을 자동 갱신하지 않고 비교만 합니다. baseline 변경은 명시적 command와 diff를 남겨야 합니다. 누가 baseline update를 승인하는지, build 선행 여부, schemaVersion runtime validation은 강제하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `6ac1ea4b5055`가 최초 generated baseline을 commit하고 `abbd530368a0`이 only `bundle:check`를 CI에 연결합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** package scripts와 CLI `main`/baseline path가 없습니다.
- **해당 SHA 핵심 코드:** `57a1b0876941d2dc1e78f4439ef9dd4f4b9edb2a` · `scripts/route-budgets.mjs — main`

```text
if (writeBaseline) {
  const baseline = {
    schemaVersion: 1,
    growthLimitPercent: 5,
    source: "Next.js production client assets (uncompressed bytes)",
    routes: measurements,
  };
  await mkdir(path.dirname(DEFAULT_BASELINE_PATH), { recursive: true });
  await writeFile(
    DEFAULT_BASELINE_PATH,
    `${JSON.stringify(baseline, null, 2)}\n`,
    "utf8",
  );
  console.log(`Wrote ${DEFAULT_BASELINE_PATH}`);
  return;
}
```

- **관찰 근거의 성격:** Exact-SHA package script/CLI implementation에서 직접 확인했습니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `6ac1ea4b5055`가 최초 generated baseline을 commit하고 `abbd530368a0`이 only `bundle:check`를 CI에 연결합니다.

### 6. `6ac1ea4b5055` — chore(perf): route bundle 기준값 기록

- **Full SHA:** `6ac1ea4b5055cad7e1eef05d8a8e7811b5cb8807`
- **Importance:** B
- **Tags:** ROUTING, PERF
- **확장 thread에서의 역할:** reviewable operational state — 여덟 public route pattern의 accepted uncompressed byte baseline을 commit합니다.

#### 해당 SHA에서 확인할 실제 코드

- `performance/route-budgets.json`의 schemaVersion, growthLimitPercent, source description과 route key set을 확인합니다.
- 여덟 route가 모두 같은 `cssBytes: 169861`, `jsBytes: 425976`을 기록한다는 generated state를 그대로 기록합니다.
- 이 file이 test result 증명서가 아니라 future checks의 accepted configuration이라는 ownership을 설명합니다.
- baseline 값을 manual edit/write command로 낮추거나 높일 수 있고 review가 필요하다는 non-guarantee를 명시합니다.

확인 원칙:

- 먼저 `6ac1ea4b5055^`와 `6ac1ea4b5055`를 비교하고, 필요한 file은 `6ac1ea4b5055:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | CLI는 baseline path를 요구하지만 committed `performance/route-budgets.json`이 없어 normal check가 성공할 reference state가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | schema version 1, growth limit 5%, uncompressed client assets라는 source 설명과 8 route patterns를 가진 baseline을 추가했습니다. 각 route는 CSS 169,861 bytes, JS 425,976 bytes를 기록합니다. |
| Build/runtime/resource owner와 lifetime | committed JSON이 accepted reference와 route coverage를 소유합니다. generator가 만들었더라도 merge review 이후 operational configuration으로 기능합니다. |
| Failure·missing output·cleanup 처리 | file 누락/invalid JSON/policy drift는 CLI failure가 됩니다. 값이 부적절하게 높게 승인되면 evaluator는 그 기준을 신뢰하므로 absolute bloat를 막지 못합니다. |
| 보장하는 것과 보장하지 않는 것 | future build가 비교할 explicit route set와 byte reference가 생깁니다. 이 숫자가 빠르거나 작은 artifact임을 독립적으로 증명하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `abbd530368a0`의 CI `bundle:check`가 이 file을 읽습니다. baseline update는 routine check가 아니라 별도 reviewable commit이어야 합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** `performance/route-budgets.json`이 없습니다.
- **해당 SHA 핵심 코드:** `6ac1ea4b5055cad7e1eef05d8a8e7811b5cb8807` · `performance/route-budgets.json`

```text
{
  "schemaVersion": 1,
  "growthLimitPercent": 5,
  "source": "Next.js production client assets (uncompressed bytes)",
  "routes": {
    "/": { "cssBytes": 169861, "jsBytes": 425976 },
    "/projects/[projectId]": {
      "cssBytes": 169861,
      "jsBytes": 425976
    }
  }
}
```

- **관찰 근거의 성격:** Exact-SHA committed generated measurement입니다. 이 작업에서 재측정하지 않았습니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `abbd530368a0`의 CI `bundle:check`가 이 file을 읽습니다. baseline update는 routine check가 아니라 별도 reviewable commit이어야 합니다.

### 7. `1529ccf225c1` — build(perf): desktop Lighthouse 실행 경계 추가

- **Full SHA:** `1529ccf225c10028607b6a8963c3280f4ab56a42`
- **Importance:** A
- **Tags:** PERF, DEPLOY
- **확장 thread에서의 역할:** production lab gate definition — 10 URL desktop matrix, median aggregation과 release thresholds를 설정합니다.

#### 해당 SHA에서 확인할 실제 코드

- `lighthouserc.cjs`가 `src/content/projects.json`에서 첫 enabled project를 찾고 없으면 config load 단계에서 실패하는지 확인합니다.
- 5 design IDs × home/project detail로 10 URLs를 구성하고 `start:performance` port 3300의 production server를 사용하는지 추적합니다.
- 3 runs, desktop preset, performance/accessibility only, headless/no-sandbox와 median assertion 설정을 기록합니다.
- performance 0.90, accessibility 0.95, LCP 2500ms, TBT 200ms, CLS 0.1의 exact threshold와 mobile/all-route non-coverage를 명시합니다.

확인 원칙:

- 먼저 `1529ccf225c1^`와 `1529ccf225c1`를 비교하고, 필요한 file은 `1529ccf225c1:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | route byte budget은 정의됐지만 rendered production page의 lab performance/accessibility와 visual stability를 release threshold로 평가하는 command/config가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `@lhci/cli`, `lighthouse:audit`, `start:performance`, `.lighthouseci` ignore와 `lighthouserc.cjs`를 추가했습니다. 첫 enabled project와 5 designs로 10 URLs를 만들고 production server에서 각 3회 desktop audit를 수행합니다. |
| Build/runtime/resource owner와 lifetime | content JSON이 project ID availability를, Lighthouse config가 URL matrix/metrics/threshold를, LHCI가 server lifecycle과 Chrome audits를 소유합니다. raw reports는 ignored `.lighthouseci`에 ephemeral하게 남습니다. |
| Failure·missing output·cleanup 처리 | enabled project가 없으면 config load가 throw합니다. server가 120초 안에 readiness pattern을 내지 않거나 median assertion이 threshold를 벗어나면 LHCI command가 실패합니다. browser sandbox는 CI compatibility를 위해 비활성화됩니다. |
| 보장하는 것과 보장하지 않는 것 | production server의 5-design home/detail 대표 표본에 desktop median gate가 생깁니다. mobile, 다른 routes, field/RUM, network diversity, long interaction INP를 직접 보장하지 않으며 TBT가 interaction proxy입니다. |
| 다음 delivery commit 또는 관련 test 연결 | `f1c72dfdd16a`가 raw reports를 reviewable summary로 만들고, `4e8f95249481`이 config rules를 unit-test하며, `abbd530368a0`이 CI에서 실행합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** LHCI dependency, package commands, config와 ignored workspace가 없습니다.
- **해당 SHA 핵심 코드:** `1529ccf225c10028607b6a8963c3280f4ab56a42` · `lighthouserc.cjs`

```text
const urls = designIds.flatMap((designId) => [
  `${baseUrl}/?view=${designId}`,
  `${baseUrl}/projects/${firstProject.id}?view=${designId}`,
]);

collect: {
  numberOfRuns: 3,
  startServerCommand: "npm run start:performance",
  settings: { preset: "desktop" },
},
assert: {
  assertions: {
    "categories:performance": ["error", { ...median, minScore: 0.9 }],
    "largest-contentful-paint": ["error", { ...median, maxNumericValue: 2_500 }],
  },
}
```

- **관찰 근거의 성격:** Exact-SHA LHCI config/package diff에서 직접 확인한 release gate definition입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `f1c72dfdd16a`가 raw reports를 reviewable summary로 만들고, `4e8f95249481`이 config rules를 unit-test하며, `abbd530368a0`이 CI에서 실행합니다.

### 8. `f1c72dfdd16a` — build(perf): Lighthouse 결과 요약기 추가

- **Full SHA:** `f1c72dfdd16a0be14f7c2a650bde9ee431ec27bf`
- **Importance:** B
- **Tags:** PERF, DEPLOY
- **확장 thread에서의 역할:** measurement provenance — raw LHRs를 URL별 runs/median과 execution environment를 가진 JSON으로 요약합니다.

#### 해당 SHA에서 확인할 실제 코드

- `scripts/summarize-lighthouse.mjs`가 `lhr-*.json`만 읽고 report가 0개면 실패하는지 확인합니다.
- `resultFromReport`, `median`, `aggregate`가 다섯 metrics를 URL별로 계산하고 URL key를 정렬하는지 추적합니다.
- `measuredAt`과 host environment가 output마다 달라질 수 있어 deterministic ordering과 byte-identical reproducibility를 구분합니다.
- summary command가 threshold를 판정하는 gate가 아니라 generated evidence writer라는 점을 명시합니다.

확인 원칙:

- 먼저 `f1c72dfdd16a^`와 `f1c72dfdd16a`를 비교하고, 필요한 file은 `f1c72dfdd16a:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | LHCI는 raw reports와 pass/fail을 만들 수 있었지만 median values와 environment를 한 reviewable repository file로 정규화하는 command가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `lighthouse:summarize`와 104-line summarizer를 추가했습니다. raw LHRs를 final URL로 group하고 performance/accessibility score, CLS, LCP, TBT의 모든 run과 median을 기록합니다. targets, measuredAt, Chrome UA, Node, platform, arch, CPU/core/memory도 output에 포함합니다. |
| Build/runtime/resource owner와 lifetime | raw `.lighthouseci` reports가 input lifetime을, summarizer가 transformation/output path를, committed result review가 provenance 해석을 소유합니다. |
| Failure·missing output·cleanup 처리 | input directory/read/JSON/required field failure가 전파되고 report 0개는 explicit error입니다. 첫 report의 environment를 전체 group 대표로 사용하며 run count가 실제로 3인지 별도 runtime validation하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | URL별 raw runs, median과 material host context가 stable key order로 보존됩니다. summary 자체는 threshold를 enforce하지 않고 timestamp/environment 때문에 동일 source에서 byte-identical하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `a39856cf734a`가 이 형식의 desktop result를 commit합니다. CI `abbd530368a0`은 `lighthouse:audit`만 실행하고 summarize/commit은 하지 않습니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** raw LHR을 repository JSON으로 요약하는 script/package command가 없습니다.
- **해당 SHA 핵심 코드:** `f1c72dfdd16a0be14f7c2a650bde9ee431ec27bf` · `scripts/summarize-lighthouse.mjs`

```text
const filenames = (await readdir(INPUT_DIRECTORY)).filter(
  (filename) => filename.startsWith("lhr-") && filename.endsWith(".json"),
);
if (filenames.length === 0) {
  throw new Error("No Lighthouse JSON reports were found in .lighthouseci.");
}

const routes = Object.fromEntries(
  [...grouped.entries()]
    .sort(([left], [right]) => left.localeCompare(right))
    .map(([url, runs]) => [url, { median: aggregate(runs), runs }]),
);
```

- **관찰 근거의 성격:** Exact-SHA summarizer implementation에서 직접 확인했습니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `a39856cf734a`가 이 형식의 desktop result를 commit합니다. CI `abbd530368a0`은 `lighthouse:audit`만 실행하고 summarize/commit은 하지 않습니다.

### 9. `4e8f95249481` — test(perf): 배포 성능 gate 규칙 검증

- **Full SHA:** `4e8f952494812b4da0125b19ce56518505392cad`
- **Importance:** A
- **Tags:** VALIDATION, PERF, TEST
- **확장 thread에서의 역할:** policy regression suite — compiler/parser assertions를 보존하며 Lighthouse matrix와 budget arithmetic을 검증합니다.

#### 해당 SHA에서 확인할 실제 코드

- 기존 `build-manifest-contract.test.ts`가 제거되고 assertions가 `performance-gates.test.ts`로 이동·확장되는 ownership transfer를 확인합니다.
- 실제 `lighthouserc.cjs`를 require해 production server, 3 runs, desktop, 10 URLs와 exact thresholds를 검사하는지 추적합니다.
- 100/1000 baseline에서 105/1050은 통과하고 106/2101은 route+asset violation이 되는 boundary fixture를 계산합니다.
- missing baseline routes는 test하지만 measured new route without baseline, CLI exit와 actual build/Lighthouse는 직접 test하지 않는 범위를 기록합니다.

확인 원칙:

- 먼저 `4e8f95249481^`와 `4e8f95249481`를 비교하고, 필요한 file은 `4e8f95249481:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | compiler/parser fixture는 test됐지만 Lighthouse configuration drift와 budget evaluator의 exact boundary/fail-closed behavior는 executable contract가 아니었습니다. |
| 실제 변경 file/symbol/command/artifact | 기존 test를 broader `src/performance/performance-gates.test.ts`로 교체했습니다. compiler/parser tests를 보존하고, 5-design×2-route×3-run desktop config와 5 thresholds를 검사합니다. pure budget fixtures로 exactly 5% pass, first-byte over violation, expected route missing을 검증합니다. |
| Build/runtime/resource owner와 lifetime | unit suite가 static config와 pure policy behavior를 소유합니다. actual build files, browser, network와 CLI process는 사용하지 않습니다. |
| Failure·missing output·cleanup 처리 | config URL/threshold/command drift, compiler flag 제거, parser regression, arithmetic/fail-closed regression이 deterministic assertion failure가 됩니다. real performance variability나 stale committed baseline은 이 test가 검출하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | gate를 약화시키는 주요 config/pure evaluator 변화가 test failure 없이 들어가기 어렵습니다. LHCI/asset collector가 실제 environment에서 성공하거나 CLI가 expected exit code를 내는지는 증명하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | Thread 2의 Tailwind transform fix가 이후 artifact correctness를 보강하고, `abbd530368a0`이 unit-tested rules를 actual CI commands로 활성화합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** compiler/parser만 다루는 36-line test가 있고 performance gate policy fixtures는 없습니다.
- **해당 SHA 핵심 코드:** `4e8f952494812b4da0125b19ce56518505392cad` · `src/performance/performance-gates.test.ts`

```text
const measurements: RouteBundleMeasurement = {
  "/": { cssBytes: 106, jsBytes: 1_050 },
  "/projects/[projectId]": { cssBytes: 210, jsBytes: 2_101 },
};

expect(evaluateRouteBudgets(measurements, baseline)).toEqual([
  expect.objectContaining({ asset: "css", route: "/" }),
  expect.objectContaining({
    asset: "js",
    route: "/projects/[projectId]",
  }),
]);
```

- **관찰 근거의 성격:** Exact-SHA test replacement/fixtures에서 직접 확인한 deterministic regression coverage입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** Thread 2의 Tailwind transform fix가 이후 artifact correctness를 보강하고, `abbd530368a0`이 unit-tested rules를 actual CI commands로 활성화합니다.

### 10. `abbd530368a0` — ci: 검증된 bundle과 Lighthouse gate 활성화

- **Full SHA:** `abbd530368a03b8dc40221d663377c1f45e254ee`
- **Importance:** A
- **Tags:** VALIDATION, PERF, DEPLOY
- **확장 thread에서의 역할:** release integration — production build output에 standalone, bundle와 Lighthouse checks를 순차 적용합니다.

#### 해당 SHA에서 확인할 실제 코드

- `test:e2e:ci`가 build 후 production config를 사용하되 `--grep-invert @visual`로 visual cases를 제외하는 정확한 scope를 확인합니다.
- CI가 E2E build 후 `build:verify`, `bundle:check`, browser path export, `lighthouse:audit`을 순서대로 실행해 `.next`를 재사용하는지 추적합니다.
- `CHROME_PATH`가 Playwright-installed Chromium executable에서 생성되어 `$GITHUB_ENV`로 전달되는 ownership을 기록합니다.
- `lighthouse:summarize`와 mobile observation이 CI gate에 포함되는지 확인해 advisory/generated evidence와 enforced checks를 분리합니다.

확인 원칙:

- 먼저 `abbd530368a0^`와 `abbd530368a0`를 비교하고, 필요한 file은 `abbd530368a0:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | bundle/Lighthouse tooling과 tests는 local에 있었지만 CI workflow는 production E2E와 standalone verify까지만 수행했습니다. |
| 실제 변경 file/symbol/command/artifact | CI E2E를 `test:e2e:ci`로 바꿔 build + non-visual production tests를 실행합니다. 이어 standalone verify, route bundle check, Playwright Chromium path export, production Lighthouse audit를 추가합니다. |
| Build/runtime/resource owner와 lifetime | 한 CI workspace의 E2E build가 `.next` producer이고 artifact/bundle/Lighthouse steps가 순차 consumers입니다. Playwright installation이 browser binary를 소유하고 workflow가 path를 LHCI environment로 전달합니다. |
| Failure·missing output·cleanup 처리 | 각 command non-zero가 후속 gate를 막습니다. visual snapshots는 `@visual` exclusion으로 general CI E2E에서 실행되지 않습니다. Lighthouse는 existing production build와 configured server가 준비되지 않으면 실패합니다. |
| 보장하는 것과 보장하지 않는 것 | CI가 standalone layout, committed route growth budget와 enforced desktop Lighthouse thresholds를 자동 차단 조건으로 사용합니다. visual suite, summarize output commit, mobile Lighthouse, field performance와 hosting runtime은 이 gate에 포함되지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `a39856cf734a`는 별도 local lab evidence를 기록하고, Thread 5의 `b94fa6dd0118`이 같은 workflow 끝에 container runtime check를 추가합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** CI에는 bundle check, Lighthouse audit와 browser path handoff가 없고 production E2E가 visual cases까지 포함합니다.
- **해당 SHA 핵심 코드:** `abbd530368a03b8dc40221d663377c1f45e254ee` · `.github/workflows/ci.yml 및 package.json`

```text
- name: Check route bundle budgets
  run: npm run bundle:check

- name: Locate Playwright Chromium for Lighthouse
  run: echo "CHROME_PATH=$(node -e 'process.stdout.write(require(\"playwright\").chromium.executablePath())')" >> "$GITHUB_ENV"

- name: Run production Lighthouse budgets
  run: npm run lighthouse:audit
```

- **관찰 근거의 성격:** Exact-SHA package/workflow diff에서 직접 확인한 CI gate graph입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `a39856cf734a`는 별도 local lab evidence를 기록하고, Thread 5의 `b94fa6dd0118`이 같은 workflow 끝에 container runtime check를 추가합니다.

### 11. `a39856cf734a` — chore(perf): 최종 lab 성능 측정 결과 기록

- **Full SHA:** `a39856cf734a309a32f5c7f239ba6bec90e1c259`
- **Importance:** C
- **Tags:** PERF
- **확장 thread에서의 역할:** generated evidence snapshot — desktop enforced matrix와 별도 mobile observation의 측정 결과를 환경과 함께 보존합니다.

#### 해당 SHA에서 확인할 실제 코드

- `performance/lighthouse-baseline.json`의 measuredAt, environment, 10 URL × 3 runs와 median values가 summarizer schema를 따르는지 확인합니다.
- `performance/lighthouse-mobile-observation.json`의 `enforcement: observation-only`, 1 run과 pass-count summary를 desktop gate와 구분합니다.
- recorded values를 이 작업에서 재실행한 결과로 표현하지 않고 historical generated evidence로만 기록합니다.

확인 원칙:

- 먼저 `a39856cf734a^`와 `a39856cf734a`를 비교하고, 필요한 file은 `a39856cf734a:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | measurement/summarizer와 CI gate는 있었지만 repository에 host context와 raw-run-derived desktop/mobile result snapshot이 없었습니다. |
| 실제 변경 file/symbol/command/artifact | Apple M1, Node v24.18.0, HeadlessChrome 147 환경의 desktop 10 URL×3 run summary와 mobile 10 URL×1 run observation을 commit했습니다. mobile summary는 performance 9/10, accessibility 10/10, LCP 3/10, TBT 9/10, CLS 10/10 target pass를 기록하고 명시적으로 non-enforced입니다. |
| Build/runtime/resource owner와 lifetime | JSON files가 historical measurement record를 소유합니다. CI threshold source는 여전히 `lighthouserc.cjs`이며 이 generated file이 gate input은 아닙니다. |
| Failure·missing output·cleanup 처리 | 이 commit은 data를 추가할 뿐 command failure를 새로 만들지 않습니다. recorded result는 stale해질 수 있고 다른 hardware/network에 일반화되지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 특정 시각·host에서 실행된 lab 결과와 mobile gap이 review 가능합니다. current build의 성능, mobile release pass 또는 CI 재현을 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | desktop file은 `f1c72dfdd16a` summary shape의 historical output이고, mobile file은 observation-only입니다. enforced gate는 `abbd530368a0`의 desktop LHCI command입니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** 두 performance result JSON files가 없습니다.
- **해당 SHA 핵심 코드:** `a39856cf734a309a32f5c7f239ba6bec90e1c259` · `performance/lighthouse-mobile-observation.json`

```text
"measurement": {
  "kind": "Lighthouse mobile lab run against the local production server",
  "enforcement": "observation-only",
  "runCountPerUrl": 1
},
"passedTargets": {
  "performanceScore": 9,
  "accessibilityScore": 10,
  "lcp": 3,
  "tbt": 9,
  "cls": 10
}
```

- **관찰 근거의 성격:** Exact-SHA committed measurement files에서 읽은 historical record입니다.
- **실행·테스트 증거:** 재실행하지 않음. JSON에 기록된 2026-08-13 측정값은 repository의 historical generated evidence이며, 현재 작업이 생산한 runtime evidence로 취급하지 않습니다.
- **다음 commit 연결:** desktop file은 `f1c72dfdd16a` summary shape의 historical output이고, mobile file은 observation-only입니다. enforced gate는 `abbd530368a0`의 desktop LHCI command입니다.

## 6. Invariant ledger

| Invariant | 이전 상태 | 도입·수정 | 검증·소비 | 남은 비보장 |
| --- | --- | --- | --- | --- |
| Manifest trust | generated JS를 직접 해석할 명시 경계 없음 | `c2fb8a7c238d`의 slice+JSON parser | `c24c350ce42c` fixtures와 collector가 소비 | future wrapper/schema |
| Route byte accounting | source/estimate 중심 | `605b64512edf`가 actual manifests/files를 join | `518ff5b51ec5`가 route/asset policy 적용 | compression·lazy/network |
| Baseline governance | accepted state 없음 | `57a1b0876941`이 write/check 분리 | `6ac1ea4b5055` committed reference + CI check | review quality·absolute cap |
| Lab performance | manual/advisory | `1529ccf225c1` desktop matrix/threshold | `4e8f95249481` config test + `abbd530368a0` CI enforcement | mobile/field/all routes |
| Measurement provenance | raw ignored reports | `f1c72dfdd16a` summary writer | `a39856cf734a` environment-bound desktop/mobile records | currentness·cross-machine equivalence |

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Fix/decision | Test·gate evidence | 한계 |
| --- | --- | --- | --- |
| generated manifest를 plain JSON/eval로 오해 | slice + semicolon + JSON.parse | ordinary/dynamic fixtures | future syntax는 fail-closed |
| route/asset가 사라져 낮은 숫자로 오인 | expected route missing/new baseline missing violations | missing route policy test | extra measured route test는 없음 |
| routine check가 baseline을 자동 승인 | write/check commands 분리 | CI는 only `bundle:check` | manual baseline approval는 process 책임 |
| Lighthouse config threshold가 약화 | exact config values | config contract test | real noise는 3-run median으로만 완화 |
| local gate가 integration에서 누락 | CI sequential steps + browser handoff | command non-zero가 release 차단 | visual/mobile/field는 비차단 |

## 8. Ownership / state / responsibility 변화

| 대상 | 이전 owner/state | 중간 변화 | 최종 owner/state |
| --- | --- | --- | --- |
| Generated manifest interpretation | 각 caller/암묵적 | `parseClientReferenceManifest` | collector가 consumer |
| Route bytes | 없음 | Next manifests + collector | baseline/evaluator/CLI |
| Accepted growth state | 없음 | committed JSON + review | CI routine read-only check |
| Lab threshold | 없음 | `lighthouserc.cjs` | unit contract + LHCI CI process |
| Browser binary | LHCI environment 가정 | Playwright install | workflow `$GITHUB_ENV` handoff |
| Result history | ignored raw reports | summarizer + committed JSON | non-authoritative historical evidence |

## 9. Thread 최종 상태

webpack production artifact의 route별 uncompressed client JS/CSS가 explicit baseline에 대해 5% 성장 및 route coverage gate를 통과해야 합니다. production server의 5-design home/detail desktop matrix도 median performance/accessibility/LCP/TBT/CLS threshold를 통과해야 CI가 성공합니다. mobile JSON은 observation-only이며 field data, all-route coverage와 absolute bundle cap은 없습니다.

## 10. 최종 product-delivery flow 정리

`npm run build --webpack` → app/build/client-reference manifests 생성 → parser가 JSON payload 추출 → collector가 actual asset bytes 계산 → `bundle:check`가 committed baseline/5% policy와 비교 → CI가 standalone verify와 bundle check를 수행 → Playwright Chromium path를 LHCI에 전달 → port 3300 production server에서 10 URLs×3 desktop runs → median threshold failure가 CI를 차단합니다.

## 11. 학습 완료 자가 점검

- [x] parser가 code execution 없이 generated wrapper payload만 해석하는 방식을 설명했습니다.
- [x] route byte accounting의 shared/route/dedup/non-inlined 규칙과 비측정 범위를 기록했습니다.
- [x] baseline write와 routine check의 ownership을 분리했습니다.
- [x] desktop Lighthouse matrix/threshold와 mobile observation-only 상태를 구분했습니다.
- [x] CI가 actual gate를 실행하지만 summary file을 자동 갱신하지 않는다고 기록했습니다.
- [ ] Exact-SHA runtime command를 직접 실행해 결과를 기록했습니다. — 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
===== END FILE: 04-release-performance-gates.md =====

===== BEGIN FILE: 05-container-packaging-and-runtime-verification.md =====
# Thread: Container packaging and runtime verification

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 원본 Development Thread를 변경하지 않고, 같은 branch history에 product-delivery 관점을 추가한 확장 workbook입니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 `commit/commit-importance.md`의 branch-scoped 분류를 사용합니다.
- Phase 1 audit에서 category/thread grouping과 commit set을 실제 history에 대조한 뒤 이 문서를 freeze했습니다.
- Phase 2는 freeze된 구조와 fixed metadata를 바꾸지 않고 learner-facing 기록만 완성합니다.
- 다른 branch의 구현이나 final HEAD를 과거 SHA 설명에 소급하지 않습니다.
- 실행하지 않은 build/test/CI/Docker 결과는 exact-SHA source inspection과 구분합니다.

## 1. Thread 목표

verified standalone output을 최소 runtime image로 옮기고, public assets를 명시적으로 포함하며, non-root process·실제 HTTP routes·content-derived assets를 ephemeral container에서 검증한 뒤 같은 contract를 CI에 연결하는 과정을 복원합니다.

### 계획된 핵심 invariant

- builder는 pinned Node/npm graph에서 production build와 standalone verification을 통과해야 runtime stage를 만들 수 있습니다.
- runner는 standalone server, `.next/static`, `public`만 명시적으로 가져오고 `node` user로 실행합니다.
- container test는 unique image/container name과 loopback ephemeral port를 사용하고 readiness를 bounded retry로 기다립니다.
- content JSON이 참조하는 `/content`·`/template` assets는 200, non-empty body와 supported MIME contract를 만족해야 합니다.
- 실패·성공 여부와 관계없이 started container와 temporary image cleanup을 시도합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- standalone output에 자동 포함되지 않는 `.next/static`과 `public`을 final image로 누가 복사하는가?
- Docker multi-stage build에서 dependency install, build args, artifact verification과 runtime user의 ownership이 어떻게 분리되는가?
- container verifier가 readiness, non-root identity, routes, dynamically discovered assets와 cleanup을 어떤 state machine으로 수행하는가?
- default template-mode image test가 production content/origin, image security, multi-architecture와 orchestrator behavior에 대해 무엇을 보장하지 않는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 변경 file, function, config, script와 workflow step을 확인했습니다.
- Source, generated artifact, CI gate, container/runtime owner를 구분했습니다.
- Missing artifact, portability failure, threshold violation, startup failure와 cleanup branch를 기록했습니다.
- Test/CI command의 technique, production path, proves/does-not-prove와 실제 실행 여부를 구분했습니다.
- 최종 product-delivery 흐름과 cross-thread handoff를 코드 없이 설명할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 확장 thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b87a2b453741` | build(docker): public 자산을 포함한 비루트 standalone image 추가 | A | DEPLOY | deployable image boundary — verified standalone artifact, static/public assets와 non-root runtime을 multi-stage Dockerfile로 구성합니다. |
| 2 | `b94fa6dd0118` | test(docker): runtime route와 public 자산 검증 자동화 | A | ARCH, VALIDATION, ROUTING | end-to-end runtime contract — image build부터 non-root identity, HTTP routes/assets와 cleanup까지 자동화하고 CI에 연결합니다. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA의 tree와 parent diff를 기준으로 작성합니다. 다른 Thread의 later commit은 관계 설명에만 사용하고 과거 구현에 소급하지 않습니다.

### 1. `b87a2b453741` — build(docker): public 자산을 포함한 비루트 standalone image 추가

- **Full SHA:** `b87a2b4537418771530ae520df448ca84142f80c`
- **Importance:** A
- **Tags:** DEPLOY
- **확장 thread에서의 역할:** deployable image boundary — verified standalone artifact, static/public assets와 non-root runtime을 multi-stage Dockerfile로 구성합니다.

#### 해당 SHA에서 확인할 실제 코드

- dependencies/builder/runner 세 stage의 base image, npm pin, workdir, copy와 command ordering을 추적합니다.
- `ARG PORTFOLIO_CONTENT_MODE=template`와 `ARG SITE_URL`이 builder environment에만 전달되고 final runtime environment에는 무엇이 남는지 확인합니다.
- builder의 `npm run build && npm run build:verify`가 image assembly를 fail-closed로 막는지 확인합니다.
- runner가 `USER node`와 `--chown=node:node` copy를 사용하고 standalone, static, public을 각각 별도 source에서 가져오는 이유를 기록합니다.
- `.dockerignore`가 credentials/local output을 build context에서 제외하지만 supply-chain/image scan을 제공하지 않는다는 범위를 명시합니다.

확인 원칙:

- 먼저 `b87a2b453741^`와 `b87a2b453741`를 비교하고, 필요한 file은 `b87a2b453741:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | standalone artifact와 CI file-layout check는 있었지만 이를 실행 가능한 image로 조립하는 Dockerfile이 없었습니다. 특히 Next standalone trace에 자동 포함되지 않는 `public` directory와 generated static assets의 배치 owner가 정의되지 않았습니다. |
| 실제 변경 file/symbol/command/artifact | Node 24.18.0 bookworm-slim 기반 3-stage Dockerfile과 `.dockerignore`를 추가했습니다. dependencies stage는 npm 11.16.0과 `npm ci`, builder는 content mode/origin args를 환경으로 전달하고 build+verify를 수행합니다. runner는 `USER node`, host 0.0.0.0, port 3100으로 standalone, `.next/static`, `public`을 `--chown=node:node`로 복사해 `node server.js`를 실행합니다. |
| Build/runtime/resource owner와 lifetime | dependencies stage가 install graph를, builder stage가 source/build output을, runner stage가 deployable filesystem/process identity를 소유합니다. multi-stage boundary에서 development node_modules/source는 final image로 직접 복사되지 않습니다. |
| Failure·missing output·cleanup 처리 | `npm ci`, content readiness/build 또는 `build:verify`가 실패하면 final image가 생성되지 않습니다. source path가 없으면 Docker COPY가 실패합니다. 이 SHA에는 built image를 실행하거나 UID/HTTP/assets를 검증하는 command가 없습니다. |
| 보장하는 것과 보장하지 않는 것 | image recipe는 pinned runtime, verified standalone entry, generated static, public assets와 named non-root user를 명시합니다. image vulnerability/signature, exact numeric UID, production content mode, runtime response, multi-arch build와 orchestrator policy는 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | `b94fa6dd0118`이 actual image를 build/run하고 Config.User, routes와 content-derived assets를 검사합니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 `Dockerfile`과 `.dockerignore`가 없습니다.
- **해당 SHA 핵심 코드:** `b87a2b4537418771530ae520df448ca84142f80c` · `Dockerfile`

```text
FROM node:24.18.0-bookworm-slim AS builder
ARG PORTFOLIO_CONTENT_MODE=template
ARG SITE_URL
ENV PORTFOLIO_CONTENT_MODE=$PORTFOLIO_CONTENT_MODE
ENV SITE_URL=$SITE_URL
COPY . .
RUN npm run build && npm run build:verify

FROM node:24.18.0-bookworm-slim AS runner
ENV NODE_ENV=production
ENV HOSTNAME=0.0.0.0
ENV PORT=3100
WORKDIR /app
USER node
COPY --from=builder --chown=node:node /app/.next/standalone ./
COPY --from=builder --chown=node:node /app/.next/static ./.next/static
COPY --from=builder --chown=node:node /app/public ./public
CMD ["node", "server.js"]
```

- **관찰 근거의 성격:** Exact-SHA Dockerfile/build-context diff에서 직접 확인한 image assembly contract입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** `b94fa6dd0118`이 actual image를 build/run하고 Config.User, routes와 content-derived assets를 검사합니다.

### 2. `b94fa6dd0118` — test(docker): runtime route와 public 자산 검증 자동화

- **Full SHA:** `b94fa6dd0118322ff57dc84b180b1c179ca8a867`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, ROUTING
- **확장 thread에서의 역할:** end-to-end runtime contract — image build부터 non-root identity, HTTP routes/assets와 cleanup까지 자동화하고 CI에 연결합니다.

#### 해당 SHA에서 확인할 실제 코드

- `scripts/verify-container-runtime.mjs`의 random suffix, image/container names와 `docker()` child-process wrapper의 capture/non-capture behavior를 추적합니다.
- `discoverAssets`가 top-level `src/content/*.json`을 recursive value traversal해 `/content/`·`/template/` string만 deduplicate하는지 확인합니다.
- build → detached run → ephemeral loopback port parse → 최대 60×1초 readiness → Config.User → two routes → every asset 순서를 state transition으로 기록합니다.
- `verifyResponse`의 status 200, non-empty body, extension-based MIME checks와 supported extension set을 확인합니다.
- `failed`/`containerStarted` flags, failure log, `finally`의 container/image removal과 cleanup error precedence를 설명합니다.
- workflow 마지막 `npm run test:container`이 Docker availability를 CI precondition으로 만드는지 확인합니다.

확인 원칙:

- 먼저 `b94fa6dd0118^`와 `b94fa6dd0118`를 비교하고, 필요한 file은 `b94fa6dd0118:<path>`의 resulting tree에서 읽습니다.
- Final HEAD의 workflow, script, Dockerfile 또는 generated output을 이 commit에 소급하지 않습니다.
- Commit subject나 body만으로 behavior를 추정하지 않고 실제 changed code/test/config를 기준으로 판단합니다.
- 실제 실행하지 않은 command 결과는 code inspection과 분리합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 전달 상태와 부족함 | Dockerfile은 있었지만 image를 실제로 시작해 non-root user, standalone HTTP routes와 copied public assets를 검증하거나 temporary resources를 정리하는 automated contract가 없었습니다. |
| 실제 변경 file/symbol/command/artifact | `test:container`, 203-line verifier와 CI step을 추가했습니다. script는 unique tag/name으로 image를 build하고 host loopback의 random published port로 container를 시작합니다. readiness 후 Docker inspect user가 `node`인지 확인하고 `/`, `/projects/example-project?view=classic`, 그리고 content JSON에서 발견한 모든 supported public asset을 요청합니다. |
| Build/runtime/resource owner와 lifetime | test script가 temporary Docker resources의 full lifecycle을 소유합니다. `docker()`가 child process exit/stdout/stderr를, flags가 state를, `finally`가 cleanup을 관리합니다. content JSON은 asset set source of truth이고 MIME map은 supported serving contract를 소유합니다. |
| Failure·missing output·cleanup 처리 | Docker command non-zero, asset set empty, port parse 실패, 60초 readiness timeout, root/empty user, non-200, empty body, MIME mismatch/unsupported extension이 test를 실패시킵니다. failure 시 container logs를 best-effort 출력합니다. started container와 image는 `finally`에서 제거하고, primary failure가 이미 있으면 cleanup error는 원인을 가리지 않도록 억제합니다. |
| 보장하는 것과 보장하지 않는 것 | default Docker build로 만들어진 image가 named `node` user로 시작하고 두 HTML routes와 content-referenced public assets를 실제 HTTP로 제공합니다. default build는 template mode이며 production `SITE_URL` readiness, numeric UID/capabilities, image CVEs/signing, healthcheck, multi-arch, load/concurrency, redirect semantics과 orchestrator restart는 보장하지 않습니다. |
| 다음 delivery commit 또는 관련 test 연결 | Thread 3의 standalone contract와 이 Thread의 Dockerfile을 actual runtime evidence로 연결하며, CI workflow의 마지막 gate가 됩니다. |

#### 코드·실행 증거 기록

- **변경 전 대응 코드:** Parent에는 container verification script/package command/CI step이 없습니다.
- **해당 SHA 핵심 코드:** `b94fa6dd0118322ff57dc84b180b1c179ca8a867` · `scripts/verify-container-runtime.mjs`

```text
} finally {
  if (containerStarted) {
    try {
      await docker(["rm", "--force", containerName], { capture: true });
    } catch (error) {
      if (!failed) throw error;
    }
  }
  try {
    await docker(["image", "rm", "--force", imageName], { capture: true });
  } catch (error) {
    if (!failed) throw error;
  }
}
```

- **관찰 근거의 성격:** Exact-SHA verifier/workflow implementation에서 직접 확인한 Docker E2E state machine입니다.
- **실행·테스트 증거:** 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
- **다음 commit 연결:** Thread 3의 standalone contract와 이 Thread의 Dockerfile을 actual runtime evidence로 연결하며, CI workflow의 마지막 gate가 됩니다.

## 6. Invariant ledger

| Invariant | 이전 상태 | 도입·수정 | 검증·소비 | 남은 비보장 |
| --- | --- | --- | --- | --- |
| Image assembly | artifact files만 local `.next`에 존재 | `b87a2b453741` multi-stage build가 verified output을 runtime stage로 copy | Docker build가 build+verify 실패를 차단 | supply-chain·multiarch |
| Runtime identity | standalone process user 미검증 | Dockerfile `USER node` + chowned files | `b94fa6dd0118`이 Config.User를 actual container에서 확인 | numeric UID/capabilities |
| Static/public delivery | standalone trace 외부 | static/public explicit copy | content-derived HTTP/MIME verification | JSON 밖 assets·unsupported types |
| Temporary resource lifetime | manual Docker lifecycle | unique names + state flags | failure logs + finally removal | daemon-level leaked state on hard termination |
| CI runtime gate | image recipe local-only | `test:container` package command | workflow last step에서 actual Docker contract 실행 | external registry/deployment |

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Fix/decision | Test·gate evidence | 한계 |
| --- | --- | --- | --- |
| standalone image에 public/static 누락 | three explicit COPY boundaries | content-derived HTTP/MIME test | JSON이 참조하지 않는 public file은 미검사 |
| container가 root로 실행 | `USER node` + chown | Docker inspect Config.User equality | numeric UID/capability policy 없음 |
| server startup 지연/실패 | bounded 60-attempt readiness loop | last error를 포함해 timeout failure | dedicated health endpoint 없음 |
| temporary image/container 누적 | random unique names + finally cleanup | primary error 보존, success cleanup failure는 표면화 | process kill/daemon crash는 제외 |
| local-only image confidence | CI `npm run test:container` | build/run/routes/assets가 integration gate | registry push/orchestrator는 없음 |

## 8. Ownership / state / responsibility 변화

| 대상 | 이전 owner/state | 중간 변화 | 최종 owner/state |
| --- | --- | --- | --- |
| Dependency graph | host install | Docker dependencies stage | builder가 consume |
| Build artifact | local `.next` | Docker builder + `build:verify` | runner에 selected copy |
| Runtime process/files | 정의 없음 | runner stage + `node` user | container verifier가 observe |
| Asset inventory | manual list 가능성 | authoritative content JSON traversal | MIME map + HTTP verifier |
| Temporary resources | manual | test script flags/names | `finally` cleanup |

## 9. Thread 최종 상태

pinned runtime에서 build/verify된 standalone artifact, generated static과 public directory를 포함한 image가 named non-root `node` user로 실행됩니다. CI는 실제 image를 ephemeral loopback port에서 시작해 two routes와 content-derived assets의 200/non-empty/MIME contract를 검사하고 resources를 정리합니다. 이는 image runtime contract이지 production hosting, registry, security scan 또는 production content publication 증명은 아닙니다.

## 10. 최종 product-delivery flow 정리

Docker context filtering → pinned Node/npm dependency stage의 `npm ci` → builder가 content args를 받고 production build + standalone verify → runner가 standalone/static/public만 chown-copy → unique image build → detached non-root container를 random loopback port에 publish → bounded readiness → user inspect → HTML routes → JSON-derived assets/MIME 검증 → failure log → container/image cleanup → CI exit status.

## 11. 학습 완료 자가 점검

- [x] standalone, generated static과 public의 서로 다른 copy ownership을 설명했습니다.
- [x] multi-stage build의 build-time args와 final runtime environment를 구분했습니다.
- [x] container verifier의 state transition, retry, failure와 cleanup을 복원했습니다.
- [x] content-derived asset discovery/MIME coverage와 누락 범위를 기록했습니다.
- [x] template-mode CI image test와 production deployment 보장을 혼동하지 않았습니다.
- [ ] Exact-SHA runtime command를 직접 실행해 결과를 기록했습니다. — 실행하지 않음. 현재 작업 환경에서는 `web/portfolio`의 Git checkout, npm dependency tree, Chromium 및 Docker daemon을 사용할 수 없었습니다. GitHub connector로 해당 SHA의 commit diff와 resulting source를 검사했으며, command 성공 결과는 주장하지 않습니다.
===== END FILE: 05-container-packaging-and-runtime-verification.md =====

===== BEGIN FILE: README.md =====
# Product delivery and runtime verification

## 범위

재현 가능한 toolchain, production-server 검증, self-contained production build, standalone artifact, release performance gate와 non-root container runtime까지 실제 제품 전달 경로를 다룹니다.

외부 hosting/provider 설정, registry publication, orchestrator와 배포 후 운영 절차는 `web/portfolio` branch history에 근거가 없으므로 Thread를 만들지 않습니다.

## Phase 1 category audit 결과

- Branch scope는 `web/portfolio`만 사용했습니다. `commit/commit-importance.md`가 이 branch의 독립 선형 history를 선언하고, 이 category의 24개 SHA는 모두 해당 분류에 존재합니다.
- 가장 이른 `f66b880a8f97`과 가장 늦은 `b94fa6dd0118`은 모두 `web/portfolio`의 ancestor로 확인했습니다. 각 SHA는 exact commit object와 diff를 별도로 검사했습니다.
- Category boundary와 5개 Thread는 유지했습니다. Commit 추가·삭제·이동·중복은 하지 않았습니다.
- 원래 scaffold의 generic investigation 문구는 exact file, function, script, workflow, artifact, failure와 non-guarantee를 묻는 commit-specific 과제로 교체했습니다.
- `5d903132306a`는 단순히 font/CSS fix에 끼워 넣지 않았습니다. Next/ESLint/SWC patch-line 정렬과 GNU·musl lockfile 조건을 다루는 framework/native portability 단계로 역할을 좁혀 Thread 2에 유지했습니다.
- Tailwind fix 뒤의 broad visual regression은 결과를 간접 보호하지만 PostCSS config를 직접 격리한 test는 아닙니다. 따라서 이 category commit map에 중복 추가하지 않고 completed record에서 test gap으로 명시합니다.
- Thread 2·3·4의 실제 commits는 history에서 일부 교차합니다. 문서 순서는 source → artifact → release gate → runtime이라는 학습 dependency 순서이며, 각 Thread 내부 commit 순서는 실제 branch 순서를 유지합니다.

## Thread 경계

1. [Reproducible toolchain and production-server verification](01-reproducible-toolchain-and-production-server-verification.md) — runtime pin, production E2E와 최초 CI gate
2. [Self-contained production build and portability](02-self-contained-production-build-and-portability.md) — local fonts, compiler, framework/native dependency와 CSS transform
3. [Standalone artifact contract and CI verification](03-standalone-artifact-contract-and-ci-verification.md) — standalone generation, minimum layout와 CI handoff
4. [Release performance gates](04-release-performance-gates.md) — webpack manifest measurement, reviewable bundle baseline, desktop Lighthouse와 CI enforcement
5. [Container packaging and runtime verification](05-container-packaging-and-runtime-verification.md) — multi-stage non-root image와 actual HTTP/public-asset verification

## Cross-thread handoff

- Thread 1의 pinned toolchain과 production E2E가 이후 모든 build/CI path의 실행 기반입니다.
- Thread 2의 webpack/CSS/font portability가 Thread 4의 measured artifact가 의미 있는 production output이 되게 합니다.
- Thread 3의 standalone contract가 Thread 5 Docker builder의 입력이며, Thread 5가 Thread 3에서 검증하지 않은 `public`과 actual runtime을 확인합니다.
- Thread 4의 CI activation 뒤 Thread 5가 container gate를 workflow 마지막 단계에 추가합니다.

## 문서 사용법

1. Thread 목표와 commit map을 먼저 읽습니다.
2. 각 SHA를 parent와 비교하고 해당 SHA의 resulting tree를 확인합니다.
3. build/test/CI/Docker command는 source inspection과 실제 실행 결과를 구분해 기록합니다.
4. artifact ownership, failure mode, cleanup과 release blocker가 되는 조건을 연결합니다.
5. 마지막에 source → build → artifact → verification → runtime의 최종 전달 흐름을 코드 없이 설명합니다.
===== END FILE: README.md =====

