===== BEGIN FILE: 01-query-state-and-route-preserving-navigation.md =====
# Development Thread 01 — Query State and Route-Preserving Navigation

## 0. Source and frozen audit scope

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/portfolio`
- Category: `02-routing-navigation-and-page-lifecycle`
- Change range for directed inspection: `902eddcef875^..b669e04c0932`
- Classification source: `commit/commit-importance.md` on `web/portfolio`
- Historical rule: inspect every commit at its exact SHA; do not project later code backward.
- Phase 1 status: audited and frozen before learner-facing completion.
- Thread boundary: URL query 해석과 내부 URL 변환 정책만 다룹니다. typed content link transport 자체는 Category 03의 독립 Thread이므로 `f63c978c71c9`를 이 Thread에서 제거했습니다.
- Execution note: source inspection and runtime command evidence must be recorded separately.

### Phase 1 audit decisions

- `f63c978c71c9`는 query helper의 consumer이지만 typed internal/external link transport를 소유하므로 Category 03으로 환원했습니다.
- `3353032ba23b`의 exact URL boundary test를 누락 보완했습니다.
- `b669e04c0932`를 후속 default-policy ownership correction으로 유지했습니다.

## 1. Learning goal and planned invariants

서버 route가 URL의 `view`/`debug` 상태를 해석하고, 내부 링크가 기존 query/hash와 configured default design 정책을 잃지 않도록 만드는 과정을 재구성합니다.

- 유효한 `view`만 선택되고 unknown/누락 값은 content에 설정된 기본 디자인으로 돌아갑니다.
- 배열형 `view`/`debug`는 첫 값만 사용합니다.
- 내부 root-relative URL만 변환하며 절대 URL과 protocol-relative URL은 그대로 둡니다.
- 기존 query/hash는 보존하고 기본 디자인의 불필요한 `view`는 제거할 수 있습니다.
- 기본 디자인 ownership은 최종적으로 switcher caller의 명시적 `defaultId`에 있습니다.

## 2. Questions to answer

- URL이 design/debug 상태의 source of truth가 된 정확한 SHA는 무엇입니까?
- 기본 디자인을 URL에 포함하거나 제거하는 결정은 어떤 입력에 의해 달라집니까?
- 왜 `URLSearchParams` 기반 재조립이 단순 문자열 연결보다 필요한가요?
- unit test가 증명하는 URL 계약과 실제 browser navigation이 별도로 남는 부분은 무엇입니까?
- 전역 기본값 의존을 explicit prop으로 바꾼 ownership 이동은 무엇을 제거합니까?

## 3. Completion criteria

- 네 commit을 정확한 chronology로 설명합니다.
- 각 SHA의 production 함수와 test assertion을 혼동하지 않습니다.
- 기본/비기본 design, debug, 기존 query/hash, external/protocol-relative URL의 결과를 설명합니다.
- `335303...`의 실행 여부를 source inspection과 구분해 기록합니다.
- `b669...`이 behavior rewrite가 아니라 default-policy ownership refactor인 이유를 설명합니다.

## 4. Fixed commit map

| # | Commit | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `902eddcef875` | feat(navigation): 템플릿 URL과 쿼리 해석 추가 | A | ARCH, ROUTING | Introduce the canonical query-state utilities for selecting a home template, enabling content-debug mode, and constructing state-preserving internal URLs. |
| 2 | `f4233f024890` | feat(home): 쿼리 기반 디자인 전환 연결 | A | ARCH, ROUTING, RENDERER | Resolve the `view` and debug query parameters in the root page and dispatch to the classic or design home on the server. |
| 3 | `3353032ba23b` | test(content): Vitest 기반 콘텐츠 계약 검증 추가 | A | CONTENT, VALIDATION, TEST | Introduce an executable test boundary for the portfolio content model with Vitest, jsdom, and Testing Library support. |
| 4 | `b669e04c0932` | refactor(navigation): 디자인 전환 URL 기본값 명시 | A | ARCH, ROUTING, REFACTOR | Make the configured default design an explicit input to every design-switcher instance and generate switch links through `createTemplateHref`. |

The commit SHA, order, subject, importance, tags, and source-defined role in this map are frozen.

## 5. Commit-by-commit reconstruction

### 1. `902eddcef875` — feat(navigation): 템플릿 URL과 쿼리 해석 추가

- Importance: **A**
- Tags: `ARCH, ROUTING`
- Fixed role: Introduce the canonical query-state utilities for selecting a home template, enabling content-debug mode, and constructing state-preserving internal URLs.

#### Historical inspection tasks

- 부모 `7c95a6f387b4`와 비교해 `src/lib/portfolio/selectors.ts` 및 새 `src/lib/portfolio/template-href.ts`의 책임이 어디서 시작되는지 확인합니다.
- `resolveHomeTemplateId`, `resolveContentDebug`, `getTemplateHref`, `createTemplateHref`의 호출 관계와 배열 쿼리에서 첫 값만 사용하는 규칙을 추적합니다.
- `createTemplateHref`가 기존 query/hash를 분해하고 `URLSearchParams`로 `view`와 `debug`를 갱신한 뒤 재조립하는 순서를 적습니다.
- 상대 내부 경로, 절대 외부 URL, protocol-relative URL(`//...`), 기본 디자인, `alwaysInclude`, 기존 `view`, 기존 hash를 각각 대입해 보장과 비보장을 구분합니다.
- 이 SHA에는 실행 테스트가 추가되지 않았다는 점을 확인하고, 코드 inspection으로만 확정할 수 있는 범위를 적습니다.

#### Reconstruction result

**이전 상태와 문제**

부모 `7c95a6f387b4`에는 aggregate content loader와 selector들이 있었지만, route query를 해석하거나 내부 URL에 선택한 design/debug 상태를 다시 싣는 공통 정책은 없었습니다. 따라서 각 caller가 문자열을 직접 조립하면 기본 design 처리, 기존 query/hash 보존, 외부 URL 비변형 규칙이 서로 달라질 위험이 있었습니다.

**구현 결정과 실행 경로**

- `src/lib/portfolio/selectors.ts`
  - `resolveHomeTemplateId(value, presentation)`은 배열이면 첫 항목만 선택합니다.
  - 선택값이 `presentation.templates`에 실제로 존재할 때만 그 ID를 반환하고, 그 외에는 `presentation.defaultHomeTemplate`로 돌아갑니다.
  - `resolveContentDebug(value)`도 배열의 첫 값을 사용하며 정확히 `"content"`일 때만 `true`입니다.
  - `getTemplateHref(...)`는 content-owned default ID를 채워 `createTemplateHref(...)`에 위임합니다.
- `src/lib/portfolio/template-href.ts`
  - template ID가 없거나 href가 `/`로 시작하지 않거나 `//`로 시작하면 원문을 즉시 반환합니다.
  - hash를 먼저 떼고, 남은 값을 path/query로 분리한 뒤 `URLSearchParams`를 만듭니다.
  - 비기본 design 또는 `alwaysInclude`이면 `view`를 설정하고, 기본 design이면 기존 `view`를 삭제합니다.
  - `contentDebug`가 true일 때만 `debug=content`를 설정합니다.
  - 마지막에 path, query, hash를 원래 순서로 다시 결합합니다.

핵심 코드는 다음 정책입니다. 이 발췌는 이 commit의 `src/lib/portfolio/template-href.ts`에서 확인한 내용입니다.

```ts
if (!templateId || !href.startsWith("/") || href.startsWith("//")) {
  return href;
}

if (options.alwaysInclude || templateId !== defaultTemplateId) {
  params.set("view", templateId);
} else {
  params.delete("view");
}
```

**소유권과 실패 경계**

URL 변환 중 생성되는 `URLSearchParams`는 함수 호출 내부의 일시 객체이며 browser history나 전역 상태를 소유하지 않습니다. 잘못된 design 값은 예외가 아니라 configured default로 정규화됩니다. 반면 malformed URL 전체를 일반 URL parser로 검증하지는 않습니다. 이 helper의 입력 계약은 root-relative application href입니다.

**보장**

- 내부 URL의 기존 query와 hash를 보존하면서 `view`/`debug`만 정책적으로 갱신합니다.
- stale default `view`를 제거할 수 있습니다.
- 외부 절대 URL과 protocol-relative URL은 손대지 않습니다.
- repeated query의 route 해석은 첫 값에 고정됩니다.

**비보장과 후속 관계**

이 SHA에는 해당 helper를 실행하는 테스트가 없습니다. 따라서 이 절의 결과는 exact diff와 함수 코드 inspection으로 확정한 것이며 runtime 실행 결과가 아닙니다. 홈 route 소비는 `f4233f024890`, URL boundary regression은 `3353032ba23b`, default ownership 명시는 `b669e04c0932`에서 이어집니다.

### 2. `f4233f024890` — feat(home): 쿼리 기반 디자인 전환 연결

- Importance: **A**
- Tags: `ARCH, ROUTING, RENDERER`
- Fixed role: Resolve the `view` and debug query parameters in the root page and dispatch to the classic or design home on the server.

#### Historical inspection tasks

- `src/app/page.tsx`가 동기 컴포넌트에서 async route entry로 바뀐 지점을 부모와 비교합니다.
- `searchParams`를 await한 뒤 `resolveHomeTemplateId`와 `resolveContentDebug`를 호출하고 `ClassicHomeRoute` 또는 `DesignHomeRoute`로 분기하는 실행 순서를 추적합니다.
- 두 renderer에 전달되는 `templateSwitcher`의 `activeId`, `contentDebug`, `currentPath`, `templates`가 어떤 URL 상태를 다음 탐색으로 운반하는지 확인합니다.
- 이 commit이 홈 route만 연결하며 다른 route와 공용 shell의 상태 보존까지 보장하지는 않는다는 경계를 적습니다.

#### Reconstruction result

**이전 상태와 문제**

홈 entry는 content-backed 두 presentation을 가지고 있었지만 URL query가 어느 renderer를 선택하는지 결정하지 않았습니다. `902edd...`에서 helper가 생겼어도 route caller가 이를 사용하지 않으면 URL은 상태의 source of truth가 될 수 없었습니다.

**구현 경로**

`src/app/page.tsx`는 async page가 되고 `searchParams?: RouteSearchParams`를 받습니다. 실행 순서는 다음과 같습니다.

1. portfolio content를 읽습니다.
2. `searchParams`가 있으면 await하고, 없으면 빈 객체를 사용합니다.
3. `resolveHomeTemplateId(params.view, content.presentation)`으로 active template을 결정합니다.
4. `resolveContentDebug(params.debug)`으로 content hint 상태를 결정합니다.
5. active template이 `classic`이면 `ClassicHomeRoute`, 그 외에는 `DesignHomeRoute`를 server에서 선택합니다.
6. 두 renderer 모두 `templateSwitcher`에 같은 `activeId`, `contentDebug`, `currentPath: "/"`, template 목록을 받습니다.

이 결정은 client-side local state가 아니라 request URL에서 매 요청 재구성됩니다. renderer는 선택 결과를 소비하지만 query parsing ownership은 App Router page에 남습니다.

**보장**

- `/?view=classic`과 같은 요청은 server render 단계에서 classic renderer로 분기합니다.
- unknown/누락 `view`는 `902edd...`의 default fallback 정책을 따릅니다.
- debug 상태와 active template이 홈의 후속 design navigation에 전달됩니다.

**비보장**

이 시점의 연결 범위는 root page뿐입니다. `/projects`, About, Resume, Contact 같은 후속 route나 공용 shell 전체의 링크가 같은 상태를 보존한다고 말할 수 없습니다. 또한 이 commit에는 route test가 추가되지 않았습니다. 실제 browser navigation 결과가 아니라 route code와 diff로 확인한 구조입니다.

### 3. `3353032ba23b` — test(content): Vitest 기반 콘텐츠 계약 검증 추가

- Importance: **A**
- Tags: `CONTENT, VALIDATION, TEST`
- Fixed role: Introduce an executable test boundary for the portfolio content model with Vitest, jsdom, and Testing Library support.

#### Historical inspection tasks

- `package.json`, `vitest.config.ts`, `src/test/setup.ts`, `src/lib/portfolio.test.ts`를 확인해 테스트 실행 경계와 jsdom 환경을 재구성합니다.
- `propagates designs and debug state on internal links only` 테스트가 기본 디자인의 `view` 제거, 비기본 디자인 추가, 기존 query/hash 보존, `alwaysInclude`, `debug=content` 전파를 어떤 기대값으로 검증하는지 적습니다.
- 절대 URL과 `//` URL을 그대로 두는 assertion이 어느 production 함수 경로를 통과하는지 확인합니다.
- 이 테스트가 URL 문자열 생성의 결정적 unit/boundary evidence라는 점과 실제 Next.js navigation, browser history, click 동작을 증명하지 않는다는 점을 구분합니다.

#### Reconstruction result

**검증 대상과 기법**

이 commit은 Vitest, jsdom, Testing Library를 프로젝트에 도입한 넓은 content-contract test boundary입니다. 이 Thread와 직접 관련된 부분은 `src/lib/portfolio.test.ts`의 내부 URL 생성 테스트입니다. production 경로는 public portfolio facade가 노출하는 `getTemplateHref`/selector를 통해 `createTemplateHref`에 도달합니다.

`propagates designs and debug state on internal links only` 계열 assertion은 다음 경계를 고정합니다.

| 입력 범주 | 기대 |
| --- | --- |
| 비기본 design | `/projects?view=<id>` |
| configured default `editorial` | 불필요한 `view` 없음 |
| 기존 query와 hash | `page=2`와 `#featured` 유지 |
| stale `view=classic` + default 선택 | 기존 `view` 삭제 |
| `alwaysInclude` | 기본 design도 명시 |
| `contentDebug: true` | `debug=content` 추가 |
| `https://example.com/...` | 원문 유지 |
| `//example.com/...` | 원문 유지 |

**무엇을 증명하는가**

이것은 pure URL transformation에 대한 결정적 unit/boundary regression입니다. 문자열 입력과 출력이 고정되어 있어 기본값 제거, query/hash 보존, 내부/외부 경계의 회귀를 빠르게 찾습니다. 특히 external URL이 application query로 오염되지 않는 것을 직접 assertion합니다.

**무엇을 증명하지 않는가**

- Next.js `Link` click이 실제로 browser history를 갱신하는지
- back/forward navigation에서 상태가 복원되는지
- 모든 route component가 helper를 빠짐없이 사용하는지
- hydration 또는 mobile interaction이 정상인지

이 환경에서는 repository checkout과 dependency 설치가 불가능해 이 테스트를 실행하지 않았습니다. 확인한 것은 exact SHA의 test source와 assertion이며, 실행 성공을 주장하지 않습니다.

**후속 관계**

이 test는 당시 configured default인 `editorial`을 기준으로 helper behavior를 고정합니다. `b669e04c0932`가 caller-owned `defaultId`를 명시화할 때 test fixture는 새 prop 계약에 맞게 갱신되지만, 이 commit 자체가 임의의 대체 default configuration까지 증명하는 것은 아닙니다.

### 4. `b669e04c0932` — refactor(navigation): 디자인 전환 URL 기본값 명시

- Importance: **A**
- Tags: `ARCH, ROUTING, REFACTOR`
- Fixed role: Make the configured default design an explicit input to every design-switcher instance and generate switch links through `createTemplateHref`.

#### Historical inspection tasks

- 부모 상태에서 `DesignSwitcher`가 `getTemplateHref`를 통해 모듈 전역 기본값에 결합되어 있던 경로를 확인합니다.
- `src/components/portfolio/design-switcher.tsx`, `site-shell.tsx`, `src/designs/*`, `src/lib/portfolio/page-context.ts`에서 `defaultId`가 생성되어 모든 switcher instance까지 전달되는 경로를 추적합니다.
- `getTemplateHref` 대신 `createTemplateHref`를 직접 호출하면서 `defaultId`가 URL 포함/제거 정책의 명시적 입력이 된 이유를 설명합니다.
- 수정된 테스트 fixture가 새 필수 prop을 제공하는지 확인하고, 대체 기본값에 대한 새로운 독립 assertion이 실제로 추가됐는지 여부를 구분합니다.
- 동작을 보존하는 refactor와 숨은 기본값 ownership 이전을 각각 적습니다.

#### Reconstruction result

**이전 가정과 결합 문제**

초기 `getTemplateHref`는 portfolio module이 아는 `presentation.defaultHomeTemplate`을 내부에서 사용했습니다. 단일 aggregate만 소비할 때는 편리하지만, switcher가 전달받은 template 집합과 URL helper가 참조하는 default source가 암묵적으로 같다는 가정이 생깁니다. renderer/view-model 경계가 커진 뒤에는 caller가 가진 presentation의 default와 module-global default가 달라도 type이나 call signature가 이를 드러내지 못합니다.

**수정된 결정**

- switcher prop에 `defaultId`가 추가됩니다.
- `PageShell`, shared/design-specific shells, page-context 준비 경로가 `content.presentation.defaultHomeTemplate`을 switcher까지 전달합니다.
- `DesignSwitcher`는 facade `getTemplateHref` 대신 lower-level `createTemplateHref(currentPath, design.id, defaultId, options)`를 호출합니다.
- 모든 switcher instance가 URL 포함/제거 정책에 필요한 default를 명시적으로 제공합니다.

결과적으로 active design 목록, current path, debug 상태, default policy를 같은 caller-owned presentation context가 제공합니다. URL helper는 더 이상 어느 portfolio aggregate가 기본값을 소유하는지 추측하지 않습니다.

**소유권 이전**

| 이전 | 이후 |
| --- | --- |
| portfolio module 내부의 암묵적 default | switcher caller가 전달하는 `defaultId` |
| helper가 전역 content와 결합 | pure helper가 명시적 인자를 소비 |
| call site만 봐서는 default 정책 불명 | prop chain에서 default source 추적 가능 |

**보장과 비보장**

동일한 current configuration에서는 기존 URL 결과를 유지하면서 dependency를 드러냅니다. test fixture들은 새 필수 prop을 제공하도록 수정되었습니다. 그러나 diff에서 별도의 “default를 다른 ID로 바꾼 경우” 전용 assertion이 새로 추가된 것은 확인되지 않았습니다. 따라서 alternate-default behavior는 `createTemplateHref` 구현과 explicit propagation으로 inspection했으며 새 runtime regression 결과로 과장하지 않습니다.

이 commit은 `902edd...`의 canonical URL 정책을 폐기하지 않습니다. 오히려 그 정책의 마지막 숨은 입력을 caller ownership으로 이동시킨 refactor입니다.

## 6. Invariant evolution

Record where each invariant was introduced, extended, shown insufficient, corrected, and verified.

| 단계 | 상태 | 근거 |
| --- | --- | --- |
| 도입 | URL query 해석과 내부 href 변환을 canonical helper로 통합 | `902eddcef875` |
| route 연결 | root page가 request query로 renderer를 server dispatch | `f4233f024890` |
| 결정적 검증 | default 제거, query/hash 보존, external 비변형을 unit assertion | `3353032ba23b` |
| ownership 보정 | configured default를 caller가 `defaultId`로 명시 | `b669e04c0932` |

최종 invariant는 “URL이 design/debug 상태의 source of truth이며, URL 변환에 필요한 default 정책까지 현재 presentation context가 명시적으로 제공한다”입니다.

## 7. Failure → Fix → Test relationships

Connect fixes and tests to the exact earlier assumption or implementation they correct or verify.

- **Failure risk → Test:** 문자열 조립이 stale `view`, hash 손실, external URL 오염을 만들 수 있는 위험을 `335303...`의 table-like URL assertions가 막습니다.
- **Hidden assumption → Refactor:** `getTemplateHref`가 module-global default와 caller default가 같다고 가정하던 결합을 `b669...`가 explicit `defaultId`로 제거합니다.
- **검증 공백:** `b669...` 뒤 alternate default configuration을 독립적으로 실행하는 새 assertion은 확인되지 않았습니다. 기존 helper tests와 prop fixture update가 있지만, 이 ownership 보정 전체를 runtime으로 검증했다고 쓰지 않습니다.

## 8. Ownership, state, and responsibility changes

Track caller/callee ownership, browser/framework state, resource lifetime, and boundaries that deliberately remain outside the Thread.

| 책임 | 초기 | 최종 |
| --- | --- | --- |
| query parsing | 없음/route별 미연결 | App Router page가 helper 호출 |
| valid design 결정 | 없음 | `resolveHomeTemplateId` |
| debug 결정 | 없음 | `resolveContentDebug` |
| 내부 URL 변환 | caller별 문자열 위험 | `createTemplateHref` |
| configured default | helper 내부 aggregate 의존 | switcher caller의 `defaultId` |
| browser history | 이 Thread가 소유하지 않음 | 여전히 Next/browser 소유 |

## 9. Final Thread state

- root-relative internal href는 기존 query/hash를 유지하면서 design/debug 상태를 정규화합니다.
- unknown design은 configured default로 복귀합니다.
- default design은 필요하지 않으면 URL에서 제거되고, switcher처럼 명시가 필요한 caller는 `alwaysInclude`를 사용할 수 있습니다.
- 외부 및 protocol-relative URL은 변형하지 않습니다.
- runtime 실행 증거는 기록하지 않았습니다. exact commit code와 test source를 inspection했습니다.

## 10. Final architecture or execution flow

1. App Router page가 request `searchParams`를 await합니다.
2. `resolveHomeTemplateId`와 `resolveContentDebug`가 배열 첫 값과 configured presentation을 기준으로 상태를 정규화합니다.
3. server page가 active renderer를 선택합니다.
4. navigation caller는 current path, target design, explicit default ID, debug 옵션을 `createTemplateHref`에 전달합니다.
5. helper는 내부 URL만 query/hash-preserving 방식으로 다시 만들고 `Link` 또는 anchor consumer에 돌려줍니다.

## 11. Learning-completion checks

- [x] 네 SHA의 subject, importance, tags, order를 frozen scaffold와 일치시켰습니다.
- [x] `f63c...`를 typed link transport story로 분리한 이유를 설명했습니다.
- [x] production code와 `335303...` test source를 구분했습니다.
- [x] runtime test를 실행하지 않았다는 제한을 명시했습니다.
- [x] 기본값 ownership이 `b669...`에서 이동하는 과정을 연결했습니다.
===== END FILE: 01-query-state-and-route-preserving-navigation.md =====

===== BEGIN FILE: 02-shared-shell-navigation-and-mobile-menu.md =====
# Development Thread 02 — Shared Shell Navigation and Mobile Menu

## 0. Source and frozen audit scope

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/portfolio`
- Category: `02-routing-navigation-and-page-lifecycle`
- Change range for directed inspection: `e936c79b98bd^..b9571c485013`
- Classification source: `commit/commit-importance.md` on `web/portfolio`
- Historical rule: inspect every commit at its exact SHA; do not project later code backward.
- Phase 1 status: audited and frozen before learner-facing completion.
- Thread boundary: 공용 header/footer/page shell의 route-state 전파와 active/mobile navigation lifecycle을 다룹니다. design switcher 내부 hydration lifecycle은 다음 Thread로 분리합니다.
- Execution note: source inspection and runtime command evidence must be recorded separately.

### Phase 1 audit decisions

- `51806e1875e7`를 실제 chronology에 맞게 mobile menu 및 switcher integration보다 앞으로 이동했습니다.
- hydration/focus 문제를 별도 native switcher Thread로 분리해 shell composition과 겹치지 않게 했습니다.

## 1. Learning goal and planned invariants

개별 route markup에 흩어진 탐색을 공용 shell로 모으고, desktop/mobile 모두 동일한 route state와 접근성 문구를 사용하도록 진화한 과정을 재구성합니다.

- brand, primary navigation, footer home link는 active design/debug 상태를 보존합니다.
- root navigation은 exact match, 다른 navigation은 exact 또는 segment-prefix match로 active 상태를 정합니다.
- mobile navigation은 native `<details>/<summary>`를 사용해 browser가 disclosure state를 소유합니다.
- active design marker는 최종적으로 전체 shell wrapper에 있고 main landmark는 별도 identity를 가집니다.
- skip link와 main `id`가 연결되며 shell label은 presentation content가 소유합니다.

## 2. Questions to answer

- header 도입부터 complete `PageShell`까지 responsibility가 어떻게 이동합니까?
- `51806...`이 `7f77...`, `b957...`보다 먼저여야 하는 이유는 무엇입니까?
- active route 판정에서 `/`를 prefix로 취급하면 어떤 오류가 발생합니까?
- native mobile disclosure가 React state를 필요로 하지 않는 범위는 어디까지입니까?
- outer design wrapper와 main template marker를 분리한 최종 구조는 무엇을 가능하게 합니까?

## 3. Completion criteria

- 여섯 commit을 실제 시간 순서로 설명합니다.
- 각 시점에 아직 없는 기능을 후대 코드에서 역투영하지 않습니다.
- desktop/mobile href와 `aria-current` 정책을 비교합니다.
- shell이 소유하는 state/labels/landmarks와 route renderer 소유를 구분합니다.
- runtime test가 실행되지 않은 경우 code inspection 사실만 기록합니다.

## 4. Fixed commit map

| # | Commit | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `e936c79b98bd` | feat(shell): 브랜드와 주 탐색 헤더 추가 | B | ROUTING | Add the persistent site header with a brand link, profile identity, primary navigation, and an accessible navigation label. |
| 2 | `a2fd51d4da44` | feat(shell): 홈 디자인 전환 탐색 추가 | B | ROUTING | Introduce an optional design switcher that lists configured home templates, marks the active choice with `aria-current`, and creates explicit `view` URLs for the current path. |
| 3 | `8e11443dc26d` | feat(shell): 공용 푸터와 페이지 셸 추가 | A | ARCH, ROUTING | Complete the shared page frame with a footer and a `PageShell` composition boundary. |
| 4 | `51806e1875e7` | style(theme): 디자인 속성을 site shell로 승격 | B | ROUTING, RENDERER | Promote the active design marker from the home-content selector to the shared site shell. |
| 5 | `7f77ec1912d6` | feat(shell): 현재 navigation 상태와 모바일 메뉴 추가 | B | ROUTING | Make global navigation route-aware and add a keyboard-operable mobile menu. |
| 6 | `b9571c485013` | feat(shell): 디자인 선택기를 공용 shell에 연결 | A | ARCH, ROUTING | Integrate the design switcher into the shared site header and make shell-level interface text come from presentation content. |

The commit SHA, order, subject, importance, tags, and source-defined role in this map are frozen.

## 5. Commit-by-commit reconstruction

### 1. `e936c79b98bd` — feat(shell): 브랜드와 주 탐색 헤더 추가

- Importance: **B**
- Tags: `ROUTING`
- Fixed role: Add the persistent site header with a brand link, profile identity, primary navigation, and an accessible navigation label.

#### Historical inspection tasks

- `src/components/portfolio/site-shell.tsx`의 새 `SiteHeader`와 호출부를 확인합니다.
- brand `/` 링크와 `site.navigation` 반복 렌더링, navigation label, profile identity의 데이터 출처를 적습니다.
- 이 시점에 active-route 표시, query-state 보존, mobile menu가 아직 없다는 이전/후속 경계를 구분합니다.

#### Reconstruction result

`src/components/portfolio/site-shell.tsx`에 첫 `SiteHeader`가 추가됩니다. header는 profile identity를 brand 영역에 표시하고, brand link는 `/`, primary navigation은 `site.navigation`을 순회해 렌더링합니다. `<nav>`에는 accessible label이 있습니다.

이 시점의 navigation은 단순 addressability만 제공합니다. 현재 route를 비교하지 않으므로 `aria-current`가 없고, design/debug query를 다시 붙이지 않으며, mobile 전용 disclosure도 없습니다. 따라서 이 commit의 역할은 “persistent shell navigation의 첫 소유 위치”를 만든 것이지 완성된 route-state contract를 만든 것이 아닙니다.

### 2. `a2fd51d4da44` — feat(shell): 홈 디자인 전환 탐색 추가

- Importance: **B**
- Tags: `ROUTING`
- Fixed role: Introduce an optional design switcher that lists configured home templates, marks the active choice with `aria-current`, and creates explicit `view` URLs for the current path.

#### Historical inspection tasks

- `TemplateSwitcherProps`와 `SiteHeader`의 optional switcher 분기를 확인합니다.
- brand, primary navigation, template links가 `getTemplateHref`로 active design/debug 상태를 전파하는지 추적합니다.
- `alwaysInclude: true`가 현재 path의 design 선택 링크에서 기본 디자인도 명시하도록 만드는지 확인합니다.

#### Reconstruction result

header가 optional `TemplateSwitcherProps`를 받도록 확장됩니다. switcher가 있을 때 brand, primary navigation, template 선택 링크는 `getTemplateHref`를 통과합니다. 그 결과 active template과 `contentDebug`가 내부 탐색에 다시 실립니다.

template 목록은 current path를 대상으로 하고, 선택된 item은 `aria-current="page"`를 가집니다. design 선택 링크에는 `alwaysInclude: true`가 사용되어 configured default도 선택 메뉴 안에서는 명시적 `view` URL을 가질 수 있습니다.

아직 mobile navigation은 없고 switcher가 header 내부의 간단한 inline 목록입니다. 즉 상태 보존은 도입됐지만 responsive lifecycle과 dedicated switcher component는 후속입니다.

### 3. `8e11443dc26d` — feat(shell): 공용 푸터와 페이지 셸 추가

- Importance: **A**
- Tags: `ARCH, ROUTING`
- Fixed role: Complete the shared page frame with a footer and a `PageShell` composition boundary.

#### Historical inspection tasks

- `SiteFooter`와 `PageShell`이 추가되기 전 각 route가 header/main/footer를 어떻게 소유했는지 부모 diff로 확인합니다.
- `PageShell`이 header, `<main data-home-template>`, children, footer를 어떤 순서로 조합하고 어떤 props를 관통시키는지 추적합니다.
- footer home link에도 design/debug 상태 보존이 적용되는지 확인합니다.
- 공용 frame이 소유하는 것과 각 route renderer가 계속 소유하는 내용을 분리합니다.

#### Reconstruction result

**구조적 변화**

새 `SiteFooter`는 `site.footer.note`, copyright 문구, 그리고 `/`로 돌아가는 상태 보존 링크를 렌더링합니다. `PageShell`은 별도의 inner main을 만드는 구조가 아니라, **하나의 root `<main data-home-template={homeTemplate}>` 안에** 다음 순서로 자식을 배치합니다.

1. `SiteHeader`
2. route-owned `children`
3. `SiteFooter`

`SiteFooter`의 home link는 `getTemplateHref("/", homeTemplate, { contentDebug })`를 사용하므로 선택한 template과 content-debug 상태를 보존합니다. header는 `templateSwitcher`를 전달받습니다. 이 commit의 diff는 `src/components/portfolio/site-shell.tsx`만 바꾸므로, composition boundary를 정의하지만 기존 route call site를 일괄 이관하지는 않습니다.

**책임 경계**

- `PageShell`: root main, header/children/footer 조합, shell-level state 전달
- `SiteFooter`: footer copy와 상태 보존 home link
- route page/renderer: page-specific content, section order, route data derivation
- URL helper: href string 정책
- browser/Next: 실제 navigation lifecycle

`PageShell`은 route availability나 current-route active 판정을 아직 소유하지 않습니다. 이 commit의 A-level 의미는 여러 route가 채택할 수 있는 공통 composition boundary를 정의했다는 데 있으며, caller migration이나 별도 content landmark까지 보장하지는 않습니다.

### 4. `51806e1875e7` — style(theme): 디자인 속성을 site shell로 승격

- Importance: **B**
- Tags: `ROUTING, RENDERER`
- Fixed role: Promote the active design marker from the home-content selector to the shared site shell.

#### Historical inspection tasks

- `src/components/portfolio/site-shell.tsx`의 `data-site-design` 위치와 기존 `data-home-template`을 비교합니다.
- `src/app/globals.css` selector가 `main[data-home-template=...]`에서 shell-wide `[data-site-design=...]`으로 바뀐 이유와 적용 범위를 확인합니다.
- 이 commit의 실제 날짜가 `7f77...`, `b957...`보다 앞이라는 chronology를 유지합니다.

#### Reconstruction result

이 commit은 active design marker를 home-template 전용 selector와 분리합니다. `PageShell`의 root `<main>`에는 기존 `data-home-template={homeTemplate}`과 새 `data-site-design={homeTemplate}`이 **같이** 놓이고, global CSS는 `main[data-home-template="classic"]` 대신 `[data-site-design="classic"]`을 사용합니다.

이 SHA에서는 marker가 붙은 DOM element 자체가 바뀌지 않습니다. root `<main>`이 이미 header, route children, footer를 모두 감싸므로 기존 selector도 그 descendants에 적용됐습니다. 실제 변화는 selector의 의미를 “home template인 main”에서 “site design boundary”로 일반화한 것입니다. 후속 `b957...`이 `data-site-design`을 outer `<div>`로 옮기고, `data-home-template`을 route body만 감싸는 inner `<main id="main-content">`에 남기면서 두 책임이 물리적으로 분리됩니다.

중요한 chronology는 이 marker 의미의 분리가 `7f77...`의 mobile navigation과 `b957...`의 dedicated switcher integration보다 먼저라는 점입니다. 후속 outer-wrapper 구조를 이 commit에 역투영하지 않았습니다.

### 5. `7f77ec1912d6` — feat(shell): 현재 navigation 상태와 모바일 메뉴 추가

- Importance: **B**
- Tags: `ROUTING`
- Fixed role: Make global navigation route-aware and add a keyboard-operable mobile menu.

#### Historical inspection tasks

- `isCurrentNavigation`의 root exact match와 비-root prefix match를 구체적인 path 예로 검증합니다.
- desktop/mobile 링크에 동일한 state-preserving href와 `aria-current` 정책이 적용되는지 확인합니다.
- native `<details>/<summary>` mobile menu가 open 상태를 browser에 맡기며 별도 React state를 두지 않는지 확인합니다.
- 이 commit에서 inline template switcher가 header에서 제거되는 변경도 thread 흐름에 연결합니다.

#### Reconstruction result

**route-aware 판정**

새 `isCurrentNavigation(href, currentPath)`는 `/`를 exact match로만 취급합니다. 그렇지 않으면 `currentPath === href` 또는 `currentPath.startsWith(`${href}/`)`를 사용합니다. 이 예외가 없으면 모든 path가 `/`로 시작하므로 Home이 항상 active가 됩니다.

**desktop/mobile 공통 계약**

- desktop과 mobile 모두 `getTemplateHref`로 design/debug 상태를 보존합니다.
- 같은 active 판정 결과로 `aria-current="page"`를 설정합니다.
- mobile은 React boolean state 대신 native `<details><summary>`를 사용합니다.
- browser가 open/close 및 keyboard disclosure semantics를 소유합니다.

동시에 이전 inline template switcher가 header에서 제거됩니다. 이 시점에는 dedicated `DesignSwitcher`가 아직 shell에 들어오지 않았으므로 design selection UI가 일시적으로 별도 통합을 기다리는 상태입니다.

native menu에 explicit close handler는 추가되지 않았습니다. route navigation이 발생하면 page transition이 disclosure lifetime을 끝내지만, same-page prevented navigation에서 닫힘을 보장하는 component state machine은 이 commit의 범위가 아닙니다.

### 6. `b9571c485013` — feat(shell): 디자인 선택기를 공용 shell에 연결

- Importance: **A**
- Tags: `ARCH, ROUTING`
- Fixed role: Integrate the design switcher into the shared site header and make shell-level interface text come from presentation content.

#### Historical inspection tasks

- `PageShell`, `SiteHeader`, `DesignSwitcher`의 새 조합 경로와 `ui` prop 전파를 추적합니다.
- skip link의 href와 `<main id="main-content">` landmark가 어떻게 맞물리는지 확인합니다.
- `data-site-design` outer wrapper와 `data-home-template` main landmark가 분리된 이유와 selector 범위를 적습니다.
- primary/mobile/design navigation의 label이 hard-coded string에서 presentation content로 이동한 ownership 변화를 설명합니다.
- 이 commit이 switcher 자체의 hydration lifecycle을 해결하지는 않으며 별도 Thread가 이어진다는 경계를 적습니다.

#### Reconstruction result

**공용 shell 완성**

`DesignSwitcher`가 `SiteHeader`에 연결되고, presentation의 공용 UI copy가 `ui` prop으로 shell까지 전달됩니다. primary/mobile/design navigation label과 menu text가 hard-coded component text가 아니라 validated presentation content의 책임이 됩니다.

markup 경계도 정리됩니다.

- outer wrapper: `data-site-design={homeTemplate}` — header, main, footer를 포함하는 full-site design scope
- main: `id="main-content"`와 `data-home-template={homeTemplate}` — skip target이자 page content landmark
- skip link: `href="#main-content"`

이 구조는 visual design scope와 semantic main landmark를 같은 element에 강제로 결합하지 않습니다.

**ownership**

`PageShell`이 shell labels, design scope, landmark, header/footer composition을 소유합니다. 각 route는 active template/debug/current path를 준비해 넘깁니다. `DesignSwitcher` 내부의 disclosure/focus/hydration 문제는 여기서 해결되지 않으며 다음 Thread의 책임입니다.

이 commit에는 이 구조를 새로 실행 검증하는 test가 포함되지 않았습니다. exact diff로 composition과 prop flow를 확인했습니다.

## 6. Invariant evolution

Record where each invariant was introduced, extended, shown insufficient, corrected, and verified.

| 단계 | 추가된 invariant |
| --- | --- |
| `e936...` | persistent header와 content-driven primary navigation |
| `a2fd...` | shell internal links의 design/debug 상태 보존 |
| `8e114...` | 하나의 root main 안에 header/route children/footer를 조합하는 `PageShell` 정의 |
| `51806...` | 동일 root에 site-design marker를 추가해 home-template selector와 의미를 분리 |
| `7f77...` | route-aware active state와 native mobile disclosure |
| `b957...` | content-owned labels, skip/main landmark, dedicated design switcher integration |

최종 shell은 state-preserving navigation, active-route semantics, mobile disclosure, full-site design marker, skip target을 공통으로 제공합니다.

## 7. Failure → Fix → Test relationships

Connect fixes and tests to the exact earlier assumption or implementation they correct or verify.

- **초기 부족 → 확장:** 단순 header는 query state와 active/mobile semantics가 없었습니다. `a2fd...`, `7f77...`이 각각 상태와 route-aware responsive behavior를 추가했습니다.
- **marker coupling → correction:** `51806...`은 동일 root에서 design marker를 `data-home-template`과 분리했고, `b957...`은 이를 outer wrapper와 inner main landmark로 물리적으로 분리했습니다.
- **integration gap → completion:** `7f77...`에서 inline switcher를 제거한 뒤 `b957...`이 dedicated `DesignSwitcher`를 공용 header에 연결했습니다.
- 이 Thread에는 직접 test commit이 없습니다. 후속 route/component characterization은 다른 Thread/카테고리에 있으며 실행 결과를 소급하지 않았습니다.

## 8. Ownership, state, and responsibility changes

Track caller/callee ownership, browser/framework state, resource lifetime, and boundaries that deliberately remain outside the Thread.

| 책임 | 최종 소유자 |
| --- | --- |
| header/footer/main composition | `PageShell` |
| primary/mobile navigation source | `site.navigation` |
| shell UI/accessibility copy | `content.presentation.ui` |
| active route 판정 | `isCurrentNavigation` |
| mobile open state | native `<details>`/browser |
| design selection interaction | `DesignSwitcher` |
| page body | route renderer |
| actual history transition | Next.js/browser |

## 9. Final Thread state

- 공용 shell의 모든 주요 internal navigation surface는 design/debug state를 보존합니다.
- desktop/mobile은 동일한 active route 정책을 사용합니다.
- Home은 exact match만 active이고 section routes는 child detail path에서도 active입니다.
- full-site design marker가 header부터 footer까지 적용되며 main landmark는 별도 유지됩니다.
- skip link와 content-owned accessibility labels가 shell contract에 포함됩니다.

## 10. Final architecture or execution flow

1. route가 content, active template, debug, current path를 준비합니다.
2. `PageShell`이 outer design wrapper와 skip link를 만듭니다.
3. `SiteHeader`가 brand, desktop navigation, native mobile navigation, design switcher를 렌더링합니다.
4. 각 internal href는 query-state helper를 통과하고 active predicate로 `aria-current`를 얻습니다.
5. main landmark가 route body를 감싸고 `SiteFooter`가 동일한 state-preserving home path를 제공합니다.

## 11. Learning-completion checks

- [x] chronology 오류였던 `51806...` 위치를 바로잡았습니다.
- [x] 각 commit 시점에 아직 없는 mobile/switcher behavior를 구분했습니다.
- [x] route active 판정의 root 예외를 설명했습니다.
- [x] shell과 route renderer ownership을 분리했습니다.
- [x] 실행하지 않은 test 결과를 주장하지 않았습니다.
===== END FILE: 02-shared-shell-navigation-and-mobile-menu.md =====

===== BEGIN FILE: 03-native-design-switcher-page-lifecycle.md =====
# Development Thread 03 — Native Design Switcher Page Lifecycle

## 0. Source and frozen audit scope

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/portfolio`
- Category: `02-routing-navigation-and-page-lifecycle`
- Change range for directed inspection: `e43e8addd7f3^..1ac7813155c6`
- Classification source: `commit/commit-importance.md` on `web/portfolio`
- Historical rule: inspect every commit at its exact SHA; do not project later code backward.
- Phase 1 status: audited and frozen before learner-facing completion.
- Thread boundary: design 선택 navigation의 disclosure state, focus, hydration, server/client ownership을 다룹니다. visual styling 자체는 Category 05, generic component-test strategy는 Category 07의 범위입니다.
- Execution note: source inspection and runtime command evidence must be recorded separately.

### Phase 1 audit decisions

- 카테고리 초안에 빠져 있던 branch source의 핵심 native design-switcher Thread를 추가했습니다.
- source-defined cross-cutting Thread 순서와 exact importance/tag를 그대로 사용했습니다.

## 1. Learning goal and planned invariants

client-owned native disclosure에서 시작해 hydration race를 재현하고, static selector markup을 server ownership으로 되돌린 뒤 최소 close action만 client island로 남기는 과정을 재구성합니다.

- native `<details>`가 open state를 소유하며 explicit close는 `open` 제거 후 summary focus를 복원합니다.
- design link는 current route와 query state를 보존하고 active item은 `aria-current="page"`를 가집니다.
- hydration 전 browser가 변경한 native open state는 합법적인 상태로 취급됩니다.
- 최종 selector static markup은 server component이고 imperative close만 client component입니다.
- source-level regression은 selector에 top-level `"use client"`나 `useRef`가 재도입되는 것을 막습니다.

## 2. Questions to answer

- 초기 `useRef` ownership이 왜 native browser state와 hydration 시점에 충돌할 수 있습니까?
- explicit close와 design-link navigation close의 focus semantics 차이는 무엇입니까?
- `suppressHydrationWarning`이 무엇을 고치고 무엇을 직접 고치지 않습니까?
- deterministic hydration test는 browser mutation을 어떤 순서로 주입합니까?
- server-first refactor 후 client boundary가 남아야 하는 최소 이유는 무엇입니까?

## 3. Completion criteria

- 구현 → interaction test → hydration fix → race regression → server refactor → structural regression 순서를 설명합니다.
- fix commit을 독립 기능처럼 서술하지 않고 이전 assumption과 연결합니다.
- test technique, production path, assertion, 비보장을 구분합니다.
- server/client ownership과 focus lifetime을 명시합니다.
- mixed-scope `09cec...`에서 이 Thread와 직접 관련된 evidence를 분리합니다.

## 4. Fixed commit map

| # | Commit | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `e43e8addd7f3` | feat(designs): 디자인 선택기 상태와 trigger 추가 | B | RENDERER | Introduce the client-side state and trigger for a route-preserving design switcher. |
| 2 | `c69ef85c98b2` | feat(designs): 디자인 선택 목록과 닫기 동작 추가 | A | ARCH, RENDERER | Complete the design-switcher sheet with an ordered list of registered designs, active-state semantics, palette previews, and explicit closing behavior. |
| 3 | `09cec616f314` | test(ui): 디자인 선택과 프로젝트 링크 계약 검증 | A | VALIDATION, TEST | Lock down the interaction contracts of the design selector and project-link components. |
| 4 | `c702b870d57a` | fix(ui): hydration 중 native details 상태 보존 | A | DEBUG | Mark the native design-switcher `details` element as an intentional hydration boundary. |
| 5 | `b6c0238ab8b8` | test(ui): details hydration 경쟁 조건 검증 | A | VALIDATION, TEST | Reproduce the design-switcher hydration race directly and lock down the intended invariant. |
| 6 | `a37cb8596733` | refactor(ui): 디자인 선택기를 server markup으로 전환 | A | ARCH, REFACTOR | Convert the design selector itself from a client component to server-rendered markup and isolate the only imperative behavior in a small `DesignSwitcherClose` client component. |
| 7 | `1ac7813155c6` | test(ui): server 선택기와 focus 복원 검증 | A | VALIDATION, A11Y, TEST | Add a structural regression check that reads the design-switcher source and rejects either a top-level `"use client"` directive or `useRef` state in the selector component. |

The commit SHA, order, subject, importance, tags, and source-defined role in this map are frozen.

## 5. Commit-by-commit reconstruction

### 1. `e43e8addd7f3` — feat(designs): 디자인 선택기 상태와 trigger 추가

- Importance: **B**
- Tags: `RENDERER`
- Fixed role: Introduce the client-side state and trigger for a route-preserving design switcher.

#### Historical inspection tasks

- `src/components/portfolio/design-switcher.tsx`의 top-level `"use client"`, `useRef` 두 개, native `<details>/<summary>`를 확인합니다.
- `SITE_DESIGNS`, content template copy, active label, two-digit count label의 계산 경로를 추적합니다.
- 이 SHA에서는 trigger만 있고 선택 panel과 explicit close가 아직 없다는 범위를 적습니다.

#### Reconstruction result

`src/components/portfolio/design-switcher.tsx`가 top-level `"use client"` component로 도입됩니다. component는 native `<details>`와 `<summary>`를 사용하면서도 두 DOM ref를 보유합니다. 하나는 details open 상태를 조작하기 위한 ref이고, 다른 하나는 close 후 focus를 돌려줄 summary ref입니다.

이 commit에서는 trigger skeleton만 구성됩니다. `SITE_DESIGNS`와 content-owned template metadata를 결합해 active design을 찾고, 없으면 registry 첫 항목으로 fallback합니다. 현재 index와 전체 개수는 두 자리 문자열로 표시됩니다. 그러나 선택 panel, design link list, explicit close button은 아직 없습니다.

따라서 browser-native disclosure를 선택한 architecture와 client hydration boundary가 먼저 생긴 상태입니다. 아직 imperative ref가 실제 close 동작에 사용되지는 않지만, 후속 lifecycle ownership의 토대가 됩니다.

### 2. `c69ef85c98b2` — feat(designs): 디자인 선택 목록과 닫기 동작 추가

- Importance: **A**
- Tags: `ARCH, RENDERER`
- Fixed role: Complete the design-switcher sheet with an ordered list of registered designs, active-state semantics, palette previews, and explicit closing behavior.

#### Historical inspection tasks

- `SITE_DESIGNS.map`으로 만든 link list, palette swatch, `aria-current`, `getTemplateHref` 호출을 확인합니다.
- close button이 `detailsRef.current?.removeAttribute("open")` 후 `summaryRef.current?.focus()`를 호출하는 순서를 추적합니다.
- design link의 `onClick`도 disclosure를 닫지만 focus를 복원하지 않는 차이를 적습니다.
- native open 상태와 imperative refs를 동시에 쓰는 ownership 구조가 이후 hydration race의 전제가 되는지 설명합니다.

#### Reconstruction result

**완성된 interaction**

선택 panel이 native details 안에 추가되고 `SITE_DESIGNS` 순서대로 각 design을 렌더링합니다. 각 item은 content template에서 label/description을 얻고 registry palette preview를 표시하며, active ID이면 `aria-current="page"`를 갖습니다. href는 현재 path와 debug state를 `getTemplateHref`로 운반합니다.

explicit close button의 실행 순서는 다음과 같습니다.

```ts
detailsRef.current?.removeAttribute("open");
summaryRef.current?.focus();
```

즉 disclosure를 닫은 뒤 keyboard focus를 trigger로 복원합니다. design link click handler는 details의 `open`만 제거하고 focus는 이동시키지 않습니다. 정상 navigation이면 새 page로 넘어가므로 explicit close와 같은 focus restoration이 필요하지 않다는 전제입니다.

**상태 소유**

- open 상태: native details DOM
- imperative close/focus: hydrated client component의 refs
- target URL: helper가 생성한 link href
- active semantics: `aria-current`
- actual route transition: Next `Link`/browser

이 조합은 native state와 React hydration이 같은 DOM을 공유합니다. 사용자가 hydration 전에 summary를 열면 server markup의 closed 상태, browser의 open state, React 첫 render 사이에 경쟁이 생길 수 있습니다. 이 risk는 아직 이 commit에서 다루지 않습니다.

### 3. `09cec616f314` — test(ui): 디자인 선택과 프로젝트 링크 계약 검증

- Importance: **A**
- Tags: `VALIDATION, TEST`
- Fixed role: Lock down the interaction contracts of the design selector and project-link components.

#### Historical inspection tasks

- `src/components/portfolio/design-switcher.test.tsx`에서 native `open` attribute를 직접 설정한 뒤 close button과 design link click을 검증하는 방법을 확인합니다.
- close button이 open 제거와 summary focus를 모두 보장하고, link click은 preventDefault 상황에서도 open을 제거하는지 assertion을 적습니다.
- 같은 commit의 `project-links.test.tsx`가 내부 case-study URL에 `view`와 `debug`를 전파하는 mixed-scope evidence임을 구분합니다.
- 이 테스트가 hydration 전 browser mutation은 재현하지 않는다는 비보장을 적습니다.

#### Reconstruction result

**design-switcher test**

`src/components/portfolio/design-switcher.test.tsx`는 component를 render한 뒤 DOM의 native state를 직접 조작합니다.

- details에 `open`을 설정하고 explicit close button을 click합니다.
- `open` attribute가 제거되는지 확인합니다.
- `document.activeElement`가 summary인지 확인해 focus restoration을 검증합니다.
- design link click에는 navigation을 막는 handler를 둔 상태에서도 disclosure가 닫히는지 확인합니다.
- content-owned trigger copy와 hidden navigation semantics도 query합니다.

이 검증은 interaction regression입니다. production path는 `DesignSwitcher`의 close handler와 link `onClick`입니다. focus를 단순 class나 snapshot이 아니라 실제 active element로 확인한다는 점이 중요합니다.

**mixed-scope evidence**

같은 commit의 `src/components/portfolio/project-links.test.tsx`는 internal case-study link가 `/projects/sample-project?view=classic&debug=content`를 얻고 external link가 `_blank`/`noreferrer`를 갖는지 검증합니다. 이 Thread에서는 switcher와 “route state를 실제 link consumer까지 전달한다”는 integration 근거만 사용합니다. project-link placement 전체는 별도 component story입니다.

**비보장**

테스트는 이미 hydrated component에서 open attribute를 바꿉니다. server string을 만든 뒤 hydration 전에 browser가 open 상태를 변경하는 경쟁 조건은 재현하지 않습니다. 실제 browser history나 page transition도 실행하지 않습니다. 이 환경에서는 source만 inspection했고 test command는 실행하지 않았습니다.

### 4. `c702b870d57a` — fix(ui): hydration 중 native details 상태 보존

- Importance: **A**
- Tags: `DEBUG`
- Fixed role: Mark the native design-switcher `details` element as an intentional hydration boundary.

#### Historical inspection tasks

- 이전 가정(서버와 첫 client render의 `<details>` 상태가 같음)과 browser가 hydration 전에 native disclosure를 열 수 있는 실제 경쟁 조건을 분리합니다.
- `src/components/portfolio/design-switcher.tsx`에서 `suppressHydrationWarning`이 정확히 어느 node에 추가되는지 확인합니다.
- 이 변경이 warning 정책을 수정하는 것과 open 상태 보존을 코드로 강제하는 것은 다르다는 점을 적습니다.
- 후속 deterministic test가 필요했던 이유를 연결합니다.

#### Reconstruction result

**이전 가정**

server는 closed `<details>` markup을 보내고, 첫 client render도 closed일 것이라는 암묵적 가정이 있었습니다. 그러나 native `<summary>`는 JavaScript hydration이 끝나기 전에도 사용자가 조작할 수 있습니다. 그 사이 browser가 `open`을 추가하면 React가 hydrate할 DOM과 server expectation이 달라집니다.

**실제 risk**

- React hydration mismatch warning
- React reconciliation이 browser가 이미 만든 legitimate native state를 다르게 취급할 가능성
- 개발 환경에서 noisy error가 실제 defect와 섞임

**수정**

`src/components/portfolio/design-switcher.tsx`의 native `<details>` node에 `suppressHydrationWarning`을 추가합니다. correction의 의미는 “이 node의 pre-hydration native state 차이는 의도된 경계”라고 React에 알리는 것입니다.

**정확한 보장 범위**

이 prop은 mismatch warning 정책을 바꿉니다. open 상태를 별도 React state로 복사하거나 강제로 재설정하는 코드는 추가하지 않습니다. 따라서 “open이 실제로 보존된다”는 동작 증거는 이 fix diff만으로 완성되지 않습니다. 후속 `b6c0238ab8b8`이 race를 deterministic하게 재현해 그 invariant를 확인합니다.

이 commit은 새 기능이 아니라 browser-native state를 합법적인 hydration 입력으로 인정하도록 초기 assumption을 수정한 root-cause fix입니다.

### 5. `b6c0238ab8b8` — test(ui): details hydration 경쟁 조건 검증

- Importance: **A**
- Tags: `VALIDATION, TEST`
- Fixed role: Reproduce the design-switcher hydration race directly and lock down the intended invariant.

#### Historical inspection tasks

- `renderToString` → container `innerHTML` → hydration 전 `details.open = true` → `hydrateRoot` 순서를 정확히 재구성합니다.
- `console.error` spy가 hydration mismatch 메시지를 수집하는 방법과 cleanup에서 root unmount, spy restore, container remove를 수행하는지 확인합니다.
- assertion이 open state 보존과 hydration error 부재를 각각 어떻게 검증하는지 적습니다.
- 실제 browser navigation이나 모든 React version을 증명하지 않는 deterministic component regression의 경계를 적습니다.

#### Reconstruction result

**주입 방식**

테스트는 별도 mocking framework로 React internals를 조작하지 않습니다. 실제 server rendering과 hydration API를 순서대로 사용해 경쟁 조건을 만듭니다.

1. `renderToString(<DesignSwitcher ... />)`로 server HTML을 생성합니다.
2. 임시 container의 `innerHTML`에 server markup을 넣습니다.
3. hydration 전에 실제 `<details>`를 찾아 `details.open = true`로 browser-side mutation을 주입합니다.
4. `hydrateRoot(container, <DesignSwitcher ... />)`를 호출합니다.
5. hydration이 끝난 뒤 details가 계속 open인지 확인합니다.
6. `console.error` spy에서 hydration mismatch error가 기록되지 않았는지 확인합니다.

**관찰과 cleanup**

테스트는 open boolean과 console error라는 두 상태를 봅니다. `finally` 경로에서 hydrated root를 unmount하고 spy를 restore하며 임시 container를 제거합니다. 따라서 후속 테스트에 React root, patched console, detached DOM을 남기지 않습니다.

**분류**

이것은 deterministic hydration-race regression입니다. production component와 React server/client API를 사용하되 full browser page를 띄우지는 않습니다.

**증명**

- hydration 전에 native details를 연 상태가 hydration 뒤에도 유지됩니다.
- 이 fixture에서 hydration mismatch error가 발생하지 않습니다.

**비증명**

- 모든 browser/React version의 timing
- 실제 pointer/keyboard event가 hydration보다 먼저 발생하는 e2e 상황
- route navigation 후 disclosure teardown
- 성능 또는 client bundle cost

이 환경에서는 해당 command를 실행하지 않았으므로 위 내용은 exact test mechanism과 assertion inspection 결과입니다.

### 6. `a37cb8596733` — refactor(ui): 디자인 선택기를 server markup으로 전환

- Importance: **A**
- Tags: `ARCH, REFACTOR`
- Fixed role: Convert the design selector itself from a client component to server-rendered markup and isolate the only imperative behavior in a small `DesignSwitcherClose` client component.

#### Historical inspection tasks

- `DesignSwitcher`에서 `"use client"`, `useRef`, ref props가 제거되고 새 `design-switcher-close.tsx`만 client boundary로 남는지 확인합니다.
- `DesignSwitcherClose`가 event target에서 `closest("details")`와 `:scope > summary`를 찾아 open 제거와 focus 복원을 수행하는지 추적합니다.
- design link의 `onClick` close가 제거되고 URL navigation에 전환 종료를 맡기는 ownership 변화를 적습니다.
- `suppressHydrationWarning`이 남는지 확인하고, static markup server ownership과 작은 imperative island의 경계를 설명합니다.
- 기존 interaction test가 어떤 assertion을 버리고 어떤 href/focus assertion을 유지했는지 비교합니다.

#### Reconstruction result

**문제 재정의**

`suppressHydrationWarning`과 regression test로 race를 다룰 수 있었지만, selector의 대부분은 static link markup입니다. 전체 component를 client boundary로 유지하면 native disclosure를 쓰면서도 모든 목록 markup과 refs를 hydrate해야 합니다. root cause를 더 좁히면 client JavaScript가 필요한 동작은 explicit close 후 focus restoration뿐입니다.

**구조 변경**

- `DesignSwitcher`에서 top-level `"use client"`, `useRef`, ref attachment를 제거합니다.
- design list, summary, details, links는 server-rendered markup이 됩니다.
- 새 `src/components/portfolio/design-switcher-close.tsx`만 client component입니다.
- close button은 click event의 current target에서 `closest("details")`를 찾고, direct summary를 `:scope > summary`로 찾습니다.
- details의 `open`을 제거한 뒤 summary가 있으면 focus합니다.
- design link의 `onClick` close handler는 제거됩니다. 실제 navigation이 disclosure의 현재 page lifetime을 끝냅니다.
- native state 차이를 허용하는 `suppressHydrationWarning`은 details에 남습니다.

**ownership 이동**

| 이전 | 이후 |
| --- | --- |
| 전체 selector markup과 refs가 client | static selector markup은 server |
| DOM refs가 component lifetime에 고정 | close button이 click 시 local DOM ancestry를 찾음 |
| link click도 imperative close | navigation에 page transition 책임 |
| explicit close/focus와 static list가 같은 boundary | explicit close만 작은 client island |

기존 test도 구조에 맞게 바뀝니다. link click close assertion은 제거되고, href 생성과 explicit close/focus 계약은 유지됩니다.

이 refactor는 behavior를 확장하기보다 hydration 대상과 client ownership을 최소화합니다. `DesignSwitcherClose`가 없는 pure server component로 만들 수 없는 이유는 explicit close button이 현재 DOM을 닫고 summary에 focus를 돌리는 imperative behavior 때문입니다.

### 7. `1ac7813155c6` — test(ui): server 선택기와 focus 복원 검증

- Importance: **A**
- Tags: `VALIDATION, A11Y, TEST`
- Fixed role: Add a structural regression check that reads the design-switcher source and rejects either a top-level `"use client"` directive or `useRef` state in the selector component.

#### Historical inspection tasks

- `readFile`과 `path.join(process.cwd(), ...)`으로 production source를 읽는 structural test를 확인합니다.
- `"use client"`와 `useRef` 문자열 부재 assertion이 어떤 architecture regression을 막는지 적습니다.
- 기존 hydration test와 explicit close focus test가 계속 남아 있는지 확인합니다.
- source-string test가 bundle 크기, RSC payload, arbitrary client import까지 증명하지 않는 한계를 구분합니다.

#### Reconstruction result

**새 regression contract**

test는 runtime render 결과만 보지 않고 production source를 직접 읽습니다. `process.cwd()`에서 `src/components/portfolio/design-switcher.tsx` 경로를 만들고 `readFile`로 문자열을 가져온 뒤 다음을 거부합니다.

- top-level/client directive 문자열 `"use client"`
- `useRef`

이 structural assertion은 “selector 본체가 다시 hydrated client component가 되지 않는다”는 architecture invariant를 보호합니다. explicit close/focus test와 hydration race test도 남아 있어 동작과 boundary를 서로 다른 방식으로 검증합니다.

**접근성 관계**

`DesignSwitcherClose`가 details를 닫은 뒤 summary focus를 복원하는 assertion은 keyboard user의 위치를 잃지 않는 계약을 계속 보호합니다. source boundary test만 통과하고 focus test가 깨질 수 있으므로 두 검증은 대체 관계가 아닙니다.

**한계**

문자열 부재 검사는 source-level guard입니다. 다른 client-only import를 간접적으로 추가하는 모든 경우, 실제 emitted bundle 크기, RSC payload, framework compiler 결과를 증명하지 않습니다. 또한 source formatting에 의존하는 측면이 있습니다. 그럼에도 “이 파일에 client directive/ref state를 되돌리는 단순 회귀”에는 매우 직접적입니다.

실행 결과는 기록하지 않았습니다. exact test source와 assertions를 inspection했습니다.

## 6. Invariant evolution

Record where each invariant was introduced, extended, shown insufficient, corrected, and verified.

| 상태 | SHA | invariant 변화 |
| --- | --- | --- |
| 도입 | `e43e8addd7f3` | native disclosure를 client wrapper로 시작 |
| 확장 | `c69ef85c98b2` | 선택 목록, active semantics, explicit close/focus |
| interaction 검증 | `09cec616f314` | close와 focus, link-driven close를 component test로 고정 |
| 부족 확인/수정 | `c702b870d57a` | pre-hydration native state 차이를 legitimate boundary로 인정 |
| 결정적 재현 | `b6c0238ab8b8` | server→browser mutation→hydrate race를 직접 주입 |
| ownership 축소 | `a37cb8596733` | static markup server, close action만 client |
| 구조 보호 | `1ac7813155c6` | selector 본체의 client/ref 회귀를 source test로 차단 |

최종 invariant는 “browser가 native disclosure state를 소유하고, server가 static selector markup을 소유하며, explicit close/focus만 최소 client island가 소유한다”입니다.

## 7. Failure → Fix → Test relationships

Connect fixes and tests to the exact earlier assumption or implementation they correct or verify.

- **Implementation → Test:** `c69ef...`의 explicit close/focus와 link close를 `09cec...`이 검증합니다.
- **Assumption → Failure risk:** closed server markup과 첫 client render가 같다는 가정이 hydration 전 native click으로 깨질 수 있습니다.
- **Fix → Deterministic test:** `c702...`가 details를 intentional hydration boundary로 표시하고 `b6c...`가 open mutation을 실제 hydration 전에 주입합니다.
- **Fix의 추가 축소:** warning suppression에 머물지 않고 `a37...`이 selector 대부분을 server로 옮겨 hydrated surface 자체를 줄입니다.
- **Architecture regression protection:** `1ac...`이 top-level client directive와 `useRef` 재도입을 막습니다.

## 8. Ownership, state, and responsibility changes

Track caller/callee ownership, browser/framework state, resource lifetime, and boundaries that deliberately remain outside the Thread.

| 자원/상태 | 초기 ownership | 최종 ownership |
| --- | --- | --- |
| details open state | native DOM + hydrated wrapper가 함께 접근 | native DOM/browser |
| static trigger/list/link markup | client component | server component |
| explicit close | client component refs | `DesignSwitcherClose` client island |
| close 후 focus | summary ref | click 시 DOM ancestry에서 찾은 summary |
| design URL | `getTemplateHref` | server-rendered link helper 결과 |
| hydration mismatch policy | 없음 | details의 intentional warning suppression |

## 9. Final Thread state

- design selector 본체는 server-rendered native disclosure입니다.
- active design과 state-preserving href는 server markup에 포함됩니다.
- browser가 hydration 전에 disclosure를 열어도 그 native state를 legitimate하게 취급합니다.
- explicit close button만 client code를 사용하며 summary focus를 복원합니다.
- interaction, hydration race, source boundary가 서로 다른 test technique으로 보호됩니다.
- 실제 test command는 실행하지 않았습니다.

## 10. Final architecture or execution flow

1. server가 registry/content에서 active design과 link 목록을 계산해 `<details>` markup을 보냅니다.
2. browser가 native summary interaction으로 open state를 관리합니다.
3. hydration 전에 open state가 바뀌어도 details boundary는 그 차이를 허용합니다.
4. design link 선택은 server-generated current-route URL로 navigation합니다.
5. 사용자가 explicit close를 누르면 작은 client component가 nearest details를 닫고 direct summary에 focus를 돌립니다.

## 11. Learning-completion checks

- [x] 초기 client implementation과 final server-first structure를 구분했습니다.
- [x] hydration fix를 이전 assumption/risk와 연결했습니다.
- [x] deterministic race 주입 순서와 cleanup을 설명했습니다.
- [x] interaction, hydration, structural test의 증명 범위를 각각 적었습니다.
- [x] 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
===== END FILE: 03-native-design-switcher-page-lifecycle.md =====

===== BEGIN FILE: 04-project-index-and-dynamic-detail-lifecycle.md =====
# Development Thread 04 — Project Index and Dynamic Detail Lifecycle

## 0. Source and frozen audit scope

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/portfolio`
- Category: `02-routing-navigation-and-page-lifecycle`
- Change range for directed inspection: `7571f1400065^..d4c7f742fb4d`
- Classification source: `commit/commit-importance.md` on `web/portfolio`
- Historical rule: inspect every commit at its exact SHA; do not project later code backward.
- Phase 1 status: audited and frozen before learner-facing completion.
- Thread boundary: App Router가 project index와 dynamic detail addressability, params, unknown-project failure를 소유하는 부분만 다룹니다. page enablement 및 global 404 presentation은 별도 Thread로 분리합니다.
- Execution note: source inspection and runtime command evidence must be recorded separately.

### Phase 1 audit decisions

- `50199...`와 custom 404 commit을 제거해 route creation과 availability/recovery를 독립 story로 분리했습니다.

## 1. Learning goal and planned invariants

`/projects` index와 `/projects/[projectId]` detail이 content/query/shell 계약에 연결되고, static params와 runtime `notFound()`가 서로 다른 책임을 가지는 과정을 재구성합니다.

- project index는 aggregated enabled project set을 partition/group하고 template/debug state를 renderer로 전달합니다.
- dynamic detail static params는 canonical project IDs에서 생성됩니다.
- runtime unknown project ID는 `getProjectById` 실패 후 `notFound()`로 종료됩니다.
- static generation 목록과 runtime failure path는 서로 대체하지 않습니다.

## 2. Questions to answer

- index route가 content derivation과 renderer dispatch를 어느 경계에서 수행합니까?
- `generateStaticParams`와 `notFound()`가 각각 어떤 요청 집합을 다룹니까?
- invalid project ID에서 query resolution과 lookup의 실제 순서는 무엇입니까?
- page enablement가 이 Thread에서 제외되어야 하는 이유는 무엇입니까?

## 3. Completion criteria

- 두 route-introduction commit만 chronology대로 설명합니다.
- 초기 index/detail code에 존재하는 로직만 사용합니다.
- static build-time enumeration과 runtime missing-ID handling을 분리합니다.
- 후속 page gate 또는 renderer architecture를 이 시점의 구현으로 오인하지 않습니다.

## 4. Fixed commit map

| # | Commit | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7571f1400065` | feat(projects): 프로젝트 목록 route 연결 | B | ROUTING, RENDERER | Connect `/projects` to the aggregated content and query-state model. |
| 2 | `d4c7f742fb4d` | feat(project): 프로젝트 상세 route 연결 | B | ROUTING, RENDERER | Add the dynamic project-detail route and generate static parameters from the enabled project collection. |

The commit SHA, order, subject, importance, tags, and source-defined role in this map are frozen.

## 5. Commit-by-commit reconstruction

### 1. `7571f1400065` — feat(projects): 프로젝트 목록 route 연결

- Importance: **B**
- Tags: `ROUTING, RENDERER`
- Fixed role: Connect `/projects` to the aggregated content and query-state model.

#### Historical inspection tasks

- `src/app/projects/page.tsx`가 content, search params, template/debug, featured/archive partition, grouping, metrics를 어떤 순서로 준비하는지 확인합니다.
- classic/design renderer 분기와 공통 `shellProps.currentPath = "/projects"`를 추적합니다.
- `src/content/site.json`의 Projects navigation 추가가 route addressability와 discovery를 동시에 연결하는지 확인합니다.

#### Reconstruction result

`src/app/projects/page.tsx`가 `/projects`의 App Router entry로 추가됩니다. route는 먼저 portfolio aggregate와 optional search params를 읽고 `view`/`debug`를 정규화합니다. 이어 presentation copy와 project collection을 사용해 featured project와 나머지 archive, configured group, route-level metric count를 준비합니다.

active template에 따라 Classic 또는 Design project-index view로 분기하지만 두 경로는 같은 content와 query-state 결과를 공유합니다. `shellProps.templateSwitcher.currentPath`는 `/projects`로 고정되어 design switch가 현재 route를 유지합니다. 같은 commit에서 `src/content/site.json`에 Projects navigation이 추가되어 addressable route가 공용 navigation에서도 발견됩니다.

이 시점의 route는 page availability flag를 검사하지 않습니다. 또한 later full-site renderer registry나 route view model을 아직 사용하지 않습니다. 이 commit의 역할은 기존 aggregate/query/shell 계약에 project index를 처음 연결하는 것입니다.

### 2. `d4c7f742fb4d` — feat(project): 프로젝트 상세 route 연결

- Importance: **B**
- Tags: `ROUTING, RENDERER`
- Fixed role: Add the dynamic project-detail route and generate static parameters from the enabled project collection.

#### Historical inspection tasks

- `generateStaticParams`가 aggregated `content.projects`의 ID를 반환하는지 확인합니다.
- `params`/`searchParams` await, template/debug resolution, `getProjectById`, `notFound()`의 실제 순서를 적습니다.
- 유효한 project일 때 `currentPath`와 detail view props가 canonical project record에서 만들어지는지 확인합니다.
- static params에 없는 runtime request와 unknown ID가 어떻게 recover되는지, page enablement는 아직 이 Thread 범위가 아님을 구분합니다.

#### Reconstruction result

**build-time enumeration**

`generateStaticParams()`는 `getPortfolioContent().projects`를 순회해 `{ projectId }` 배열을 반환합니다. 이 aggregate는 앞선 loader에서 disabled project를 걸러낸 상태이므로 source-defined role의 “enabled project collection”과 일치합니다.

**runtime page 순서**

1. content를 읽습니다.
2. dynamic `params`를 await해 `projectId`를 얻습니다.
3. optional `searchParams`를 await합니다.
4. template/debug를 resolve합니다.
5. `getProjectById(projectId, content)`를 호출합니다.
6. 결과가 없으면 `notFound()`로 종료합니다.
7. 성공 시 canonical project ID를 포함한 current path와 detail view props를 구성합니다.

static params는 known project pages를 미리 만들 수 있게 하지만 runtime에서 알려지지 않은 path가 절대 요청되지 않는다는 보장은 아닙니다. 그래서 `getProjectById` 실패 path가 별도로 필요합니다.

**보장과 경계**

- known project는 query-state-preserving shell 안에서 detail view로 연결됩니다.
- unknown project ID는 null dereference나 빈 page 대신 App Router not-found flow로 전환됩니다.
- 이 SHA의 invalid-ID path는 query parsing 뒤에 실행됩니다.
- `projects` page capability gate는 아직 없습니다. 그 정책은 `50199...`에서 별도 도입됩니다.

## 6. Invariant evolution

Record where each invariant was introduced, extended, shown insufficient, corrected, and verified.

| 단계 | invariant |
| --- | --- |
| `7571...` | `/projects`가 aggregate/query/shell contract에 연결되고 navigation에서 발견 가능 |
| `d4c7...` | canonical project IDs를 static params로 노출하며 runtime unknown ID는 `notFound()` |

이 Thread의 최종 invariant는 “project collection의 addressability는 index와 generated detail paths로 제공하되, runtime lookup 실패는 명시적 not-found lifecycle로 종료한다”입니다.

## 7. Failure → Fix → Test relationships

Connect fixes and tests to the exact earlier assumption or implementation they correct or verify.

- **Index → Detail:** index가 project collection과 internal detail navigation의 진입점을 제공하고 다음 commit이 `[projectId]` addressability를 완성합니다.
- **Enumeration → Recovery:** `generateStaticParams`는 build-time known set을 제공하지만 unknown runtime path를 처리하지 않으므로 `getProjectById` + `notFound()`가 별도로 존재합니다.
- **분리된 후속 정책:** `50199...`의 page capability gate와 `154b...`의 global 404 presentation은 이 route-creation Thread의 후속/인접 정책이지 초기 구현의 일부가 아닙니다.

## 8. Ownership, state, and responsibility changes

Track caller/callee ownership, browser/framework state, resource lifetime, and boundaries that deliberately remain outside the Thread.

| 책임 | 소유자 |
| --- | --- |
| project aggregate 및 enabled set | portfolio content layer |
| `/projects` route data 준비 | project index App Router page |
| dynamic ID extraction | App Router `params` |
| canonical lookup | `getProjectById` |
| missing ID transition | Next `notFound()` |
| page body presentation | Classic/Design route views |
| page capability | 이 Thread 이후 별도 route gate |

## 9. Final Thread state

- project index와 dynamic detail path가 모두 query-state-aware shell에 연결됩니다.
- known detail paths는 aggregate project IDs에서 생성됩니다.
- unknown ID는 explicit not-found path를 가집니다.
- static params와 runtime validation을 동일한 보장으로 혼동하지 않습니다.
- page enablement와 custom 404 UI는 별도 Thread에 있습니다.

## 10. Final architecture or execution flow

1. `/projects` 요청이 aggregate와 query state를 준비해 index renderer를 선택합니다.
2. 사용자가 project detail link로 이동합니다.
3. `[projectId]` page가 params와 query를 읽습니다.
4. canonical project lookup이 성공하면 current path와 detail props를 구성합니다.
5. lookup이 실패하면 `notFound()`가 global not-found lifecycle로 넘깁니다.

## 11. Learning-completion checks

- [x] 두 route-introduction commit만 유지했습니다.
- [x] static params와 runtime missing-ID handling을 구분했습니다.
- [x] `d4c7...`의 실제 query/lookup/notFound 순서를 적었습니다.
- [x] 후속 page gate와 404 UI를 초기 commit에 역투영하지 않았습니다.
- [x] runtime 실행 결과를 주장하지 않았습니다.
===== END FILE: 04-project-index-and-dynamic-detail-lifecycle.md =====

===== BEGIN FILE: 05-page-availability-and-auxiliary-route-lifecycle.md =====
# Development Thread 05 — Page Availability and Auxiliary Route Lifecycle

## 0. Source and frozen audit scope

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/portfolio`
- Category: `02-routing-navigation-and-page-lifecycle`
- Change range for directed inspection: `7fe913796acb^..50199be241c8`
- Classification source: `commit/commit-importance.md` on `web/portfolio`
- Historical rule: inspect every commit at its exact SHA; do not project later code backward.
- Phase 1 status: audited and frozen before learner-facing completion.
- Thread boundary: about/resume/contact/journey/interview-map 및 projects capability의 route-level enablement를 다룹니다. 각 페이지의 풍부한 화면 기능은 Category 04 범위입니다.
- Execution note: source inspection and runtime command evidence must be recorded separately.

### Phase 1 audit decisions

- 기존 project/auxiliary Thread에 중복 배치된 `50199...`를 단일 ownership Thread로 통합했습니다.
- route 생성 commit 중 availability lifecycle을 설명하는 최소 commit만 남기고 feature-section commit은 제외했습니다.

## 1. Learning goal and planned invariants

초기 항상-open 보조 route와 도입 시점부터 gated인 route의 비대칭을 확인하고, selector와 cross-route gate가 이를 하나의 page availability 정책으로 통합한 과정을 재구성합니다.

- page flag가 없거나 true이면 enabled이고 오직 explicit false만 disabled입니다.
- disabled route는 query parsing, renderer preparation, page-specific lookup 전에 `notFound()`로 종료됩니다.
- journey/interview-map은 도입 시부터 gated이며 기존 about/resume/contact/projects는 후속 commit에서 이관됩니다.
- project index와 detail은 동일한 `projects` capability를 공유합니다.
- route gate는 navigation discovery, sitemap, static params를 자동으로 변경하지 않습니다.

## 2. Questions to answer

- 왜 `!== false`가 default-open 정책입니까?
- selector 존재와 caller 적용 사이의 시간차가 어떤 비대칭을 만들었습니까?
- 도입 시 gated인 route와 retrofit된 route의 lifecycle 순서는 어떻게 다릅니까?
- `50199...`에서 gate가 가장 먼저 실행되는 것이 어떤 work를 방지합니까?
- route availability와 navigation/sitemap publication은 왜 별도 정책입니까?

## 3. Completion criteria

- 일곱 commit을 exact chronology로 설명합니다.
- 초기 route에 gate가 없었다는 사실을 final code로 덮지 않습니다.
- 각 page ID와 gate 위치를 구체적으로 적습니다.
- `50199...`이 하지 않은 작업을 명시합니다.
- runtime test 실행 여부와 source inspection을 구분합니다.

## 4. Fixed commit map

| # | Commit | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7fe913796acb` | feat(about): 프로필과 원칙 소개 추가 | B | RENDERER | Create the About route from profile and presentation content. |
| 2 | `e655951b0706` | feat(resume): 이력 소개와 요약 추가 | B | RENDERER | Create the Resume route with profile identity, presentation-owned hero copy, an optional download action, and structured summary paragraphs. |
| 3 | `bfcdf44eb34c` | feat(contact): 연락 페이지 소개 추가 | B | RENDERER | Create the Contact route with profile identity and contact-owned title and introductory text. |
| 4 | `3e2e95a3a28c` | feat(content): 페이지 활성화 selector 추가 | B | CONTENT, ROUTING | Add a typed selector for the site's optional page flags. |
| 5 | `0facaf123f29` | feat(journey): 여정 route 소개 추가 | B | ROUTING, RENDERER | Introduce the journey narrative as an optional route using the portfolio's standard page lifecycle. |
| 6 | `ddba753f7f51` | feat(interview-map): 근거 route 소개 추가 | B | ROUTING, RENDERER | Introduce the interview-evidence map as an optional, independently addressable route. |
| 7 | `50199be241c8` | feat(routes): 비활성 페이지 route 차단 | A | ARCH, ROUTING | Enforce page-enablement settings at the route boundary for about, contact, projects, project details, and resume. |

The commit SHA, order, subject, importance, tags, and source-defined role in this map are frozen.

## 5. Commit-by-commit reconstruction

### 1. `7fe913796acb` — feat(about): 프로필과 원칙 소개 추가

- Importance: **B**
- Tags: `RENDERER`
- Fixed role: Create the About route from profile and presentation content.

#### Historical inspection tasks

- `src/app/about/page.tsx`의 content load, query resolution, shell currentPath `/about`, profile/principles rendering을 확인합니다.
- 이 SHA에는 page enablement 검사와 `notFound()`가 없어 직접 접근이 항상 렌더링되는지 확인합니다.
- 이 commit에서는 route lifecycle만 다루고 후속 About feature sections는 다른 카테고리임을 구분합니다.

#### Reconstruction result

`src/app/about/page.tsx`가 처음 생기며 content를 읽고 optional `searchParams`를 await해 active template과 debug mode를 계산합니다. `PageShell`에는 current path `/about`과 같은 template/debug state가 전달되고, body는 profile identity와 presentation-owned principles를 렌더링합니다.

중요한 이전 상태는 page availability 검사 부재입니다. 이 SHA의 function에는 `isSitePageEnabled("about", ...)`나 `notFound()`가 없습니다. 따라서 content configuration에 page flag 개념이 나중에 생기더라도 이 시점의 About 직접 요청은 route 자체에서 차단되지 않습니다.

후속 profile photo, skills, experience, curation section은 page feature construction이며 이 Thread의 route lifecycle 핵심에서 제외했습니다.

### 2. `e655951b0706` — feat(resume): 이력 소개와 요약 추가

- Importance: **B**
- Tags: `RENDERER`
- Fixed role: Create the Resume route with profile identity, presentation-owned hero copy, an optional download action, and structured summary paragraphs.

#### Historical inspection tasks

- `src/app/resume/page.tsx`의 표준 query/shell lifecycle과 `/resume` currentPath를 확인합니다.
- `downloadUrl`이 truthy일 때만 anchor를 렌더링하는 분기를 적습니다.
- 이 SHA에는 page enablement gate가 없다는 이전 상태를 확인합니다.

#### Reconstruction result

Resume route도 초기 표준 lifecycle을 따릅니다. content를 읽고 `view`/`debug`를 해석한 뒤 `/resume` current path를 가진 `PageShell` 안에 identity, hero copy, structured summary를 렌더링합니다. `content.resume.downloadUrl`이 존재할 때만 download anchor를 표시하므로 optional content branch는 명시적입니다.

그러나 page capability 검사는 없습니다. 이 시점에는 navigation 노출 여부와 직접 URL 접근을 route에서 통제하는 공통 selector가 적용되지 않았습니다. 따라서 “later config에서 disabled이면 404”라는 final behavior를 이 commit에 적용하면 chronology 오류입니다.

### 3. `bfcdf44eb34c` — feat(contact): 연락 페이지 소개 추가

- Importance: **B**
- Tags: `RENDERER`
- Fixed role: Create the Contact route with profile identity and contact-owned title and introductory text.

#### Historical inspection tasks

- `src/app/contact/page.tsx`의 표준 query/shell lifecycle과 `/contact` currentPath를 확인합니다.
- content source hint와 profile/contact source ownership을 구분합니다.
- 이 SHA에는 page enablement gate가 없다는 이전 상태를 확인합니다.

#### Reconstruction result

Contact route는 content aggregate에서 profile과 `contact` source를 읽고 query-state-aware `PageShell`을 구성합니다. current path는 `/contact`이며 intro 단계에서는 title과 introductory text가 중심입니다. `ContentHint`는 source location을 사용자에게 노출할 수 있지만 route availability를 결정하지 않습니다.

About/Resume와 동일하게 `isSitePageEnabled`와 `notFound()`가 없습니다. 그러므로 세 early auxiliary route는 route creation 시점에 default-open이 아니라 사실상 unconditional-open이었습니다. 후속 selector 및 gate가 이 상태를 정규화합니다.

### 4. `3e2e95a3a28c` — feat(content): 페이지 활성화 selector 추가

- Importance: **B**
- Tags: `CONTENT, ROUTING`
- Fixed role: Add a typed selector for the site's optional page flags.

#### Historical inspection tasks

- `src/lib/portfolio/selectors.ts`와 facade export에서 `isSitePageEnabled`가 추가되는지 확인합니다.
- `content.site.pages?.[pageId] !== false`의 default-open 의미를 truth table로 적습니다.
- selector 추가만으로 기존 route가 차단되는 것은 아니라는 caller ownership 경계를 확인합니다.

#### Reconstruction result

`src/lib/portfolio/selectors.ts`와 public facade에 다음 typed selector가 추가됩니다.

```ts
export function isSitePageEnabled(
  pageId: SitePageId,
  content: PortfolioContent = getPortfolioContent(),
) {
  return content.site.pages?.[pageId] !== false;
}
```

truth table은 명확합니다.

| `site.pages`/값 | 결과 |
| --- | --- |
| `pages` 없음 | enabled |
| 해당 key 없음 | enabled |
| `true` | enabled |
| `false` | disabled |

정책은 fail-closed가 아니라 **explicit-disable, default-open**입니다. optional starter content에서 flag가 누락되어도 기존 route를 깨지 않기 위한 호환 정책입니다.

이 commit은 selector만 추가합니다. existing About/Resume/Contact/Projects callers를 수정하지 않으므로 selector 존재가 곧 enforcement를 의미하지 않습니다. route가 직접 호출해야만 policy가 실행됩니다. 이 caller gap이 후속 비대칭의 핵심입니다.

### 5. `0facaf123f29` — feat(journey): 여정 route 소개 추가

- Importance: **B**
- Tags: `ROUTING, RENDERER`
- Fixed role: Introduce the journey narrative as an optional route using the portfolio's standard page lifecycle.

#### Historical inspection tasks

- `JourneyPage`가 content를 읽은 직후 `isSitePageEnabled("journey", content)`를 검사하고 실패 시 `notFound()`를 호출하는 순서를 확인합니다.
- gate가 search params 처리와 renderer 준비보다 먼저 실행되는지 확인합니다.
- 이 route는 도입 시점부터 fail-closed였다는 점을 이전 About/Resume/Contact와 비교합니다.

#### Reconstruction result

Journey route는 도입 순간부터 availability-aware입니다. function은 content를 읽은 직후 다음 순서로 실행됩니다.

1. `isSitePageEnabled("journey", content)` 확인
2. false이면 `notFound()` 호출
3. gate 통과 후에만 search params await
4. template/debug 계산
5. journey presentation/narrative 준비
6. shell과 body 렌더링

따라서 disabled Journey 요청은 query parsing이나 renderer/data 준비를 하지 않습니다. 이 lifecycle은 earlier About/Resume/Contact와 다릅니다. selector가 이미 존재하는 시점에 route가 추가되어 처음부터 caller contract를 따랐기 때문입니다.

### 6. `ddba753f7f51` — feat(interview-map): 근거 route 소개 추가

- Importance: **B**
- Tags: `ROUTING, RENDERER`
- Fixed role: Introduce the interview-evidence map as an optional, independently addressable route.

#### Historical inspection tasks

- `InterviewMapPage`의 `interviewMap` page ID와 gate 위치를 확인합니다.
- gate 통과 후 query/shell/data 준비가 이어지는 순서를 적습니다.
- 외부 reference link와 page body 기능이 아니라 route availability lifecycle에 초점을 맞춥니다.

#### Reconstruction result

Interview Map route도 동일한 도입 패턴을 사용합니다. content load 직후 `isSitePageEnabled("interviewMap", content)`를 검사하고 false이면 `notFound()`로 종료합니다. 그 뒤에만 query state, presentation copy, interview data, shell props를 준비합니다.

page ID가 URL segment 문자열 `interview-map`이 아니라 typed content key `interviewMap`이라는 점이 중요합니다. route path와 content capability key의 mapping은 caller가 명시적으로 소유합니다.

외부 reference repository link와 track feature는 route body 기능입니다. 이 Thread에서 중요한 사실은 route가 최초 commit부터 disabled capability를 직접 접근할 수 없게 했다는 것입니다.

### 7. `50199be241c8` — feat(routes): 비활성 페이지 route 차단

- Importance: **A**
- Tags: `ARCH, ROUTING`
- Fixed role: Enforce page-enablement settings at the route boundary for about, contact, projects, project details, and resume.

#### Historical inspection tasks

- `about`, `contact`, `projects`, `projects/[projectId]`, `resume`의 각 route에서 content load 직후 어떤 page ID로 gate가 추가되는지 표로 정리합니다.
- project index와 detail이 동일한 `projects` capability를 공유하며 detail은 params/query/project lookup보다 먼저 차단되는지 확인합니다.
- Journey/Interview에는 변경이 없는 이유가 이미 도입 시 gate를 가진 상태인지 이전 SHA와 비교합니다.
- 이 commit이 navigation 항목 제거, sitemap 정책, `generateStaticParams` filtering까지 수행하는지 여부를 diff로 구분합니다.
- 이전 비대칭 → 실제 risk → 공통 route-boundary invariant로의 correction을 재구성합니다.

#### Reconstruction result

**이전 비대칭**

- Journey/Interview Map: 도입 시점부터 gate 존재
- About/Resume/Contact/Projects index/detail: 이미 존재하지만 selector를 호출하지 않음

따라서 같은 `site.pages` configuration이 route마다 다르게 적용될 수 있었습니다. navigation에서 숨겼더라도 사용자가 URL을 직접 입력하면 old routes가 렌더링될 위험이 있었습니다.

**cross-route correction**

각 page는 content load 직후 다음 gate를 받습니다.

| Route | capability key |
| --- | --- |
| `/about` | `about` |
| `/contact` | `contact` |
| `/projects` | `projects` |
| `/projects/[projectId]` | `projects` |
| `/resume` | `resume` |

project detail의 gate는 `params` await, query parsing, project lookup보다 먼저입니다. disabled projects capability에서는 존재 여부를 조사하거나 renderer context를 만들지 않고 바로 not-found lifecycle로 전환합니다. index와 detail이 한 capability를 공유하므로 list만 막고 deep link를 남기는 불일치도 제거됩니다.

Journey/Interview files에는 diff가 없습니다. 이미 같은 invariant를 만족했기 때문입니다.

**보장**

- explicit false인 page의 direct route request는 route boundary에서 `notFound()`를 호출합니다.
- gate가 expensive/irrelevant page work보다 먼저 실행됩니다.
- 기존 unconditional route와 newer gated route의 policy가 통일됩니다.

**비보장**

diff는 `site.navigation`을 필터링하지 않고 `generateStaticParams`를 바꾸지 않으며 sitemap/indexing 정책도 다루지 않습니다. 즉 “request rendering gate”만 보장합니다. discovery/publication은 별도 selector/SEO 정책의 책임입니다.

이 commit에는 새 route-gate test가 포함되지 않았습니다. exact route diffs로 enforcement를 확인했으며 runtime server 결과는 실행하지 않았습니다.

## 6. Invariant evolution

Record where each invariant was introduced, extended, shown insufficient, corrected, and verified.

| 단계 | 상태 |
| --- | --- |
| early auxiliary routes | About, Resume, Contact가 gate 없이 생성됨 |
| selector 도입 | missing/true는 enabled, false만 disabled |
| new optional routes | Journey와 Interview Map이 도입 시부터 gate 사용 |
| cross-route correction | 기존 About/Contact/Projects/Resume도 content load 직후 gate 적용 |

최종 invariant는 “optional page flag가 explicit false이면 해당 capability의 모든 owned route가 page-specific work 전에 `notFound()`로 종료한다”입니다.

## 7. Failure → Fix → Test relationships

Connect fixes and tests to the exact earlier assumption or implementation they correct or verify.

- **Previous assumption:** navigation/config가 route 접근 가능성도 자연스럽게 통제할 것이라는 암묵적 상태.
- **Actual risk:** 기존 route는 selector가 생긴 뒤에도 caller가 아니어서 direct URL 접근이 계속 가능.
- **Root cause:** page availability가 data model에만 있고 route boundary enforcement가 일관되지 않음.
- **Correction:** `50199...`가 early routes와 project deep link 모두에 같은 gate를 추가.
- **Regression evidence:** 이 selected thread에는 전용 test commit이 없습니다. later route matrix가 존재하더라도 실행하거나 이 commit의 직접 regression으로 소급하지 않았습니다.

## 8. Ownership, state, and responsibility changes

Track caller/callee ownership, browser/framework state, resource lifetime, and boundaries that deliberately remain outside the Thread.

| 책임 | 소유자 |
| --- | --- |
| page flags | validated `content.site.pages` |
| default-open 해석 | `isSitePageEnabled` |
| path→capability key mapping | 각 App Router page |
| request 차단 | 각 route의 early `notFound()` |
| navigation 노출 | site navigation consumer, 별도 |
| static params/sitemap/indexing | 별도 build/SEO policy |
| not-found presentation | global `not-found.tsx` |

## 9. Final Thread state

- explicit false는 route rendering을 차단합니다.
- 누락 flag는 compatibility상 enabled입니다.
- Journey/Interview는 처음부터 정책을 지켰고 older routes는 retrofit됐습니다.
- project index와 detail이 동일한 capability 아래 묶입니다.
- route gate만으로 navigation, sitemap, static generation이 자동 동기화된다고 가정하지 않습니다.

## 10. Final architecture or execution flow

1. App Router page가 validated portfolio content를 읽습니다.
2. path가 소유한 `SitePageId`로 `isSitePageEnabled`를 호출합니다.
3. false이면 즉시 `notFound()`로 global recovery boundary에 넘깁니다.
4. enabled이면 그때 search params, renderer context, page-specific data를 준비합니다.
5. route body가 선택된 shell/renderer로 출력됩니다.

## 11. Learning-completion checks

- [x] early ungated state를 final code로 덮지 않았습니다.
- [x] selector의 default-open truth table을 설명했습니다.
- [x] Journey/Interview와 retrofit routes의 chronology 차이를 적었습니다.
- [x] project index/detail capability 공유를 확인했습니다.
- [x] route gate가 하지 않는 navigation/sitemap/static-param 작업을 명시했습니다.
===== END FILE: 05-page-availability-and-auxiliary-route-lifecycle.md =====

===== BEGIN FILE: 06-custom-not-found-recovery.md =====
# Development Thread 06 — Custom Not-Found Recovery

## 0. Source and frozen audit scope

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/portfolio`
- Category: `02-routing-navigation-and-page-lifecycle`
- Change range for directed inspection: `154b7e6cb54b^..850e084b3911`
- Classification source: `commit/commit-importance.md` on `web/portfolio`
- Historical rule: inspect every commit at its exact SHA; do not project later code backward.
- Phase 1 status: audited and frozen before learner-facing completion.
- Thread boundary: Next App Router global not-found presentation과 semantic recovery link를 다룹니다. 개별 route가 `notFound()`를 호출하는 availability 정책은 이전 Thread의 책임입니다.
- Execution note: source inspection and runtime command evidence must be recorded separately.

### Phase 1 audit decisions

- project dynamic-detail story에 잘못 결합된 global 404를 독립 Thread로 분리했습니다.

## 1. Learning goal and planned invariants

framework not-found boundary에 portfolio shell, no-index metadata, 명시적 home recovery path를 추가하고 focused component test로 최소 계약을 보호한 과정을 재구성합니다.

- global not-found page는 configured default design의 shared shell을 사용합니다.
- crawler metadata는 index/follow를 모두 false로 둡니다.
- 사용자에게 level-1 error heading과 semantic `/` link를 제공합니다.
- focused test는 heading/link contract만 증명하며 HTTP status와 router integration은 별도입니다.

## 2. Questions to answer

- global 404가 project detail Thread와 독립이어야 하는 이유는 무엇입니까?
- not-found page가 query state를 보존하지 않고 default design을 쓰는 실제 구현은 무엇입니까?
- component test의 assertion과 metadata/HTTP 비보장은 어떻게 구분됩니까?

## 3. Completion criteria

- feature와 test commit을 연결합니다.
- metadata, shell, recovery link의 소유 위치를 적습니다.
- 실행하지 않은 Next server behavior를 검증했다고 쓰지 않습니다.
- test가 증명하지 않는 범위를 명시합니다.

## 4. Fixed commit map

| # | Commit | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `154b7e6cb54b` | feat(site): 사용자 정의 404 페이지 추가 | B | - | Provide a portfolio-styled not-found page with an explicit route back to the home page. |
| 2 | `850e084b3911` | test(site): 404 복귀 동선 검증 | B | VALIDATION, TEST | Verify that the custom not-found page communicates the error through its primary heading and exposes a semantic link back to `/`. |

The commit SHA, order, subject, importance, tags, and source-defined role in this map are frozen.

## 5. Commit-by-commit reconstruction

### 1. `154b7e6cb54b` — feat(site): 사용자 정의 404 페이지 추가

- Importance: **B**
- Tags: `-`
- Fixed role: Provide a portfolio-styled not-found page with an explicit route back to the home page.

#### Historical inspection tasks

- `src/app/not-found.tsx`의 metadata, content load, `PageShell`, heading/body, `Link href="/"`를 확인합니다.
- `robots: { index: false, follow: false }`가 page component와 별도 export인지 확인합니다.
- 404 component가 request query를 받지 않고 configured default design으로 shell을 구성하는지 확인합니다.

#### Reconstruction result

`src/app/not-found.tsx`가 App Router global not-found component로 추가됩니다.

**metadata**

- title: `Page not found`
- robots: `index: false`, `follow: false`

**render path**

component는 `getPortfolioContent()`를 호출하고 `content.presentation.defaultHomeTemplate`을 `PageShell`의 home template으로 사용합니다. profile, site, presentation UI도 shell에 전달합니다. request `searchParams`를 받지 않으므로 failed URL의 `view`/`debug`를 보존하지 않습니다.

body는 visible `404`, level-1 `Page not found` heading, 설명, Next `Link` 기반 `Return home` action을 렌더링합니다. home href는 정확히 `/`입니다.

이 route는 특정 project missing-ID 전용이 아닙니다. unknown route, page gate, dynamic lookup 등 Next의 not-found boundary로 들어오는 여러 실패의 공통 presentation입니다. 따라서 project lifecycle에서 분리하는 것이 맞습니다.

코드 inspection으로 component와 metadata export를 확인했으며 HTTP status를 실제 server에서 실행하지 않았습니다.

### 2. `850e084b3911` — test(site): 404 복귀 동선 검증

- Importance: **B**
- Tags: `VALIDATION, TEST`
- Fixed role: Verify that the custom not-found page communicates the error through its primary heading and exposes a semantic link back to `/`.

#### Historical inspection tasks

- `src/app/not-found.test.tsx`가 component를 직접 render하고 level-1 heading 및 named link의 href를 확인하는지 적습니다.
- 이 테스트가 HTTP 404 status, Next router의 `notFound()` 연동, metadata export를 검증하지 않는다는 범위를 구분합니다.
- 검증 유형을 focused component regression/semantic recovery evidence로 분류합니다.

#### Reconstruction result

`src/app/not-found.test.tsx`는 `NotFound` component를 직접 Testing Library로 render합니다. assertion은 두 개입니다.

- role `heading`, level 1, accessible name `Page not found`
- role `link`, name `Return home`, attribute `href="/"`

이것은 사용자에게 오류가 semantic heading으로 전달되고 명확한 recovery path가 존재하는지를 보호하는 focused component regression입니다. raw text snapshot보다 role/name을 사용하므로 accessibility-facing contract를 직접 확인합니다.

다만 다음은 테스트하지 않습니다.

- 실제 unknown URL의 HTTP status가 404인지
- route의 `notFound()`가 이 component를 선택하는지
- exported metadata의 noindex/nofollow
- design query preservation
- browser click 후 home이 성공적으로 로드되는지

이 환경에서는 test command를 실행하지 않았습니다. exact test source와 assertion을 inspection했습니다.

## 6. Invariant evolution

Record where each invariant was introduced, extended, shown insufficient, corrected, and verified.

| 단계 | invariant |
| --- | --- |
| `154b...` | global not-found boundary가 portfolio shell, no-index metadata, `/` recovery link 제공 |
| `850e...` | H1 error communication과 semantic home link를 focused test로 고정 |

최종 invariant는 “not-found presentation은 index 대상이 아니며, 사용자가 semantic heading으로 실패를 인지하고 root로 복귀할 수 있다”입니다.

## 7. Failure → Fix → Test relationships

Connect fixes and tests to the exact earlier assumption or implementation they correct or verify.

- **Failure → Recovery UI:** route gate나 missing lookup이 `notFound()`를 호출하면 global component가 공통 recovery surface를 제공합니다.
- **Feature → Test:** heading과 home link를 `850e...`가 직접 assertion합니다.
- **검증 공백:** metadata, HTTP status, App Router integration은 selected test가 증명하지 않습니다.

## 8. Ownership, state, and responsibility changes

Track caller/callee ownership, browser/framework state, resource lifetime, and boundaries that deliberately remain outside the Thread.

| 책임 | 소유자 |
| --- | --- |
| not-found transition 결정 | individual route/Next router |
| global failure presentation | `src/app/not-found.tsx` |
| default design choice | `content.presentation.defaultHomeTemplate` |
| crawler policy | exported metadata |
| recovery navigation | Next `Link` to `/` |
| focused semantic regression | `not-found.test.tsx` |

## 9. Final Thread state

- 모든 App Router not-found path가 사용할 수 있는 portfolio-styled recovery page가 있습니다.
- failed request query를 재구성하지 않고 configured default design을 사용합니다.
- page는 noindex/nofollow metadata를 선언합니다.
- component test는 H1과 root link만 보호합니다.

## 10. Final architecture or execution flow

1. route 또는 framework가 요청을 not-found lifecycle로 전환합니다.
2. global `NotFound`가 portfolio content를 읽습니다.
3. configured default design으로 shared shell을 구성합니다.
4. H1 error message와 `/` recovery link를 렌더링합니다.
5. 사용자가 link를 선택하면 root route로 새 navigation을 시작합니다.

## 11. Learning-completion checks

- [x] global 404를 project detail과 분리했습니다.
- [x] metadata와 component body를 구분했습니다.
- [x] test의 정확한 role/name/href assertion을 설명했습니다.
- [x] HTTP/router integration 비보장을 명시했습니다.
- [x] test를 실행했다고 주장하지 않았습니다.
===== END FILE: 06-custom-not-found-recovery.md =====

===== BEGIN FILE: 07-page-context-consolidation.md =====
# Development Thread 07 — Page Context Consolidation

## 0. Source and frozen audit scope

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/portfolio`
- Category: `02-routing-navigation-and-page-lifecycle`
- Change range for directed inspection: `42bef4e5783c^..349317425bd2`
- Classification source: `commit/commit-importance.md` on `web/portfolio`
- Historical rule: inspect every commit at its exact SHA; do not project later code backward.
- Phase 1 status: audited and frozen before learner-facing completion.
- Thread boundary: 모든 App Router page의 공통 content/query/shell 초기화 ownership만 다룹니다. availability gate, route-specific lookup/derivation, renderer dispatch는 helper 밖에 남습니다.
- Execution note: source inspection and runtime command evidence must be recorded separately.

### Phase 1 audit decisions

- `42bef4e5783c`를 실제 chronology에 맞춰 Thread 첫머리의 characterization baseline으로 이동했습니다.
- route군별 migration order를 branch history와 일치시켰습니다.

## 1. Learning goal and planned invariants

route characterization baseline을 먼저 고정한 뒤 `resolvePortfolioPageContext`를 도입하고 route군별로 이관하여 중복 query/shell 초기화를 제거한 과정을 재구성합니다.

- characterization test는 refactor 이전 route behavior를 canonical content 기준으로 고정합니다.
- resolver는 content, currentPath, searchParams를 받아 template/debug와 shell props를 한 번 계산합니다.
- caller가 이미 읽은 content를 주입해 동일 요청에서 aggregate load ownership을 유지할 수 있습니다.
- availability gate는 resolver 전에, route-specific lookup/derivation과 renderer dispatch는 resolver 밖에 남습니다.
- 모든 public page의 common initialization은 최종적으로 하나의 helper에 모입니다.

## 2. Questions to answer

- `42bef...`을 refactor 후 test로 오해하면 어떤 chronology 오류가 생깁니까?
- `PortfolioPagePath` union은 어떤 path를 허용하며 dynamic detail을 어떻게 표현합니까?
- content injection은 어떤 중복과 consistency risk를 줄입니까?
- 왜 availability와 project lookup을 common helper에 넣지 않았습니까?
- route별 단계적 migration이 기존 characterization과 어떻게 연결됩니까?

## 3. Completion criteria

- characterization baseline을 첫 commit으로 설명합니다.
- Home 도입과 세 migration commit의 ownership 이동을 구분합니다.
- helper가 소유하는 값과 계속 caller에 남는 값을 표로 정리합니다.
- existing tests를 실제 실행하지 않았다면 source-level safety net으로만 기록합니다.
- refactor 전후 behavior preservation을 근거 없는 runtime claim으로 쓰지 않습니다.

## 4. Fixed commit map

| # | Commit | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `42bef4e5783c` | test(routes): 홈과 route presentation 계약 검증 | A | ARCH, VALIDATION, ROUTING | Add route-level characterization tests that compare presentation shells against canonical content rather than hard-coded snapshots. |
| 2 | `1cf65b708476` | refactor(routes): 홈 page context 통합 | A | ARCH, ROUTING, REFACTOR | Introduce `resolvePortfolioPageContext` as the single owner of common page initialization and migrate Home to it. |
| 3 | `2075e54ff947` | refactor(projects): 프로젝트 page context 통합 | B | RENDERER, REFACTOR | Migrate both the project archive and project-detail routes to the shared page-context resolver. |
| 4 | `a9fdde29221d` | refactor(routes): 소개와 학습 route context 통합 | B | ROUTING, REFACTOR | Apply the common page-context boundary to About, Journey, and Interview Map. |
| 5 | `349317425bd2` | refactor(routes): 이력과 연락 context 통합 | B | ROUTING, REFACTOR | Complete the page-context migration for Resume and Contact. |

The commit SHA, order, subject, importance, tags, and source-defined role in this map are frozen.

## 5. Commit-by-commit reconstruction

### 1. `42bef4e5783c` — test(routes): 홈과 route presentation 계약 검증

- Importance: **A**
- Tags: `ARCH, VALIDATION, ROUTING`
- Fixed role: Add route-level characterization tests that compare presentation shells against canonical content rather than hard-coded snapshots.

#### Historical inspection tasks

- 이 commit이 resolver refactor보다 먼저라는 chronology를 확인하고 characterization baseline으로 다룹니다.
- `src/app/page.test.tsx`, `src/app/routes.test.tsx`, `src/app/journey/page.test.tsx`의 route matrix와 canonical content lookup을 확인합니다.
- 각 route에서 H1, `data-home-template`, content hint, state-preserving Projects link, design switcher current/target href를 어떤 방식으로 검증하는지 적습니다.
- 반복 `view`/`debug` query에서 첫 값을 사용하는 test와 다섯 design/default/unknown fallback test를 구분합니다.
- 직접 함수 unit test가 아니라 async page component를 render하는 characterization의 강점과 browser-level 비보장을 적습니다.

#### Reconstruction result

**chronology와 목적**

이 commit은 `resolvePortfolioPageContext` 도입보다 먼저입니다. 따라서 refactor 결과를 검증하기 위해 나중에 작성된 test가 아니라, 기존 route behavior를 먼저 기록한 **characterization baseline**입니다.

**검증 표면**

- `src/app/page.test.tsx`
  - original design/classic이 같은 journey evidence를 노출하는지 확인합니다.
  - query가 없으면 `editorial` default를 사용합니다.
  - 다섯 design 각각에서 root marker, H1, featured project, project navigation href, design switcher link count를 확인합니다.
  - unknown design이 editorial로 fallback하는지 확인합니다.
  - `debug=content`와 design state가 project/design links에 보존되는지 확인합니다.
- `src/app/routes.test.tsx`
  - Home, About, Contact, Interview Map, Journey, Projects, Project Detail, Resume의 8개 case를 async page component로 직접 render합니다.
  - canonical content에서 heading/labels를 읽어 hard-coded snapshot 의존을 줄입니다.
  - classic shell marker, content source hints, Projects href, switcher active/target href를 확인합니다.
  - repeated `view`/`debug` 배열에서 첫 값을 사용하는 route behavior를 검증합니다.
- `src/app/journey/page.test.tsx`
  - content-owned skip/navigation/menu labels와 milestone labels를 확인합니다.

**test 성격**

이것은 pure helper unit test보다 넓고 full browser e2e보다 좁은 route-component characterization입니다. App Router page function을 실제 props로 호출하고 resulting React tree를 render해 공통 presentation contract를 기록합니다.

**증명하지 않는 것**

HTTP request/response, browser history, network navigation, hydration timing, production build artifact는 실행하지 않습니다. 또한 refactor commit 자체가 아직 존재하지 않으므로 resolver implementation을 직접 검증하지 않습니다. 후속 migration이 이 baseline을 깨지 않아야 한다는 safety net으로 기능합니다.

이 환경에서는 source를 inspection했으며 Vitest command는 실행하지 않았습니다.

### 2. `1cf65b708476` — refactor(routes): 홈 page context 통합

- Importance: **A**
- Tags: `ARCH, ROUTING, REFACTOR`
- Fixed role: Introduce `resolvePortfolioPageContext` as the single owner of common page initialization and migrate Home to it.

#### Historical inspection tasks

- `src/lib/portfolio/page-context.ts`의 `PortfolioPagePath` union, optional content injection, async searchParams, resolver return shape를 확인합니다.
- `activeTemplate`, `contentDebug`, `shellProps`가 한 번 계산되고 `templateSwitcher.currentPath`에 typed path가 들어가는지 추적합니다.
- Home에서 제거된 중복 초기화와 남아 있는 renderer dispatch/view preparation을 구분합니다.
- helper가 page availability, project lookup, renderer 선택까지 소유하지 않는다는 경계를 적습니다.
- pre-existing `42bef...` characterization이 refactor safety net으로 어떤 계약을 제공하는지 연결합니다.

#### Reconstruction result

**중복된 이전 상태**

Home은 직접 content를 읽고, search params를 await하고, template/debug를 resolve하며, shell이 필요한 공통 props를 다시 조립했습니다. 같은 pattern이 다른 public routes에도 반복됐습니다. path 한 곳을 잘못 쓰거나 UI/templates/debug prop을 누락해도 compiler가 공통 initialization의 동일성을 강제하지 못했습니다.

**새 ownership boundary**

`src/lib/portfolio/page-context.ts`에 `resolvePortfolioPageContext`가 추가됩니다.

입력:

- optional injected `content`
- typed `currentPath: PortfolioPagePath`
- optional async `searchParams`

`PortfolioPagePath`는 `/`, `/about`, `/contact`, `/interview-map`, `/journey`, `/projects`, `/resume`, ``/projects/${string}``만 허용합니다.

처리:

1. content가 없으면 `getPortfolioContent()`를 사용합니다.
2. search params를 한 번 await합니다.
3. `view`와 `debug`를 각각 canonical selector로 정규화합니다.
4. `activeTemplate`, `content`, `contentDebug`, complete `shellProps`를 반환합니다.
5. switcher props에는 active ID, debug, typed current path, template 목록이 포함됩니다.

Home은 직접 초기화 코드를 제거하고 resolver 결과를 사용합니다. 그러나 `hasDedicatedRouteRenderer`/`renderDesignRoute`와 Home-specific rendering 선택은 caller에 남습니다.

**경계**

helper는 page availability를 판단하지 않고 project ID도 lookup하지 않으며 route view model도 만들지 않습니다. 즉 “모든 page behavior”가 아니라 **공통 request presentation context**만 소유합니다. optional content injection은 caller가 이미 읽은 같은 aggregate instance를 재사용하게 해 같은 request에서 data source가 갈라지는 위험을 줄입니다.

`42bef...` test는 helper를 직접 부르지 않지만 Home의 visible shell/query behavior를 refactor 전 기준으로 고정합니다. 실행은 하지 않았으므로 behavior preservation을 runtime 결과로 주장하지 않습니다.

### 3. `2075e54ff947` — refactor(projects): 프로젝트 page context 통합

- Importance: **B**
- Tags: `RENDERER, REFACTOR`
- Fixed role: Migrate both the project archive and project-detail routes to the shared page-context resolver.

#### Historical inspection tasks

- `/projects`와 `/projects/${projectId}`가 기존 content instance를 resolver에 주입해 중복 load를 피하는지 확인합니다.
- page enablement gate가 resolver 호출 전에 그대로 남는지 확인합니다.
- detail route가 raw `projectId`로 currentPath를 만든 뒤 lookup/notFound를 수행하지만 shell은 성공 시에만 render되는 흐름을 적습니다.

#### Reconstruction result

Project index와 detail이 shared resolver로 이관됩니다. 두 route 모두 availability 검사에 사용할 content를 먼저 읽고 그 **같은 content instance**를 resolver의 `content` 인자로 전달합니다. 이 때문에 helper가 aggregate를 다시 읽지 않습니다.

- index current path: `/projects`
- detail current path: ``/projects/${projectId}``

기존 route-local search params await, template/debug resolve, hand-built shell props가 제거됩니다. page enablement gate는 resolver 전에 그대로 남습니다.

detail의 순서는 미묘합니다.

1. content load
2. projects capability gate
3. params await
4. resolver로 raw projectId 기반 currentPath와 query context 준비
5. `getProjectById`
6. missing이면 `notFound()`
7. 성공하면 prepared `shellProps`로 render

unknown project에서도 context object는 한 번 계산되지만 shell은 lookup 성공 후에만 render됩니다. project lookup을 helper에 넣지 않은 덕분에 common context가 domain-specific failure를 소유하지 않습니다.

### 4. `a9fdde29221d` — refactor(routes): 소개와 학습 route context 통합

- Importance: **B**
- Tags: `ROUTING, REFACTOR`
- Fixed role: Apply the common page-context boundary to About, Journey, and Interview Map.

#### Historical inspection tasks

- About, Journey, Interview Map에서 제거된 query/template/debug/shell 중복과 새 resolver 호출을 비교합니다.
- 각 route의 정확한 currentPath literal을 확인합니다.
- availability gate와 route-specific data/renderer preparation이 helper 밖에 남는지 확인합니다.

#### Reconstruction result

About, Journey, Interview Map 세 route가 같은 pattern으로 이동합니다.

- `/about`
- `/journey`
- `/interview-map`

각 route에서 직접 수행하던 search params await, `resolveHomeTemplateId`, `resolveContentDebug`, hand-built `PageShell` props가 제거되고 `resolvePortfolioPageContext({ content, currentPath, searchParams })` 결과를 사용합니다.

중요하게 availability gate는 이동하지 않습니다. 각 page는 content를 읽고 자신의 `SitePageId`를 검사한 뒤에 resolver를 호출합니다. 또한 curation, journey narrative, interview track 같은 route-specific data preparation과 dedicated renderer dispatch도 caller에 남습니다.

따라서 migration은 behavior를 한 helper로 “흡수”한 것이 아니라 중복된 presentation context 준비만 치환한 local refactor입니다. 세 route의 서로 다른 content semantics는 보존됩니다.

### 5. `349317425bd2` — refactor(routes): 이력과 연락 context 통합

- Importance: **B**
- Tags: `ROUTING, REFACTOR`
- Fixed role: Complete the page-context migration for Resume and Contact.

#### Historical inspection tasks

- Resume `/resume`, Contact `/contact`의 resolver 호출과 `PageShell {...shellProps}` 전환을 확인합니다.
- preferred contacts, resume project selection 같은 route-specific derivation이 이동하지 않았는지 확인합니다.
- 이 commit 이후 category 내 public page들의 common initialization ownership이 어디로 모이는지 적습니다.

#### Reconstruction result

마지막으로 Resume와 Contact가 resolver를 사용합니다.

- Resume current path: `/resume`
- Contact current path: `/contact`

두 route에서 template/debug 계산과 verbose `PageShell` props가 제거되고 `shellProps` spread로 바뀝니다. 기존 page enablement gate는 resolver 앞에 남습니다.

route-specific derivation도 그대로입니다.

- Resume: configured resume project selection, optional download/content sections
- Contact: preferred contact link selection과 contact-specific body data

이 commit 이후 Home, Projects index/detail, About, Journey, Interview Map, Resume, Contact가 공통 query/shell initialization boundary를 공유합니다. helper가 content/query/path를 presentation context로 바꾸고, 각 caller는 availability, domain lookup/derivation, renderer 선택을 계속 소유합니다.

## 6. Invariant evolution

Record where each invariant was introduced, extended, shown insufficient, corrected, and verified.

| 단계 | invariant 변화 |
| --- | --- |
| characterization | `42bef...`이 refactor 전 route/shell/query behavior를 기록 |
| boundary 도입 | `1cf65...`이 typed path와 common context resolver를 만들고 Home 이관 |
| project migration | `2075...`가 index/detail을 같은 content instance로 이관 |
| profile/learning migration | `a9fd...`가 About/Journey/Interview 이관 |
| completion | `349...`가 Resume/Contact 이관 |

최종 invariant는 “모든 public page의 template/debug/shell 초기화는 `resolvePortfolioPageContext`가 소유하고, framework/domain-specific decisions는 각 route가 소유한다”입니다.

## 7. Failure → Fix → Test relationships

Connect fixes and tests to the exact earlier assumption or implementation they correct or verify.

- **Characterization → Refactor:** `42bef...`이 visible route contract를 먼저 고정하고 다음 commit들이 implementation ownership만 이동합니다.
- **Duplication risk → Boundary:** route마다 query await와 shell props를 반복하던 상태를 `1cf65...`이 typed helper로 통합합니다.
- **Incremental migration:** project routes, profile/learning routes, resume/contact 순으로 caller가 이동합니다. 한 번에 전 route를 바꾸지 않아 diff 범위를 제한합니다.
- **검증 한계:** selected migration commits에 새 test는 없습니다. existing characterization source가 safety net이지만 이 환경에서 실행하지 않았으므로 “모두 통과했다”고 기록하지 않습니다.

## 8. Ownership, state, and responsibility changes

Track caller/callee ownership, browser/framework state, resource lifetime, and boundaries that deliberately remain outside the Thread.

| 책임 | resolver | route caller |
| --- | --- | --- |
| content fallback load | 예 | caller가 content를 주입할 수도 있음 |
| search params await | 예 | 아니요 |
| template/debug 정규화 | 예 | 아니요 |
| shell props 구성 | 예 | 아니요 |
| typed current path | 입력으로 요구 | 정확한 path 제공 |
| page availability gate | 아니요 | 예 |
| dynamic params/project lookup | 아니요 | 예 |
| route-specific derivation | 아니요 | 예 |
| renderer dispatch | 아니요 | 예 |

## 9. Final Thread state

- 공통 page initialization 코드가 한 helper로 모였습니다.
- path union이 supported public path를 type-level로 제한합니다.
- caller는 이미 읽은 content를 주입해 동일 aggregate를 재사용합니다.
- availability와 domain failure는 helper 밖에 남아 responsibility가 과도하게 확장되지 않습니다.
- characterization tests는 refactor 이전 behavior를 기준으로 하지만 실제 실행 결과는 기록하지 않았습니다.

## 10. Final architecture or execution flow

1. route가 필요한 경우 content를 먼저 읽고 availability를 검사합니다.
2. route가 exact typed current path와 search params, 기존 content를 resolver에 전달합니다.
3. resolver가 query를 await하고 active template/debug를 정규화합니다.
4. resolver가 공통 shell props와 normalized context를 반환합니다.
5. caller가 project lookup, route view data, renderer dispatch를 수행하고 shell/context를 소비합니다.

## 11. Learning-completion checks

- [x] `42bef...`을 characterization baseline으로 첫 위치에 두었습니다.
- [x] resolver 입력/출력과 typed path union을 설명했습니다.
- [x] availability, lookup, renderer dispatch가 caller에 남는 이유를 구분했습니다.
- [x] migration 순서를 exact chronology로 유지했습니다.
- [x] existing tests를 실행하지 않고 통과했다고 주장하지 않았습니다.
===== END FILE: 07-page-context-consolidation.md =====

===== BEGIN FILE: README.md =====
# Category 02 — Routing, Navigation, and Page Lifecycle

## Scope

This category reconstructs the App Router lifecycle that turns URL state and validated page configuration into addressable routes, shared navigation, page availability, not-found recovery, and common page context.

It includes:

- query-state normalization and state-preserving internal URLs
- shared header, footer, mobile navigation, and active-route semantics
- native design-switcher disclosure, hydration, focus, and server/client ownership
- project index and dynamic detail addressability
- page capability gates for optional routes
- global not-found recovery
- common page-context consolidation

It excludes:

- typed content-link transport and external-anchor security, which belong to Category 03
- page body feature construction, which belongs to Category 04
- visual-system styling and complete renderer construction, which belong to Category 05
- generic test/performance strategy, which belongs to Category 07
- source-defined cross-cutting architecture narratives, which may reuse selected commits in Category 09

## Phase 1 audit outcome

The draft category had five generic Threads. Repository evidence required these material changes before freeze:

- removed `f63c978c71c9` from query navigation because Category 03 already owns the typed internal/external link transport story
- reordered `51806e1875e7` before the later mobile-menu and shared-switcher commits
- moved `42bef4e5783c` to the beginning of page-context consolidation because it is pre-refactor characterization
- added the omitted native design-switcher lifecycle from client disclosure through hydration regression and server-first reduction
- separated global 404 recovery from project dynamic-route creation
- consolidated duplicated page-enablement work into one availability Thread
- added exact URL helper tests from `3353032ba23b`
- retained only commits whose code materially belongs to this category boundary

The resulting seven-Thread scaffold is frozen for Phase 2.

## Frozen file set

| Thread | File | Engineering story | Commits |
| ---: | --- | --- | ---: |
| 01 | `01-query-state-and-route-preserving-navigation.md` | Query State and Route-Preserving Navigation | 4 |
| 02 | `02-shared-shell-navigation-and-mobile-menu.md` | Shared Shell Navigation and Mobile Menu | 6 |
| 03 | `03-native-design-switcher-page-lifecycle.md` | Native Design Switcher Page Lifecycle | 7 |
| 04 | `04-project-index-and-dynamic-detail-lifecycle.md` | Project Index and Dynamic Detail Lifecycle | 2 |
| 05 | `05-page-availability-and-auxiliary-route-lifecycle.md` | Page Availability and Auxiliary Route Lifecycle | 7 |
| 06 | `06-custom-not-found-recovery.md` | Custom Not-Found Recovery | 2 |
| 07 | `07-page-context-consolidation.md` | Page Context Consolidation | 5 |

`README.md` plus these seven files are the complete category file set. The completed directory must have the exact same relative paths and no extras.

## Historical validation basis

- Branch-local classification: `commit/commit-importance.md`
- Exact commit inspection through the GitHub connector
- The classification source describes `web/portfolio` as a complete independent linear root-to-head history
- Ancestry anchors were checked with GitHub compare: the earliest referenced `902eddcef875` and latest referenced `b669e04c0932` are merge-base ancestors of `web/portfolio` with `behind_by: 0`
- Every frozen SHA is present in the branch-local classification and was resolved to its exact commit subject/diff

## Completion and execution record

- Phase 2 completion: seven Thread documents were completed from the frozen scaffold; `README.md` is the eighth corresponding file.
- Repository evidence: every frozen commit was inspected at its exact SHA through the GitHub connector, with subjects/diffs matched to branch-local `commit/commit-importance.md`.
- Ancestry validation: all 33 SHAs are present in the branch-local complete linear classification; GitHub compare additionally returned each ancestry anchor as its own merge base against `web/portfolio` with `behind_by: 0` (`902eddcef875` → 464 later commits, `b669e04c0932` → 29 later commits).
- Repository test execution: not performed. A source checkout was unavailable in the local runtime.
- Executed environment check:
  - Command: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git web/portfolio`
  - Result: exit status `128`; `Could not resolve host: github.com`
- Evidence therefore distinguishes exact historical code/test inspection from runtime execution. No Vitest, Playwright, Next build, or server command is reported as executed.
- Local deliverable validation covered exact file-set parity, frozen scaffold checksums, fixed commit-map parity, unique SHA count, heading parity, balanced code fences, unfinished-marker removal, importance-depth differentiation, archive contents, and ZIP integrity.
===== END FILE: README.md =====

