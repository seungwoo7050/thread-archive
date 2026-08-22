===== BEGIN FILE: 01-dual-home-composition-and-content-driven-sections.md =====
# Development Thread: Dual home composition and content-driven sections

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: 두 홈 route의 hero 구성, 대표 프로젝트·지표·기술·여정·연락 섹션, 섹션 표시와 순서 결정.
- 제외: `view` query 해석과 route dispatcher 자체는 category 02, terminal 타이핑 상태 기계는 category 03, full-site renderer 추출은 category 05, route view-model 소유권은 category 09가 담당합니다.
- `cdb68fdf59f9`는 Classic 홈의 최초 composition이 없던 draft 공백을 메우므로 추가했습니다. terminal interaction/style 후속 커밋은 이 Thread의 기능 경계를 넘으므로 추가하지 않았습니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 11
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

Design과 Classic 홈이 같은 콘텐츠 aggregate를 서로 다른 hero와 섹션 구성으로 표현하고, Design 홈의 섹션 순서가 코드 순서가 아니라 콘텐츠 배열을 따르게 되는 과정을 복원합니다.

### Fixed invariants

- Design과 Classic은 같은 `PortfolioContent`의 profile, links, projects, skills, journey, contact를 사용하지만 표현 구조는 독립적으로 선택합니다.
- 공용 섹션은 각 template의 `sections` 구성에 포함될 때만 렌더링됩니다.
- Design 홈의 최종 섹션 순서는 `presentation.home.design.sections` 배열의 순서가 결정합니다.
- 데이터가 비어 있거나 참조가 없으면 해당 카드·링크·목록이 자연스럽게 비며, 이 Thread 시점에는 별도의 오류 복구 UI를 보장하지 않습니다.

## 2. Core engineering questions

1. Design과 Classic hero는 동일한 원본 데이터에서 어떤 서로 다른 파생값을 계산하는가?
2. 각 공용 섹션은 어느 파일이 데이터를 해석하고 어느 route가 표시 여부를 결정하는가?
3. `includes()` 기반 고정 JSX 순서와 `sections.map()` 기반 콘텐츠 순서의 차이는 무엇인가?
4. featured project, skill ID, preferred link가 없을 때 실제 DOM은 어떻게 달라지는가?
5. 후대 `HomeViewModel` 도입은 이 Thread의 기능 결정을 어떻게 보존하되 어디에서 소유하게 되는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `3475ba3efdb2` | feat(home): 디자인 홈 소개 영역 구성 | B | RENDERER | Design 홈의 profile 기반 hero와 shell을 최초 구성합니다. |
| 2 | `b027f42669aa` | feat(home): 대표 프로젝트 쇼케이스 추가 | B | RENDERER | Design hero에 lead/supporting 프로젝트와 작업 지표를 추가합니다. |
| 3 | `cdb68fdf59f9` | feat(home): 클래식 홈 히어로 구성 | B | RENDERER | 같은 콘텐츠 aggregate 위에 Classic 홈 hero를 추가해 dual-home composition의 두 번째 축을 만듭니다. |
| 4 | `07dd465dbe20` | feat(home): 디자인 대표 프로젝트 섹션 추가 | B | RENDERER | Design 홈에 구성 가능한 featured-project section을 추가합니다. |
| 5 | `33bec9cf6325` | feat(home): 클래식 대표 프로젝트 섹션 추가 | B | RENDERER | Classic 홈에도 같은 featured source를 Classic 표현으로 추가합니다. |
| 6 | `afaf5a393518` | feat(home): 작업 지표 섹션 추가 | B | RENDERER | route-local 지표 계산을 공용 `WorkMapSection`으로 추출하고 두 홈에 연결합니다. |
| 7 | `df26861cdc24` | feat(home): 기술 집중 영역 추가 | B | RENDERER | skills의 focus areas를 양쪽 홈의 공용 섹션으로 노출합니다. |
| 8 | `e3aebedaa46b` | feat(home): 선택 기술 스택 영역 추가 | B | RENDERER | skill group ID를 기준으로 홈에 노출할 canonical tech stack을 파생합니다. |
| 9 | `9a107fe185cf` | feat(home): 공용 여정 섹션 추가 | B | RENDERER | 전체 journey 목록을 홈 preview로 재사용합니다. |
| 10 | `bb8ccf341f39` | feat(home): 연락 미리보기 추가 | B | RENDERER | preferred contact links와 availability를 홈 끝의 공용 CTA로 추가합니다. |
| 11 | `92799407457e` | refactor(design-home): 홈 섹션 순서를 콘텐츠로 연결 | B | CONTENT, RENDERER, REFACTOR | Design 홈의 활성 섹션뿐 아니라 DOM 순서까지 콘텐츠 배열이 결정하게 합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `3475ba3efdb2` — feat(home): 디자인 홈 소개 영역 구성

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Design 홈의 profile 기반 hero와 shell을 최초 구성합니다.

#### Exact inspection targets

- `src/designs/design/home-route.tsx` — `DesignHomeRoute`, `HeroSection`
- `PageShell`, `ContentLinkView`, `ProfilePhoto` 호출 경로
- `content.profile`, `content.links`, `homeTemplate="design"`

#### Commit-specific investigation tasks

1. 부모 상태와 비교해 `DesignHomeRoute`/`HeroSection`가 처음 생기는 파일 경계를 확인하고 `PageShell` 호출까지 추적합니다.
2. `profile.photo`와 github/resume/website 링크 필터가 각각 어떤 조건부 DOM을 만드는지 확인합니다.
3. 이 SHA에는 Classic home이나 template dispatcher가 없는지 파일 트리와 imports로 검증합니다.

#### Learner evidence record

- **Previous state:** 직전에는 콘텐츠를 소비하는 Design 홈 route가 없어서 profile과 링크가 실제 첫 화면에 조립되지 않았습니다.
- **Implementation decision and path:** `DesignHomeRoute`가 `PageShell` 안에 hero를 만들고 profile name/role/headline/summary, optional photo, github/resume/website 유형 링크를 출력합니다.
- **Ownership and data lifetime:** route가 `PortfolioContent` 전체를 받아 hero가 직접 profile과 link 유형을 해석합니다. `PageShell`은 공통 chrome을 소유합니다.
- **Failure, absence, and non-guarantee:** photo가 없으면 `ProfilePhoto`를 생략합니다. 링크가 없으면 CTA 영역은 빈 목록이 되며 별도 empty state는 없습니다.
- **Resulting guarantee:** 콘텐츠 기반 Design hero가 존재하고 template ID가 `design`으로 고정된다는 것만 보장합니다. Classic 구성이나 route dispatch는 아직 없습니다.
- **Relationship to later work:** `b027...`이 같은 hero에 대표 프로젝트와 지표를 더하고, 후대 category 02의 dispatcher가 이 route를 선택 가능하게 만듭니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `3475ba3efdb2` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `b027f42669aa` — feat(home): 대표 프로젝트 쇼케이스 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Design hero에 lead/supporting 프로젝트와 작업 지표를 추가합니다.

#### Exact inspection targets

- `src/designs/design/home-route.tsx` — `DesignHomeRoute`, `HeroSection`, 당시의 `getWorkMapStats`
- `getFeaturedProjects`, `getTemplateHref`, `ProjectScreenshot`
- `projects[0]`, `projects.slice(1, 3)`의 경계

#### Commit-specific investigation tasks

1. `getFeaturedProjects(content)` 결과가 lead 1개와 supporting 최대 2개로 분할되는 slice/index 규칙을 기록합니다.
2. 당시 route-local `getWorkMapStats`의 curriculum/product/reliability 산식과 특정 ID/경로 의존을 확인합니다.
3. lead project가 없을 때 showcase wrapper 전체가 생략되는지 JSX branch를 확인합니다.

#### Learner evidence record

- **Previous state:** Design hero는 profile과 링크만 보여 주어 대표 작업과 archive 규모를 증거로 제시하지 못했습니다.
- **Implementation decision and path:** `getFeaturedProjects(content)` 결과를 hero로 전달하고 첫 항목을 lead, 다음 두 항목을 supporting으로 사용합니다. 당시 로컬 `getWorkMapStats`가 curriculum/product/reliability 수를 계산합니다.
- **Ownership and data lifetime:** featured 선별은 selector가, lead/supporting 위치와 지표 표시 형식은 Design hero가 소유합니다.
- **Failure, absence, and non-guarantee:** lead가 없으면 showcase 전체를 렌더링하지 않습니다. supporting이 부족하면 존재하는 항목만 출력됩니다. 특정 project ID와 screenshot 경로에 의존한 지표는 아직 route-local 가정입니다.
- **Resulting guarantee:** 대표 프로젝트 최대 세 개와 세 지표를 hero에서 보여 주지만 데이터 정확성이나 selector 회귀 테스트는 보장하지 않습니다.
- **Relationship to later work:** `07dd...`이 hero 외부의 독립 featured section을 추가하고 `afaf...`이 지표 계산을 공용 컴포넌트로 이동합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `b027f42669aa` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `cdb68fdf59f9` — feat(home): 클래식 홈 히어로 구성

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 같은 콘텐츠 aggregate 위에 Classic 홈 hero를 추가해 dual-home composition의 두 번째 축을 만듭니다.

#### Exact inspection targets

- `src/designs/classic/home-route.tsx` — `ClassicHomeRoute`, `ClassicHeroSection`
- `AnimatedTerminal`에 전달되는 `profile`, project/stack count, terminal copy
- github/resume/website 링크 필터와 optional photo

#### Commit-specific investigation tasks

1. `ClassicHomeRoute`에서 `ClassicHeroSection`과 `AnimatedTerminal`로 이어지는 caller→callee prop 흐름을 추적합니다.
2. profile/project/stack count 중 route가 계산해 넘기는 값과 terminal이 소유하는 표현을 구분합니다.
3. 이 SHA가 terminal state machine을 새로 구현하는지, 기존 component를 composition하는지만 diff로 판별합니다.

#### Learner evidence record

- **Previous state:** 직전에는 Design 홈만 실제 route component로 존재해 'dual home'이라는 기능 서사를 완성할 Classic composition이 없었습니다.
- **Implementation decision and path:** `ClassicHomeRoute`가 `homeTemplate="classic"`인 `PageShell`을 만들고 profile hero와 `AnimatedTerminal`을 나란히 배치합니다.
- **Ownership and data lifetime:** Classic hero가 링크 유형 필터와 profile 표현을 소유하고, terminal 컴포넌트에는 이미 계산한 project/stack count와 presentation copy를 전달합니다.
- **Failure, absence, and non-guarantee:** photo와 링크는 optional입니다. 이 커밋은 terminal의 시간 기반 state machine이나 reduced-motion fallback을 새로 구현하지 않습니다.
- **Resulting guarantee:** 같은 `PortfolioContent`로 Design과 구별되는 Classic hero를 구성할 수 있음을 보장합니다. 사용자가 어느 template를 선택하는지는 보장하지 않습니다.
- **Relationship to later work:** category 02의 `f423...`이 query 기반 선택을 연결하고, 이 Thread의 `33bec...`부터 Classic에도 콘텐츠 섹션이 누적됩니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `cdb68fdf59f9` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `07dd465dbe20` — feat(home): 디자인 대표 프로젝트 섹션 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Design 홈에 구성 가능한 featured-project section을 추가합니다.

#### Exact inspection targets

- `src/designs/design/home-route.tsx` — `FeaturedProjectsSection`
- `presentation.home.design.sections.includes("featured")`
- `ProjectCard` variant와 `projects.slice()` 경계

#### Commit-specific investigation tasks

1. `sections.includes("featured")`가 section 전체를 가드하는 위치와 `FeaturedProjectsSection`의 1+2 slicing을 확인합니다.
2. hero showcase와 새 featured body section이 같은 selector 결과를 어떻게 재사용하는지 비교합니다.
3. featured 목록이 비어 있을 때 heading/card DOM이 어떻게 남거나 사라지는지 exact JSX에서 기록합니다.

#### Learner evidence record

- **Previous state:** 대표 프로젝트는 hero showcase에만 묶여 있어 홈 본문에서 독립적인 증거 섹션으로 조절할 수 없었습니다.
- **Implementation decision and path:** `featured` ID가 Design section 목록에 있을 때만 `FeaturedProjectsSection`을 렌더링하고 lead 한 개와 supporting 두 개를 `ProjectCard`로 표현합니다.
- **Ownership and data lifetime:** presentation 배열이 섹션의 활성 여부를 소유하지만 이 시점의 JSX 배치 순서 자체는 route 코드가 소유합니다.
- **Failure, absence, and non-guarantee:** featured project가 비어 있으면 목록이 비며 별도 메시지는 없습니다. 배열에 `featured`가 없으면 섹션 전체가 사라집니다.
- **Resulting guarantee:** 콘텐츠 설정으로 Design featured section을 켜고 끌 수 있음을 보장합니다. 설정 배열의 위치가 실제 DOM 순서를 바꾸지는 않습니다.
- **Relationship to later work:** `33bec...`이 Classic 대응 섹션을 만들고 `927...`이 Design의 DOM 순서까지 콘텐츠 배열에 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `07dd465dbe20` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `33bec9cf6325` — feat(home): 클래식 대표 프로젝트 섹션 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Classic 홈에도 같은 featured source를 Classic 표현으로 추가합니다.

#### Exact inspection targets

- `src/designs/classic/home-route.tsx` — `ClassicHomeRoute`의 featured section
- `presentation.home.classic.sections.includes("featured")`
- `getFeaturedProjects` 결과와 `ProjectCard` 소비 방식

#### Commit-specific investigation tasks

1. Classic route가 featured source를 어떤 component/variant로 표현하고 Design DOM과 어디서 갈라지는지 확인합니다.
2. Classic `sections` 배열의 `featured` 포함 여부가 section 존재를 어떻게 결정하는지 추적합니다.
3. 같은 source를 공유해도 template별 layout을 독립 소유한다는 근거를 파일·symbol 단위로 기록합니다.

#### Learner evidence record

- **Previous state:** Classic 홈은 hero와 terminal만 있어 대표 프로젝트를 본문 증거로 보여 주지 못했습니다.
- **Implementation decision and path:** Classic section 목록에 `featured`가 있을 때 featured projects를 카드 목록으로 렌더링합니다.
- **Ownership and data lifetime:** 선별 데이터는 공용 selector가 제공하고, 표시 밀도와 배치는 Classic route가 소유합니다.
- **Failure, absence, and non-guarantee:** 설정에서 빠지면 섹션을 생략하고, 프로젝트가 없으면 빈 목록이 됩니다. Design과 동일한 DOM을 보장하지 않습니다.
- **Resulting guarantee:** 두 template가 같은 featured source를 각자 표현할 수 있음을 보장합니다.
- **Relationship to later work:** `afaf...`부터 work map 등 공용 섹션이 두 route에 동시에 연결됩니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `33bec9cf6325` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.6. `afaf5a393518` — feat(home): 작업 지표 섹션 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** route-local 지표 계산을 공용 `WorkMapSection`으로 추출하고 두 홈에 연결합니다.

#### Exact inspection targets

- `src/components/portfolio/work-map-section.tsx` — `getWorkMapStats`, `WorkMapSection`, `ArchiveStat`
- `src/designs/design/home-route.tsx`, `src/designs/classic/home-route.tsx`의 `workMap` guard
- curriculum/project/reliability count 산식

#### Commit-specific investigation tasks

1. Design hero의 로컬 지표 helper가 `work-map-section.tsx`로 이동한 전후를 부모 diff와 비교합니다.
2. `getWorkMapStats`의 세 산식과 presentation card `countKey` lookup을 추적합니다.
3. Design/Classic 각각의 `workMap` guard와 reliability project 미존재 시 0 처리 branch를 확인합니다.

#### Learner evidence record

- **Previous state:** Design hero 안에만 지표 계산이 있었고 Classic에는 동일한 작업 지도가 없었습니다.
- **Implementation decision and path:** 새 공용 컴포넌트가 세 지표를 계산해 presentation의 card `countKey`에 대응시키며, 두 route는 각자의 `sections`에 `workMap`이 있을 때 이를 호출합니다.
- **Ownership and data lifetime:** 지표 산식과 카드 반복은 공용 컴포넌트가, 표시 여부는 각 template section 목록이 소유합니다.
- **Failure, absence, and non-guarantee:** reliability project를 찾지 못하면 0을 사용합니다. count key가 타입 계약 밖이면 처리하지 않으며, 산식의 의미 정확성은 이 커밋만으로 검증되지 않습니다.
- **Resulting guarantee:** 두 홈이 동일한 산식과 copy source를 공유한다는 것을 보장합니다.
- **Relationship to later work:** `383...`(상세 Thread 관련 후속)가 공용 `getProjectMetricValue`로 산식을 중앙화하지만 그 selector 정책은 category 01에 속합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `afaf5a393518` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.7. `df26861cdc24` — feat(home): 기술 집중 영역 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** skills의 focus areas를 양쪽 홈의 공용 섹션으로 노출합니다.

#### Exact inspection targets

- `src/components/portfolio/technical-focus-section.tsx` — `TechnicalFocusSection`
- `content.skills.focusAreas.map()`
- 두 home route의 `technicalFocus` section guard

#### Commit-specific investigation tasks

1. `TechnicalFocusSection`이 `presentation.home.shared.technicalFocus`와 `skills.focusAreas`를 어떻게 결합하는지 확인합니다.
2. 두 home route의 `technicalFocus` section guard가 동일 component를 호출하는지 기록합니다.
3. focusAreas가 빈 경우 heading과 card grid의 실제 DOM 상태를 exact SHA에서 판별합니다.

#### Learner evidence record

- **Previous state:** 홈에는 프로젝트 지표는 있었지만 기술적 집중 분야를 직접 설명하는 영역이 없었습니다.
- **Implementation decision and path:** presentation copy와 `skills.focusAreas`를 결합한 카드 목록을 만들고 두 template의 section 설정에 따라 연결합니다.
- **Ownership and data lifetime:** focus area 데이터는 skills content가 소유하고 카드 반복·지연 표현은 공용 컴포넌트가 소유합니다.
- **Failure, absence, and non-guarantee:** focusAreas가 비면 heading 아래 카드가 없으며 별도 empty state는 없습니다. 중복 title은 React key 충돌 가능성을 이 커밋이 방지하지 않습니다.
- **Resulting guarantee:** 양쪽 홈에서 같은 집중 분야 데이터를 일관되게 표시할 수 있음을 보장합니다.
- **Relationship to later work:** `e3aebe...`이 같은 skills graph를 사용해 canonical tech stack을 파생합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `df26861cdc24` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.8. `e3aebedaa46b` — feat(home): 선택 기술 스택 영역 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** skill group ID를 기준으로 홈에 노출할 canonical tech stack을 파생합니다.

#### Exact inspection targets

- `src/components/portfolio/selected-stack-section.tsx` — `SelectedStackSection`
- `Set(content.skills.groups.flatMap(...))`와 `content.techStack.filter(...)`
- `TechMarquee`, `StackList`, 두 route의 `stack` guard

#### Commit-specific investigation tasks

1. skills group item IDs로 `Set`을 만들고 canonical `techStack`을 filter하는 교집합 규칙을 재현합니다.
2. `TechMarquee`에 전달되는 항목과 `StackList`에 전달되는 raw group IDs의 차이를 확인합니다.
3. group에는 있지만 techStack에는 없는 ID가 어느 표현에서 사라지는지 기록합니다.

#### Learner evidence record

- **Previous state:** focus area는 설명 텍스트만 제공했고, 그룹에 속한 기술 ID와 canonical 기술 메타데이터를 함께 보여 주지 못했습니다.
- **Implementation decision and path:** group item ID 집합을 만들고 `techStack`에서 일치하는 항목만 골라 marquee에 전달하며, 각 그룹은 `StackList`로 그대로 렌더링합니다.
- **Ownership and data lifetime:** skills groups가 선택 집합과 그룹 순서를 소유하고, techStack가 icon/label 메타데이터를 소유합니다.
- **Failure, absence, and non-guarantee:** group에 존재하지만 techStack에 없는 ID는 marquee에서 빠집니다. `StackList`가 그 ID를 어떻게 표시하는지는 별도 공용 primitive 책임입니다.
- **Resulting guarantee:** 홈의 selected stack가 skill-group membership에서 파생된다는 것을 보장합니다.
- **Relationship to later work:** `9a107...`은 다른 aggregate인 journey를 공용 섹션으로 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `e3aebedaa46b` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.9. `9a107fe185cf` — feat(home): 공용 여정 섹션 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 전체 journey 목록을 홈 preview로 재사용합니다.

#### Exact inspection targets

- `src/components/portfolio/home-journey-section.tsx` — `HomeJourneySection`
- `JourneyList` props: `items`, `homeTemplate`, `variant="paired-centerline"`
- 두 route의 `journey` guard

#### Commit-specific investigation tasks

1. `HomeJourneySection`이 `content.journey`를 `JourneyList`에 전달하는 prop 흐름을 추적합니다.
2. `journeyNarrative.milestones`가 이 SHA의 home preview source가 아닌지 확인합니다.
3. template/debug state가 project case-study links까지 전달될 수 있는 props를 기록합니다.

#### Learner evidence record

- **Previous state:** journey 콘텐츠는 존재했지만 홈에서 현재 작업의 시간적 맥락을 보여 주는 섹션이 없었습니다.
- **Implementation decision and path:** 공용 heading과 `JourneyList`를 결합한 섹션을 만들고 두 template에 조건부 연결합니다.
- **Ownership and data lifetime:** journey 항목의 원본과 순서는 `content.journey`가, 링크 상태 보존과 layout은 `JourneyList`가 소유합니다.
- **Failure, absence, and non-guarantee:** journey가 비면 timeline 내용이 없고 별도 메시지는 없습니다. 이 커밋은 `/journey`의 narrative milestone을 사용하지 않습니다.
- **Resulting guarantee:** 홈 preview가 전체 journey 원본을 재사용하고 현재 template/debug 상태를 project 링크에 전달할 수 있음을 보장합니다.
- **Relationship to later work:** `ba813...` 등 Journey route Thread는 별도의 narrative milestone과 anchor-project 의미를 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `9a107fe185cf` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.10. `bb8ccf341f39` — feat(home): 연락 미리보기 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** preferred contact links와 availability를 홈 끝의 공용 CTA로 추가합니다.

#### Exact inspection targets

- `src/components/portfolio/home-contact-preview.tsx` — `HomeContactPreview`
- `getPreferredContactLinks(content)`
- 두 route의 `contact` guard와 `ContentLinkView`

#### Commit-specific investigation tasks

1. `HomeContactPreview`가 `getPreferredContactLinks` 결과와 contact availability를 결합하는 위치를 확인합니다.
2. preferred links가 0개일 때 별도 empty-state branch가 아직 없는지 JSX를 확인합니다.
3. Design/Classic route의 `contact` section guard와 공용 CTA component 호출을 비교합니다.

#### Learner evidence record

- **Previous state:** 홈에서 여정까지 본 뒤 실제 연락 수단으로 이동할 연결이 없었습니다.
- **Implementation decision and path:** contact availability와 presentation copy를 보여 주고 selector가 반환한 preferred links를 CTA로 렌더링합니다.
- **Ownership and data lifetime:** 선호 순서·활성 링크 결정은 selector가, 홈의 CTA 표현은 공용 preview가 소유합니다.
- **Failure, absence, and non-guarantee:** preferred links가 비면 CTA 목록만 비고 별도 empty state는 없습니다. Contact route의 fallback 규칙은 이 커밋에 없습니다.
- **Resulting guarantee:** 두 홈이 같은 선호 연락 결과를 사용한다는 것을 보장합니다.
- **Relationship to later work:** Contact Thread의 `119...`와 `9e99...`가 placement vocabulary와 explicit empty state를 발전시킵니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `bb8ccf341f39` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.11. `92799407457e` — refactor(design-home): 홈 섹션 순서를 콘텐츠로 연결

- **Importance:** B
- **Tags:** CONTENT, RENDERER, REFACTOR
- **Source-defined role:** Design 홈의 활성 섹션뿐 아니라 DOM 순서까지 콘텐츠 배열이 결정하게 합니다.

#### Exact inspection targets

- `src/designs/design/home-route.tsx` — `DesignHomeRoute`, `HomeSection`
- `HomeSectionId`, `sections.map((sectionId) => ...)`
- 각 ID별 component dispatch와 unknown ID의 `null`

#### Commit-specific investigation tasks

1. 이전 `includes()` 나열과 새 `sections.map()`/`HomeSection` dispatch를 부모 diff로 직접 비교합니다.
2. 각 `HomeSectionId`가 어떤 component로 매핑되고 알 수 없는 ID가 어떤 값을 반환하는지 확인합니다.
3. 배열 순서·중복 ID·React key가 DOM order/duplication에 미치는 비보장 범위를 기록합니다.

#### Learner evidence record

- **Previous state:** 각 section은 `includes()`로 켜고 껐지만 JSX에 적힌 고정 순서로만 배치되어 presentation 배열의 순서 의미가 무시됐습니다.
- **Implementation decision and path:** `sections.map()`이 배열 순서대로 `HomeSection`을 호출하고, `HomeSection`이 featured/workMap/technicalFocus/stack/journey/contact를 분기합니다.
- **Ownership and data lifetime:** presentation content가 활성 집합과 순서를 함께 소유하고 route는 ID-to-component dispatch만 소유합니다.
- **Failure, absence, and non-guarantee:** 알 수 없는 ID는 `null`입니다. 중복 ID가 있으면 같은 key와 중복 section 문제가 생길 수 있으며 이 커밋은 schema validation을 설명하지 않습니다.
- **Resulting guarantee:** Design 홈의 본문 순서가 콘텐츠 편집으로 변경된다는 것을 보장합니다. Classic의 순서 정책까지 변경하지는 않습니다.
- **Relationship to later work:** 이 SHA는 이미 `HomeViewModel`을 소비하지만, 그 파생 데이터 소유권·renderer contract는 category 09에서 별도 복원합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `92799407457e` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| 소개 구성 | `3475...`가 Design hero를, `cdb68...`가 Classic hero를 만들어 같은 aggregate의 두 표현을 성립시켰습니다. |
| 증거 확장 | `b027...`부터 featured project, work map, skills, journey, contact가 단계적으로 추가됐습니다. |
| 공유 경계 | `afaf...` 이후 공용 섹션은 데이터 해석을 공유하고 두 route는 section ID로 포함 여부를 정했습니다. |
| 순서 소유권 | `927...`에서 Design의 section 배열이 활성 여부뿐 아니라 DOM 순서까지 소유하게 됐습니다. |

## 7. Failure → Fix → Test and later relationships

- 직접적인 fix/test commit은 이 frozen map에 없습니다. route projection·renderer matrix·회귀 테스트는 category 09/07의 경계입니다.
- `927...`은 기능 오류 수정이라기보다 이전의 '배열은 활성 집합만 표현한다'는 제한을 제거한 ownership refactor입니다.
- Terminal state와 reduced-motion 검증은 category 03의 별도 Thread가 담당하므로 이 문서가 실행 증거를 대신 주장하지 않습니다.

## 8. Ownership, state, and responsibility changes

- 콘텐츠 JSON/aggregate: profile, presentation section IDs/copy, projects, skills, journey, contact 원본.
- 공용 selectors/components: featured/preferred 선별, work-map 산식, selected stack 교집합, journey/contact 표현.
- Design/Classic route: hero 표현과 각 template의 section 포함/배치.
- 후대 route view model: category 09에서 원본 aggregate를 renderer-safe projection으로 변환합니다.

## 9. Final thread state

Design과 Classic은 같은 콘텐츠를 다른 hero로 표현하고 공용 evidence sections를 공유합니다. Design 본문은 content-owned section order를 따르며, optional 데이터는 해당 부분을 생략합니다.

## 10. Final architecture and execution flow

콘텐츠 aggregate 로드 → 선택된 template route에 전달 → hero가 template별 profile/project 파생값 소비 → section ID 목록 순회 또는 guard → 공용 섹션이 projects/skills/journey/contact를 해석 → 내부 링크가 template/debug 상태를 보존해 다음 route로 이동.

## 11. Minimal historical code evidence

`92799407457e`, `src/designs/design/home-route.tsx`, `DesignHomeRoute`/`HomeSection`:
```tsx
const sections = content.presentation.home.design.sections;
{sections.map((sectionId) => (
  <HomeSection key={sectionId} sectionId={sectionId} {...sectionProps} />
))}
```
이 발췌는 활성 여부와 순서를 같은 콘텐츠 배열이 소유하게 된 결정을 보여 줍니다.

## 12. Learning-completion checks

- [x] Frozen commit map 11개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 01-dual-home-composition-and-content-driven-sections.md =====

===== BEGIN FILE: 02-project-index-grouping-and-dual-presentation.md =====
# Development Thread: Project index grouping and dual presentation

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: group ordering, featured/track partition, Design/Classic 프로젝트 목록 표현, `/projects` 통합.
- 제외: query-state 해석의 일반 계약은 category 02, renderer 추출과 dispatcher 아키텍처는 category 05/09, project card primitive는 category 03이 담당합니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 8
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

프로젝트 인덱스가 featured와 archive 항목을 분리하고, 콘텐츠가 정한 group 순서를 보존하면서 Design과 Classic에 서로 다른 정보 밀도로 전달되는 과정을 복원합니다.

### Fixed invariants

- configured category는 presentation의 group 순서를 따르고 unknown category는 뒤에서 알파벳순입니다.
- featured project와 non-featured track project는 서로 다른 presentation role을 가집니다.
- Design과 Classic은 같은 partitions/groups/metrics를 받되 다른 lead·grid·index 표현을 사용합니다.
- source-only/curriculum count는 프로젝트 원본에서 파생되며 route 통합 시점에는 route가 계산 책임을 가집니다.

## 2. Core engineering questions

1. `groupProjects`가 configured/unknown group의 상대 순서를 어떻게 결정하는가?
2. Design의 featured list와 grouped list, Classic의 lead와 compact index는 어떤 입력을 공유하는가?
3. `ProjectsPage`가 계산·routing·shell·renderer dispatch를 동시에 소유한 결과는 무엇인가?
4. 빈 featured list, unknown category, 짧은 group에서 각 표현은 무엇을 생략하는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `e3fc25f46b42` | feat(projects): 프로젝트 그룹 정렬 규칙 추가 | B | RENDERER | 프로젝트 category grouping과 stable presentation order 규칙을 추가합니다. |
| 2 | `cd2eac06b220` | feat(projects): 디자인 프로젝트 소개 영역 추가 | B | RENDERER | Design 프로젝트 페이지의 hero와 archive metrics vocabulary를 만듭니다. |
| 3 | `e547aafcb77a` | feat(projects): 디자인 대표 프로젝트 목록 추가 | B | RENDERER | Design index에 featured projects를 별도 증거 목록으로 추가합니다. |
| 4 | `8f67e1990157` | feat(projects): 디자인 프로젝트 그룹 목록 추가 | B | RENDERER | Design index에 group별 project archive를 추가합니다. |
| 5 | `6d5b1dbbd3ae` | feat(projects): 클래식 프로젝트 소개와 터미널 추가 | B | RENDERER | Classic 프로젝트 index의 hero와 terminal-like summary를 만듭니다. |
| 6 | `b7ed48a4d77c` | feat(projects): 클래식 대표 프로젝트 추가 | B | RENDERER | Classic index에 첫 featured project를 lead evidence로 추가합니다. |
| 7 | `76a3155e757d` | feat(projects): 클래식 그룹 인덱스 추가 | B | RENDERER | Classic index에 category별 compact archive를 추가합니다. |
| 8 | `7571f1400065` | feat(projects): 프로젝트 목록 route 연결 | B | ROUTING, RENDERER | `/projects`에서 콘텐츠 로드, partitions/groups/metrics, shell, template dispatch를 통합합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `e3fc25f46b42` — feat(projects): 프로젝트 그룹 정렬 규칙 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 프로젝트 category grouping과 stable presentation order 규칙을 추가합니다.

#### Exact inspection targets

- `src/lib/portfolio/project-groups.ts` — `GroupedProjects`, `groupProjects`
- `Map<string, PortfolioProject[]>` 누적
- `groupOrder.indexOf`와 unknown group `localeCompare`

#### Commit-specific investigation tasks

1. `groupProjects`의 Map 축적과 sort callback을 따라 known/known, known/unknown, unknown/unknown 세 경우를 표로 재현합니다.
2. 프로젝트 입력 순서가 각 category 내부에서 보존되는지 배열 append 방식을 확인합니다.
3. presentation에 없는 category가 뒤로 이동하고 서로는 `localeCompare`되는지 exact code로 확인합니다.

#### Learner evidence record

- **Previous state:** 프로젝트 목록을 category별로 묶고 콘텐츠가 지정한 순서로 정렬하는 공통 규칙이 없었습니다.
- **Implementation decision and path:** 각 project를 category map에 누적한 뒤 configured group index로 정렬하고, 둘 다 unknown이면 category 이름의 `localeCompare`를 사용합니다.
- **Ownership and data lifetime:** project의 `category`가 membership을, `ProjectPageContent.groups`가 known category 순서를 소유합니다.
- **Failure, absence, and non-guarantee:** unknown category는 버리지 않고 known groups 뒤에 둡니다. 같은 category 안의 project 입력 순서는 보존되지만 명시적 secondary sort는 없습니다.
- **Resulting guarantee:** 모든 project가 정확히 한 category tuple에 들어가고 unknown group도 노출된다는 것을 보장합니다.
- **Relationship to later work:** 후속 Design/Classic view들이 같은 `GroupedProjects`를 소비하고 `757...`에서 route가 실제 계산합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `e3fc25f46b42` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `cd2eac06b220` — feat(projects): 디자인 프로젝트 소개 영역 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Design 프로젝트 페이지의 hero와 archive metrics vocabulary를 만듭니다.

#### Exact inspection targets

- `src/designs/design/projects/projects-route.tsx` — `DesignProjectsView`의 hero/intro
- projects page presentation copy와 total/curriculum/source-only count props
- `ContentHint`가 가리키는 projects presentation source

#### Commit-specific investigation tasks

1. Design projects view의 hero가 project-page presentation copy와 aggregate metrics 중 무엇을 직접 소비하는지 확인합니다.
2. route component가 아직 존재하지 않는 시점인지 imports/callers를 확인해 renderer와 lifecycle을 구분합니다.
3. empty project catalog에서 hero가 유지되는지 project-dependent DOM branch를 확인합니다.

#### Learner evidence record

- **Previous state:** grouping helper는 있었지만 이를 사용자에게 소개할 Design index 화면이 없었습니다.
- **Implementation decision and path:** Design view가 title/body와 세 count를 hero 정보 계층으로 구성합니다.
- **Ownership and data lifetime:** route가 전달한 count 값의 의미는 input contract가 소유하고 Design view는 label/copy와 배치를 소유합니다.
- **Failure, absence, and non-guarantee:** count가 0이어도 숫자 0으로 표시됩니다. 계산의 정확성은 이 view가 검증하지 않습니다.
- **Resulting guarantee:** Design 프로젝트 index가 프로젝트 archive 규모를 소개할 수 있음을 보장합니다.
- **Relationship to later work:** `e547...`과 `8f67...`이 실제 featured/grouped content를 같은 view에 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `cd2eac06b220` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `e547aafcb77a` — feat(projects): 디자인 대표 프로젝트 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Design index에 featured projects를 별도 증거 목록으로 추가합니다.

#### Exact inspection targets

- `src/designs/design/projects/projects-route.tsx` — `DesignProjectsView` featured block
- `featuredProjects` prop와 `ProjectCard`
- priority/variant와 template-preserving link

#### Commit-specific investigation tasks

1. Design view가 `featuredProjects`를 어떤 card variant와 priority 규칙으로 표현하는지 추적합니다.
2. featured와 non-featured source가 route에서 분리되기 전 이 renderer가 기대하는 props를 기록합니다.
3. featured list가 비어 있을 때 section heading/CTA의 실제 상태를 확인합니다.

#### Learner evidence record

- **Previous state:** Design hero는 수치만 보여 주고 대표 프로젝트로 이어지는 시각적 증거가 없었습니다.
- **Implementation decision and path:** featured projects를 독립 section에서 카드로 반복하고 현재 template/debug 상태를 project detail link에 전달합니다.
- **Ownership and data lifetime:** featured membership은 caller가, 카드 표현은 Design view/`ProjectCard`가 소유합니다.
- **Failure, absence, and non-guarantee:** featuredProjects가 비면 카드 목록이 비며 별도 empty state는 없습니다.
- **Resulting guarantee:** 대표 프로젝트가 non-featured archive와 분리되어 강조된다는 것을 보장합니다.
- **Relationship to later work:** `8f67...`이 나머지 프로젝트를 grouped archive로 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `e547aafcb77a` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `8f67e1990157` — feat(projects): 디자인 프로젝트 그룹 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Design index에 group별 project archive를 추가합니다.

#### Exact inspection targets

- `src/designs/design/projects/projects-route.tsx` — grouped sections
- `GroupedProjects` tuple의 category/project list 소비
- presentation `groups` copy lookup과 unknown group label fallback

#### Commit-specific investigation tasks

1. Design grouped-project section이 `GroupedProjects` tuple의 category와 projects를 어떤 중첩 반복으로 소비하는지 확인합니다.
2. group copy lookup과 unknown category label 처리 여부를 exact SHA에서 확인합니다.
3. group 내부 카드 링크가 template/debug state를 보존하는 prop 흐름을 기록합니다.

#### Learner evidence record

- **Previous state:** featured 항목 외의 track projects는 화면에 노출되지 않았습니다.
- **Implementation decision and path:** grouped tuple을 순회하며 category heading과 project cards를 렌더링합니다.
- **Ownership and data lifetime:** group 순서는 `groupProjects` 결과가, 각 group의 설명 copy와 card layout은 Design view가 소유합니다.
- **Failure, absence, and non-guarantee:** 빈 group tuple은 렌더링되지 않습니다. presentation에 설명이 없는 unknown category는 제한된 fallback 표현만 가집니다.
- **Resulting guarantee:** non-featured project가 category archive로 모두 도달 가능하다는 것을 보장합니다.
- **Relationship to later work:** Classic view는 같은 입력을 더 조밀한 terminal/index 형태로 소비합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `8f67e1990157` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `6d5b1dbbd3ae` — feat(projects): 클래식 프로젝트 소개와 터미널 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Classic 프로젝트 index의 hero와 terminal-like summary를 만듭니다.

#### Exact inspection targets

- `src/designs/classic/projects/projects-route.tsx` — `ClassicProjectsView`의 intro/terminal
- projects/count props
- group/project 수의 제한·요약 표현

#### Commit-specific investigation tasks

1. Classic projects intro/terminal이 어떤 counts와 page copy를 입력으로 받는지 추적합니다.
2. Design hero와 다른 정보 밀도·terminal composition을 파일 경계로 구분합니다.
3. terminal interaction 구현이 아니라 renderer composition만 추가되는지 diff 범위를 확인합니다.

#### Learner evidence record

- **Previous state:** 프로젝트 index에는 Design 표현만 있어 Classic template에서 같은 archive를 표현할 독립 구조가 없었습니다.
- **Implementation decision and path:** Classic view가 소개 copy와 count를 terminal 형식의 compact summary로 구성합니다.
- **Ownership and data lifetime:** 입력 데이터 파생은 caller가, terminal line과 밀도 제한은 Classic view가 소유합니다.
- **Failure, absence, and non-guarantee:** 긴 목록은 preview 범위로 잘릴 수 있으며 이 커밋은 전체 archive 접근을 아직 제공하지 않습니다.
- **Resulting guarantee:** Classic template에 맞는 프로젝트 intro가 존재함을 보장합니다.
- **Relationship to later work:** `b7ed...`이 lead project, `76a...`가 전체 grouped index를 이어 붙입니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `6d5b1dbbd3ae` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.6. `b7ed48a4d77c` — feat(projects): 클래식 대표 프로젝트 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Classic index에 첫 featured project를 lead evidence로 추가합니다.

#### Exact inspection targets

- `src/designs/classic/projects/projects-route.tsx` — lead featured block
- `featuredProjects[0]` 또는 동등한 첫 항목 선택
- screenshot/link/deployment 정보

#### Commit-specific investigation tasks

1. Classic featured presentation이 첫 featured project를 lead로 선택하는 index/slice 규칙을 확인합니다.
2. 나머지 featured projects를 어느 위치에서 처리하거나 처리하지 않는지 기록합니다.
3. featured가 0개인 branch에서 lead block이 생략되는지 확인합니다.

#### Learner evidence record

- **Previous state:** Classic intro는 archive 개요만 있고 사용자가 바로 확인할 대표 사례가 없었습니다.
- **Implementation decision and path:** featured list의 첫 project를 lead로 선택해 핵심 설명과 detail link를 보여 줍니다.
- **Ownership and data lifetime:** featured 순서가 lead 선택을 소유하고 Classic view는 첫 항목만 강조합니다.
- **Failure, absence, and non-guarantee:** featured가 없으면 lead block을 생략합니다. 두 번째 이후 featured는 이 lead 영역에서 보장되지 않습니다.
- **Resulting guarantee:** Classic index가 최소 한 개 대표 사례를 강조할 수 있음을 보장합니다.
- **Relationship to later work:** `76a...`이 나머지 grouped project를 compact archive로 제공합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `b7ed48a4d77c` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.7. `76a3155e757d` — feat(projects): 클래식 그룹 인덱스 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Classic index에 category별 compact archive를 추가합니다.

#### Exact inspection targets

- `src/designs/classic/projects/projects-route.tsx` — group index
- `groupedProjects`와 project availability/stack preview
- stack item 수 제한과 detail link

#### Commit-specific investigation tasks

1. Classic group index가 grouped tuples를 어떻게 compact navigation/content로 바꾸는지 추적합니다.
2. category 순서가 renderer에서 재정렬되지 않고 `groupProjects` 결과를 따르는지 확인합니다.
3. group 또는 project가 비었을 때 index count/link DOM의 실제 상태를 기록합니다.

#### Learner evidence record

- **Previous state:** Classic의 lead project만으로는 전체 non-featured archive에 접근할 수 없었습니다.
- **Implementation decision and path:** group tuple을 순회하고 각 project의 상태·요약·제한된 stack을 compact row/card로 렌더링합니다.
- **Ownership and data lifetime:** group membership/order는 helper 결과가, row의 정보 밀도와 stack truncation은 Classic view가 소유합니다.
- **Failure, absence, and non-guarantee:** stack이 제한 수를 넘으면 나머지는 표시하지 않습니다. unknown group을 제거하지 않습니다.
- **Resulting guarantee:** Classic에서도 전체 grouped archive로 이동할 수 있음을 보장합니다.
- **Relationship to later work:** `757...`이 실제 route에서 partitions/groups/counts를 한 번 계산해 두 view 중 하나에 전달합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `76a3155e757d` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.8. `7571f1400065` — feat(projects): 프로젝트 목록 route 연결

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Source-defined role:** `/projects`에서 콘텐츠 로드, partitions/groups/metrics, shell, template dispatch를 통합합니다.

#### Exact inspection targets

- `src/app/projects/page.tsx` — `ProjectsPage`
- `getPortfolioContent`, `resolveHomeTemplateId`, `groupProjects`
- `featuredProjects`, `trackProjects`, `sourceOnlyCount`, `curriculumCount`, `shellProps`
- `src/content/site.json`의 Projects nav

#### Commit-specific investigation tasks

1. `ProjectsPage`에서 featured/non-featured partition, grouping, source-only/curriculum counts의 계산 순서를 추적합니다.
2. `activeTemplate` branch가 Design/Classic views에 전달하는 props를 비교하고 shell/current path를 기록합니다.
3. `src/content/site.json`에 `/projects` navigation이 추가되는 integration 범위를 확인합니다.

#### Learner evidence record

- **Previous state:** 두 renderer와 helper는 존재했지만 실제 route가 입력을 조립하고 선택하지 않았습니다.
- **Implementation decision and path:** route가 featured/non-featured를 분리하고 group과 두 count를 계산한 뒤 Classic 또는 Design view에 동일한 payload를 전달합니다.
- **Ownership and data lifetime:** 이 시점에는 route가 data projection과 template dispatch를 모두 소유하며 renderer는 전달된 배열/숫자를 표현합니다.
- **Failure, absence, and non-guarantee:** invalid template/debug 처리 방식은 resolver 책임입니다. project가 0개여도 route는 성공하고 빈 표현을 전달합니다.
- **Resulting guarantee:** 동일한 derived data가 두 presentation에 제공되고 `/projects`가 site navigation에 노출됨을 보장합니다.
- **Relationship to later work:** 후대 view-model/renderer ownership 이동은 category 09, query/navigation lifecycle은 category 02에서 다룹니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `7571f1400065` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| 정렬 정책 | `e3fc...`가 category grouping과 configured/unknown order를 명시했습니다. |
| Design 표현 | `cd2...`–`8f67...`이 hero, featured evidence, grouped archive를 분리했습니다. |
| Classic 표현 | `6d5...`–`76a...`이 compact intro, lead, grouped index를 만들었습니다. |
| route 통합 | `757...`이 하나의 partitions/groups/metrics payload로 두 표현을 연결했습니다. |

## 7. Failure → Fix → Test and later relationships

- 이 commit set에는 direct regression test가 없습니다. grouping/view-model characterization은 category 07/09에서 소유합니다.
- unknown category를 뒤로 보내되 버리지 않는 정책은 failure tolerance이며, 별도 error UI가 아니라 deterministic ordering으로 처리됩니다.
- `757...`의 route-local projection은 후대 category 09의 `ProjectIndexViewModel`로 이동하지만 이 Thread의 feature semantics는 유지됩니다.

## 8. Ownership, state, and responsibility changes

- `groupProjects`: category membership과 order 파생.
- `ProjectsPage`: 당시 featured/track 분할, counts, shell/dispatch.
- Design view: 넓은 hero·featured cards·group sections.
- Classic view: terminal summary·single lead·compact group index.

## 9. Final thread state

모든 프로젝트는 featured 또는 track presentation에 들어가며, track은 configured category order와 unknown fallback order를 따릅니다. 두 template는 같은 derived data를 서로 다른 밀도로 표현합니다.

## 10. Final architecture and execution flow

콘텐츠 로드 → featured/non-featured 분할 → non-featured category grouping/ordering → source-only/curriculum count 계산 → template 선택 → 동일 payload를 Design 또는 Classic view에 전달 → project detail link 생성.

## 11. Minimal historical code evidence

`e3fc25f46b42`, `src/lib/portfolio/project-groups.ts`, `groupProjects`:
```ts
if (leftIndex === -1 && rightIndex === -1) return left.localeCompare(right);
if (leftIndex === -1) return 1;
if (rightIndex === -1) return -1;
return leftIndex - rightIndex;
```
configured group을 우선하고 unknown group을 버리지 않는 정렬 계약입니다.

## 12. Learning-completion checks

- [x] Frozen commit map 8개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 02-project-index-grouping-and-dual-presentation.md =====

===== BEGIN FILE: 03-project-detail-case-study-composition.md =====
# Development Thread: Project detail case-study composition

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: 상세 case-study 정보 계층, gallery/stack/decision evidence, dynamic route, missing ID, highlights.
- 제외: project link placement selector는 category 01/03, multi-renderer projection은 category 09, metadata/SEO는 category 06이 담당합니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 7
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

프로젝트 상세 화면이 공용 section primitives에서 출발해 문제·해결·구조·증거·의사결정·결과를 조립하고, dynamic route의 missing-project 처리와 highlights까지 완성되는 과정을 복원합니다.

### Fixed invariants

- 상세 route는 path ID를 canonical project로 해석하지 못하면 `notFound()`로 종료합니다.
- case-study section copy는 presentation content가, 실제 body/list/media는 project content가 제공합니다.
- 대표 screenshot과 gallery, stack, decisions, highlights, tradeoffs, results는 서로 다른 evidence role을 유지합니다.
- 각 historical SHA는 그 시점에 존재한 section만 설명하며 후대 view-model을 소급하지 않습니다.

## 2. Core engineering questions

1. 공용 section primitive가 어떤 반복 구조를 추상화하고 어떤 semantics는 호출자에게 남기는가?
2. intro에서 results까지 section이 추가될 때 project schema의 어느 필드를 소비하는가?
3. `generateStaticParams`, `getProjectById`, `notFound()`의 관계는 무엇인가?
4. `383...`은 상세 highlights 추가와 metric 계산 중앙화를 왜 같은 integration point에서 수행하는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `e5b26b762c50` | feat(project): 상세 화면 섹션 프리미티브 추가 | B | RENDERER | case-study 본문을 조립할 제목·2열 본문·목록 primitive를 추가합니다. |
| 2 | `06fff9a6e93b` | feat(project): 프로젝트 상세 소개 추가 | B | RENDERER | 상세 hero, 뒤로가기, 상태·링크·대표 screenshot을 조립합니다. |
| 3 | `9c0c37fa5c3c` | feat(project): 프로젝트 문제와 해결 설명 추가 | B | RENDERER | case study의 problem과 solution을 명시적 두 section으로 추가합니다. |
| 4 | `cabf3a0e378f` | feat(project): 프로젝트 구조와 증거 갤러리 추가 | B | RENDERER | architecture 설명과 screenshot gallery를 상세 본문에 추가합니다. |
| 5 | `1eac524fc8ff` | feat(project): 프로젝트 기술과 의사결정 추가 | B | RENDERER | stack, decisions, tradeoffs, results를 case-study 후반에 추가합니다. |
| 6 | `d4c7f742fb4d` | feat(project): 프로젝트 상세 route 연결 | B | ROUTING, RENDERER | dynamic route, static params, missing-project 404와 shell을 연결합니다. |
| 7 | `383a3b86e119` | feat(content): 프로젝트 지표를 화면에 적용 | B | CONTENT | 상세에 highlights를 노출하고 project metrics의 소비를 공용 selector로 전환합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `e5b26b762c50` — feat(project): 상세 화면 섹션 프리미티브 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** case-study 본문을 조립할 제목·2열 본문·목록 primitive를 추가합니다.

#### Exact inspection targets

- `src/components/portfolio/project-detail-sections.tsx` — `SectionTitle`, `TwoColumnSection`, `ListSection`
- list key가 item text인 점
- eyebrow/title/body/items prop 경계

#### Commit-specific investigation tasks

1. `SectionTitle`, `TwoColumnSection`, `ListSection`의 prop 계약과 semantic elements를 비교합니다.
2. 각 primitive가 layout만 소유하고 project content lookup은 수행하지 않는지 확인합니다.
3. 빈 `items`를 전달했을 때 `ListSection`의 heading/empty `<ul>` 상태를 기록합니다.

#### Learner evidence record

- **Previous state:** 상세 page의 반복되는 section heading과 list 구조를 일관되게 만들 공용 primitive가 없었습니다.
- **Implementation decision and path:** `SectionTitle`, `TwoColumnSection`, `ListSection`이 typography/layout을 고정하고 실제 의미와 콘텐츠는 props로 받습니다.
- **Ownership and data lifetime:** primitive는 DOM/표현을 소유하고 어떤 project field가 어느 section에 들어가는지는 이후 `ProjectDetailView`가 소유합니다.
- **Failure, absence, and non-guarantee:** 빈 `items`는 빈 `<ul>`이 되며 자동 생략하지 않습니다. duplicate item text는 key 충돌 가능성이 있습니다.
- **Resulting guarantee:** 상세 section의 반복 DOM contract를 재사용할 수 있음을 보장합니다.
- **Relationship to later work:** `06fff...`부터 실제 project detail view가 이 primitive를 소비합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `e5b26b762c50` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `06fff9a6e93b` — feat(project): 프로젝트 상세 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 상세 hero, 뒤로가기, 상태·링크·대표 screenshot을 조립합니다.

#### Exact inspection targets

- `src/components/portfolio/project-detail-view.tsx` — `ProjectDetailView`
- `ProjectLinks`, deployment badge/status, `ProjectScreenshot priority`
- `pageCopy`, `project`, `homeTemplate`, `contentDebug` props

#### Commit-specific investigation tasks

1. `ProjectDetailView` 초기 hero가 project category/title/summary/status/links/screenshot 중 무엇을 소비하는지 추적합니다.
2. link renderer와 screenshot component로 전달되는 project data와 template/debug props를 확인합니다.
3. 후속 problem/solution/architecture sections가 아직 없는지 exact file에서 확인합니다.

#### Learner evidence record

- **Previous state:** section primitive만 있고 사용자에게 보이는 project detail composition이 없었습니다.
- **Implementation decision and path:** `ProjectDetailView`가 project title/category/summary, index back link, deployment/link actions, 대표 screenshot을 첫 정보 계층으로 구성합니다.
- **Ownership and data lifetime:** project content가 사실 데이터를, presentation copy가 label을, view가 순서와 강조를 소유합니다.
- **Failure, absence, and non-guarantee:** optional link/media는 각 child component의 조건에 따라 생략됩니다. 이 시점에는 route ID 검증이 아직 연결되지 않았습니다.
- **Resulting guarantee:** canonical project object를 받으면 상세 소개를 렌더링할 수 있음을 보장합니다.
- **Relationship to later work:** `9c0...`–`1eac...`이 본문 evidence를 추가하고 `d4c...`이 dynamic route를 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `06fff9a6e93b` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `9c0c37fa5c3c` — feat(project): 프로젝트 문제와 해결 설명 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** case study의 problem과 solution을 명시적 두 section으로 추가합니다.

#### Exact inspection targets

- `src/components/portfolio/project-detail-view.tsx` — `TwoColumnSection` 호출
- `project.problem`, `project.solution`
- `presentation.pages.projectDetail.sections`의 label/title

#### Commit-specific investigation tasks

1. problem과 solution fields가 `TwoColumnSection`에 어떤 presentation labels와 함께 연결되는지 확인합니다.
2. 두 서술이 project object에서 직접 읽히는지 별도 derived model이 있는지 판별합니다.
3. 빈 문자열/필드 부재에 대한 condition이 있는지 exact JSX를 기록합니다.

#### Learner evidence record

- **Previous state:** 소개는 프로젝트가 무엇인지 보여 주지만 왜 만들었고 어떻게 해결했는지 설명하지 못했습니다.
- **Implementation decision and path:** problem과 solution body를 각각 presentation label과 결합해 순차 렌더링합니다.
- **Ownership and data lifetime:** project가 서술 내용을, presentation이 heading vocabulary를, view가 문제→해결 순서를 소유합니다.
- **Failure, absence, and non-guarantee:** 빈 문자열이어도 section DOM은 남습니다. 내용 필수성 검증은 content schema 책임이며 이 커밋의 UI fallback은 없습니다.
- **Resulting guarantee:** 각 project가 문제와 해결을 독립적인 case-study 단계로 제시함을 보장합니다.
- **Relationship to later work:** `cabf...`이 solution 뒤에 구조와 시각 증거를 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `9c0c37fa5c3c` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `cabf3a0e378f` — feat(project): 프로젝트 구조와 증거 갤러리 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** architecture 설명과 screenshot gallery를 상세 본문에 추가합니다.

#### Exact inspection targets

- `src/components/portfolio/project-detail-view.tsx` — architecture section, screenshot gallery
- `project.architecture`, `project.screenshots` 또는 gallery source
- 대표 image와 보조 image의 역할 구분

#### Commit-specific investigation tasks

1. architecture body와 gallery images가 각각 어떤 component/section에 추가되는지 추적합니다.
2. 대표 screenshot과 gallery의 중복 제거가 이 SHA에 존재하는지 확인합니다.
3. gallery가 빈 경우 wrapper/heading을 생략하는 조건이 있는지 기록합니다.

#### Learner evidence record

- **Previous state:** 문제·해결 서술만으로는 구현 구조와 관찰 가능한 산출물을 증명하기 어려웠습니다.
- **Implementation decision and path:** architecture body와 여러 screenshot을 별도 section으로 렌더링해 설명과 시각 증거를 연결합니다.
- **Ownership and data lifetime:** project content가 media 순서/설명을, view가 gallery layout과 section 위치를 소유합니다.
- **Failure, absence, and non-guarantee:** 보조 screenshot이 비면 gallery item이 없습니다. 대표 screenshot과의 중복 제거는 이 시점의 view가 보장하지 않습니다.
- **Resulting guarantee:** 구조 설명과 복수 visual evidence를 case study에서 보여 줄 수 있음을 보장합니다.
- **Relationship to later work:** 후대 category 09의 detail view model이 대표 이미지 중복 제거를 route projection으로 이동합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `cabf3a0e378f` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `1eac524fc8ff` — feat(project): 프로젝트 기술과 의사결정 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** stack, decisions, tradeoffs, results를 case-study 후반에 추가합니다.

#### Exact inspection targets

- `src/components/portfolio/project-detail-view.tsx` — `StackList`, 여러 `ListSection`
- `project.stack`, `decisions`, `tradeoffs`, `results`
- section order와 empty-array behavior

#### Commit-specific investigation tasks

1. stack, decisions, tradeoffs, results가 어떤 section primitive와 presentation labels에 대응하는지 표로 작성합니다.
2. stack IDs의 canonical metadata 해석 책임이 어디에 있는지 확인합니다.
3. 각 list가 비었을 때 route가 section을 가드하는지 `ListSection`에 빈 배열을 넘기는지 기록합니다.

#### Learner evidence record

- **Previous state:** 상세에는 문제·해결·구조·media만 있고 기술 선택과 결과를 평가할 근거가 부족했습니다.
- **Implementation decision and path:** 기술 스택을 canonical list component로, decisions/tradeoffs/results를 독립 list sections로 구성합니다.
- **Ownership and data lifetime:** project 배열 순서가 항목 순서를, view가 section 순서를 소유합니다.
- **Failure, absence, and non-guarantee:** 빈 배열이어도 `ListSection` 자체가 자동으로 사라지지 않습니다. ID-to-tech metadata 해석의 정확성은 `StackList` 책임입니다.
- **Resulting guarantee:** 기술 선택, 판단, 비용, 결과를 분리해 설명할 수 있음을 보장합니다.
- **Relationship to later work:** `d4c...`이 이 완성된 view를 실제 dynamic route에 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `1eac524fc8ff` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.6. `d4c7f742fb4d` — feat(project): 프로젝트 상세 route 연결

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Source-defined role:** dynamic route, static params, missing-project 404와 shell을 연결합니다.

#### Exact inspection targets

- `src/app/projects/[projectId]/page.tsx` — `generateStaticParams`, `ProjectDetailPage`
- `getProjectById(projectId, content)`, `notFound()`
- `PageShell`, template switcher currentPath, `ProjectDetailView` call

#### Commit-specific investigation tasks

1. `generateStaticParams`가 project IDs를 어떻게 생성하고 `ProjectDetailPage`가 async params를 어떻게 해석하는지 추적합니다.
2. `getProjectById` 실패가 `notFound()`로 전환되는 위치와 그 이후 code reachability를 확인합니다.
3. shell/template switcher currentPath와 `ProjectDetailView` prop 전달을 기록합니다.

#### Learner evidence record

- **Previous state:** 상세 view는 canonical project object를 요구했지만 URL ID에서 이를 얻는 production route가 없었습니다.
- **Implementation decision and path:** `generateStaticParams`가 모든 project ID를 반환하고 page가 params/query를 해석한 뒤 selector로 project를 찾습니다. 찾지 못하면 `notFound()`를 호출합니다.
- **Ownership and data lifetime:** route가 ID 해석·404·shell/template state를 소유하고 view는 valid project의 case-study 표현만 소유합니다.
- **Failure, absence, and non-guarantee:** unknown ID는 빈 상세가 아니라 404입니다. content load/schema failure는 이 route가 복구하지 않습니다.
- **Resulting guarantee:** 등록된 ID는 상세 view로, 미등록 ID는 Next.js not-found 경계로 간다는 것을 보장합니다.
- **Relationship to later work:** `383...`이 highlights evidence를 추가하고 후대 category 09가 renderer-safe detail projection을 도입합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `d4c7f742fb4d` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.7. `383a3b86e119` — feat(content): 프로젝트 지표를 화면에 적용

- **Importance:** B
- **Tags:** CONTENT
- **Source-defined role:** 상세에 highlights를 노출하고 project metrics의 소비를 공용 selector로 전환합니다.

#### Exact inspection targets

- `src/components/portfolio/project-detail-view.tsx` — highlights `ListSection`
- `src/app/projects/page.tsx`, `src/components/portfolio/work-map-section.tsx` — `getProjectMetricValue`
- `project.highlights`와 presentation highlights copy

#### Commit-specific investigation tasks

1. 한 커밋에서 `ProjectDetailView` highlights 추가와 project metric selector 적용을 파일별로 분리해 기록합니다.
2. highlights가 decisions와 tradeoffs 사이 어느 위치에 들어가며 어떤 labels를 쓰는지 확인합니다.
3. projects page/work-map의 inline 산식이 `getProjectMetricValue`로 대체되는 전후를 비교합니다.

#### Learner evidence record

- **Previous state:** project highlights는 content에 있어도 상세에서 보이지 않았고, project count 산식은 여러 consumer에 중복돼 있었습니다.
- **Implementation decision and path:** decisions 뒤에 highlights list를 추가하고 projects page/work-map의 직접 filter 산식을 `getProjectMetricValue` 호출로 교체합니다.
- **Ownership and data lifetime:** 상세 view는 highlights 배치를, selector는 metric 의미와 산식을 소유합니다.
- **Failure, absence, and non-guarantee:** highlights가 비면 빈 list section이 될 수 있습니다. selector 자체의 correctness test는 이 commit map에 포함되지 않습니다.
- **Resulting guarantee:** project highlights가 case-study 증거로 노출되고 동일 metric key가 consumer 간 같은 값을 사용함을 보장합니다.
- **Relationship to later work:** metric selector 정책은 category 01, route projection과 regression tests는 category 09/07에서 깊게 다룹니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `383a3b86e119` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| 표현 primitive | `e5b...`가 section DOM을 고정했습니다. |
| case-study 서사 | `06fff...`–`1eac...`이 소개→문제/해결→구조/갤러리→기술/결정/결과를 누적했습니다. |
| route failure boundary | `d4c...`이 unknown ID를 `notFound()`로 명시했습니다. |
| 증거 보강 | `383...`이 highlights를 노출하고 metric consumer를 중앙 selector에 연결했습니다. |

## 7. Failure → Fix → Test and later relationships

- `d4c...`은 missing project를 fail-closed 404로 처리하지만 이 frozen map에 직접 404 test commit은 없습니다.
- `383...`은 기능 확장과 중복 산식 제거를 함께 수행합니다. metric selector의 full policy는 이 Thread의 범위를 넘습니다.
- 후대 detail links, view models, renderer matrix는 각각 category 01/03, 09, 07이 담당합니다.

## 8. Ownership, state, and responsibility changes

- project content: case-study facts와 evidence arrays.
- presentation content: section labels/titles.
- `ProjectDetailView`: section order와 표현.
- dynamic route: ID resolution, not-found, template/debug shell state.

## 9. Final thread state

유효한 project ID는 완전한 case-study 정보 계층으로 연결되고, 유효하지 않은 ID는 404입니다. 상세은 problem/solution/architecture/media/stack/decisions/highlights/tradeoffs/results를 분리합니다.

## 10. Final architecture and execution flow

static params 생성 또는 request ID 수신 → content aggregate 로드 → template/debug 해석 → project ID selector → missing이면 `notFound()` → valid project와 page copy를 상세 view에 전달 → 각 evidence section을 순서대로 렌더링.

## 11. Minimal historical code evidence

`d4c7f742fb4d`, `src/app/projects/[projectId]/page.tsx`, `ProjectDetailPage`:
```tsx
const project = getProjectById(projectId, content);
if (!project) {
  notFound();
}
```
unknown ID가 부분 성공이나 빈 상세로 이어지지 않는 route failure boundary입니다.

## 12. Learning-completion checks

- [x] Frozen commit map 7개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 03-project-detail-case-study-composition.md =====

===== BEGIN FILE: 04-about-profile-skills-experience-and-curation.md =====
# Development Thread: About profile, skills, experience, and curation

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: `/about`의 profile·principles·journey·skills·focus areas·experience·curation 표현과 큐레이션 프로젝트 참조 해석.
- 제외: 공용 `JourneyList`, `StackList`, 이미지 primitive의 내부 상호작용은 category 03, route query/lifecycle은 category 02, 후대 About view-model 소유권은 category 09가 담당합니다.
- 큐레이션의 schema validation과 fail-closed ingestion은 category 01/09의 책임이며, 이 Thread는 유효한 aggregate를 About 경험으로 조립하는 소비자 규칙에 집중합니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 9
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

About route가 profile 소개에서 시작해 journey, 기술 그룹, 경력, 큐레이션 기준과 프로젝트 범주를 단계적으로 결합하는 과정을 복원하고, optional media·page enablement·누락 프로젝트 참조의 실제 처리 규칙을 확인합니다.

### Fixed invariants

- About의 기본 profile·principles는 `profile`과 `presentation.pages.about`에서 오며 route가 별도 하드코딩된 자기소개를 소유하지 않습니다.
- profile photo는 optional이며 없으면 소개 본문은 유지되고 이미지 부분만 생략됩니다.
- curation section은 `isSitePageEnabled("curation", content)`가 참일 때만 존재합니다.
- 큐레이션 category의 `projectIds`는 원본 순서대로 해석하며 찾지 못한 ID는 링크로 만들지 않습니다.
- curation criteria, categories, omissions, next-review 정보는 각각 다른 의미를 가지며 누락 참조를 임의의 프로젝트로 대체하지 않습니다.

## 2. Core engineering questions

1. 초기 About route는 profile과 principles의 어느 필드를 직접 소비하는가?
2. journey·skills·experience가 추가되면서 route가 어떤 공용 renderer를 호출하고 어떤 데이터는 계속 직접 반복하는가?
3. photo가 없거나 skills/experience 배열이 비면 어느 DOM만 생략되는가?
4. curation page flag와 curation data의 존재는 어떻게 구분되는가?
5. category `projectIds` 중 해석할 수 없는 값은 어떤 방식으로 제거되며 category 설명 자체는 남는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `7fe913796acb` | feat(about): 프로필과 원칙 소개 추가 | B | RENDERER | About route의 profile hero와 principles 목록을 최초 구성합니다. |
| 2 | `edf8e75716a3` | feat(about): 여정 요약 추가 | B | RENDERER | About에 전체 journey의 요약 timeline을 추가합니다. |
| 3 | `bea99bce4478` | feat(about): 기술 그룹 소개 추가 | B | RENDERER | skills group의 기술 ID와 설명을 About evidence로 추가합니다. |
| 4 | `a00a6bf1af58` | feat(about): 프로필 사진 소개 추가 | B | RENDERER | optional profile photo를 About hero에 통합합니다. |
| 5 | `4b9c2894a756` | feat(about): 기술 집중 영역 추가 | B | RENDERER | skills의 focus areas를 About 페이지에 설명형 evidence로 추가합니다. |
| 6 | `a7723eb193eb` | feat(about): 경력 목록 추가 | B | RENDERER | experience records를 About route의 경력 evidence로 추가합니다. |
| 7 | `924bcd75aade` | feat(about): 큐레이션 기준 소개 추가 | B | CONTENT, RENDERER | page flag로 보호되는 curation intro와 criteria를 About에 추가합니다. |
| 8 | `80903ec6197f` | feat(about): 큐레이션 프로젝트 범주 추가 | B | CONTENT, RENDERER | curation category의 project IDs를 해석해 route-preserving evidence links로 연결합니다. |
| 9 | `a65f27363837` | feat(about): 큐레이션 공백과 재검토 추가 | B | CONTENT, RENDERER | curation의 omissions와 next-review 정보를 공개해 현재 선택의 비보장 범위를 명시합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `7fe913796acb` — feat(about): 프로필과 원칙 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** About route의 profile hero와 principles 목록을 최초 구성합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — `AboutPage`
- `src/content/profile.json` — `name`, `handle`, `summary`, `principles`
- `src/content/presentation.json` — `pages.about.hero`, `pages.about.principles`
- `PageShell`, `ContentHint`, `SectionHeading` 호출 경계

#### Commit-specific investigation tasks

1. `AboutPage` hero와 principles section이 profile/presentation의 어떤 fields를 직접 소비하는지 추적합니다.
2. principles의 key와 card 반복, content-debug hint 경로를 확인합니다.
3. photo/journey/skills/experience/curation이 아직 없는 최소 state를 기록합니다.

#### Learner evidence record

- **Previous state:** 직전에는 `/about`에서 profile summary와 principles를 실제 페이지 구조로 제시하는 route가 없었습니다.
- **Implementation decision and path:** `AboutPage`가 content aggregate와 query state를 읽어 `PageShell`을 만들고, profile identity/summary와 principles 카드를 두 섹션으로 렌더링합니다.
- **Ownership and data lifetime:** profile과 presentation JSON이 문구를 소유하고, route가 섹션 배치와 principles 반복을 소유합니다. shell은 navigation/template switcher chrome을 소유합니다.
- **Failure, absence, and non-guarantee:** principles가 비면 heading 아래 카드 목록만 비며 별도 empty state는 없습니다. 이 시점에는 photo, journey, skills, experience, curation이 없습니다.
- **Resulting guarantee:** About의 최소 profile/principles 경험과 content-debug source hint를 보장합니다. 후속 evidence sections나 page disable 정책은 보장하지 않습니다.
- **Relationship to later work:** `edf8...`부터 같은 route에 journey와 다른 evidence sections가 누적됩니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `7fe913796acb` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `edf8e75716a3` — feat(about): 여정 요약 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** About에 전체 journey의 요약 timeline을 추가합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — `AboutPage`의 journey section
- `JourneyList` 호출과 `content.journey` 전달
- `presentation.pages.about.journey` heading copy
- project case-study 링크의 `homeTemplate`/`contentDebug` 전달

#### Commit-specific investigation tasks

1. About journey section이 `content.journey`를 `JourneyList`에 전달하는 prop 경계를 확인합니다.
2. homeTemplate/contentDebug가 case-study links에 보존되는지 기록합니다.
3. Journey narrative milestones가 이 summary source에 포함되지 않는지 확인합니다.

#### Learner evidence record

- **Previous state:** profile과 principles만으로는 경력의 시간적 순서와 프로젝트 전개를 보여 주지 못했습니다.
- **Implementation decision and path:** About route가 `content.journey`를 공용 `JourneyList`에 전달하고 presentation copy로 별도 journey section을 구성합니다.
- **Ownership and data lifetime:** journey 원본과 순서는 content가, timeline DOM과 project 링크 표현은 `JourneyList`가, section 배치는 About route가 소유합니다.
- **Failure, absence, and non-guarantee:** journey 배열이 비면 timeline 항목이 없으며 별도 오류나 placeholder를 만들지 않습니다. narrative milestone의 state/reason/result는 이 목록에 포함되지 않습니다.
- **Resulting guarantee:** About에서 전체 journey chronology를 재사용할 수 있음을 보장합니다. `/journey`의 깊은 의사결정 narrative는 보장하지 않습니다.
- **Relationship to later work:** `bea99...`가 기술 그룹을, Journey Thread가 별도의 milestone narrative를 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `edf8e75716a3` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `bea99bce4478` — feat(about): 기술 그룹 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** skills group의 기술 ID와 설명을 About evidence로 추가합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — skills/group section
- `content.skills.groups` 반복
- `StackList` 호출과 group `items` 전달
- `src/content/site.json` — About navigation 연결 여부

#### Commit-specific investigation tasks

1. skills groups의 title/items가 About cards와 `StackList`로 어떻게 분리 전달되는지 추적합니다.
2. canonical tech metadata lookup을 About route가 직접 수행하는지 확인합니다.
3. skills groups가 비어 있을 때 section DOM과 navigation integration을 기록합니다.

#### Learner evidence record

- **Previous state:** About에는 시간적 맥락은 있었지만 어떤 기술 묶음을 다루는지 구조적으로 보여 주는 영역이 없었습니다.
- **Implementation decision and path:** 각 skills group의 title과 item ID 목록을 카드로 구성하고, 실제 기술 표시를 공용 `StackList`에 위임합니다.
- **Ownership and data lifetime:** group membership와 순서는 skills content가, 기술 ID의 표시 해석은 `StackList`/canonical tech stack이, 카드 배치는 About route가 소유합니다.
- **Failure, absence, and non-guarantee:** groups가 비면 섹션 내용이 비고, canonical metadata가 없는 ID의 표현은 `StackList`의 별도 fallback 정책에 따릅니다.
- **Resulting guarantee:** About가 skills taxonomy를 content-driven하게 표현한다는 것을 보장합니다. 아이콘 매핑의 완전성은 보장하지 않습니다.
- **Relationship to later work:** `4b9c...`가 group membership과 별개인 focus-area 설명을 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `bea99bce4478` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `a00a6bf1af58` — feat(about): 프로필 사진 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** optional profile photo를 About hero에 통합합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — hero의 `ProfilePhoto` 조건부 렌더링
- `content.profile.photo`
- `ProfilePhoto` prop 전달
- photo가 없는 branch의 기존 text layout

#### Commit-specific investigation tasks

1. `content.profile.photo` 존재 여부가 `ProfilePhoto` component를 가드하는 exact branch를 확인합니다.
2. photo가 없어도 hero text가 유지되는 layout을 부모 state와 비교합니다.
3. image loading/fallback 구현이 이 route commit에 포함되는지 별도 primitive인지 구분합니다.

#### Learner evidence record

- **Previous state:** About hero는 identity와 summary만 보여 주고 profile의 photo metadata를 소비하지 않았습니다.
- **Implementation decision and path:** `content.profile.photo`가 있을 때만 공용 `ProfilePhoto`를 렌더링하도록 hero 구성을 확장합니다.
- **Ownership and data lifetime:** photo metadata는 profile content가, 이미지 최적화·fallback 표현은 공용 component가, 존재 여부에 따른 배치는 About route가 소유합니다.
- **Failure, absence, and non-guarantee:** photo가 없으면 이미지 부분만 `null`이고 name, handle, title, summary는 그대로 남습니다. 깨진 asset의 runtime 처리까지 이 커밋이 보장하지 않습니다.
- **Resulting guarantee:** 이미지 부재가 About 전체 실패로 전파되지 않는 optional media 계약을 보장합니다.
- **Relationship to later work:** `4b9c...`은 별도 skills focus evidence를 추가하며 photo 조건과 독립적입니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `a00a6bf1af58` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `4b9c2894a756` — feat(about): 기술 집중 영역 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** skills의 focus areas를 About 페이지에 설명형 evidence로 추가합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — focus-area section
- `content.skills.focusAreas`
- `presentation.pages.about`의 해당 heading/copy
- `ContentHint`가 가리키는 `src/content/skills.json` 경로

#### Commit-specific investigation tasks

1. focus-area title/body 반복과 presentation heading의 source를 확인합니다.
2. skills group ID 목록과 focus-area 설명이 별도 data paths인지 구분합니다.
3. 빈 focusAreas의 DOM 상태와 duplicate-title key 비보장을 기록합니다.

#### Learner evidence record

- **Previous state:** 기술 그룹은 ID 묶음만 보여 주어 어떤 문제 영역에 집중하는지 설명하지 못했습니다.
- **Implementation decision and path:** focus area의 title/body를 독립 카드로 반복해 기술 선택의 서술적 맥락을 추가합니다.
- **Ownership and data lifetime:** focus-area 문구와 순서는 skills content가, 카드 구조와 section placement는 About route가 소유합니다.
- **Failure, absence, and non-guarantee:** 배열이 비면 카드가 없으며 별도 empty state는 없습니다. title 중복이나 content quality 검증은 이 route의 책임이 아닙니다.
- **Resulting guarantee:** About가 기술 taxonomy와 기술적 관심 설명을 별도 evidence로 제공함을 보장합니다.
- **Relationship to later work:** `a772...`가 실제 experience records를 연결해 설명을 경력 근거로 확장합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `4b9c2894a756` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.6. `a7723eb193eb` — feat(about): 경력 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** experience records를 About route의 경력 evidence로 추가합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — experience section과 item 반복
- `content.experience`
- role/company/period/description 또는 highlights 소비 위치
- `ContentHint`의 `src/content/experience.json` 참조

#### Commit-specific investigation tasks

1. experience records가 role/company/period/description/highlights 중 어떤 fields로 표현되는지 exact SHA에서 확인합니다.
2. profile/skills claims와 experience evidence의 section ordering을 기록합니다.
3. experience가 비었을 때 section guard가 있는지 확인합니다.

#### Learner evidence record

- **Previous state:** About는 원칙과 기술을 설명했지만 실제 역할·기간·업무 경험을 직접 제시하지 않았습니다.
- **Implementation decision and path:** experience 배열을 순서대로 카드/목록으로 렌더링해 profile claim을 경력 기록과 연결합니다.
- **Ownership and data lifetime:** 경력 데이터와 순서는 experience content가, 표현 계층과 section placement는 About route가 소유합니다.
- **Failure, absence, and non-guarantee:** experience가 비어도 이전 About sections는 유지되며 이 시점에는 별도의 empty-state copy를 만들지 않습니다.
- **Resulting guarantee:** About가 실제 experience content를 profile 설명과 같은 route에서 제시함을 보장합니다.
- **Relationship to later work:** `924...`부터 프로젝트 선택 기준을 설명하는 별도의 curation 영역이 조건부로 추가됩니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `a7723eb193eb` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.7. `924bcd75aade` — feat(about): 큐레이션 기준 소개 추가

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Source-defined role:** page flag로 보호되는 curation intro와 criteria를 About에 추가합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — `isSitePageEnabled("curation", content)`, `CurationSection`
- `content.curation.intro`, `content.curation.criteria.items`
- `presentation.pages.about.curation`
- `CurationSection`의 `aria-label`과 criteria card 반복

#### Commit-specific investigation tasks

1. `isSitePageEnabled("curation", content)`가 `CurationSection` 전체를 가드하는 위치를 확인합니다.
2. curation intro와 criteria items가 presentation copy와 어떻게 결합되는지 추적합니다.
3. page flag off와 criteria empty를 서로 다른 상태로 기록합니다.

#### Learner evidence record

- **Previous state:** About는 무엇을 했는지는 보여 줬지만 어떤 기준으로 대표 evidence를 선택하는지 공개하지 않았습니다.
- **Implementation decision and path:** curation page가 활성화된 경우에만 intro와 criteria cards를 렌더링하고 presentation copy를 section heading에 사용합니다.
- **Ownership and data lifetime:** site page flag가 section 존재를, curation content가 기준 문구를, About route가 표현 구조를 소유합니다.
- **Failure, absence, and non-guarantee:** flag가 꺼져 있으면 curation section 전체가 없습니다. criteria가 비면 intro와 heading은 남고 card 목록만 비며, 이 커밋은 project links를 아직 만들지 않습니다.
- **Resulting guarantee:** 비활성 curation을 암묵적으로 공개하지 않고, 활성 상태에서 선택 기준을 명시적으로 보여 줌을 보장합니다.
- **Relationship to later work:** `809...`가 기준 설명을 실제 project categories와 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `924bcd75aade` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.8. `80903ec6197f` — feat(about): 큐레이션 프로젝트 범주 추가

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Source-defined role:** curation category의 project IDs를 해석해 route-preserving evidence links로 연결합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — `CurationSection`, `CurationCategoryCard`
- `category.projectIds.map(...find...).filter(Boolean)`
- `getTemplateHref(`/projects/${project.id}`, homeTemplate, { contentDebug })`
- `content.curation.categories`와 `PortfolioContent.projects`

#### Commit-specific investigation tasks

1. `CurationCategoryCard`의 `projectIds.map(find).filter(Boolean)` lookup을 단계별로 재현합니다.
2. 모든 project ID가 누락될 때 category rationale과 links list 중 무엇이 남는지 확인합니다.
3. `getTemplateHref`가 template/debug state를 detail links에 보존하는지 기록합니다.

#### Learner evidence record

- **Previous state:** curation 기준은 설명만 있었고 어떤 프로젝트가 각 기준 범주를 뒷받침하는지 탐색할 수 없었습니다.
- **Implementation decision and path:** 각 category의 label/rationale을 유지하면서 project IDs를 project aggregate에 해석하고, 찾은 항목만 detail 링크로 출력합니다.
- **Ownership and data lifetime:** category가 ID 순서와 rationale을, project aggregate가 표시 title과 실제 route ID를, route helper가 template/debug query 보존을 소유합니다.
- **Failure, absence, and non-guarantee:** 찾지 못한 project ID는 `filter(Boolean)`으로 제거됩니다. 모두 누락돼도 category card와 rationale은 남고 project link list만 생략됩니다.
- **Resulting guarantee:** 잘못된 참조가 깨진 링크로 공개되지 않고, 해석 가능한 근거만 원본 category 순서로 연결됨을 보장합니다.
- **Relationship to later work:** `a65f...`가 아직 의도적으로 비워 둔 영역과 재검토 시점을 큐레이션 설명에 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `80903ec6197f` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.9. `a65f27363837` — feat(about): 큐레이션 공백과 재검토 추가

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Source-defined role:** curation의 omissions와 next-review 정보를 공개해 현재 선택의 비보장 범위를 명시합니다.

#### Exact inspection targets

- `src/app/about/page.tsx` — `CurationSection`의 omissions/next-review 영역
- `src/content/curation.json` — omissions와 review 관련 필드
- `src/content/presentation.json` — About curation labels
- 비어 있는 omissions/review 값의 조건부 branch

#### Commit-specific investigation tasks

1. curation omissions와 next-review data가 criteria/categories와 다른 section/labels로 표현되는지 확인합니다.
2. 빈 omissions 또는 review 정보에 대한 conditional branch를 기록합니다.
3. 이 data가 자동 검증·scheduler가 아니라 편집된 non-guarantee disclosure임을 code/content 경계로 확인합니다.

#### Learner evidence record

- **Previous state:** criteria와 categories만 있으면 현재 evidence가 완전하고 영구적인 선택처럼 보일 수 있었습니다.
- **Implementation decision and path:** 의도적으로 제외되거나 아직 부족한 항목과 다음 재검토 정보를 curation section에 추가해 선택의 시간적·범위 한계를 표시합니다.
- **Ownership and data lifetime:** curation content가 공백과 review 계획을 소유하고, presentation이 label을, route가 해당 정보를 criteria/categories와 구분해 배치합니다.
- **Failure, absence, and non-guarantee:** 공백 목록이 비면 목록 내용이 없고, review 값이 없을 때 임의 날짜를 만들지 않습니다. 이 정보는 자동 검증이나 scheduler가 아닙니다.
- **Resulting guarantee:** 현재 curation이 완전성 보장이 아니라 명시된 기준·공백·재검토를 가진 편집 상태임을 보여 줍니다.
- **Relationship to later work:** 후대 route projection은 category 09에서 이 데이터를 renderer-safe model로 옮기지만 의미는 그대로 유지합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `a65f27363837` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| 기본 identity | `7fe...`가 profile summary와 principles를 About의 최소 계약으로 만들었습니다. |
| 근거 축 확장 | `edf...`부터 journey, skills, photo, focus areas, experience가 각각 독립 evidence section으로 추가됐습니다. |
| 큐레이션 활성 경계 | `924...`가 page flag를 section 존재의 선행 조건으로 만들었습니다. |
| 참조 해석 | `809...`가 category project IDs를 해석하되 누락 참조는 링크로 만들지 않는 정책을 도입했습니다. |
| 비보장 공개 | `a65...`가 omissions와 review 정보를 통해 큐레이션의 한계를 명시했습니다. |

## 7. Failure → Fix → Test and later relationships

- 이 frozen map에는 생산 코드를 직접 교정하는 fix commit이나 전용 regression test가 없습니다.
- `809...`의 missing-project drop은 runtime exception 복구가 아니라 소비자 측 참조 해석 정책입니다. schema-level invalid reference 차단은 category 01/09에서 다룹니다.
- 공용 Journey/Stack/ProfilePhoto의 interaction·fallback 회귀는 category 03/07의 테스트 책임이므로 이 문서에서 실행됐다고 주장하지 않습니다.

## 8. Ownership, state, and responsibility changes

- Profile/skills/experience/journey/curation JSON: claim과 배열 순서, page-specific evidence 원본.
- Site/presentation: curation 활성 여부와 route copy/labels.
- About route: section composition, optional photo, curation categories와 reference resolution.
- 공용 components: timeline, stack, photo의 표현 세부사항.
- 후대 About view model: category 09에서 raw aggregate 접근을 줄이되 이 route-feature 의미를 보존합니다.

## 9. Final thread state

About route는 profile·principles를 중심으로 journey, skills, focus areas, experience를 조립하고, curation이 활성화된 경우 기준·범주·공백·재검토를 추가합니다. 누락 project reference는 링크에서 제거되지만 category 설명은 유지됩니다.

## 10. Final architecture and execution flow

content aggregate와 page flag 로드 → `/about` 활성 route 구성 → profile/principles 및 optional photo 렌더링 → journey/skills/focus/experience evidence 반복 → curation flag 확인 → criteria와 categories 렌더링 → category project IDs를 aggregate에 해석 → 찾은 프로젝트만 template/debug-preserving detail 링크로 출력 → omissions/review로 비보장 범위 공개.

## 11. Minimal historical code evidence

`80903ec6197f`, `src/app/about/page.tsx`, `CurationCategoryCard`:
```tsx
const projects = category.projectIds
  .map((projectId) => content.projects.find((project) => project.id === projectId))
  .filter((project): project is NonNullable<typeof project> => Boolean(project));
```
이 발췌는 category 순서를 보존하면서 해석할 수 없는 참조를 깨진 링크로 만들지 않는 소비자 정책을 보여 줍니다.

## 12. Learning-completion checks

- [x] Frozen commit map 9개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 04-about-profile-skills-experience-and-curation.md =====

===== BEGIN FILE: 05-resume-evidence-and-conditional-sections.md =====
# Development Thread: Resume evidence and conditional sections

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: `/resume`의 hero·summary·download CTA, 선택 프로젝트, training, profile metadata, experience, education, notes 구성.
- 제외: route query/template lifecycle은 category 02, project-link transport와 공용 카드 primitive는 category 03, `getResumeProjects` selector의 일반 정책은 category 01, 후대 Resume view-model ownership은 category 09가 담당합니다.
- 이 Thread는 HTML resume 경험을 다루며 실제 PDF 파일 생성·배포·다운로드 성공 여부는 범위 밖입니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 7
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

Resume route가 profile summary에서 선택 프로젝트, training, location/availability, experience, education, notes로 확장되는 과정을 복원하고, 각 optional section과 download CTA가 서로 독립적으로 생략되는 계약을 확인합니다.

### Fixed invariants

- `resume.downloadUrl`이 빈 값이면 다운로드 CTA를 만들지 않으며 임의의 기본 파일 경로를 추정하지 않습니다.
- 선택 프로젝트는 `resume.projectIds`의 선언 순서를 보존하고 해석 가능한 project만 렌더링합니다.
- training, experience, education, notes는 서로 독립적인 데이터 집합이며 한 집합의 부재가 다른 section을 숨기지 않습니다.
- 후반 조건부 sections는 각 배열 길이를 직접 확인하고 빈 wrapper/heading을 남기지 않습니다.
- Resume route는 profile location/availability와 resume content를 결합하지만 원본 데이터의 lifetime은 application content aggregate가 소유합니다.

## 2. Core engineering questions

1. 초기 Resume hero와 summary는 어떤 원본을 결합하며 download CTA는 어떤 조건에서 존재하는가?
2. `getResumeProjects`는 project ID 순서와 누락 참조를 어떻게 처리하는가?
3. training과 experience/education은 어떤 데이터 구조와 presentation copy를 사용하며 서로 어떤 관계가 없는가?
4. location과 availability는 Resume 자체 데이터인가 profile 데이터인가?
5. experience, education, notes가 비어 있을 때 heading까지 사라지는지 item만 사라지는지 exact SHA에서 확인할 수 있는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `e655951b0706` | feat(resume): 이력 소개와 요약 추가 | B | RENDERER | Resume route의 hero, summary, optional download CTA를 최초 구성합니다. |
| 2 | `b399ce0c7f84` | feat(resume): 선택 프로젝트 경력 추가 | B | RENDERER | resume가 명시한 project IDs를 case-study evidence로 연결합니다. |
| 3 | `4d17fd7ab81b` | feat(resume): 교육 과정 요약 추가 | B | RENDERER | resume training records와 Resume navigation entry를 추가합니다. |
| 4 | `f64763cffcff` | feat(resume): 프로필 위치와 가용성 추가 | B | RENDERER | profile의 location과 availability를 Resume identity evidence에 통합합니다. |
| 5 | `3c113f1995ff` | feat(resume): 경력 이력 추가 | B | RENDERER | experience history를 데이터가 있을 때만 나타나는 Resume section으로 추가합니다. |
| 6 | `732ae5a6785c` | feat(resume): 교육 이력 추가 | B | RENDERER | education history를 별도의 조건부 Resume section으로 추가합니다. |
| 7 | `579ea168daa8` | feat(resume): Resume 안내 기록 추가 | B | RENDERER | resume notes를 데이터가 있을 때만 공개하는 마지막 안내 section으로 추가합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `e655951b0706` — feat(resume): 이력 소개와 요약 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Resume route의 hero, summary, optional download CTA를 최초 구성합니다.

#### Exact inspection targets

- `src/app/resume/page.tsx` — `ResumePage`
- `content.resume.summary`, `content.resume.downloadUrl`
- `content.profile.koreanName`, `content.profile.handle`
- `presentation.pages.resume.hero`, `presentation.pages.resume.summary`
- `downloadUrl ? <a ...> : null` branch

#### Commit-specific investigation tasks

1. `ResumePage` hero/summary가 profile, resume, presentation fields를 어떻게 결합하는지 추적합니다.
2. `downloadUrl` truthy branch가 CTA 전체를 가드하고 fallback path를 만들지 않는지 확인합니다.
3. summary가 빈 경우 heading과 반복 결과의 실제 DOM 상태를 기록합니다.

#### Learner evidence record

- **Previous state:** 직전에는 `/resume`에서 profile identity와 resume summary를 조립하거나 다운로드 가능 여부를 표현하는 route가 없었습니다.
- **Implementation decision and path:** `ResumePage`가 `PageShell` 안에 hero와 summary를 만들고, `downloadUrl`이 truthy일 때만 다운로드 anchor를 추가합니다.
- **Ownership and data lifetime:** resume content가 summary와 URL을, presentation이 labels/copy를, route가 CTA 존재와 section layout을 소유합니다.
- **Failure, absence, and non-guarantee:** download URL이 빈 문자열·undefined이면 CTA는 `null`입니다. summary가 비면 heading은 남고 item 목록만 비며, 이 시점에는 선택 프로젝트나 경력 섹션이 없습니다.
- **Resulting guarantee:** 없는 다운로드 파일을 추정하지 않고, content가 제공한 URL만 공개하는 최소 Resume 경험을 보장합니다.
- **Relationship to later work:** `b399...`가 선택 project evidence를 추가하고 후속 commits가 독립 sections를 누적합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `e655951b0706` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `b399ce0c7f84` — feat(resume): 선택 프로젝트 경력 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** resume가 명시한 project IDs를 case-study evidence로 연결합니다.

#### Exact inspection targets

- `src/app/resume/page.tsx` — selected-project section
- `getResumeProjects(content)` 호출
- `content.resume.projectIds`와 `content.projects`
- `ProjectCard` 또는 detail link에 `homeTemplate`/`contentDebug` 전달

#### Commit-specific investigation tasks

1. `getResumeProjects` 입력과 결과를 따라 `resume.projectIds` 순서가 보존되는지 확인합니다.
2. 누락 project ID가 selector 결과에서 어떻게 처리되는지 exact selector/consumer state를 비교합니다.
3. project cards/links에 template/debug state가 전달되는지 기록합니다.

#### Learner evidence record

- **Previous state:** Resume summary는 역량을 주장했지만 구체적인 프로젝트 근거로 이동할 수 없었습니다.
- **Implementation decision and path:** 공용 selector가 resume의 project ID 목록을 aggregate에 해석한 결과를 route가 카드/링크로 렌더링합니다.
- **Ownership and data lifetime:** `resume.projectIds`가 선택과 순서를, project aggregate가 표시 데이터와 route ID를, selector가 누락 참조 제거를 소유합니다.
- **Failure, absence, and non-guarantee:** project ID가 해석되지 않으면 결과에서 빠지고, 선택 결과가 비면 item 목록이 비어 별도 fallback project를 고르지 않습니다.
- **Resulting guarantee:** Resume의 사례 선택이 global featured flag가 아니라 resume 전용 ID 목록을 따르고 원본 순서를 보존함을 보장합니다.
- **Relationship to later work:** `4d17...`은 프로젝트와 별개의 training evidence를 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `b399ce0c7f84` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `4d17fd7ab81b` — feat(resume): 교육 과정 요약 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** resume training records와 Resume navigation entry를 추가합니다.

#### Exact inspection targets

- `src/app/resume/page.tsx` — training section
- `content.resume.training.map(...)`
- training item의 `name`, `period`, `description`
- `src/content/site.json` — `/resume` navigation item

#### Commit-specific investigation tasks

1. `content.resume.training` item fields와 card 반복을 확인하고 section에 length guard가 있는지 기록합니다.
2. `src/content/site.json`의 `/resume` navigation 추가를 route renderer change와 분리합니다.
3. training과 later education history가 같은 source인지 다른 source인지 구분합니다.

#### Learner evidence record

- **Previous state:** 선택 프로젝트는 있었지만 formal/structured training의 기간과 설명을 보여 주지 못했고 site navigation에도 Resume 진입점이 없었습니다.
- **Implementation decision and path:** training records를 카드 목록으로 출력하고 site navigation에 Resume route를 추가합니다.
- **Ownership and data lifetime:** training data가 item 순서와 문구를, site content가 navigation 노출을, route가 card presentation을 소유합니다.
- **Failure, absence, and non-guarantee:** training 배열이 비면 이 커밋의 구현상 section wrapper와 heading은 남을 수 있습니다. 후대 experience/education/notes의 length guard를 이 과거 상태에 소급하지 않습니다.
- **Resulting guarantee:** training records와 `/resume` 진입 경로를 제공함을 보장합니다. 비어 있는 training section의 완전한 생략은 보장하지 않습니다.
- **Relationship to later work:** `f647...`이 profile metadata를 hero/evidence에 더하고 `3c113...`부터 별도 조건부 sections를 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `4d17fd7ab81b` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `f64763cffcff` — feat(resume): 프로필 위치와 가용성 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** profile의 location과 availability를 Resume identity evidence에 통합합니다.

#### Exact inspection targets

- `src/app/resume/page.tsx` — profile metadata 영역
- `content.profile.location`, `content.profile.availability`
- `ContentHint`가 가리키는 `src/content/profile.json`
- 값 존재 여부에 따른 separator/DOM 처리

#### Commit-specific investigation tasks

1. profile location/availability가 Resume의 어느 위치에 삽입되고 어떤 source hint를 갖는지 확인합니다.
2. 두 값의 optional branch와 separator 처리를 exact JSX에서 기록합니다.
3. resume JSON에 값을 복제하지 않고 profile aggregate를 재사용하는지 확인합니다.

#### Learner evidence record

- **Previous state:** Resume hero는 이름과 handle만 사용해 현재 위치와 가용성 정보를 전달하지 못했습니다.
- **Implementation decision and path:** profile metadata를 Resume 상단에 추가해 resume summary와 현재 상태를 함께 보여 줍니다.
- **Ownership and data lifetime:** profile content가 location/availability 값을, Resume route가 표시 위치와 문장 조합을 소유합니다.
- **Failure, absence, and non-guarantee:** 값이 없을 때 임의 문구를 만들지 않습니다. 두 값의 독립적 optional 처리 방식은 해당 SHA의 JSX 조건을 기준으로 기록해야 하며 final HEAD를 소급하지 않습니다.
- **Resulting guarantee:** Resume가 별도 중복 데이터 없이 profile aggregate의 현재 상태를 재사용함을 보장합니다.
- **Relationship to later work:** `3c113...`이 work experience를 독립 section으로 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `f64763cffcff` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `3c113f1995ff` — feat(resume): 경력 이력 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** experience history를 데이터가 있을 때만 나타나는 Resume section으로 추가합니다.

#### Exact inspection targets

- `src/app/resume/page.tsx` — experience length guard와 item 반복
- `content.experience`
- `presentation.pages.resume.experience`
- company/role/period/highlights 표현 위치

#### Commit-specific investigation tasks

1. experience 배열 length guard가 heading과 item wrapper 전체를 감싸는지 확인합니다.
2. experience item의 role/company/period/highlights 표현과 ordering을 기록합니다.
3. experience 부재가 training/summary sections에 영향을 주지 않는지 JSX sibling 구조로 확인합니다.

#### Learner evidence record

- **Previous state:** Resume는 projects와 training은 보여 줬지만 실제 조직·역할 단위의 경력 기록을 별도 section으로 제공하지 않았습니다.
- **Implementation decision and path:** `content.experience.length > 0` 조건 아래 heading과 records를 함께 렌더링합니다.
- **Ownership and data lifetime:** experience content가 records와 순서를, route가 section의 존재·layout을, presentation이 label을 소유합니다.
- **Failure, absence, and non-guarantee:** experience가 비면 item뿐 아니라 section 전체가 없습니다. training이나 summary에는 영향을 주지 않습니다.
- **Resulting guarantee:** 빈 경력 heading을 남기지 않는 독립적 conditional-section 계약을 보장합니다.
- **Relationship to later work:** `732...`이 같은 원칙으로 education을 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `3c113f1995ff` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.6. `732ae5a6785c` — feat(resume): 교육 이력 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** education history를 별도의 조건부 Resume section으로 추가합니다.

#### Exact inspection targets

- `src/app/resume/page.tsx` — education length guard
- `content.resume.education` 또는 해당 education source
- `presentation.pages.resume.education`
- institution/program/period/description 반복

#### Commit-specific investigation tasks

1. education 배열 length guard와 item fields를 exact SHA에서 확인합니다.
2. training section과 education section의 source/labels/semantics 차이를 기록합니다.
3. experience 존재 여부와 독립적으로 education visibility가 결정되는지 확인합니다.

#### Learner evidence record

- **Previous state:** training 요약은 있었지만 education history를 별도 이력으로 구분해 표현하지 않았습니다.
- **Implementation decision and path:** education 배열이 비어 있지 않을 때만 heading과 education cards/rows를 렌더링합니다.
- **Ownership and data lifetime:** education content가 항목과 순서를, route가 section 생략 여부와 표현을 소유합니다.
- **Failure, absence, and non-guarantee:** education이 비면 section 전체가 없습니다. experience가 존재하는지와 무관하게 독립적으로 평가됩니다.
- **Resulting guarantee:** training과 education을 같은 의미로 합치지 않고 독립 evidence sections로 유지함을 보장합니다.
- **Relationship to later work:** `579...`가 마지막으로 작성자 안내 notes를 별도 조건부 section으로 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `732ae5a6785c` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.7. `579ea168daa8` — feat(resume): Resume 안내 기록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** resume notes를 데이터가 있을 때만 공개하는 마지막 안내 section으로 추가합니다.

#### Exact inspection targets

- `src/app/resume/page.tsx` — notes length guard와 list
- `content.resume.notes`
- `presentation.pages.resume.notes`
- notes item의 key와 semantic list 구조

#### Commit-specific investigation tasks

1. resume notes length guard가 heading과 list 전체를 감싸는지 확인합니다.
2. notes가 validation error가 아니라 사용자 안내 content로 표현되는 semantic 위치를 기록합니다.
3. notes 부재가 다른 Resume evidence sections를 숨기지 않는지 확인합니다.

#### Learner evidence record

- **Previous state:** Resume의 사실 evidence는 있었지만 문서 사용·해석에 필요한 편집자 notes를 별도로 전달할 수 없었습니다.
- **Implementation decision and path:** notes 배열이 비어 있지 않을 때만 heading과 notes 목록을 추가합니다.
- **Ownership and data lifetime:** resume content가 notes 문구와 순서를, route가 section visibility와 list semantics를 소유합니다.
- **Failure, absence, and non-guarantee:** notes가 비면 section 전체가 없습니다. notes는 validation warning이나 runtime error가 아니며 다른 evidence를 차단하지 않습니다.
- **Resulting guarantee:** 선택적 안내를 빈 UI 없이 추가할 수 있고, summary/project/training/experience/education의 존재와 독립적임을 보장합니다.
- **Relationship to later work:** 후대 Resume view model은 category 09에서 이 파생·조건 데이터를 renderer boundary로 옮깁니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `579ea168daa8` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| 최소 Resume | `e655...`가 hero·summary와 truthy `downloadUrl` 기반 CTA를 만들었습니다. |
| 사례 근거 | `b399...`가 resume 전용 project ID 순서에 따라 case-study evidence를 연결했습니다. |
| 교육·현재 상태 | `4d17...`과 `f647...`이 training, location, availability를 추가했습니다. |
| 독립 조건부 sections | `3c113...`, `732...`, `579...`가 experience, education, notes를 각각 배열 길이로 가드했습니다. |

## 7. Failure → Fix → Test and later relationships

- 이 frozen map에는 전용 fix/test commit이 없습니다. project ID 참조와 conditional section은 정적 구현으로 확인했으며 실행 회귀를 주장하지 않습니다.
- `4d17...`의 training section과 후대 sections의 length guard는 같은 시점의 계약이 아닙니다. 후대 guard를 training에 소급하지 않습니다.
- `getResumeProjects`의 selector 구현·테스트는 category 01/09에서 소유하고, 이 문서는 Resume가 그 결과를 어떻게 소비하는지에 집중합니다.

## 8. Ownership, state, and responsibility changes

- Profile content: identity, location, availability.
- Resume content: summary, download URL, selected project IDs, training, education, notes.
- Experience content: work-history records.
- Selectors: resume project IDs를 project objects로 해석하며 원본 순서를 보존.
- Resume route: section existence, ordering, links, semantic presentation.

## 9. Final thread state

Resume route는 content-provided download URL만 CTA로 공개하고, summary·선택 프로젝트·training·profile metadata를 보여 줍니다. experience, education, notes는 각각 데이터가 있을 때만 section 전체가 존재하며 서로 독립적입니다.

## 10. Final architecture and execution flow

content aggregate 로드 → Resume hero/profile metadata 구성 → `downloadUrl` truthiness로 CTA 결정 → summary 렌더링 → resume project IDs를 selector로 해석하고 찾은 사례를 원본 순서로 출력 → training 렌더링 → experience/education/notes 배열을 각각 검사해 non-empty section만 추가.

## 11. Minimal historical code evidence

`e655951b0706`, `src/app/resume/page.tsx`, `ResumePage`:
```tsx
{content.resume.downloadUrl ? (
  <a href={content.resume.downloadUrl}>{pageCopy.hero.downloadLabel}</a>
) : null}
```
이 발췌는 route가 추정 경로를 만들지 않고 content가 제공한 다운로드 가능성만 공개하는 계약을 보여 줍니다.

## 12. Learning-completion checks

- [x] Frozen commit map 7개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 05-resume-evidence-and-conditional-sections.md =====

===== BEGIN FILE: 06-contact-preference-fallback-and-empty-state.md =====
# Development Thread: Contact preference, fallback, and empty state

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: `/contact` hero, availability/notes, preferred contact links, placement selector integration, accessible link target/copy, no-link empty state.
- 제외: 외부 링크의 보안 속성·transport는 category 03/06, selector의 전체 cross-route 정책은 category 01, route view-model ownership과 renderer matrix는 category 09가 담당합니다.
- `bc651...`은 여러 공용 UI를 함께 바꾸지만 Contact의 링크 target/copy 계약에 직접 연결되는 integration commit으로만 다룹니다.
- Phase 1에서 draft의 `9e99...` → `bc651...` 순서를 실제 이력인 `bc651...` → `9e99...`로 교정했습니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 5
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

Contact route가 소개 화면에서 preferred-link 선택, placement vocabulary, 공용 UI copy, 명시적 empty state로 발전하는 과정을 실제 순서대로 복원하고, 링크가 없을 때 깨진 CTA나 침묵하는 빈 영역을 만들지 않는 경계를 확인합니다.

### Fixed invariants

- Contact는 `getPreferredContactLinks(content)`가 반환한 순서와 활성 결과를 사용하며 route가 임의로 social link를 고르지 않습니다.
- 링크 placement는 typed `LinkPlacement` vocabulary로 표현되며 contact용 전역 링크와 project card/detail 링크는 같은 helper를 오용하지 않습니다.
- preferred links가 하나 이상이면 링크 목록을, 0개이면 `presentation.ui.emptyStates.contactLinks`를 렌더링합니다.
- 링크가 없어도 availability/notes와 Contact route 자체는 유지되며 없는 URL을 합성하지 않습니다.
- 링크 CTA의 최소 터치 target과 UI 문구는 presentation/shared UI 계약에서 오며 Contact 전용 하드코딩 fallback이 아닙니다.

## 2. Core engineering questions

1. 초기 Contact route는 어떤 profile/contact 필드를 보여 주고 아직 무엇을 제공하지 않는가?
2. `getPreferredContactLinks`의 결과가 route에서 어떤 순서로 렌더링되며 fallback source는 무엇인가?
3. `LinkPlacement`와 `getContentLinksByPlacement`가 기존 project-link helper를 어떻게 일반화하는가?
4. `bc651...`이 공용 UI copy와 target size를 먼저 제공한 뒤 `9e99...`이 그 copy를 어떤 branch에서 소비하는가?
5. preferred links가 0개일 때 availability와 notes는 남고 링크 영역만 empty state로 전환되는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `bfcdf44eb34c` | feat(contact): 연락 페이지 소개 추가 | B | RENDERER | Contact route의 profile identity, title, intro를 최초 구성합니다. |
| 2 | `f344a492043c` | feat(contact): 선호 연락 수단과 안내 추가 | B | RENDERER | preferred links, availability, notes를 Contact body에 연결합니다. |
| 3 | `119ff9a92090` | feat(content): 링크 배치 selector 추가 | B | CONTENT | 전역·프로젝트 링크를 typed placement로 선택할 수 있는 selector vocabulary를 도입합니다. |
| 4 | `bc651dd85e14` | feat(content): 공용 UI 접근성 문구 적용 | B | CONTENT, A11Y | 공용 UI labels와 최소 interactive target을 content-driven 계약으로 적용합니다. |
| 5 | `9e99c9531cb8` | feat(contact): 연락 링크 빈 상태 추가 | B | RENDERER | preferred-link 결과가 0개일 때 명시적 Contact empty state를 렌더링합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `bfcdf44eb34c` — feat(contact): 연락 페이지 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** Contact route의 profile identity, title, intro를 최초 구성합니다.

#### Exact inspection targets

- `src/app/contact/page.tsx` — `ContactPage`
- `content.contact.title`, `content.contact.intro`
- `content.profile.name`, `content.profile.handle`
- `PageShell`과 template switcher의 `/contact` current path

#### Commit-specific investigation tasks

1. `ContactPage`의 최소 hero가 contact title/intro와 profile identity를 어떻게 결합하는지 확인합니다.
2. shell/template switcher의 currentPath가 `/contact`로 설정되는지 기록합니다.
3. preferred links, availability, notes가 아직 없는 이전 state를 명시합니다.

#### Learner evidence record

- **Previous state:** 직전에는 연락 목적과 작성자 identity를 실제 `/contact` route에서 보여 주는 화면이 없었습니다.
- **Implementation decision and path:** `ContactPage`가 aggregate와 query state를 읽어 shell 안에 profile identity, contact title, intro를 렌더링합니다.
- **Ownership and data lifetime:** contact content가 title/intro를, profile이 identity를, route가 hero composition을 소유합니다.
- **Failure, absence, and non-guarantee:** 이 시점에는 preferred links, availability, notes, empty state가 없습니다. 소개 데이터가 비어 있을 때 별도 validation/fallback을 제공하지 않습니다.
- **Resulting guarantee:** Contact route의 최소 정보 구조와 route-preserving shell을 보장합니다. 실제 연락 가능한 수단은 아직 보장하지 않습니다.
- **Relationship to later work:** `f344...`가 선호 연락 수단과 안내를 실제 CTA 영역으로 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `bfcdf44eb34c` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `f344a492043c` — feat(contact): 선호 연락 수단과 안내 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** preferred links, availability, notes를 Contact body에 연결합니다.

#### Exact inspection targets

- `src/app/contact/page.tsx` — `preferredLinks`, links section, availability/notes 영역
- `getPreferredContactLinks(content)`
- `ContentLinkView` 반복과 `homeTemplate`/`contentDebug` 전달
- `content.contact.availability`, contact notes 관련 필드

#### Commit-specific investigation tasks

1. `getPreferredContactLinks(content)` 호출 결과가 `ContentLinkView` cards로 전달되는 흐름을 추적합니다.
2. availability와 notes가 links list와 어느 sibling 위치에 있는지 확인합니다.
3. preferred result가 0개일 때 별도 branch가 없이 map만 비는지 기록합니다.

#### Learner evidence record

- **Previous state:** Contact route는 목적만 설명하고 실제 연락 링크나 응답 가능 상태를 제공하지 않았습니다.
- **Implementation decision and path:** selector 결과를 CTA cards로 반복하고 availability와 notes를 같은 route에 추가합니다.
- **Ownership and data lifetime:** 선호 링크 계산·순서는 selector가, availability/notes 원본은 contact content가, card layout은 route가 소유합니다.
- **Failure, absence, and non-guarantee:** preferred links가 비면 이 시점에는 단순히 빈 반복 결과가 되어 링크 영역에 설명이 없습니다. URL을 임의로 합성하지는 않습니다.
- **Resulting guarantee:** 활성 preferred links와 안내 정보를 content 기반으로 표현함을 보장합니다. no-link 상태의 명시적 설명은 아직 없습니다.
- **Relationship to later work:** `119...`이 placement vocabulary를 일반화하고, `9e99...`이 0개 결과를 explicit empty state로 고칩니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `f344a492043c` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `119ff9a92090` — feat(content): 링크 배치 selector 추가

- **Importance:** B
- **Tags:** CONTENT
- **Source-defined role:** 전역·프로젝트 링크를 typed placement로 선택할 수 있는 selector vocabulary를 도입합니다.

#### Exact inspection targets

- `src/lib/portfolio/types.ts` — `LinkPlacement`, `ContentLink.placements`
- `src/lib/portfolio/selectors.ts` — `getContentLinksByPlacement`, `getProjectLinksForPlacement`
- `getProjectCardLinks`, `getProjectDetailLinks` wrapper
- `src/lib/portfolio.ts` — public exports
- deployment 상태에 따른 project-link filter branch

#### Commit-specific investigation tasks

1. `LinkPlacement` union과 global/project placement selectors의 signatures를 비교합니다.
2. `getProjectCardLinks`/`getProjectDetailLinks` wrappers가 generic helper에 어떤 placement를 고정하는지 확인합니다.
3. placement filter 뒤 project deployment/live filter가 적용되는 순서와 fallback 부재를 기록합니다.

#### Learner evidence record

- **Previous state:** 링크 placement literal과 card 전용 filter가 흩어져 있어 contact/detail/footer 등 다른 소비자가 같은 vocabulary를 안전하게 재사용하기 어려웠습니다.
- **Implementation decision and path:** `LinkPlacement` union을 도입하고 전역 content links와 project links를 placement별로 선택하는 generic helpers를 추가합니다. 기존 card helper는 generic project helper를 호출합니다.
- **Ownership and data lifetime:** content의 `placements` 배열이 노출 위치를, selectors가 placement와 deployment 상태 filter를, route/component가 표현을 소유합니다.
- **Failure, absence, and non-guarantee:** placement가 없거나 요청 placement를 포함하지 않는 링크는 결과에서 제외됩니다. project link는 live/deployment 정책도 통과해야 하며, selector는 fallback URL을 만들지 않습니다.
- **Resulting guarantee:** Contact를 포함한 consumers가 문자열 ad-hoc filter 대신 같은 typed placement contract를 사용할 수 있음을 보장합니다.
- **Relationship to later work:** `bc651...`이 공용 문구/target 계약을 추가하고 `9e99...`이 Contact의 zero-result branch를 명시합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `119ff9a92090` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `bc651dd85e14` — feat(content): 공용 UI 접근성 문구 적용

- **Importance:** B
- **Tags:** CONTENT, A11Y
- **Source-defined role:** 공용 UI labels와 최소 interactive target을 content-driven 계약으로 적용합니다.

#### Exact inspection targets

- `src/content/presentation.json` 및 presentation type/schema — shared UI labels/empty-state copy
- `src/components/portfolio/project-links.tsx` — `min-h-11` link targets
- `src/components/portfolio/journey-list.tsx` — content-provided case-study label
- `src/components/portfolio/animated-terminal.tsx`, `tech-marquee.tsx` — injected aria labels
- Contact empty-state copy가 저장되는 `presentation.ui.emptyStates.contactLinks`

#### Commit-specific investigation tasks

1. presentation UI copy/aria labels가 어떤 components에 prop으로 주입되는지 파일별로 분리합니다.
2. `project-links.tsx`의 `h-9`→`min-h-11` 변경이 어떤 interactive targets에 적용되는지 확인합니다.
3. Contact empty-state copy가 content vocabulary에 존재하지만 Contact branch는 아직 추가되지 않았는지 chronology를 확인합니다.

#### Learner evidence record

- **Previous state:** 여러 공용 component가 영어 label·aria text·고정 높이를 자체 소유해 presentation content와 접근성 target 계약이 분산돼 있었습니다.
- **Implementation decision and path:** UI labels를 presentation content에서 주입하고 일부 link buttons를 최소 높이 44px 상당의 `min-h-11`로 확장합니다. Contact의 후속 no-link branch가 사용할 shared empty-state copy도 content vocabulary에 포함됩니다.
- **Ownership and data lifetime:** presentation content가 사용자 노출/aria copy를, shared components가 semantic target과 적용 위치를, route가 어떤 state에서 copy를 보여 줄지 소유합니다.
- **Failure, absence, and non-guarantee:** 이 커밋 자체가 Contact의 `preferredLinks.length === 0` branch를 추가하지는 않습니다. copy가 존재해도 consumer가 사용하지 않으면 화면에 나타나지 않습니다.
- **Resulting guarantee:** 공용 labels를 하드코딩하지 않고 interactive targets의 최소 크기를 강화할 기반을 보장합니다.
- **Relationship to later work:** `9e99...`이 준비된 Contact empty-state copy를 실제 zero-link branch에 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `bc651dd85e14` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `9e99c9531cb8` — feat(contact): 연락 링크 빈 상태 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** preferred-link 결과가 0개일 때 명시적 Contact empty state를 렌더링합니다.

#### Exact inspection targets

- `src/app/contact/page.tsx` — `preferredLinks.length > 0` branch
- `ContentLinkView` card의 `min-h-11`
- `content.presentation.ui.emptyStates.contactLinks`
- availability/notes block이 branch 밖에 남는지 확인

#### Commit-specific investigation tasks

1. `preferredLinks.length > 0`의 두 branches를 따라 list와 empty-state DOM을 비교합니다.
2. empty-state 문구가 `presentation.ui.emptyStates.contactLinks`에서 오는지 확인합니다.
3. availability/notes가 branch 밖에 남고 link cards가 `min-h-11`을 사용하는지 기록합니다.

#### Learner evidence record

- **Previous state:** selector가 0개를 반환하면 링크 반복이 아무 DOM도 만들지 않아 사용자는 데이터 부재와 렌더링 실패를 구분할 수 없었습니다.
- **Implementation decision and path:** non-empty 결과는 링크 cards로 렌더링하고, empty 결과는 dashed card 안에 shared presentation copy를 표시합니다.
- **Ownership and data lifetime:** selector가 상태를 계산하고, Contact route가 list-vs-empty 전환을, presentation content가 설명 문구를 소유합니다.
- **Failure, absence, and non-guarantee:** 링크가 없어도 availability/notes와 page shell은 유지됩니다. 이 branch는 연락 수단을 자동 생성하거나 retry하지 않습니다.
- **Resulting guarantee:** Contact가 0개 링크를 침묵하는 빈 영역으로 처리하지 않고 명시적 비가용 상태로 보여 줌을 보장합니다.
- **Relationship to later work:** 후대 Contact view model과 renderer matrix 검증은 category 09/07에서 같은 결과 계약을 보호합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `9e99c9531cb8` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| 소개 route | `bfc...`가 Contact identity와 intro만 제공했습니다. |
| 실제 수단 | `f344...`가 preferred links와 availability/notes를 연결했지만 zero-result 설명은 없었습니다. |
| 선택 vocabulary | `119...`이 link placement와 project/global selector 경계를 typed contract로 정리했습니다. |
| 공용 접근성 기반 | `bc651...`이 shared copy/aria labels와 최소 target을 먼저 제공했습니다. |
| 명시적 absence | `9e99...`이 preferred link 0개를 presentation-owned empty state로 전환했습니다. |

## 7. Failure → Fix → Test and later relationships

- Failure → Fix: `f344...`의 empty iteration은 오류를 던지지 않지만 사용자에게 상태를 설명하지 못했습니다. `9e99...`이 0개 결과를 explicit branch로 보정했습니다.
- Integration dependency: `bc651...`이 empty-state copy를 content vocabulary에 둔 뒤 `9e99...`이 실제 Contact consumer에서 사용합니다. 실제 시간 순서는 이 순서입니다.
- 전용 regression test는 frozen map에 없습니다. route-view-model/renderer matrix 및 broad accessibility tests는 category 09/07이 담당합니다.

## 8. Ownership, state, and responsibility changes

- Contact content: title, intro, availability, notes와 선호 연락 설정.
- Link selectors: enabled/placement/deployment 기반 결과와 순서.
- Presentation UI: empty-state/aria/label copy.
- Contact route: preferred list와 empty-state branch, availability/notes 배치.
- ContentLinkView/shared link components: 외부 링크 semantics와 interactive target 표현.

## 9. Final thread state

Contact route는 preferred selector 결과가 있으면 순서대로 접근 가능한 링크 cards를 보여 주고, 없으면 presentation-owned empty-state 문구를 보여 줍니다. availability와 notes는 두 branch 모두에서 유지되며 없는 링크를 합성하지 않습니다.

## 10. Final architecture and execution flow

content aggregate 로드 → Contact hero 구성 → preferred selector가 enabled/placement 정책으로 links 계산 → 결과 길이 검사 → non-empty이면 `ContentLinkView` cards 렌더링, empty이면 shared empty-state copy 렌더링 → branch와 무관하게 availability/notes 제공.

## 11. Minimal historical code evidence

`9e99c9531cb8`, `src/app/contact/page.tsx`, `ContactPage`:
```tsx
{preferredLinks.length > 0 ? (
  preferredLinks.map((link) => <ContentLinkView key={link.id ?? link.href} link={link} />)
) : (
  <p>{content.presentation.ui.emptyStates.contactLinks}</p>
)}
```
이 발췌는 selector의 0개 결과를 별도 상태로 표현하되 URL을 만들어 내지 않는 수정 계약을 보여 줍니다.

## 12. Learning-completion checks

- [x] Frozen commit map 5개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 06-contact-preference-fallback-and-empty-state.md =====

===== BEGIN FILE: 07-journey-narrative-milestones-and-timeline.md =====
# Development Thread: Journey narrative, milestones, and timeline

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: `/journey` page enablement, narrative intro, milestone decision record, anchor-project resolution, 전체 `journey` timeline, current-position summary.
- 제외: `JourneyList`의 paired/compact responsive primitive는 category 03, page lifecycle/query state는 category 02, route projection과 resolved-reference ownership은 category 09가 담당합니다.
- 이 Thread는 milestone reference의 소비자-side drop 정책을 다루며 schema ingestion의 fail-closed 보장은 category 01/09에 남깁니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 6
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

Journey route가 feature-flagged 소개에서 결정 milestone, state/reason/result 근거, anchor projects, 전체 chronology, 현재 방향으로 확장되는 과정을 복원하고, narrative와 timeline이 서로 다른 원본과 책임을 갖는 이유를 확인합니다.

### Fixed invariants

- `isSitePageEnabled("journey", content)`가 거짓이면 route는 partial page를 보여 주지 않고 `notFound()`로 종료합니다.
- `journeyNarrative.milestones`는 결정의 상태·이유·결과와 anchor project를 설명하고, `journey`는 더 넓은 전체 시간축을 설명합니다.
- milestone의 `anchorProjectIds`는 선언 순서대로 해석하며 찾지 못한 project는 링크로 만들지 않습니다.
- anchor project가 하나도 없어도 milestone의 date/title/state/reason/result는 유지됩니다.
- current-position summary는 과거 chronology와 구분된 현재 방향이며 timeline item으로 임의 삽입되지 않습니다.

## 2. Core engineering questions

1. Journey page flag가 꺼져 있을 때 route는 어떤 control flow로 종료되는가?
2. 초기 milestone은 어떤 최소 필드를 갖고 `e292...`에서 state/reason/result가 어떻게 추가되는가?
3. anchor project 참조를 해석할 수 없을 때 milestone card 자체가 사라지는가, links만 사라지는가?
4. `journeyNarrative.milestones`와 `content.journey`는 왜 하나의 배열로 합쳐지지 않는가?
5. 현재 방향 summary가 chronology와 어떤 별도 presentation copy/data를 소비하는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `0facaf123f29` | feat(journey): 여정 route 소개 추가 | B | ROUTING, RENDERER | feature-flagged Journey route와 narrative hero를 최초 구성합니다. |
| 2 | `fa94b86ac46e` | feat(journey): 결정 milestone 목록 추가 | B | RENDERER | journey narrative의 milestone 순서를 numbered decision list로 추가합니다. |
| 3 | `e292451824ba` | feat(journey): milestone 결정 근거 추가 | B | RENDERER | 각 milestone에 state, reason, result를 추가해 단순 chronology를 decision narrative로 확장합니다. |
| 4 | `ba8130c82d16` | feat(journey): milestone 프로젝트 근거 연결 | B | RENDERER | milestone anchor project IDs를 해석해 template-preserving case-study links를 추가합니다. |
| 5 | `a1136f34f998` | feat(journey): 전체 여정 타임라인 추가 | B | RENDERER | 결정 milestones 아래에 전체 `journey` chronology를 별도 timeline으로 추가합니다. |
| 6 | `694cf57b0162` | feat(journey): 현재 방향 요약 추가 | B | RENDERER | journey narrative의 current-position 정보를 별도 closing section으로 추가합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `0facaf123f29` — feat(journey): 여정 route 소개 추가

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Source-defined role:** feature-flagged Journey route와 narrative hero를 최초 구성합니다.

#### Exact inspection targets

- `src/app/journey/page.tsx` — `JourneyPage`
- `isSitePageEnabled("journey", content)`와 `notFound()`
- `content.journeyNarrative.intro`
- `presentation.pages.journey.hero`
- `PageShell` current path `/journey`

#### Commit-specific investigation tasks

1. content load 직후 `isSitePageEnabled("journey", content)`와 `notFound()`의 실행 순서를 확인합니다.
2. enabled branch의 hero가 presentation copy와 `journeyNarrative.intro`를 어떻게 결합하는지 추적합니다.
3. disabled 상태에서 shell/partial content가 생성되지 않는지 control flow로 기록합니다.

#### Learner evidence record

- **Previous state:** 직전에는 journey content가 있어도 독립 `/journey` route와 page enablement 경계가 없었습니다.
- **Implementation decision and path:** content aggregate를 로드한 직후 journey page flag를 검사하고, disabled이면 `notFound()`로 종료합니다. enabled이면 narrative intro를 hero에 렌더링합니다.
- **Ownership and data lifetime:** site page configuration이 route 존재를, journey narrative가 intro를, route가 not-found transition과 hero composition을 소유합니다.
- **Failure, absence, and non-guarantee:** disabled 상태에서는 shell이나 partial content를 렌더링하지 않습니다. enabled지만 intro가 빈 경우를 위한 별도 fallback copy는 없습니다.
- **Resulting guarantee:** 비활성 Journey가 공개되지 않는 fail-closed route boundary와 최소 narrative hero를 보장합니다.
- **Relationship to later work:** `fa94...`가 hero 아래에 결정 milestone 목록을 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `0facaf123f29` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `fa94b86ac46e` — feat(journey): 결정 milestone 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** journey narrative의 milestone 순서를 numbered decision list로 추가합니다.

#### Exact inspection targets

- `src/app/journey/page.tsx` — milestone section과 `MilestoneCard` 초기 형태
- `content.journeyNarrative.milestones`
- milestone `date`, `title` 및 초기 summary fields
- `presentation.pages.journey.narrative` heading/labels

#### Commit-specific investigation tasks

1. milestones의 content order, numbering, date/title fields를 `MilestoneCard` 초기 형태에서 확인합니다.
2. state/reason/result와 project links가 아직 없는지 exact file로 확인합니다.
3. 빈 milestones에서 section heading/list의 실제 DOM 상태를 기록합니다.

#### Learner evidence record

- **Previous state:** Journey hero는 전체 설명만 제공해 실제 전환점의 순서와 개별 결정을 구분하지 못했습니다.
- **Implementation decision and path:** narrative milestones를 content 순서대로 반복하고 각 항목에 번호·날짜·제목과 당시 제공된 설명을 표시합니다.
- **Ownership and data lifetime:** milestone 배열이 순서를, presentation이 labels를, route/card가 번호와 card hierarchy를 소유합니다.
- **Failure, absence, and non-guarantee:** milestones가 비면 section heading 아래 항목이 없으며 별도 empty state는 없습니다. project evidence나 state/reason/result 구조는 아직 완성되지 않았습니다.
- **Resulting guarantee:** 결정 전환점을 narrative 순서대로 탐색할 수 있음을 보장합니다.
- **Relationship to later work:** `e292...`가 각 milestone을 상태 변화와 근거·결과를 가진 decision record로 확장합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `fa94b86ac46e` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `e292451824ba` — feat(journey): milestone 결정 근거 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 각 milestone에 state, reason, result를 추가해 단순 chronology를 decision narrative로 확장합니다.

#### Exact inspection targets

- `src/app/journey/page.tsx` — `MilestoneCard` description list
- `JourneyMilestone`의 `state`, `reason`, `result`
- `presentation.pages.journey.narrative.labels`
- `<dl>`, `<dt>`, `<dd>` semantic structure

#### Commit-specific investigation tasks

1. milestone의 state/reason/result가 `<dl>/<dt>/<dd>`에 어떤 labels로 대응되는지 표로 작성합니다.
2. 세 fields의 순서와 content source를 확인합니다.
3. 빈 값에 대해 renderer가 자동 문구를 생성하는지 확인합니다.

#### Learner evidence record

- **Previous state:** 날짜와 제목만으로는 무엇이 바뀌었고 왜 바뀌었으며 결과가 무엇인지 복원할 수 없었습니다.
- **Implementation decision and path:** milestone card에 state/reason/result를 label-value 형태로 렌더링해 이전 상태에서 의사결정과 결과로 이어지는 인과를 명시합니다.
- **Ownership and data lifetime:** journey narrative content가 세 설명을, presentation이 labels를, card가 semantic pairing과 layout을 소유합니다.
- **Failure, absence, and non-guarantee:** 해당 문자열이 비어 있을 때 route가 자동 설명을 생성하지 않습니다. 실제 project evidence는 아직 연결되지 않습니다.
- **Resulting guarantee:** 각 milestone이 단순 이력 날짜가 아니라 state → reason → result를 가진 결정 기록임을 보장합니다.
- **Relationship to later work:** `ba813...`이 결정 설명을 실제 anchor project case studies와 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `e292451824ba` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `ba8130c82d16` — feat(journey): milestone 프로젝트 근거 연결

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** milestone anchor project IDs를 해석해 template-preserving case-study links를 추가합니다.

#### Exact inspection targets

- `src/app/journey/page.tsx` — `MilestoneCard`
- `milestone.anchorProjectIds.map(...find...).filter(Boolean)`
- `getTemplateHref(`/projects/${project.id}`, homeTemplate, { contentDebug })`
- `PortfolioContent.projects`와 `JourneyMilestone`

#### Commit-specific investigation tasks

1. `anchorProjectIds.map(find).filter(Boolean)` lookup과 link list guard를 추적합니다.
2. 모든 anchors가 누락돼도 milestone state/reason/result가 남는지 확인합니다.
3. `getTemplateHref`가 current template/debug state를 project links에 보존하는지 기록합니다.

#### Learner evidence record

- **Previous state:** state/reason/result는 설명은 제공했지만 독자가 그 결정의 구현 근거로 이동할 수 없었습니다.
- **Implementation decision and path:** 각 milestone의 anchor IDs를 project aggregate에서 찾고, 해석 가능한 projects만 detail links로 출력합니다.
- **Ownership and data lifetime:** milestone content가 anchor ID 순서를, project aggregate가 title/route ID를, route helper가 현재 template/debug query 보존을 소유합니다.
- **Failure, absence, and non-guarantee:** 찾지 못한 ID는 `filter(Boolean)`으로 제거됩니다. 결과가 0개여도 milestone 설명은 남고 links list만 `null`입니다.
- **Resulting guarantee:** 누락 참조를 깨진 링크로 만들지 않으면서 결정 narrative와 실제 case study를 연결함을 보장합니다.
- **Relationship to later work:** `a113...`이 narrative milestones와 별개의 전체 journey chronology를 같은 route에 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `ba8130c82d16` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `a1136f34f998` — feat(journey): 전체 여정 타임라인 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 결정 milestones 아래에 전체 `journey` chronology를 별도 timeline으로 추가합니다.

#### Exact inspection targets

- `src/app/journey/page.tsx` — timeline section
- `JourneyList` 호출
- `content.journey`
- `presentation.pages.journey.timeline`
- case-study label과 template/debug props 전달

#### Commit-specific investigation tasks

1. 새 timeline이 `journeyNarrative.milestones`가 아니라 `content.journey`를 `JourneyList`에 전달하는지 확인합니다.
2. milestone section과 full timeline의 heading/source/order를 비교합니다.
3. 두 arrays를 자동 병합·중복 제거하지 않는지 code path를 기록합니다.

#### Learner evidence record

- **Previous state:** milestones는 중요한 전환점만 설명해 전체 프로젝트·학습 chronology를 대체하지 못했습니다.
- **Implementation decision and path:** 별도 timeline heading과 `JourneyList`를 추가하고 더 넓은 `content.journey` 배열을 전달합니다.
- **Ownership and data lifetime:** journey narrative가 선별된 결정 기록을, `content.journey`가 전체 시간축을, 공용 `JourneyList`가 timeline DOM과 project links를 소유합니다.
- **Failure, absence, and non-guarantee:** timeline 배열이 비어도 milestone narrative는 유지됩니다. 두 배열을 자동 병합하거나 중복 제거하지 않습니다.
- **Resulting guarantee:** 중요한 결정 narrative와 포괄적 chronology를 같은 페이지에서 서로 다른 의미로 제공함을 보장합니다.
- **Relationship to later work:** `694...`가 과거 이력과 구분되는 현재 방향 요약을 마지막에 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `a1136f34f998` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.6. `694cf57b0162` — feat(journey): 현재 방향 요약 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** journey narrative의 current-position 정보를 별도 closing section으로 추가합니다.

#### Exact inspection targets

- `src/app/journey/page.tsx` — current-position/current-direction section
- `content.journeyNarrative.currentPosition` 또는 해당 current state field
- `presentation.pages.journey`의 closing labels/copy
- timeline 뒤의 section ordering

#### Commit-specific investigation tasks

1. current-position/current-direction data와 presentation labels가 어느 closing section에 연결되는지 확인합니다.
2. timeline item으로 삽입되지 않고 별도 summary로 남는지 기록합니다.
3. 빈 current state에서 임의 future goal을 생성하는 branch가 없는지 확인합니다.

#### Learner evidence record

- **Previous state:** 페이지가 과거 milestones와 timeline으로 끝나 현재 어떤 방향에 있고 무엇을 다음 기준으로 삼는지 설명하지 못했습니다.
- **Implementation decision and path:** narrative가 제공하는 현재 위치/방향 정보를 timeline과 구분된 closing section에 렌더링합니다.
- **Ownership and data lifetime:** journey narrative content가 현재 상태를, presentation이 framing copy를, route가 마지막 section 배치를 소유합니다.
- **Failure, absence, and non-guarantee:** 현재 방향 값이 비어 있을 때 임의 목표를 생성하지 않습니다. 이 정보는 자동 progress tracker나 future guarantee가 아닙니다.
- **Resulting guarantee:** 과거 chronology와 현재 방향을 혼합하지 않고, 현재 편집 시점의 상태를 별도 claim으로 공개함을 보장합니다.
- **Relationship to later work:** 후대 Journey view model은 category 09에서 anchor references를 미리 해석하고 renderer의 raw lookup을 제거합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `694cf57b0162` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| Route 존재 | `0fac...`이 page flag를 검사해 disabled Journey를 `notFound()`로 닫았습니다. |
| 결정 목록 | `fa94...`가 milestones를 도입하고 `e292...`가 state/reason/result 인과로 확장했습니다. |
| 근거 연결 | `ba813...`이 anchor project IDs를 해석하고 누락 참조는 links에서 제거했습니다. |
| 두 시간축 | `a113...`이 선별 milestone과 별개인 전체 journey timeline을 추가했습니다. |
| 현재 상태 | `694...`가 과거 기록 뒤에 현재 방향을 별도 claim으로 배치했습니다. |

## 7. Failure → Fix → Test and later relationships

- Failure/absence correction: `ba813...`은 누락 anchor reference를 예외로 만들지 않고 해당 link만 제거합니다. milestone narrative는 보존됩니다.
- Milestone list와 full timeline은 중복 구현이 아니라 서로 다른 source와 granularity를 가진 병렬 evidence입니다.
- 전용 fix/test commit은 frozen map에 없습니다. 후대 resolved-reference view model과 renderer-boundary tests는 category 09에서 다룹니다.

## 8. Ownership, state, and responsibility changes

- Site page config: `/journey` 공개 여부.
- Journey narrative: intro, milestones, state/reason/result, anchor IDs, current position.
- Journey array: 전체 chronology와 item 순서.
- Journey route: feature gate, milestone reference resolution, section ordering.
- JourneyList: timeline DOM, case-study link expression, responsive presentation.

## 9. Final thread state

Journey route는 disabled이면 404로 닫히고, enabled이면 narrative intro, 결정 milestones, 해석 가능한 anchor projects, 별도의 전체 chronology, 현재 방향을 순서대로 제공합니다. 누락 anchor는 link만 제거하며 milestone 설명은 남습니다.

## 10. Final architecture and execution flow

content aggregate 로드 → journey page flag 검사 → disabled이면 `notFound()` → enabled이면 narrative hero → milestones 순회 및 state/reason/result 출력 → anchor IDs를 project aggregate에 해석하고 찾은 links만 생성 → 별도 `content.journey` timeline 렌더링 → current-position closing summary.

## 11. Minimal historical code evidence

`ba8130c82d16`, `src/app/journey/page.tsx`, `MilestoneCard`:
```tsx
const anchorProjects = milestone.anchorProjectIds
  .map((projectId) => content.projects.find((project) => project.id === projectId))
  .filter((project): project is NonNullable<typeof project> => Boolean(project));
```
이 발췌는 milestone 자체와 optional project evidence를 분리하고, 누락 참조를 깨진 route로 만들지 않는 정책을 보여 줍니다.

## 12. Learning-completion checks

- [x] Frozen commit map 6개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 07-journey-narrative-milestones-and-timeline.md =====

===== BEGIN FILE: 08-interview-evidence-map-and-gaps.md =====
# Development Thread: Interview evidence map and gaps

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Category:** `04-route-features-and-evidence-experiences`  
> **Workbook state:** completed workbook  
> **Historical scope:** commits reachable from `web/portfolio` only

## 0. Phase 1 audit result and category boundary

- 포함: `/interview-map` page enablement, reference repository, topic anchors, evidence gaps, track summaries, external references/questions, project answer/depth mapping.
- 제외: 외부 링크 security transport는 category 03/06, route lifecycle/query mechanics는 category 02, 후대 Interview view-model과 renderer data ownership은 category 09가 담당합니다.
- 이 Thread에서 `gaps`는 오류나 placeholder가 아니라 의도적으로 공개하는 evidence limitation입니다.

- **Audit decision:** 이 Thread는 독립적인 route feature/evidence story로 유지합니다.
- **Frozen commit count:** 6
- **Importance profile:** branch-local source classification상 이 Thread의 commit은 모두 B입니다. 다른 category의 S/A-level cross-cutting architecture를 중복 편입하지 않았습니다.

## 1. Thread goal

Interview Map route가 feature-flagged 소개에서 topic index, 공개 gaps, track별 외부 참조와 질문, project-backed answers로 확장되는 과정을 복원하고, 근거 부족과 누락 project reference를 숨기지 않는 표현 정책을 확인합니다.

### Fixed invariants

- `isSitePageEnabled("interviewMap", content)`가 거짓이면 partial evidence page를 공개하지 않고 `notFound()`로 종료합니다.
- topic index는 track IDs/labels에 대한 page-local navigation이며 source data의 순서를 따릅니다.
- `gaps`는 숨기거나 자동으로 보충하지 않고 별도 목록으로 공개합니다.
- track의 external reference/question과 project answer/depth는 다른 evidence 열로 유지됩니다.
- answer의 project ID를 찾지 못하면 행을 제거하지 않고 raw ID를 표시해 참조 공백을 드러냅니다.

## 2. Core engineering questions

1. Interview Map page flag가 꺼졌을 때 route는 어디에서 종료되는가?
2. reference repository, topic index, gaps는 각각 어떤 목적을 가지며 동일한 evidence가 아닌 이유는 무엇인가?
3. track section은 item count, intro, external reference, question을 어떤 계층으로 표현하는가?
4. answer project ID lookup을 위해 어떤 map을 만들고, lookup failure를 왜 drop하지 않고 raw ID로 보여 주는가?
5. answer label/title과 depth explanation은 table에서 어떤 별도 열로 유지되는가?

## 3. Completion criteria

- [x] 모든 commit을 부모 상태와 exact SHA에서 비교하고 final HEAD를 과거에 투영하지 않았습니다.
- [x] 각 commit의 concrete file/function/component/data field와 caller→callee 또는 data flow를 기록했습니다.
- [x] optional data, missing reference, empty array, disabled page 등 실제 failure/absence branch를 설명했습니다.
- [x] 소유권·표시 책임·상태 전환과 적용되지 않는 resource cleanup을 구분했습니다.
- [x] 보장과 비보장을 분리하고 후속 commit/category와의 관계를 연결했습니다.
- [x] 실행하지 않은 build/test/runtime 결과를 통과했다고 표시하지 않았습니다.

## 4. Frozen commit map

| Order | SHA | Subject | Importance | Tags | Source-defined role |
|---:|---|---|:---:|---|---|
| 1 | `ddba753f7f51` | feat(interview-map): 근거 route 소개 추가 | B | ROUTING, RENDERER | feature-flagged Interview Map route, intro, reference repository link를 최초 구성합니다. |
| 2 | `cb161d118ddd` | feat(interview-map): 인터뷰 주제 인덱스 추가 | B | RENDERER | track 목록을 page-local topic anchor index로 추가합니다. |
| 3 | `9704aa5f7b59` | feat(interview-map): 근거 공백 목록 추가 | B | RENDERER | 현재 evidence의 부족 영역을 명시적인 gaps 목록으로 공개합니다. |
| 4 | `a3fb132d33f7` | feat(interview-map): 주제 track 소개 추가 | B | RENDERER | 각 interview track의 intro와 item count를 독립 section으로 구성합니다. |
| 5 | `cdcbbc937490` | feat(interview-map): 주제와 외부 참조 표 추가 | B | RENDERER | track items의 topic, external reference, interview question을 표 구조로 추가합니다. |
| 6 | `1cd28c140350` | feat(interview-map): 프로젝트 답변 근거 연결 | B | RENDERER | 각 question의 answers를 project case studies와 depth explanation으로 연결합니다. |

## 5. Commit-by-commit historical investigation

### 5.1. `ddba753f7f51` — feat(interview-map): 근거 route 소개 추가

- **Importance:** B
- **Tags:** ROUTING, RENDERER
- **Source-defined role:** feature-flagged Interview Map route, intro, reference repository link를 최초 구성합니다.

#### Exact inspection targets

- `src/app/interview-map/page.tsx` — `InterviewMapPage`
- `isSitePageEnabled("interviewMap", content)`와 `notFound()`
- `content.interviewMap.intro`, `content.interviewMap.referenceRepo`
- `presentation.pages.interviewMap.hero`
- external anchor의 `target="_blank"`, `rel="noreferrer"`

#### Commit-specific investigation tasks

1. `isSitePageEnabled("interviewMap", content)`와 `notFound()`가 partial page보다 먼저 실행되는지 확인합니다.
2. intro/referenceRepo와 presentation hero copy의 data flow를 추적합니다.
3. external anchor의 target/rel semantics와 fallback URL 부재를 기록합니다.

#### Learner evidence record

- **Previous state:** 직전에는 interview preparation evidence와 reference repository를 독립 route로 공개하는 화면이 없었습니다.
- **Implementation decision and path:** content aggregate 로드 직후 page flag를 검사하고 disabled이면 404로 종료합니다. enabled이면 intro와 reference repository anchor를 hero에 렌더링합니다.
- **Ownership and data lifetime:** site page config가 route 공개 여부를, interview-map content가 intro/repository를, route가 not-found transition과 hero composition을 소유합니다.
- **Failure, absence, and non-guarantee:** disabled 상태에서는 shell/intro를 일부만 공개하지 않습니다. reference link가 빈 경우 임의 repository URL을 만들지 않으며 이 SHA의 content validation 밖입니다.
- **Resulting guarantee:** Interview Map의 fail-closed route boundary와 source repository로의 명시적 진입점을 보장합니다.
- **Relationship to later work:** `cb161...`이 route 안의 track navigation index를 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `ddba753f7f51` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.2. `cb161d118ddd` — feat(interview-map): 인터뷰 주제 인덱스 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** track 목록을 page-local topic anchor index로 추가합니다.

#### Exact inspection targets

- `src/app/interview-map/page.tsx` — topic index/nav section
- `content.interviewMap.tracks`
- track `id`, `label` 또는 title을 fragment href로 변환하는 위치
- `presentation.pages.interviewMap`의 index labels

#### Commit-specific investigation tasks

1. tracks의 ID/label/order가 topic index fragment links로 변환되는 규칙을 확인합니다.
2. link href와 후속 section ID가 일치하도록 어떤 field를 공유하는지 기록합니다.
3. tracks가 비어 있을 때 hero/index DOM 상태를 확인합니다.

#### Learner evidence record

- **Previous state:** hero와 reference repository만 있어 여러 interview tracks 중 원하는 주제로 바로 이동할 수 없었습니다.
- **Implementation decision and path:** track 순서대로 topic links를 만들고 각 link를 later track section의 fragment ID와 연결합니다.
- **Ownership and data lifetime:** interview-map content가 track ID와 순서를, route가 fragment navigation presentation을 소유합니다.
- **Failure, absence, and non-guarantee:** tracks가 비면 topic links가 없으며 route intro는 유지됩니다. ID uniqueness나 fragment validity는 이 renderer가 자동 수정하지 않습니다.
- **Resulting guarantee:** page 내부에서 content-defined track 순서로 탐색할 수 있음을 보장합니다.
- **Relationship to later work:** `9704...`가 topic 목록과 별개로 evidence gaps를 명시합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `cb161d118ddd` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.3. `9704aa5f7b59` — feat(interview-map): 근거 공백 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 현재 evidence의 부족 영역을 명시적인 gaps 목록으로 공개합니다.

#### Exact inspection targets

- `src/app/interview-map/page.tsx` — gaps section
- `content.interviewMap.gaps`
- `presentation.pages.interviewMap.gaps`
- gap item semantic list/card 반복

#### Commit-specific investigation tasks

1. `interviewMap.gaps`가 별도 section/list로 렌더링되는 위치와 presentation framing을 확인합니다.
2. gap이 runtime error/empty-state와 다른 content claim임을 data model과 DOM으로 구분합니다.
3. 빈 gaps에서 자동 gap 추론이나 placeholder 생성이 없는지 기록합니다.

#### Learner evidence record

- **Previous state:** topic index만 있으면 모든 주제가 충분한 evidence를 갖춘 것처럼 보일 수 있었습니다.
- **Implementation decision and path:** content-defined gaps를 별도 section에 렌더링해 아직 부족하거나 후속 근거가 필요한 영역을 공개합니다.
- **Ownership and data lifetime:** interview-map content가 gap 문구와 순서를, presentation이 framing copy를, route가 독립 section placement를 소유합니다.
- **Failure, absence, and non-guarantee:** gaps가 비면 별도 항목이 없고, route가 자동으로 gap을 추론하거나 issue를 생성하지 않습니다. gap은 runtime error가 아닙니다.
- **Resulting guarantee:** 근거 map이 완전성을 과장하지 않고 명시된 한계를 함께 보여 줌을 보장합니다.
- **Relationship to later work:** `a3fb...`이 실제 track sections를 추가해 topic index와 content body를 연결합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `9704aa5f7b59` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.4. `a3fb132d33f7` — feat(interview-map): 주제 track 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 각 interview track의 intro와 item count를 독립 section으로 구성합니다.

#### Exact inspection targets

- `src/app/interview-map/page.tsx` — `TrackSection` 초기 형태
- `content.interviewMap.tracks` 반복
- track ID를 section `id`로 적용
- track title/intro/item count와 alternating background

#### Commit-specific investigation tasks

1. `TrackSection`이 track ID, title, intro, item count를 어떻게 소비하는지 확인합니다.
2. alternating background/index 기반 presentation과 semantic section ID를 기록합니다.
3. 0-item track이 section 자체를 생략하는지 0 count를 보여 주는지 exact JSX에서 판별합니다.

#### Learner evidence record

- **Previous state:** topic index는 이동 대상만 제공하고 각 track의 범위·설명·evidence 규모를 실제 본문으로 보여 주지 않았습니다.
- **Implementation decision and path:** 각 track을 fragment target section으로 만들고 title, intro, item count를 표시합니다.
- **Ownership and data lifetime:** track content가 ID/title/intro/items를, route가 section ID·alternating layout·count presentation을 소유합니다.
- **Failure, absence, and non-guarantee:** track items가 비어도 track intro와 0 count는 유지될 수 있으며 임의 질문을 생성하지 않습니다.
- **Resulting guarantee:** topic index가 실제 track content section으로 연결되고 각 track의 범위를 먼저 설명함을 보장합니다.
- **Relationship to later work:** `cdc...`가 track items를 reference/question table로 확장합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `a3fb132d33f7` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.5. `cdcbbc937490` — feat(interview-map): 주제와 외부 참조 표 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** track items의 topic, external reference, interview question을 표 구조로 추가합니다.

#### Exact inspection targets

- `src/app/interview-map/page.tsx` — `TrackSection` table
- track item의 topic/title, reference link, question fields
- `presentation.pages.interviewMap.tracks` column labels
- external anchor semantics와 table header/body 관계

#### Commit-specific investigation tasks

1. track item의 topic/reference/question이 table headers와 cells에 어떻게 매핑되는지 확인합니다.
2. external reference anchor semantics와 question text source를 기록합니다.
3. answer/depth columns가 아직 없는 이전 state를 명시합니다.

#### Learner evidence record

- **Previous state:** track intro와 count만으로는 어떤 자료를 보고 어떤 질문에 답해야 하는지 구체적인 evidence path가 없었습니다.
- **Implementation decision and path:** 각 track item을 table row로 표현하고 topic, 외부 참조, 질문을 별도 columns에 배치합니다.
- **Ownership and data lifetime:** track item content가 reference/question을, presentation이 column labels를, route가 semantic table과 external-link expression을 소유합니다.
- **Failure, absence, and non-guarantee:** reference가 유효하지 않을 때 임의 URL을 대체하지 않습니다. 아직 project answers와 depth는 table에 없습니다.
- **Resulting guarantee:** 각 interview topic이 외부 학습 source와 질문으로 추적 가능함을 보장합니다.
- **Relationship to later work:** `1cd...`이 같은 rows에 project-backed answers와 depth를 추가합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `cdcbbc937490` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

### 5.6. `1cd28c140350` — feat(interview-map): 프로젝트 답변 근거 연결

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined role:** 각 question의 answers를 project case studies와 depth explanation으로 연결합니다.

#### Exact inspection targets

- `src/app/interview-map/page.tsx` — `TrackSection`
- `projectsById = new Map(content.projects.map(...))`
- `item.answers.map(...)`
- missing project branch가 raw `answer.projectId`를 출력하는 위치
- `getTemplateHref`를 사용한 project detail link
- answer/depth table columns

#### Commit-specific investigation tasks

1. `projectsById` Map 구성 비용/범위와 `item.answers` lookup 순서를 추적합니다.
2. lookup success의 project detail link와 failure의 raw project ID branch를 비교합니다.
3. answer와 depth lists가 같은 answer order를 유지하면서 별도 columns에 표현되는지 확인합니다.

#### Learner evidence record

- **Previous state:** 외부 reference와 질문은 있었지만 실제 구현 사례로 어떤 답변을 할 수 있고 그 깊이가 무엇인지 보여 주지 못했습니다.
- **Implementation decision and path:** track section마다 project lookup map을 만들고 answer IDs를 project로 해석합니다. 찾으면 template/debug-preserving detail link를, 못 찾으면 raw ID를 표시하며 depth는 별도 열에 유지합니다.
- **Ownership and data lifetime:** answer content가 project ID와 depth 순서를, project aggregate가 title/route ID를, route가 lookup과 missing-reference 표현을 소유합니다.
- **Failure, absence, and non-guarantee:** 누락 project를 drop하지 않습니다. raw ID는 참조 공백을 드러내지만 클릭 가능한 근거를 제공하지 않으며 자동 복구도 하지 않습니다.
- **Resulting guarantee:** 질문 → project answer → depth의 대응을 유지하고, 깨진 참조를 조용히 숨기지 않는 evidence-audit 계약을 보장합니다.
- **Relationship to later work:** 후대 Interview view model은 category 09에서 각 answer에 `project | null`을 붙여 renderer의 raw lookup 책임을 제거합니다.
- **Resource acquisition and cleanup:** 이 SHA의 관련 diff에는 파일·소켓·타이머 등 수동 자원 획득/해제 경로가 없습니다. 따라서 cleanup 분석은 적용 대상이 아니며, content aggregate와 React element의 render-time 소비 경계만 기록했습니다.

#### Execution evidence

- **Static historical inspection:** GitHub read-only connector로 정확한 `1cd28c140350` commit object와 diff를 조회하고 위 파일·symbol을 그 SHA 상태에서 검토했습니다.
- **Executed repository commands:** 없음. 컨테이너에서 GitHub DNS 해석 실패로 branch checkout을 만들 수 없었으므로 이 SHA에서 build/test/runtime pass를 주장하지 않습니다.
- **Evidence classification:** 위 기록은 exact-SHA code inspection 결과입니다. 실행 결과나 final HEAD에서 역추론한 내용이 아닙니다.

## 6. Invariant evolution

| Stage | Historical reconstruction |
|---|---|
| Route와 source | `ddba...`가 page flag와 reference repository를 가진 최소 Interview Map을 만들었습니다. |
| 탐색과 한계 | `cb161...`이 topic index를, `9704...`가 explicit gaps를 추가했습니다. |
| Track 계층 | `a3fb...`가 track intro/count를 만들고 `cdc...`가 reference/question table로 확장했습니다. |
| 프로젝트 답변 | `1cd...`이 answers와 depth를 추가하고 lookup failure를 raw ID로 공개했습니다. |

## 7. Failure → Fix → Test and later relationships

- Missing-reference policy는 About/Journey와 다릅니다. 그 route들은 해석 불가 links를 제거하지만 Interview Map은 evidence audit 목적상 raw project ID를 남깁니다.
- `9704...`의 gaps는 fix 대상 오류가 아니라 의도적인 non-guarantee disclosure입니다.
- 전용 regression test는 frozen map에 없습니다. project-or-null projection과 renderer boundary tests는 category 09에서 후속 보호됩니다.

## 8. Ownership, state, and responsibility changes

- Site page config: Interview Map 공개 여부.
- Interview-map content: intro, repository, tracks, items, references, questions, answers, depth, gaps.
- Project aggregate: answer ID가 참조하는 실제 project title/route.
- Interview route: topic anchors, table composition, project lookup, raw-ID missing state.
- 후대 view model: category 09에서 project resolution을 route projection 단계로 이동.

## 9. Final thread state

Interview Map은 disabled이면 404로 닫히고, enabled이면 source repository, topic index, explicit gaps, track별 reference/question table, project-backed answers와 depth를 제공합니다. project lookup 실패는 숨기지 않고 raw ID로 남깁니다.

## 10. Final architecture and execution flow

content aggregate 로드 → Interview Map page flag 검사 → disabled이면 `notFound()` → enabled이면 intro/reference repository → track IDs로 topic index 생성 → gaps 공개 → tracks 순회해 intro/count와 reference/question rows 생성 → project lookup map 구성 → each answer ID lookup → 성공 시 detail link, 실패 시 raw ID → depth를 대응 순서로 별도 열에 출력.

## 11. Minimal historical code evidence

`1cd28c140350`, `src/app/interview-map/page.tsx`, `TrackSection`:
```tsx
const project = projectsById.get(answer.projectId);
if (!project) {
  return <li key={answer.projectId}>{answer.projectId}</li>;
}
```
이 발췌는 증거 참조 실패를 조용히 제거하지 않고 감사 가능한 raw identifier로 남기는 정책을 보여 줍니다.

## 12. Learning-completion checks

- [x] Frozen commit map 6개를 모두 completion record와 연결했습니다.
- [x] SHA, subject, importance, tags, source-defined role의 고정 정보를 scaffold와 동일하게 유지했습니다.
- [x] 각 historical claim을 해당 SHA diff에 한정하고 later implementation을 소급하지 않았습니다.
- [x] fix/test가 없거나 다른 category 소유인 경우 그 사실을 명시했습니다.
- [x] runtime execution status를 명시했으며 실행하지 않은 command를 성공으로 기록하지 않았습니다.
- [x] learner placeholder를 남기지 않았습니다.
===== END FILE: 08-interview-evidence-map-and-gaps.md =====

===== BEGIN FILE: README.md =====
# 04-route-features-and-evidence-experiences

> **Repository:** `https://github.com/seungwoo7050/42-archive`  
> **Branch:** `web/portfolio`  
> **Workbook state:** completed workbook

## 1. Category boundary

이 category는 route별 사용자 경험과 그 화면에서 증거 데이터를 구성·생략·연결하는 규칙을 다룹니다.

- 포함: home, project index/detail, About, Resume, Contact, Journey, Interview Map의 feature composition.
- 제외: content ingestion/schema foundation은 category 01, query/navigation lifecycle은 category 02, shared UI/interaction primitives는 category 03, visual systems는 category 05, SEO/security는 category 06, broad test strategy는 category 07, production delivery는 category 08, route projection/renderer ownership은 category 09.
- category 09의 `03-route-projections-and-renderer-data-ownership.md`가 후대 view-model 도입·renderer 강제·projection regression을 전담하므로 이 category에 같은 commit을 중복 편입하지 않았습니다.

## 2. Phase 1 audit decisions

- Thread 수는 8개로 유지했습니다. 각 route experience는 독립적인 data interpretation과 absence policy를 가져 merge/split 대상이 아닙니다.
- `01-dual-home-composition-and-content-driven-sections.md`에 `cdb68fdf59f9` (`feat(home): 클래식 홈 히어로 구성`)를 추가했습니다. 이 commit 없이는 dual-home story의 Classic composition이 비어 있었습니다.
- `06-contact-preference-fallback-and-empty-state.md`의 마지막 두 commit 순서를 실제 이력에 맞게 `bc651dd85e14` → `9e99c9531cb8`로 교정했습니다.
- 나머지 commit set은 유지했습니다. broad route-view-model, renderer registry, visual-system, terminal-state, release/test commits는 sibling category와 중복되므로 추가하지 않았습니다.
- 기존의 범용 investigation prompt는 exact file, function/component, JSON field, selector, branch, missing-reference rule을 지정하는 commit-specific tasks로 대체했습니다.
- source classification에 따라 이 category의 59개 frozen commit은 모두 importance B입니다. S/A/C를 임의로 부여하지 않았습니다.

## 3. Historical and source validation basis

- branch-local `commit/commit-importance.md`와 `commit/commit-bodies.md`를 subject, importance, tags, role의 source로 사용했습니다.
- 모든 frozen SHA는 branch-local classification에서 확인하고 exact commit object/diff를 조회했습니다.
- category의 가장 이른 SHA `3475ba3efdb2`와 `web/portfolio` head 비교에서 merge base가 동일하고 head가 452 commits ahead, 0 behind임을 확인했습니다.
- 다른 branch의 구현·test·documentation을 대체 evidence로 사용하지 않았습니다.
- final HEAD code를 과거 commit 설명에 소급하지 않았습니다.

## 4. Thread inventory

| Order | Thread | Frozen commits | Primary route/experience |
|---:|---|---:|---|
| 1 | [`01-dual-home-composition-and-content-driven-sections.md`](./01-dual-home-composition-and-content-driven-sections.md) | 11 | Design/Classic home |
| 2 | [`02-project-index-grouping-and-dual-presentation.md`](./02-project-index-grouping-and-dual-presentation.md) | 8 | Project index |
| 3 | [`03-project-detail-case-study-composition.md`](./03-project-detail-case-study-composition.md) | 7 | Project detail |
| 4 | [`04-about-profile-skills-experience-and-curation.md`](./04-about-profile-skills-experience-and-curation.md) | 9 | About |
| 5 | [`05-resume-evidence-and-conditional-sections.md`](./05-resume-evidence-and-conditional-sections.md) | 7 | Resume |
| 6 | [`06-contact-preference-fallback-and-empty-state.md`](./06-contact-preference-fallback-and-empty-state.md) | 5 | Contact |
| 7 | [`07-journey-narrative-milestones-and-timeline.md`](./07-journey-narrative-milestones-and-timeline.md) | 6 | Journey |
| 8 | [`08-interview-evidence-map-and-gaps.md`](./08-interview-evidence-map-and-gaps.md) | 6 | Interview Map |

## 5. Shared completion rules

- 각 scaffold file은 같은 filename과 relative path의 completed counterpart 하나만 가집니다.
- SHA, commit order, subject, importance, tags, source-defined role, fixed invariant는 Phase 2에서 변경하지 않습니다.
- B-level commit은 concrete implementation role, data/DOM state, optional/missing branch, ownership, later relationship을 이해할 정도로 기록하되 cross-cutting S-level architecture를 중복 설명하지 않습니다.
- 실제 실행하지 않은 build/test/runtime 결과는 작성하지 않습니다.
- manual resource acquisition/cleanup이 없는 UI commit은 적용 불가라고 명시하고 억지 cleanup story를 만들지 않습니다.

## 6. Completion status

- **Status:** completed
- **Completed counterparts:** 8 / 8
- **Commit records completed:** 59 / 59
- **Historical inspection:** 각 참조 SHA의 GitHub commit object/diff를 exact SHA에서 정적으로 검토했습니다.
- **Runtime evidence:** repository checkout이 GitHub DNS 실패로 생성되지 않아 build/test command는 실행하지 않았습니다. 따라서 runtime pass를 주장하지 않습니다.
- **Placeholder status:** completed 문서에 learner marker가 남아 있지 않습니다.
===== END FILE: README.md =====

