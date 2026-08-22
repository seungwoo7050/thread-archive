===== BEGIN FILE: 01-cross-design-token-shell-and-regression-contracts.md =====
# Thread: Cross-design token, shell, and regression contracts

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `05-full-site-visual-systems`
>
> Phase 1 audit에서 확정한 authoritative scaffold입니다. Phase 2에서는 answer marker 내부만 채웁니다.

## 0. Scope and authority

- Commit SHA, subject, importance, tags는 branch의 `commit/commit-importance.md`와 exact commit metadata를 기준으로 고정했습니다.
- **Thread boundary:** 공통 registry·renderer input·App Router delegation·Design/Classic shell boundary·activation·cross-design regression만 포함합니다. Editorial/Brutalist/Cinematic 내부 조립은 각 전용 Thread에, Design/Classic의 route별 extraction은 Thread 5에 둡니다.
- 다른 branch, final HEAD의 후대 구현, 실행하지 않은 command 결과를 사용하지 않습니다.

## 1. Thread goal

다섯 디자인이 독립적인 full-site renderer로 동작하면서도 route vocabulary, prepared input, shell state, token 역할과 회귀 검증 경계를 공유하게 되는 과정을 복원합니다.

### Frozen invariant target

최종 invariant는 App Router가 framework concerns와 exact route view model을 소유하고, registry가 디자인을 선택하며, 각 renderer는 discriminated prepared input만 받아 자체 shell/presentation을 반환한다는 것입니다.

## 2. Commit map

| 순서 | Commit | Subject | Importance | Tags | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `418e7bc1d8bb` | feat(designs): site design 정의 registry 추가 | A | ARCH, RENDERER | 사이트 디자인 vocabulary와 선택 registry의 시작점 |
| 2 | `e14202198948` | feat(designs): route renderer 계약 추가 | A | ARCH, ROUTING, RENDERER | 여덟 public route를 닫힌 union으로 표현하는 첫 renderer 계약 |
| 3 | `6fc28f4c6586` | refactor(designs): 확장 renderer lazy registry 추가 | A | ARCH, RENDERER, REFACTOR | 확장 디자인의 capability 조회와 lazy loader 경계 |
| 4 | `dc2cf72a768d` | refactor(routes): 확장 디자인 renderer 위임 경계 추가 | S | ARCH, ROUTING, RENDERER | 모든 public route에서 framework 책임과 full-site presentation을 분리한 최초의 공통 경계 |
| 5 | `c6acfe562694` | feat(editorial): renderer를 디자인 registry에 활성화 | A | ARCH, RENDERER | 완성된 Editorial을 selectable full-site renderer로 승격 |
| 6 | `dd71d28143a8` | feat(designs): Brutalist renderer 활성화 | A | ARCH, RENDERER | 완성된 Brutalist를 registry·content 계약에 연결 |
| 7 | `b8de57f130eb` | feat(designs): Cinematic renderer 활성화 | A | ARCH, RENDERER | Cinematic의 선택 가능 상태와 module API 축소 |
| 8 | `1598a87702f6` | test(design): view model 기반 renderer matrix 검증 | A | ARCH, VALIDATION, RENDERER | 다섯 디자인×여섯 migrated route의 정적 DOM compatibility matrix |
| 9 | `4732301a7d2c` | style(designs): route renderer 디자인 토큰 확장 | A | ARCH, ROUTING, RENDERER | color 외 일곱 cross-design token family의 도입 |
| 10 | `969741c3469d` | refactor(shell): 디자인 renderer 셸 경계 추가 | A | ARCH, ROUTING, RENDERER | Design·Classic도 prepared shell props를 받아 독립 root marker를 갖는 전환 |
| 11 | `8a48460df4c3` | refactor(journey): 모든 renderer에 여정 view model 적용 | A | ARCH, RENDERER, REFACTOR | 여정 reference resolution을 renderer 밖으로 이동 |
| 12 | `aef265b9bd01` | refactor(interview): 모든 renderer에 인터뷰 view model 적용 | A | ARCH, RENDERER, REFACTOR | 인터뷰 answer-project join을 route view model에 통일 |
| 13 | `f8b0ab7b08aa` | refactor(designs): renderer 입력을 route view model로 제한 | S | ARCH, ROUTING, RENDERER | renderer가 전역 content graph를 읽지 못하게 하는 discriminated input invariant |
| 14 | `380b2a025070` | refactor(designs): 모든 route를 registry renderer로 위임 | S | ARCH, ROUTING, RENDERER | 여덟 App Router page와 다섯 디자인을 하나의 dispatch architecture로 폐쇄 |
| 15 | `055b733cbb7e` | test(design): 독립 renderer와 design token 경계 검증 | A | ARCH, VALIDATION, RENDERER | 8×5 route matrix와 token/renderer marker의 정적 회귀 계약 |
| 16 | `882a2f9d753e` | test(visual): 다섯 디자인 회귀 기준 추가 | A | TEST | Playwright screenshot baseline과 snapshot manifest 계약 |

## 3. Historical baseline

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:baseline:BEGIN -->
- **직전 상태:** 초기에는 `SiteDesignId`와 route-specific presentation ownership이 App Router와 shared components에 분산돼 있었고, 확장 디자인은 registry 후보일 뿐 public route에서 실행되지 않았습니다.
- **경계 판단:** 공통 registry·renderer input·App Router delegation·Design/Classic shell boundary·activation·cross-design regression만 포함합니다. Editorial/Brutalist/Cinematic 내부 조립은 각 전용 Thread에, Design/Classic의 route별 extraction은 Thread 5에 둡니다.
- **복원 기준:** 각 commit의 parent와 exact SHA tree만 사용하고 final HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:baseline:END -->

## 4. Commit-by-commit reconstruction

### 1. `418e7bc1d8bb` — feat(designs): site design 정의 registry 추가

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Thread role:** 사이트 디자인 vocabulary와 선택 registry의 시작점

#### Commit-specific investigation

- `418e7bc1d8bb^`와 `418e7bc1d8bb`를 비교하고 `src/designs/config.ts`에서 **`SiteDesignDefinition`, 디자인 ID 목록, 기본 디자인 선택과 unknown ID fallback**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `사이트 디자인 vocabulary와 선택 registry의 시작점` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `등록된 디자인이 실제 모든 route를 렌더링한다는 보장은 아직 없다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: 다음 `e14202198948`가 이 ID vocabulary 위에 route renderer 입력 계약을 얹는다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:418e7bc1d8bb:BEGIN -->
- **직전 상태:** 직전 상태에는 **`SiteDesignDefinition`, 디자인 ID 목록, 기본 디자인 선택과 unknown ID fallback**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **구현 결정:** 이 커밋은 `design`과 `classic`을 데이터 정의로 등록하고, 조회 실패 시 첫 정의를 반환하는 registry를 만든다. 디자인 선택은 더 이상 route별 문자열 분기에만 의존하지 않지만, 아직 route renderer 계약은 없다.
- **파일·symbol:** `src/designs/config.ts`에서 `SiteDesignDefinition`, 디자인 ID 목록, 기본 디자인 선택과 unknown ID fallback를 확인했습니다.
- **소유권:** `src/designs/config.ts`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **보장/비보장:** 이 SHA는 `사이트 디자인 vocabulary와 선택 registry의 시작점` 경계를 고정합니다. 등록된 디자인이 실제 모든 route를 렌더링한다는 보장은 아직 없다.
- **역사적 연결:** 다음 `e14202198948`가 이 ID vocabulary 위에 route renderer 입력 계약을 얹는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:418e7bc1d8bb:END -->

### 2. `e14202198948` — feat(designs): route renderer 계약 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** 여덟 public route를 닫힌 union으로 표현하는 첫 renderer 계약

#### Commit-specific investigation

- `e14202198948^`와 `e14202198948`를 비교하고 `src/designs/types.ts`에서 **`DesignRouteName`, `DesignRouteProps`, route별 optional project/currentPath/contentDebug 입력**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `여덟 public route를 닫힌 union으로 표현하는 첫 renderer 계약` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `타입 정의만으로 App Router가 위임하거나 미지원 renderer를 처리하지는 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `6fc28f4c6586`가 이 타입을 lazy registry의 loader 경계로 사용하고, `f8b0ab7b08aa`가 나중에 raw-content 접근을 금지한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:e14202198948:BEGIN -->
- **직전 상태:** 직전 상태에는 **`DesignRouteName`, `DesignRouteProps`, route별 optional project/currentPath/contentDebug 입력**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **구현 결정:** home, projects, project-detail, about, resume, contact, journey, interview-map을 닫힌 route union으로 고정하고 renderer가 받을 공통 props를 정의한다. 이 시점 입력은 여전히 전체 `PortfolioContent`라서 renderer가 전역 콘텐츠 그래프에 접근할 수 있다.
- **파일·symbol:** `src/designs/types.ts`에서 `DesignRouteName`, `DesignRouteProps`, route별 optional project/currentPath/contentDebug 입력를 확인했습니다.
- **소유권:** `src/designs/types.ts`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **보장/비보장:** 이 SHA는 `여덟 public route를 닫힌 union으로 표현하는 첫 renderer 계약` 경계를 고정합니다. 타입 정의만으로 App Router가 위임하거나 미지원 renderer를 처리하지는 않는다.
- **역사적 연결:** `6fc28f4c6586`가 이 타입을 lazy registry의 loader 경계로 사용하고, `f8b0ab7b08aa`가 나중에 raw-content 접근을 금지한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:e14202198948:END -->

### 3. `6fc28f4c6586` — refactor(designs): 확장 renderer lazy registry 추가

- **Importance:** A
- **Tags:** ARCH, RENDERER, REFACTOR
- **Thread role:** 확장 디자인의 capability 조회와 lazy loader 경계

#### Commit-specific investigation

- `6fc28f4c6586^`와 `6fc28f4c6586`를 비교하고 `src/designs/registry.tsx`에서 **`routeLoaders`, `hasDedicatedRouteRenderer`, `renderDesignRoute`의 null 반환 경로**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `확장 디자인의 capability 조회와 lazy loader 경계` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `lazy registry는 fallback UI나 오류 보고를 제공하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `dc2cf72a768d`가 public route에서 이 null 가능 경계를 소비하고, 각 activation 커밋이 loader를 채운다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:6fc28f4c6586:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`routeLoaders`, `hasDedicatedRouteRenderer`, `renderDesignRoute`의 null 반환 경로**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **구현 결정:** Editorial·Brutalist·Cinematic을 dedicated renderer 후보로 분류하지만 `routeLoaders`가 비어 있어 실제 loader가 없으면 `renderDesignRoute`는 `null`을 반환한다. capability와 구현 등록을 분리해 점진적 활성화가 가능해졌으나, 이 시점에는 확장 디자인을 선택해도 전용 화면이 보장되지 않는다.
- **파일·symbol:** `src/designs/registry.tsx`에서 `routeLoaders`, `hasDedicatedRouteRenderer`, `renderDesignRoute`의 null 반환 경로를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/designs/registry.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **보장/비보장:** 이 SHA는 `확장 디자인의 capability 조회와 lazy loader 경계` 경계를 고정합니다. lazy registry는 fallback UI나 오류 보고를 제공하지 않는다.
- **역사적 연결:** `dc2cf72a768d`가 public route에서 이 null 가능 경계를 소비하고, 각 activation 커밋이 loader를 채운다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```tsx
// 6fc28f4c6586 · src/designs/registry.tsx
const loader = routeLoaders[designId];
if (!loader) return null;
```
<!-- LEARNER-ANSWER:commit:6fc28f4c6586:END -->

### 4. `dc2cf72a768d` — refactor(routes): 확장 디자인 renderer 위임 경계 추가

- **Importance:** S
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** 모든 public route에서 framework 책임과 full-site presentation을 분리한 최초의 공통 경계

#### Commit-specific investigation

- `dc2cf72a768d^`와 `dc2cf72a768d`를 비교하고 `src/app/page.tsx 외 7개 public route page, src/designs/registry.tsx`에서 **각 page의 content loading/notFound/metadata 이후 `hasDedicatedRouteRenderer`와 `renderDesignRoute` 호출, null 결과와 Design·Classic fallback**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `모든 public route에서 framework 책임과 full-site presentation을 분리한 최초의 공통 경계` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 단계의 공통 입력은 raw content이며, route별 view model 제한과 단일 dispatch는 아직 아니다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Editorial/Brutalist/Cinematic activation이 이 경계를 실제 구현에 연결하고, `380b2a025070`이 최종적으로 Design·Classic까지 같은 경로로 통일한다.
- App Router → view-model/content boundary → registry/dispatcher → shell/view의 전체 call path를 parent와 비교합니다.
- 호환 branch가 남아 있는지, 제거되는 시점은 언제인지, 이 invariant가 다른 네 Thread에 미치는 영향을 연결합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:dc2cf72a768d:BEGIN -->
#### 복원 결과
- **직전 상태와 위험:** 동작 자체는 앞선 구현에 존재했지만 **각 page의 content loading/notFound/metadata 이후 `hasDedicatedRouteRenderer`와 `renderDesignRoute` 호출, null 결과와 Design·Classic fallback**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다. 이 분산 상태는 route/design마다 다른 파생·fallback·shell 처리를 허용해 같은 입력이 다른 실행 경로를 탈 위험이 있었습니다.
- **핵심 결정:** 직전에는 registry가 있어도 public route가 이를 호출하지 않아 확장 디자인이 실행될 수 없었다. 이 커밋은 여덟 App Router page가 route 이름·현재 경로·필요한 project를 준비한 뒤 registry로 넘기게 한다. App Router는 framework 정책을 유지하고, dedicated renderer는 전체 presentation을 소유한다. 아직 Design·Classic은 기존 shared path로 남고 loader 미등록 결과는 기존 화면으로 후퇴한다.
- **실제 inspection 지점:** `src/app/page.tsx 외 7개 public route page, src/designs/registry.tsx`에서 각 page의 content loading/notFound/metadata 이후 `hasDedicatedRouteRenderer`와 `renderDesignRoute` 호출, null 결과와 Design·Classic fallback를 parent diff와 resulting tree로 추적했습니다.
- **책임과 상태 전환:** 표현·조립 책임은 `src/app/page.tsx 외 7개 public route page, src/designs/registry.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **실패·비보장:** 이 단계의 공통 입력은 raw content이며, route별 view model 제한과 단일 dispatch는 아직 아니다.
- **후속 폐쇄:** Editorial/Brutalist/Cinematic activation이 이 경계를 실제 구현에 연결하고, `380b2a025070`이 최종적으로 Design·Classic까지 같은 경로로 통일한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```tsx
// dc2cf72a768d · public route pages
const rendered = await renderDesignRoute(activeTemplate, {
  route: "...",
  content,
  currentPath,
  contentDebug,
});
if (rendered) return rendered;
```
<!-- LEARNER-ANSWER:commit:dc2cf72a768d:END -->

### 5. `c6acfe562694` — feat(editorial): renderer를 디자인 registry에 활성화

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Thread role:** 완성된 Editorial을 selectable full-site renderer로 승격

#### Commit-specific investigation

- `c6acfe562694^`와 `c6acfe562694`를 비교하고 `src/designs/registry.tsx, src/designs/config.ts, src/content/presentation.json, src/lib/portfolio/content-loader.ts`에서 **Editorial loader 등록, 디자인 정의·표시 copy·content validation의 동시 확장**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `완성된 Editorial을 selectable full-site renderer로 승격` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `활성화는 시각 정확성이나 모든 content 조합을 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 2의 `46e23d922c2e` dispatcher가 선행 구현이며, 이 커밋은 그 결과를 공통 선택 경계에 연결한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:c6acfe562694:BEGIN -->
- **직전 상태:** 직전 상태에는 **Editorial loader 등록, 디자인 정의·표시 copy·content validation의 동시 확장**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **구현 결정:** Editorial module이 존재하는 것과 사용자가 선택 가능한 것은 별개였다. 이 커밋은 registry loader와 config/content 경계를 함께 갱신해 `editorial` 선택이 여덟 route renderer로 도달하도록 한다.
- **파일·symbol:** `src/designs/registry.tsx, src/designs/config.ts, src/content/presentation.json, src/lib/portfolio/content-loader.ts`에서 Editorial loader 등록, 디자인 정의·표시 copy·content validation의 동시 확장를 확인했습니다.
- **소유권:** `src/designs/registry.tsx, src/designs/config.ts, src/content/presentation.json, src/lib/portfolio/content-loader.ts`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **보장/비보장:** 이 SHA는 `완성된 Editorial을 selectable full-site renderer로 승격` 경계를 고정합니다. 활성화는 시각 정확성이나 모든 content 조합을 검증하지 않는다.
- **역사적 연결:** Thread 2의 `46e23d922c2e` dispatcher가 선행 구현이며, 이 커밋은 그 결과를 공통 선택 경계에 연결한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:c6acfe562694:END -->

### 6. `dd71d28143a8` — feat(designs): Brutalist renderer 활성화

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Thread role:** 완성된 Brutalist를 registry·content 계약에 연결

#### Commit-specific investigation

- `dd71d28143a8^`와 `dd71d28143a8`를 비교하고 `src/designs/registry.tsx, src/designs/config.ts, presentation/content validation files`에서 **Brutalist loader와 selectable design metadata, route entry point**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `완성된 Brutalist를 registry·content 계약에 연결` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `CSS breakpoint와 route별 empty state의 품질은 activation 자체가 보장하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 3의 `caa7df81d899` 단일 entry point를 소비한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:dd71d28143a8:BEGIN -->
- **직전 상태:** 직전 상태에는 **Brutalist loader와 selectable design metadata, route entry point**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **구현 결정:** Brutalist의 내부 route 집합이 완성된 뒤 registry가 module entry를 lazy-load하도록 바뀐다. 선택 ID, 사용자 노출 copy, loader가 한 commit에서 일치하므로 unknown/registered/implemented 상태가 갈라지지 않는다.
- **파일·symbol:** `src/designs/registry.tsx, src/designs/config.ts, presentation/content validation files`에서 Brutalist loader와 selectable design metadata, route entry point를 확인했습니다.
- **소유권:** `src/designs/registry.tsx, src/designs/config.ts, presentation/content validation files`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **보장/비보장:** 이 SHA는 `완성된 Brutalist를 registry·content 계약에 연결` 경계를 고정합니다. CSS breakpoint와 route별 empty state의 품질은 activation 자체가 보장하지 않는다.
- **역사적 연결:** Thread 3의 `caa7df81d899` 단일 entry point를 소비한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:dd71d28143a8:END -->

### 7. `b8de57f130eb` — feat(designs): Cinematic renderer 활성화

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Thread role:** Cinematic의 선택 가능 상태와 module API 축소

#### Commit-specific investigation

- `b8de57f130eb^`와 `b8de57f130eb`를 비교하고 `src/designs/registry.tsx, src/designs/config.ts, src/designs/cinematic/cinematic-route.tsx, content files`에서 **Cinematic loader 등록과 route entry export만 남기는 공개 범위**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic의 선택 가능 상태와 module API 축소` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `별도 exhaustive dispatcher 함수가 추가되는 것은 아니며, route entry 내부의 switch/분기 품질에 의존한다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 4의 route composition 전체를 공통 registry에 연결한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b8de57f130eb:BEGIN -->
- **직전 상태:** 직전 상태에는 **Cinematic loader 등록과 route entry export만 남기는 공개 범위**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **구현 결정:** Cinematic을 selectable design으로 등록하면서 내부 view/helper export를 제거하고 registry가 route entry 하나만 알도록 정리한다. activation과 module ownership 축소가 함께 일어나 registry가 내부 조립 세부사항에 결합되지 않는다.
- **파일·symbol:** `src/designs/registry.tsx, src/designs/config.ts, src/designs/cinematic/cinematic-route.tsx, content files`에서 Cinematic loader 등록과 route entry export만 남기는 공개 범위를 확인했습니다.
- **소유권:** `src/designs/registry.tsx, src/designs/config.ts, src/designs/cinematic/cinematic-route.tsx, content files`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **보장/비보장:** 이 SHA는 `Cinematic의 선택 가능 상태와 module API 축소` 경계를 고정합니다. 별도 exhaustive dispatcher 함수가 추가되는 것은 아니며, route entry 내부의 switch/분기 품질에 의존한다.
- **역사적 연결:** Thread 4의 route composition 전체를 공통 registry에 연결한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b8de57f130eb:END -->

### 8. `1598a87702f6` — test(design): view model 기반 renderer matrix 검증

- **Importance:** A
- **Tags:** ARCH, VALIDATION, RENDERER
- **Thread role:** 다섯 디자인×여섯 migrated route의 정적 DOM compatibility matrix

#### Commit-specific investigation

- `1598a87702f6^`와 `1598a87702f6`를 비교하고 `src/designs/route-view-models.test.tsx`에서 **`designIds`, `routes`, Testing Library render, `data-site-design`와 non-empty `h1` assertion**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `다섯 디자인×여섯 migrated route의 정적 DOM compatibility matrix` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `브라우저 레이아웃, CSS 적용, interaction, journey/interview는 이 SHA의 검증 범위가 아니다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `055b733cbb7e`가 journey/interview를 추가하고 Design·Classic 독립 renderer marker를 검증한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1598a87702f6:BEGIN -->
- **직전 상태:** 관련 production 경계는 존재했지만 **`designIds`, `routes`, Testing Library render, `data-site-design`와 non-empty `h1` assertion**를 실패 시 자동으로 탐지하는 회귀 증거가 없었습니다.
- **구현 결정:** Vitest/jsdom에서 home·projects·project-detail·about·resume·contact를 다섯 디자인으로 직접 호출한다. enabled project가 없으면 fixture 전제 오류를 내고, 각 조합에서 디자인 root와 비어 있지 않은 `h1`을 확인한다. 이는 30개 조합의 server component 결과가 기본 HTML 경계를 유지함을 증명한다.
- **파일·symbol:** `src/designs/route-view-models.test.tsx`에서 `designIds`, `routes`, Testing Library render, `data-site-design`와 non-empty `h1` assertion를 확인했습니다.
- **소유권:** production owner는 변경하지 않고 `src/designs/route-view-models.test.tsx`가 회귀 판정과 fixture 전제를 소유합니다.
- **보장/비보장:** 이 SHA는 `다섯 디자인×여섯 migrated route의 정적 DOM compatibility matrix` 경계를 고정합니다. 브라우저 레이아웃, CSS 적용, interaction, journey/interview는 이 SHA의 검증 범위가 아니다.
- **역사적 연결:** `055b733cbb7e`가 journey/interview를 추가하고 Design·Classic 독립 renderer marker를 검증한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1598a87702f6:END -->

### 9. `4732301a7d2c` — style(designs): route renderer 디자인 토큰 확장

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** color 외 일곱 cross-design token family의 도입

#### Commit-specific investigation

- `4732301a7d2c^`와 `4732301a7d2c`를 비교하고 `src/app/globals.css`에서 **`--type-display`, `--type-body`, `--space-section`, `--breakpoint-content`, `--motion-fast`, `--layer-navigation`, `--content-width`의 design scope**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `color 외 일곱 cross-design token family의 도입` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `모든 renderer가 모든 토큰을 실제 소비하거나 시각적으로 동일한 의미를 갖는다는 보장은 없다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `055b733cbb7e`의 정적 token contract가 이름 존재와 Design·Classic scope를 보호한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4732301a7d2c:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **`--type-display`, `--type-body`, `--space-section`, `--breakpoint-content`, `--motion-fast`, `--layer-navigation`, `--content-width`의 design scope**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **구현 결정:** 각 `[data-site-design]` scope에 typography, rhythm, breakpoint value, motion, navigation layer, content width를 정의한다. 공통 consumer는 특히 content width를 사용하지만, CSS custom property에 breakpoint 숫자를 둔 것만으로 media query가 자동 적용되지는 않는다.
- **파일·symbol:** `src/app/globals.css`에서 `--type-display`, `--type-body`, `--space-section`, `--breakpoint-content`, `--motion-fast`, `--layer-navigation`, `--content-width`의 design scope를 확인했습니다.
- **소유권:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/app/globals.css`의 scoped stylesheet가 소유합니다.
- **보장/비보장:** 이 SHA는 `color 외 일곱 cross-design token family의 도입` 경계를 고정합니다. 모든 renderer가 모든 토큰을 실제 소비하거나 시각적으로 동일한 의미를 갖는다는 보장은 없다.
- **역사적 연결:** `055b733cbb7e`의 정적 token contract가 이름 존재와 Design·Classic scope를 보호한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4732301a7d2c:END -->

### 10. `969741c3469d` — refactor(shell): 디자인 renderer 셸 경계 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** Design·Classic도 prepared shell props를 받아 독립 root marker를 갖는 전환

#### Commit-specific investigation

- `969741c3469d^`와 `969741c3469d`를 비교하고 `src/components/portfolio/site-shell.tsx, src/designs/shell-props.ts`에서 **`createDesignShellProps`, `data-route-renderer`, footerLinks/templateSwitcher/currentPath 조립**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Design·Classic도 prepared shell props를 받아 독립 root marker를 갖는 전환` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `Editorial·Brutalist·Cinematic의 자체 shell을 이 helper로 강제하지는 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 5의 Design·Classic extraction이 이 helper를 소비하고, `055b733cbb7e`가 marker를 검사한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:969741c3469d:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createDesignShellProps`, `data-route-renderer`, footerLinks/templateSwitcher/currentPath 조립**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **구현 결정:** 공용 `PageShell`에 renderer marker를 선택적으로 붙이고, route renderer가 필요한 shell props를 한 helper에서 조립한다. footer/navigation/debug/current path의 반복 해석이 App Router나 개별 section에서 분산되지 않게 된다.
- **파일·symbol:** `src/components/portfolio/site-shell.tsx, src/designs/shell-props.ts`에서 `createDesignShellProps`, `data-route-renderer`, footerLinks/templateSwitcher/currentPath 조립를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/components/portfolio/site-shell.tsx, src/designs/shell-props.ts` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **보장/비보장:** 이 SHA는 `Design·Classic도 prepared shell props를 받아 독립 root marker를 갖는 전환` 경계를 고정합니다. Editorial·Brutalist·Cinematic의 자체 shell을 이 helper로 강제하지는 않는다.
- **역사적 연결:** Thread 5의 Design·Classic extraction이 이 helper를 소비하고, `055b733cbb7e`가 marker를 검사한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:969741c3469d:END -->

### 11. `8a48460df4c3` — refactor(journey): 모든 renderer에 여정 view model 적용

- **Importance:** A
- **Tags:** ARCH, RENDERER, REFACTOR
- **Thread role:** 여정 reference resolution을 renderer 밖으로 이동

#### Commit-specific investigation

- `8a48460df4c3^`와 `8a48460df4c3`를 비교하고 `src/app/journey/page.tsx, src/lib/portfolio/view-models.ts, 다섯 renderer의 journey path`에서 **`createJourneyViewModel`, resolved milestones/anchor projects, raw `PortfolioContent` 의존 제거**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `여정 reference resolution을 renderer 밖으로 이동` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `표현 방식과 empty-state copy는 각 renderer가 계속 소유한다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `f8b0ab7b08aa`가 이 전환을 공통 type contract로 강제한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:8a48460df4c3:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createJourneyViewModel`, resolved milestones/anchor projects, raw `PortfolioContent` 의존 제거**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **구현 결정:** Journey route가 prepared view model을 만들고 다섯 renderer가 동일한 resolved milestone 구조를 소비하도록 바뀐다. project ID lookup과 누락 판단이 renderer별로 반복되지 않아 같은 source가 디자인마다 다른 링크 결과를 만들 위험을 줄인다.
- **파일·symbol:** `src/app/journey/page.tsx, src/lib/portfolio/view-models.ts, 다섯 renderer의 journey path`에서 `createJourneyViewModel`, resolved milestones/anchor projects, raw `PortfolioContent` 의존 제거를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/app/journey/page.tsx, src/lib/portfolio/view-models.ts, 다섯 renderer의 journey path` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **보장/비보장:** 이 SHA는 `여정 reference resolution을 renderer 밖으로 이동` 경계를 고정합니다. 표현 방식과 empty-state copy는 각 renderer가 계속 소유한다.
- **역사적 연결:** `f8b0ab7b08aa`가 이 전환을 공통 type contract로 강제한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:8a48460df4c3:END -->

### 12. `aef265b9bd01` — refactor(interview): 모든 renderer에 인터뷰 view model 적용

- **Importance:** A
- **Tags:** ARCH, RENDERER, REFACTOR
- **Thread role:** 인터뷰 answer-project join을 route view model에 통일

#### Commit-specific investigation

- `aef265b9bd01^`와 `aef265b9bd01`를 비교하고 `src/app/interview-map/page.tsx, src/lib/portfolio/view-models.ts, 다섯 renderer의 interview path`에서 **`createInterviewMapViewModel`, answer의 resolved project 또는 null, track/question hierarchy 보존**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `인터뷰 answer-project join을 route view model에 통일` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `null을 어떤 문구·레이아웃으로 보여 줄지는 cross-design 공통 보장이 아니다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `f8b0ab7b08aa`의 discriminated union에 interview variant가 들어간다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:aef265b9bd01:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createInterviewMapViewModel`, answer의 resolved project 또는 null, track/question hierarchy 보존**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **구현 결정:** 각 renderer가 `Map`이나 `find`로 answer project를 다시 찾던 책임을 view-model boundary로 옮긴다. 누락 참조는 `null`로 명시되어 presentation은 링크 생성이 아니라 표시 정책만 결정한다.
- **파일·symbol:** `src/app/interview-map/page.tsx, src/lib/portfolio/view-models.ts, 다섯 renderer의 interview path`에서 `createInterviewMapViewModel`, answer의 resolved project 또는 null, track/question hierarchy 보존를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/app/interview-map/page.tsx, src/lib/portfolio/view-models.ts, 다섯 renderer의 interview path` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **보장/비보장:** 이 SHA는 `인터뷰 answer-project join을 route view model에 통일` 경계를 고정합니다. null을 어떤 문구·레이아웃으로 보여 줄지는 cross-design 공통 보장이 아니다.
- **역사적 연결:** `f8b0ab7b08aa`의 discriminated union에 interview variant가 들어간다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:aef265b9bd01:END -->

### 13. `f8b0ab7b08aa` — refactor(designs): renderer 입력을 route view model로 제한

- **Importance:** S
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** renderer가 전역 content graph를 읽지 못하게 하는 discriminated input invariant

#### Commit-specific investigation

- `f8b0ab7b08aa^`와 `f8b0ab7b08aa`를 비교하고 `src/designs/types.ts, src/designs/registry.tsx, src/lib/portfolio/view-models.ts, renderer entry files`에서 **`PreparedDesignRouteProps`, route discriminator별 view-model variant, footerLinks와 resolved references**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `renderer가 전역 content graph를 읽지 못하게 하는 discriminated input invariant` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `런타임 외부 JSON이 이미 검증됐다는 사실이나 CSS/DOM 품질을 이 타입만으로 보장하지는 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `380b2a025070`이 모든 App Router page를 이 계약의 단일 caller로 만든다.
- App Router → view-model/content boundary → registry/dispatcher → shell/view의 전체 call path를 parent와 비교합니다.
- 호환 branch가 남아 있는지, 제거되는 시점은 언제인지, 이 invariant가 다른 네 Thread에 미치는 영향을 연결합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f8b0ab7b08aa:BEGIN -->
#### 복원 결과
- **직전 상태와 위험:** 동작 자체는 앞선 구현에 존재했지만 **`PreparedDesignRouteProps`, route discriminator별 view-model variant, footerLinks와 resolved references**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다. 이 분산 상태는 route/design마다 다른 파생·fallback·shell 처리를 허용해 같은 입력이 다른 실행 경로를 탈 위험이 있었습니다.
- **핵심 결정:** 직전까지 migration 호환 때문에 raw `PortfolioContent`와 prepared model이 함께 통과할 수 있었다. 이 커밋은 공통 props를 route discriminator와 정확히 대응하는 view-model union으로 바꿔 renderer가 다른 route 데이터나 전역 content graph를 읽는 것을 타입 수준에서 금지한다. 데이터 파생·참조 해석은 route/content boundary가 소유하고 renderer는 presentation만 소유한다.
- **실제 inspection 지점:** `src/designs/types.ts, src/designs/registry.tsx, src/lib/portfolio/view-models.ts, renderer entry files`에서 `PreparedDesignRouteProps`, route discriminator별 view-model variant, footerLinks와 resolved references를 parent diff와 resulting tree로 추적했습니다.
- **책임과 상태 전환:** 표현·조립 책임은 `src/designs/types.ts, src/designs/registry.tsx, src/lib/portfolio/view-models.ts, renderer entry files` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **실패·비보장:** 런타임 외부 JSON이 이미 검증됐다는 사실이나 CSS/DOM 품질을 이 타입만으로 보장하지는 않는다.
- **후속 폐쇄:** `380b2a025070`이 모든 App Router page를 이 계약의 단일 caller로 만든다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```ts
// f8b0ab7b08aa · renderer contract
type PreparedDesignRouteProps =
  | { route: "home"; content: HomeViewModel; ... }
  | { route: "projects"; content: ProjectsViewModel; ... }
  | ...;
```
<!-- LEARNER-ANSWER:commit:f8b0ab7b08aa:END -->

### 14. `380b2a025070` — refactor(designs): 모든 route를 registry renderer로 위임

- **Importance:** S
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** 여덟 App Router page와 다섯 디자인을 하나의 dispatch architecture로 폐쇄

#### Commit-specific investigation

- `380b2a025070^`와 `380b2a025070`를 비교하고 `src/app/page.tsx 외 7개 page, src/designs/registry.tsx, Design/Classic entry modules`에서 **page별 exact view-model creation → `renderDesignRoute` → registry loader/dispatcher; direct Design·Classic imports 제거**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `여덟 App Router page와 다섯 디자인을 하나의 dispatch architecture로 폐쇄` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `registry가 반환한 DOM의 pixel-level correctness나 client interaction은 별도 test 책임이다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `055b733cbb7e`가 8×5 matrix와 독립 marker를 확인하며, Thread 5의 dispatchers가 Design·Classic branch를 제공한다.
- App Router → view-model/content boundary → registry/dispatcher → shell/view의 전체 call path를 parent와 비교합니다.
- 호환 branch가 남아 있는지, 제거되는 시점은 언제인지, 이 invariant가 다른 네 Thread에 미치는 영향을 연결합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:380b2a025070:BEGIN -->
#### 복원 결과
- **직전 상태와 위험:** 동작 자체는 앞선 구현에 존재했지만 **page별 exact view-model creation → `renderDesignRoute` → registry loader/dispatcher; direct Design·Classic imports 제거**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다. 이 분산 상태는 route/design마다 다른 파생·fallback·shell 처리를 허용해 같은 입력이 다른 실행 경로를 탈 위험이 있었습니다.
- **핵심 결정:** 이전에는 dedicated 세 디자인만 registry를 쓰고 Design·Classic은 page별 직접 분기로 남아 있었다. 이 커밋은 각 page가 framework 책임(content load, enablement, params, metadata/notFound)을 처리한 뒤 정확한 route view model을 만들고, 모든 디자인을 registry 하나로 호출하게 한다. renderer 선택과 presentation ownership이 App Router에서 제거된다.
- **실제 inspection 지점:** `src/app/page.tsx 외 7개 page, src/designs/registry.tsx, Design/Classic entry modules`에서 page별 exact view-model creation → `renderDesignRoute` → registry loader/dispatcher; direct Design·Classic imports 제거를 parent diff와 resulting tree로 추적했습니다.
- **책임과 상태 전환:** 표현·조립 책임은 `src/app/page.tsx 외 7개 page, src/designs/registry.tsx, Design/Classic entry modules` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **실패·비보장:** registry가 반환한 DOM의 pixel-level correctness나 client interaction은 별도 test 책임이다.
- **후속 폐쇄:** `055b733cbb7e`가 8×5 matrix와 독립 marker를 확인하며, Thread 5의 dispatchers가 Design·Classic branch를 제공한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```tsx
// 380b2a025070 · App Router pages
return renderDesignRoute(activeTemplate, {
  route: "...",
  content: viewModel,
  currentPath,
  contentDebug,
});
```
<!-- LEARNER-ANSWER:commit:380b2a025070:END -->

### 15. `055b733cbb7e` — test(design): 독립 renderer와 design token 경계 검증

- **Importance:** A
- **Tags:** ARCH, VALIDATION, RENDERER
- **Thread role:** 8×5 route matrix와 token/renderer marker의 정적 회귀 계약

#### Commit-specific investigation

- `055b733cbb7e^`와 `055b733cbb7e`를 비교하고 `src/designs/design-tokens.test.ts, src/designs/route-view-models.test.tsx`에서 **일곱 token family 문자열 검사, journey/interview 추가, Design·Classic `data-route-renderer` assertion**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `8×5 route matrix와 token/renderer marker의 정적 회귀 계약` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `정규식 기반 CSS 검사는 token 소비·계산값·시각 대비를 증명하지 않고, jsdom은 layout을 계산하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `882a2f9d753e`가 실제 Chromium screenshot surface를 추가한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:055b733cbb7e:BEGIN -->
- **직전 상태:** 관련 production 경계는 존재했지만 **일곱 token family 문자열 검사, journey/interview 추가, Design·Classic `data-route-renderer` assertion**를 실패 시 자동으로 탐지하는 회귀 증거가 없었습니다.
- **구현 결정:** 기존 여섯 route matrix를 journey와 interview-map까지 늘려 40개 조합을 렌더링하고, Design·Classic이 shared fallback이 아니라 독립 renderer root를 냈는지 확인한다. 별도 test는 globals.css에 일곱 token family가 있고 두 renderer scope에 content-width가 있는지 정적으로 검사한다.
- **파일·symbol:** `src/designs/design-tokens.test.ts, src/designs/route-view-models.test.tsx`에서 일곱 token family 문자열 검사, journey/interview 추가, Design·Classic `data-route-renderer` assertion를 확인했습니다.
- **소유권:** production owner는 변경하지 않고 `src/designs/design-tokens.test.ts, src/designs/route-view-models.test.tsx`가 회귀 판정과 fixture 전제를 소유합니다.
- **보장/비보장:** 이 SHA는 `8×5 route matrix와 token/renderer marker의 정적 회귀 계약` 경계를 고정합니다. 정규식 기반 CSS 검사는 token 소비·계산값·시각 대비를 증명하지 않고, jsdom은 layout을 계산하지 않는다.
- **역사적 연결:** `882a2f9d753e`가 실제 Chromium screenshot surface를 추가한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:055b733cbb7e:END -->

### 16. `882a2f9d753e` — test(visual): 다섯 디자인 회귀 기준 추가

- **Importance:** A
- **Tags:** TEST
- **Thread role:** Playwright screenshot baseline과 snapshot manifest 계약

#### Commit-specific investigation

- `882a2f9d753e^`와 `882a2f9d753e`를 비교하고 `tests/e2e/visual.spec.ts, src/designs/visual-regression-contract.test.ts, playwright.config.ts, tests/e2e/visual.spec.ts-snapshots/`에서 **`prepareStablePage`, reduced motion/networkidle/font/image 대기, 15 PNG manifest, `maxDiffPixelRatio: 0.01`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Playwright screenshot baseline과 snapshot manifest 계약` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `모든 route, 모든 project, accessibility, 동적 interaction을 포괄하지 않으며 이 작업 환경에서는 실행하지 않았다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: 이 test는 `380b2a025070` 이후 cross-design presentation의 브라우저 회귀 방어다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:882a2f9d753e:BEGIN -->
- **직전 상태:** 관련 production 경계는 존재했지만 **`prepareStablePage`, reduced motion/networkidle/font/image 대기, 15 PNG manifest, `maxDiffPixelRatio: 0.01`**를 실패 시 자동으로 탐지하는 회귀 증거가 없었습니다.
- **구현 결정:** 다섯 디자인의 home을 Chromium desktop/mobile로, 첫 enabled project detail을 desktop Chromium으로 캡처한다. 안정화를 위해 reduced motion을 강제하고 network idle·font·image 완료를 기다린다. Vitest manifest test는 정확히 15개 baseline 이름이 존재하는지 확인한다.
- **파일·symbol:** `tests/e2e/visual.spec.ts, src/designs/visual-regression-contract.test.ts, playwright.config.ts, tests/e2e/visual.spec.ts-snapshots/`에서 `prepareStablePage`, reduced motion/networkidle/font/image 대기, 15 PNG manifest, `maxDiffPixelRatio: 0.01`를 확인했습니다.
- **소유권:** production owner는 변경하지 않고 `tests/e2e/visual.spec.ts, src/designs/visual-regression-contract.test.ts, playwright.config.ts, tests/e2e/visual.spec.ts-snapshots/`가 회귀 판정과 fixture 전제를 소유합니다.
- **보장/비보장:** 이 SHA는 `Playwright screenshot baseline과 snapshot manifest 계약` 경계를 고정합니다. 모든 route, 모든 project, accessibility, 동적 interaction을 포괄하지 않으며 이 작업 환경에서는 실행하지 않았다.
- **역사적 연결:** 이 test는 `380b2a025070` 이후 cross-design presentation의 브라우저 회귀 방어다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:882a2f9d753e:END -->

## 5. Invariant evolution

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:invariant:BEGIN -->
최종 invariant는 App Router가 framework concerns와 exact route view model을 소유하고, registry가 디자인을 선택하며, 각 renderer는 discriminated prepared input만 받아 자체 shell/presentation을 반환한다는 것입니다.

- 도입·확장·폐쇄의 순서는 commit map에 고정했습니다.
- B-level construction은 route/style surface를 단계적으로 넓히고, A/S-level commit은 owner·dispatch·검증 invariant를 바꿉니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:invariant:END -->

## 6. Failure → Fix → Test and ownership relations

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:relations:BEGIN -->
`dc2cf72a768d`가 최초 위임 경계를 만들고 세 activation이 dedicated renderer를 연결합니다. `8a48460df4c3`·`aef265b9bd01`이 마지막 raw-reference join을 제거한 뒤 `f8b0ab7b08aa`가 타입으로 제한하고 `380b2a025070`이 모든 route/design을 통일합니다. `1598a87702f6`·`055b733cbb7e`·`882a2f9d753e`가 정적 DOM, token, browser screenshot의 서로 다른 층을 보호합니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:relations:END -->

## 7. Final architecture or execution flow

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:flow:BEGIN -->
요청 route → content load/enablement/params 처리 → route view model 생성 → template resolve → registry loader → 디자인 dispatcher → design-owned shell/body → static/visual contract 검증 순서입니다.

각 단계에서 optional/empty/missing reference가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 commit의 non-guarantee에 남겼습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:flow:END -->

## 8. Runtime and verification evidence

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:runtime:BEGIN -->
- **실행한 repository test/build:** 없음.
- **정적 확인:** 지정 branch의 commit classification, commit bodies, exact commit diff와 historical file 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 container에서 직접 clone 시 DNS가 `github.com`을 해석하지 못해 historical worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next build 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** code/test implementation의 존재와 범위는 inspection으로 확인했고, runtime pass/fail은 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:runtime:END -->

## 9. Learning-completion checks

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:checks:BEGIN -->
- [x] 모든 고정 SHA·subject·importance·tags를 commit map과 commit section에서 동일하게 유지했습니다.
- [x] 각 SHA에 concrete file과 symbol/selector/route focus를 기록했습니다.
- [x] 이전 상태, owner, absence/fallback, guarantee/non-guarantee, 후속 관계를 채웠습니다.
- [x] S/A/B depth를 구분했습니다.
- [x] 실행하지 않은 test를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:checks:END -->
===== END FILE: 01-cross-design-token-shell-and-regression-contracts.md =====

===== BEGIN FILE: 02-editorial-design-system-construction.md =====
# Thread: Editorial design system construction

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `05-full-site-visual-systems`
>
> Phase 1 audit에서 확정한 authoritative scaffold입니다. Phase 2에서는 answer marker 내부만 채웁니다.

## 0. Scope and authority

- Commit SHA, subject, importance, tags는 branch의 `commit/commit-importance.md`와 exact commit metadata를 기준으로 고정했습니다.
- **Thread boundary:** Editorial stylesheet와 `editorial-route.tsx` 내부 construction만 포함합니다. registry activation은 Thread 1에 둡니다. C-level formatting-only media-rule 정리는 제외했습니다.
- 다른 branch, final HEAD의 후대 구현, 실행하지 않은 command 결과를 사용하지 않습니다.

## 1. Thread goal

Editorial의 scoped stylesheet가 full-site 지면 grammar를 만들고, 하나의 module이 여덟 route와 shell·링크·빈 상태·참조 해석을 점진적으로 완성하는 과정을 복원합니다.

### Frozen invariant target

최종 invariant는 `EditorialRoute` 하나만 외부에 노출되고, route discriminator가 여덟 private view 중 하나를 선택한 뒤 항상 `EditorialShell`로 감싸며, absence/failure를 route별 명시적 UI로 표현한다는 것입니다.

## 2. Commit map

| 순서 | Commit | Subject | Importance | Tags | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7546ac248334` | style(editorial): 지면과 masthead 토큰 구성 | B | RENDERER | Editorial 시각 계약의 root token·paper texture·focus·skip-link·masthead 기초 |
| 2 | `80b86ed4a1ff` | style(editorial): wordmark와 navigation 계층 구성 | B | ROUTING, RENDERER | Editorial 시각 계약의 wordmark·desktop navigation·design switcher·footer CTA |
| 3 | `4d646fb2924a` | style(editorial): footer와 hero 활자 체계 구성 | B | RENDERER | Editorial 시각 계약의 footer와 home hero typography |
| 4 | `6434531645b7` | style(editorial): hero spread 레이아웃 구성 | B | RENDERER | Editorial 시각 계약의 12-column hero spread와 공통 section rhythm |
| 5 | `a97066f07dfd` | style(editorial): lead story와 매체 표현 구성 | B | RENDERER | Editorial 시각 계약의 lead story와 framed media |
| 6 | `a9674cf2fa94` | style(editorial): 이미지 프레임과 feature 열 구성 | B | RENDERER | Editorial 시각 계약의 image placeholder·project index row·split feature columns |
| 7 | `4708ca281c16` | style(editorial): 원칙 목록과 contact strip 구성 | B | RENDERER | Editorial 시각 계약의 principle cards·sidebar feature·tag·contact strip |
| 8 | `ebe23d211852` | style(editorial): contact와 archive 지면 구성 | B | RENDERER | Editorial 시각 계약의 contact strip 완성과 page/archive layout |
| 9 | `931226268687` | style(editorial): archive group과 case link 구성 | B | RENDERER | Editorial 시각 계약의 archive group과 case-study opening |
| 10 | `2cc074f728cb` | style(editorial): case link와 dark section 구성 | B | RENDERER | Editorial 시각 계약의 bordered links·cover·three-column narrative·dark architecture |
| 11 | `34a9c958801c` | style(editorial): dark section과 decision 열 구성 | B | RENDERER | Editorial 시각 계약의 architecture evidence·image pair·decision columns |
| 12 | `13f49ab0c1f7` | style(editorial): 결과 spread와 profile facts 구성 | B | RENDERER | Editorial 시각 계약의 result·exit navigation·missing project·profile hero |
| 13 | `124f6a6fec62` | style(editorial): profile summary와 skill group 구성 | B | RENDERER | Editorial 시각 계약의 portrait·principles grid·skills spread |
| 14 | `c28bb0a5eb01` | style(editorial): 기술 그룹과 curation 본문 구성 | B | RENDERER | Editorial 시각 계약의 technology/experience rows와 curation spread |
| 15 | `4ce0333849cc` | style(editorial): curation panel과 프로젝트 목록 구성 | B | RENDERER | Editorial 시각 계약의 criteria/category/omission panels와 project links |
| 16 | `586626a79cb1` | style(editorial): curation link와 resume 도입부 구성 | B | RENDERER | Editorial 시각 계약의 touch target·next-review panel·resume header/body |
| 17 | `21d63d1975b3` | style(editorial): resume identity와 프로젝트 행 구성 | B | RENDERER | Editorial 시각 계약의 resume definition list·numbered sections·project/training rows |
| 18 | `543f4b1062e3` | style(editorial): resume 사례와 contact 본문 구성 | B | RENDERER | Editorial 시각 계약의 resume case link·contact hero/availability/channels/notes |
| 19 | `e988e97415af` | style(editorial): contact note와 milestone link 구성 | B | RENDERER | Editorial 시각 계약의 contact notes와 journey milestone date/story |
| 20 | `1da39994d9e3` | style(editorial): milestone과 현재 방향 지면 구성 | B | RENDERER | Editorial 시각 계약의 secondary timeline과 current-position panel |
| 21 | `0c3ba4ca1d48` | style(editorial): 현재 방향과 interview track 구성 | B | RENDERER | Editorial 시각 계약의 current position typography와 sticky horizontal chapter nav |
| 22 | `af5688dd1c3a` | style(editorial): interview 답변과 근거 표현 구성 | B | RENDERER | Editorial 시각 계약의 question/evidence paired ledger |
| 23 | `0c7b77c2528a` | style(editorial): 공백 목록과 중형 화면 경계 구성 | B | RENDERER | Editorial 시각 계약의 unresolved gaps spread와 1180px adaptation |
| 24 | `a854cb45cc22` | style(editorial): tablet masthead와 hero 재배치 | B | RENDERER | Editorial 시각 계약의 tablet native disclosure와 12→8-column hero |
| 25 | `3f82e8a7c308` | style(editorial): tablet route 지면 재배치 | B | ROUTING, RENDERER | Editorial 시각 계약의 route spread의 tablet reading order |
| 26 | `10a442435e1a` | style(editorial): tablet 세부 간격 정리 | B | RENDERER | Editorial 시각 계약의 tablet journey intro readable measure |
| 27 | `afaf24796399` | style(editorial): mobile navigation과 hero 구성 | B | ROUTING, RENDERER | Editorial 시각 계약의 mobile masthead metadata·stacked grids·linear hero |
| 28 | `499c0e660caf` | style(editorial): mobile 본문과 표 구성 | B | RENDERER | Editorial 시각 계약의 mobile case/profile/resume/milestone/curation/interview reflow |
| 29 | `f7a81e0fe1d3` | style(editorial): mobile footer와 동작 감소 구성 | B | RENDERER, A11Y | Editorial 시각 계약의 small-screen footer spacing와 reduced-motion |
| 30 | `1c55d7422273` | feat(editorial): route 계약과 navigation helper 추가 | B | ROUTING, RENDERER | Editorial route boundary |
| 31 | `e078d79d24c8` | feat(editorial): debug note와 이미지 프레임 추가 | B | RENDERER | 재사용 presentation primitives |
| 32 | `1b353fe5ba7b` | feat(editorial): 콘텐츠 링크와 방향 표식 추가 | B | CONTENT, RENDERER | content-aware link primitive |
| 33 | `794615a037d3` | feat(editorial): masthead와 footer shell 추가 | B | ROUTING, RENDERER | shared `EditorialShell` |
| 34 | `b7fd9118025e` | feat(editorial): 섹션 표식과 프로젝트 인덱스 추가 | B | RENDERER | section/project list primitives |
| 35 | `5c82371743ba` | feat(editorial): 홈 hero spread 추가 | B | RENDERER | Home route 시작 |
| 36 | `96ba59901181` | feat(editorial): 홈 lead story 추가 | B | RENDERER | Home lead project |
| 37 | `4c8270522400` | feat(editorial): 홈 대표 프로젝트 목록 추가 | B | RENDERER | Home remaining featured list |
| 38 | `983131c5a266` | feat(editorial): 홈 원칙과 기술 sidebar 추가 | B | RENDERER | Home principles/system section |
| 39 | `f01b60fc368e` | feat(editorial): 홈 contact strip 추가 | B | RENDERER | Home contact CTA |
| 40 | `4e69ba2ee361` | feat(editorial): 프로젝트 archive route 추가 | B | ROUTING, RENDERER | Projects archive |
| 41 | `c722cdd08ef8` | feat(editorial): 프로젝트 상세 서사와 구조 추가 | B | RENDERER | Project detail first complete path |
| 42 | `f38556a17e8b` | feat(editorial): 프로젝트 증거와 결과 spread 추가 | B | RENDERER | Project detail completion |
| 43 | `cc1b2233287f` | feat(editorial): About 정체성과 원칙 소개 추가 | B | RENDERER | About identity |
| 44 | `5f0193979568` | feat(editorial): About 기술과 경력 소개 추가 | B | RENDERER | About skills/experience |
| 45 | `5c95665ca9d2` | feat(editorial): About 큐레이션 기준 추가 | B | CONTENT, RENDERER | feature-gated curation start |
| 46 | `4a7c3a3c9cde` | feat(editorial): About 큐레이션 범주 추가 | B | CONTENT, RENDERER | curation category project join |
| 47 | `c0d0004e9355` | feat(editorial): About 큐레이션 공백과 재검토 추가 | B | CONTENT, RENDERER | curation completion |
| 48 | `119d19ab41b1` | feat(editorial): Resume 정체성과 프로젝트 경력 추가 | B | RENDERER | Resume start |
| 49 | `4df2710fa7f9` | feat(editorial): Resume 경력과 교육 기록 추가 | B | RENDERER | Resume completion |
| 50 | `61d6952850cd` | feat(editorial): Contact desk route 추가 | B | ROUTING, RENDERER | Contact route |
| 51 | `08fa527b9b65` | feat(editorial): Journey milestone spread 추가 | B | RENDERER | Journey narrative start |
| 52 | `96b66af4d5a7` | feat(editorial): Journey timeline과 현재 방향 추가 | B | RENDERER | Journey completion |
| 53 | `5e2f37861d3d` | feat(editorial): Interview Map 소개와 chapter 추가 | B | RENDERER | Interview start |
| 54 | `94deba32f56a` | feat(editorial): Interview 답변 근거와 공백 추가 | B | RENDERER | Interview completion |
| 55 | `46e23d922c2e` | feat(editorial): route dispatcher 추가 | A | ARCH, ROUTING, RENDERER | Editorial 여덟 route와 shared shell을 하나의 public entry로 폐쇄 |

## 3. Historical baseline

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:baseline:BEGIN -->
- **직전 상태:** Editorial 선택 ID와 공통 route delegation은 존재했지만 전용 CSS, shell, route view가 없었습니다. 초기 route 구현은 raw content에서 grouping·metric·project reference를 직접 파생했습니다.
- **경계 판단:** Editorial stylesheet와 `editorial-route.tsx` 내부 construction만 포함합니다. registry activation은 Thread 1에 둡니다. C-level formatting-only media-rule 정리는 제외했습니다.
- **복원 기준:** 각 commit의 parent와 exact SHA tree만 사용하고 final HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:baseline:END -->

## 4. Commit-by-commit reconstruction

### 1. `7546ac248334` — style(editorial): 지면과 masthead 토큰 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 root token·paper texture·focus·skip-link·masthead 기초

#### Commit-specific investigation

- `7546ac248334^`와 `7546ac248334`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 root token·paper texture·focus·skip-link·masthead 기초에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 root token·paper texture·focus·skip-link·masthead 기초` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:7546ac248334:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 root token·paper texture·focus·skip-link·masthead 기초에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 scoped paper/ink palette, box reset, focus-visible 및 masthead geometry를 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 root token·paper texture·focus·skip-link·masthead 기초에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:7546ac248334:END -->

### 2. `80b86ed4a1ff` — style(editorial): wordmark와 navigation 계층 구성

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Editorial 시각 계약의 wordmark·desktop navigation·design switcher·footer CTA

#### Commit-specific investigation

- `80b86ed4a1ff^`와 `80b86ed4a1ff`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 wordmark·desktop navigation·design switcher·footer CTA에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 wordmark·desktop navigation·design switcher·footer CTA` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:80b86ed4a1ff:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 wordmark·desktop navigation·design switcher·footer CTA에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 shell의 상단·하단 hierarchy와 route navigation 자리를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 wordmark·desktop navigation·design switcher·footer CTA에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:80b86ed4a1ff:END -->

### 3. `4d646fb2924a` — style(editorial): footer와 hero 활자 체계 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 footer와 home hero typography

#### Commit-specific investigation

- `4d646fb2924a^`와 `4d646fb2924a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 footer와 home hero typography에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 footer와 home hero typography` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4d646fb2924a:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 footer와 home hero typography에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 display/body scale 및 hero 상단 구조를 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 footer와 home hero typography에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4d646fb2924a:END -->

### 4. `6434531645b7` — style(editorial): hero spread 레이아웃 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 12-column hero spread와 공통 section rhythm

#### Commit-specific investigation

- `6434531645b7^`와 `6434531645b7`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 12-column hero spread와 공통 section rhythm에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 12-column hero spread와 공통 section rhythm` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:6434531645b7:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 12-column hero spread와 공통 section rhythm에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 hero 하단·metric/media 배치와 spread 간격을 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 12-column hero spread와 공통 section rhythm에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:6434531645b7:END -->

### 5. `a97066f07dfd` — style(editorial): lead story와 매체 표현 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 lead story와 framed media

#### Commit-specific investigation

- `a97066f07dfd^`와 `a97066f07dfd`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 lead story와 framed media에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 lead story와 framed media` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:a97066f07dfd:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 lead story와 framed media에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 대표 프로젝트 narrative와 image frame의 crop/placeholder 경계를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 lead story와 framed media에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:a97066f07dfd:END -->

### 6. `a9674cf2fa94` — style(editorial): 이미지 프레임과 feature 열 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 image placeholder·project index row·split feature columns

#### Commit-specific investigation

- `a9674cf2fa94^`와 `a9674cf2fa94`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 image placeholder·project index row·split feature columns에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 image placeholder·project index row·split feature columns` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:a9674cf2fa94:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 image placeholder·project index row·split feature columns에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 반복 가능한 media/index/feature 지면 단위를 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 image placeholder·project index row·split feature columns에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:a9674cf2fa94:END -->

### 7. `4708ca281c16` — style(editorial): 원칙 목록과 contact strip 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 principle cards·sidebar feature·tag·contact strip

#### Commit-specific investigation

- `4708ca281c16^`와 `4708ca281c16`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 principle cards·sidebar feature·tag·contact strip에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 principle cards·sidebar feature·tag·contact strip` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4708ca281c16:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 principle cards·sidebar feature·tag·contact strip에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 home/about/contact가 공유할 카드와 CTA grammar를 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 principle cards·sidebar feature·tag·contact strip에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4708ca281c16:END -->

### 8. `ebe23d211852` — style(editorial): contact와 archive 지면 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 contact strip 완성과 page/archive layout

#### Commit-specific investigation

- `ebe23d211852^`와 `ebe23d211852`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 contact strip 완성과 page/archive layout에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 contact strip 완성과 page/archive layout` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:ebe23d211852:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 contact strip 완성과 page/archive layout에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 공통 page hero와 archive index를 위한 지면 primitive를 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 contact strip 완성과 page/archive layout에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:ebe23d211852:END -->

### 9. `931226268687` — style(editorial): archive group과 case link 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 archive group과 case-study opening

#### Commit-specific investigation

- `931226268687^`와 `931226268687`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 archive group과 case-study opening에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 archive group과 case-study opening` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:931226268687:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 archive group과 case-study opening에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 category group, case link, 상세 도입부의 구조를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 archive group과 case-study opening에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:931226268687:END -->

### 10. `2cc074f728cb` — style(editorial): case link와 dark section 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 bordered links·cover·three-column narrative·dark architecture

#### Commit-specific investigation

- `2cc074f728cb^`와 `2cc074f728cb`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 bordered links·cover·three-column narrative·dark architecture에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 bordered links·cover·three-column narrative·dark architecture` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:2cc074f728cb:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 bordered links·cover·three-column narrative·dark architecture에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 상세 route의 링크·cover·problem/solution·architecture 대비를 확장한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 bordered links·cover·three-column narrative·dark architecture에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:2cc074f728cb:END -->

### 11. `34a9c958801c` — style(editorial): dark section과 decision 열 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 architecture evidence·image pair·decision columns

#### Commit-specific investigation

- `34a9c958801c^`와 `34a9c958801c`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 architecture evidence·image pair·decision columns에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 architecture evidence·image pair·decision columns` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:34a9c958801c:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 architecture evidence·image pair·decision columns에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 dark evidence section과 의사결정/상충 열을 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 architecture evidence·image pair·decision columns에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:34a9c958801c:END -->

### 12. `13f49ab0c1f7` — style(editorial): 결과 spread와 profile facts 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 result·exit navigation·missing project·profile hero

#### Commit-specific investigation

- `13f49ab0c1f7^`와 `13f49ab0c1f7`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 result·exit navigation·missing project·profile hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 result·exit navigation·missing project·profile hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:13f49ab0c1f7:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 result·exit navigation·missing project·profile hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 상세 종료/복구 상태와 About identity 지면을 함께 연다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 result·exit navigation·missing project·profile hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:13f49ab0c1f7:END -->

### 13. `124f6a6fec62` — style(editorial): profile summary와 skill group 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 portrait·principles grid·skills spread

#### Commit-specific investigation

- `124f6a6fec62^`와 `124f6a6fec62`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 portrait·principles grid·skills spread에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 portrait·principles grid·skills spread` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:124f6a6fec62:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 portrait·principles grid·skills spread에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 About의 portrait, 원칙, 기술 그룹 hierarchy를 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 portrait·principles grid·skills spread에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:124f6a6fec62:END -->

### 14. `c28bb0a5eb01` — style(editorial): 기술 그룹과 curation 본문 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 technology/experience rows와 curation spread

#### Commit-specific investigation

- `c28bb0a5eb01^`와 `c28bb0a5eb01`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 technology/experience rows와 curation spread에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 technology/experience rows와 curation spread` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:c28bb0a5eb01:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 technology/experience rows와 curation spread에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 반복 row와 비대칭 큐레이션 본문을 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 technology/experience rows와 curation spread에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:c28bb0a5eb01:END -->

### 15. `4ce0333849cc` — style(editorial): curation panel과 프로젝트 목록 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 criteria/category/omission panels와 project links

#### Commit-specific investigation

- `4ce0333849cc^`와 `4ce0333849cc`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 criteria/category/omission panels와 project links에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 criteria/category/omission panels와 project links` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4ce0333849cc:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 criteria/category/omission panels와 project links에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 큐레이션 archive의 번호·grid·card·link 구조를 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 criteria/category/omission panels와 project links에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4ce0333849cc:END -->

### 16. `586626a79cb1` — style(editorial): curation link와 resume 도입부 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 touch target·next-review panel·resume header/body

#### Commit-specific investigation

- `586626a79cb1^`와 `586626a79cb1`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 touch target·next-review panel·resume header/body에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 touch target·next-review panel·resume header/body` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:586626a79cb1:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 touch target·next-review panel·resume header/body에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 큐레이션 종료와 Resume 2-column 시작을 연결한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 touch target·next-review panel·resume header/body에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:586626a79cb1:END -->

### 17. `21d63d1975b3` — style(editorial): resume identity와 프로젝트 행 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 resume definition list·numbered sections·project/training rows

#### Commit-specific investigation

- `21d63d1975b3^`와 `21d63d1975b3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 resume definition list·numbered sections·project/training rows에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 resume definition list·numbered sections·project/training rows` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:21d63d1975b3:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 resume definition list·numbered sections·project/training rows에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 이력 정보 hierarchy와 반복 row를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 resume definition list·numbered sections·project/training rows에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:21d63d1975b3:END -->

### 18. `543f4b1062e3` — style(editorial): resume 사례와 contact 본문 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 resume case link·contact hero/availability/channels/notes

#### Commit-specific investigation

- `543f4b1062e3^`와 `543f4b1062e3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 resume case link·contact hero/availability/channels/notes에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 resume case link·contact hero/availability/channels/notes` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:543f4b1062e3:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 resume case link·contact hero/availability/channels/notes에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 Resume에서 Contact로 이어지는 route-specific 지면을 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 resume case link·contact hero/availability/channels/notes에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:543f4b1062e3:END -->

### 19. `e988e97415af` — style(editorial): contact note와 milestone link 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 contact notes와 journey milestone date/story

#### Commit-specific investigation

- `e988e97415af^`와 `e988e97415af`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 contact notes와 journey milestone date/story에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 contact notes와 journey milestone date/story` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:e988e97415af:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 contact notes와 journey milestone date/story에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 지원 note list와 여정 milestone spread를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 contact notes와 journey milestone date/story에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:e988e97415af:END -->

### 20. `1da39994d9e3` — style(editorial): milestone과 현재 방향 지면 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 secondary timeline과 current-position panel

#### Commit-specific investigation

- `1da39994d9e3^`와 `1da39994d9e3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 secondary timeline과 current-position panel에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 secondary timeline과 current-position panel` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1da39994d9e3:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 secondary timeline과 current-position panel에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 여정의 상세 milestone과 broader archive/current state를 시각적으로 분리한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 secondary timeline과 current-position panel에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1da39994d9e3:END -->

### 21. `0c3ba4ca1d48` — style(editorial): 현재 방향과 interview track 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 current position typography와 sticky horizontal chapter nav

#### Commit-specific investigation

- `0c3ba4ca1d48^`와 `0c3ba4ca1d48`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 current position typography와 sticky horizontal chapter nav에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 current position typography와 sticky horizontal chapter nav` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:0c3ba4ca1d48:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 current position typography와 sticky horizontal chapter nav에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 Interview Map의 in-page 탐색과 현재 방향 표현을 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 current position typography와 sticky horizontal chapter nav에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:0c3ba4ca1d48:END -->

### 22. `af5688dd1c3a` — style(editorial): interview 답변과 근거 표현 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 question/evidence paired ledger

#### Commit-specific investigation

- `af5688dd1c3a^`와 `af5688dd1c3a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 question/evidence paired ledger에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 question/evidence paired ledger` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:af5688dd1c3a:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 question/evidence paired ledger에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 질문·프로젝트 답변·depth를 두 열 ledger로 배치한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 question/evidence paired ledger에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:af5688dd1c3a:END -->

### 23. `0c7b77c2528a` — style(editorial): 공백 목록과 중형 화면 경계 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 unresolved gaps spread와 1180px adaptation

#### Commit-specific investigation

- `0c7b77c2528a^`와 `0c7b77c2528a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 unresolved gaps spread와 1180px adaptation에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 unresolved gaps spread와 1180px adaptation` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:0c7b77c2528a:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 unresolved gaps spread와 1180px adaptation에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 근거 공백을 dark spread로 만들고 첫 중형 폭 조정을 시작한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 unresolved gaps spread와 1180px adaptation에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:0c7b77c2528a:END -->

### 24. `a854cb45cc22` — style(editorial): tablet masthead와 hero 재배치

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 tablet native disclosure와 12→8-column hero

#### Commit-specific investigation

- `a854cb45cc22^`와 `a854cb45cc22`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 tablet native disclosure와 12→8-column hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 tablet native disclosure와 12→8-column hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:a854cb45cc22:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 tablet native disclosure와 12→8-column hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 desktop nav를 `<details>` menu로 바꾸고 hero grid를 축소한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 tablet native disclosure와 12→8-column hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:a854cb45cc22:END -->

### 25. `3f82e8a7c308` — style(editorial): tablet route 지면 재배치

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Editorial 시각 계약의 route spread의 tablet reading order

#### Commit-specific investigation

- `3f82e8a7c308^`와 `3f82e8a7c308`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 route spread의 tablet reading order에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 route spread의 tablet reading order` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:3f82e8a7c308:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 route spread의 tablet reading order에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 상세·About·Resume·Journey·Interview 주요 grid를 단일 열 중심으로 재배치한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 route spread의 tablet reading order에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:3f82e8a7c308:END -->

### 26. `10a442435e1a` — style(editorial): tablet 세부 간격 정리

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 tablet journey intro readable measure

#### Commit-specific investigation

- `10a442435e1a^`와 `10a442435e1a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 tablet journey intro readable measure에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 tablet journey intro readable measure` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:10a442435e1a:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 tablet journey intro readable measure에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 timeline introductory column 폭을 제한한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 tablet journey intro readable measure에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:10a442435e1a:END -->

### 27. `afaf24796399` — style(editorial): mobile navigation과 hero 구성

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Editorial 시각 계약의 mobile masthead metadata·stacked grids·linear hero

#### Commit-specific investigation

- `afaf24796399^`와 `afaf24796399`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 mobile masthead metadata·stacked grids·linear hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 mobile masthead metadata·stacked grids·linear hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:afaf24796399:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 mobile masthead metadata·stacked grids·linear hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 첫 mobile breakpoint에서 navigation과 주요 route grid를 선형화한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 mobile masthead metadata·stacked grids·linear hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:afaf24796399:END -->

### 28. `499c0e660caf` — style(editorial): mobile 본문과 표 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Editorial 시각 계약의 mobile case/profile/resume/milestone/curation/interview reflow

#### Commit-specific investigation

- `499c0e660caf^`와 `499c0e660caf`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 mobile case/profile/resume/milestone/curation/interview reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 mobile case/profile/resume/milestone/curation/interview reflow` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:499c0e660caf:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 mobile case/profile/resume/milestone/curation/interview reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 나머지 장문 본문·facts·표 형태를 좁은 화면 reading order로 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 mobile case/profile/resume/milestone/curation/interview reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:499c0e660caf:END -->

### 29. `f7a81e0fe1d3` — style(editorial): mobile footer와 동작 감소 구성

- **Importance:** B
- **Tags:** RENDERER, A11Y
- **Thread role:** Editorial 시각 계약의 small-screen footer spacing와 reduced-motion

#### Commit-specific investigation

- `f7a81e0fe1d3^`와 `f7a81e0fe1d3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **parent diff에서 small-screen footer spacing와 reduced-motion에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 시각 계약의 small-screen footer spacing와 reduced-motion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f7a81e0fe1d3:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 small-screen footer spacing와 reduced-motion에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/editorial/editorial.module.css`의 해당 SHA diff가 최소 화면 간격을 마감하고 animation/transition을 줄이는 media rule을 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Editorial stylesheet에 남는다.
- **inspection:** `src/designs/editorial/editorial.module.css`의 parent diff에서 small-screen footer spacing와 reduced-motion에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/editorial/editorial.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:f7a81e0fe1d3:END -->

### 30. `1c55d7422273` — feat(editorial): route 계약과 navigation helper 추가

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Editorial route boundary

#### Commit-specific investigation

- `1c55d7422273^`와 `1c55d7422273`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`EditorialRouteName`, 공통 props, `editorialHref`와 internal/external route 판정**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial route boundary` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `794615a037d3` shell과 최종 `46e23d922c2e` dispatcher가 이 계약을 소비한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1c55d7422273:BEGIN -->
- **직전 상태:** 직전 상태에는 **`EditorialRouteName`, 공통 props, `editorialHref`와 internal/external route 판정**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 여덟 route 이름을 닫힌 union으로 두고 content/project/currentPath/debug를 전달하는 모듈 계약을 만든다. 링크 helper는 선택한 디자인과 debug query를 내부 경로에 보존한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `EditorialRouteName`, 공통 props, `editorialHref`와 internal/external route 판정를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** `794615a037d3` shell과 최종 `46e23d922c2e` dispatcher가 이 계약을 소비한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1c55d7422273:END -->

### 31. `e078d79d24c8` — feat(editorial): debug note와 이미지 프레임 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** 재사용 presentation primitives

#### Commit-specific investigation

- `e078d79d24c8^`와 `e078d79d24c8`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`DebugNote`, semantic image/placeholder frame**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `재사용 presentation primitives` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:e078d79d24c8:BEGIN -->
- **직전 상태:** 직전 상태에는 **`DebugNote`, semantic image/placeholder frame**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** debug mode일 때만 source hint를 노출하고 project image 유무에 따라 semantic media 또는 placeholder를 반환하는 두 primitive를 추가한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `DebugNote`, semantic image/placeholder frame를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:e078d79d24c8:END -->

### 32. `1b353fe5ba7b` — feat(editorial): 콘텐츠 링크와 방향 표식 추가

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread role:** content-aware link primitive

#### Commit-specific investigation

- `1b353fe5ba7b^`와 `1b353fe5ba7b`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`EditorialContentLink`, internal `Link`와 external/mailto anchor 분기**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `content-aware link primitive` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1b353fe5ba7b:BEGIN -->
- **직전 상태:** 직전 상태에는 **`EditorialContentLink`, internal `Link`와 external/mailto anchor 분기**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 내부 경로는 template/debug state를 보존하고 외부 목적지는 일반 anchor 속성을 사용한다. route composition이 링크 종류를 매번 재판정하지 않게 된다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `EditorialContentLink`, internal `Link`와 external/mailto anchor 분기를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1b353fe5ba7b:END -->

### 33. `794615a037d3` — feat(editorial): masthead와 footer shell 추가

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** shared `EditorialShell`

#### Commit-specific investigation

- `794615a037d3^`와 `794615a037d3`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **desktop/mobile navigation, design switcher, main landmark, footerLinks**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `shared `EditorialShell`` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `46e23d922c2e`가 모든 route body를 이 shell 내부에 배치한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:794615a037d3:BEGIN -->
- **직전 상태:** 직전 상태에는 **desktop/mobile navigation, design switcher, main landmark, footerLinks**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** canonical navigation과 footer link를 content에서 읽고 현재 경로·debug·template 상태를 보존하는 full-site shell을 만든다. `<main>`과 skip target도 이 shell이 소유한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 desktop/mobile navigation, design switcher, main landmark, footerLinks를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** `46e23d922c2e`가 모든 route body를 이 shell 내부에 배치한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:794615a037d3:END -->

### 34. `b7fd9118025e` — feat(editorial): 섹션 표식과 프로젝트 인덱스 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** section/project list primitives

#### Commit-specific investigation

- `b7fd9118025e^`와 `b7fd9118025e`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`SectionKicker`, project index row와 renderer-preserving link**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `section/project list primitives` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b7fd9118025e:BEGIN -->
- **직전 상태:** 직전 상태에는 **`SectionKicker`, project index row와 renderer-preserving link**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 번호 표식과 프로젝트 metadata/action row를 반복 가능한 component로 분리해 home/archive가 같은 DOM 계약을 사용한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `SectionKicker`, project index row와 renderer-preserving link를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b7fd9118025e:END -->

### 35. `5c82371743ba` — feat(editorial): 홈 hero spread 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home route 시작

#### Commit-specific investigation

- `5c82371743ba^`와 `5c82371743ba`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`HomeRoute`, presentation-configured section order, hero, featured fallback, current year**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home route 시작` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 시점에는 모든 configured section 구현이 아직 존재하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: 후속 96ba/4c827/9831/f01이 section node를 채운다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5c82371743ba:BEGIN -->
- **직전 상태:** 직전 상태에는 **`HomeRoute`, presentation-configured section order, hero, featured fallback, current year**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Home route가 configured section ID를 순회하는 dispatcher로 시작된다. featured가 비면 전체 projects를 사용하고 현재 연도를 runtime에서 계산한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `HomeRoute`, presentation-configured section order, hero, featured fallback, current year를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 시점에는 모든 configured section 구현이 아직 존재하지 않는다.
- **다음 관계:** 후속 96ba/4c827/9831/f01이 section node를 채운다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5c82371743ba:END -->

### 36. `96ba59901181` — feat(editorial): 홈 lead story 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home lead project

#### Commit-specific investigation

- `96ba59901181^`와 `96ba59901181`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **첫 selected project를 narrative feature로 렌더링하는 lead section**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home lead project` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:96ba59901181:BEGIN -->
- **직전 상태:** 직전 상태에는 **첫 selected project를 narrative feature로 렌더링하는 lead section**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 선택 목록의 첫 프로젝트를 큰 story와 media로 사용하고 detail link에 template/debug state를 보존한다. project가 없으면 해당 section을 생략한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 첫 selected project를 narrative feature로 렌더링하는 lead section를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:96ba59901181:END -->

### 37. `4c8270522400` — feat(editorial): 홈 대표 프로젝트 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home remaining featured list

#### Commit-specific investigation

- `4c8270522400^`와 `4c8270522400`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **lead를 제외한 selected projects와 shared project index row**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home remaining featured list` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4c8270522400:BEGIN -->
- **직전 상태:** 직전 상태에는 **lead를 제외한 selected projects와 shared project index row**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 첫 항목을 lead로 사용한 뒤 `slice(1)`의 나머지를 index row로 렌더링해 중복 노출을 피한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 lead를 제외한 selected projects와 shared project index row를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4c8270522400:END -->

### 38. `983131c5a266` — feat(editorial): 홈 원칙과 기술 sidebar 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home principles/system section

#### Commit-specific investigation

- `983131c5a266^`와 `983131c5a266`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **profile principles, current journey, tech stack sidebar**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home principles/system section` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: 후속 cross-design view-model 제한이 이 파생 책임을 route boundary로 옮긴다.

#### Learning record

<!-- LEARNER-ANSWER:commit:983131c5a266:BEGIN -->
- **직전 상태:** 직전 상태에는 **profile principles, current journey, tech stack sidebar**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 원칙을 주요 narrative로 두고 최근 여정·stack을 supporting column에 배치한다. source content를 직접 읽는 초기 renderer 책임이 남아 있다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 profile principles, current journey, tech stack sidebar를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** 후속 cross-design view-model 제한이 이 파생 책임을 route boundary로 옮긴다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:983131c5a266:END -->

### 39. `f01b60fc368e` — feat(editorial): 홈 contact strip 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home contact CTA

#### Commit-specific investigation

- `f01b60fc368e^`와 `f01b60fc368e`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **contact availability와 preferred links를 사용하는 마지막 configured section**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home contact CTA` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f01b60fc368e:BEGIN -->
- **직전 상태:** 직전 상태에는 **contact availability와 preferred links를 사용하는 마지막 configured section**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Home section map을 contact CTA까지 완성하며 preferred link가 없을 때 section action이 비어 있음을 허용한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 contact availability와 preferred links를 사용하는 마지막 configured section를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:f01b60fc368e:END -->

### 40. `4e69ba2ee361` — feat(editorial): 프로젝트 archive route 추가

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Projects archive

#### Commit-specific investigation

- `4e69ba2ee361^`와 `4e69ba2ee361`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ProjectsRoute`, project grouping, metrics, featured/archive partitions, explicit empty copy**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Projects archive` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: 후속 project view model migration이 grouping/metric 소유권을 content boundary로 이동한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4e69ba2ee361:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectsRoute`, project grouping, metrics, featured/archive partitions, explicit empty copy**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 이 시점 renderer가 프로젝트 grouping과 metric 계산을 직접 수행하고 category별 archive를 만든다. 목록이 비면 shared empty-state copy를 표시한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `ProjectsRoute`, project grouping, metrics, featured/archive partitions, explicit empty copy를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** 후속 project view model migration이 grouping/metric 소유권을 content boundary로 이동한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4e69ba2ee361:END -->

### 41. `c722cdd08ef8` — feat(editorial): 프로젝트 상세 서사와 구조 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail first complete path

#### Commit-specific investigation

- `c722cdd08ef8^`와 `c722cdd08ef8`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ProjectDetailRoute`, missing-project guard, facts, links, cover, problem/solution/architecture**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail first complete path` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `f38556a17e8b`가 증거·결과·exit를 완성한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:c722cdd08ef8:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectDetailRoute`, missing-project guard, facts, links, cover, problem/solution/architecture**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** project가 없으면 dereference 전에 recoverable missing spread와 archive link를 반환한다. 유효한 경우 canonical detail links와 narrative/architecture를 구성한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `ProjectDetailRoute`, missing-project guard, facts, links, cover, problem/solution/architecture를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** `f38556a17e8b`가 증거·결과·exit를 완성한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:c722cdd08ef8:END -->

### 42. `f38556a17e8b` — feat(editorial): 프로젝트 증거와 결과 spread 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail completion

#### Commit-specific investigation

- `f38556a17e8b^`와 `f38556a17e8b`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **highlights, optional supporting images, decisions, tradeoffs, results, archive exit**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f38556a17e8b:BEGIN -->
- **직전 상태:** 직전 상태에는 **highlights, optional supporting images, decisions, tradeoffs, results, archive exit**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** optional gallery는 실제 이미지가 있을 때만 보이고 evidence lists와 결과/종료 동선을 추가한다. 상세 route가 하나의 완결된 case study가 된다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 highlights, optional supporting images, decisions, tradeoffs, results, archive exit를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:f38556a17e8b:END -->

### 43. `cc1b2233287f` — feat(editorial): About 정체성과 원칙 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** About identity

#### Commit-specific investigation

- `cc1b2233287f^`와 `cc1b2233287f`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`AboutRoute`, optional photo, profile facts, principles**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `About identity` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:cc1b2233287f:BEGIN -->
- **직전 상태:** 직전 상태에는 **`AboutRoute`, optional photo, profile facts, principles**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** profile identity와 원칙을 canonical content에서 읽고 photo가 있을 때만 media frame을 배치한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `AboutRoute`, optional photo, profile facts, principles를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:cc1b2233287f:END -->

### 44. `5f0193979568` — feat(editorial): About 기술과 경력 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** About skills/experience

#### Commit-specific investigation

- `5f0193979568^`와 `5f0193979568`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **focus areas, skill groups, chronological experience**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `About skills/experience` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5f0193979568:BEGIN -->
- **직전 상태:** 직전 상태에는 **focus areas, skill groups, chronological experience**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** About route에 기술과 경력 record를 추가하며 각 배열의 source order를 presentation order로 사용한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 focus areas, skill groups, chronological experience를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5f0193979568:END -->

### 45. `5c95665ca9d2` — feat(editorial): About 큐레이션 기준 추가

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread role:** feature-gated curation start

#### Commit-specific investigation

- `5c95665ca9d2^`와 `5c95665ca9d2`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`isSitePageEnabled("curation", content)`와 criteria**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `feature-gated curation start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5c95665ca9d2:BEGIN -->
- **직전 상태:** 직전 상태에는 **`isSitePageEnabled("curation", content)`와 criteria**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** curation capability가 켜졌을 때만 criteria section을 추가한다. disabled 상태는 빈 panel이 아니라 section 자체 부재다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `isSitePageEnabled("curation", content)`와 criteria를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5c95665ca9d2:END -->

### 46. `4a7c3a3c9cde` — feat(editorial): About 큐레이션 범주 추가

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread role:** curation category project join

#### Commit-specific investigation

- `4a7c3a3c9cde^`와 `4a7c3a3c9cde`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **category projectIds를 canonical projects에 `find`/filter하여 link 생성**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `curation category project join` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4a7c3a3c9cde:BEGIN -->
- **직전 상태:** 직전 상태에는 **category projectIds를 canonical projects에 `find`/filter하여 link 생성**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 유효한 project ID만 category card에 남기며 누락 참조는 링크를 만들지 않는다. 이 join은 이 시점 renderer 내부 책임이다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 category projectIds를 canonical projects에 `find`/filter하여 link 생성를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4a7c3a3c9cde:END -->

### 47. `c0d0004e9355` — feat(editorial): About 큐레이션 공백과 재검토 추가

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread role:** curation completion

#### Commit-specific investigation

- `c0d0004e9355^`와 `c0d0004e9355`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **omissions list와 nextReview panel**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `curation completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:c0d0004e9355:BEGIN -->
- **직전 상태:** 직전 상태에는 **omissions list와 nextReview panel**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 큐레이션에서 의도적으로 제외한 항목과 다음 재검토 조건을 별도 section으로 노출해 absence를 암묵적 누락으로 만들지 않는다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 omissions list와 nextReview panel를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:c0d0004e9355:END -->

### 48. `119d19ab41b1` — feat(editorial): Resume 정체성과 프로젝트 경력 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Resume start

#### Commit-specific investigation

- `119d19ab41b1^`와 `119d19ab41b1`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ResumeRoute`, optional download, identity facts, summary, resolved resume projects, empty fallback**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Resume start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:119d19ab41b1:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ResumeRoute`, optional download, identity facts, summary, resolved resume projects, empty fallback**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** download URL은 있을 때만 anchor를 만들고 resume project ID를 canonical project로 해석한 결과를 표시한다. 해석 결과가 비면 projects archive empty copy를 사용한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `ResumeRoute`, optional download, identity facts, summary, resolved resume projects, empty fallback를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:119d19ab41b1:END -->

### 49. `4df2710fa7f9` — feat(editorial): Resume 경력과 교육 기록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Resume completion

#### Commit-specific investigation

- `4df2710fa7f9^`와 `4df2710fa7f9`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **experience, training, education, notes와 `EvidenceList` empty label**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Resume completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4df2710fa7f9:BEGIN -->
- **직전 상태:** 직전 상태에는 **experience, training, education, notes와 `EvidenceList` empty label**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Resume에 네 개의 후속 section을 source order로 추가하고 notes는 공용 empty-state primitive로 비어 있음을 명시한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 experience, training, education, notes와 `EvidenceList` empty label를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4df2710fa7f9:END -->

### 50. `61d6952850cd` — feat(editorial): Contact desk route 추가

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Contact route

#### Commit-specific investigation

- `61d6952850cd^`와 `61d6952850cd`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ContactRoute`, preferred-contact ordering, availability, notes, explicit empty links**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Contact route` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:61d6952850cd:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ContactRoute`, preferred-contact ordering, availability, notes, explicit empty links**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** `getPreferredContactLinks` 결과를 action card로 표시하고 비면 `emptyStates.contactLinks`를 보여 준다. 연락 가능 상태와 notes는 별도 column이 소유한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `ContactRoute`, preferred-contact ordering, availability, notes, explicit empty links를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:61d6952850cd:END -->

### 51. `08fa527b9b65` — feat(editorial): Journey milestone spread 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Journey narrative start

#### Commit-specific investigation

- `08fa527b9b65^`와 `08fa527b9b65`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`JourneyRoute`, milestones, anchorProjectIds resolution, empty journey**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Journey narrative start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:08fa527b9b65:BEGIN -->
- **직전 상태:** 직전 상태에는 **`JourneyRoute`, milestones, anchorProjectIds resolution, empty journey**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** milestone마다 anchor IDs를 canonical projects로 해석하고 유효한 것만 nav link로 남긴다. milestone 배열이 비면 명시적 journey empty row를 렌더링한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `JourneyRoute`, milestones, anchorProjectIds resolution, empty journey를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:08fa527b9b65:END -->

### 52. `96b66af4d5a7` — feat(editorial): Journey timeline과 현재 방향 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Journey completion

#### Commit-specific investigation

- `96b66af4d5a7^`와 `96b66af4d5a7`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **dated timeline, optional linked project, currentPosition**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Journey completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:96b66af4d5a7:BEGIN -->
- **직전 상태:** 직전 상태에는 **dated timeline, optional linked project, currentPosition**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** broader journey archive를 source order로 렌더링하고 projectId가 실제 project로 해석될 때만 link를 만든다. 마지막에 current-position title/body를 별도 high-contrast section으로 고정한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 dated timeline, optional linked project, currentPosition를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:96b66af4d5a7:END -->

### 53. `5e2f37861d3d` — feat(editorial): Interview Map 소개와 chapter 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Interview start

#### Commit-specific investigation

- `5e2f37861d3d^`와 `5e2f37861d3d`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`InterviewMapRoute`, external reference repository, track fragment index, `projectById` Map**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Interview start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5e2f37861d3d:BEGIN -->
- **직전 상태:** 직전 상태에는 **`InterviewMapRoute`, external reference repository, track fragment index, `projectById` Map**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 소개와 외부 reference link를 만들고 configured track으로 fragment navigation을 생성한다. project lookup용 Map을 준비하지만 answer evidence body는 다음 commit 전에는 없다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 `InterviewMapRoute`, external reference repository, track fragment index, `projectById` Map를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5e2f37861d3d:END -->

### 54. `94deba32f56a` — feat(editorial): Interview 답변 근거와 공백 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Interview completion

#### Commit-specific investigation

- `94deba32f56a^`와 `94deba32f56a`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **tracks/questions/references/answers/depth, missing project/empty answers, gaps**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Interview completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:94deba32f56a:BEGIN -->
- **직전 상태:** 직전 상태에는 **tracks/questions/references/answers/depth, missing project/empty answers, gaps**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** answer project가 Map에 없으면 링크 대신 `noMappedEvidence`를 보이되 depth는 유지한다. track/item/answer가 비어 있는 각 계층과 gaps list를 명시적으로 표시한다.
- **inspection:** `src/designs/editorial/editorial-route.tsx`의 tracks/questions/references/answers/depth, missing project/empty answers, gaps를 확인했습니다.
- **책임/경계:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:94deba32f56a:END -->

### 55. `46e23d922c2e` — feat(editorial): route dispatcher 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** Editorial 여덟 route와 shared shell을 하나의 public entry로 폐쇄

#### Commit-specific investigation

- `46e23d922c2e^`와 `46e23d922c2e`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`renderRoute`의 exhaustive switch와 exported `EditorialRoute`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Editorial 여덟 route와 shared shell을 하나의 public entry로 폐쇄` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `TypeScript union의 exhaustiveness에 의존하며 runtime unknown string에 대한 별도 default UI는 없다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 1의 `c6acfe562694`가 이 entry를 registry에 활성화한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:46e23d922c2e:BEGIN -->
- **직전 상태:** 직전 상태에는 **`renderRoute`의 exhaustive switch와 exported `EditorialRoute`**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **구현 결정:** 각 route function이 따로 존재하던 상태에서 `renderRoute`가 discriminated route를 여덟 view로 매핑하고, public `EditorialRoute`는 결과를 항상 `EditorialShell` 안에 넣는다. 외부 registry는 내부 view 이름을 알 필요가 없고 shell/navigation/footer가 모든 route에 일관되게 적용된다.
- **파일·symbol:** `src/designs/editorial/editorial-route.tsx`에서 `renderRoute`의 exhaustive switch와 exported `EditorialRoute`를 확인했습니다.
- **소유권:** `src/designs/editorial/editorial-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **보장/비보장:** 이 SHA는 `Editorial 여덟 route와 shared shell을 하나의 public entry로 폐쇄` 경계를 고정합니다. TypeScript union의 exhaustiveness에 의존하며 runtime unknown string에 대한 별도 default UI는 없다.
- **역사적 연결:** Thread 1의 `c6acfe562694`가 이 entry를 registry에 활성화한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```tsx
// 46e23d922c2e · src/designs/editorial/editorial-route.tsx
export function EditorialRoute(props: EditorialRouteProps) {
  return <EditorialShell {...props}>{renderRoute(props)}</EditorialShell>;
}
```
<!-- LEARNER-ANSWER:commit:46e23d922c2e:END -->

## 5. Invariant evolution

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:invariant:BEGIN -->
최종 invariant는 `EditorialRoute` 하나만 외부에 노출되고, route discriminator가 여덟 private view 중 하나를 선택한 뒤 항상 `EditorialShell`로 감싸며, absence/failure를 route별 명시적 UI로 표현한다는 것입니다.

- 도입·확장·폐쇄의 순서는 commit map에 고정했습니다.
- B-level construction은 route/style surface를 단계적으로 넓히고, A/S-level commit은 owner·dispatch·검증 invariant를 바꿉니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:invariant:END -->

## 6. Failure → Fix → Test and ownership relations

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:relations:BEGIN -->
CSS는 desktop 지면을 먼저 완성한 뒤 1180/tablet/mobile/reduced-motion 경계를 추가합니다. TSX는 contract/primitives/shell 뒤 Home→Projects→Detail→About→Resume→Contact→Journey→Interview 순서로 확장됩니다. project/detail/contact/journey/interview의 empty·missing reference branch가 후속 route completion에서 보강되고, `46e23d922c2e`가 최종 public API를 닫습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:relations:END -->

## 7. Final architecture or execution flow

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:flow:BEGIN -->
registry entry → `EditorialRoute` → `renderRoute` switch → private route view → shared Editorial primitives/link helper → `EditorialShell`의 masthead/main/footer 순서입니다.

각 단계에서 optional/empty/missing reference가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 commit의 non-guarantee에 남겼습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:flow:END -->

## 8. Runtime and verification evidence

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:runtime:BEGIN -->
- **실행한 repository test/build:** 없음.
- **정적 확인:** 지정 branch의 commit classification, commit bodies, exact commit diff와 historical file 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 container에서 직접 clone 시 DNS가 `github.com`을 해석하지 못해 historical worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next build 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** code/test implementation의 존재와 범위는 inspection으로 확인했고, runtime pass/fail은 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:runtime:END -->

## 9. Learning-completion checks

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·subject·importance·tags를 commit map과 commit section에서 동일하게 유지했습니다.
- [x] 각 SHA에 concrete file과 symbol/selector/route focus를 기록했습니다.
- [x] 이전 상태, owner, absence/fallback, guarantee/non-guarantee, 후속 관계를 채웠습니다.
- [x] S/A/B depth를 구분했습니다.
- [x] 실행하지 않은 test를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:checks:END -->
===== END FILE: 02-editorial-design-system-construction.md =====

===== BEGIN FILE: 03-brutalist-design-system-construction.md =====
# Thread: Brutalist design system construction

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `05-full-site-visual-systems`
>
> Phase 1 audit에서 확정한 authoritative scaffold입니다. Phase 2에서는 answer marker 내부만 채웁니다.

## 0. Scope and authority

- Commit SHA, subject, importance, tags는 branch의 `commit/commit-importance.md`와 exact commit metadata를 기준으로 고정했습니다.
- **Thread boundary:** Brutalist stylesheet와 `brutalist-route.tsx` construction만 포함합니다. activation은 Thread 1에, formatting-only media consolidation은 제외합니다.
- 다른 branch, final HEAD의 후대 구현, 실행하지 않은 command 결과를 사용하지 않습니다.

## 1. Thread goal

Brutalist의 고대비 visual grammar, responsive/print 계약, content adapter, 여덟 route와 single public entry가 구축되는 과정을 복원합니다.

### Frozen invariant target

최종 invariant는 Brutalist module이 route entry 하나만 공개하고, shell·navigation·link policy·content adapter·view가 같은 module의 private implementation으로 유지되며, mobile/reduced-motion/print와 missing/empty states가 시각 계약에 포함된다는 것입니다.

## 2. Commit map

| 순서 | Commit | Subject | Importance | Tags | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `162542118ba4` | style(brutalist): 화면 토큰과 brand mark 구성 | B | RENDERER | Brutalist 시각 계약의 scoped palette·focus·skip-link·header/brand 기초 |
| 2 | `a2539ef309d1` | style(brutalist): header 상태와 home hero 구성 | B | RENDERER | Brutalist 시각 계약의 desktop header status/switcher/navigation/debug와 home hero |
| 3 | `1faf77ef9916` | style(brutalist): hero stamp와 action row 구성 | B | RENDERER | Brutalist 시각 계약의 hero stamp·copy·oversized title·action row |
| 4 | `75913149fe24` | style(brutalist): 주요 action과 section 경계 구성 | B | RENDERER | Brutalist 시각 계약의 high-contrast actions·4-column metrics·signal strip·section boundaries |
| 5 | `f4e53be5ea42` | style(brutalist): section header와 프로젝트 지표 구성 | B | RENDERER | Brutalist 시각 계약의 numbered section-header grid와 project index row |
| 6 | `ebfe79d62e53` | style(brutalist): 프로젝트 지표와 card 번호 구성 | B | RENDERER | Brutalist 시각 계약의 project metadata/tag/action과 principle card |
| 7 | `aaf26e755213` | style(brutalist): 원칙 카드와 contact band 구성 | B | RENDERER | Brutalist 시각 계약의 principles·tech wall·compact timeline·large contact band |
| 8 | `16336e1dc469` | style(brutalist): contact 링크와 프로젝트 group 구성 | B | RENDERER | Brutalist 시각 계약의 contact actions·page hero·inline metrics·grouped archive |
| 9 | `2660465c0904` | style(brutalist): 교차 group과 상세 lead 구성 | B | RENDERER | Brutalist 시각 계약의 alternating groups와 case-study lead |
| 10 | `4621b0a3cb1f` | style(brutalist): 상세 fact와 소개 본문 구성 | B | RENDERER | Brutalist 시각 계약의 detail facts·media frame·placeholder·intro band |
| 11 | `1d5445cc6f4a` | style(brutalist): 상세 본문과 gallery grid 구성 | B | RENDERER | Brutalist 시각 계약의 labeled evidence sections·numbered lists·2-column gallery |
| 12 | `bb5008a8c7b3` | style(brutalist): 다음 프로젝트와 focus card 구성 | B | RENDERER, A11Y | Brutalist 시각 계약의 next-project·not-found recovery·About portrait/skills |
| 13 | `8de2180bcc58` | style(brutalist): focus card와 criteria grid 구성 | B | RENDERER, A11Y | Brutalist 시각 계약의 focus/skill cards와 dark curation criteria |
| 14 | `a34cd7cd88bf` | style(brutalist): criteria 본문과 재검토 영역 구성 | B | RENDERER | Brutalist 시각 계약의 curation category·omission·next-review |
| 15 | `5ca14417cf22` | style(brutalist): 재검토와 resume entry 구성 | B | RENDERER | Brutalist 시각 계약의 review panel 마감과 resume repeated sections |
| 16 | `fad7f0216645` | style(brutalist): resume 본문과 contact hero 구성 | B | RENDERER | Brutalist 시각 계약의 resume project rows/notes와 Contact blue hero |
| 17 | `8175392db042` | style(brutalist): contact 상태와 note 목록 구성 | B | RENDERER | Brutalist 시각 계약의 availability badge·channel grid·note list |
| 18 | `6fa3a9dc8665` | style(brutalist): note 목록과 anchor link 구성 | B | RENDERER | Brutalist 시각 계약의 notes/evidence gaps와 Journey milestone card |
| 19 | `242ba8e66e0b` | style(brutalist): archive timeline과 track navigation 구성 | B | ROUTING, RENDERER | Brutalist 시각 계약의 journey archive/current callout와 Interview track shell |
| 20 | `95e55eda6c51` | style(brutalist): track 목록과 question prompt 구성 | B | RENDERER | Brutalist 시각 계약의 track index·track/question hierarchy |
| 21 | `11f229d630e9` | style(brutalist): 답변 근거와 footer lead 구성 | B | RENDERER | Brutalist 시각 계약의 question reference·answer evidence·empty answer·footer lead |
| 22 | `b170c73a36d0` | style(brutalist): footer metadata와 blink 동작 구성 | B | RENDERER, SEO | Brutalist 시각 계약의 footer metadata·dashed empty state·crawl/blink animations |
| 23 | `f810c49022be` | style(brutalist): tablet grid 재배치 | B | RENDERER | Brutalist 시각 계약의 980px tablet grid reflow |
| 24 | `b57da6a41419` | style(brutalist): mobile header와 hero 구성 | B | RENDERER | Brutalist 시각 계약의 720px native details menu와 stacked header/hero |
| 25 | `8168bc76c3e3` | style(brutalist): mobile 프로젝트와 상세 화면 구성 | B | RENDERER | Brutalist 시각 계약의 mobile metrics/project rows/detail/gallery reflow |
| 26 | `5551f3fdbb94` | style(brutalist): mobile profile과 resume 구성 | B | RENDERER | Brutalist 시각 계약의 mobile profile/curation/resume/contact/current-position |
| 27 | `7c08aea7a2f7` | style(brutalist): mobile 여정과 interview 구성 | B | RENDERER | Brutalist 시각 계약의 mobile journey/interview/footer/missing page |
| 28 | `077ff3d49f30` | style(brutalist): 소형 화면과 인쇄 경계 구성 | B | RENDERER | Brutalist 시각 계약의 430px hardening·reduced motion·print |
| 29 | `3e6ec5262bdd` | feat(brutalist): 콘텐츠와 탐색 조회 도우미 추가 | B | CONTENT, ROUTING, RENDERER | content/navigation adapter layer |
| 30 | `08a2b0c0998f` | feat(brutalist): route 레이블과 기본 shell 구성 | B | ROUTING, RENDERER | shared shell start |
| 31 | `cf2fdb36f9fc` | feat(brutalist): 주 탐색과 모바일 메뉴 추가 | B | ROUTING, RENDERER | canonical navigation |
| 32 | `5b44afbc46ef` | feat(brutalist): footer와 홈 히어로 연결 | B | RENDERER | footer + Home start |
| 33 | `b477ba477127` | feat(brutalist): 홈 섹션 공용 프리미티브 추가 | B | RENDERER | Home/archive primitives |
| 34 | `b30b9b1c3505` | feat(brutalist): 대표 작업과 작업 원칙 구성 | B | RENDERER | Home middle + Projects hero |
| 35 | `85ea663aaf19` | feat(brutalist): 홈 여정과 프로젝트 archive 구성 | B | RENDERER | Home completion + Projects archive |
| 36 | `d6b9a99e11ae` | feat(brutalist): 프로젝트 상세 표시 프리미티브 추가 | B | RENDERER | Project detail primitives |
| 37 | `b8268f47e89a` | feat(brutalist): 프로젝트 상세 hero와 소개 구성 | B | RENDERER | Project detail valid/missing boundary |
| 38 | `05b838d52a8b` | feat(brutalist): 프로젝트 상세 본문과 gallery 구성 | B | RENDERER | Project detail evidence body |
| 39 | `80724a26820b` | feat(brutalist): 프로필과 기술 소개 구성 | B | RENDERER | About identity |
| 40 | `3399b55c3aee` | feat(brutalist): 큐레이션과 경력 소개 구성 | B | CONTENT, RENDERER | About experience + feature-gated curation |
| 41 | `70cf13ef1715` | feat(brutalist): 이력 hero와 경력 요약 구성 | B | RENDERER | Resume start |
| 42 | `1ea2a1345b76` | feat(brutalist): 프로젝트 결과와 의사결정 구성 | B | RENDERER | Project detail completion |
| 43 | `5fa378250d64` | feat(brutalist): 선택 프로젝트와 이력 세부 구성 | B | RENDERER | Resume completion |
| 44 | `b535539ae016` | feat(brutalist): 연락 수단과 안내 구성 | B | RENDERER | Contact route |
| 45 | `15a765ecb2aa` | feat(brutalist): 여정 milestone 구성 | B | RENDERER | Journey narrative start |
| 46 | `388446b1a982` | feat(brutalist): 여정 archive와 인터뷰 map 머리말 구성 | B | RENDERER | Journey completion + Interview start |
| 47 | `f3fc6200a45b` | feat(brutalist): 인터뷰 근거 archive 구성 | B | RENDERER | Interview evidence |
| 48 | `da8e59d56783` | feat(brutalist): 인터뷰 근거 공백 구성 | B | RENDERER | Interview gaps |
| 49 | `e6268c4b7c74` | refactor(brutalist): 내부 helper 공개 범위 정리 | B | RENDERER, REFACTOR | module API narrowing |
| 50 | `caa7df81d899` | feat(brutalist): 모든 route를 renderer에 통합 | A | ARCH, ROUTING, RENDERER | Brutalist public API와 여덟 route dispatch의 최종 경계 |

## 3. Historical baseline

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:baseline:BEGIN -->
- **직전 상태:** 공통 route delegation은 있었지만 Brutalist는 CSS/DOM/shell/helper가 전혀 없는 상태였습니다. 구현 초기는 renderer 내부 content lookup helper를 직접 소유했습니다.
- **경계 판단:** Brutalist stylesheet와 `brutalist-route.tsx` construction만 포함합니다. activation은 Thread 1에, formatting-only media consolidation은 제외합니다.
- **복원 기준:** 각 commit의 parent와 exact SHA tree만 사용하고 final HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:baseline:END -->

## 4. Commit-by-commit reconstruction

### 1. `162542118ba4` — style(brutalist): 화면 토큰과 brand mark 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 scoped palette·focus·skip-link·header/brand 기초

#### Commit-specific investigation

- `162542118ba4^`와 `162542118ba4`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 scoped palette·focus·skip-link·header/brand 기초에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 scoped palette·focus·skip-link·header/brand 기초` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:162542118ba4:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 scoped palette·focus·skip-link·header/brand 기초에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 paper/ink/blue/yellow token, box sizing, focus-visible, selection, skip link와 brand mark를 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 scoped palette·focus·skip-link·header/brand 기초에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:162542118ba4:END -->

### 2. `a2539ef309d1` — style(brutalist): header 상태와 home hero 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 desktop header status/switcher/navigation/debug와 home hero

#### Commit-specific investigation

- `a2539ef309d1^`와 `a2539ef309d1`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 desktop header status/switcher/navigation/debug와 home hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 desktop header status/switcher/navigation/debug와 home hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:a2539ef309d1:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 desktop header status/switcher/navigation/debug와 home hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 명시적 상태 cell과 큰 hero grid를 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 desktop header status/switcher/navigation/debug와 home hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:a2539ef309d1:END -->

### 3. `1faf77ef9916` — style(brutalist): hero stamp와 action row 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 hero stamp·copy·oversized title·action row

#### Commit-specific investigation

- `1faf77ef9916^`와 `1faf77ef9916`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 hero stamp·copy·oversized title·action row에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 hero stamp·copy·oversized title·action row` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1faf77ef9916:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 hero stamp·copy·oversized title·action row에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 hero의 시각적 stamp와 유연한 action row를 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 hero stamp·copy·oversized title·action row에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1faf77ef9916:END -->

### 4. `75913149fe24` — style(brutalist): 주요 action과 section 경계 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 high-contrast actions·4-column metrics·signal strip·section boundaries

#### Commit-specific investigation

- `75913149fe24^`와 `75913149fe24`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 high-contrast actions·4-column metrics·signal strip·section boundaries에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 high-contrast actions·4-column metrics·signal strip·section boundaries` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:75913149fe24:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 high-contrast actions·4-column metrics·signal strip·section boundaries에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 공용 버튼 grammar와 metric/signal/section 구획을 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 high-contrast actions·4-column metrics·signal strip·section boundaries에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:75913149fe24:END -->

### 5. `f4e53be5ea42` — style(brutalist): section header와 프로젝트 지표 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 numbered section-header grid와 project index row

#### Commit-specific investigation

- `f4e53be5ea42^`와 `f4e53be5ea42`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 numbered section-header grid와 project index row에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 numbered section-header grid와 project index row` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f4e53be5ea42:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 numbered section-header grid와 project index row에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 홈·archive에서 재사용할 번호 header와 project metadata row를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 numbered section-header grid와 project index row에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:f4e53be5ea42:END -->

### 6. `ebfe79d62e53` — style(brutalist): 프로젝트 지표와 card 번호 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 project metadata/tag/action과 principle card

#### Commit-specific investigation

- `ebfe79d62e53^`와 `ebfe79d62e53`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 project metadata/tag/action과 principle card에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 project metadata/tag/action과 principle card` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:ebfe79d62e53:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 project metadata/tag/action과 principle card에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 project index를 완성하고 첫 principle card system을 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 project metadata/tag/action과 principle card에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:ebfe79d62e53:END -->

### 7. `aaf26e755213` — style(brutalist): 원칙 카드와 contact band 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 principles·tech wall·compact timeline·large contact band

#### Commit-specific investigation

- `aaf26e755213^`와 `aaf26e755213`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 principles·tech wall·compact timeline·large contact band에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 principles·tech wall·compact timeline·large contact band` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:aaf26e755213:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 principles·tech wall·compact timeline·large contact band에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 Home 후반의 principle/stack/journey/contact 지면을 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 principles·tech wall·compact timeline·large contact band에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:aaf26e755213:END -->

### 8. `16336e1dc469` — style(brutalist): contact 링크와 프로젝트 group 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 contact actions·page hero·inline metrics·grouped archive

#### Commit-specific investigation

- `16336e1dc469^`와 `16336e1dc469`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 contact actions·page hero·inline metrics·grouped archive에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 contact actions·page hero·inline metrics·grouped archive` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:16336e1dc469:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 contact actions·page hero·inline metrics·grouped archive에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 Contact action과 Projects archive 공용 구조를 시작한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 contact actions·page hero·inline metrics·grouped archive에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:16336e1dc469:END -->

### 9. `2660465c0904` — style(brutalist): 교차 group과 상세 lead 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 alternating groups와 case-study lead

#### Commit-specific investigation

- `2660465c0904^`와 `2660465c0904`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 alternating groups와 case-study lead에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 alternating groups와 case-study lead` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:2660465c0904:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 alternating groups와 case-study lead에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 archive group의 교차 배경과 상세 도입 grid를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 alternating groups와 case-study lead에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:2660465c0904:END -->

### 10. `4621b0a3cb1f` — style(brutalist): 상세 fact와 소개 본문 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 detail facts·media frame·placeholder·intro band

#### Commit-specific investigation

- `4621b0a3cb1f^`와 `4621b0a3cb1f`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 detail facts·media frame·placeholder·intro band에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 detail facts·media frame·placeholder·intro band` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4621b0a3cb1f:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 detail facts·media frame·placeholder·intro band에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 project detail의 fact/media/intro 기초를 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 detail facts·media frame·placeholder·intro band에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4621b0a3cb1f:END -->

### 11. `1d5445cc6f4a` — style(brutalist): 상세 본문과 gallery grid 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 labeled evidence sections·numbered lists·2-column gallery

#### Commit-specific investigation

- `1d5445cc6f4a^`와 `1d5445cc6f4a`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 labeled evidence sections·numbered lists·2-column gallery에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 labeled evidence sections·numbered lists·2-column gallery` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1d5445cc6f4a:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 labeled evidence sections·numbered lists·2-column gallery에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 장문 case-study evidence와 gallery를 구조화한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 labeled evidence sections·numbered lists·2-column gallery에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1d5445cc6f4a:END -->

### 12. `bb5008a8c7b3` — style(brutalist): 다음 프로젝트와 focus card 구성

- **Importance:** B
- **Tags:** RENDERER, A11Y
- **Thread role:** Brutalist 시각 계약의 next-project·not-found recovery·About portrait/skills

#### Commit-specific investigation

- `bb5008a8c7b3^`와 `bb5008a8c7b3`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 next-project·not-found recovery·About portrait/skills에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 next-project·not-found recovery·About portrait/skills` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:bb5008a8c7b3:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 next-project·not-found recovery·About portrait/skills에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 상세 continuation/복구 상태와 About 시작 지면을 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 next-project·not-found recovery·About portrait/skills에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:bb5008a8c7b3:END -->

### 13. `8de2180bcc58` — style(brutalist): focus card와 criteria grid 구성

- **Importance:** B
- **Tags:** RENDERER, A11Y
- **Thread role:** Brutalist 시각 계약의 focus/skill cards와 dark curation criteria

#### Commit-specific investigation

- `8de2180bcc58^`와 `8de2180bcc58`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 focus/skill cards와 dark curation criteria에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 focus/skill cards와 dark curation criteria` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:8de2180bcc58:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 focus/skill cards와 dark curation criteria에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 About 기술 카드와 numbered curation criteria를 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 focus/skill cards와 dark curation criteria에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:8de2180bcc58:END -->

### 14. `a34cd7cd88bf` — style(brutalist): criteria 본문과 재검토 영역 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 curation category·omission·next-review

#### Commit-specific investigation

- `a34cd7cd88bf^`와 `a34cd7cd88bf`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 curation category·omission·next-review에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 curation category·omission·next-review` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:a34cd7cd88bf:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 curation category·omission·next-review에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 큐레이션 presentation을 category/omission/review까지 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 curation category·omission·next-review에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:a34cd7cd88bf:END -->

### 15. `5ca14417cf22` — style(brutalist): 재검토와 resume entry 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 review panel 마감과 resume repeated sections

#### Commit-specific investigation

- `5ca14417cf22^`와 `5ca14417cf22`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 review panel 마감과 resume repeated sections에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 review panel 마감과 resume repeated sections` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5ca14417cf22:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 review panel 마감과 resume repeated sections에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 큐레이션 종료와 Resume entry grammar를 연결한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 review panel 마감과 resume repeated sections에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5ca14417cf22:END -->

### 16. `fad7f0216645` — style(brutalist): resume 본문과 contact hero 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 resume project rows/notes와 Contact blue hero

#### Commit-specific investigation

- `fad7f0216645^`와 `fad7f0216645`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 resume project rows/notes와 Contact blue hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 resume project rows/notes와 Contact blue hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:fad7f0216645:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 resume project rows/notes와 Contact blue hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 Resume 후반과 Contact 시작을 구성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 resume project rows/notes와 Contact blue hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:fad7f0216645:END -->

### 17. `8175392db042` — style(brutalist): contact 상태와 note 목록 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 availability badge·channel grid·note list

#### Commit-specific investigation

- `8175392db042^`와 `8175392db042`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 availability badge·channel grid·note list에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 availability badge·channel grid·note list` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:8175392db042:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 availability badge·channel grid·note list에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 Contact 상태/수단과 반복 note 기반을 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 availability badge·channel grid·note list에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:8175392db042:END -->

### 18. `6fa3a9dc8665` — style(brutalist): note 목록과 anchor link 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 notes/evidence gaps와 Journey milestone card

#### Commit-specific investigation

- `6fa3a9dc8665^`와 `6fa3a9dc8665`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 notes/evidence gaps와 Journey milestone card에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 notes/evidence gaps와 Journey milestone card` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:6fa3a9dc8665:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 notes/evidence gaps와 Journey milestone card에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 Contact note를 마감하고 Journey anchor card를 연다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 notes/evidence gaps와 Journey milestone card에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:6fa3a9dc8665:END -->

### 19. `242ba8e66e0b` — style(brutalist): archive timeline과 track navigation 구성

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Brutalist 시각 계약의 journey archive/current callout와 Interview track shell

#### Commit-specific investigation

- `242ba8e66e0b^`와 `242ba8e66e0b`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 journey archive/current callout와 Interview track shell에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 journey archive/current callout와 Interview track shell` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:242ba8e66e0b:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 journey archive/current callout와 Interview track shell에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 여정 archive와 인터뷰 in-page navigation 기반을 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 journey archive/current callout와 Interview track shell에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:242ba8e66e0b:END -->

### 20. `95e55eda6c51` — style(brutalist): track 목록과 question prompt 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 track index·track/question hierarchy

#### Commit-specific investigation

- `95e55eda6c51^`와 `95e55eda6c51`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 track index·track/question hierarchy에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 track index·track/question hierarchy` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:95e55eda6c51:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 track index·track/question hierarchy에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 Interview 질문 hierarchy와 track 목록을 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 track index·track/question hierarchy에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:95e55eda6c51:END -->

### 21. `11f229d630e9` — style(brutalist): 답변 근거와 footer lead 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 question reference·answer evidence·empty answer·footer lead

#### Commit-specific investigation

- `11f229d630e9^`와 `11f229d630e9`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 question reference·answer evidence·empty answer·footer lead에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 question reference·answer evidence·empty answer·footer lead` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:11f229d630e9:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 question reference·answer evidence·empty answer·footer lead에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 답변 근거와 빈 답변 표현, footer 도입부를 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 question reference·answer evidence·empty answer·footer lead에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:11f229d630e9:END -->

### 22. `b170c73a36d0` — style(brutalist): footer metadata와 blink 동작 구성

- **Importance:** B
- **Tags:** RENDERER, SEO
- **Thread role:** Brutalist 시각 계약의 footer metadata·dashed empty state·crawl/blink animations

#### Commit-specific investigation

- `b170c73a36d0^`와 `b170c73a36d0`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 footer metadata·dashed empty state·crawl/blink animations에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 footer metadata·dashed empty state·crawl/blink animations` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b170c73a36d0:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 footer metadata·dashed empty state·crawl/blink animations에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 footer와 공용 empty block, 두 animation을 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 footer metadata·dashed empty state·crawl/blink animations에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b170c73a36d0:END -->

### 23. `f810c49022be` — style(brutalist): tablet grid 재배치

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 980px tablet grid reflow

#### Commit-specific investigation

- `f810c49022be^`와 `f810c49022be`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 980px tablet grid reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 980px tablet grid reflow` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f810c49022be:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 980px tablet grid reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 desktop grid를 tablet column/reading order로 재배치한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 980px tablet grid reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:f810c49022be:END -->

### 24. `b57da6a41419` — style(brutalist): mobile header와 hero 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 720px native details menu와 stacked header/hero

#### Commit-specific investigation

- `b57da6a41419^`와 `b57da6a41419`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 720px native details menu와 stacked header/hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 720px native details menu와 stacked header/hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b57da6a41419:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 720px native details menu와 stacked header/hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 desktop nav를 `<details>`로 바꾸고 status/debug/hero를 stack한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 720px native details menu와 stacked header/hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b57da6a41419:END -->

### 25. `8168bc76c3e3` — style(brutalist): mobile 프로젝트와 상세 화면 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 mobile metrics/project rows/detail/gallery reflow

#### Commit-specific investigation

- `8168bc76c3e3^`와 `8168bc76c3e3`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 mobile metrics/project rows/detail/gallery reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 mobile metrics/project rows/detail/gallery reflow` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:8168bc76c3e3:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 mobile metrics/project rows/detail/gallery reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 홈·프로젝트·상세·curation grid를 좁은 화면 reading order로 바꾼다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 mobile metrics/project rows/detail/gallery reflow에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:8168bc76c3e3:END -->

### 26. `5551f3fdbb94` — style(brutalist): mobile profile과 resume 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 mobile profile/curation/resume/contact/current-position

#### Commit-specific investigation

- `5551f3fdbb94^`와 `5551f3fdbb94`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 mobile profile/curation/resume/contact/current-position에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 mobile profile/curation/resume/contact/current-position` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5551f3fdbb94:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 mobile profile/curation/resume/contact/current-position에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 About/Resume/Contact 계열의 좁은 화면 배치를 이어서 완성한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 mobile profile/curation/resume/contact/current-position에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5551f3fdbb94:END -->

### 27. `7c08aea7a2f7` — style(brutalist): mobile 여정과 interview 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 mobile journey/interview/footer/missing page

#### Commit-specific investigation

- `7c08aea7a2f7^`와 `7c08aea7a2f7`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 mobile journey/interview/footer/missing page에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 mobile journey/interview/footer/missing page` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:7c08aea7a2f7:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 mobile journey/interview/footer/missing page에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 여정·인터뷰·footer·복구 상태의 mobile layout을 마감한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 mobile journey/interview/footer/missing page에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:7c08aea7a2f7:END -->

### 28. `077ff3d49f30` — style(brutalist): 소형 화면과 인쇄 경계 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Brutalist 시각 계약의 430px hardening·reduced motion·print

#### Commit-specific investigation

- `077ff3d49f30^`와 `077ff3d49f30`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **parent diff에서 430px hardening·reduced motion·print에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist 시각 계약의 430px hardening·reduced motion·print` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:077ff3d49f30:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 430px hardening·reduced motion·print에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/brutalist/brutalist.module.css`의 해당 SHA diff가 최소 viewport, animation 제거, 인쇄용 배경/내비게이션 경계를 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Brutalist stylesheet에 남는다.
- **inspection:** `src/designs/brutalist/brutalist.module.css`의 parent diff에서 430px hardening·reduced motion·print에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/brutalist/brutalist.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:077ff3d49f30:END -->

### 29. `3e6ec5262bdd` — feat(brutalist): 콘텐츠와 탐색 조회 도우미 추가

- **Importance:** B
- **Tags:** CONTENT, ROUTING, RENDERER
- **Thread role:** content/navigation adapter layer

#### Commit-specific investigation

- `3e6ec5262bdd^`와 `3e6ec5262bdd`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **renderer-preserving href, template/tag/group/metric/navigation helpers**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `content/navigation adapter layer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:3e6ec5262bdd:BEGIN -->
- **직전 상태:** 직전 상태에는 **renderer-preserving href, template/tag/group/metric/navigation helpers**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Brutalist가 raw content를 반복 탐색하지 않도록 링크·프로젝트 group·metric·navigation 조회 helper를 한 module에 모은다. 이 시점 helper는 향후 view들이 공유한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 renderer-preserving href, template/tag/group/metric/navigation helpers를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:3e6ec5262bdd:END -->

### 30. `08a2b0c0998f` — feat(brutalist): route 레이블과 기본 shell 구성

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** shared shell start

#### Commit-specific investigation

- `08a2b0c0998f^`와 `08a2b0c0998f`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **exhaustive route label resolver와 `BrutalistShell` root/skip/header/main**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `shared shell start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:08a2b0c0998f:BEGIN -->
- **직전 상태:** 직전 상태에는 **exhaustive route label resolver와 `BrutalistShell` root/skip/header/main**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 여덟 route에 대한 label resolver를 switch로 닫고 root, skip link, header, main landmark를 만든다. 아직 nav/footer/body는 후속 commit에서 채워진다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 exhaustive route label resolver와 `BrutalistShell` root/skip/header/main를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:08a2b0c0998f:END -->

### 31. `cf2fdb36f9fc` — feat(brutalist): 주 탐색과 모바일 메뉴 추가

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** canonical navigation

#### Commit-specific investigation

- `cf2fdb36f9fc^`와 `cf2fdb36f9fc`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **desktop/mobile nav, current-state, debug, `ActionLink` internal/external/mailto 분기**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `canonical navigation` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:cf2fdb36f9fc:BEGIN -->
- **직전 상태:** 직전 상태에는 **desktop/mobile nav, current-state, debug, `ActionLink` internal/external/mailto 분기**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** canonical nav를 desktop과 native disclosure 양쪽에 연결하고 내부 route에는 renderer/debug query를 보존한다. 외부/mailto는 anchor 속성을 사용한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 desktop/mobile nav, current-state, debug, `ActionLink` internal/external/mailto 분기를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:cf2fdb36f9fc:END -->

### 32. `5b44afbc46ef` — feat(brutalist): footer와 홈 히어로 연결

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** footer + Home start

#### Commit-specific investigation

- `5b44afbc46ef^`와 `5b44afbc46ef`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **placement-filtered footerLinks, `HomeRoute` hero, current year/metrics/configured section loop**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `footer + Home start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5b44afbc46ef:BEGIN -->
- **직전 상태:** 직전 상태에는 **placement-filtered footerLinks, `HomeRoute` hero, current year/metrics/configured section loop**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** footer link를 placement로 필터링하고 Home hero를 canonical profile/presentation에 연결한다. Home은 configured section 순서를 순회하지만 일부 section은 아직 후속 구현 전이다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 placement-filtered footerLinks, `HomeRoute` hero, current year/metrics/configured section loop를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5b44afbc46ef:END -->

### 33. `b477ba477127` — feat(brutalist): 홈 섹션 공용 프리미티브 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home/archive primitives

#### Commit-specific investigation

- `b477ba477127^`와 `b477ba477127`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **signal strip, numbered section header, renderer-preserving project row, contact primitive**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home/archive primitives` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b477ba477127:BEGIN -->
- **직전 상태:** 직전 상태에는 **signal strip, numbered section header, renderer-preserving project row, contact primitive**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Home과 archive가 반복 사용할 시각/링크 단위를 private component로 추출해 route별 markup 중복을 줄인다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 signal strip, numbered section header, renderer-preserving project row, contact primitive를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b477ba477127:END -->

### 34. `b30b9b1c3505` — feat(brutalist): 대표 작업과 작업 원칙 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home middle + Projects hero

#### Commit-specific investigation

- `b30b9b1c3505^`와 `b30b9b1c3505`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **signal/featured/system sections와 Projects route hero**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home middle + Projects hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b30b9b1c3505:BEGIN -->
- **직전 상태:** 직전 상태에는 **signal/featured/system sections와 Projects route hero**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** configured Home sequence에 대표 프로젝트와 작업 원칙을 추가하고 Projects archive의 hero/metric 경계를 시작한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 signal/featured/system sections와 Projects route hero를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b30b9b1c3505:END -->

### 35. `85ea663aaf19` — feat(brutalist): 홈 여정과 프로젝트 archive 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home completion + Projects archive

#### Commit-specific investigation

- `85ea663aaf19^`와 `85ea663aaf19`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`slice(-4).reverse()` recent journey copy, contact band, project groups, explicit empty archive**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home completion + Projects archive` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:85ea663aaf19:BEGIN -->
- **직전 상태:** 직전 상태에는 **`slice(-4).reverse()` recent journey copy, contact band, project groups, explicit empty archive**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 최근 네 여정을 복사한 배열에서 역순으로 표시해 source를 mutate하지 않는다. Projects route는 canonical groups를 순회하고 비면 명시적 empty block을 보인다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 `slice(-4).reverse()` recent journey copy, contact band, project groups, explicit empty archive를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:85ea663aaf19:END -->

### 36. `d6b9a99e11ae` — feat(brutalist): 프로젝트 상세 표시 프리미티브 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail primitives

#### Commit-specific investigation

- `d6b9a99e11ae^`와 `d6b9a99e11ae`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **optimized media, ordered actions, text/list section shells, page labels, curation heading**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail primitives` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:d6b9a99e11ae:BEGIN -->
- **직전 상태:** 직전 상태에는 **optimized media, ordered actions, text/list section shells, page labels, curation heading**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 상세 route가 사용할 media/action/evidence primitive를 먼저 정의한다. action 순서와 optional media 처리 정책이 route body 밖으로 분리된다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 optimized media, ordered actions, text/list section shells, page labels, curation heading를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:d6b9a99e11ae:END -->

### 37. `b8268f47e89a` — feat(brutalist): 프로젝트 상세 hero와 소개 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail valid/missing boundary

#### Commit-specific investigation

- `b8268f47e89a^`와 `b8268f47e89a`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`ProjectDetailRoute` unresolved guard, hero, facts, actions, intro**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail valid/missing boundary` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `05b838d52a8b`와 `1ea2a1345b76`가 본문·결과를 완성한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b8268f47e89a:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectDetailRoute` unresolved guard, hero, facts, actions, intro**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** project가 없으면 모든 field 접근 전에 missing view를 반환한다. 유효하면 hero/facts/actions/intro를 구성하고 archive 복귀 경로를 보존한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 `ProjectDetailRoute` unresolved guard, hero, facts, actions, intro를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** `05b838d52a8b`와 `1ea2a1345b76`가 본문·결과를 완성한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b8268f47e89a:END -->

### 38. `05b838d52a8b` — feat(brutalist): 프로젝트 상세 본문과 gallery 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail evidence body

#### Commit-specific investigation

- `05b838d52a8b^`와 `05b838d52a8b`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **problem/solution/architecture/screenshots/resolved stack**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail evidence body` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:05b838d52a8b:BEGIN -->
- **직전 상태:** 직전 상태에는 **problem/solution/architecture/screenshots/resolved stack**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 상세 본문과 gallery를 구성하며 stack ID는 이미 해석된 표시 데이터를 사용한다. optional screenshot 배열은 존재하는 항목만 렌더링한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 problem/solution/architecture/screenshots/resolved stack를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:05b838d52a8b:END -->

### 39. `80724a26820b` — feat(brutalist): 프로필과 기술 소개 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** About identity

#### Commit-specific investigation

- `80724a26820b^`와 `80724a26820b`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`AboutRoute`, optional photo, principles, focus areas, skill groups**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `About identity` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:80724a26820b:BEGIN -->
- **직전 상태:** 직전 상태에는 **`AboutRoute`, optional photo, principles, focus areas, skill groups**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** profile identity와 기술 영역을 canonical content에서 구성하고 photo가 없으면 media 영역을 생략한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 `AboutRoute`, optional photo, principles, focus areas, skill groups를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:80724a26820b:END -->

### 40. `3399b55c3aee` — feat(brutalist): 큐레이션과 경력 소개 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread role:** About experience + feature-gated curation

#### Commit-specific investigation

- `3399b55c3aee^`와 `3399b55c3aee`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **experience, `isSitePageEnabled`, category project resolution/filter**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `About experience + feature-gated curation` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:3399b55c3aee:BEGIN -->
- **직전 상태:** 직전 상태에는 **experience, `isSitePageEnabled`, category project resolution/filter**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 경력 archive를 추가하고 curation capability가 켜진 경우에만 criteria/category/omission/review를 렌더링한다. 잘못된 project ID는 link 없이 제외한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 experience, `isSitePageEnabled`, category project resolution/filter를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:3399b55c3aee:END -->

### 41. `70cf13ef1715` — feat(brutalist): 이력 hero와 경력 요약 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Resume start

#### Commit-specific investigation

- `70cf13ef1715^`와 `70cf13ef1715`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`ResumeRoute`, identity/availability, optional download, summary, experience**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Resume start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:70cf13ef1715:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ResumeRoute`, identity/availability, optional download, summary, experience**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** download URL 유무를 분기하고 numbered summary와 dated experience를 구성한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 `ResumeRoute`, identity/availability, optional download, summary, experience를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:70cf13ef1715:END -->

### 42. `1ea2a1345b76` — feat(brutalist): 프로젝트 결과와 의사결정 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail completion

#### Commit-specific investigation

- `1ea2a1345b76^`와 `1ea2a1345b76`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **highlights/decisions/tradeoffs/results와 explicit empty labels**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1ea2a1345b76:BEGIN -->
- **직전 상태:** 직전 상태에는 **highlights/decisions/tradeoffs/results와 explicit empty labels**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 각 evidence 배열을 별도 section으로 표시하고 비어 있을 때 route copy가 지정한 empty message를 사용한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 highlights/decisions/tradeoffs/results와 explicit empty labels를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1ea2a1345b76:END -->

### 43. `5fa378250d64` — feat(brutalist): 선택 프로젝트와 이력 세부 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Resume completion

#### Commit-specific investigation

- `5fa378250d64^`와 `5fa378250d64`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **resume projectIds resolution/filter, training/education/notes empty fallback**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Resume completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:5fa378250d64:BEGIN -->
- **직전 상태:** 직전 상태에는 **resume projectIds resolution/filter, training/education/notes empty fallback**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 선택 project ID를 canonical project로 해석해 유효 항목만 남기고 training/education/notes를 추가한다. 비어 있는 목록에는 명시적 fallback이 적용된다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 resume projectIds resolution/filter, training/education/notes empty fallback를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:5fa378250d64:END -->

### 44. `b535539ae016` — feat(brutalist): 연락 수단과 안내 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Contact route

#### Commit-specific investigation

- `b535539ae016^`와 `b535539ae016`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **preferred IDs 우선, placement fallback, explicit empty channels, notes**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Contact route` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b535539ae016:BEGIN -->
- **직전 상태:** 직전 상태에는 **preferred IDs 우선, placement fallback, explicit empty channels, notes**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** preferred contact가 해석되지 않으면 placement 기반 links로 후퇴하고 그래도 비면 empty block을 렌더링한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 preferred IDs 우선, placement fallback, explicit empty channels, notes를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b535539ae016:END -->

### 45. `15a765ecb2aa` — feat(brutalist): 여정 milestone 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Journey narrative start

#### Commit-specific investigation

- `15a765ecb2aa^`와 `15a765ecb2aa`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **milestones, anchorProjectIds resolution/filter, empty narrative**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Journey narrative start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:15a765ecb2aa:BEGIN -->
- **직전 상태:** 직전 상태에는 **milestones, anchorProjectIds resolution/filter, empty narrative**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** milestone의 project 참조를 유효 project로만 연결하고 milestone이 없으면 journey empty state를 표시한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 milestones, anchorProjectIds resolution/filter, empty narrative를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:15a765ecb2aa:END -->

### 46. `388446b1a982` — feat(brutalist): 여정 archive와 인터뷰 map 머리말 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Journey completion + Interview start

#### Commit-specific investigation

- `388446b1a982^`와 `388446b1a982`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **dated journey archive/current position, Interview intro/reference/track fragment nav, project Map**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Journey completion + Interview start` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:388446b1a982:BEGIN -->
- **직전 상태:** 직전 상태에는 **dated journey archive/current position, Interview intro/reference/track fragment nav, project Map**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Journey를 full archive와 current state까지 완성하고 Interview hero와 track index를 연다. Interview answer join용 project Map을 준비한다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 dated journey archive/current position, Interview intro/reference/track fragment nav, project Map를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:388446b1a982:END -->

### 47. `f3fc6200a45b` — feat(brutalist): 인터뷰 근거 archive 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Interview evidence

#### Commit-specific investigation

- `f3fc6200a45b^`와 `f3fc6200a45b`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **tracks/questions/references, valid project answers, no-mapped-evidence fallback**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Interview evidence` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f3fc6200a45b:BEGIN -->
- **직전 상태:** 직전 상태에는 **tracks/questions/references, valid project answers, no-mapped-evidence fallback**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 각 answer의 project를 Map에서 찾아 유효하면 detail link를 만들고, 없거나 answer 배열이 비면 명시적 근거 없음 문구를 보인다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 tracks/questions/references, valid project answers, no-mapped-evidence fallback를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:f3fc6200a45b:END -->

### 48. `da8e59d56783` — feat(brutalist): 인터뷰 근거 공백 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Interview gaps

#### Commit-specific investigation

- `da8e59d56783^`와 `da8e59d56783`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **gaps title/body/items와 empty list handling**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Interview gaps` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:da8e59d56783:BEGIN -->
- **직전 상태:** 직전 상태에는 **gaps title/body/items와 empty list handling**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Interview Map 마지막에 선언된 evidence gaps를 별도 section으로 추가해 미지원 영역을 숨기지 않는다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 gaps title/body/items와 empty list handling를 확인했습니다.
- **책임/경계:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:da8e59d56783:END -->

### 49. `e6268c4b7c74` — refactor(brutalist): 내부 helper 공개 범위 정리

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** module API narrowing

#### Commit-specific investigation

- `e6268c4b7c74^`와 `e6268c4b7c74`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **content adapters, primitives, individual views의 `export` 제거**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `module API narrowing` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `caa7df81d899`가 유일한 public route entry를 추가한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:e6268c4b7c74:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **content adapters, primitives, individual views의 `export` 제거**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** registry가 알 필요 없는 helper/view를 module-private로 바꾼다. runtime output을 바꾸는 새 분기보다 ownership을 외부 entry 하나로 수렴시키는 변화다.
- **inspection:** `src/designs/brutalist/brutalist-route.tsx`의 content adapters, primitives, individual views의 `export` 제거를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/designs/brutalist/brutalist-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** `caa7df81d899`가 유일한 public route entry를 추가한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:e6268c4b7c74:END -->

### 50. `caa7df81d899` — feat(brutalist): 모든 route를 renderer에 통합

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** Brutalist public API와 여덟 route dispatch의 최종 경계

#### Commit-specific investigation

- `caa7df81d899^`와 `caa7df81d899`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`BrutalistRoute`, exhaustive switch, shared shell wrapping**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Brutalist public API와 여덟 route dispatch의 최종 경계` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `runtime unknown route에 대한 별도 default UI는 없고 closed union의 compile-time exhaustiveness에 의존한다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 1의 `dd71d28143a8` activation이 이 entry를 lazy registry에 연결한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:caa7df81d899:BEGIN -->
- **직전 상태:** 직전 상태에는 **`BrutalistRoute`, exhaustive switch, shared shell wrapping**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **구현 결정:** module-private view들이 완성된 뒤 exported `BrutalistRoute` 하나가 discriminated route를 여덟 view로 매핑하고 결과를 shared shell에 넣는다. 외부 registry는 helper·view·content adapter를 import할 수 없으며 full-site shell이 모든 route에 적용된다.
- **파일·symbol:** `src/designs/brutalist/brutalist-route.tsx`에서 `BrutalistRoute`, exhaustive switch, shared shell wrapping를 확인했습니다.
- **소유권:** `src/designs/brutalist/brutalist-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **보장/비보장:** 이 SHA는 `Brutalist public API와 여덟 route dispatch의 최종 경계` 경계를 고정합니다. runtime unknown route에 대한 별도 default UI는 없고 closed union의 compile-time exhaustiveness에 의존한다.
- **역사적 연결:** Thread 1의 `dd71d28143a8` activation이 이 entry를 lazy registry에 연결한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```tsx
// caa7df81d899 · src/designs/brutalist/brutalist-route.tsx
export function BrutalistRoute(props: BrutalistRouteProps) {
  return <BrutalistShell {...props}>{renderRoute(props)}</BrutalistShell>;
}
```
<!-- LEARNER-ANSWER:commit:caa7df81d899:END -->

## 5. Invariant evolution

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:invariant:BEGIN -->
최종 invariant는 Brutalist module이 route entry 하나만 공개하고, shell·navigation·link policy·content adapter·view가 같은 module의 private implementation으로 유지되며, mobile/reduced-motion/print와 missing/empty states가 시각 계약에 포함된다는 것입니다.

- 도입·확장·폐쇄의 순서는 commit map에 고정했습니다.
- B-level construction은 route/style surface를 단계적으로 넓히고, A/S-level commit은 owner·dispatch·검증 invariant를 바꿉니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:invariant:END -->

## 6. Failure → Fix → Test and ownership relations

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:relations:BEGIN -->
스타일은 route 전체의 desktop grammar를 순차 구축하고 980/720/mobile/430/reduced-motion/print로 좁힙니다. TSX는 adapters/shell/navigation 뒤 Home→Projects→Detail→About→Resume→Contact→Journey→Interview를 완성합니다. `e6268c4b7c74`가 helper export를 회수하고 `caa7df81d899`가 public route entry를 확정합니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:relations:END -->

## 7. Final architecture or execution flow

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:flow:BEGIN -->
registry entry → `BrutalistRoute` → closed route switch → route view → private content/navigation helpers와 visual primitives → `BrutalistShell` root/header/main/footer 순서입니다.

각 단계에서 optional/empty/missing reference가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 commit의 non-guarantee에 남겼습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:flow:END -->

## 8. Runtime and verification evidence

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:runtime:BEGIN -->
- **실행한 repository test/build:** 없음.
- **정적 확인:** 지정 branch의 commit classification, commit bodies, exact commit diff와 historical file 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 container에서 직접 clone 시 DNS가 `github.com`을 해석하지 못해 historical worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next build 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** code/test implementation의 존재와 범위는 inspection으로 확인했고, runtime pass/fail은 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:runtime:END -->

## 9. Learning-completion checks

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·subject·importance·tags를 commit map과 commit section에서 동일하게 유지했습니다.
- [x] 각 SHA에 concrete file과 symbol/selector/route focus를 기록했습니다.
- [x] 이전 상태, owner, absence/fallback, guarantee/non-guarantee, 후속 관계를 채웠습니다.
- [x] S/A/B depth를 구분했습니다.
- [x] 실행하지 않은 test를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:checks:END -->
===== END FILE: 03-brutalist-design-system-construction.md =====

===== BEGIN FILE: 04-cinematic-design-system-construction.md =====
# Thread: Cinematic design system construction

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `05-full-site-visual-systems`
>
> Phase 1 audit에서 확정한 authoritative scaffold입니다. Phase 2에서는 answer marker 내부만 채웁니다.

## 0. Scope and authority

- Commit SHA, subject, importance, tags는 branch의 `commit/commit-importance.md`와 exact commit metadata를 기준으로 고정했습니다.
- **Thread boundary:** Cinematic CSS module과 route module construction을 포함합니다. registry activation/API 연결은 Thread 1에 둡니다.
- 다른 branch, final HEAD의 후대 구현, 실행하지 않은 command 결과를 사용하지 않습니다.

## 1. Thread goal

Cinematic의 암실 palette와 image-led chapter grammar, shared frame/media, full-site route composition과 reference/empty-state 경계를 복원합니다.

### Frozen invariant target

최종 invariant는 route가 공통 `Frame`과 `Media` boundary를 사용하고 internal link가 선택 상태를 보존하며, route별 content join과 absence는 명시적으로 처리하되 각 route가 보장하지 않는 empty/reference 상태도 그대로 남긴다는 것입니다.

## 2. Commit map

| 순서 | Commit | Subject | Importance | Tags | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `74a27c95eb1c` | style(cinematic): 암실 palette와 shell 기초 구성 | B | ROUTING, RENDERER | Cinematic 시각 계약의 dark scoped tokens·selection/focus/skip-link·sticky glass header |
| 2 | `197c0781f1b9` | feat(cinematic): 링크와 chapter 표기 프리미티브 추가 | B | RENDERER | navigation primitives |
| 3 | `3b72294a0fd7` | style(cinematic): 모바일 탐색과 hero 매체 구성 | B | ROUTING, RENDERER | Cinematic 시각 계약의 native mobile disclosure와 image-led hero |
| 4 | `e2dbb1b7c7d0` | feat(cinematic): 공용 frame과 media 추가 | B | RENDERER | shared full-site frame |
| 5 | `22c4593809bf` | feat(cinematic): 프로젝트 chapter 추가 | B | RENDERER | reusable project chapter |
| 6 | `bb7a742122fd` | style(cinematic): chapter와 archive 지면 구성 | B | RENDERER | Cinematic 시각 계약의 long-form chapter/archive/sticky copy/media hover/dual panel |
| 7 | `29430d7dfe67` | feat(cinematic-home): 소개와 대표 프로젝트 구성 | B | RENDERER | Home route |
| 8 | `f417e3e70b1f` | feat(cinematic-projects): 프로젝트 archive 구성 | B | RENDERER | Projects archive |
| 9 | `1f4c35853502` | style(cinematic): 상세와 이력 grid 구성 | B | RENDERER | Cinematic 시각 계약의 project evidence·profile essays·resume grid |
| 10 | `2e9f70067daf` | feat(cinematic-project): 상세 hero와 매체 구성 | B | RENDERER | Project detail boundary |
| 11 | `2f404402a2ea` | feat(cinematic-project): 상세 서사와 gallery 구성 | B | RENDERER | Project detail completion |
| 12 | `95ee01decc8f` | style(cinematic): 프로필과 콘텐츠 section 구성 | B | CONTENT, RENDERER | Cinematic 시각 계약의 profile facts·long-form sections·chronology·evidence/contact/gaps |
| 13 | `4eefc512d05c` | feat(cinematic-about): 프로필과 경력 소개 구성 | B | RENDERER | About route |
| 14 | `ee692d893a11` | feat(cinematic-about): 큐레이션 archive 구성 | B | CONTENT, RENDERER | feature-gated curation |
| 15 | `52f13fcc5a12` | style(cinematic): 여정 timeline과 답변 근거 구성 | B | RENDERER | Cinematic 시각 계약의 milestone/timeline/current-position/interview evidence grammar |
| 16 | `7cc23349f59f` | feat(cinematic): 이력과 연락 route 구성 | B | ROUTING, RENDERER | Resume and Contact |
| 17 | `c3aba5da6a10` | style(cinematic): 인터뷰 근거와 반응형 동작 구성 | B | RENDERER | Cinematic 시각 계약의 Interview evidence completion·980/640 reflow·reduced motion |
| 18 | `bddb3cc18eed` | feat(cinematic-journey): 여정 archive 구성 | B | RENDERER | Journey route |
| 19 | `2a0f0aadee1c` | feat(cinematic-interview): 인터뷰 근거 map 구성 | B | RENDERER | Interview route |

## 3. Historical baseline

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:baseline:BEGIN -->
- **직전 상태:** 공통 delegation은 있었지만 Cinematic 전용 frame, media, link, chapter, responsive layout과 route view가 없었습니다.
- **경계 판단:** Cinematic CSS module과 route module construction을 포함합니다. registry activation/API 연결은 Thread 1에 둡니다.
- **복원 기준:** 각 commit의 parent와 exact SHA tree만 사용하고 final HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:baseline:END -->

## 4. Commit-by-commit reconstruction

### 1. `74a27c95eb1c` — style(cinematic): 암실 palette와 shell 기초 구성

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Cinematic 시각 계약의 dark scoped tokens·selection/focus/skip-link·sticky glass header

#### Commit-specific investigation

- `74a27c95eb1c^`와 `74a27c95eb1c`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **parent diff에서 dark scoped tokens·selection/focus/skip-link·sticky glass header에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic 시각 계약의 dark scoped tokens·selection/focus/skip-link·sticky glass header` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:74a27c95eb1c:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 dark scoped tokens·selection/focus/skip-link·sticky glass header에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/cinematic/cinematic.module.css`의 해당 SHA diff가 암실 palette와 shell root/accessibility 계약을 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Cinematic stylesheet에 남는다.
- **inspection:** `src/designs/cinematic/cinematic.module.css`의 parent diff에서 dark scoped tokens·selection/focus/skip-link·sticky glass header에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/cinematic/cinematic.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:74a27c95eb1c:END -->

### 2. `197c0781f1b9` — feat(cinematic): 링크와 chapter 표기 프리미티브 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** navigation primitives

#### Commit-specific investigation

- `197c0781f1b9^`와 `197c0781f1b9`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`routeHref`, `isCurrentNavigation`, `CinematicLink`, `ChapterLabel`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `navigation primitives` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:197c0781f1b9:BEGIN -->
- **직전 상태:** 직전 상태에는 **`routeHref`, `isCurrentNavigation`, `CinematicLink`, `ChapterLabel`**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 내부 route는 cinematic/debug state를 보존하고 외부/mailto는 anchor로 분기한다. chapter label을 반복 가능한 DOM 단위로 만든다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `routeHref`, `isCurrentNavigation`, `CinematicLink`, `ChapterLabel`를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:197c0781f1b9:END -->

### 3. `3b72294a0fd7` — style(cinematic): 모바일 탐색과 hero 매체 구성

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Cinematic 시각 계약의 native mobile disclosure와 image-led hero

#### Commit-specific investigation

- `3b72294a0fd7^`와 `3b72294a0fd7`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **parent diff에서 native mobile disclosure와 image-led hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic 시각 계약의 native mobile disclosure와 image-led hero` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:3b72294a0fd7:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 native mobile disclosure와 image-led hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/cinematic/cinematic.module.css`의 해당 SHA diff가 mobile navigation과 two-column hero/media composition을 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Cinematic stylesheet에 남는다.
- **inspection:** `src/designs/cinematic/cinematic.module.css`의 parent diff에서 native mobile disclosure와 image-led hero에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/cinematic/cinematic.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:3b72294a0fd7:END -->

### 4. `e2dbb1b7c7d0` — feat(cinematic): 공용 frame과 media 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** shared full-site frame

#### Commit-specific investigation

- `e2dbb1b7c7d0^`와 `e2dbb1b7c7d0`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`Frame`, `Media`, canonical nav/footer, root/main shell**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `shared full-site frame` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: 후속 route view들이 `Frame` 안에 body만 제공한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:e2dbb1b7c7d0:BEGIN -->
- **직전 상태:** 직전 상태에는 **`Frame`, `Media`, canonical nav/footer, root/main shell**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 모든 route가 사용할 root frame과 media boundary를 만든다. navigation/footer는 canonical shell content를 읽고 current path/template/debug state를 링크에 보존한다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `Frame`, `Media`, canonical nav/footer, root/main shell를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** 후속 route view들이 `Frame` 안에 body만 제공한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:e2dbb1b7c7d0:END -->

### 5. `22c4593809bf` — feat(cinematic): 프로젝트 chapter 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** reusable project chapter

#### Commit-specific investigation

- `22c4593809bf^`와 `22c4593809bf`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`ProjectChapter`, sticky evidence copy, media link, accessible aria-label**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `reusable project chapter` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:22c4593809bf:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectChapter`, sticky evidence copy, media link, accessible aria-label**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** 프로젝트 summary/facts/action과 media를 하나의 chapter로 묶는다. archive와 Home이 같은 representation을 재사용하며 link의 label을 명시한다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `ProjectChapter`, sticky evidence copy, media link, accessible aria-label를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:22c4593809bf:END -->

### 6. `bb7a742122fd` — style(cinematic): chapter와 archive 지면 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Cinematic 시각 계약의 long-form chapter/archive/sticky copy/media hover/dual panel

#### Commit-specific investigation

- `bb7a742122fd^`와 `bb7a742122fd`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **parent diff에서 long-form chapter/archive/sticky copy/media hover/dual panel에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic 시각 계약의 long-form chapter/archive/sticky copy/media hover/dual panel` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:bb7a742122fd:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 long-form chapter/archive/sticky copy/media hover/dual panel에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/cinematic/cinematic.module.css`의 해당 SHA diff가 ProjectChapter와 archive가 사용할 장문 지면·sticky relationship을 만든다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Cinematic stylesheet에 남는다.
- **inspection:** `src/designs/cinematic/cinematic.module.css`의 parent diff에서 long-form chapter/archive/sticky copy/media hover/dual panel에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/cinematic/cinematic.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:bb7a742122fd:END -->

### 7. `29430d7dfe67` — feat(cinematic-home): 소개와 대표 프로젝트 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Home route

#### Commit-specific investigation

- `29430d7dfe67^`와 `29430d7dfe67`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`HomeView`, presentation section order, featured→all fallback, `slice(0, 4)`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home route` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:29430d7dfe67:BEGIN -->
- **직전 상태:** 직전 상태에는 **`HomeView`, presentation section order, featured→all fallback, `slice(0, 4)`**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** presentation에 정의된 section ID 순서대로 node를 재생한다. featured가 비면 전체 projects로 후퇴하고 최대 네 개를 chapter로 보여 준다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `HomeView`, presentation section order, featured→all fallback, `slice(0, 4)`를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:29430d7dfe67:END -->

### 8. `f417e3e70b1f` — feat(cinematic-projects): 프로젝트 archive 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Projects archive

#### Commit-specific investigation

- `f417e3e70b1f^`와 `f417e3e70b1f`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`ProjectsView`, all projects as `ProjectChapter`, padded count**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Projects archive` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `빈 project archive는 단순 빈 목록이 되며 명시적 recovery copy를 보장하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:f417e3e70b1f:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectsView`, all projects as `ProjectChapter`, padded count**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** canonical projects 전체를 chapter representation으로 순회하고 숫자 label을 padding한다. 이 SHA에는 projects 배열이 비었을 때 별도 empty message가 없다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `ProjectsView`, all projects as `ProjectChapter`, padded count를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 빈 project archive는 단순 빈 목록이 되며 명시적 recovery copy를 보장하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:f417e3e70b1f:END -->

### 9. `1f4c35853502` — style(cinematic): 상세와 이력 grid 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Cinematic 시각 계약의 project evidence·profile essays·resume grid

#### Commit-specific investigation

- `1f4c35853502^`와 `1f4c35853502`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **parent diff에서 project evidence·profile essays·resume grid에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic 시각 계약의 project evidence·profile essays·resume grid` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:1f4c35853502:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 project evidence·profile essays·resume grid에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/cinematic/cinematic.module.css`의 해당 SHA diff가 상세/프로필/이력 장문 content의 공용 grid를 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Cinematic stylesheet에 남는다.
- **inspection:** `src/designs/cinematic/cinematic.module.css`의 parent diff에서 project evidence·profile essays·resume grid에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/cinematic/cinematic.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:1f4c35853502:END -->

### 10. `2e9f70067daf` — feat(cinematic-project): 상세 hero와 매체 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail boundary

#### Commit-specific investigation

- `2e9f70067daf^`와 `2e9f70067daf`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`ProjectDetailView`, unresolved guard, archive back link, facts, hero media**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail boundary` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: `2f404402a2ea`가 full narrative와 galleries를 추가한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:2e9f70067daf:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectDetailView`, unresolved guard, archive back link, facts, hero media**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** project가 없으면 field 접근 전에 missing view를 반환하고, 유효하면 project facts와 hero media를 구성한다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `ProjectDetailView`, unresolved guard, archive back link, facts, hero media를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** `2f404402a2ea`가 full narrative와 galleries를 추가한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:2e9f70067daf:END -->

### 11. `2f404402a2ea` — feat(cinematic-project): 상세 서사와 gallery 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Project detail completion

#### Commit-specific investigation

- `2f404402a2ea^`와 `2f404402a2ea`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **optional narrative/evidence arrays, resolved stack fallback, detail links, hero screenshot de-duplication**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail completion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:2f404402a2ea:BEGIN -->
- **직전 상태:** 직전 상태에는 **optional narrative/evidence arrays, resolved stack fallback, detail links, hero screenshot de-duplication**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** problem/solution/architecture/decisions/highlights/tradeoffs/results와 gallery를 조건부로 구성한다. hero screenshot은 supporting images에서 제거해 중복 표시를 피하고, stack lookup 실패 시 raw ID text를 보존한다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 optional narrative/evidence arrays, resolved stack fallback, detail links, hero screenshot de-duplication를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:2f404402a2ea:END -->

### 12. `95ee01decc8f` — style(cinematic): 프로필과 콘텐츠 section 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread role:** Cinematic 시각 계약의 profile facts·long-form sections·chronology·evidence/contact/gaps

#### Commit-specific investigation

- `95ee01decc8f^`와 `95ee01decc8f`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **parent diff에서 profile facts·long-form sections·chronology·evidence/contact/gaps에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic 시각 계약의 profile facts·long-form sections·chronology·evidence/contact/gaps` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:95ee01decc8f:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 profile facts·long-form sections·chronology·evidence/contact/gaps에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/cinematic/cinematic.module.css`의 해당 SHA diff가 About/Resume/Contact/Journey/Interview가 공유할 content grammar를 추가한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Cinematic stylesheet에 남는다.
- **inspection:** `src/designs/cinematic/cinematic.module.css`의 parent diff에서 profile facts·long-form sections·chronology·evidence/contact/gaps에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/cinematic/cinematic.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:95ee01decc8f:END -->

### 13. `4eefc512d05c` — feat(cinematic-about): 프로필과 경력 소개 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** About route

#### Commit-specific investigation

- `4eefc512d05c^`와 `4eefc512d05c`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`AboutView`, optional profile photo, facts, principles, skills/groups, experience**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `About route` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:4eefc512d05c:BEGIN -->
- **직전 상태:** 직전 상태에는 **`AboutView`, optional profile photo, facts, principles, skills/groups, experience**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** canonical profile/skills/experience를 장문 cinematic section으로 구성하고 photo 유무를 분기한다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `AboutView`, optional profile photo, facts, principles, skills/groups, experience를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:4eefc512d05c:END -->

### 14. `ee692d893a11` — feat(cinematic-about): 큐레이션 archive 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread role:** feature-gated curation

#### Commit-specific investigation

- `ee692d893a11^`와 `ee692d893a11`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`isSitePageEnabled`, category projectIds resolution/filter, omissions/nextReview**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `feature-gated curation` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:ee692d893a11:BEGIN -->
- **직전 상태:** 직전 상태에는 **`isSitePageEnabled`, category projectIds resolution/filter, omissions/nextReview**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** curation capability가 켜졌을 때만 archive를 추가하고 category ID를 canonical projects로 해석해 유효한 링크만 남긴다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `isSitePageEnabled`, category projectIds resolution/filter, omissions/nextReview를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:ee692d893a11:END -->

### 15. `52f13fcc5a12` — style(cinematic): 여정 timeline과 답변 근거 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Cinematic 시각 계약의 milestone/timeline/current-position/interview evidence grammar

#### Commit-specific investigation

- `52f13fcc5a12^`와 `52f13fcc5a12`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **parent diff에서 milestone/timeline/current-position/interview evidence grammar에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic 시각 계약의 milestone/timeline/current-position/interview evidence grammar` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:52f13fcc5a12:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 milestone/timeline/current-position/interview evidence grammar에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/cinematic/cinematic.module.css`의 해당 SHA diff가 Journey와 Interview route의 장문 timeline/evidence 지면을 정의한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Cinematic stylesheet에 남는다.
- **inspection:** `src/designs/cinematic/cinematic.module.css`의 parent diff에서 milestone/timeline/current-position/interview evidence grammar에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/cinematic/cinematic.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** 이 CSS만으로 대응 DOM class가 실제 route에서 사용되거나 브라우저별 layout이 정확하다는 사실은 보장되지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:52f13fcc5a12:END -->

### 16. `7cc23349f59f` — feat(cinematic): 이력과 연락 route 구성

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Thread role:** Resume and Contact

#### Commit-specific investigation

- `7cc23349f59f^`와 `7cc23349f59f`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **resume selected-project ID resolution/filter, optional download, contact preferred→placement fallback, empty channels**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Resume and Contact` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `Resume의 모든 선택 배열이 빈 경우 각각 별도 empty-state를 제공하는 것은 아니며 notes에만 명시적 fallback이 있다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:7cc23349f59f:BEGIN -->
- **직전 상태:** 직전 상태에는 **resume selected-project ID resolution/filter, optional download, contact preferred→placement fallback, empty channels**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** Resume는 project IDs를 해석해 유효한 선택만 표시하고 download를 조건부로 렌더링한다. Contact는 preferred links가 없으면 placement 기반 links로 후퇴하고 그래도 없으면 명시적 empty copy를 보인다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 resume selected-project ID resolution/filter, optional download, contact preferred→placement fallback, empty channels를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** Resume의 모든 선택 배열이 빈 경우 각각 별도 empty-state를 제공하는 것은 아니며 notes에만 명시적 fallback이 있다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:7cc23349f59f:END -->

### 17. `c3aba5da6a10` — style(cinematic): 인터뷰 근거와 반응형 동작 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Cinematic 시각 계약의 Interview evidence completion·980/640 reflow·reduced motion

#### Commit-specific investigation

- `c3aba5da6a10^`와 `c3aba5da6a10`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **parent diff에서 Interview evidence completion·980/640 reflow·reduced motion에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Cinematic 시각 계약의 Interview evidence completion·980/640 reflow·reduced motion` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:c3aba5da6a10:BEGIN -->
- **직전 상태:** 직전 stylesheet에는 **parent diff에서 Interview evidence completion·980/640 reflow·reduced motion에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인**를 완결하는 selector/media-rule 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** `src/designs/cinematic/cinematic.module.css`의 해당 SHA diff가 sticky/grid를 좁은 화면에서 static/stacked로 바꾸고 animation/transition을 억제한다. 이전 상태에는 이 selector 묶음 또는 breakpoint 재배치가 없었고, 이후 DOM이 사용할 지면 규칙을 이 시점부터 제공한다. 소유권은 콘텐츠/route가 아니라 Cinematic stylesheet에 남는다.
- **inspection:** `src/designs/cinematic/cinematic.module.css`의 parent diff에서 Interview evidence completion·980/640 reflow·reduced motion에 대응하는 selector·media rule·token과 기존 선언의 재배치를 확인를 확인했습니다.
- **책임/경계:** DOM은 route module이 계속 소유하고, 이 SHA가 추가한 visual/layout state는 `src/designs/cinematic/cinematic.module.css`의 scoped stylesheet가 소유합니다.
- **비보장:** reduced-motion·focus·semantic 보조는 CSS/DOM 계약의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:c3aba5da6a10:END -->

### 18. `bddb3cc18eed` — feat(cinematic-journey): 여정 archive 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Journey route

#### Commit-specific investigation

- `bddb3cc18eed^`와 `bddb3cc18eed`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **milestone anchor project resolution/filter, dated archive, direct `item.projectId` URL, current position**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Journey route` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `archive `projectId`가 canonical project에 존재한다는 보장은 이 renderer에서 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:bddb3cc18eed:BEGIN -->
- **직전 상태:** 직전 상태에는 **milestone anchor project resolution/filter, dated archive, direct `item.projectId` URL, current position**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** milestone anchor IDs는 실제 project로 해석한 뒤 링크하지만 broader archive의 `item.projectId`는 존재 여부를 검사하지 않고 URL로 변환한다. narrative와 chronological history/current state를 세 구획으로 구성한다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 milestone anchor project resolution/filter, dated archive, direct `item.projectId` URL, current position를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** archive `projectId`가 canonical project에 존재한다는 보장은 이 renderer에서 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:bddb3cc18eed:END -->

### 19. `2a0f0aadee1c` — feat(cinematic-interview): 인터뷰 근거 map 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread role:** Interview route

#### Commit-specific investigation

- `2a0f0aadee1c^`와 `2a0f0aadee1c`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`projectsById` Map, external reference, track/question answers, missing/empty evidence, gaps**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Interview route` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 1의 `b8de57f130eb`가 완성된 route entry를 registry에 활성화한다.

#### Learning record

<!-- LEARNER-ANSWER:commit:2a0f0aadee1c:BEGIN -->
- **직전 상태:** 직전 상태에는 **`projectsById` Map, external reference, track/question answers, missing/empty evidence, gaps**에 해당하는 실행 가능한 route/component 경계가 아직 없었습니다.
- **변경:** answer project를 Map으로 해석하고 누락이면 `noMappedEvidence`를 표시한다. answer 배열과 gaps 배열이 비어 있는 경우도 각각 fallback copy를 사용한다.
- **inspection:** `src/designs/cinematic/cinematic-route.tsx`의 `projectsById` Map, external reference, track/question answers, missing/empty evidence, gaps를 확인했습니다.
- **책임/경계:** `src/designs/cinematic/cinematic-route.tsx`가 이 기능의 DOM 조립·분기·링크/상태 표현을 소유하며 source content의 원본 수명은 변경하지 않습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** Thread 1의 `b8de57f130eb`가 완성된 route entry를 registry에 활성화한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:2a0f0aadee1c:END -->

## 5. Invariant evolution

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:invariant:BEGIN -->
최종 invariant는 route가 공통 `Frame`과 `Media` boundary를 사용하고 internal link가 선택 상태를 보존하며, route별 content join과 absence는 명시적으로 처리하되 각 route가 보장하지 않는 empty/reference 상태도 그대로 남긴다는 것입니다.

- 도입·확장·폐쇄의 순서는 commit map에 고정했습니다.
- B-level construction은 route/style surface를 단계적으로 넓히고, A/S-level commit은 owner·dispatch·검증 invariant를 바꿉니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:invariant:END -->

## 6. Failure → Fix → Test and ownership relations

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:relations:BEGIN -->
CSS와 TSX가 교차하며 shell·chapter·archive·detail/profile/resume/journey/interview 지면을 만든 뒤 반응형/reduced-motion을 마감합니다. Project detail과 Interview는 unresolved reference를 명시적으로 처리하지만 Projects archive의 empty state와 Journey archive의 direct projectId URL은 별도 검증이 없어 non-guarantee로 남습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:relations:END -->

## 7. Final architecture or execution flow

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:flow:BEGIN -->
registry entry → Cinematic route entry → route-specific view → `Frame`의 navigation/main/footer → `CinematicLink`/`ProjectChapter`/`Media` primitives 순서입니다.

각 단계에서 optional/empty/missing reference가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 commit의 non-guarantee에 남겼습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:flow:END -->

## 8. Runtime and verification evidence

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:runtime:BEGIN -->
- **실행한 repository test/build:** 없음.
- **정적 확인:** 지정 branch의 commit classification, commit bodies, exact commit diff와 historical file 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 container에서 직접 clone 시 DNS가 `github.com`을 해석하지 못해 historical worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next build 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** code/test implementation의 존재와 범위는 inspection으로 확인했고, runtime pass/fail은 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:runtime:END -->

## 9. Learning-completion checks

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·subject·importance·tags를 commit map과 commit section에서 동일하게 유지했습니다.
- [x] 각 SHA에 concrete file과 symbol/selector/route focus를 기록했습니다.
- [x] 이전 상태, owner, absence/fallback, guarantee/non-guarantee, 후속 관계를 채웠습니다.
- [x] S/A/B depth를 구분했습니다.
- [x] 실행하지 않은 test를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:checks:END -->
===== END FILE: 04-cinematic-design-system-construction.md =====

===== BEGIN FILE: 05-design-and-classic-renderer-extraction.md =====
# Thread: Design and Classic renderer extraction

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `05-full-site-visual-systems`
>
> Phase 1 audit에서 확정한 authoritative scaffold입니다. Phase 2에서는 answer marker 내부만 채웁니다.

## 0. Scope and authority

- Commit SHA, subject, importance, tags는 branch의 `commit/commit-importance.md`와 exact commit metadata를 기준으로 고정했습니다.
- **Thread boundary:** route별 extraction과 최종 Design/Classic dispatcher만 포함합니다. 새로운 visual feature construction이나 cross-design registry unification은 각각 다른 category/Thread에 둡니다.
- 다른 branch, final HEAD의 후대 구현, 실행하지 않은 command 결과를 사용하지 않습니다.

## 1. Thread goal

App Router와 공용 component에 남아 있던 Design·Classic presentation을 route module로 옮기고, 각 디자인을 단일 dispatcher 아래 닫는 ownership transfer를 복원합니다.

### Frozen invariant target

최종 invariant는 App Router가 content/metadata/notFound/structured-data/view-model 생성만 소유하고, Design·Classic module이 route guard·shell·presentation·private helper를 소유하며, 외부는 각 `index.tsx` dispatcher 하나만 호출한다는 것입니다.

## 2. Commit map

| 순서 | Commit | Subject | Importance | Tags | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b8d35db40ed1` | refactor(routes): Design 홈 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | Home ownership transfer |
| 2 | `29943a185465` | refactor(routes): Design 프로젝트 목록 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | Projects ownership transfer |
| 3 | `b76fa3f1d6be` | refactor(routes): Design 프로젝트 상세 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | Project detail ownership transfer |
| 4 | `e4feecdc7a04` | refactor(routes): Design 소개 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | About ownership transfer |
| 5 | `43fac33fdda3` | refactor(routes): Design 이력 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | Resume ownership transfer |
| 6 | `b11a4e4a3d72` | refactor(routes): Design 연락 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | Contact ownership transfer |
| 7 | `8a0bd21f7557` | refactor(routes): Design 여정 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | Journey ownership transfer |
| 8 | `2fee9efda711` | refactor(routes): Design 인터뷰 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | Interview ownership transfer |
| 9 | `05e1cebd0b70` | refactor(design): Design route dispatcher 추가 | A | ARCH, ROUTING, RENDERER | Design의 여덟 독립 module을 하나의 public dispatcher로 통합 |
| 10 | `15ab994dabfd` | refactor(classic-home): 홈 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic Home ownership transfer and shared-component cleanup |
| 11 | `91e44a4de72c` | refactor(classic-projects): 프로젝트 목록 renderer를 이동 | B | RENDERER, REFACTOR | Classic Projects ownership transfer |
| 12 | `7a65f0522061` | refactor(classic-project): 상세 renderer를 독립 모듈로 완성 | B | RENDERER, REFACTOR | Classic Project detail ownership transfer |
| 13 | `25fa6b575c31` | refactor(classic-about): 소개 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic About large presentation transfer |
| 14 | `88fb0a09db5e` | refactor(classic-resume): 이력 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic Resume large presentation transfer |
| 15 | `17e0a9ad0acb` | refactor(classic-contact): 연락 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic Contact transfer |
| 16 | `c44f91b0d40d` | refactor(classic-journey): 여정 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic Journey transfer and shared prepared model |
| 17 | `a5d8f288baa2` | refactor(classic-interview): 인터뷰 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic Interview transfer and shared prepared model |
| 18 | `6b193d084e69` | refactor(classic): Classic route dispatcher 추가 | A | ARCH, ROUTING, RENDERER | Classic의 여덟 독립 route module을 하나의 public entry로 통합 |

## 3. Historical baseline

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:baseline:BEGIN -->
- **직전 상태:** Design과 Classic은 사이트의 초기 renderer였기 때문에 App Router page와 shared component가 shell·derived props·markup을 직접 소유했습니다. dedicated 세 디자인과 달리 public single-entry module 경계가 없었습니다.
- **경계 판단:** route별 extraction과 최종 Design/Classic dispatcher만 포함합니다. 새로운 visual feature construction이나 cross-design registry unification은 각각 다른 category/Thread에 둡니다.
- **복원 기준:** 각 commit의 parent와 exact SHA tree만 사용하고 final HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:baseline:END -->

## 4. Commit-by-commit reconstruction

### 1. `b8d35db40ed1` — refactor(routes): Design 홈 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** Home ownership transfer

#### Commit-specific investigation

- `b8d35db40ed1^`와 `b8d35db40ed1`를 비교하고 `src/app/page.tsx, src/designs/design/home-route.tsx`에서 **App page의 Design branch, `HomeRoute` route guard, `createDesignShellProps`, private `HomeView`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Home ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: 후속 `05e1cebd0b70` dispatcher가 이 module을 단일 Design entry 아래 묶는다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b8d35db40ed1:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 Design branch, `HomeRoute` route guard, `createDesignShellProps`, private `HomeView`**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** App Router가 Design home의 shell/section composition을 직접 조립하던 책임을 renderer module로 넘긴다. page는 content/template/debug를 준비하고 route props만 전달하며 module이 route discriminator와 shell을 소유한다.
- **inspection:** `src/app/page.tsx, src/designs/design/home-route.tsx`의 App page의 Design branch, `HomeRoute` route guard, `createDesignShellProps`, private `HomeView`를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/page.tsx, src/designs/design/home-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **다음 관계:** 후속 `05e1cebd0b70` dispatcher가 이 module을 단일 Design entry 아래 묶는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b8d35db40ed1:END -->

### 2. `29943a185465` — refactor(routes): Design 프로젝트 목록 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** Projects ownership transfer

#### Commit-specific investigation

- `29943a185465^`와 `29943a185465`를 비교하고 `src/app/projects/page.tsx, src/designs/design/projects-route.tsx`에서 **App page의 derived pageCopy/metrics/groups 제거, renderer route guard와 shell/derived field consumption**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Projects ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:29943a185465:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 derived pageCopy/metrics/groups 제거, renderer route guard와 shell/derived field consumption**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** Projects page에서 shell·copy·featured/group/metric 조립을 제거하고 prepared projects view model 전체를 renderer에 넘긴다. renderer가 route guard와 Design shell을 소유하고 내부 view는 private가 된다.
- **inspection:** `src/app/projects/page.tsx, src/designs/design/projects-route.tsx`의 App page의 derived pageCopy/metrics/groups 제거, renderer route guard와 shell/derived field consumption를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/projects/page.tsx, src/designs/design/projects-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:29943a185465:END -->

### 3. `b76fa3f1d6be` — refactor(routes): Design 프로젝트 상세 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** Project detail ownership transfer

#### Commit-specific investigation

- `b76fa3f1d6be^`와 `b76fa3f1d6be`를 비교하고 `src/app/projects/[projectId]/page.tsx, src/designs/design/project-detail-route.tsx`에서 **App page의 StructuredData/notFound 유지, renderer guard/shell/hero/body, helper private화**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Project detail ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b76fa3f1d6be:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 StructuredData/notFound 유지, renderer guard/shell/hero/body, helper private화**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** framework metadata·notFound·structured data는 App page에 남기고 visual hero/body/shell은 Design module로 이동한다. route discriminator가 `project-detail`이 아니면 null을 반환하며 내부 section helper는 공개되지 않는다.
- **inspection:** `src/app/projects/[projectId]/page.tsx, src/designs/design/project-detail-route.tsx`의 App page의 StructuredData/notFound 유지, renderer guard/shell/hero/body, helper private화를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/projects/[projectId]/page.tsx, src/designs/design/project-detail-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b76fa3f1d6be:END -->

### 4. `e4feecdc7a04` — refactor(routes): Design 소개 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** About ownership transfer

#### Commit-specific investigation

- `e4feecdc7a04^`와 `e4feecdc7a04`를 비교하고 `src/app/about/page.tsx, src/designs/design/about-route.tsx`에서 **App page의 Design branch, renderer shell, experience/curation completion, private curation helpers**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `About ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:e4feecdc7a04:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 Design branch, renderer shell, experience/curation completion, private curation helpers**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** App page는 content load·page enablement·template dispatch만 남기고 About identity/skills/experience/feature-gated curation 조립을 module로 넘긴다. Curation helper도 module-private가 된다.
- **inspection:** `src/app/about/page.tsx, src/designs/design/about-route.tsx`의 App page의 Design branch, renderer shell, experience/curation completion, private curation helpers를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/about/page.tsx, src/designs/design/about-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:e4feecdc7a04:END -->

### 5. `43fac33fdda3` — refactor(routes): Design 이력 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** Resume ownership transfer

#### Commit-specific investigation

- `43fac33fdda3^`와 `43fac33fdda3`를 비교하고 `src/app/resume/page.tsx, src/designs/design/resume-route.tsx`에서 **App branch 제거, renderer shell, optional experience/education/notes**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Resume ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:43fac33fdda3:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App branch 제거, renderer shell, optional experience/education/notes**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** prepared resume view model을 Design renderer가 받아 shell과 전체 section 순서를 소유한다. optional arrays는 길이가 있을 때만 section을 추가하고 App page는 presentation markup에서 벗어난다.
- **inspection:** `src/app/resume/page.tsx, src/designs/design/resume-route.tsx`의 App branch 제거, renderer shell, optional experience/education/notes를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/resume/page.tsx, src/designs/design/resume-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:43fac33fdda3:END -->

### 6. `b11a4e4a3d72` — refactor(routes): Design 연락 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** Contact ownership transfer

#### Commit-specific investigation

- `b11a4e4a3d72^`와 `b11a4e4a3d72`를 비교하고 `src/app/contact/page.tsx, src/designs/design/contact-route.tsx`에서 **page availability/context resolution 이후 Design route entry 호출**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Contact ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:b11a4e4a3d72:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **page availability/context resolution 이후 Design route entry 호출**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** Contact App page는 enablement·content·template/debug를 준비한 뒤 renderer를 호출한다. availability/preferred links/notes와 shell composition은 Design module 책임이 된다.
- **inspection:** `src/app/contact/page.tsx, src/designs/design/contact-route.tsx`의 page availability/context resolution 이후 Design route entry 호출를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/contact/page.tsx, src/designs/design/contact-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:b11a4e4a3d72:END -->

### 7. `8a0bd21f7557` — refactor(routes): Design 여정 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** Journey ownership transfer

#### Commit-specific investigation

- `8a0bd21f7557^`와 `8a0bd21f7557`를 비교하고 `src/app/journey/page.tsx, src/designs/design/journey-route.tsx, src/lib/portfolio/view-models.ts`에서 **`createJourneyViewModel`, renderer route guard/shell/timeline/current, private `MilestoneCard`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Journey ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:8a0bd21f7557:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createJourneyViewModel`, renderer route guard/shell/timeline/current, private `MilestoneCard`**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** App page가 raw content가 아니라 journey-specific view model을 명시적으로 만들고 renderer에 전달한다. milestone project resolution은 view model에, visual order와 shell은 module에 위치한다.
- **inspection:** `src/app/journey/page.tsx, src/designs/design/journey-route.tsx, src/lib/portfolio/view-models.ts`의 `createJourneyViewModel`, renderer route guard/shell/timeline/current, private `MilestoneCard`를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/journey/page.tsx, src/designs/design/journey-route.tsx, src/lib/portfolio/view-models.ts` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:8a0bd21f7557:END -->

### 8. `2fee9efda711` — refactor(routes): Design 인터뷰 renderer로 위임

- **Importance:** B
- **Tags:** ROUTING, RENDERER, REFACTOR
- **Thread role:** Interview ownership transfer

#### Commit-specific investigation

- `2fee9efda711^`와 `2fee9efda711`를 비교하고 `src/app/interview-map/page.tsx, src/designs/design/interview-map-route.tsx, src/lib/portfolio/view-models.ts`에서 **`createInterviewMapViewModel`, renderer route guard/shell/gaps, private `TrackSection`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Interview ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:2fee9efda711:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createInterviewMapViewModel`, renderer route guard/shell/gaps, private `TrackSection`**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** App page가 interview-specific projection을 만들고 Design module이 intro/track table/gaps/shell을 소유한다. answer-project resolution은 view model에 남고 TrackSection은 module-private다.
- **inspection:** `src/app/interview-map/page.tsx, src/designs/design/interview-map-route.tsx, src/lib/portfolio/view-models.ts`의 `createInterviewMapViewModel`, renderer route guard/shell/gaps, private `TrackSection`를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/interview-map/page.tsx, src/designs/design/interview-map-route.tsx, src/lib/portfolio/view-models.ts` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:2fee9efda711:END -->

### 9. `05e1cebd0b70` — refactor(design): Design route dispatcher 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** Design의 여덟 독립 module을 하나의 public dispatcher로 통합

#### Commit-specific investigation

- `05e1cebd0b70^`와 `05e1cebd0b70`를 비교하고 `src/designs/design/index.tsx`에서 **`DesignRoute` switch와 home/projects/project-detail/about/resume/contact/journey/interview imports**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Design의 여덟 독립 module을 하나의 public dispatcher로 통합` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `각 module 내부 empty-state/visual correctness는 dispatcher가 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 1의 `380b2a025070`이 이 dispatcher를 registry의 Design loader로 호출한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:05e1cebd0b70:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`DesignRoute` switch와 home/projects/project-detail/about/resume/contact/journey/interview imports**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **구현 결정:** route별 module extraction이 끝난 뒤 `DesignRoute` 하나가 prepared discriminated props를 정확한 module로 전달한다. registry나 App Router는 개별 Design file 경로를 알 필요가 없고, route 추가 시 dispatcher union과 module 집합이 명시적으로 함께 바뀌어야 한다.
- **파일·symbol:** `src/designs/design/index.tsx`에서 `DesignRoute` switch와 home/projects/project-detail/about/resume/contact/journey/interview imports를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/designs/design/index.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **보장/비보장:** 이 SHA는 `Design의 여덟 독립 module을 하나의 public dispatcher로 통합` 경계를 고정합니다. 각 module 내부 empty-state/visual correctness는 dispatcher가 검증하지 않는다.
- **역사적 연결:** Thread 1의 `380b2a025070`이 이 dispatcher를 registry의 Design loader로 호출한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```tsx
// 05e1cebd0b70 · src/designs/design/index.tsx
export default function DesignRoute(props: DesignRouteProps) {
  switch (props.route) {
    case "home": return <HomeRoute {...props} />;
    // ... seven remaining routes
  }
}
```
<!-- LEARNER-ANSWER:commit:05e1cebd0b70:END -->

### 10. `15ab994dabfd` — refactor(classic-home): 홈 renderer를 독립 모듈로 이동

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic Home ownership transfer and shared-component cleanup

#### Commit-specific investigation

- `15ab994dabfd^`와 `15ab994dabfd`를 비교하고 `src/app/page.tsx, src/designs/classic/home-route.tsx, 삭제된 src/components/portfolio/home-*.tsx`에서 **App page의 Classic branch, route guard/shell, configured section order, Classic-only shared component 삭제/흡수**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic Home ownership transfer and shared-component cleanup` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:15ab994dabfd:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 Classic branch, route guard/shell, configured section order, Classic-only shared component 삭제/흡수**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** Classic 전용 Home section들이 공용 component 폴더에 흩어진 상태를 module 안으로 흡수하고 원래 files를 삭제한다. App page는 prepared props를 넘기고 Classic module이 shell·section order·visual helper를 소유한다.
- **inspection:** `src/app/page.tsx, src/designs/classic/home-route.tsx, 삭제된 src/components/portfolio/home-*.tsx`의 App page의 Classic branch, route guard/shell, configured section order, Classic-only shared component 삭제/흡수를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/page.tsx, src/designs/classic/home-route.tsx, 삭제된 src/components/portfolio/home-*.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:15ab994dabfd:END -->

### 11. `91e44a4de72c` — refactor(classic-projects): 프로젝트 목록 renderer를 이동

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic Projects ownership transfer

#### Commit-specific investigation

- `91e44a4de72c^`와 `91e44a4de72c`를 비교하고 `src/app/projects/page.tsx, src/designs/classic/projects-route.tsx`에서 **App page의 PageShell/metrics/groups 제거, renderer route guard/shell, prepared tuples**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic Projects ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:91e44a4de72c:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 PageShell/metrics/groups 제거, renderer route guard/shell, prepared tuples**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** App page에서 pageCopy·featured·grouped projects·metric 값을 해석하던 코드를 제거한다. Classic renderer가 prepared model과 shell을 소비하고 내부 view는 private가 된다.
- **inspection:** `src/app/projects/page.tsx, src/designs/classic/projects-route.tsx`의 App page의 PageShell/metrics/groups 제거, renderer route guard/shell, prepared tuples를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/projects/page.tsx, src/designs/classic/projects-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:91e44a4de72c:END -->

### 12. `7a65f0522061` — refactor(classic-project): 상세 renderer를 독립 모듈로 완성

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic Project detail ownership transfer

#### Commit-specific investigation

- `7a65f0522061^`와 `7a65f0522061`를 비교하고 `src/app/projects/[projectId]/page.tsx, src/designs/classic/project-detail-route.tsx`에서 **StructuredData는 page에 유지, renderer guard/shell/hero/body와 private section helpers**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic Project detail ownership transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:7a65f0522061:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **StructuredData는 page에 유지, renderer guard/shell/hero/body와 private section helpers**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** App page는 structured data와 framework 경계를 유지하고 Classic module이 route discriminator, shell, project hero/body를 소유한다. visual helper는 외부 API에서 제거된다.
- **inspection:** `src/app/projects/[projectId]/page.tsx, src/designs/classic/project-detail-route.tsx`의 StructuredData는 page에 유지, renderer guard/shell/hero/body와 private section helpers를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/projects/[projectId]/page.tsx, src/designs/classic/project-detail-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:7a65f0522061:END -->

### 13. `25fa6b575c31` — refactor(classic-about): 소개 renderer를 독립 모듈로 이동

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic About large presentation transfer

#### Commit-specific investigation

- `25fa6b575c31^`와 `25fa6b575c31`를 비교하고 `src/app/about/page.tsx, src/designs/classic/about-route.tsx`에서 **App page에서 제거된 300여 line identity/principles/journey/skills/experience/curation markup, route module shell**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic About large presentation transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:25fa6b575c31:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page에서 제거된 300여 line identity/principles/journey/skills/experience/curation markup, route module shell**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** About page에 직접 있던 Classic 전체 presentation과 curation helper를 dedicated module로 옮긴다. App page는 Classic/Design 구현을 선택하고 동일 prepared props를 전달하는 얇은 adapter가 된다.
- **inspection:** `src/app/about/page.tsx, src/designs/classic/about-route.tsx`의 App page에서 제거된 300여 line identity/principles/journey/skills/experience/curation markup, route module shell를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/about/page.tsx, src/designs/classic/about-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:25fa6b575c31:END -->

### 14. `88fb0a09db5e` — refactor(classic-resume): 이력 renderer를 독립 모듈로 이동

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic Resume large presentation transfer

#### Commit-specific investigation

- `88fb0a09db5e^`와 `88fb0a09db5e`를 비교하고 `src/app/resume/page.tsx, src/designs/classic/resume-route.tsx`에서 **App page에서 제거된 hero/summary/projects/training/experience/education/notes, module shell**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic Resume large presentation transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:88fb0a09db5e:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page에서 제거된 hero/summary/projects/training/experience/education/notes, module shell**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** Resume의 200여 line presentation을 Classic module로 이동해 prepared resume model과 shell 조립을 한 owner에 둔다. App page는 enablement/context/template selection만 남긴다.
- **inspection:** `src/app/resume/page.tsx, src/designs/classic/resume-route.tsx`의 App page에서 제거된 hero/summary/projects/training/experience/education/notes, module shell를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/resume/page.tsx, src/designs/classic/resume-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:88fb0a09db5e:END -->

### 15. `17e0a9ad0acb` — refactor(classic-contact): 연락 renderer를 독립 모듈로 이동

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic Contact transfer

#### Commit-specific investigation

- `17e0a9ad0acb^`와 `17e0a9ad0acb`를 비교하고 `src/app/contact/page.tsx, src/designs/classic/contact-route.tsx`에서 **App page의 hero/availability/preferred link/empty/notes 제거와 module route guard**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic Contact transfer` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:17e0a9ad0acb:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 hero/availability/preferred link/empty/notes 제거와 module route guard**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** Classic Contact markup과 empty-contact-links branch를 module로 옮긴다. page는 Design/Classic route component를 선택하고 prepared props만 전달한다.
- **inspection:** `src/app/contact/page.tsx, src/designs/classic/contact-route.tsx`의 App page의 hero/availability/preferred link/empty/notes 제거와 module route guard를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/contact/page.tsx, src/designs/classic/contact-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:17e0a9ad0acb:END -->

### 16. `c44f91b0d40d` — refactor(classic-journey): 여정 renderer를 독립 모듈로 이동

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic Journey transfer and shared prepared model

#### Commit-specific investigation

- `c44f91b0d40d^`와 `c44f91b0d40d`를 비교하고 `src/app/journey/page.tsx, src/designs/classic/journey-route.tsx`에서 **App page의 raw milestone resolution 제거, `createJourneyViewModel`, module `MilestoneCard`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic Journey transfer and shared prepared model` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:c44f91b0d40d:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 raw milestone resolution 제거, `createJourneyViewModel`, module `MilestoneCard`**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** Classic/Design 모두 같은 prepared journey model을 받도록 App page에서 projection을 한 번 만든다. Classic module은 shell·milestone/timeline/current presentation을 소유하고 raw project lookup을 하지 않는다.
- **inspection:** `src/app/journey/page.tsx, src/designs/classic/journey-route.tsx`의 App page의 raw milestone resolution 제거, `createJourneyViewModel`, module `MilestoneCard`를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/journey/page.tsx, src/designs/classic/journey-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:c44f91b0d40d:END -->

### 17. `a5d8f288baa2` — refactor(classic-interview): 인터뷰 renderer를 독립 모듈로 이동

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread role:** Classic Interview transfer and shared prepared model

#### Commit-specific investigation

- `a5d8f288baa2^`와 `a5d8f288baa2`를 비교하고 `src/app/interview-map/page.tsx, src/designs/classic/interview-map-route.tsx`에서 **App page의 project Map/table markup 제거, `createInterviewMapViewModel`, module `TrackSection`**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic Interview transfer and shared prepared model` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:a5d8f288baa2:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App page의 project Map/table markup 제거, `createInterviewMapViewModel`, module `TrackSection`**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **변경:** Classic/Design이 동일 interview projection을 소비하도록 하고 App page에서 200여 line table presentation과 project lookup을 제거한다. module은 shell과 semantic table/gaps를 소유한다.
- **inspection:** `src/app/interview-map/page.tsx, src/designs/classic/interview-map-route.tsx`의 App page의 project Map/table markup 제거, `createInterviewMapViewModel`, module `TrackSection`를 확인했습니다.
- **책임/경계:** 표현·조립 책임은 `src/app/interview-map/page.tsx, src/designs/classic/interview-map-route.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **비보장:** 이 commit은 해당 route/section의 정적 composition을 추가하지만 다른 route, 모든 빈 상태, 브라우저 layout을 자동으로 검증하지 않는다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.
<!-- LEARNER-ANSWER:commit:a5d8f288baa2:END -->

### 18. `6b193d084e69` — refactor(classic): Classic route dispatcher 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Thread role:** Classic의 여덟 독립 route module을 하나의 public entry로 통합

#### Commit-specific investigation

- `6b193d084e69^`와 `6b193d084e69`를 비교하고 `src/designs/classic/index.tsx`에서 **`ClassicRoute` exhaustive switch와 8 module imports**가 처음 생기거나 이동한 위치를 찾습니다.
- 직전 owner와 이 SHA 이후 owner를 구분하고, `Classic의 여덟 독립 route module을 하나의 public entry로 통합` 역할이 caller/callee·DOM·content-reference 경계에 어떤 변화를 주는지 적습니다.
- absence, unsupported route, missing reference, empty list, optional media/link 또는 responsive fallback 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `dispatcher는 module 반환값의 내용이나 runtime unknown string을 별도 검증하지 않는다.`라는 한계를 코드와 test 범위에서 확인합니다.
- 후속 관계: Thread 1의 `380b2a025070`이 Design과 함께 registry path로 통일한다.
- 같은 contract를 소비하는 다른 route/design과 비교하되, 이 SHA 이후 코드를 현재 commit의 구현으로 소급하지 않습니다.

#### Learning record

<!-- LEARNER-ANSWER:commit:6b193d084e69:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`ClassicRoute` exhaustive switch와 8 module imports**의 조립·조회·공개 책임이 App Router, raw content consumer 또는 여러 module에 분산돼 있었습니다.
- **구현 결정:** route-by-route extraction 후 `ClassicRoute`가 prepared discriminated props를 각 module로 전달한다. 외부 registry는 Classic 내부 file 구조를 알지 않고 single entry만 lazy-load할 수 있다.
- **파일·symbol:** `src/designs/classic/index.tsx`에서 `ClassicRoute` exhaustive switch와 8 module imports를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/designs/classic/index.tsx` 쪽으로 이동하고, 이전 caller는 framework/context 준비 또는 public entry 호출만 남습니다.
- **보장/비보장:** 이 SHA는 `Classic의 여덟 독립 route module을 하나의 public entry로 통합` 경계를 고정합니다. dispatcher는 module 반환값의 내용이나 runtime unknown string을 별도 검증하지 않는다.
- **역사적 연결:** Thread 1의 `380b2a025070`이 Design과 함께 registry path로 통일한다.
- **실행 증거:** 이 SHA의 repository test/runtime command는 실행하지 않았습니다. 기록은 branch의 exact commit diff와 해당 tree에 대한 정적 inspection 결과입니다.

```tsx
// 6b193d084e69 · src/designs/classic/index.tsx
export default function ClassicRoute(props: DesignRouteProps) {
  switch (props.route) {
    case "home": return <HomeRoute {...props} />;
    // ... seven remaining routes
  }
}
```
<!-- LEARNER-ANSWER:commit:6b193d084e69:END -->

## 5. Invariant evolution

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:invariant:BEGIN -->
최종 invariant는 App Router가 content/metadata/notFound/structured-data/view-model 생성만 소유하고, Design·Classic module이 route guard·shell·presentation·private helper를 소유하며, 외부는 각 `index.tsx` dispatcher 하나만 호출한다는 것입니다.

- 도입·확장·폐쇄의 순서는 commit map에 고정했습니다.
- B-level construction은 route/style surface를 단계적으로 넓히고, A/S-level commit은 owner·dispatch·검증 invariant를 바꿉니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:invariant:END -->

## 6. Failure → Fix → Test and ownership relations

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:relations:BEGIN -->
Design은 route별 dedicated module을 연결한 뒤 `05e1cebd0b70`에서 dispatcher를 만들고, Classic은 page/shared components의 대규모 presentation을 이동한 뒤 `6b193d084e69`에서 동일 경계를 만듭니다. 이후 Thread 1의 `380b2a025070`이 둘을 registry path로 통일합니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:relations:END -->

## 7. Final architecture or execution flow

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:flow:BEGIN -->
App Router page → route-specific view model/context → Design 또는 Classic dispatcher → dedicated route module → `createDesignShellProps` → design-owned body 순서입니다.

각 단계에서 optional/empty/missing reference가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 commit의 non-guarantee에 남겼습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:flow:END -->

## 8. Runtime and verification evidence

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:runtime:BEGIN -->
- **실행한 repository test/build:** 없음.
- **정적 확인:** 지정 branch의 commit classification, commit bodies, exact commit diff와 historical file 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 container에서 직접 clone 시 DNS가 `github.com`을 해석하지 못해 historical worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next build 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** code/test implementation의 존재와 범위는 inspection으로 확인했고, runtime pass/fail은 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:runtime:END -->

## 9. Learning-completion checks

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·subject·importance·tags를 commit map과 commit section에서 동일하게 유지했습니다.
- [x] 각 SHA에 concrete file과 symbol/selector/route focus를 기록했습니다.
- [x] 이전 상태, owner, absence/fallback, guarantee/non-guarantee, 후속 관계를 채웠습니다.
- [x] S/A/B depth를 구분했습니다.
- [x] 실행하지 않은 test를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:checks:END -->
===== END FILE: 05-design-and-classic-renderer-extraction.md =====

===== BEGIN FILE: README.md =====
# Full-site visual systems

> Repository: `seungwoo7050/42-archive`
>
> Branch: `web/portfolio`
>
> Category: `05-full-site-visual-systems`
>
> Audited branch span: `cce7dd020563` → `aff0acdd4cf9`

## Category boundary

이 category는 다섯 full-site visual system의 독립 renderer 구축과 그 renderer들을 공통 route/shell/input/test 계약에 연결하는 history를 다룹니다.

- 포함: cross-design registry·delegation·prepared renderer input·token/shell boundary·activation·regression, Editorial/Brutalist/Cinematic full construction, Design/Classic route-module extraction.
- 제외: content schema 자체의 최초 설계, 일반 App Router lifecycle, shared interaction primitive의 독립 history, deployment/CI, formatting-only C commit.
- 예외적으로 `8a48460df4c3`와 `aef265b9bd01`은 content view-model construction이 아니라 **모든 renderer에 적용되는 ownership transfer**이므로 Thread 1에 포함했습니다.

## Phase 1 audit result

- Thread 수는 5개로 유지했습니다. 각 visual system의 독립 engineering story가 분명해 split/merge가 필요하지 않았습니다.
- 기존 scaffold의 generic investigation prompt를 exact file·function·selector group·failure/empty/reference/test task로 교체했습니다.
- Editorial과 Brutalist의 누락된 desktop→tablet→mobile construction 및 route composition commit을 복구했습니다.
- Cinematic의 `1f4c35853502`, `52f13fcc5a12`처럼 detail/resume/journey/interview 지면을 실제로 연결하는 중간 commit을 복구했습니다.
- `dc2cf72a768d`, `8a48460df4c3`, `aef265b9bd01`, `f8b0ab7b08aa`, `380b2a025070`처럼 architecture closure에 필요한 S/A commit을 cross-design Thread에 추가했습니다.
- activation commit `c6acfe562694`, `dd71d28143a8`, `b8de57f130eb`은 renderer 내부 construction Thread에서 제거하고 공통 registry ownership을 다루는 Thread 1로 이동했습니다.
- formatting-only C commit `ea073db5f785`, `4e54f8fef892`, `6fe74d2dd94d`, `87ac2ce7b285`은 material history를 바꾸지 않아 포함하지 않았습니다.
- frozen commit 수: **158개 고유 SHA** (1: 16, 2: 55, 3: 50, 4: 19, 5: 18).

## Frozen thread order

1. [Cross-design token, shell, and regression contracts](./01-cross-design-token-shell-and-regression-contracts.md)
2. [Editorial design system construction](./02-editorial-design-system-construction.md)
3. [Brutalist design system construction](./03-brutalist-design-system-construction.md)
4. [Cinematic design system construction](./04-cinematic-design-system-construction.md)
5. [Design and Classic renderer extraction](./05-design-and-classic-renderer-extraction.md)

## Study discipline

- 각 commit은 exact SHA의 parent diff와 resulting tree를 기준으로 읽습니다.
- final HEAD의 helper·file layout·test를 earlier SHA에 소급하지 않습니다.
- static inspection과 실제 command execution을 분리합니다.
- route별 missing/empty/unsupported/reference failure와 **보장하지 않는 것**을 반드시 남깁니다.
- S/A/B depth를 동일하게 만들지 않습니다.

## Phase 2 completion and validation

<!-- LEARNER-ANSWER:readme:completion:BEGIN -->
- scaffold와 completed는 각각 `README.md` 포함 6개 파일이며 상대 경로가 정확히 일치합니다.
- frozen scaffold tree SHA-256: `857ec3ae46a37ca14339974872c258bfefb9182deddc8cf23f340b85e730aacf`.
- Phase 2 후 scaffold manifest를 다시 계산해 동결 hash가 변하지 않았음을 확인했습니다.
- 158개 SHA는 중복이 없고, branch의 `commit/commit-importance.md` 선형 history에 존재하며 선택 commit의 metadata와 대조했습니다.
- completed의 answer marker 바깥 text는 scaffold와 byte-equivalent skeleton으로 검증했습니다.
- 모든 learner-facing marker는 비어 있지 않으며 S/A/B depth가 다르게 생성됐습니다.
- repository test/build/Playwright는 실행하지 않았습니다. container DNS 제한으로 checkout/worktree를 만들 수 없었고, 통과 결과를 작성하지 않았습니다.
- local structural validation: `PASS`; packaged file count `12`; extra file `0`.

<!-- LEARNER-ANSWER:readme:completion:END -->
===== END FILE: README.md =====

