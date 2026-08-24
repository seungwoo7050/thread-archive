===== BEGIN FILE: 01-content-link-security-and-transport.md =====
# 개발 과정: 콘텐츠 링크 보안·노출 위치·이동 방식

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

연락처와 프로젝트 링크의 선택 규칙, 내부·외부 이동 방식, 배치별 노출, 사용 가능 여부 필터링과 공용 렌더러 정리를 실제 커밋 순서로 복원합니다.

**경계:** 이 개발 과정은 링크가 어디에 노출되고 어떤 요소·URL로 이동하는지를 소유합니다. `ProjectCard`의 카드 조립과 마우스 오버 표현은 06 개발 과정에 남기며, 라우트 실행 주기 자체는 이 범위에 포함하지 않습니다.

### 고정 불변 조건

- 프로젝트 카드와 상세 화면에 표시할 링크는 선택자가 결정하고, 홈 소개 영역은 같은 `hero` 노출 위치 값을 직접 확인합니다. 링크 렌더러는 전달받은 링크의 이동 방식만 결정합니다.
- 외부 링크는 `<a>` 요소와 외부 링크 속성을 사용하고, 내부 링크는 Next Link와 현재 `view`·`디버그` 쿼리 보존 규칙을 사용합니다.
- 공개 중이 아닌 프로젝트의 데모와 해당 노출 위치에 없는 링크는 렌더링되지 않습니다.
- 선택 결과가 비면 동작 영역도 렌더링하지 않습니다.

## 2. 핵심 질문

- 초기 선택자가 연락처·프로젝트 링크 사용 가능 여부를 어떤 단일 허용 값 집합으로 만들었는가?
- `ContentLinkView`가 내부 링크와 외부 링크의 이동 방식을 나눌 때 보장하는 속성과 검증하지 않는 항목은 무엇인가?
- 카드·상세·소개 영역 노출 위치가 언제 도입되고 각 소비자의 필터링이 선택자 또는 직접 판정 함수 중 어디에 남았는가?
- 09cec616f314의 테스트가 원본 순서, 쿼리 보존, 외부 링크 속성, 빈 출력을 어떤 방법으로 고정하는가?
- 44e4d062da50의 리팩터링이 동작을 바꾸지 않고 어떤 렌더링 책임만 합쳤는가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 중요도 S·A는 설계 결정, 소유 주체, 실패 처리와 후속 근거를 중요도 B보다 깊게 복원합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `ba8da56d3fcf` | feat(portfolio): 연락과 프로젝트 링크 선택기 추가 | A | CONTENT | 규칙 소유 주체 도입 |
| 2 | `f63c978c71c9` | feat(ui): 내부 외부 콘텐츠 링크 렌더링 | A | CONTENT | 링크 이동 방식 렌더러 도입 |
| 3 | `e37ea9c2819a` | feat(project): 프로젝트 링크 그룹 추가 | B | RENDERER | 상세 링크 소비자 |
| 4 | `1ef269fbdb49` | feat(project): 프로젝트 카드 링크 추가 | B | RENDERER | 카드 링크 소비자 |
| 5 | `daa6815a6dfa` | feat(project): 카드 링크를 콘텐츠 배치 기준으로 선택 | B | CONTENT, RENDERER | 카드 노출 위치 적용 |
| 6 | `119ff9a92090` | feat(content): 링크 배치 selector 추가 | B | CONTENT | 노출 위치 값 집합 일반화 |
| 7 | `2d87b62dcce8` | refactor(project): 상세 링크를 배치 기준으로 선택 | B | RENDERER, REFACTOR | 상세 선택자 전환 |
| 8 | `ee2c118a76d6` | feat(content): 홈 링크를 배치 기준으로 선택 | B | CONTENT | 소개 영역 소비자 전환 |
| 9 | `09cec616f314` | test(ui): 디자인 선택과 프로젝트 링크 계약 검증 | A | VALIDATION, TEST | 결정적 컴포넌트 규칙 검증 |
| 10 | `44e4d062da50` | refactor(ui): 프로젝트 링크 렌더링 중복 제거 | B | VALIDATION, REFACTOR | 공용 목록 렌더러 추출 |

## 5. 커밋별 학습 기록

### 1. `ba8da56d3fcf` — feat(portfolio): 연락과 프로젝트 링크 선택기 추가

- **중요도:** A
- **태그:** CONTENT
- **개발 과정에서의 역할:** 규칙 소유 주체 도입

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/selectors.ts`의 `getPreferredContactLinks`, `getProjectLink`, `isProjectLive`, `getProjectCardLinks`, `getExternalLinkProps`를 부모 커밋과 비교합니다.
- `src/lib/portfolio.ts`가 새 선택자를 공개 API에 어떻게 노출하는지 확인합니다.
- 선호 ID 누락, 데모 링크 부재, 공개 중이 아닌 프로젝트, 내부 링크의 반환값을 각각 추적합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-ba8da56d3fcf-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 직전 파일 트리에는 연락처 우선 ID를 실제 링크로 해석하거나 프로젝트 데모 사용 가능 여부와 외부 이동 방식 속성을 공통으로 계산하는 선택자가 없었습니다. 소비자가 같은 판단을 반복할 여지가 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `selectors.ts`가 우선 순서를 유지하면서 존재하는 링크만 남기고, 타입별 프로젝트 링크 조회, `status === "live"`와 데모 존재를 함께 요구하는 배포 중 판정, 초기 카드 동작 선택, 외부 링크 속성을 추가했습니다. `portfolio.ts`가 이를 공개 항목으로 묶었습니다. |
| 데이터·상태·DOM·자원 소유 주체 | 콘텐츠 데이터가 사실 원본을 소유하고 선택자가 파생 결과를 소유합니다. 컴포넌트는 선택자의 반환 배열·불리언·속성을 소비할 뿐 배포 상태 의미를 새로 결정하지 않는 방향이 시작됐습니다. |
| 실패·누락·대체 처리 | 없는 우선 ID는 타입 검사 필터로 제거되고, 없는 프로젝트 링크는 `null`, 배포 중 조건을 만족하지 않는 데모는 카드 결과에서 제외됩니다. 내부 링크의 외부 속성은 빈 객체입니다. |
| 보장하는 것과 보장하지 않는 것 | 연락처 순서와 초기 프로젝트 동작 사용 가능 여부를 중앙화합니다. 다만 노출 위치 값 집합은 아직 없고, href 프로토콜·콘텐츠 유효성과 실제 DOM 렌더링은 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | f63c978c71c9가 링크 이동 방식 렌더러를 만들고, e37ea9c2819a·1ef269fbdb49가 상세·카드 소비자를 붙입니다. daa6815a6dfa 이후 노출 위치가 규칙 입력으로 추가됩니다. |
<!-- learner:commit-ba8da56d3fcf-record:end -->

#### 최소 코드 증거

<!-- learner:commit-ba8da56d3fcf-excerpt:start -->
- **커밋:** `ba8da56d3fcf`
- **경로:** `src/lib/portfolio/selectors.ts`
- **위치:** `isProjectLive / getExternalLinkProps`

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

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-ba8da56d3fcf-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-ba8da56d3fcf-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-ba8da56d3fcf-execution:end -->

### 2. `f63c978c71c9` — feat(ui): 내부 외부 콘텐츠 링크 렌더링

- **중요도:** A
- **태그:** CONTENT
- **개발 과정에서의 역할:** 링크 이동 방식 렌더러 도입

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/content-link.tsx`의 `ContentLinkView` 두 반환 브랜치를 비교합니다.
- 외부 브랜치가 `getExternalLinkProps`를 펼치기하고 내부 브랜치가 `getTemplateHref`에 `homeTemplate`·`contentDebug`를 전달하는 호출 경로를 확인합니다.
- 이 컴포넌트가 표시 여부, href 검증, 문구·아이콘 스타일 적용을 소유하는지 구분합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-f63c978c71c9-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 선택자는 이동 방식에 필요한 정보만 반환했고, 각 소비자가 `<a>`와 `next/link`를 직접 선택하면 외부 속성이나 쿼리 보존이 서로 달라질 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `ContentLinkView`가 `link.external`을 유일한 이동 방식 분기값으로 사용합니다. 외부 브랜치는 원래 href를 가진 `<a>`와 선택자 속성을 사용하고, 내부 브랜치는 `getTemplateHref(link.href, homeTemplate, { contentDebug })`를 거친 Next `Link`를 사용합니다. |
| 데이터·상태·DOM·자원 소유 주체 | ContentLink 데이터가 외부 여부와 href를 소유하고, 렌더러가 DOM 요소 선택을 소유합니다. 내부 URL의 `view`·`디버그` 합성은 `getTemplateHref`가 소유합니다. |
| 실패·누락·대체 처리 | 외부 false이면 외부 속성을 붙이지 않습니다. 이 커밋에는 잘못된 형식의 URL, 지원하지 않는 처리 형식, 빈 문구를 거부하는 브랜치가 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 내부·외부 링크 이동 방식과 내부 쿼리 보존 경로를 일관되게 만듭니다. 어떤 링크를 보여 줄지는 보장하지 않고 호출자가 children/className도 제공합니다. |
| 다음 커밋 또는 관련 테스트 연결 | e37ea9c2819a 이후 모든 프로젝트 동작이 이 렌더러를 통과합니다. 09cec616f314가 내부 쿼리와 외부 `target`/`rel`을 컴포넌트 테스트로 검증합니다. |
<!-- learner:commit-f63c978c71c9-record:end -->

#### 최소 코드 증거

<!-- learner:commit-f63c978c71c9-excerpt:start -->
- **커밋:** `f63c978c71c9`
- **경로:** `src/components/portfolio/content-link.tsx`
- **위치:** `ContentLinkView`

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

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-f63c978c71c9-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-f63c978c71c9-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-f63c978c71c9-execution:end -->

### 3. `e37ea9c2819a` — feat(project): 프로젝트 링크 그룹 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 상세 링크 소비자

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/project-links.tsx`의 `ProjectLinks`와 로컬 표시 여부 판정 함수를 확인합니다.
- 데모 사용 가능 여부, `excludeCaseStudy`, 빈 결과, `ContentLinkView` 호출을 각각 추적합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-e37ea9c2819a-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 프로젝트 상세에서 프로젝트.링크를 공통 링크 이동 방식 렌더러로 출력하는 동작 그룹이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `ProjectLinks`가 프로젝트 링크를 필터링한 뒤 flex 동작 그룹으로 렌더링합니다. 데모는 `isProjectLive`, 프로젝트 사례는 속성에 따라 제외하고, 각 항목은 `ContentLinkView`를 사용합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 이 시점에는 상세 컴포넌트가 일부 표시 여부 필터링을 소유합니다. 이동 방식은 `ContentLinkView`가 소유합니다. |
| 실패·누락·대체 처리 | 필터 결과가 비면 `null`입니다. Offline 데모와 명시적으로 제외한 프로젝트 사례는 DOM에 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Detail 동작 그룹을 재사용할 수 있게 하지만 노출 위치는 아직 읽지 않고 필터링이 선택자와 컴포넌트에 나뉩니다. |
| 다음 커밋 또는 관련 테스트 연결 | 2d87b62dcce8이 상세 노출 위치 선택을 선택자로 옮기고, 44e4d062da50이 목록 렌더링을 합칩니다. |
<!-- learner:commit-e37ea9c2819a-record:end -->

#### 최소 코드 증거

<!-- learner:commit-e37ea9c2819a-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-e37ea9c2819a-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-e37ea9c2819a-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-e37ea9c2819a-execution:end -->

### 4. `1ef269fbdb49` — feat(project): 프로젝트 카드 링크 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 카드 링크 소비자

#### 해당 SHA에서 확인할 실제 코드

- `ProjectCardLinks`가 `getProjectCardLinks` 결과를 어떻게 렌더링하는지 확인합니다.
- `ProjectLinks`와 중복된 null 브랜치, 클래스 선택, 아이콘 choice를 비교합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-1ef269fbdb49-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail 동작은 생겼지만 카드가 선택자의 카드 동작 결과를 표현하는 래퍼는 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 같은 파일에 `ProjectCardLinks`를 추가해 `getProjectCardLinks(project)` 결과를 `ContentLinkView`로 렌더링했습니다. |
| 데이터·상태·DOM·자원 소유 주체 | 카드 표시 여부는 선택자, DOM·클래스·아이콘은 래퍼가 소유하지만 상세 래퍼와 마크업 책임이 중복됩니다. |
| 실패·누락·대체 처리 | 빈 배열이면 `null`입니다. Demo 주요 스타일 적용과 외부·내부 아이콘 분기는 렌더링에서 처리됩니다. |
| 보장하는 것과 보장하지 않는 것 | 카드 소비자를 연결하지만 콘텐츠에서 지정한 노출 위치는 아직 반영하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | daa6815a6dfa가 카드 선택자를 노출 위치 기반으로 바꾸고, 44e4d062da50이 중복 목록 렌더러를 제거합니다. |
<!-- learner:commit-1ef269fbdb49-record:end -->

#### 최소 코드 증거

<!-- learner:commit-1ef269fbdb49-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-1ef269fbdb49-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-1ef269fbdb49-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-1ef269fbdb49-execution:end -->

### 5. `daa6815a6dfa` — feat(project): 카드 링크를 콘텐츠 배치 기준으로 선택

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** 카드 노출 위치 적용

#### 해당 SHA에서 확인할 실제 코드

- `getProjectCardLinks`의 기존 타입 허용 목록과 새 `placements.includes("card")` 검사 단계를 비교합니다.
- Placement를 통과한 데모와 데모가 아닌 링크가 각각 어떤 추가 조건을 거치는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-daa6815a6dfa-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 카드 링크는 데모·GitHub·프로젝트 사례로 코드에 고정된 타입 집합으로 선택되어 콘텐츠가 같은 타입을 특정 표시 영역에서 숨기거나 다른 타입을 카드에 배치할 수 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 선택자가 먼저 `link.placements?.includes("card")`를 요구합니다. Demo만 기존 배포 중 판정을 추가로 통과하고, 노출 위치가 카드인 다른 타입은 허용합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 콘텐츠가 노출 위치 의도를 소유하고 선택자가 노출 위치와 실행 시점 사용 가능 여부를 결합합니다. 카드 컴포넌트는 결과를 그대로 소비합니다. |
| 실패·누락·대체 처리 | 노출 위치가 없거나 카드를 포함하지 않으면 제외됩니다. Placement가 있어도 배포 중이 아닌 데모는 제외됩니다. |
| 보장하는 것과 보장하지 않는 것 | Type 허용 목록보다 콘텐츠에서 지정한 노출 위치를 우선하는 카드 규칙을 보장합니다. 다른 표시 영역의 일반적인 선택자는 아직 없습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 119ff9a92090이 노출 위치 타입과 일반적인 선택자를 도입해 이 결정을 카드·상세·사이트로 일반화합니다. |
<!-- learner:commit-daa6815a6dfa-record:end -->

#### 최소 코드 증거

<!-- learner:commit-daa6815a6dfa-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-daa6815a6dfa-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-daa6815a6dfa-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-daa6815a6dfa-execution:end -->

### 6. `119ff9a92090` — feat(content): 링크 배치 selector 추가

- **중요도:** B
- **태그:** CONTENT
- **개발 과정에서의 역할:** 노출 위치 값 집합 일반화

#### 해당 SHA에서 확인할 실제 코드

- `LinkPlacement` 유니언과 `getProjectLinksForPlacement`, 카드·상세 래퍼, 사이트 링크 노출 위치 선택자를 확인합니다.
- 원본 배열 순서가 필터 결과에서도 유지되는지와 노출 위치 누락이 어떻게 처리되는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-119ff9a92090-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 카드만 노출 위치를 직접 읽고 상세·사이트 소비자는 별도 규칙을 유지해 표시 영역 허용 값 집합과 선택 책임이 분산되어 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `hero`, `contact`, `card`, `detail`, `footer` 노출 위치 값 집합과 일반적인 필터를 추가하고 카드·상세 래퍼 및 사이트 콘텐츠 선택자를 만들었습니다. |
| 데이터·상태·DOM·자원 소유 주체 | Placement 포함 관계의 공통 해석은 선택자가 소유하고, 각 래퍼는 표시 영역 이름만 고정합니다. |
| 실패·누락·대체 처리 | 선택적 노출 위치가 없으면 일반적인 필터에서 제외됩니다. 실행 시점 데모 사용 가능 여부는 표시 영역 래퍼·소비자가 추가로 처리해야 합니다. |
| 보장하는 것과 보장하지 않는 것 | 모든 표시 영역이 같은 노출 위치 값 집합을 사용할 기반을 보장하지만 소비자 전환은 아직 끝나지 않았습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 2d87b62dcce8은 상세 소비자를 선택자로 이동하고, ee2c118a76d6은 소개 영역 소비자가 동일 노출 위치 값 집합을 직접 판정 함수로 채택하게 합니다. |
<!-- learner:commit-119ff9a92090-record:end -->

#### 최소 코드 증거

<!-- learner:commit-119ff9a92090-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-119ff9a92090-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-119ff9a92090-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-119ff9a92090-execution:end -->

### 7. `2d87b62dcce8` — refactor(project): 상세 링크를 배치 기준으로 선택

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** 상세 선택자 전환

#### 해당 SHA에서 확인할 실제 코드

- `ProjectLinks`가 원본 `project.links` 대신 `getProjectDetailLinks(project)`를 호출하도록 바뀐 부분을 확인합니다.
- Placement 선택 후에도 `excludeCaseStudy`와 데모 배포 중 검사가 어느 레이어에 남는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-2d87b62dcce8-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail 래퍼가 원본 링크 배열에서 자체 필터링하여 상세 노출 위치를 반영하지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 입력 배열을 `getProjectDetailLinks(project)` 결과로 교체하고, 프로젝트 사례 제외 옵션과 데모 배포 중 검사는 컴포넌트에 유지했습니다. |
| 데이터·상태·DOM·자원 소유 주체 | 선택자가 상세 포함 관계를 소유하고 컴포넌트가 호출자 옵션 및 실행 시점 화면 구성 필터링을 소유합니다. |
| 실패·누락·대체 처리 | Detail 노출 위치가 없는 링크는 컴포넌트에 도달하지 않습니다. 빈 결과는 기존 null 브랜치로 이어집니다. |
| 보장하는 것과 보장하지 않는 것 | Detail 표시 영역이 콘텐츠 노출 위치를 따르게 하지만 데모 필터링이 완전히 선택자로 이동한 것은 아닙니다. |
| 다음 커밋 또는 관련 테스트 연결 | 09cec616f314가 상세 원본 순서와 필터링을 검증하고, 44e4d062da50이 렌더링 중복만 정리합니다. |
<!-- learner:commit-2d87b62dcce8-record:end -->

#### 최소 코드 증거

<!-- learner:commit-2d87b62dcce8-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-2d87b62dcce8-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-2d87b62dcce8-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-2d87b62dcce8-execution:end -->

### 8. `ee2c118a76d6` — feat(content): 홈 링크를 배치 기준으로 선택

- **중요도:** B
- **태그:** CONTENT
- **개발 과정에서의 역할:** 소개 영역 소비자 전환

#### 해당 SHA에서 확인할 실제 코드

- Classic/Design 홈 라우트에서 코드에 고정된 링크 타입 필터가 소개 영역 노출 위치 필터로 교체된 변경 내용을 확인합니다.
- 두 디자인 라우트가 같은 콘텐츠 검증 규칙을 읽되 각 라우트의 화면 구성 마크업은 유지되는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-ee2c118a76d6-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 홈 소개 영역은 github·이력서·website 타입을 코드에 직접 열거해 콘텐츠 노출 위치와 분리되어 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 두 홈 구현이 `link.placements?.includes("hero")`를 사용하도록 변경됐습니다. |
| 데이터·상태·DOM·자원 소유 주체 | 콘텐츠가 소개 영역 포함 관계를 선언하고 각 디자인 라우트가 그 노출 위치 판정 함수와 레이아웃·스타일 적용을 직접 적용합니다. 이 시점에는 소개 영역 선택 소유 주체가 선택자로 이동하지 않았습니다. |
| 실패·누락·대체 처리 | 소개 영역 노출 위치가 없는 링크는 타입과 무관하게 제외됩니다. 이 커밋은 일반적인 사이트 선택자를 호출하지 않고 동일 판정 함수를 두 라우트에 둡니다. |
| 보장하는 것과 보장하지 않는 것 | 홈 링크 집합이 콘텐츠 노출 위치를 따르게 합니다. Footer·연락처 전환이나 이동 방식 변경은 포함하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 후속 리팩터링 대상은 남지만, 09cec616f314의 프로젝트 링크 테스트와 별개로 홈 소비자 동작은 이 커밋에 전용 테스트가 없습니다. |
<!-- learner:commit-ee2c118a76d6-record:end -->

#### 최소 코드 증거

<!-- learner:commit-ee2c118a76d6-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-ee2c118a76d6-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-ee2c118a76d6-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-ee2c118a76d6-execution:end -->

### 9. `09cec616f314` — test(ui): 디자인 선택과 프로젝트 링크 계약 검증

- **중요도:** A
- **태그:** VALIDATION, TEST
- **개발 과정에서의 역할:** 결정적 컴포넌트 규칙 검증

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/project-links.test.tsx`의 테스트 입력과 단언문을 배포 환경 선택자·렌더러 경로별로 매핑합니다.
- 내부 URL의 `view`·`debug`, 외부 `target`·`rel`, 배포되지 않은 데모, 프로젝트 사례 exclusion, 카드 노출 위치, 빈 DOM을 구분합니다.
- 같은 커밋의 `design-switcher.test.tsx`가 브라우저 기본 `<details>` 닫기·포커스를 검증하지만 이 개발 과정에서는 링크 규칙과의 경계만 기록합니다.
- JSDOM 컴포넌트 테스트가 실제 탐색·브라우저 보안·네트워크 응답을 증명하지 않는다는 한계를 명시합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-09cec616f314-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Placement와 이동 방식 규칙은 구현돼 있었지만 원본 순서, 쿼리 유지, 외부 링크 속성, 누락 분기를 한 번에 깨뜨리지 못하게 하는 결정적인 회귀 검증 근거가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | Testing Library로 실제 컴포넌트를 렌더링합니다. 상세 링크는 원본 순서와 내부·외부 속성을 검사하고, 배포되지 않은 데모·프로젝트 사례 제외 조건을 재현합니다. 카드 링크는 카드 노출 위치가 아닌 원본 링크를 제외하며, 모두 필터된 래퍼는 빈 DOM임을 확인합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 테스트 입력이 입력 상태를 소유하고 배포 환경 선택자·렌더러가 실제 결과 DOM을 소유합니다. 단언은 href·속성·문구·빈 컨테이너를 관찰합니다. |
| 실패·누락·대체 처리 | 직접 오류를 주입하는 대신 콘텐츠 테스트 입력의 배포 상태, 노출 위치와 제외 옵션을 바꾸는 경계 테스트입니다. 외부 탐색이나 대상 페이지 응답은 실행하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 해당 SHA의 컴포넌트 단위 규칙을 결정적으로 고정합니다. 브라우저 새 창 격리 전체, 라우트 전환, 잘못된 형식의 콘텐츠 검증, CSS 상호작용은 증명하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 44e4d062da50은 이 테스트 규칙을 유지하면서 중복 렌더러를 추출합니다. 디자인 선택기 테스트의 후속 하이드레이션 변경 과정는 08 개발 과정의 외부 관계 및 07 분류가 소유합니다. |
<!-- learner:commit-09cec616f314-record:end -->

#### 최소 코드 증거

<!-- learner:commit-09cec616f314-excerpt:start -->
- **커밋:** `09cec616f314`
- **경로:** `src/components/portfolio/project-links.test.tsx`
- **위치:** `project link transport assertions`

```tsx
expect(links[0]).toHaveAttribute(
  "href",
  "/projects/sample-project?view=classic&debug=content",
);
expect(links[0]).not.toHaveAttribute("target");
expect(links[1]).toHaveAttribute("target", "_blank");
expect(links[1]).toHaveAttribute("rel", "noreferrer");
```

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-09cec616f314-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-09cec616f314-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-09cec616f314-execution:end -->

### 10. `44e4d062da50` — refactor(ui): 프로젝트 링크 렌더링 중복 제거

- **중요도:** B
- **태그:** VALIDATION, REFACTOR
- **개발 과정에서의 역할:** 공용 목록 렌더러 추출

#### 해당 SHA에서 확인할 실제 코드

- 새 로컬 `ProjectLinkList`와 두 공개 래퍼의 이전·after를 비교합니다.
- 빈 브랜치, 키, min-height, 주요 클래스, 외부·내부 아이콘과 `ContentLinkView` 속성이 한 곳으로 이동했는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-44e4d062da50-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail·카드 래퍼가 같은 빈 검사, classes, 키, 아이콘, 이동 방식 호출을 복제해 한쪽만 변경될 위험이 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 로컬 `ProjectLinkList`가 빈 `null`과 항목 렌더링을 소유하고 `ProjectLinks`·`ProjectCardLinks`는 선택만 수행해 링크를 전달합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 공개 래퍼는 표시 영역별 선택을, 내부 목록은 공용 DOM 표현을 소유합니다. |
| 실패·누락·대체 처리 | 빈 배열은 공용 목록에서 한 번 처리됩니다. 새 실패 분기는 없으며 이동 방식은 계속 `ContentLinkView`에 위임됩니다. |
| 보장하는 것과 보장하지 않는 것 | 09cec616f314에서 고정한 동작을 유지하면서 중복된 마크업을 제거합니다. 독립적인 new 기능나 선택자 변경는 아닙니다. |
| 다음 커밋 또는 관련 테스트 연결 | 프로젝트 동작의 최종 layering은 콘텐츠 → 선택자 → 표시 영역 래퍼 → ProjectLinkList → `ContentLinkView`입니다. 소개 영역은 콘텐츠 노출 위치를 라우트 소비자가 직접 필터링한 뒤 `ContentLinkView`를 사용합니다. |
<!-- learner:commit-44e4d062da50-record:end -->

#### 최소 코드 증거

<!-- learner:commit-44e4d062da50-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-44e4d062da50-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-44e4d062da50-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-44e4d062da50-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 가용성·외부 속성 중앙화 | ba8da56d3fcf | 선택자.ts의 null·필터·배포 중·외부 브랜치 | 노출 위치가 없어 표시 영역별 의미는 불완전 | 가용성과 이동 방식 입력은 선택자가 소유 |
| 내부·외부 이동 방식 일원화 | f63c978c71c9 | `ContentLinkView`의 `<a>`/Next Link 브랜치 | 표시 여부는 호출자에 남음 | 모든 링크 DOM 이동 방식이 공용 렌더러를 통과 |
| 배치 기반 노출 | daa6815a6dfa → 119ff9a92090 | 카드 검사 단계, LinkPlacement와 일반적인 선택자 | 소비자 전환 필요 | 카드·상세·소개 영역이 콘텐츠 노출 위치를 따름 |
| 결정적 회귀 보호 | 09cec616f314 | 프로젝트 링크.테스트.tsx 테스트 입력·단언문 | 브라우저 탐색·URL 검증은 미검증 | 원본 순서·쿼리·속성·누락을 컴포넌트 단위에서 고정 |
| 표현 중복 제거 | 44e4d062da50 | ProjectLinkList 추출 | 표시 영역 선택자는 의도적으로 별도 | 선택과 렌더링 책임이 분리 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| Hard-coded 타입 선택 | 콘텐츠 노출 위치를 표현할 수 없음 | daa6815a6dfa/119ff9a92090에서 노출 위치 선택자 도입 | 09cec616f314이 카드·상세 필터링을 검증 |
| 내부·외부 렌더링 중복 위험 | 외부 속성 또는 쿼리 보존 불일치 가능 | f63c978c71c9에서 `ContentLinkView` 도입 | 09cec616f314이 이동 방식 속성을 검증 |
| Detail·카드 마크업 중복 | 한 래퍼만 스타일 적용·null 동작이 달라질 수 있음 | 44e4d062da50에서 ProjectLinkList 추출 | 앞선 09cec616f314 규칙이 동작 기준 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 초기 | 원본 콘텐츠와 각 소비자 | 사용 가능 여부·이동 방식·표시 영역 규칙이 분산될 수 있음 |
| ba8/f63 | 선택자 + `ContentLinkView` | 규칙과 링크 이동 방식 렌더러가 분리됨 |
| daa/119/2d/ee | 콘텐츠 노출 위치 + 선택자·소개 영역 소비자 | 카드·상세는 선택자가 해석하고 소개 영역은 같은 노출 위치 판정 함수를 소비자가 직접 적용 |
| 44e4 최종 | 표시 영역 래퍼 + ProjectLinkList + `ContentLinkView` | 선택, 공용 DOM, 이동 방식이 단계별 소유 주체를 가짐 |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- 선호 연락처와 프로젝트 카드·상세 링크는 콘텐츠 원본에서 선택자를 거쳐 파생되며, 홈 소개 영역은 라우트 소비자가 콘텐츠 노출 위치를 직접 필터링합니다.
- 카드·상세·소개 영역은 콘텐츠 노출 위치를 따릅니다. 프로젝트 데모는 카드·상세 표시 여부 경로에서 배포 상태가 공개 중인지와 데모 href 존재 여부를 추가로 요구합니다.
- 표시 영역 래퍼는 선택된 배열과 로컬 옵션만 처리하고, 공용 목록이 빈·classes·아이콘를 처리합니다.
- `ContentLinkView`는 외부 기준과 내부 Next Link를 분기합니다.
- 잘못된 href, 원격 접근 가능 여부, 실제 브라우저 탐색은 이 개발 과정이 보장하지 않습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. 콘텐츠 로더가 `ContentLink`와 프로젝트 배포 상태·노출 위치 데이터를 제공합니다.
2. 카드·상세는 선택자가 노출 위치와 사용 가능 여부를 적용해 원본 순서를 보존한 배열을 만들고, 소개 영역 소비자는 같은 `hero` 노출 위치 판정 함수를 직접 적용합니다.
3. ProjectLinks 또는 `ProjectCardLinks`가 표시 영역 옵션을 적용합니다.
4. ProjectLinkList가 빈 배열이면 아무 DOM도 만들지 않고, 아니면 공통 동작 마크업을 만듭니다.
5. `ContentLinkView`가 외부이면 기준 속성을, 내부이면 템플릿·디버그가 보존된 app URL을 사용합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] S·A 설명을 중요도 B보다 깊게 작성했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 01-content-link-security-and-transport.md =====

===== BEGIN FILE: 02-content-media-loading-and-layout-stability.md =====
# 개발 과정: 콘텐츠 미디어 로딩과 레이아웃 안정성

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

프로필 사진과 프로젝트 화면 캡처 공용 컴포넌트가 콘텐츠 메타데이터를 어떻게 보존하고, 소개 영역과 갤러리 소비자가 즉시 로딩·지연 로딩 및 고정 aspect 레이아웃을 어떻게 선택하는지 복원합니다.

**경계:** 이 개발 과정은 공용 미디어 DOM과 로딩·레이아웃 규칙을 다룹니다. 이미지 최적화 처리 단계, 원본 자산 생성, 프로젝트 상세 정보 설계 자체는 포함하지 않습니다.

### 고정 불변 조건

- alt/src는 콘텐츠가 소유하고 미디어 공용 컴포넌트는 이를 임의로 대체하지 않습니다.
- `ProjectScreenshot`의 우선순위만 즉시 로딩을 선택하며 기본값은 지연 로딩입니다.
- `ProjectScreenshot`의 `<img>`에 고정 가로세로 비율이 있어 이미지 로딩 전에도 화면 캡처 높이를 계산할 수 있습니다.
- 선택적 프로필 사진이 없으면 소비자는 빈 자리표시자가 아니라 미디어 DOM 자체를 생략합니다.

## 2. 핵심 질문

- aa115c73ae30에서 원본 `<img>`를 감싸는 두 공용 컴포넌트가 어떤 로딩·화면 비율·`alt` 규칙을 갖는가?
- Lead·상세 소개 영역이 우선순위를 명시하고 갤러리가 기본값 지연 로딩을 유지하는 소비자 차이는 무엇인가?
- `profile.photo ? ... : null` 브랜치가 선택적 콘텐츠를 어떻게 표현하는가?
- 이 이력에 오류 대체 처리, 반응형 `srcset`, 프레임워크 이미지 최적화이 실제로 존재하는가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 이 개발 과정은 중요도 B 커밋만 포함하므로 각 커밋의 구체적인 역할, 구분 지점, 실패와 보장하지 않는 범위를 필요한 범위로 기록합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `aa115c73ae30` | feat(ui): 콘텐츠 이미지 프리미티브 추가 | B | CONTENT | 미디어 공용 컴포넌트 도입 |
| 2 | `b027f42669aa` | feat(home): 대표 프로젝트 쇼케이스 추가 | B | RENDERER | 대표 화면 캡처 소비자 |
| 3 | `06fff9a6e93b` | feat(project): 프로젝트 상세 소개 추가 | B | RENDERER | 상세 소개 영역 소비자 |
| 4 | `cabf3a0e378f` | feat(project): 프로젝트 구조와 증거 갤러리 추가 | B | RENDERER | 갤러리 지연 로딩 소비자 |
| 5 | `a00a6bf1af58` | feat(about): 프로필 사진 소개 추가 | B | RENDERER | 선택적 프로필 소비자 |

## 5. 커밋별 학습 기록

### 1. `aa115c73ae30` — feat(ui): 콘텐츠 이미지 프리미티브 추가

- **중요도:** B
- **태그:** CONTENT
- **개발 과정에서의 역할:** 미디어 공용 컴포넌트 도입

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/profile-photo.tsx`와 `project-screenshot.tsx`의 원본 `img` 속성을 확인합니다.
- `ProjectScreenshot`의 `priority` 기본값, 로딩 브랜치, `<img>` 가로세로 비율, figure 넘침 프레임, 객체 위치와 마우스 오버 클래스를 확인합니다.
- 같은 커밋의 ContentHint는 별도 검토 대상임을 표시하고 미디어 근거와 섞지 않습니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-aa115c73ae30-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Profile·프로젝트 이미지를 여러 소비자가 직접 렌더링하면 로딩 모드, 잘라내기, 화면 비율 확보와 `alt` 전달이 달라질 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `ProfilePhoto`와 `ProjectScreenshot`을 추가했습니다. Profile 사진은 즉시 로딩 `img`를 사용하고, 화면 캡처는 `priority=false` 기본값에서 지연 로딩, true에서 즉시 로딩을 사용합니다. 화면 캡처 `<img>`는 16:10 aspect와 object-fit 잘라내기를, `figure` 래퍼는 넘침과 프레임을 고정합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 콘텐츠 이미지 객체가 src/alt를 소유하고 공용 컴포넌트가 DOM 속성과 화면 구성 프레임을 소유합니다. 브라우저가 실제 이미지 요청과 decode 수명을 소유합니다. |
| 실패·누락·대체 처리 | 이미지 로딩 오류를 대체하는 상태나 콜백은 없습니다. 유효하지 않은 src/alt는 이 컴포넌트가 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 공용 로딩과 레이아웃 규칙을 만듭니다. 반응형 원본 선택, 프레임워크 이미지 최적화와 `width`·`height` 기반 고유 크기 계산은 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | b027f42669aa와 06fff9a6e93b가 우선순위 소비자를, cabf3a0e378f가 지연 로딩 갤러리 소비자를 연결합니다. |
<!-- learner:commit-aa115c73ae30-record:end -->

#### 최소 코드 증거

<!-- learner:commit-aa115c73ae30-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-aa115c73ae30-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-aa115c73ae30-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-aa115c73ae30-execution:end -->

### 2. `b027f42669aa` — feat(home): 대표 프로젝트 쇼케이스 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 대표 화면 캡처 소비자

#### 해당 SHA에서 확인할 실제 코드

- Design 홈 대표 프로젝트 브랜치에서 `ProjectScreenshot image={leadProject.screenshot} priority` 호출을 확인합니다.
- Lead 프로젝트 누락이 섹션 DOM에 미치는 영향을 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b027f42669aa-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 공용 화면 캡처 공용 컴포넌트는 있었지만 홈 대표 화면이 어떤 미디어를 즉시 로딩 처리할지 결정하지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | Design 홈이 대표 선택자의 대표 프로젝트를 조건부로 잡아 `ProjectScreenshot`에 `priority`를 전달합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 홈 라우트가 첫 화면에 보이는 우선순위 결정을 소유하고 공용 컴포넌트가 로딩 속성로 변환합니다. |
| 실패·누락·대체 처리 | Lead 프로젝트가 없으면 해당 화면 캡처 브랜치가 렌더링되지 않습니다. Broken 이미지 대체 처리는 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 홈 대표 화면 캡처가 즉시 로딩과 공용 화면 비율·프레임 규칙을 사용합니다. 다른 대표 이미지까지 모두 즉시 로딩된다는 보장은 이 커밋만으로는 없습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 06fff9a6e93b가 상세 소개 영역에도 같은 우선순위 규칙을 적용합니다. |
<!-- learner:commit-b027f42669aa-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b027f42669aa-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-b027f42669aa-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b027f42669aa-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-b027f42669aa-execution:end -->

### 3. `06fff9a6e93b` — feat(project): 프로젝트 상세 소개 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 상세 소개 영역 소비자

#### 해당 SHA에서 확인할 실제 코드

- 새 `ProjectDetailView`에서 소개 영역 화면 캡처 위치와 `priority` 속성을 확인합니다.
- Back 링크, 메타데이터, 링크와 화면 캡처 공용 컴포넌트의 담당 작업 구분 지점을 구분합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-06fff9a6e93b-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 프로젝트 상세 view와 그 소개 영역 미디어 소비자가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `ProjectDetailView`가 콘텐츠 요약·설명·동작 옆에 `ProjectScreenshot image={project.screenshot} priority`를 배치합니다. |
| 데이터·상태·DOM·자원 소유 주체 | Detail view가 소개 영역 노출 위치와 우선순위를, 공용 컴포넌트가 프레임·로딩을 소유합니다. |
| 실패·누락·대체 처리 | 프로젝트가 존재한다는 전제에서 view가 호출됩니다. 이 컴포넌트에는 누락된 화면 캡처 대체 처리가 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 상세 첫 화면 미디어는 즉시 로딩이고 공용 aspect·프레임 규칙을 사용합니다. Detail 라우팅/`notFound()`와 데이터 검증은 다른 레이어의 책임입니다. |
| 다음 커밋 또는 관련 테스트 연결 | cabf3a0e378f가 같은 view에 근거 갤러리를 추가하면서 기본값 지연 로딩 동작을 대조시킵니다. |
<!-- learner:commit-06fff9a6e93b-record:end -->

#### 최소 코드 증거

<!-- learner:commit-06fff9a6e93b-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-06fff9a6e93b-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-06fff9a6e93b-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-06fff9a6e93b-execution:end -->

### 4. `cabf3a0e378f` — feat(project): 프로젝트 구조와 증거 갤러리 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 갤러리 지연 로딩 소비자

#### 해당 SHA에서 확인할 실제 코드

- `ProjectDetailView`의 `project.screenshots.map`과 각 `ProjectScreenshot` 호출에 우선순위가 없는지 확인합니다.
- 설계 목록과 갤러리가 콘텐츠 배열 순서를 그대로 사용하는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-cabf3a0e378f-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Detail 소개 영역 한 장만 표현됐고 추가 근거 화면 캡처를 순서대로 보여 주는 갤러리가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 설계 섹션과 화면 캡처 섹션을 추가하고 `project.screenshots.map`에서 기본값 `ProjectScreenshot`을 사용합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 콘텐츠 배열이 근거 순서를 소유하고 상세 view가 반복을 소유합니다. 기초 컴포넌트 기본값이 지연 로딩을 결정합니다. |
| 실패·누락·대체 처리 | 빈 화면 캡처 배열이면 맵 결과가 비어 있지만 섹션 래퍼 자체는 남습니다. 이미지별 오류 대체 처리는 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 소개 영역은 즉시 로딩하고 갤러리는 지연 로딩하도록 구분합니다. 화면에 보이는 시점을 기준으로 한 별도 사전 로딩이나 동시 실행 제어는 없습니다. |
| 다음 커밋 또는 관련 테스트 연결 | a00a6bf1af58은 다른 선택적 미디어인 프로필 사진의 소비자 규칙을 완성합니다. |
<!-- learner:commit-cabf3a0e378f-record:end -->

#### 최소 코드 증거

<!-- learner:commit-cabf3a0e378f-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-cabf3a0e378f-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-cabf3a0e378f-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-cabf3a0e378f-execution:end -->

### 5. `a00a6bf1af58` — feat(about): 프로필 사진 소개 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 선택적 프로필 소비자

#### 해당 SHA에서 확인할 실제 코드

- `src/app/about/page.tsx`의 반응형 그리드 변경과 `content.profile.photo ? <ProfilePhoto .../> : null`을 확인합니다.
- ContentHint 경로가 사진까지 확장된 이유와 사진 누락 시 레이아웃 브랜치를 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-a00a6bf1af58-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | `ProfilePhoto` 공용 컴포넌트는 있었지만 소개 영역이 사진 필드를 소비하지 않아 선택적 프로필 미디어 규칙이 실제 라우트에서 완결되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 소개 영역을 문구·사진 그리드로 바꾸고 사진이 있을 때만 `ProfilePhoto`를 렌더링합니다. ContentHint 경로도 사진을 포함합니다. |
| 데이터·상태·DOM·자원 소유 주체 | Profile 콘텐츠가 존재 여부를 소유하고 소개 페이지가 조건부 조립을 소유합니다. `ProfilePhoto`가 `img` DOM을 소유합니다. |
| 실패·누락·대체 처리 | Photo가 없으면 null이며 빈 프레임이나 인위적으로 만든 대체 처리를 만들지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 선택적 프로필 사진이 소개 라우트에서 안전하게 소비됩니다. 이미지 로딩 실패나 `alt` 문구 품질은 여전히 콘텐츠와 브라우저가 담당하는 범위입니다. |
| 다음 커밋 또는 관련 테스트 연결 | 이 커밋으로 공용 컴포넌트 생성 → 대표·상세·갤러리·프로필 소비자 연결이 모두 확인됩니다. |
<!-- learner:commit-a00a6bf1af58-record:end -->

#### 최소 코드 증거

<!-- learner:commit-a00a6bf1af58-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-a00a6bf1af58-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-a00a6bf1af58-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-a00a6bf1af58-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Media 메타데이터 보존 | aa115c73ae30 | `ProfilePhoto`/`ProjectScreenshot`의 src·alt 전달 | 콘텐츠 검증 없음 | 공용 컴포넌트가 원본 메타데이터를 그대로 DOM에 전달 |
| 첫 화면에 보이는 우선순위 | b027f42669aa, 06fff9a6e93b | 우선순위 속성 → 즉시 로딩 | 실제 네트워크 우선순위 측정 없음 | 홈·상세 소개 영역이 즉시 로딩을 명시 |
| 근거 갤러리 지연 로딩 기본값 | cabf3a0e378f | 우선순위 없는 `ProjectScreenshot` 반복 | 빈 섹션 래퍼는 남음 | 갤러리 이미지는 지연 로딩 속성 사용 |
| 선택적 사진 누락 | a00a6bf1af58 | 조건부 `ProfilePhoto` | 로딩 오류 대체 처리 없음 | 사진 필드 부재 시 미디어 DOM 생략 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| 각 소비자의 직접 `img` 렌더링 위험 | 로딩·잘라내기·alt 처리 불일치 | aa115c73ae30 공용 컴포넌트 | 후속 소비자 변경 내용으로 사용 확인; 전용 테스트는 없음 |
| 모든 이미지를 같은 로딩으로 처리 | 소개 영역 지연 또는 갤러리 과도 즉시 로딩 가능 | 우선순위 속성과 소비자별 선택 | 정적 속성 경로만 확인; 네트워크 성능 테스트 없음 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 콘텐츠 | Image src/alt와 선택적 존재 여부 | 프로젝트·프로필 데이터 객체 |
| 기초 컴포넌트 | `img` 속성, 프레임, 잘라내기, 로딩 대응 | `ProfilePhoto`/`ProjectScreenshot` |
| 소비자 | 어디에 배치하고 우선순위를 켤지 | 홈, ProjectDetailView, 소개 |
| 브라우저 | 실제 fetch/decode·오류 렌더링 | Application 코드가 수명을 직접 관리하지 않음 |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- `ProfilePhoto`와 `ProjectScreenshot`이 원본 콘텐츠 미디어를 공통 DOM 규칙으로 렌더링합니다.
- 홈 대표와 상세 소개 영역은 우선순위로 즉시 로딩을 선택하고 상세 갤러리는 기본 지연 로딩을 사용합니다.
- 화면 캡처 `<img>`의 16:10 가로세로 비율이 높이를 계산하게 하고 figure 래퍼가 넘침·프레임을, 이미지가 object-fit 잘라내기를 적용합니다.
- 선택적 프로필 사진은 누락 시 완전히 생략됩니다.
- 오류 대체 처리, 반응형 원본 집합, 이미지 최적화과 실행 시점 성능 측정은 이 개발 과정에 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. 콘텐츠 로더가 프로필·프로젝트 이미지 메타데이터를 제공합니다.
2. 라우트·view가 미디어 존재 여부와 첫 화면에 보이는 여부를 판단합니다.
3. 소비자가 `ProfilePhoto` 또는 `ProjectScreenshot`과 선택적 우선순위를 호출합니다.
4. 기초 컴포넌트가 프레임과 `img` 속성을 생성합니다.
5. 브라우저가 즉시 로딩·지연 로딩 hint에 따라 자원을 요청하고 decode합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] 이 개발 과정에는 S·A 커밋이 없으며 중요도 B 범위에서 저장소에 맞춘 깊이를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 02-content-media-loading-and-layout-stability.md =====

===== BEGIN FILE: 03-progressive-reveal-to-server-first-rendering.md =====
# 개발 과정: 점진적 표시에서 서버 렌더링 우선 방식으로 전환

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

IntersectionObserver 기반 등장 효과 공용 컴포넌트가 동작 줄이기 설정 범위를 넓힌 뒤, 왜 클라이언트 실행 주기를 제거하고 서버에서 즉시 표시되는 마크업으로 전환됐는지 복원합니다.

**경계:** 이 개발 과정은 공용 등장 효과의 표시 여부·실행 주기와 동작 효과 대체 처리를 다룹니다. 개별 섹션 콘텐츠, 라우트 조립, 전역 성능 테스트 설계는 포함하지 않습니다.

### 고정 불변 조건

- 콘텐츠는 관찰자 지원, 하이드레이션 완료, 애니메이션 실행 여부와 무관하게 읽을 수 있어야 합니다.
- 관찰자를 사용하는 동안에는 첫 intersect에서만 표시되는로 전환하고 관찰자를 해제합니다.
- 동작 줄이기 설정에서는 opacity·변환·전환 효과 때문에 콘텐츠가 숨지 않습니다.
- 최종 상태에서는 등장 효과 마크업이 서버에서 이미 표시되는이며 클라이언트 효과를 요구하지 않습니다.

## 2. 핵심 질문

- 907d85b77bac의 서버 초기 렌더링과 브라우저 초기 상태가 실제로 같은가?
- IntersectionObserver 미지원 대체 처리, 첫 번째 교집합, 정리는 어떤 브랜치로 구현되는가?
- 29bb40579cb2와 af9191fc15ad가 동작 줄이기 설정 대상과 강도를 어떻게 확장하는가?
- b8164cfdddbd가 어떤 훅·참조·클라이언트 실행 범위를 제거하며 delay 속성의 의미는 무엇이 남는가?
- 최종 서버 렌더링 우선 전환을 직접 검증하는 테스트가 이 고정된 개발 과정에 존재하는가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 중요도 S·A는 설계 결정, 소유 주체, 실패 처리와 후속 근거를 중요도 B보다 깊게 복원합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `907d85b77bac` | feat(ui): 뷰포트 진입 공개 효과 추가 | B | - | 관찰자 기반 등장 효과 도입 |
| 2 | `29bb40579cb2` | style(a11y): 동적 목록의 모션 감소 지원 | B | RENDERER, A11Y | 동작 줄이기 설정 대상 확장 |
| 3 | `af9191fc15ad` | style(a11y): 모바일 헤더와 동작 감소 보강 | A | ARCH, RENDERER, A11Y | 전역 동작 효과 대체 처리 강화 |
| 4 | `b8164cfdddbd` | refactor(ui): reveal 콘텐츠를 server에서 즉시 표시 | A | ARCH, CONTENT, REFACTOR | 서버 렌더링 우선 표시 여부 전환 |

## 5. 커밋별 학습 기록

### 1. `907d85b77bac` — feat(ui): 뷰포트 진입 공개 효과 추가

- **중요도:** B
- **태그:** -
- **개발 과정에서의 역할:** 관찰자 기반 등장 효과 도입

#### 해당 SHA에서 확인할 실제 코드

- `src/components/portfolio/reveal.tsx`의 `useState` 초기화 함수를 SSR과 브라우저에서 각각 평가합니다.
- `useEffect`의 노드 없음·no-IntersectionObserver·교집합·정리 브랜치를 추적합니다.
- `src/app/globals.css`의 숨김, 표시 여부, 동작 줄이기 설정 선택자를 컴포넌트 클래스 전환 효과와 연결합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-907d85b77bac-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 등장 효과 래퍼가 없어서 콘텐츠는 즉시 보였고 화면 크기 진입점에 맞춘 공통 전환 효과와 정리 규칙이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 클라이언트 컴포넌트 `Reveal`을 추가했습니다. 상태는 브라우저에 IntersectionObserver가 없을 때만 초기 true이고, 효과는 노드를 observe해 첫 intersect에서 표시되는로 바꾸고 disconnect합니다. CSS는 기본 숨김·blur/translate, `is-visible`에서 복구합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 컴포넌트가 표시되는 상태와 관찰자 수명을 소유하고 브라우저 관찰자가 교집합 이벤트를 제공합니다. CSS가 시각 전환 효과를 소유합니다. |
| 실패·누락·대체 처리 | 참조 노드가 없으면 효과 종료, 관찰자 API가 없으면 초기 브라우저 상태에서 표시됩니다. 정리와 첫 교차 모두 disconnect합니다. SSR에서는 `window`가 없어 초기 상태가 `false`입니다. |
| 보장하는 것과 보장하지 않는 것 | 관찰자 지원 브라우저에서 처음 화면에 들어올 때의 등장 효과와 정리를 보장합니다. JavaScript·하이드레이션 전 서버 마크업은 숨김 클래스 상태일 수 있고 관찰자 콜백 시점은 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 29bb40579cb2·af9191fc15ad가 대체 처리를 강화하고, b8164cfdddbd가 하이드레이션 의존 자체를 제거합니다. |
<!-- learner:commit-907d85b77bac-record:end -->

#### 최소 코드 증거

<!-- learner:commit-907d85b77bac-excerpt:start -->
- **커밋:** `907d85b77bac`
- **경로:** `src/components/portfolio/reveal.tsx`
- **위치:** `Reveal useEffect`

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

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-907d85b77bac-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-907d85b77bac-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-907d85b77bac-execution:end -->

### 2. `29bb40579cb2` — style(a11y): 동적 목록의 모션 감소 지원

- **중요도:** B
- **태그:** RENDERER, A11Y
- **개발 과정에서의 역할:** 동작 줄이기 설정 대상 확장

#### 해당 SHA에서 확인할 실제 코드

- `prefers-reduced-motion` 블록에서 새로 추가된 `.tech-chip`, 경력·타임라인 가상 요소를 확인합니다.
- 기존 `.reveal-item`, `.motion-card`와 같은 변환·전환 효과 비활성화를 공유하는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-29bb40579cb2-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 초기 동작 줄이기 설정 블록은 등장 효과 자체를 즉시 보이게 했지만 이후 생긴 동적 목록·칩 안내선 가상 요소까지 모두 비활성화하지는 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 동작 줄이기 설정 선택 함수 목록을 기술 칩, 경력 행 식별 속성, 쌍으로 배치된·타임라인 안내선 가상 요소로 확장해 변환과 전환 효과를 제거했습니다. |
| 데이터·상태·DOM·자원 소유 주체 | User 미디어 설정이 규칙 입력이고 globals.css가 여러 공용 컴포넌트의 동작 효과 대체 처리를 소유합니다. |
| 실패·누락·대체 처리 | CSS 미디어 쿼리가 적용되지 않는 환경에는 영향이 없습니다. 애니메이션 속성 전체를 전역으로 제한하는 단계는 아직 아닙니다. |
| 보장하는 것과 보장하지 않는 것 | 동적 목록의 동작 효과를 줄이지만 클라이언트 등장 효과 상태·하이드레이션 의존성은 그대로입니다. |
| 다음 커밋 또는 관련 테스트 연결 | af9191fc15ad가 모든 전환 시간의 상한 설정와 마우스 오버 비활성화로 규칙을 더 강하게 만듭니다. |
<!-- learner:commit-29bb40579cb2-record:end -->

#### 최소 코드 증거

<!-- learner:commit-29bb40579cb2-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-29bb40579cb2-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-29bb40579cb2-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-29bb40579cb2-execution:end -->

### 3. `af9191fc15ad` — style(a11y): 모바일 헤더와 동작 감소 보강

- **중요도:** A
- **태그:** ARCH, RENDERER, A11Y
- **개발 과정에서의 역할:** 전역 동작 효과 대체 처리 강화

#### 해당 SHA에서 확인할 실제 코드

- `prefers-reduced-motion`의 universal 선택자와 가상 요소 선택자가 애니메이션·전환 지속 시간, 재생 횟수, 스크롤 동작을 어떻게 덮는지 확인합니다.
- `.motion-card:hover`, 터미널 wrap 등 명시적 변환·애니메이션 비활성화와 모든 전환 시간의 상한 설정의 역할을 구분합니다.
- 같은 커밋의 모바일 헤더 배경 흐림 효과 제거는 동작 효과 규칙과 별도 반응형 성능 결정임을 기록합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-af9191fc15ad-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 기초 컴포넌트별 선택자만으로는 새 애니메이션이나 마우스 오버 변환이 빠질 수 있었고, 모바일 헤더의 배경 흐림 효과도 작은 화면 크기에서 계속 적용됐습니다. |
| 실제 변경 파일·심볼·호출 경로 | 동작 줄이기 설정에서 모든 요소와 가상 요소의 애니메이션 지속 시간을 0.01ms, 재생 횟수를 1, 전환 지속 시간을 0.01ms로 강제하고 스크롤 동작을 auto로 둡니다. 특정 animated 요소와 마우스 오버 변환도 명시적으로 비활성화합니다. 640px 이하 헤더 배경 흐림 효과도 제거합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 전역 스타일시트가 사용자 설정에 대한 여러 컴포넌트에 공통으로 적용되는 규칙 소유 주체가 됩니다. 개별 컴포넌트는 해당 규칙을 반복 구현할 필요가 줄어듭니다. |
| 실패·누락·대체 처리 | CSS를 비활성화하거나 미디어 쿼리가 일치하지 않으면 적용되지 않습니다. 0.01ms는 애니메이션 속성을 제거하는 것이 아니라 사실상 즉시 종료시키는 규칙입니다. |
| 보장하는 것과 보장하지 않는 것 | 새 동작 효과 공용 컴포넌트가 추가돼도 기본 전환 시간 상한를 받게 합니다. 하지만 등장 효과의 숨김 초기 클래스와 하이드레이션 시점 문제 자체를 해결하지는 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | b8164cfdddbd가 표시 여부 상태 소유 주체를 클라이언트에서 서버 마크업으로 이동해 콘텐츠 사용 가능 여부를 동작 효과와 분리합니다. |
<!-- learner:commit-af9191fc15ad-record:end -->

#### 최소 코드 증거

<!-- learner:commit-af9191fc15ad-excerpt:start -->
- **커밋:** `af9191fc15ad`
- **경로:** `src/app/globals.css`
- **위치:** `universal selector inside prefers-reduced-motion`

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

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-af9191fc15ad-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-af9191fc15ad-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-af9191fc15ad-execution:end -->

### 4. `b8164cfdddbd` — refactor(ui): reveal 콘텐츠를 server에서 즉시 표시

- **중요도:** A
- **태그:** ARCH, CONTENT, REFACTOR
- **개발 과정에서의 역할:** 서버 렌더링 우선 표시 여부 전환

#### 해당 SHA에서 확인할 실제 코드

- `Reveal`에서 제거된 `"use client"`, 훅, 참조, 관찰자와 남은 속성·마크업을 비교합니다.
- 항상 `reveal-item is-visible`인 서버 출력이 globals.css와 결합해 어떤 상태를 제거하는지 확인합니다.
- `delay`가 여전히 인라인 transitionDelay를 만들지만 화면 크기 진입점 열기 버튼은 사라졌다는 점을 구분합니다.
- 후속 테스트가 고정된 커밋 집합 안에 있는지 확인하고 없으면 명시합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b8164cfdddbd-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 초기 디자인은 SSR에서 숨김 마크업을 만들고 하이드레이션·효과·관찰자가 성공해야 콘텐츠를 표시되는로 바꿨습니다. Progressive enhancement가 아니라 JavaScript 사용 가능 여부가 콘텐츠 표시 여부의 전제가 되는 위험이 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 등장 효과를 서버 컴포넌트와 호환되는 입력에만 의존하는 함수으로 바꿔 훅·참조·관찰자·클라이언트 지시문을 제거했습니다. Class는 항상 `reveal-item is-visible`이며 delay 스타일과 `as` polymorphism만 유지합니다. |
| 데이터·상태·DOM·자원 소유 주체 | Visibility 소유 주체가 클라이언트 상태·관찰자에서 서버 마크업으로 이동했습니다. CSS는 표현만 소유하고 실행 주기 자원은 더 이상 없습니다. |
| 실패·누락·대체 처리 | 관찰자 미지원, 하이드레이션 지연, 효과 미실행 브랜치 자체가 제거됩니다. 별도 정리도 필요 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 콘텐츠가 첫 HTML에서 표시됨을 보장하고 이 래퍼 자체의 클라이언트 실행 범위와 실행 주기를 제거합니다. 실제 생성된 번들 크기는 측정하지 않았습니다. 화면에 들어오는 시점 애니메이션은 더 이상 보장하지 않습니다. `delay`는 인라인 `transitionDelay`로 serialize되지만 컴포넌트가 표시 여부 클래스 전환 효과를 일으키지는 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 이 고정된 개발 과정에는 이 리팩터링를 실행한 전용 회귀 테스트가 없습니다. 분류 07의 별도 성능·회귀 분류와 정적 원본 정적 검토가 후속 검증 맥락입니다. |
<!-- learner:commit-b8164cfdddbd-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b8164cfdddbd-excerpt:start -->
- **커밋:** `b8164cfdddbd`
- **경로:** `src/components/portfolio/reveal.tsx`
- **위치:** `Reveal after refactor`

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

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-b8164cfdddbd-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b8164cfdddbd-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-b8164cfdddbd-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 첫 교차 시점 등장 효과 | 907d85b77bac | IntersectionObserver + 표시 여부 상태 | 서버 렌더링 시 숨김·하이드레이션 의존 | 후속 리팩터링 전까지 브라우저 상호작용 제공 |
| 동작 줄이기 설정 콘텐츠 표시 여부 | 907d85b77bac → af9191fc15ad | 컴포넌트 선택자에서 전역 전환 시간 상한까지 확장 | 클라이언트 상태는 여전히 필요 | 동작 효과 설정에서 숨김·긴 애니메이션 방지 |
| 서버 렌더링 우선 사용 가능 여부 | b8164cfdddbd | 클라이언트 지시문·훅·관찰자 제거, 표시 여부 고정 | 화면 크기 등장 효과 동작 포기 | 첫 HTML에서 콘텐츠 표시되는 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| 서버 렌더링 시 숨김 + 하이드레이션·관찰자 필요 | JS 지연·실패 시 콘텐츠 표시 여부 위험 | b8164cfdddbd 서버에서 즉시 표시되는 마크업 | 고정된 개발 과정 내 전용 실행 시점 테스트 없음; 정확한 변경 내용으로 정적 확인 |
| 기초 컴포넌트별 동작 줄이기 설정 누락 가능 | 새 애니메이션이 설정를 무시할 수 있음 | 29bb40579cb2와 af9191fc15ad에서 범위 확장 | CSS 선택자 정적 검토; 시각 브라우저 테스트 없음 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 907d 초기 | 등장 효과 클라이언트 상태 + IntersectionObserver | 표시 여부와 관찰자 수명을 컴포넌트가 관리 |
| 29bb/af919 | 전역 CSS | 동작 줄이기 설정 규칙이 여러 컴포넌트에 공통으로 적용되는 소유 주체로 확대 |
| b816 최종 | 서버 마크업 | 콘텐츠 표시 여부는 서버 렌더링이 소유; 클라이언트 실행 주기 제거 |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- 등장 효과는 서버와 호환되는 래퍼이며 첫 마크업부터 `is-visible`입니다.
- IntersectionObserver, 상태, 참조, 효과, disconnect 정리는 제거됐습니다.
- 전역 동작 줄이기 설정 규칙은 애니메이션·전환 효과를 사실상 즉시 종료하고 특정 변환을 비활성화합니다.
- 화면 진입 기반 등장 효과는 최종 규칙이 아닙니다.
- 고정된 개발 과정 안에는 서버 렌더링 우선 불변 조건을 실행 검증하는 전용 테스트가 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. 서버 컴포넌트가 `Reveal`을 일반 래퍼처럼 호출합니다.
2. 등장 효과는 선택된 요소 타입에 children, className, delay 스타일을 붙입니다.
3. Markup은 처음부터 `reveal-item is-visible`입니다.
4. 일반 동작 효과에서도 컴포넌트는 표시 여부 클래스를 전환하지 않습니다. 동작 줄이기 설정에서는 전역 덮어쓰기가 남은 시점·변환 선언을 비활성화합니다.
5. 클라이언트 관찰자 실행 주기는 존재하지 않습니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] S·A 설명을 중요도 B보다 깊게 작성했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 03-progressive-reveal-to-server-first-rendering.md =====

===== BEGIN FILE: 04-terminal-state-machine-and-motion-fallback.md =====
# 개발 과정: 터미널 상태 기계와 모션 대체 처리

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

AnimatedTerminal의 자리표시자 형식 변환, typing/hold/erase 타이머 상태 기계, 정리, 터미널 CSS와 동작 줄이기 설정 동작, Classic 소개 영역 통합을 복원합니다.

**경계:** 이 개발 과정은 터미널 미리보기 공용 컴포넌트의 상태와 표현을 다룹니다. 터미널 콘텐츠 스키마 검증, 홈 라우트 전체 섹션 설계, 일반 동작 효과 규칙은 각각 다른 개발 과정의 책임입니다.

### 고정 불변 조건

- 터미널 명령 출력은 프로필·프로젝트·기술 스택 의존성이 바뀔 때 파생되고 상태 기계는 그 결과 배열을 순환합니다.
- Effect 실행마다 최대 한 시간 제한을 소유하며 의존성 변화·해제에서 clear합니다.
- 동작 줄이기 설정에서는 타이머 progression을 시작하지 않고 읽을 수 있는 초기 명령·출력을 유지합니다.
- CSS 애니메이션은 동작 줄이기 설정에서 비활성화됩니다.
- 소비자는 터미널 명령가 비어 있지 않다는 전제를 제공합니다.

## 2. 핵심 질문

- formatTerminalLine이 어떤 자리표시자만 치환하며 알 수 없는 자리표시자는 어떻게 되는가?
- 초기 commandIndex/typedCommand·단계가 동작 줄이기 설정 즉시 반환과 결합해 무엇을 표시하는가?
- typing/hold/erase 각 브랜치의 시간 제한과 다음 상태, 정리는 무엇인가?
- CSS 프레임·sheen·출력·커서 애니메이션이 어느 커밋에 나뉘어 추가되는가?
- 빈 명령 배열에 대한 검사가 실제로 존재하는가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 이 개발 과정은 중요도 B 커밋만 포함하므로 각 커밋의 구체적인 역할, 구분 지점, 실패와 보장하지 않는 범위를 필요한 범위로 기록합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `f60d46857715` | feat(home): 애니메이션 터미널 상호작용 추가 | B | RENDERER | 타이머 상태 기계 도입 |
| 2 | `1ff1da788f7a` | style(home): 터미널 프레임과 부유 장식 추가 | B | RENDERER | 프레임·sheen 표현 |
| 3 | `335a00fcf40c` | style(home): 터미널 출력과 커서 동작 추가 | B | RENDERER | 출력·커서 동작 효과 |
| 4 | `cdb68fdf59f9` | feat(home): 클래식 홈 히어로 구성 | B | RENDERER | Classic 소개 영역 통합 |

## 5. 커밋별 학습 기록

### 1. `f60d46857715` — feat(home): 애니메이션 터미널 상호작용 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 타이머 상태 기계 도입

#### 해당 SHA에서 확인할 실제 코드

- `formatTerminalLine`의 치환 키와 `useMemo` 의존성을 확인합니다.
- `commandIndex`, `typedCommand`, `phase` 초기값과 `activeCommand` access를 확인합니다.
- typing/hold/erase 브랜치별 delay와 상태 갱신, modulo progression, 정리를 표로 재구성합니다.
- `matchMedia(prefers-reduced-motion)` 즉시 반환에서 초기 DOM이 무엇인지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-f60d46857715-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Classic 홈에 데이터 기반 터미널 미리보기와 타이머 실행 주기가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `AnimatedTerminal`을 클라이언트 컴포넌트로 추가했습니다. `formatTerminalLine`은 `{handle}`, `{role}`, `{location}`, `{projectCount}`, `{stackCount}`를 `replaceAll`로 치환하고 알 수 없는 토큰은 그대로 둡니다. 그 결과 배열 위에 목록·타입이 지정된 문구·단계 상태를 두며, 효과는 typing 42ms, 완료된 typing 후 520ms, hold 1700ms, erase 24ms, 다음 명령 전 220ms 시간 제한을 사용하고 정리에서 clear합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 컴포넌트가 formatted 명령, 상태, 시간 제한 식별자을 소유합니다. 콘텐츠가 명령·출력 템플릿을, 브라우저 타이머가 scheduling을 소유합니다. |
| 실패·누락·대체 처리 | 동작 줄이기 설정 match이면 효과가 시간 제한을 만들지 않습니다. 정리은 의존성 변경·해제에서 대기 중인 타이머를 해제합니다. 그러나 빈 명령를 막는 검사가 없어 `activeCommand.command` access와 modulo 길이가 비어 있지 않은 입력을 전제로 합니다. |
| 보장하는 것과 보장하지 않는 것 | 정상적인 비어 있지 않은 입력에서 순환 입력 상태와 타이머 정리를 보장합니다. 실제 경과 시간의 정확성, 백그라운드 탭의 타이머 제한과 빈 스키마 복구는 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 1ff1da788f7a·335a00fcf40c가 시각 프레임·출력 동작 효과를 추가하고 cdb68fdf59f9가 실제 콘텐츠 소비자를 연결합니다. |
<!-- learner:commit-f60d46857715-record:end -->

#### 최소 코드 증거

<!-- learner:commit-f60d46857715-excerpt:start -->
- **커밋:** `f60d46857715`
- **경로:** `src/components/portfolio/animated-terminal.tsx`
- **위치:** `AnimatedTerminal effect`

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

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-f60d46857715-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-f60d46857715-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-f60d46857715-execution:end -->

### 2. `1ff1da788f7a` — style(home): 터미널 프레임과 부유 장식 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프레임·sheen 표현

#### 해당 SHA에서 확인할 실제 코드

- globals.css의 `.hero-terminal-wrap`, `.terminal-window`, titlebar·본문과 `terminal-sheen` 키프레임을 확인합니다.
- 장식용 가상 요소의 pointer-events와 동작 줄이기 설정 선택자가 어떤 애니메이션을 끄는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-1ff1da788f7a-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 상태 전이는 문구 DOM만 제공했고 터미널 프레임, titlebar, sheen, floating 장식이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 터미널 래퍼·프레임·제목 표시줄·본문 CSS와 가상 요소 장식, 광택 키프레임을 추가했습니다. 장식 요소에는 `pointer-events: none`을 적용합니다. |
| 데이터·상태·DOM·자원 소유 주체 | CSS가 시각 수명과 stacking을 소유하고 컴포넌트 상태에는 변화가 없습니다. |
| 실패·누락·대체 처리 | 동작 줄이기 설정에서 터미널 window sheen은 끄지만 이 커밋 시점의 선택자가 모든 래퍼 애니메이션을 포함하는지는 제한적입니다. 후속 전역 규칙이 보강합니다. |
| 보장하는 것과 보장하지 않는 것 | 터미널 chrome과 장식을 제공하지만 입력 상태의 정확성나 접근성 의미 구조를 바꾸지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 335a00fcf40c가 출력·커서 동작 효과를 추가하고 af9191fc15ad가 더 넓은 동작 줄이기 설정 규칙을 적용합니다. |
<!-- learner:commit-1ff1da788f7a-record:end -->

#### 최소 코드 증거

<!-- learner:commit-1ff1da788f7a-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-1ff1da788f7a-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-1ff1da788f7a-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-1ff1da788f7a-execution:end -->

### 3. `335a00fcf40c` — style(home): 터미널 출력과 커서 동작 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 출력·커서 동작 효과

#### 해당 SHA에서 확인할 실제 코드

- `.terminal-line`, `.terminal-output`, 가상 식별 속성, `.terminal-caret`와 키프레임을 확인합니다.
- 동작 줄이기 설정 선택자가 출력·커서 애니메이션을 모두 끄는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-335a00fcf40c-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 터미널 프레임은 있었지만 긴 줄 줄바꿈, 출력 등장 효과와 커서 표시가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 어디서나 줄바꿈, 출력 식별 속성·진입점 애니메이션, 깜박이는 커서와 관련 키프레임을 추가했습니다. |
| 데이터·상태·DOM·자원 소유 주체 | CSS가 일시적인 시각 효과를 소유하고 React 상태는 타입이 지정된 문구·출력 표시 여부만 소유합니다. |
| 실패·누락·대체 처리 | 동작 줄이기 설정에서 출력과 커서 애니메이션을 비활성화합니다. 커서는 `aria-hidden`이라 보조 기술용 문구에 포함되지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 긴 터미널 줄의 넘침 위험을 줄이고 시각 피드백을 제공합니다. 화면 읽기 프로그램 실시간 안내나 타이핑 내용 낭독은 제공하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | cdb68fdf59f9가 터미널 컴포넌트를 Classic 소개 영역에 배치합니다. |
<!-- learner:commit-335a00fcf40c-record:end -->

#### 최소 코드 증거

<!-- learner:commit-335a00fcf40c-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-335a00fcf40c-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-335a00fcf40c-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-335a00fcf40c-execution:end -->

### 4. `cdb68fdf59f9` — feat(home): 클래식 홈 히어로 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Classic 소개 영역 통합

#### 해당 SHA에서 확인할 실제 코드

- `src/designs/classic/home-route.tsx`의 `ClassicHeroSection`에서 AnimatedTerminal 속성을 추적합니다.
- 프로필, 프로젝트.길이, techStack.길이와 화면 구성 터미널 콘텐츠가 형식 변환 입력으로 전달되는지 확인합니다.
- ContentHint 경로와 터미널 노출 위치 래퍼를 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-cdb68fdf59f9-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 터미널 공용 컴포넌트와 CSS는 있었지만 실제 Classic 홈 콘텐츠·데이터 개수를 전달하는 라우트 소비자가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | Classic 소개 영역이 AnimatedTerminal에 프로필, 프로젝트 개수, 기술 스택 개수, 화면 구성 터미널 객체를 전달하고 터미널 래퍼에 배치합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 라우트가 콘텐츠 집계 방식과 노출 위치를, 터미널 컴포넌트가 형식 변환·상태를 소유합니다. |
| 실패·누락·대체 처리 | 라우트는 명령 빈 상태를 별도 검사하지 않습니다. 콘텐츠 규칙이 유효한·비어 있지 않은 터미널 데이터를 제공해야 합니다. |
| 보장하는 것과 보장하지 않는 것 | 실제 포트폴리오 데이터를 터미널 상태 기계에 연결합니다. 다른 디자인 템플릿에서의 사용이나 스키마 검증은 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 이 커밋으로 콘텐츠 → 형식 변환 → 상태 기계 → CSS 프레임의 실행 경로가 완성됩니다. |
<!-- learner:commit-cdb68fdf59f9-record:end -->

#### 최소 코드 증거

<!-- learner:commit-cdb68fdf59f9-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-cdb68fdf59f9-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-cdb68fdf59f9-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-cdb68fdf59f9-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 비어 있지 않은 순환 명령 상태 | `f60d46857715` | 단계별 타이머와 나머지 연산으로 목록 순환 | 빈 명령 목록 검사 없음 | 유효한 콘텐츠에서 입력·유지·삭제를 반복 |
| 타이머 정리 | f60d46857715 | 효과 정리 clearTimeout | 브라우저 throttling은 제어하지 않음 | 의존성 변경·해제 시 pending 타이머 해제 |
| 동작 줄이기 설정 readable 초기 상태 | f60d46857715 + CSS 커밋 | matchMedia 즉시 반환, 애니메이션 없음 | 실제 OS·브라우저 조합표 미실행 | 첫 전체 명령·출력 유지, 시각 애니메이션 중단 |
| Real 데이터 통합 | cdb68fdf59f9 | ClassicHeroSection 속성 | 스키마 검증은 외부 | 프로필·개수·터미널 콘텐츠를 실제 소비자가 전달 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| 타이머 효과가 정리되지 않을 위험 | 컴포넌트 해제 뒤 상태 갱신 또는 중복 타이머 | `f60d46857715`의 정리 함수에서 `clearTimeout` 호출 | 전용 가짜 타이머 테스트는 고정된 개발 과정에 없음 |
| 동작 줄이기 설정 무시 위험 | typing/CSS 애니메이션 지속 | 효과 즉시 반환 + CSS 애니메이션 없음 | 정적 브랜치 확인; 브라우저 설정 테스트 없음 |
| 빈 명령 입력 | activeCommand un정의된 및 modulo 0 위험 | 이 개발 과정에서 수정 없음 | 콘텐츠 검증이 전제이며 회귀 테스트 없음 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 콘텐츠 | 터미널 명령·출력 템플릿 | 화면 구성 데이터 |
| 라우트 | 프로필·projectCount·stackCount 전달과 노출 위치 | ClassicHeroSection |
| AnimatedTerminal | formatted 명령 배열, 단계·목록·문구 상태, 시간 제한 정리 | 클라이언트 컴포넌트 |
| CSS | 프레임, sheen, 출력, 커서, 동작 줄이기 설정 시각 대체 처리 | globals.css |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- Classic 소개 영역이 콘텐츠 값으로 결정되는 터미널 미리보기를 렌더링합니다.
- 상태 전이는 typing → hold → erase → next 명령을 반복하고 대기 중인 타이머를 정리합니다.
- 동작 줄이기 설정이면 타이머 progression을 시작하지 않고 CSS 애니메이션도 중단됩니다.
- 형식 변환은 정해진 자리표시자만 replaceAll하며 알 수 없는 토큰은 그대로 남습니다.
- Commands 빈 복구와 전용 타이머·브라우저 테스트는 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. ClassicHeroSection이 프로필·개수·터미널 콘텐츠를 AnimatedTerminal에 전달합니다.
2. useMemo가 각 출력 줄의 알려진 자리표시자를 실제 값으로 치환합니다.
3. 상태가 현재 선택된 명령과 단계를 선택합니다.
4. Effect가 동작 효과 설정을 확인하고 단계에 맞는 단일 시간 제한을 예약합니다.
5. Render는 타입이 지정된 명령과 조건부 출력을 터미널 DOM에 놓고 CSS가 프레임·동작 효과를 표현합니다.
6. Effect rerun 또는 해제 시 대기 중인 타이머를 clear합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] 이 개발 과정에는 S·A 커밋이 없으며 중요도 B 범위에서 저장소에 맞춘 깊이를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 04-terminal-state-machine-and-motion-fallback.md =====

===== BEGIN FILE: 05-technology-stack-icon-list-and-marquee.md =====
# 개발 과정: 기술 스택 아이콘·목록·흐르는 표시

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

Simple Icons 대응과 handcrafted 대체 처리, ID 기반 기술 스택 목록, 복제한 트랙 흐르는 목록, 마우스 오버 일시 정지와 동작 줄이기 설정 대체 처리를 순서대로 복원합니다.

**경계:** 이 개발 과정은 기술 스택 시각 공용 컴포넌트를 다룹니다. 기술 스택 콘텐츠 스키마와 `resolveTechStackItem`의 검증·대체 처리 기준, 홈 섹션 전체 조립은 다른 개발 과정이 소유합니다.

### 고정 불변 조건

- 알려진 브랜드 아이콘은 타입이 지정된 맵을 사용하고 맵에 없는 지원 대상 의미상 아이콘은 로컬 SVG 대체 처리를 사용합니다.
- `StackList`는 콘텐츠 ID 순서를 보존하고 선택적 한도는 앞에서부터 필요한 개수만 선택합니다.
- 흐르는 목록은 보조 기술에 동일 항목을 두 번 노출하지 않습니다.
- 연속 동작 효과는 마우스를 올리면 일시 정지하고 동작 줄이기 설정에서 중단됩니다.

## 2. 핵심 질문

- Partial Record를 사용한 이유와 맵 조회 실패가 실행 시점에서 어떤 브랜치로 이어지는가?
- FallbackIcon의 이름이 있는 분기와 최종 일반적인 circle이 어떤 지원하지 않는 상태를 표현하는가?
- `StackList`가 ID를 해석 함수로 넘기고 색상 CSS 변수를 만드는 담당 작업 연결 순서는 무엇인가?
- 흐르는 목록이 트랙을 복제하면서 키와 `aria-hidden`을 어떻게 다르게 만드는가?
- 빈 항목, 18개 초과 항목, 마우스 오버·동작 줄이기 설정에서 실제 규칙은 무엇인가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 이 개발 과정은 중요도 B 커밋만 포함하므로 각 커밋의 구체적인 역할, 구분 지점, 실패와 보장하지 않는 범위를 필요한 범위로 기록합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `ebc245105c03` | feat(stack): 기술 스택 아이콘 매핑 추가 | B | RENDERER | 브랜드 아이콘 맵 도입 |
| 2 | `3d9b847d8094` | feat(stack): 기술 스택 폴백 아이콘 추가 | B | RENDERER | 아이콘 렌더러와 대체 처리 |
| 3 | `6aa8ee3b90b1` | feat(stack): 공용 기술 스택 목록 추가 | B | RENDERER | ID 기반 칩 목록 |
| 4 | `48559efebf68` | feat(stack): 기술 스택 마키 프리미티브 추가 | B | RENDERER | 복제 트랙 구조 |
| 5 | `3b9c1a636356` | style(stack): 기술 스택 마키 동작 추가 | B | RENDERER | 연속 동작 효과와 대체 처리 |

## 5. 커밋별 학습 기록

### 1. `ebc245105c03` — feat(stack): 기술 스택 아이콘 매핑 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 브랜드 아이콘 맵 도입

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/tech-icon.tsx`의 Simple Icons 가져오기와 `Partial<Record<TechStackIcon, SimpleIcon>>`을 확인합니다.
- Union 전체를 강제하지 않는 일부 맵이 후속 대체 처리 필요성을 어떻게 남기는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-ebc245105c03-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 기술 스택 항목의 아이콘 식별자를 SVG 경로로 바꾸는 공용 대응이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | C/CMake/C++/Docker/ESLint/Next/Node/PostgreSQL/Prisma/React/Redis/Tailwind/TypeScript/Vitest를 Simple Icons 객체에 매핑하는 일부 레코드를 추가했습니다. |
| 데이터·상태·DOM·자원 소유 주체 | TechStackIcon 식별자는 콘텐츠와 타입이 소유하고, 맵은 브랜드 SVG 원본 선택을 소유합니다. |
| 실패·누락·대체 처리 | 맵 조회 실패를 처리하는 렌더러는 아직 없으므로 이 커밋만으로는 아이콘 출력을 만들지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 알려진 브랜드 식별자의 원본을 중앙화합니다. 모든 유니언 member를 Simple Icons가 지원한다는 보장은 의도적으로 하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 3d9b847d8094가 맵 조회 실패와 의미상 아이콘을 처리하는 TechIcon/FallbackIcon을 추가합니다. |
<!-- learner:commit-ebc245105c03-record:end -->

#### 최소 코드 증거

<!-- learner:commit-ebc245105c03-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-ebc245105c03-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-ebc245105c03-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-ebc245105c03-execution:end -->

### 2. `3d9b847d8094` — feat(stack): 기술 스택 폴백 아이콘 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 아이콘 렌더러와 대체 처리

#### 해당 SHA에서 확인할 실제 코드

- `TechIcon`의 simpleIcon truthy 브랜치와 대체 처리 SVG 브랜치를 확인합니다.
- `FallbackIcon`의 터미널·shield·검사·database·순서·box/api/json·기본값 분기를 목록합니다.
- `aria-hidden`과 색상/`currentColor` 사용을 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-3d9b847d8094-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Brand 대응은 있었지만 맵 조회 실패를 실제 SVG로 표현하거나 공개 아이콘 컴포넌트로 렌더링하는 경로가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `TechIcon`은 대응되는 Simple Icons 경로를 사용하고, 찾지 못하면 `FallbackIcon`이 이름이 있는 선 아이콘이나 일반 원형 표시점을 반환합니다. 바깥 `svg`는 장식용으로 `aria-hidden`을 적용합니다. |
| 데이터·상태·DOM·자원 소유 주체 | TechIcon이 브랜치 선택과 SVG 래퍼를, FallbackIcon이 의미상 경로를 소유합니다. 문구는 호출자가 별도로 제공합니다. |
| 실패·누락·대체 처리 | 알 수 대응표에 없는 맵 아이콘도 일반적인 대체 처리로 DOM을 만듭니다. 유효하지 않은 색상은 직접 검증하지 않으며 줄 아이콘은 `currentColor`를 사용합니다. |
| 보장하는 것과 보장하지 않는 것 | 지원하는 유니언 아이콘이 빈 자리 없이 시각 표시를 갖게 합니다. Icon만으로 접근 가능한 이름을 제공하지는 않으며 호출자 문구가 필요합니다. |
| 다음 커밋 또는 관련 테스트 연결 | 6aa8ee3b90b1이 문구와 아이콘을 함께 제공하는 `StackList`를 만듭니다. |
<!-- learner:commit-3d9b847d8094-record:end -->

#### 최소 코드 증거

<!-- learner:commit-3d9b847d8094-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-3d9b847d8094-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-3d9b847d8094-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-3d9b847d8094-execution:end -->

### 3. `6aa8ee3b90b1` — feat(stack): 공용 기술 스택 목록 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** ID 기반 칩 목록

#### 해당 SHA에서 확인할 실제 코드

- `StackList`의 선택적 한도 브랜치와 `resolveTechStackItem(item)` 호출을 확인합니다.
- 키가 원본 ID인지, CSS 변수 색상과 TechIcon·문구가 어떻게 연결되는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-6aa8ee3b90b1-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Icon 렌더러는 있었지만 프로젝트 기술 스택 ID 배열을 순서가 지정된 칩로 바꾸는 공용 목록이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | Items를 선택적 `slice(0, limit)`한 뒤 각 ID를 해석 함수로 해석하고, 색상 CSS 변수, TechIcon, 문구를 가진 목록 항목을 생성합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 호출자가 ID 순서와 한도를, 해석 함수가 기준 기술 스택 메타데이터를, `StackList`가 칩 DOM을 소유합니다. |
| 실패·누락·대체 처리 | 빈 항목은 빈 `<ul>`을 만듭니다. 누락된 ID의 처리 방식은 `resolveTechStackItem`에 위임되며 이 컴포넌트가 catch하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 콘텐츠 순서와 한도를 보존하는 공용 칩 목록을 제공합니다. 해석 함수의 정확성와 중복된 ID 제거는 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 48559efebf68은 전체 `TechStackItem` 배열을 사용하는 별도 연속 흐르는 목록 공용 컴포넌트를 추가합니다. |
<!-- learner:commit-6aa8ee3b90b1-record:end -->

#### 최소 코드 증거

<!-- learner:commit-6aa8ee3b90b1-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-6aa8ee3b90b1-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-6aa8ee3b90b1-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-6aa8ee3b90b1-execution:end -->

### 4. `48559efebf68` — feat(stack): 기술 스택 마키 프리미티브 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 복제 트랙 구조

#### 해당 SHA에서 확인할 실제 코드

- `TechMarquee`의 `slice(0, 18)`과 두 `TechMarqueeTrack` 호출을 확인합니다.
- 두 번째 트랙의 `aria-hidden`과 배포 중·ghost 키 접두사를 확인합니다.
- Outer ARIA 문구, inner 목록 의미 구조와 빈 입력 동작을 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-48559efebf68-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 정적 칩 목록만 있었고 연속 scrolling을 위한 중복된 순서 구성이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 최대 18개를 선택해 동일 트랙을 두 번 렌더링합니다. 두 번째 트랙은 `aria-hidden`이고 키에는 ghost 접두사를 사용해 React 식별 정보를 분리합니다. |
| 데이터·상태·DOM·자원 소유 주체 | TechMarquee가 표시되는 일부 항목과 트랙 중복을, 각 트랙이 목록 DOM을 소유합니다. CSS가 실제 이동 효과를 소유할 예정입니다. |
| 실패·누락·대체 처리 | 18개를 넘는 항목은 잘립니다. 빈 입력이면 두 빈 목록이 남습니다. Duplicate 의미상 안내는 복제 트랙의 `aria-hidden`으로 막습니다. |
| 보장하는 것과 보장하지 않는 것 | Seamless 애니메이션을 위한 DOM과 접근성 구분 지점을 만듭니다. 이 커밋만으로는 트랙이 움직이지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 3b9c1a636356이 연속 CSS, 마우스 오버 일시 정지와 동작 줄이기 설정 대체 처리를 추가합니다. |
<!-- learner:commit-48559efebf68-record:end -->

#### 최소 코드 증거

<!-- learner:commit-48559efebf68-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-48559efebf68-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-48559efebf68-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-48559efebf68-execution:end -->

### 5. `3b9c1a636356` — style(stack): 기술 스택 마키 동작 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 연속 동작 효과와 대체 처리

#### 해당 SHA에서 확인할 실제 코드

- `.stack-marquee-viewport`, 두 트랙의 width·부족한 부분, `stack-scroll` 변환을 확인합니다.
- 마우스 오버 일시 정지 선택자와 동작 줄이기 설정 `animation: none`을 확인합니다.
- Mask·넘침가 콘텐츠 clipping과 시각 fade를 어떻게 만드는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-3b9c1a636356-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 중복 트랙는 정적이었고 화면 크기 clipping, 이동 효과, 일시 정지·대체 처리 규칙이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 표시 영역을 콘텐츠 너비의 flex 컨테이너로 만들고 각 트랙에 34s linear infinite 애니메이션을 적용합니다. Keyframe은 한 트랙 폭과 부족한 부분만큼 왼쪽 이동하며 마우스를 올리면 일시 정지하고 동작 줄이기 설정에서 애니메이션을 제거합니다. |
| 데이터·상태·DOM·자원 소유 주체 | CSS가 시각 progression을 소유하고 DOM 중복·의미 구조는 컴포넌트가 소유합니다. |
| 실패·누락·대체 처리 | 동작 줄이기 설정이면 두 트랙이 정지해 중복된 시각 행이 나란히 남을 수 있지만 복제 트랙은 보조 기술 접근 트리에서 계속 숨겨집니다. |
| 보장하는 것과 보장하지 않는 것 | 일반 동작 효과의 연속 loop, 마우스 오버 일시 정지, 동작 줄이기 설정 포커스 지점을 제공합니다. 정확히 끊김 없는 픽셀 연결와 프레임 속도 성능은 실행 시점 테스트로 검증되지 않았습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 이 커밋으로 콘텐츠 메타데이터 → 아이콘 대체 처리 → 목록·흐르는 목록 DOM → 동작 효과 규칙 흐름이 완성됩니다. |
<!-- learner:commit-3b9c1a636356-record:end -->

#### 최소 코드 증거

<!-- learner:commit-3b9c1a636356-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-3b9c1a636356-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-3b9c1a636356-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-3b9c1a636356-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 알려진 브랜드 아이콘 | ebc245105c03 | Simple Icons 일부 맵 | 맵 조회 실패 렌더러 부재 | 후속 TechIcon이 대응된 경로 사용 |
| 맵 조회 실패 대체 처리 | 3d9b847d8094 | 이름이 있는 SVG 분기 + 일반적인 circle | 접근 가능한 이름은 호출자 문구 의존 | 시각 빈 자리 방지 |
| 순서가 지정된 기술 스택 칩 | 6aa8ee3b90b1 | 앞부분 선택 + `resolveTechStackItem` + 키 | 누락된 ID 동작은 해석 함수 범위 | ID 순서와 한도 보존 |
| 중복 트랙 의미 구조 | 48559efebf68 | 배포 중·복제 트랙, `aria-hidden` | 빈 입력 null 처리 없음 | 중복 시각 트랙을 보조 기술 접근 트리에서 한 번만 노출 |
| Motion 대체 처리 | 3b9c1a636356 | 마우스 오버 일시 정지·동작 줄이기 설정 없음 | 브라우저 시각 성능 미검증 | 연속 동작 효과를 사용자 설정에 맞게 중단 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| Partial 브랜드 맵 | 맵 조회 실패 시 아이콘 부재 | 3d9b847d8094 대체 처리 렌더러 | 전용 아이콘 스냅샷·테스트 없음 |
| 끊김 없는 반복를 위한 중복된 DOM | 화면 읽기 도구 중복 발표 위험 | 48559efebf68 복제 트랙 `aria-hidden` | 정적 DOM 의미 구조 확인; 보조 기술 실행 시점 테스트 없음 |
| 연속 흐르는 목록 | 동작 효과 sensitivity 및 상호작용 방해 | 3b9c1a636356 마우스 오버 일시 정지·동작 줄이기 설정 stop | CSS 정적 검토; 시각 회귀 테스트 없음 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 콘텐츠·타입 | 기술 스택 ID, 문구, 색상, 아이콘 식별자 | `TechStackItem` |
| TechIcon | Simple Icons vs 대체 처리 SVG 선택 | 장식용 아이콘 DOM |
| `StackList` | ID 해석, 선택적 한도, 칩 DOM | 정적 목록 |
| TechMarquee | 18-항목 cap, 중복된 트랙, `aria-hidden` | 연속 목록 구성 |
| CSS | 스크롤, mask, 마우스 오버 일시 정지, 동작 줄이기 설정 | 시각 동작 |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- 알려진 브랜드 아이콘은 Simple Icons 경로를, 나머지 지원 대상 아이콘은 이름이 있는·일반적인 대체 처리를 사용합니다.
- `StackList`는 ID 순서와 선택적 한도를 보존한 접근 가능한 문구 칩를 만듭니다.
- TechMarquee는 최대 18개를 배포 중·복제 트랙으로 복제하고 ghost를 `aria-hidden` 처리합니다.
- 흐르는 목록은 일반 동작 효과에서 순환하고 마우스를 올리면 일시 정지하며 동작 줄이기 설정에서 정지합니다.
- 누락된 ID 검증, 빈 흐르는 목록 생략, 실행 시점 시각·보조 기술 테스트는 이 개발 과정에 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. 호출자가 기술 스택 ID 또는 해석된 `TechStackItem` 배열을 전달합니다.
2. 정적 목록은 해석 함수로 기준 메타데이터를 얻고, 흐르는 목록은 앞 18개를 선택합니다.
3. TechIcon이 Simple Icons 맵을 조회하고 찾지 못하면 FallbackIcon을 선택합니다.
4. `StackList`는 한 목록을, TechMarquee는 배포 중·`aria-hidden` ghost 두 목록을 렌더링합니다.
5. CSS가 색상 변수과 흐르는 목록 이동 효과·일시 정지·동작 줄이기 설정을 적용합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] 이 개발 과정에는 S·A 커밋이 없으며 중요도 B 범위에서 저장소에 맞춘 깊이를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 05-technology-stack-icon-list-and-marquee.md =====

===== BEGIN FILE: 06-project-card-composition-and-interaction.md =====
# 개발 과정: 프로젝트 카드 조립과 상호작용

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

`AvailabilityBadge`와 `ProjectCard`가 화면 캡처, 기술 스택, 주요 내용, 동작 링크를 어떻게 조립하고, 대표 소비자 및 마우스 오버·동작 줄이기 설정 표현까지 확장되는지 복원합니다.

**경계:** 1단계에서 기존 `project-card-actions-and-evidence-components` 개발 과정의 링크 규칙 커밋을 01 개발 과정으로 이동했습니다. 이 개발 과정은 카드 조립과 상호작용만 소유하며 링크 노출 위치·이동 방식은 01 개발 과정의 선택자·렌더러를 소비합니다.

### 고정 불변 조건

- 배지 표시 여부와 표현 방식은 배포 상태 데이터에서 정하며, `showBadge=false`이면 DOM을 만들지 않습니다.
- `ProjectCard`는 근거 공용 컴포넌트를 조립하지만 링크 사용 가능 여부와 이동 방식을 다시 구현하지 않습니다.
- Case-study 주요 탐색은 현재 템플릿·디버그 상태를 보존합니다.
- 대표·기본 표현 방식은 레이아웃과 기술 스택 표시 개수만 바꾸며 데이터의 의미는 바꾸지 않습니다.
- 마우스 오버 overlay는 pointer 이벤트를 가로채지 않고 동작 줄이기 설정에서는 카드 변환·전환 효과가 제거됩니다.

## 2. 핵심 질문

- `AvailabilityBadge`가 showBadge, 상태 표현 단계, `isProjectLive`를 어떻게 조합하는가?
- `ProjectCard`가 직접 만드는 프로젝트 사례 링크와 `ProjectCardLinks` 동작 그룹의 책임 차이는 무엇인가?
- 대표 표현 방식이 화면 캡처 우선순위, 그리드, 기술 스택 표시 개수와 주요 내용 개수에 미치는 영향은 무엇입니까?
- Design 대표 섹션은 카드 몇 개를 어떤 표현 방식으로 사용합니까?
- CSS 가상 overlay가 상호작용을 방해하지 않도록 하는 규칙과 동작 줄이기 설정 대체 처리는 무엇인가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 이 개발 과정은 중요도 B 커밋만 포함하므로 각 커밋의 구체적인 역할, 구분 지점, 실패와 보장하지 않는 범위를 필요한 범위로 기록합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `53ca0860ef3a` | feat(project): 프로젝트 배포 상태 배지 추가 | B | RENDERER | 사용 가능 여부 배지 공용 컴포넌트 |
| 2 | `b72c52a22690` | feat(project): 프로젝트 카드 프리미티브 추가 | B | RENDERER | 카드 조립 소유 주체 |
| 3 | `07dd465dbe20` | feat(home): 디자인 대표 프로젝트 섹션 추가 | B | RENDERER | 대표 섹션 소비자 |
| 4 | `4000a8657a62` | style(project): 프로젝트 카드 상호작용 추가 | B | RENDERER | 마우스 오버·overlay와 동작 효과 대체 처리 |

## 5. 커밋별 학습 기록

### 1. `53ca0860ef3a` — feat(project): 프로젝트 배포 상태 배지 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 사용 가능 여부 배지 공용 컴포넌트

#### 해당 SHA에서 확인할 실제 코드

- 새 `availability-badge.tsx`의 초기 null, statusTone 조회, 기본값 표현 단계와 배포 중 표시점 브랜치를 확인합니다.
- `isProjectLive`가 배지 문구가 아니라 표시 여부만 제어하는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-53ca0860ef3a-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 프로젝트 카드·상세에서 배포 상태 문구와 사용 가능 여부 표현 단계를 일관되게 표현하는 공용 컴포넌트가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `AvailabilityBadge`는 `showBadge`가 `false`이면 `null`을 반환합니다. 그 외에는 상태에 맞는 클래스를 선택하고, `isProjectLive`가 참일 때만 강조 표시점을 추가한 뒤 콘텐츠 문구를 표시합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 배포 데이터가 상태·`showBadge`·문구를 제공하고, 배지 컴포넌트가 표현 방식과 표시점 DOM을 만듭니다. 실제 배포 중 여부는 선택자가 판단합니다. |
| 실패·누락·대체 처리 | 알 수 없는 상태는 원본만 제공되는 상태로 표시합니다. `showBadge`가 `false`이면 문구도 렌더링하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 배포 배지의 생략 규칙·표현 방식·배포 중 표시점을 공통으로 처리합니다. 입력 상태의 유효성이나 실제 서비스 상태는 검증하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | b72c52a22690이 `ProjectCard` 안에서 배지를 소비합니다. |
<!-- learner:commit-53ca0860ef3a-record:end -->

#### 최소 코드 증거

<!-- learner:commit-53ca0860ef3a-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-53ca0860ef3a-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-53ca0860ef3a-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-53ca0860ef3a-execution:end -->

### 2. `b72c52a22690` — feat(project): 프로젝트 카드 프리미티브 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 카드 조립 소유 주체

#### 해당 SHA에서 확인할 실제 코드

- 새 `project-card.tsx`의 `caseStudyHref`, 대표 브랜치와 하위 공용 컴포넌트 calls를 확인합니다.
- 화면 캡처·제목이 같은 내부 라우트를 가리키는지, 동작 그룹은 `ProjectCardLinks`에 위임되는지 확인합니다.
- 기술 스택 한도 6/4와 주요 내용 앞 두 개 선택, 우선순위 전달을 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b72c52a22690-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 배지, 화면 캡처, 기술 스택 목록과 동작 링크 공용 컴포넌트는 있었지만 프로젝트 근거를 한 카드 규칙으로 조립하는 컴포넌트가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `ProjectCard`가 템플릿·디버그를 고려하는 프로젝트 사례 href를 만들고 화면 캡처·제목 링크, ContentHint, 분류, `AvailabilityBadge`, 요약, `StackList`, 앞의 주요 내용 두 개, `ProjectCardLinks`를 조립합니다. 대표 표현 방식은 2열 레이아웃과 기술 스택 한도 6을 사용하고 기본값은 한도 4입니다. |
| 데이터·상태·DOM·자원 소유 주체 | 카드가 조립·레이아웃 표현 방식과 주요 상세 라우트를 소유합니다. 배지, 화면 캡처, 기술 스택, 동작 선택·이동 방식은 각 하위 소유 주체에 위임됩니다. |
| 실패·누락·대체 처리 | `ProjectCard` 자체는 누락된 프로젝트 필드를 검사하지 않습니다. Action 링크가 비어도 주요 화면 캡처·제목 프로젝트 사례 탐색은 존재합니다. |
| 보장하는 것과 보장하지 않는 것 | 일관된 프로젝트 근거 카드를 제공합니다. 콘텐츠 유효성, 상세 라우트 존재 여부, 링크 노출 위치 규칙을 재정의하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 07dd465dbe20이 실제 Design 홈 섹션에서 대표·기본값 표현 방식을 소비하고 4000a8657a62가 상호작용 CSS를 추가합니다. |
<!-- learner:commit-b72c52a22690-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b72c52a22690-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-b72c52a22690-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b72c52a22690-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-b72c52a22690-execution:end -->

### 3. `07dd465dbe20` — feat(home): 디자인 대표 프로젝트 섹션 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 대표 섹션 소비자

#### 해당 SHA에서 확인할 실제 코드

- `FeaturedProjectsSection`의 섹션 검사 단계와 프로젝트 slices를 확인합니다.
- 첫 프로젝트는 대표 표현 방식, 다음 두 프로젝트는 기본값인지와 모든 카드에 우선순위가 전달되는지 확인합니다.
- 등장 효과 래퍼와 SectionHeading이 카드 공용 컴포넌트의 책임과 섞이지 않는지 구분합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-07dd465dbe20-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | `ProjectCard`는 있었지만 Design 홈 화면 구성 설정이 대표 섹션을 켤 때 실제 카드 배치 순서를 만드는 소비자가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 섹션 키가 활성화되어 있으면 첫 번째 프로젝트를 대표 표현으로, 다음 두 프로젝트를 2열 기본값 카드로 렌더링합니다. 세 카드 모두 `priority`를 전달하고 등장 효과로 감쌉니다. |
| 데이터·상태·DOM·자원 소유 주체 | 화면 구성 설정이 섹션 존재 여부를, 홈 섹션이 순위·앞부분 선택·레이아웃을, 카드가 내부 근거 조립을 소유합니다. |
| 실패·누락·대체 처리 | 대표 프로젝트가 0~2개여도 맵은 가능한 항목만 렌더링합니다. 빈일 때 섹션 제목·래퍼는 남을 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | 상위 three 대표 프로젝트의 카드 조립을 실제 라우트에 연결합니다. 모든 프로젝트 이미지를 지연 로딩로 유지하거나 빈 섹션을 숨기는 것은 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 4000a8657a62가 카드 클래스를 이용한 마우스 오버·overlay 동작을 추가합니다. |
<!-- learner:commit-07dd465dbe20-record:end -->

#### 최소 코드 증거

<!-- learner:commit-07dd465dbe20-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-07dd465dbe20-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-07dd465dbe20-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-07dd465dbe20-execution:end -->

### 4. `4000a8657a62` — style(project): 프로젝트 카드 상호작용 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 마우스 오버·overlay와 동작 효과 대체 처리

#### 해당 SHA에서 확인할 실제 코드

- `.motion-card`, `.project-card::before`, `.project-screenshot::after`와 마우스 오버 선택자를 확인합니다.
- 가상 요소의 `pointer-events: none`, children z-index과 동작 줄이기 설정 변환·전환 효과 비활성화를 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-4000a8657a62-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 카드 마크업은 있었지만 lift, ambient overlay, 화면 캡처 sheen과 동작 줄이기 설정별 상호작용 대체 처리가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 동작 효과 카드의 마우스 오버에 `translateY`를 적용하고 프로젝트 카드와 화면 캡처 가상 요소에 투명도 전환 효과를 추가했습니다. 오버레이는 `pointer-events: none`이고 실제 자식 요소는 `z-index: 1`입니다. 동작 줄이기 설정에서는 카드 변형과 전환 효과를 제거합니다. |
| 데이터·상태·DOM·자원 소유 주체 | CSS가 일시적인 상호작용 표현을 소유하고 의미상 링크·buttons는 기존 DOM이 소유합니다. |
| 실패·누락·대체 처리 | Pointer events를 가상 overlay가 가로채지 않습니다. 동작 줄이기 설정은 변환·전환 효과를 끄지만 색상 마우스 오버 클래스 등 동작 효과가 아닌 효과 피드백은 남을 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | 카드 상호작용을 강화하면서 클릭 대상과 동작 효과 설정을 보존합니다. 키보드 관련 시각 회귀나 브라우저 합성 성능은 검증하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 이 개발 과정의 최종 카드 경로는 공용 하위 공용 컴포넌트 + 표시 영역 소비자 + 클릭을 막지 않는 CSS overlay입니다. |
<!-- learner:commit-4000a8657a62-record:end -->

#### 최소 코드 증거

<!-- learner:commit-4000a8657a62-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-4000a8657a62-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-4000a8657a62-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-4000a8657a62-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 배지 누락·표현 단계 | 53ca0860ef3a | showBadge null, statusTone 기본값, 배포 중 표시점 | 실제 service health 미검증 | 배포 상태 화면 구성을 일관되게 표현 |
| 카드 조립 | b72c52a22690 | `ProjectCard` 하위 호출 참조 관계 | 콘텐츠 유효성은 외부 | 근거 공용 컴포넌트를 한 컴포넌트로 조립 |
| 대표 프로젝트 사용 | 07dd465dbe20 | 첫 항목·후속 항목 분할과 표현 방식 | 빈 섹션 생략 없음 | Design 홈 상위 3개 표시 순서 |
| 상호작용을 막지 않는 상호작용 | 4000a8657a62 | pointer-events 없음, z-index, 동작 줄이기 설정 | 시각·브라우저 테스트 없음 | 마우스 오버 장식이 링크를 가리지 않음 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| 카드가 링크 규칙을 직접 소유할 위험 | 01 개발 과정과 중복·불일치 | b72c52a22690은 `ProjectCardLinks`를 소비만 함 | 09cec616f314이 하위 링크 규칙을 검증 |
| 장식 오버레이가 클릭 대상을 가릴 위험 | 마우스 오버 시 링크를 누를 수 없음 | `4000a8657a62`에서 `pointer-events: none`과 낮은 `z-index`를 적용 | CSS는 정적으로 확인했으며 브라우저 클릭 영역 검사는 실행하지 않음 |
| 마우스 오버 시 떠오르는 효과가 동작 줄이기 설정을 무시할 위험 | 동작 효과 민감도 문제 | `4000a8657a62`에서 동작 줄이기 환경의 카드 효과를 비활성화 | 전역 접근성 규칙은 03 개발 과정에서 추가로 보강 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 콘텐츠·선택자 | 배포 상태, 대표 순서와 링크 동작 결정 | 카드는 해당 의미를 다시 계산하지 않음 |
| `AvailabilityBadge` | 배지 DOM·표현 단계·배포 중 표시점 | 선택적 상태 표시 |
| `ProjectCard` | 근거 조립, 주요 라우트, 표현 방식 레이아웃 | 공용 하위 공용 컴포넌트 호출 |
| 홈 섹션 | 상위 프로젝트 분할과 카드 표현 방식 선택 | 표시 영역 수준의 조립 |
| CSS | 마우스 오버 lift/overlays·동작 줄이기 설정 | 의미 상호작용을 가리지 않음 |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- `ProjectCard`는 배지, 화면 캡처, 기술 스택, 주요 내용와 동작 링크를 공통 구조로 조립합니다.
- 화면 캡처와 제목은 템플릿·디버그를 고려하는 프로젝트 사례 라우트를 직접 제공합니다.
- Action 링크 표시 여부·이동 방식은 01 개발 과정의 `ProjectCardLinks`/`ContentLinkView`에 위임됩니다.
- Design 대표 섹션은 첫 카드를 대표 프로젝트로, 다음 두 카드를 기본값으로 사용합니다.
- 마우스 오버 overlays는 pointer 이벤트를 받지 않고 동작 줄이기 설정에서 lift·전환 효과를 제거합니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. 홈 소비자가 대표 프로젝트를 순서대로 선택하고 카드 표현 방식·우선순위를 정합니다.
2. `ProjectCard`가 프로젝트 데이터로 주요 프로젝트 사례 URL을 만듭니다.
3. 카드가 `AvailabilityBadge`, `ProjectScreenshot`, `StackList`, 주요 내용, `ProjectCardLinks`를 조립합니다.
4. 각 하위 공용 컴포넌트가 자기 규칙·DOM을 처리합니다.
5. CSS가 마우스 오버 장식와 lift를 적용하되 포인터 이벤트를 통과시키고 동작 줄이기 설정에서 이동 효과를 제거합니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] 이 개발 과정에는 S·A 커밋이 없으며 중요도 B 범위에서 저장소에 맞춘 깊이를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 06-project-card-composition-and-interaction.md =====

===== BEGIN FILE: 07-journey-timeline-primitives-and-responsive-layout.md =====
# 개발 과정: 여정 타임라인 공용 컴포넌트와 반응형 레이아웃

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

여정 항목의 기간 표시 형식과 선택적 프로젝트 사례 링크, 간결형·쌍형 표현, 홀수 행 처리, 데스크톱 중앙선과 모바일 단일 열 축소를 실제 이력으로 복원합니다.

**경계:** 이 개발 과정은 여정 화면 구성 공용 컴포넌트와 반응형 CSS를 다룹니다. 여정 JSON 스키마·검증, 라우트 섹션 복사, 등장 효과의 최종 서버 렌더링 우선 리팩터링은 각각 콘텐츠 시스템과 03 개발 과정이 소유합니다.

### 고정 불변 조건

- 여정 원본 순서는 유지되고 첫 항목만 쌍으로 배치된 타임라인의 시작 카드로 분리됩니다.
- 나머지 항목은 두 개씩 묶이며 마지막 홀수 항목은 단독 행으로 손실 없이 렌더링됩니다.
- projectId가 있는 항목만 템플릿·디버그를 고려하는 프로젝트 사례 링크를 가집니다.
- 모바일은 같은 의미상 순서가 지정된 목록을 단일 열로 재배치하며 항목 순서를 바꾸지 않습니다.
- 애니메이션 설정값은 래퍼 선택만 바꾸고 여정 데이터 의미 구조는 바꾸지 않습니다.

## 2. 핵심 질문

- `formatYearMonth`가 ISO 날짜를 검증하지 않고 어떤 부분 문자열 변환만 수행하는가?
- 첫 항목과 `projectItems` 분리, `chunkPairs`, `is-single` 클래스가 빈·하나·짝수·홀수 개 입력을 어떻게 처리하는가?
- 간결형과 쌍으로 배치된 표현 방식에서 등장 효과 래퍼 위치와 지연 값 계산은 어떻게 다른가?
- 데스크톱 중앙선의 3열 그리드가 모바일에서 같은 DOM을 어떻게 단일 열로 재배치하는가?
- 03 개발 과정의 서버 렌더링 우선 등장 효과 전환이 `animated` 설정값의 최종 의미를 어떻게 약화시키는가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 이 개발 과정은 중요도 B 커밋만 포함하므로 각 커밋의 구체적인 역할, 구분 지점, 실패와 보장하지 않는 범위를 필요한 범위로 기록합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `f60edf14dae0` | feat(journey): 여정 날짜와 카드 프리미티브 추가 | B | RENDERER | 진입점·카드 기반 도입 |
| 2 | `3c0a1154ba08` | feat(journey): 중앙선 여정 목록 추가 | B | RENDERER | 쌍으로 배치된 타임라인 조립 |
| 3 | `b3182e35aed0` | feat(journey): 여정 목록 변형 연결 | B | RENDERER | 공개 표현 방식 분기 함수 |
| 4 | `377fa128f82b` | style(journey): 여정 타임라인 시각 계층 추가 | B | RENDERER | 간결형 안내선·노드 표현 |
| 5 | `7995a3cf5435` | style(journey): 데스크톱 중앙선 여정 구성 | B | RENDERER | 데스크톱 중앙선 배치 |
| 6 | `6394188022fd` | style(journey): 모바일 중앙선 여정 구성 | B | RENDERER | 모바일 단일 열 축소 |

## 5. 커밋별 학습 기록

### 1. `f60edf14dae0` — feat(journey): 여정 날짜와 카드 프리미티브 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 진입점·카드 기반 도입

#### 해당 SHA에서 확인할 실제 코드

- `journey-list.tsx`의 `formatYearMonth`, `getJourneyPeriod`, `JourneyEntry`, `JourneyCard`, `chunkPairs`를 확인합니다.
- `endDate` 누락·동일 여부, `projectId` 누락, 쌍 잘라내기의 구분 지점을 각각 추적합니다.
- `getTemplateHref`에 `homeTemplate`/`contentDebug`가 전달되는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-f60edf14dae0-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 여정 항목을 공통 날짜·본문·카드 구조로 표현하거나 두 개씩 묶는 공용 컴포넌트가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 날짜 문자열을 YYYY.MM 형식으로 바꾸고, 같거나 없는 `endDate`는 단일 기간, 다른 `endDate`는 범위로 만듭니다. `JourneyEntry`는 분류·제목·본문과 선택적 프로젝트 링크를 렌더링하고 `JourneyCard`가 쌍형 카드 래퍼를 제공합니다. `chunkPairs`는 목록을 두 항목씩 묶습니다. |
| 데이터·상태·DOM·자원 소유 주체 | 여정 데이터가 날짜·`endDate`·`projectId`를 소유하고 도우미 함수가 표시 문자열과 항목 쌍 배열을 파생합니다. `JourneyEntry`가 의미상 콘텐츠와 선택적 링크를 소유합니다. |
| 실패·누락·대체 처리 | `projectId`가 없으면 링크는 null입니다. 날짜 형식은 `slice(0, 7)` 전제이며 유효하지 않은·짧은 입력을 거부하지 않습니다. 빈 배열을 묶으면 빈 배열이 됩니다. |
| 보장하는 것과 보장하지 않는 것 | 기간, 선택적 링크, 항목 쌍 분리의 공통 기반을 제공합니다. 전체 날짜 검증이나 `projectId` 참조 무결성은 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 3c0a1154ba08이 첫 항목 + 쌍으로 배치된 행 조립을 추가합니다. |
<!-- learner:commit-f60edf14dae0-record:end -->

#### 최소 코드 증거

<!-- learner:commit-f60edf14dae0-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-f60edf14dae0-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-f60edf14dae0-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-f60edf14dae0-execution:end -->

### 2. `3c0a1154ba08` — feat(journey): 중앙선 여정 목록 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 쌍으로 배치된 타임라인 조립

#### 해당 SHA에서 확인할 실제 코드

- `PairedJourneyList`의 `[startItem, ...projectItems]`, `chunkPairs`, 첫 항목 브랜치를 확인합니다.
- 쌍 길이 1 클래스, 중간 노드, 선택적 두 번째 카드와 키 생성 방식을 확인합니다.
- `animated`가 true/false일 때에서 요소 타입과 delay가 어떻게 달라지는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-3c0a1154ba08-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Entry·카드 도우미 함수는 있었지만 첫 주요 시점을 중앙의 시작 카드로 두고 나머지를 중앙선을 기준으로 쌍을 이루는 행으로 만드는 의미상 목록이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 첫 항목을 시작 카드로 분리하고 나머지를 두 개씩 묶습니다. 한 항목만 남으면 `is-single`을 적용하고, 두 번째 항목이 있으면 중앙 노드 뒤에 두 번째 카드를 둡니다. 애니메이션 모드에서는 시작 카드와 각 행을 `Reveal as="li"`로, 그렇지 않으면 일반 `li`로 렌더링합니다. |
| 데이터·상태·DOM·자원 소유 주체 | `PairedJourneyList`가 항목 그룹화, 행 식별 정보와 애니메이션 delay를 소유합니다. `JourneyCard`는 항목 표현을 재사용합니다. |
| 실패·누락·대체 처리 | 빈 입력이면 시작 카드와 행 모두 없어 빈 `<ol>`이 남습니다. 홀수 마지막 항목은 `is-single` 행으로 손실 없이 남고 노드는 DOM에 있지만 CSS가 숨길 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | 첫 주요 시점과 쌍으로 배치된 행의 원본 순서 및 마지막 홀수 항목 보존을 보장합니다. 데스크톱·모바일 배치는 아직 없습니다. |
| 다음 커밋 또는 관련 테스트 연결 | b3182e35aed0이 공개 표현 방식 분기 함수를, 7995a3cf5435·6394188022fd가 배치를 추가합니다. |
<!-- learner:commit-3c0a1154ba08-record:end -->

#### 최소 코드 증거

<!-- learner:commit-3c0a1154ba08-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-3c0a1154ba08-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-3c0a1154ba08-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-3c0a1154ba08-execution:end -->

### 3. `b3182e35aed0` — feat(journey): 여정 목록 변형 연결

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 공개 표현 방식 분기 함수

#### 해당 SHA에서 확인할 실제 코드

- Exported `JourneyList`의 defaults와 쌍으로 배치된 중앙선 즉시 반환을 확인합니다.
- 간결형 행의 animated·일반 분기와 outer 등장 효과 래퍼 차이를 확인합니다.
- Index-based delay와 안정적인 키가 원본 순서에 어떻게 연결되는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-b3182e35aed0-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 쌍형 렌더러는 내부 함수였고 기존 간결한 목록과 하나의 공개 API에서 선택할 표현 방식 규칙이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `JourneyList`가 기본값 간결한, 선택적 쌍으로 배치된 중앙선, `animated` 설정값을 받습니다. 쌍형이면 전용 컴포넌트에 위임하고, 간결한이면 각 행을 등장 효과·일반 `li`로 만든 뒤 animated일 때 목록 전체도 등장 효과 래퍼로 감쌉니다. |
| 데이터·상태·DOM·자원 소유 주체 | 공개 컴포넌트가 표현 방식 분기를 소유하고 각 표현 방식이 DOM 배치를 소유합니다. Data 순서는 호출자 배열과 맵 순서가 소유합니다. |
| 실패·누락·대체 처리 | 알 수 없는 표현 방식은 TypeScript 유니언 밖이며 실행 시점에서는 간결한 브랜치로 떨어집니다. 빈 간결한 입력도 빈 순서가 지정된 목록 래퍼를 반환합니다. |
| 보장하는 것과 보장하지 않는 것 | 한 API로 간결형·쌍형 표현과 애니메이션 래퍼를 선택합니다. 표현 방식별 반응형 스타일은 후속 CSS가 필요합니다. |
| 다음 커밋 또는 관련 테스트 연결 | 377fa128f82b가 간결한 타임라인 시각 배치 순서를, 7995a3cf5435/6394188022fd가 쌍으로 배치된 반응형 레이아웃을 추가합니다. |
<!-- learner:commit-b3182e35aed0-record:end -->

#### 최소 코드 증거

<!-- learner:commit-b3182e35aed0-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-b3182e35aed0-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-b3182e35aed0-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-b3182e35aed0-execution:end -->

### 4. `377fa128f82b` — style(journey): 여정 타임라인 시각 계층 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 간결형 안내선·노드 표현

#### 해당 SHA에서 확인할 실제 코드

- `.timeline-list`의 여백·x축 변수, 안내선 가상 요소와 `.experience-row::before`를 확인합니다.
- `is-visible` 클래스가 guide scaleY와 노드 opacity/scale를 어떻게 바꾸는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-377fa128f82b-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 간결형 순서가 지정된 목록에는 의미상 행만 있고 vertical guide와 행별 노드 시각 배치 순서가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 타임라인 래퍼에 세로 그라데이션 안내선과 행 가상 노드를 추가하고 `is-visible` 상태에 맞춰 크기와 투명도 전환 효과를 적용했습니다. |
| 데이터·상태·DOM·자원 소유 주체 | CSS가 시각 안내선·노드를 소유하고 순서가 지정된 목록·진입점 의미 구조는 컴포넌트가 소유합니다. |
| 실패·누락·대체 처리 | Guide·노드는 가상 요소라 접근성 파일 트리에 추가되지 않습니다. 등장 효과 클래스가 표시되는 전환을 제공한다는 당시 전제에 의존합니다. |
| 보장하는 것과 보장하지 않는 것 | 간결형 타임라인의 시각적 연결을 제공합니다. 서버 렌더링 우선 등장 효과 전환 뒤에는 항목이 화면에 들어오는 화면 크기 시점을 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 29bb40579cb2/af9191fc15ad의 동작 줄이기 설정 규칙과 b8164cfdddbd의 서버 렌더링 우선 리팩터링이 개발 과정 간으로 이 클래스 동작을 바꿉니다. |
<!-- learner:commit-377fa128f82b-record:end -->

#### 최소 코드 증거

<!-- learner:commit-377fa128f82b-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-377fa128f82b-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-377fa128f82b-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-377fa128f82b-execution:end -->

### 5. `7995a3cf5435` — style(journey): 데스크톱 중앙선 여정 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 데스크톱 중앙선 배치

#### 해당 SHA에서 확인할 실제 코드

- `.paired-timeline::before`, 시작 식별 속성·카드, 3열 행과 중앙 노드를 확인합니다.
- `.is-single` 행이 1열 가운데 정렬한 카드로 바뀌고 노드를 숨기는 규칙을 확인합니다.
- Classic 템플릿의 카드 shadow 덮어쓰기를 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-7995a3cf5435-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 쌍형 DOM은 있었지만 중앙선, 첫 번째 주요 시점, 양쪽 카드 그리드와 마지막 홀수 항목 레이아웃이 시각적으로 정의되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 50% 위치의 중앙선, 가운데 정렬한 시작 카드와 식별 속성, `1fr 2.25rem 1fr` 행 그리드, 중앙 노드를 추가합니다. 단일 행은 최대 너비를 제한한 1열 가운데 정렬 카드가 되고 노드는 표시하지 않습니다. |
| 데이터·상태·DOM·자원 소유 주체 | CSS가 데스크톱 배치와 시각 중앙선을 소유합니다. DOM 그룹화·순서는 `PairedJourneyList`가 소유합니다. |
| 실패·누락·대체 처리 | 홀수 마지막 항목은 중앙선 양쪽을 억지로 채우지 않고 가운데 정렬한 single 카드가 됩니다. Narrow 화면 크기 대체 처리는 아직 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 데스크톱에서 첫 항목 + 쌍형 행 + 마지막 단독 항목 배치 순서를 보장합니다. Small-screen readability는 다음 커밋이 처리합니다. |
| 다음 커밋 또는 관련 테스트 연결 | 6394188022fd가 767px 이하에서 같은 DOM을 단일 열 타임라인으로 축소합니다. |
<!-- learner:commit-7995a3cf5435-record:end -->

#### 최소 코드 증거

<!-- learner:commit-7995a3cf5435-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-7995a3cf5435-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-7995a3cf5435-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-7995a3cf5435-execution:end -->

### 6. `6394188022fd` — style(journey): 모바일 중앙선 여정 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 모바일 단일 열 축소

#### 해당 SHA에서 확인할 실제 코드

- max-width 767px 미디어 쿼리에서 중앙선 x, 목록 padding, 시작·행 그리드를 확인합니다.
- 데스크톱 중앙 노드를 숨기고 각 카드 가상 노드를 만드는 선택자를 확인합니다.
- 카드 border/shadow/radius·문구 alignment가 모바일에서 어떻게 단순화되는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-6394188022fd-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 데스크톱 3열 쌍으로 배치된 레이아웃은 작은 화면 크기에서 폭이 부족하고 중앙선·식별 속성이 콘텐츠와 충돌할 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 중앙선을 왼쪽 0.45rem로 옮기고 목록에 여백을 둡니다. 시작·쌍·단일 행을 1열로 배치하고 중앙 노드를 숨긴 뒤 각 카드 왼쪽에 타임라인 노드를 표시합니다. 카드의 가로 테두리·그림자·모서리 둥글기를 제거하고 전체 너비에서 문구를 왼쪽 정렬합니다. |
| 데이터·상태·DOM·자원 소유 주체 | CSS 미디어 쿼리가 시각 재배치를 소유하며 DOM과 항목 순서는 바뀌지 않습니다. |
| 실패·누락·대체 처리 | 안전한 데이터 대체 처리를 추가하는 커밋은 아닙니다. 모바일에서도 빈 목록은 빈 래퍼이고 유효하지 않은 날짜는 그대로 형식 변환됩니다. |
| 보장하는 것과 보장하지 않는 것 | 같은 순서가 지정된 데이터와 링크를 모바일 단일 열 타임라인으로 읽을 수 있게 합니다. 실제 기기 시각 회귀와 터치 동작은 검증하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 이 커밋으로 형식화 함수 → 진입점·카드 → 표현 방식 → 데스크톱·모바일 화면 구성 흐름이 완성됩니다. |
<!-- learner:commit-6394188022fd-record:end -->

#### 최소 코드 증거

<!-- learner:commit-6394188022fd-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-6394188022fd-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-6394188022fd-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-6394188022fd-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 기간 표시 형식 | f60edf14dae0 | 문자열 잘라내기·치환과 endDate 비교 | 날짜 검증 없음 | 유효한 YYYY-MM 입력의 표시 기간 |
| 원본 순서와 홀수 마지막 항목 보존 | 3c0a1154ba08 | 첫 항목 분리 + `chunkPairs` + `is-single` | 빈 래퍼 null 처리 없음 | 모든 항목을 순서대로 렌더링 |
| 표현 방식 분기 | b3182e35aed0 | 쌍형 표현의 즉시 반환·간결한 대체 처리 | 알 수 없는 실행 시점 값은 간결형 | 한 공개 API로 두 레이아웃 선택 |
| 데스크톱 중앙선 | 7995a3cf5435 | 3열 행·마지막 단독 항목 | 작은 화면 부적합 | 넓은 화면 크기 배치 |
| 모바일 재배치 | `6394188022fd` | 왼쪽 안내선·1열·카드 노드 | 실제 기기 시각 테스트 없음 | DOM 순서를 바꾸지 않는 반응형 축소 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| 프로젝트 항목 수가 홀수인 경우 | 마지막 항목 손실 또는 빈 반대 열 위험 | `chunkPairs` + `is-single` + 가운데 정렬한 카드 | 전용 단위 테스트 없음; 정확한 브랜치 정적 검토 |
| 데스크톱 쌍형 그리드를 모바일에서 사용할 경우 | 좁은 카드와 중앙선 충돌 | 6394188022fd 1열 미디어 쿼리 | 시각 회귀 테스트 없음 |
| 등장 효과와 표시 여부의 결합 | JS·하이드레이션 의존 시 안내선·노드 표시 여부 위험 | 03 개발 과정의 b8164cfdddbd 서버에서 즉시 표시되는 리팩터링 | 해당 여정 커밋 자체에는 테스트 없음 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 여정 데이터 | 날짜·endDate·분류·제목·본문·projectId와 순서 | 콘텐츠 원본 |
| 도우미 함수 | 표시 기간과 쌍 그룹화 | `formatYearMonth`/`getJourneyPeriod`/`chunkPairs` |
| `JourneyList` | 표현 방식 분기, 애니메이션 래퍼 | 공개 공용 컴포넌트 |
| `PairedJourneyList` | 첫 항목, 쌍, 홀수 마지막 항목, 키·지연값 | 쌍형 DOM 소유 주체 |
| CSS | 간결형 안내선, 데스크톱 중앙선, 모바일 축소 | 반응형 표현 |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- `JourneyList`는 간결형과 쌍형 중앙선 두 표현 방식을 제공합니다.
- 쌍형 표현 방식은 첫 항목을 시작 카드로 두고 나머지를 두 개씩 묶으며 홀수 마지막 항목을 가운데 정렬한 단독 행으로 보존합니다.
- `projectId`가 있는 항목만 템플릿·디버그를 고려하는 프로젝트 사례 링크를 만듭니다.
- 데스크톱은 중앙선 양쪽 카드, 모바일은 같은 DOM의 왼쪽 안내선이 있는 단일 열을 사용합니다.
- 날짜 검증, 빈 목록 생략, 전용 단위·시각 테스트는 이 개발 과정에 없습니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. 호출자가 순서가 지정된 JourneyItem 배열과 표현 방식·animated 옵션를 전달합니다.
2. `JourneyList`가 쌍으로 배치된 표현 방식면 `PairedJourneyList`로, 아니면 간결한 맵으로 분기합니다.
3. Entry 도우미 함수가 기간·분류·제목·본문과 선택적 프로젝트 링크를 만듭니다.
4. 쌍형 렌더러는 첫 항목을 분리하고 남은 항목을 쌍으로 묶습니다.
5. CSS가 간결형 안내선 또는 데스크톱 중앙선을 적용하고 767px 이하에서 같은 DOM을 1열으로 바꿉니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] 이 개발 과정에는 S·A 커밋이 없으며 중요도 B 범위에서 저장소에 맞춘 깊이를 유지했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 07-journey-timeline-primitives-and-responsive-layout.md =====

===== BEGIN FILE: 08-design-switcher-disclosure-and-responsive-sheet.md =====
# 개발 과정: 디자인 선택기 펼침 동작과 반응형 선택 패널

> 프로젝트: 42 Archive Portfolio
>
> 브랜치: `web/portfolio`
>
> 분류: `03-shared-ui-interaction-and-responsive-primitives`

## 0. 분류 출처와 1단계 고정 범위

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md` 분류와 해당 커밋을 정확히 확인한 결과를 대조했습니다.
- 이 문서의 개발 과정 범위, 커밋 집합, 순서, 역할과 커밋별 확인 사항은 1단계 검토에서 확정했습니다.
- 2단계에서는 이 고정 문구와 커밋 메타데이터를 바꾸지 않고 학습자용 영역만 채웁니다.
- 다른 브랜치와 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표와 경계

브라우저 기본 `<details>/<summary>`를 기반으로 데스크톱 메뉴와 모바일 아래 선택 패널을 공유하고, 현재 디자인·개수, 템플릿 상태를 보존하는 링크, 명시적 닫기와 포커스 복원을 구현한 초기 변경 과정을 복원합니다.

**경계:** 1단계에서 분류 README가 약속했지만 기존 7개 개발 과정이 다루지 않던 반응형 펼침 요소 공용 컴포넌트 구현 변경 과정을 추가했습니다. 실제 공용 셸 연결은 분류 02의 `b9571c485013`이, 후속 하이드레이션 경쟁 상태 수정·테스트와 서버 컴포넌트 리팩터링은 분류 07이 소유합니다. 이 확정된 커밋 목록에는 공용 컴포넌트 구현 커밋만 두고 인접 분류의 통합·수정·근거는 관계로 명시합니다.

### 고정 불변 조건

- 펼침 메뉴 `open` 상태는 React 불리언이 아니라 브라우저 기본 `<details>` 요소가 소유합니다.
- 데스크톱 dropdown과 모바일 아래 선택 패널는 동일한 의미상 `<details>` 요소·탐색·목록 DOM을 CSS로 재배치합니다.
- Active 디자인은 `aria-current`와 현재 선택된 클래스를 가지며 링크는 현재 경로와 콘텐츠 디버그 상태를 보존합니다.
- 명시적 닫기 버튼은 `open` 속성을 제거한 뒤 요약에 포커스를 복원합니다.
- 초기 클라이언트 구현은 하이드레이션 전에 브라우저 기본 `open` 상태가 바뀌는 경쟁 조건을 완전히 해결하지 못합니다.

## 2. 핵심 질문

- 데스크톱 CSS가 요약 식별 속성·포커스와 절대 패널 stacking을 어떻게 정의하는가?
- 모바일 CSS가 배경, 안전 영역 여백, 고정 패널과 스크롤 범위 제한을 같은 `<details>` 요소 `open` 속성에 연결하는가?
- DesignSwitcher가 현재 선택된 ID를 찾지 못한 경우, 콘텐츠 복사 miss, 개수 문구를 어떻게 대체 처리·형식하는가?
- 닫기 버튼과 디자인 링크 클릭이 `open` 상태·포커스를 각각 어떻게 처리하는가?
- 분류 02의 b9571c485013이 `SiteHeader`에서 어떤 속성과 조건으로 이 공용 컴포넌트를 실제 라우트 셸에 연결하는가?
- c702b870d57a → b6c0238ab8b8 → a37cb8596733 → 1ac7813155c6의 후속 수정이 초기 가정을 어떻게 바꾸는가?

## 3. 완료 기준

- 각 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 구분해 실제 파일·심볼·호출 경로를 기록합니다.
- 이전 상태, 소유 주체, 상태 전환, 누락·실패 분기, 보장 범위와 보장하지 않는 범위를 커밋별로 분리합니다.
- 수정과 테스트는 실제로 수정·검증하는 실제 코드 경로에 연결합니다.
- 실행하지 않은 명령 결과를 만들지 않습니다.
- 중요도 S·A는 설계 결정, 소유 주체, 실패 처리와 후속 근거를 중요도 B보다 깊게 복원합니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 개발 과정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `cc13abb3b66f` | style(designs): 디자인 선택기 기본 메뉴 구성 | B | RENDERER | 데스크톱 펼침 요소 스타일 적용 |
| 2 | `28dfc5087474` | style(designs): 모바일 디자인 선택 sheet 구성 | B | RENDERER | 모바일 아래 선택 패널 스타일 적용 |
| 3 | `e43e8addd7f3` | feat(designs): 디자인 선택기 상태와 trigger 추가 | B | RENDERER | 클라이언트 `<details>` 요소 열기 버튼과 현재 선택 상태 |
| 4 | `c69ef85c98b2` | feat(designs): 디자인 선택 목록과 닫기 동작 추가 | A | ARCH, RENDERER | 탐색 목록과 명시적인 상태 전환 |

### 관련 커밋 — 이 고정된 맵 밖의 경계

아래 커밋은 같은 배포 환경 변경 과정의 통합·수정·근거이지만 인접 분류가 소유합니다. 분류 중복을 피하기 위해 이 개발 과정의 커밋 목록에는 넣지 않습니다.

- `b9571c485013` — feat(셸): 디자인 선택기를 공용 셸에 연결 (분류 02)
- `c702b870d57a` — 수정(ui): 하이드레이션 중 브라우저 기본 `<details>` 상태 보존 (분류 07)
- `b6c0238ab8b8` — 테스트(ui): `<details>` 요소 하이드레이션 경쟁 조건 검증 (분류 07)
- `a37cb8596733` — 리팩터링(ui): 디자인 선택기를 서버 마크업으로 전환 (분류 07)
- `1ac7813155c6` — 테스트(ui): 서버 선택기와 포커스 복원 검증 (분류 07)

## 5. 커밋별 학습 기록

### 1. `cc13abb3b66f` — style(designs): 디자인 선택기 기본 메뉴 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 데스크톱 펼침 요소 스타일 적용

#### 해당 SHA에서 확인할 실제 코드

- 새 `design-switcher.module.css`의 최상위 z-index, 요약 식별 속성 제거·포커스 표시 여부, 절대 패널과 목록·링크 classes를 확인합니다.
- `.sheetHeader`가 데스크톱에서 표시하지 않음이고 현재 선택된·마우스 오버·포커스 스타일이 공유되는지 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-cc13abb3b66f-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Design 선택자에 필요한 데스크톱 펼침 요소 스타일 적용과 포커스가 표시되는 treatment가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 상대 위치 최상위와 z-index, 사용자 정의 `<summary>` 열기 버튼, 숨긴 브라우저 기본 WebKit 표시 요소, 포커스 outline, 절대 right-aligned 패널, 목록·링크·색상 미리보기·복사·숫자 스타일을 추가했습니다. Sheet 헤더는 데스크톱에서 숨깁니다. |
| 데이터·상태·DOM·자원 소유 주체 | 브라우저 기본 `<details>` 요소가 `open` 상태를 소유할 예정이고 CSS가 데스크톱 배치·시각 상태를 소유합니다. |
| 실패·누락·대체 처리 | 패널 width는 화면 크기에서 2rem을 뺀 값으로 제한됩니다. CSS만 추가되어 실제 `<details>` 요소 DOM이나 닫기 동작은 아직 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 키보드 포커스가 보이는 요약 포커스와 데스크톱 메뉴 화면 구성 기반을 만듭니다. 의미 마크업과 상태 전환은 보장하지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 28dfc5087474가 같은 클래스 집합의 모바일 선택 패널 레이아웃을, e43e8addd7f3가 실제 `<details>` 요소·요약 DOM을 추가합니다. |
<!-- learner:commit-cc13abb3b66f-record:end -->

#### 최소 코드 증거

<!-- learner:commit-cc13abb3b66f-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-cc13abb3b66f-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-cc13abb3b66f-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-cc13abb3b66f-execution:end -->

### 2. `28dfc5087474` — style(designs): 모바일 디자인 선택 sheet 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 모바일 아래 선택 패널 스타일 적용

#### 해당 SHA에서 확인할 실제 코드

- max-width 640px의 `.root[open]::before`, 고정된 `.panel`, 안전 영역 padding과 z-index을 확인합니다.
- Sheet 헤더·버튼 크기, max-height·넘침·over스크롤 동작을 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-28dfc5087474-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 데스크톱 절대 dropdown CSS만 있어 좁은 화면 크기에서 패널 너비와 위치 닫기 affordance가 적절하지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 모바일에서는 최상위 요소를 정적으로 두고, `open` 상태일 때 화면을 덮는 배경을 표시합니다. 패널은 아래 고정된, 스크롤 가능한 max-height, 안전 영역을 고려한 padding과 overscroll contain을 사용합니다. Sheet 헤더와 2.75rem 닫기 버튼을 표시합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 브라우저 기본 `open` 속성이 배경 표시 여부를, 미디어 쿼리가 모바일 배치를 소유합니다. |
| 실패·누락·대체 처리 | CSS는 본문 스크롤 잠금이나 배경 클릭 닫기를 구현하지 않습니다. 넘침 스크롤은 패널 안에서만 처리합니다. |
| 보장하는 것과 보장하지 않는 것 | 같은 의미상 패널을 모바일 선택 패널로 표현할 레이아웃을 제공합니다. 실제 버튼·DOM과 포커스 동작은 후속 컴포넌트가 필요합니다. |
| 다음 커밋 또는 관련 테스트 연결 | e43e8addd7f3/c69ef85c98b2가 열기 버튼, 탐색 목록과 닫기 버튼을 연결합니다. |
<!-- learner:commit-28dfc5087474-record:end -->

#### 최소 코드 증거

<!-- learner:commit-28dfc5087474-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-28dfc5087474-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-28dfc5087474-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-28dfc5087474-execution:end -->

### 3. `e43e8addd7f3` — feat(designs): 디자인 선택기 상태와 trigger 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 클라이언트 `<details>` 요소 열기 버튼과 현재 선택 상태

#### 해당 SHA에서 확인할 실제 코드

- 새 `design-switcher.tsx`의 클라이언트 지시문, 참조, SITE_DESIGNS/templateCopy/`activeIndex` 대체 처리를 확인합니다.
- Count 템플릿의 padStart와 요약 ARIA 문구·콘텐츠를 확인합니다.
- Import된 Link/getTemplateHref가 이 SHA에서 아직 사용되지 않는다는 staged 구현 상태를 확인합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-e43e8addd7f3-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | CSS는 준비됐지만 브라우저 기본 `<details>`/요약 열기 버튼과 현재 디자인 문구·개수 계산이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 클라이언트 DesignSwitcher를 추가해 `<details>` 요소·요약을 렌더링하고 `<details>` 요소·요약 참조를 만듭니다. Template 복사 맵을 만들고 현재 선택된 ID를 SITE_DESIGNS에서 찾되 찾지 못하면 첫 번째 디자인으로 대체 처리합니다. Count 템플릿의 목록·전체을 2자리로 치환합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 브라우저 기본 `<details>` 요소가 `open` 속성을 소유하고 컴포넌트가 현재 선택된 파생값 계산·개수·ARIA 문구를 소유합니다. 참조는 후속 닫기 전환 효과를 위해 준비됩니다. |
| 실패·누락·대체 처리 | Active ID를 찾지 못한 경우 시 현재 선택된 객체는 첫 번째 디자인으로 대체 처리하지만 `activeIndex`는 -1이므로 개수의 목록이 `00`이 될 수 있습니다. 누락된 복사는 디자인 ID 문구로 대체 처리합니다. |
| 보장하는 것과 보장하지 않는 것 | 브라우저 기본 열기 버튼과 현재 선택된 복사를 제공합니다. List, 탐색 링크, 닫기 동작은 아직 렌더링하지 않으며 하이드레이션 경쟁 상태도 다루지 않습니다. |
| 다음 커밋 또는 관련 테스트 연결 | c69ef85c98b2가 참조를 사용한 닫기·포커스 및 전체 디자인 목록을 추가합니다. |
<!-- learner:commit-e43e8addd7f3-record:end -->

#### 최소 코드 증거

<!-- learner:commit-e43e8addd7f3-excerpt:start -->
별도 코드 발췌는 싣지 않았습니다. 위 기록에서 해당 SHA의 변경 경로와 심볼을 특정했으며, 직선적인 마크업·CSS 추가를 반복 인용해 원본 코드 나열로 만드는 것을 피했습니다.
<!-- learner:commit-e43e8addd7f3-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-e43e8addd7f3-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-e43e8addd7f3-execution:end -->

### 4. `c69ef85c98b2` — feat(designs): 디자인 선택 목록과 닫기 동작 추가

- **중요도:** A
- **태그:** ARCH, RENDERER
- **개발 과정에서의 역할:** 탐색 목록과 명시적인 상태 전환

#### 해당 SHA에서 확인할 실제 코드

- `nav`/`ul`/SITE_DESIGNS 맵의 현재 선택된 `aria-current`, 색상 미리보기, 복사 대체 문구와 숫자를 확인합니다.
- `getTemplateHref(currentPath, design.id, { contentDebug })` 호출 경로를 확인합니다.
- 닫기 버튼의 `removeAttribute("open")` → 요약 포커스 순서와 링크 클릭 닫기 브랜치를 비교합니다.
- 브라우저가 관리하는 상태 소유 주체와 DOM 참조 사이의 소유 주체, 누락된 참조 선택적 연결, 탐색 발생 전 닫기의 보장하지 않는 범위를 기록합니다.
- 먼저 부모와의 변경 차이와 해당 SHA의 변경 후 파일 트리를 구분합니다.
- 최종 HEAD의 도우미 함수·파일 배치·동작을 이 SHA의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- learner:commit-c69ef85c98b2-record:start -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 열기 버튼은 열릴 수 있었지만 선택 목록, 현재 선택된 의미 구조, 템플릿 상태를 보존하는 href, 명시적인 모바일 닫기·포커스 복원이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 패널 안에 이름이 지정된 탐색 영역과 목록을 추가하고 SITE_DESIGNS 순서대로 링크를 만듭니다. 현재 항목은 `aria-current="page"`와 현재 선택된 클래스를 사용합니다. 링크는 `currentPath`/view/contentDebug를 보존합니다. 닫기 버튼은 `<details>` 요소 `open` 속성을 제거하고 요약에 포커스하며, 링크 클릭도 `open` 속성을 제거합니다. |
| 데이터·상태·DOM·자원 소유 주체 | 브라우저 기본 `<details>` 요소가 펼침 상태를, 컴포넌트 참조·이벤트 처리기가 명시적 상태 전환을, getTemplateHref가 목적지 쿼리를 소유합니다. Template 복사 맵은 문구·설명 대체 처리를 제공합니다. |
| 실패·누락·대체 처리 | 참조가 없으면 선택적 연결으로 닫기·포커스가 아무 작업도 하지 않음입니다. Link 클릭 닫기는 탐색 성공을 기다리지 않으며 포커스를 복원하지 않습니다. Backdrop 클릭·이스케이프 처리 사용자 정의 처리기·본문 lock은 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 데스크톱·모바일이 같은 의미상 목록을 공유하고 명시적 닫기 포커스 규칙을 갖습니다. 그러나 전체 컴포넌트가 클라이언트 실행 범위이고 하이드레이션 전에 브라우저 기본 `open` 상태가 바뀌면 서버·클라이언트 속성 불일치 가능성이 남습니다. |
| 다음 커밋 또는 관련 테스트 연결 | 분류 02의 b9571c485013이 이 컴포넌트를 `SiteHeader`/`PageShell`의 조건부 소비자로 연결합니다. 09cec616f314이 기본 닫기·링크 규칙을 컴포넌트 테스트로 확인하고, 분류 07의 c702/b6c/a37/1ac가 하이드레이션 경쟁 상태를 재현한 뒤 서버 마크업 + 작은 클라이언트 닫기 컴포넌트로 보정된 불변 조건을 만듭니다. |
<!-- learner:commit-c69ef85c98b2-record:end -->

#### 최소 코드 증거

<!-- learner:commit-c69ef85c98b2-excerpt:start -->
- **커밋:** `c69ef85c98b2`
- **경로:** `src/components/portfolio/design-switcher.tsx`
- **위치:** `close button handler`

```tsx
onClick={() => {
  detailsRef.current?.removeAttribute("open");
  summaryRef.current?.focus();
}}
```

이 발췌는 해당 SHA의 결정·상태·소유 주체를 보여 주는 최소 부분입니다. 후속 커밋의 코드는 섞지 않았습니다.
<!-- learner:commit-c69ef85c98b2-excerpt:end -->

#### 실행·테스트 증거

<!-- learner:commit-c69ef85c98b2-execution:start -->
- **정적 검토:** `web/portfolio` 전용 커밋 분류와 해당 SHA의 변경 내용과 변경 파일을 대조했습니다.
- **실행:** 하지 않았습니다. 작업 환경에서 `github.com` DNS 해석이 실패해 실행 가능한 저장소 사본을 만들 수 없었고, GitHub 연결 도구는 원본과 변경 내용 정적 검토만 제공했습니다. 따라서 명령, 통과·실패, 시점, 브라우저 결과를 주장하지 않습니다.
<!-- learner:commit-c69ef85c98b2-execution:end -->

## 6. 불변 조건 기록

<!-- learner:thread-ledger:start -->
| 불변 조건 | 도입·변경 커밋 | 실제 코드·테스트 근거 | 부족함이 드러난 지점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 브라우저 기본 펼침 상태 | e43e8addd7f3 | `<details>` 요소·요약 DOM | 클라이언트 하이드레이션 경쟁 상태 미해결 | 초기 개발 과정에서는 브라우저 기본 `open` 속성이 상태 소유 주체 |
| 같은 DOM을 사용하는 반응형 화면 구성 | `cc13abb3b66f → 28dfc5087474` | 데스크톱 절대 위치 패널과 모바일 고정 선택 패널 | 본문 스크롤 잠금과 배경 클릭 닫기는 없음 | 640px 기준에서 CSS만으로 배치 전환 |
| 템플릿 상태를 보존하는 선택 | c69ef85c98b2 | getTemplateHref `currentPath`/디자인·디버그 | 목적지 응답 미검증 | 선택 URL과 현재 선택된 의미 구조 제공 |
| 명시적 닫기·포커스 | c69ef85c98b2 | remove open → 요약.포커스 | 링크 클릭은 포커스 복원 안 함 | 닫기 버튼 경로의 포커스 규칙 |
| 공용 셸 통합 | b9571c485013 (외부 관계) | `SiteHeader`의 조건부 `DesignSwitcher` 호출과 `PageShell` 속성 | `templateSwitcher`가 없으면 선택자 자체가 렌더링되지 않음 | 분류 02가 실제 라우트 넓은 소비자 연결을 소유 |
| Corrected 서버 렌더링 우선 소유 주체 | a37cb8596733 (외부 관계) | 서버 DesignSwitcher + 클라이언트 DesignSwitcherClose | 고정된 맵 밖 분류 07 | 후속 설계에서 마크업은 서버, 닫기 처리기만 클라이언트 |
<!-- learner:thread-ledger:end -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:thread-relations:start -->
| 실패·위험 | 실제 영향·근본 원인 | 수정·결정 | 회귀 근거 또는 공백 |
| --- | --- | --- | --- |
| 기초 컴포넌트는 존재하지만 셸 소비자 없음 | 공용 라우트 셸에서 아직 도달할 수 없음 | b9571c485013이 `SiteHeader`에서 `templateSwitcher` 존재 시 `DesignSwitcher`를 호출 | 분류 02 정확한 변경 내용으로 정적 확인; 실행 시점 통합 테스트는 실행하지 않음 |
| 클라이언트에서 렌더링된 브라우저 기본 `<details>` | 하이드레이션 전 사용자가 `open` 속성을 바꾸면 속성 불일치 위험 | c702b870d57a suppress 경고 후 a37cb8596733 서버 마크업 전환 | b6c0238ab8b8 서버 렌더링 → 브라우저 기본 `open` 상태 변경 → 하이드레이션 결정적인 테스트, 1ac7813155c6 원본 입력 검증 지점 테스트 |
| 닫기 처리기가 전체 선택기 참조를 요구 | 전체 목록이 클라이언트 실행 범위에 묶임 | a37cb8596733에서 작은 `DesignSwitcherClose` 최소 클라이언트 컴포넌트로 이동 | 1ac7813155c6가 원본에 `use client`·`useRef`가 없음을 검증 |
| Link 클릭에서 `open` 속성 제거 | 탐색을 이벤트 처리기 의미 구조에 결합 | a37cb8596733에서 링크 onClick 제거; 라우트 탐색에 맡김 | Updated 컴포넌트 테스트는 명시적 닫기·포커스와 href를 검증 |
<!-- learner:thread-relations:end -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:thread-ownership:start -->
| 단계 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| CSS 준비 | 데스크톱·모바일 배치 | `<details>` 요소 `open` 속성 선택자를 예상 |
| e43 초기 컴포넌트 | 브라우저 기본 `<details>` 요소 상태 + 클라이언트 참조 | 열기 버튼·현재 선택된 문구만 존재 |
| c69 완성 | 클라이언트 컴포넌트가 목록, 링크, 참조, 닫기·포커스를 모두 소유 | 초기 functional 규칙 |
| 외부 분류 02/b957 | SiteHeader와 PageShell이 컴포넌트 접근 가능 여부와 UI·템플릿 속성을 소유 | 실제 공용 셸 소비자 |
| 후속 분류 07 | 서버 마크업 + 클라이언트 닫기 컴포넌트 | 브라우저가 관리하는 상태를 하이드레이션 중에도 안전하게 보존하도록 소유 주체를 조정 |
<!-- learner:thread-ownership:end -->

## 9. 개발 과정 최종 상태

<!-- learner:thread-final-state:start -->
- 고정 커밋 수 집합의 최종 상태는 클라이언트 DesignSwitcher 안의 브라우저 기본 `<details>`/요약·탐색·목록이며, 이 집합만으로는 아직 공용 셸에 연결되지 않습니다.
- 실제 라우트 넓은 접근 가능 여부는 인접 분류 02의 b9571c485013이 `SiteHeader`/`PageShell`에 조건부 소비자를 추가하면서 생깁니다.
- 데스크톱은 절대 dropdown, 모바일은 배경이 있는 고정된 아래 선택 패널을 같은 DOM으로 표현합니다.
- Active 항목과 개수·복사 대체 처리, 템플릿·디버그를 고려하는 링크가 구현돼 있습니다.
- 명시적 닫기 버튼은 `open` 속성을 제거하고 요약 포커스를 복원합니다.
- 이 초기 상태의 하이드레이션 경쟁 조건은 후속 분류 07 커밋에서 재현·수정·검증됩니다.
<!-- learner:thread-final-state:end -->

## 10. 최종 실행 순서

<!-- learner:thread-flow:start -->
1. 고정된 맵은 DesignSwitcher의 컴포넌트 규칙을 완성하고, 분류 02의 b9571c485013이 `SiteHeader`에서 `templateSwitcher`가 있을 때 현재 디자인, 현재 경로, 템플릿, UI 복사와 contentDebug를 전달합니다.
2. DesignSwitcher가 템플릿 맵과 현재 선택된·개수 문구를 파생합니다.
3. 브라우저 기본 `<details>` 요소·요약이 브라우저가 관리하는 펼침 요소를 제공합니다.
4. Nav 목록이 SITE_DESIGNS 순서와 현재 선택된 `aria-current`를 렌더링하고 getTemplateHref로 목적지를 만듭니다.
5. CSS가 화면 크기에 따라 데스크톱 dropdown 또는 모바일 아래 선택 패널로 재배치합니다.
6. 닫기 버튼은 `open` 속성을 제거하고 요약 포커스를 복원합니다.
7. 후속 서버 렌더링 우선 리팩터링에서는 마크업 소유 주체가 서버로 이동하고 닫기 상호작용만 최소 클라이언트 컴포넌트에 남습니다.
<!-- learner:thread-flow:end -->

## 11. 학습 완료 확인

<!-- learner:thread-checklist:start -->
- [x] 모든 커밋을 해당 SHA의 변경 내용과 변경 후 파일 기준으로 기록했습니다.
- [x] SHA, 제목, 순서, 중요도, 태그와 개발 과정 역할을 검토 후 고정한 문서 틀과 동일하게 유지했습니다.
- [x] 이전 상태, 소유 주체, 누락·실패, 보장 범위, 보장하지 않는 범위, 후속 참조 관계를 채웠습니다.
- [x] S·A 설명을 중요도 B보다 깊게 작성했습니다.
- [x] 실행 상태를 사실대로 기록했습니다: 실행 명령은 실행하지 않았고 정적 검토와 구분했습니다.
- [x] 빈 학습자용 답변 cell을 남기지 않았습니다.
<!-- learner:thread-checklist:end -->
===== END FILE: 08-design-switcher-disclosure-and-responsive-sheet.md =====

===== BEGIN FILE: README.md =====
# 03-공용 UI 상호작용과 반응형 공용 컴포넌트

`web/portfolio` 브랜치의 공용 UI 상호작용과 반응형 공용 컴포넌트 이력을 학습하는 분류 workbook입니다.

## 1단계 검토 결과

- 분류 구분 지점은 공용 렌더러·컴포넌트·CSS 공용 컴포넌트와 반응형 상호작용에 한정했습니다.
- 라우트 실행 주기, 콘텐츠 스키마·로더, 디자인 라우트 조립 전체, 하이드레이션·성능 회귀 설계는 인접 분류가 소유합니다.
- 기존 01과 06에 중복됐던 `e37ea9c2819a`, `1ef269fbdb49`, `daa6815a6dfa`는 링크 규칙 변경 과정인 01에만 귀속시켰습니다.
- 01에는 노출 위치 값 집합, 상세 선택자 전환, 소개 영역 노출 위치 adoption, 회귀 테스트와 렌더링 중복 제거 커밋을 추가해 규칙 → 소비자 → 테스트 → 리팩터링 흐름을 완성했습니다.
- 02에는 실제 `ProfilePhoto` 소비자인 `a00a6bf1af58`을 추가했습니다.
- 06은 링크 동작 규칙을 제거하고 `AvailabilityBadge` → `ProjectCard` → 대표 소비자 → 상호작용 CSS의 카드 조립 변경 과정으로 보정했습니다.
- README가 약속했지만 빠져 있던 브라우저 기본 펼침 요소·모바일 선택 패널 구현 변경 과정을 08 개발 과정으로 추가했습니다.
- 08의 실제 공용 셸 통합은 분류 02의 `b9571c485013`이, 후속 하이드레이션 수정·테스트·서버 리팩터링은 분류 07이 소유하므로 여기서는 중복하지 않고 관계만 고정했습니다.
- 나머지 개발 과정의 경계와 커밋 순서는 실제 이력에 맞아 유지했습니다.
- 공용 의존성과 소비자 변경 과정이 전역 이력에서 서로 교차하므로 개발 과정 번호를 전체 커밋 시간 순서대로 강제하지 않았습니다. 각 개발 과정 내부는 시간 순서이고 분류 목록은 의존성·학습 경계를 따릅니다.

## Branch·메타데이터 검증 방식

- Branch의 `commit/commit-importance.md`는 최상위부터 최종 커밋까지 독립 linear 이력 476개를 분류합니다.
- 이 분류가 참조하는 모든 커밋은 그 브랜치별 목록에서 확인하고 해당 SHA 커밋 view로 제목와 변경된 파일을 재확인했습니다.
- 다른 브랜치의 구현, 테스트, docs 또는 최종 HEAD를 과거 SHA 설명에 사용하지 않았습니다.
- 실행 가능한 저장소 사본은 작업 환경의 GitHub DNS 해석 실패로 만들지 못했습니다. 실행 시점 테스트 결과는 없으며 모든 완료 문서가 이를 명시합니다.

## 개발 과정 목록

| 순서 | 개발 과정 | 커밋 수 | 경계 |
| --- | --- | --- | --- |
| 1 | [콘텐츠 링크 보안, 노출 위치, 및 이동 방식](01-content-link-security-and-transport.md) | 10 | 이 개발 과정은 링크가 어디에 노출되고 어떤 요소·URL로 이동하는지를 소유합니다. `ProjectCard`의 카드 조립과 마우스 오버 표현은 06 개발 과정에 남기며, 라우트 실행 주기 자체는 이 범위에 포함하지 않습니다. |
| 2 | [콘텐츠 미디어 로딩 및 레이아웃 안정성](02-content-media-loading-and-layout-stability.md) | 5 | 이 개발 과정은 공용 미디어 DOM과 로딩·레이아웃 규칙을 다룹니다. 이미지 최적화 처리 단계, 원본 자산 생성, 프로젝트 상세 정보 설계 자체는 포함하지 않습니다. |
| 3 | [점진적 등장 효과에서 서버 우선 렌더링으로](03-progressive-reveal-to-서버 우선-rendering.md) | 4 | 이 개발 과정은 공용 등장 효과의 표시 여부·실행 주기와 동작 효과 대체 처리를 다룹니다. 개별 섹션 콘텐츠, 라우트 조립, 전역 성능 테스트 설계는 포함하지 않습니다. |
| 4 | [터미널 상태 기계 및 동작 효과 대체 처리](04-terminal-state-machine-and-motion-fallback.md) | 4 | 이 개발 과정은 터미널 미리보기 공용 컴포넌트의 상태와 표현을 다룹니다. 터미널 콘텐츠 스키마 검증, 홈 라우트 전체 섹션 설계, 일반 동작 효과 규칙은 각각 다른 개발 과정의 책임입니다. |
| 5 | [기술 스택 아이콘, 목록, 및 흐르는 목록](05-technology-stack-icon-list-and-marquee.md) | 5 | 이 개발 과정은 기술 스택 시각 공용 컴포넌트를 다룹니다. 기술 스택 콘텐츠 스키마와 `resolveTechStackItem`의 검증·대체 처리 기준, 홈 섹션 전체 조립은 다른 개발 과정이 소유합니다. |
| 6 | [프로젝트 카드 조립 및 상호작용](06-project-card-composition-and-interaction.md) | 4 | 1단계에서 기존 `project-card-actions-and-evidence-components` 개발 과정의 링크 규칙 커밋을 01 개발 과정으로 이동했습니다. 이 개발 과정은 카드 조립과 상호작용만 소유하며 링크 노출 위치·이동 방식은 01 개발 과정의 선택자·렌더러를 소비합니다. |
| 7 | [여정 타임라인 공용 컴포넌트 및 반응형 레이아웃](07-journey-timeline-primitives-and-responsive-layout.md) | 6 | 이 개발 과정은 여정 화면 구성 공용 컴포넌트와 반응형 CSS를 다룹니다. 여정 JSON 스키마·검증, 라우트 섹션 복사, 등장 효과의 최종 서버 렌더링 우선 리팩터링은 각각 콘텐츠 시스템과 03 개발 과정이 소유합니다. |
| 8 | [Design 선택기 펼침 요소 및 반응형 선택 패널](08-design-switcher-disclosure-and-responsive-sheet.md) | 4 | 1단계에서 분류 README가 약속했지만 기존 7개 개발 과정이 다루지 않던 반응형 펼침 요소 공용 컴포넌트 구현 변경 과정을 추가했습니다. 실제 공용 셸 연결은 분류 02의 `b9571c485013`이, 후속 하이드레이션 경쟁 상태 수정·테스트와 서버 컴포넌트 리팩터링은 분류 07이 소유합니다. 이 확정된 커밋 목록에는 공용 컴포넌트 구현 커밋만 두고 인접 분류의 통합·수정·근거는 관계로 명시합니다. |

## 구조 규칙

- `scaffold/`는 1단계 검토 뒤 동결된 기준이 되는 workbook입니다.
- `completed/`는 동일 파일 집합·상대 경로·고정된 구성을 유지하고 학습자용 영역만 채운 counterpart입니다.
- Scaffold와 완료된 사이에 SHA, 제목, 순서, 중요도, 태그, 역할 또는 불변 조건 차이가 있으면 유효하지 않은 deliverable입니다.
===== END FILE: README.md =====

