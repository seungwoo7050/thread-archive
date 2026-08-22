===== BEGIN FILE: 01-application-shell-auth-entry-and-navigation-identity.md =====
# Application shell·auth entry·navigation identity

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

Next.js browser runtime, 공통 layout/navigation, 개발 로그인 진입을 구성하고 현재 사용자 식별이 끝나기 전에는 잘못된 profile target을 노출하지 않도록 navigation identity를 안정화하는 과정을 복원합니다.

### 직접 연결되는 불변식

- `RootLayout`은 문서 metadata와 global style을, `AppShell`은 공통 navigation/layout을, route page는 인증 분기와 화면별 상태를 소유합니다.
- navigation item identity는 URL 변화와 분리된 논리적 `id`를 사용합니다.
- 현재 사용자를 알기 전 profile 항목은 `/` 같은 대체 URL로 이동하지 않습니다.

## 2. 핵심 질문

- Next.js runtime과 shared package 소비 경계는 어떤 설정 파일에서 확정됩니까?
- `RootLayout`, `AppShell`, `HomePage`, `LoginPanel`의 책임과 상태 수명은 어떻게 나뉩니까?
- session 해석 전후 profile URL 변경이 active state와 React key에 어떤 영향을 줍니까?
- 초기 E2E가 실제 API 성공과 sample fallback을 구분하지 못하는 범위는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f5c151c7cc7d` | `chore(web): Next.js runtime 경계 구성` | B | PROTOCOL, PERSISTENCE, WEB | web workspace를 Next.js application과 shared package 소비 경계로 구성합니다. |
| 2 | `4071b935cb24` | `chore(web): Tailwind style build 구성` | B | WEB | Tailwind/PostCSS source scan과 style processing 경계를 구성합니다. |
| 3 | `ce174d6b3633` | `feat(web): 한국어 로비 shell 초기화` | B | REALTIME, WEB | 한국어 metadata, design token, base style, focus-visible을 갖는 App Router shell을 만듭니다. |
| 4 | `77f35c72cd7b` | `feat(web): 공통 내비게이션 프레임 구현` | B | WEB, OPERATIONS | responsive sidebar, active route, content width를 `AppShell`이 소유합니다. |
| 5 | `f27199fdcd34` | `feat(web): 개발용 로그인 패널 추가` | B | AUTH, WEB | 개발 로그인 입력과 mutation 상태를 `LoginPanel`에 추가합니다. |
| 6 | `52ddc3acfcce` | `feat(web): 로비 인증 진입 연결` | B | AUTH, WEB | home route가 current session에 따라 LoginPanel과 authenticated lobby를 분기합니다. |
| 7 | `d755b8dae2c1` | `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성` | B | PERSISTENCE, TOURNAMENT, WEB | Playwright desktop/mobile browser 검증 경계를 구성합니다. |
| 8 | `ec000bed0414` | `build(web): production start와 TS cache 정책 구성` | B | WEB | web application에 production start와 package-local test command를 추가하고 type-check cache 생성을 끕니다. |
| 9 | `34eccd6c7150` | `feat(profile): 현재 프로필과 공유 기능 연결` | B | WEB | 현재 session을 읽어 profile navigation target과 profile 공유 기능을 연결합니다. |
| 10 | `a56a4dee9219` | `fix(web): 안정적인 navigation key 사용` | B | WEB | 현재 URL 대신 stable logical ID를 React key로 사용합니다. |
| 11 | `bc8d023b2999` | `fix(web): profile link 전 사용자 식별 대기` | B | WEB | current user가 resolve되기 전에 잘못된 profile link를 만들지 않습니다. |

## 5. Commit별 학습 기록

### 5.1. `chore(web): Next.js runtime 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `f5c151c7cc7d` |
| Importance | B |
| Tags | PROTOCOL, PERSISTENCE, WEB |
| Source에서 확정된 역할 | web workspace를 Next.js application과 shared package 소비 경계로 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/package.json`의 `dev`, `build`, `typecheck` script와 dependency를 확인합니다.
- `apps/web/next.config.mjs`의 `transpilePackages`와 `apps/web/tsconfig.json`의 path alias를 확인합니다.
- `next-env.d.ts` 외에 아직 route, production start, package-local test가 없는지 parent와 비교합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | web workspace가 실행 가능한 Next.js 애플리케이션으로 정의되지 않았고 shared package를 browser build가 소비하는 규칙도 없었습니다. |
| 해결하려던 문제 | 브라우저 코드를 둘 위치는 있었지만 개발 서버, production build, type-check, shared contract transpilation의 실행 경계가 없었습니다. |
| 핵심 결정 | `@pong-pong/web` package를 만들고 Next.js/React/TypeScript 설정을 묶었으며 `@pong-pong/shared`를 transpilation 대상으로 지정했습니다. |
| 입력 → 상태 전이 → 출력 | workspace command가 web package script를 호출하고 Next.js가 `src`와 shared package를 TypeScript 설정에 따라 compile합니다. |
| ownership/lifetime/cleanup | package script와 framework config가 build 책임을 소유합니다. 이 SHA에는 component resource나 teardown 대상이 없습니다. |
| failure/rollback/retry | 잘못된 설정은 build/type-check 단계에서 실패하지만 이 commit 자체에는 route나 runtime test가 없어 실제 화면 동작은 검증하지 않습니다. |
| 보장하는 것 | web application을 독립적으로 개발·build·type-check할 최소 runtime boundary를 보장합니다. |
| 보장하지 않는 것 | route, global style, production `start`, browser test, API 연결은 아직 보장하지 않습니다. |
| 후속 연결 | `4071b935cb24`가 style processing을, `ce174d6b3633`이 첫 App Router shell을 추가합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show f5c151c7cc7d^` 및 `git diff f5c151c7cc7d^ f5c151c7cc7d`로 비교합니다.
- Thread 내 다음 관련 SHA: `4071b935cb24` — `chore(web): Tailwind style build 구성`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `chore(web): Tailwind style build 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `4071b935cb24` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | Tailwind/PostCSS source scan과 style processing 경계를 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/postcss.config.mjs`의 Tailwind/Autoprefixer plugin 순서를 확인합니다.
- `apps/web/tailwind.config.ts`의 `content` glob이 `src/**/*.ts(x)`만 조사하는지 확인합니다.
- theme extension의 color/shadow token이 이후 global CSS와 component class에서 어떻게 소비되는지 추적합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Next.js build는 존재했지만 utility class를 CSS로 변환하고 실제 source 사용량을 수집하는 설정이 없었습니다. |
| 해결하려던 문제 | TSX에서 작성할 class가 production CSS에 포함될 근거가 필요했습니다. |
| 핵심 결정 | PostCSS에 Tailwind와 Autoprefixer를 연결하고 web `src` 트리만 scan하도록 제한했습니다. |
| 입력 → 상태 전이 → 출력 | TS/TSX source의 class token이 Tailwind scan 대상이 되고 PostCSS가 build 시 CSS를 생성합니다. |
| ownership/lifetime/cleanup | style build 설정이 source scan 범위를 소유합니다. runtime state나 cleanup은 없습니다. |
| failure/rollback/retry | glob 밖 파일의 class는 수집되지 않습니다. 이 commit은 접근성이나 실제 layout을 검증하지 않습니다. |
| 보장하는 것 | web source에서 사용하는 utility class와 project token을 build 결과에 포함할 수 있습니다. |
| 보장하지 않는 것 | 첫 화면, global reset, focus style은 아직 없습니다. |
| 후속 연결 | `ce174d6b3633`이 이 pipeline을 사용하는 global CSS와 layout을 추가합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `f5c151c7cc7d` — `chore(web): Next.js runtime 경계 구성`
- Thread 내 다음 관련 SHA: `ce174d6b3633` — `feat(web): 한국어 로비 shell 초기화`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `feat(web): 한국어 로비 shell 초기화`

| 항목 | 값 |
| --- | --- |
| SHA | `ce174d6b3633` |
| Importance | B |
| Tags | REALTIME, WEB |
| Source에서 확정된 역할 | 한국어 metadata, design token, base style, focus-visible을 갖는 App Router shell을 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/layout.tsx`의 `metadata`, `<html lang="ko">`, global CSS import를 확인합니다.
- `apps/web/src/app/globals.css`의 CSS variable, base element, `.card`, `.focus-ring` 규칙을 확인합니다.
- `apps/web/src/app/page.tsx`가 정적 shell만 렌더링하며 인증·server state를 아직 소유하지 않는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | framework와 style build는 있었지만 browser document, root layout, visible route가 없었습니다. |
| 해결하려던 문제 | 한국어 문서와 모든 route가 공유할 기본 시각·focus 규칙이 필요했습니다. |
| 핵심 결정 | `RootLayout`에 metadata와 `lang="ko"`를 두고 global tokens/base style을 한 번 import했습니다. |
| 입력 → 상태 전이 → 출력 | App Router가 root layout을 통해 document shell을 만들고 정적 `HomePage`를 children으로 렌더링합니다. |
| ownership/lifetime/cleanup | layout은 document-level metadata/style 수명을 소유하고 page는 route markup만 소유합니다. |
| failure/rollback/retry | CSS variable이나 class 오류는 compile/runtime styling 문제로 드러나지만 데이터 failure branch는 아직 없습니다. |
| 보장하는 것 | 모든 route가 한국어 document와 동일한 global visual/focus baseline을 공유합니다. |
| 보장하지 않는 것 | navigation, authenticated session, responsive application frame은 보장하지 않습니다. |
| 후속 연결 | `77f35c72cd7b`가 공통 navigation frame을 별도 component로 분리합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `4071b935cb24` — `chore(web): Tailwind style build 구성`
- Thread 내 다음 관련 SHA: `77f35c72cd7b` — `feat(web): 공통 내비게이션 프레임 구현`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `feat(web): 공통 내비게이션 프레임 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `77f35c72cd7b` |
| Importance | B |
| Tags | WEB, OPERATIONS |
| Source에서 확정된 역할 | responsive sidebar, active route, content width를 `AppShell`이 소유합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/components/AppShell.tsx`의 nav 배열, `usePathname`, active 판정, responsive markup을 확인합니다.
- 초기 profile href가 `/profile/tester`로 고정되어 있는지 확인합니다.
- sidebar의 “서버 준비 완료” 표현이 실제 health signal을 읽는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 각 route가 공통 navigation과 content width를 반복해야 하는 상태였습니다. |
| 해결하려던 문제 | responsive application frame과 active route 표시는 한 component가 일관되게 소유해야 했습니다. |
| 핵심 결정 | `AppShell`을 client component로 만들고 고정 nav 목록, pathname 기반 active 판정, desktop sidebar/mobile header를 구현했습니다. |
| 입력 → 상태 전이 → 출력 | `usePathname()` 결과와 각 item의 `href`/prefix를 비교해 class를 선택하고 children을 공통 main 영역에 렌더링합니다. |
| ownership/lifetime/cleanup | `AppShell`이 navigation definition과 responsive presentation을 소유합니다. profile target과 상태 문구도 당시에는 shell 내부 상수였습니다. |
| failure/rollback/retry | pathname prefix 판정은 동작하지만 fixed tester profile과 근거 없는 server-ready 문구를 그대로 표시합니다. |
| 보장하는 것 | route 간 공통 shell과 active navigation을 제공합니다. |
| 보장하지 않는 것 | 실제 current user identity나 server readiness와 일치한다는 보장은 없습니다. |
| 후속 연결 | `34eccd6c7150`, `a56a4dee9219`, `bc8d023b2999`가 profile target과 item identity를 단계적으로 바로잡습니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `ce174d6b3633` — `feat(web): 한국어 로비 shell 초기화`
- Thread 내 다음 관련 SHA: `f27199fdcd34` — `feat(web): 개발용 로그인 패널 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `feat(web): 개발용 로그인 패널 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `f27199fdcd34` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | 개발 로그인 입력과 mutation 상태를 `LoginPanel`에 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/components/LoginPanel.tsx`의 controlled `handle`, `displayName`, error state를 확인합니다.
- `devLogin` 호출 성공 시 parent callback으로 어떤 `SessionUser`를 전달하는지 확인합니다.
- pending 중 중복 제출 방지와 error message의 정보 범위를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 home route는 정적이며 인증된 사용자로 진입할 UI가 없었습니다. |
| 해결하려던 문제 | 개발 환경에서 browser가 인증 endpoint를 호출하고 성공 사용자 정보를 상위 화면에 전달할 진입점이 필요했습니다. |
| 핵심 결정 | 두 입력을 component local state로 관리하고 `devLogin` 성공 결과를 callback으로 넘겼습니다. |
| 입력 → 상태 전이 → 출력 | 사용자 click → async `devLogin(handle, displayName)` → 성공 시 parent callback, 실패 시 local error 표시 순서입니다. |
| ownership/lifetime/cleanup | 입력·pending·error는 panel instance가 소유합니다. component unmount 뒤 request 취소는 구현하지 않았습니다. |
| failure/rollback/retry | 실패는 하나의 일반 문구로 표시하며 structured error나 abort를 구분하지 않습니다. |
| 보장하는 것 | 개발 로그인 intent를 typed API helper에 연결합니다. |
| 보장하지 않는 것 | session 복원, cookie 정책, route 보호는 보장하지 않습니다. |
| 후속 연결 | `52ddc3acfcce`가 home route의 인증 분기와 연결합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `77f35c72cd7b` — `feat(web): 공통 내비게이션 프레임 구현`
- Thread 내 다음 관련 SHA: `52ddc3acfcce` — `feat(web): 로비 인증 진입 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `feat(web): 로비 인증 진입 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `52ddc3acfcce` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | home route가 current session에 따라 LoginPanel과 authenticated lobby를 분기합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 `getMe`/`getLobby` effect와 `me`, players, chat state 초기값을 확인합니다.
- login callback이 current user와 lobby reload를 어떤 순서로 수행하는지 확인합니다.
- request 실패 뒤 sample 초기값이 성공 화면으로 남는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 로그인 panel과 shell은 있었지만 home route가 현재 session을 해석하거나 authenticated 화면으로 전환하지 않았습니다. |
| 해결하려던 문제 | 새로고침과 로그인 성공 모두에서 current user를 판별하고 로비 데이터를 가져와야 했습니다. |
| 핵심 결정 | `HomePage`를 client component로 전환하고 mount effect에서 `getMe`와 `getLobby`를 호출해 인증 분기를 만들었습니다. |
| 입력 → 상태 전이 → 출력 | mount → session/lobby load → `me`가 없으면 `LoginPanel`, 있으면 shell 내부 authenticated content를 렌더링합니다. |
| ownership/lifetime/cleanup | page가 session 및 lobby read state를 직접 소유합니다. effect cancellation이나 generation guard는 없습니다. |
| failure/rollback/retry | 요청 실패를 catch하지만 sample players/chat 초기값을 비우지 않아 server failure가 정상 데이터처럼 보일 수 있습니다. |
| 보장하는 것 | home route가 인증 여부에 따라 진입 UI를 전환합니다. |
| 보장하지 않는 것 | 실패와 실제 sample 성공을 구별하거나 stale effect update를 막지는 않습니다. |
| 후속 연결 | `be31566ac0fd`가 sample fallback을 제거하고, 이후 query Thread가 effect ownership을 cache로 옮깁니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `f27199fdcd34` — `feat(web): 개발용 로그인 패널 추가`
- Thread 내 다음 관련 SHA: `d755b8dae2c1` — `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.7. `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `d755b8dae2c1` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT, WEB |
| Source에서 확정된 역할 | Playwright desktop/mobile browser 검증 경계를 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `playwright.config.ts`의 desktop/mobile project, base URL, trace/screenshot 조건을 확인합니다.
- `tests/e2e`에서 로그인 후 한국어 navigation과 route heading을 어떤 locator로 검증하는지 확인합니다.
- canvas pixel alpha 검사와 server/API 성공 여부의 관계를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | browser route와 canvas는 있었지만 실제 브라우저에서 navigation·responsive surface를 검증하는 suite가 없었습니다. |
| 해결하려던 문제 | 정적 markup을 넘어 사용자 진입, 한국어 메뉴, canvas drawing이 browser에서 존재하는지 확인할 필요가 있었습니다. |
| 핵심 결정 | Playwright runner와 desktop/mobile Chromium project를 추가하고 route 이동 및 canvas non-empty pixel을 검사했습니다. |
| 입력 → 상태 전이 → 출력 | test가 page를 열고 login UI를 조작한 뒤 navigation link를 따라가며 heading과 canvas pixel buffer를 관찰합니다. |
| ownership/lifetime/cleanup | Playwright context/page가 browser resource를 소유하고 runner가 test 종료 시 정리합니다. |
| failure/rollback/retry | failure trace/screenshot은 남기지만 app의 sample fallback 때문에 API 실패가 성공처럼 보이는 경우까지 배제하지 못합니다. |
| 보장하는 것 | 한국어 navigation과 canvas가 실제 browser DOM/canvas에서 작동하는 넓은 E2E 증거를 제공합니다. |
| 보장하지 않는 것 | server data의 진실성, 모든 route action, network failure branch는 증명하지 않습니다. |
| 후속 연결 | `8b2679d9e190`이 action 범위를 넓히고 `be31566ac0fd`가 sample-success 허점을 닫습니다. |

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | 한국어 application shell과 canvas route가 browser에서 접근·렌더링됩니다. |
| 재현한 실패/경계 | route가 누락되거나 menu locator가 바뀌거나 canvas가 실제로 그려지지 않는 경우입니다. |
| 테스트 기법 | Playwright desktop/mobile navigation과 canvas pixel alpha 검사입니다. |
| 증명하는 것 | DOM navigation, route heading, canvas non-empty drawing의 broad integration을 증명합니다. |
| 증명하지 않는 것 | API가 실제 server data를 반환했는지, sample fallback이 개입하지 않았는지는 증명하지 않습니다. |
| 검증 분류 | 브라우저 통합·시각 surface smoke evidence |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `52ddc3acfcce` — `feat(web): 로비 인증 진입 연결`
- Thread 내 다음 관련 SHA: `ec000bed0414` — `build(web): production start와 TS cache 정책 구성`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.8. `build(web): production start와 TS cache 정책 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `ec000bed0414` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | web application에 production start와 package-local test command를 추가하고 type-check cache 생성을 끕니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/package.json`의 `start`, `test` script가 추가되고 기존 `dev`/`build`/`typecheck`와 어떻게 구분되는지 확인합니다.
- `apps/web/tsconfig.json`의 `incremental: false`가 type-check 뒤 `.tsbuildinfo`를 남기지 않게 하는지 확인합니다.
- `next start --hostname 0.0.0.0 --port 3000`이 container/runtime에서 외부 interface에 bind하는 production 실행 계약인지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 web package는 개발 서버·build·type-check만 제공했고 production build를 실제로 serve하는 `start`와 workspace test command가 없었습니다. |
| 해결하려던 문제 | Compose/production runtime이 build 결과를 일정한 package script로 실행하고 root test orchestration이 web package를 건너뛰지 않게 해야 했습니다. |
| 핵심 결정 | `start`를 Next production server로, `test`를 no-test 상태도 정상인 Vitest command로 정의하고 incremental TypeScript cache 생성을 비활성화했습니다. |
| 입력 → 상태 전이 → 출력 | workspace/runtime command → web package `start` → Next build artifact serving; root test → package `test`; type-check → no emit/no incremental cache 순서입니다. |
| ownership/lifetime/cleanup | package scripts가 production process command를 소유합니다. 이 SHA는 process shutdown이나 artifact contents를 직접 검증하지 않습니다. |
| failure/rollback/retry | build artifact가 없으면 `next start`가 실패하고 test file이 생기면 Vitest가 이를 실행합니다. `--passWithNoTests`는 test 부재를 실패로 보지 않습니다. |
| 보장하는 것 | web package가 개발·build·production serve·type-check·test에 필요한 명시적 command surface를 갖습니다. |
| 보장하지 않는 것 | production artifact 완전성, container routing, browser behavior 자체는 보장하지 않습니다. |
| 후속 연결 | 이후 application shell/navigation 수정은 이 runtime command surface 위에서 계속됩니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `d755b8dae2c1` — `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성`
- Thread 내 다음 관련 SHA: `34eccd6c7150` — `feat(profile): 현재 프로필과 공유 기능 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.9. `feat(profile): 현재 프로필과 공유 기능 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `34eccd6c7150` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | 현재 session을 읽어 profile navigation target과 profile 공유 기능을 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppShell`의 `getMe` effect, `profileHref`, profile `matchPrefix`를 확인합니다.
- profile page의 clipboard API 호출과 성공/실패 notice를 확인합니다.
- session 해석 전 profile href가 무엇으로 설정되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shell의 profile link는 고정 tester handle을 가리켰고 profile 공유 버튼도 실제 browser API와 연결되지 않았습니다. |
| 해결하려던 문제 | 현재 로그인한 사용자에 맞는 profile target과 URL 복사 동작이 필요했습니다. |
| 핵심 결정 | `AppShell`이 `getMe`를 호출해 profile URL을 만들고 profile route는 `navigator.clipboard.writeText`를 사용했습니다. |
| 입력 → 상태 전이 → 출력 | shell mount → current session load → `profileHref` 계산; share click → current URL write → notice 갱신 순서입니다. |
| ownership/lifetime/cleanup | shell이 session lookup state를 직접 소유하고 profile page가 clipboard feedback을 소유합니다. |
| failure/rollback/retry | session 해석 전에는 profile href가 `/`로 대체되어 잘못된 navigation 가능성이 남고 clipboard failure는 notice로만 처리합니다. |
| 보장하는 것 | 고정 tester link를 current user handle 기반 link로 바꿉니다. |
| 보장하지 않는 것 | session 해석 전 안전한 disabled 상태와 stable key는 아직 보장하지 않습니다. |
| 후속 연결 | `a56a4dee9219`가 key를 안정화하고 `bc8d023b2999`가 unresolved profile navigation을 막습니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `ec000bed0414` — `build(web): production start와 TS cache 정책 구성`
- Thread 내 다음 관련 SHA: `a56a4dee9219` — `fix(web): 안정적인 navigation key 사용`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.10. `fix(web): 안정적인 navigation key 사용`

| 항목 | 값 |
| --- | --- |
| SHA | `a56a4dee9219` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | 현재 URL 대신 stable logical ID를 React key로 사용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppShell` nav item에 추가된 `id`와 `.map()`의 `key` 변경을 확인합니다.
- session resolve 전후 profile `href` 변경과 component identity의 관계를 비교합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | nav item key가 `href`였으므로 profile URL이 `/`에서 `/profile/<handle>`로 바뀌면 같은 논리 항목이 다른 React element로 재생성됐습니다. |
| 해결하려던 문제 | 비동기 session 해석이 navigation item identity까지 바꾸지 않아야 했습니다. |
| 핵심 결정 | 각 item에 불변 논리 ID를 추가하고 React key를 `href` 대신 `id`로 사용했습니다. |
| 입력 → 상태 전이 → 출력 | session data가 profile href만 갱신하고 React reconciliation은 동일 `id`를 가진 항목을 유지합니다. |
| ownership/lifetime/cleanup | navigation definition이 logical identity를 소유하며 URL은 mutable presentation 값으로 남습니다. |
| failure/rollback/retry | 잘못된 href 자체를 막지는 않습니다. unresolved 상태에서는 여전히 `/` 값이 존재합니다. |
| 보장하는 것 | session 해석 중에도 navigation item identity와 local DOM state를 유지합니다. |
| 보장하지 않는 것 | profile target의 유효성은 다음 fix 전까지 보장하지 않습니다. |
| 후속 연결 | `bc8d023b2999`가 unresolved item을 disabled presentation으로 바꿉니다. |

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | `href`가 navigation item identity로 충분하다고 가정했습니다. |
| 실제 실패/위험 | current session 해석으로 profile URL이 바뀔 때 같은 메뉴가 remount됩니다. |
| 근본 원인 | 논리 ID와 목적 URL을 같은 값으로 사용했습니다. |
| 수정된 불변식 | item identity는 고정 `id`, 목적지는 mutable `href`가 소유합니다. |
| 회귀 근거 | 이 SHA에는 별도 test가 없으며 code diff로만 확인됩니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `34eccd6c7150` — `feat(profile): 현재 프로필과 공유 기능 연결`
- Thread 내 다음 관련 SHA: `bc8d023b2999` — `fix(web): profile link 전 사용자 식별 대기`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.11. `fix(web): profile link 전 사용자 식별 대기`

| 항목 | 값 |
| --- | --- |
| SHA | `bc8d023b2999` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | current user가 resolve되기 전에 잘못된 profile link를 만들지 않습니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppShell`의 profile item rendering 분기와 `aria-disabled` 처리 여부를 확인합니다.
- unresolved profile item이 `<Link href="/">`가 아닌 비활성 element인지 확인합니다.
- session이 resolve된 뒤 같은 logical `id`로 link가 활성화되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | stable key는 확보했지만 current user를 모르는 동안 profile target이 `/`로 설정되어 lobby link처럼 동작했습니다. |
| 해결하려던 문제 | 아직 알 수 없는 identity를 유효한 URL로 가장하지 않아야 했습니다. |
| 핵심 결정 | profile URL이 없을 때 link 대신 `aria-disabled` 비활성 항목을 렌더링하고 session이 준비된 뒤 실제 link로 전환했습니다. |
| 입력 → 상태 전이 → 출력 | session query pending → disabled nav item; user resolve → `/profile/<handle>` link 생성 순서입니다. |
| ownership/lifetime/cleanup | shell이 profile target readiness와 navigation presentation을 소유합니다. |
| failure/rollback/retry | session request가 영구 실패하면 profile 항목은 비활성 상태로 남으며 별도 복구 UI는 없습니다. |
| 보장하는 것 | 사용자 식별 전 잘못된 profile route로 이동하지 않습니다. |
| 보장하지 않는 것 | session 자체의 인증 정확성이나 retry는 이 commit이 보장하지 않습니다. |
| 후속 연결 | query cache Thread에서 shell session lookup ownership이 `meQuery`로 통합됩니다. |

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | 사용자 미확정 상태를 `/` fallback으로 표현해도 된다고 봤습니다. |
| 실제 실패/위험 | profile 메뉴가 lobby로 이동하며 의미가 다른 action을 수행합니다. |
| 근본 원인 | unknown identity를 valid href로 변환했습니다. |
| 수정된 불변식 | profile identity가 없으면 navigation action 자체를 비활성화합니다. |
| 회귀 근거 | 별도 자동 test는 추가되지 않았고 rendering branch를 exact SHA에서 확인했습니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `a56a4dee9219` — `fix(web): 안정적인 navigation key 사용`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| runtime 경계 도입 | `f5c151c7cc7d` | Next.js/shared package build와 type-check 경계가 생겼습니다. |
| document·style owner 도입 | `4071b935cb24` → `ce174d6b3633` | style build와 `RootLayout`이 공통 browser shell의 기반을 소유합니다. |
| navigation owner 도입 | `77f35c72cd7b` | `AppShell`이 responsive navigation과 active route를 소유합니다. |
| production command 경계 | `ec000bed0414` | web package가 build 결과 serve와 package-local test command를 갖고 TypeScript cache를 남기지 않게 됐습니다. |
| auth entry 연결 | `f27199fdcd34` → `52ddc3acfcce` | 로그인 intent와 home route 인증 분기가 연결됐지만 sample fallback은 남았습니다. |
| navigation identity 교정 | `34eccd6c7150` → `a56a4dee9219` → `bc8d023b2999` | current profile URL, stable ID, unresolved disabled 상태가 순서대로 정리됐습니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| profile href를 React key로 사용 | session 해석 후 URL 변경으로 remount 가능 | `a56a4dee9219`가 logical ID를 분리 | 별도 test 없음 | profile href를 React key로 사용 → session 해석 후 URL 변경으로 remount 가능 → `a56a4dee9219`가 logical ID를 분리 → 별도 test 없음 |
| unknown user를 `/` href로 대체 | profile action이 lobby 이동으로 오인 | `bc8d023b2999`가 disabled item 렌더링 | 별도 test 없음 | unknown user를 `/` href로 대체 → profile action이 lobby 이동으로 오인 → `bc8d023b2999`가 disabled item 렌더링 → 별도 test 없음 |
| E2E가 sample fallback도 통과 | 실제 server failure를 성공처럼 볼 수 있음 | resource-screen Thread의 `be31566ac0fd` | 후속 화면 tests와 static branch inspection | E2E가 sample fallback도 통과 → 실제 server failure를 성공처럼 볼 수 있음 → resource-screen Thread의 `be31566ac0fd` → 후속 화면 tests와 static branch inspection |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| browser build | 없음 | `apps/web` package scripts + Next config | process command 수명 |
| document/global style | 없음 | `RootLayout` + `globals.css` | application document 수명 |
| navigation | route별 반복 가능 상태 | `AppShell` | mounted shell 수명 |
| login input/error | 없음 | `LoginPanel` local state | panel instance 수명 |
| current user/profile target | 고정 tester → page/shell effect | `AppShell` session state, 이후 query로 이전 예정 | session lookup 수명 |
| production web process command | 없음 | `apps/web/package.json`의 `start` | process 수명 |

## 9. Thread 최종 상태

마지막 SHA에서 web package는 개발·build·production start·type-check·test command를 제공하고, RootLayout은 한국어 문서/global style을, AppShell은 stable logical ID를 가진 공통 navigation을 소유합니다. HomePage/LoginPanel은 개발 인증 진입을 담당하며 current user가 확정되기 전 profile 항목은 이동 가능한 링크가 아닙니다. session과 resource state의 canonical ownership은 아직 component effect에 남아 있어 이후 API/query Thread에서 교체됩니다.

### 최종 실행 흐름

1. web workspace는 `dev`/`build`/`start`/`typecheck`/`test` command와 shared package transpilation 규칙으로 실행됩니다.
2. workspace command가 `@pong-pong/web`의 Next.js script를 실행합니다.
3. `RootLayout`이 metadata, `lang="ko"`, global CSS를 한 번 적용합니다.
4. `HomePage`가 current session을 조회하고 미인증이면 `LoginPanel`, 인증이면 `AppShell` 내부 화면을 선택합니다.
5. `AppShell`은 pathname과 stable nav `id`로 active item을 계산합니다.
6. profile identity가 없으면 비활성 item을, 준비되면 `/profile/<handle>` link를 렌더링합니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 01-application-shell-auth-entry-and-navigation-identity.md =====

===== BEGIN FILE: 02-browser-http-adapter-runtime-validation-and-cookie-credentials.md =====
# Browser HTTP adapter·runtime validation·cookie credentials

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

여러 화면이 공유하는 HTTP adapter를 만들고 body/header 규칙, structured error, runtime response parsing, AbortSignal, HttpOnly cookie와 one-time WebSocket ticket 경계로 발전시키는 과정을 복원합니다.

### 직접 연결되는 불변식

- durable session credential은 browser JavaScript가 저장하거나 읽지 않고 `credentials: "include"`로 cookie를 전달합니다.
- 성공 응답도 shared runtime schema를 통과해야 하며 malformed payload를 TypeScript 단언만으로 성공 처리하지 않습니다.
- body가 없는 request에는 adapter가 임의로 JSON content type을 붙이지 않습니다.
- WebSocket 연결은 durable credential 대신 짧은 수명의 one-time ticket을 AbortSignal과 함께 요청합니다.

## 2. 핵심 질문

- 초기 `apiFetch`가 request header, credential, generic response를 어떤 가정으로 처리합니까?
- body 없는 request와 caller-supplied header가 `Headers`에서 어떻게 보존됩니까?
- cookie-only 전환에서 token helper, Authorization header, WebSocket URL이 어떻게 제거됩니까?
- schema violation, structured API error, 401 session expiry, request abort가 각각 어떤 branch로 나뉩니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `20618b30eda9` | `feat(web): 인증 API client 구현` | B | AUTH, REALTIME, TOURNAMENT | authentication과 초기 read model용 공통 browser HTTP adapter를 구현합니다. |
| 2 | `bfae9539cfe5` | `feat(web): 사용자 동작용 API 함수 추가` | B | TOURNAMENT, WEB | tournament join, profile/friend, admin mutation adapter를 추가합니다. |
| 3 | `177fa0b8502a` | `fix(web): body 없는 요청에서 JSON header 제외` | B | AUTH, WEB | body 없는 요청에서 JSON content type을 선언하지 않도록 request header construction을 수정합니다. |
| 4 | `4bc5bba93c4a` | `test(web): API client 동작 검증` | B | AUTH, PROTOCOL, WEB | request headers/body, response parsing, error, abort behavior를 검증합니다. |
| 5 | `353ca9a17415` | `fix(web): browser token 저장 제거` | A | AUTH, PROTOCOL, REALTIME | browser-managed durable token을 제거하고 cookie-only HTTP 및 one-time WebSocket ticket 경계로 전환합니다. |
| 6 | `2aa5fbca9890` | `test(web): cookie 기반 API 경계 검증` | B | AUTH, REALTIME, WEB | cookie-only와 runtime-validated browser API 경계를 확장 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(web): 인증 API client 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `20618b30eda9` |
| Importance | B |
| Tags | AUTH, REALTIME, TOURNAMENT |
| Source에서 확정된 역할 | authentication과 초기 read model용 공통 browser HTTP adapter를 구현합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/api.ts`의 `API_BASE`, token storage helper, generic `apiFetch<T>`를 확인합니다.
- `fetch` option의 `credentials`, `Authorization`, `Content-Type` 설정 조건을 확인합니다.
- `devLogin`, `getMe`, `getLobby`, `getDashboard`, `getLeaderboard`, `getTournaments`의 반환 타입과 fallback을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 각 화면이 HTTP request를 직접 만들거나 아직 sample data만 사용했고 공통 인증·오류 처리가 없었습니다. |
| 해결하려던 문제 | 중복 request 설정을 줄이고 초기 service endpoint를 typed helper로 노출할 adapter가 필요했습니다. |
| 핵심 결정 | `apiFetch<T>`를 만들고 localStorage token, `credentials: "include"`, JSON header, generic response cast를 한곳에 결합했습니다. |
| 입력 → 상태 전이 → 출력 | helper 호출 → localStorage token 조회 → fetch → non-OK error → `response.json() as T` 반환 순서입니다. |
| ownership/lifetime/cleanup | browser localStorage가 durable token을, adapter가 request construction을 소유합니다. in-flight request cancellation은 없습니다. |
| failure/rollback/retry | 모든 request에 JSON header를 붙이고 성공 payload를 runtime 검증하지 않습니다. 일부 read helper는 실패 시 sample data를 반환합니다. |
| 보장하는 것 | 초기 endpoint 호출 방식과 오류 throw를 한 모듈로 통합합니다. |
| 보장하지 않는 것 | credential 노출 방지, body별 header 정확성, malformed response 거부는 보장하지 않습니다. |
| 후속 연결 | `177fa0b8502a`가 header 규칙을 고치고 `353ca9a17415`가 credential·parsing 경계를 재설계합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show 20618b30eda9^` 및 `git diff 20618b30eda9^ 20618b30eda9`로 비교합니다.
- Thread 내 다음 관련 SHA: `bfae9539cfe5` — `feat(web): 사용자 동작용 API 함수 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `feat(web): 사용자 동작용 API 함수 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `bfae9539cfe5` |
| Importance | B |
| Tags | TOURNAMENT, WEB |
| Source에서 확정된 역할 | tournament join, profile/friend, admin mutation adapter를 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `api.ts`의 `joinTournament`, `getProfile`, `requestFriend`, `setUserStatus` path/method/body를 확인합니다.
- 각 helper가 기존 generic `apiFetch`의 token/header/response 가정을 그대로 상속하는지 확인합니다.
- route parameter와 mutation body가 URL encoding 또는 JSON serialization되는 위치를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 공통 adapter는 login과 read model만 지원해 profile, tournament, admin UI가 실제 action을 보낼 수 없었습니다. |
| 해결하려던 문제 | 화면별 user intent를 동일한 request/error 규칙으로 연결할 endpoint helper가 필요했습니다. |
| 핵심 결정 | 동적 path와 JSON body를 조립하는 helper를 추가하고 기존 `apiFetch`를 재사용했습니다. |
| 입력 → 상태 전이 → 출력 | page action → endpoint helper → `apiFetch` → typed response 반환 순서입니다. |
| ownership/lifetime/cleanup | adapter가 endpoint URL/method/body serialization을 소유하고 screen은 user intent와 local feedback을 소유합니다. |
| failure/rollback/retry | 기반 adapter의 unchecked response와 browser token 문제는 그대로 전파됩니다. |
| 보장하는 것 | profile/friend, tournament, admin action을 한 공통 HTTP boundary로 보냅니다. |
| 보장하지 않는 것 | mutation invalidation, abort, runtime schema는 아직 보장하지 않습니다. |
| 후속 연결 | `051eac1b4aee`, `bfea82733512`, `a4f665fd2999`가 이 helper를 화면에 연결합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `20618b30eda9` — `feat(web): 인증 API client 구현`
- Thread 내 다음 관련 SHA: `177fa0b8502a` — `fix(web): body 없는 요청에서 JSON header 제외`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `fix(web): body 없는 요청에서 JSON header 제외`

| 항목 | 값 |
| --- | --- |
| SHA | `177fa0b8502a` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | body 없는 요청에서 JSON content type을 선언하지 않도록 request header construction을 수정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apiFetch`가 plain object 대신 `Headers`를 생성하는 변경을 확인합니다.
- `init.body !== undefined`와 caller-supplied `content-type` 조건을 확인합니다.
- Authorization header 추가가 content type 조건과 독립적인지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 adapter는 GET과 body 없는 POST에도 `Content-Type: application/json`을 무조건 붙였습니다. |
| 해결하려던 문제 | 실제 body representation과 header가 불일치하면 preflight/cache/server parsing 동작을 불필요하게 바꿀 수 있었습니다. |
| 핵심 결정 | caller header를 `Headers`로 정규화하고 body가 있으면서 content type이 없을 때만 JSON을 설정했습니다. |
| 입력 → 상태 전이 → 출력 | request init → `Headers(init.headers)` → 조건부 JSON header → 조건부 Authorization → fetch 순서입니다. |
| ownership/lifetime/cleanup | adapter가 최종 header set을 소유하지만 caller가 명시한 content type은 보존합니다. |
| failure/rollback/retry | body가 문자열이라고 해서 항상 JSON인지 검증하지 않으며 runtime response cast 문제도 남습니다. |
| 보장하는 것 | body 없는 request가 JSON payload를 보낸다고 거짓 선언하지 않습니다. |
| 보장하지 않는 것 | cookie-only credential과 response schema 검증은 보장하지 않습니다. |
| 후속 연결 | `4bc5bba93c4a`가 이 규칙을 unit test로 고정하고 `353ca9a17415`가 token을 제거합니다. |

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | 공통 API request는 모두 JSON이라고 가정했습니다. |
| 실제 실패/위험 | GET/body-less POST까지 JSON content type을 선언해 request 의미가 실제 body와 불일치합니다. |
| 근본 원인 | header construction이 body 존재 여부와 분리되어 있었습니다. |
| 수정된 불변식 | adapter는 body가 있을 때만 기본 JSON content type을 추가하고 caller 값을 덮지 않습니다. |
| 회귀 근거 | `4bc5bba93c4a`의 header test가 후속 증거입니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `bfae9539cfe5` — `feat(web): 사용자 동작용 API 함수 추가`
- Thread 내 다음 관련 SHA: `4bc5bba93c4a` — `test(web): API client 동작 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `test(web): API client 동작 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4bc5bba93c4a` |
| Importance | B |
| Tags | AUTH, PROTOCOL, WEB |
| Source에서 확정된 역할 | request headers/body, response parsing, error, abort behavior를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/api.test.ts`의 fake Storage와 mocked fetch setup을 확인합니다.
- token/credentials/header/body, non-OK error, endpoint envelope test가 실제 어떤 production helper를 호출하는지 확인합니다.
- 당시 test가 runtime schema가 아니라 JSON object equality만 확인하는 한계를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | adapter의 token·header·error 규칙이 code inspection에만 의존했습니다. |
| 해결하려던 문제 | 공통 helper의 작은 변경이 모든 browser endpoint에 퍼지므로 deterministic regression이 필요했습니다. |
| 핵심 결정 | storage와 fetch를 mock하고 대표 endpoint를 통해 request init과 반환/error를 관찰했습니다. |
| 입력 → 상태 전이 → 출력 | test helper 호출 → mocked fetch call 기록 → URL/init/return 또는 rejection assertion 순서입니다. |
| ownership/lifetime/cleanup | test가 mock lifecycle을 소유하고 production adapter는 실제 branch를 그대로 실행합니다. |
| failure/rollback/retry | 네트워크/browser cookie stack은 실행하지 않으며 당시 token storage 자체를 올바른 계약으로 전제합니다. |
| 보장하는 것 | 초기 adapter의 header/body/credential/error behavior를 deterministic하게 고정합니다. |
| 보장하지 않는 것 | malformed successful payload 거부와 cookie-only 인증은 증명하지 않습니다. |
| 후속 연결 | `353ca9a17415`가 전제를 바꾸므로 `2aa5fbca9890`에서 test contract도 다시 작성됩니다. |

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | 공통 adapter가 당시 정의된 token, credentials, header, error 규칙을 모든 helper에 적용합니다. |
| 재현한 실패/경계 | body 없는 request의 header, non-OK response, endpoint envelope 처리입니다. |
| 테스트 기법 | fake Storage와 mocked `fetch` call inspection입니다. |
| 증명하는 것 | browser-independent deterministic request construction을 증명합니다. |
| 증명하지 않는 것 | 실제 cookie 보안 속성, server runtime schema, browser CORS 동작은 증명하지 않습니다. |
| 검증 분류 | adapter 단위 회귀 테스트 |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `177fa0b8502a` — `fix(web): body 없는 요청에서 JSON header 제외`
- Thread 내 다음 관련 SHA: `353ca9a17415` — `fix(web): browser token 저장 제거`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `fix(web): browser token 저장 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `353ca9a17415` |
| Importance | A |
| Tags | AUTH, PROTOCOL, REALTIME |
| Source에서 확정된 역할 | browser-managed durable token을 제거하고 cookie-only HTTP 및 one-time WebSocket ticket 경계로 전환합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/api.ts`에서 token storage/get/set와 Authorization header가 제거되는지 확인합니다.
- `apiFetch`의 shared schema parameter, `safeParse`/parse failure, `ApiError`, session-expired event를 확인합니다.
- `requestWsTicket(signal)`과 play/lobby socket URL이 raw session token 대신 ticket을 사용하는지 확인합니다.
- ticket request의 `AbortController`, 교체 시 abort, stale completion 방지 경로를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | durable session token을 localStorage에서 JavaScript가 읽고 HTTP Authorization과 WebSocket query에 재사용했습니다. 성공 응답도 generic cast만 거쳤습니다. |
| 해결하려던 문제 | XSS가 durable credential을 읽을 수 있고, WebSocket URL에 장기 credential이 노출되며, malformed success payload가 내부 state로 들어오는 복합 위험이 있었습니다. |
| 핵심 결정 | HTTP는 HttpOnly cookie만 전달하고, WebSocket은 authenticated HTTP로 받은 짧은 one-time ticket을 사용하며, 모든 성공 응답을 shared runtime schema로 검증하도록 바꿨습니다. |
| 입력 → 상태 전이 → 출력 | endpoint helper → `apiFetch(path, schema, init)` → cookie 포함 fetch → non-OK structured `ApiError` 또는 schema parse → typed value; socket은 `requestWsTicket(signal)` → ticket URL → 연결 순서입니다. |
| ownership/lifetime/cleanup | browser는 durable credential을 소유하지 않습니다. adapter가 response trust boundary와 session-expired event를, caller/client가 AbortController와 socket lifetime을 소유합니다. |
| failure/rollback/retry | 401은 session-expired event를 발생시키고 structured error를 보존합니다. abort는 caller가 구분할 수 있게 전달합니다. schema mismatch는 성공으로 반환하지 않습니다. |
| 보장하는 것 | JavaScript가 durable session secret을 저장하지 않고 HTTP/WS credential 역할이 분리되며 successful response shape가 runtime에 강제됩니다. |
| 보장하지 않는 것 | server가 cookie/ticket을 안전하게 발급·소비하는지 자체적으로 증명하지 않으며, 모든 caller가 event를 올바르게 처리한다는 보장도 없습니다. |
| 후속 연결 | `2aa5fbca9890`이 schema, error, abort, 모든 endpoint helper를 새 계약으로 검증하고 transport Thread가 ticket cancellation을 더 깊게 다룹니다. |

#### A-level 불변식 종합

- **핵심 책임:** browser-managed durable token을 제거하고 cookie-only HTTP 및 one-time WebSocket ticket 경계로 전환합니다.
- **실패 영향:** XSS가 durable credential을 읽을 수 있고, WebSocket URL에 장기 credential이 노출되며, malformed success payload가 내부 state로 들어오는 복합 위험이 있었습니다.
- **결과 불변식:** JavaScript가 durable session secret을 저장하지 않고 HTTP/WS credential 역할이 분리되며 successful response shape가 runtime에 강제됩니다.
- **남은 제한:** server가 cookie/ticket을 안전하게 발급·소비하는지 자체적으로 증명하지 않으며, 모든 caller가 event를 올바르게 처리한다는 보장도 없습니다.

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | localStorage token 하나를 HTTP와 WebSocket에 재사용해도 된다고 가정했습니다. |
| 실제 실패/위험 | durable credential이 JavaScript와 URL에 노출되고 successful payload도 검증되지 않았습니다. |
| 근본 원인 | session identity, realtime admission, response typing을 한 browser token/generic cast에 결합했습니다. |
| 수정된 불변식 | durable identity는 HttpOnly cookie, realtime admission은 one-time ticket, response trust는 runtime schema가 각각 소유합니다. |
| 회귀 근거 | `2aa5fbca9890`의 cookie/schema/AbortSignal table-driven tests입니다. |

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `353ca9a17415` |
| 파일 | `apps/web/src/lib/api.ts` |
| 함수/위치 | `apiFetch / requestWsTicket` |
| 근거 요약 | HTTP 요청은 `credentials: "include"`를 사용하고 성공 payload는 endpoint별 schema로 parse합니다. `requestWsTicket`은 선택적 `AbortSignal`을 그대로 fetch에 전달합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `4bc5bba93c4a` — `test(web): API client 동작 검증`
- Thread 내 다음 관련 SHA: `2aa5fbca9890` — `test(web): cookie 기반 API 경계 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `test(web): cookie 기반 API 경계 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `2aa5fbca9890` |
| Importance | B |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | cookie-only와 runtime-validated browser API 경계를 확장 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `api.test.ts`에서 Authorization header 부재와 `credentials: "include"`를 확인합니다.
- schema-invalid 2xx, structured non-2xx, 401 session-expired event, aborted fetch를 각각 확인합니다.
- `it.each` endpoint table이 모든 helper의 runtime parsing과 signal 전달을 실행하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production adapter는 cookie/schema/ticket 경계로 바뀌었지만 기존 token 중심 test는 새 불변식을 충분히 설명하지 못했습니다. |
| 해결하려던 문제 | 보안 전환의 성공 조건과 negative branch를 각 endpoint에서 deterministic하게 고정해야 했습니다. |
| 핵심 결정 | mocked fetch 응답을 success, malformed success, structured error, abort로 바꾸고 모든 endpoint helper를 table-driven 방식으로 호출했습니다. |
| 입력 → 상태 전이 → 출력 | helper call → mocked response/signal → schema parse 또는 `ApiError`/abort → event 및 call init assertion 순서입니다. |
| ownership/lifetime/cleanup | test가 fetch mock과 event listener를 소유하며 production adapter의 실제 parse/error branch를 실행합니다. |
| failure/rollback/retry | 실제 browser cookie jar와 server ticket consumption은 실행하지 않습니다. mocked fetch가 전달 계약만 증명합니다. |
| 보장하는 것 | Authorization 부재, cookie 전달, schema fail-closed, structured error, session expiry, AbortSignal forwarding을 증명합니다. |
| 보장하지 않는 것 | cross-origin deployment와 실제 WebSocket handshake 성공은 증명하지 않습니다. |
| 후속 연결 | 이 test가 Thread의 cookie-only adapter invariant를 회귀로 보호합니다. |

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | HTTP credential은 cookie이며 successful payload는 endpoint schema를 통과하고 request cancellation이 caller signal을 따릅니다. |
| 재현한 실패/경계 | malformed 2xx, structured error, 401, abort, endpoint별 잘못된 envelope입니다. |
| 테스트 기법 | mocked fetch, event listener, table-driven endpoint helper 호출입니다. |
| 증명하는 것 | adapter 내부의 fail-closed parsing과 request init을 deterministic하게 증명합니다. |
| 증명하지 않는 것 | 실제 Set-Cookie 속성, CORS, server-side ticket one-time consumption은 증명하지 않습니다. |
| 검증 분류 | runtime contract·boundary unit regression |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `353ca9a17415` — `fix(web): browser token 저장 제거`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| 공통 adapter 도입 | `20618b30eda9` | request construction은 통합됐지만 browser token과 unchecked cast를 사용했습니다. |
| endpoint surface 확장 | `bfae9539cfe5` | profile/tournament/admin action도 같은 불완전한 adapter 계약을 공유했습니다. |
| header 의미 교정 | `177fa0b8502a` → `4bc5bba93c4a` | body 유무와 JSON header가 일치하고 초기 regression test가 생겼습니다. |
| credential·trust 경계 재설계 | `353ca9a17415` | HttpOnly cookie, one-time WS ticket, runtime schema, structured error로 역할을 분리했습니다. |
| 새 경계 검증 | `2aa5fbca9890` | negative schema/error/abort와 모든 endpoint helper가 새 invariant에 맞는지 고정했습니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| 모든 request에 JSON header | body 없는 요청 의미 불일치 | `177fa0b8502a` | `4bc5bba93c4a` | 모든 request에 JSON header → body 없는 요청 의미 불일치 → `177fa0b8502a` → `4bc5bba93c4a` |
| localStorage bearer token과 unchecked cast | durable credential 노출·malformed success 수용 | `353ca9a17415` | `2aa5fbca9890` | localStorage bearer token과 unchecked cast → durable credential 노출·malformed success 수용 → `353ca9a17415` → `2aa5fbca9890` |
| ticket request가 교체 중 완료 | stale socket 생성 가능 | `353ca9a17415`에서 AbortSignal 도입, transport client에서 generation fencing | `b5691b01a09b` | ticket request가 교체 중 완료 → stale socket 생성 가능 → `353ca9a17415`에서 AbortSignal 도입, transport client에서 generation fencing → `b5691b01a09b` |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| durable session | browser localStorage | HttpOnly cookie/server | session cookie lifetime |
| HTTP request construction | 화면별 또는 초기 generic helper | `apiFetch` | request lifetime |
| response trust | TypeScript generic cast | endpoint별 shared runtime schema | response parse lifetime |
| realtime credential | durable token query | one-time ticket response | 짧은 ticket TTL/한 번 소비 |
| cancellation | 없음 | caller AbortController → fetch signal | caller operation lifetime |

## 9. Thread 최종 상태

마지막 SHA에서 browser HTTP adapter는 durable token을 저장하거나 Authorization으로 재전송하지 않습니다. 모든 request는 cookie를 포함하고 body가 있을 때만 기본 JSON header를 붙이며, 성공 payload도 endpoint별 shared schema로 검증합니다. WebSocket은 abort 가능한 ticket request를 통해 one-time credential을 얻습니다.

### 최종 실행 흐름

1. screen/hook이 endpoint helper를 호출하고 필요하면 `AbortSignal`을 전달합니다.
2. `apiFetch`가 caller headers를 보존하면서 body가 있는 경우에만 기본 JSON content type을 추가합니다.
3. `fetch`는 `credentials: "include"`로 HttpOnly cookie를 자동 전달합니다.
4. non-OK 응답은 structured `ApiError`로 변환되고 401은 session-expired event를 발생시킵니다.
5. 2xx payload는 shared schema를 통과해야 typed value가 caller에 반환됩니다.
6. realtime caller는 `requestWsTicket(signal)` 결과의 one-time ticket으로 socket을 엽니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 02-browser-http-adapter-runtime-validation-and-cookie-credentials.md =====

===== BEGIN FILE: 03-resource-screens-actions-and-truthful-server-state.md =====
# Resource screens·actions·truthful server state

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

lobby, dashboard, leaderboard, tournament, profile, admin 화면을 server read/action에 연결하고 request failure를 sample success로 위장하던 초기 설계를 loading/error/empty state로 교정하는 과정을 복원합니다.

### 직접 연결되는 불변식

- server-backed screen은 request 실패나 미확정 상태를 sample user, ranking, tournament, chat으로 대체하지 않습니다.
- 각 route는 route parameter와 user intent를 소유하되 resource의 진실성은 API response에서만 얻습니다.
- mutation 성공·실패 feedback은 실제 response 결과와 연결되며 inert control을 성공처럼 표시하지 않습니다.

## 2. 핵심 질문

- 각 초기 screen의 sample initial state가 실제 request failure를 어떻게 가렸습니까?
- profile handle, selected tournament, admin target user가 mutation 입력으로 어떻게 전달됩니까?
- loading, error, empty, success가 어떤 state 조합과 rendering branch로 구분됩니까?
- 초기 E2E action test가 sample fallback을 허용한 구체적 assertion은 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ea1f1b7ba543` | `feat(web): 로그인 사용자 로비 화면 구성` | B | REALTIME, WEB, OPERATIONS | authenticated home route를 lobby read model과 action을 갖는 화면으로 만듭니다. |
| 2 | `cbe876359d31` | `feat(web): 플레이어 대시보드 구현` | B | WEB | dashboard read model을 표시하는 route를 만듭니다. |
| 3 | `cb295396771f` | `feat(web): 순위표 화면 추가` | B | WEB | leaderboard projection을 표시합니다. |
| 4 | `4370ac3162b2` | `feat(web): 토너먼트 대진표 화면 추가` | B | TOURNAMENT, WEB | tournament list/create를 연결한 초기 bracket screen을 만듭니다. |
| 5 | `0afc0a0694bd` | `feat(web): 공개 프로필 화면 추가` | B | WEB | handle-scoped profile route를 만듭니다. |
| 6 | `5e11e944244d` | `feat(web): 관리자 화면 추가` | B | WEB | user/status를 표시하는 protected admin screen을 만듭니다. |
| 7 | `051eac1b4aee` | `feat(profile): 친구 요청 동작 연결` | B | AUTH, WEB | profile target을 friendship mutation에 연결합니다. |
| 8 | `bfea82733512` | `feat(admin): 사용자 상태 변경 동작 연결` | B | AUTH, WEB | admin status control을 authenticated mutation에 연결합니다. |
| 9 | `a4f665fd2999` | `feat(tournament): 생성과 참가 동작 연결` | B | AUTH, TOURNAMENT | selected tournament state와 create/join action을 연결합니다. |
| 10 | `8b2679d9e190` | `test(e2e): 화면 action의 실제 API 연결 검증` | B | REALTIME, TOURNAMENT, WEB | AI 시작, match chat, friendship, tournament, admin action을 browser에서 검증합니다. |
| 11 | `e0ef3fec89a6` | `feat(chat): 로비 채팅 입력 화면 추가` | B | WEB | 로비 채팅 writer와 bounded local history를 HomePage에 연결하고 lobby read fallback 일부를 제거합니다. |
| 12 | `4f9b3b312d0e` | `fix(lobby): 로비 상태 표현 개선` | B | REALTIME | 고정 wait/주간 증가 문구를 제거하고 server `LobbyStats`를 화면 지표의 source로 사용합니다. |
| 13 | `cd3787eefd6a` | `feat(chat): 로비 채팅과 접속 상태 실시간 반영` | B | AUTH, REALTIME, WEB | HomePage가 lobby WebSocket을 직접 소유해 chat fan-out과 presence-triggered reload를 local state에 반영합니다. |
| 14 | `be31566ac0fd` | `fix(web): 로그인 화면의 sample fallback 제거` | A | AUTH, TOURNAMENT, WEB | authenticated/server-backed screen이 failure를 sample success data로 대체하지 않게 합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(web): 로그인 사용자 로비 화면 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `ea1f1b7ba543` |
| Importance | B |
| Tags | REALTIME, WEB, OPERATIONS |
| Source에서 확정된 역할 | authenticated home route를 lobby read model과 action을 갖는 화면으로 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 players/chat/stat state와 `getLobby` 호출을 확인합니다.
- `StatCard`, player list, lobby chat, fixed wait/progress copy가 실제 server field와 연결되는지 확인합니다.
- sample initial value와 catch branch가 화면에 남기는 상태를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 인증 분기는 있었지만 authenticated content는 단순 요약 수준이었고 lobby resource를 충분히 표시하지 않았습니다. |
| 해결하려던 문제 | 로그인 사용자가 online player, chat, matchmaking 진입 정보를 한 화면에서 볼 필요가 있었습니다. |
| 핵심 결정 | 공통 shell 내부에 hero, stat cards, player list, chat surface를 구성하고 lobby read를 연결했습니다. |
| 입력 → 상태 전이 → 출력 | session 확인 → lobby request → page local players/chat 갱신 → cards/list rendering 순서입니다. |
| ownership/lifetime/cleanup | page가 lobby resource와 presentation을 함께 소유하고 child stat card는 표시만 담당합니다. |
| failure/rollback/retry | request failure 시 sample players/chat가 남고 “30초”, progress copy 일부는 server와 무관한 상수입니다. |
| 보장하는 것 | authenticated lobby의 주요 presentation surface를 제공합니다. |
| 보장하지 않는 것 | 표시 값이 모두 authoritative server state라는 보장은 없습니다. |
| 후속 연결 | `be31566ac0fd`가 sample fallback을 제거하고 query Thread가 server state ownership을 이전합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show ea1f1b7ba543^` 및 `git diff ea1f1b7ba543^ ea1f1b7ba543`로 비교합니다.
- Thread 내 다음 관련 SHA: `cbe876359d31` — `feat(web): 플레이어 대시보드 구현`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `feat(web): 플레이어 대시보드 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `cbe876359d31` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | dashboard read model을 표시하는 route를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/dashboard/page.tsx`의 `getDashboard` effect와 sample initial state를 확인합니다.
- rating/win/loss/recent match field가 실제 response에서 읽히는지 확인합니다.
- SVG rating graph가 response history가 아니라 고정 coordinate인지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | lobby 밖에서 개인 전적과 최근 경기를 보는 route가 없었습니다. |
| 해결하려던 문제 | dashboard projection을 browser route로 제공할 필요가 있었습니다. |
| 핵심 결정 | sample dashboard를 initial state로 두고 mount effect에서 `getDashboard` 결과로 교체하는 화면을 추가했습니다. |
| 입력 → 상태 전이 → 출력 | mount → request → state replace → summary/recent match render 순서입니다. |
| ownership/lifetime/cleanup | page가 dashboard fetch와 data를 직접 소유합니다. |
| failure/rollback/retry | request 실패를 무시해 sample 전적이 남고 SVG graph는 server history를 반영하지 않습니다. |
| 보장하는 것 | dashboard route와 기본 read model presentation을 제공합니다. |
| 보장하지 않는 것 | 실패·loading·실제 history graph의 정확성은 보장하지 않습니다. |
| 후속 연결 | `be31566ac0fd`가 null/loading/error/empty branch로 교정합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `ea1f1b7ba543` — `feat(web): 로그인 사용자 로비 화면 구성`
- Thread 내 다음 관련 SHA: `cb295396771f` — `feat(web): 순위표 화면 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `feat(web): 순위표 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `cb295396771f` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | leaderboard projection을 표시합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/leaderboard/page.tsx`의 `getLeaderboard` effect와 sample entries를 확인합니다.
- rank, display name, rating, wins/losses rendering과 key를 확인합니다.
- request failure와 empty ranking이 구분되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 공개 ranking projection을 표시할 browser route가 없었습니다. |
| 해결하려던 문제 | 서버 leaderboard를 table/card 형태로 노출할 필요가 있었습니다. |
| 핵심 결정 | sample entries를 초기값으로 사용하고 effect가 성공하면 server entries로 교체했습니다. |
| 입력 → 상태 전이 → 출력 | mount → leaderboard request → local array replace → rank rows render 순서입니다. |
| ownership/lifetime/cleanup | page가 read request와 ranking array를 소유합니다. |
| failure/rollback/retry | 실패 시 sample ranking이 유지되어 public data가 존재하는 것처럼 보입니다. |
| 보장하는 것 | leaderboard route와 success presentation을 제공합니다. |
| 보장하지 않는 것 | failure와 genuine empty leaderboard의 차이는 보장하지 않습니다. |
| 후속 연결 | `be31566ac0fd`가 빈 초기값과 명시적 error/empty branch를 추가합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `cbe876359d31` — `feat(web): 플레이어 대시보드 구현`
- Thread 내 다음 관련 SHA: `4370ac3162b2` — `feat(web): 토너먼트 대진표 화면 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `feat(web): 토너먼트 대진표 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `4370ac3162b2` |
| Importance | B |
| Tags | TOURNAMENT, WEB |
| Source에서 확정된 역할 | tournament list/create를 연결한 초기 bracket screen을 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/tournaments/page.tsx`의 sample tournaments, selected item, create control을 확인합니다.
- bracket column이 첫 tournament의 entries를 `slice`해 임의 배치하는지 확인합니다.
- create button이 실제 API mutation인지 local-only인지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 토너먼트 list와 bracket을 보는 route가 없었습니다. |
| 해결하려던 문제 | 토너먼트 상태를 탐색하고 생성 intent를 표현할 초기 화면이 필요했습니다. |
| 핵심 결정 | sample tournament를 기반으로 list와 bracket-shaped presentation을 만들고 초기 create control을 배치했습니다. |
| 입력 → 상태 전이 → 출력 | page state에서 tournament/selection을 읽어 entries를 잘라 bracket card로 렌더링합니다. |
| ownership/lifetime/cleanup | page가 selected tournament와 임시 bracket projection을 소유합니다. |
| failure/rollback/retry | entry slicing은 실제 bracket match contract가 아니며 request failure를 sample로 가립니다. |
| 보장하는 것 | 초기 tournament browsing surface를 제공합니다. |
| 보장하지 않는 것 | 실제 bracket progression, create/join persistence, truthful failure는 보장하지 않습니다. |
| 후속 연결 | `a4f665fd2999`가 create/join action을 연결하고 `be31566ac0fd`가 sample fallback을 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `cb295396771f` — `feat(web): 순위표 화면 추가`
- Thread 내 다음 관련 SHA: `0afc0a0694bd` — `feat(web): 공개 프로필 화면 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `feat(web): 공개 프로필 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `0afc0a0694bd` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | handle-scoped profile route를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/profile/[handle]/page.tsx`의 params 처리와 sample user 초기값을 확인합니다.
- handle 길이 등에서 파생한 가짜 statistic/prose가 있는지 확인합니다.
- 친구 요청·공유 button이 실제 handler와 연결됐는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 사용자 handle로 접근하는 public profile route가 없었습니다. |
| 해결하려던 문제 | 개별 사용자의 표시 이름, rating, 전적, action surface를 제공할 필요가 있었습니다. |
| 핵심 결정 | dynamic route와 sample user 기반 profile card를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | route params → local handle → sample/profile presentation 순서입니다. |
| ownership/lifetime/cleanup | route page가 handle과 displayed user를 소유합니다. |
| failure/rollback/retry | server lookup이 없거나 실패해도 sample user와 가짜 숫자·문구가 표시되고 action은 inert합니다. |
| 보장하는 것 | profile URL과 기본 presentation을 제공합니다. |
| 보장하지 않는 것 | target identity, 실제 user existence, friend action 성공은 보장하지 않습니다. |
| 후속 연결 | `051eac1b4aee`가 API lookup/friend action을 연결하고 `be31566ac0fd`가 fallback을 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `4370ac3162b2` — `feat(web): 토너먼트 대진표 화면 추가`
- Thread 내 다음 관련 SHA: `5e11e944244d` — `feat(web): 관리자 화면 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `feat(web): 관리자 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `5e11e944244d` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | user/status를 표시하는 protected admin screen을 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/admin/page.tsx`의 sample users와 `getAdminUsers` 호출을 확인합니다.
- request error를 catch한 뒤 sample rows가 유지되는지 확인합니다.
- status/review control이 실제 mutation과 연결됐는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 관리자가 사용자 목록과 상태를 보는 browser route가 없었습니다. |
| 해결하려던 문제 | protected admin read model을 표시하는 초기 surface가 필요했습니다. |
| 핵심 결정 | sample user list를 초기값으로 두고 admin API가 성공하면 교체하는 route를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | mount → protected list request → local array replace → row/control render 순서입니다. |
| ownership/lifetime/cleanup | page가 admin data와 control presentation을 소유합니다. |
| failure/rollback/retry | 권한·network failure를 무시하고 sample users를 유지하며 review button 일부는 inert합니다. |
| 보장하는 것 | admin route와 user status presentation을 제공합니다. |
| 보장하지 않는 것 | 실제 authorization 성공이나 action 적용을 보장하지 않습니다. |
| 후속 연결 | `bfea82733512`가 status mutation을 연결하고 `be31566ac0fd`가 fake rows를 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `0afc0a0694bd` — `feat(web): 공개 프로필 화면 추가`
- Thread 내 다음 관련 SHA: `051eac1b4aee` — `feat(profile): 친구 요청 동작 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.7. `feat(profile): 친구 요청 동작 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `051eac1b4aee` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | profile target을 friendship mutation에 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `profile/[handle]/page.tsx`의 `getProfile(handle)`와 `requestFriend(handle)` 호출을 확인합니다.
- request pending/notice와 button disabled 조건을 확인합니다.
- profile load failure가 sample user를 그대로 두는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | profile route는 sample presentation과 inert friend control만 제공했습니다. |
| 해결하려던 문제 | route handle로 실제 profile을 조회하고 friend request를 보낼 필요가 있었습니다. |
| 핵심 결정 | mount/handle effect에서 profile을 조회하고 click handler가 `requestFriend`를 호출하도록 연결했습니다. |
| 입력 → 상태 전이 → 출력 | route handle → profile GET → displayed user; click → friend POST → notice 갱신 순서입니다. |
| ownership/lifetime/cleanup | page가 target handle, request pending, notice를 소유합니다. |
| failure/rollback/retry | profile GET 실패를 무시해 sample user가 남고 mutation failure는 notice만 바꿉니다. |
| 보장하는 것 | friend button이 실제 authenticated API call을 수행합니다. |
| 보장하지 않는 것 | displayed profile이 반드시 server response라는 보장은 아직 없습니다. |
| 후속 연결 | `be31566ac0fd`가 null initial state와 load error branch를 도입합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `5e11e944244d` — `feat(web): 관리자 화면 추가`
- Thread 내 다음 관련 SHA: `bfea82733512` — `feat(admin): 사용자 상태 변경 동작 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.8. `feat(admin): 사용자 상태 변경 동작 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `bfea82733512` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | admin status control을 authenticated mutation에 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `admin/page.tsx`의 active↔banned target 계산과 `setUserStatus` 호출을 확인합니다.
- 성공 response로 row를 교체하는지, 실패 시 어떤 message를 남기는지 확인합니다.
- admin list load failure 뒤 sample row action이 계속 가능한지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | admin screen은 user status를 표시했지만 control이 server mutation을 수행하지 않았습니다. |
| 해결하려던 문제 | 관리자 intent를 target user/status update와 연결하고 결과를 화면에 반영해야 했습니다. |
| 핵심 결정 | 현재 status의 반대 값을 계산해 API를 호출하고 returned user로 local row를 교체했습니다. |
| 입력 → 상태 전이 → 출력 | button click → target status 계산 → authenticated mutation → matching row replace/message 순서입니다. |
| ownership/lifetime/cleanup | page가 pending target과 local row list를 소유합니다. |
| failure/rollback/retry | mutation failure는 명시하지만 초기 list failure 시 sample users가 남아 실제 권한 상태를 가장합니다. |
| 보장하는 것 | status control이 실제 endpoint와 response에 연결됩니다. |
| 보장하지 않는 것 | 초기 user list의 진실성과 audit persistence는 보장하지 않습니다. |
| 후속 연결 | `be31566ac0fd`가 admin data를 빈 상태에서 시작하고 load failure를 차단합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `051eac1b4aee` — `feat(profile): 친구 요청 동작 연결`
- Thread 내 다음 관련 SHA: `a4f665fd2999` — `feat(tournament): 생성과 참가 동작 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.9. `feat(tournament): 생성과 참가 동작 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `a4f665fd2999` |
| Importance | B |
| Tags | AUTH, TOURNAMENT |
| Source에서 확정된 역할 | selected tournament state와 create/join action을 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tournaments/page.tsx`의 selected ID, create form, `createTournament`, `joinTournament` 호출을 확인합니다.
- 생성 성공 후 새 tournament를 selection에 반영하는 순서를 확인합니다.
- join target이 현재 selection과 current user/session을 어떻게 요구하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 토너먼트 화면은 sample bracket과 local presentation만 있었고 create/join이 server에 반영되지 않았습니다. |
| 해결하려던 문제 | 명시적 selection을 기준으로 생성·참가 intent를 실제 API와 연결해야 했습니다. |
| 핵심 결정 | selected tournament ID를 state로 두고 create 성공 항목을 list/selection에 반영하며 join mutation을 현재 selection에 보냈습니다. |
| 입력 → 상태 전이 → 출력 | form/button intent → API mutation → returned tournament 또는 list reload → selection/presentation 갱신 순서입니다. |
| ownership/lifetime/cleanup | page가 selection, form, pending/message를 소유합니다. |
| failure/rollback/retry | 초기 list load failure는 sample tournaments를 유지하고 bracket 자체는 실제 match contract가 아닙니다. |
| 보장하는 것 | create/join action이 실제 authenticated endpoint를 호출합니다. |
| 보장하지 않는 것 | server state의 canonical cache ownership과 truthful initial load는 보장하지 않습니다. |
| 후속 연결 | `be31566ac0fd`가 sample list를 제거하고 query Thread가 mutation invalidation을 정리합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `bfea82733512` — `feat(admin): 사용자 상태 변경 동작 연결`
- Thread 내 다음 관련 SHA: `8b2679d9e190` — `test(e2e): 화면 action의 실제 API 연결 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.10. `test(e2e): 화면 action의 실제 API 연결 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `8b2679d9e190` |
| Importance | B |
| Tags | REALTIME, TOURNAMENT, WEB |
| Source에서 확정된 역할 | AI 시작, match chat, friendship, tournament, admin action을 browser에서 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tests/e2e`의 action별 scenario와 locator/assertion을 확인합니다.
- admin test가 실제 row와 sample fallback 중 하나를 허용하는 assertion인지 확인합니다.
- server mutation 이후 어떤 visible text/state만 확인하고 persistence는 직접 확인하지 않는지 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 browser suite는 navigation과 canvas만 검증해 새 action wiring의 회귀를 잡지 못했습니다. |
| 해결하려던 문제 | 화면 control이 실제 endpoint/WS command를 호출하고 visible feedback을 만드는지 넓게 확인할 필요가 있었습니다. |
| 핵심 결정 | Playwright scenario를 추가해 AI match/chat, friend request, tournament create/join, admin status를 조작했습니다. |
| 입력 → 상태 전이 → 출력 | browser action → network/realtime path → visible UI assertion 순서이며 일부 scenario는 기존 sample data도 허용합니다. |
| ownership/lifetime/cleanup | runner가 browser context를 소유하며 application state는 실제 실행 중 server/fixture에 의존합니다. |
| failure/rollback/retry | sample fallback을 허용하는 assertion 때문에 load failure가 있어도 일부 test가 통과할 수 있습니다. |
| 보장하는 것 | 주요 interactive control이 browser에서 작동하는 broad integration evidence를 제공합니다. |
| 보장하지 않는 것 | 모든 action의 durable commit, 권한 audit, failure state 진실성을 증명하지 않습니다. |
| 후속 연결 | `be31566ac0fd`가 이 suite의 false-positive 가능성을 만든 sample fallback을 제거합니다. |

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | 화면 control이 inert markup이 아니라 실제 HTTP/WS action을 호출합니다. |
| 재현한 실패/경계 | AI start, chat, friend, tournament, admin status의 browser integration입니다. |
| 테스트 기법 | Playwright user action과 visible feedback assertion입니다. |
| 증명하는 것 | 여러 UI→adapter/transport 경로의 broad integration을 증명합니다. |
| 증명하지 않는 것 | sample fallback이 없는 truthful load나 DB durable side effect 전부를 증명하지 않습니다. |
| 검증 분류 | 브라우저 action 통합 테스트 |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `a4f665fd2999` — `feat(tournament): 생성과 참가 동작 연결`
- Thread 내 다음 관련 SHA: `e0ef3fec89a6` — `feat(chat): 로비 채팅 입력 화면 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.11. `feat(chat): 로비 채팅 입력 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e0ef3fec89a6` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | 로비 채팅 writer와 bounded local history를 HomePage에 연결하고 lobby read fallback 일부를 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 `chatInput`, `notice`, `submitLobbyChat`과 최근 20개 유지 식을 확인합니다.
- `apps/web/src/lib/api.ts`에서 `getLobby`의 sample fallback이 제거되고 `sendLobbyChat`가 `/chat/lobby` POST를 호출하는지 확인합니다.
- page의 `players`/`chat` 초기값은 여전히 sample이며 load 실패 notice도 “샘플 화면”을 유지한다고 명시하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 로비는 chat을 읽어 표시했지만 controlled input이나 실제 writer가 없었고 `getLobby` 자체가 실패를 sample response로 대체했습니다. |
| 해결하려던 문제 | 사용자 입력을 trim해 server에 보내고 성공 message를 local history에 반영하되 API helper가 read failure를 성공 response로 만들지 않아야 했습니다. |
| 핵심 결정 | `sendLobbyChat` adapter와 controlled form을 추가하고 성공 시 기존 배열의 최근 19개 뒤에 응답 message를 붙였습니다. `getLobby` helper의 fallback은 제거했습니다. |
| 입력 → 상태 전이 → 출력 | form submit → trim/empty guard → POST `/chat/lobby` → returned message append/입력 clear; 실패 → notice 유지 순서입니다. |
| ownership/lifetime/cleanup | HomePage가 input, notice, local chat array를 소유합니다. HTTP request cancellation이나 shared cache ownership은 없습니다. |
| failure/rollback/retry | 빈 입력은 전송하지 않고 POST 실패는 notice로 표시합니다. 그러나 initial page state는 sample chat/users이므로 read failure 뒤 fabricated content가 남습니다. |
| 보장하는 것 | 로비 chat control이 실제 HTTP writer와 연결되고 history가 20개로 제한됩니다. |
| 보장하지 않는 것 | 실시간 fan-out, duplicate suppression, truthful initial read state는 아직 보장하지 않습니다. |
| 후속 연결 | `4f9b3b312d0e`가 metric presentation을 고치고 `cd3787eefd6a`가 WebSocket fan-out을 추가하며 `be31566ac0fd`가 남은 sample state를 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `8b2679d9e190` — `test(e2e): 화면 action의 실제 API 연결 검증`
- Thread 내 다음 관련 SHA: `4f9b3b312d0e` — `fix(lobby): 로비 상태 표현 개선`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.12. `fix(lobby): 로비 상태 표현 개선`

| 항목 | 값 |
| --- | --- |
| SHA | `4f9b3b312d0e` |
| Importance | B |
| Tags | REALTIME |
| Source에서 확정된 역할 | 고정 wait/주간 증가 문구를 제거하고 server `LobbyStats`를 화면 지표의 source로 사용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 `LobbyStats | null` state와 lobby response에서 `stats`를 설정하는 위치를 확인합니다.
- “이번 주 +2”, 고정 “30초”, `players.length` 기반 online count가 어떤 server-derived 표현으로 바뀌는지 확인합니다.
- `averageWaitSeconds == null`을 “대기 없음”으로 표시하고 queued/activeRooms/playingPlayers를 함께 노출하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 로비는 실제 server 상태와 무관한 “이번 주 +2”, 고정 30초 wait, list 길이 기반 online 수를 사실처럼 표시했습니다. |
| 해결하려던 문제 | 운영 상태를 browser가 임의로 만들지 말고 `/lobby` read model의 live metric만 표시해야 했습니다. |
| 핵심 결정 | `LobbyStats`를 별도 state로 받아 online/playing/queued/activeRooms/averageWaitSeconds를 렌더링하고 알 수 없는 상태는 “확인 중” 또는 “대기 없음”으로 구분했습니다. |
| 입력 → 상태 전이 → 출력 | lobby response → `stats` state → 각 StatCard 값/힌트; null metric → explicit unknown/no-wait presentation 순서입니다. |
| ownership/lifetime/cleanup | server가 metric 계산을, page가 현재 response projection만 소유합니다. |
| failure/rollback/retry | stats가 아직 없으면 fabricated number를 쓰지 않습니다. stale response나 realtime refresh는 다음 commit 전까지 해결하지 않습니다. |
| 보장하는 것 | 로비 상태 카드가 고정 문구 대신 server read model을 표시합니다. |
| 보장하지 않는 것 | 지표의 서버 계산 정확성이나 live update cadence는 보장하지 않습니다. |
| 후속 연결 | `cd3787eefd6a`가 presence event에서 lobby를 reload해 실시간 갱신합니다. |

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | player 배열 길이와 고정 문구로 live lobby 상태를 충분히 표현할 수 있다고 봤습니다. |
| 실제 실패/위험 | 실제 queue/room/wait 상태와 무관한 숫자를 사용자에게 사실처럼 보여 줍니다. |
| 근본 원인 | browser presentation 상수와 server metric source가 분리되지 않았습니다. |
| 수정된 불변식 | 온라인·경기 중·queue·room·wait 표현은 `LobbyStats` 또는 명시적 unknown/no-wait 상태에서만 나옵니다. |
| 회귀 근거 | 이 SHA에는 별도 test가 없고 이후 E2E/query migration에서 화면을 재사용합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `e0ef3fec89a6` — `feat(chat): 로비 채팅 입력 화면 추가`
- Thread 내 다음 관련 SHA: `cd3787eefd6a` — `feat(chat): 로비 채팅과 접속 상태 실시간 반영`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.13. `feat(chat): 로비 채팅과 접속 상태 실시간 반영`

| 항목 | 값 |
| --- | --- |
| SHA | `cd3787eefd6a` |
| Importance | B |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | HomePage가 lobby WebSocket을 직접 소유해 chat fan-out과 presence-triggered reload를 local state에 반영합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 `socketRef`, `userId`, `loadLobby`, WebSocket effect dependency를 확인합니다.
- 당시 `getToken()`을 query string으로 넣고 raw `JSON.parse(... as ServerEvent)`를 사용하는 인증·validation 한계를 확인합니다.
- `chat.message` duplicate ID 제거/최근 20개 유지, `presence.changed`의 lobby reload, cleanup에서 handler null/close/current ref clear 순서를 확인합니다.
- chat submit이 open socket이면 realtime command를, 아니면 HTTP fallback을 사용하는 branch를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | HTTP-only writer는 다른 browser의 chat과 presence를 반영하지 못했고 live metric도 재요청 계기가 없었습니다. |
| 해결하려던 문제 | 인증된 사용자 수명에 맞춰 lobby socket을 열고 fan-out event를 local state에 반영하면서 unmount/user change에서 정확히 정리해야 했습니다. |
| 핵심 결정 | `loadLobby` callback과 `socketRef`를 도입해 lobby chat을 dedupe/append하고 presence event마다 read model을 reload했습니다. writer는 socket open 여부에 따라 WS/HTTP를 선택했습니다. |
| 입력 → 상태 전이 → 출력 | session user resolve → token 기반 socket open → raw event parse → chat local update 또는 presence reload; cleanup → handlers 제거 → socket close → current ref clear 순서입니다. |
| ownership/lifetime/cleanup | HomePage effect가 lobby socket과 local server-state 복사본을 함께 소유합니다. user ID 변화/unmount가 socket lifetime을 끝냅니다. |
| failure/rollback/retry | socket이 열리지 않으면 HTTP writer로 fallback하고 reload 실패는 notice로 표시합니다. malformed event parsing은 catch 없이 effect callback을 실패시킬 수 있습니다. |
| 보장하는 것 | 여러 browser의 lobby chat과 presence 변화가 현재 화면에 반영되고 obsolete socket은 cleanup됩니다. |
| 보장하지 않는 것 | cookie-only credential, one-time ticket, runtime parser, shared Query cache owner는 아직 보장하지 않습니다. |
| 후속 연결 | `353ca9a17415`가 token/raw parse 경계를 고치고 `931800f796e1`이 같은 live update를 canonical query cache로 이전합니다. |

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `cd3787eefd6a` |
| 파일 | `apps/web/src/app/page.tsx` |
| 함수/위치 | `lobby WebSocket effect` |
| 근거 요약 | 현재 user가 있을 때 socket을 열고 lobby chat ID를 중복 제거해 최근 20개만 유지합니다. cleanup은 handler를 끊고 open/connecting socket을 닫은 뒤 current ref를 비웁니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `4f9b3b312d0e` — `fix(lobby): 로비 상태 표현 개선`
- Thread 내 다음 관련 SHA: `be31566ac0fd` — `fix(web): 로그인 화면의 sample fallback 제거`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.14. `fix(web): 로그인 화면의 sample fallback 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `be31566ac0fd` |
| Importance | A |
| Tags | AUTH, TOURNAMENT, WEB |
| Source에서 확정된 역할 | authenticated/server-backed screen이 failure를 sample success data로 대체하지 않게 합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `home`, `dashboard`, `leaderboard`, `profile`, `tournaments`, `admin` page의 initial state가 빈 배열/`null`로 바뀌는지 확인합니다.
- 각 page의 loading, error, empty, success rendering branch와 action disabled 조건을 확인합니다.
- `apps/web/src/lib/sample.ts` import 제거 범위와 아직 남은 canvas-only sample 용도를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 여러 resource screen이 sample object를 초기값으로 사용하고 load error를 무시해 존재하지 않는 사용자·전적·순위·토너먼트·관리 데이터를 정상처럼 표시했습니다. |
| 해결하려던 문제 | 실패와 empty를 success로 위장하면 사용자 action이 잘못된 target에 적용되고 E2E도 false positive가 될 수 있었습니다. |
| 핵심 결정 | server-backed state를 빈 배열/`null`에서 시작하고 loading/error/empty/success를 명시적으로 렌더링하며 data가 없을 때 action을 차단했습니다. |
| 입력 → 상태 전이 → 출력 | mount/request → loading branch → success data 또는 error branch → empty/success presentation 순서로 바뀝니다. |
| ownership/lifetime/cleanup | 각 page는 아직 request state를 직접 소유하지만 displayed resource는 server response만 채울 수 있습니다. sample fixture ownership은 server-backed screen에서 제거됩니다. |
| failure/rollback/retry | request 실패는 visible error로 남고 이전 sample 성공 상태로 rollback하지 않습니다. retry mechanism은 화면별로 제한적입니다. |
| 보장하는 것 | server-backed screen이 fabricated success data를 표시하지 않고 loading/error/empty를 구분합니다. |
| 보장하지 않는 것 | component effect 중복, shared cache, automatic cancellation은 아직 보장하지 않습니다. |
| 후속 연결 | React Query Thread가 같은 truthful state를 단일 cache owner와 AbortSignal로 이전합니다. |

#### A-level 불변식 종합

- **핵심 책임:** authenticated/server-backed screen이 failure를 sample success data로 대체하지 않게 합니다.
- **실패 영향:** 실패와 empty를 success로 위장하면 사용자 action이 잘못된 target에 적용되고 E2E도 false positive가 될 수 있었습니다.
- **결과 불변식:** server-backed screen이 fabricated success data를 표시하지 않고 loading/error/empty를 구분합니다.
- **남은 제한:** component effect 중복, shared cache, automatic cancellation은 아직 보장하지 않습니다.

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | sample 초기값을 두면 API가 없어도 화면을 계속 사용할 수 있다고 봤습니다. |
| 실제 실패/위험 | network/auth failure가 실제 사용자·전적·권한·토너먼트가 존재하는 것처럼 표시되고 action/test도 잘못된 성공을 만듭니다. |
| 근본 원인 | demo fixture와 server-backed resource state를 같은 변수에 넣고 catch에서 상태를 비우지 않았습니다. |
| 수정된 불변식 | server-backed data는 server success만 채우며 pending/error/empty는 별도 상태로 표시합니다. |
| 회귀 근거 | 기존 E2E의 허점은 code branch로 닫혔고 후속 query tests가 cache-level consistency를 검증합니다. |

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `be31566ac0fd` |
| 파일 | `apps/web/src/app/*/page.tsx` |
| 함수/위치 | `resource initialization/render branches` |
| 근거 요약 | 각 화면은 sample object 대신 `null` 또는 빈 배열에서 시작하고 `isLoading`, error, empty, success 조건을 별도로 렌더링합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `cd3787eefd6a` — `feat(chat): 로비 채팅과 접속 상태 실시간 반영`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| resource surface 도입 | `ea1f1b7ba543` → `5e11e944244d` | 각 route가 lobby/dashboard/leaderboard/tournament/profile/admin read model을 표시하지만 sample 초기값을 사용했습니다. |
| action wiring | `051eac1b4aee` → `a4f665fd2999` | profile/admin/tournament intent가 실제 endpoint로 연결됐습니다. |
| lobby writer와 live metric | `e0ef3fec89a6` → `4f9b3b312d0e` | chat writer를 연결하고 고정 wait/온라인 표현을 server LobbyStats로 교체했습니다. |
| component-owned realtime state | `cd3787eefd6a` | HomePage가 socket, deduplicated chat, presence reload를 직접 소유하는 선행 상태가 생겼습니다. |
| broad browser evidence | `8b2679d9e190` | 주요 control은 E2E에서 조작됐지만 sample fallback을 허용했습니다. |
| truthful state 교정 | `be31566ac0fd` | sample success를 제거하고 loading/error/empty/success를 분리했습니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| 고정 wait/주간 증가/players.length 지표 | server와 다른 lobby 상태 표시 | `4f9b3b312d0e` | 후속 query/cache와 browser scenario | 고정 wait/주간 증가/players.length 지표 → server와 다른 lobby 상태 표시 → `4f9b3b312d0e` → 후속 query/cache와 browser scenario |
| sample initial state + ignored load error | failure가 valid resource로 보임 | `be31566ac0fd` | 후속 query/cache test와 explicit rendering branch | sample initial state + ignored load error → failure가 valid resource로 보임 → `be31566ac0fd` → 후속 query/cache test와 explicit rendering branch |
| inert profile/admin/tournament controls | user intent가 server에 도달하지 않음 | `051eac1b4aee`, `bfea82733512`, `a4f665fd2999` | `8b2679d9e190` | inert profile/admin/tournament controls → user intent가 server에 도달하지 않음 → `051eac1b4aee`, `bfea82733512`, `a4f665fd2999` → `8b2679d9e190` |
| E2E가 sample row도 허용 | 실패해도 action test 일부가 통과 | `be31566ac0fd`가 fallback 제거 | 완전한 durable 검증은 아님 | E2E가 sample row도 허용 → 실패해도 action test 일부가 통과 → `be31566ac0fd`가 fallback 제거 → 완전한 durable 검증은 아님 |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| lobby resource | sample fixture + page effect | page의 빈 state + server response | route mount 수명 |
| lobby realtime socket | 없음/HTTP-only | `HomePage.socketRef`와 local state | user ID 또는 route mount 수명 |
| dashboard/leaderboard | sample object/array | null/empty → success response | route mount 수명 |
| profile target | sample user | route handle + server profile 또는 explicit error | handle별 route 수명 |
| tournament selection | sample first item | server list + explicit selected ID | page 수명 |
| admin rows | sample users | server list only | protected route 수명 |

## 9. Thread 최종 상태

마지막 SHA에서 lobby/dashboard/leaderboard/profile/tournament/admin 화면은 server response가 오기 전 fabricated resource를 표시하지 않습니다. 로비 chat writer와 presence-triggered live reload는 먼저 HomePage local state/socket으로 구현됐고 profile/admin/tournament action도 실제 adapter에 연결됐습니다. 고정 wait·주간 증가 문구는 server LobbyStats projection으로 교체됐습니다. 다만 request/socket/cache lifecycle은 route component가 직접 소유하므로 다음 Thread에서 cookie/runtime adapter와 React Query owner로 이전됩니다.

### 최종 실행 흐름

1. 로비는 초기 HTTP read로 users/chat/stats를 받고 현재 user 수명에 맞춰 lobby socket을 엽니다.
2. chat event는 ID 중복을 제거해 최근 20개 local history를 만들고 presence event는 lobby read를 다시 수행합니다.
3. route가 mount되면 resource state는 `null` 또는 빈 배열에서 시작합니다.
4. request pending 동안 loading branch를 렌더링하고 action target이 없으면 control을 비활성화합니다.
5. success payload만 resource state를 채우며 genuine empty는 별도 empty presentation으로 나타납니다.
6. non-OK/network failure는 error branch를 렌더링하고 sample data로 대체하지 않습니다.
7. 사용자 action은 현재 route handle/selected ID/target user를 endpoint helper에 전달하고 success response로 화면을 갱신합니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 03-resource-screens-actions-and-truthful-server-state.md =====

===== BEGIN FILE: 04-game-connection-reducer-and-transport-client.md =====
# Game connection reducer와 transport client

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

play component에 흩어진 socket callback과 mutable state를 pure reducer 및 transport-neutral `GameSocketClient`로 분리하고, ticket 요청부터 socket 교체·재연결·message parsing·room-scoped chat filtering까지의 수명주기를 한 연결 계층이 소유하도록 하는 과정을 복원합니다.

### 직접 연결되는 불변식

- browser connection state transition은 pure reducer가, ticket/socket/timer resource는 `GameSocketClient`가 소유합니다.
- 교체된 ticket response와 stale socket event는 generation/current-socket guard를 통과하지 못합니다.
- snapshot sequence는 단조 증가하며 같은 값이나 더 오래된 snapshot은 current state를 덮지 않습니다.
- reconnect는 existing room continuation이며 최초 queue/tournament command를 다시 보내지 않습니다.
- match chat은 현재 active room과 `scope: "match"`가 모두 일치할 때만 reducer에 전달됩니다.

### 교차 Thread 주의

`4f5199097284`는 cross-cutting commit입니다. 이 문서는 transport/reducer/hook/play-page 변경만 고정합니다. 같은 SHA의 `HomePage`와 `demoPolicy` 변경은 `08-guest-browser-policy-and-transient-results.md`에서 별도로 조사합니다.

## 2. 핵심 질문

- reducer action vocabulary와 transport resource owner가 어떤 파일에서 분리됩니까?
- ticket request가 늦게 resolve되거나 socket이 교체될 때 어떤 generation/identity guard가 stale completion을 버립니까?
- remote close와 explicit close가 reconnect timer, deadline, initial command 면에서 어떻게 다릅니까?
- 현재 room이 없는 terminal state에서만 새 match를 시작하도록 어떤 predicate를 사용합니까?
- 다른 room/lobby chat이 active game message list에 들어오지 않는 filter는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1bf328ce92a5` | `refactor(web): game input 직렬화 경계 분리` | B | WEB | keyboard state를 protocol command 방향으로 바꾸는 pure helper를 분리합니다. |
| 2 | `ffcbdd403a06` | `refactor(web): game connection 상태 reducer 분리` | B | REALTIME, WEB | browser game connection state/action vocabulary를 정의합니다. |
| 3 | `d8311e74373e` | `refactor(web): game connection 전이 규칙 완성` | A | REALTIME, WEB, OPERATIONS | open/matched/snapshot/reconnecting/finished/failed transition을 reducer에 완성합니다. |
| 4 | `bfded21cd1ac` | `refactor(web): GameSocketClient 연결 수명주기 분리` | A | AUTH, REALTIME, WEB | ticket request, socket replacement, close/teardown을 transport-neutral client가 소유합니다. |
| 5 | `92ad229a23d3` | `refactor(web): GameSocketClient 메시지 처리를 분리` | A | AUTH, PROTOCOL, SIMULATION | message parse/dispatch까지 client로 이동해 ticket-to-socket lifecycle을 통합합니다. |
| 6 | `b5691b01a09b` | `test(web): game connection lifecycle 검증` | A | AUTH, PROTOCOL, REALTIME | connection replacement, teardown, stale event, command lifecycle을 deterministic test로 고정합니다. |
| 7 | `4f5199097284` | `fix(web): 중단된 game reconnect 복구` | A | AUTH, REALTIME, WEB | fresh ticket으로 existing room을 복구하고 original matchmaking intent를 재전송하지 않습니다. |
| 8 | `85edd6d1e26a` | `fix(web): 현재 경기방의 채팅만 표시` | B | REALTIME, WEB | inbound match chat을 current active room predicate로 filtering합니다. |
| 9 | `02775797ab63` | `test(web): 매치 채팅 room filtering 검증` | B | REALTIME, WEB, TEST | match chat room filter의 positive/negative boundary를 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `refactor(web): game input 직렬화 경계 분리`

| 항목 | 값 |
| --- | --- |
| SHA | `1bf328ce92a5` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | keyboard state를 protocol command 방향으로 바꾸는 pure helper를 분리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/gameInput.ts`의 `directionForKey`와 `isEditableTarget`을 확인합니다.
- Arrow/W/S 외 key와 input/textarea/select/contenteditable target이 어떤 값을 반환하는지 확인합니다.
- 아직 play page가 helper를 호출하지 않는지 parent/diff에서 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | play page의 keyboard 해석이 DOM event handler 내부에 직접 섞여 있었습니다. |
| 해결하려던 문제 | 입력 의미를 socket lifecycle과 분리해 단위 검증하고 migration에서 재사용할 필요가 있었습니다. |
| 핵심 결정 | key→`-1\|0\|1\|null` mapping과 editable target 판정을 side effect 없는 helper로 추출했습니다. |
| 입력 → 상태 전이 → 출력 | DOM key/target → pure helper → 이후 caller가 command 전송 여부를 결정합니다. |
| ownership/lifetime/cleanup | helper는 입력 해석만 소유하고 key listener와 socket 전송 수명은 아직 page가 소유합니다. |
| failure/rollback/retry | 지원하지 않는 key는 command를 만들지 않고 editable target은 game control 대상에서 제외할 수 있습니다. |
| 보장하는 것 | 입력 의미를 protocol transport와 독립적으로 재사용·test할 수 있습니다. |
| 보장하지 않는 것 | 실제 listener cleanup, neutral input 전송, mobile pointer는 아직 보장하지 않습니다. |
| 후속 연결 | `1ae6fa7836d8`에서 keyboard/touch caller가 이 helper를 사용하고 `b5691b01a09b`가 mapping을 test합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show 1bf328ce92a5^` 및 `git diff 1bf328ce92a5^ 1bf328ce92a5`로 비교합니다.
- Thread 내 다음 관련 SHA: `ffcbdd403a06` — `refactor(web): game connection 상태 reducer 분리`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `refactor(web): game connection 상태 reducer 분리`

| 항목 | 값 |
| --- | --- |
| SHA | `ffcbdd403a06` |
| Importance | B |
| Tags | REALTIME, WEB |
| Source에서 확정된 역할 | browser game connection state/action vocabulary를 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/gameConnection.ts`의 `GameConnectionState`, status union, action union, initial state를 확인합니다.
- 이 SHA의 reducer가 아직 state를 그대로 반환하는 skeleton인지 확인합니다.
- page local fields와 새 state shape의 중복을 비교합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | room, snapshot, notice, opponent, messages, status가 play page의 여러 `useState`에 분산돼 있었습니다. |
| 해결하려던 문제 | connection lifecycle을 하나의 명시적 state/action vocabulary로 표현할 기반이 필요했습니다. |
| 핵심 결정 | state shape와 action union, initial state, reducer boundary를 만들었지만 transition body는 아직 완성하지 않았습니다. |
| 입력 → 상태 전이 → 출력 | 향후 transport callback이 action을 만들고 reducer가 새 state를 반환하도록 호출 경계를 정의했습니다. |
| ownership/lifetime/cleanup | reducer가 connection state의 예정 owner가 되지만 이 SHA에서는 실제 page가 여전히 owner입니다. |
| failure/rollback/retry | action type은 제한하지만 no-op reducer 때문에 lifecycle behavior는 아직 바뀌지 않습니다. |
| 보장하는 것 | 후속 transition 구현에 필요한 상태·사건 어휘를 고정합니다. |
| 보장하지 않는 것 | open/matched/snapshot/close/failure의 실제 전이는 보장하지 않습니다. |
| 후속 연결 | `d8311e74373e`가 reducer transition을 완성합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `1bf328ce92a5` — `refactor(web): game input 직렬화 경계 분리`
- Thread 내 다음 관련 SHA: `d8311e74373e` — `refactor(web): game connection 전이 규칙 완성`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `refactor(web): game connection 전이 규칙 완성`

| 항목 | 값 |
| --- | --- |
| SHA | `d8311e74373e` |
| Importance | A |
| Tags | REALTIME, WEB, OPERATIONS |
| Source에서 확정된 역할 | open/matched/snapshot/reconnecting/finished/failed transition을 reducer에 완성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `gameConnectionReducer`의 각 action branch와 state reset/preserve field를 표로 추적합니다.
- `snapshotReceived`가 `sequence <= lastSnapshotSequence`를 거부하는 조건을 확인합니다.
- `socketClosed`가 `roomId` 유무에 따라 `reconnecting`/`failed`를 선택하는지 확인합니다.
- `gameFinished`가 room, direction-sensitive state, notice를 어떻게 정리하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | action vocabulary는 있었지만 reducer가 no-op라 transport event를 일관된 state transition으로 바꾸지 못했습니다. |
| 해결하려던 문제 | 비동기 callback 순서가 달라도 stale snapshot, reconnect, finish가 명시적 불변식에 따라 처리돼야 했습니다. |
| 핵심 결정 | 각 action의 immutable transition을 구현하고 last snapshot sequence gate, bounded message list, room-aware close, terminal reset을 추가했습니다. |
| 입력 → 상태 전이 → 출력 | transport event → action → reducer branch → 새 `GameConnectionState` → React render 순서입니다. |
| ownership/lifetime/cleanup | reducer가 synchronous state transition을 소유합니다. socket/ticket/timer resource는 소유하지 않습니다. |
| failure/rollback/retry | stale/equal snapshot은 state를 그대로 반환하고, room이 있는 close는 reconnecting, 없는 close는 failed로 갑니다. |
| 보장하는 것 | connection 상태가 단일 transition table을 따르고 snapshot이 뒤로 가지 않습니다. |
| 보장하지 않는 것 | 비동기 resource가 stale action을 dispatch하지 않는지는 reducer만으로 보장하지 않습니다. |
| 후속 연결 | `bfded21cd1ac`과 `92ad229a23d3`가 transport lifetime/generation을 분리하고 `b5691b01a09b`가 transition을 검증합니다. |

#### A-level 불변식 종합

- **핵심 책임:** open/matched/snapshot/reconnecting/finished/failed transition을 reducer에 완성합니다.
- **실패 영향:** 비동기 callback 순서가 달라도 stale snapshot, reconnect, finish가 명시적 불변식에 따라 처리돼야 했습니다.
- **결과 불변식:** connection 상태가 단일 transition table을 따르고 snapshot이 뒤로 가지 않습니다.
- **남은 제한:** 비동기 resource가 stale action을 dispatch하지 않는지는 reducer만으로 보장하지 않습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `d8311e74373e` |
| 파일 | `apps/web/src/game/gameConnection.ts` |
| 함수/위치 | `gameConnectionReducer` |
| 근거 요약 | `snapshot.sequence <= state.lastSnapshotSequence`이면 기존 state를 반환하고, close 시 `roomId`가 있으면 `reconnecting`, 없으면 `failed`로 전이합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `ffcbdd403a06` — `refactor(web): game connection 상태 reducer 분리`
- Thread 내 다음 관련 SHA: `bfded21cd1ac` — `refactor(web): GameSocketClient 연결 수명주기 분리`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `refactor(web): GameSocketClient 연결 수명주기 분리`

| 항목 | 값 |
| --- | --- |
| SHA | `bfded21cd1ac` |
| Importance | A |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | ticket request, socket replacement, close/teardown을 transport-neutral client가 소유합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/GameSocketClient.ts`의 `socket`, `ticketRequest`, `generation`, `inputSequence` field를 확인합니다.
- `connect`, `replaceConnection`, `close`, `isCurrent`가 AbortController와 handler 해제를 어떤 순서로 수행하는지 확인합니다.
- ticket resolve 뒤 generation이 바뀐 경우 socket을 만들지 않는 guard를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | play page가 AbortController, WebSocket ref, event handler, input sequence를 직접 소유해 connection 교체와 unmount cleanup이 UI 코드에 섞였습니다. |
| 해결하려던 문제 | ticket request와 socket이 서로 다른 시점에 완료되므로 이전 연결의 completion을 확실히 폐기할 단일 resource owner가 필요했습니다. |
| 핵심 결정 | `GameSocketClient`가 current socket, ticket request, monotonically changing generation, input sequence를 소유하고 replacement 시 모두 정리하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | `connect` → generation 증가/기존 resource 정리 → ticket request → current generation 확인 → socket 생성/handler 설치 순서입니다. |
| ownership/lifetime/cleanup | client instance가 ticket AbortController와 socket handler/lifetime을 소유합니다. React state는 callback 밖 reducer가 소유합니다. |
| failure/rollback/retry | explicit replacement는 pending ticket을 abort하고 old handlers를 null로 만든 뒤 OPEN/CONNECTING socket을 닫습니다. stale completion은 generation check에서 무시됩니다. |
| 보장하는 것 | 동시에 하나의 current ticket/socket만 connection state를 변경할 수 있습니다. |
| 보장하지 않는 것 | message runtime parsing과 full reconnect policy는 아직 완성되지 않았습니다. |
| 후속 연결 | `92ad229a23d3`이 parsing/dispatch를 client에 넣고 `b5691b01a09b`가 aborted ticket 경계를 검증합니다. |

#### A-level 불변식 종합

- **핵심 책임:** ticket request, socket replacement, close/teardown을 transport-neutral client가 소유합니다.
- **실패 영향:** ticket request와 socket이 서로 다른 시점에 완료되므로 이전 연결의 completion을 확실히 폐기할 단일 resource owner가 필요했습니다.
- **결과 불변식:** 동시에 하나의 current ticket/socket만 connection state를 변경할 수 있습니다.
- **남은 제한:** message runtime parsing과 full reconnect policy는 아직 완성되지 않았습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `d8311e74373e` — `refactor(web): game connection 전이 규칙 완성`
- Thread 내 다음 관련 SHA: `92ad229a23d3` — `refactor(web): GameSocketClient 메시지 처리를 분리`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `refactor(web): GameSocketClient 메시지 처리를 분리`

| 항목 | 값 |
| --- | --- |
| SHA | `92ad229a23d3` |
| Importance | A |
| Tags | AUTH, PROTOCOL, SIMULATION |
| Source에서 확정된 역할 | message parse/dispatch까지 client로 이동해 ticket-to-socket lifecycle을 통합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `GameSocketClient`가 ticket response로 URL을 만들고 `parseServerEvent`를 호출하는 위치를 확인합니다.
- parse failure가 `onFailure`로 전달되는지, stale socket의 message가 `isCurrent`에서 버려지는지 확인합니다.
- `send`/`sendDirection`의 readyState, room ID, `inputSeq` 증가 조건을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | resource lifetime은 client로 옮겼지만 message parsing과 protocol dispatch 일부가 caller에 남아 transport abstraction이 완결되지 않았습니다. |
| 해결하려던 문제 | 모든 inbound event가 current socket과 shared runtime parser를 통과하고 outbound input sequence도 client가 소유해야 했습니다. |
| 핵심 결정 | ticket URL, socket factory, shared parser, callback dispatch, ready-state guarded send, monotonic input sequence를 client 안에 넣었습니다. |
| 입력 → 상태 전이 → 출력 | ticket resolve → socket open → optional initial command; message → current guard → parser → `onEvent`; direction → seq 증가 → serialized send 순서입니다. |
| ownership/lifetime/cleanup | client가 raw transport bytes와 input sequence를 소유하고 reducer/hook은 parsed domain event만 받습니다. |
| failure/rollback/retry | malformed event는 failure callback으로 가고 closed/non-open socket의 send는 false를 반환합니다. |
| 보장하는 것 | raw WebSocket payload가 parser 없이 UI state로 들어오지 않고 input command sequence가 connection 내에서 단조 증가합니다. |
| 보장하지 않는 것 | remote close 후 자동 retry와 room continuation은 아직 보장하지 않습니다. |
| 후속 연결 | `b5691b01a09b`가 parser failure와 input sequence를 test하고 `4f5199097284`가 reconnect policy를 추가합니다. |

#### A-level 불변식 종합

- **핵심 책임:** message parse/dispatch까지 client로 이동해 ticket-to-socket lifecycle을 통합합니다.
- **실패 영향:** 모든 inbound event가 current socket과 shared runtime parser를 통과하고 outbound input sequence도 client가 소유해야 했습니다.
- **결과 불변식:** raw WebSocket payload가 parser 없이 UI state로 들어오지 않고 input command sequence가 connection 내에서 단조 증가합니다.
- **남은 제한:** remote close 후 자동 retry와 room continuation은 아직 보장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `bfded21cd1ac` — `refactor(web): GameSocketClient 연결 수명주기 분리`
- Thread 내 다음 관련 SHA: `b5691b01a09b` — `test(web): game connection lifecycle 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `test(web): game connection lifecycle 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `b5691b01a09b` |
| Importance | A |
| Tags | AUTH, PROTOCOL, REALTIME |
| Source에서 확정된 역할 | connection replacement, teardown, stale event, command lifecycle을 deterministic test로 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `GameSocketClient.test.ts`의 deferred ticket, fake socket, parser failure, sent frame assertions를 확인합니다.
- `gameConnection.test.ts`의 lifecycle transition과 stale snapshot case를 확인합니다.
- `gameInput.test.ts`의 key/editable mapping 범위를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | reducer와 client 분리는 완료됐지만 asynchronous race와 stale event 방지가 실제로 동작한다는 자동 증거가 없었습니다. |
| 해결하려던 문제 | pending ticket 취소, replacement, malformed message, input sequence, reducer ordering을 deterministic하게 재현해야 했습니다. |
| 핵심 결정 | deferred Promise와 fake socket으로 첫 ticket을 취소한 뒤 두 번째 연결만 생성하고, raw messages와 sent frames를 직접 관찰했습니다. |
| 입력 → 상태 전이 → 출력 | test가 fake clock/socket/ticket provider를 제어 → production client/reducer 실행 → callback/state/frame assertion 순서입니다. |
| ownership/lifetime/cleanup | test fixture가 transport doubles를 소유하고 production client cleanup을 직접 호출합니다. |
| failure/rollback/retry | 취소된 첫 ticket은 socket을 만들지 않고 malformed event는 failure로 가며 direction마다 seq가 증가합니다. |
| 보장하는 것 | generation fencing, parser boundary, reducer stale-snapshot invariant를 deterministic하게 증명합니다. |
| 보장하지 않는 것 | 실제 browser/network reconnect, server room reservation, latency 조건은 증명하지 않습니다. |
| 후속 연결 | `4f5199097284`의 reconnect 추가 뒤 `06d2eb7a93cc`가 fresh-ticket continuation을 추가 검증합니다. |

#### A-level 불변식 종합

- **핵심 책임:** connection replacement, teardown, stale event, command lifecycle을 deterministic test로 고정합니다.
- **실패 영향:** pending ticket 취소, replacement, malformed message, input sequence, reducer ordering을 deterministic하게 재현해야 했습니다.
- **결과 불변식:** generation fencing, parser boundary, reducer stale-snapshot invariant를 deterministic하게 증명합니다.
- **남은 제한:** 실제 browser/network reconnect, server room reservation, latency 조건은 증명하지 않습니다.

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | 현재 generation/socket만 event를 전달하고 reducer는 stale state를 거부합니다. |
| 재현한 실패/경계 | 취소된 ticket resolve, malformed event, socket replacement, duplicate/stale snapshot, input sequence입니다. |
| 테스트 기법 | deferred Promise, fake WebSocket, reducer table, pure input helper test입니다. |
| 증명하는 것 | race와 protocol boundary를 browser 없이 deterministic하게 증명합니다. |
| 증명하지 않는 것 | 실제 network reconnect와 server-side room recovery는 증명하지 않습니다. |
| 검증 분류 | deterministic lifecycle·boundary regression |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `92ad229a23d3` — `refactor(web): GameSocketClient 메시지 처리를 분리`
- Thread 내 다음 관련 SHA: `4f5199097284` — `fix(web): 중단된 game reconnect 복구`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.7. `fix(web): 중단된 game reconnect 복구`

| 항목 | 값 |
| --- | --- |
| SHA | `4f5199097284` |
| Importance | A |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | fresh ticket으로 existing room을 복구하고 original matchmaking intent를 재전송하지 않습니다. |

#### 해당 SHA에서 확인할 실제 코드

- 이 Thread에서는 `GameSocketClient.ts`, `gameConnection.ts`, `useGameConnection.ts`, `play/page.tsx`만 조사합니다. 같은 SHA의 `HomePage`/`demoPolicy` 변경은 guest Thread에서 다룹니다.
- `RECONNECT_WINDOW_MS`, initial/max delay, deadline, attempts, timer cleanup을 확인합니다.
- `openSocket(..., initialEvent: null, reconnected: true)`가 새 ticket을 사용하되 최초 queue command를 보내지 않는지 확인합니다.
- `canStartNewMatch`가 `roomId === null`과 terminal status를 동시에 요구하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | remote close는 reducer를 reconnecting으로 표시할 수 있었지만 client는 재연결하지 않았고, 단순 `connect` 재사용 시 원래 queue/tournament command를 다시 보내 두 번째 match intent를 만들 위험이 있었습니다. |
| 해결하려던 문제 | existing room recovery는 새 admission ticket과 bounded retry를 사용하되 original matchmaking command를 반복하지 않아야 했습니다. |
| 핵심 결정 | 15초 window, 250ms exponential delay(2초 상한), fresh ticket socket, `initialEvent=null`, room-aware `onClosed`, `canStartNewMatch` guard를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | remote close → reducer `socketClosed` → room이 있으면 retry 승인 → timer → fresh ticket → 새 socket open → no initial command → server snapshot으로 room 복구 순서입니다. |
| ownership/lifetime/cleanup | `GameSocketClient`가 retry timer/deadline/attempt를 소유하고 hook의 stateRef가 현재 room을 판단합니다. explicit `close`는 timer와 request/socket을 모두 해제합니다. |
| failure/rollback/retry | ticket failure도 window 안에서는 retry하고 deadline을 넘으면 failure action을 만듭니다. duplicate start button과 hook connect는 predicate에서 거부됩니다. |
| 보장하는 것 | 끊긴 socket이 existing room을 bounded하게 이어가며 재연결 때문에 두 번째 matchmaking intent를 만들지 않습니다. |
| 보장하지 않는 것 | server가 room reservation을 실제 유지하는지, 15초 이후 복구되는지는 browser client만으로 보장하지 않습니다. |
| 후속 연결 | `06d2eb7a93cc`가 fresh ticket/no original command를 fake timer로 검증하고 `1abda1299ad8`이 browser E2E reconnect를 검증합니다. |

#### A-level 불변식 종합

- **핵심 책임:** fresh ticket으로 existing room을 복구하고 original matchmaking intent를 재전송하지 않습니다.
- **실패 영향:** existing room recovery는 새 admission ticket과 bounded retry를 사용하되 original matchmaking command를 반복하지 않아야 했습니다.
- **결과 불변식:** 끊긴 socket이 existing room을 bounded하게 이어가며 재연결 때문에 두 번째 matchmaking intent를 만들지 않습니다.
- **남은 제한:** server가 room reservation을 실제 유지하는지, 15초 이후 복구되는지는 browser client만으로 보장하지 않습니다.

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | close 상태 표시만으로 reconnect가 충분하거나 최초 `connect`를 재사용해도 된다고 봤습니다. |
| 실제 실패/위험 | 연결이 복구되지 않거나 queue command가 반복되어 existing room과 새 match intent가 충돌합니다. |
| 근본 원인 | initial admission과 room continuation을 같은 command-bearing open path로 취급했습니다. |
| 수정된 불변식 | reconnect는 fresh ticket을 사용하지만 initial matchmaking command는 `null`이며 current room이 있는 동안 새 match를 시작하지 않습니다. |
| 회귀 근거 | `06d2eb7a93cc` unit tests와 `1abda1299ad8` Playwright reconnect scenario입니다. |

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `4f5199097284` |
| 파일 | `apps/web/src/game/GameSocketClient.ts` |
| 함수/위치 | `scheduleReconnect / openSocket` |
| 근거 요약 | 재연결은 `openSocket(generation, null, handlers, true)`를 호출해 새 ticket을 얻되 최초 queue command를 다시 보내지 않습니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `b5691b01a09b` — `test(web): game connection lifecycle 검증`
- Thread 내 다음 관련 SHA: `85edd6d1e26a` — `fix(web): 현재 경기방의 채팅만 표시`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.8. `fix(web): 현재 경기방의 채팅만 표시`

| 항목 | 값 |
| --- | --- |
| SHA | `85edd6d1e26a` |
| Importance | B |
| Tags | REALTIME, WEB |
| Source에서 확정된 역할 | inbound match chat을 current active room predicate로 filtering합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/chatScope.ts`의 `isChatForActiveRoom` 조건을 확인합니다.
- `useGameConnection` event handler가 reducer dispatch 전에 predicate를 호출하는지 확인합니다.
- active room 없음, lobby scope, 다른 room ID가 모두 false인지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | parsed `chat.message`면 scope/room 관계와 무관하게 current game message list로 전달될 수 있었습니다. |
| 해결하려던 문제 | 다른 match 또는 lobby chat이 active game UI에 나타나지 않아야 했습니다. |
| 핵심 결정 | `scope === "match"`, active room 존재, `message.roomId === activeRoomId`를 모두 요구하는 pure predicate를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | server event → current room read → predicate → true일 때만 reducer chat action 순서입니다. |
| ownership/lifetime/cleanup | hook이 active room context를, pure helper가 acceptance rule을 소유합니다. |
| failure/rollback/retry | 조건이 하나라도 맞지 않으면 event를 조용히 무시합니다. |
| 보장하는 것 | 현재 방의 match chat만 game message state에 들어갑니다. |
| 보장하지 않는 것 | server-side chat authorization이나 persistence는 보장하지 않습니다. |
| 후속 연결 | `02775797ab63`이 모든 negative 조합을 unit test로 고정합니다. |

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | parsed chat event는 현재 화면에 표시해도 된다고 가정했습니다. |
| 실제 실패/위험 | 다른 room이나 lobby message가 active match UI에 섞입니다. |
| 근본 원인 | protocol validity와 UI room membership을 같은 것으로 취급했습니다. |
| 수정된 불변식 | runtime-valid event라도 current room/scope predicate를 통과해야 표시합니다. |
| 회귀 근거 | `02775797ab63`의 exact/other/lobby/no-room table입니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `4f5199097284` — `fix(web): 중단된 game reconnect 복구`
- Thread 내 다음 관련 SHA: `02775797ab63` — `test(web): 매치 채팅 room filtering 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.9. `test(web): 매치 채팅 room filtering 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `02775797ab63` |
| Importance | B |
| Tags | REALTIME, WEB, TEST |
| Source에서 확정된 역할 | match chat room filter의 positive/negative boundary를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/chatScope.test.ts`의 exact active-room, other-room, lobby, null-room case를 확인합니다.
- test가 `isChatForActiveRoom` pure production function을 직접 호출하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | room filter fix는 작지만 다른 scope/room 조합에서 다시 완화될 위험이 있었습니다. |
| 해결하려던 문제 | accepted case 하나와 거부해야 할 모든 주요 case를 고정할 필요가 있었습니다. |
| 핵심 결정 | pure predicate를 table-like assertions로 직접 호출했습니다. |
| 입력 → 상태 전이 → 출력 | event fixture + active room → predicate result assertion 순서입니다. |
| ownership/lifetime/cleanup | test에는 resource cleanup이 없고 production hook dispatch는 실행하지 않습니다. |
| failure/rollback/retry | predicate는 active room이 없거나 scope/room ID가 다르면 false를 반환하며 retry나 fallback은 없습니다. |
| 보장하는 것 | exact active room만 true이며 다른 room/lobby/no active room은 false입니다. |
| 보장하지 않는 것 | browser rendering, server authorization, chat order는 증명하지 않습니다. |
| 후속 연결 | current room filtering fix의 deterministic regression evidence입니다. |

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | 현재 active room의 match-scope chat만 game reducer로 전달할 수 있습니다. |
| 재현한 실패/경계 | 다른 room, lobby scope, room 없는 상태입니다. |
| 테스트 기법 | pure predicate direct unit test입니다. |
| 증명하는 것 | acceptance rule의 complete branch 조합을 증명합니다. |
| 증명하지 않는 것 | hook integration과 server-side audience authorization은 증명하지 않습니다. |
| 검증 분류 | scope/identity boundary unit regression |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `85edd6d1e26a` — `fix(web): 현재 경기방의 채팅만 표시`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| 입력 해석 분리 | `1bf328ce92a5` | keyboard 의미를 pure helper로 분리했습니다. |
| state vocabulary 도입·완성 | `ffcbdd403a06` → `d8311e74373e` | reducer가 lifecycle와 snapshot ordering을 소유합니다. |
| transport lifetime 이전 | `bfded21cd1ac` → `92ad229a23d3` | ticket, socket, parser, sequence를 `GameSocketClient`가 소유합니다. |
| deterministic 검증 | `b5691b01a09b` | replacement/stale/parse/input sequence를 test로 고정했습니다. |
| bounded reconnect 교정 | `4f5199097284` | fresh ticket, no duplicate initial intent, deadline/backoff가 추가됐습니다. |
| room-scoped chat 교정 | `85edd6d1e26a` → `02775797ab63` | current room predicate와 unit regression이 추가됐습니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| page-local async refs | stale ticket/socket가 state 변경 가능 | `bfded21cd1ac` generation owner | `b5691b01a09b` | page-local async refs → stale ticket/socket가 state 변경 가능 → `bfded21cd1ac` generation owner → `b5691b01a09b` |
| remote close 후 상태만 reconnecting | 실제 연결 복구 없음/intent 중복 가능 | `4f5199097284` | `06d2eb7a93cc`, `1abda1299ad8` | remote close 후 상태만 reconnecting → 실제 연결 복구 없음/intent 중복 가능 → `4f5199097284` → `06d2eb7a93cc`, `1abda1299ad8` |
| parsed chat이면 표시 | 다른 room/lobby message 혼입 | `85edd6d1e26a` | `02775797ab63` | parsed chat이면 표시 → 다른 room/lobby message 혼입 → `85edd6d1e26a` → `02775797ab63` |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| keyboard meaning | page event handler | `gameInput.ts` pure helper | event 처리 순간 |
| connection state | 여러 page `useState` | `gameConnectionReducer` | hook/reducer 수명 |
| ticket request | page ref | `GameSocketClient.ticketRequest` | connect attempt 수명 |
| socket/handlers | page ref/callback | `GameSocketClient.socket` + generation | current connection 수명 |
| reconnect timer/deadline | 없음 | `GameSocketClient` | 15초 recovery window |
| chat acceptance | 모든 parsed event | hook + `isChatForActiveRoom` | active room 수명 |

## 9. Thread 최종 상태

마지막 SHA에서 reducer는 browser-visible game connection state를, `GameSocketClient`는 ticket/socket/reconnect timer/input sequence를 소유합니다. stale generation과 stale snapshot은 폐기되고 reconnect는 fresh ticket으로 existing room을 이어가며 initial matchmaking command를 반복하지 않습니다. game chat은 현재 room과 scope가 일치할 때만 state로 들어갑니다.

### 최종 실행 흐름

1. hook이 `GameSocketClient.connect(initialEvent, handlers)`를 호출합니다.
2. client가 이전 request/socket/timer를 정리하고 generation을 증가시킨 뒤 ticket을 요청합니다.
3. current generation이면 socket을 열고 최초 연결에서만 initial command를 전송합니다.
4. raw message는 shared parser를 통과한 뒤 hook callback이 reducer action으로 변환합니다.
5. reducer는 stale snapshot을 거부하고 room-aware lifecycle state를 반환합니다.
6. remote close에서 room이 남아 있으면 bounded timer가 fresh ticket socket을 열되 initial command는 보내지 않습니다.
7. chat event는 current room predicate를 통과한 경우에만 message action으로 dispatch됩니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 04-game-connection-reducer-and-transport-client.md =====

===== BEGIN FILE: 05-game-connection-hook-migration-and-legacy-removal.md =====
# Game connection hook 전환과 legacy 제거

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

reducer와 transport client를 `useGameConnection`으로 조합하고 play page의 inline socket, timer input, command, duplicate state를 replacement-first 순서로 제거해 page가 presentation과 user intent만 소유하도록 하는 migration을 복원합니다.

### 직접 연결되는 불변식

- play page는 raw WebSocket, ticket request, protocol parsing, connection state를 직접 소유하지 않습니다.
- 모든 rendered game state와 ready/chat/pause/input command는 하나의 hook surface를 통해 오갑니다.
- legacy path는 replacement가 caller에 연결된 뒤에만 제거되며 migration 종료 시 duplicate owner가 남지 않습니다.
- keyboard/touch 입력은 방향 변화만 전송하고 key release, blur, hidden, pointer 종료에서 neutral `0`으로 복귀합니다.

## 2. 핵심 질문

- hook이 client callback을 reducer action으로 바꾸는 exact mapping은 무엇입니까?
- migration 중 legacy/new path가 동시에 connection 또는 command를 만들지 않는 근거는 무엇입니까?
- 각 제거 commit 전에 replacement path가 이미 어떤 caller를 제공합니까?
- keyboard/touch listener와 neutralization cleanup은 어떤 browser event에서 실행됩니까?
- 마지막 SHA에서 play page에 남는 local state와 책임은 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `9d70eb12e1d7` | `refactor(web): game connection hook 상태 연결` | B | AUTH, REALTIME, TOURNAMENT | client와 reducer를 React hook 뒤에서 조합합니다. |
| 2 | `748079c73eea` | `refactor(web): game connection hook 명령 연결` | B | PROTOCOL, SIMULATION, REALTIME | ready/input/chat/pause/resume command surface를 hook에 추가합니다. |
| 3 | `c33412d639c5` | `refactor(play): connection hook 전환 경계 준비` | B | PERSISTENCE, WEB | legacy path를 유지한 채 hook을 play page boundary에 도입합니다. |
| 4 | `898d0884ee37` | `refactor(play): 자동 경기 진입을 connection hook으로 전환` | B | REALTIME, TOURNAMENT, WEB | URL-driven queue/AI/tournament intent를 hook으로 전환합니다. |
| 5 | `9a67234633b4` | `refactor(play): 경기 상태와 명령을 connection hook에 연결` | B | REALTIME, PERSISTENCE, WEB | render state와 ready/chat/pause command를 hook output으로 전환합니다. |
| 6 | `1ae6fa7836d8` | `feat(play): keyboard와 touch paddle 입력 연결` | B | PROTOCOL, SIMULATION, REALTIME | keyboard와 mobile pointer control을 transition-based paddle command에 연결합니다. |
| 7 | `31b5122add6f` | `refactor(play): legacy paddle input loop 제거` | B | SIMULATION, REALTIME, WEB | component-local keyboard state와 50ms command loop를 제거합니다. |
| 8 | `fa35e8d15b4e` | `refactor(play): legacy WebSocket lifecycle 제거` | B | AUTH, PROTOCOL, REALTIME | inline socket create/listener/close path를 제거합니다. |
| 9 | `365d66c72343` | `refactor(play): legacy 경기 명령 제거` | B | PROTOCOL, REALTIME, WEB | page-local ready/chat/pause/resume/close command path를 제거합니다. |
| 10 | `06664abbae7f` | `refactor(play): legacy socket 상태 제거` | B | AUTH, PROTOCOL, REALTIME | duplicate socket/game-state field와 old API import를 제거합니다. |
| 11 | `9faada0df1d7` | `refactor(play): connection hook 전환 마무리` | A | PROTOCOL, REALTIME, TOURNAMENT | page가 hook의 단일 state/command surface만 소비하도록 migration을 끝냅니다. |

## 5. Commit별 학습 기록

### 5.1. `refactor(web): game connection hook 상태 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `9d70eb12e1d7` |
| Importance | B |
| Tags | AUTH, REALTIME, TOURNAMENT |
| Source에서 확정된 역할 | client와 reducer를 React hook 뒤에서 조합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/useGameConnection.ts`의 `useReducer`, memoized `GameSocketClient`, cleanup effect를 확인합니다.
- server event별 reducer action mapping과 failure message 변환을 확인합니다.
- 이 SHA에서 hook이 제공하는 connection initiator와 아직 제공하지 않는 command를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | reducer와 transport client는 분리됐지만 React component가 둘을 조합할 stable lifecycle boundary가 없었습니다. |
| 해결하려던 문제 | client instance를 rerender마다 재생성하지 않고 callback을 reducer action에 연결하며 unmount에서 close해야 했습니다. |
| 핵심 결정 | `useGameConnection`이 reducer와 memoized client를 만들고 parsed event를 action으로 dispatch하며 cleanup에서 client를 닫도록 했습니다. |
| 입력 → 상태 전이 → 출력 | React caller → hook connect → client callback → event mapping → reducer dispatch → state 반환 순서입니다. |
| ownership/lifetime/cleanup | hook instance가 client lifetime과 reducer를 조합하고 client는 transport resource, reducer는 state를 계속 소유합니다. |
| failure/rollback/retry | unmount cleanup은 client close를 호출하고 unknown failure는 user-facing notice로 축약합니다. |
| 보장하는 것 | React tree 안에서 connection state와 transport를 하나의 reusable hook으로 결합합니다. |
| 보장하지 않는 것 | ready/chat/pause/input command surface와 play page adoption은 아직 보장하지 않습니다. |
| 후속 연결 | `748079c73eea`가 command surface를 추가하고 `c33412d639c5`부터 page migration을 시작합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show 9d70eb12e1d7^` 및 `git diff 9d70eb12e1d7^ 9d70eb12e1d7`로 비교합니다.
- Thread 내 다음 관련 SHA: `748079c73eea` — `refactor(web): game connection hook 명령 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `refactor(web): game connection hook 명령 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `748079c73eea` |
| Importance | B |
| Tags | PROTOCOL, SIMULATION, REALTIME |
| Source에서 확정된 역할 | ready/input/chat/pause/resume command surface를 hook에 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `useGameConnection`의 `ready`, `sendChat`, `pause`, `resume`, `setDirection` callback을 확인합니다.
- 각 command가 current `roomId`, status, trimmed body 조건을 확인한 뒤 client `send`를 호출하는지 확인합니다.
- send failure 시 reducer notice를 바꾸는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | hook은 connection state를 제공했지만 active match command는 여전히 play page의 socket ref를 통해 전송됐습니다. |
| 해결하려던 문제 | state owner와 command owner가 같아야 room/status guard가 한 곳에서 일관되게 적용됩니다. |
| 핵심 결정 | hook에 command callbacks를 추가하고 current reducer state에 따라 valid command만 client로 보냈습니다. |
| 입력 → 상태 전이 → 출력 | page intent → hook callback → room/status validation → client send/sequence → failure notice 순서입니다. |
| ownership/lifetime/cleanup | hook이 command policy를, client가 serialization/transport를 소유합니다. |
| failure/rollback/retry | room/status/body 조건이 맞지 않으면 전송하지 않고 closed socket이면 visible failure state로 연결할 수 있습니다. |
| 보장하는 것 | caller가 raw socket 없이 active game command를 수행할 수 있습니다. |
| 보장하지 않는 것 | page가 실제로 hook command만 사용하는지는 아직 보장하지 않습니다. |
| 후속 연결 | `9a67234633b4`가 rendered state/command caller를 hook으로 옮깁니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `9d70eb12e1d7` — `refactor(web): game connection hook 상태 연결`
- Thread 내 다음 관련 SHA: `c33412d639c5` — `refactor(play): connection hook 전환 경계 준비`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `refactor(play): connection hook 전환 경계 준비`

| 항목 | 값 |
| --- | --- |
| SHA | `c33412d639c5` |
| Importance | B |
| Tags | PERSISTENCE, WEB |
| Source에서 확정된 역할 | legacy path를 유지한 채 hook을 play page boundary에 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/play/page.tsx`에 `useGameConnection`, `directionForKey`, `isEditableTarget` import와 hook call이 추가되는지 확인합니다.
- legacy socket refs/state/functions가 그대로 남아 있는지 확인합니다.
- 새 hook output이 아직 render/command caller에 사용되지 않는 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 새 hook은 독립적으로 존재했지만 실제 play page는 inline implementation만 사용했습니다. |
| 해결하려던 문제 | 큰 일괄 rewrite 대신 replacement를 먼저 page에 주입해 단계적으로 caller를 옮길 필요가 있었습니다. |
| 핵심 결정 | hook과 input helper를 page에 생성하되 기존 path를 제거하지 않는 migration seam을 만들었습니다. |
| 입력 → 상태 전이 → 출력 | render 시 hook instance가 만들어지지만 기존 event/command는 아직 legacy state/socket을 사용합니다. |
| ownership/lifetime/cleanup | 일시적으로 hook과 legacy page가 각각 state/client를 가질 수 있으므로 아직 단일 owner가 아닙니다. |
| failure/rollback/retry | 새 hook이 실제 connect를 호출하지 않는 한 두 socket이 동시에 열리지는 않지만 duplicate implementation은 남습니다. |
| 보장하는 것 | 후속 commit이 작은 단위로 caller를 이동할 준비를 합니다. |
| 보장하지 않는 것 | 이 SHA 자체는 ownership 이전을 완료하지 않습니다. |
| 후속 연결 | `898d0884ee37`이 첫 실제 caller인 URL-driven auto entry를 hook으로 전환합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `748079c73eea` — `refactor(web): game connection hook 명령 연결`
- Thread 내 다음 관련 SHA: `898d0884ee37` — `refactor(play): 자동 경기 진입을 connection hook으로 전환`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `refactor(play): 자동 경기 진입을 connection hook으로 전환`

| 항목 | 값 |
| --- | --- |
| SHA | `898d0884ee37` |
| Importance | B |
| Tags | REALTIME, TOURNAMENT, WEB |
| Source에서 확정된 역할 | URL-driven queue/AI/tournament intent를 hook으로 전환합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 search params 처리와 `autoStartedRef`를 확인합니다.
- queue, AI, tournament mode가 각각 hook의 어떤 connect callback을 호출하는지 확인합니다.
- legacy auto-connect function이 더 이상 effect caller에서 사용되지 않는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | page에 hook이 있었지만 URL query로 자동 진입하는 effect는 legacy connection function을 호출했습니다. |
| 해결하려던 문제 | 동일 route intent가 새 connection owner를 사용해야 후속 state/command migration이 가능했습니다. |
| 핵심 결정 | auto-start effect의 queue/AI/tournament branch를 hook callback으로 교체하고 한 번만 실행하는 ref를 유지했습니다. |
| 입력 → 상태 전이 → 출력 | search params → mode/tournament 판정 → auto-start guard → hook connect callback 순서입니다. |
| ownership/lifetime/cleanup | page가 URL intent와 one-shot guard를, hook이 connection start를 소유합니다. |
| failure/rollback/retry | effect dependency가 바뀌어도 `autoStartedRef`가 같은 mount에서 중복 시작을 막습니다. |
| 보장하는 것 | URL-driven connection은 새 client/reducer path로 들어갑니다. |
| 보장하지 않는 것 | manual buttons와 rendered state/commands는 아직 legacy일 수 있습니다. |
| 후속 연결 | `9a67234633b4`가 화면 state와 command를 hook으로 연결합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `c33412d639c5` — `refactor(play): connection hook 전환 경계 준비`
- Thread 내 다음 관련 SHA: `9a67234633b4` — `refactor(play): 경기 상태와 명령을 connection hook에 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `refactor(play): 경기 상태와 명령을 connection hook에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `9a67234633b4` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, WEB |
| Source에서 확정된 역할 | render state와 ready/chat/pause command를 hook output으로 전환합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`가 score, phase, room, status, messages, opponent를 hook state에서 읽는지 확인합니다.
- ready/chat/pause/resume handler가 hook command를 호출하는지 확인합니다.
- legacy functions/refs가 아직 파일에 남지만 caller가 사라진 범위를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | auto entry만 새 path를 사용하고 UI는 legacy state/socket을 읽어 두 owner가 서로 다른 내용을 렌더링할 수 있었습니다. |
| 해결하려던 문제 | 화면이 새 reducer state를 source로 사용하고 주요 command도 같은 hook을 거쳐야 했습니다. |
| 핵심 결정 | render derivation과 ready/chat/pause/resume caller를 hook state/command로 교체했습니다. |
| 입력 → 상태 전이 → 출력 | hook reducer state → score/phase/canX/opponent/messages derivation → UI; user action → hook command 순서입니다. |
| ownership/lifetime/cleanup | hook이 rendered connection state와 command policy의 실질적 owner가 되고 page에는 presentation state가 남습니다. |
| failure/rollback/retry | legacy definitions은 dead/unused 상태로 남아 bundle/typecheck에 존재하며 입력 loop도 아직 별도입니다. |
| 보장하는 것 | visible game state와 주요 command가 동일 hook owner를 사용합니다. |
| 보장하지 않는 것 | keyboard/touch와 legacy resource code 제거는 아직 완료되지 않았습니다. |
| 후속 연결 | `1ae6fa7836d8`이 입력을 새 path에 연결하고 이후 네 제거 commit이 dead owner를 삭제합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `898d0884ee37` — `refactor(play): 자동 경기 진입을 connection hook으로 전환`
- Thread 내 다음 관련 SHA: `1ae6fa7836d8` — `feat(play): keyboard와 touch paddle 입력 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `feat(play): keyboard와 touch paddle 입력 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `1ae6fa7836d8` |
| Importance | B |
| Tags | PROTOCOL, SIMULATION, REALTIME |
| Source에서 확정된 역할 | keyboard와 mobile pointer control을 transition-based paddle command에 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 `changeDirection` callback과 previous direction ref를 확인합니다.
- keydown/keyup, `blur`, `visibilitychange`, editable target, touch pointer down/up/cancel/leave listener를 확인합니다.
- direction이 실제로 바뀔 때만 hook `setDirection`을 호출하고 종료 event에서 `0`을 보내는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 기존 keyboard loop는 현재 방향을 50ms마다 반복 전송했고 mobile touch control과 focus/visibility neutralization이 부족했습니다. |
| 해결하려던 문제 | 사용자 intent 변화만 전송하되 key release, focus loss, hidden tab, pointer 종료에서 paddle가 계속 움직이지 않아야 했습니다. |
| 핵심 결정 | previous direction을 ref로 기억하고 변화가 있을 때만 command를 보내며 keyboard/touch/blur/visibility cleanup을 모두 neutral `0`에 연결했습니다. |
| 입력 → 상태 전이 → 출력 | DOM/pointer event → pure key/target helper → `changeDirection` dedupe → hook `setDirection` → client sequence send 순서입니다. |
| ownership/lifetime/cleanup | page가 DOM listener와 current physical input ref를 소유하고 hook/client가 protocol command를 소유합니다. |
| failure/rollback/retry | playing이 아니거나 editable target이면 movement를 시작하지 않고 release/blur/hidden/pointer end는 `0`을 보냅니다. effect cleanup도 listener를 제거하고 neutralize합니다. |
| 보장하는 것 | keyboard와 touch 입력이 변화 기반으로 전송되고 browser lifecycle 종료점에서 neutral 상태로 돌아갑니다. |
| 보장하지 않는 것 | server가 direction을 수락했는지나 packet loss 보상은 보장하지 않습니다. |
| 후속 연결 | `31b5122add6f`가 기존 50ms page loop를 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `9a67234633b4` — `refactor(play): 경기 상태와 명령을 connection hook에 연결`
- Thread 내 다음 관련 SHA: `31b5122add6f` — `refactor(play): legacy paddle input loop 제거`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.7. `refactor(play): legacy paddle input loop 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `31b5122add6f` |
| Importance | B |
| Tags | SIMULATION, REALTIME, WEB |
| Source에서 확정된 역할 | component-local keyboard state와 50ms command loop를 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`에서 old direction ref, key listener, `setInterval(50)` 전송 effect가 삭제되는지 확인합니다.
- `1ae6fa7836d8`의 replacement listener가 그대로 남아 input caller를 제공하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 새 transition-based input이 동작하지만 old 50ms polling loop도 남아 duplicate command를 보낼 수 있었습니다. |
| 해결하려던 문제 | 한 physical input에 두 protocol writers가 존재하지 않아야 했습니다. |
| 핵심 결정 | legacy direction state와 timer effect를 삭제하고 새 `changeDirection` path만 유지했습니다. |
| 입력 → 상태 전이 → 출력 | keyboard/touch event → hook command의 한 경로만 남습니다. |
| ownership/lifetime/cleanup | page는 DOM listener만 소유하고 periodic command timer ownership은 제거됩니다. |
| failure/rollback/retry | cleanup 대상 interval 자체가 사라져 duplicate send와 timer leak 가능성을 없앱니다. |
| 보장하는 것 | paddle input command writer가 하나로 줄어듭니다. |
| 보장하지 않는 것 | legacy socket lifecycle과 다른 command/state는 아직 파일에 남습니다. |
| 후속 연결 | `fa35e8d15b4e`가 다음 중복 owner인 inline WebSocket lifecycle을 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `1ae6fa7836d8` — `feat(play): keyboard와 touch paddle 입력 연결`
- Thread 내 다음 관련 SHA: `fa35e8d15b4e` — `refactor(play): legacy WebSocket lifecycle 제거`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.8. `refactor(play): legacy WebSocket lifecycle 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `fa35e8d15b4e` |
| Importance | B |
| Tags | AUTH, PROTOCOL, REALTIME |
| Source에서 확정된 역할 | inline socket create/listener/close path를 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`에서 ticket request, `new WebSocket`, onopen/onmessage/onerror/onclose 설정 함수가 삭제되는지 확인합니다.
- URL auto-start와 manual start가 모두 hook connect를 사용해 replacement가 완성됐는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | hook/client가 실제 connection을 소유하는데 page의 old socket factory와 event handlers가 dead code로 남아 있었습니다. |
| 해결하려던 문제 | 동일한 lifecycle 구현이 두 곳에 남으면 후속 변경이 한쪽에만 적용되고 실수로 재사용될 수 있었습니다. |
| 핵심 결정 | legacy connection functions와 raw event parsing path를 삭제했습니다. |
| 입력 → 상태 전이 → 출력 | 모든 connection intent가 hook → client path로만 흐릅니다. |
| ownership/lifetime/cleanup | `GameSocketClient`가 유일한 ticket/socket owner가 되고 page의 resource acquisition 책임이 사라집니다. |
| failure/rollback/retry | page는 더 이상 old handler cleanup을 수행하지 않습니다. hook cleanup이 client close를 담당합니다. |
| 보장하는 것 | raw WebSocket lifecycle implementation이 한 곳으로 통합됩니다. |
| 보장하지 않는 것 | page-local command wrappers와 duplicate state fields는 아직 남을 수 있습니다. |
| 후속 연결 | `365d66c72343`과 `06664abbae7f`가 남은 command/state를 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `31b5122add6f` — `refactor(play): legacy paddle input loop 제거`
- Thread 내 다음 관련 SHA: `365d66c72343` — `refactor(play): legacy 경기 명령 제거`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.9. `refactor(play): legacy 경기 명령 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `365d66c72343` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, WEB |
| Source에서 확정된 역할 | page-local ready/chat/pause/resume/close command path를 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 old `ready`, chat submit, toggle pause, close function과 cleanup effect 삭제를 확인합니다.
- UI handler가 hook command를 직접 호출하는 replacement를 유지하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | UI는 hook command를 사용하지만 같은 이름/역할의 page-local socket command 함수가 dead code로 남아 있었습니다. |
| 해결하려던 문제 | protocol command policy는 hook 한 곳만 소유해야 했습니다. |
| 핵심 결정 | legacy command와 socket close wrapper를 삭제하고 UI→hook caller만 남겼습니다. |
| 입력 → 상태 전이 → 출력 | button/form intent → hook command → client send의 단일 경로입니다. |
| ownership/lifetime/cleanup | hook이 command policy와 client close를 소유하고 page는 form input/DOM event만 소유합니다. |
| failure/rollback/retry | old cleanup effect가 사라져도 hook unmount cleanup이 client를 닫습니다. |
| 보장하는 것 | ready/chat/pause/resume의 duplicate implementation을 제거합니다. |
| 보장하지 않는 것 | duplicate state/ref/import는 다음 commit 전까지 남을 수 있습니다. |
| 후속 연결 | `06664abbae7f`가 사용되지 않는 socket/game fields를 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `fa35e8d15b4e` — `refactor(play): legacy WebSocket lifecycle 제거`
- Thread 내 다음 관련 SHA: `06664abbae7f` — `refactor(play): legacy socket 상태 제거`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.10. `refactor(play): legacy socket 상태 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `06664abbae7f` |
| Importance | B |
| Tags | AUTH, PROTOCOL, REALTIME |
| Source에서 확정된 역할 | duplicate socket/game-state field와 old API import를 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`에서 raw parser, `requestWsTicket`, WS URL, snapshot/room/status/messages state와 socket/ticket/sequence refs 삭제를 확인합니다.
- 렌더링에 필요한 값이 hook state 또는 presentation-only state에서만 오는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | legacy lifecycle/command는 삭제됐지만 관련 imports, refs, state fields가 남아 ownership을 혼동시켰습니다. |
| 해결하려던 문제 | migration의 실제 완료 조건은 old owner의 data structure까지 사라지는 것이었습니다. |
| 핵심 결정 | raw transport imports와 duplicate state/ref를 삭제했습니다. |
| 입력 → 상태 전이 → 출력 | page render는 hook state를 읽고 local state는 chat input/touch 등 presentation intent로 제한됩니다. |
| ownership/lifetime/cleanup | socket/ticket/snapshot lifecycle owner가 page에서 완전히 사라집니다. |
| failure/rollback/retry | 삭제된 resource에 대한 cleanup도 더 이상 page에 존재하지 않으며 hook/client만 담당합니다. |
| 보장하는 것 | duplicate game connection state가 제거됩니다. |
| 보장하지 않는 것 | hook output alias와 migration scaffolding 일부는 아직 남습니다. |
| 후속 연결 | `9faada0df1d7`이 최종 API surface와 markup을 정리합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `365d66c72343` — `refactor(play): legacy 경기 명령 제거`
- Thread 내 다음 관련 SHA: `9faada0df1d7` — `refactor(play): connection hook 전환 마무리`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.11. `refactor(play): connection hook 전환 마무리`

| 항목 | 값 |
| --- | --- |
| SHA | `9faada0df1d7` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, TOURNAMENT |
| Source에서 확정된 역할 | page가 hook의 단일 state/command surface만 소비하도록 migration을 끝냅니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 hook destructuring과 queue/AI/tournament auto-start caller를 확인합니다.
- 남은 alias/ref가 제거되고 aria-live notice, button guards, presentation local state만 남는지 확인합니다.
- `useGameConnection` cleanup과 page effects가 duplicate connection을 만들지 않는 dependency를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | legacy resource는 제거됐지만 adapter aliases와 migration-oriented indirection이 남아 최종 owner가 코드 표면에서 명확하지 않았습니다. |
| 해결하려던 문제 | page가 hook contract를 직접 소비하고 남은 책임을 presentation/user intent로 제한해야 했습니다. |
| 핵심 결정 | hook state/commands를 직접 destructure하고 route auto-start, render derivation, controls를 그 surface에 맞춰 정리했습니다. |
| 입력 → 상태 전이 → 출력 | URL/user event → hook command → client/reducer → hook state → page render의 한 방향 흐름으로 완성됩니다. |
| ownership/lifetime/cleanup | hook이 connection lifecycle/state/commands를 단독 소유하고 page는 search params, controlled chat input, DOM input listeners, presentation을 소유합니다. |
| failure/rollback/retry | unmount는 hook cleanup을 통해 client를 닫고 effect guards가 같은 mount의 duplicate auto-start를 막습니다. |
| 보장하는 것 | play page에 raw transport/protocol state owner가 남지 않고 replacement-first migration이 종료됩니다. |
| 보장하지 않는 것 | server recovery, browser cache, canvas interpolation correctness는 이 Thread 단독으로 보장하지 않습니다. |
| 후속 연결 | transport Thread의 tests와 rendering Thread의 snapshot/input invariants가 최종 hook surface를 보완합니다. |

#### A-level 불변식 종합

- **핵심 책임:** page가 hook의 단일 state/command surface만 소비하도록 migration을 끝냅니다.
- **실패 영향:** page가 hook contract를 직접 소비하고 남은 책임을 presentation/user intent로 제한해야 했습니다.
- **결과 불변식:** play page에 raw transport/protocol state owner가 남지 않고 replacement-first migration이 종료됩니다.
- **남은 제한:** server recovery, browser cache, canvas interpolation correctness는 이 Thread 단독으로 보장하지 않습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `9faada0df1d7` |
| 파일 | `apps/web/src/app/play/page.tsx` |
| 함수/위치 | `PlayPage hook consumption` |
| 근거 요약 | 최종 page는 `useGameConnection()`의 state와 command를 직접 소비하며 socket, ticket, raw parser, input sequence를 선언하지 않습니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `06664abbae7f` — `refactor(play): legacy socket 상태 제거`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| React 조합 경계 | `9d70eb12e1d7` → `748079c73eea` | hook이 client/reducer와 active command를 조합했습니다. |
| replacement 주입 | `c33412d639c5` | legacy를 유지한 채 새 hook이 page에 들어왔습니다. |
| caller 이전 | `898d0884ee37` → `9a67234633b4` → `1ae6fa7836d8` | auto entry, rendered state/commands, keyboard/touch 순서로 새 owner에 연결됐습니다. |
| legacy owner 제거 | `31b5122add6f` → `06664abbae7f` | timer, WebSocket, command, state/ref를 단계적으로 삭제했습니다. |
| migration 완료 | `9faada0df1d7` | page는 hook의 단일 contract만 소비합니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| new hook과 legacy page가 공존 | duplicate socket/command 위험 | replacement caller를 먼저 옮긴 뒤 각 legacy layer 삭제 | transport unit tests가 replacement behavior를 보호 | new hook과 legacy page가 공존 → duplicate socket/command 위험 → replacement caller를 먼저 옮긴 뒤 각 legacy layer 삭제 → transport unit tests가 replacement behavior를 보호 |
| 50ms polling input과 transition input 공존 | duplicate direction frame | `31b5122add6f` | input helper/client tests | 50ms polling input과 transition input 공존 → duplicate direction frame → `31b5122add6f` → input helper/client tests |
| raw socket fields가 page에 잔존 | ownership 혼동과 stale maintenance | `fa35e8d15b4e` → `06664abbae7f` | `9faada0df1d7` 최종 structural completion | raw socket fields가 page에 잔존 → ownership 혼동과 stale maintenance → `fa35e8d15b4e` → `06664abbae7f` → `9faada0df1d7` 최종 structural completion |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| connection composition | play page | `useGameConnection` | hook instance 수명 |
| raw transport | play page refs/functions | `GameSocketClient` | current connection 수명 |
| connection state | page `useState` | hook reducer | hook 수명 |
| game commands | page-local functions | hook callbacks | current room/status 수명 |
| physical input listeners | legacy keyboard/timer | page DOM listener + hook command | page mount 수명 |
| neutral input cleanup | 부분적 keyup | keyup/blur/hidden/pointer end/effect cleanup | browser focus/pointer lifecycle |

## 9. Thread 최종 상태

마지막 SHA에서 `PlayPage`는 raw socket, ticket, parser, snapshot sequence, input sequence, connection status를 선언하지 않습니다. URL과 사용자 intent를 hook command에 전달하고 hook state를 렌더링하며, DOM keyboard/touch listener는 방향 변화와 neutralization만 소유합니다. transport와 reducer는 별도 Thread의 전용 owner로 남습니다.

### 최종 실행 흐름

1. page가 route query를 해석해 queue/AI/tournament intent를 hook connect callback에 전달합니다.
2. hook이 `GameSocketClient` callback을 reducer action으로 바꾸고 state를 반환합니다.
3. page는 hook state에서 room, phase, score, opponent, messages, command availability를 파생합니다.
4. button/form intent는 hook의 ready/chat/pause/resume command를 호출합니다.
5. keyboard/touch event는 pure helper와 direction dedupe를 거쳐 hook input command로 전달됩니다.
6. keyup, blur, hidden, pointer 종료, effect cleanup은 neutral `0`을 전송합니다.
7. unmount 시 hook cleanup이 client의 request/socket/timer를 정리합니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 05-game-connection-hook-migration-and-legacy-removal.md =====

===== BEGIN FILE: 06-react-query-cache-ownership-and-invalidation.md =====
# React Query cache ownership과 invalidation

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

component-owned fetch effect를 하나의 `QueryClient`, stable query key, endpoint별 stale time, AbortSignal, mutation invalidation, session-expiry cache eviction, WebSocket cache update 규칙을 갖는 canonical browser server-state owner로 이동하는 과정을 복원합니다.

### 직접 연결되는 불변식

- browser server state는 application 수명 동안 하나의 `QueryClient`와 scope가 명시된 stable key가 소유합니다.
- query function은 React Query가 제공한 `AbortSignal`을 endpoint helper에 전달합니다.
- 401/session-expired는 private cache를 제거하고 `me`를 `null`로 만들되 public leaderboard/profile/tournament cache를 불필요하게 지우지 않습니다.
- mutation은 관련 exact key만 invalidation하며 WebSocket push도 같은 canonical cache를 갱신합니다.
- authentication error는 retry하지 않고 retry 가능한 일반 failure만 제한적으로 재시도합니다.

## 2. 핵심 질문

- query key가 global, handle, user, tournament scope를 어떤 tuple로 표현합니까?
- session expiry 중 fetching query를 제거할 때 current callback을 방해하지 않도록 어떤 scheduling을 사용합니까?
- lobby HTTP read, chat mutation, WebSocket push가 같은 cache entry를 어떻게 갱신합니까?
- 각 screen migration 뒤 local effect/data state가 실제로 제거됐습니까?
- test가 exact invalidation과 public/private cache 보존을 어떤 `QueryClient`/`QueryObserver`로 증명합니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `e000e3d6a460` | `refactor(web): query key와 retry 정책 정의` | B | AUTH, TOURNAMENT, WEB | screen migration 전에 cache key vocabulary와 retry policy를 정의합니다. |
| 2 | `d05a962d8829` | `refactor(web): session query와 cache invalidation 추가` | A | TOURNAMENT, WEB, OBSERVABILITY | endpoint별 query option과 session-expiry invalidation을 정의합니다. |
| 3 | `80ec34fde74c` | `refactor(web): React Query provider 연결` | B | AUTH, WEB, OBSERVABILITY | application root에 단일 `QueryClient`를 설치합니다. |
| 4 | `931800f796e1` | `refactor(web): lobby와 login을 query cache로 전환` | B | AUTH, REALTIME, WEB | HTTP load, socket event, login/chat mutation이 동일 lobby/session cache owner를 갱신합니다. |
| 5 | `8a44c23f15de` | `refactor(web): dashboard와 leaderboard를 query cache로 전환` | B | WEB | dashboard와 leaderboard component effect를 shared cache로 이동합니다. |
| 6 | `e2ccee689642` | `refactor(web): profile 조회를 query cache로 전환` | B | WEB | route handle을 query identity로 사용합니다. |
| 7 | `045d0cd2c171` | `refactor(web): tournament 조회와 mutation을 query cache로 전환` | B | TOURNAMENT, WEB | tournament read와 create/join mutation을 cache로 이동합니다. |
| 8 | `0b1a6bcb4311` | `refactor(web): admin 조회와 mutation을 query cache로 전환` | B | AUTH, WEB | admin read/status mutation을 cache로 이동합니다. |
| 9 | `0e0c9645ab2d` | `refactor(web): shell의 session 소비를 query cache로 통합` | B | AUTH, WEB | `AppShell`의 private session effect를 shared `me` query로 교체합니다. |
| 10 | `1ebdce4cdf0a` | `test(web): query cache key·retry·invalidation 검증` | A | AUTH, TOURNAMENT, WEB | key scoping, retry exclusion, mutation/session invalidation을 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `refactor(web): query key와 retry 정책 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `e000e3d6a460` |
| Importance | B |
| Tags | AUTH, TOURNAMENT, WEB |
| Source에서 확정된 역할 | screen migration 전에 cache key vocabulary와 retry policy를 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/query.ts`의 `queryKeys`, `mutationInvalidations`, `invalidateExactQueries`, retry predicate를 확인합니다.
- profile handle, tournament/admin/lobby/me key가 서로 prefix collision을 만들지 않는지 확인합니다.
- `ApiError` 401과 일반 failure의 retry 횟수를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 각 page가 effect/local state를 사용해 동일 endpoint를 중복 요청하고 cache identity와 invalidation 규칙이 없었습니다. |
| 해결하려던 문제 | screen을 옮기기 전에 server resource를 식별하고 mutation 영향 범위를 표현할 shared vocabulary가 필요했습니다. |
| 핵심 결정 | tuple 형태 query key, mutation별 invalidation key, `exact: true` helper, 401 무재시도/기타 1회 retry 정책을 정의했습니다. |
| 입력 → 상태 전이 → 출력 | query/mutation caller가 shared key/policy를 사용하지만 이 SHA에서는 아직 screen이 cache를 소비하지 않습니다. |
| ownership/lifetime/cleanup | query module이 key와 invalidation/retry 정책을 소유합니다. |
| failure/rollback/retry | 401은 즉시 종료하고 다른 failure는 제한 횟수만 retry합니다. 실제 fetch cancellation과 session eviction은 아직 없습니다. |
| 보장하는 것 | 이후 migration이 동일 key와 retry semantics를 사용할 기반을 제공합니다. |
| 보장하지 않는 것 | QueryClient instance나 endpoint option, screen adoption은 아직 보장하지 않습니다. |
| 후속 연결 | `d05a962d8829`가 query options와 session eviction을, `80ec34fde74c`가 application provider를 추가합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show e000e3d6a460^` 및 `git diff e000e3d6a460^ e000e3d6a460`로 비교합니다.
- Thread 내 다음 관련 SHA: `d05a962d8829` — `refactor(web): session query와 cache invalidation 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `refactor(web): session query와 cache invalidation 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `d05a962d8829` |
| Importance | A |
| Tags | TOURNAMENT, WEB, OBSERVABILITY |
| Source에서 확정된 역할 | endpoint별 query option과 session-expiry invalidation을 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `query.ts`의 `meQueryOptions`, lobby/dashboard/leaderboard/profile/tournament/admin options와 stale time을 확인합니다.
- 각 `queryFn({ signal })`이 API helper로 signal을 전달하는지 확인합니다.
- `expireSession`이 private keys를 remove하고 `me`를 `null`로 set하는 순서와 fetching query 제거 defer를 확인합니다.
- public key가 eviction 대상에서 제외되는 근거를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | key vocabulary는 있었지만 endpoint별 query function, cancellation, session-expiry cache transition이 정의되지 않았습니다. |
| 해결하려던 문제 | 401 이후 stale private data가 남거나 active fetch 제거가 observer callback을 비정상 상태에 둘 수 있었습니다. |
| 핵심 결정 | endpoint options를 만들고 React Query signal을 전달했으며 private cache를 선택적으로 제거하고 `me`를 null로 설정하는 `expireSession`을 추가했습니다. fetching query 제거는 다음 task로 미뤘습니다. |
| 입력 → 상태 전이 → 출력 | session-expired event → private query 상태 조사 → idle query 즉시 remove/fetching query deferred remove → `me` null set 순서입니다. |
| ownership/lifetime/cleanup | QueryClient가 server state를 소유하고 `expireSession`이 auth transition을 조정합니다. in-flight request lifetime은 query signal이 관리합니다. |
| failure/rollback/retry | public profile/leaderboard/tournament는 보존하고 private lobby/dashboard/admin 등은 제거합니다. current fetching callback을 동기 remove하지 않습니다. |
| 보장하는 것 | session expiry가 private cache를 명시적으로 무효화하고 active query를 pending에 가두지 않는 기반을 만듭니다. |
| 보장하지 않는 것 | root provider와 실제 screen adoption, exact behavior test는 아직 없습니다. |
| 후속 연결 | `80ec34fde74c`가 event listener/provider를 설치하고 `1ebdce4cdf0a`가 active unauthorized query까지 검증합니다. |

#### A-level 불변식 종합

- **핵심 책임:** endpoint별 query option과 session-expiry invalidation을 정의합니다.
- **실패 영향:** 401 이후 stale private data가 남거나 active fetch 제거가 observer callback을 비정상 상태에 둘 수 있었습니다.
- **결과 불변식:** session expiry가 private cache를 명시적으로 무효화하고 active query를 pending에 가두지 않는 기반을 만듭니다.
- **남은 제한:** root provider와 실제 screen adoption, exact behavior test는 아직 없습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `d05a962d8829` |
| 파일 | `apps/web/src/lib/query.ts` |
| 함수/위치 | `expireSession / query options` |
| 근거 요약 | query function은 제공된 `signal`을 API helper에 전달하고, session expiry는 private keys만 제거한 뒤 `me`를 `null`로 설정합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `e000e3d6a460` — `refactor(web): query key와 retry 정책 정의`
- Thread 내 다음 관련 SHA: `80ec34fde74c` — `refactor(web): React Query provider 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `refactor(web): React Query provider 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `80ec34fde74c` |
| Importance | B |
| Tags | AUTH, WEB, OBSERVABILITY |
| Source에서 확정된 역할 | application root에 단일 `QueryClient`를 설치합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/components/QueryProvider.tsx`의 `useState(() => new QueryClient(...))`를 확인합니다.
- default query/mutation options와 window focus refetch 설정을 확인합니다.
- session-expired event listener 등록/해제와 `expireSession(client)` 호출을 확인합니다.
- `layout.tsx`에서 provider가 application children을 감싸는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | query options는 존재했지만 React tree에 stable client가 없어 screen이 shared cache를 사용할 수 없었습니다. |
| 해결하려던 문제 | rerender마다 client가 재생성되지 않고 session event를 한 곳에서 처리하는 application lifetime owner가 필요했습니다. |
| 핵심 결정 | `useState` initializer로 QueryClient를 한 번 만들고 root provider와 session-expired listener를 연결했습니다. |
| 입력 → 상태 전이 → 출력 | application mount → QueryClient 생성 → Context 제공 → event listener; unmount → listener 제거 순서입니다. |
| ownership/lifetime/cleanup | provider component가 QueryClient와 global event listener lifetime을 소유합니다. |
| failure/rollback/retry | mutation은 기본 retry하지 않고 query는 shared retry policy를 사용하며 focus refetch 정책을 고정합니다. |
| 보장하는 것 | application 수명 동안 canonical cache client 하나를 제공합니다. |
| 보장하지 않는 것 | 각 screen이 실제로 local effect를 제거하는지는 아직 보장하지 않습니다. |
| 후속 연결 | `931800f796e1`부터 screen별 migration이 진행됩니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `d05a962d8829` — `refactor(web): session query와 cache invalidation 추가`
- Thread 내 다음 관련 SHA: `931800f796e1` — `refactor(web): lobby와 login을 query cache로 전환`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `refactor(web): lobby와 login을 query cache로 전환`

| 항목 | 값 |
| --- | --- |
| SHA | `931800f796e1` |
| Importance | B |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | HTTP load, socket event, login/chat mutation이 동일 lobby/session cache owner를 갱신합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `HomePage`의 `useQuery(meQueryOptions/lobbyQueryOptions)`와 제거된 fetch effect/local data state를 확인합니다.
- login mutation이 `me` cache를 set하고 어떤 exact keys를 invalidation하는지 확인합니다.
- lobby chat mutation과 WebSocket `chat.message`가 `queryKeys.lobby()`를 dedupe하고 최근 20개로 제한하는지 확인합니다.
- presence event가 lobby key를 invalidation하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | home/login이 page effect와 local arrays를 소유해 HTTP response, mutation result, WebSocket push가 서로 다른 state를 갱신했습니다. |
| 해결하려던 문제 | lobby의 canonical state를 한 cache entry로 통합하고 auth/chat mutation과 push event가 같은 owner를 사용해야 했습니다. |
| 핵심 결정 | `me`/`lobby` queries로 읽기를 옮기고 login/chat mutation 및 WebSocket event가 query cache를 set/invalidate하도록 바꿨습니다. |
| 입력 → 상태 전이 → 출력 | query fetch → lobby cache; chat POST/WS event → current cache update(dedupe/last20) → exact invalidation; login success → me set → related invalidation 순서입니다. |
| ownership/lifetime/cleanup | QueryClient가 session/lobby server state를, page는 chat input/notice와 socket ref만 소유합니다. |
| failure/rollback/retry | duplicate message ID는 제거되고 최근 20개만 유지됩니다. mutation error는 cache success로 위장하지 않습니다. |
| 보장하는 것 | HTTP와 realtime update가 하나의 lobby cache를 공유합니다. |
| 보장하지 않는 것 | lobby WebSocket 자체의 transport lifetime은 여전히 page effect가 소유합니다. |
| 후속 연결 | 후속 commits가 나머지 screen을 같은 cache model로 옮깁니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `80ec34fde74c` — `refactor(web): React Query provider 연결`
- Thread 내 다음 관련 SHA: `8a44c23f15de` — `refactor(web): dashboard와 leaderboard를 query cache로 전환`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `refactor(web): dashboard와 leaderboard를 query cache로 전환`

| 항목 | 값 |
| --- | --- |
| SHA | `8a44c23f15de` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | dashboard와 leaderboard component effect를 shared cache로 이동합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 두 page에서 `useEffect`, local resource/error/loading state가 제거되는지 확인합니다.
- `dashboardQueryOptions`, `leaderboardQueryOptions`와 `useQuery` result branch를 확인합니다.
- 기존 truthful loading/error/empty rendering이 query status로 보존되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | truthful state fix 뒤에도 dashboard/leaderboard는 route mount마다 직접 fetch하고 서로 독립 lifecycle을 가졌습니다. |
| 해결하려던 문제 | 동일 read model을 shared cache, stale time, cancellation 규칙으로 이동해야 했습니다. |
| 핵심 결정 | local effect/data state를 제거하고 endpoint query options를 `useQuery`로 소비했습니다. |
| 입력 → 상태 전이 → 출력 | page render → cache hit 또는 query fetch(signal) → query status/data → existing loading/error/empty/success UI 순서입니다. |
| ownership/lifetime/cleanup | QueryClient가 resource와 request lifetime을 소유하고 page는 presentation만 소유합니다. |
| failure/rollback/retry | unmount/stale query는 signal로 cancellation될 수 있으며 error는 query state로 남습니다. |
| 보장하는 것 | dashboard/leaderboard가 canonical cache와 retry/abort 정책을 사용합니다. |
| 보장하지 않는 것 | mutation이나 realtime update는 이 두 read-only screen에서 다루지 않습니다. |
| 후속 연결 | `1ebdce4cdf0a`가 key와 retry policy를 직접 검증합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `931800f796e1` — `refactor(web): lobby와 login을 query cache로 전환`
- Thread 내 다음 관련 SHA: `e2ccee689642` — `refactor(web): profile 조회를 query cache로 전환`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `refactor(web): profile 조회를 query cache로 전환`

| 항목 | 값 |
| --- | --- |
| SHA | `e2ccee689642` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | route handle을 query identity로 사용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `profile/[handle]/page.tsx`에서 params handle을 `profileQueryOptions(handle)`에 직접 전달하는지 확인합니다.
- handle 변경용 별도 effect/local profile state가 제거되는지 확인합니다.
- friend mutation 성공 뒤 어떤 exact key를 invalidation하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | profile page가 route handle 변화와 fetch를 effect/local state로 동기화했습니다. |
| 해결하려던 문제 | resource identity 자체인 handle을 query key에 포함해 다른 profile cache collision과 stale effect를 피해야 했습니다. |
| 핵심 결정 | route handle을 query option 입력으로 직접 사용하고 friend mutation은 관련 friendship/profile key만 invalidation하도록 옮겼습니다. |
| 입력 → 상태 전이 → 출력 | params resolve → handle-scoped query key → fetch(signal) → profile data; friend action → mutation → exact invalidation 순서입니다. |
| ownership/lifetime/cleanup | QueryClient가 handle별 profile state를, page는 clipboard/action notice만 소유합니다. |
| failure/rollback/retry | handle이 바뀌면 다른 key가 선택돼 이전 profile을 같은 state variable에 덮지 않습니다. |
| 보장하는 것 | profile cache가 route identity로 분리되고 request cancellation을 상속합니다. |
| 보장하지 않는 것 | server가 handle uniqueness를 보장하는지는 browser query가 증명하지 않습니다. |
| 후속 연결 | `1ebdce4cdf0a`가 key scoping을 회귀로 고정합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `8a44c23f15de` — `refactor(web): dashboard와 leaderboard를 query cache로 전환`
- Thread 내 다음 관련 SHA: `045d0cd2c171` — `refactor(web): tournament 조회와 mutation을 query cache로 전환`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.7. `refactor(web): tournament 조회와 mutation을 query cache로 전환`

| 항목 | 값 |
| --- | --- |
| SHA | `045d0cd2c171` |
| Importance | B |
| Tags | TOURNAMENT, WEB |
| Source에서 확정된 역할 | tournament read와 create/join mutation을 cache로 이동합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tournaments/page.tsx`의 tournaments/me queries와 selected ID local state를 구분합니다.
- create/join mutation 성공 후 `mutationInvalidations`의 exact key를 확인합니다.
- pending 상태에서 중복 submit/join을 막는 disabled 조건을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | tournament list와 mutation result를 page local array가 소유해 다른 route/cache와 일관된 갱신 규칙이 없었습니다. |
| 해결하려던 문제 | read state는 QueryClient로 옮기고 화면 고유 selection만 local에 남겨야 했습니다. |
| 핵심 결정 | tournaments/me를 query로, create/join을 mutation으로 바꾸고 success 후 정의된 exact keys를 invalidation했습니다. |
| 입력 → 상태 전이 → 출력 | query cache → list; selection local; action → mutation → exact invalidation/refetch → updated list 순서입니다. |
| ownership/lifetime/cleanup | QueryClient가 server list/session을, page가 selected tournament ID와 controlled input을 소유합니다. |
| failure/rollback/retry | pending 중 duplicate action을 막고 mutation failure는 query data를 임의로 성공 상태로 바꾸지 않습니다. |
| 보장하는 것 | tournament read/mutation이 cache invalidation contract를 따릅니다. |
| 보장하지 않는 것 | server-side bracket atomicity는 browser cache가 보장하지 않습니다. |
| 후속 연결 | `1ebdce4cdf0a`가 invalidation 범위를 검증합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `e2ccee689642` — `refactor(web): profile 조회를 query cache로 전환`
- Thread 내 다음 관련 SHA: `0b1a6bcb4311` — `refactor(web): admin 조회와 mutation을 query cache로 전환`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.8. `refactor(web): admin 조회와 mutation을 query cache로 전환`

| 항목 | 값 |
| --- | --- |
| SHA | `0b1a6bcb4311` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | admin read/status mutation을 cache로 이동합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `admin/page.tsx`의 users/actions queries와 status mutation을 확인합니다.
- mutation success 후 admin users와 actions key를 모두 exact invalidation하는지 확인합니다.
- message가 local copied users가 아니라 query/mutation status에서 파생되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | admin page가 protected users를 local array로 복제하고 mutation 결과로 직접 교체했습니다. |
| 해결하려던 문제 | authorization-sensitive server state를 shared cache와 session eviction 규칙에 포함해야 했습니다. |
| 핵심 결정 | admin users/actions를 query로, status 변경을 mutation으로 옮기고 관련 두 key를 invalidation했습니다. |
| 입력 → 상태 전이 → 출력 | query → protected rows; action → mutation → users/actions invalidation → refetch/render 순서입니다. |
| ownership/lifetime/cleanup | QueryClient가 protected admin state를 소유하고 page는 pending target/notice를 소유합니다. |
| failure/rollback/retry | session-expired 시 private admin keys가 eviction 대상이 되고 mutation failure는 cache를 성공으로 덮지 않습니다. |
| 보장하는 것 | admin state가 auth transition과 exact invalidation 규칙을 따릅니다. |
| 보장하지 않는 것 | server authorization/audit atomicity 자체는 증명하지 않습니다. |
| 후속 연결 | `0e0c9645ab2d`가 shell session까지 같은 cache owner로 통합합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `045d0cd2c171` — `refactor(web): tournament 조회와 mutation을 query cache로 전환`
- Thread 내 다음 관련 SHA: `0e0c9645ab2d` — `refactor(web): shell의 session 소비를 query cache로 통합`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.9. `refactor(web): shell의 session 소비를 query cache로 통합`

| 항목 | 값 |
| --- | --- |
| SHA | `0e0c9645ab2d` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | `AppShell`의 private session effect를 shared `me` query로 교체합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppShell.tsx`에서 `useEffect`, local me state, direct `getMe` import가 제거되는지 확인합니다.
- `useQuery(meQueryOptions())` 결과로 profile href와 disabled state를 계산하는지 확인합니다.
- HomePage/LoginPanel과 shell이 동일 `queryKeys.me()`를 소비하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | screen은 QueryClient를 사용하지만 shell만 별도 `getMe` effect/state를 가져 current identity owner가 둘이었습니다. |
| 해결하려던 문제 | navigation profile identity도 login/session-expiry와 같은 canonical cache를 사용해야 했습니다. |
| 핵심 결정 | `AppShell`의 direct effect를 제거하고 shared me query를 소비했습니다. |
| 입력 → 상태 전이 → 출력 | login/session event가 me cache를 갱신 → shell observer가 rerender → profile href/disabled navigation 갱신 순서입니다. |
| ownership/lifetime/cleanup | QueryClient가 current session의 유일한 browser server-state owner가 됩니다. |
| failure/rollback/retry | session expiry eviction과 query status가 shell에도 동일하게 반영됩니다. |
| 보장하는 것 | home, login, shell이 동일 session cache를 공유합니다. |
| 보장하지 않는 것 | browser navigation의 server authorization 자체는 보장하지 않습니다. |
| 후속 연결 | `1ebdce4cdf0a`가 session-expiry behavior를 direct client/observer test로 고정합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `0b1a6bcb4311` — `refactor(web): admin 조회와 mutation을 query cache로 전환`
- Thread 내 다음 관련 SHA: `1ebdce4cdf0a` — `test(web): query cache key·retry·invalidation 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.10. `test(web): query cache key·retry·invalidation 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1ebdce4cdf0a` |
| Importance | A |
| Tags | AUTH, TOURNAMENT, WEB |
| Source에서 확정된 역할 | key scoping, retry exclusion, mutation/session invalidation을 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/query.test.ts`에서 fresh `QueryClient`와 `QueryObserver`를 만드는 setup을 확인합니다.
- query key tuple exact equality와 adjacent/prefix key가 invalidation되지 않는 case를 확인합니다.
- 401 retry count, private eviction/public preservation, fetching unauthorized query가 error/idle로 settle하는지 확인합니다.
- mutation invalidation table이 login/friend/tournament/admin 영향 범위를 검증하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | screen migration은 완료됐지만 key typo, broad prefix invalidation, auth retry, session-expiry 중 active observer 정지 같은 회귀를 막는 직접 증거가 없었습니다. |
| 해결하려던 문제 | cache는 보이지 않는 global state이므로 exact identity와 transition을 real QueryClient로 검증해야 했습니다. |
| 핵심 결정 | isolated QueryClient에 public/private/active queries를 심고 mutation invalidation과 `expireSession`을 호출하며 observer 상태와 fetch count를 관찰했습니다. |
| 입력 → 상태 전이 → 출력 | test setup → cache seed/observer subscribe → production policy 실행 → exact cache entries/status/retry count assertion 순서입니다. |
| ownership/lifetime/cleanup | 각 test가 QueryClient/observer를 생성·정리하고 production functions를 직접 호출합니다. |
| failure/rollback/retry | 401은 retry하지 않고 private data만 제거되며 public keys는 남습니다. fetching unauthorized query도 pending에 고정되지 않습니다. |
| 보장하는 것 | key contract, exact invalidation, auth retry exclusion, session cache eviction의 high-value invariant를 증명합니다. |
| 보장하지 않는 것 | 실제 browser focus/network timing과 server response correctness는 증명하지 않습니다. |
| 후속 연결 | 이 test가 Thread의 canonical cache ownership을 최종 회귀로 보호합니다. |

#### A-level 불변식 종합

- **핵심 책임:** key scoping, retry exclusion, mutation/session invalidation을 검증합니다.
- **실패 영향:** cache는 보이지 않는 global state이므로 exact identity와 transition을 real QueryClient로 검증해야 했습니다.
- **결과 불변식:** key contract, exact invalidation, auth retry exclusion, session cache eviction의 high-value invariant를 증명합니다.
- **남은 제한:** 실제 browser focus/network timing과 server response correctness는 증명하지 않습니다.

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | stable scoped keys, exact invalidation, 401 무재시도, private-only session eviction이 유지됩니다. |
| 재현한 실패/경계 | prefix/adjacent key 오염, active unauthorized fetch, public cache 과잉 제거입니다. |
| 테스트 기법 | real `QueryClient`/`QueryObserver`, seeded cache, controlled query function입니다. |
| 증명하는 것 | cache transition과 observer settle 상태를 library implementation 위에서 deterministic하게 증명합니다. |
| 증명하지 않는 것 | 실제 network/browser focus/refetch cadence와 server data correctness는 증명하지 않습니다. |
| 검증 분류 | cache consistency·auth regression evidence |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `1ebdce4cdf0a` |
| 파일 | `apps/web/src/lib/query.test.ts` |
| 함수/위치 | `session expiration tests` |
| 근거 요약 | private queries를 제거하고 `me`를 `null`로 만들면서 leaderboard/profile/tournament cache는 유지하며 active unauthorized observer가 pending에 남지 않는지 확인합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `0e0c9645ab2d` — `refactor(web): shell의 session 소비를 query cache로 통합`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| key/retry vocabulary | `e000e3d6a460` | resource identity와 mutation 영향 범위가 코드로 정의됐습니다. |
| query/session transition | `d05a962d8829` | AbortSignal, stale time, private cache eviction이 추가됐습니다. |
| application owner | `80ec34fde74c` | browser 수명 동안 QueryClient 하나가 설치됐습니다. |
| screen migration | `931800f796e1` → `0e0c9645ab2d` | lobby부터 shell session까지 component effect ownership이 cache로 이동했습니다. |
| cache regression | `1ebdce4cdf0a` | exact keys, retry, invalidation, eviction, active observer settle이 검증됐습니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| component별 effect/local copy | 중복 fetch·stale identity·일관되지 않은 mutation | QueryClient/key/options + screen migration | `1ebdce4cdf0a` | component별 effect/local copy → 중복 fetch·stale identity·일관되지 않은 mutation → QueryClient/key/options + screen migration → `1ebdce4cdf0a` |
| 401도 일반 failure처럼 retry | 불필요한 auth traffic과 stale private data | retry predicate + `expireSession` | `1ebdce4cdf0a` | 401도 일반 failure처럼 retry → 불필요한 auth traffic과 stale private data → retry predicate + `expireSession` → `1ebdce4cdf0a` |
| fetching query를 동기 remove | current callback/observer가 불안정 상태 가능 | `d05a962d8829` deferred removal | active unauthorized observer test | fetching query를 동기 remove → current callback/observer가 불안정 상태 가능 → `d05a962d8829` deferred removal → active unauthorized observer test |
| WS/HTTP가 별도 lobby state | message/presence 불일치 | `931800f796e1` same cache update/invalidation | query cache tests는 key 범위, WS integration은 code inspection | WS/HTTP가 별도 lobby state → message/presence 불일치 → `931800f796e1` same cache update/invalidation → query cache tests는 key 범위, WS integration은 code inspection |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| QueryClient | 없음/각 component | `QueryProvider`의 단일 instance | application mount 수명 |
| server resource data | route local state | query key별 cache entry | stale/cache lifecycle |
| request cancellation | 대부분 없음 | query context `AbortSignal` | observer/query lifetime |
| session transition | 각 component 독립 처리 | `expireSession` + provider event listener | browser session lifetime |
| mutation refresh | local array patch/reload | exact invalidation policy | mutation completion 후 |
| lobby push | 별도 local chat state | `queryKeys.lobby()` cache update | socket event lifetime |

## 9. Thread 최종 상태

마지막 SHA에서 browser server state는 root의 단일 QueryClient가 소유합니다. route handle과 resource scope가 stable key에 포함되고 query function은 AbortSignal을 전달합니다. login, mutation, WebSocket push, session expiry는 정의된 exact key를 set/invalidate/remove하며 401은 retry하지 않습니다. screen에는 selection, form input, notice 같은 presentation state만 남습니다.

### 최종 실행 흐름

1. root `QueryProvider`가 QueryClient를 한 번 생성하고 session-expired listener를 설치합니다.
2. screen이 endpoint별 query option을 사용하면 cache hit 또는 signal-aware fetch가 실행됩니다.
3. HTTP success가 key별 cache entry를 채우고 observer가 loading/error/empty/success UI를 갱신합니다.
4. mutation success는 정의된 exact keys만 invalidation해 필요한 query만 refetch합니다.
5. lobby WebSocket event는 같은 lobby cache를 dedupe/update하거나 exact invalidation합니다.
6. 401 event는 private keys를 제거하고 current user cache를 `null`로 전환하며 public cache는 보존합니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 06-react-query-cache-ownership-and-invalidation.md =====

===== BEGIN FILE: 07-authoritative-snapshot-rendering-and-input.md =====
# Authoritative snapshot rendering과 입력

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

sample canvas와 browser key event에서 출발해 server snapshot이 score/player/phase의 유일한 source가 되고, room-scoped monotonic input command와 bounded interpolation으로 화면을 projection하는 구조로 발전하는 과정을 복원합니다.

### 직접 연결되는 불변식

- browser는 score, phase, winner, room state를 계산하지 않고 accepted server snapshot을 렌더링합니다.
- snapshot이 없을 때 실제 경기처럼 보이는 sample score/opponent를 만들지 않습니다.
- input은 room identity, protocol version, monotonic `inputSeq`, direction intent만 전달합니다.
- 같거나 더 오래된 snapshot은 거부되고 interpolation은 accepted snapshot state 복사본만 사용합니다.
- input loop와 render animation은 room/phase 변경 및 unmount에서 정리됩니다.

## 2. 핵심 질문

- 초기 canvas sample과 play route sample state가 어떤 false presentation을 만들었습니까?
- 첫 WebSocket 구현은 credential, parsing, stale snapshot, cleanup을 어떻게 처리했습니까?
- 50ms input sampling과 80ms interpolation의 owner·buffer·cleanup은 무엇입니까?
- versioned protocol 전환에서 `v`, `roomId`, `inputSeq`, snapshot `sequence`가 어떤 gate를 만듭니까?
- `PongCanvas`가 nested `snapshot.state`를 복제·보간하되 game rule을 계산하지 않는 근거는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3449f7988e1b` | `feat(web): 퐁 캔버스 미리보기 구현` | B | SIMULATION, REALTIME, WEB | shared dimensions와 `GameSnapshot`으로 field/paddle/ball/score를 그립니다. |
| 2 | `91962d36bd59` | `feat(play): 경기장 화면 구성` | B | PROTOCOL, REALTIME, WEB | Pong canvas 중심의 초기 play route를 추가합니다. |
| 3 | `737aa99cb4cb` | `feat(play): WebSocket 경기 연결 구현` | B | AUTH, PROTOCOL, SIMULATION | play screen을 realtime protocol client로 전환합니다. |
| 4 | `977ca863050f` | `feat(play): keyboard paddle 입력 연결` | B | PROTOCOL, SIMULATION, REALTIME | Arrow/W/S input을 room-scoped game command로 mapping합니다. |
| 5 | `afbd8847b1dd` | `feat(play): 경기 채팅 입력 연결` | B | REALTIME, WEB | 현재 room과 inline WebSocket에 match-chat form을 연결합니다. |
| 6 | `3cd56054bdab` | `fix(play): 패들 조작과 Canvas rendering 개선` | B | SIMULATION, REALTIME, WEB | persistent direction을 50ms마다 sampling하고 snapshot을 80ms 보간합니다. |
| 7 | `6a7aa285fe68` | `fix(play): 실제 경기 상태에 맞게 세션 표시` | A | SIMULATION, REALTIME, WEB | score/opponent/ready/chat/input/terminal cleanup을 latest server snapshot에서 파생합니다. |
| 8 | `e4e2dec55805` | `feat(play): 일시정지와 재개 UI 연결` | B | REALTIME, WEB | server snapshot phase에서 pause/resume availability를 파생하고 current room command를 전송합니다. |
| 9 | `8a8787d03a19` | `feat(play): versioned game input과 snapshot 소비` | A | PROTOCOL, SIMULATION, REALTIME | inputSeq와 shared event parser, monotonic snapshot acceptance를 browser에 적용합니다. |
| 10 | `868ced55a626` | `refactor(web): PongCanvas snapshot state 렌더링` | B | SIMULATION, REALTIME, WEB | canvas/interpolation buffer를 nested snapshot state에 맞춥니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(web): 퐁 캔버스 미리보기 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `3449f7988e1b` |
| Importance | B |
| Tags | SIMULATION, REALTIME, WEB |
| Source에서 확정된 역할 | shared dimensions와 `GameSnapshot`으로 field/paddle/ball/score를 그립니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/components/PongCanvas.tsx`의 default `sampleSnapshot`, canvas ref/effect, DPR scaling을 확인합니다.
- field, center line, paddles, ball, scores가 snapshot/shared dimensions에서 읽히는지 확인합니다.
- canvas resize/animation cleanup과 snapshot prop mutation 여부를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 브라우저 UI에 game field를 시각화하는 component가 없었습니다. |
| 해결하려던 문제 | shared game dimensions와 snapshot shape를 이용해 server game을 표시할 reusable canvas가 필요했습니다. |
| 핵심 결정 | `PongCanvas`를 추가하고 prop이 없으면 sample snapshot을 사용해 2D context에 field와 score를 그렸습니다. |
| 입력 → 상태 전이 → 출력 | render/effect → DPR에 맞춰 canvas size 설정 → snapshot field 읽기 → draw operations 순서입니다. |
| ownership/lifetime/cleanup | component가 canvas DOM/context와 drawing effect를 소유하고 snapshot object 자체는 읽기만 합니다. |
| failure/rollback/retry | context를 얻지 못하면 그리지 않으며 sample default 때문에 실제 connection 없이도 경기 장면처럼 보입니다. |
| 보장하는 것 | shared dimensions와 snapshot 값을 browser canvas에 projection합니다. |
| 보장하지 않는 것 | authoritative source, no-snapshot empty state, interpolation은 보장하지 않습니다. |
| 후속 연결 | `91962d36bd59`가 전용 play route에서 sample canvas를 사용하고 `6a7aa285fe68`이 fabricated session을 제거합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show 3449f7988e1b^` 및 `git diff 3449f7988e1b^ 3449f7988e1b`로 비교합니다.
- Thread 내 다음 관련 SHA: `91962d36bd59` — `feat(play): 경기장 화면 구성`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `feat(play): 경기장 화면 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `91962d36bd59` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, WEB |
| Source에서 확정된 역할 | Pong canvas 중심의 초기 play route를 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/play/page.tsx`의 sample snapshot, score/opponent/status, ready/chat/control markup을 확인합니다.
- button/form이 실제 handler와 연결됐는지 확인합니다.
- server room이 없어도 playing-like state가 표시되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | canvas component는 있었지만 game-specific route와 control surface가 없었습니다. |
| 해결하려던 문제 | 경기장 layout, opponent/score/status, input controls의 UI 구조를 먼저 만들 필요가 있었습니다. |
| 핵심 결정 | sample snapshot과 local presentation을 사용한 `/play` route를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | route render → sample state derivation → canvas/score/control markup 순서이며 network input은 없습니다. |
| ownership/lifetime/cleanup | page가 sample match state와 presentation을 모두 소유합니다. |
| failure/rollback/retry | control은 inert하거나 local presentation에만 영향을 주며 실제 room/authority가 없습니다. |
| 보장하는 것 | 후속 realtime integration을 위한 play screen layout을 제공합니다. |
| 보장하지 않는 것 | 표시된 score/opponent/phase가 실제 server state라는 보장은 없습니다. |
| 후속 연결 | `737aa99cb4cb`이 first WebSocket connection을 연결합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `3449f7988e1b` — `feat(web): 퐁 캔버스 미리보기 구현`
- Thread 내 다음 관련 SHA: `737aa99cb4cb` — `feat(play): WebSocket 경기 연결 구현`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `feat(play): WebSocket 경기 연결 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `737aa99cb4cb` |
| Importance | B |
| Tags | AUTH, PROTOCOL, SIMULATION |
| Source에서 확정된 역할 | play screen을 realtime protocol client로 전환합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 token query WebSocket URL, `new WebSocket`, onopen/onmessage/onclose handler를 확인합니다.
- `JSON.parse(...) as ServerEvent` 단언과 event type 분기를 확인합니다.
- sample snapshot이 initial state로 남는지, stale sequence/handler cleanup이 있는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | play route는 sample state만 표시하고 server room/event를 소비하지 않았습니다. |
| 해결하려던 문제 | queue join/ready와 snapshot 수신을 실제 WebSocket으로 연결할 초기 path가 필요했습니다. |
| 핵심 결정 | browser token을 query에 넣어 socket을 열고 raw JSON을 `ServerEvent`로 단언해 matched/snapshot/finished를 local state에 반영했습니다. |
| 입력 → 상태 전이 → 출력 | token read → socket open → queue command → raw message parse/cast → event별 local state update 순서입니다. |
| ownership/lifetime/cleanup | page가 durable token, socket, handler, room/snapshot/status state를 모두 소유합니다. |
| failure/rollback/retry | runtime schema가 없고 stale snapshot gate와 robust replacement cleanup이 없으며 sample initial state가 유지됩니다. |
| 보장하는 것 | play screen이 server WebSocket event를 소비하고 ready command를 보낼 수 있습니다. |
| 보장하지 않는 것 | credential 안전성, malformed event 거부, monotonic ordering, authoritative no-sample state는 보장하지 않습니다. |
| 후속 연결 | `353ca9a17415`가 ticket/schema boundary를, `6a7aa285fe68`과 `8a8787d03a19`가 state/order를 교정합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `91962d36bd59` — `feat(play): 경기장 화면 구성`
- Thread 내 다음 관련 SHA: `977ca863050f` — `feat(play): keyboard paddle 입력 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `feat(play): keyboard paddle 입력 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `977ca863050f` |
| Importance | B |
| Tags | PROTOCOL, SIMULATION, REALTIME |
| Source에서 확정된 역할 | Arrow/W/S input을 room-scoped game command로 mapping합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 `keydown`/`keyup` listener와 command payload를 확인합니다.
- 어떤 keydown이 `-1`/`1`을 보내고 모든 keyup이 `0`을 보내는지 확인합니다.
- editing target, key repeat, socket readyState, preventDefault, listener cleanup을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | WebSocket 경기 연결은 있었지만 사용자가 paddle intent를 보낼 입력 path가 없었습니다. |
| 해결하려던 문제 | keyboard event를 current room command로 변환할 최소 구현이 필요했습니다. |
| 핵심 결정 | keydown에서 Arrow/W/S를 direction으로 변환해 `game.input`을 보내고 keyup에서 stop command를 보냈습니다. |
| 입력 → 상태 전이 → 출력 | window key event → room/sender check → serialized input command 순서입니다. |
| ownership/lifetime/cleanup | page가 key listener와 raw send를 소유합니다. |
| failure/rollback/retry | 모든 keyup이 stop을 보내고 editable target/key repeat/focus loss를 충분히 구분하지 않으며 socket 상태 guard가 제한적입니다. |
| 보장하는 것 | keyboard로 paddle direction intent를 server에 보냅니다. |
| 보장하지 않는 것 | 일정 cadence, mobile input, neutralization completeness, monotonic input sequence는 보장하지 않습니다. |
| 후속 연결 | `3cd56054bdab`이 persistent direction sampling으로 바꾸고 hook migration의 `1ae6fa7836d8`이 transition-based 입력으로 재설계합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `737aa99cb4cb` — `feat(play): WebSocket 경기 연결 구현`
- Thread 내 다음 관련 SHA: `afbd8847b1dd` — `feat(play): 경기 채팅 입력 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `feat(play): 경기 채팅 입력 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `afbd8847b1dd` |
| Importance | B |
| Tags | REALTIME, WEB |
| Source에서 확정된 역할 | 현재 room과 inline WebSocket에 match-chat form을 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/play/page.tsx`의 `chatInput`, `sendChat`, trim/room/socket guard를 확인합니다.
- 전송 payload의 `scope: "match"`, `roomId`, `body`와 당시 version field 부재를 확인합니다.
- 성공 확인 없이 input을 즉시 비우며 server error/echo가 local writer와 어떻게 연결되지 않는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | play 화면에는 chat input markup이 있었지만 controlled state나 WebSocket send 동작이 없었습니다. |
| 해결하려던 문제 | 현재 room에 속한 non-empty message intent를 realtime protocol로 보낼 최소 writer가 필요했습니다. |
| 핵심 결정 | `chatInput` state와 submit handler를 추가해 socket/room/body가 모두 있을 때 match-scoped command를 전송하고 input을 비웠습니다. |
| 입력 → 상태 전이 → 출력 | form submit → trim → socket/room/body guard → JSON command send → input clear 순서입니다. |
| ownership/lifetime/cleanup | PlayPage가 input state와 inline socket writer를 직접 소유합니다. message acknowledgement/rollback은 없습니다. |
| failure/rollback/retry | room이 없거나 input이 비면 보내지 않습니다. socket send failure, server reject, 다른 room event filtering은 처리하지 않습니다. |
| 보장하는 것 | match chat UI가 current room identifier를 포함한 realtime command를 전송합니다. |
| 보장하지 않는 것 | versioned schema, delivery confirmation, active-room inbound filtering, dedicated hook ownership은 보장하지 않습니다. |
| 후속 연결 | `8a8787d03a19`가 command를 versioned contract로 만들고 Thread 05가 writer를 hook으로 이전한 뒤 legacy 함수를 제거합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `977ca863050f` — `feat(play): keyboard paddle 입력 연결`
- Thread 내 다음 관련 SHA: `3cd56054bdab` — `fix(play): 패들 조작과 Canvas rendering 개선`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `fix(play): 패들 조작과 Canvas rendering 개선`

| 항목 | 값 |
| --- | --- |
| SHA | `3cd56054bdab` |
| Importance | B |
| Tags | SIMULATION, REALTIME, WEB |
| Source에서 확정된 역할 | persistent direction을 50ms마다 sampling하고 snapshot을 80ms 보간합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 direction ref와 `setInterval(50)` input loop를 확인합니다.
- `PongCanvas.tsx`의 snapshot buffer(최근 8개), requestAnimationFrame, 80ms render target을 확인합니다.
- snapshot deep copy, two-frame interpolation, extrapolation 부재, interval/RAF cleanup을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 첫 key handler는 browser key-repeat cadence와 모든 keyup 처리에 의존해 command 빈도와 방향 상태가 불안정했고 canvas는 snapshot을 즉시 점프해 렌더링했습니다. |
| 해결하려던 문제 | input cadence를 browser repeat와 분리하고 delayed snapshots 사이를 부드럽게 projection할 필요가 있었습니다. |
| 핵심 결정 | current direction을 ref에 저장해 room이 있을 때 50ms마다 보내고, canvas는 최근 snapshot 복사본을 버퍼링해 현재 시각보다 80ms 뒤처진 target을 두 frame 사이에서 보간했습니다. |
| 입력 → 상태 전이 → 출력 | key event → direction ref; interval → command; snapshot prop → copied buffer; RAF → target time frame pair → linear interpolation → draw 순서입니다. |
| ownership/lifetime/cleanup | page가 input interval/direction을, canvas가 RAF와 bounded snapshot buffer를 소유합니다. |
| failure/rollback/retry | unmount/room change에서 interval과 RAF를 정리합니다. frame pair가 없으면 가장 가까운 state를 사용하며 미래 extrapolation은 하지 않습니다. |
| 보장하는 것 | input cadence와 rendering cadence를 분리하고 snapshot object mutation 없이 부드럽게 표시합니다. |
| 보장하지 않는 것 | sample initial state, protocol version/sequence, background tab neutralization은 아직 보장하지 않습니다. |
| 후속 연결 | `6a7aa285fe68`이 actual session gating을, `8a8787d03a19`이 monotonic protocol을 적용합니다. |

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | keydown repeat와 즉시 snapshot draw가 충분하다고 가정했습니다. |
| 실제 실패/위험 | OS/browser repeat 차이로 input cadence가 흔들리고 network snapshot 간 화면이 점프합니다. |
| 근본 원인 | physical input event와 transport cadence, network update와 render cadence를 직접 결합했습니다. |
| 수정된 불변식 | direction은 별도 state로 sampling하고 renderer는 bounded delayed snapshot buffer를 projection합니다. |
| 회귀 근거 | 이 SHA에는 전용 test가 없으며 후속 client/input tests가 command sequence를 검증합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `afbd8847b1dd` — `feat(play): 경기 채팅 입력 연결`
- Thread 내 다음 관련 SHA: `6a7aa285fe68` — `fix(play): 실제 경기 상태에 맞게 세션 표시`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.7. `fix(play): 실제 경기 상태에 맞게 세션 표시`

| 항목 | 값 |
| --- | --- |
| SHA | `6a7aa285fe68` |
| Importance | A |
| Tags | SIMULATION, REALTIME, WEB |
| Source에서 확정된 역할 | score/opponent/ready/chat/input/terminal cleanup을 latest server snapshot에서 파생합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 snapshot initial state가 sample에서 `null`로 바뀌는지 확인합니다.
- ready/chat/input availability가 room과 snapshot phase를 어떤 조건으로 요구하는지 확인합니다.
- connect/finish/close 시 snapshot, room, direction, timers/handlers를 정리하는 순서를 확인합니다.
- `PongCanvas`가 null snapshot에서 neutral empty frame을 그리는지 확인합니다.
- 같은 commit의 server 변경은 이 Thread 설명에 역투영하지 않고 browser diff만 분리합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 실제 room이 없거나 request가 실패해도 sample score/opponent/field가 표시됐고 input loop와 controls가 real phase와 충분히 묶이지 않았습니다. |
| 해결하려던 문제 | browser가 server-authoritative game을 표시한다면 snapshot 부재와 lifecycle phase를 명시적으로 반영해야 했습니다. |
| 핵심 결정 | snapshot을 `null`에서 시작하고 room/phase에 따라 ready/chat/input을 제한하며 connection start/finish/close에서 stale game state와 direction을 정리했습니다. |
| 입력 → 상태 전이 → 출력 | socket event → latest server snapshot/phase → derived score/opponent/control availability; no snapshot → neutral canvas/“경기 전” presentation 순서입니다. |
| ownership/lifetime/cleanup | server snapshot이 display source를 소유하고 page는 accepted latest value와 transport resource를 관리합니다. cleanup이 room/snapshot/direction/handler를 초기화합니다. |
| failure/rollback/retry | snapshot이 없으면 fabricated match를 보이지 않고 playing phase에서만 input timer를 활성화합니다. terminal/close에서 stale room command를 막습니다. |
| 보장하는 것 | score, opponent, phase, action availability가 실제 received snapshot에서만 파생됩니다. |
| 보장하지 않는 것 | raw message schema와 snapshot sequence ordering은 아직 보장하지 않습니다. |
| 후속 연결 | `8a8787d03a19`이 version/parser/sequence gate를 적용하고 `868ced55a626`이 canvas를 nested state contract에 맞춥니다. |

#### A-level 불변식 종합

- **핵심 책임:** score/opponent/ready/chat/input/terminal cleanup을 latest server snapshot에서 파생합니다.
- **실패 영향:** browser가 server-authoritative game을 표시한다면 snapshot 부재와 lifecycle phase를 명시적으로 반영해야 했습니다.
- **결과 불변식:** score, opponent, phase, action availability가 실제 received snapshot에서만 파생됩니다.
- **남은 제한:** raw message schema와 snapshot sequence ordering은 아직 보장하지 않습니다.

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | sample match state를 초기값으로 두고 server snapshot이 오면 교체해도 된다고 봤습니다. |
| 실제 실패/위험 | connection 전/실패/종료 상태가 실제 경기처럼 표시되고 stale input/room이 남습니다. |
| 근본 원인 | preview fixture와 authoritative session state를 같은 변수에 사용했습니다. |
| 수정된 불변식 | snapshot 부재는 경기 부재이며 score/control은 latest server phase에서만 파생합니다. |
| 회귀 근거 | 후속 connection reducer/client tests와 browser E2E가 lifecycle을 간접 검증합니다. |

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `6a7aa285fe68` |
| 파일 | `apps/web/src/app/play/page.tsx` |
| 함수/위치 | `session state derivation` |
| 근거 요약 | snapshot은 `null`에서 시작하고 room과 `snapshot.state.phase === "playing"`일 때만 input을 활성화하며 finish/close에서 room과 direction을 정리합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `3cd56054bdab` — `fix(play): 패들 조작과 Canvas rendering 개선`
- Thread 내 다음 관련 SHA: `e4e2dec55805` — `feat(play): 일시정지와 재개 UI 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.8. `feat(play): 일시정지와 재개 UI 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `e4e2dec55805` |
| Importance | B |
| Tags | REALTIME, WEB |
| Source에서 확정된 역할 | server snapshot phase에서 pause/resume availability를 파생하고 current room command를 전송합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 `canPause`, `canResume`, button label/handler를 확인합니다.
- snapshot phase가 `paused`/`playing`일 때만 해당 command를 보내는지 확인합니다.
- browser가 자체적으로 phase를 변경하지 않고 server snapshot을 기다리는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | server protocol과 hub가 pause/resume을 지원해도 play UI에는 해당 user intent와 phase 표시가 없었습니다. |
| 해결하려던 문제 | current room과 authoritative phase에 맞는 control만 활성화해야 했습니다. |
| 핵심 결정 | snapshot phase에서 pause/resume 가능 여부를 파생하고 current room ID를 포함한 command를 보냈습니다. |
| 입력 → 상태 전이 → 출력 | snapshot phase → enabled control; click → room-scoped pause/resume command → 후속 server snapshot → UI phase 갱신 순서입니다. |
| ownership/lifetime/cleanup | server가 phase authority를, page는 button intent를 소유합니다. |
| failure/rollback/retry | command 전송만으로 local phase를 낙관적으로 바꾸지 않으며 room/phase가 맞지 않으면 control을 비활성화합니다. |
| 보장하는 것 | pause/resume UI가 authoritative phase와 일치합니다. |
| 보장하지 않는 것 | server가 command를 수락하는 조건과 pause 중 input reset은 이 browser commit이 보장하지 않습니다. |
| 후속 연결 | 후속 protocol/version migration에서도 같은 phase-derived presentation이 유지됩니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `6a7aa285fe68` — `fix(play): 실제 경기 상태에 맞게 세션 표시`
- Thread 내 다음 관련 SHA: `8a8787d03a19` — `feat(play): versioned game input과 snapshot 소비`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.9. `feat(play): versioned game input과 snapshot 소비`

| 항목 | 값 |
| --- | --- |
| SHA | `8a8787d03a19` |
| Importance | A |
| Tags | PROTOCOL, SIMULATION, REALTIME |
| Source에서 확정된 역할 | inputSeq와 shared event parser, monotonic snapshot acceptance를 browser에 적용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `play/page.tsx`의 모든 client command에 `v: 1`이 포함되는지 확인합니다.
- `game.input`의 `inputSeq` 증가와 reconnect/new room에서 reset되는 위치를 확인합니다.
- raw message가 `parseServerEvent`를 통과하고 `snapshot.sequence <= lastSequence`를 거부하는지 확인합니다.
- flat snapshot field 참조가 `snapshot.state.*`로 이동하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | browser는 unversioned command와 raw JSON cast를 사용했고 delayed/duplicate snapshot이 current display를 뒤로 돌릴 수 있었습니다. |
| 해결하려던 문제 | wire contract version과 client input/snapshot ordering을 runtime에 강제해야 했습니다. |
| 핵심 결정 | 모든 command에 `v: 1`을 넣고 input sequence를 단조 증가시키며 shared parser로 event를 검증하고 accepted snapshot sequence gate를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | DOM/user intent → versioned command + roomId + inputSeq; raw event → shared parse → sequence gate → nested state adoption 순서입니다. |
| ownership/lifetime/cleanup | browser connection state가 last input/snapshot sequence를 소유하고 shared schema가 wire trust boundary를 소유합니다. |
| failure/rollback/retry | malformed event는 state로 들어오지 않고 같은/오래된 snapshot은 무시합니다. new connection/room에서 relevant sequence를 reset합니다. |
| 보장하는 것 | wire message와 rendered snapshot이 versioned runtime contract 및 monotonic ordering을 따릅니다. |
| 보장하지 않는 것 | network loss 자체를 복구하거나 server simulation correctness를 증명하지 않습니다. |
| 후속 연결 | `868ced55a626`이 reusable canvas도 nested state contract로 정렬하고 이후 reducer/client가 ordering owner를 분리합니다. |

#### A-level 불변식 종합

- **핵심 책임:** inputSeq와 shared event parser, monotonic snapshot acceptance를 browser에 적용합니다.
- **실패 영향:** wire contract version과 client input/snapshot ordering을 runtime에 강제해야 했습니다.
- **결과 불변식:** wire message와 rendered snapshot이 versioned runtime contract 및 monotonic ordering을 따릅니다.
- **남은 제한:** network loss 자체를 복구하거나 server simulation correctness를 증명하지 않습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `8a8787d03a19` |
| 파일 | `apps/web/src/app/play/page.tsx` |
| 함수/위치 | `message/input handlers` |
| 근거 요약 | 모든 command는 `v: 1`과 room ID를 포함하고 input은 증가하는 `inputSeq`를 사용합니다. parsed snapshot은 이전 sequence 이하이면 폐기합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `e4e2dec55805` — `feat(play): 일시정지와 재개 UI 연결`
- Thread 내 다음 관련 SHA: `868ced55a626` — `refactor(web): PongCanvas snapshot state 렌더링`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.10. `refactor(web): PongCanvas snapshot state 렌더링`

| 항목 | 값 |
| --- | --- |
| SHA | `868ced55a626` |
| Importance | B |
| Tags | SIMULATION, REALTIME, WEB |
| Source에서 확정된 역할 | canvas/interpolation buffer를 nested snapshot state에 맞춥니다. |

#### 해당 SHA에서 확인할 실제 코드

- `PongCanvas.tsx`의 draw, clone, interpolation 함수가 `snapshot.state`의 paddle/ball/score를 읽는지 확인합니다.
- empty snapshot이 valid sequence/serverTime/state shape를 갖는지 확인합니다.
- buffer copy가 source snapshot을 mutate하지 않는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | play page는 nested versioned snapshot을 소비했지만 reusable canvas의 helper 일부는 이전 flat field shape에 묶여 있었습니다. |
| 해결하려던 문제 | renderer와 interpolation buffer도 authoritative shared shape를 그대로 따라야 했습니다. |
| 핵심 결정 | draw/clone/interpolate/empty snapshot을 nested `state` 구조에 맞춰 정리했습니다. |
| 입력 → 상태 전이 → 출력 | accepted snapshot prop → deep clone buffer → delayed interpolation of state fields → draw nested paddles/ball/scores 순서입니다. |
| ownership/lifetime/cleanup | canvas가 presentation buffer/RAF를 소유하고 source snapshot authority는 caller/server에 남습니다. |
| failure/rollback/retry | source object를 직접 변경하지 않고 null 상태에는 neutral empty snapshot을 사용합니다. |
| 보장하는 것 | reusable renderer가 versioned nested snapshot contract와 일치합니다. |
| 보장하지 않는 것 | renderer는 collision, score, phase transition을 계산하지 않습니다. |
| 후속 연결 | 후속 connection refactor가 snapshot acceptance를 reducer에 옮겨도 canvas는 projection component로 유지됩니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `8a8787d03a19` — `feat(play): versioned game input과 snapshot 소비`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| preview renderer | `3449f7988e1b` → `91962d36bd59` | canvas/play screen은 생겼지만 sample match를 실제처럼 표시했습니다. |
| 첫 realtime/input | `737aa99cb4cb` → `977ca863050f` | socket과 keyboard command가 연결됐지만 token/cast/order/cleanup이 불완전했습니다. |
| match chat writer | `afbd8847b1dd` | 현재 room과 inline socket을 사용하는 첫 chat command path가 생겼습니다. |
| cadence 분리 | `3cd56054bdab` | 50ms input sampling과 80ms delayed interpolation이 도입됐습니다. |
| authoritative presentation 교정 | `6a7aa285fe68` | no-snapshot는 no-game이며 control이 server phase를 따릅니다. |
| pause projection | `e4e2dec55805` | pause/resume intent가 current room과 snapshot phase에 묶였습니다. |
| version/order contract | `8a8787d03a19` → `868ced55a626` | versioned parser, monotonic sequence, nested renderer가 완성됐습니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| sample snapshot을 runtime state로 사용 | connection 전/실패도 실제 경기처럼 표시 | `6a7aa285fe68` | 후속 reducer/client lifecycle tests | sample snapshot을 runtime state로 사용 → connection 전/실패도 실제 경기처럼 표시 → `6a7aa285fe68` → 후속 reducer/client lifecycle tests |
| keydown repeat·즉시 draw | 불안정 input cadence·화면 점프 | `3cd56054bdab` | 전용 rendering test는 없음 | keydown repeat·즉시 draw → 불안정 input cadence·화면 점프 → `3cd56054bdab` → 전용 rendering test는 없음 |
| raw JSON cast·sequence gate 없음 | malformed/stale snapshot 수용 | `8a8787d03a19` | shared protocol tests와 후속 client/reducer tests | raw JSON cast·sequence gate 없음 → malformed/stale snapshot 수용 → `8a8787d03a19` → shared protocol tests와 후속 client/reducer tests |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| game authority | sample/browser state 혼합 | server snapshot | room/session 수명 |
| canvas resources | 단발 drawing | `PongCanvas` context/RAF/buffer | component mount 수명 |
| input cadence | browser key repeat | 50ms page interval, 이후 hook transition command로 이전 | active room/playing 수명 |
| snapshot ordering | 없음 | browser last sequence gate, 이후 reducer | connection/room 수명 |
| protocol trust | JSON cast | shared runtime parser | message 처리 순간 |
| pause state | 없음/local 가능 | server snapshot phase | room phase 수명 |
| match chat input | 정적 markup | PlayPage controlled input/inline socket, 이후 hook으로 이전 | active room/page 수명 |

## 9. Thread 최종 상태

마지막 SHA에서 canvas는 accepted nested server snapshot만 복제·보간·렌더링하고 game rule을 계산하지 않습니다. snapshot이 없으면 neutral pre-game surface를 보이며 score/opponent/control은 authoritative phase에서 파생됩니다. client input은 versioned room-scoped monotonic command입니다. 후속 hook migration에서는 입력 writer와 snapshot acceptance owner가 각각 client/reducer로 이동하지만 이 projection invariant는 유지됩니다.

### 최종 실행 흐름

1. server event가 shared runtime parser를 통과하고 stale/equal sequence gate를 통과합니다.
2. accepted snapshot의 nested `state`가 current game source가 됩니다.
3. page는 phase에서 score/opponent/ready/chat/pause/input availability를 파생합니다.
4. `PongCanvas`는 snapshot 복사본을 bounded buffer에 넣고 80ms delayed target의 두 frame을 보간합니다.
5. RAF가 interpolated paddle/ball/score를 draw하며 source snapshot을 mutate하지 않습니다.
6. keyboard intent는 active room/playing 조건에서 versioned `game.input`과 증가하는 `inputSeq`로 전송됩니다.
7. match chat intent도 current room ID와 non-empty body를 요구한 뒤 realtime command로 전송됩니다.
8. room/finish/close/unmount에서는 input timer, direction, socket handler, animation resource를 정리합니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 07-authoritative-snapshot-rendering-and-input.md =====

===== BEGIN FILE: 08-guest-browser-policy-and-transient-results.md =====
# Guest browser policy와 transient results

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

deployment mode를 browser capability policy로 변환하고 guest login, route/navigation restriction, lobby/play presentation, non-persisted result notice, active room 복귀, reconnect를 연결한 뒤 abuse·recovery·browser E2E로 검증하는 과정을 복원합니다.

### 직접 연결되는 불변식

- server `APP_MODE`와 browser `NEXT_PUBLIC_APP_MODE`는 같은 demo capability를 선택합니다.
- demo navigation과 middleware는 registered-only 화면을 각각 presentation과 direct URL 경계에서 차단합니다.
- guest UI는 durable 전적·rating·ranking·chat을 제공하는 것처럼 표시하지 않습니다.
- `persisted: false` 결과는 임시 결과로 명시하며 dashboard/history로 취급하지 않습니다.
- lobby socket에서 active room event를 회수하면 새 matchmaking intent를 만들지 않고 `/play`로 복귀합니다.
- 최종 browser E2E는 guest entry, restricted navigation, PvP, 6초 AI fallback, fresh-ticket reconnect를 분리 검증합니다.

### 범위 주의

이 Thread는 browser-owned guest capability와 그 통합 evidence를 다룹니다. `eaa4fdaba361`의 server-side guest result producer(`persisted: false`, `matchId: null`, 2분 retention)와 `2b274686e6d4`의 server runtime bound는 upstream 근거로만 참조하며 이 카테고리의 commit map에는 넣지 않습니다. `9f49db1d9f1d`와 `06d2eb7a93cc`는 mixed test commits이므로 browser policy/transport 관련 test만 학습 범위로 삼고 server-only assertion을 browser 보장으로 해석하지 않습니다.

### 교차 Thread 주의

`4f5199097284`는 Thread 04와 의도적으로 중복됩니다. 여기서는 `HomePage`와 `demoPolicy`의 guest route/result handling만 고정하고 transport/reducer/hook 구현은 Thread 04에서만 해설합니다.

## 2. 핵심 질문

- runtime mode와 session secret/proxy trust가 어떤 config에서 fail-closed로 선택됩니까?
- `createNavigation`, restricted paths, presentation flags가 shell/page/middleware에서 어떻게 공유됩니까?
- guest login은 사용자 입력 없이 server-named identity를 cookie session으로 어떻게 수신합니까?
- lobby가 `game.finished persisted:false`, `queue.matched`, `game.snapshot`을 각각 어떻게 처리합니까?
- mixed guest tests에서 browser policy와 server resource abuse evidence를 어떻게 구분합니까?
- Playwright가 실제 WebSocket frame timing과 forced reconnect를 어떤 technique로 관찰합니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, 함수, class, state, caller/callee, failure branch, cleanup을 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.
- 중요도는 A-level의 위험·소유권·회귀 근거를 B-level보다 깊게 기록합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f801ccd09cf0` | `feat(guest): guest runtime 환경 경계 구성` | A | AUTH, WEB | application mode와 proxy trust를 runtime/public browser configuration에 명시합니다. |
| 2 | `e39316254e44` | `feat(web): 비회원 체험 정책 경계 추가` | B | WEB | guest demo surface용 centralized browser policy module을 도입합니다. |
| 3 | `a9fc8a8328b2` | `feat(web): guest login API와 middleware 연결` | B | AUTH, PROTOCOL, TOURNAMENT | guest session adapter와 registered-only screen 차단 middleware를 연결합니다. |
| 4 | `7fdef5d224c4` | `feat(web): LoginPanel guest 진입 연결` | B | WEB | demo mode에서 login panel을 server-managed guest entry에 연결합니다. |
| 5 | `584d17f3aad1` | `feat(web): guest lobby presentation 적용` | B | REALTIME, WEB | guest lobby가 durable progress와 unsupported interaction을 광고하지 않게 합니다. |
| 6 | `658fafd43f88` | `feat(web): demo navigation 정책 연결` | B | WEB | `AppShell`이 capability에 맞는 navigation만 구성합니다. |
| 7 | `fe0f3e0ad0ad` | `feat(web): guest play presentation 적용` | B | WEB | demo mode에서 지원하지 않는 match chat을 숨깁니다. |
| 8 | `618330916629` | `test(web): 비회원 체험 진입 흐름 검증` | B | AUTH, WEB, TEST | guest entry와 browser API boundary를 검증합니다. |
| 9 | `9f49db1d9f1d` | `test(guest): 체험 기능 오용 방지 검증` | A | AUTH, WEB, RISK | guest-mode abuse와 browser capability isolation의 negative boundary를 확장 검증합니다. |
| 10 | `4f5199097284` | `fix(web): 중단된 game reconnect 복구` | A | AUTH, REALTIME, WEB | guest lobby에서 transient result를 명시하고 active room event를 회수하면 game screen으로 복귀합니다. |
| 11 | `06d2eb7a93cc` | `test(guest): 체험 환경의 복구 경계 검증` | A | AUTH, SIMULATION, REALTIME | fresh-ticket reconnect, duplicate match 차단, transient result presentation을 확장 검증합니다. |
| 12 | `1abda1299ad8` | `test(e2e): 비회원 체험 브라우저 흐름 검증` | A | AUTH, REALTIME, WEB | demo-mode guest entry, restricted navigation, two-browser PvP, 6초 AI fallback, ticket-based reconnect를 Playwright로 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(guest): guest runtime 환경 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `f801ccd09cf0` |
| Importance | A |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | application mode와 proxy trust를 runtime/public browser configuration에 명시합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.env.example`, `apps/api/src/env.ts`, browser public env 사용 위치를 확인합니다.
- `APP_MODE`의 development/test/production/demo 값과 `NEXT_PUBLIC_APP_MODE` 연결을 확인합니다.
- demo/production에서 `SESSION_SECRET` 32바이트 이상을 요구하고 `TRUST_PROXY`가 explicit opt-in인지 확인합니다.
- browser build-time public mode와 server runtime mode가 서로 독립적으로 잘못 설정될 가능성도 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | guest 기능은 예정돼 있었지만 deployment가 demo인지 production인지, browser가 어떤 capability를 표시해야 하는지 명시적 runtime contract가 없었습니다. |
| 해결하려던 문제 | 공개 guest traffic을 허용하는 mode에서는 약한/기본 secret과 신뢰되지 않은 forwarded address가 인증·rate-limit 경계를 무너뜨릴 수 있었습니다. |
| 핵심 결정 | `APP_MODE`, `TRUST_PROXY`, `NEXT_PUBLIC_APP_MODE`를 환경 계약에 추가하고 demo/production에서 강한 session secret을 필수로 만들었습니다. |
| 입력 → 상태 전이 → 출력 | process env → `readEnv/readAppMode` validation → API construction; browser build env → policy selection 순서입니다. |
| ownership/lifetime/cleanup | server env parser가 secret/proxy/mode를, browser build가 public mode 값을 소유합니다. 설정은 process/build 수명 동안 고정됩니다. |
| failure/rollback/retry | 알 수 없는 mode/약한 secret은 startup에서 실패하고 proxy parsing은 명시한 경우에만 활성화됩니다. |
| 보장하는 것 | guest capability를 암묵적 NODE_ENV가 아니라 명시적 mode와 fail-closed secret 요구에 연결합니다. |
| 보장하지 않는 것 | server와 browser env가 deployment에서 반드시 같은 값이라는 자동 교차 검증은 이 commit만으로 보장하지 않습니다. |
| 후속 연결 | `e39316254e44`가 browser mode를 구체적인 navigation/presentation policy로 변환하고 `9f49db1d9f1d`/`06d2eb7a93cc`가 env negative case를 검증합니다. |

#### A-level 불변식 종합

- **핵심 책임:** application mode와 proxy trust를 runtime/public browser configuration에 명시합니다.
- **실패 영향:** 공개 guest traffic을 허용하는 mode에서는 약한/기본 secret과 신뢰되지 않은 forwarded address가 인증·rate-limit 경계를 무너뜨릴 수 있었습니다.
- **결과 불변식:** guest capability를 암묵적 NODE_ENV가 아니라 명시적 mode와 fail-closed secret 요구에 연결합니다.
- **남은 제한:** server와 browser env가 deployment에서 반드시 같은 값이라는 자동 교차 검증은 이 commit만으로 보장하지 않습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `f801ccd09cf0` |
| 파일 | `apps/api/src/env.ts` |
| 함수/위치 | `readEnv / readAppMode` |
| 근거 요약 | demo와 production은 명시적 강한 `SESSION_SECRET`을 요구하고 proxy address 해석은 `TRUST_PROXY`가 켜진 경우에만 허용합니다. |

#### 비교 기준

- 이 commit의 parent를 `git show f801ccd09cf0^` 및 `git diff f801ccd09cf0^ f801ccd09cf0`로 비교합니다.
- Thread 내 다음 관련 SHA: `e39316254e44` — `feat(web): 비회원 체험 정책 경계 추가`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.2. `feat(web): 비회원 체험 정책 경계 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e39316254e44` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | guest demo surface용 centralized browser policy module을 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/demoPolicy.ts`의 registered navigation, demo navigation, restricted path, presentation flags를 확인합니다.
- `createNavigation(demoMode, profileHref)`, `isDemoRestrictedPath`, `isDemoMode`의 pure/input-dependent 범위를 확인합니다.
- demo mode에서 lobby/play만 남고 persisted progress, leaderboard, lobby/match chat flags가 false인지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 각 component가 demo 여부를 개별 조건문으로 해석하면 navigation과 page capability가 서로 달라질 수 있었습니다. |
| 해결하려던 문제 | guest에게 허용/금지할 browser surface를 한 module에서 정의할 필요가 있었습니다. |
| 핵심 결정 | registered navigation 목록과 demo presentation flags/restricted paths를 `demoPolicy.ts`에 모았습니다. |
| 입력 → 상태 전이 → 출력 | public mode + profile href → navigation items; pathname → restricted 여부; page → presentation flags 소비 순서입니다. |
| ownership/lifetime/cleanup | policy module이 capability description을 소유하고 shell/page/middleware는 이를 소비합니다. |
| failure/rollback/retry | pure function은 정책을 계산하지만 아직 실제 component나 middleware에 연결되지 않았습니다. |
| 보장하는 것 | guest browser surface의 single policy vocabulary를 제공합니다. |
| 보장하지 않는 것 | server route authorization이나 guest session 생성은 보장하지 않습니다. |
| 후속 연결 | `a9fc8a8328b2`가 middleware/API adapter를, 이후 UI commits가 policy를 실제 rendering에 연결합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `f801ccd09cf0` — `feat(guest): guest runtime 환경 경계 구성`
- Thread 내 다음 관련 SHA: `a9fc8a8328b2` — `feat(web): guest login API와 middleware 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.3. `feat(web): guest login API와 middleware 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `a9fc8a8328b2` |
| Importance | B |
| Tags | AUTH, PROTOCOL, TOURNAMENT |
| Source에서 확정된 역할 | guest session adapter와 registered-only screen 차단 middleware를 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/api.ts`의 `guestLogin` endpoint, HTTP method/body, shared response schema를 확인합니다.
- `apps/web/src/middleware.ts`가 demo mode에서 `isDemoRestrictedPath`를 사용해 어떤 status/response를 반환하는지 확인합니다.
- navigation 숨김과 direct URL 차단이 별도 계층임을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | browser policy는 있었지만 guest session을 만드는 adapter와 직접 URL로 registered-only page에 접근하는 server-side route guard가 없었습니다. |
| 해결하려던 문제 | UI 숨김만으로는 주소 입력을 막지 못하고 guest login success payload도 runtime 검증이 필요했습니다. |
| 핵심 결정 | `POST /auth/guest`를 shared schema로 parse하는 helper와 demo restricted path를 404로 처리하는 Next middleware를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | login caller → cookie-based guest endpoint → schema-validated user; request pathname → middleware policy → restricted면 404 순서입니다. |
| ownership/lifetime/cleanup | API adapter가 response trust를, middleware가 browser route delivery를 소유합니다. |
| failure/rollback/retry | malformed guest response는 성공하지 않고 restricted direct request는 page component까지 도달하지 않습니다. |
| 보장하는 것 | guest session entry adapter와 direct route presentation guard를 제공합니다. |
| 보장하지 않는 것 | API server의 registered-only authorization과 session signing은 browser commit이 증명하지 않습니다. |
| 후속 연결 | `7fdef5d224c4`가 login panel에 helper를 연결하고 `618330916629`가 body/credentials/schema를 검증합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `e39316254e44` — `feat(web): 비회원 체험 정책 경계 추가`
- Thread 내 다음 관련 SHA: `7fdef5d224c4` — `feat(web): LoginPanel guest 진입 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.4. `feat(web): LoginPanel guest 진입 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `7fdef5d224c4` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | demo mode에서 login panel을 server-managed guest entry에 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `LoginPanel.tsx`의 `demoMode`, mutationFn branch, guest response `.user` 사용을 확인합니다.
- demo mode에서 handle/displayName input이 렌더링되지 않는지 확인합니다.
- success 시 `queryKeys.me()` set과 login invalidation, pending/error button label을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | login panel은 항상 사용자가 handle/display name을 입력하고 development login을 호출했습니다. |
| 해결하려던 문제 | guest identity는 사용자가 이름을 선택하는 registered/development account가 아니라 server가 발급하는 transient session이어야 했습니다. |
| 핵심 결정 | demo mode에서는 inputs를 숨기고 `guestLogin()` 결과의 server-named user를 current session cache에 넣도록 mutation을 분기했습니다. |
| 입력 → 상태 전이 → 출력 | button click → guest POST(no body) → response user → me cache set → login-related invalidation → lobby render 순서입니다. |
| ownership/lifetime/cleanup | server가 guest identity naming/session을, QueryClient가 current user cache를, panel이 pending/error presentation을 소유합니다. |
| failure/rollback/retry | request failure는 entry를 성공 처리하지 않고 error를 표시하며 duplicate pending click을 막습니다. |
| 보장하는 것 | profile input 없이 server-managed guest session으로 진입합니다. |
| 보장하지 않는 것 | guest capability가 UI 전체에서 숨겨졌다는 보장은 아직 없습니다. |
| 후속 연결 | `584d17f3aad1`, `658fafd43f88`, `fe0f3e0ad0ad`가 lobby/nav/play presentation을 순서대로 적용합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `a9fc8a8328b2` — `feat(web): guest login API와 middleware 연결`
- Thread 내 다음 관련 SHA: `584d17f3aad1` — `feat(web): guest lobby presentation 적용`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.5. `feat(web): guest lobby presentation 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `584d17f3aad1` |
| Importance | B |
| Tags | REALTIME, WEB |
| Source에서 확정된 역할 | guest lobby가 durable progress와 unsupported interaction을 광고하지 않게 합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`가 `isDemoMode`와 `demoLobbyPresentation`을 소비하는 위치를 확인합니다.
- “전적 저장”→“결과 미저장”, description, leaderboard link, wins/rating cards, lobby chat 조건을 확인합니다.
- online/wait metrics와 play entry는 guest에게 계속 노출되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | guest가 로그인해도 registered lobby copy와 wins/rating/ranking/chat이 그대로 보여 durable account capability를 암시했습니다. |
| 해결하려던 문제 | transient guest가 실제로 사용할 수 없는 progress/social 기능을 화면에서 제거하고 결과 미저장을 명시해야 했습니다. |
| 핵심 결정 | central presentation flags로 copy와 section visibility를 분기해 persisted stats, leaderboard link, lobby chat을 숨겼습니다. |
| 입력 → 상태 전이 → 출력 | demo mode → policy flags → lobby hero/stat/link/chat conditional render 순서입니다. |
| ownership/lifetime/cleanup | policy가 capability를, HomePage가 conditional presentation과 live lobby metrics를 소유합니다. |
| failure/rollback/retry | unsupported section은 DOM에 렌더링하지 않으며 online/wait와 play entry는 유지합니다. |
| 보장하는 것 | guest lobby가 durable progress를 약속하지 않습니다. |
| 보장하지 않는 것 | direct registered page 접근과 play match chat은 이 commit만으로 막지 않습니다. |
| 후속 연결 | `658fafd43f88`과 `fe0f3e0ad0ad`가 남은 navigation/play surface를 적용합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `7fdef5d224c4` — `feat(web): LoginPanel guest 진입 연결`
- Thread 내 다음 관련 SHA: `658fafd43f88` — `feat(web): demo navigation 정책 연결`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.6. `feat(web): demo navigation 정책 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `658fafd43f88` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | `AppShell`이 capability에 맞는 navigation만 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppShell.tsx`가 inline nav array 대신 `createNavigation(isDemoMode(), profileHref)`를 호출하는지 확인합니다.
- `NavigationId`→Lucide icon mapping이 policy data와 presentation dependency를 분리하는지 확인합니다.
- demo mode에서 lobby/play 두 link만 DOM에 있는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | central policy가 있어도 shell은 여전히 dashboard/leaderboard/tournament/profile/admin 전체 nav를 자체 생성했습니다. |
| 해결하려던 문제 | browser capability와 navigation 목록이 같은 source를 사용해야 했습니다. |
| 핵심 결정 | shell이 `createNavigation` 결과만 렌더링하고 icon component는 ID별 별도 map으로 연결했습니다. |
| 입력 → 상태 전이 → 출력 | mode/profile href → policy nav items → icon lookup/active class → link render 순서입니다. |
| ownership/lifetime/cleanup | policy가 item selection을, shell이 icon/active/layout을 소유합니다. |
| failure/rollback/retry | demo policy에 없는 item은 navigation DOM에 생성되지 않습니다. |
| 보장하는 것 | 주소 직접 입력 차단은 middleware에 의존합니다. |
| 보장하지 않는 것 | guest shell이 lobby/play capability만 광고합니다. |
| 후속 연결 | `9f49db1d9f1d`가 demo/registered item ID 배열을 pure test로 고정합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `584d17f3aad1` — `feat(web): guest lobby presentation 적용`
- Thread 내 다음 관련 SHA: `fe0f3e0ad0ad` — `feat(web): guest play presentation 적용`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.7. `feat(web): guest play presentation 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `fe0f3e0ad0ad` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | demo mode에서 지원하지 않는 match chat을 숨깁니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/play/page.tsx`의 `demoMode`와 `showMatchChat` 조건을 확인합니다.
- match chat aside/form 전체가 조건부 렌더링되는지 확인합니다.
- game controls/opponent/canvas는 guest에게 유지되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | lobby와 navigation은 제한됐지만 play page의 match chat은 guest에게 계속 표시됐습니다. |
| 해결하려던 문제 | guest capability matrix에서 금지된 social interaction을 실제 match surface에도 적용해야 했습니다. |
| 핵심 결정 | play page가 central policy의 `showMatchChat`을 소비해 chat section 전체를 조건부 렌더링했습니다. |
| 입력 → 상태 전이 → 출력 | demo mode/policy flag → match chat DOM 포함 여부; 나머지 game UI는 동일하게 유지됩니다. |
| ownership/lifetime/cleanup | policy가 capability를, play page가 presentation을 소유합니다. |
| failure/rollback/retry | guest mode에서는 input/form/메시지 목록이 생성되지 않습니다. |
| 보장하는 것 | guest match chat을 browser UI에서 제공하지 않습니다. |
| 보장하지 않는 것 | server가 chat command를 별도로 거부하는지는 browser presentation이 증명하지 않습니다. |
| 후속 연결 | `9f49db1d9f1d`가 presentation flags를 test합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `658fafd43f88` — `feat(web): demo navigation 정책 연결`
- Thread 내 다음 관련 SHA: `618330916629` — `test(web): 비회원 체험 진입 흐름 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.8. `test(web): 비회원 체험 진입 흐름 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `618330916629` |
| Importance | B |
| Tags | AUTH, WEB, TEST |
| Source에서 확정된 역할 | guest entry와 browser API boundary를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/api.test.ts`의 “starts a server-named guest session without profile input” case를 확인합니다.
- URL `/auth/guest`, method POST, `credentials: include`, body undefined, response schema result를 확인합니다.
- endpoint helper table에 `guestLogin`이 포함돼 malformed response parsing도 공유하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | guest helper와 LoginPanel 연결은 있었지만 no-input/no-body/cookie/schema contract가 자동화되지 않았습니다. |
| 해결하려던 문제 | guest entry가 development login input이나 bearer token을 다시 요구하지 않는지 deterministic하게 고정해야 했습니다. |
| 핵심 결정 | mocked fetch로 guest response를 반환하고 production `guestLogin()`의 URL/init/result를 직접 검사했습니다. |
| 입력 → 상태 전이 → 출력 | helper call → POST no body + cookie credentials → schema parse → response equality assertion 순서입니다. |
| ownership/lifetime/cleanup | test가 fetch mock을 소유하고 adapter의 실제 request/parse path를 실행합니다. |
| failure/rollback/retry | body와 Authorization이 없고 cookie가 포함되며 malformed success는 endpoint table에서 거부됩니다. |
| 보장하는 것 | browser guest adapter의 entry contract를 증명합니다. |
| 보장하지 않는 것 | LoginPanel DOM, middleware, 실제 Set-Cookie/session 생성은 증명하지 않습니다. |
| 후속 연결 | `1abda1299ad8`이 실제 browser entry와 navigation을 최종 E2E로 검증합니다. |

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | guest session은 profile input/body 없이 cookie-based POST와 runtime schema로 시작됩니다. |
| 재현한 실패/경계 | 잘못된 URL/method/body/credential 또는 malformed guest response입니다. |
| 테스트 기법 | mocked fetch와 endpoint table-driven parsing test입니다. |
| 증명하는 것 | browser API adapter의 deterministic request/response contract를 증명합니다. |
| 증명하지 않는 것 | 실제 server cookie 발급과 UI rendering은 증명하지 않습니다. |
| 검증 분류 | browser API boundary unit regression |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `fe0f3e0ad0ad` — `feat(web): guest play presentation 적용`
- Thread 내 다음 관련 SHA: `9f49db1d9f1d` — `test(guest): 체험 기능 오용 방지 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.9. `test(guest): 체험 기능 오용 방지 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `9f49db1d9f1d` |
| Importance | A |
| Tags | AUTH, WEB, RISK |
| Source에서 확정된 역할 | guest-mode abuse와 browser capability isolation의 negative boundary를 확장 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 이 Thread에서는 `apps/web/src/lib/demoPolicy.test.ts`와 `apps/api/src/env.test.ts`의 mode/secret/proxy case를 우선 조사합니다.
- `createNavigation(true/false)`, presentation flags, restricted path positive/negative case를 확인합니다.
- 같은 commit의 `GuestAccess`, `GameHub`, HTTP abuse tests는 server-side evidence이며 browser policy 보장과 구분해 기록합니다.
- strong secret과 explicit proxy trust가 false/true input에서 어떻게 test되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | happy-path guest entry만으로는 registered menu 노출, persisted capability copy, direct path 허용, 약한 secret, unbounded ticket/resource 같은 오용을 배제할 수 없었습니다. |
| 해결하려던 문제 | guest mode는 public traffic 경계이므로 positive UI보다 negative capability/abuse matrix가 더 중요했습니다. |
| 핵심 결정 | browser policy pure tests와 server env/resource isolation tests를 한 regression commit에 추가했습니다. |
| 입력 → 상태 전이 → 출력 | mode/path/policy fixture 또는 controlled server resource → production function/service → allowed/blocked/count/expiry assertion 순서입니다. |
| ownership/lifetime/cleanup | browser policy test는 pure values를, server tests는 fake timers와 GuestAccess/GameHub resources를 소유합니다. 두 evidence layer를 혼동하지 않아야 합니다. |
| failure/rollback/retry | demo nav는 lobby/play만 남고 persisted/ranking/chat flags는 false이며 restricted path를 true로 판정합니다. server tests는 별도로 secret/ticket/lease/result bounds를 검증합니다. |
| 보장하는 것 | guest capability가 registered surface로 새지 않는 high-risk negative evidence를 제공합니다. |
| 보장하지 않는 것 | browser test 부분은 실제 middleware response나 UI DOM을 실행하지 않고, server-only test를 browser enforcement 증거로 사용할 수 없습니다. |
| 후속 연결 | `4f5199097284`가 lobby recovery/transient presentation을 추가하고 `06d2eb7a93cc`가 복구 경계를 확장합니다. |

#### A-level 불변식 종합

- **핵심 책임:** guest-mode abuse와 browser capability isolation의 negative boundary를 확장 검증합니다.
- **실패 영향:** guest mode는 public traffic 경계이므로 positive UI보다 negative capability/abuse matrix가 더 중요했습니다.
- **결과 불변식:** guest capability가 registered surface로 새지 않는 high-risk negative evidence를 제공합니다.
- **남은 제한:** browser test 부분은 실제 middleware response나 UI DOM을 실행하지 않고, server-only test를 browser enforcement 증거로 사용할 수 없습니다.

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | demo mode는 제한된 browser capability만 노출하고 runtime 설정은 약한 trust configuration을 거부합니다. |
| 재현한 실패/경계 | registered nav/path 노출, persisted progress/chat copy, weak secret, proxy/ticket/resource abuse입니다. |
| 테스트 기법 | pure policy tests + server fake-timer/resource tests를 한 commit에서 분리 실행하도록 작성했습니다. |
| 증명하는 것 | policy matrix와 여러 high-risk negative branch를 증명합니다. |
| 증명하지 않는 것 | browser DOM/middleware E2E와 모든 server isolation을 하나의 end-to-end chain으로 증명하지 않습니다. |
| 검증 분류 | high-risk auth/capability negative regression |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `9f49db1d9f1d` |
| 파일 | `apps/web/src/lib/demoPolicy.test.ts` |
| 함수/위치 | `guest demo presentation policy` |
| 근거 요약 | demo navigation은 `lobby`, `play`만 반환하고 persisted progress, leaderboard, lobby/match chat flags는 모두 false이며 registered paths는 restricted로 판정합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `618330916629` — `test(web): 비회원 체험 진입 흐름 검증`
- Thread 내 다음 관련 SHA: `4f5199097284` — `fix(web): 중단된 game reconnect 복구`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.10. `fix(web): 중단된 game reconnect 복구`

| 항목 | 값 |
| --- | --- |
| SHA | `4f5199097284` |
| Importance | A |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | guest lobby에서 transient result를 명시하고 active room event를 회수하면 game screen으로 복귀합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 이 Thread에서는 `apps/web/src/app/page.tsx`와 `apps/web/src/lib/demoPolicy.ts`만 조사합니다. 같은 SHA의 client/reducer/hook reconnect는 Thread 04에서 다룹니다.
- `formatTransientResultNotice`가 `persisted: false`, score를 어떤 문구로 표시하는지 확인합니다.
- `shouldResumeGameFromLobby`가 `queue.matched`와 `game.snapshot`만 true인지 확인합니다.
- HomePage lobby socket handler가 resume event를 `/play` redirect보다 먼저 처리하고 transient finish는 notice로 표시하는지 확인합니다.
- upstream `eaa4fdaba361`의 guest result `persisted:false` contract를 later code로 역투영하지 말고 producer dependency로만 연결합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | guest가 game screen 밖에서 socket event를 회수할 때 active room을 놓치거나, 종료 결과를 일반 durable match처럼 해석할 수 있었습니다. transport close도 existing room을 재개하지 못했습니다. |
| 해결하려던 문제 | lobby는 active-room event를 새 matchmaking으로 처리하지 않고 game screen으로 복귀해야 하며 non-persisted result는 명시적으로 transient라고 알려야 했습니다. |
| 핵심 결정 | pure policy helpers를 추가하고 lobby socket handler가 queue/snapshot이면 `/play`로 이동하며 `game.finished`의 `persisted:false`면 score와 미저장 notice를 표시하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | lobby server event → runtime parse → demo active-room predicate면 redirect; 아니면 transient finish predicate면 formatted notice; 그 밖의 lobby event는 기존 cache update 순서입니다. |
| ownership/lifetime/cleanup | demo policy가 event classification/wording을, HomePage가 navigation/notice side effect를 소유합니다. transport retry timer는 Thread 04의 client가 소유합니다. |
| failure/rollback/retry | 정책에 포함되지 않은 event는 기존 lobby 처리로 계속 전달되며 redirect/notice 자체의 실패에 별도 rollback은 없습니다. |
| 보장하는 것 | active room event를 lobby chat/presence로 처리하지 않고 즉시 game route로 되돌립니다. transient result는 rating/history를 암시하지 않습니다. |
| 보장하지 않는 것 | guest 결과가 실제로 DB에 저장되지 않는지는 upstream server commit이 보장하며 browser helper만으로 증명하지 않습니다. |
| 후속 연결 | `06d2eb7a93cc`가 formatting/resume predicate와 fresh-ticket reconnect를 검증하고 `1abda1299ad8`이 browser reconnect를 실행합니다. |

#### A-level 불변식 종합

- **핵심 책임:** guest lobby에서 transient result를 명시하고 active room event를 회수하면 game screen으로 복귀합니다.
- **실패 영향:** lobby는 active-room event를 새 matchmaking으로 처리하지 않고 game screen으로 복귀해야 하며 non-persisted result는 명시적으로 transient라고 알려야 했습니다.
- **결과 불변식:** active room event를 lobby chat/presence로 처리하지 않고 즉시 game route로 되돌립니다. transient result는 rating/history를 암시하지 않습니다.
- **남은 제한:** guest 결과가 실제로 DB에 저장되지 않는지는 upstream server commit이 보장하며 browser helper만으로 증명하지 않습니다.

#### Fix 복원

| 항목 | 근거 |
| --- | --- |
| 이전 가정 | lobby socket은 lobby event만 받고 finish 결과는 일반 match와 같은 presentation으로 처리할 수 있다고 봤습니다. |
| 실제 실패/위험 | 복구된 room을 놓치거나 transient guest result를 durable history로 오인합니다. |
| 근본 원인 | deployment capability와 cross-route realtime event를 분류하는 browser policy가 없었습니다. |
| 수정된 불변식 | active room event는 `/play` 복귀 신호이고 `persisted:false` finish는 명시적인 임시 결과 notice입니다. |
| 회귀 근거 | `06d2eb7a93cc`의 pure policy tests와 `1abda1299ad8`의 reconnect E2E입니다. |

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `4f5199097284` |
| 파일 | `apps/web/src/lib/demoPolicy.ts` |
| 함수/위치 | `formatTransientResultNotice / shouldResumeGameFromLobby` |
| 근거 요약 | non-persisted result는 “전적에 저장되지 않았습니다”로 표시하고 `queue.matched`/`game.snapshot`은 lobby에서 `/play`로 돌아갈 신호로 분류합니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `9f49db1d9f1d` — `test(guest): 체험 기능 오용 방지 검증`
- Thread 내 다음 관련 SHA: `06d2eb7a93cc` — `test(guest): 체험 환경의 복구 경계 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.11. `test(guest): 체험 환경의 복구 경계 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `06d2eb7a93cc` |
| Importance | A |
| Tags | AUTH, SIMULATION, REALTIME |
| Source에서 확정된 역할 | fresh-ticket reconnect, duplicate match 차단, transient result presentation을 확장 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/demoPolicy.test.ts`의 transient notice와 lobby resume event case를 확인합니다.
- `apps/web/src/game/GameSocketClient.test.ts`가 close 후 fresh ticket으로 reconnect하고 original queue command를 보내지 않는지 확인합니다.
- `apps/web/src/game/gameConnection.test.ts`의 `canStartNewMatch` reconnecting/current-room negative case를 확인합니다.
- 같은 commit의 GuestAccess cleanup/env tests는 upstream `2b274686e6d4`를 검증하는 server evidence이며 browser guarantee와 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | reconnect/presentation fix는 여러 cross-layer condition을 추가했지만 transient wording, active room redirect, no duplicate intent, timer cleanup을 직접 고정하지 않았습니다. |
| 해결하려던 문제 | guest recovery는 짧은 ticket, room state, lobby/game route, process memory가 함께 움직여 happy path만으로 안전성을 판단할 수 없었습니다. |
| 핵심 결정 | fake timers/fake sockets와 pure policy assertions를 추가해 reconnect와 browser presentation을 deterministic하게 재현했습니다. |
| 입력 → 상태 전이 → 출력 | initial connect/send → fake close → timer advance → second ticket/socket → no sent initial frame; policy input → exact result/boolean assertion 순서입니다. |
| ownership/lifetime/cleanup | test가 timers/socket doubles를 소유하고 cleanup 후 real timers로 복원합니다. server resource cases는 별도 owner의 cleanup을 검증합니다. |
| failure/rollback/retry | reconnect는 ticket provider를 다시 호출하고 second socket에 original queue command가 없으며 current room에서는 새 match predicate가 false입니다. transient notice와 resume event 분류도 고정됩니다. |
| 보장하는 것 | browser recovery와 presentation의 high-risk invariant를 deterministic하게 증명합니다. |
| 보장하지 않는 것 | 실제 browser route navigation, real network/ticket server, PvP/AI timing은 증명하지 않습니다. |
| 후속 연결 | `1abda1299ad8`이 실제 demo-mode browser entry, PvP, AI fallback, forced socket reconnect를 최종 통합 검증합니다. |

#### A-level 불변식 종합

- **핵심 책임:** fresh-ticket reconnect, duplicate match 차단, transient result presentation을 확장 검증합니다.
- **실패 영향:** guest recovery는 짧은 ticket, room state, lobby/game route, process memory가 함께 움직여 happy path만으로 안전성을 판단할 수 없었습니다.
- **결과 불변식:** browser recovery와 presentation의 high-risk invariant를 deterministic하게 증명합니다.
- **남은 제한:** 실제 browser route navigation, real network/ticket server, PvP/AI timing은 증명하지 않습니다.

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | guest room recovery는 fresh ticket을 쓰되 original match intent를 반복하지 않고 transient result/route presentation이 명확합니다. |
| 재현한 실패/경계 | close/retry timer, duplicate queue command, reconnecting room의 새 match start, persisted:false wording, active-room lobby event입니다. |
| 테스트 기법 | fake timers, fake sockets, pure policy/reducer tests입니다. |
| 증명하는 것 | browser transport/policy branch를 deterministic하게 증명합니다. |
| 증명하지 않는 것 | 실제 deployment browser/network와 server resource isolation 전체는 증명하지 않습니다. |
| 검증 분류 | high-risk recovery deterministic regression |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 비교 기준

- Thread 내 직전 관련 SHA: `4f5199097284` — `fix(web): 중단된 game reconnect 복구`
- Thread 내 다음 관련 SHA: `1abda1299ad8` — `test(e2e): 비회원 체험 브라우저 흐름 검증`
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

### 5.12. `test(e2e): 비회원 체험 브라우저 흐름 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1abda1299ad8` |
| Importance | A |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | demo-mode guest entry, restricted navigation, two-browser PvP, 6초 AI fallback, ticket-based reconnect를 Playwright로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `e2e:guest-demo` command와 `E2E_APP_MODE=demo`, single worker 조건을 확인합니다.
- `tests/e2e/guest-demo.spec.ts`의 `enterAsGuest`, two isolated contexts, frame watcher, `routeWebSocket` forced close를 확인합니다.
- guest name regex, navigation link text, hidden admin, PvP ready/playing, AI fallback 5.5~10초, reconnecting→playing assertion을 확인합니다.
- test skip 조건이 demo mode/chromium desktop에 어떻게 제한되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | unit/policy tests는 실제 Next middleware, cookie session, 두 browser matching, elapsed-time fallback, forced network interruption을 한 흐름에서 실행하지 않았습니다. |
| 해결하려던 문제 | public demo의 핵심 약속을 실제 browser와 running services에서 통합 검증할 최종 evidence가 필요했습니다. |
| 핵심 결정 | demo 전용 Playwright suite를 만들고 isolated browser contexts, WebSocket frame timestamp 관찰, routeWebSocket forced close를 사용했습니다. |
| 입력 → 상태 전이 → 출력 | guest button → server-named cookie session → restricted nav; 두 contexts → queue/ready/PvP; single guest queue → sent/received frame time delta → AI; active game → routed socket close → second connection → playing 복귀 순서입니다. |
| ownership/lifetime/cleanup | Playwright context가 guest cookie isolation을, test route가 socket interception을, running app/server가 실제 lifecycle을 소유합니다. finally에서 contexts를 닫습니다. |
| failure/rollback/retry | suite는 demo mode가 아니면 skip하고 time/reconnect scenarios는 desktop project에서 한 번만 실행합니다. reconnect는 connection count와 UI state로 확인합니다. |
| 보장하는 것 | guest browser의 핵심 entry/capability/matchmaking/recovery를 broad end-to-end로 증명하도록 작성됐습니다. |
| 보장하지 않는 것 | 이 작업 환경에서는 suite를 실제 실행하지 못했으므로 문서에 pass 결과를 기록하지 않습니다. DB non-persistence나 장시간 부하 한계도 이 E2E만으로는 증명하지 않습니다. |
| 후속 연결 | 이 commit이 Thread의 최종 통합 evidence이며 단위 test의 policy/transport 보장을 실제 UI 흐름과 연결합니다. |

#### A-level 불변식 종합

- **핵심 책임:** demo-mode guest entry, restricted navigation, two-browser PvP, 6초 AI fallback, ticket-based reconnect를 Playwright로 검증합니다.
- **실패 영향:** public demo의 핵심 약속을 실제 browser와 running services에서 통합 검증할 최종 evidence가 필요했습니다.
- **결과 불변식:** guest browser의 핵심 entry/capability/matchmaking/recovery를 broad end-to-end로 증명하도록 작성됐습니다.
- **남은 제한:** 이 작업 환경에서는 suite를 실제 실행하지 못했으므로 문서에 pass 결과를 기록하지 않습니다. DB non-persistence나 장시간 부하 한계도 이 E2E만으로는 증명하지 않습니다.

#### Test 복원

| 항목 | 근거 |
| --- | --- |
| 검증 대상 production 불변식 | demo guest가 입력 없이 진입하고 제한된 메뉴만 보며 PvP/AI/reconnect를 사용할 수 있습니다. |
| 재현한 실패/경계 | cookie/session entry, two-browser isolation, 6초 AI fallback, forced WebSocket close와 fresh reconnect입니다. |
| 테스트 기법 | Playwright serial suite, isolated contexts, WebSocket frame timestamp, `routeWebSocket` interception입니다. |
| 증명하는 것 | 작성된 scenario가 실제 running browser/service boundary를 통과할 때의 end-to-end contract를 검증하도록 구성됐음을 code inspection으로 확인했습니다. |
| 증명하지 않는 것 | 이번 workbook 환경에서 test가 pass했다는 runtime evidence, DB non-persistence, sustained load는 증명하지 않습니다. |
| 검증 분류 | demo-mode browser end-to-end regression |

> 실행 상태: 이 workbook 작성 환경에서는 repository test command를 실행하지 않았습니다. 위 내용은 해당 SHA의 test source와 production path를 정적 조사해 복원한 것이며 pass 결과를 주장하지 않습니다.

#### 핵심 코드 근거

| 항목 | 값 |
| --- | --- |
| SHA | `1abda1299ad8` |
| 파일 | `tests/e2e/guest-demo.spec.ts` |
| 함수/위치 | `guest demo browser flow` |
| 근거 요약 | 두 browser context의 PvP, queue frame 시점으로 측정한 6초 AI fallback, routed WebSocket 강제 종료 뒤 두 번째 연결과 playing 복귀를 검증하도록 작성됐습니다. |

#### 비교 기준

- Thread 내 직전 관련 SHA: `06d2eb7a93cc` — `test(guest): 체험 환경의 복구 경계 검증`
- 이 SHA가 이 Thread의 마지막 고정 commit입니다.
- 설명은 이 SHA에서 존재하는 코드만 사용하며 후속 HEAD 구현을 역투영하지 않습니다.

## 6. 불변식 발전 기록

| 단계 | 관련 SHA | 학습 기록 |
| --- | --- | --- |
| runtime mode/secret 경계 | `f801ccd09cf0` | demo capability와 fail-closed secret/proxy 설정이 명시됐습니다. |
| browser policy 도입 | `e39316254e44` → `a9fc8a8328b2` | navigation/presentation/restricted path와 guest adapter/middleware가 정의됐습니다. |
| UI capability 적용 | `7fdef5d224c4` → `fe0f3e0ad0ad` | guest login, lobby, shell, play가 중앙 policy를 소비합니다. |
| entry/abuse evidence | `618330916629` → `9f49db1d9f1d` | API entry와 negative capability/secret/resource boundary를 검증합니다. |
| transient/recovery 교정 | `4f5199097284` | active room 복귀와 non-persisted result notice, transport reconnect가 같은 cross-cutting fix에서 연결됐습니다. |
| recovery regression | `06d2eb7a93cc` | fresh ticket/no duplicate intent/policy formatting을 deterministic하게 고정했습니다. |
| browser E2E | `1abda1299ad8` | entry, restricted nav, PvP, AI fallback, reconnect scenario를 running-system test로 작성했습니다. |

## 7. Failure → Fix → Test 관계

| 이전 상태/가정 | 실패 또는 위험 | Fix 연결 | Test/후속 근거 | 관계 해설 |
| --- | --- | --- | --- | --- |
| mode가 implicit/weak secret 허용 | guest identity와 rate-limit trust 약화 | `f801ccd09cf0` | `9f49db1d9f1d`, `06d2eb7a93cc` env tests | mode가 implicit/weak secret 허용 → guest identity와 rate-limit trust 약화 → `f801ccd09cf0` → `9f49db1d9f1d`, `06d2eb7a93cc` env tests |
| central policy 미적용 | registered capability가 guest UI에 노출 | `584d17f3aad1`, `658fafd43f88`, `fe0f3e0ad0ad` | `9f49db1d9f1d`, `1abda1299ad8` | central policy 미적용 → registered capability가 guest UI에 노출 → `584d17f3aad1`, `658fafd43f88`, `fe0f3e0ad0ad` → `9f49db1d9f1d`, `1abda1299ad8` |
| lobby에서 active room/transient result 미분류 | room 상실 또는 durable 결과 오인 | `4f5199097284` | `06d2eb7a93cc`, `1abda1299ad8` | lobby에서 active room/transient result 미분류 → room 상실 또는 durable 결과 오인 → `4f5199097284` → `06d2eb7a93cc`, `1abda1299ad8` |
| remote close 후 initial command 재전송 가능 | duplicate matchmaking intent | Thread 04 범위의 `4f5199097284` | `06d2eb7a93cc`, `1abda1299ad8` | remote close 후 initial command 재전송 가능 → duplicate matchmaking intent → Thread 04 범위의 `4f5199097284` → `06d2eb7a93cc`, `1abda1299ad8` |

## 8. Ownership·state·lifetime 변화

| 대상 | 이전 owner/state | 이후 owner/state | 수명/cleanup |
| --- | --- | --- | --- |
| deployment capability | 암묵적 env | validated APP_MODE + public browser mode | process/build 수명 |
| browser capability matrix | component별 조건 가능 | `demoPolicy.ts` | application build/runtime 수명 |
| direct route presentation | navigation 숨김만 | Next middleware restricted path | request 수명 |
| guest identity entry | development form inputs | server-named guest response + cookie session | 2시간 guest session(서버 계약) |
| transient result presentation | 일반 finish와 미분리 | demo policy formatter + HomePage notice | result notice 수명 |
| active room recovery navigation | lobby에 머무름 | lobby event predicate + `/play` redirect | socket event 처리 순간 |
| reconnect transport | 재연결 없음/intent 재사용 | `GameSocketClient` fresh ticket/backoff | 15초 reconnect window |

## 9. Thread 최종 상태

마지막 SHA에서 demo browser는 입력 없이 server-named guest session에 진입하고 lobby/play만 navigation으로 노출합니다. middleware는 registered-only direct URL을 404로 차단하고 lobby/play는 durable progress·ranking·chat을 광고하지 않습니다. lobby에서 active room event를 회수하면 `/play`로 복귀하며 non-persisted result는 전적 미저장 임시 결과로 표시됩니다. transport는 fresh ticket으로 bounded reconnect하되 original match intent를 반복하지 않습니다.

### 최종 실행 흐름

1. deployment가 validated `APP_MODE=demo`와 matching public browser mode로 시작합니다.
2. LoginPanel은 profile inputs 없이 `guestLogin()`을 호출하고 cookie session의 server-named user를 `me` cache에 넣습니다.
3. `AppShell`은 `createNavigation(true, ...)` 결과인 lobby/play만 렌더링하고 middleware는 restricted direct path를 404 처리합니다.
4. HomePage/PlayPage는 presentation flags로 persisted progress, ranking, lobby/match chat을 숨기고 결과 미저장을 명시합니다.
5. lobby socket에서 `queue.matched`/`game.snapshot`을 받으면 `/play`로 이동하고 `persisted:false game.finished`는 transient notice로 표시합니다.
6. game socket close 시 client는 fresh ticket으로 bounded reconnect하며 original queue command를 다시 보내지 않습니다.
7. unit/policy tests가 negative/recovery branches를, Playwright suite가 entry/PvP/AI/reconnect browser flow를 검증하도록 구성됩니다.

## 10. 학습 완료 점검

- [x] 모든 commit을 지정 SHA의 parent/diff와 비교했습니다.
- [x] 후속 HEAD 코드를 이전 SHA 설명에 역투영하지 않았습니다.
- [x] owner, lifetime, cleanup, failure branch와 non-guarantee를 기록했습니다.
- [x] fix는 이전 가정과 root cause에, test는 production path와 증명 범위에 연결했습니다.
- [x] A/B importance 깊이를 구분했고 source subject/tag/role을 유지했습니다.
- [x] 실행하지 않은 project test에 pass 결과를 만들지 않았습니다.
===== END FILE: 08-guest-browser-policy-and-transient-results.md =====

===== BEGIN FILE: README.md =====
# Browser application architecture Development Thread workbook

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- 동결 Thread: 8개
- Commit mapping: 83개, 고유 SHA 82개

## 1. 카테고리 경계

이 카테고리는 browser application이 직접 소유하는 Next.js runtime과 shell, navigation identity, HTTP adapter, resource screen, realtime transport/reducer/hook, React Query cache, authoritative snapshot projection, user input, guest presentation policy를 다룹니다.

다음은 이 카테고리의 주된 소유 범위가 아닙니다.

- server simulation, `GameHub` room lifecycle, persistence transaction과 database migration
- shared protocol 생산자 자체의 전체 발전사
- container orchestration, database readiness, metrics, drain 등 운영 subsystem 전체
- guest identity·ticket·result 보존의 server 구현 전체

browser가 이들을 소비하거나 표현하는 경계는 포함하지만 server-side 구현은 upstream dependency 또는 다른 카테고리의 근거로 구분합니다.

## 2. Phase 1 audit 결과

초기 category scaffold는 6개 Thread와 62개 commit mapping으로 구성되어 있었습니다. 실제 브랜치 역사를 대조한 뒤 8개 Thread와 83개 mapping으로 보정했습니다.

- 기존 `Application shell·resource screen·API adapter` Thread는 runtime/navigation, HTTP trust boundary, server-backed screen이라는 독립된 소유권 이야기를 한 문서에 묶고 있어 3개 Thread로 분리했습니다.
- 초기 play route는 generic screen 목록에서 authoritative rendering/input Thread로 이동했습니다.
- production web start, lobby writer/live metrics/socket, match-chat writer, cookie-only credential 전환, API/query regression, reconnect, room-scoped chat, guest browser E2E 등 실제 fix·test가 전제로 삼는 누락 commit을 추가했습니다.
- generic investigation 문구를 각 SHA의 실제 파일·함수·state·cleanup·negative branch를 지목하는 작업으로 교체했습니다.
- source classification의 subject, importance, tags와 commit 순서를 유지했습니다.
- 선택된 browser history에는 A/B importance만 존재하므로 학습 깊이를 인위적으로 S/C로 재분류하지 않았습니다.

### 의도적 중복 SHA

`4f5199097284`는 이 category의 유일한 의도적 중복입니다.

- Thread 04는 `GameSocketClient`, reducer, hook, play control의 fresh-ticket reconnect와 duplicate match intent 차단만 다룹니다.
- Thread 08은 `HomePage`와 `demoPolicy`의 active-room route recovery와 transient result notice만 다룹니다.
- 같은 SHA를 사용하지만 조사 파일과 책임이 겹치지 않으며 한 commit이 실제로 두 browser subsystem을 함께 수정한 사실을 보존합니다.

## 3. 동결 Thread 목록

| 순서 | 파일 | Thread | Commit 수 |
| ---: | --- | --- | ---: |
| 1 | [`01-application-shell-auth-entry-and-navigation-identity.md`](01-application-shell-auth-entry-and-navigation-identity.md) | Application shell·auth entry·navigation identity | 11 |
| 2 | [`02-browser-http-adapter-runtime-validation-and-cookie-credentials.md`](02-browser-http-adapter-runtime-validation-and-cookie-credentials.md) | Browser HTTP adapter·runtime validation·cookie credentials | 6 |
| 3 | [`03-resource-screens-actions-and-truthful-server-state.md`](03-resource-screens-actions-and-truthful-server-state.md) | Resource screens·actions·truthful server state | 14 |
| 4 | [`04-game-connection-reducer-and-transport-client.md`](04-game-connection-reducer-and-transport-client.md) | Game connection reducer와 transport client | 9 |
| 5 | [`05-game-connection-hook-migration-and-legacy-removal.md`](05-game-connection-hook-migration-and-legacy-removal.md) | Game connection hook 전환과 legacy 제거 | 11 |
| 6 | [`06-react-query-cache-ownership-and-invalidation.md`](06-react-query-cache-ownership-and-invalidation.md) | React Query cache ownership과 invalidation | 10 |
| 7 | [`07-authoritative-snapshot-rendering-and-input.md`](07-authoritative-snapshot-rendering-and-input.md) | Authoritative snapshot rendering과 입력 | 10 |
| 8 | [`08-guest-browser-policy-and-transient-results.md`](08-guest-browser-policy-and-transient-results.md) | Guest browser policy와 transient results | 12 |

## 4. 역사 및 근거 규율

- 지정 브랜치의 `commit/commit-importance.md`는 이 branch가 root `72ac4c1870f`부터 HEAD `71c5c13480f0`까지 433개인 독립 선형 역사라고 명시합니다.
- 모든 선택 SHA는 해당 branch source classification에 존재하고 GitHub exact-SHA commit inspection으로 resolve됨을 확인했습니다.
- 각 구현 설명은 해당 SHA의 diff와 당시 파일만 사용합니다. final HEAD 코드를 이전 commit의 동작으로 사용하지 않습니다.
- server-side upstream commit은 browser consumer가 의존하는 contract를 설명할 때만 명시적으로 구분해 언급합니다.
- 실제로 실행하지 않은 test, build, browser command에는 성공 결과를 기록하지 않습니다.

## 5. Phase 2 실행 환경

이 작업 환경에서는 외부 DNS 제한으로 repository를 로컬 checkout할 수 없었습니다. 따라서 project source의 build/test/E2E command는 실행하지 않았고, GitHub connector를 사용한 exact-SHA 정적 inspection만 수행했습니다.

실제로 실행한 검증은 생성된 workbook 자체의 파일 대응, commit metadata 보존, placeholder 제거, scaffold freeze hash, Markdown 기본 형식, ZIP member와 CRC 검사입니다.

## 6. Phase 2 완료 기록

- [x] frozen scaffold 8개 Thread와 completed 8개 Thread가 1:1로 대응합니다.
- [x] README를 포함한 상대 경로와 파일명이 동일합니다.
- [x] commit SHA, subject, importance, tags, role, commit order를 frozen scaffold와 동일하게 유지했습니다.
- [x] 각 SHA의 concrete investigation task를 보존하고 learner-facing 기록을 exact-SHA evidence로 채웠습니다.
- [x] fix/test는 이전 가정·failure·root cause·corrected invariant·증명/비증명 범위에 연결했습니다.
- [x] unfinished placeholder가 completed에 남지 않았습니다.
- [x] project command를 실행하지 않았다는 제한을 명시하고 runtime pass evidence를 만들지 않았습니다.
- [x] local structural/hash/archive validation을 실행해 통과한 경우에만 이 파일과 ZIP을 산출했습니다.
===== END FILE: README.md =====

