===== BEGIN FILE: 01-content-contract-test-harness.md =====
# Thread: Content contract test harness

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `07-testing-performance-and-regression-strategy`
>
> Phase 1에서 감사·수정한 뒤 동결한 scaffold를 기준으로 합니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance와 tags는 branch-local `commit/commit-importance.md` 분류와 exact commit metadata를 기준으로 고정했습니다.
- 이 문서의 Thread goal, commit grouping과 source-defined 역할은 Phase 1 category audit 결과입니다.
- Phase 2에서는 SHA, 순서, subject, importance, tags, 역할, 질문과 문서 구조를 바꾸지 않습니다.
- 다른 branch 또는 final HEAD의 구현을 earlier SHA 설명에 소급하지 않습니다.
- Runtime evidence는 실제로 실행한 command만 기록하며, 미실행 상태를 통과로 해석하지 않습니다.

## 1. Thread 목표

Vitest/jsdom/Testing Library 실행 경계를 만들고 content ingestion, public selector surface, clone ownership, route projection rules를 단계적으로 executable contract로 고정하는 과정을 복원합니다.

### 동결된 핵심 invariant

- Test는 production loader·validator·selector·view-model path를 직접 호출하며 별도 모형 구현을 진실의 source로 만들지 않습니다.
- `getPortfolioContent()`가 반환하는 mutable aggregate는 호출 간 격리되고, 의도적으로 immutable한 root metadata만 reference를 공유합니다.
- Route view model은 shared shell fields와 route-specific fields만 노출하며 full `PortfolioContent` spread를 허용하지 않습니다.
- Unknown reference의 null/omission/fallback policy는 route factory에서 명시되고 renderer가 다시 검색하지 않습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 최초 harness는 어떤 config·setup·production path를 연결했고 malformed fixture를 어디서 주입했는가?
- Public facade 목록과 clone/reference boundary는 왜 같은 contract에서 검사되는가?
- Home, index, detail, about, resume, contact의 selection/order/fallback은 renderer 이전 어디서 결정되는가?
- Journey와 interview map의 unresolved reference는 각각 어떤 형태로 남거나 제거되는가?

## 3. 완료 기준

- 각 referenced SHA의 exact parent diff와 resulting changed files를 확인합니다.
- Commit별 previous state, implementation decision, ownership/lifetime, failure path와 non-guarantee를 구분합니다.
- Fix는 earlier assumption과 root cause에 연결하고, test는 production path·technique·proves/does-not-prove를 구분합니다.
- A-level은 subsystem·failure·verification 관계까지, B-level은 local role과 후속 연결까지만 설명합니다.
- Thread-level invariant evolution, Failure → Fix → Test, ownership transfer와 final flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3353032ba23b` | test(content): Vitest 기반 콘텐츠 계약 검증 추가 | A | CONTENT, VALIDATION, TEST | test runner·jsdom setup과 production content contract의 최초 실행 경계 |
| 2 | `dc07871c4d24` | test(portfolio): selector와 presentation 회귀 계약 보강 | A | CONTENT, TEST | public module surface와 mutable/immutable copy ownership 회귀 계약 |
| 3 | `b77b386b344e` | test(content): route view model 파생 규칙 검증 | A | ARCH, CONTENT, VALIDATION | 여섯 route view-model factory의 selection·ordering·fallback contract |
| 4 | `527b9f872333` | test(content): scoped view model과 연락처 회귀 검증 | A | CONTENT, VALIDATION, TEST | 여덟 route의 scoped field whitelist와 unresolved-reference hardening |

## 5. Commit별 학습 기록

각 section은 해당 SHA의 tree와 parent diff만 기준으로 작성합니다. 같은 SHA가 다른 category Thread에 등장하더라도 여기서는 위 역할과 파일 범위만 설명합니다.

### 1. `3353032ba23b` — test(content): Vitest 기반 콘텐츠 계약 검증 추가

- **Full SHA:** `3353032ba23bf0890b3ac0410e3b55638bc70df6`
- **Importance:** A
- **Tags:** CONTENT, VALIDATION, TEST
- **이 Thread에서의 역할:** test runner·jsdom setup과 production content contract의 최초 실행 경계

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `test`·`test:watch` script와 Vitest/jsdom/Testing Library devDependencies
- `vitest.config.ts`의 jsdom environment, test include pattern, `src/test/setup.ts` 연결
- `src/test/setup.ts`의 jest-dom matcher 등록
- `src/lib/portfolio.test.ts`가 `loadPortfolioSource`, `validatePortfolioAssets`, selectors와 `PortfolioContentError`를 실제로 호출하는 방식
- 정상 source와 의도적으로 변형한 fixture가 ID, cross-reference, asset, route, metric, selector failure를 어떻게 관찰하는지

#### Commit-specific investigation

- `3353032ba23b^`와 `3353032ba23b`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `test runner·jsdom setup과 production content contract의 최초 실행 경계`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `dc07871c4d24`가 같은 suite에 public facade surface와 copy/reference ownership을 추가하고, 이후 `b77b...`·`527b...`가 route view-model projection contract를 별도 test file로 확장합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 직전 parent에는 content loader, asset validation, selectors가 이미 있었지만 이를 반복 실행하는 Vitest 설정·test script·jsdom setup·contract suite가 없었습니다. 따라서 schema와 selector가 바뀌어도 build 또는 수동 화면 확인 전에는 회귀를 빠르게 고정할 수 없었습니다. |
| 실제 변경 file/symbol/call path | `package.json`/`package-lock.json`에 Vitest, jsdom, Testing Library, jest-dom을 추가하고 `vitest.config.ts`, `src/test/setup.ts`, `src/lib/portfolio.test.ts`를 만들었습니다. 테스트는 별도 모형 구현을 만들지 않고 production loader와 validation/selectors를 직접 호출합니다. |
| Data/state/DOM/resource owner와 lifetime | Content truth의 owner는 production의 `loadPortfolioSource`와 validation/selectors에 남습니다. Test suite는 JSON source를 읽고 필요할 때 clone을 변형해 input을 만들며, thrown `PortfolioContentError`와 반환값을 관찰하는 consumer입니다. Test harness가 production state를 대신 소유하지 않습니다. |
| Failure·absence·fallback·cleanup | 정상 fixture만 확인하지 않고 잘못된 ID·참조·asset·route/metric 조건을 주입하여 fail-closed path를 통과시킵니다. Error capture helper는 예상한 `PortfolioContentError`를 분리해 메시지와 issue를 검증합니다. 단, 이 commit의 failure injection은 loader/selector 입력 변형이지 filesystem·network failure injection은 아닙니다. |
| Test technique와 실행 증거 | Unit/contract testing입니다. jsdom은 DOM matcher를 제공하지만 이 commit의 핵심은 실제 content ingestion과 selector path를 deterministic fixture로 실행하는 것입니다. Exact test body와 changed files는 확인했으나 test command는 실행하지 않았습니다. |
| 보장하는 것 | 이 SHA에서 `npm test`로 실행 가능한 content contract boundary가 생기고, branch의 현재 JSON source가 loader·asset validation·selector 규칙을 만족하는지 코드로 표현됩니다. |
| 보장하지 않는 것 | Route component의 실제 render, browser navigation, hydration, CSS, production build와 deployed asset availability는 증명하지 않습니다. Test가 추가된 시점의 fixture와 assertion 범위 밖 규칙도 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `dc07871c4d24`가 같은 suite에 public facade surface와 copy/reference ownership을 추가하고, 이후 `b77b...`·`527b...`가 route view-model projection contract를 별도 test file로 확장합니다. |

#### 최소 code evidence

- **Commit:** `3353032ba23bf0890b3ac0410e3b55638bc70df6`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `package.json`의 `test`·`test:watch` script와 Vitest/jsdom/Testing Library devDependencies; `vitest.config.ts`의 jsdom environment, test include pattern, `src/test/setup.ts` 연결; `src/test/setup.ts`의 jest-dom matcher 등록

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 2. `dc07871c4d24` — test(portfolio): selector와 presentation 회귀 계약 보강

- **Full SHA:** `dc07871c4d24ddcd85aa15d41ab7fb334ed784a6`
- **Importance:** A
- **Tags:** CONTENT, TEST
- **이 Thread에서의 역할:** public module surface와 mutable/immutable copy ownership 회귀 계약

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio.test.ts`의 `import * as portfolio`와 정확한 export-name 목록
- `getPortfolioContent()` 두 호출 사이의 root, `projects`, project item, nested `links`, top-level `links` identity 비교
- `site`, `profile`, `presentation`, `journey`가 같은 reference로 유지되는 assertion
- 이 copy policy가 production `src/lib/portfolio.ts`의 facade 구현과 일치하는지

#### Commit-specific investigation

- `dc07871c4d24^`와 `dc07871c4d24`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `public module surface와 mutable/immutable copy ownership 회귀 계약`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `b77b386b344e`는 aggregate를 renderer에 직접 넘기는 대신 route factory가 파생 결과를 준비한다는 다음 ownership boundary를 테스트합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 초기 suite는 content의 값과 selector 결과를 확인했지만 public facade에서 어떤 symbol이 노출되어야 하는지, 호출자가 받은 aggregate 중 어떤 부분을 독립적으로 변경할 수 있는지는 고정하지 않았습니다. Export drift나 clone 범위 변화가 컴파일만 통과할 수 있었습니다. |
| 실제 변경 file/symbol/call path | `src/lib/portfolio.test.ts` 하나를 확장하여 public exports를 정확한 목록으로 비교하고, 두 aggregate 호출의 identity를 단계별로 비교합니다. Root aggregate, projects array/items/nested links와 top-level links는 새 객체이고, site/profile/presentation/journey는 동일 source reference라는 비대칭 copy policy를 명시합니다. |
| Data/state/DOM/resource owner와 lifetime | `getPortfolioContent()`가 요청별 mutable collection copy의 owner입니다. 반면 site/profile/presentation/journey는 canonical validated source가 공유하는 read-only reference로 취급됩니다. Test는 deep clone 전체를 요구하지 않고 실제 의도된 ownership 경계를 고정합니다. |
| Failure·absence·fallback·cleanup | 이 commit은 runtime error branch를 새로 만들지 않습니다. 위험은 public export의 누락·추가와 clone boundary가 조용히 바뀌어 caller mutation이 다른 consumer에 전파되는 것입니다. Identity assertions가 그 regression을 즉시 드러냅니다. |
| Test technique와 실행 증거 | Public API characterization와 reference-identity contract test입니다. 실제 facade module을 namespace import하고 실제 `getPortfolioContent()`를 두 번 호출합니다. |
| 보장하는 것 | 허용된 public selector 목록과 mutable collection의 fresh-copy 범위가 명시적으로 고정됩니다. |
| 보장하지 않는 것 | 공유 reference를 `Object.freeze` 하거나 caller mutation을 차단하지는 않습니다. `site` 등 shared object를 변경해도 안전하다는 뜻이 아니라, 이 시점에는 shared immutable input으로 취급한다는 계약입니다. Nested object 전체의 deep-copy 여부도 assertion에 없는 범위는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `b77b386b344e`는 aggregate를 renderer에 직접 넘기는 대신 route factory가 파생 결과를 준비한다는 다음 ownership boundary를 테스트합니다. |

#### 최소 code evidence

- **Commit:** `dc07871c4d24ddcd85aa15d41ab7fb334ed784a6`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `src/lib/portfolio.test.ts`의 `import * as portfolio`와 정확한 export-name 목록; `getPortfolioContent()` 두 호출 사이의 root, `projects`, project item, nested `links`, top-level `links` identity 비교; `site`, `profile`, `presentation`, `journey`가 같은 reference로 유지되는 assertion

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 3. `b77b386b344e` — test(content): route view model 파생 규칙 검증

- **Full SHA:** `b77b386b344e60e1aa2ed3eafd76ab5dafb32342`
- **Importance:** A
- **Tags:** ARCH, CONTENT, VALIDATION
- **이 Thread에서의 역할:** 여섯 route view-model factory의 selection·ordering·fallback contract

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/view-models.test.ts`의 `createHomeViewModel`, `createProjectIndexViewModel`, `createProjectDetailViewModel`
- `createAboutViewModel`, `createResumeViewModel`, `createContactViewModel`
- Home의 fixed date, featured/lead fallback, hero/footer placement, metric derivation
- Project group order·중복 방지, detail unknown-ID `null`, stack/image/link preparation
- About/resume/contact의 source-order 유지, unknown reference omission, cinematic contact fallback

#### Commit-specific investigation

- `b77b386b344e^`와 `b77b386b344e`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `여섯 route view-model factory의 selection·ordering·fallback contract`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `527b9f872333`가 journey/interview를 추가하고, route별 허용 source field와 full-content spread 금지를 structural regression으로 강화합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Route view-model factories는 존재했지만 renderer가 의존하는 파생 규칙을 독립적으로 고정한 test가 없었습니다. Production implementation이 같은 shape를 반환해도 project order, unknown reference 처리, lead/contact fallback이 바뀌면 각 design renderer가 서로 다른 결과를 낼 수 있었습니다. |
| 실제 변경 file/symbol/call path | 새 `src/lib/portfolio/view-models.test.ts`가 실제 `getPortfolioContent()`와 여섯 factory를 호출합니다. Home은 2026-07-23의 고정 시간을 주입하고, project index는 configured group order와 전체 project의 exactly-once inclusion을, detail은 links/stack/supporting images와 missing ID를 검증합니다. About/resume/contact도 ID resolution과 order/fallback을 확인합니다. |
| Data/state/DOM/resource owner와 lifetime | Raw aggregate 검색과 파생 결정의 owner는 view-model factory입니다. Renderer는 이미 resolved project objects, link lists, metric values와 fallback result를 받습니다. Test fixture가 time을 주입해 `currentYear` 계산의 비결정성을 제거합니다. |
| Failure·absence·fallback·cleanup | Unknown project detail은 `null`, resume의 unknown project ID는 omission, contact는 preferred list가 비면 contact-placement links로 fallback합니다. Supporting image는 cover image와 중복되지 않아야 합니다. 이 서로 다른 absence policy를 하나의 generic rule로 뭉개지 않습니다. |
| Test technique와 실행 증거 | Pure factory characterization입니다. Production factories를 직접 호출하고 object/result ordering을 비교합니다. DOM이나 route page를 render하지 않습니다. |
| 보장하는 것 | Home/projects/project-detail/about/resume/contact의 주요 derivation과 missing-reference policy가 renderer 이전 단계에서 재현 가능하게 고정됩니다. |
| 보장하지 않는 것 | Journey와 interview-map route는 아직 이 test matrix에 없고, route 모델이 허용된 source field만 노출하는지도 아직 검증하지 않습니다. Type-level broad content leakage는 통과할 수 있습니다. |
| 다음 commit/관련 test 연결 | `527b9f872333`가 journey/interview를 추가하고, route별 허용 source field와 full-content spread 금지를 structural regression으로 강화합니다. |

#### 최소 code evidence

- **Commit:** `b77b386b344e60e1aa2ed3eafd76ab5dafb32342`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `src/lib/portfolio/view-models.test.ts`의 `createHomeViewModel`, `createProjectIndexViewModel`, `createProjectDetailViewModel`; `createAboutViewModel`, `createResumeViewModel`, `createContactViewModel`; Home의 fixed date, featured/lead fallback, hero/footer placement, metric derivation

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 4. `527b9f872333` — test(content): scoped view model과 연락처 회귀 검증

- **Full SHA:** `527b9f872333cbd45f6ab436a7d0e6178ccba6d3`
- **Importance:** A
- **Tags:** CONTENT, VALIDATION, TEST
- **이 Thread에서의 역할:** 여덟 route의 scoped field whitelist와 unresolved-reference hardening

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/view-models.test.ts`의 route별 `sourceFields` whitelist
- 공통 `site`, `profile`, `presentation`, `footerLinks`와 route-specific field 비교
- `readFileSync`로 `src/lib/portfolio/view-models.ts`를 읽어 `PortfolioContent &` 및 `...content`를 거부하는 structural assertion
- projects/about model의 `contact` 전달
- Journey milestone의 missing anchor omission과 interview answer의 `{ projectId, project: null }` 유지

#### Commit-specific investigation

- `527b9f872333^`와 `527b9f872333`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `여덟 route의 scoped field whitelist와 unresolved-reference hardening`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: 이 commit으로 content contract Thread가 route projection ownership까지 닫히며, `1598a...`·`055b...`의 renderer matrix가 같은 production factories의 최종 consumer를 검증합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 여섯 factory의 값 규칙은 테스트됐지만 route model이 시간이 지나며 전체 `PortfolioContent`를 intersection 또는 spread로 다시 노출하는 것을 막지 못했습니다. Journey/interview-map과 projects/about contact data도 기존 matrix 밖이었습니다. |
| 실제 변경 file/symbol/call path | Test matrix를 여덟 route model로 확장하고 각 모델이 source aggregate에서 노출해도 되는 field 이름을 whitelist로 비교합니다. Production source text를 읽어 broad type intersection과 object spread의 두 구체적 회귀 패턴을 거부합니다. Journey와 interview reference resolution 및 contact 전달도 추가합니다. |
| Data/state/DOM/resource owner와 lifetime | Shared shell data는 각 view model이 공통으로 전달하지만, route-specific aggregate slice와 ID resolution은 각 factory가 소유합니다. Journey는 존재하는 project object만 `anchorProjects`에 넣고, interview는 원래 `projectId`를 증거로 보존하면서 resolved object를 `null`로 둡니다. |
| Failure·absence·fallback·cleanup | Missing reference가 route마다 다른 의미를 갖습니다. Journey presentation은 깨진 anchor를 렌더 목록에서 제거하고, interview evidence는 unresolved ID 자체가 진단 정보이므로 삭제하지 않습니다. Structural test는 broad spread를 failure로 만들지만 semantic alias나 helper를 통한 우회까지 탐지하지는 못합니다. |
| Test technique와 실행 증거 | Runtime factory assertions과 source-text structural regression을 결합합니다. Test는 production view-model functions와 production source file을 모두 대상으로 합니다. |
| 보장하는 것 | 여덟 route의 permitted source exposure, 공통 shell field, contact propagation, journey/interview absence semantics가 고정됩니다. |
| 보장하지 않는 것 | TypeScript compiler 수준의 완전한 information-flow proof는 아닙니다. Regex는 정확히 `PortfolioContent &`와 `...content` 패턴만 막고, renderer가 받은 field를 올바르게 표현하는지는 route/render/browser Thread의 책임입니다. |
| 다음 commit/관련 test 연결 | 이 commit으로 content contract Thread가 route projection ownership까지 닫히며, `1598a...`·`055b...`의 renderer matrix가 같은 production factories의 최종 consumer를 검증합니다. |

#### 최소 code evidence

- **Commit:** `527b9f872333cbd45f6ab436a7d0e6178ccba6d3`
- **Path:** `src/lib/portfolio/view-models.test.ts`
- **Location:** `does not build route models by spreading the full content object`

```ts
expect(source).not.toMatch(/PortfolioContent\s*&/);
expect(source).not.toMatch(/\.\.\.content\b/);
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

## 6. Invariant evolution ledger

| Invariant | 도입/변경 SHA | Historical evidence | 상태 |
| --- | --- | --- | --- |
| Production content path가 executable contract의 source다. | 3353032ba23b | `portfolio.test.ts`가 loader, asset validator와 selectors를 직접 호출 | 도입 |
| Mutable aggregate는 호출 간 격리된다. | dc07871c4d24 | projects/items/nested links는 fresh reference, site/profile/presentation/journey는 shared | 강화 |
| Renderer는 full content가 아니라 route projection을 받는다. | b77b386b344e → 527b9f872333 | factory tests, source-spread guard와 permitted source field table | 도입 후 범위 축소 |
| Unknown references는 route별 explicit absence로 처리된다. | b77b386b344e → 527b9f872333 | missing detail→null, resume/journey unknown ID omission, interview answer project→null | 확장·고정 |

## 7. Failure → Fix → Test

| Earlier failure/risk | Fix SHA | Corrected decision | Regression evidence |
| --- | --- | --- | --- |
| Malformed ID/cross-reference/asset가 정상 content처럼 통과할 위험 | 3353032ba23b | Production validation path에 변형 fixture를 넣고 `PortfolioContentError` 관찰 | Content contract tests |
| 호출자가 returned project/link를 mutation해 다음 요청을 오염할 위험 | dc07871c4d24 | Mutable nested collections의 reference identity를 두 호출 사이 비교 | Clone-boundary regression |
| Renderer가 full aggregate를 받아 local search/spread를 반복할 위험 | b77b386b344e → 527b9f872333 | Route factory result와 source-text guard로 data scope 제한 | View-model regression |
| Unknown project reference가 crash 또는 stale object로 흘러갈 위험 | b77b386b344e → 527b9f872333 | null/omission policy를 route별 assertion으로 고정 | Boundary tests |

## 8. Ownership/state/responsibility 변화

| 대상 | 초기 owner/state | 최종 owner/state | Evidence |
| --- | --- | --- | --- |
| Raw JSON/source | Content files와 loader | 변경 없음 | `loadPortfolioSource` |
| Validation/derived aggregate | Validator와 selector helpers | 변경 없음 | `validatePortfolioAssets`, `getPortfolioContent` |
| Mutable returned data | 암묵적 | 호출별 cloned projects/links | `getPortfolioContent()` identity assertions |
| Route-specific projection | Renderer/local search 가능 | View-model factories | `create*ViewModel` |
| Regression evidence | 없음 | Vitest suites | `portfolio.test.ts`, `view-models.test.ts` |

## 9. 최종 Thread state

다음 내용을 코드 없이 설명합니다: 최종 owner, input→decision→output, failure/absence policy, regression evidence와 명시적 non-guarantee.

Thread 종료 시 content source는 production loader/validator를 거쳐 aggregate가 되고, caller가 수정할 수 있는 project/link graph는 호출마다 격리됩니다. Public module surface와 route factory가 사용할 수 있는 field가 test로 제한되며, unresolved references는 route별 null 또는 omission policy로 변환됩니다. Browser rendering은 이 Thread의 보장 범위가 아닙니다.

## 10. 최종 실행 흐름

| 단계 | Owner / mechanism | Input | Output/state | Failure/non-guarantee |
| ---: | --- | --- | --- | --- |
| 1. Source load | `loadPortfolioSource` | JSON source | raw source | loader/parse error |
| 2. Validation/aggregate | `validatePortfolioAssets`·`getPortfolioContent` | raw source | validated aggregate/fresh mutable graph | `PortfolioContentError` |
| 3. Public selection | `portfolio` facade selectors | aggregate | placement/order/status-derived values | empty/unknown policy |
| 4. Route projection | `create*ViewModel` | aggregate + route key/date | shared shell + route-specific model | null/omitted unresolved reference |
| 5. Regression evidence | `portfolio.test.ts`·`view-models.test.ts` | production functions + deterministic fixtures | assertions | test failure |

## 11. 학습 완료 확인

- [x] 모든 referenced SHA를 exact historical diff 기준으로 설명했습니다.
- [x] Commit map의 SHA·순서·subject·importance·tags를 변경하지 않았습니다.
- [x] Fix를 earlier failure/assumption에 연결했습니다.
- [x] Test가 실행하는 production path와 증명하지 않는 범위를 구분했습니다.
- [x] 정적 inspection과 실제 command execution을 구분했습니다.
- [x] Thread-level invariant, ownership과 final flow를 완성했습니다.
- [x] 실행 제한을 명시했습니다: 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다.
===== END FILE: 01-content-contract-test-harness.md =====

===== BEGIN FILE: 02-route-presentation-characterization.md =====
# Thread: Route presentation characterization

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `07-testing-performance-and-regression-strategy`
>
> Phase 1에서 감사·수정한 뒤 동결한 scaffold를 기준으로 합니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance와 tags는 branch-local `commit/commit-importance.md` 분류와 exact commit metadata를 기준으로 고정했습니다.
- 이 문서의 Thread goal, commit grouping과 source-defined 역할은 Phase 1 category audit 결과입니다.
- Phase 2에서는 SHA, 순서, subject, importance, tags, 역할, 질문과 문서 구조를 바꾸지 않습니다.
- 다른 branch 또는 final HEAD의 구현을 earlier SHA 설명에 소급하지 않습니다.
- Runtime evidence는 실제로 실행한 command만 기록하며, 미실행 상태를 통과로 해석하지 않습니다.

## 1. Thread 목표

Hard-coded output snapshot 대신 canonical content와 query policy를 기준으로 App Router pages를 characterization하고, five-design route output과 independent renderer boundary를 jsdom·browser matrices로 확장하는 과정을 복원합니다.

### 동결된 핵심 invariant

- Expected labels, projects와 route enablement는 production content에서 파생하며 test 전용 복제 truth를 만들지 않습니다.
- Missing/unknown/repeated `view`·`debug` query는 route resolver의 explicit fallback/first-value policy를 따릅니다.
- 다섯 design은 public route에서 자신을 식별하는 root boundary와 primary heading을 유지합니다.
- Design/Classic independent renderer marker와 shared design-token vocabulary는 structural tests로 보호됩니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- jsdom page tests가 five-design Home과 Classic route shell에서 무엇을 characterization했는가?
- Playwright matrix가 jsdom이 보지 못하는 response, viewport, native interaction과 layout risk를 어떻게 추가했는가?
- View-model migration 뒤 five-design compatibility를 빠르게 확인하는 renderer matrix는 어떤 route set을 사용했는가?
- Renderer marker와 CSS token source-text test가 실제 pixel/behavior 보장과 어떻게 다른가?

## 3. 완료 기준

- 각 referenced SHA의 exact parent diff와 resulting changed files를 확인합니다.
- Commit별 previous state, implementation decision, ownership/lifetime, failure path와 non-guarantee를 구분합니다.
- Fix는 earlier assumption과 root cause에 연결하고, test는 production path·technique·proves/does-not-prove를 구분합니다.
- A-level은 subsystem·failure·verification 관계까지, B-level은 local role과 후속 연결까지만 설명합니다.
- Thread-level invariant evolution, Failure → Fix → Test, ownership transfer와 final flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `42bef4e5783c` | test(routes): 홈과 route presentation 계약 검증 | A | ARCH, VALIDATION, ROUTING | Canonical content 기반 jsdom Home/public-route characterization |
| 2 | `31c438b52e4b` | test(e2e): 다섯 디자인의 route matrix 검증 | A | ARCH, VALIDATION, ROUTING | Playwright five-design desktop/mobile public route matrix |
| 3 | `1598a87702f6` | test(design): view model 기반 renderer matrix 검증 | A | ARCH, VALIDATION, RENDERER | View-model consumer를 통과하는 six-route×five-design fast matrix |
| 4 | `055b733cbb7e` | test(design): 독립 renderer와 design token 경계 검증 | A | ARCH, VALIDATION, RENDERER | Journey/interview 확장, independent renderer marker와 token source contract |

## 5. Commit별 학습 기록

각 section은 해당 SHA의 tree와 parent diff만 기준으로 작성합니다. 같은 SHA가 다른 category Thread에 등장하더라도 여기서는 위 역할과 파일 범위만 설명합니다.

### 1. `42bef4e5783c` — test(routes): 홈과 route presentation 계약 검증

- **Full SHA:** `42bef4e5783ce471f04f4e538801a72aa89ddbf6`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, ROUTING
- **이 Thread에서의 역할:** Canonical content 기반 jsdom Home/public-route characterization

#### 해당 SHA에서 확인할 실제 코드

- `src/app/page.test.tsx`의 five-design table, default/unknown design fallback, content-debug query propagation
- `src/app/routes.test.tsx`의 여덟 route Classic shell matrix와 repeated query의 first-value policy
- `src/app/journey/page.test.tsx`의 content-owned shell labels와 milestone vocabulary
- 각 test가 page function을 직접 await한 뒤 Testing Library로 semantic role/heading/link를 찾는 호출 경로
- Hard-coded full DOM snapshot 대신 JSON/content selector에서 expected value를 가져오는 부분

#### Commit-specific investigation

- `42bef4e5783c^`와 `42bef4e5783c`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Canonical content 기반 jsdom Home/public-route characterization`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `31c438b52e4b`가 같은 public surface를 실제 Chromium desktop/mobile route×design matrix로 확장합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 개별 route와 다섯 presentation이 구현돼 있었지만, 같은 canonical content와 query policy를 유지하는지 한곳에서 비교하지 않았습니다. Design별 markup이 달라 string snapshot을 공유하기 어렵고, 특정 route만 debug/view propagation을 빠뜨릴 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | 세 test file을 추가합니다. Home test는 두 원래 presentation의 journey evidence, default Editorial, unknown fallback, 다섯 design root, featured project와 design navigation, debug query preservation을 확인합니다. Routes test는 여덟 page function을 Classic view로 render해 heading, shell marker, source debug note와 design links를 비교합니다. Journey test는 content-owned labels를 확인합니다. |
| Data/state/DOM/resource owner와 lifetime | Query parsing과 presentation 선택은 실제 page/helper가 소유합니다. Test는 `getPortfolioContent()`에서 canonical expected text를 읽고 async page output을 render하는 observer입니다. Expected content를 test-local copy로 재정의하지 않습니다. |
| Failure·absence·fallback·cleanup | Unknown design은 Editorial로 fallback하고, repeated `view`/`debug` 배열은 첫 값을 사용합니다. 최소 한 enabled project가 없으면 characterization fixture가 명시적으로 실패합니다. Disabled route policy 자체는 이 test의 중심이 아닙니다. |
| Test technique와 실행 증거 | jsdom route characterization입니다. Async server page function을 직접 실행해 React output을 semantic query로 검사합니다. Browser network, CSS layout 또는 hydration은 포함하지 않습니다. |
| 보장하는 것 | Home의 five-design dispatch와 주요 public route의 Classic shell/query/content contract가 content source에 연결됩니다. |
| 보장하지 않는 것 | 모든 route×design 조합을 검증하지 않으며, visual equivalence·responsive layout·real navigation·browser accessibility는 보장하지 않습니다. Text/role assertion 밖의 상세 DOM 변경은 통과할 수 있습니다. |
| 다음 commit/관련 test 연결 | `31c438b52e4b`가 같은 public surface를 실제 Chromium desktop/mobile route×design matrix로 확장합니다. |

#### 최소 code evidence

- **Commit:** `42bef4e5783ce471f04f4e538801a72aa89ddbf6`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `src/app/page.test.tsx`의 five-design table, default/unknown design fallback, content-debug query propagation; `src/app/routes.test.tsx`의 여덟 route Classic shell matrix와 repeated query의 first-value policy; `src/app/journey/page.test.tsx`의 content-owned shell labels와 milestone vocabulary

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 2. `31c438b52e4b` — test(e2e): 다섯 디자인의 route matrix 검증

- **Full SHA:** `31c438b52e4b5f87d7e88ce047dd1997aa8ef054`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, ROUTING
- **이 Thread에서의 역할:** Playwright five-design desktop/mobile public route matrix

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `test:e2e` script와 `@playwright/test` dependency
- `playwright.config.ts`의 testDir, timeout, single worker, webServer와 `chromium`/`mobile-chrome` projects
- `tests/e2e/portfolio.spec.ts` 내부의 design IDs, enabled route definitions와 content JSON fixture
- response, design root, heading/content/media, internal href, switcher, invalid view fallback, overflow, reduced-motion, touch/focus assertions
- dev compiler cold-route race를 피하기 위해 worker 수를 1로 둔 이유와 병렬성 비보장

#### Commit-specific investigation

- `31c438b52e4b^`와 `31c438b52e4b`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Playwright five-design desktop/mobile public route matrix`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `1598a87702f6`는 browser보다 빠른 renderer compatibility matrix를 추가하고, `84c71d...`가 이 matrix의 route/design vocabulary를 `site-matrix.ts`로 추출해 Axe suite와 공유합니다. `882a...`는 같은 Playwright projects를 snapshot baseline에 사용합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | jsdom tests는 page output을 확인했지만 실제 browser request, CSS viewport, native disclosure, focus와 image loading을 관찰하지 못했습니다. 또한 all-design coverage가 Home 중심이어서 다른 route의 design dispatch 누락이 남을 수 있었습니다. |
| 실제 변경 file/symbol/call path | Playwright를 설치하고 deterministic local web server를 사용하는 config와 약 489-line browser suite를 추가합니다. Suite는 source JSON에서 enabled route와 expected content를 구성하고, 다섯 design을 desktop/mobile project에서 방문하여 route response와 presentation root를 검사합니다. |
| Data/state/DOM/resource owner와 lifetime | Route availability의 source는 `site.json`, design vocabulary는 presentation/config와 test matrix가 소유합니다. Browser page가 production Next route를 요청하고, Playwright는 network/DOM/layout/focus를 관찰합니다. 이 commit 시점에는 matrix helper가 `portfolio.spec.ts` 내부에 있습니다. |
| Failure·absence·fallback·cleanup | Invalid `view`는 default presentation으로 돌아가야 하고, mobile/desktop에서 horizontal overflow나 unusable touch/focus state가 드러나면 test가 실패합니다. Single worker는 Next development compiler의 cold-route invalidation을 피하지만 concurrency behavior를 검증하지 않습니다. |
| Test technique와 실행 증거 | Broad Playwright integration characterization입니다. 실제 Chromium 계열 page, CSS, native controls와 links를 통과합니다. Visual pixel comparison과 Axe rule engine은 아직 없습니다. |
| 보장하는 것 | 다섯 design이 enabled public routes에서 response·root·핵심 content/navigation을 유지하고 desktop/mobile browser에서 기본 interaction/layout contract를 충족하는지 한 suite에서 확인할 수 있습니다. |
| 보장하지 않는 것 | Cross-browser 보장, exact visual baseline, WCAG 전체, production build artifact, disabled route와 모든 dynamic data state는 보장하지 않습니다. 한 worker이므로 병렬 route compilation 회귀도 탐지하지 않습니다. |
| 다음 commit/관련 test 연결 | `1598a87702f6`는 browser보다 빠른 renderer compatibility matrix를 추가하고, `84c71d...`가 이 matrix의 route/design vocabulary를 `site-matrix.ts`로 추출해 Axe suite와 공유합니다. `882a...`는 같은 Playwright projects를 snapshot baseline에 사용합니다. |

#### 최소 code evidence

- **Commit:** `31c438b52e4b5f87d7e88ce047dd1997aa8ef054`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `package.json`의 `test:e2e` script와 `@playwright/test` dependency; `playwright.config.ts`의 testDir, timeout, single worker, webServer와 `chromium`/`mobile-chrome` projects; `tests/e2e/portfolio.spec.ts` 내부의 design IDs, enabled route definitions와 content JSON fixture

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 3. `1598a87702f6` — test(design): view model 기반 renderer matrix 검증

- **Full SHA:** `1598a87702f6988102fffbb728ba314db6b08604`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, RENDERER
- **이 Thread에서의 역할:** View-model consumer를 통과하는 six-route×five-design fast matrix

#### 해당 SHA에서 확인할 실제 코드

- `src/designs/route-view-models.test.tsx`의 `designIds` literal과 six-route table
- Home/projects/project-detail/about/resume/contact page function을 실제로 호출하는 `renderPage` closures
- 첫 enabled project fixture와 missing fixture failure
- `data-site-design` root와 non-empty `h1` assertion이 잡는 failure와 놓치는 failure

#### Commit-specific investigation

- `1598a87702f6^`와 `1598a87702f6`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `View-model consumer를 통과하는 six-route×five-design fast matrix`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `055b733cbb7e`가 Journey/Interview를 matrix에 넣고 Design/Classic의 독립 renderer marker를 추가 확인합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | View-model factories가 도입된 뒤 모든 renderer가 새 shape를 받을 수 있는지 한 번에 확인하는 test가 없었습니다. TypeScript가 통과해도 route page에서 잘못된 registry dispatch나 renderer runtime throw가 특정 design에만 남을 수 있었습니다. |
| 실제 변경 file/symbol/call path | `src/designs/route-view-models.test.tsx`를 추가해 다섯 design 각각에 대해 six route page output을 render합니다. 각 case는 expected `data-site-design` root와 non-empty `h1`을 요구합니다. |
| Data/state/DOM/resource owner와 lifetime | Route page가 view model 생성과 renderer registry dispatch를 소유합니다. Test는 그 실제 page entry를 호출하므로 renderer를 isolated mock으로 대체하지 않습니다. Fixture project는 canonical content의 첫 enabled item입니다. |
| Failure·absence·fallback·cleanup | Enabled project가 없으면 suite 초기화가 명시적으로 실패합니다. Renderer가 throw하거나 expected design root/h1을 만들지 않으면 해당 matrix case가 실패합니다. |
| Test technique와 실행 증거 | Fast integration/compatibility matrix입니다. jsdom에서 30 combinations를 통과시키되 assertion은 intentionally shallow합니다. |
| 보장하는 것 | View-model migration 이후 six public route가 다섯 renderer에서 최소 HTML boundary와 primary heading을 유지함을 검증합니다. |
| 보장하지 않는 것 | 내용의 정확성, landmark, links, responsive CSS, browser-only behavior와 Journey/Interview routes는 아직 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `055b733cbb7e`가 Journey/Interview를 matrix에 넣고 Design/Classic의 독립 renderer marker를 추가 확인합니다. |

#### 최소 code evidence

- **Commit:** `1598a87702f6988102fffbb728ba314db6b08604`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `src/designs/route-view-models.test.tsx`의 `designIds` literal과 six-route table; Home/projects/project-detail/about/resume/contact page function을 실제로 호출하는 `renderPage` closures; 첫 enabled project fixture와 missing fixture failure

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 4. `055b733cbb7e` — test(design): 독립 renderer와 design token 경계 검증

- **Full SHA:** `055b733cbb7e897df7c75c90164b99f5fd2d9724`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, RENDERER
- **이 Thread에서의 역할:** Journey/interview 확장, independent renderer marker와 token source contract

#### 해당 SHA에서 확인할 실제 코드

- `src/designs/route-view-models.test.tsx`에 Journey/Interview page를 추가한 부분
- Design/Classic에서만 `data-route-renderer`를 요구하는 branch와 `data-site-design` 공통 assertion
- `src/designs/design-tokens.test.ts`의 seven token-family list
- `src/app/globals.css`를 읽는 source-text test와 Design/Classic scope 안의 `--content-width` regex
- 이 Thread에서는 renderer independence 증거와 token test의 역할을 구분할 것

#### Commit-specific investigation

- `055b733cbb7e^`와 `055b733cbb7e`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Journey/interview 확장, independent renderer marker와 token source contract`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: Visual Thread에서는 이 SHA가 snapshot보다 먼저 오는 token/boundary prerequisite로 다시 사용됩니다. 이후 `882a2f...`가 실제 pixels를 baseline으로 고정합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 기존 matrix는 six routes만 포함하고 Design과 Classic이 같은 presentation implementation으로 다시 합쳐져도 `data-site-design`만 맞으면 통과할 수 있었습니다. Journey/Interview coverage와 최소 token vocabulary도 별도 guard가 없었습니다. |
| 실제 변경 file/symbol/call path | Renderer matrix에 Journey와 Interview Map을 추가하여 40 combinations로 확장하고, Design/Classic root 아래 각각 `data-route-renderer` marker를 요구합니다. 별도 `design-tokens.test.ts`는 typography, spacing, breakpoint, motion, layer와 width token family가 존재하고 Design/Classic scope가 explicit한지 확인합니다. |
| Data/state/DOM/resource owner와 lifetime | Page/registry가 design dispatch를 소유하고 각 independent renderer가 marker를 출력합니다. CSS token 값의 owner는 `globals.css`; test는 source를 읽어 vocabulary와 explicit scope만 고정합니다. |
| Failure·absence·fallback·cleanup | Journey/Interview renderer 누락, Design/Classic implementation boundary collapse, 필수 token 선언 삭제를 deterministic failure로 만듭니다. Regex는 CSS cascade의 computed result나 token 값의 적절성을 평가하지 않습니다. |
| Test technique와 실행 증거 | jsdom route integration과 static CSS contract를 결합합니다. 이 Thread의 핵심 evidence는 route matrix의 두 추가 route와 `data-route-renderer` assertion입니다. |
| 보장하는 것 | 모든 enabled public route가 five-design dispatch에 참여하고 Design/Classic이 독립 renderer boundary를 유지한다는 최소 계약이 완성됩니다. |
| 보장하지 않는 것 | Editorial/Brutalist/Cinematic에 독립 marker를 요구하지 않고, token value·contrast·responsive behavior·pixel output은 보장하지 않습니다. Static source가 존재해도 unused token일 수 있습니다. |
| 다음 commit/관련 test 연결 | Visual Thread에서는 이 SHA가 snapshot보다 먼저 오는 token/boundary prerequisite로 다시 사용됩니다. 이후 `882a2f...`가 실제 pixels를 baseline으로 고정합니다. |

#### 최소 code evidence

- **Commit:** `055b733cbb7e897df7c75c90164b99f5fd2d9724`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `src/designs/route-view-models.test.tsx`에 Journey/Interview page를 추가한 부분; Design/Classic에서만 `data-route-renderer`를 요구하는 branch와 `data-site-design` 공통 assertion; `src/designs/design-tokens.test.ts`의 seven token-family list

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

## 6. Invariant evolution ledger

| Invariant | 도입/변경 SHA | Historical evidence | 상태 |
| --- | --- | --- | --- |
| Route expectation은 canonical content에서 파생한다. | 42bef4e5783c | `getPortfolioContent()` 기반 heading/labels/projects | 도입 |
| Five-design dispatch와 query fallback이 route output에서 유지된다. | 42bef4e5783c → 31c438b52e4b | jsdom query tests와 browser route matrix | 도입 후 browser 확장 |
| View-model migration은 all-design HTML boundary를 보존한다. | 1598a87702f6 → 055b733cbb7e | route-view-models matrix가 6→8 routes로 확장 | 확장 |
| Design/Classic은 독립 renderer/token scope를 갖는다. | 055b733cbb7e | `data-route-renderer`, token-family/source-scope tests | 고정 |

## 7. Failure → Fix → Test

| Earlier failure/risk | Fix SHA | Corrected decision | Regression evidence |
| --- | --- | --- | --- |
| Unknown view가 empty page 또는 arbitrary renderer로 흐를 위험 | 42bef4e5783c | Editorial fallback assertion | jsdom characterization |
| Repeated query array를 잘못 해석해 design/debug state가 달라질 위험 | 42bef4e5783c | 첫 query value assertion | Route test |
| 특정 route/design만 renderer migration에서 누락될 위험 | 31c438b52e4b → 1598a87702f6 → 055b733cbb7e | Browser matrix + fast 5×8 matrix | Integration regression |
| Shared CSS refactor가 required token/independent scope를 제거할 위험 | 055b733cbb7e | CSS source token-family/scope assertions | Structural regression |

## 8. Ownership/state/responsibility 변화

| 대상 | 초기 owner/state | 최종 owner/state | Evidence |
| --- | --- | --- | --- |
| Query normalization | Page마다 직접 해석 가능 | Route resolver contract | `view`/`debug` first value and fallback |
| Expected content | Hard-coded text 가능 | Canonical content aggregate | `getPortfolioContent`/JSON fixtures |
| Design dispatch | Home 중심 | All public route pages/registry | `data-site-design` |
| Browser matrix vocabulary | Spec 내부 | 초기에는 `portfolio.spec.ts` 내부 | 후속 category Thread에서 `site-matrix.ts`로 추출 |
| Renderer/token boundary | 암묵적 CSS/markup | Explicit marker와 token tests | `data-route-renderer`, `design-tokens.test.ts` |

## 9. 최종 Thread state

다음 내용을 코드 없이 설명합니다: 최종 owner, input→decision→output, failure/absence policy, regression evidence와 명시적 non-guarantee.

최종적으로 page tests는 canonical content와 query resolver를 기준으로 route output을 설명하고, Playwright는 실제 desktop/mobile browser에서 five-design public route matrix를 통과합니다. View-model migration은 별도 fast matrix로 보호되고 Design/Classic의 independent renderer marker와 shared token vocabulary가 structural contract가 됩니다. Exact pixels와 WCAG는 다른 Threads가 보완합니다.

## 10. 최종 실행 흐름

| 단계 | Owner / mechanism | Input | Output/state | Failure/non-guarantee |
| ---: | --- | --- | --- | --- |
| 1. Query input | App Router page props | `searchParams`/`params` | resolved view/debug/project | missing/unknown/repeated fallback |
| 2. Content/view model | `getPortfolioContent` + route factory | resolved route state | route data | missing project→not-found/null |
| 3. Design dispatch | registry/route page | route data + design ID | design-specific root | default Editorial |
| 4. Renderer output | five design renderers | view model | HTML heading/navigation/content | missing boundary assertion |
| 5. Characterization | Vitest + Playwright matrices | canonical expectations + rendered page | DOM/browser assertions | contract failure |

## 11. 학습 완료 확인

- [x] 모든 referenced SHA를 exact historical diff 기준으로 설명했습니다.
- [x] Commit map의 SHA·순서·subject·importance·tags를 변경하지 않았습니다.
- [x] Fix를 earlier failure/assumption에 연결했습니다.
- [x] Test가 실행하는 production path와 증명하지 않는 범위를 구분했습니다.
- [x] 정적 inspection과 실제 command execution을 구분했습니다.
- [x] Thread-level invariant, ownership과 final flow를 완성했습니다.
- [x] 실행 제한을 명시했습니다: 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다.
===== END FILE: 02-route-presentation-characterization.md =====

===== BEGIN FILE: 03-component-interaction-and-hydration-regression.md =====
# Thread: Component interaction and hydration regression

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `07-testing-performance-and-regression-strategy`
>
> Phase 1에서 감사·수정한 뒤 동결한 scaffold를 기준으로 합니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance와 tags는 branch-local `commit/commit-importance.md` 분류와 exact commit metadata를 기준으로 고정했습니다.
- 이 문서의 Thread goal, commit grouping과 source-defined 역할은 Phase 1 category audit 결과입니다.
- Phase 2에서는 SHA, 순서, subject, importance, tags, 역할, 질문과 문서 구조를 바꾸지 않습니다.
- 다른 branch 또는 final HEAD의 구현을 earlier SHA 설명에 소급하지 않습니다.
- Runtime evidence는 실제로 실행한 command만 기록하며, 미실행 상태를 통과로 해석하지 않습니다.

## 1. Thread 목표

Native `<details>` design switcher의 초기 client ownership부터 close/focus interaction, pre-hydration race, deterministic regression test와 server/client boundary 축소까지 복원합니다.

### 동결된 핵심 invariant

- Hydration 전 `details.open`은 native browser state이며 React가 초기 server expectation으로 덮어쓰지 않습니다.
- Explicit close는 `open`을 제거하고 trigger summary로 focus를 복원합니다.
- Static selector markup/content/href는 server component가 소유하고 client code는 explicit close action으로 제한됩니다.
- Behavior tests와 source-boundary test가 각각 interaction 결과와 ownership architecture를 보호합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 selector가 왜 전체 client component와 두 DOM ref를 가졌고 어떤 기능이 아직 없었는가?
- Close button과 design link는 native open state와 focus를 어떻게 바꿨는가?
- Server paint 뒤 hydration 전 사용자가 disclosure를 열면 어떤 mismatch가 생기며 어떻게 재현했는가?
- Server markup 전환은 refs/link handlers를 어디서 제거하고 어떤 client island만 남겼는가?

## 3. 완료 기준

- 각 referenced SHA의 exact parent diff와 resulting changed files를 확인합니다.
- Commit별 previous state, implementation decision, ownership/lifetime, failure path와 non-guarantee를 구분합니다.
- Fix는 earlier assumption과 root cause에 연결하고, test는 production path·technique·proves/does-not-prove를 구분합니다.
- A-level은 subsystem·failure·verification 관계까지, B-level은 local role과 후속 연결까지만 설명합니다.
- Thread-level invariant evolution, Failure → Fix → Test, ownership transfer와 final flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `e43e8addd7f3` | feat(designs): 디자인 선택기 상태와 trigger 추가 | B | RENDERER | Client selector skeleton, native trigger와 DOM refs |
| 2 | `c69ef85c98b2` | feat(designs): 디자인 선택 목록과 닫기 동작 추가 | A | ARCH, RENDERER | Design list, route links, close와 focus restoration 구현 |
| 3 | `09cec616f314` | test(ui): 디자인 선택과 프로젝트 링크 계약 검증 | A | VALIDATION, TEST | Component interaction와 project-link policy regression tests |
| 4 | `c702b870d57a` | fix(ui): hydration 중 native details 상태 보존 | A | DEBUG | Pre-hydration native `open` mismatch 허용 fix |
| 5 | `b6c0238ab8b8` | test(ui): details hydration 경쟁 조건 검증 | A | VALIDATION, TEST | SSR→native mutation→hydrate deterministic race test |
| 6 | `a37cb8596733` | refactor(ui): 디자인 선택기를 server markup으로 전환 | A | ARCH, REFACTOR | Static markup server ownership과 close-only client island |
| 7 | `1ac7813155c6` | test(ui): server 선택기와 focus 복원 검증 | A | VALIDATION, A11Y, TEST | Server component source boundary와 focus behavior 최종 regression |

## 5. Commit별 학습 기록

각 section은 해당 SHA의 tree와 parent diff만 기준으로 작성합니다. 같은 SHA가 다른 category Thread에 등장하더라도 여기서는 위 역할과 파일 범위만 설명합니다.

### 1. `e43e8addd7f3` — feat(designs): 디자인 선택기 상태와 trigger 추가

- **Full SHA:** `e43e8addd7f39e003f52d26e2cbda902f7dd0471`
- **Importance:** B
- **Tags:** RENDERER
- **이 Thread에서의 역할:** Client selector skeleton, native trigger와 DOM refs

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/design-switcher.tsx`의 새 `DesignSwitcher` client component
- `detailsRef`·`summaryRef`, `SITE_DESIGNS`, `templateCopy`, `activeIndex`·`active` fallback
- `designSwitcherAriaTemplate`와 `designSwitcherCountTemplate`로 만드는 접근성 label/count
- resulting JSX가 이 SHA에서는 `<details>`와 `<summary>`까지만 포함하고 선택 목록·닫기 동작은 아직 없다는 점

#### Commit-specific investigation

- `e43e8addd7f3^`와 `e43e8addd7f3`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 이 B-level commit은 `Client selector skeleton, native trigger와 DOM refs`의 local 기반만 설명하고, 후속 commit이 완성하는 기능을 이 시점에 소급하지 않습니다.
- DOM/data owner와 아직 존재하지 않는 behavior를 구분하고 다음 후속 관계만 확인합니다: `c69ef85c98b2`가 이 skeleton에 selection list와 닫기 동작을 연결합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 직전 상태에는 selectable design registry와 URL helper가 있었지만, 공용 UI에서 현재 디자인을 표시하고 native disclosure를 여는 전용 component가 없었습니다. |
| 실제 변경 file/symbol/call path | `"use client"` component를 새로 만들고 `HTMLDetailsElement`와 summary를 위한 두 ref, 현재 design lookup, content-owned label/count 계산, native `<details>/<summary>` trigger를 추가했습니다. 이 SHA에서는 import된 `Link`와 URL helper가 아직 selection list로 이어지지 않습니다. |
| Data/state/DOM/resource owner와 lifetime | 브라우저가 `open` 상태를 소유하는 native `<details>`가 disclosure state의 owner이고 React component는 두 DOM ref를 보유합니다. Active design의 fallback은 `SITE_DESIGNS[activeIndex] ?? SITE_DESIGNS[0]`가 결정합니다. |
| Failure·absence·fallback·cleanup | Unknown `activeId`는 첫 registry item으로 fallback하지만, 빈 registry는 방어하지 않습니다. 선택 목록이나 explicit close path가 아직 없고 hydration mismatch도 처리하지 않습니다. |
| 보장/비보장과 후속 연결 | 현재 design의 label/count를 content template에서 만들고 keyboard-operable native summary를 렌더링하는 최소 trigger가 생깁니다. 실제 design 전환, active link semantics, close/focus 복원, outside interaction, hydration race와 browser-level 접근성은 보장하지 않습니다. `c69ef85c98b2`가 이 skeleton에 selection list와 닫기 동작을 연결합니다. |

#### 최소 code evidence

- **Commit:** `e43e8addd7f39e003f52d26e2cbda902f7dd0471`
- **Path:** `src/components/portfolio/design-switcher.tsx`
- **Location:** DesignSwitcher 반환부

```tsx
<details className={styles.root} ref={detailsRef}>
  <summary
    aria-label={ui.designSwitcherAriaTemplate.replace(
      "{label}",
      activeLabel,
    )}
    ref={summaryRef}
  >
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 2. `c69ef85c98b2` — feat(designs): 디자인 선택 목록과 닫기 동작 추가

- **Full SHA:** `c69ef85c98b2659d5f014505964e3103e435fbb8`
- **Importance:** A
- **Tags:** ARCH, RENDERER
- **이 Thread에서의 역할:** Design list, route links, close와 focus restoration 구현

#### 해당 SHA에서 확인할 실제 코드

- `DesignSwitcher`의 새 `<nav>`, sheet header, close `<button>`, design `<ul>`
- close button의 `detailsRef.current?.removeAttribute("open")`와 `summaryRef.current?.focus()`
- `SITE_DESIGNS.map`의 `aria-current`, palette swatch, content label/description, ordinal
- `getTemplateHref(currentPath, design.id, { contentDebug })`와 link `onClick` close path

#### Commit-specific investigation

- `c69ef85c98b2^`와 `c69ef85c98b2`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Design list, route links, close와 focus restoration 구현`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `09cec616f314`가 jsdom interaction contract를 추가하고, `c702b870d57a`가 native state와 hydration 사이의 mismatch를 수정합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 초기 trigger는 disclosure를 열 수는 있었지만 선택 가능한 design link, 현재 항목 semantics, explicit close와 focus 복원이 없었습니다. |
| 실제 변경 file/symbol/call path | 모든 registered design을 source order로 렌더링하는 navigation panel을 추가했습니다. Close button은 native `open` attribute를 제거하고 summary로 focus를 되돌리며, design link click도 이동 전에 disclosure를 닫습니다. URL은 current path와 debug state를 보존합니다. |
| Data/state/DOM/resource owner와 lifetime | Registry가 design 순서와 palette를 소유하고 presentation content가 label/description을 소유합니다. Component의 refs가 DOM mutation과 focus restoration을 직접 수행하며, Next `Link`가 route transition을 수행합니다. |
| Failure·absence·fallback·cleanup | Unknown active design은 앞 commit과 동일하게 첫 item으로 fallback합니다. Close button은 ref가 없으면 optional chaining으로 no-op합니다. Link click close는 navigation이 실제로 완료되는지와 독립이며, hydration 전 사용자 토글은 아직 고려하지 않습니다. |
| Test technique와 실행 증거 | 상호작용 구현 commit입니다. 이 SHA에는 자동화 test가 없으므로 close/focus와 URL behavior는 code inspection으로만 확인했습니다. |
| 보장하는 것 | 한 component 안에서 disclosure, 현재 design 표시, route-preserving design links, explicit close와 focus restoration이 연결됩니다. |
| 보장하지 않는 것 | Hydration race, real browser focus order, touch target/viewport layout, failed navigation 후 open state, JavaScript 비활성 상태의 전체 경험은 증명하지 않습니다. |
| 다음 commit/관련 test 연결 | `09cec616f314`가 jsdom interaction contract를 추가하고, `c702b870d57a`가 native state와 hydration 사이의 mismatch를 수정합니다. |

#### 최소 code evidence

- **Commit:** `c69ef85c98b2659d5f014505964e3103e435fbb8`
- **Path:** `src/components/portfolio/design-switcher.tsx`
- **Location:** explicit close button handler

```tsx
onClick={() => {
  detailsRef.current?.removeAttribute("open");
  summaryRef.current?.focus();
}}
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 3. `09cec616f314` — test(ui): 디자인 선택과 프로젝트 링크 계약 검증

- **Full SHA:** `09cec616f3147524e078c59b608ba899c4c368d7`
- **Importance:** A
- **Tags:** VALIDATION, TEST
- **이 Thread에서의 역할:** Component interaction와 project-link policy regression tests

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/design-switcher.test.tsx`의 `DesignSwitcher` suite
- `screen.getByLabelText`, hidden navigation query, `within`, `fireEvent`, 직접 설정한 `details[open]`
- close button 후 `open` 제거·summary focus, prevented navigation 상태에서 design link click 후 close
- `src/components/portfolio/project-links.test.tsx`의 source order, template/debug query, external target/rel, placement/offline/case-study filtering, empty wrapper cases

#### Commit-specific investigation

- `09cec616f314^`와 `09cec616f314`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Component interaction와 project-link policy regression tests`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `c702b870d57a`의 hydration fix 이전 기준선이며, `b6c0238ab8b8`가 이 suite에 실제 SSR→hydrate race를 추가합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Interaction code는 존재했지만 ref 기반 close/focus와 link selection policy가 refactor 중 깨져도 이를 즉시 감지할 test가 없었습니다. |
| 실제 변경 file/symbol/call path | Testing Library로 production components를 직접 render합니다. Switcher test는 content-provided copy와 count, explicit close, focus restoration, link click close를 검사합니다. Project-link tests는 synthetic project fixture로 source order, internal/external transport, deployment/placement filtering과 zero-result omission을 검사합니다. |
| Data/state/DOM/resource owner와 lifetime | Test가 DOM fixture와 synthetic project object를 소유합니다. 상태 전이는 native `open` attribute를 test가 준비하고 production handler가 제거합니다. Production selectors/components가 filtering과 href construction을 계속 소유합니다. |
| Failure·absence·fallback·cleanup | Navigation 자체는 capture listener로 `preventDefault`하여 jsdom을 떠나지 않고 close만 관찰합니다. 따라서 route transition 성공/실패는 검사하지 않습니다. Empty result는 wrapper 자체가 없어야 한다는 explicit policy로 고정됩니다. |
| Test technique와 실행 증거 | Deterministic component regression testing입니다. Real browser hydration이나 layout이 아니라 jsdom DOM state와 accessible queries를 사용합니다. Exact tests를 확인했지만 command는 실행하지 않았습니다. |
| 보장하는 것 | 디자인 copy/count, active navigation access, explicit close/focus, selection close와 project-link rendering/filtering의 현재 contract를 재현 가능한 test로 표현합니다. |
| 보장하지 않는 것 | Pre-hydration native interaction, browser navigation, CSS visibility, pointer/touch behavior, screen-reader output과 network policy는 증명하지 않습니다. |
| 다음 commit/관련 test 연결 | `c702b870d57a`의 hydration fix 이전 기준선이며, `b6c0238ab8b8`가 이 suite에 실제 SSR→hydrate race를 추가합니다. |

#### 최소 code evidence

- **Commit:** `09cec616f3147524e078c59b608ba899c4c368d7`
- **Path:** `src/components/portfolio/design-switcher.test.tsx`
- **Location:** `renders selector copy from content and clears native open state`

```tsx
details?.setAttribute("open", "");
fireEvent.click(closeButton);
expect(details).not.toHaveAttribute("open");
expect(summary).toHaveFocus();
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 4. `c702b870d57a` — fix(ui): hydration 중 native details 상태 보존

- **Full SHA:** `c702b870d57ab7932218a77e40a5d92302a58e79`
- **Importance:** A
- **Tags:** DEBUG
- **이 Thread에서의 역할:** Pre-hydration native `open` mismatch 허용 fix

#### 해당 SHA에서 확인할 실제 코드

- parent `44e4d062da50...`와 이 SHA 사이의 단일-line diff
- `src/components/portfolio/design-switcher.tsx`의 root `<details>`
- `suppressHydrationWarning`이 추가된 정확한 위치와 여전히 남아 있는 client directive/ref ownership
- 후속 `b6c0238ab8b8` test가 재현하는 SSR markup → native `open` mutation → hydration 순서

#### Commit-specific investigation

- `c702b870d57a^`와 `c702b870d57a`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Pre-hydration native `open` mismatch 허용 fix`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `b6c0238ab8b8`가 이 exact race를 SSR/hydrate API로 재현하고, `a37cb8596733`가 더 근본적으로 server markup과 client action을 분리합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Server-rendered `<details>`에는 `open`이 없더라도 hydration 전에 사용자가 summary를 열 수 있습니다. 그 시점의 DOM은 server/React 기대치와 달라지며, 기존 component는 이 native state 변화를 intentional boundary로 선언하지 않았습니다. |
| 실제 변경 file/symbol/call path | Root `<details>`에 `suppressHydrationWarning`을 한 개 추가했습니다. 다른 state logic, refs, close handlers 또는 renderer structure는 바꾸지 않았습니다. |
| Data/state/DOM/resource owner와 lifetime | Hydration 전 `open` attribute는 browser/native control이 소유한다는 판단을 root boundary에 기록합니다. React는 나머지 component를 hydrate하지만 이 root attribute mismatch warning을 억제합니다. |
| Failure·absence·fallback·cleanup | 이 fix는 root의 hydration mismatch를 의도적으로 허용합니다. Descendant mismatch, 잘못된 server markup 또는 다른 hydration error를 일반적으로 숨기는 정책은 추가하지 않습니다. 이 commit 자체에는 test가 없습니다. |
| Test technique와 실행 증거 | Targeted integration fix입니다. 실제 보존 여부는 code 한 줄만으로 runtime evidence라고 할 수 없고, 후속 deterministic hydration test가 그 intended invariant를 확인합니다. |
| 보장하는 것 | Native disclosure가 hydration 전에 바뀔 수 있다는 상태 소유권을 인정하고 해당 root mismatch를 허용합니다. |
| 보장하지 않는 것 | Client component 비용을 제거하지 않고, 모든 hydration mismatch를 해결하지 않으며, user-visible flicker나 browser matrix를 측정하지 않습니다. |
| 다음 commit/관련 test 연결 | `b6c0238ab8b8`가 이 exact race를 SSR/hydrate API로 재현하고, `a37cb8596733`가 더 근본적으로 server markup과 client action을 분리합니다. |

#### 최소 code evidence

- **Commit:** `c702b870d57ab7932218a77e40a5d92302a58e79`
- **Path:** `src/components/portfolio/design-switcher.tsx`
- **Location:** DesignSwitcher root

```tsx
<details
  className={styles.root}
  ref={detailsRef}
  suppressHydrationWarning
>
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 5. `b6c0238ab8b8` — test(ui): details hydration 경쟁 조건 검증

- **Full SHA:** `b6c0238ab8b8de588371c1f23f3170560398883c`
- **Importance:** A
- **Tags:** VALIDATION, TEST
- **이 Thread에서의 역할:** SSR→native mutation→hydrate deterministic race test

#### 해당 SHA에서 확인할 실제 코드

- `renderToString`, DOM container, `details.open = true`, `hydrateRoot`의 순서
- `act` 안의 hydration과 microtask flush
- `vi.spyOn(console, "error")` 및 hydration-related message filtering
- `open` attribute 보존 assertion과 root unmount, spy restore, container removal cleanup

#### Commit-specific investigation

- `b6c0238ab8b8^`와 `b6c0238ab8b8`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `SSR→native mutation→hydrate deterministic race test`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `a37cb8596733`는 test가 보호하는 native root를 유지하면서 bulk markup을 server component로 옮깁니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Fix는 한 줄의 hydration-warning suppression이었지만, 실제 race를 재현하는 regression evidence가 없으면 향후 attribute나 component boundary 변경으로 다시 깨질 수 있었습니다. |
| 실제 변경 file/symbol/call path | Production `DesignSwitcher`를 server string으로 렌더링하고 DOM에 삽입한 뒤 hydration 전에 `details.open = true`로 native state를 바꿉니다. 같은 element를 `hydrateRoot`로 hydrate하여 `open` 보존과 hydration error 부재를 검사합니다. |
| Data/state/DOM/resource owner와 lifetime | Test container와 React root lifetime은 test가 소유합니다. Native DOM이 hydration 전 state를 소유하고 React root가 이후 lifecycle을 이어받습니다. `finally`에서 root unmount, console spy restore, DOM removal을 수행합니다. |
| Failure·absence·fallback·cleanup | Missing `<details>`는 명시적 error입니다. Console error 전체를 무조건 허용하지 않고 hydration/server HTML mismatch 관련 message를 추려 empty를 요구합니다. Test failure 중에도 cleanup이 수행됩니다. |
| Test technique와 실행 증거 | Deterministic hydration regression test입니다. Browser network/paint 없이 React server/client APIs와 jsdom을 사용합니다. Exact code는 확인했지만 실행하지 않았습니다. |
| 보장하는 것 | 해당 component에서 pre-hydration native `open` state가 hydration 뒤에도 남고, test가 정의한 hydration mismatch messages가 발생하지 않아야 한다는 invariant를 고정합니다. |
| 보장하지 않는 것 | Real Chromium의 event timing, streaming SSR, concurrent navigation, CSS transition, 다른 attributes/descendants와 production bundle behavior는 증명하지 않습니다. |
| 다음 commit/관련 test 연결 | `a37cb8596733`는 test가 보호하는 native root를 유지하면서 bulk markup을 server component로 옮깁니다. |

#### 최소 code evidence

- **Commit:** `b6c0238ab8b8de588371c1f23f3170560398883c`
- **Path:** `src/components/portfolio/design-switcher.test.tsx`
- **Location:** `tolerates native open state changed before hydration`

```tsx
container.innerHTML = renderToString(switcher);
details.open = true;
root = hydrateRoot(container, switcher);
expect(details).toHaveAttribute("open");
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 6. `a37cb8596733` — refactor(ui): 디자인 선택기를 server markup으로 전환

- **Full SHA:** `a37cb85967331fcec7d8c7202b4103ba0780947e`
- **Importance:** A
- **Tags:** ARCH, REFACTOR
- **이 Thread에서의 역할:** Static markup server ownership과 close-only client island

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/design-switcher-close.tsx`와 `DesignSwitcherClose`
- `design-switcher.tsx`에서 제거된 `"use client"`, `useRef`, DOM refs와 link `onClick`
- server `DesignSwitcher`가 유지하는 content lookup, href, `<details>`, `suppressHydrationWarning`
- close island의 `event.currentTarget.closest("details")`, direct summary query, open 제거와 focus
- 수정된 component test가 href와 explicit close/focus만 검사하고 link-click close assertion을 제거한 이유

#### Commit-specific investigation

- `a37cb8596733^`와 `a37cb8596733`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Static markup server ownership과 close-only client island`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `1ac7813155c6`가 source boundary를 regression test로 고정하고, Thread 5의 performance tests가 broader client/network behavior를 다룹니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 전체 switcher가 client component여서 static registry/list markup까지 hydration 대상이었고 두 ref가 root와 summary lifetime을 소유했습니다. Link click close도 client handler에 의존했습니다. |
| 실제 변경 file/symbol/call path | Main `DesignSwitcher`에서 client directive와 refs를 제거하여 server component markup으로 전환했습니다. Explicit close button만 `DesignSwitcherClose` client component로 분리하고, handler가 event target에서 가까운 details와 direct summary를 찾아 open 제거/focus 복원을 수행합니다. |
| Data/state/DOM/resource owner와 lifetime | Browser가 native details state를 계속 소유합니다. Server component는 markup/content/href를 소유하고 작은 close island만 explicit DOM mutation을 소유합니다. Design link는 native/Next navigation으로 페이지를 떠나므로 더 이상 별도 close handler를 갖지 않습니다. |
| Failure·absence·fallback·cleanup | Close button이 expected DOM hierarchy 밖에 있으면 closest/query가 null이 되어 optional no-op합니다. Link navigation이 취소되면 disclosure를 강제로 닫지 않는 behavior로 바뀝니다. `suppressHydrationWarning`은 native pre-hydration state를 위해 유지됩니다. |
| Test technique와 실행 증거 | Server-first ownership refactor입니다. Component test는 URL과 close/focus contract를 갱신하지만 이 commit은 client JavaScript byte 수를 측정하지 않습니다. |
| 보장하는 것 | Static selector markup과 registry rendering이 server에서 제공되고, client code는 explicit close action으로 제한됩니다. |
| 보장하지 않는 것 | Zero-JavaScript라고 보장하지 않으며 Next `Link`, close island, hydration warning boundary가 남습니다. Bundle 감소량과 production interaction latency는 별도 성능 evidence가 필요합니다. |
| 다음 commit/관련 test 연결 | `1ac7813155c6`가 source boundary를 regression test로 고정하고, Thread 5의 performance tests가 broader client/network behavior를 다룹니다. |

#### 최소 code evidence

- **Commit:** `a37cb85967331fcec7d8c7202b4103ba0780947e`
- **Path:** `src/components/portfolio/design-switcher-close.tsx`
- **Location:** DesignSwitcherClose click handler

```tsx
const details = event.currentTarget.closest("details");
const summary = details?.querySelector<HTMLElement>(":scope > summary");

details?.removeAttribute("open");
summary?.focus();
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 7. `1ac7813155c6` — test(ui): server 선택기와 focus 복원 검증

- **Full SHA:** `1ac7813155c61e79f9c7ccc736b120f1a2c802a1`
- **Importance:** A
- **Tags:** VALIDATION, A11Y, TEST
- **이 Thread에서의 역할:** Server component source boundary와 focus behavior 최종 regression

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/design-switcher.test.tsx`가 `node:fs/promises.readFile`로 production source를 읽는 방식
- `"use client"`와 `useRef` 부재 assertion
- 기존 hydration race test와 explicit close/focus test가 같은 suite에 계속 남아 있는지
- `package-lock.json`의 side-channel dependency metadata 변경은 UI contract와 독립인 부수 변경이라는 점

#### Commit-specific investigation

- `1ac7813155c6^`와 `1ac7813155c6`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Server component source boundary와 focus behavior 최종 regression`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: 이 Thread의 최종 상태를 고정하며, `dfeb...`·`787...` 같은 browser performance tests가 runtime 비용과 interaction을 별도로 관찰합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Server/client split은 architecture decision이었지만 runtime DOM test만으로는 main file에 client directive/ref가 다시 추가되어도 UI 결과가 같으면 감지하지 못할 수 있었습니다. |
| 실제 변경 file/symbol/call path | Test가 `src/components/portfolio/design-switcher.tsx` source text를 읽어 `"use client"`와 `useRef`가 없어야 한다고 요구합니다. 기존 interaction/hydration tests와 함께 structural ownership과 behavior를 동시에 보호합니다. |
| Data/state/DOM/resource owner와 lifetime | Production server component가 markup을 소유하고 `DesignSwitcherClose`가 client action을 소유한다는 파일 경계를 test가 감시합니다. Test는 source file read lifetime만 소유합니다. |
| Failure·absence·fallback·cleanup | Working directory나 file path가 바뀌면 architecture가 유지돼도 source-text test는 실패할 수 있습니다. 반대로 다른 client-only mechanism을 도입해도 두 문자열을 피하면 이 assertion만으로는 잡지 못합니다. |
| Test technique와 실행 증거 | Static architecture regression plus existing functional tests입니다. Build output 또는 React Flight payload를 분석하는 test가 아닙니다. Exact diff는 확인했지만 command는 실행하지 않았습니다. |
| 보장하는 것 | Main switcher source에 client directive와 `useRef`가 없어야 하고 explicit close/focus와 hydration race tests가 함께 유지되는 contract가 생깁니다. |
| 보장하지 않는 것 | 실제 client bundle 크기, all-transitive-import server safety, runtime latency와 assistive technology behavior는 증명하지 않습니다. |
| 다음 commit/관련 test 연결 | 이 Thread의 최종 상태를 고정하며, `dfeb...`·`787...` 같은 browser performance tests가 runtime 비용과 interaction을 별도로 관찰합니다. |

#### 최소 code evidence

- **Commit:** `1ac7813155c61e79f9c7ccc736b120f1a2c802a1`
- **Path:** `src/components/portfolio/design-switcher.test.tsx`
- **Location:** `keeps the selector markup in a server component`

```tsx
expect(source).not.toContain('"use client"');
expect(source).not.toContain("useRef");
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

## 6. Invariant evolution ledger

| Invariant | 도입/변경 SHA | Historical evidence | 상태 |
| --- | --- | --- | --- |
| Native details가 disclosure state의 base owner다. | e43e8addd7f3 → c702b870d57a | Native element + root hydration-warning boundary | 도입 후 hydration ownership 수정 |
| Explicit close는 summary focus를 복원한다. | c69ef85c98b2 → 09cec616f314 → a37cb8596733 | Ref handler→component test→event-local close island | 도입·검증·owner 축소 |
| Pre-hydration open state는 hydrate 뒤 보존된다. | c702b870d57a → b6c0238ab8b8 | `suppressHydrationWarning` + deterministic hydrate test | 수정·검증 |
| Bulk selector markup은 server component에 남는다. | a37cb8596733 → 1ac7813155c6 | Client directive/ref 제거 + source-text guard | 도입·고정 |

## 7. Failure → Fix → Test

| Earlier failure/risk | Fix SHA | Corrected decision | Regression evidence |
| --- | --- | --- | --- |
| 선택기를 닫은 뒤 focus가 body에 남을 위험 | c69ef85c98b2 → 09cec616f314 | Open 제거 후 summary focus assertion | Component regression |
| Native open과 hydration expectation이 충돌할 위험 | c702b870d57a → b6c0238ab8b8 | Root mismatch boundary + SSR/pre-hydration mutation test | Fix→Test |
| Static list 전체가 불필요한 client ownership으로 회귀할 위험 | a37cb8596733 → 1ac7813155c6 | Server/client split + source guard | Architecture regression |
| Link navigation cancel 상태에서 close semantics를 과도하게 강제할 위험 | a37cb8596733 | Link onClick close 제거, explicit close만 client mutation | Responsibility correction |

## 8. Ownership/state/responsibility 변화

| 대상 | 초기 owner/state | 최종 owner/state | Evidence |
| --- | --- | --- | --- |
| Disclosure `open` state | Native details + client refs | Native details | `<details suppressHydrationWarning>` |
| Static design list/content/href | Client `DesignSwitcher` | Server `DesignSwitcher` | `design-switcher.tsx` |
| Explicit close/focus | Parent refs/inline handler | Local event-derived client island | `DesignSwitcherClose` |
| Route transition close | Link onClick mutation | Navigation lifecycle | Link handler 제거 |
| Regression evidence | 없음 | Component + hydration + source-boundary tests | `design-switcher.test.tsx` |

## 9. 최종 Thread state

다음 내용을 코드 없이 설명합니다: 최종 owner, input→decision→output, failure/absence policy, regression evidence와 명시적 non-guarantee.

최종 선택기는 server component가 native disclosure markup, content와 route links를 출력하고, browser가 hydration 전후 `open` state를 소유합니다. Explicit close만 작은 client component가 closest details/direct summary를 찾아 state를 닫고 focus를 복원합니다. SSR→pre-hydration toggle→hydrate race와 server-source boundary가 tests로 고정되지만 client bundle 크기는 이 Thread가 측정하지 않습니다.

## 10. 최종 실행 흐름

| 단계 | Owner / mechanism | Input | Output/state | Failure/non-guarantee |
| ---: | --- | --- | --- | --- |
| 1. Server markup | `DesignSwitcher` | active design/templates/ui/path | details/summary/nav/links | unknown active→first registry item |
| 2. Native pre-hydration state | Browser `<details>` | summary interaction | `open` attribute | React expectation과 mismatch 가능 |
| 3. Hydration boundary | `suppressHydrationWarning` | server markup + mutated DOM | state-tolerant hydration | descendant mismatch는 별도 |
| 4. Explicit close | `DesignSwitcherClose` | click event | open 제거 + summary focus | unexpected hierarchy→no-op |
| 5. Regression evidence | `design-switcher.test.tsx` | SSR/jsdom/source text | behavior/architecture assertions | test failure |

## 11. 학습 완료 확인

- [x] 모든 referenced SHA를 exact historical diff 기준으로 설명했습니다.
- [x] Commit map의 SHA·순서·subject·importance·tags를 변경하지 않았습니다.
- [x] Fix를 earlier failure/assumption에 연결했습니다.
- [x] Test가 실행하는 production path와 증명하지 않는 범위를 구분했습니다.
- [x] 정적 inspection과 실제 command execution을 구분했습니다.
- [x] Thread-level invariant, ownership과 final flow를 완성했습니다.
- [x] 실행 제한을 명시했습니다: 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다.
===== END FILE: 03-component-interaction-and-hydration-regression.md =====

===== BEGIN FILE: 04-browser-accessibility-route-matrix.md =====
# Thread: Browser accessibility route matrix

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `07-testing-performance-and-regression-strategy`
>
> Phase 1에서 감사·수정한 뒤 동결한 scaffold를 기준으로 합니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance와 tags는 branch-local `commit/commit-importance.md` 분류와 exact commit metadata를 기준으로 고정했습니다.
- 이 문서의 Thread goal, commit grouping과 source-defined 역할은 Phase 1 category audit 결과입니다.
- Phase 2에서는 SHA, 순서, subject, importance, tags, 역할, 질문과 문서 구조를 바꾸지 않습니다.
- 다른 branch 또는 final HEAD의 구현을 earlier SHA 설명에 소급하지 않습니다.
- Runtime evidence는 실제로 실행한 command만 기록하며, 미실행 상태를 통과로 해석하지 않습니다.

## 1. Thread 목표

브라우저 route harness 위에서 design별 contrast, skip-link focusability, definition semantics를 수정하고 shared design×enabled-route Axe/keyboard matrix로 연결하는 과정을 복원합니다.

### 동결된 핵심 invariant

- 각 enabled route/design output에는 하나의 main, banner와 contentinfo landmark가 존재합니다.
- Skip link는 첫 keyboard focus가 되고 activation 뒤 main landmark가 programmatic focus를 받습니다.
- Foreground token은 surface별 contrast 요구에 맞게 분리되며 definition list는 term/value semantics를 유지합니다.
- Accessibility suite와 general E2E suite는 동일한 enabled-route/design matrix를 사용합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 Playwright route matrix가 accessibility automation의 어떤 browser/server foundation을 제공했는가?
- 밝은/어두운 surface에 하나의 color token을 재사용한 가정은 어떻게 수정되었는가?
- Hash target main에 `tabIndex={-1}`가 필요한 이유와 누락된 Brutalist 보강 순서는 무엇인가?
- Axe tags, landmark cardinality와 skip-link keyboard test가 무엇을 증명하고 무엇을 놓치는가?

## 3. 완료 기준

- 각 referenced SHA의 exact parent diff와 resulting changed files를 확인합니다.
- Commit별 previous state, implementation decision, ownership/lifetime, failure path와 non-guarantee를 구분합니다.
- Fix는 earlier assumption과 root cause에 연결하고, test는 production path·technique·proves/does-not-prove를 구분합니다.
- A-level은 subsystem·failure·verification 관계까지, B-level은 local role과 후속 연결까지만 설명합니다.
- Thread-level invariant evolution, Failure → Fix → Test, ownership transfer와 final flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `31c438b52e4b` | test(e2e): 다섯 디자인의 route matrix 검증 | A | ARCH, VALIDATION, ROUTING | Real-browser five-design route/viewport/interaction foundation |
| 2 | `5a6fd8a802ff` | fix(a11y): 디자인별 색상 대비 보정 | A | A11Y, DEBUG | Surface-specific shared/Editorial contrast token correction |
| 3 | `a15e117cb51b` | fix(a11y): skip link focus target 복원 | A | A11Y, DEBUG | Shared/Cinematic/Editorial skip-target focusability |
| 4 | `e1aac08e0e9e` | fix(a11y): Brutalist 지표의 definition semantics 수정 | A | ARCH, RENDERER, A11Y | Brutalist skip target와 metric definition semantics correction |
| 5 | `84c71d027630` | test(a11y): 디자인×route WCAG 행렬 추가 | A | ARCH, ROUTING, A11Y | Shared site matrix, Axe WCAG scan과 keyboard skip-link regression |

## 5. Commit별 학습 기록

각 section은 해당 SHA의 tree와 parent diff만 기준으로 작성합니다. 같은 SHA가 다른 category Thread에 등장하더라도 여기서는 위 역할과 파일 범위만 설명합니다.

### 1. `31c438b52e4b` — test(e2e): 다섯 디자인의 route matrix 검증

- **Full SHA:** `31c438b52e4b5f87d7e88ce047dd1997aa8ef054`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, ROUTING
- **이 Thread에서의 역할:** Real-browser five-design route/viewport/interaction foundation

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `test:e2e` script와 `@playwright/test` dependency
- `playwright.config.ts`의 testDir, timeout, single worker, webServer와 `chromium`/`mobile-chrome` projects
- `tests/e2e/portfolio.spec.ts` 내부의 design IDs, enabled route definitions와 content JSON fixture
- response, design root, heading/content/media, internal href, switcher, invalid view fallback, overflow, reduced-motion, touch/focus assertions
- dev compiler cold-route race를 피하기 위해 worker 수를 1로 둔 이유와 병렬성 비보장

#### Commit-specific investigation

- `31c438b52e4b^`와 `31c438b52e4b`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Real-browser five-design route/viewport/interaction foundation`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `1598a87702f6`는 browser보다 빠른 renderer compatibility matrix를 추가하고, `84c71d...`가 이 matrix의 route/design vocabulary를 `site-matrix.ts`로 추출해 Axe suite와 공유합니다. `882a...`는 같은 Playwright projects를 snapshot baseline에 사용합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | jsdom tests는 page output을 확인했지만 실제 browser request, CSS viewport, native disclosure, focus와 image loading을 관찰하지 못했습니다. 또한 all-design coverage가 Home 중심이어서 다른 route의 design dispatch 누락이 남을 수 있었습니다. |
| 실제 변경 file/symbol/call path | Playwright를 설치하고 deterministic local web server를 사용하는 config와 약 489-line browser suite를 추가합니다. Suite는 source JSON에서 enabled route와 expected content를 구성하고, 다섯 design을 desktop/mobile project에서 방문하여 route response와 presentation root를 검사합니다. |
| Data/state/DOM/resource owner와 lifetime | Route availability의 source는 `site.json`, design vocabulary는 presentation/config와 test matrix가 소유합니다. Browser page가 production Next route를 요청하고, Playwright는 network/DOM/layout/focus를 관찰합니다. 이 commit 시점에는 matrix helper가 `portfolio.spec.ts` 내부에 있습니다. |
| Failure·absence·fallback·cleanup | Invalid `view`는 default presentation으로 돌아가야 하고, mobile/desktop에서 horizontal overflow나 unusable touch/focus state가 드러나면 test가 실패합니다. Single worker는 Next development compiler의 cold-route invalidation을 피하지만 concurrency behavior를 검증하지 않습니다. |
| Test technique와 실행 증거 | Broad Playwright integration characterization입니다. 실제 Chromium 계열 page, CSS, native controls와 links를 통과합니다. Visual pixel comparison과 Axe rule engine은 아직 없습니다. |
| 보장하는 것 | 다섯 design이 enabled public routes에서 response·root·핵심 content/navigation을 유지하고 desktop/mobile browser에서 기본 interaction/layout contract를 충족하는지 한 suite에서 확인할 수 있습니다. |
| 보장하지 않는 것 | Cross-browser 보장, exact visual baseline, WCAG 전체, production build artifact, disabled route와 모든 dynamic data state는 보장하지 않습니다. 한 worker이므로 병렬 route compilation 회귀도 탐지하지 않습니다. |
| 다음 commit/관련 test 연결 | `1598a87702f6`는 browser보다 빠른 renderer compatibility matrix를 추가하고, `84c71d...`가 이 matrix의 route/design vocabulary를 `site-matrix.ts`로 추출해 Axe suite와 공유합니다. `882a...`는 같은 Playwright projects를 snapshot baseline에 사용합니다. |

#### 최소 code evidence

- **Commit:** `31c438b52e4b5f87d7e88ce047dd1997aa8ef054`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `package.json`의 `test:e2e` script와 `@playwright/test` dependency; `playwright.config.ts`의 testDir, timeout, single worker, webServer와 `chromium`/`mobile-chrome` projects; `tests/e2e/portfolio.spec.ts` 내부의 design IDs, enabled route definitions와 content JSON fixture

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 2. `5a6fd8a802ff` — fix(a11y): 디자인별 색상 대비 보정

- **Full SHA:** `5a6fd8a802ffa82bbe615563a864c413ec4264aa`
- **Importance:** A
- **Tags:** A11Y, DEBUG
- **이 Thread에서의 역할:** Surface-specific shared/Editorial contrast token correction

#### 해당 SHA에서 확인할 실제 코드

- `src/app/globals.css`의 `--accent` 값과 dark tooltip text color 변경
- `src/designs/editorial/editorial-route.module.css`의 `--vermilion`, `--vermilion-text`, 새 `--vermilion-on-dark`
- dark section title, architecture evidence, next-review, gaps selectors가 어느 token으로 이동하는지
- 이 commit에 계산된 contrast ratio나 자동 test 결과가 포함되지 않았다는 점

#### Commit-specific investigation

- `5a6fd8a802ff^`와 `5a6fd8a802ff`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Surface-specific shared/Editorial contrast token correction`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `84c71d027630`의 Axe route×design matrix가 resulting pages의 자동 검출 가능한 contrast/semantic violations를 검사하도록 연결됩니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Shared accent와 Editorial vermilion 계열이 여러 surface에서 재사용되었습니다. 특히 밝은 paper용 text token을 dark panel에서도 쓰는 구조는 배경별 대비 요구를 하나의 값으로 만족한다고 가정했습니다. |
| 실제 변경 file/symbol/call path | Shared accent를 더 어두운 값으로 조정하고 dark tooltip text를 밝은 고정색으로 바꿨습니다. Editorial에서는 base vermilion을 조정하고 `--vermilion-on-dark`를 추가하여 dark architecture/curation/gap surfaces의 labels와 evidence text에 별도 적용했습니다. |
| Data/state/DOM/resource owner와 lifetime | 각 renderer stylesheet가 surface-specific color ownership을 가집니다. 밝은 배경 text는 `--vermilion-text`, dark surface text는 `--vermilion-on-dark`가 담당하도록 책임이 분리됩니다. |
| Failure·absence·fallback·cleanup | 이 SHA의 diff에는 ratio calculator, fixture 또는 test가 없습니다. 따라서 수치상 WCAG 통과를 runtime evidence로 주장할 수 없고, 변경되지 않은 selector나 dynamic color 조합은 별도 검증이 필요합니다. |
| Test technique와 실행 증거 | Targeted CSS accessibility fix입니다. Exact property/selector diff를 정적으로 확인했습니다. |
| 보장하는 것 | 문제가 확인된 shared/Editorial foreground-background 조합이 더 이상 같은 약한 token을 공유하지 않도록 code policy가 바뀝니다. |
| 보장하지 않는 것 | 모든 텍스트·state·pseudo-element의 contrast, forced-colors, user stylesheet와 실제 device rendering은 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `84c71d027630`의 Axe route×design matrix가 resulting pages의 자동 검출 가능한 contrast/semantic violations를 검사하도록 연결됩니다. |

#### 최소 code evidence

- **Commit:** `5a6fd8a802ffa82bbe615563a864c413ec4264aa`
- **Path:** `src/designs/editorial/editorial-route.module.css`
- **Location:** Editorial root tokens와 dark section selector

```css
--vermilion-text: #a93222;
--vermilion-on-dark: #e15b43;

.darkSectionTitle > span {
  color: var(--vermilion-on-dark);
}
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 3. `a15e117cb51b` — fix(a11y): skip link focus target 복원

- **Full SHA:** `a15e117cb51b0c4611a8e8c418e2d4d1f702d3fd`
- **Importance:** A
- **Tags:** A11Y, DEBUG
- **이 Thread에서의 역할:** Shared/Cinematic/Editorial skip-target focusability

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/site-shell.tsx`의 `#main-content`
- `src/designs/cinematic/cinematic-route.tsx`의 `#cinematic-content`
- `src/designs/editorial/editorial-route.tsx`의 `#editorial-main`
- 각 `<main>`에 동일하게 추가된 `tabIndex={-1}`과 keyboard tab order에 미치는 범위
- Brutalist main이 이 SHA에는 포함되지 않고 다음 fix에서 보강되는 chronology

#### Commit-specific investigation

- `a15e117cb51b^`와 `a15e117cb51b`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Shared/Cinematic/Editorial skip-target focusability`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `e1aac08e0e9e`가 Brutalist main을 같은 invariant에 포함하고, `84c71d027630`이 keyboard sequence를 browser test로 고정합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Skip link href가 main ID를 가리켜 scroll 위치는 바뀔 수 있었지만, main landmark가 programmatically focusable하지 않으면 Enter 뒤 keyboard focus가 반복 navigation을 건너뛰었다고 보장할 수 없었습니다. |
| 실제 변경 file/symbol/call path | Shared shell, Cinematic frame, Editorial shell의 `<main>`에 `tabIndex={-1}`를 추가했습니다. 이는 일반 Tab 순서에는 넣지 않으면서 fragment navigation 시 focus target이 될 수 있게 합니다. |
| Data/state/DOM/resource owner와 lifetime | 각 renderer shell이 자신의 main ID와 landmark를 소유합니다. Browser가 fragment navigation/focus transition을 수행하고 component는 focusability contract만 제공합니다. |
| Failure·absence·fallback·cleanup | 이 commit은 세 shell만 수정합니다. Brutalist main은 아직 누락되어 있고, 실제 첫 Tab→Enter→focus sequence를 자동화하지 않습니다. 잘못된 href/duplicate ID도 여기서 검사하지 않습니다. |
| Test technique와 실행 증거 | Cross-renderer accessibility fix입니다. Static JSX diff를 확인했고 browser command는 실행하지 않았습니다. |
| 보장하는 것 | Shared, Cinematic, Editorial routes에서 skip target main이 programmatic focus를 받을 수 있는 markup이 됩니다. |
| 보장하지 않는 것 | Brutalist, custom renderer, assistive technology announcement, sticky-header scroll offset와 browser별 fragment focus 차이는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `e1aac08e0e9e`가 Brutalist main을 같은 invariant에 포함하고, `84c71d027630`이 keyboard sequence를 browser test로 고정합니다. |

#### 최소 code evidence

- **Commit:** `a15e117cb51b0c4611a8e8c418e2d4d1f702d3fd`
- **Path:** `src/components/portfolio/site-shell.tsx`
- **Location:** PageShell main landmark

```tsx
<main
  data-home-template={homeTemplate}
  id="main-content"
  tabIndex={-1}
>
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 4. `e1aac08e0e9e` — fix(a11y): Brutalist 지표의 definition semantics 수정

- **Full SHA:** `e1aac08e0e9e65695b0a60d53eacd812b88c1d76`
- **Importance:** A
- **Tags:** ARCH, RENDERER, A11Y
- **이 Thread에서의 역할:** Brutalist skip target와 metric definition semantics correction

#### 해당 SHA에서 확인할 실제 코드

- `src/designs/brutalist/brutalist-route.tsx`의 `#brutalist-main`에 `tabIndex={-1}`
- `HomeView` metric block의 `<dt>`, value `<dd>`, description `<p>` 구조
- description이 두 번째 `<dd className={styles.metricDescription}>`로 바뀌는 semantics
- `src/designs/brutalist/brutalist.module.css` selector가 `.metricBlock p`에서 `.metricDescription`으로 이동하는 점

#### Commit-specific investigation

- `e1aac08e0e9e^`와 `e1aac08e0e9e`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Brutalist skip target와 metric definition semantics correction`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `84c71d027630`이 all-design enabled-route Axe scan과 skip-link keyboard flow를 추가하여 세 accessibility fixes를 broader matrix에 연결합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 앞 commit이 세 shell의 main focusability를 고쳤지만 Brutalist는 별도 shell이라 빠져 있었습니다. 또한 metric definition list의 description을 일반 paragraph로 넣어 term/value 관계를 semantic structure가 완전히 표현하지 못했습니다. |
| 실제 변경 file/symbol/call path | Brutalist main에 `tabIndex={-1}`를 추가하고 각 metric description을 `<p>` 대신 추가 `<dd>`로 렌더링했습니다. CSS는 element name 의존 대신 semantic class를 대상으로 바꿨습니다. |
| Data/state/DOM/resource owner와 lifetime | Brutalist shell이 main focus target을 소유합니다. Metric term 뒤의 value와 optional description 모두 definition value로 DOM semantics에 포함되고 stylesheet는 class presentation만 소유합니다. |
| Failure·absence·fallback·cleanup | Description이 없는 metric은 추가 `<dd>`를 렌더링하지 않습니다. 이 fix는 Brutalist home metric block만 다루며 다른 custom `<dl>` 구조를 전수 검사하지 않습니다. |
| Test technique와 실행 증거 | Semantic markup correction입니다. JSX/CSS diff를 확인했지만 accessibility tree 또는 screen reader output을 실행하지 않았습니다. |
| 보장하는 것 | Brutalist도 skip-link focus invariant에 들어오고, metric description이 definition list의 value semantics로 표현됩니다. |
| 보장하지 않는 것 | 읽기 순서의 실제 announcement, terminology quality, all-page landmarks와 WCAG 전체 준수는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `84c71d027630`이 all-design enabled-route Axe scan과 skip-link keyboard flow를 추가하여 세 accessibility fixes를 broader matrix에 연결합니다. |

#### 최소 code evidence

- **Commit:** `e1aac08e0e9e65695b0a60d53eacd812b88c1d76`
- **Path:** `src/designs/brutalist/brutalist-route.tsx`
- **Location:** HomeView metric definition list

```tsx
<dt>{metric.label}</dt>
<dd>{String(metric.value).padStart(2, "0")}</dd>
{metric.description ? (
  <dd className={styles.metricDescription}>
    {metric.description}
  </dd>
) : null}
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 5. `84c71d027630` — test(a11y): 디자인×route WCAG 행렬 추가

- **Full SHA:** `84c71d02763081ccc6d1142f42bc69693f29124c`
- **Importance:** A
- **Tags:** ARCH, ROUTING, A11Y
- **이 Thread에서의 역할:** Shared site matrix, Axe WCAG scan과 keyboard skip-link regression

#### 해당 SHA에서 확인할 실제 코드

- `package.json`/lockfile의 `@axe-core/playwright` dependency
- 새 `tests/e2e/site-matrix.ts`의 five design IDs, first enabled project, route definitions, page-enabled filtering, `withExplicitDesign`
- `tests/e2e/accessibility.spec.ts`의 WCAG tag set와 `expectAccessibleRoute`
- response, design root, main/banner/contentinfo count, formatted violations, zero-violation assertion
- 첫 `Tab`으로 skip link focus 후 `Enter`로 main focus를 확인하는 keyboard test
- `portfolio.spec.ts`가 같은 matrix helper를 사용하도록 중복을 제거한 integration change

#### Commit-specific investigation

- `84c71d027630^`와 `84c71d027630`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Shared site matrix, Axe WCAG scan과 keyboard skip-link regression`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: 이 Thread의 CSS/markup fixes를 one-off 수정에서 category-wide automated evidence로 승격합니다. Release CI에서 실제 실행되는지는 category 08의 책임입니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Browser route tests는 이미 있었지만 Axe rule engine, shared route vocabulary와 keyboard skip-link end-state가 없었습니다. Accessibility fixes가 특정 renderer에만 적용되어도 category-wide regression이 이를 보장하지 못했습니다. |
| 실제 변경 file/symbol/call path | `site-matrix.ts`로 design IDs와 enabled routes를 한 owner에 모으고, AxeBuilder가 configured WCAG 2.0/2.1/2.2 A/AA tags로 각 resulting page를 분석하게 했습니다. 각 design마다 enabled routes를 순회하며 landmark cardinality와 violations를 검사하고, 별도 test가 skip-link focus transfer를 실행합니다. |
| Data/state/DOM/resource owner와 lifetime | Content files가 route enablement/first project를 소유하고 site-matrix가 test enumeration을 소유합니다. Production route/renderer가 DOM을 만들고 Playwright+Axe가 browser accessibility tree에 근거한 violations를 관찰합니다. |
| Failure·absence·fallback·cleanup | Enabled project가 없으면 helper가 즉시 error를 던집니다. Non-OK response, wrong design root, duplicate/missing landmarks, Axe violation, focus transfer 실패가 명시적으로 test를 실패시킵니다. Violation formatter는 design/path/node target과 failure summary를 남깁니다. |
| Test technique와 실행 증거 | Browser-level automated accessibility matrix입니다. Axe가 검출 가능한 rules와 scripted keyboard path를 다루며 manual screen-reader/usability review를 대체하지 않습니다. Exact test code는 확인했지만 command는 실행하지 않았습니다. |
| 보장하는 것 | 다섯 design의 모든 enabled route가 동일한 route source를 사용해 자동 accessibility scan을 받고, skip link가 main focus까지 이동해야 하는 regression contract가 생깁니다. |
| 보장하지 않는 것 | Axe 밖의 인지·콘텐츠 품질, 모든 assistive technology/browser, disabled routes, authenticated/dynamic states, color appearance의 수동 판단과 full keyboard journey는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | 이 Thread의 CSS/markup fixes를 one-off 수정에서 category-wide automated evidence로 승격합니다. Release CI에서 실제 실행되는지는 category 08의 책임입니다. |

#### 최소 code evidence

- **Commit:** `84c71d02763081ccc6d1142f42bc69693f29124c`
- **Path:** `tests/e2e/accessibility.spec.ts`
- **Location:** expectAccessibleRoute

```ts
const results = await new AxeBuilder({ page })
  .withTags([...wcag22AATags])
  .analyze();

expect(results.violations).toEqual([]);
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

## 6. Invariant evolution ledger

| Invariant | 도입/변경 SHA | Historical evidence | 상태 |
| --- | --- | --- | --- |
| Accessibility는 actual route/design DOM에서 검사한다. | 31c438b52e4b → 84c71d027630 | Playwright server/browser foundation + AxeBuilder | 기반 후 자동화 |
| Color ownership은 light/dark surface별로 분리된다. | 5a6fd8a802ff | Shared accent와 Editorial on-dark token | 수정 |
| 모든 design main은 skip-link focus target이다. | a15e117cb51b → e1aac08e0e9e → 84c71d027630 | tabIndex fixes + first Tab/Enter/main-focus test | 부분 수정→완성→검증 |
| Metric description은 definition value semantics를 갖는다. | e1aac08e0e9e | `<p>`→`<dd>` | 수정 |

## 7. Failure → Fix → Test

| Earlier failure/risk | Fix SHA | Corrected decision | Regression evidence |
| --- | --- | --- | --- |
| Dark surface에서 light-surface token을 재사용해 contrast가 낮아질 위험 | 5a6fd8a802ff | On-dark token 분리 | CSS fix; 후속 Axe scan |
| Skip link가 scroll만 하고 keyboard focus를 옮기지 못할 위험 | a15e117cb51b → e1aac08e0e9e → 84c71d027630 | All-shell main focusability + keyboard assertion | Fix→Fix→Test |
| `<dl>` 내부 description이 term/value 관계를 잃을 위험 | e1aac08e0e9e | Description을 `<dd>`로 변환 | Semantic fix |
| Route/design 추가가 accessibility scan에서 누락될 위험 | 84c71d027630 | Shared `site-matrix.ts`와 enabled-page filtering | Matrix regression |

## 8. Ownership/state/responsibility 변화

| 대상 | 초기 owner/state | 최종 owner/state | Evidence |
| --- | --- | --- | --- |
| Route/design enumeration | Portfolio spec 내부 | Shared `site-matrix.ts` | General E2E와 a11y suite 공유 |
| Surface color policy | Shared token 재사용 | Renderer/surface-specific token | globals + Editorial stylesheet |
| Main focus target | Shell별 불균일 | All five design shells | `tabIndex={-1}` |
| Semantic metric structure | Brutalist presentation markup | Brutalist semantic markup | `dt`/`dd` group |
| Automated evidence | General route checks | Axe + landmark + keyboard checks | `accessibility.spec.ts` |

## 9. 최종 Thread state

다음 내용을 코드 없이 설명합니다: 최종 owner, input→decision→output, failure/absence policy, regression evidence와 명시적 non-guarantee.

최종 상태에서 route/design vocabulary는 shared helper가 content enablement에서 파생하고, Axe suite가 five-design enabled routes를 actual browser에서 분석합니다. Contrast token, focus target과 definition semantics fixes는 같은 matrix 및 skip-link keyboard flow에 연결됩니다. Automated rules는 manual screen-reader와 usability 판단을 대체하지 않습니다.

## 10. 최종 실행 흐름

| 단계 | Owner / mechanism | Input | Output/state | Failure/non-guarantee |
| ---: | --- | --- | --- | --- |
| 1. Matrix construction | `site-matrix.ts` | content pages/projects + design IDs | enabled routes/design URLs | enabled project 없음→error |
| 2. Route render | Next server + renderer | explicit design URL | browser DOM/CSS | non-OK/wrong root |
| 3. Landmark/semantics | Shell/renderer markup | content/view model | main/banner/footer/dl | missing/duplicate/invalid structure |
| 4. Axe analysis | `AxeBuilder` | rendered page | tag-filtered violations | formatted node failures |
| 5. Keyboard verification | Playwright | Tab→Enter | skip link focus→main focus | focus assertion failure |

## 11. 학습 완료 확인

- [x] 모든 referenced SHA를 exact historical diff 기준으로 설명했습니다.
- [x] Commit map의 SHA·순서·subject·importance·tags를 변경하지 않았습니다.
- [x] Fix를 earlier failure/assumption에 연결했습니다.
- [x] Test가 실행하는 production path와 증명하지 않는 범위를 구분했습니다.
- [x] 정적 inspection과 실제 command execution을 구분했습니다.
- [x] Thread-level invariant, ownership과 final flow를 완성했습니다.
- [x] 실행 제한을 명시했습니다: 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다.
===== END FILE: 04-browser-accessibility-route-matrix.md =====

===== BEGIN FILE: 05-client-performance-and-server-first-optimization.md =====
# Thread: Client performance and server-first optimization

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `07-testing-performance-and-regression-strategy`
>
> Phase 1에서 감사·수정한 뒤 동결한 scaffold를 기준으로 합니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance와 tags는 branch-local `commit/commit-importance.md` 분류와 exact commit metadata를 기준으로 고정했습니다.
- 이 문서의 Thread goal, commit grouping과 source-defined 역할은 Phase 1 category audit 결과입니다.
- Phase 2에서는 SHA, 순서, subject, importance, tags, 역할, 질문과 문서 구조를 바꾸지 않습니다.
- 다른 branch 또는 final HEAD의 구현을 earlier SHA 설명에 소급하지 않습니다.
- Runtime evidence는 실제로 실행한 command만 기록하며, 미실행 상태를 통과로 해석하지 않습니다.

## 1. Thread 목표

Client reveal state, global font loading과 automatic route prefetch를 줄이고 actual browser request/font behavior 및 native interaction latency를 regression tests로 고정하는 과정을 복원합니다.

### 동결된 핵심 invariant

- Core content visibility는 hydration·IntersectionObserver progress에 의존하지 않습니다.
- Font preload/consumer는 locale·renderer 필요 범위로 제한되고 Geist Mono는 audited initial routes에서 불필요하게 요청되지 않습니다.
- Internal links는 idle/viewport route prefetch를 명시적으로 끄되 user-initiated navigation은 유지합니다.
- Design switcher close와 mobile menu open은 defined Event Timing measurement에서 3-sample median·maximum 200ms 이하를 목표로 합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Reveal의 observer/state lifecycle을 제거하면 어떤 animation을 포기하고 어떤 server-first invariant를 얻는가?
- Next localFont options, locale predicate와 CSS consumer가 실제 request behavior에 어떻게 연결되는가?
- 모든 renderer Link call site의 prefetch opt-out을 browser request stream에서 어떻게 검증하는가?
- Event Timing sample은 trusted click, interaction ID, below-threshold entry와 paint settlement를 어떻게 처리하는가?

## 3. 완료 기준

- 각 referenced SHA의 exact parent diff와 resulting changed files를 확인합니다.
- Commit별 previous state, implementation decision, ownership/lifetime, failure path와 non-guarantee를 구분합니다.
- Fix는 earlier assumption과 root cause에 연결하고, test는 production path·technique·proves/does-not-prove를 구분합니다.
- A-level은 subsystem·failure·verification 관계까지, B-level은 local role과 후속 연결까지만 설명합니다.
- Thread-level invariant evolution, Failure → Fix → Test, ownership transfer와 final flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b8164cfdddbd` | refactor(ui): reveal 콘텐츠를 server에서 즉시 표시 | A | ARCH, CONTENT, REFACTOR | Observer/client state 제거와 immediate server-visible content |
| 2 | `b09775ec17c3` | perf(font): route별 글꼴 로딩 비용 축소 | A | ARCH, ROUTING, PERF | Locale/consumer-aware local font loading policy |
| 3 | `2c0c9bb34b77` | perf(navigation): 유휴 route prefetch 비활성화 | A | ARCH, ROUTING, PERF | Shared/five-renderer internal Link idle-prefetch opt-out |
| 4 | `dfeb324572fa` | test(perf): 유휴 route 요청과 글꼴 경계 검증 | A | ARCH, VALIDATION, ROUTING | Actual `_rsc` request, preload와 font resource browser regression |
| 5 | `787478032d27` | test(perf): 사용자 상호작용 지연 측정 추가 | A | PERF, TEST | Trusted Event Timing samples와 200ms interaction target |

## 5. Commit별 학습 기록

각 section은 해당 SHA의 tree와 parent diff만 기준으로 작성합니다. 같은 SHA가 다른 category Thread에 등장하더라도 여기서는 위 역할과 파일 범위만 설명합니다.

### 1. `b8164cfdddbd` — refactor(ui): reveal 콘텐츠를 server에서 즉시 표시

- **Full SHA:** `b8164cfdddbdda1bbff45f06649fc706cf552123`
- **Importance:** A
- **Tags:** ARCH, CONTENT, REFACTOR
- **이 Thread에서의 역할:** Observer/client state 제거와 immediate server-visible content

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/reveal.tsx`에서 제거된 `"use client"`, React hooks와 `Ref` type
- 초기 `visible` 계산의 `window`/`IntersectionObserver` branch
- `useEffect`의 observer 생성, rootMargin/threshold, observe/disconnect cleanup
- 최종 class가 항상 `reveal-item is-visible`이며 polymorphic `as`, `className`, `delay`는 유지되는 점

#### Commit-specific investigation

- `b8164cfdddbd^`와 `b8164cfdddbd`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Observer/client state 제거와 immediate server-visible content`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `b09775ec17c3`과 `2c0c9bb34b77`이 font/network 비용을 줄이고, `dfeb...`·`787...`가 browser-observable regression evidence를 추가합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Reveal은 content를 server markup에 포함하더라도 visible class를 client state가 결정했습니다. IntersectionObserver 지원 환경에서는 hydration/effect/intersection 전까지 hidden presentation이 유지되고 observer/ref/state가 모든 instance에 client runtime 비용을 만들었습니다. |
| 실제 변경 file/symbol/call path | Client directive와 state/effect/ref/observer lifecycle을 전부 제거했습니다. Component는 server-compatible function이 되어 항상 `reveal-item is-visible`를 출력하고, optional transition delay만 inline style로 유지합니다. |
| Data/state/DOM/resource owner와 lifetime | Visibility decision은 browser observer가 아니라 server-rendered markup이 소유합니다. Component는 element type과 class/transition-delay presentation만 계산하며 observer resource와 cleanup owner가 사라집니다. |
| Failure·absence·fallback·cleanup | IntersectionObserver 미지원 fallback과 observer cleanup branch 자체가 없어집니다. 대신 scroll-enter animation behavior도 제거됩니다. `delay`가 남아 있어 CSS transition이 존재할 수 있지만 content visibility는 client progress에 의존하지 않습니다. |
| Test technique와 실행 증거 | Server-first refactor입니다. Exact source diff를 확인했으며 JavaScript byte·paint timing은 이 commit에서 측정되지 않았습니다. |
| 보장하는 것 | Reveal-wrapped content가 hydration 또는 viewport intersection을 기다리지 않고 visible class를 가진 server output으로 제공됩니다. |
| 보장하지 않는 것 | 모든 client JavaScript 제거, LCP 개선량, CSS animation cost, below-the-fold rendering 최적화와 user-perceived 성능 개선 수치는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `b09775ec17c3`과 `2c0c9bb34b77`이 font/network 비용을 줄이고, `dfeb...`·`787...`가 browser-observable regression evidence를 추가합니다. |

#### 최소 code evidence

- **Commit:** `b8164cfdddbdda1bbff45f06649fc706cf552123`
- **Path:** `src/components/portfolio/reveal.tsx`
- **Location:** Reveal 반환부

```tsx
return (
  <Component
    className={`reveal-item is-visible ${className}`}
    style={{ transitionDelay: `${delay}ms` }}
  >
    {children}
  </Component>
);
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 2. `b09775ec17c3` — perf(font): route별 글꼴 로딩 비용 축소

- **Full SHA:** `b09775ec17c3c393829d7b77d36bd34d34e6c04d`
- **Importance:** A
- **Tags:** ARCH, ROUTING, PERF
- **이 Thread에서의 역할:** Locale/consumer-aware local font loading policy

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`의 Geist Sans/Mono/Korean Serif `localFont` options
- `display: "swap"`에서 `"optional"`로의 변경과 Mono/Korean `preload: false`
- `usesKoreanSerif` language predicate와 non-Korean `serifFallback` CSS custom property
- `<html>` className/style에서 Korean serif variable을 조건부로 주입하는 방식
- `design-switcher.module.css`가 Geist Mono variable 대신 system monospace stack을 사용하는 두 selector

#### Commit-specific investigation

- `b09775ec17c3^`와 `b09775ec17c3`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Locale/consumer-aware local font loading policy`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `dfeb324572fa`가 generated `@font-face`, preload links와 actual font requests를 browser에서 검사합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Root layout은 세 local font variable을 모든 route의 `<html>`에 붙였습니다. 실제 design/locale이 font를 쓰지 않아도 preload 또는 generated font-face consumption이 초기 request 비용으로 이어질 수 있었고 switcher의 작은 ordinal text가 Geist Mono dependency를 유지했습니다. |
| 실제 변경 file/symbol/call path | 세 font의 display policy를 `optional`로 바꾸고 Mono와 Korean Serif preload를 끕니다. Korean serif variable은 site language가 `ko`로 시작할 때만 class에 추가하며 그 외에는 CSS fallback variable을 inline style로 제공합니다. Switcher count/number는 system monospace로 바꿉니다. |
| Data/state/DOM/resource owner와 lifetime | Root layout이 locale에 따른 font-variable ownership을 갖고 Next font loader가 font-face/preload behavior를 생성합니다. Individual renderer CSS는 variable 또는 fallback stack을 소비합니다. Switcher는 더 이상 Geist Mono resource를 요구하지 않습니다. |
| Failure·absence·fallback·cleanup | Locale predicate는 `site.language.toLowerCase().startsWith("ko")`만 봅니다. `optional`은 font가 반드시 내려받지 않는다는 절대 보장이 아니며 실제 CSS consumer가 남아 있으면 request가 발생할 수 있습니다. Korean locale에서는 serif variable이 유지됩니다. |
| Test technique와 실행 증거 | Static performance policy change입니다. Source options와 CSS consumers를 확인했으며 network trace는 후속 test 전까지 없습니다. |
| 보장하는 것 | 불필요한 global font preload를 줄이고 non-Korean routes에 serif system fallback을 제공하며 switcher의 Mono dependency를 제거합니다. |
| 보장하지 않는 것 | 각 route의 실제 font bytes, cache state, browser별 optional behavior, CLS/LCP 개선량과 all-design font usage를 이 commit만으로 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `dfeb324572fa`가 generated `@font-face`, preload links와 actual font requests를 browser에서 검사합니다. |

#### 최소 code evidence

- **Commit:** `b09775ec17c3c393829d7b77d36bd34d34e6c04d`
- **Path:** `src/app/layout.tsx`
- **Location:** Geist Mono localFont configuration

```tsx
const geistMono = localFont({
  display: "optional",
  preload: false,
  src: "./fonts/GeistMono-Variable.woff2",
  variable: "--font-geist-mono",
  weight: "100 900",
});
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 3. `2c0c9bb34b77` — perf(navigation): 유휴 route prefetch 비활성화

- **Full SHA:** `2c0c9bb34b77655f40eb13b51fd1e754881a7fdb`
- **Importance:** A
- **Tags:** ARCH, ROUTING, PERF
- **이 Thread에서의 역할:** Shared/five-renderer internal Link idle-prefetch opt-out

#### 해당 SHA에서 확인할 실제 코드

- `ContentLinkView`, `DesignSwitcher`, `ProjectCard`, `SiteHeader`, `SiteFooter`의 internal `<Link>`
- Design/Classic home·project routes의 action/back/detail links
- Editorial, Brutalist, Cinematic renderer의 brand/navigation/project/action links
- 각 href/query-building helper는 유지되고 `prefetch={false}`만 추가되는지
- external `<a>`나 실제 user-initiated navigation은 이 정책의 대상이 아니라는 점

#### Commit-specific investigation

- `2c0c9bb34b77^`와 `2c0c9bb34b77`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Shared/five-renderer internal Link idle-prefetch opt-out`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `dfeb324572fa`가 five-design home/detail에서 `_rsc` requests를 실제 browser request stream으로 관찰합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 많은 route/design links가 viewport에 동시에 나타나는 portfolio 구조에서 Next의 default prefetch는 사용자가 아무 동작을 하지 않아도 여러 RSC route request를 만들 수 있었습니다. Design마다 같은 destination links가 반복되어 idle network work가 증폭될 수 있었습니다. |
| 실제 변경 file/symbol/call path | 공유 shell/content/project components와 독립 renderer의 모든 확인된 internal Next `Link`에 `prefetch={false}`를 추가했습니다. Existing href construction, debug/design query propagation, labels와 navigation semantics는 유지합니다. |
| Data/state/DOM/resource owner와 lifetime | Link가 destination을 소유하지 않고 route helper가 href를 계속 결정합니다. 변경된 ownership은 background acquisition policy이며, browser/Next가 idle prefetch를 시작하지 않도록 각 Link call site가 explicit opt-out을 선언합니다. User click navigation은 그대로 Next Link가 수행합니다. |
| Failure·absence·fallback·cleanup | 누락된 internal Link가 있으면 prefetch가 남을 수 있습니다. `prefetch={false}`는 speculative browser preload, image/font request 또는 click 이후 RSC request를 막지 않습니다. External links에는 적용되지 않습니다. |
| Test technique와 실행 증거 | Cross-renderer network policy refactor입니다. Large diff의 Link call sites를 확인했으며 actual request absence는 후속 Playwright test가 담당합니다. |
| 보장하는 것 | 감사된 public route links는 idle/viewport 상태에서 Next route prefetch를 요청하지 않도록 명시됩니다. |
| 보장하지 않는 것 | Zero background network, navigation latency 개선, hover behavior, future Next semantics와 every dynamically introduced Link coverage는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `dfeb324572fa`가 five-design home/detail에서 `_rsc` requests를 실제 browser request stream으로 관찰합니다. |

#### 최소 code evidence

- **Commit:** `2c0c9bb34b77655f40eb13b51fd1e754881a7fdb`
- **Path:** `src/components/portfolio/site-shell.tsx`
- **Location:** internal navigation Link pattern

```tsx
<Link
  href={getTemplateHref(item.href, templateSwitcher?.activeId, {
    contentDebug: templateSwitcher?.contentDebug,
  })}
  prefetch={false}
>
  {item.label}
</Link>
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 4. `dfeb324572fa` — test(perf): 유휴 route 요청과 글꼴 경계 검증

- **Full SHA:** `dfeb324572fac339cc3bd3162da88c4bc1e68c7e`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, ROUTING
- **이 Thread에서의 역할:** Actual `_rsc` request, preload와 font resource browser regression

#### 해당 SHA에서 확인할 실제 코드

- `tests/e2e/performance.spec.ts`의 `designIds × initialRoutes(home, first project detail)` loop
- `page.on("request")` listener가 `_rsc` query와 `resourceType() === "font"`를 기록하는 방식
- page visit, first heading visibility, 1-second idle window와 listener 제거 시점
- same-origin stylesheet `cssRules`에서 Geist Mono `CSSFontFaceRule` source URL을 추출하는 code
- `link[rel~="preload"][as="font"]` manifest와 actual font requests assertions
- Design/Editorial만 Geist Mono download absence를 요구하고 모든 design은 preload absence를 요구하는 scope

#### Commit-specific investigation

- `dfeb324572fa^`와 `dfeb324572fa`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Actual `_rsc` request, preload와 font resource browser regression`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `787478032d27`가 network absence와 별개로 user interaction latency를 Event Timing으로 측정합니다. Bundle/Lighthouse/release gates는 category 08의 별도 Thread가 소유합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Font/prefetch source policy는 추가되었지만 browser가 generated CSS와 Next runtime을 통해 무엇을 실제로 요청하는지 증명하지 못했습니다. Static grep만으로 framework-generated preload/RSC behavior를 확인할 수 없었습니다. |
| 실제 변경 file/symbol/call path | Playwright test가 각 design의 home과 first project detail을 방문하면서 request listener를 설치합니다. `_rsc` requests와 font resources를 모으고, loaded stylesheets에서 Geist Mono source path를 찾아 preload manifest 및 observed requests와 비교합니다. |
| Data/state/DOM/resource owner와 lifetime | Browser request stream과 CSSOM이 runtime evidence source입니다. `site-matrix`가 design/fixture enumeration을 소유하고 test가 listener lifetime을 page goto부터 1-second idle window까지 소유합니다. Production Link/font config는 decision owner로 남습니다. |
| Failure·absence·fallback·cleanup | Enabled project가 없으면 test module이 error를 던집니다. Cross-origin stylesheet access는 catch 후 건너뜁니다. Geist Mono face 자체가 없으면 별도 assertion이 실패하여 단순 삭제로 test를 우회할 수 없습니다. Listener는 assertion 전 제거됩니다. |
| Test technique와 실행 증거 | Browser network/CSSOM regression test입니다. Five designs × two initial routes를 configured Playwright projects에서 실행하도록 정의합니다. Exact code는 확인했지만 현재 환경에서는 실행하지 않았습니다. |
| 보장하는 것 | 관찰 window 동안 idle `_rsc` request가 없어야 하고 Geist Mono는 preload되지 않아야 하며, Design/Editorial initial routes에서는 해당 font resource가 실제로 내려받아지지 않아야 합니다. |
| 보장하지 않는 것 | 다른 routes, interaction/hover 후 requests, cache-cold/warm variance, all fonts, transferred bytes, navigation speed와 production CDN behavior는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | `787478032d27`가 network absence와 별개로 user interaction latency를 Event Timing으로 측정합니다. Bundle/Lighthouse/release gates는 category 08의 별도 Thread가 소유합니다. |

#### 최소 code evidence

- **Commit:** `dfeb324572fac339cc3bd3162da88c4bc1e68c7e`
- **Path:** `tests/e2e/performance.spec.ts`
- **Location:** request listener

```ts
if (url.searchParams.has("_rsc")) {
  prefetchedRoutes.push(`${url.pathname}${url.search}`);
}
if (request.resourceType() === "font") {
  loadedFonts.push(url.pathname);
}
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 5. `787478032d27` — test(perf): 사용자 상호작용 지연 측정 추가

- **Full SHA:** `787478032d27be5dc2435ad3378318644251c10d`
- **Importance:** A
- **Tags:** PERF, TEST
- **이 Thread에서의 역할:** Trusted Event Timing samples와 200ms interaction target

#### 해당 SHA에서 확인할 실제 코드

- `tests/e2e/interaction-performance.spec.ts`의 16ms observer threshold, 200ms target, 3-sample constants
- `PerformanceObserver` event entries, `interactionId`, `startTime`, trusted click counter를 저장하는 probe
- double `requestAnimationFrame`+timer settle, warmup, probe reset/read lifecycle
- entry가 없을 때 `<16ms` upper bound로 해석하고 한 trusted click·한 interaction ID를 요구하는 logic
- 세 sample의 median과 maximum을 모두 200ms 이하로 assert하고 diagnostic line을 출력하는 code
- all-design switcher close와 mobile project에서만 menu-open을 측정하는 scenario scope

#### Commit-specific investigation

- `787478032d27^`와 `787478032d27`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Trusted Event Timing samples와 200ms interaction target`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: 이 Thread의 runtime interaction evidence를 완성합니다. Release-level Lighthouse/bundle budget 및 CI enforcement는 category 08로 분리됩니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Server-first/network optimizations은 있었지만 사용자가 실제 native disclosure를 조작할 때 처리 지연이 목표 안에 있는지 정량 contract가 없었습니다. 단일 wall-clock click measurement는 warmup·paint·untrusted dispatch를 섞을 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | Chromium Event Timing observer를 설치하고 browser-trusted click을 정확히 하나 포함하는 sample만 받습니다. Warmup 뒤 각 scenario를 세 번 측정하고 동일 interaction ID의 event entries 중 최대 duration을 sample upper bound로 사용합니다. Median과 maximum 모두 200ms target을 넘으면 실패합니다. |
| Data/state/DOM/resource owner와 lifetime | Browser PerformanceObserver가 event timing entries를 소유하고 test probe가 한 page lifetime 동안 수집 buffer와 sample start를 관리합니다. Production native disclosure/close island가 interaction path이며 test는 state visible/hidden과 focus end-state도 확인합니다. |
| Failure·absence·fallback·cleanup | Event Timing 미지원이면 즉시 실패합니다. Observer threshold 아래라 entry가 없으면 `<16ms`로 conservative upper bound를 기록합니다. Trusted click 수가 1이 아니거나 interaction ID가 여러 개면 측정 자체를 invalid로 실패시킵니다. Desktop에서는 mobile menu test를 skip합니다. |
| Test technique와 실행 증거 | Browser interaction-performance regression입니다. Three-sample median/max gate와 paint settle을 사용하지만 lab measurement이며 statistical benchmark suite는 아닙니다. Exact implementation을 확인했으나 실행하지 않았습니다. |
| 보장하는 것 | 다섯 design의 switcher-close와 mobile viewport의 menu-open이 defined browser project에서 3회 sample median·maximum 200ms 이하라는 executable target을 갖습니다. |
| 보장하지 않는 것 | Real-user INP distribution, low-end devices, network-dependent navigation, long sessions, every interaction, cross-browser timing API와 production telemetry를 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | 이 Thread의 runtime interaction evidence를 완성합니다. Release-level Lighthouse/bundle budget 및 CI enforcement는 category 08로 분리됩니다. |

#### 최소 code evidence

- **Commit:** `787478032d27be5dc2435ad3378318644251c10d`
- **Path:** `tests/e2e/interaction-performance.spec.ts`
- **Location:** reportAndAssertSamples

```ts
expect(medianUpperBoundMs).toBeLessThanOrEqual(
  INTERACTION_TARGET_MS,
);
expect(maxUpperBoundMs).toBeLessThanOrEqual(
  INTERACTION_TARGET_MS,
);
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

## 6. Invariant evolution ledger

| Invariant | 도입/변경 SHA | Historical evidence | 상태 |
| --- | --- | --- | --- |
| Content는 server output에서 즉시 visible하다. | b8164cfdddbd | Reveal이 항상 `is-visible`, no client hooks/observer | 수정 |
| Font acquisition은 need-based이고 audited mono preload가 없다. | b09775ec17c3 → dfeb324572fa | localFont policy + CSSOM/request assertions | 정책→검증 |
| Audited internal links는 idle RSC prefetch를 하지 않는다. | 2c0c9bb34b77 → dfeb324572fa | `prefetch={false}` + `_rsc` request capture | 정책→검증 |
| Core disclosure interactions는 explicit lab target을 가진다. | 787478032d27 | 3 samples, median/max ≤200ms | 측정·gate |

## 7. Failure → Fix → Test

| Earlier failure/risk | Fix SHA | Corrected decision | Regression evidence |
| --- | --- | --- | --- |
| Hydration/observer가 늦거나 없을 때 content가 hidden으로 남을 위험 | b8164cfdddbd | Client visibility state 제거 | Server-first correction |
| 사용하지 않는 font가 preload/download될 위험 | b09775ec17c3 → dfeb324572fa | Options/consumer 축소 + CSSOM/network observation | Fix→Test |
| 링크가 많은 page가 idle RSC requests를 폭증시킬 위험 | 2c0c9bb34b77 → dfeb324572fa | Cross-renderer opt-out + request listener | Fix→Test |
| Interaction 측정이 synthetic click/다중 event/threshold absence를 잘못 해석할 위험 | 787478032d27 | Trusted count, one interaction ID, `<16ms` upper bound | Measurement validity checks |

## 8. Ownership/state/responsibility 변화

| 대상 | 초기 owner/state | 최종 owner/state | Evidence |
| --- | --- | --- | --- |
| Content visibility | Client Reveal observer/state | Server markup | `Reveal` |
| Font preload/use | Global root classes | Locale + actual CSS consumers | `layout.tsx`, CSS |
| Idle route acquisition | Next default Link policy | Explicit call-site opt-out | `prefetch={false}` |
| Runtime network evidence | 없음 | Playwright request/CSSOM probe | `performance.spec.ts` |
| Interaction timing evidence | 없음 | PerformanceObserver probe | `interaction-performance.spec.ts` |

## 9. 최종 Thread state

다음 내용을 코드 없이 설명합니다: 최종 owner, input→decision→output, failure/absence policy, regression evidence와 명시적 non-guarantee.

최종 상태에서는 content reveal이 server-visible이고 font 및 route prefetch policy가 initial request cost를 줄이는 방향으로 explicit해집니다. Playwright가 five-design home/detail의 idle `_rsc`와 font request/preload를 관찰하고, Event Timing test가 disclosure interactions의 lab target을 검사합니다. Bundle size, Lighthouse와 release CI enforcement는 category 08이 소유합니다.

## 10. 최종 실행 흐름

| 단계 | Owner / mechanism | Input | Output/state | Failure/non-guarantee |
| ---: | --- | --- | --- | --- |
| 1. Server content | `Reveal` | children/as/delay | visible markup | observer 없음 |
| 2. Font policy | Root layout/Next font | site language + font definitions | conditional variables/preloads | browser optional behavior |
| 3. Link policy | Shared/renderer Next Links | internal href | no idle prefetch opt-in | 누락된 call site 가능 |
| 4. Network verification | `performance.spec.ts` | browser requests/CSSOM/preloads | absence/presence assertions | window 밖 behavior 미보장 |
| 5. Interaction verification | `interaction-performance.spec.ts` | trusted click/Event Timing | 3-sample upper bounds | unsupported API/invalid sample failure |

## 11. 학습 완료 확인

- [x] 모든 referenced SHA를 exact historical diff 기준으로 설명했습니다.
- [x] Commit map의 SHA·순서·subject·importance·tags를 변경하지 않았습니다.
- [x] Fix를 earlier failure/assumption에 연결했습니다.
- [x] Test가 실행하는 production path와 증명하지 않는 범위를 구분했습니다.
- [x] 정적 inspection과 실제 command execution을 구분했습니다.
- [x] Thread-level invariant, ownership과 final flow를 완성했습니다.
- [x] 실행 제한을 명시했습니다: 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다.
===== END FILE: 05-client-performance-and-server-first-optimization.md =====

===== BEGIN FILE: 06-visual-regression-and-responsive-baselines.md =====
# Thread: Visual regression and responsive baselines

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `07-testing-performance-and-regression-strategy`
>
> Phase 1에서 감사·수정한 뒤 동결한 scaffold를 기준으로 합니다.

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance와 tags는 branch-local `commit/commit-importance.md` 분류와 exact commit metadata를 기준으로 고정했습니다.
- 이 문서의 Thread goal, commit grouping과 source-defined 역할은 Phase 1 category audit 결과입니다.
- Phase 2에서는 SHA, 순서, subject, importance, tags, 역할, 질문과 문서 구조를 바꾸지 않습니다.
- 다른 branch 또는 final HEAD의 구현을 earlier SHA 설명에 소급하지 않습니다.
- Runtime evidence는 실제로 실행한 command만 기록하며, 미실행 상태를 통과로 해석하지 않습니다.

## 1. Thread 목표

Playwright route/viewport foundation과 renderer/token structural contracts 위에 five-design responsive screenshot baselines와 exact snapshot manifest를 추가하는 과정을 복원합니다.

### 동결된 핵심 invariant

- Visual baseline은 production route/design output과 configured desktop/mobile projects에서 생성됩니다.
- Capture 전 reduced motion, network idle, font readiness와 image settlement를 기다립니다.
- Home은 five-design desktop/mobile, first project detail은 five-design desktop baseline을 정확히 갖습니다.
- Baseline file set은 `SITE_DESIGN_IDS`에서 계산한 exact 15-file manifest와 일치합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 route matrix가 snapshot capture에 필요한 server, viewport와 design vocabulary를 무엇까지 제공했는가?
- Token/renderer structural tests는 pixel baseline 이전에 어떤 빠른 failure signal을 주는가?
- Stable-page helper가 animation/font/image nondeterminism을 어떻게 줄이는가?
- 1% pixel tolerance와 15-file manifest가 보장하지 않는 route/state/platform 범위는 무엇인가?

## 3. 완료 기준

- 각 referenced SHA의 exact parent diff와 resulting changed files를 확인합니다.
- Commit별 previous state, implementation decision, ownership/lifetime, failure path와 non-guarantee를 구분합니다.
- Fix는 earlier assumption과 root cause에 연결하고, test는 production path·technique·proves/does-not-prove를 구분합니다.
- A-level은 subsystem·failure·verification 관계까지, B-level은 local role과 후속 연결까지만 설명합니다.
- Thread-level invariant evolution, Failure → Fix → Test, ownership transfer와 final flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서 확인할 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `31c438b52e4b` | test(e2e): 다섯 디자인의 route matrix 검증 | A | ARCH, VALIDATION, ROUTING | Desktop/mobile Playwright projects와 five-design route foundation |
| 2 | `055b733cbb7e` | test(design): 독립 renderer와 design token 경계 검증 | A | ARCH, VALIDATION, RENDERER | Renderer marker와 design-token structural precondition |
| 3 | `882a2f9d753e` | test(visual): 다섯 디자인 회귀 기준 추가 | A | TEST | Stable screenshot suite, 15 PNG baselines와 exact manifest contract |

## 5. Commit별 학습 기록

각 section은 해당 SHA의 tree와 parent diff만 기준으로 작성합니다. 같은 SHA가 다른 category Thread에 등장하더라도 여기서는 위 역할과 파일 범위만 설명합니다.

### 1. `31c438b52e4b` — test(e2e): 다섯 디자인의 route matrix 검증

- **Full SHA:** `31c438b52e4b5f87d7e88ce047dd1997aa8ef054`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, ROUTING
- **이 Thread에서의 역할:** Desktop/mobile Playwright projects와 five-design route foundation

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `test:e2e` script와 `@playwright/test` dependency
- `playwright.config.ts`의 testDir, timeout, single worker, webServer와 `chromium`/`mobile-chrome` projects
- `tests/e2e/portfolio.spec.ts` 내부의 design IDs, enabled route definitions와 content JSON fixture
- response, design root, heading/content/media, internal href, switcher, invalid view fallback, overflow, reduced-motion, touch/focus assertions
- dev compiler cold-route race를 피하기 위해 worker 수를 1로 둔 이유와 병렬성 비보장

#### Commit-specific investigation

- `31c438b52e4b^`와 `31c438b52e4b`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Desktop/mobile Playwright projects와 five-design route foundation`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: `1598a87702f6`는 browser보다 빠른 renderer compatibility matrix를 추가하고, `84c71d...`가 이 matrix의 route/design vocabulary를 `site-matrix.ts`로 추출해 Axe suite와 공유합니다. `882a...`는 같은 Playwright projects를 snapshot baseline에 사용합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | jsdom tests는 page output을 확인했지만 실제 browser request, CSS viewport, native disclosure, focus와 image loading을 관찰하지 못했습니다. 또한 all-design coverage가 Home 중심이어서 다른 route의 design dispatch 누락이 남을 수 있었습니다. |
| 실제 변경 file/symbol/call path | Playwright를 설치하고 deterministic local web server를 사용하는 config와 약 489-line browser suite를 추가합니다. Suite는 source JSON에서 enabled route와 expected content를 구성하고, 다섯 design을 desktop/mobile project에서 방문하여 route response와 presentation root를 검사합니다. |
| Data/state/DOM/resource owner와 lifetime | Route availability의 source는 `site.json`, design vocabulary는 presentation/config와 test matrix가 소유합니다. Browser page가 production Next route를 요청하고, Playwright는 network/DOM/layout/focus를 관찰합니다. 이 commit 시점에는 matrix helper가 `portfolio.spec.ts` 내부에 있습니다. |
| Failure·absence·fallback·cleanup | Invalid `view`는 default presentation으로 돌아가야 하고, mobile/desktop에서 horizontal overflow나 unusable touch/focus state가 드러나면 test가 실패합니다. Single worker는 Next development compiler의 cold-route invalidation을 피하지만 concurrency behavior를 검증하지 않습니다. |
| Test technique와 실행 증거 | Broad Playwright integration characterization입니다. 실제 Chromium 계열 page, CSS, native controls와 links를 통과합니다. Visual pixel comparison과 Axe rule engine은 아직 없습니다. |
| 보장하는 것 | 다섯 design이 enabled public routes에서 response·root·핵심 content/navigation을 유지하고 desktop/mobile browser에서 기본 interaction/layout contract를 충족하는지 한 suite에서 확인할 수 있습니다. |
| 보장하지 않는 것 | Cross-browser 보장, exact visual baseline, WCAG 전체, production build artifact, disabled route와 모든 dynamic data state는 보장하지 않습니다. 한 worker이므로 병렬 route compilation 회귀도 탐지하지 않습니다. |
| 다음 commit/관련 test 연결 | `1598a87702f6`는 browser보다 빠른 renderer compatibility matrix를 추가하고, `84c71d...`가 이 matrix의 route/design vocabulary를 `site-matrix.ts`로 추출해 Axe suite와 공유합니다. `882a...`는 같은 Playwright projects를 snapshot baseline에 사용합니다. |

#### 최소 code evidence

- **Commit:** `31c438b52e4b5f87d7e88ce047dd1997aa8ef054`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `package.json`의 `test:e2e` script와 `@playwright/test` dependency; `playwright.config.ts`의 testDir, timeout, single worker, webServer와 `chromium`/`mobile-chrome` projects; `tests/e2e/portfolio.spec.ts` 내부의 design IDs, enabled route definitions와 content JSON fixture

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 2. `055b733cbb7e` — test(design): 독립 renderer와 design token 경계 검증

- **Full SHA:** `055b733cbb7e897df7c75c90164b99f5fd2d9724`
- **Importance:** A
- **Tags:** ARCH, VALIDATION, RENDERER
- **이 Thread에서의 역할:** Renderer marker와 design-token structural precondition

#### 해당 SHA에서 확인할 실제 코드

- `src/designs/route-view-models.test.tsx`에 Journey/Interview page를 추가한 부분
- Design/Classic에서만 `data-route-renderer`를 요구하는 branch와 `data-site-design` 공통 assertion
- `src/designs/design-tokens.test.ts`의 seven token-family list
- `src/app/globals.css`를 읽는 source-text test와 Design/Classic scope 안의 `--content-width` regex
- 이 Thread에서는 renderer independence 증거와 token test의 역할을 구분할 것

#### Commit-specific investigation

- `055b733cbb7e^`와 `055b733cbb7e`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Renderer marker와 design-token structural precondition`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: Visual Thread에서는 이 SHA가 snapshot보다 먼저 오는 token/boundary prerequisite로 다시 사용됩니다. 이후 `882a2f...`가 실제 pixels를 baseline으로 고정합니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | 기존 matrix는 six routes만 포함하고 Design과 Classic이 같은 presentation implementation으로 다시 합쳐져도 `data-site-design`만 맞으면 통과할 수 있었습니다. Journey/Interview coverage와 최소 token vocabulary도 별도 guard가 없었습니다. |
| 실제 변경 file/symbol/call path | Renderer matrix에 Journey와 Interview Map을 추가하여 40 combinations로 확장하고, Design/Classic root 아래 각각 `data-route-renderer` marker를 요구합니다. 별도 `design-tokens.test.ts`는 typography, spacing, breakpoint, motion, layer와 width token family가 존재하고 Design/Classic scope가 explicit한지 확인합니다. |
| Data/state/DOM/resource owner와 lifetime | Page/registry가 design dispatch를 소유하고 각 independent renderer가 marker를 출력합니다. CSS token 값의 owner는 `globals.css`; test는 source를 읽어 vocabulary와 explicit scope만 고정합니다. |
| Failure·absence·fallback·cleanup | Journey/Interview renderer 누락, Design/Classic implementation boundary collapse, 필수 token 선언 삭제를 deterministic failure로 만듭니다. Regex는 CSS cascade의 computed result나 token 값의 적절성을 평가하지 않습니다. |
| Test technique와 실행 증거 | jsdom route integration과 static CSS contract를 결합합니다. 이 Thread의 핵심 evidence는 route matrix의 두 추가 route와 `data-route-renderer` assertion입니다. |
| 보장하는 것 | 모든 enabled public route가 five-design dispatch에 참여하고 Design/Classic이 독립 renderer boundary를 유지한다는 최소 계약이 완성됩니다. |
| 보장하지 않는 것 | Editorial/Brutalist/Cinematic에 독립 marker를 요구하지 않고, token value·contrast·responsive behavior·pixel output은 보장하지 않습니다. Static source가 존재해도 unused token일 수 있습니다. |
| 다음 commit/관련 test 연결 | Visual Thread에서는 이 SHA가 snapshot보다 먼저 오는 token/boundary prerequisite로 다시 사용됩니다. 이후 `882a2f...`가 실제 pixels를 baseline으로 고정합니다. |

#### 최소 code evidence

- **Commit:** `055b733cbb7e897df7c75c90164b99f5fd2d9724`
- **Excerpt:** 생략했습니다. 이 commit의 핵심은 여러 assertion/configuration에 걸쳐 있어 일부 줄만 인용하면 test 범위를 왜곡할 수 있습니다.
- **대신 확인한 위치:** `src/designs/route-view-models.test.tsx`에 Journey/Interview page를 추가한 부분; Design/Classic에서만 `data-route-renderer`를 요구하는 branch와 `data-site-design` 공통 assertion; `src/designs/design-tokens.test.ts`의 seven token-family list

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

### 3. `882a2f9d753e` — test(visual): 다섯 디자인 회귀 기준 추가

- **Full SHA:** `882a2f9d753ea8ff97cc8ce6a202aeb0e394597d`
- **Importance:** A
- **Tags:** TEST
- **이 Thread에서의 역할:** Stable screenshot suite, 15 PNG baselines와 exact manifest contract

#### 해당 SHA에서 확인할 실제 코드

- `playwright.config.ts`와 `playwright.production.config.ts`의 shared `snapshotPathTemplate`
- `tests/e2e/visual.spec.ts`의 `prepareStablePage`: reduced motion, `networkidle`, `document.fonts.ready`, all image load/error settlement
- five design loop, home test의 desktop/mobile project assertion, project detail의 desktop-only skip
- `toHaveScreenshot` options: fullPage, disabled animations, `maxDiffPixelRatio: 0.01`
- `src/designs/visual-regression-contract.test.ts`가 `SITE_DESIGN_IDS`에서 exact 15-file manifest를 만드는 방식
- 추가된 PNG set: design당 home desktop/mobile 두 장과 project desktop 한 장

#### Commit-specific investigation

- `882a2f9d753e^`와 `882a2f9d753e`의 diff에서 위 파일·symbol이 실제로 추가·변경·제거된 범위를 구분합니다.
- 직전 state에서 `Stable screenshot suite, 15 PNG baselines와 exact manifest contract`가 필요해진 구체적 부족함 또는 잘못된 가정을 찾습니다.
- Production path와 test path를 분리하고, state/data/resource owner와 lifetime·cleanup을 실제 symbol 기준으로 추적합니다.
- Failure/absence/fallback branch와 test technique을 구분하고, 이 SHA가 보장하지 않는 범위를 명시합니다.
- 다음 후속 관계를 대조하되 later code를 이 SHA의 구현으로 설명하지 않습니다: 앞선 route matrix와 token/renderer structural tests를 pixel evidence로 보완합니다. Baseline update의 타당성은 자동으로 판단하지 않으므로 review ownership은 여전히 사람에게 남습니다.

#### 학습자 증거

| 확인·기록 항목 | 기록 |
| --- | --- |
| 직전 state와 부족함 | Route/browser characterization은 semantics와 layout invariants를 검사했지만 typography, spacing, wrapping, image crop처럼 DOM assertion으로 표현하기 어려운 변화는 놓칠 수 있었습니다. Baseline file이 빠지거나 과도하게 추가되어도 별도 contract가 없었습니다. |
| 실제 변경 file/symbol/call path | 두 Playwright config에 deterministic snapshot naming을 추가하고 five-design visual suite와 15개 PNG baseline을 커밋했습니다. Page 준비는 motion을 줄이고 network/font/image settlement를 기다립니다. 별도 Vitest test는 directory의 PNG manifest가 expected exact set과 일치해야 한다고 요구합니다. |
| Data/state/DOM/resource owner와 lifetime | Playwright project name과 snapshot template가 platform/viewport filename을 소유합니다. `SITE_DESIGN_IDS`가 design enumeration을 소유하고 first enabled project가 detail fixture를 결정합니다. Repository에 커밋된 PNG가 review baseline입니다. |
| Failure·absence·fallback·cleanup | Enabled project가 없으면 test module이 error를 던집니다. Image error도 settlement 대상으로 처리하므로 hang은 피하지만 broken image 자체를 이 helper가 실패시키지는 않습니다. Pixel difference가 1%를 넘으면 실패하고 manifest의 누락·추가도 실패합니다. |
| Test technique와 실행 증거 | Visual regression testing plus snapshot-manifest contract입니다. Full-page browser screenshots를 비교하지만 exact browser/font/platform reproducibility에 의존합니다. Baselines와 test code는 확인했으나 screenshot command는 실행하지 않았습니다. |
| 보장하는 것 | 다섯 design의 home desktop/mobile와 first project desktop에 대해 안정화 절차, filename, pixel tolerance와 exact baseline count가 repository contract가 됩니다. |
| 보장하지 않는 것 | 다른 public routes, project detail mobile, interaction/open states, hover/focus/forced-colors, multiple projects, all OS/browser rendering과 1% 이하의 local visual drift는 보장하지 않습니다. |
| 다음 commit/관련 test 연결 | 앞선 route matrix와 token/renderer structural tests를 pixel evidence로 보완합니다. Baseline update의 타당성은 자동으로 판단하지 않으므로 review ownership은 여전히 사람에게 남습니다. |

#### 최소 code evidence

- **Commit:** `882a2f9d753ea8ff97cc8ce6a202aeb0e394597d`
- **Path:** `tests/e2e/visual.spec.ts`
- **Location:** home visual test

```ts
await expect(page).toHaveScreenshot(`home-${designId}.png`, {
  animations: "disabled",
  fullPage: true,
  maxDiffPixelRatio: 0.01,
});
```

이 excerpt는 위 state/ownership/contract를 식별하는 데 필요한 부분만 남긴 것입니다.

#### 실행 증거

| 항목 | 기록 |
| --- | --- |
| 해당 SHA에서 실행한 command | 미실행 |
| 실제 결과 | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| 정적 검토 범위 | Exact commit diff, changed files, 위에 열거한 symbol/test/config |

## 6. Invariant evolution ledger

| Invariant | 도입/변경 SHA | Historical evidence | 상태 |
| --- | --- | --- | --- |
| Visual evidence는 actual browser route output에서 얻는다. | 31c438b52e4b | Playwright server/projects/route matrix | 기반 |
| Renderer/token boundary는 pixel diff보다 빠르게 구조적으로 실패한다. | 055b733cbb7e | Marker/token source tests | 보강 |
| Capture는 motion/font/image settlement 뒤 수행된다. | 882a2f9d753e | `prepareStablePage` | 도입 |
| Baseline set은 five design당 3개, 총 15개다. | 882a2f9d753e | PNG files + exact manifest Vitest | 고정 |

## 7. Failure → Fix → Test

| Earlier failure/risk | Fix SHA | Corrected decision | Regression evidence |
| --- | --- | --- | --- |
| Browser/viewport foundation 없이 local screenshot이 서로 다른 조건으로 생성될 위험 | 31c438b52e4b | Configured Chromium/mobile projects와 web server | Harness |
| Renderer extraction/token 삭제가 screenshot review 전까지 늦게 발견될 위험 | 055b733cbb7e | Structural marker/token tests | Fast regression |
| Font/image/animation timing 차이로 flaky pixel diff가 발생할 위험 | 882a2f9d753e | Reduced motion + network/font/image settlement | Stabilization |
| Baseline 누락·임의 추가로 coverage가 조용히 변할 위험 | 882a2f9d753e | Exact filename manifest | Manifest regression |

## 8. Ownership/state/responsibility 변화

| 대상 | 초기 owner/state | 최종 owner/state | Evidence |
| --- | --- | --- | --- |
| Route/design/viewport | E2E spec 내부 | Playwright config + matrix | Chromium/mobile projects |
| Renderer/token structure | 암묵적 | Static contract tests | `route-view-models`, `design-tokens` |
| Capture stabilization | 수동 timing 가능 | `prepareStablePage` | Visual spec |
| Reference pixels | 없음 | Committed PNGs | `visual.spec.ts-snapshots/` |
| Coverage manifest | Directory 관행 | Vitest exact set | `visual-regression-contract.test.ts` |

## 9. 최종 Thread state

다음 내용을 코드 없이 설명합니다: 최종 owner, input→decision→output, failure/absence policy, regression evidence와 명시적 non-guarantee.

최종 visual contract는 다섯 design의 home desktop/mobile와 first project desktop 15개 full-page PNG를 repository baseline으로 둡니다. Capture는 motion/network/font/image settlement를 거치며 1% pixel-diff tolerance를 사용하고, 별도 test가 exact filename set을 보호합니다. 다른 routes와 interaction states는 명시적으로 coverage 밖입니다.

## 10. 최종 실행 흐름

| 단계 | Owner / mechanism | Input | Output/state | Failure/non-guarantee |
| ---: | --- | --- | --- | --- |
| 1. Browser project | Playwright config | desktop/mobile project + web server | stable viewport/runtime | project/platform 차이 |
| 2. Route selection | `site-matrix` | design + home/first detail | explicit URL | enabled project 없음→error |
| 3. Page stabilization | `prepareStablePage` | loaded page | settled motion/fonts/images | image error는 settle만 함 |
| 4. Screenshot comparison | `toHaveScreenshot` | full page | ≤1% diff | threshold 초과 failure |
| 5. Manifest verification | `visual-regression-contract.test.ts` | snapshot directory + design IDs | exact 15 names | 누락/추가 failure |

## 11. 학습 완료 확인

- [x] 모든 referenced SHA를 exact historical diff 기준으로 설명했습니다.
- [x] Commit map의 SHA·순서·subject·importance·tags를 변경하지 않았습니다.
- [x] Fix를 earlier failure/assumption에 연결했습니다.
- [x] Test가 실행하는 production path와 증명하지 않는 범위를 구분했습니다.
- [x] 정적 inspection과 실제 command execution을 구분했습니다.
- [x] Thread-level invariant, ownership과 final flow를 완성했습니다.
- [x] 실행 제한을 명시했습니다: 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다.
===== END FILE: 06-visual-regression-and-responsive-baselines.md =====

===== BEGIN FILE: README.md =====
# Testing, performance, and regression strategy

## 범위

이 category는 production behavior를 검증·특성화하거나 client/runtime cost를 줄인 뒤 회귀 evidence로 고정하는 source-level engineering stories를 다룹니다.

- 포함: content/unit contracts, route characterization, component interaction와 hydration, browser accessibility matrix, server-first/client performance, visual regression.
- 제외: production bundle baseline, route growth budget, Lighthouse threshold, CI activation, standalone/Docker delivery. 이 연속 이력은 `08-product-delivery-and-runtime-verification/04-release-performance-gates.md` 및 category 08이 소유합니다.

## Phase 1 category audit 결과

- Category boundary는 적절합니다. Test/optimization 자체와 release enforcement를 분리해 category 08과 중복하지 않습니다.
- Thread 수와 filename은 6개로 유지했습니다. 독립 engineering story를 새로 합치거나 분리할 근거는 없었습니다.
- `03-component-interaction-and-hydration-regression.md`에 초기 selector ownership을 보여 주는 `e43e8addd7f3`, `c69ef85c98b2`를 앞에 추가했습니다.
- `06-visual-regression-and-responsive-baselines.md`는 actual chronology에 맞춰 `31c438b52e4b` → `055b733cbb7e` → `882a2f9d753e`로 고쳤습니다.
- `31c438b52e4b`는 route-browser foundation, accessibility foundation, visual foundation이라는 서로 다른 파일/검증 역할로 Threads 2·4·6에 의도적으로 재사용됩니다.
- `055b733cbb7e`는 renderer characterization과 visual structural precondition이라는 서로 다른 역할로 Threads 2·6에 재사용됩니다.
- 나머지 commit은 이동·삭제하지 않았고, 범용 조사 문구만 exact file/symbol/test/config 단위 질문으로 교체했습니다.
- Source 분류상 이 category의 고유 commit은 A-level 24개, B-level 1개이며 S/C-level은 없습니다. 중요도를 인위적으로 재분류하지 않았습니다.

## 동결된 Thread 순서

1. [Content contract test harness](01-content-contract-test-harness.md)
2. [Route presentation characterization](02-route-presentation-characterization.md)
3. [Component interaction and hydration regression](03-component-interaction-and-hydration-regression.md)
4. [Browser accessibility route matrix](04-browser-accessibility-route-matrix.md)
5. [Client performance and server-first optimization](05-client-performance-and-server-first-optimization.md)
6. [Visual regression and responsive baselines](06-visual-regression-and-responsive-baselines.md)

## Commit coverage

| 항목 | 값 |
| --- | ---: |
| Thread 수 | 6 |
| Commit 참조 수 | 28 |
| 고유 SHA 수 | 25 |
| A-level 고유 SHA | 24 |
| B-level 고유 SHA | 1 |
| 추가된 SHA | 2 |
| 제거된 SHA | 0 |
| 순서가 수정된 Thread | 1 |

## Phase 2 completion record

| 검증 항목 | 기록 |
| --- | --- |
| Scaffold/completed counterpart | README 포함 7개 파일이 1:1 대응하며 extra file 없음 |
| Frozen scaffold integrity | Phase 2 전후 SHA-256 manifest 동일 |
| Commit identity | 28개 참조의 short/full SHA, subject, importance, tags와 순서 보존 |
| Branch scope | Earliest referenced SHA가 `web/portfolio` head의 ancestor임을 branch compare로 확인하고, 25개 SHA 모두 branch-local classification/exact commit API에서 resolve |
| Historical evidence | 각 SHA의 exact commit diff/changed files만 사용; final HEAD를 earlier implementation으로 사용하지 않음 |
| Runtime tests | 미실행; 통과 주장 없음 |
| Execution limitation | 실행 환경에서 `github.com` DNS 해석이 실패하여 repository를 local clone하지 못했습니다. 따라서 이 문서의 역사 증거는 GitHub의 exact commit API에서 확인한 parent diff, changed file, 해당 SHA의 file content와 branch-local classification에 한정합니다. Vitest·Playwright·Axe·screenshot·performance command는 실행하지 않았고, 통과했다고 기록하지 않습니다. |
| Markdown/package | Heading/table/fence/placeholder/ZIP member validation 완료 |

## 문서 사용법

1. Thread 목표·invariant·commit map을 먼저 읽습니다.
2. 각 SHA를 parent와 비교하고 해당 SHA의 changed files/resulting locations만 설명합니다.
3. Fix는 이전 assumption과, test는 실제 production path 및 non-guarantee와 연결합니다.
4. Runtime command가 없으면 정적 inspection만 수행했다는 사실을 유지합니다.
5. 마지막에 invariant ledger, ownership transfer와 코드 없는 final flow로 학습을 마칩니다.
===== END FILE: README.md =====

