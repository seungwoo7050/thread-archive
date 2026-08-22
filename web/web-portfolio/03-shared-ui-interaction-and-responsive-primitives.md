===== BEGIN FILE: 01-content-link-security-and-transport.md =====
# Thread: Content link security, placement, and transport

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

연락처와 프로젝트 링크의 선택 정책, 내부/외부 transport, 배치별 노출, availability filtering과 공용 renderer 정리를 실제 commit 순서로 복원합니다.

**경계:** 이 Thread는 링크가 어디에 노출되고 어떤 element/URL로 이동하는지를 소유합니다. ProjectCard의 카드 조립과 hover 표현은 06 Thread에 남기며, route lifecycle 자체는 이 범위에 포함하지 않습니다.

### 고정 invariant

- Project card/detail의 가용성·placement 선택은 selector가 맡고, hero consumer는 이 history에서 같은 placement vocabulary를 직접 적용합니다. Transport renderer는 surface membership을 다시 결정하지 않습니다.
- 외부 링크는 anchor transport와 외부 속성을 사용하고, 내부 링크는 Next Link와 현재 view/debug query 보존 규칙을 사용합니다.
- live가 아닌 프로젝트의 demo와 해당 placement에 없는 링크는 렌더링되지 않습니다.
- 선택 결과가 비면 action container도 렌더링하지 않습니다.

## 2. 핵심 질문

- 초기 selector가 contact/project link availability를 어떤 단일 vocabulary로 만들었는가?
- ContentLinkView가 internal/external transport를 분기할 때 보장하는 속성과 보장하지 않는 검증은 무엇인가?
- card/detail/hero placement가 언제 도입되고 각 consumer의 filtering이 selector 또는 direct predicate 중 어디에 남았는가?
- 09cec616f314의 test가 source order, query 보존, external attributes, empty output을 어떤 technique로 고정하는가?
- 44e4d062da50의 refactor가 behavior를 바꾸지 않고 어떤 렌더링 책임만 합쳤는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- S/A-level은 architecture/owner/failure/later evidence를 B-level보다 깊게 복원합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `ba8da56d3fcf` | feat(portfolio): 연락과 프로젝트 링크 선택기 추가 | A | CONTENT | 정책 owner 도입 |
| 2 | `f63c978c71c9` | feat(ui): 내부 외부 콘텐츠 링크 렌더링 | A | CONTENT | transport renderer 도입 |
| 3 | `e37ea9c2819a` | feat(project): 프로젝트 링크 그룹 추가 | B | RENDERER | detail action consumer |
| 4 | `1ef269fbdb49` | feat(project): 프로젝트 카드 링크 추가 | B | RENDERER | card action consumer |
| 5 | `daa6815a6dfa` | feat(project): 카드 링크를 콘텐츠 배치 기준으로 선택 | B | CONTENT, RENDERER | card placement 적용 |
| 6 | `119ff9a92090` | feat(content): 링크 배치 selector 추가 | B | CONTENT | placement vocabulary 일반화 |
| 7 | `2d87b62dcce8` | refactor(project): 상세 링크를 배치 기준으로 선택 | B | RENDERER, REFACTOR | detail selector migration |
| 8 | `ee2c118a76d6` | feat(content): 홈 링크를 배치 기준으로 선택 | B | CONTENT | hero consumer migration |
| 9 | `09cec616f314` | test(ui): 디자인 선택과 프로젝트 링크 계약 검증 | A | VALIDATION, TEST | 결정적 component contract 검증 |
| 10 | `44e4d062da50` | refactor(ui): 프로젝트 링크 렌더링 중복 제거 | B | VALIDATION, REFACTOR | 공용 list renderer 추출 |

## 5. Commit별 학습 기록

### 1. `ba8da56d3fcf` — feat(portfolio): 연락과 프로젝트 링크 선택기 추가

- **Importance:** A
- **Tags:** CONTENT
- **Thread 역할:** 정책 owner 도입

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/selectors.ts`의 `getPreferredContactLinks`, `getProjectLink`, `isProjectLive`, `getProjectCardLinks`, `getExternalLinkProps`를 parent와 비교합니다.
- `src/lib/portfolio.ts` export surface가 새 selector를 어떤 public boundary로 노출하는지 확인합니다.
- preferred ID 누락, demo link 부재, non-live deployment, internal link의 반환값을 각각 추적합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-ba8da56d3fcf-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 직전 tree에는 contact preferred ID를 실제 link로 해석하거나 프로젝트 demo availability와 외부 transport 속성을 공통으로 계산하는 selector가 없었습니다. Consumer가 같은 판단을 반복할 여지가 있었습니다. |
| 실제 변경 file/symbol/call path | `selectors.ts`가 preferred 순서를 유지하면서 존재하는 link만 남기고, type별 project link 조회, `status === "live"`와 demo 존재를 함께 요구하는 live 판정, 초기 card action 선택, 외부 link props를 추가했습니다. `portfolio.ts`가 이를 public export로 묶었습니다. |
| Data/state/DOM/resource owner | Content data가 사실 원본을 소유하고 selector가 파생 결과를 소유합니다. Component는 selector의 반환 배열·boolean·props를 소비할 뿐 deployment 의미를 새로 결정하지 않는 방향이 시작됐습니다. |
| Failure·absence·fallback 처리 | 없는 preferred ID는 type guard filter로 제거되고, 없는 project link는 `null`, live 조건을 만족하지 않는 demo는 card 결과에서 제외됩니다. Internal link의 external props는 빈 object입니다. |
| 보장하는 것과 보장하지 않는 것 | 연락처 순서와 초기 project action availability를 중앙화합니다. 다만 placement vocabulary는 아직 없고, href scheme/content validity와 실제 DOM rendering은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | f63c978c71c9가 transport renderer를 만들고, e37ea9c2819a·1ef269fbdb49가 detail/card consumer를 붙입니다. daa6815a6dfa 이후 placement가 정책 입력으로 추가됩니다. |
<!-- learner:commit-ba8da56d3fcf-record:end -->

#### 최소 코드 증거

<!-- learner:commit-ba8da56d3fcf-excerpt:start -->
- **Commit:** `ba8da56d3fcf`
- **Path:** `src/lib/portfolio/selectors.ts`
- **Location:** `isProjectLive / getExternalLinkProps`

```tsx
export function getProjectLink(project: PortfolioProject, type: LinkType) {
  return project.links.find((link) => link.type === type) ?? null;
}

export function isProjectLive(project: PortfolioProject) {
  return Boolean(
    project.deployment.status === "live" && getProjectLink(project, "demo"),
  );
}

export function getProjectCardLinks(project: PortfolioProject) {
  return project.links.filter((link) => {
    if (link.type === "demo") {
      return isProjectLive(project);
    }

    return link.type === "github" || link.type === "case-study";
  });
}

export function getExternalLinkProps(link: ContentLink) {
  if (!link.external) {
    return {};
  }

  return {
    rel: "noreferrer",
    target: "_blank",
  };
}
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-ba8da56d3fcf-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-ba8da56d3fcf-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-ba8da56d3fcf-execution:end -->

### 2. `f63c978c71c9` — feat(ui): 내부 외부 콘텐츠 링크 렌더링

- **Importance:** A
- **Tags:** CONTENT
- **Thread 역할:** transport renderer 도입

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/content-link.tsx`의 `ContentLinkView` 두 return branch를 비교합니다.
- 외부 branch가 `getExternalLinkProps`를 spread하고 내부 branch가 `getTemplateHref`에 `homeTemplate`·`contentDebug`를 전달하는 call path를 확인합니다.
- 이 component가 visibility, href validation, label/icon styling을 소유하는지 구분합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-f63c978c71c9-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Selector는 transport에 필요한 정보만 반환했고, 각 consumer가 `<a>`와 `next/link`를 직접 선택하면 외부 속성이나 query 보존이 서로 달라질 수 있었습니다. |
| 실제 변경 file/symbol/call path | `ContentLinkView`가 `link.external`을 유일한 transport 분기값으로 사용합니다. External branch는 원래 href를 가진 `<a>`와 selector props를 사용하고, internal branch는 `getTemplateHref(link.href, homeTemplate, { contentDebug })`를 거친 Next `Link`를 사용합니다. |
| Data/state/DOM/resource owner | ContentLink data가 external 여부와 href를 소유하고, renderer가 DOM element 선택을 소유합니다. 내부 URL의 view/debug 합성은 `getTemplateHref`가 소유합니다. |
| Failure·absence·fallback 처리 | External false이면 외부 속성을 붙이지 않습니다. 이 commit에는 malformed URL, unsupported protocol, 빈 label을 거부하는 branch가 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 내부·외부 link transport와 내부 query 보존 경로를 일관되게 만듭니다. 어떤 link를 보여 줄지는 보장하지 않고 caller가 children/className도 제공합니다. |
| 다음 commit 또는 관련 test 연결 | e37ea9c2819a 이후 모든 project action이 이 renderer를 통과합니다. 09cec616f314가 internal query와 external `target`/`rel`을 component test로 검증합니다. |
<!-- learner:commit-f63c978c71c9-record:end -->

#### 최소 코드 증거

<!-- learner:commit-f63c978c71c9-excerpt:start -->
- **Commit:** `f63c978c71c9`
- **Path:** `src/components/portfolio/content-link.tsx`
- **Location:** `ContentLinkView`

```tsx
if (link.external) {
  return (
    <a className={className} href={link.href} {...getExternalLinkProps(link)}>
      {children}
    </a>
  );
}

return (
  <Link
    className={className}
    href={getTemplateHref(link.href, homeTemplate, { contentDebug })}
  >
    {children}
  </Link>
);
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-f63c978c71c9-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-f63c978c71c9-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-f63c978c71c9-execution:end -->

### 3. `e37ea9c2819a` — feat(project): 프로젝트 링크 그룹 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** detail action consumer

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/project-links.tsx`의 `ProjectLinks`와 local visibility predicate를 확인합니다.
- demo availability, `excludeCaseStudy`, empty result, `ContentLinkView` 호출을 각각 추적합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-e37ea9c2819a-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Project detail에서 project.links를 공통 transport renderer로 출력하는 action group이 없었습니다. |
| 실제 변경 file/symbol/call path | `ProjectLinks`가 project links를 filtering한 뒤 flex action group으로 렌더링합니다. demo는 `isProjectLive`, case-study는 prop에 따라 제외하고, 각 item은 `ContentLinkView`를 사용합니다. |
| Data/state/DOM/resource owner | 이 시점에는 detail component가 일부 visibility filtering을 소유합니다. Transport는 ContentLinkView가 소유합니다. |
| Failure·absence·fallback 처리 | 필터 결과가 비면 `null`입니다. Offline demo와 명시적으로 제외한 case-study는 DOM에 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Detail action group을 재사용할 수 있게 하지만 placement는 아직 읽지 않고 filtering이 selector와 component에 나뉩니다. |
| 다음 commit 또는 관련 test 연결 | 2d87b62dcce8이 detail placement selection을 selector로 옮기고, 44e4d062da50이 list rendering을 합칩니다. |
<!-- learner:commit-e37ea9c2819a-record:end -->

#### 최소 코드 증거

<!-- learner:commit-e37ea9c2819a-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-e37ea9c2819a-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-e37ea9c2819a-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-e37ea9c2819a-execution:end -->

### 4. `1ef269fbdb49` — feat(project): 프로젝트 카드 링크 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** card action consumer

#### 해당 SHA에서 확인할 실제 코드

- `ProjectCardLinks`가 `getProjectCardLinks` 결과를 어떻게 렌더링하는지 확인합니다.
- `ProjectLinks`와 중복된 null branch, class selection, icon choice를 비교합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-1ef269fbdb49-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail action은 생겼지만 card가 selector의 card action 결과를 표현하는 wrapper는 없었습니다. |
| 실제 변경 file/symbol/call path | 같은 file에 `ProjectCardLinks`를 추가해 `getProjectCardLinks(project)` 결과를 `ContentLinkView`로 렌더링했습니다. |
| Data/state/DOM/resource owner | Card visibility는 selector, DOM/class/icon은 wrapper가 소유하지만 detail wrapper와 markup 책임이 중복됩니다. |
| Failure·absence·fallback 처리 | 빈 배열이면 `null`입니다. Demo primary styling과 external/internal icon 분기는 rendering에서 처리됩니다. |
| 보장하는 것과 보장하지 않는 것 | Card consumer를 연결하지만 content-authored placement는 아직 반영하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | daa6815a6dfa가 card selector를 placement 기반으로 바꾸고, 44e4d062da50이 중복 list renderer를 제거합니다. |
<!-- learner:commit-1ef269fbdb49-record:end -->

#### 최소 코드 증거

<!-- learner:commit-1ef269fbdb49-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-1ef269fbdb49-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-1ef269fbdb49-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-1ef269fbdb49-execution:end -->

### 5. `daa6815a6dfa` — feat(project): 카드 링크를 콘텐츠 배치 기준으로 선택

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** card placement 적용

#### 해당 SHA에서 확인할 실제 코드

- `getProjectCardLinks`의 기존 type whitelist와 새 `placements.includes("card")` gate를 비교합니다.
- Placement를 통과한 demo와 non-demo가 각각 어떤 추가 조건을 거치는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-daa6815a6dfa-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Card links는 demo/github/case-study라는 hard-coded type 집합으로 선택되어 content가 같은 type을 특정 surface에서 숨기거나 다른 type을 card에 배치할 수 없었습니다. |
| 실제 변경 file/symbol/call path | Selector가 먼저 `link.placements?.includes("card")`를 요구합니다. Demo만 기존 live 판정을 추가로 통과하고, placement가 card인 다른 type은 허용합니다. |
| Data/state/DOM/resource owner | Content가 placement 의도를 소유하고 selector가 placement와 runtime availability를 결합합니다. Card component는 결과를 그대로 소비합니다. |
| Failure·absence·fallback 처리 | placements가 없거나 card를 포함하지 않으면 제외됩니다. Placement가 있어도 non-live demo는 제외됩니다. |
| 보장하는 것과 보장하지 않는 것 | Type whitelist보다 content-authored placement를 우선하는 card 정책을 보장합니다. 다른 surface의 generic selector는 아직 없습니다. |
| 다음 commit 또는 관련 test 연결 | 119ff9a92090이 placement type과 generic selector를 도입해 이 결정을 card/detail/site로 일반화합니다. |
<!-- learner:commit-daa6815a6dfa-record:end -->

#### 최소 코드 증거

<!-- learner:commit-daa6815a6dfa-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-daa6815a6dfa-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-daa6815a6dfa-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-daa6815a6dfa-execution:end -->

### 6. `119ff9a92090` — feat(content): 링크 배치 selector 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** placement vocabulary 일반화

#### 해당 SHA에서 확인할 실제 코드

- `LinkPlacement` union과 `getProjectLinksForPlacement`, card/detail wrapper, site-link placement selector를 확인합니다.
- Source array order가 filter 결과에서도 유지되는지와 placement absence가 어떻게 처리되는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-119ff9a92090-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Card만 placement를 직접 읽고 detail·site consumer는 별도 규칙을 유지해 surface vocabulary와 선택 책임이 분산되어 있었습니다. |
| 실제 변경 file/symbol/call path | `hero`, `contact`, `card`, `detail`, `footer` placement vocabulary와 generic filter를 추가하고 card/detail wrapper 및 site content selector를 만들었습니다. |
| Data/state/DOM/resource owner | Placement membership의 공통 해석은 selector가 소유하고, 각 wrapper는 surface 이름만 고정합니다. |
| Failure·absence·fallback 처리 | Optional placements가 없으면 generic filter에서 제외됩니다. Runtime demo availability는 surface wrapper/consumer가 추가로 처리해야 합니다. |
| 보장하는 것과 보장하지 않는 것 | 모든 surface가 같은 placement vocabulary를 사용할 기반을 보장하지만 consumer migration은 아직 끝나지 않았습니다. |
| 다음 commit 또는 관련 test 연결 | 2d87b62dcce8은 detail consumer를 selector로 이동하고, ee2c118a76d6은 hero consumer가 동일 placement vocabulary를 direct predicate로 채택하게 합니다. |
<!-- learner:commit-119ff9a92090-record:end -->

#### 최소 코드 증거

<!-- learner:commit-119ff9a92090-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-119ff9a92090-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-119ff9a92090-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-119ff9a92090-execution:end -->

### 7. `2d87b62dcce8` — refactor(project): 상세 링크를 배치 기준으로 선택

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread 역할:** detail selector migration

#### 해당 SHA에서 확인할 실제 코드

- `ProjectLinks`가 raw `project.links` 대신 `getProjectDetailLinks(project)`를 호출하도록 바뀐 부분을 확인합니다.
- Placement selection 후에도 `excludeCaseStudy`와 demo live check가 어느 layer에 남는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-2d87b62dcce8-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail wrapper가 raw link array에서 자체 filtering하여 detail placement를 반영하지 않았습니다. |
| 실제 변경 file/symbol/call path | 입력 array를 `getProjectDetailLinks(project)` 결과로 교체하고, case-study 제외 option과 demo live check는 component에 유지했습니다. |
| Data/state/DOM/resource owner | Selector가 detail membership을 소유하고 component가 caller option 및 runtime presentation filtering을 소유합니다. |
| Failure·absence·fallback 처리 | Detail placement가 없는 link는 component에 도달하지 않습니다. Empty result는 기존 null branch로 이어집니다. |
| 보장하는 것과 보장하지 않는 것 | Detail surface가 content placement를 따르게 하지만 demo filtering이 완전히 selector로 이동한 것은 아닙니다. |
| 다음 commit 또는 관련 test 연결 | 09cec616f314가 detail source order와 filtering을 검증하고, 44e4d062da50이 rendering 중복만 정리합니다. |
<!-- learner:commit-2d87b62dcce8-record:end -->

#### 최소 코드 증거

<!-- learner:commit-2d87b62dcce8-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-2d87b62dcce8-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-2d87b62dcce8-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-2d87b62dcce8-execution:end -->

### 8. `ee2c118a76d6` — feat(content): 홈 링크를 배치 기준으로 선택

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** hero consumer migration

#### 해당 SHA에서 확인할 실제 코드

- Classic/Design home route에서 hard-coded link type filter가 hero placement filter로 교체된 diff를 확인합니다.
- 두 design route가 같은 content contract를 읽되 각 route의 presentation markup은 유지되는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-ee2c118a76d6-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 홈 hero는 github/resume/website type을 코드에 직접 열거해 content placement와 분리되어 있었습니다. |
| 실제 변경 file/symbol/call path | 두 home implementation이 `link.placements?.includes("hero")`를 사용하도록 변경됐습니다. |
| Data/state/DOM/resource owner | Content가 hero membership을 선언하고 각 design route가 그 placement predicate와 layout·styling을 직접 적용합니다. 이 시점에는 hero selection owner가 selector로 이동하지 않았습니다. |
| Failure·absence·fallback 처리 | Hero placement가 없는 link는 type과 무관하게 제외됩니다. 이 commit은 generic site selector를 호출하지 않고 동일 predicate를 두 route에 둡니다. |
| 보장하는 것과 보장하지 않는 것 | 홈 link set이 content placement를 따르게 합니다. Footer/contact migration이나 transport 변경은 포함하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 후속 refactor 대상은 남지만, 09cec616f314의 project-link tests와 별개로 home consumer behavior는 이 commit에 dedicated test가 없습니다. |
<!-- learner:commit-ee2c118a76d6-record:end -->

#### 최소 코드 증거

<!-- learner:commit-ee2c118a76d6-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-ee2c118a76d6-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-ee2c118a76d6-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-ee2c118a76d6-execution:end -->

### 9. `09cec616f314` — test(ui): 디자인 선택과 프로젝트 링크 계약 검증

- **Importance:** A
- **Tags:** VALIDATION, TEST
- **Thread 역할:** 결정적 component contract 검증

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/project-links.test.tsx`의 fixtures와 assertion을 production selector/renderer path별로 매핑합니다.
- Internal URL의 `view`·`debug`, external `target`·`rel`, offline demo, case-study exclusion, card placement, empty DOM을 구분합니다.
- 같은 commit의 `design-switcher.test.tsx`가 native details close/focus를 검증하지만 이 Thread에서는 link contract와의 경계만 기록합니다.
- JSDOM component test가 실제 navigation·browser security·network response를 증명하지 않는다는 한계를 명시합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-09cec616f314-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Placement와 transport 규칙은 구현돼 있었지만 source order, query preservation, external attributes, absence branches를 한 번에 깨뜨리지 못하게 하는 deterministic regression evidence가 없었습니다. |
| 실제 변경 file/symbol/call path | Testing Library로 production components를 렌더링합니다. Detail links는 source order와 내부/외부 속성을 검사하고, offline demo·case-study exclusion을 재현합니다. Card links는 card placement가 아닌 source link를 제외하며, 모두 필터된 wrapper는 empty DOM임을 확인합니다. |
| Data/state/DOM/resource owner | Test fixture가 입력 상태를 소유하고 production selector/renderer가 실제 결과 DOM을 소유합니다. Assertion은 href/attributes/text/empty container를 관찰합니다. |
| Failure·absence·fallback 처리 | Failure injection 대신 content fixture의 deployment status, placements, exclude option을 조절하는 경계 테스트입니다. External navigation이나 target page 응답은 실행하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 해당 SHA의 component-level contract를 결정적으로 고정합니다. Browser 새 창 isolation 전체, route transition, malformed content validation, CSS interaction은 증명하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 44e4d062da50은 이 test contract를 유지하면서 중복 renderer를 추출합니다. Design-switcher test의 후속 hydration story는 08 Thread의 외부 관계 및 07 category가 소유합니다. |
<!-- learner:commit-09cec616f314-record:end -->

#### 최소 코드 증거

<!-- learner:commit-09cec616f314-excerpt:start -->
- **Commit:** `09cec616f314`
- **Path:** `src/components/portfolio/project-links.test.tsx`
- **Location:** `project link transport assertions`

```tsx
expect(links[0]).toHaveAttribute(
  "href",
  "/projects/sample-project?view=classic&debug=content",
);
expect(links[0]).not.toHaveAttribute("target");
expect(links[1]).toHaveAttribute("target", "_blank");
expect(links[1]).toHaveAttribute("rel", "noreferrer");
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-09cec616f314-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-09cec616f314-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-09cec616f314-execution:end -->

### 10. `44e4d062da50` — refactor(ui): 프로젝트 링크 렌더링 중복 제거

- **Importance:** B
- **Tags:** VALIDATION, REFACTOR
- **Thread 역할:** 공용 list renderer 추출

#### 해당 SHA에서 확인할 실제 코드

- 새 local `ProjectLinkList`와 두 public wrapper의 before/after를 비교합니다.
- Empty branch, key, min-height, primary class, external/internal icon과 `ContentLinkView` props가 한 곳으로 이동했는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-44e4d062da50-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail/card wrapper가 같은 empty check, classes, key, icon, transport 호출을 복제해 한쪽만 변경될 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | Local `ProjectLinkList`가 empty `null`과 item rendering을 소유하고 `ProjectLinks`·`ProjectCardLinks`는 selection만 수행해 links를 전달합니다. |
| Data/state/DOM/resource owner | Public wrapper는 surface-specific selection을, private list는 shared DOM representation을 소유합니다. |
| Failure·absence·fallback 처리 | 빈 배열은 공용 list에서 한 번 처리됩니다. 새 failure branch는 없으며 transport는 계속 ContentLinkView에 위임됩니다. |
| 보장하는 것과 보장하지 않는 것 | 09cec616f314에서 고정한 behavior를 유지하면서 duplicate markup을 제거합니다. 독립적인 new feature나 selector change는 아닙니다. |
| 다음 commit 또는 관련 test 연결 | Project action의 최종 layering은 content → selector → surface wrapper → ProjectLinkList → ContentLinkView입니다. Hero는 content placement를 route consumer가 직접 filtering한 뒤 ContentLinkView를 사용합니다. |
<!-- learner:commit-44e4d062da50-record:end -->

#### 최소 코드 증거

<!-- learner:commit-44e4d062da50-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-44e4d062da50-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-44e4d062da50-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-44e4d062da50-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 가용성·외부 속성 중앙화 | ba8da56d3fcf | selectors.ts의 null/filter/live/external branch | placement가 없어 surface별 의미는 불완전 | 가용성과 transport input은 selector가 소유 |
| 내부/외부 transport 일원화 | f63c978c71c9 | ContentLinkView의 `<a>`/Next Link branch | visibility는 caller에 남음 | 모든 link DOM transport가 공용 renderer를 통과 |
| 배치 기반 노출 | daa6815a6dfa → 119ff9a92090 | card gate, LinkPlacement와 generic selector | consumer migration 필요 | card/detail/hero가 content placement를 따름 |
| 결정적 회귀 보호 | 09cec616f314 | project-links.test.tsx fixture/assertions | 브라우저 navigation·URL validation은 미검증 | source order·query·attributes·absence를 component level에서 고정 |
| 표현 중복 제거 | 44e4d062da50 | ProjectLinkList 추출 | surface selector는 의도적으로 별도 | selection과 rendering 책임이 분리 |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| Hard-coded type selection | Content placement를 표현할 수 없음 | daa6815a6dfa/119ff9a92090에서 placement selector 도입 | 09cec616f314이 card/detail filtering을 검증 |
| Internal/external rendering duplication 위험 | 외부 속성 또는 query 보존 불일치 가능 | f63c978c71c9에서 ContentLinkView 도입 | 09cec616f314이 transport attributes를 검증 |
| Detail/card markup duplication | 한 wrapper만 styling/null behavior가 달라질 수 있음 | 44e4d062da50에서 ProjectLinkList 추출 | 앞선 09cec616f314 contract가 behavior 기준 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| 초기 | Raw content와 각 consumer | availability·transport·surface 규칙이 분산될 수 있음 |
| ba8/f63 | Selectors + ContentLinkView | 정책과 transport renderer가 분리됨 |
| daa/119/2d/ee | Content placements + selectors/hero consumers | card/detail은 selector가 해석하고 hero는 같은 placement predicate를 consumer가 직접 적용 |
| 44e4 최종 | Surface wrapper + ProjectLinkList + ContentLinkView | selection, shared DOM, transport가 단계별 owner를 가짐 |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- Preferred contact와 project card/detail links는 content 원본에서 selector를 거쳐 파생되며, home hero는 route consumer가 content placement를 직접 filtering합니다.
- Card/detail/hero는 content placement를 따릅니다. Project demo는 card/detail visibility path에서 deployment live와 demo href 존재를 추가로 요구합니다.
- Surface wrapper는 선택된 배열과 local option만 처리하고, shared list가 empty·classes·icons를 처리합니다.
- ContentLinkView는 external anchor와 internal Next Link를 분기합니다.
- Malformed href, remote availability, 실제 browser navigation은 이 Thread가 보장하지 않습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. Content loader가 `ContentLink`와 project deployment/placement data를 제공합니다.
2. Card/detail은 selector가 placement와 availability를 적용해 source order를 보존한 배열을 만들고, hero consumer는 같은 `hero` placement predicate를 직접 적용합니다.
3. ProjectLinks 또는 ProjectCardLinks가 surface option을 적용합니다.
4. ProjectLinkList가 빈 배열이면 아무 DOM도 만들지 않고, 아니면 공통 action markup을 만듭니다.
5. ContentLinkView가 external이면 anchor 속성을, internal이면 template/debug가 보존된 app URL을 사용합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] S/A-level 설명을 B-level보다 깊게 작성했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 01-content-link-security-and-transport.md =====

===== BEGIN FILE: 02-content-media-loading-and-layout-stability.md =====
# Thread: Content media loading and layout stability

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

프로필 사진과 프로젝트 screenshot primitive가 content metadata를 어떻게 보존하고, hero와 gallery consumer가 eager/lazy loading 및 고정 aspect layout을 어떻게 선택하는지 복원합니다.

**경계:** 이 Thread는 shared media DOM과 loading/layout contract를 다룹니다. Image optimization pipeline, source asset generation, project detail 정보 architecture 자체는 포함하지 않습니다.

### 고정 invariant

- alt/src는 content가 소유하고 media primitive는 이를 임의로 대체하지 않습니다.
- ProjectScreenshot의 priority만 eager loading을 선택하며 기본값은 lazy입니다.
- ProjectScreenshot의 `<img>`에 고정 aspect ratio가 있어 image load 전에도 screenshot 높이를 계산할 수 있습니다.
- Optional profile photo가 없으면 consumer는 빈 placeholder가 아니라 media DOM 자체를 생략합니다.

## 2. 핵심 질문

- aa115c73ae30에서 raw `<img>`를 감싸는 두 primitive가 어떤 loading·aspect·alt contract를 갖는가?
- Lead/detail hero가 priority를 명시하고 gallery가 default lazy를 유지하는 consumer 차이는 무엇인가?
- `profile.photo ? ... : null` branch가 optional content를 어떻게 표현하는가?
- 이 history에 error fallback, responsive `srcset`, framework image optimization이 실제로 존재하는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- 이 Thread는 B-level commit만 포함하므로 각 commit의 concrete role, boundary, failure/non-guarantee를 필요한 범위로 기록합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `aa115c73ae30` | feat(ui): 콘텐츠 이미지 프리미티브 추가 | B | CONTENT | media primitive 도입 |
| 2 | `b027f42669aa` | feat(home): 대표 프로젝트 쇼케이스 추가 | B | RENDERER | lead screenshot consumer |
| 3 | `06fff9a6e93b` | feat(project): 프로젝트 상세 소개 추가 | B | RENDERER | detail hero consumer |
| 4 | `cabf3a0e378f` | feat(project): 프로젝트 구조와 증거 갤러리 추가 | B | RENDERER | gallery lazy consumer |
| 5 | `a00a6bf1af58` | feat(about): 프로필 사진 소개 추가 | B | RENDERER | optional profile consumer |

## 5. Commit별 학습 기록

### 1. `aa115c73ae30` — feat(ui): 콘텐츠 이미지 프리미티브 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** media primitive 도입

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/profile-photo.tsx`와 `project-screenshot.tsx`의 raw img attributes를 확인합니다.
- ProjectScreenshot의 `priority` default, loading branch, `<img>` aspect ratio, figure overflow frame, object position과 hover class를 확인합니다.
- 같은 commit의 ContentHint는 별도 concern임을 표시하고 media evidence와 섞지 않습니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-aa115c73ae30-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Profile/project image를 여러 consumer가 직접 렌더링하면 loading mode, crop, aspect reservation과 alt 전달이 달라질 수 있었습니다. |
| 실제 변경 file/symbol/call path | `ProfilePhoto`와 `ProjectScreenshot`을 추가했습니다. Profile photo는 eager img를 사용하고, screenshot은 `priority=false` 기본값에서 lazy, true에서 eager를 사용합니다. Screenshot `<img>`는 16:10 aspect와 object-fit crop을, figure wrapper는 overflow와 frame을 고정합니다. |
| Data/state/DOM/resource owner | Content image object가 src/alt를 소유하고 primitive가 DOM attributes와 presentation frame을 소유합니다. Browser가 실제 image request와 decode lifetime을 소유합니다. |
| Failure·absence·fallback 처리 | Image load error를 대체하는 state나 callback은 없습니다. Invalid src/alt는 이 component가 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Shared loading/layout contract를 만듭니다. Responsive source selection, framework Image optimization, width/height 기반 intrinsic sizing은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | b027f42669aa와 06fff9a6e93b가 priority consumer를, cabf3a0e378f가 lazy gallery consumer를 연결합니다. |
<!-- learner:commit-aa115c73ae30-record:end -->

#### 최소 코드 증거

<!-- learner:commit-aa115c73ae30-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-aa115c73ae30-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-aa115c73ae30-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-aa115c73ae30-execution:end -->

### 2. `b027f42669aa` — feat(home): 대표 프로젝트 쇼케이스 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** lead screenshot consumer

#### 해당 SHA에서 확인할 실제 코드

- Design home lead-project branch에서 `ProjectScreenshot image={leadProject.screenshot} priority` 호출을 확인합니다.
- Lead project absence가 section DOM에 미치는 영향을 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b027f42669aa-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Shared screenshot primitive는 있었지만 home showcase가 어떤 media를 eager 처리할지 결정하지 않았습니다. |
| 실제 변경 file/symbol/call path | Design home이 featured selector의 lead project를 조건부로 잡아 `ProjectScreenshot`에 `priority`를 전달합니다. |
| Data/state/DOM/resource owner | Home route가 above-the-fold priority 결정을 소유하고 primitive가 loading attribute로 변환합니다. |
| Failure·absence·fallback 처리 | Lead project가 없으면 해당 screenshot branch가 렌더링되지 않습니다. Broken image fallback은 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Home lead screenshot이 eager loading과 shared aspect/frame contract를 사용합니다. 다른 featured image까지 모두 eager라는 보장은 이 commit만으로는 없습니다. |
| 다음 commit 또는 관련 test 연결 | 06fff9a6e93b가 detail hero에도 같은 priority contract를 적용합니다. |
<!-- learner:commit-b027f42669aa-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b027f42669aa-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-b027f42669aa-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b027f42669aa-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-b027f42669aa-execution:end -->

### 3. `06fff9a6e93b` — feat(project): 프로젝트 상세 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** detail hero consumer

#### 해당 SHA에서 확인할 실제 코드

- 새 `ProjectDetailView`에서 hero screenshot 위치와 `priority` prop을 확인합니다.
- Back link, metadata, links와 screenshot primitive의 responsibility boundary를 구분합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-06fff9a6e93b-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Project detail view와 그 hero media consumer가 없었습니다. |
| 실제 변경 file/symbol/call path | `ProjectDetailView`가 content summary/description/actions 옆에 `ProjectScreenshot image={project.screenshot} priority`를 배치합니다. |
| Data/state/DOM/resource owner | Detail view가 hero placement와 priority를, primitive가 frame/loading을 소유합니다. |
| Failure·absence·fallback 처리 | Project가 존재한다는 전제에서 view가 호출됩니다. 이 component에는 missing screenshot fallback이 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 상세 첫 화면 media는 eager이고 shared aspect/frame contract를 사용합니다. Detail routing/not-found와 data validation은 다른 layer의 책임입니다. |
| 다음 commit 또는 관련 test 연결 | cabf3a0e378f가 같은 view에 evidence gallery를 추가하면서 default lazy behavior를 대조시킵니다. |
<!-- learner:commit-06fff9a6e93b-record:end -->

#### 최소 코드 증거

<!-- learner:commit-06fff9a6e93b-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-06fff9a6e93b-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-06fff9a6e93b-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-06fff9a6e93b-execution:end -->

### 4. `cabf3a0e378f` — feat(project): 프로젝트 구조와 증거 갤러리 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** gallery lazy consumer

#### 해당 SHA에서 확인할 실제 코드

- `ProjectDetailView`의 `project.screenshots.map`과 각 `ProjectScreenshot` 호출에 priority가 없는지 확인합니다.
- Architecture list와 gallery가 content array order를 그대로 사용하는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-cabf3a0e378f-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail hero 한 장만 표현됐고 추가 evidence screenshots를 순서대로 보여 주는 gallery가 없었습니다. |
| 실제 변경 file/symbol/call path | Architecture section과 screenshots section을 추가하고 `project.screenshots.map`에서 default ProjectScreenshot을 사용합니다. |
| Data/state/DOM/resource owner | Content array가 evidence order를 소유하고 detail view가 반복을 소유합니다. Primitive 기본값이 lazy loading을 결정합니다. |
| Failure·absence·fallback 처리 | 빈 screenshots 배열이면 map 결과가 비어 있지만 section wrapper 자체는 남습니다. Per-image error fallback은 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Hero eager/gallery lazy라는 소비 차이를 만듭니다. Viewport 기반 custom prefetch나 concurrency control은 없습니다. |
| 다음 commit 또는 관련 test 연결 | a00a6bf1af58은 다른 optional media인 profile photo의 consumer contract를 완성합니다. |
<!-- learner:commit-cabf3a0e378f-record:end -->

#### 최소 코드 증거

<!-- learner:commit-cabf3a0e378f-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-cabf3a0e378f-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-cabf3a0e378f-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-cabf3a0e378f-execution:end -->

### 5. `a00a6bf1af58` — feat(about): 프로필 사진 소개 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** optional profile consumer

#### 해당 SHA에서 확인할 실제 코드

- `src/app/about/page.tsx`의 responsive grid 변경과 `content.profile.photo ? <ProfilePhoto .../> : null`을 확인합니다.
- ContentHint path가 photo까지 확장된 이유와 photo absence 시 layout branch를 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-a00a6bf1af58-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | ProfilePhoto primitive는 있었지만 About hero가 photo field를 소비하지 않아 optional profile media contract가 실제 route에서 완결되지 않았습니다. |
| 실제 변경 file/symbol/call path | About hero를 text/photo grid로 바꾸고 photo가 있을 때만 ProfilePhoto를 렌더링합니다. ContentHint path도 photo를 포함합니다. |
| Data/state/DOM/resource owner | Profile content가 presence를 소유하고 About page가 conditional composition을 소유합니다. ProfilePhoto가 img DOM을 소유합니다. |
| Failure·absence·fallback 처리 | Photo가 없으면 null이며 빈 frame이나 synthetic fallback을 만들지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Optional profile photo가 About route에서 안전하게 소비됩니다. Image load 실패나 alt quality는 여전히 content/브라우저 범위입니다. |
| 다음 commit 또는 관련 test 연결 | 이 commit으로 primitive 생성 → lead/detail/gallery/profile consumer 연결이 모두 확인됩니다. |
<!-- learner:commit-a00a6bf1af58-record:end -->

#### 최소 코드 증거

<!-- learner:commit-a00a6bf1af58-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-a00a6bf1af58-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-a00a6bf1af58-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-a00a6bf1af58-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Media metadata 보존 | aa115c73ae30 | ProfilePhoto/ProjectScreenshot의 src·alt 전달 | content validation 없음 | primitive가 원본 metadata를 그대로 DOM에 전달 |
| Above-the-fold priority | b027f42669aa, 06fff9a6e93b | priority prop → eager loading | 실제 network priority 측정 없음 | home/detail hero가 eager를 명시 |
| Evidence gallery lazy default | cabf3a0e378f | priority 없는 ProjectScreenshot 반복 | 빈 section wrapper는 남음 | gallery image는 lazy attribute 사용 |
| Optional photo absence | a00a6bf1af58 | conditional ProfilePhoto | load error fallback 없음 | photo field 부재 시 media DOM 생략 |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| 각 consumer의 직접 img rendering 위험 | loading/crop/alt handling 불일치 | aa115c73ae30 shared primitive | 후속 consumer diff로 사용 확인; dedicated test는 없음 |
| 모든 image를 같은 loading으로 처리 | hero 지연 또는 gallery 과도 eager 가능 | priority prop과 consumer별 선택 | 정적 attribute path만 확인; network 성능 test 없음 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| Content | Image src/alt와 optional presence | Project/profile data object |
| Primitive | img attributes, frame, crop, loading mapping | ProfilePhoto / ProjectScreenshot |
| Consumer | 어디에 배치하고 priority를 켤지 | Home, ProjectDetailView, About |
| Browser | 실제 fetch/decode/error rendering | Application code가 lifetime을 직접 관리하지 않음 |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- ProfilePhoto와 ProjectScreenshot이 raw content media를 공통 DOM contract로 렌더링합니다.
- Home lead와 detail hero는 priority로 eager를 선택하고 detail gallery는 기본 lazy를 사용합니다.
- Screenshot `<img>`의 16:10 aspect ratio가 높이를 계산하게 하고 figure wrapper가 overflow/frame을, image가 object-fit crop을 적용합니다.
- Optional profile photo는 absence 시 완전히 생략됩니다.
- Error fallback, responsive source set, image optimization과 runtime performance measurement는 이 Thread에 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. Content loader가 profile/project image metadata를 제공합니다.
2. Route/view가 media presence와 above-the-fold 여부를 판단합니다.
3. Consumer가 ProfilePhoto 또는 ProjectScreenshot과 optional priority를 호출합니다.
4. Primitive가 frame과 img attributes를 생성합니다.
5. Browser가 eager/lazy hint에 따라 resource를 요청하고 decode합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] 이 Thread에는 S/A-level commit이 없으며 B-level 범위에서 repository-specific depth를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 02-content-media-loading-and-layout-stability.md =====

===== BEGIN FILE: 03-progressive-reveal-to-server-first-rendering.md =====
# Thread: Progressive reveal to server-first rendering

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

IntersectionObserver 기반 reveal primitive가 reduced-motion 범위를 넓힌 뒤, 왜 client lifecycle을 제거하고 server-visible markup으로 전환됐는지 복원합니다.

**경계:** 이 Thread는 shared Reveal의 visibility/lifecycle과 motion fallback을 다룹니다. 개별 section content, route composition, 전역 performance test architecture는 포함하지 않습니다.

### 고정 invariant

- Content는 observer 지원, hydration 완료, animation 실행 여부와 무관하게 읽을 수 있어야 합니다.
- Observer를 사용하는 동안에는 첫 intersect에서만 visible로 전환하고 observer를 해제합니다.
- Reduced-motion에서는 opacity/transform/transition 때문에 content가 숨지 않습니다.
- 최종 상태에서는 Reveal markup이 server에서 이미 visible이며 client effect를 요구하지 않습니다.

## 2. 핵심 질문

- 907d85b77bac의 server initial render와 browser initial state가 실제로 같은가?
- IntersectionObserver 미지원 fallback, first intersection, cleanup은 어떤 branch로 구현되는가?
- 29bb40579cb2와 af9191fc15ad가 reduced-motion 대상과 강도를 어떻게 확장하는가?
- b8164cfdddbd가 어떤 hook/ref/client boundary를 제거하며 delay prop의 의미는 무엇이 남는가?
- 최종 server-first 전환을 직접 검증하는 test가 이 frozen Thread에 존재하는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- S/A-level은 architecture/owner/failure/later evidence를 B-level보다 깊게 복원합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `907d85b77bac` | feat(ui): 뷰포트 진입 공개 효과 추가 | B | - | observer 기반 reveal 도입 |
| 2 | `29bb40579cb2` | style(a11y): 동적 목록의 모션 감소 지원 | B | RENDERER, A11Y | reduced-motion 대상 확장 |
| 3 | `af9191fc15ad` | style(a11y): 모바일 헤더와 동작 감소 보강 | A | ARCH, RENDERER, A11Y | 전역 motion fallback 강화 |
| 4 | `b8164cfdddbd` | refactor(ui): reveal 콘텐츠를 server에서 즉시 표시 | A | ARCH, CONTENT, REFACTOR | server-first visibility 전환 |

## 5. Commit별 학습 기록

### 1. `907d85b77bac` — feat(ui): 뷰포트 진입 공개 효과 추가

- **Importance:** B
- **Tags:** -
- **Thread 역할:** observer 기반 reveal 도입

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/reveal.tsx`의 `useState` initializer를 SSR과 browser에서 각각 평가합니다.
- `useEffect`의 no-node/no-IntersectionObserver/intersection/cleanup branch를 추적합니다.
- `src/app/globals.css`의 hidden, visible, reduced-motion selector를 component class transition과 연결합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-907d85b77bac-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Reveal wrapper가 없어서 content는 즉시 보였고 viewport entry에 맞춘 공통 transition과 cleanup contract가 없었습니다. |
| 실제 변경 file/symbol/call path | Client component `Reveal`을 추가했습니다. State는 browser에 IntersectionObserver가 없을 때만 초기 true이고, effect는 node를 observe해 첫 intersect에서 visible로 바꾸고 disconnect합니다. CSS는 기본 hidden/blur/translate, `is-visible`에서 복구합니다. |
| Data/state/DOM/resource owner | Component가 visible state와 observer lifetime을 소유하고 browser observer가 intersection event를 제공합니다. CSS가 visual transition을 소유합니다. |
| Failure·absence·fallback 처리 | Ref node가 없으면 effect 종료, observer API가 없으면 initializer의 browser state로 visible입니다. Cleanup과 first-intersect 모두 disconnect합니다. SSR에서는 `window`가 없어 initial state가 false입니다. |
| 보장하는 것과 보장하지 않는 것 | Observer 지원 browser에서 first-entry reveal과 cleanup을 보장합니다. JavaScript/hydration 전 server markup은 hidden class 상태일 수 있고 observer callback timing은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 29bb40579cb2·af9191fc15ad가 fallback을 강화하고, b8164cfdddbd가 hydration 의존 자체를 제거합니다. |
<!-- learner:commit-907d85b77bac-record:end -->

#### 최소 코드 증거

<!-- learner:commit-907d85b77bac-excerpt:start -->
- **Commit:** `907d85b77bac`
- **Path:** `src/components/portfolio/reveal.tsx`
- **Location:** `Reveal useEffect`

```tsx
useEffect(() => {
  const node = ref.current;

  if (!node) {
    return;
  }

  if (!("IntersectionObserver" in window)) {
    return;
  }

  const observer = new IntersectionObserver(
    ([entry]) => {
      if (entry.isIntersecting) {
        setVisible(true);
        observer.disconnect();
      }
    },
    { rootMargin: "0px 0px -12% 0px", threshold: 0.12 },
  );

  observer.observe(node);

  return () => observer.disconnect();
}, []);
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-907d85b77bac-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-907d85b77bac-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-907d85b77bac-execution:end -->

### 2. `29bb40579cb2` — style(a11y): 동적 목록의 모션 감소 지원

- **Importance:** B
- **Tags:** RENDERER, A11Y
- **Thread 역할:** reduced-motion 대상 확장

#### 해당 SHA에서 확인할 실제 코드

- `prefers-reduced-motion` block에서 새로 추가된 `.tech-chip`, experience/timeline pseudo-elements를 확인합니다.
- 기존 `.reveal-item`, `.motion-card`와 같은 transform/transition neutralization을 공유하는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-29bb40579cb2-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 초기 reduced-motion block은 Reveal 자체를 즉시 보이게 했지만 이후 생긴 동적 list/chip guide pseudo-elements까지 모두 neutralize하지 않았습니다. |
| 실제 변경 file/symbol/call path | Reduced-motion selector 목록을 tech chip, experience row marker, paired/timeline guide pseudo-elements로 확장해 transform과 transition을 제거했습니다. |
| Data/state/DOM/resource owner | User media preference가 policy input이고 globals.css가 여러 primitive의 motion fallback을 소유합니다. |
| Failure·absence·fallback 처리 | CSS media query가 적용되지 않는 환경에는 영향이 없습니다. Animation property 전체를 전역으로 제한하는 단계는 아직 아닙니다. |
| 보장하는 것과 보장하지 않는 것 | 동적 list의 motion을 줄이지만 client Reveal state/hydration dependency는 그대로입니다. |
| 다음 commit 또는 관련 test 연결 | af9191fc15ad가 wildcard timing clamp와 hover neutralization으로 정책을 더 강하게 만듭니다. |
<!-- learner:commit-29bb40579cb2-record:end -->

#### 최소 코드 증거

<!-- learner:commit-29bb40579cb2-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-29bb40579cb2-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-29bb40579cb2-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-29bb40579cb2-execution:end -->

### 3. `af9191fc15ad` — style(a11y): 모바일 헤더와 동작 감소 보강

- **Importance:** A
- **Tags:** ARCH, RENDERER, A11Y
- **Thread 역할:** 전역 motion fallback 강화

#### 해당 SHA에서 확인할 실제 코드

- `prefers-reduced-motion`의 universal selector와 pseudo-element selector가 animation/transition duration, iteration, scroll behavior를 어떻게 덮는지 확인합니다.
- `.motion-card:hover`, terminal wrap 등 명시적 transform/animation neutralization과 wildcard timing clamp의 역할을 구분합니다.
- 같은 commit의 mobile header backdrop-filter 제거는 motion policy와 별도 responsive performance decision임을 기록합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-af9191fc15ad-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Primitive별 selector만으로는 새 animation이나 hover transform이 빠질 수 있었고, mobile header backdrop blur도 작은 viewport에서 계속 적용됐습니다. |
| 실제 변경 file/symbol/call path | Reduced-motion에서 모든 element와 pseudo-element의 animation duration을 0.01ms, iteration을 1, transition duration을 0.01ms로 강제하고 scroll behavior를 auto로 둡니다. 특정 animated elements와 hover transform도 명시적으로 neutralize합니다. 640px 이하 header backdrop-filter도 제거합니다. |
| Data/state/DOM/resource owner | Global stylesheet가 user preference에 대한 cross-component policy owner가 됩니다. 개별 component는 해당 policy를 반복 구현할 필요가 줄어듭니다. |
| Failure·absence·fallback 처리 | CSS를 비활성화하거나 media query가 일치하지 않으면 적용되지 않습니다. 0.01ms는 animation property를 제거하는 것이 아니라 사실상 즉시 종료시키는 정책입니다. |
| 보장하는 것과 보장하지 않는 것 | 새 motion primitive가 추가돼도 기본 timing clamp를 받게 합니다. 하지만 Reveal의 hidden initial class와 hydration timing 문제 자체를 해결하지는 않습니다. |
| 다음 commit 또는 관련 test 연결 | b8164cfdddbd가 visibility state owner를 client에서 server markup으로 이동해 content availability를 motion과 분리합니다. |
<!-- learner:commit-af9191fc15ad-record:end -->

#### 최소 코드 증거

<!-- learner:commit-af9191fc15ad-excerpt:start -->
- **Commit:** `af9191fc15ad`
- **Path:** `src/app/globals.css`
- **Location:** `universal selector inside prefers-reduced-motion`

```css
*,
*::before,
*::after {
  animation-duration: 0.01ms !important;
  animation-iteration-count: 1 !important;
  scroll-behavior: auto !important;
  transition-duration: 0.01ms !important;
}
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-af9191fc15ad-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-af9191fc15ad-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-af9191fc15ad-execution:end -->

### 4. `b8164cfdddbd` — refactor(ui): reveal 콘텐츠를 server에서 즉시 표시

- **Importance:** A
- **Tags:** ARCH, CONTENT, REFACTOR
- **Thread 역할:** server-first visibility 전환

#### 해당 SHA에서 확인할 실제 코드

- `Reveal`에서 제거된 `"use client"`, hooks, ref, observer와 남은 props/markup을 비교합니다.
- 항상 `reveal-item is-visible`인 server output이 globals.css와 결합해 어떤 state를 제거하는지 확인합니다.
- `delay`가 여전히 inline transitionDelay를 만들지만 viewport entry trigger는 사라졌다는 점을 구분합니다.
- 후속 test가 frozen commit set 안에 있는지 확인하고 없으면 명시합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b8164cfdddbd-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 초기 design은 SSR에서 hidden markup을 만들고 hydration/effect/observer가 성공해야 content를 visible로 바꿨습니다. Progressive enhancement가 아니라 JavaScript availability가 content visibility의 전제가 되는 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | Reveal을 server component-compatible pure function으로 바꿔 hooks/ref/observer/client directive를 제거했습니다. Class는 항상 `reveal-item is-visible`이며 delay style과 `as` polymorphism만 유지합니다. |
| Data/state/DOM/resource owner | Visibility owner가 client state/observer에서 server markup으로 이동했습니다. CSS는 표현만 소유하고 lifecycle resource는 더 이상 없습니다. |
| Failure·absence·fallback 처리 | Observer 미지원, hydration 지연, effect 미실행 branch 자체가 제거됩니다. 별도 cleanup도 필요 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Content가 첫 HTML에서 visible임을 보장하고 이 wrapper 자체의 client boundary와 lifecycle을 제거합니다. 실제 emitted bundle 크기는 측정하지 않았습니다. Viewport 진입 시점 animation은 더 이상 보장하지 않습니다. `delay`는 inline `transitionDelay`로 serialize되지만 component가 visibility class transition을 일으키지는 않습니다. |
| 다음 commit 또는 관련 test 연결 | 이 frozen Thread에는 이 refactor를 실행한 dedicated regression test가 없습니다. Category 07의 별도 performance/regression 분류와 정적 source inspection이 후속 검증 맥락입니다. |
<!-- learner:commit-b8164cfdddbd-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b8164cfdddbd-excerpt:start -->
- **Commit:** `b8164cfdddbd`
- **Path:** `src/components/portfolio/reveal.tsx`
- **Location:** `Reveal after refactor`

```tsx
export function Reveal({
  as = "div",
  children,
  className = "",
  delay = 0,
}: {
  as?: "div" | "li";
  children: React.ReactNode;
  className?: string;
  delay?: number;
}) {
  const Component = as;

  return (
    <Component
      className={`reveal-item is-visible ${className}`}
      style={{ transitionDelay: `${delay}ms` }}
    >
      {children}
    </Component>
  );
}
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-b8164cfdddbd-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b8164cfdddbd-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-b8164cfdddbd-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| First-intersection reveal | 907d85b77bac | IntersectionObserver + is-visible state | SSR hidden/hydration 의존 | 후속 refactor 전까지 browser interaction 제공 |
| Reduced-motion content visibility | 907d85b77bac → af9191fc15ad | component selector에서 global timing clamp까지 확장 | client state는 여전히 필요 | motion preference에서 hidden/long animation 방지 |
| Server-first availability | b8164cfdddbd | client directive/hooks/observer 제거, is-visible 고정 | viewport reveal behavior 포기 | 첫 HTML에서 content visible |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| SSR hidden + hydration/observer 필요 | JS 지연/실패 시 content visibility 위험 | b8164cfdddbd server-visible markup | Frozen Thread 내 dedicated runtime test 없음; exact diff로 정적 확인 |
| Primitive별 reduced-motion 누락 가능 | 새 animation이 preference를 무시할 수 있음 | 29bb40579cb2와 af9191fc15ad에서 범위 확장 | CSS selector inspection; visual browser test 없음 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| 907d 초기 | Reveal client state + IntersectionObserver | visibility와 observer lifetime을 component가 관리 |
| 29bb/af919 | Global CSS | reduced-motion policy가 cross-component owner로 확대 |
| b816 최종 | Server markup | content visibility는 server render가 소유; client lifecycle 제거 |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- Reveal은 server-compatible wrapper이며 첫 markup부터 `is-visible`입니다.
- IntersectionObserver, state, ref, effect, disconnect cleanup은 제거됐습니다.
- Global reduced-motion policy는 animation/transition을 사실상 즉시 종료하고 특정 transform을 neutralize합니다.
- Viewport 진입 기반 reveal은 최종 contract가 아닙니다.
- Frozen Thread 안에는 server-first invariant를 실행 검증하는 dedicated test가 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. Server component가 `Reveal`을 일반 wrapper처럼 호출합니다.
2. Reveal은 선택된 element type에 children, className, delay style을 붙입니다.
3. Markup은 처음부터 `reveal-item is-visible`입니다.
4. Normal motion에서도 component는 visibility class를 전환하지 않습니다. Reduced-motion에서는 global override가 남은 timing/transform declaration을 neutralize합니다.
5. Client observer lifecycle은 존재하지 않습니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] S/A-level 설명을 B-level보다 깊게 작성했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 03-progressive-reveal-to-server-first-rendering.md =====

===== BEGIN FILE: 04-terminal-state-machine-and-motion-fallback.md =====
# Thread: Terminal state machine and motion fallback

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

AnimatedTerminal의 placeholder formatting, typing/hold/erase timer state machine, cleanup, terminal CSS와 reduced-motion behavior, Classic hero integration을 복원합니다.

**경계:** 이 Thread는 terminal preview primitive의 state와 표현을 다룹니다. Terminal content schema validation, home route 전체 section architecture, general motion policy는 각각 다른 Thread의 책임입니다.

### 고정 invariant

- Terminal command output은 profile/project/stack dependency가 바뀔 때 파생되고 state machine은 그 결과 배열을 순환합니다.
- Effect 실행마다 최대 한 timeout을 소유하며 dependency 변화·unmount에서 clear합니다.
- Reduced-motion에서는 timer progression을 시작하지 않고 읽을 수 있는 초기 command/output을 유지합니다.
- CSS animation은 reduced-motion에서 비활성화됩니다.
- Consumer는 terminal commands가 비어 있지 않다는 전제를 제공합니다.

## 2. 핵심 질문

- formatTerminalLine이 어떤 placeholder만 치환하며 unknown placeholder는 어떻게 되는가?
- 초기 commandIndex/typedCommand/phase가 reduced-motion early return과 결합해 무엇을 표시하는가?
- typing/hold/erase 각 branch의 timeout과 다음 state, cleanup은 무엇인가?
- CSS frame/sheen/output/caret animation이 어느 commit에 나뉘어 추가되는가?
- 빈 commands 배열에 대한 guard가 실제로 존재하는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- 이 Thread는 B-level commit만 포함하므로 각 commit의 concrete role, boundary, failure/non-guarantee를 필요한 범위로 기록합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `f60d46857715` | feat(home): 애니메이션 터미널 상호작용 추가 | B | RENDERER | timer state machine 도입 |
| 2 | `1ff1da788f7a` | style(home): 터미널 프레임과 부유 장식 추가 | B | RENDERER | frame/sheen 표현 |
| 3 | `335a00fcf40c` | style(home): 터미널 출력과 커서 동작 추가 | B | RENDERER | output/caret motion |
| 4 | `cdb68fdf59f9` | feat(home): 클래식 홈 히어로 구성 | B | RENDERER | Classic hero integration |

## 5. Commit별 학습 기록

### 1. `f60d46857715` — feat(home): 애니메이션 터미널 상호작용 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** timer state machine 도입

#### 해당 SHA에서 확인할 실제 코드

- `formatTerminalLine`의 replacement key와 `useMemo` dependency를 확인합니다.
- `commandIndex`, `typedCommand`, `phase` 초기값과 `activeCommand` access를 확인합니다.
- typing/hold/erase branch별 delay와 state update, modulo progression, cleanup을 표로 재구성합니다.
- `matchMedia(prefers-reduced-motion)` early return에서 initial DOM이 무엇인지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-f60d46857715-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Classic home에 data-driven terminal preview와 timer lifecycle이 없었습니다. |
| 실제 변경 file/symbol/call path | `AnimatedTerminal`을 client component로 추가했습니다. `formatTerminalLine`은 `{handle}`, `{role}`, `{location}`, `{projectCount}`, `{stackCount}`를 `replaceAll`로 치환하고 unknown token은 그대로 둡니다. 그 결과 배열 위에 index/typed text/phase state를 두며, effect는 typing 42ms, completed typing 후 520ms, hold 1700ms, erase 24ms, 다음 command 전 220ms timeout을 사용하고 cleanup에서 clear합니다. |
| Data/state/DOM/resource owner | Component가 formatted commands, state, timeout handle을 소유합니다. Content가 command/output template을, browser timer가 scheduling을 소유합니다. |
| Failure·absence·fallback 처리 | Reduced-motion match이면 effect가 timeout을 만들지 않습니다. Cleanup은 dependency change/unmount에서 pending timeout을 해제합니다. 그러나 empty commands를 막는 guard가 없어 `activeCommand.command` access와 modulo length가 non-empty input을 전제로 합니다. |
| 보장하는 것과 보장하지 않는 것 | 정상 non-empty input에서 순환 typing state와 timer cleanup을 보장합니다. Exact wall-clock timing, background-tab throttling, empty schema recovery는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 1ff1da788f7a·335a00fcf40c가 visual frame/output motion을 추가하고 cdb68fdf59f9가 real content consumer를 연결합니다. |
<!-- learner:commit-f60d46857715-record:end -->

#### 최소 코드 증거

<!-- learner:commit-f60d46857715-excerpt:start -->
- **Commit:** `f60d46857715`
- **Path:** `src/components/portfolio/animated-terminal.tsx`
- **Location:** `AnimatedTerminal effect`

```tsx
if (phase === "erase") {
  if (typedCommand.length > 0) {
    timeout = setTimeout(() => {
      setTypedCommand(activeCommand.command.slice(0, typedCommand.length - 1));
    }, 24);
  } else {
    timeout = setTimeout(() => {
      setCommandIndex((current) => (current + 1) % commands.length);
      setPhase("typing");
    }, 220);
  }
}

return () => clearTimeout(timeout);
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-f60d46857715-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-f60d46857715-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-f60d46857715-execution:end -->

### 2. `1ff1da788f7a` — style(home): 터미널 프레임과 부유 장식 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** frame/sheen 표현

#### 해당 SHA에서 확인할 실제 코드

- globals.css의 `.hero-terminal-wrap`, `.terminal-window`, titlebar/body와 `terminal-sheen` keyframes를 확인합니다.
- decorative pseudo-elements의 pointer-events와 reduced-motion selector가 어떤 animation을 끄는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-1ff1da788f7a-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | State machine은 text DOM만 제공했고 terminal frame, titlebar, sheen, floating decoration이 없었습니다. |
| 실제 변경 file/symbol/call path | Terminal wrapper/frame/titlebar/body CSS와 pseudo-element decoration, sheen keyframes를 추가했습니다. Decorative layers는 pointer-events none입니다. |
| Data/state/DOM/resource owner | CSS가 visual lifetime과 stacking을 소유하고 component state에는 변화가 없습니다. |
| Failure·absence·fallback 처리 | Reduced-motion에서 terminal window sheen은 끄지만 이 commit 시점의 selector가 모든 wrapper animation을 포함하는지는 제한적입니다. 후속 global policy가 보강합니다. |
| 보장하는 것과 보장하지 않는 것 | Terminal chrome과 decoration을 제공하지만 typing state correctness나 accessibility semantics를 바꾸지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 335a00fcf40c가 output/caret motion을 추가하고 af9191fc15ad가 더 넓은 reduced-motion policy를 적용합니다. |
<!-- learner:commit-1ff1da788f7a-record:end -->

#### 최소 코드 증거

<!-- learner:commit-1ff1da788f7a-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-1ff1da788f7a-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-1ff1da788f7a-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-1ff1da788f7a-execution:end -->

### 3. `335a00fcf40c` — style(home): 터미널 출력과 커서 동작 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** output/caret motion

#### 해당 SHA에서 확인할 실제 코드

- `.terminal-line`, `.terminal-output`, pseudo marker, `.terminal-caret`와 keyframes를 확인합니다.
- Reduced-motion selector가 output/caret animation을 모두 끄는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-335a00fcf40c-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Terminal frame은 있었지만 long line wrapping, output entry effect와 caret indicator가 없었습니다. |
| 실제 변경 file/symbol/call path | Anywhere wrapping, output marker/entry animation, blinking caret와 관련 keyframes를 추가했습니다. |
| Data/state/DOM/resource owner | CSS가 transient visual effect를 소유하고 React state는 typed text/output visibility만 소유합니다. |
| Failure·absence·fallback 처리 | Reduced-motion에서 output과 caret animation을 none으로 둡니다. Caret는 aria-hidden이라 assistive text에 포함되지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Long terminal line의 overflow risk를 줄이고 visual feedback을 제공합니다. Screen-reader live announcement나 typing narration은 제공하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | cdb68fdf59f9가 terminal component를 Classic hero에 배치합니다. |
<!-- learner:commit-335a00fcf40c-record:end -->

#### 최소 코드 증거

<!-- learner:commit-335a00fcf40c-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-335a00fcf40c-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-335a00fcf40c-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-335a00fcf40c-execution:end -->

### 4. `cdb68fdf59f9` — feat(home): 클래식 홈 히어로 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** Classic hero integration

#### 해당 SHA에서 확인할 실제 코드

- `src/designs/classic/home-route.tsx`의 `ClassicHeroSection`에서 AnimatedTerminal props를 추적합니다.
- profile, projects.length, techStack.length와 presentation terminal content가 formatting input으로 전달되는지 확인합니다.
- ContentHint path와 terminal placement wrapper를 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-cdb68fdf59f9-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Terminal primitive와 CSS는 있었지만 실제 Classic home content/data counts를 전달하는 route consumer가 없었습니다. |
| 실제 변경 file/symbol/call path | Classic hero가 AnimatedTerminal에 profile, project count, stack count, presentation terminal object를 전달하고 terminal wrapper에 배치합니다. |
| Data/state/DOM/resource owner | Route가 content aggregation과 placement를, terminal component가 formatting/state를 소유합니다. |
| Failure·absence·fallback 처리 | Route는 commands empty를 별도 검사하지 않습니다. Content contract가 valid/non-empty terminal data를 제공해야 합니다. |
| 보장하는 것과 보장하지 않는 것 | 실제 portfolio data를 terminal state machine에 연결합니다. 다른 design template에서의 사용이나 schema validation은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 이 commit으로 content → formatting → state machine → CSS frame의 실행 경로가 완성됩니다. |
<!-- learner:commit-cdb68fdf59f9-record:end -->

#### 최소 코드 증거

<!-- learner:commit-cdb68fdf59f9-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-cdb68fdf59f9-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-cdb68fdf59f9-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-cdb68fdf59f9-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Non-empty cyclic command state | f60d46857715 | phase별 timeout과 modulo index | empty commands guard 없음 | valid content에서 typing/hold/erase 반복 |
| Timer cleanup | f60d46857715 | effect cleanup clearTimeout | browser throttling은 제어하지 않음 | dependency change/unmount 시 pending timer 해제 |
| Reduced-motion readable initial state | f60d46857715 + CSS commits | matchMedia early return, animation none | 실제 OS/browser matrix 미실행 | 첫 full command/output 유지, visual animation 중단 |
| Real data integration | cdb68fdf59f9 | ClassicHeroSection props | schema validation은 외부 | profile/count/terminal content를 실제 consumer가 전달 |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| Timer effect가 cleanup되지 않을 위험 | unmount 후 state update 또는 중복 timer | f60d46857715 cleanup에서 clearTimeout | Dedicated fake-timer test는 frozen Thread에 없음 |
| Motion preference 무시 위험 | typing/CSS animation 지속 | effect early return + CSS animation none | 정적 branch 확인; browser preference test 없음 |
| Empty command input | activeCommand undefined 및 modulo zero 위험 | 이 Thread에서 fix 없음 | Content validation이 전제이며 regression test 없음 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| Content | Terminal command/output templates | presentation data |
| Route | profile·projectCount·stackCount 전달과 placement | ClassicHeroSection |
| AnimatedTerminal | formatted command array, phase/index/text state, timeout cleanup | client component |
| CSS | frame, sheen, output, caret, reduced-motion visual fallback | globals.css |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- Classic hero가 content-driven terminal preview를 렌더링합니다.
- State machine은 typing → hold → erase → next command를 반복하고 pending timeout을 cleanup합니다.
- Reduced-motion이면 timer progression을 시작하지 않고 CSS animations도 중단됩니다.
- Formatting은 정해진 placeholder만 replaceAll하며 unknown token은 그대로 남습니다.
- Commands empty recovery와 dedicated timer/browser test는 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. ClassicHeroSection이 profile/counts/terminal content를 AnimatedTerminal에 전달합니다.
2. useMemo가 각 output line의 known placeholder를 실제 값으로 치환합니다.
3. State가 active command와 phase를 선택합니다.
4. Effect가 motion preference를 확인하고 phase에 맞는 단일 timeout을 예약합니다.
5. Render는 typed command와 조건부 output을 terminal DOM에 놓고 CSS가 frame/motion을 표현합니다.
6. Effect rerun 또는 unmount 시 pending timeout을 clear합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] 이 Thread에는 S/A-level commit이 없으며 B-level 범위에서 repository-specific depth를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 04-terminal-state-machine-and-motion-fallback.md =====

===== BEGIN FILE: 05-technology-stack-icon-list-and-marquee.md =====
# Thread: Technology stack icon, list, and marquee

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

simple-icons mapping과 handcrafted fallback, ID 기반 stack list, duplicated-track marquee, hover pause와 reduced-motion fallback을 순서대로 복원합니다.

**경계:** 이 Thread는 stack visual primitives를 다룹니다. Tech stack content schema와 `resolveTechStackItem`의 validation/fallback source, home section 전체 composition은 다른 Thread가 소유합니다.

### 고정 invariant

- Known brand icon은 typed map을 사용하고 map에 없는 supported semantic icon은 local SVG fallback을 사용합니다.
- StackList는 content ID order를 보존하고 optional limit은 앞에서부터 slice합니다.
- Marquee는 assistive technology에 동일 항목을 두 번 노출하지 않습니다.
- Continuous motion은 hover에서 pause하고 reduced-motion에서 중단됩니다.

## 2. 핵심 질문

- Partial Record를 사용한 이유와 map miss가 runtime에서 어떤 branch로 이어지는가?
- FallbackIcon의 named branches와 final generic circle이 어떤 unsupported state를 표현하는가?
- StackList가 ID를 resolver로 넘기고 color CSS variable을 만드는 responsibility chain은 무엇인가?
- Marquee가 track을 복제하면서 key와 aria-hidden을 어떻게 다르게 만드는가?
- 빈 items, 18개 초과 items, hover/reduced-motion에서 실제 contract는 무엇인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- 이 Thread는 B-level commit만 포함하므로 각 commit의 concrete role, boundary, failure/non-guarantee를 필요한 범위로 기록합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `ebc245105c03` | feat(stack): 기술 스택 아이콘 매핑 추가 | B | RENDERER | brand icon map 도입 |
| 2 | `3d9b847d8094` | feat(stack): 기술 스택 폴백 아이콘 추가 | B | RENDERER | icon renderer와 fallback |
| 3 | `6aa8ee3b90b1` | feat(stack): 공용 기술 스택 목록 추가 | B | RENDERER | ID 기반 chip list |
| 4 | `48559efebf68` | feat(stack): 기술 스택 마키 프리미티브 추가 | B | RENDERER | 복제 track 구조 |
| 5 | `3b9c1a636356` | style(stack): 기술 스택 마키 동작 추가 | B | RENDERER | continuous motion과 fallback |

## 5. Commit별 학습 기록

### 1. `ebc245105c03` — feat(stack): 기술 스택 아이콘 매핑 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** brand icon map 도입

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/tech-icon.tsx`의 simple-icons imports와 `Partial<Record<TechStackIcon, SimpleIcon>>`을 확인합니다.
- Union 전체를 강제하지 않는 partial map이 후속 fallback 필요성을 어떻게 남기는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-ebc245105c03-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Tech stack item의 icon identifier를 SVG path로 바꾸는 공용 mapping이 없었습니다. |
| 실제 변경 file/symbol/call path | C/CMake/C++/Docker/ESLint/Next/Node/PostgreSQL/Prisma/React/Redis/Tailwind/TypeScript/Vitest를 simple-icons object에 매핑하는 partial record를 추가했습니다. |
| Data/state/DOM/resource owner | TechStackIcon identifier는 content/types가 소유하고, map은 brand SVG source 선택을 소유합니다. |
| Failure·absence·fallback 처리 | Map miss를 처리하는 renderer는 아직 없으므로 이 commit만으로는 icon output을 만들지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Known brand identifiers의 source를 중앙화합니다. 모든 union member를 simple-icons가 지원한다는 보장은 의도적으로 하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 3d9b847d8094가 map miss와 semantic icon을 처리하는 TechIcon/FallbackIcon을 추가합니다. |
<!-- learner:commit-ebc245105c03-record:end -->

#### 최소 코드 증거

<!-- learner:commit-ebc245105c03-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-ebc245105c03-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-ebc245105c03-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-ebc245105c03-execution:end -->

### 2. `3d9b847d8094` — feat(stack): 기술 스택 폴백 아이콘 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** icon renderer와 fallback

#### 해당 SHA에서 확인할 실제 코드

- `TechIcon`의 simpleIcon truthy branch와 fallback SVG branch를 확인합니다.
- `FallbackIcon`의 terminal/shield/check/database/flow/box/api/json/default branches를 inventory합니다.
- `aria-hidden`과 color/currentColor 사용을 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-3d9b847d8094-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Brand mapping은 있었지만 map miss를 실제 SVG로 표현하거나 public icon component로 렌더링하는 path가 없었습니다. |
| 실제 변경 file/symbol/call path | `TechIcon`은 mapped simple icon path를 채우고, miss이면 `FallbackIcon`이 named line icon 또는 generic circle/dot을 반환합니다. Outer svg는 decorative `aria-hidden`입니다. |
| Data/state/DOM/resource owner | TechIcon이 branch 선택과 SVG wrapper를, FallbackIcon이 semantic path를 소유합니다. Label text는 caller가 별도로 제공합니다. |
| Failure·absence·fallback 처리 | Unknown-to-map icon도 generic fallback으로 DOM을 만듭니다. Invalid color는 직접 검증하지 않으며 line icons는 currentColor를 사용합니다. |
| 보장하는 것과 보장하지 않는 것 | Supported union icon이 빈 자리 없이 시각 표시를 갖게 합니다. Icon만으로 accessible name을 제공하지는 않으며 caller label이 필요합니다. |
| 다음 commit 또는 관련 test 연결 | 6aa8ee3b90b1이 label과 icon을 함께 제공하는 StackList를 만듭니다. |
<!-- learner:commit-3d9b847d8094-record:end -->

#### 최소 코드 증거

<!-- learner:commit-3d9b847d8094-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-3d9b847d8094-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-3d9b847d8094-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-3d9b847d8094-execution:end -->

### 3. `6aa8ee3b90b1` — feat(stack): 공용 기술 스택 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** ID 기반 chip list

#### 해당 SHA에서 확인할 실제 코드

- `StackList`의 optional limit branch와 `resolveTechStackItem(item)` call을 확인합니다.
- key가 raw ID인지, CSS variable color와 TechIcon/label이 어떻게 연결되는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-6aa8ee3b90b1-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Icon renderer는 있었지만 project stack ID array를 ordered chips로 바꾸는 공용 list가 없었습니다. |
| 실제 변경 file/symbol/call path | Items를 optional `slice(0, limit)`한 뒤 각 ID를 resolver로 해석하고, color CSS variable, TechIcon, label을 가진 list item을 생성합니다. |
| Data/state/DOM/resource owner | Caller가 ID order와 limit을, resolver가 canonical stack metadata를, StackList가 chip DOM을 소유합니다. |
| Failure·absence·fallback 처리 | Empty items는 빈 `<ul>`을 만듭니다. Missing ID의 처리 방식은 `resolveTechStackItem`에 위임되며 이 component가 catch하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Content order와 limit을 보존하는 공용 chip list를 제공합니다. Resolver correctness와 duplicate ID 제거는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 48559efebf68은 full TechStackItem array를 사용하는 별도 continuous marquee primitive를 추가합니다. |
<!-- learner:commit-6aa8ee3b90b1-record:end -->

#### 최소 코드 증거

<!-- learner:commit-6aa8ee3b90b1-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-6aa8ee3b90b1-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-6aa8ee3b90b1-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-6aa8ee3b90b1-execution:end -->

### 4. `48559efebf68` — feat(stack): 기술 스택 마키 프리미티브 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** 복제 track 구조

#### 해당 SHA에서 확인할 실제 코드

- `TechMarquee`의 `slice(0, 18)`과 두 `TechMarqueeTrack` call을 확인합니다.
- 두 번째 track의 `aria-hidden`과 live/ghost key prefix를 확인합니다.
- Outer aria-label, inner list semantics와 empty input behavior를 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-48559efebf68-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Static chip list만 있었고 continuous scrolling을 위한 duplicate sequence structure가 없었습니다. |
| 실제 변경 file/symbol/call path | 최대 18개를 선택해 동일 track을 두 번 렌더링합니다. 두 번째 track은 `aria-hidden`이고 key에는 ghost prefix를 사용해 React identity를 분리합니다. |
| Data/state/DOM/resource owner | TechMarquee가 visible subset과 track duplication을, each track이 list DOM을 소유합니다. CSS가 실제 movement를 소유할 예정입니다. |
| Failure·absence·fallback 처리 | 18개를 넘는 items는 잘립니다. Empty input이면 두 empty list가 남습니다. Duplicate semantic announcement는 ghost track의 aria-hidden으로 막습니다. |
| 보장하는 것과 보장하지 않는 것 | Seamless animation을 위한 DOM과 accessibility boundary를 만듭니다. 이 commit만으로는 track이 움직이지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 3b9c1a636356이 continuous CSS, hover pause와 reduced-motion fallback을 추가합니다. |
<!-- learner:commit-48559efebf68-record:end -->

#### 최소 코드 증거

<!-- learner:commit-48559efebf68-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-48559efebf68-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-48559efebf68-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-48559efebf68-execution:end -->

### 5. `3b9c1a636356` — style(stack): 기술 스택 마키 동작 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** continuous motion과 fallback

#### 해당 SHA에서 확인할 실제 코드

- `.stack-marquee-viewport`, 두 track의 width/gap, `stack-scroll` transform을 확인합니다.
- Hover pause selector와 reduced-motion `animation: none`을 확인합니다.
- Mask/overflow가 content clipping과 visual fade를 어떻게 만드는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-3b9c1a636356-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Duplicate tracks는 정적이었고 viewport clipping, movement, pause/fallback policy가 없었습니다. |
| 실제 변경 file/symbol/call path | Viewport를 max-content flex로 만들고 각 track에 34s linear infinite animation을 적용합니다. Keyframe은 한 track 폭과 gap만큼 왼쪽 이동하며 hover에서 pause하고 reduced-motion에서 animation을 제거합니다. |
| Data/state/DOM/resource owner | CSS가 visual progression을 소유하고 DOM duplication/semantics는 component가 소유합니다. |
| Failure·absence·fallback 처리 | Reduced-motion이면 두 track이 정지해 duplicate visual rows가 나란히 남을 수 있지만 ghost track은 assistive tree에서 계속 숨겨집니다. |
| 보장하는 것과 보장하지 않는 것 | Normal motion의 continuous loop, hover pause, reduced-motion stop을 제공합니다. Exact seamless pixel continuity와 frame-rate 성능은 runtime test로 검증되지 않았습니다. |
| 다음 commit 또는 관련 test 연결 | 이 commit으로 content metadata → icon fallback → list/marquee DOM → motion policy 흐름이 완성됩니다. |
<!-- learner:commit-3b9c1a636356-record:end -->

#### 최소 코드 증거

<!-- learner:commit-3b9c1a636356-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-3b9c1a636356-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-3b9c1a636356-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-3b9c1a636356-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Known brand icon | ebc245105c03 | simple-icons partial map | map miss renderer 부재 | 후속 TechIcon이 mapped path 사용 |
| Map miss fallback | 3d9b847d8094 | named SVG branches + generic circle | accessible name은 caller label 의존 | visual 빈 자리 방지 |
| Ordered stack chips | 6aa8ee3b90b1 | slice + resolveTechStackItem + key | missing ID behavior는 resolver 범위 | ID order와 limit 보존 |
| Duplicate track semantics | 48559efebf68 | live/ghost tracks, aria-hidden | empty input null 처리 없음 | 중복 시각 track을 assistive tree에서 한 번만 노출 |
| Motion fallback | 3b9c1a636356 | hover pause/reduced-motion none | browser visual performance 미검증 | continuous motion을 사용자 preference에 맞게 중단 |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| Partial brand map | Map miss 시 icon 부재 | 3d9b847d8094 fallback renderer | Dedicated icon snapshot/test 없음 |
| Seamless loop를 위한 duplicate DOM | Screen reader 중복 발표 위험 | 48559efebf68 ghost track aria-hidden | 정적 DOM semantics 확인; AT runtime test 없음 |
| Continuous marquee | motion sensitivity 및 interaction 방해 | 3b9c1a636356 hover pause/reduced-motion stop | CSS inspection; visual regression test 없음 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| Content/types | stack ID, label, color, icon identifier | TechStackItem |
| TechIcon | simple-icons vs fallback SVG 선택 | decorative icon DOM |
| StackList | ID resolve, optional limit, chip DOM | static list |
| TechMarquee | 18-item cap, duplicate tracks, aria-hidden | continuous list structure |
| CSS | scroll, mask, hover pause, reduced-motion | visual behavior |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- Known brand icon은 simple-icons path를, 나머지 supported icon은 named/generic fallback을 사용합니다.
- StackList는 ID order와 optional limit을 보존한 accessible text chips를 만듭니다.
- TechMarquee는 최대 18개를 live/ghost track으로 복제하고 ghost를 aria-hidden 처리합니다.
- Marquee는 normal motion에서 순환하고 hover에서 pause하며 reduced-motion에서 정지합니다.
- Missing ID validation, empty marquee suppression, runtime visual/AT test는 이 Thread에 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. Caller가 stack IDs 또는 resolved TechStackItem array를 전달합니다.
2. Static list는 resolver로 canonical metadata를 얻고, marquee는 앞 18개를 선택합니다.
3. TechIcon이 simple-icons map을 조회하고 miss이면 FallbackIcon을 선택합니다.
4. StackList는 한 list를, TechMarquee는 live/aria-hidden ghost 두 list를 렌더링합니다.
5. CSS가 color variable과 marquee movement/pause/reduced-motion을 적용합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] 이 Thread에는 S/A-level commit이 없으며 B-level 범위에서 repository-specific depth를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 05-technology-stack-icon-list-and-marquee.md =====

===== BEGIN FILE: 06-project-card-composition-and-interaction.md =====
# Thread: Project card composition and interaction

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

AvailabilityBadge와 ProjectCard가 screenshot, stack, highlights, action links를 어떻게 조립하고, featured consumer 및 hover/reduced-motion 표현까지 확장되는지 복원합니다.

**경계:** Phase 1에서 기존 `project-card-actions-and-evidence-components` Thread의 link-policy commits를 01 Thread로 이동했습니다. 이 Thread는 card composition과 interaction만 소유하며 link placement/transport는 01 Thread의 selector·renderer를 소비합니다.

### 고정 invariant

- Badge 표시 여부와 tone은 deployment data에서 파생되고 showBadge=false이면 DOM을 만들지 않습니다.
- ProjectCard는 evidence primitives를 조립하지만 link availability와 transport를 다시 구현하지 않습니다.
- Case-study primary navigation은 current template/debug state를 보존합니다.
- Featured/default variant는 layout와 stack limit만 바꾸고 data semantics는 바꾸지 않습니다.
- Hover overlay는 pointer event를 가로채지 않고 reduced-motion에서는 card transform/transition이 제거됩니다.

## 2. 핵심 질문

- AvailabilityBadge가 showBadge, status tone, `isProjectLive`를 어떻게 조합하는가?
- ProjectCard가 직접 만드는 case-study link와 ProjectCardLinks action group의 책임 차이는 무엇인가?
- featured variant가 screenshot priority, grid, stack limit, highlights count에 미치는 영향은 무엇인가?
- Design featured section이 몇 개의 card를 어떤 variant로 소비하는가?
- CSS pseudo overlay가 interaction을 방해하지 않도록 하는 rule과 reduced-motion fallback은 무엇인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- 이 Thread는 B-level commit만 포함하므로 각 commit의 concrete role, boundary, failure/non-guarantee를 필요한 범위로 기록합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `53ca0860ef3a` | feat(project): 프로젝트 배포 상태 배지 추가 | B | RENDERER | availability badge primitive |
| 2 | `b72c52a22690` | feat(project): 프로젝트 카드 프리미티브 추가 | B | RENDERER | card composition owner |
| 3 | `07dd465dbe20` | feat(home): 디자인 대표 프로젝트 섹션 추가 | B | RENDERER | featured section consumer |
| 4 | `4000a8657a62` | style(project): 프로젝트 카드 상호작용 추가 | B | RENDERER | hover/overlay와 motion fallback |

## 5. Commit별 학습 기록

### 1. `53ca0860ef3a` — feat(project): 프로젝트 배포 상태 배지 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** availability badge primitive

#### 해당 SHA에서 확인할 실제 코드

- 새 `availability-badge.tsx`의 early null, statusTone lookup, default tone과 live dot branch를 확인합니다.
- `isProjectLive`가 badge label이 아니라 dot만 제어하는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-53ca0860ef3a-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Project card/detail에서 deployment label과 availability tone을 일관되게 표현하는 primitive가 없었습니다. |
| 실제 변경 file/symbol/call path | `AvailabilityBadge`가 `showBadge` false면 null을 반환하고 status별 class tone을 선택합니다. `isProjectLive`인 경우에만 accent dot을 추가하고 content label을 표시합니다. |
| Data/state/DOM/resource owner | Deployment data가 status/showBadge/label을, badge component가 tone과 dot DOM을 소유합니다. Live 의미 자체는 selector가 소유합니다. |
| Failure·absence·fallback 처리 | Unknown status는 source-only tone으로 fallback합니다. showBadge false면 label도 렌더링하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Deployment badge의 absence/tone/live indicator를 공통화합니다. Status validation과 actual endpoint health는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | b72c52a22690이 ProjectCard 안에서 badge를 소비합니다. |
<!-- learner:commit-53ca0860ef3a-record:end -->

#### 최소 코드 증거

<!-- learner:commit-53ca0860ef3a-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-53ca0860ef3a-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-53ca0860ef3a-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-53ca0860ef3a-execution:end -->

### 2. `b72c52a22690` — feat(project): 프로젝트 카드 프리미티브 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** card composition owner

#### 해당 SHA에서 확인할 실제 코드

- 새 `project-card.tsx`의 `caseStudyHref`, featured branch와 child primitive calls를 확인합니다.
- Screenshot/title가 같은 internal route를 가리키는지, action group은 `ProjectCardLinks`에 위임되는지 확인합니다.
- Stack limit 6/4, highlight slice 2, priority propagation을 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b72c52a22690-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Badge, screenshot, stack list와 action link primitive는 있었지만 project evidence를 한 card contract로 조립하는 component가 없었습니다. |
| 실제 변경 file/symbol/call path | `ProjectCard`가 template/debug-aware case-study href를 만들고 screenshot/title link, ContentHint, category, AvailabilityBadge, summary, StackList, first two highlights, ProjectCardLinks를 조립합니다. Featured variant는 two-column layout과 stack limit 6을 사용하고 default는 limit 4입니다. |
| Data/state/DOM/resource owner | Card가 composition/layout variant와 primary detail route를 소유합니다. Badge, screenshot, stack, action selection/transport는 각 child owner에 위임됩니다. |
| Failure·absence·fallback 처리 | ProjectCard 자체는 missing project field를 검사하지 않습니다. Action links가 비어도 primary screenshot/title case-study navigation은 존재합니다. |
| 보장하는 것과 보장하지 않는 것 | 일관된 project evidence card를 제공합니다. Content validity, detail route existence, link placement rules을 재정의하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 07dd465dbe20이 실제 Design home section에서 featured/default variants를 소비하고 4000a8657a62가 interaction CSS를 추가합니다. |
<!-- learner:commit-b72c52a22690-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b72c52a22690-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-b72c52a22690-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b72c52a22690-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-b72c52a22690-execution:end -->

### 3. `07dd465dbe20` — feat(home): 디자인 대표 프로젝트 섹션 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** featured section consumer

#### 해당 SHA에서 확인할 실제 코드

- `FeaturedProjectsSection`의 section gate와 project slices를 확인합니다.
- 첫 project는 featured variant, 다음 두 project는 default인지와 모든 card에 priority가 전달되는지 확인합니다.
- Reveal wrapper와 SectionHeading이 card primitive의 책임과 섞이지 않는지 구분합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-07dd465dbe20-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | ProjectCard는 있었지만 Design home presentation config가 featured section을 켤 때 실제 card hierarchy를 만드는 consumer가 없었습니다. |
| 실제 변경 file/symbol/call path | Section key가 enabled일 때 first project를 featured variant로, 다음 두 project를 two-column default cards로 렌더링합니다. 세 card 모두 `priority`를 전달하고 Reveal로 감쌉니다. |
| Data/state/DOM/resource owner | Presentation config가 section presence를, home section이 ranking/slice/layout을, card가 내부 evidence composition을 소유합니다. |
| Failure·absence·fallback 처리 | Featured projects가 0~2개여도 map은 가능한 항목만 렌더링합니다. Empty일 때 section heading/wrapper는 남을 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | Top three featured projects의 card composition을 실제 route에 연결합니다. 모든 project image를 lazy로 유지하거나 empty section을 숨기는 것은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 4000a8657a62가 card class를 이용한 hover/overlay behavior를 추가합니다. |
<!-- learner:commit-07dd465dbe20-record:end -->

#### 최소 코드 증거

<!-- learner:commit-07dd465dbe20-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-07dd465dbe20-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-07dd465dbe20-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-07dd465dbe20-execution:end -->

### 4. `4000a8657a62` — style(project): 프로젝트 카드 상호작용 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** hover/overlay와 motion fallback

#### 해당 SHA에서 확인할 실제 코드

- `.motion-card`, `.project-card::before`, `.project-screenshot::after`와 hover selectors를 확인합니다.
- Pseudo-elements의 `pointer-events: none`, children z-index와 reduced-motion transform/transition neutralization을 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-4000a8657a62-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Card markup은 있었지만 lift, ambient overlay, screenshot sheen과 reduced-motion-specific interaction fallback이 없었습니다. |
| 실제 변경 file/symbol/call path | Motion card hover에서 translateY, project card와 screenshot pseudo-element opacity transition을 추가했습니다. Overlay는 pointer-events none이고 real children은 z-index 1입니다. Reduced-motion에서 motion-card transform과 transition을 제거합니다. |
| Data/state/DOM/resource owner | CSS가 transient interaction 표현을 소유하고 semantic links/buttons는 기존 DOM이 소유합니다. |
| Failure·absence·fallback 처리 | Pointer events를 pseudo overlay가 가로채지 않습니다. Reduced-motion은 transform/transition을 끄지만 color hover class 등 비-motion feedback은 남을 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | Card interaction을 강화하면서 click targets와 motion preference를 보존합니다. Keyboard-specific visual regression이나 browser compositing 성능은 검증하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 이 Thread의 최종 card path는 shared child primitives + surface consumer + non-blocking CSS overlay입니다. |
<!-- learner:commit-4000a8657a62-record:end -->

#### 최소 코드 증거

<!-- learner:commit-4000a8657a62-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-4000a8657a62-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-4000a8657a62-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-4000a8657a62-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Badge absence/tone | 53ca0860ef3a | showBadge null, statusTone default, live dot | actual service health 미검증 | deployment presentation을 일관되게 표현 |
| Card composition | b72c52a22690 | ProjectCard child call graph | content validity는 외부 | evidence primitives를 한 component로 조립 |
| Featured consumption | 07dd465dbe20 | first/next slices와 variants | empty section suppression 없음 | Design home top-three hierarchy |
| Non-blocking interaction | 4000a8657a62 | pointer-events none, z-index, reduced-motion | visual/browser test 없음 | hover overlay가 links를 가리지 않음 |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| Card가 link policy를 직접 소유할 위험 | 01 Thread와 중복·불일치 | b72c52a22690은 ProjectCardLinks를 소비만 함 | 09cec616f314이 child link contract를 검증 |
| Decorative overlay가 click target을 가릴 위험 | hover 시 navigation 불가 | 4000a8657a62 pointer-events none + child z-index | 정적 CSS 확인; browser hit-test 미실행 |
| Hover lift가 reduced-motion을 무시 | motion sensitivity | 4000a8657a62 motion-card neutralization | Global a11y policy는 03 Thread에서 추가 보강 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| Content/selectors | Deployment status, featured ordering, link actions | Card가 의미를 재계산하지 않음 |
| AvailabilityBadge | Badge DOM/tone/live dot | Optional status indicator |
| ProjectCard | Evidence composition, primary route, variant layout | Shared child primitives 호출 |
| Home section | Top project slicing과 card variant 선택 | Surface-level composition |
| CSS | Hover lift/overlays/reduced-motion | Semantic interaction을 가리지 않음 |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- ProjectCard는 badge, screenshot, stack, highlights와 action links를 공통 구조로 조립합니다.
- Screenshot와 title은 template/debug-aware case-study route를 직접 제공합니다.
- Action link visibility/transport는 01 Thread의 ProjectCardLinks/ContentLinkView에 위임됩니다.
- Design featured section은 첫 card를 featured, 다음 두 card를 default로 사용합니다.
- Hover overlays는 pointer event를 받지 않고 reduced-motion에서 lift/transition을 제거합니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. Home consumer가 featured projects를 순서대로 선택하고 card variant/priority를 정합니다.
2. ProjectCard가 project data로 primary case-study URL을 만듭니다.
3. Card가 AvailabilityBadge, ProjectScreenshot, StackList, highlights, ProjectCardLinks를 조립합니다.
4. 각 child primitive가 자기 정책/DOM을 처리합니다.
5. CSS가 hover overlay와 lift를 적용하되 pointer events를 통과시키고 reduced-motion에서 movement를 제거합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] 이 Thread에는 S/A-level commit이 없으며 B-level 범위에서 repository-specific depth를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 06-project-card-composition-and-interaction.md =====

===== BEGIN FILE: 07-journey-timeline-primitives-and-responsive-layout.md =====
# Thread: Journey timeline primitives and responsive layout

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

Journey item의 period formatting과 optional case-study link, compact/paired variants, odd-row handling, desktop centerline과 mobile single-column collapse를 실제 history로 복원합니다.

**경계:** 이 Thread는 journey presentation primitive와 responsive CSS를 다룹니다. Journey JSON schema/validation, route section copy, Reveal의 최종 server-first refactor는 각각 content system과 03 Thread가 소유합니다.

### 고정 invariant

- Journey source order는 유지되고 첫 item만 paired timeline의 start card로 분리됩니다.
- 나머지 items는 두 개씩 묶이며 마지막 홀수 item은 단독 row로 손실 없이 렌더링됩니다.
- projectId가 있는 item만 template/debug-aware case-study link를 가집니다.
- Mobile은 같은 semantic ordered list를 단일 column으로 재배치하며 item order를 바꾸지 않습니다.
- Animated flag는 wrapper 선택만 바꾸고 journey data semantics는 바꾸지 않습니다.

## 2. 핵심 질문

- `formatYearMonth`가 ISO date를 검증하지 않고 어떤 substring transform만 수행하는가?
- 첫 item과 projectItems 분리, `chunkPairs`, `is-single` class가 empty/one/even/odd input을 어떻게 처리하는가?
- Compact와 paired variants에서 Reveal wrapper 위치와 delay 계산은 어떻게 다른가?
- Desktop centerline의 three-column grid가 mobile에서 같은 DOM을 어떻게 flatten하는가?
- 03 Thread의 server-first Reveal 전환이 animated flag의 최종 의미를 어떻게 약화시키는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- 이 Thread는 B-level commit만 포함하므로 각 commit의 concrete role, boundary, failure/non-guarantee를 필요한 범위로 기록합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `f60edf14dae0` | feat(journey): 여정 날짜와 카드 프리미티브 추가 | B | RENDERER | entry/card 기반 도입 |
| 2 | `3c0a1154ba08` | feat(journey): 중앙선 여정 목록 추가 | B | RENDERER | paired timeline composition |
| 3 | `b3182e35aed0` | feat(journey): 여정 목록 변형 연결 | B | RENDERER | public variant dispatcher |
| 4 | `377fa128f82b` | style(journey): 여정 타임라인 시각 계층 추가 | B | RENDERER | compact guide/node 표현 |
| 5 | `7995a3cf5435` | style(journey): 데스크톱 중앙선 여정 구성 | B | RENDERER | desktop centerline geometry |
| 6 | `6394188022fd` | style(journey): 모바일 중앙선 여정 구성 | B | RENDERER | mobile single-column collapse |

## 5. Commit별 학습 기록

### 1. `f60edf14dae0` — feat(journey): 여정 날짜와 카드 프리미티브 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** entry/card 기반 도입

#### 해당 SHA에서 확인할 실제 코드

- `journey-list.tsx`의 `formatYearMonth`, `getJourneyPeriod`, `JourneyEntry`, `JourneyCard`, `chunkPairs`를 확인합니다.
- endDate absence/equality, projectId absence, pair slicing의 boundary를 각각 추적합니다.
- `getTemplateHref`에 homeTemplate/contentDebug가 전달되는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-f60edf14dae0-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Journey item을 공통 날짜/본문/card 구조로 표현하거나 두 개씩 묶는 primitive가 없었습니다. |
| 실제 변경 file/symbol/call path | Date를 YYYY.MM으로 자르고, 동일하거나 없는 endDate는 단일 period, 다른 endDate는 range로 만듭니다. Entry는 category/title/body와 optional project link를 렌더링하고 JourneyCard가 paired card wrapper를 제공합니다. `chunkPairs`는 index를 2씩 증가해 slice합니다. |
| Data/state/DOM/resource owner | Journey data가 date/endDate/projectId를 소유하고 helper가 display string/pair array를 파생합니다. Entry가 semantic content와 optional link를 소유합니다. |
| Failure·absence·fallback 처리 | projectId가 없으면 link는 null입니다. Date format은 `slice(0, 7)` 전제이며 invalid/short input을 reject하지 않습니다. Empty array의 chunk result는 empty입니다. |
| 보장하는 것과 보장하지 않는 것 | Period, optional link, pair chunking의 공통 기반을 제공합니다. Full date validation이나 projectId referential integrity는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 3c0a1154ba08이 first item + paired rows composition을 추가합니다. |
<!-- learner:commit-f60edf14dae0-record:end -->

#### 최소 코드 증거

<!-- learner:commit-f60edf14dae0-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-f60edf14dae0-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-f60edf14dae0-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-f60edf14dae0-execution:end -->

### 2. `3c0a1154ba08` — feat(journey): 중앙선 여정 목록 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** paired timeline composition

#### 해당 SHA에서 확인할 실제 코드

- `PairedJourneyList`의 `[startItem, ...projectItems]`, `chunkPairs`, first-item branch를 확인합니다.
- Pair length 1 class, middle node, optional second card와 key 생성 방식을 확인합니다.
- animated true/false에서 element type과 delay가 어떻게 달라지는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-3c0a1154ba08-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Entry/card helper는 있었지만 첫 milestone을 중심 start로 두고 나머지를 paired centerline rows로 만드는 semantic list가 없었습니다. |
| 실제 변경 file/symbol/call path | 첫 item을 start card로 분리하고 나머지를 pairs로 만듭니다. Pair가 하나뿐이면 `is-single`, second item이 있으면 center node 뒤에 두 번째 card를 둡니다. Animated mode에서는 start/rows를 `Reveal as="li"`로, 아니면 plain li로 렌더링합니다. |
| Data/state/DOM/resource owner | PairedJourneyList가 item grouping, row identity와 animation delay를 소유합니다. JourneyCard는 entry representation을 재사용합니다. |
| Failure·absence·fallback 처리 | Empty input이면 start와 rows 모두 없어 empty `<ol>`이 남습니다. Odd tail은 is-single row로 손실 없이 남고 node는 DOM에 있지만 CSS가 숨길 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | First milestone와 paired rows의 source order 및 odd-tail preservation을 보장합니다. Desktop/mobile geometry는 아직 없습니다. |
| 다음 commit 또는 관련 test 연결 | b3182e35aed0이 public variant dispatcher를, 7995a3cf5435·6394188022fd가 geometry를 추가합니다. |
<!-- learner:commit-3c0a1154ba08-record:end -->

#### 최소 코드 증거

<!-- learner:commit-3c0a1154ba08-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-3c0a1154ba08-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-3c0a1154ba08-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-3c0a1154ba08-execution:end -->

### 3. `b3182e35aed0` — feat(journey): 여정 목록 변형 연결

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** public variant dispatcher

#### 해당 SHA에서 확인할 실제 코드

- Exported `JourneyList`의 defaults와 paired-centerline early return을 확인합니다.
- Compact rows의 animated/plain branches와 outer Reveal wrapper 차이를 확인합니다.
- Index-based delay와 stable key가 source order에 어떻게 연결되는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b3182e35aed0-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Paired renderer는 내부 함수였고 기존 compact list와 하나의 public API에서 선택할 variant contract가 없었습니다. |
| 실제 변경 file/symbol/call path | `JourneyList`가 default compact, optional paired-centerline, animated flag를 받습니다. Paired이면 전용 component에 위임하고, compact이면 각 row를 Reveal/plain li로 만든 뒤 animated일 때 list 전체도 Reveal wrapper로 감쌉니다. |
| Data/state/DOM/resource owner | Public component가 variant dispatch를 소유하고 각 variant가 DOM geometry를 소유합니다. Data order는 caller array와 map order가 소유합니다. |
| Failure·absence·fallback 처리 | Unknown variant는 TypeScript union 밖이며 runtime에서는 compact branch로 떨어집니다. Empty compact input도 empty ordered list wrapper를 반환합니다. |
| 보장하는 것과 보장하지 않는 것 | 한 API로 compact/paired 및 animation wrapper를 선택합니다. Variant-specific responsive style은 후속 CSS가 필요합니다. |
| 다음 commit 또는 관련 test 연결 | 377fa128f82b가 compact timeline visual hierarchy를, 7995a3cf5435/6394188022fd가 paired responsive layout을 추가합니다. |
<!-- learner:commit-b3182e35aed0-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b3182e35aed0-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-b3182e35aed0-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b3182e35aed0-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-b3182e35aed0-execution:end -->

### 4. `377fa128f82b` — style(journey): 여정 타임라인 시각 계층 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** compact guide/node 표현

#### 해당 SHA에서 확인할 실제 코드

- `.timeline-list` gutter/x variables, guide pseudo-element와 `.experience-row::before`를 확인합니다.
- `is-visible` class가 guide scaleY와 node opacity/scale를 어떻게 바꾸는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-377fa128f82b-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Compact ordered list에는 semantic rows만 있고 vertical guide와 per-row node visual hierarchy가 없었습니다. |
| 실제 변경 file/symbol/call path | Timeline wrapper에 vertical gradient guide와 row pseudo nodes를 추가하고 `is-visible`에 맞춰 scale/opacity transition을 적용했습니다. |
| Data/state/DOM/resource owner | CSS가 visual guide/node를 소유하고 ordered list/entry semantics는 component가 소유합니다. |
| Failure·absence·fallback 처리 | Guide/node는 pseudo-elements라 accessibility tree에 추가되지 않습니다. Reveal class가 visible 전환을 제공한다는 당시 전제에 의존합니다. |
| 보장하는 것과 보장하지 않는 것 | Compact timeline의 시각적 연결을 제공합니다. Server-first Reveal 전환 뒤에는 entry-on-viewport timing을 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 29bb40579cb2/af9191fc15ad의 reduced-motion 정책과 b8164cfdddbd의 server-first refactor가 cross-thread로 이 class behavior를 바꿉니다. |
<!-- learner:commit-377fa128f82b-record:end -->

#### 최소 코드 증거

<!-- learner:commit-377fa128f82b-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-377fa128f82b-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-377fa128f82b-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-377fa128f82b-execution:end -->

### 5. `7995a3cf5435` — style(journey): 데스크톱 중앙선 여정 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** desktop centerline geometry

#### 해당 SHA에서 확인할 실제 코드

- `.paired-timeline::before`, start marker/card, three-column row와 center node를 확인합니다.
- `.is-single` row가 one-column centered card로 바뀌고 node를 숨기는 rule을 확인합니다.
- Classic template의 card shadow override를 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-7995a3cf5435-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Paired DOM은 있었지만 centerline, first milestone, two-sided card grid와 odd-tail layout이 시각적으로 정의되지 않았습니다. |
| 실제 변경 file/symbol/call path | 50% centerline, centered start card/marker, `1fr 2.25rem 1fr` row grid, center node를 추가합니다. Single row는 one-column centered max-width card가 되고 node는 display none입니다. |
| Data/state/DOM/resource owner | CSS가 desktop geometry와 visual centerline을 소유합니다. DOM grouping/order는 PairedJourneyList가 소유합니다. |
| Failure·absence·fallback 처리 | Odd tail은 centerline 양쪽을 억지로 채우지 않고 centered single card가 됩니다. Narrow viewport fallback은 아직 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Desktop에서 first + paired + single-tail hierarchy를 보장합니다. Small-screen readability는 다음 commit이 처리합니다. |
| 다음 commit 또는 관련 test 연결 | 6394188022fd가 767px 이하에서 같은 DOM을 single-column timeline으로 collapse합니다. |
<!-- learner:commit-7995a3cf5435-record:end -->

#### 최소 코드 증거

<!-- learner:commit-7995a3cf5435-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-7995a3cf5435-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-7995a3cf5435-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-7995a3cf5435-execution:end -->

### 6. `6394188022fd` — style(journey): 모바일 중앙선 여정 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** mobile single-column collapse

#### 해당 SHA에서 확인할 실제 코드

- max-width 767px media query에서 centerline x, list padding, start/row grid를 확인합니다.
- Desktop center node를 숨기고 각 card pseudo-node를 만드는 selector를 확인합니다.
- Card border/shadow/radius/text alignment가 mobile에서 어떻게 단순화되는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-6394188022fd-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Desktop three-column paired layout은 작은 viewport에서 폭이 부족하고 centerline/marker가 content와 충돌할 수 있었습니다. |
| 실제 변경 file/symbol/call path | Centerline을 left 0.45rem로 옮기고 list에 gutter를 둡니다. Start/paired/single rows를 one-column으로 만들고 center node를 숨긴 뒤 각 card에 left timeline node를 제공합니다. Card는 horizontal border/shadow/radius를 제거하고 full width text-left가 됩니다. |
| Data/state/DOM/resource owner | CSS media query가 visual reflow를 소유하며 DOM과 item order는 바뀌지 않습니다. |
| Failure·absence·fallback 처리 | Safe data fallback을 추가하는 commit은 아닙니다. Mobile에서도 empty list는 empty wrapper이고 invalid date는 그대로 formatting됩니다. |
| 보장하는 것과 보장하지 않는 것 | 같은 ordered data와 links를 모바일 single-column timeline으로 읽을 수 있게 합니다. 실제 device visual regression과 touch behavior는 검증하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 이 commit으로 formatter → entry/card → variants → desktop/mobile presentation 흐름이 완성됩니다. |
<!-- learner:commit-6394188022fd-record:end -->

#### 최소 코드 증거

<!-- learner:commit-6394188022fd-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-6394188022fd-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-6394188022fd-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-6394188022fd-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Period formatting | f60edf14dae0 | slice/replace와 endDate comparison | date validation 없음 | valid YYYY-MM input의 display period |
| Source order와 odd tail 보존 | 3c0a1154ba08 | first split + chunkPairs + is-single | empty wrapper null 처리 없음 | 모든 item을 순서대로 렌더링 |
| Variant dispatch | b3182e35aed0 | paired early return/compact fallback | unknown runtime value는 compact | 한 public API로 두 layout 선택 |
| Desktop centerline | 7995a3cf5435 | three-column rows/single tail | small screen 부적합 | wide viewport geometry |
| Mobile reflow | 6394188022fd | left guide/one-column/card nodes | device visual test 없음 | DOM order를 바꾸지 않는 responsive collapse |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| Odd number of project items | 마지막 item 손실 또는 빈 반대 column 위험 | chunkPairs + is-single + centered card | Dedicated unit test 없음; exact branch inspection |
| Desktop paired grid on mobile | 좁은 card와 centerline 충돌 | 6394188022fd one-column media query | Visual regression test 없음 |
| Reveal is-visible coupling | JS/hydration 의존 시 guide/node visibility 위험 | 03 Thread의 b8164cfdddbd server-visible refactor | Frozen journey commits 자체에는 test 없음 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| Journey data | date/endDate/category/title/body/projectId와 order | Content source |
| Helpers | display period와 pair grouping | formatYearMonth/getJourneyPeriod/chunkPairs |
| JourneyList | variant dispatch, animated wrappers | Public primitive |
| PairedJourneyList | first item, pairs, odd tail, keys/delays | Paired DOM owner |
| CSS | compact guide, desktop centerline, mobile collapse | Responsive presentation |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- JourneyList는 compact와 paired-centerline 두 variant를 제공합니다.
- Paired variant는 첫 item을 start로 두고 나머지를 두 개씩 묶으며 odd tail을 centered single row로 보존합니다.
- projectId가 있는 entry만 template/debug-aware case-study link를 만듭니다.
- Desktop은 centerline two-sided cards, mobile은 같은 DOM의 left-guide single column을 사용합니다.
- Date validation, empty-list suppression, dedicated unit/visual tests는 이 Thread에 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. Caller가 ordered JourneyItem array와 variant/animated options를 전달합니다.
2. JourneyList가 paired variant면 PairedJourneyList로, 아니면 compact map으로 분기합니다.
3. Entry helper가 period/category/title/body와 optional project link를 만듭니다.
4. Paired renderer는 first item을 분리하고 남은 items를 pairs로 묶습니다.
5. CSS가 compact guide 또는 desktop centerline을 적용하고 767px 이하에서 same DOM을 one-column으로 바꿉니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] 이 Thread에는 S/A-level commit이 없으며 B-level 범위에서 repository-specific depth를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 07-journey-timeline-primitives-and-responsive-layout.md =====

===== BEGIN FILE: 08-design-switcher-disclosure-and-responsive-sheet.md =====
# Thread: Design switcher disclosure and responsive sheet

> Project: 42 Archive Portfolio
>
> Branch: `web/portfolio`
>
> Category: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 Phase 1 고정 범위

- Commit SHA, subject, importance와 tags는 branch의 `commit/commit-importance.md` 분류와 exact commit resolution을 대조했습니다.
- 이 문서의 Thread boundary, commit set, order, 역할과 commit-specific investigation task는 Phase 1 audit에서 확정했습니다.
- Phase 2에서는 이 fixed text와 commit metadata를 바꾸지 않고 learner-facing section만 채웁니다.
- 다른 branch와 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표와 경계

Native `<details>/<summary>`를 기반으로 desktop menu와 mobile bottom sheet를 공유하고, active design/count, template-preserving links, explicit close와 focus restoration을 구현한 초기 story를 복원합니다.

**경계:** Phase 1에서 category README가 약속했지만 기존 7개 Thread가 다루지 않던 responsive disclosure primitive 구현 story를 추가했습니다. 실제 shared-shell 연결은 category 02의 `b9571c485013`이, 후속 hydration race fix/test와 server-component refactor는 category 07이 소유합니다. 이 frozen commit map에는 primitive 구현 commit만 두고 인접 category의 integration/correction/evidence는 관계로 명시합니다.

### 고정 invariant

- Disclosure open state는 React boolean이 아니라 native details element가 소유합니다.
- Desktop dropdown과 mobile bottom sheet는 동일한 semantic details/nav/list DOM을 CSS로 재배치합니다.
- Active design은 aria-current와 active class를 가지며 링크는 current path와 content-debug state를 보존합니다.
- 명시적 close button은 open attribute를 제거한 뒤 summary에 focus를 복원합니다.
- 초기 client implementation은 hydration 전에 native open state가 바뀌는 경쟁 조건을 완전히 해결하지 못합니다.

## 2. 핵심 질문

- Desktop CSS가 summary marker/focus와 absolute panel stacking을 어떻게 정의하는가?
- Mobile CSS가 backdrop, safe-area padding, fixed panel과 overscroll을 같은 details open attribute에 연결하는가?
- DesignSwitcher가 active ID miss, content copy miss, count label을 어떻게 fallback/format하는가?
- Close button과 design link click이 open state/focus를 각각 어떻게 처리하는가?
- Category 02의 b9571c485013이 `SiteHeader`에서 어떤 props와 조건으로 이 primitive를 실제 route shell에 연결하는가?
- c702b870d57a → b6c0238ab8b8 → a37cb8596733 → 1ac7813155c6의 후속 correction이 초기 가정을 어떻게 바꾸는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree를 구분해 실제 file/symbol/call path를 기록합니다.
- Previous state, owner, state transition, absence/failure branch, guarantee/non-guarantee를 commit별로 분리합니다.
- Fix와 test는 실제로 수정·검증하는 production path에 연결합니다.
- 실행하지 않은 command 결과를 만들지 않습니다.
- S/A-level은 architecture/owner/failure/later evidence를 B-level보다 깊게 복원합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `cc13abb3b66f` | style(designs): 디자인 선택기 기본 메뉴 구성 | B | RENDERER | desktop disclosure styling |
| 2 | `28dfc5087474` | style(designs): 모바일 디자인 선택 sheet 구성 | B | RENDERER | mobile bottom-sheet styling |
| 3 | `e43e8addd7f3` | feat(designs): 디자인 선택기 상태와 trigger 추가 | B | RENDERER | client details trigger와 active state |
| 4 | `c69ef85c98b2` | feat(designs): 디자인 선택 목록과 닫기 동작 추가 | A | ARCH, RENDERER | nav list와 explicit state transition |

### 관련 commit — 이 frozen map 밖의 경계

아래 commit은 같은 production story의 integration/correction/evidence이지만 인접 category가 소유합니다. Category 중복을 피하기 위해 이 Thread의 commit map에는 넣지 않습니다.

- `b9571c485013` — feat(shell): 디자인 선택기를 공용 shell에 연결 (Category 02)
- `c702b870d57a` — fix(ui): hydration 중 native details 상태 보존 (Category 07)
- `b6c0238ab8b8` — test(ui): details hydration 경쟁 조건 검증 (Category 07)
- `a37cb8596733` — refactor(ui): 디자인 선택기를 server markup으로 전환 (Category 07)
- `1ac7813155c6` — test(ui): server 선택기와 focus 복원 검증 (Category 07)

## 5. Commit별 학습 기록

### 1. `cc13abb3b66f` — style(designs): 디자인 선택기 기본 메뉴 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** desktop disclosure styling

#### 해당 SHA에서 확인할 실제 코드

- 새 `design-switcher.module.css`의 root z-index, summary marker removal/focus-visible, absolute panel와 list/link classes를 확인합니다.
- `.sheetHeader`가 desktop에서 display none이고 active/hover/focus style이 공유되는지 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-cc13abb3b66f-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Design selector에 필요한 desktop disclosure styling과 focus-visible treatment가 없었습니다. |
| 실제 변경 file/symbol/call path | Relative root와 z-index, custom summary trigger, hidden native webkit marker, focus outline, absolute right-aligned panel, list/link/swatch/copy/number styles를 추가했습니다. Sheet header는 desktop에서 숨깁니다. |
| Data/state/DOM/resource owner | Native details가 open state를 소유할 예정이고 CSS가 desktop geometry/visual state를 소유합니다. |
| Failure·absence·fallback 처리 | Panel width는 viewport에서 2rem을 뺀 값으로 제한됩니다. CSS만 추가되어 실제 details DOM이나 close behavior는 아직 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Keyboard-visible summary focus와 desktop menu presentation 기반을 만듭니다. Semantic markup과 state transition은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | 28dfc5087474가 same class set의 mobile sheet layout을, e43e8addd7f3가 actual details/summary DOM을 추가합니다. |
<!-- learner:commit-cc13abb3b66f-record:end -->

#### 최소 코드 증거

<!-- learner:commit-cc13abb3b66f-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-cc13abb3b66f-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-cc13abb3b66f-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-cc13abb3b66f-execution:end -->

### 2. `28dfc5087474` — style(designs): 모바일 디자인 선택 sheet 구성

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** mobile bottom-sheet styling

#### 해당 SHA에서 확인할 실제 코드

- max-width 640px의 `.root[open]::before`, fixed `.panel`, safe-area padding과 z-index를 확인합니다.
- Sheet header/button size, max-height/overflow/overscroll behavior를 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-28dfc5087474-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Desktop absolute dropdown CSS만 있어 좁은 viewport에서 panel width/position과 close affordance가 적절하지 않았습니다. |
| 실제 변경 file/symbol/call path | Mobile에서 root를 static으로 두고 open일 때 fixed backdrop을 만듭니다. Panel은 bottom fixed, scrollable max-height, safe-area-aware padding과 overscroll contain을 사용합니다. Sheet header와 2.75rem close button을 표시합니다. |
| Data/state/DOM/resource owner | Native open attribute가 backdrop visibility를, media query가 mobile geometry를 소유합니다. |
| Failure·absence·fallback 처리 | CSS는 body scroll lock이나 backdrop click close를 구현하지 않습니다. Overscroll은 panel 내부에 contain합니다. |
| 보장하는 것과 보장하지 않는 것 | 같은 semantic panel을 mobile sheet로 표현할 layout을 제공합니다. 실제 button/DOM과 focus behavior는 후속 component가 필요합니다. |
| 다음 commit 또는 관련 test 연결 | e43e8addd7f3/c69ef85c98b2가 trigger, nav list와 close button을 연결합니다. |
<!-- learner:commit-28dfc5087474-record:end -->

#### 최소 코드 증거

<!-- learner:commit-28dfc5087474-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-28dfc5087474-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-28dfc5087474-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-28dfc5087474-execution:end -->

### 3. `e43e8addd7f3` — feat(designs): 디자인 선택기 상태와 trigger 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** client details trigger와 active state

#### 해당 SHA에서 확인할 실제 코드

- 새 `design-switcher.tsx`의 client directive, refs, SITE_DESIGNS/templateCopy/activeIndex fallback을 확인합니다.
- Count template의 padStart와 summary aria-label/content를 확인합니다.
- Import된 Link/getTemplateHref가 이 SHA에서 아직 사용되지 않는다는 staged implementation 상태를 확인합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-e43e8addd7f3-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | CSS는 준비됐지만 native details/summary trigger와 active design label/count 계산이 없었습니다. |
| 실제 변경 file/symbol/call path | Client DesignSwitcher를 추가해 details/summary를 렌더링하고 details/summary refs를 만듭니다. Template copy map을 만들고 active ID를 SITE_DESIGNS에서 찾되 miss이면 first design으로 fallback합니다. Count template의 index/total을 2자리로 치환합니다. |
| Data/state/DOM/resource owner | Native details element가 open attribute를 소유하고 component가 active derivation/count/aria label을 소유합니다. Refs는 후속 close transition을 위해 준비됩니다. |
| Failure·absence·fallback 처리 | Active ID miss 시 active object는 first design으로 fallback하지만 activeIndex는 -1이므로 count의 index가 `00`이 될 수 있습니다. Missing copy는 design ID label로 fallback합니다. |
| 보장하는 것과 보장하지 않는 것 | Native trigger와 active copy를 제공합니다. List, navigation links, close behavior는 아직 렌더링하지 않으며 hydration race도 다루지 않습니다. |
| 다음 commit 또는 관련 test 연결 | c69ef85c98b2가 refs를 사용한 close/focus 및 full design list를 추가합니다. |
<!-- learner:commit-e43e8addd7f3-record:end -->

#### 최소 코드 증거

<!-- learner:commit-e43e8addd7f3-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 exact SHA의 변경 path와 symbol을 특정했으며, 직선적인 markup/CSS 추가를 반복 인용해 source dump로 만드는 것을 피했습니다.
<!-- learner:commit-e43e8addd7f3-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-e43e8addd7f3-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-e43e8addd7f3-execution:end -->

### 4. `c69ef85c98b2` — feat(designs): 디자인 선택 목록과 닫기 동작 추가

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Thread 역할:** nav list와 explicit state transition

#### 해당 SHA에서 확인할 실제 코드

- `nav`/`ul`/SITE_DESIGNS map의 active aria-current, swatches, copy fallback과 number를 확인합니다.
- `getTemplateHref(currentPath, design.id, { contentDebug })` call path를 확인합니다.
- Close button의 `removeAttribute("open")` → summary focus 순서와 link click close branch를 비교합니다.
- Native state owner와 imperative refs 사이의 ownership, missing refs optional chaining, navigation 발생 전 close의 non-guarantee를 기록합니다.
- 먼저 parent diff와 해당 SHA의 resulting tree를 구분합니다.
- Final HEAD의 helper·file layout·behavior를 이 SHA에 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-c69ef85c98b2-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Trigger는 열릴 수 있었지만 선택 목록, active semantics, template-preserving href, explicit mobile close/focus restoration이 없었습니다. |
| 실제 변경 file/symbol/call path | Panel 안에 labelled nav/list를 추가하고 SITE_DESIGNS 순서대로 links를 만듭니다. Active item은 aria-current=page와 active class를 사용합니다. Link는 currentPath/view/contentDebug를 보존합니다. Close button은 details open을 제거하고 summary에 focus하며, link click도 open을 제거합니다. |
| Data/state/DOM/resource owner | Native details가 disclosure state를, component refs/event handlers가 명시적 state transition을, getTemplateHref가 destination query를 소유합니다. Template copy map은 label/description fallback을 제공합니다. |
| Failure·absence·fallback 처리 | Refs가 없으면 optional chaining으로 close/focus가 no-op입니다. Link click close는 navigation 성공을 기다리지 않으며 focus를 복원하지 않습니다. Backdrop click/Escape custom handler/body lock은 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Desktop/mobile이 같은 semantic list를 공유하고 explicit close focus contract를 갖습니다. 그러나 whole component가 client boundary이고 hydration 전에 native open이 바뀌면 server/client attribute mismatch 가능성이 남습니다. |
| 다음 commit 또는 관련 test 연결 | Category 02의 b9571c485013이 이 component를 `SiteHeader`/`PageShell`의 조건부 consumer로 연결합니다. 09cec616f314이 기본 close/link contract를 component test로 확인하고, Category 07의 c702/b6c/a37/1ac가 hydration race를 재현한 뒤 server-markup + tiny client close island로 corrected invariant를 만듭니다. |
<!-- learner:commit-c69ef85c98b2-record:end -->

#### 최소 코드 증거

<!-- learner:commit-c69ef85c98b2-excerpt:start -->
- **Commit:** `c69ef85c98b2`
- **Path:** `src/components/portfolio/design-switcher.tsx`
- **Location:** `close button handler`

```tsx
onClick={() => {
  detailsRef.current?.removeAttribute("open");
  summaryRef.current?.focus();
}}
```

이 발췌는 해당 SHA의 decision/state/ownership을 보여 주는 최소 부분입니다. 후속 commit의 코드는 섞지 않았습니다.
<!-- learner:commit-c69ef85c98b2-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-c69ef85c98b2-execution:start -->
- **정적 검토:** `web/portfolio` 전용 commit classification과 해당 exact SHA의 commit diff/changed-file view를 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 checkout을 만들 수 없었고, GitHub connector는 source/diff inspection만 제공했습니다. 따라서 command, pass/fail, timing, browser 결과를 주장하지 않습니다.
<!-- learner:commit-c69ef85c98b2-execution:end -->

## 6. Invariant ledger

<!-- learner:thread-ledger:start -->
| Invariant | 도입·변경 commit | 실제 code/test evidence | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Native disclosure state | e43e8addd7f3 | details/summary DOM | client hydration race 미해결 | 초기 Thread에서는 native open attribute가 state owner |
| Responsive same-DOM presentation | cc13abb3b66f → 28dfc5087474 | desktop absolute panel/mobile fixed sheet | body lock/backdrop close 없음 | 640px boundary에서 CSS만 geometry 전환 |
| Template-preserving selection | c69ef85c98b2 | getTemplateHref currentPath/design/debug | destination response 미검증 | 선택 URL과 active semantics 제공 |
| Explicit close/focus | c69ef85c98b2 | remove open → summary.focus | link click은 focus 복원 안 함 | Close button 경로의 focus contract |
| Shared-shell integration | b9571c485013 (외부 관계) | `SiteHeader`의 조건부 `DesignSwitcher` call과 `PageShell` props | `templateSwitcher`가 없으면 selector 자체가 렌더링되지 않음 | Category 02가 실제 route-wide consumer wiring을 소유 |
| Corrected server-first owner | a37cb8596733 (외부 관계) | server DesignSwitcher + client DesignSwitcherClose | frozen map 밖 category 07 | 후속 architecture에서 markup은 server, close handler만 client |
<!-- learner:thread-ledger:end -->

## 7. Failure → Fix → Test 관계

<!-- learner:thread-relations:start -->
| Failure/위험 | 실제 영향·root cause | Fix/결정 | Regression evidence 또는 공백 |
| --- | --- | --- | --- |
| Primitive는 존재하지만 shell consumer 없음 | 공용 route shell에서 아직 도달할 수 없음 | b9571c485013이 `SiteHeader`에서 `templateSwitcher` 존재 시 `DesignSwitcher`를 호출 | Category 02 exact diff로 정적 확인; runtime integration test는 실행하지 않음 |
| Client-rendered native details | Hydration 전 사용자가 open을 바꾸면 attribute mismatch 위험 | c702b870d57a suppress warning 후 a37cb8596733 server markup 전환 | b6c0238ab8b8 SSR→native open→hydrate deterministic test, 1ac7813155c6 source-boundary test |
| Close handler가 whole switcher refs를 요구 | 전체 list가 client boundary에 묶임 | a37cb8596733에서 small DesignSwitcherClose client island로 이동 | 1ac7813155c6가 source에 use client/useRef가 없음을 검증 |
| Link click에서 open 제거 | Navigation을 event handler semantics에 결합 | a37cb8596733에서 link onClick 제거; route navigation에 맡김 | Updated component test는 explicit close/focus와 href를 검증 |
<!-- learner:thread-relations:end -->

## 8. Ownership·state·responsibility 변화

<!-- learner:thread-ownership:start -->
| 단계 | Owner | 책임 변화 |
| --- | --- | --- |
| CSS 준비 | Desktop/mobile geometry | details open attribute selector를 예상 |
| e43 초기 component | Native details state + client refs | Trigger/active label만 존재 |
| c69 완성 | Client component가 list, links, refs, close/focus를 모두 소유 | Initial functional contract |
| 외부 category 02 / b957 | SiteHeader와 PageShell이 component reachability와 UI/template props를 소유 | 실제 shared-shell consumer |
| 후속 category 07 | Server markup + client close island | Native state를 hydration-safe하게 보존하는 corrected owner |
<!-- learner:thread-ownership:end -->

## 9. 최종 Thread 상태

<!-- learner:thread-final-state:start -->
- Frozen commit set의 최종 상태는 client DesignSwitcher 안의 native details/summary/nav/list이며, 이 set만으로는 아직 shared shell에 연결되지 않습니다.
- 실제 route-wide reachability는 인접 category 02의 b9571c485013이 `SiteHeader`/`PageShell`에 조건부 consumer를 추가하면서 생깁니다.
- Desktop은 absolute dropdown, mobile은 backdrop이 있는 fixed bottom sheet를 같은 DOM으로 표현합니다.
- Active item과 count/copy fallback, template/debug-aware links가 구현돼 있습니다.
- Explicit close button은 open을 제거하고 summary focus를 복원합니다.
- 이 초기 상태의 hydration 경쟁 조건은 후속 category 07 commits에서 재현·수정·검증됩니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 흐름

<!-- learner:thread-flow:start -->
1. Frozen map은 DesignSwitcher의 component contract를 완성하고, category 02의 b9571c485013이 `SiteHeader`에서 `templateSwitcher`가 있을 때 active design, current path, templates, UI copy와 contentDebug를 전달합니다.
2. DesignSwitcher가 template map과 active/count label을 파생합니다.
3. Native details/summary가 browser-managed disclosure를 제공합니다.
4. Nav list가 SITE_DESIGNS 순서와 active aria-current를 렌더링하고 getTemplateHref로 destination을 만듭니다.
5. CSS가 viewport에 따라 desktop dropdown 또는 mobile bottom sheet로 재배치합니다.
6. Close button은 open attribute를 제거하고 summary focus를 복원합니다.
7. 후속 server-first refactor에서는 markup owner가 server로 이동하고 close interaction만 client island에 남습니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 commit을 exact SHA diff와 resulting file 기준으로 기록했습니다.
- [x] SHA, subject, order, importance, tags와 Thread 역할을 frozen scaffold와 동일하게 유지했습니다.
- [x] Previous state, owner, absence/failure, guarantee/non-guarantee와 later relation을 채웠습니다.
- [x] S/A-level 설명을 B-level보다 깊게 작성했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: runtime command는 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 learner-facing answer cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 08-design-switcher-disclosure-and-responsive-sheet.md =====

===== BEGIN FILE: README.md =====
# 03-shared-ui-interaction-and-responsive-primitives

`web/portfolio` branch의 shared UI interaction과 responsive primitive history를 학습하는 category workbook입니다.

## Phase 1 audit 결과

- Category boundary는 shared renderer/component/CSS primitive와 responsive interaction에 한정했습니다.
- Route lifecycle, content schema/loader, design route composition 전체, hydration/performance regression architecture는 인접 category가 소유합니다.
- 기존 01과 06에 중복됐던 `e37ea9c2819a`, `1ef269fbdb49`, `daa6815a6dfa`는 link policy story인 01에만 귀속시켰습니다.
- 01에는 placement vocabulary, detail selector migration, hero placement adoption, regression test와 rendering dedup commits를 추가해 policy → consumer → test → refactor 흐름을 완성했습니다.
- 02에는 실제 `ProfilePhoto` consumer인 `a00a6bf1af58`을 추가했습니다.
- 06은 link action 정책을 제거하고 AvailabilityBadge → ProjectCard → featured consumer → interaction CSS의 card composition story로 보정했습니다.
- README가 약속했지만 빠져 있던 native disclosure/mobile sheet 구현 story를 08 Thread로 추가했습니다.
- 08의 실제 shared-shell integration은 category 02의 `b9571c485013`이, 후속 hydration fix/test/server refactor는 category 07이 소유하므로 여기서는 중복하지 않고 관계만 고정했습니다.
- 나머지 Thread의 경계와 commit order는 실제 history에 맞아 유지했습니다.
- Shared dependency와 consumer story가 전역 history에서 서로 교차하므로 Thread 번호를 전역 commit chronology로 강제하지 않았습니다. 각 Thread 내부는 chronological order이고 category index는 dependency·학습 경계를 따릅니다.

## Branch·metadata 검증 방식

- Branch의 `commit/commit-importance.md`는 root부터 head까지 독립 linear history 476개를 분류합니다.
- 이 category가 참조하는 모든 commit은 그 branch-scoped 목록에서 확인하고 exact SHA commit view로 subject와 changed files를 재확인했습니다.
- 다른 branch의 구현, test, docs 또는 final HEAD를 과거 SHA 설명에 사용하지 않았습니다.
- 실행 가능한 checkout은 작업 환경의 GitHub DNS 해석 실패로 만들지 못했습니다. Runtime test 결과는 없으며 모든 completed 문서가 이를 명시합니다.

## Thread index

| 순서 | Thread | Commit 수 | 경계 |
| --- | --- | --- | --- |
| 1 | [Content link security, placement, and transport](01-content-link-security-and-transport.md) | 10 | 이 Thread는 링크가 어디에 노출되고 어떤 element/URL로 이동하는지를 소유합니다. ProjectCard의 카드 조립과 hover 표현은 06 Thread에 남기며, route lifecycle 자체는 이 범위에 포함하지 않습니다. |
| 2 | [Content media loading and layout stability](02-content-media-loading-and-layout-stability.md) | 5 | 이 Thread는 shared media DOM과 loading/layout contract를 다룹니다. Image optimization pipeline, source asset generation, project detail 정보 architecture 자체는 포함하지 않습니다. |
| 3 | [Progressive reveal to server-first rendering](03-progressive-reveal-to-server-first-rendering.md) | 4 | 이 Thread는 shared Reveal의 visibility/lifecycle과 motion fallback을 다룹니다. 개별 section content, route composition, 전역 performance test architecture는 포함하지 않습니다. |
| 4 | [Terminal state machine and motion fallback](04-terminal-state-machine-and-motion-fallback.md) | 4 | 이 Thread는 terminal preview primitive의 state와 표현을 다룹니다. Terminal content schema validation, home route 전체 section architecture, general motion policy는 각각 다른 Thread의 책임입니다. |
| 5 | [Technology stack icon, list, and marquee](05-technology-stack-icon-list-and-marquee.md) | 5 | 이 Thread는 stack visual primitives를 다룹니다. Tech stack content schema와 `resolveTechStackItem`의 validation/fallback source, home section 전체 composition은 다른 Thread가 소유합니다. |
| 6 | [Project card composition and interaction](06-project-card-composition-and-interaction.md) | 4 | Phase 1에서 기존 `project-card-actions-and-evidence-components` Thread의 link-policy commits를 01 Thread로 이동했습니다. 이 Thread는 card composition과 interaction만 소유하며 link placement/transport는 01 Thread의 selector·renderer를 소비합니다. |
| 7 | [Journey timeline primitives and responsive layout](07-journey-timeline-primitives-and-responsive-layout.md) | 6 | 이 Thread는 journey presentation primitive와 responsive CSS를 다룹니다. Journey JSON schema/validation, route section copy, Reveal의 최종 server-first refactor는 각각 content system과 03 Thread가 소유합니다. |
| 8 | [Design switcher disclosure and responsive sheet](08-design-switcher-disclosure-and-responsive-sheet.md) | 4 | Phase 1에서 category README가 약속했지만 기존 7개 Thread가 다루지 않던 responsive disclosure primitive 구현 story를 추가했습니다. 실제 shared-shell 연결은 category 02의 `b9571c485013`이, 후속 hydration race fix/test와 server-component refactor는 category 07이 소유합니다. 이 frozen commit map에는 primitive 구현 commit만 두고 인접 category의 integration/correction/evidence는 관계로 명시합니다. |

## 구조 규칙

- `scaffold/`는 Phase 1 audit 뒤 동결된 authoritative workbook입니다.
- `completed/`는 동일 file set·relative path·fixed structure를 유지하고 learner-facing section만 채운 counterpart입니다.
- Scaffold와 completed 사이에 SHA, subject, order, importance, tags, role 또는 invariant 차이가 있으면 invalid deliverable입니다.
===== END FILE: README.md =====

