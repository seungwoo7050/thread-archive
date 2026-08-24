===== BEGIN FILE: 01-cross-design-token-shell-and-regression-contracts.md =====
# 개발 과정: 디자인 공통 토큰·셸·회귀 검증 규칙

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `05-full-site-visual-systems`
>
> 1단계 검토에서 확정한 기준 문서 틀입니다. 2단계에서는 답변 식별 속성 내부만 채웁니다.

## 0. 범위와 기준

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md`와 해당 커밋의 정확한 메타데이터를 기준으로 고정했습니다.
- **개발 과정 범위:** 공통 등록부·렌더러 입력·App Router 위임·Design/Classic 셸 조립 단위·활성화·디자인 공통 회귀만 포함합니다. Editorial/Brutalist/Cinematic 내부 조립은 각 전용 개발 과정에, Design/Classic의 라우트별 분리는 개발 과정 5에 둡니다.
- 다른 브랜치, 최종 HEAD의 후대 구현, 실행하지 않은 명령 결과를 사용하지 않습니다.

## 1. 개발 과정 목표

다섯 디자인이 독립적인 사이트 전체 렌더러로 동작하면서도 라우트 허용 값 집합, 준비된 입력, 셸 상태, 토큰 역할과 회귀 검증 경계를 공유하게 되는 과정을 복원합니다.

### 고정된 불변 조건

최종 불변 조건은 App Router가 프레임워크가 처리해야 할 작업과 정확한 라우트 뷰 모델을 소유하고, 등록부가 디자인을 선택하며, 각 렌더러는 종류가 구분된 준비된 입력만 받아 자체 셸·화면 구성을 반환한다는 것입니다.

## 2. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `418e7bc1d8bb` | feat(designs): site design 정의 registry 추가 | A | ARCH, RENDERER | 사이트 디자인 허용 값 집합과 선택 등록부의 시작점 |
| 2 | `e14202198948` | feat(designs): route renderer 계약 추가 | A | ARCH, ROUTING, RENDERER | 여덟 공개 라우트를 닫힌 유니언으로 표현하는 첫 렌더러 입력 규칙 |
| 3 | `6fc28f4c6586` | refactor(designs): 확장 renderer lazy registry 추가 | A | ARCH, RENDERER, REFACTOR | 확장 디자인의 지원 여부 조회와 지연 로더 호출 지점 |
| 4 | `dc2cf72a768d` | refactor(routes): 확장 디자인 renderer 위임 경계 추가 | S | ARCH, ROUTING, RENDERER | 모든 공개 라우트에서 프레임워크 책임과 사이트 전체 화면 구성을 분리한 최초의 공통 경계 |
| 5 | `c6acfe562694` | feat(editorial): renderer를 디자인 registry에 활성화 | A | ARCH, RENDERER | 완성된 Editorial을 선택 가능한 사이트 전체 렌더러로 승격 |
| 6 | `dd71d28143a8` | feat(designs): Brutalist renderer 활성화 | A | ARCH, RENDERER | 완성된 Brutalist를 등록부·콘텐츠 규칙에 연결 |
| 7 | `b8de57f130eb` | feat(designs): Cinematic renderer 활성화 | A | ARCH, RENDERER | Cinematic의 선택 가능 상태와 모듈 공개 범위 축소 |
| 8 | `1598a87702f6` | test(design): view model 기반 renderer matrix 검증 | A | ARCH, VALIDATION, RENDERER | 다섯 디자인×여섯 전환 완료 라우트의 정적 DOM 호환 조합표 |
| 9 | `4732301a7d2c` | style(designs): route renderer 디자인 토큰 확장 | A | ARCH, ROUTING, RENDERER | 색상 외 일곱 디자인 공통 토큰 종류의 도입 |
| 10 | `969741c3469d` | refactor(shell): 디자인 renderer 셸 경계 추가 | A | ARCH, ROUTING, RENDERER | Design·Classic도 준비된 셸 속성을 받아 독립 최상위 식별 속성을 갖는 전환 |
| 11 | `8a48460df4c3` | refactor(journey): 모든 renderer에 여정 view model 적용 | A | ARCH, RENDERER, REFACTOR | 여정 참조 해석을 렌더러 밖으로 이동 |
| 12 | `aef265b9bd01` | refactor(interview): 모든 renderer에 인터뷰 view model 적용 | A | ARCH, RENDERER, REFACTOR | 인터뷰 답변과 프로젝트 연결을 라우트 뷰 모델에 통일 |
| 13 | `f8b0ab7b08aa` | refactor(designs): renderer 입력을 route view model로 제한 | S | ARCH, ROUTING, RENDERER | 렌더러가 전역 콘텐츠 참조 관계를 읽지 못하게 하는 라우트 종류로 구분되는 입력 불변 조건 |
| 14 | `380b2a025070` | refactor(designs): 모든 route를 registry renderer로 위임 | S | ARCH, ROUTING, RENDERER | 여덟 App Router 페이지와 다섯 디자인을 하나의 분기 설계로 폐쇄 |
| 15 | `055b733cbb7e` | test(design): 독립 renderer와 design token 경계 검증 | A | ARCH, VALIDATION, RENDERER | 8×5 라우트 조합표와 토큰·렌더러 식별 속성의 정적 회귀 규칙 |
| 16 | `882a2f9d753e` | test(visual): 다섯 디자인 회귀 기준 추가 | A | TEST | Playwright 화면 캡처 기준선과 스냅샷 목록 계약 |

## 3. 변경 전 기준

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:baseline:BEGIN -->
- **직전 상태:** 초기에는 `SiteDesignId`와 라우트 전용 화면 구성 담당이 App Router와 공용 컴포넌트에 분산돼 있었고, 확장 디자인은 등록부 후보일 뿐 공개 라우트에서 실행되지 않았습니다.
- **경계 판단:** 공통 등록부·렌더러 입력·App Router 위임·Design/Classic 셸 조립 단위·활성화·디자인 공통 회귀만 포함합니다. Editorial/Brutalist/Cinematic 내부 조립은 각 전용 개발 과정에, Design/Classic의 라우트별 분리는 개발 과정 5에 둡니다.
- **복원 기준:** 각 커밋의 부모 커밋과 해당 SHA 파일 트리만 사용하고 최종 HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:baseline:END -->

## 4. 커밋별 복원 기록

### 1. `418e7bc1d8bb` — feat(designs): site design 정의 registry 추가

- **중요도:** A
- **태그:** ARCH, RENDERER
- **개발 과정에서의 역할:** 사이트 디자인 허용 값 집합과 선택 등록부의 시작점

#### 커밋별 확인 사항

- `418e7bc1d8bb^`와 `418e7bc1d8bb`를 비교하고 `src/designs/config.ts`에서 **`SiteDesignDefinition`, 디자인 ID 목록, 기본 디자인 선택과 알 수 없는 ID 대체 처리** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `사이트 디자인 허용 값과 선택 등록부의 시작점` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `등록된 디자인이 실제로 모든 라우트를 렌더링한다고 보장할 수는 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 다음 `e14202198948`가 이 ID 허용 값 집합 위에 라우트 렌더러 입력 규칙을 얹는다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:418e7bc1d8bb:BEGIN -->
- **직전 상태:** 직전 상태에는 **`SiteDesignDefinition`, 디자인 ID 목록, 기본 디자인 선택과 알 수 없는 ID 대체 처리** 관련 라우트·컴포넌트가 아직 없었습니다.
- **구현 결정:** 이 커밋은 `design`과 `classic`을 데이터 정의로 등록하고, 조회 실패 시 첫 정의를 반환하는 등록부를 만듭니다. 디자인 선택은 더 이상 라우트별 문자열 분기에만 의존하지 않지만, 아직 라우트 렌더러 입력 규칙은 없습니다.
- **파일·심볼:** `src/designs/config.ts`에서 `SiteDesignDefinition`, 디자인 ID 목록, 기본 디자인 선택과 알 수 없는 ID 대체 처리를 확인했습니다.
- **소유권:** `src/designs/config.ts`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `사이트 디자인 허용 값과 선택 등록부의 시작점`. 등록된 디자인이 실제 모든 라우트를 렌더링한다는 보장은 아직 없습니다.
- **역사적 연결:** 다음 `e14202198948`가 이 ID 허용 값 집합 위에 라우트 렌더러 입력 규칙을 얹는다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:418e7bc1d8bb:END -->

### 2. `e14202198948` — feat(designs): route renderer 계약 추가

- **중요도:** A
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** 여덟 공개 라우트를 닫힌 유니언으로 표현하는 첫 렌더러 입력 규칙

#### 커밋별 확인 사항

- `e14202198948^`와 `e14202198948`를 비교하고 `src/designs/types.ts`에서 **`DesignRouteName`, `DesignRouteProps`, 라우트별 선택적 프로젝트/`currentPath`/contentDebug 입력** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `8개 공개 라우트를 닫힌 유니언으로 표현하는 첫 렌더러 입력 규칙` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `타입 정의만으로 App Router 위임이나 지원하지 않는 렌더러 처리가 이루어지지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `6fc28f4c6586`가 이 타입을 지연 로딩 등록부의 로더 호출 지점로 사용하고, `f8b0ab7b08aa`가 후속에 원본 콘텐츠 접근을 금지합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:e14202198948:BEGIN -->
- **직전 상태:** 직전 상태에는 **`DesignRouteName`, `DesignRouteProps`, 라우트별 선택적 프로젝트/`currentPath`/contentDebug 입력** 관련 라우트·컴포넌트가 아직 없었습니다.
- **구현 결정:** 홈, 프로젝트, 프로젝트 상세, 소개, 이력서, 연락처, 여정, 인터뷰 맵을 닫힌 라우트 유니언으로 고정하고 렌더러가 받을 공통 속성을 정의합니다. 이 시점 입력은 여전히 전체 `PortfolioContent`라서 렌더러가 전역 콘텐츠 그래프에 접근할 수 있습니다.
- **파일·심볼:** `src/designs/types.ts`에서 `DesignRouteName`, `DesignRouteProps`, 라우트별 선택적 프로젝트/`currentPath`/contentDebug 입력을 확인했습니다.
- **소유권:** `src/designs/types.ts`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `8개 공개 라우트를 닫힌 유니언으로 표현하는 첫 렌더러 입력 규칙`. 타입 정의만으로 App Router가 위임하거나 미지원 렌더러를 처리하지는 않습니다.
- **역사적 연결:** `6fc28f4c6586`가 이 타입을 지연 로딩 등록부의 로더 호출 지점로 사용하고, `f8b0ab7b08aa`가 후속에 원본 콘텐츠 접근을 금지합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:e14202198948:END -->

### 3. `6fc28f4c6586` — refactor(designs): 확장 renderer lazy registry 추가

- **중요도:** A
- **태그:** ARCH, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 확장 디자인의 지원 여부 조회와 지연 로더 호출 지점

#### 커밋별 확인 사항

- `6fc28f4c6586^`와 `6fc28f4c6586`를 비교하고 `src/designs/registry.tsx`에서 **`routeLoaders`, `hasDedicatedRouteRenderer`, `renderDesignRoute`의 null 반환 경로** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `확장 디자인의 지원 여부 조회와 지연 로더 호출 지점` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `지연 등록부는 대체 화면이나 오류 보고 기능을 제공하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `dc2cf72a768d`가 공개 라우트에서 이 null 가능 경계를 소비하고, 각 활성화 커밋이 로더를 채운다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:6fc28f4c6586:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`routeLoaders`, `hasDedicatedRouteRenderer`, `renderDesignRoute`의 null 반환 경로**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **구현 결정:** Editorial·Brutalist·Cinematic을 전용 렌더러 후보로 분류하지만 `routeLoaders`가 비어 있어 실제 로더가 없으면 `renderDesignRoute`는 `null`을 반환합니다. 지원 여부와 구현 등록을 분리해 점진적 활성화가 가능해졌으나, 이 시점에는 확장 디자인을 선택해도 전용 화면이 보장되지 않습니다.
- **파일·심볼:** `src/designs/registry.tsx`에서 `routeLoaders`, `hasDedicatedRouteRenderer`, `renderDesignRoute`의 null 반환 경로를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/designs/registry.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `확장 디자인의 지원 여부 조회와 지연 로더 호출 지점`. 지연 로딩 등록부는 대체 처리 UI나 오류 보고를 제공하지 않습니다.
- **역사적 연결:** `dc2cf72a768d`가 공개 라우트에서 이 null 가능 경계를 소비하고, 각 활성화 커밋이 로더를 채운다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

```tsx
// 6fc28f4c6586 · src/designs/registry.tsx
const loader = routeLoaders[designId];
if (!loader) return null;
```
<!-- LEARNER-ANSWER:commit:6fc28f4c6586:END -->

### 4. `dc2cf72a768d` — refactor(routes): 확장 디자인 renderer 위임 경계 추가

- **중요도:** S
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** 모든 공개 라우트에서 프레임워크 책임과 사이트 전체 화면 구성을 분리한 최초의 공통 경계

#### 커밋별 확인 사항

- `dc2cf72a768d^`와 `dc2cf72a768d`를 비교하고 `src/app/page.tsx 외 7개 공개 라우트 페이지, src/designs/registry.tsx`에서 **각 페이지의 콘텐츠 로딩·notFound·메타데이터 이후 `hasDedicatedRouteRenderer`와 `renderDesignRoute` 호출, null 결과와 Design·Classic 대체 처리** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `모든 공개 라우트에서 프레임워크 처리와 사이트 전체 화면 구성을 처음 분리한 공통 지점` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 단계의 공통 입력은 검증 전 콘텐츠이며, 라우트별 뷰 모델 제한과 단일 분기 방식은 아직 적용되지 않았습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: Editorial/Brutalist/Cinematic 활성화가 이 경계를 실제 구현에 연결하고, `380b2a025070`이 최종적으로 Design·Classic까지 같은 경로로 통일합니다.
- App Router → 뷰 모델·콘텐츠 처리 지점 → 등록부·분기 함수 → 셸·view의 전체 호출 경로를 부모 커밋과 비교합니다.
- 호환 브랜치가 남아 있는지, 제거되는 시점은 언제인지, 이 불변 조건이 다른 네 개발 과정에 미치는 영향을 연결합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:dc2cf72a768d:BEGIN -->
#### 복원 결과
- **직전 상태와 위험:** 동작 자체는 앞선 구현에 존재했지만 **각 페이지의 콘텐츠 로딩·notFound·메타데이터 이후 `hasDedicatedRouteRenderer`와 `renderDesignRoute` 호출, null 결과와 Design·Classic 대체 처리**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다. 이 분산 상태는 라우트·디자인마다 다른 파생·대체 처리·셸 처리를 허용해 같은 입력이 다른 실행 경로를 탈 위험이 있었습니다.
- **핵심 결정:** 직전에는 등록부가 있어도 공개 라우트가 이를 호출하지 않아 확장 디자인이 실행될 수 없었다. 이 커밋은 여덟 App Router 페이지가 라우트 이름·현재 경로·필요한 프로젝트를 준비한 뒤 등록부로 넘기게 합니다. App Router는 프레임워크 규칙을 유지하고, 전용 렌더러는 전체 화면 구성을 소유합니다. 아직 Design·Classic은 기존 공용 경로로 남고 로더 미등록 결과는 기존 화면으로 후퇴합니다.
- **실제 정적 검토 지점:** `src/app/page.tsx 외 7개 공개 라우트 페이지, src/designs/registry.tsx`에서 각 페이지의 콘텐츠 로딩·notFound·메타데이터 이후 `hasDedicatedRouteRenderer`와 `renderDesignRoute` 호출, null 결과와 Design·Classic 대체 처리를 부모와의 변경 차이 및 변경 후 파일 트리로 추적했습니다.
- **책임과 상태 전환:** 표현·조립 책임은 `src/app/page.tsx 외 7개 공개 라우트 페이지, src/designs/registry.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **실패·비보장:** 이 단계의 공통 입력은 원본 콘텐츠이며, 라우트별 뷰 모델 제한과 단일 분기는 아직 아닙니다.
- **후속 폐쇄:** Editorial/Brutalist/Cinematic 활성화가 이 경계를 실제 구현에 연결하고, `380b2a025070`이 최종적으로 Design·Classic까지 같은 경로로 통일합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

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

- **중요도:** A
- **태그:** ARCH, RENDERER
- **개발 과정에서의 역할:** 완성된 Editorial을 선택 가능한 사이트 전체 렌더러로 승격

#### 커밋별 확인 사항

- `c6acfe562694^`와 `c6acfe562694`를 비교하고 `src/designs/registry.tsx, src/designs/config.ts, src/content/presentation.json, src/lib/portfolio/content-loader.ts`에서 **Editorial 로더 등록, 디자인 정의·표시 문구·콘텐츠 검증의 동시 확장** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `완성된 Editorial을 선택 가능한 사이트 전체 렌더러로 등록` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `등록만으로 시각적 정확성이나 모든 콘텐츠 조합이 검증되지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 2의 `46e23d922c2e` 분기 함수가 선행 구현이며, 이 커밋은 그 결과를 공통 선택 경계에 연결합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:c6acfe562694:BEGIN -->
- **직전 상태:** 직전 상태에는 **Editorial 로더 등록, 디자인 정의·표시 문구·콘텐츠 검증의 동시 확장** 관련 라우트·컴포넌트가 아직 없었습니다.
- **구현 결정:** Editorial 모듈이 존재하는 것과 사용자가 선택 가능한 것은 별개였습니다. 이 커밋은 등록부 로더와 설정·콘텐츠 경계를 함께 갱신해 `editorial` 선택이 여덟 라우트 렌더러로 도달하도록 합니다.
- **파일·심볼:** `src/designs/registry.tsx, src/designs/config.ts, src/content/presentation.json, src/lib/portfolio/content-loader.ts`에서 Editorial 로더 등록, 디자인 정의·표시 문구·콘텐츠 검증의 동시 확장을 확인했습니다.
- **소유권:** `src/designs/registry.tsx, src/designs/config.ts, src/content/presentation.json, src/lib/portfolio/content-loader.ts`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `완성된 Editorial을 선택 가능한 사이트 전체 렌더러로 등록`. 활성화는 시각 정확성이나 모든 콘텐츠 조합을 검증하지 않습니다.
- **역사적 연결:** 개발 과정 2의 `46e23d922c2e` 분기 함수가 선행 구현이며, 이 커밋은 그 결과를 공통 선택 경계에 연결합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:c6acfe562694:END -->

### 6. `dd71d28143a8` — feat(designs): Brutalist renderer 활성화

- **중요도:** A
- **태그:** ARCH, RENDERER
- **개발 과정에서의 역할:** 완성된 Brutalist를 등록부·콘텐츠 규칙에 연결

#### 커밋별 확인 사항

- `dd71d28143a8^`와 `dd71d28143a8`를 비교하고 `src/designs/registry.tsx`, `src/designs/config.ts`, 화면 구성 및 콘텐츠 검증 파일에서 **Brutalist 로더와 선택 가능한 디자인 메타데이터, 라우트 진입점** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `완성된 Brutalist를 등록부와 콘텐츠 규칙에 연결` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `등록 자체만으로 CSS 화면 너비 기준 처리와 라우트별 빈 상태의 품질까지 보장되지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 3의 `caa7df81d899` 단일 진입점을 소비합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:dd71d28143a8:BEGIN -->
- **직전 상태:** 직전 상태에는 **Brutalist 로더와 선택 가능한 디자인 메타데이터, 라우트 진입점** 관련 라우트·컴포넌트가 아직 없었습니다.
- **구현 결정:** Brutalist의 내부 라우트 집합이 완성된 뒤 등록부가 모듈 진입점을 지연 로딩하도록 바뀝니다. 선택 ID, 사용자에게 표시할 문구, 로더가 한 커밋에서 일치하므로 등록되지 않음·등록됨·구현됨 상태가 갈라지지 않습니다.
- **파일·심볼:** `src/designs/registry.tsx`, `src/designs/config.ts`, 화면 구성 및 콘텐츠 검증 파일에서 Brutalist 로더와 선택 가능한 디자인 메타데이터, 라우트 진입점을 확인했습니다.
- **소유권:** `src/designs/registry.tsx`, `src/designs/config.ts`, 화면 구성 및 콘텐츠 검증 파일이 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `완성된 Brutalist를 등록부와 콘텐츠 규칙에 연결`. CSS 화면 너비 기준과 라우트별 빈 상태의 품질은 등록 자체만으로 보장되지 않습니다.
- **역사적 연결:** 개발 과정 3의 `caa7df81d899` 단일 진입점을 소비합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:dd71d28143a8:END -->

### 7. `b8de57f130eb` — feat(designs): Cinematic renderer 활성화

- **중요도:** A
- **태그:** ARCH, RENDERER
- **개발 과정에서의 역할:** Cinematic의 선택 가능 상태와 모듈 공개 범위 축소

#### 커밋별 확인 사항

- `b8de57f130eb^`와 `b8de57f130eb`를 비교하고 `src/designs/registry.tsx`, `src/designs/config.ts`, `src/designs/cinematic/cinematic-route.tsx`, 콘텐츠 파일에서 **Cinematic 로더 등록과 라우트 진입점 공개만 남기는 공개 범위** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic을 선택 가능하게 만들고 모듈 공개 API를 축소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `모든 경우를 빠짐없이 처리하는 별도 디스패처를 추가하지 않으며, 라우트 진입점 내부의 `switch` 분기 품질에 의존합니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 4의 라우트 조립 전체를 공통 등록부에 연결합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b8de57f130eb:BEGIN -->
- **직전 상태:** 직전 상태에는 **Cinematic 로더 등록과 라우트 진입점 공개만 남기는 공개 범위** 관련 라우트·컴포넌트가 아직 없었습니다.
- **구현 결정:** Cinematic을 선택 가능한 디자인으로 등록하면서 내부 뷰와 도우미 함수 공개를 제거하고 등록부가 라우트 진입점 하나만 알도록 정리합니다. 활성화와 모듈 소유 주체 축소가 함께 일어나 등록부가 내부 조립 세부사항에 결합되지 않습니다.
- **파일·심볼:** `src/designs/registry.tsx`, `src/designs/config.ts`, `src/designs/cinematic/cinematic-route.tsx`, 콘텐츠 파일에서 Cinematic 로더 등록과 라우트 진입점 공개만 남기는 공개 범위를 확인했습니다.
- **소유권:** `src/designs/registry.tsx`, `src/designs/config.ts`, `src/designs/cinematic/cinematic-route.tsx`, 콘텐츠 파일이 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `Cinematic을 선택 가능하게 만들고 모듈 공개 API를 축소`. 별도 누락 없는 분기 함수가 추가되는 것은 아니며, 라우트 진입점 내부의 전환·분기 품질에 의존합니다.
- **역사적 연결:** 개발 과정 4의 라우트 조립 전체를 공통 등록부에 연결합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b8de57f130eb:END -->

### 8. `1598a87702f6` — test(design): view model 기반 renderer matrix 검증

- **중요도:** A
- **태그:** ARCH, VALIDATION, RENDERER
- **개발 과정에서의 역할:** 다섯 디자인×여섯 전환 완료 라우트의 정적 DOM 호환 조합표

#### 커밋별 확인 사항

- `1598a87702f6^`와 `1598a87702f6`를 비교하고 `src/designs/route-view-models.test.tsx`에서 **`designIds`, `routes`, Testing Library 렌더링, `data-site-design`와 비어 있지 않은 `h1` 단언문** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `5개 디자인×6개 전환 완료 라우트의 정적 DOM 호환성 조합표` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `브라우저 레이아웃, CSS 적용, 상호작용, 여정·인터뷰 라우트는 이 SHA의 검증 범위가 아닙니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `055b733cbb7e`가 여정·인터뷰를 추가하고 Design·Classic 독립 렌더러 식별 속성을 검증합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1598a87702f6:BEGIN -->
- **직전 상태:** 관련 배포 환경 검사 범위는 존재했지만 **`designIds`, `routes`, Testing Library 렌더링, `data-site-design`와 비어 있지 않은 `h1` 단언문**을 실패 시 자동으로 탐지하는 회귀 증거가 없었습니다.
- **구현 결정:** Vitest/jsdom에서 홈·프로젝트·프로젝트 상세·소개·이력서·연락처를 다섯 디자인으로 직접 호출합니다. 활성화된 프로젝트가 없으면 테스트 입력 전제 오류를 내고, 각 조합에서 디자인 최상위와 비어 있지 않은 `h1`을 확인합니다. 이는 30개 조합의 서버 컴포넌트 결과가 기본 HTML 경계를 유지함을 증명합니다.
- **파일·심볼:** `src/designs/route-view-models.test.tsx`에서 `designIds`, `routes`, Testing Library 렌더링, `data-site-design`와 비어 있지 않은 `h1` 단언문을 확인했습니다.
- **소유권:** 실제 코드의 소유 주체는 변경하지 않고 `src/designs/route-view-models.test.tsx`가 회귀 판정과 테스트 입력 전제를 소유합니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `5개 디자인×6개 전환 완료 라우트의 정적 DOM 호환성 조합표`. 브라우저 레이아웃, CSS 적용, 상호작용, 여정·인터뷰는 이 SHA의 검증 범위가 아닙니다.
- **역사적 연결:** `055b733cbb7e`가 여정·인터뷰를 추가하고 Design·Classic 독립 렌더러 식별 속성을 검증합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1598a87702f6:END -->

### 9. `4732301a7d2c` — style(designs): route renderer 디자인 토큰 확장

- **중요도:** A
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** 색상 외 일곱 디자인 공통 토큰 종류의 도입

#### 커밋별 확인 사항

- `4732301a7d2c^`와 `4732301a7d2c`를 비교하고 `src/app/globals.css`에서 **`--type-display`, `--type-body`, `--space-section`, `--breakpoint-content`, `--motion-fast`, `--layer-navigation`, `--content-width`의 디자인 범위** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `색상 외 7개 디자인 공통 토큰 묶음 도입` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `모든 렌더러가 모든 토큰을 실제 사용하거나 같은 시각적 의미로 해석한다고 보장하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `055b733cbb7e`의 정적 토큰 규칙이 이름 존재와 Design·Classic 범위를 보호합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4732301a7d2c:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **`--type-display`, `--type-body`, `--space-section`, `--breakpoint-content`, `--motion-fast`, `--layer-navigation`, `--content-width`의 디자인 범위**를 완결하는 선택자·미디어 규칙 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **구현 결정:** 각 `[data-site-design]` 범위에 타이포그래피, rhythm, 화면 너비 기준 값, 동작 효과, 탐색 레이어, 콘텐츠 width를 정의합니다. 공통 소비자는 특히 콘텐츠 width를 사용하지만, CSS 사용자 정의 속성에 화면 너비 기준 숫자를 둔 것만으로 미디어 쿼리가 자동 적용되지는 않습니다.
- **파일·심볼:** `src/app/globals.css`에서 `--type-display`, `--type-body`, `--space-section`, `--breakpoint-content`, `--motion-fast`, `--layer-navigation`, `--content-width`의 디자인 범위를 확인했습니다.
- **소유권:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/app/globals.css` 스타일시트가 해당 디자인에 적용합니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `색상 외 7개 디자인 공통 토큰 묶음 도입`. 모든 렌더러가 모든 토큰을 실제 소비하거나 시각적으로 동일한 의미를 갖는다는 보장은 없습니다.
- **역사적 연결:** `055b733cbb7e`의 정적 토큰 규칙이 이름 존재와 Design·Classic 범위를 보호합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4732301a7d2c:END -->

### 10. `969741c3469d` — refactor(shell): 디자인 renderer 셸 경계 추가

- **중요도:** A
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** Design·Classic도 준비된 셸 속성을 받아 독립 최상위 식별 속성을 갖는 전환

#### 커밋별 확인 사항

- `969741c3469d^`와 `969741c3469d`를 비교하고 `src/components/portfolio/site-shell.tsx, src/designs/shell-props.ts`에서 **`createDesignShellProps`, `data-route-renderer`, `footerLinks`/`templateSwitcher`/`currentPath` 조립** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Design·Classic도 준비된 셸 속성을 받아 독립된 최상위 식별 속성을 갖도록 전환` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 도우미 함수는 Editorial·Brutalist·Cinematic의 자체 셸에 사용을 강제하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 5의 Design·Classic 분리가 이 도우미 함수를 소비하고, `055b733cbb7e`가 식별 속성을 검사합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:969741c3469d:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createDesignShellProps`, `data-route-renderer`, `footerLinks`/`templateSwitcher`/`currentPath` 조립**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **구현 결정:** 공용 `PageShell`에 렌더러 식별 속성을 선택적으로 붙이고, 라우트 렌더러가 필요한 셸 속성을 한 도우미 함수에서 조립합니다. 푸터·탐색·디버그·현재 경로의 반복 해석이 App Router나 개별 섹션에서 분산되지 않게 됩니다.
- **파일·심볼:** `src/components/portfolio/site-shell.tsx, src/designs/shell-props.ts`에서 `createDesignShellProps`, `data-route-renderer`, `footerLinks`/`templateSwitcher`/`currentPath` 조립을 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/components/portfolio/site-shell.tsx, src/designs/shell-props.ts` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `Design·Classic도 준비된 셸 속성을 받아 독립된 최상위 식별 속성을 갖도록 전환`. Editorial·Brutalist·Cinematic의 자체 셸을 이 도우미 함수로 강제하지는 않습니다.
- **역사적 연결:** 개발 과정 5의 Design·Classic 분리가 이 도우미 함수를 소비하고, `055b733cbb7e`가 식별 속성을 검사합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:969741c3469d:END -->

### 11. `8a48460df4c3` — refactor(journey): 모든 renderer에 여정 view model 적용

- **중요도:** A
- **태그:** ARCH, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 여정 참조 해석을 렌더러 밖으로 이동

#### 커밋별 확인 사항

- `8a48460df4c3^`와 `8a48460df4c3`를 비교하고 `src/app/journey/page.tsx, src/lib/portfolio/view-models.ts, 다섯 렌더러의 여정 path`에서 **`createJourneyViewModel`, 해석된 주요 시점·기준 프로젝트, 원본 `PortfolioContent` 의존 제거** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `여정 참조 해석을 렌더러 밖으로 이동` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `표현 방식과 빈 상태 문구는 각 렌더러가 계속 담당합니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `f8b0ab7b08aa`가 이 전환을 공통 타입 규칙으로 강제합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:8a48460df4c3:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createJourneyViewModel`, 해석된 주요 시점·기준 프로젝트, 원본 `PortfolioContent` 의존 제거**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **구현 결정:** 여정 라우트가 준비된 뷰 모델을 만들고 다섯 렌더러가 동일한 해석된 주요 시점 구조를 소비하도록 바뀝니다. 프로젝트 ID 조회와 누락 판단이 렌더러별로 반복되지 않아 같은 원본이 디자인마다 다른 링크 결과를 만들 위험을 줄입니다.
- **파일·심볼:** `src/app/journey/page.tsx, src/lib/portfolio/view-models.ts, 다섯 렌더러의 여정 path`에서 `createJourneyViewModel`, 해석된 주요 시점·기준 프로젝트, 원본 `PortfolioContent` 의존 제거를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/app/journey/page.tsx, src/lib/portfolio/view-models.ts, 다섯 렌더러의 여정 path` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `여정 참조 해석을 렌더러 밖으로 이동`. 표현 방식과 빈 상태 문구는 각 렌더러가 계속 소유합니다.
- **역사적 연결:** `f8b0ab7b08aa`가 이 전환을 공통 타입 규칙으로 강제합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:8a48460df4c3:END -->

### 12. `aef265b9bd01` — refactor(interview): 모든 renderer에 인터뷰 view model 적용

- **중요도:** A
- **태그:** ARCH, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 인터뷰 답변과 프로젝트 연결을 라우트 뷰 모델에 통일

#### 커밋별 확인 사항

- `aef265b9bd01^`와 `aef265b9bd01`를 비교하고 `src/app/interview-map/page.tsx, src/lib/portfolio/view-models.ts, 다섯 렌더러의 인터뷰 path`에서 **`createInterviewMapViewModel`, 답변에 연결된 프로젝트 또는 `null`, 트랙·질문 계층 보존** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `인터뷰 답변과 프로젝트 연결을 라우트 뷰 모델에서 통일` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `null`을 어떤 문구와 배치로 보여 줄지는 디자인 공통 보장 사항이 아닙니다. 이 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `f8b0ab7b08aa`의 구분된 유니언에 인터뷰 변형이 들어간다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:aef265b9bd01:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createInterviewMapViewModel`, 답변에 연결된 프로젝트 또는 `null`, 트랙·질문 계층 보존**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **구현 결정:** 각 렌더러가 `Map`이나 `find`로 답변에 연결된 프로젝트를 다시 찾던 책임을 뷰 모델 구분 지점으로 옮긴다. 누락 참조는 `null`로 명시되어 화면 구성 코드는 링크를 만들지 않고 표시 방식만 결정합니다.
- **파일·심볼:** `src/app/interview-map/page.tsx, src/lib/portfolio/view-models.ts, 다섯 렌더러의 인터뷰 path`에서 `createInterviewMapViewModel`, 답변에 연결된 프로젝트 또는 `null`, 트랙·질문 계층 보존을 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/app/interview-map/page.tsx, src/lib/portfolio/view-models.ts, 다섯 렌더러의 인터뷰 path` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **보장·비보장:** 이 SHA는 인터뷰 답변과 프로젝트 연결을 라우트 뷰 모델에 통일합니다. 다만 `null`을 어떤 문구와 배치로 보여 줄지는 디자인마다 다를 수 있습니다.
- **역사적 연결:** `f8b0ab7b08aa`의 구분된 유니언에 인터뷰 변형이 들어간다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:aef265b9bd01:END -->

### 13. `f8b0ab7b08aa` — refactor(designs): renderer 입력을 route view model로 제한

- **중요도:** S
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** 렌더러가 전역 콘텐츠 참조 관계를 읽지 못하게 하는 라우트 종류로 구분되는 입력 불변 조건

#### 커밋별 확인 사항

- `f8b0ab7b08aa^`와 `f8b0ab7b08aa`를 비교하고 `src/designs/types.ts, src/designs/registry.tsx, src/lib/portfolio/view-models.ts, renderer entry files`에서 **`PreparedDesignRouteProps`, 라우트 구분값별 뷰 모델 표현 방식, `footerLinks`와 해석된 참조** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `렌더러가 전체 콘텐츠 관계를 직접 읽지 못하게 하는 구분된 입력 조건` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 타입만으로 외부 JSON의 실행 시점 검증이나 CSS·DOM 품질까지 보장할 수는 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `380b2a025070`이 모든 App Router 페이지를 이 계약의 단일 호출자로 만듭니다.
- App Router → 뷰 모델·콘텐츠 처리 지점 → 등록부·분기 함수 → 셸·view의 전체 호출 경로를 부모 커밋과 비교합니다.
- 호환 브랜치가 남아 있는지, 제거되는 시점은 언제인지, 이 불변 조건이 다른 네 개발 과정에 미치는 영향을 연결합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f8b0ab7b08aa:BEGIN -->
#### 복원 결과
- **직전 상태와 위험:** 동작 자체는 앞선 구현에 존재했지만 **`PreparedDesignRouteProps`, 라우트 구분값별 뷰 모델 표현 방식, `footerLinks`와 해석된 참조**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다. 이 분산 상태는 라우트·디자인마다 다른 파생·대체 처리·셸 처리를 허용해 같은 입력이 다른 실행 경로를 탈 위험이 있었습니다.
- **핵심 결정:** 직전까지 전환 호환 때문에 원본 `PortfolioContent`와 준비된 모델이 함께 통과할 수 있었다. 이 커밋은 공통 속성을 라우트 구분값과 정확히 대응하는 뷰 모델 유니언으로 바꿔 렌더러가 다른 라우트 데이터나 전역 콘텐츠 참조 관계를 읽는 것을 타입 수준에서 금지합니다. 데이터 파생·참조 해석은 라우트·콘텐츠 처리 지점이 소유하고 렌더러는 화면 구성만 소유합니다.
- **실제 정적 검토 지점:** `src/designs/types.ts, src/designs/registry.tsx, src/lib/portfolio/view-models.ts, renderer entry files`에서 `PreparedDesignRouteProps`, 라우트 구분값별 뷰 모델 표현 방식, `footerLinks`와 해석된 참조를 부모와의 변경 차이 및 변경 후 파일 트리로 추적했습니다.
- **책임과 상태 전환:** 표현·조립 책임은 `src/designs/types.ts, src/designs/registry.tsx, src/lib/portfolio/view-models.ts, renderer entry files` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **실패·비보장:** 런타임 외부 JSON이 이미 검증됐다는 사실이나 CSS/DOM 품질을 이 타입만으로 보장하지는 않습니다.
- **후속 폐쇄:** `380b2a025070`이 모든 App Router 페이지를 이 계약의 단일 호출자로 만듭니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

```ts
// f8b0ab7b08aa · renderer contract
type PreparedDesignRouteProps =
  | { route: "home"; content: HomeViewModel; ... }
  | { route: "projects"; content: ProjectsViewModel; ... }
  | ...;
```
<!-- LEARNER-ANSWER:commit:f8b0ab7b08aa:END -->

### 14. `380b2a025070` — refactor(designs): 모든 route를 registry renderer로 위임

- **중요도:** S
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** 여덟 App Router 페이지와 다섯 디자인을 하나의 분기 설계로 폐쇄

#### 커밋별 확인 사항

- `380b2a025070^`와 `380b2a025070`를 비교하고 `src/app/page.tsx 외 7개 페이지, src/designs/registry.tsx, Design/Classic entry modules`에서 **페이지별 정확한 뷰 모델 생성 → `renderDesignRoute` → 등록부 로더·분기 함수; 직접 Design·Classic 가져오기 제거** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `8개 App Router 페이지와 5개 디자인을 하나의 분기 방식으로 통일` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `등록부가 반환한 DOM의 픽셀 단위 정확성과 클라이언트 상호작용은 별도 테스트에서 검증합니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `055b733cbb7e`가 8×5 조합표와 독립 식별 속성을 확인하며, 개발 과정 5의 분기 함수가 Design·Classic 브랜치를 제공합니다.
- App Router → 뷰 모델·콘텐츠 처리 지점 → 등록부·분기 함수 → 셸·view의 전체 호출 경로를 부모 커밋과 비교합니다.
- 호환 브랜치가 남아 있는지, 제거되는 시점은 언제인지, 이 불변 조건이 다른 네 개발 과정에 미치는 영향을 연결합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:380b2a025070:BEGIN -->
#### 복원 결과
- **직전 상태와 위험:** 동작 자체는 앞선 구현에 존재했지만 **페이지별 정확한 뷰 모델 생성 → `renderDesignRoute` → 등록부 로더·분기 함수; 직접 Design·Classic 가져오기 제거**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다. 이 분산 상태는 라우트·디자인마다 다른 파생·대체 처리·셸 처리를 허용해 같은 입력이 다른 실행 경로를 탈 위험이 있었습니다.
- **핵심 결정:** 이전에는 전용 세 디자인만 등록부를 쓰고 Design·Classic은 페이지별 직접 분기로 남아 있었다. 이 커밋은 각 페이지가 프레임워크 책임(콘텐츠 로딩, 활성화 여부, 매개변수, 메타데이터·notFound)을 처리한 뒤 정확한 라우트 뷰 모델을 만들고, 모든 디자인을 등록부 하나로 호출하게 합니다. 렌더러 선택과 화면 구성 담당이 App Router에서 제거됩니다.
- **실제 정적 검토 지점:** `src/app/page.tsx 외 7개 페이지, src/designs/registry.tsx, Design/Classic entry modules`에서 페이지별 정확한 뷰 모델 생성 → `renderDesignRoute` → 등록부 로더·분기 함수; 직접 Design·Classic 가져오기 제거를 부모와의 변경 차이 및 변경 후 파일 트리로 추적했습니다.
- **책임과 상태 전환:** 표현·조립 책임은 `src/app/page.tsx 외 7개 페이지, src/designs/registry.tsx, Design/Classic entry modules` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **실패·비보장:** 등록부가 반환한 DOM의 픽셀 수준의 정확성이나 클라이언트 상호작용은 별도 테스트 책임입니다.
- **후속 폐쇄:** `055b733cbb7e`가 8×5 조합표와 독립 식별 속성을 확인하며, 개발 과정 5의 분기 함수가 Design·Classic 브랜치를 제공합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

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

- **중요도:** A
- **태그:** ARCH, VALIDATION, RENDERER
- **개발 과정에서의 역할:** 8×5 라우트 조합표와 토큰·렌더러 식별 속성의 정적 회귀 규칙

#### 커밋별 확인 사항

- `055b733cbb7e^`와 `055b733cbb7e`를 비교하고 `src/designs/design-tokens.test.ts, src/designs/route-view-models.test.tsx`에서 **일곱 토큰 종류 문자열 검사, 여정·인터뷰 추가, Design·Classic `data-route-renderer` 단언문** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `8×5 라우트 조합표와 토큰·렌더러 식별 속성의 정적 회귀 규칙` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `정규식 기반 CSS 검사는 토큰 사용 여부·계산값·시각 대비를 증명하지 못하며, jsdom은 레이아웃을 계산하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `882a2f9d753e`가 실제 Chromium 화면 캡처 표시 영역을 추가합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:055b733cbb7e:BEGIN -->
- **직전 상태:** 관련 배포 환경 검사 범위는 존재했지만 **일곱 토큰 종류 문자열 검사, 여정·인터뷰 추가, Design·Classic `data-route-renderer` 단언문**을 실패 시 자동으로 탐지하는 회귀 증거가 없었습니다.
- **구현 결정:** 기존 여섯 라우트 조합표를 여정과 인터뷰 맵까지 늘려 40개 조합을 렌더링하고, Design·Classic이 공용 대체 처리가 아니라 독립 렌더러 최상위 요소를 냈는지 확인합니다. 별도 테스트는 globals.css에 일곱 토큰 종류가 있고 두 렌더러 범위에 콘텐츠 너비가 있는지 정적으로 검사합니다.
- **파일·심볼:** `src/designs/design-tokens.test.ts, src/designs/route-view-models.test.tsx`에서 일곱 토큰 종류 문자열 검사, 여정·인터뷰 추가, Design·Classic `data-route-renderer` 단언문을 확인했습니다.
- **소유권:** 실제 코드의 소유 주체는 변경하지 않고 `src/designs/design-tokens.test.ts, src/designs/route-view-models.test.tsx`가 회귀 판정과 테스트 입력 전제를 소유합니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `8×5 라우트 조합표와 토큰·렌더러 식별 속성의 정적 회귀 규칙`. 정규식 기반 CSS 검사는 토큰 소비·계산값·시각 대비를 증명하지 않고, jsdom은 레이아웃을 계산하지 않습니다.
- **역사적 연결:** `882a2f9d753e`가 실제 Chromium 화면 캡처 표시 영역을 추가합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:055b733cbb7e:END -->

### 16. `882a2f9d753e` — test(visual): 다섯 디자인 회귀 기준 추가

- **중요도:** A
- **태그:** TEST
- **개발 과정에서의 역할:** Playwright 화면 캡처 기준선과 스냅샷 목록 계약

#### 커밋별 확인 사항

- `882a2f9d753e^`와 `882a2f9d753e`를 비교하고 `tests/e2e/visual.spec.ts, src/designs/visual-regression-contract.test.ts, playwright.config.ts, tests/e2e/visual.spec.ts-snapshots/`에서 **`prepareStablePage`, 동작 줄이기 설정·networkidle·글꼴·이미지 대기, 15 PNG 명세 파일, `maxDiffPixelRatio: 0.01`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Playwright 화면 캡처 기준 이미지와 스냅샷 명세 파일 규칙` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `모든 라우트·프로젝트·접근성 상태·동적 상호작용을 포괄하지 않으며, 이 작업 환경에서는 실행하지 않았습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 이 테스트는 `380b2a025070` 이후 디자인 공통 화면 구성의 브라우저 회귀 방어다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:882a2f9d753e:BEGIN -->
- **직전 상태:** 관련 배포 환경 검사 범위는 존재했지만 **`prepareStablePage`, 동작 줄이기 설정·networkidle·글꼴·이미지 대기, 15 PNG 명세 파일, `maxDiffPixelRatio: 0.01`**를 실패 시 자동으로 탐지하는 회귀 증거가 없었습니다.
- **구현 결정:** 다섯 디자인의 홈을 Chromium 데스크톱·모바일로, 첫 활성화된 프로젝트 상세를 데스크톱 Chromium으로 캡처합니다. 안정화를 위해 동작 줄이기 설정을 강제하고 네트워크 idle·글꼴·이미지 완료를 기다린다. Vitest 명세 파일 테스트는 정확히 15개 기준선 이름이 존재하는지 확인합니다.
- **파일·심볼:** `tests/e2e/visual.spec.ts, src/designs/visual-regression-contract.test.ts, playwright.config.ts, tests/e2e/visual.spec.ts-snapshots/`에서 `prepareStablePage`, 동작 줄이기 설정·networkidle·글꼴·이미지 대기, 15 PNG 명세 파일, `maxDiffPixelRatio: 0.01`를 확인했습니다.
- **소유권:** 실제 코드의 소유 주체는 변경하지 않고 `tests/e2e/visual.spec.ts, src/designs/visual-regression-contract.test.ts, playwright.config.ts, tests/e2e/visual.spec.ts-snapshots/`가 회귀 판정과 테스트 입력 전제를 소유합니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `Playwright 화면 캡처 기준 이미지와 스냅샷 명세 파일 규칙`. 모든 라우트, 모든 프로젝트, 접근성, 동적 상호작용을 포괄하지 않으며 이 작업 환경에서는 실행하지 않았다.
- **역사적 연결:** 이 테스트는 `380b2a025070` 이후 디자인 공통 화면 구성의 브라우저 회귀 방어다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:882a2f9d753e:END -->

## 5. 불변 조건 변화

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:invariant:BEGIN -->
최종 불변 조건은 App Router가 프레임워크가 처리해야 할 작업과 정확한 라우트 뷰 모델을 소유하고, 등록부가 디자인을 선택하며, 각 렌더러는 종류가 구분된 준비된 입력만 받아 자체 셸·화면 구성을 반환한다는 것입니다.

- 도입·확장·폐쇄의 순서는 커밋 목록에 고정했습니다.
- 중요도 B 구축은 라우트·스타일 표시 영역을 단계적으로 넓히고, A·중요도 S 커밋은 소유 주체·분기·검증 불변 조건을 바꿉니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:invariant:END -->

## 6. 실패 → 수정 → 테스트와 소유 주체 변화

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:relations:BEGIN -->
`dc2cf72a768d`가 최초 위임 지점를 만들고 세 활성화가 전용 렌더러를 연결합니다. `8a48460df4c3`·`aef265b9bd01`이 마지막 원본 참조 결합을 제거한 뒤 `f8b0ab7b08aa`가 타입으로 제한하고 `380b2a025070`이 모든 라우트·디자인을 통일합니다. `1598a87702f6`·`055b733cbb7e`·`882a2f9d753e`가 정적 DOM, 토큰, 브라우저 화면 캡처의 서로 다른 층을 보호합니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:relations:END -->

## 7. 최종 구성과 실행 순서

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:flow:BEGIN -->
요청 라우트 → 콘텐츠 로딩·활성화 여부·매개변수 처리 → 라우트 뷰 모델 생성 → 템플릿 해석 → 등록부 로더 → 디자인 분기 함수 → 디자인이 소유한 셸·본문 → 정적·시각 규칙 검증 순서입니다.

각 단계에서 선택적·빈·누락된 참조가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 커밋의 보장하지 않는 범위에 남겼습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:flow:END -->

## 8. 실행 및 검증 근거

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:runtime:BEGIN -->
- **실행한 저장소 테스트·빌드:** 없음.
- **정적 확인:** 지정 브랜치의 커밋 분류, 커밋 bodies, 해당 커밋의 변경 내용과 변경 이력의 파일 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 컨테이너에서 직접 복제본 시 DNS가 `github.com`을 해석하지 못해 변경 이력의 worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next 빌드 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** 코드·테스트 구현의 존재와 범위는 정적 검토로 확인했고, 실행 성공·실패는 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:runtime:END -->

## 9. 학습 완료 확인

<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:checks:BEGIN -->
- [x] 모든 고정 SHA·제목·중요도·태그를 커밋 목록과 커밋 섹션에서 동일하게 유지했습니다.
- [x] 각 SHA에 구체적인 파일과 심볼·선택자·라우트 포커스를 기록했습니다.
- [x] 이전 상태, 소유 주체, 누락·대체 처리, 보장 범위와 보장하지 않는 범위, 후속 관계를 채웠습니다.
- [x] S/A/B 깊이를 구분했습니다.
- [x] 실행하지 않은 테스트를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:01-cross-design-token-shell-and-regression-contracts.md:checks:END -->
===== END FILE: 01-cross-design-token-shell-and-regression-contracts.md =====

===== BEGIN FILE: 02-editorial-design-system-construction.md =====
# 개발 과정: Editorial 디자인 구성

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `05-full-site-visual-systems`
>
> 1단계 검토에서 확정한 기준 문서 틀입니다. 2단계에서는 답변 식별 속성 내부만 채웁니다.

## 0. 범위와 기준

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md`와 해당 커밋의 정확한 메타데이터를 기준으로 고정했습니다.
- **개발 과정 범위:** Editorial 스타일시트와 `editorial-route.tsx` 내부 구축만 포함합니다. 등록부 활성화는 개발 과정 1에 둡니다. C 수준 서식만 변경한 미디어 규칙 정리는 제외했습니다.
- 다른 브랜치, 최종 HEAD의 후대 구현, 실행하지 않은 명령 결과를 사용하지 않습니다.

## 1. 개발 과정 목표

Editorial의 범위가 제한된 스타일시트가 사이트 전체 지면 문법을 만들고, 하나의 모듈이 여덟 라우트와 셸·링크·빈 상태·참조 해석을 점진적으로 완성하는 과정을 복원합니다.

### 고정된 불변 조건

최종 불변 조건은 `EditorialRoute` 하나만 외부에 노출되고, 라우트 구분값이 여덟 내부 view 중 하나를 선택한 뒤 항상 `EditorialShell`로 감싸며, 누락·실패를 라우트별 명시적 UI로 표현한다는 것입니다.

## 2. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `7546ac248334` | style(editorial): 지면과 masthead 토큰 구성 | B | RENDERER | Editorial 시각 표현 규칙의 최상위 토큰·종이 질감·포커스·건너뛰기 링크·상단 제목 영역 기초 |
| 2 | `80b86ed4a1ff` | style(editorial): wordmark와 navigation 계층 구성 | B | ROUTING, RENDERER | Editorial 시각 표현 규칙의 wordmark·데스크톱 탐색·디자인 선택기·푸터 행동 유도 링크 |
| 3 | `4d646fb2924a` | style(editorial): footer와 hero 활자 체계 구성 | B | RENDERER | Editorial 시각 표현 규칙의 푸터와 홈 소개 영역 타이포그래피 |
| 4 | `6434531645b7` | style(editorial): hero spread 레이아웃 구성 | B | RENDERER | Editorial 시각 표현 규칙의 12열 소개 영역 펼치기와 공통 섹션 rhythm |
| 5 | `a97066f07dfd` | style(editorial): lead story와 매체 표현 구성 | B | RENDERER | Editorial 시각 표현 규칙의 대표 사례 설명과 프레임 이미지 |
| 6 | `a9674cf2fa94` | style(editorial): 이미지 프레임과 feature 열 구성 | B | RENDERER | Editorial 시각 표현 규칙의 이미지 자리표시자·프로젝트 목록 행·분리 기능 열 |
| 7 | `4708ca281c16` | style(editorial): 원칙 목록과 contact strip 구성 | B | RENDERER | Editorial 시각 표현 규칙의 원칙 카드·옆 영역 기능·태그·연락처 가로 영역 |
| 8 | `ebe23d211852` | style(editorial): contact와 archive 지면 구성 | B | RENDERER | Editorial 시각 표현 규칙의 연락처 가로 영역 완성과 페이지·아카이브 레이아웃 |
| 9 | `931226268687` | style(editorial): archive group과 case link 구성 | B | RENDERER | Editorial 시각 표현 규칙의 아카이브 그룹과 프로젝트 사례 도입부 |
| 10 | `2cc074f728cb` | style(editorial): case link와 dark section 구성 | B | RENDERER | Editorial 시각 표현 규칙의 테두리가 있는 링크·표지·3열 설명·어두운 설계 |
| 11 | `34a9c958801c` | style(editorial): dark section과 decision 열 구성 | B | RENDERER | Editorial 시각 표현 규칙의 설계 근거·이미지 쌍·결정 열 |
| 12 | `13f49ab0c1f7` | style(editorial): 결과 spread와 profile facts 구성 | B | RENDERER | Editorial 시각 표현 규칙의 결과·종료 탐색·누락된 프로젝트·프로필 소개 영역 |
| 13 | `124f6a6fec62` | style(editorial): profile summary와 skill group 구성 | B | RENDERER | Editorial 시각 표현 규칙의 프로필 사진·원칙 그리드·기술 목록 펼치기 |
| 14 | `c28bb0a5eb01` | style(editorial): 기술 그룹과 curation 본문 구성 | B | RENDERER | Editorial 시각 표현 규칙의 기술·경력 행과 선별 기록 펼치기 |
| 15 | `4ce0333849cc` | style(editorial): curation panel과 프로젝트 목록 구성 | B | RENDERER | Editorial 시각 표현 규칙의 기준·분류·제외 패널s와 프로젝트 링크 |
| 16 | `586626a79cb1` | style(editorial): curation link와 resume 도입부 구성 | B | RENDERER | Editorial 시각 표현 규칙의 터치 영역·다음 검토 패널·이력서 헤더·본문 |
| 17 | `21d63d1975b3` | style(editorial): resume identity와 프로젝트 행 구성 | B | RENDERER | Editorial 시각 표현 규칙의 이력서 정의 목록·번호가 있는 섹션·프로젝트·교육 행 |
| 18 | `543f4b1062e3` | style(editorial): resume 사례와 contact 본문 구성 | B | RENDERER | Editorial 시각 표현 규칙의 이력서 프로젝트 사례 링크·연락처 소개 영역·사용 가능 여부·연락 채널·안내 |
| 19 | `e988e97415af` | style(editorial): contact note와 milestone link 구성 | B | RENDERER | Editorial 시각 표현 규칙의 연락처 안내와 여정 주요 시점 날짜·변경 과정 |
| 20 | `1da39994d9e3` | style(editorial): milestone과 현재 방향 지면 구성 | B | RENDERER | Editorial 시각 표현 규칙의 보조 타임라인과 현재 위치 패널 |
| 21 | `0c3ba4ca1d48` | style(editorial): 현재 방향과 interview track 구성 | B | RENDERER | Editorial 시각 표현 규칙의 현재 위치 타이포그래피와 고정 horizontal 장 탐색 |
| 22 | `af5688dd1c3a` | style(editorial): interview 답변과 근거 표현 구성 | B | RENDERER | Editorial 시각 표현 규칙의 질문·근거 쌍으로 배치된 목록 |
| 23 | `0c7b77c2528a` | style(editorial): 공백 목록과 중형 화면 경계 구성 | B | RENDERER | Editorial 시각 표현 규칙의 해석되지 않은 부족한 부분 펼치기와 1180px 화면 너비 대응 |
| 24 | `a854cb45cc22` | style(editorial): tablet masthead와 hero 재배치 | B | RENDERER | Editorial 시각 표현 규칙의 태블릿 브라우저 기본 펼침 요소와 12→8-열 소개 영역 |
| 25 | `3f82e8a7c308` | style(editorial): tablet route 지면 재배치 | B | ROUTING, RENDERER | Editorial 시각 표현 규칙의 라우트 펼치기의 태블릿 읽기 순서 |
| 26 | `10a442435e1a` | style(editorial): tablet 세부 간격 정리 | B | RENDERER | Editorial 시각 표현 규칙의 태블릿 여정 도입부 읽기 폭 |
| 27 | `afaf24796399` | style(editorial): mobile navigation과 hero 구성 | B | ROUTING, RENDERER | Editorial 시각 표현 규칙의 모바일 상단 제목 영역 메타데이터·세로로 쌓이는 그리드·선형 소개 영역 |
| 28 | `499c0e660caf` | style(editorial): mobile 본문과 표 구성 | B | RENDERER | Editorial 시각 표현 규칙의 모바일 프로젝트 사례·프로필·이력서·주요 시점·선별 기록·인터뷰 재배치 |
| 29 | `f7a81e0fe1d3` | style(editorial): mobile footer와 동작 감소 구성 | B | RENDERER, A11Y | Editorial 시각 표현 규칙의 작은 화면 푸터 간격과 동작 줄이기 설정 |
| 30 | `1c55d7422273` | feat(editorial): route 계약과 navigation helper 추가 | B | ROUTING, RENDERER | Editorial 라우트 구분 지점 |
| 31 | `e078d79d24c8` | feat(editorial): debug note와 이미지 프레임 추가 | B | RENDERER | 재사용 화면 구성 공용 컴포넌트 |
| 32 | `1b353fe5ba7b` | feat(editorial): 콘텐츠 링크와 방향 표식 추가 | B | CONTENT, RENDERER | 콘텐츠를 고려하는 링크 공용 컴포넌트 |
| 33 | `794615a037d3` | feat(editorial): masthead와 footer shell 추가 | B | ROUTING, RENDERER | 공용 `EditorialShell` |
| 34 | `b7fd9118025e` | feat(editorial): 섹션 표식과 프로젝트 인덱스 추가 | B | RENDERER | 섹션·프로젝트 목록 공용 컴포넌트 |
| 35 | `5c82371743ba` | feat(editorial): 홈 hero spread 추가 | B | RENDERER | 홈 라우트 시작 |
| 36 | `96ba59901181` | feat(editorial): 홈 lead story 추가 | B | RENDERER | 홈 대표 프로젝트 |
| 37 | `4c8270522400` | feat(editorial): 홈 대표 프로젝트 목록 추가 | B | RENDERER | 홈 남은 대표 목록 |
| 38 | `983131c5a266` | feat(editorial): 홈 원칙과 기술 sidebar 추가 | B | RENDERER | 홈 원칙·시스템 섹션 |
| 39 | `f01b60fc368e` | feat(editorial): 홈 contact strip 추가 | B | RENDERER | 홈 연락처 행동 유도 링크 |
| 40 | `4e69ba2ee361` | feat(editorial): 프로젝트 archive route 추가 | B | ROUTING, RENDERER | 프로젝트 아카이브 |
| 41 | `c722cdd08ef8` | feat(editorial): 프로젝트 상세 서사와 구조 추가 | B | RENDERER | 프로젝트 상세 첫 번째 전체 경로 |
| 42 | `f38556a17e8b` | feat(editorial): 프로젝트 증거와 결과 spread 추가 | B | RENDERER | 프로젝트 상세 완료 |
| 43 | `cc1b2233287f` | feat(editorial): About 정체성과 원칙 소개 추가 | B | RENDERER | 소개 식별 정보 |
| 44 | `5f0193979568` | feat(editorial): About 기술과 경력 소개 추가 | B | RENDERER | 소개 기술 목록·경력 |
| 45 | `5c95665ca9d2` | feat(editorial): About 큐레이션 기준 추가 | B | CONTENT, RENDERER | 기능 플래그로 제어되는 선별 기록 시작 |
| 46 | `4a7c3a3c9cde` | feat(editorial): About 큐레이션 범주 추가 | B | CONTENT, RENDERER | 선별 기록 분류 프로젝트 결합 |
| 47 | `c0d0004e9355` | feat(editorial): About 큐레이션 공백과 재검토 추가 | B | CONTENT, RENDERER | 선별 기록 완료 |
| 48 | `119d19ab41b1` | feat(editorial): Resume 정체성과 프로젝트 경력 추가 | B | RENDERER | 이력서 시작 |
| 49 | `4df2710fa7f9` | feat(editorial): Resume 경력과 교육 기록 추가 | B | RENDERER | 이력서 완료 |
| 50 | `61d6952850cd` | feat(editorial): Contact desk route 추가 | B | ROUTING, RENDERER | 연락처 라우트 |
| 51 | `08fa527b9b65` | feat(editorial): Journey milestone spread 추가 | B | RENDERER | 여정 설명 시작 |
| 52 | `96b66af4d5a7` | feat(editorial): Journey timeline과 현재 방향 추가 | B | RENDERER | 여정 완료 |
| 53 | `5e2f37861d3d` | feat(editorial): Interview Map 소개와 chapter 추가 | B | RENDERER | 인터뷰 시작 |
| 54 | `94deba32f56a` | feat(editorial): Interview 답변 근거와 공백 추가 | B | RENDERER | 인터뷰 완료 |
| 55 | `46e23d922c2e` | feat(editorial): route dispatcher 추가 | A | ARCH, ROUTING, RENDERER | Editorial 여덟 라우트와 공용 셸을 하나의 공개 진입점으로 폐쇄 |

## 3. 변경 전 기준

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:baseline:BEGIN -->
- **직전 상태:** Editorial 선택 ID와 공통 라우트 위임은 존재했지만 전용 CSS, 셸, 라우트 view가 없었습니다. 초기 라우트 구현은 원본 콘텐츠에서 그룹화·지표·프로젝트 참조를 직접 파생했습니다.
- **경계 판단:** Editorial 스타일시트와 `editorial-route.tsx` 내부 구축만 포함합니다. 등록부 활성화는 개발 과정 1에 둡니다. C 수준 서식만 변경한 미디어 규칙 정리는 제외했습니다.
- **복원 기준:** 각 커밋의 부모 커밋과 해당 SHA 파일 트리만 사용하고 최종 HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:baseline:END -->

## 4. 커밋별 복원 기록

### 1. `7546ac248334` — style(editorial): 지면과 masthead 토큰 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 최상위 토큰·종이 질감·포커스·건너뛰기 링크·상단 제목 영역 기초

#### 커밋별 확인 사항

- `7546ac248334^`와 `7546ac248334`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **최상위 토큰·종이 질감·포커스·건너뛰기 링크·상단 제목 영역 기초에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 최상위 토큰·종이 질감·포커스·본문 건너뛰기 링크·상단 제목 영역 기초` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:7546ac248334:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **최상위 토큰·종이 질감·포커스·건너뛰기 링크·상단 제목 영역 기초에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 범위가 제한된 종이·잉크 색상표, 박스 기본값 초기화, 포커스가 표시되는 및 상단 제목 영역 배치를 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 최상위 토큰·종이 질감·포커스·건너뛰기 링크·상단 제목 영역 기초에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:7546ac248334:END -->

### 2. `80b86ed4a1ff` — style(editorial): wordmark와 navigation 계층 구성

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 wordmark·데스크톱 탐색·디자인 선택기·푸터 행동 유도 링크

#### 커밋별 확인 사항

- `80b86ed4a1ff^`와 `80b86ed4a1ff`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **wordmark·데스크톱 탐색·디자인 선택기·푸터 행동 유도 링크에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 워드마크·데스크톱 탐색·디자인 선택기·푸터 행동 유도 링크` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:80b86ed4a1ff:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **wordmark·데스크톱 탐색·디자인 선택기·푸터 행동 유도 링크에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 셸의 상단·하단 배치 순서와 라우트 탐색 자리를 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 wordmark·데스크톱 탐색·디자인 선택기·푸터 행동 유도 링크에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:80b86ed4a1ff:END -->

### 3. `4d646fb2924a` — style(editorial): footer와 hero 활자 체계 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 푸터와 홈 소개 영역 타이포그래피

#### 커밋별 확인 사항

- `4d646fb2924a^`와 `4d646fb2924a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **푸터와 홈 소개 영역 타이포그래피에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 푸터와 홈 소개 영역의 글꼴 표현` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4d646fb2924a:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **푸터와 홈 소개 영역 타이포그래피에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 제목·본문 글자 크기 및 소개 영역 상단 구조를 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 푸터와 홈 소개 영역 타이포그래피에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4d646fb2924a:END -->

### 4. `6434531645b7` — style(editorial): hero spread 레이아웃 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 12열 소개 영역 펼치기와 공통 섹션 rhythm

#### 커밋별 확인 사항

- `6434531645b7^`와 `6434531645b7`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **12열 소개 영역 펼치기와 공통 섹션 rhythm에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 12열 소개 영역과 공통 섹션 간격` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:6434531645b7:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **12열 소개 영역 펼치기와 공통 섹션 rhythm에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 소개 영역 하단·지표·미디어 배치와 펼치기 간격을 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 12열 소개 영역 펼치기와 공통 섹션 rhythm에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:6434531645b7:END -->

### 5. `a97066f07dfd` — style(editorial): lead story와 매체 표현 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 대표 사례 설명과 프레임 이미지

#### 커밋별 확인 사항

- `a97066f07dfd^`와 `a97066f07dfd`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **대표 사례 설명과 프레임 이미지에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 대표 사례 설명과 프레임 이미지` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:a97066f07dfd:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **대표 사례 설명과 프레임 이미지에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 대표 프로젝트 설명과 이미지 프레임의 잘라내기·자리표시자 경계를 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 대표 사례 설명과 프레임 이미지에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:a97066f07dfd:END -->

### 6. `a9674cf2fa94` — style(editorial): 이미지 프레임과 feature 열 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 이미지 자리표시자·프로젝트 목록 행·분리 기능 열

#### 커밋별 확인 사항

- `a9674cf2fa94^`와 `a9674cf2fa94`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **이미지 자리표시자·프로젝트 목록 행·분리 기능 열에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 이미지 자리표시자·프로젝트 목록 행·두 열 기능 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:a9674cf2fa94:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **이미지 자리표시자·프로젝트 목록 행·분리 기능 열에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 반복 가능한 미디어·목록·기능 배치 단위를 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 이미지 자리표시자·프로젝트 목록 행·분리 기능 열에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:a9674cf2fa94:END -->

### 7. `4708ca281c16` — style(editorial): 원칙 목록과 contact strip 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 원칙 카드·옆 영역 기능·태그·연락처 가로 영역

#### 커밋별 확인 사항

- `4708ca281c16^`와 `4708ca281c16`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **원칙 카드·옆 영역 기능·태그·연락처 가로 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 원칙 카드·사이드 영역·태그·연락처 가로 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4708ca281c16:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **원칙 카드·옆 영역 기능·태그·연락처 가로 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 홈·소개·연락처가 공유할 카드와 행동 유도 링크 문법을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 원칙 카드·옆 영역 기능·태그·연락처 가로 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4708ca281c16:END -->

### 8. `ebe23d211852` — style(editorial): contact와 archive 지면 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 연락처 가로 영역 완성과 페이지·아카이브 레이아웃

#### 커밋별 확인 사항

- `ebe23d211852^`와 `ebe23d211852`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **연락처 가로 영역 완성과 페이지·아카이브 레이아웃에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 연락처 가로 영역 완성과 페이지·아카이브 레이아웃` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:ebe23d211852:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **연락처 가로 영역 완성과 페이지·아카이브 레이아웃에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 공통 페이지 소개 영역과 아카이브 목록을 위한 공용 배치 규칙을 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 연락처 가로 영역 완성과 페이지·아카이브 레이아웃에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:ebe23d211852:END -->

### 9. `931226268687` — style(editorial): archive group과 case link 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 아카이브 그룹과 프로젝트 사례 도입부

#### 커밋별 확인 사항

- `931226268687^`와 `931226268687`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **아카이브 그룹과 프로젝트 사례 도입부에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 아카이브 그룹과 프로젝트 사례 도입부` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:931226268687:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **아카이브 그룹과 프로젝트 사례 도입부에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 분류 그룹, 프로젝트 사례 링크, 상세 도입부의 구조를 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 아카이브 그룹과 프로젝트 사례 도입부에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:931226268687:END -->

### 10. `2cc074f728cb` — style(editorial): case link와 dark section 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 테두리가 있는 링크·표지·3열 설명·어두운 설계

#### 커밋별 확인 사항

- `2cc074f728cb^`와 `2cc074f728cb`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **테두리가 있는 링크·표지·3열 설명·어두운 설계에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 테두리 링크·표지·3열 설명 영역·어두운 설계 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:2cc074f728cb:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **테두리가 있는 링크·표지·3열 설명·어두운 설계에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 상세 라우트의 링크·표지·문제·해결책·설계 대비를 확장합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 테두리가 있는 링크·표지·3열 설명·어두운 설계에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:2cc074f728cb:END -->

### 11. `34a9c958801c` — style(editorial): dark section과 decision 열 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 설계 근거·이미지 쌍·결정 열

#### 커밋별 확인 사항

- `34a9c958801c^`와 `34a9c958801c`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **설계 근거·이미지 쌍·결정 열에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 설계 근거·이미지 쌍·결정 사항 열` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:34a9c958801c:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **설계 근거·이미지 쌍·결정 열에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 어두운 근거 섹션과 의사결정·상충 열을 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 설계 근거·이미지 쌍·결정 열에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:34a9c958801c:END -->

### 12. `13f49ab0c1f7` — style(editorial): 결과 spread와 profile facts 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 결과·종료 탐색·누락된 프로젝트·프로필 소개 영역

#### 커밋별 확인 사항

- `13f49ab0c1f7^`와 `13f49ab0c1f7`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **결과·종료 탐색·누락된 프로젝트·프로필 소개 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 결과·다음 이동 링크·프로젝트 없음 화면·프로필 소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:13f49ab0c1f7:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **결과·종료 탐색·누락된 프로젝트·프로필 소개 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 상세 종료·복구 상태와 소개 식별 정보 지면을 함께 엽니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 결과·종료 탐색·누락된 프로젝트·프로필 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:13f49ab0c1f7:END -->

### 13. `124f6a6fec62` — style(editorial): profile summary와 skill group 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 프로필 사진·원칙 그리드·기술 목록 펼치기

#### 커밋별 확인 사항

- `124f6a6fec62^`와 `124f6a6fec62`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **프로필 사진·원칙 그리드·기술 목록 펼치기에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 프로필 사진·원칙 그리드·기술 목록 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:124f6a6fec62:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **프로필 사진·원칙 그리드·기술 목록 펼치기에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 소개의 프로필 사진, 원칙, 기술 그룹 배치 순서를 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 프로필 사진·원칙 그리드·기술 목록 펼치기에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:124f6a6fec62:END -->

### 14. `c28bb0a5eb01` — style(editorial): 기술 그룹과 curation 본문 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 기술·경력 행과 선별 기록 펼치기

#### 커밋별 확인 사항

- `c28bb0a5eb01^`와 `c28bb0a5eb01`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **기술·경력 행과 선별 기록 펼치기에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 기술·경력 행과 선별 기록 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:c28bb0a5eb01:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **기술·경력 행과 선별 기록 펼치기에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 반복 행과 비대칭 선별 기록 본문을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 기술·경력 행과 선별 기록 펼치기에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:c28bb0a5eb01:END -->

### 15. `4ce0333849cc` — style(editorial): curation panel과 프로젝트 목록 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 기준·분류·제외 패널s와 프로젝트 링크

#### 커밋별 확인 사항

- `4ce0333849cc^`와 `4ce0333849cc`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **기준·분류·제외 패널s와 프로젝트 링크에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 선정 기준·분류·제외 항목 패널과 프로젝트 링크` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4ce0333849cc:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **기준·분류·제외 패널s와 프로젝트 링크에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 선별 기록 아카이브의 번호·그리드·카드·링크 구조를 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 기준·분류·제외 패널s와 프로젝트 링크에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4ce0333849cc:END -->

### 16. `586626a79cb1` — style(editorial): curation link와 resume 도입부 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 터치 영역·다음 검토 패널·이력서 헤더·본문

#### 커밋별 확인 사항

- `586626a79cb1^`와 `586626a79cb1`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **터치 영역·다음 검토 패널·이력서 헤더·본문에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 터치 영역·다음 검토 패널·이력서 머리말·본문` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:586626a79cb1:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **터치 영역·다음 검토 패널·이력서 헤더·본문에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 선별 기록 종료와 이력서 2열 시작을 연결합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 터치 영역·다음 검토 패널·이력서 헤더·본문에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:586626a79cb1:END -->

### 17. `21d63d1975b3` — style(editorial): resume identity와 프로젝트 행 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 이력서 정의 목록·번호가 있는 섹션·프로젝트·교육 행

#### 커밋별 확인 사항

- `21d63d1975b3^`와 `21d63d1975b3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **이력서 정의 목록·번호가 있는 섹션·프로젝트·교육 행에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 이력서 정의 목록·번호가 있는 섹션·프로젝트·교육 행` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:21d63d1975b3:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **이력서 정의 목록·번호가 있는 섹션·프로젝트·교육 행에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 이력 정보 배치 순서와 반복 행을 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 이력서 정의 목록·번호가 있는 섹션·프로젝트·교육 행에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:21d63d1975b3:END -->

### 18. `543f4b1062e3` — style(editorial): resume 사례와 contact 본문 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 이력서 프로젝트 사례 링크·연락처 소개 영역·사용 가능 여부·연락 채널·안내

#### 커밋별 확인 사항

- `543f4b1062e3^`와 `543f4b1062e3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **이력서 프로젝트 사례 링크·연락처 소개 영역·사용 가능 여부·연락 채널·안내에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 이력서 프로젝트 사례 링크·연락처 소개·응답 가능 상태·연락 채널·안내` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:543f4b1062e3:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **이력서 프로젝트 사례 링크·연락처 소개 영역·사용 가능 여부·연락 채널·안내에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 이력서에서 연락처로 이어지는 라우트 전용 지면을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 이력서 프로젝트 사례 링크·연락처 소개 영역·사용 가능 여부·연락 채널·안내에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:543f4b1062e3:END -->

### 19. `e988e97415af` — style(editorial): contact note와 milestone link 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 연락처 안내와 여정 주요 시점 날짜·변경 과정

#### 커밋별 확인 사항

- `e988e97415af^`와 `e988e97415af`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **연락처 안내와 여정 주요 시점 날짜·변경 과정에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 연락 안내와 여정 주요 시점의 날짜·설명` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:e988e97415af:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **연락처 안내와 여정 주요 시점 날짜·변경 과정에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 지원 안내 목록과 여정 주요 시점 펼치기를 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 연락처 안내와 여정 주요 시점 날짜·변경 과정에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:e988e97415af:END -->

### 20. `1da39994d9e3` — style(editorial): milestone과 현재 방향 지면 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 보조 타임라인과 현재 위치 패널

#### 커밋별 확인 사항

- `1da39994d9e3^`와 `1da39994d9e3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **보조 타임라인과 현재 위치 패널에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 보조 타임라인과 현재 위치 패널` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1da39994d9e3:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **보조 타임라인과 현재 위치 패널에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 여정의 상세 주요 시점과 더 넓은 아카이브·현재 상태를 시각적으로 분리합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 보조 타임라인과 현재 위치 패널에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1da39994d9e3:END -->

### 21. `0c3ba4ca1d48` — style(editorial): 현재 방향과 interview track 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 현재 위치 타이포그래피와 고정 horizontal 장 탐색

#### 커밋별 확인 사항

- `0c3ba4ca1d48^`와 `0c3ba4ca1d48`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **현재 위치 타이포그래피와 고정 horizontal 장 탐색에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 현재 위치 글꼴 표현과 고정 가로 장 탐색` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:0c3ba4ca1d48:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **현재 위치 타이포그래피와 고정 horizontal 장 탐색에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 인터뷰 맵의 페이지 내부 탐색과 현재 방향 표현을 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 현재 위치 타이포그래피와 고정 horizontal 장 탐색에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:0c3ba4ca1d48:END -->

### 22. `af5688dd1c3a` — style(editorial): interview 답변과 근거 표현 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 질문·근거 쌍으로 배치된 목록

#### 커밋별 확인 사항

- `af5688dd1c3a^`와 `af5688dd1c3a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **질문·근거 쌍으로 배치된 목록에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 질문·근거 쌍 목록` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:af5688dd1c3a:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **질문·근거 쌍으로 배치된 목록에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 질문·프로젝트 답변·깊이를 두 열 목록으로 배치합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 질문·근거 쌍으로 배치된 목록에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:af5688dd1c3a:END -->

### 23. `0c7b77c2528a` — style(editorial): 공백 목록과 중형 화면 경계 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 해석되지 않은 부족한 부분 펼치기와 1180px 화면 너비 대응

#### 커밋별 확인 사항

- `0c7b77c2528a^`와 `0c7b77c2528a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **해석되지 않은 부족한 부분 펼치기와 1180px 화면 너비 대응에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 미해결 항목 영역과 1180px 화면 너비 대응` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:0c7b77c2528a:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **해석되지 않은 부족한 부분 펼치기와 1180px 화면 너비 대응에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 근거 공백을 어두운 펼치기로 만들고 첫 중형 폭 조정을 시작합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 해석되지 않은 부족한 부분 펼치기와 1180px 화면 너비 대응에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:0c7b77c2528a:END -->

### 24. `a854cb45cc22` — style(editorial): tablet masthead와 hero 재배치

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 태블릿 브라우저 기본 펼침 요소와 12→8-열 소개 영역

#### 커밋별 확인 사항

- `a854cb45cc22^`와 `a854cb45cc22`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **태블릿 브라우저 기본 펼침 요소와 12→8-열 소개 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 태블릿용 브라우저 기본 펼침 메뉴와 12열에서 8열로 줄어드는 소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:a854cb45cc22:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **태블릿 브라우저 기본 펼침 요소와 12→8-열 소개 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 데스크톱 탐색을 `<details>` 메뉴로 바꾸고 소개 영역 그리드를 축소합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 태블릿 브라우저 기본 펼침 요소와 12→8-열 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:a854cb45cc22:END -->

### 25. `3f82e8a7c308` — style(editorial): tablet route 지면 재배치

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 라우트 펼치기의 태블릿 읽기 순서

#### 커밋별 확인 사항

- `3f82e8a7c308^`와 `3f82e8a7c308`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **라우트 펼치기의 태블릿 읽기 순서에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 라우트 본문의 태블릿 읽기 순서` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:3f82e8a7c308:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **라우트 펼치기의 태블릿 읽기 순서에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 상세·소개·이력서·여정·인터뷰 주요 그리드를 단일 열 중심으로 재배치합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 라우트 펼치기의 태블릿 읽기 순서에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:3f82e8a7c308:END -->

### 26. `10a442435e1a` — style(editorial): tablet 세부 간격 정리

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 태블릿 여정 도입부 읽기 폭

#### 커밋별 확인 사항

- `10a442435e1a^`와 `10a442435e1a`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **태블릿 여정 도입부 읽기 폭에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 태블릿 여정 도입부의 읽기 폭` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:10a442435e1a:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **태블릿 여정 도입부 읽기 폭에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 타임라인 도입 열 폭을 제한합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 태블릿 여정 도입부 읽기 폭에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:10a442435e1a:END -->

### 27. `afaf24796399` — style(editorial): mobile navigation과 hero 구성

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 모바일 상단 제목 영역 메타데이터·세로로 쌓이는 그리드·선형 소개 영역

#### 커밋별 확인 사항

- `afaf24796399^`와 `afaf24796399`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **모바일 상단 제목 영역 메타데이터, 세로로 쌓이는 그리드, 선형 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가 및 기존 선언의 이동** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 모바일 상단 제목 영역 메타데이터·세로로 쌓이는 그리드·한 열 소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:afaf24796399:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **부모 커밋과 비교해 모바일 상단 제목 영역 메타데이터, 세로로 쌓이는 그리드, 선형 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가 및 기존 선언의 이동**를 완결하는 선택자·미디어 규칙 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 첫 모바일 화면 너비 기준에서 탐색과 주요 라우트 그리드를 선형화합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모 커밋과 비교해 모바일 상단 제목 영역 메타데이터, 세로로 쌓이는 그리드, 선형 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가 및 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:afaf24796399:END -->

### 28. `499c0e660caf` — style(editorial): mobile 본문과 표 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 모바일 프로젝트 사례·프로필·이력서·주요 시점·선별 기록·인터뷰 재배치

#### 커밋별 확인 사항

- `499c0e660caf^`와 `499c0e660caf`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **모바일 프로젝트 사례·프로필·이력서·주요 시점·선별 기록·인터뷰 재배치에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 모바일 프로젝트 사례·프로필·이력서·주요 시점·선별 기록·인터뷰 재배치` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:499c0e660caf:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **모바일 프로젝트 사례·프로필·이력서·주요 시점·선별 기록·인터뷰 재배치에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 나머지 장문 본문·핵심 정보·표 형태를 좁은 화면 읽기 순서로 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 모바일 프로젝트 사례·프로필·이력서·주요 시점·선별 기록·인터뷰 재배치에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:499c0e660caf:END -->

### 29. `f7a81e0fe1d3` — style(editorial): mobile footer와 동작 감소 구성

- **중요도:** B
- **태그:** RENDERER, A11Y
- **개발 과정에서의 역할:** Editorial 시각 표현 규칙의 작은 화면 푸터 간격과 동작 줄이기 설정

#### 커밋별 확인 사항

- `f7a81e0fe1d3^`와 `f7a81e0fe1d3`를 비교하고 `src/designs/editorial/editorial.module.css`에서 **작은 화면 푸터 간격과 동작 줄이기 설정에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 시각 표현 규칙의 작은 화면 푸터 간격과 동작 줄이기` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `동작 줄이기·포커스·의미 구조 보완은 CSS와 DOM 규칙의 일부만 다룹니다. 실제 WCAG 적합성이나 모든 보조기기에서의 동작까지 단독으로 보장하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f7a81e0fe1d3:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **작은 화면 푸터 간격과 동작 줄이기 설정에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/editorial/editorial.module.css`에 추가된 CSS가 최소 화면 간격을 마감하고 애니메이션·전환 효과를 줄이는 미디어 규칙을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Editorial 스타일시트에 남습니다.
- **정적 검토:** `src/designs/editorial/editorial.module.css`의 부모와의 변경 차이에서 작은 화면 푸터 간격과 동작 줄이기 설정에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/editorial/editorial.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 동작 줄이기 설정·포커스·의미상 보조는 CSS·DOM 규칙의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:f7a81e0fe1d3:END -->

### 30. `1c55d7422273` — feat(editorial): route 계약과 navigation helper 추가

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** Editorial 라우트 구분 지점

#### 커밋별 확인 사항

- `1c55d7422273^`와 `1c55d7422273`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`EditorialRouteName`, 공통 속성, `editorialHref`와 내부·외부 라우트 판정** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial 라우트 진입점` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `794615a037d3` 셸과 최종 `46e23d922c2e` 분기 함수가 이 계약을 소비합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1c55d7422273:BEGIN -->
- **직전 상태:** 직전 상태에는 **`EditorialRouteName`, 공통 속성, `editorialHref`와 내부·외부 라우트 판정** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 여덟 라우트 이름을 닫힌 유니언으로 두고 콘텐츠·프로젝트/`currentPath`/디버그를 전달하는 모듈 계약을 만듭니다. 링크 도우미 함수는 선택한 디자인과 디버그 쿼리를 내부 경로에 보존합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `EditorialRouteName`, 공통 속성, `editorialHref`와 내부·외부 라우트 판정을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** `794615a037d3` 셸과 최종 `46e23d922c2e` 분기 함수가 이 계약을 소비합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1c55d7422273:END -->

### 31. `e078d79d24c8` — feat(editorial): debug note와 이미지 프레임 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 재사용 화면 구성 공용 컴포넌트

#### 커밋별 확인 사항

- `e078d79d24c8^`와 `e078d79d24c8`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`DebugNote`, 의미상 이미지·자리표시자 프레임** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `재사용 가능한 화면 구성 요소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:e078d79d24c8:BEGIN -->
- **직전 상태:** 직전 상태에는 **`DebugNote`, 의미상 이미지·자리표시자 프레임** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 디버그 모드일 때만 원본 hint를 노출하고 프로젝트 이미지 유무에 따라 의미상 미디어 또는 자리표시자를 반환하는 두 공용 컴포넌트를 추가합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `DebugNote`, 의미상 이미지·자리표시자 프레임를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:e078d79d24c8:END -->

### 32. `1b353fe5ba7b` — feat(editorial): 콘텐츠 링크와 방향 표식 추가

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** 콘텐츠를 고려하는 링크 공용 컴포넌트

#### 커밋별 확인 사항

- `1b353fe5ba7b^`와 `1b353fe5ba7b`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`EditorialContentLink`, 내부 `Link`와 외부·mailto 기준 분기** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `콘텐츠 정보를 반영하는 링크 요소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1b353fe5ba7b:BEGIN -->
- **직전 상태:** 직전 상태에는 **`EditorialContentLink`, 내부 `Link`와 외부·mailto 기준 분기** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 내부 경로는 템플릿·디버그 상태를 보존하고 외부 목적지는 일반 기준 속성을 사용합니다. 라우트 조립이 링크 종류를 매번 재판정하지 않게 됩니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `EditorialContentLink`, 내부 `Link`와 외부·mailto 기준 분기를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1b353fe5ba7b:END -->

### 33. `794615a037d3` — feat(editorial): masthead와 footer shell 추가

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** 공용 `EditorialShell`

#### 커밋별 확인 사항

- `794615a037d3^`와 `794615a037d3`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **데스크톱·모바일 탐색, 디자인 선택기, 주요 콘텐츠 영역, `footerLinks`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, 공용 `EditorialShell` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `46e23d922c2e`가 모든 라우트 본문을 이 셸 내부에 배치합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:794615a037d3:BEGIN -->
- **직전 상태:** 직전 상태에는 **데스크톱·모바일 탐색, 디자인 선택기, 주요 콘텐츠 영역, `footerLinks`** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 기준 탐색과 푸터 링크를 콘텐츠에서 읽고 현재 경로·디버그·템플릿 상태를 보존하는 사이트 전체 셸을 만듭니다. `<main>`과 건너뛰기 대상도 이 셸이 소유합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 데스크톱·모바일 탐색, 디자인 선택기, 주요 콘텐츠 영역, `footerLinks`를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** `46e23d922c2e`가 모든 라우트 본문을 이 셸 내부에 배치합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:794615a037d3:END -->

### 34. `b7fd9118025e` — feat(editorial): 섹션 표식과 프로젝트 인덱스 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 섹션·프로젝트 목록 공용 컴포넌트

#### 커밋별 확인 사항

- `b7fd9118025e^`와 `b7fd9118025e`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`SectionKicker`, 프로젝트 목록 행과 렌더러 보존 링크** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `섹션·프로젝트 목록 요소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b7fd9118025e:BEGIN -->
- **직전 상태:** 직전 상태에는 **`SectionKicker`, 프로젝트 목록 행과 렌더러 보존 링크** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 번호 표식과 프로젝트 메타데이터·동작 링크 행을 반복 가능한 컴포넌트로 분리해 홈·아카이브가 같은 DOM 계약을 사용합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `SectionKicker`, 프로젝트 목록 행과 렌더러 보존 링크를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b7fd9118025e:END -->

### 35. `5c82371743ba` — feat(editorial): 홈 hero spread 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 라우트 시작

#### 커밋별 확인 사항

- `5c82371743ba^`와 `5c82371743ba`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`HomeRoute`, 화면 설정에서 지정한 섹션 순서, 소개 영역, 대표 대체 처리, 현재 연도** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Home 라우트 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 시점에는 설정 가능한 모든 섹션의 구현이 아직 존재하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 후속 96ba/4c827/9831/f01이 섹션 노드를 채운다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5c82371743ba:BEGIN -->
- **직전 상태:** 직전 상태에는 **`HomeRoute`, 화면 설정에서 지정한 섹션 순서, 소개 영역, 대표 대체 처리, 현재 연도** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 홈 라우트가 설정된 섹션 ID를 순회하는 분기 함수로 시작됩니다. 대표가 비면 전체 프로젝트를 사용하고 현재 연도를 실행 시점에서 계산합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `HomeRoute`, 화면 설정에서 지정한 섹션 순서, 소개 영역, 대표 대체 처리, 현재 연도를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 시점에는 모든 설정된 섹션 구현이 아직 존재하지 않습니다.
- **다음 관계:** 후속 96ba/4c827/9831/f01이 섹션 노드를 채운다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5c82371743ba:END -->

### 36. `96ba59901181` — feat(editorial): 홈 lead story 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 대표 프로젝트

#### 커밋별 확인 사항

- `96ba59901181^`와 `96ba59901181`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **첫 선택한 프로젝트를 설명 기능로 렌더링하는 대표 섹션** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈 대표 프로젝트` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:96ba59901181:BEGIN -->
- **직전 상태:** 직전 상태에는 **첫 선택한 프로젝트를 설명 기능로 렌더링하는 대표 섹션** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 선택 목록의 첫 프로젝트를 큰 변경 과정과 미디어로 사용하고 상세 링크에 템플릿·디버그 상태를 보존합니다. 프로젝트가 없으면 해당 섹션을 생략합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 첫 선택한 프로젝트를 설명 기능로 렌더링하는 대표 섹션을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:96ba59901181:END -->

### 37. `4c8270522400` — feat(editorial): 홈 대표 프로젝트 목록 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 남은 대표 목록

#### 커밋별 확인 사항

- `4c8270522400^`와 `4c8270522400`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **대표를 제외한 선택한 프로젝트와 공용 프로젝트 목록 행** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈의 나머지 대표 프로젝트 목록` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4c8270522400:BEGIN -->
- **직전 상태:** 직전 상태에는 **대표를 제외한 선택한 프로젝트와 공용 프로젝트 목록 행** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 첫 항목을 대표로 사용한 뒤 `slice(1)`의 나머지를 목록 행로 렌더링해 중복 노출을 피합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 대표를 제외한 선택한 프로젝트와 공용 프로젝트 목록 행을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4c8270522400:END -->

### 38. `983131c5a266` — feat(editorial): 홈 원칙과 기술 sidebar 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 원칙·시스템 섹션

#### 커밋별 확인 사항

- `983131c5a266^`와 `983131c5a266`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **프로필 원칙, 현재 여정, 기술 스택 sidebar** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈 원칙·시스템 섹션` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 후속 디자인 공통 뷰 모델 제한이 이 파생 책임을 라우트 구분 지점으로 옮긴다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:983131c5a266:BEGIN -->
- **직전 상태:** 직전 상태에는 **프로필 원칙, 현재 여정, 기술 스택 sidebar** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 원칙을 주요 설명으로 두고 최근 여정·기술 스택을 보조 열에 배치합니다. 원본 콘텐츠를 직접 읽는 초기 렌더러 책임이 남아 있습니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 프로필 원칙, 현재 여정, 기술 스택 sidebar를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** 후속 디자인 공통 뷰 모델 제한이 이 파생 책임을 라우트 구분 지점으로 옮긴다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:983131c5a266:END -->

### 39. `f01b60fc368e` — feat(editorial): 홈 contact strip 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 연락처 행동 유도 링크

#### 커밋별 확인 사항

- `f01b60fc368e^`와 `f01b60fc368e`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **연락처 사용 가능 여부와 우선 링크를 사용하는 마지막 설정된 섹션** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈 연락 행동 유도 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f01b60fc368e:BEGIN -->
- **직전 상태:** 직전 상태에는 **연락처 사용 가능 여부와 우선 링크를 사용하는 마지막 설정된 섹션** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 홈 섹션 맵을 연락처 행동 유도 링크까지 완성하며 우선 링크가 없을 때 섹션 동작이 비어 있음을 허용합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 연락처 사용 가능 여부와 우선 링크를 사용하는 마지막 설정된 섹션을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:f01b60fc368e:END -->

### 40. `4e69ba2ee361` — feat(editorial): 프로젝트 archive route 추가

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** 프로젝트 아카이브

#### 커밋별 확인 사항

- `4e69ba2ee361^`와 `4e69ba2ee361`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ProjectsRoute`, 프로젝트 그룹화, 지표, 대표·아카이브 분할 결과, 명시적인 빈 복사** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 아카이브` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 후속 프로젝트 뷰 모델 전환이 그룹화·지표 소유권을 콘텐츠 처리 지점로 이동합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4e69ba2ee361:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectsRoute`, 프로젝트 그룹화, 지표, 대표·아카이브 분할 결과, 명시적인 빈 복사** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 이 시점 렌더러가 프로젝트 묶음과 지표 계산을 직접 수행하고 분류별 아카이브를 만듭니다. 목록이 비면 공용 빈 상태 문구를 표시합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `ProjectsRoute`, 프로젝트 그룹화, 지표, 대표·아카이브 분할 결과, 명시적인 빈 복사를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** 후속 프로젝트 뷰 모델 전환이 그룹화·지표 소유권을 콘텐츠 처리 지점로 이동합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4e69ba2ee361:END -->

### 41. `c722cdd08ef8` — feat(editorial): 프로젝트 상세 서사와 구조 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 첫 번째 전체 경로

#### 커밋별 확인 사항

- `c722cdd08ef8^`와 `c722cdd08ef8`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ProjectDetailRoute`, 프로젝트 누락 검사, 핵심 정보, 링크, 표지, 문제·해결책·설계** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 화면의 첫 완성 경로` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `f38556a17e8b`가 증거·결과·종료를 완성합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:c722cdd08ef8:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectDetailRoute`, 프로젝트 누락 검사, 핵심 정보, 링크, 표지, 문제·해결책·설계** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 프로젝트가 없으면 dereference 전에 recoverable 누락된 펼치기와 아카이브 링크를 반환합니다. 유효한 경우 기준 상세 링크와 설명·설계를 구성합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `ProjectDetailRoute`, 프로젝트 누락 검사, 핵심 정보, 링크, 표지, 문제·해결책·설계를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** `f38556a17e8b`가 증거·결과·종료를 완성합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:c722cdd08ef8:END -->

### 42. `f38556a17e8b` — feat(editorial): 프로젝트 증거와 결과 spread 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 완료

#### 커밋별 확인 사항

- `f38556a17e8b^`와 `f38556a17e8b`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **주요 내용, 선택적 보조 이미지, 결정, 절충안, 결과, 아카이브 종료** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f38556a17e8b:BEGIN -->
- **직전 상태:** 직전 상태에는 **주요 내용, 선택적 보조 이미지, 결정, 절충안, 결과, 아카이브 종료** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 선택적 갤러리는 실제 이미지가 있을 때만 보이고 근거 목록과 결과·종료 동선을 추가합니다. 상세 라우트가 하나의 완결된 프로젝트 사례가 됩니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 주요 내용, 선택적 보조 이미지, 결정, 절충안, 결과, 아카이브 종료를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:f38556a17e8b:END -->

### 43. `cc1b2233287f` — feat(editorial): About 정체성과 원칙 소개 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 소개 식별 정보

#### 커밋별 확인 사항

- `cc1b2233287f^`와 `cc1b2233287f`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`AboutRoute`, 선택적 사진, 프로필 핵심 정보, 원칙** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `소개 페이지의 프로필 정보` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:cc1b2233287f:BEGIN -->
- **직전 상태:** 직전 상태에는 **`AboutRoute`, 선택적 사진, 프로필 핵심 정보, 원칙** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 프로필 식별 정보와 원칙을 기준 콘텐츠에서 읽고 사진이 있을 때만 미디어 프레임을 배치합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `AboutRoute`, 선택적 사진, 프로필 핵심 정보, 원칙을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:cc1b2233287f:END -->

### 44. `5f0193979568` — feat(editorial): About 기술과 경력 소개 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 소개 기술 목록·경력

#### 커밋별 확인 사항

- `5f0193979568^`와 `5f0193979568`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **포커스 분야, 기술 그룹, chronological 경력** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `소개 페이지의 기술·경력` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5f0193979568:BEGIN -->
- **직전 상태:** 직전 상태에는 **포커스 분야, 기술 그룹, chronological 경력** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 소개 라우트에 기술과 경력 레코드를 추가하며 각 배열의 원본 순서를 화면 구성 순서로 사용합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 포커스 분야, 기술 그룹, chronological 경력을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5f0193979568:END -->

### 45. `5c95665ca9d2` — feat(editorial): About 큐레이션 기준 추가

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** 기능 플래그로 제어되는 선별 기록 시작

#### 커밋별 확인 사항

- `5c95665ca9d2^`와 `5c95665ca9d2`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`isSitePageEnabled("curation", content)`와 기준** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `기능 설정에 따른 선별 기록 화면 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5c95665ca9d2:BEGIN -->
- **직전 상태:** 직전 상태에는 **`isSitePageEnabled("curation", content)`와 기준** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 선별 기록 지원 여부가 켜졌을 때만 기준 섹션을 추가합니다. 비활성화된 상태는 빈 패널이 아니라 섹션 자체 부재다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `isSitePageEnabled("curation", content)`와 기준을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5c95665ca9d2:END -->

### 46. `4a7c3a3c9cde` — feat(editorial): About 큐레이션 범주 추가

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** 선별 기록 분류 프로젝트 결합

#### 커밋별 확인 사항

- `4a7c3a3c9cde^`와 `4a7c3a3c9cde`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **분류 프로젝트 ID를 기준 프로젝트에 `find`/필터하여 링크 생성** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `선별 분류와 프로젝트 연결` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4a7c3a3c9cde:BEGIN -->
- **직전 상태:** 직전 상태에는 **분류 프로젝트 ID를 기준 프로젝트에 `find`/필터하여 링크 생성** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 유효한 프로젝트 ID만 분류 카드에 남기며 누락 참조는 링크를 만들지 않습니다. 이 결합은 이 시점 렌더러 내부 책임입니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 분류 프로젝트 ID를 기준 프로젝트에 `find`/필터하여 링크 생성를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4a7c3a3c9cde:END -->

### 47. `c0d0004e9355` — feat(editorial): About 큐레이션 공백과 재검토 추가

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** 선별 기록 완료

#### 커밋별 확인 사항

- `c0d0004e9355^`와 `c0d0004e9355`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **제외 항목 목록과 nextReview 패널** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `선별 기록 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:c0d0004e9355:BEGIN -->
- **직전 상태:** 직전 상태에는 **제외 항목 목록과 nextReview 패널** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 선별 기록에서 의도적으로 제외한 항목과 다음 재검토 조건을 별도 섹션으로 노출해 누락을 암묵적 누락으로 만들지 않습니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 제외 항목 목록과 nextReview 패널을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:c0d0004e9355:END -->

### 48. `119d19ab41b1` — feat(editorial): Resume 정체성과 프로젝트 경력 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 이력서 시작

#### 커밋별 확인 사항

- `119d19ab41b1^`와 `119d19ab41b1`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ResumeRoute`, 선택적 다운로드, 식별 정보 핵심 정보, 요약, 해석된 이력서 프로젝트, 빈 대체 처리** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `이력서 화면 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:119d19ab41b1:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ResumeRoute`, 선택적 다운로드, 식별 정보 핵심 정보, 요약, 해석된 이력서 프로젝트, 빈 대체 처리** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 다운로드 URL은 있을 때만 기준을 만들고 이력서 프로젝트 ID를 기준 프로젝트로 해석한 결과를 표시합니다. 해석 결과가 비면 프로젝트 아카이브 빈 복사를 사용합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `ResumeRoute`, 선택적 다운로드, 식별 정보 핵심 정보, 요약, 해석된 이력서 프로젝트, 빈 대체 처리를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:119d19ab41b1:END -->

### 49. `4df2710fa7f9` — feat(editorial): Resume 경력과 교육 기록 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 이력서 완료

#### 커밋별 확인 사항

- `4df2710fa7f9^`와 `4df2710fa7f9`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **경력, 교육, 학력, 안내와 `EvidenceList` 빈 문구** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `이력서 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4df2710fa7f9:BEGIN -->
- **직전 상태:** 직전 상태에는 **경력, 교육, 학력, 안내와 `EvidenceList` 빈 문구** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 이력서에 네 개의 후속 섹션을 원본 순서로 추가하고 안내는 공용 빈 상태 공용 컴포넌트로 비어 있음을 명시합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 경력, 교육, 학력, 안내와 `EvidenceList` 빈 문구를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4df2710fa7f9:END -->

### 50. `61d6952850cd` — feat(editorial): Contact desk route 추가

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** 연락처 라우트

#### 커밋별 확인 사항

- `61d6952850cd^`와 `61d6952850cd`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`ContactRoute`, 우선 연락처 순서 결정, 사용 가능 여부, 안내, 명시적인 빈 링크** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `연락처 라우트` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:61d6952850cd:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ContactRoute`, 우선 연락처 순서 결정, 사용 가능 여부, 안내, 명시적인 빈 링크** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** `getPreferredContactLinks` 결과를 동작 카드로 표시하고 비면 `emptyStates.contactLinks`를 보여 줍니다. 연락 가능 상태와 안내는 별도 열이 소유합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `ContactRoute`, 우선 연락처 순서 결정, 사용 가능 여부, 안내, 명시적인 빈 링크를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:61d6952850cd:END -->

### 51. `08fa527b9b65` — feat(editorial): Journey milestone spread 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 여정 설명 시작

#### 커밋별 확인 사항

- `08fa527b9b65^`와 `08fa527b9b65`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`JourneyRoute`, 주요 시점, anchorProjectIds 해석, 빈 여정** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `여정 설명 화면 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:08fa527b9b65:BEGIN -->
- **직전 상태:** 직전 상태에는 **`JourneyRoute`, 주요 시점, anchorProjectIds 해석, 빈 여정** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 주요 시점마다 기준 ID를 기준 프로젝트로 해석하고 유효한 것만 탐색 링크로 남깁니다. 주요 시점 배열이 비면 명시적 여정 빈 행을 렌더링합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `JourneyRoute`, 주요 시점, anchorProjectIds 해석, 빈 여정을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:08fa527b9b65:END -->

### 52. `96b66af4d5a7` — feat(editorial): Journey timeline과 현재 방향 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 여정 완료

#### 커밋별 확인 사항

- `96b66af4d5a7^`와 `96b66af4d5a7`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **날짜가 있는 타임라인, 선택적 연결된 프로젝트, `currentPosition`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `여정 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:96b66af4d5a7:BEGIN -->
- **직전 상태:** 직전 상태에는 **날짜가 있는 타임라인, 선택적 연결된 프로젝트, `currentPosition`** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 더 넓은 여정 아카이브를 원본 순서로 렌더링하고 projectId가 실제 프로젝트로 해석될 때만 링크를 만듭니다. 마지막에 현재 위치 제목·본문을 별도 고대비 섹션으로 고정합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 날짜가 있는 타임라인, 선택적 연결된 프로젝트, `currentPosition`를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:96b66af4d5a7:END -->

### 53. `5e2f37861d3d` — feat(editorial): Interview Map 소개와 chapter 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 인터뷰 시작

#### 커밋별 확인 사항

- `5e2f37861d3d^`와 `5e2f37861d3d`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`InterviewMapRoute`, 외부 참조 저장소, 트랙 프래그먼트 목록, `projectById` 맵** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `인터뷰 화면 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5e2f37861d3d:BEGIN -->
- **직전 상태:** 직전 상태에는 **`InterviewMapRoute`, 외부 참조 저장소, 트랙 프래그먼트 목록, `projectById` 맵** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 소개와 외부 참조 링크를 만들고 설정된 트랙으로 프래그먼트 탐색을 생성합니다. 프로젝트 조회용 맵을 준비하지만 답변 근거 본문은 다음 커밋 전에는 없습니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 `InterviewMapRoute`, 외부 참조 저장소, 트랙 프래그먼트 목록, `projectById` 맵을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5e2f37861d3d:END -->

### 54. `94deba32f56a` — feat(editorial): Interview 답변 근거와 공백 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 인터뷰 완료

#### 커밋별 확인 사항

- `94deba32f56a^`와 `94deba32f56a`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **트랙·질문·참조·답변·깊이, 누락된 프로젝트·빈 답변, 부족한 부분** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `인터뷰 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:94deba32f56a:BEGIN -->
- **직전 상태:** 직전 상태에는 **트랙·질문·참조·답변·깊이, 누락된 프로젝트·빈 답변, 부족한 부분** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 답변과 프로젝트가 맵에 없으면 링크 대신 `noMappedEvidence`를 보이되 깊이는 유지합니다. 트랙·항목·답변이 비어 있는 각 계층과 부족한 부분 목록을 명시적으로 표시합니다.
- **정적 검토:** `src/designs/editorial/editorial-route.tsx`의 트랙·질문·참조·답변·깊이, 누락된 프로젝트·빈 답변, 부족한 부분을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:94deba32f56a:END -->

### 55. `46e23d922c2e` — feat(editorial): route dispatcher 추가

- **중요도:** A
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** Editorial 여덟 라우트와 공용 셸을 하나의 공개 진입점으로 폐쇄

#### 커밋별 확인 사항

- `46e23d922c2e^`와 `46e23d922c2e`를 비교하고 `src/designs/editorial/editorial-route.tsx`에서 **`renderRoute`의 누락 없는 전환과 exported `EditorialRoute`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Editorial의 8개 라우트와 공용 셸을 하나의 공개 진입점으로 통일` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `TypeScript 유니언의 누락 검사에 의존하며, 실행 시점의 알 수 없는 문자열을 위한 별도 기본 화면은 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 1의 `c6acfe562694`가 이 진입점을 등록부에 활성화합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:46e23d922c2e:BEGIN -->
- **직전 상태:** 직전 상태에는 **`renderRoute`의 누락 없는 전환과 exported `EditorialRoute`** 관련 라우트·컴포넌트가 아직 없었습니다.
- **구현 결정:** 각 라우트 함수가 따로 존재하던 상태에서 `renderRoute`가 종류가 구분된 라우트를 여덟 view로 매핑하고, 공개 `EditorialRoute`는 결과를 항상 `EditorialShell` 안에 넣는다. 외부 등록부는 내부 view 이름을 알 필요가 없고 셸·탐색·푸터가 모든 라우트에 일관되게 적용됩니다.
- **파일·심볼:** `src/designs/editorial/editorial-route.tsx`에서 `renderRoute`의 누락 없는 전환과 exported `EditorialRoute`를 확인했습니다.
- **소유권:** `src/designs/editorial/editorial-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `Editorial의 8개 라우트와 공용 셸을 하나의 공개 진입점으로 통일`. TypeScript 유니언의 누락 검사에 의존하며 실행 중 알 수 없는 문자열에 대한 별도 기본 화면는 없습니다.
- **역사적 연결:** 개발 과정 1의 `c6acfe562694`가 이 진입점을 등록부에 활성화합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

```tsx
// 46e23d922c2e · src/designs/editorial/editorial-route.tsx
export function EditorialRoute(props: EditorialRouteProps) {
  return <EditorialShell {...props}>{renderRoute(props)}</EditorialShell>;
}
```
<!-- LEARNER-ANSWER:commit:46e23d922c2e:END -->

## 5. 불변 조건 변화

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:invariant:BEGIN -->
최종 불변 조건은 `EditorialRoute` 하나만 외부에 노출되고, 라우트 구분값이 여덟 내부 view 중 하나를 선택한 뒤 항상 `EditorialShell`로 감싸며, 누락·실패를 라우트별 명시적 UI로 표현한다는 것입니다.

- 도입·확장·폐쇄의 순서는 커밋 목록에 고정했습니다.
- 중요도 B 구축은 라우트·스타일 표시 영역을 단계적으로 넓히고, A·중요도 S 커밋은 소유 주체·분기·검증 불변 조건을 바꿉니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:invariant:END -->

## 6. 실패 → 수정 → 테스트와 소유 주체 변화

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:relations:BEGIN -->
CSS는 데스크톱 지면을 먼저 완성한 뒤 1180px·태블릿·모바일 화면 대응과 동작 줄이기 설정을 추가합니다. TSX는 규칙·공용 컴포넌트·셸 뒤 홈→프로젝트→Detail→소개→이력서→연락처→여정→인터뷰 순서로 확장됩니다. 프로젝트·상세·연락처·여정·인터뷰의 빈·누락된 참조 브랜치가 후속 라우트 완료에서 보강되고, `46e23d922c2e`가 최종 공개 API를 닫습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:relations:END -->

## 7. 최종 구성과 실행 순서

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:flow:BEGIN -->
등록부 진입점 → `EditorialRoute` → `renderRoute` 전환 → 내부 라우트 view → 공용 Editorial 공용 컴포넌트·링크 도우미 함수 → `EditorialShell`의 상단 제목 영역·main·푸터 순서입니다.

각 단계에서 선택적·빈·누락된 참조가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 커밋의 보장하지 않는 범위에 남겼습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:flow:END -->

## 8. 실행 및 검증 근거

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:runtime:BEGIN -->
- **실행한 저장소 테스트·빌드:** 없음.
- **정적 확인:** 지정 브랜치의 커밋 분류, 커밋 bodies, 해당 커밋의 변경 내용과 변경 이력의 파일 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 컨테이너에서 직접 복제본 시 DNS가 `github.com`을 해석하지 못해 변경 이력의 worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next 빌드 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** 코드·테스트 구현의 존재와 범위는 정적 검토로 확인했고, 실행 성공·실패는 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:runtime:END -->

## 9. 학습 완료 확인

<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·제목·중요도·태그를 커밋 목록과 커밋 섹션에서 동일하게 유지했습니다.
- [x] 각 SHA에 구체적인 파일과 심볼·선택자·라우트 포커스를 기록했습니다.
- [x] 이전 상태, 소유 주체, 누락·대체 처리, 보장 범위와 보장하지 않는 범위, 후속 관계를 채웠습니다.
- [x] S/A/B 깊이를 구분했습니다.
- [x] 실행하지 않은 테스트를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:02-editorial-design-system-construction.md:checks:END -->
===== END FILE: 02-editorial-design-system-construction.md =====

===== BEGIN FILE: 03-brutalist-design-system-construction.md =====
# 개발 과정: Brutalist 디자인 구성

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `05-full-site-visual-systems`
>
> 1단계 검토에서 확정한 기준 문서 틀입니다. 2단계에서는 답변 식별 속성 내부만 채웁니다.

## 0. 범위와 기준

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md`와 해당 커밋의 정확한 메타데이터를 기준으로 고정했습니다.
- **개발 과정 범위:** Brutalist 스타일시트와 `brutalist-route.tsx` 구축만 포함합니다. 활성화는 개발 과정 1에, 서식만 변경한 미디어 consolidation은 제외합니다.
- 다른 브랜치, 최종 HEAD의 후대 구현, 실행하지 않은 명령 결과를 사용하지 않습니다.

## 1. 개발 과정 목표

Brutalist의 고대비시각 문법, 반응형·인쇄 규칙, 콘텐츠 변환 함수, 여덟 라우트와 단일 공개 진입점이 구축되는 과정을 복원합니다.

### 고정된 불변 조건

최종 불변 조건은 Brutalist 모듈이 라우트 진입점 하나만 공개하고, 셸·탐색·링크 규칙·콘텐츠 변환 함수·view가 같은 모듈의 내부 구현으로 유지되며, 모바일·동작 줄이기 설정·print와 누락되거나 빈 상태가 시각 표현 규칙에 포함된다는 것입니다.

## 2. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `162542118ba4` | style(brutalist): 화면 토큰과 brand mark 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 범위가 제한된 색상표·포커스·건너뛰기 링크·헤더·브랜드 기초 |
| 2 | `a2539ef309d1` | style(brutalist): header 상태와 home hero 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 데스크톱 헤더 상태·선택기·탐색·디버그와 홈 소개 영역 |
| 3 | `1faf77ef9916` | style(brutalist): hero stamp와 action row 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 소개 영역 stamp·복사·큰 제목·동작 링크 행 |
| 4 | `75913149fe24` | style(brutalist): 주요 action과 section 경계 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 고대동작 효과가 아닌·4-열 지표·신호 표시줄·섹션 구분선 |
| 5 | `f4e53be5ea42` | style(brutalist): section header와 프로젝트 지표 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 번호가 있는 섹션 헤더 그리드와 프로젝트 목록 행 |
| 6 | `ebfe79d62e53` | style(brutalist): 프로젝트 지표와 card 번호 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 프로젝트 메타데이터·태그·동작과 원칙 카드 |
| 7 | `aaf26e755213` | style(brutalist): 원칙 카드와 contact band 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 원칙·기술 목록·간결한 타임라인·큰 연락처 영역 |
| 8 | `16336e1dc469` | style(brutalist): contact 링크와 프로젝트 group 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 연락처 동작·페이지 소개 영역·인라인 지표·그룹화된 아카이브 |
| 9 | `2660465c0904` | style(brutalist): 교차 group과 상세 lead 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 alternating 그룹과 프로젝트 사례 대표 |
| 10 | `4621b0a3cb1f` | style(brutalist): 상세 fact와 소개 본문 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 상세 핵심 정보·미디어 프레임·자리표시자·도입부 영역 |
| 11 | `1d5445cc6f4a` | style(brutalist): 상세 본문과 gallery grid 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 labeled 근거 섹션·번호가 있는 목록·2열 갤러리 |
| 12 | `bb5008a8c7b3` | style(brutalist): 다음 프로젝트와 focus card 구성 | B | RENDERER, A11Y | Brutalist 시각 표현 규칙의 다음 프로젝트·찾을 수 없음 복구·소개 프로필 사진·기술 목록 |
| 13 | `8de2180bcc58` | style(brutalist): focus card와 criteria grid 구성 | B | RENDERER, A11Y | Brutalist 시각 표현 규칙의 포커스·기술 카드와 어두운 선별 기록 기준 |
| 14 | `a34cd7cd88bf` | style(brutalist): criteria 본문과 재검토 영역 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 선별 기록 분류·제외·다음 검토 |
| 15 | `5ca14417cf22` | style(brutalist): 재검토와 resume entry 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 검토 패널 마감과 이력서 반복 섹션 |
| 16 | `fad7f0216645` | style(brutalist): resume 본문과 contact hero 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 이력서 프로젝트 행·안내와 연락처 blue 소개 영역 |
| 17 | `8175392db042` | style(brutalist): contact 상태와 note 목록 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 사용 가능 여부 배지·channel 그리드·안내 목록 |
| 18 | `6fa3a9dc8665` | style(brutalist): note 목록과 anchor link 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 안내·근거 부족한 부분과 여정 주요 시점 카드 |
| 19 | `242ba8e66e0b` | style(brutalist): archive timeline과 track navigation 구성 | B | ROUTING, RENDERER | Brutalist 시각 표현 규칙의 여정 아카이브·현재 callout와 인터뷰 트랙 셸 |
| 20 | `95e55eda6c51` | style(brutalist): track 목록과 question prompt 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 트랙 목록·트랙·질문 배치 순서 |
| 21 | `11f229d630e9` | style(brutalist): 답변 근거와 footer lead 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 질문 참조·답변 근거·빈 답변·푸터 대표 |
| 22 | `b170c73a36d0` | style(brutalist): footer metadata와 blink 동작 구성 | B | RENDERER, SEO | Brutalist 시각 표현 규칙의 푸터 메타데이터·dashed 빈 상태·흐르는·깜박이는 애니메이션 |
| 23 | `f810c49022be` | style(brutalist): tablet grid 재배치 | B | RENDERER | Brutalist 시각 표현 규칙의 980px 태블릿 그리드 재배치 |
| 24 | `b57da6a41419` | style(brutalist): mobile header와 hero 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 720px 브라우저 기본 `<details>` 메뉴와 세로로 쌓인 헤더·소개 영역 |
| 25 | `8168bc76c3e3` | style(brutalist): mobile 프로젝트와 상세 화면 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 모바일 지표·프로젝트 행·상세·갤러리 재배치 |
| 26 | `5551f3fdbb94` | style(brutalist): mobile profile과 resume 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 모바일 프로필·선별 기록·이력서·연락처·현재 위치 |
| 27 | `7c08aea7a2f7` | style(brutalist): mobile 여정과 interview 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 모바일 여정·인터뷰·푸터·누락된 페이지 |
| 28 | `077ff3d49f30` | style(brutalist): 소형 화면과 인쇄 경계 구성 | B | RENDERER | Brutalist 시각 표현 규칙의 430px 강화·동작 줄이기 설정·print |
| 29 | `3e6ec5262bdd` | feat(brutalist): 콘텐츠와 탐색 조회 도우미 추가 | B | CONTENT, ROUTING, RENDERER | 콘텐츠·탐색 변환 함수 레이어 |
| 30 | `08a2b0c0998f` | feat(brutalist): route 레이블과 기본 shell 구성 | B | ROUTING, RENDERER | 공용 셸 시작 |
| 31 | `cf2fdb36f9fc` | feat(brutalist): 주 탐색과 모바일 메뉴 추가 | B | ROUTING, RENDERER | 기준 탐색 |
| 32 | `5b44afbc46ef` | feat(brutalist): footer와 홈 히어로 연결 | B | RENDERER | 푸터 + 홈 시작 |
| 33 | `b477ba477127` | feat(brutalist): 홈 섹션 공용 프리미티브 추가 | B | RENDERER | 홈·아카이브 공용 컴포넌트 |
| 34 | `b30b9b1c3505` | feat(brutalist): 대표 작업과 작업 원칙 구성 | B | RENDERER | 홈 중간 + 프로젝트 소개 영역 |
| 35 | `85ea663aaf19` | feat(brutalist): 홈 여정과 프로젝트 archive 구성 | B | RENDERER | 홈 완료 + 프로젝트 아카이브 |
| 36 | `d6b9a99e11ae` | feat(brutalist): 프로젝트 상세 표시 프리미티브 추가 | B | RENDERER | 프로젝트 상세 공용 컴포넌트 |
| 37 | `b8268f47e89a` | feat(brutalist): 프로젝트 상세 hero와 소개 구성 | B | RENDERER | 프로젝트 상세 유효한·누락된 구분 지점 |
| 38 | `05b838d52a8b` | feat(brutalist): 프로젝트 상세 본문과 gallery 구성 | B | RENDERER | 프로젝트 상세 근거 본문 |
| 39 | `80724a26820b` | feat(brutalist): 프로필과 기술 소개 구성 | B | RENDERER | 소개 식별 정보 |
| 40 | `3399b55c3aee` | feat(brutalist): 큐레이션과 경력 소개 구성 | B | CONTENT, RENDERER | 소개 경력 + 기능 플래그로 제어되는 선별 기록 |
| 41 | `70cf13ef1715` | feat(brutalist): 이력 hero와 경력 요약 구성 | B | RENDERER | 이력서 시작 |
| 42 | `1ea2a1345b76` | feat(brutalist): 프로젝트 결과와 의사결정 구성 | B | RENDERER | 프로젝트 상세 완료 |
| 43 | `5fa378250d64` | feat(brutalist): 선택 프로젝트와 이력 세부 구성 | B | RENDERER | 이력서 완료 |
| 44 | `b535539ae016` | feat(brutalist): 연락 수단과 안내 구성 | B | RENDERER | 연락처 라우트 |
| 45 | `15a765ecb2aa` | feat(brutalist): 여정 milestone 구성 | B | RENDERER | 여정 설명 시작 |
| 46 | `388446b1a982` | feat(brutalist): 여정 archive와 인터뷰 map 머리말 구성 | B | RENDERER | 여정 완료 + 인터뷰 시작 |
| 47 | `f3fc6200a45b` | feat(brutalist): 인터뷰 근거 archive 구성 | B | RENDERER | 인터뷰 근거 |
| 48 | `da8e59d56783` | feat(brutalist): 인터뷰 근거 공백 구성 | B | RENDERER | 인터뷰 부족한 부분 |
| 49 | `e6268c4b7c74` | refactor(brutalist): 내부 helper 공개 범위 정리 | B | RENDERER, REFACTOR | 모듈 공개 범위 축소 |
| 50 | `caa7df81d899` | feat(brutalist): 모든 route를 renderer에 통합 | A | ARCH, ROUTING, RENDERER | Brutalist 공개 API와 여덟 라우트 분기의 최종 경계 |

## 3. 변경 전 기준

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:baseline:BEGIN -->
- **직전 상태:** 공통 라우트 위임은 있었지만 Brutalist는 CSS/DOM·셸·도우미 함수가 전혀 없는 상태였습니다. 구현 초기는 렌더러 내부 콘텐츠 조회 도우미 함수를 직접 소유했습니다.
- **경계 판단:** Brutalist 스타일시트와 `brutalist-route.tsx` 구축만 포함합니다. 활성화는 개발 과정 1에, 서식만 변경한 미디어 consolidation은 제외합니다.
- **복원 기준:** 각 커밋의 부모 커밋과 해당 SHA 파일 트리만 사용하고 최종 HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:baseline:END -->

## 4. 커밋별 복원 기록

### 1. `162542118ba4` — style(brutalist): 화면 토큰과 brand mark 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 범위가 제한된 색상표·포커스·건너뛰기 링크·헤더·브랜드 기초

#### 커밋별 확인 사항

- `162542118ba4^`와 `162542118ba4`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **범위가 제한된 색상표·포커스·건너뛰기 링크·헤더·브랜드 기초에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 디자인 범위에 한정된 색상표·포커스·본문 건너뛰기 링크·헤더·브랜드 기초` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:162542118ba4:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **범위가 제한된 색상표·포커스·건너뛰기 링크·헤더·브랜드 기초에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 종이·잉크·파랑·노랑 토큰, 박스 크기 계산, 포커스 표시 여부, 선택, 건너뛰기 링크와 브랜드 표식를 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 범위가 제한된 색상표·포커스·건너뛰기 링크·헤더·브랜드 기초에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:162542118ba4:END -->

### 2. `a2539ef309d1` — style(brutalist): header 상태와 home hero 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 데스크톱 헤더 상태·선택기·탐색·디버그와 홈 소개 영역

#### 커밋별 확인 사항

- `a2539ef309d1^`와 `a2539ef309d1`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **데스크톱 헤더 상태·선택기·탐색·디버그와 홈 소개 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 데스크톱 헤더 상태·선택기·탐색·디버그와 홈 소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:a2539ef309d1:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **데스크톱 헤더 상태·선택기·탐색·디버그와 홈 소개 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 명시적 상태 셀과 큰 소개 영역 그리드를 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 데스크톱 헤더 상태·선택기·탐색·디버그와 홈 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:a2539ef309d1:END -->

### 3. `1faf77ef9916` — style(brutalist): hero stamp와 action row 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 소개 영역 stamp·복사·큰 제목·동작 링크 행

#### 커밋별 확인 사항

- `1faf77ef9916^`와 `1faf77ef9916`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **소개 영역 stamp·복사·큰 제목·동작 링크 행에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 소개 영역 표식·문구·큰 제목·동작 행` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1faf77ef9916:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **소개 영역 stamp·복사·큰 제목·동작 링크 행에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 소개 영역의 시각적 표식와 유연한 동작 링크 행을 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 소개 영역 stamp·복사·큰 제목·동작 링크 행에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1faf77ef9916:END -->

### 4. `75913149fe24` — style(brutalist): 주요 action과 section 경계 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 고대동작 효과가 아닌·4-열 지표·신호 표시줄·섹션 구분선

#### 커밋별 확인 사항

- `75913149fe24^`와 `75913149fe24`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **고대동작 효과가 아닌·4-열 지표·신호 표시줄·섹션 구분선에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 고대비 동작 요소·4열 지표·상태 표시줄·섹션 구분선` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:75913149fe24:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **고대동작 효과가 아닌·4-열 지표·신호 표시줄·섹션 구분선에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 공용 버튼 문법과 지표·신호·섹션 구획을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 고대동작 효과가 아닌·4-열 지표·신호 표시줄·섹션 구분선에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:75913149fe24:END -->

### 5. `f4e53be5ea42` — style(brutalist): section header와 프로젝트 지표 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 번호가 있는 섹션 헤더 그리드와 프로젝트 목록 행

#### 커밋별 확인 사항

- `f4e53be5ea42^`와 `f4e53be5ea42`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **번호가 있는 섹션 헤더 그리드와 프로젝트 목록 행에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 번호가 있는 섹션 헤더 그리드와 프로젝트 목록 행` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f4e53be5ea42:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **번호가 있는 섹션 헤더 그리드와 프로젝트 목록 행에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 홈·아카이브에서 재사용할 번호 헤더와 프로젝트 메타데이터 행을 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 번호가 있는 섹션 헤더 그리드와 프로젝트 목록 행에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:f4e53be5ea42:END -->

### 6. `ebfe79d62e53` — style(brutalist): 프로젝트 지표와 card 번호 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 프로젝트 메타데이터·태그·동작과 원칙 카드

#### 커밋별 확인 사항

- `ebfe79d62e53^`와 `ebfe79d62e53`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **프로젝트 메타데이터·태그·동작과 원칙 카드에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 프로젝트 메타데이터·태그·동작과 원칙 카드` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:ebfe79d62e53:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **프로젝트 메타데이터·태그·동작과 원칙 카드에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 프로젝트 목록을 완성하고 첫 원칙 카드 시스템을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 프로젝트 메타데이터·태그·동작과 원칙 카드에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:ebfe79d62e53:END -->

### 7. `aaf26e755213` — style(brutalist): 원칙 카드와 contact band 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 원칙·기술 목록·간결한 타임라인·큰 연락처 영역

#### 커밋별 확인 사항

- `aaf26e755213^`와 `aaf26e755213`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **원칙·기술 목록·간결한 타임라인·큰 연락처 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 원칙·기술 목록 벽·압축 타임라인·큰 연락 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:aaf26e755213:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **원칙·기술 목록·간결한 타임라인·큰 연락처 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 홈 후반의 원칙·기술 스택·여정·연락처 지면을 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 원칙·기술 목록·간결한 타임라인·큰 연락처 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:aaf26e755213:END -->

### 8. `16336e1dc469` — style(brutalist): contact 링크와 프로젝트 group 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 연락처 동작·페이지 소개 영역·인라인 지표·그룹화된 아카이브

#### 커밋별 확인 사항

- `16336e1dc469^`와 `16336e1dc469`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **연락처 동작·페이지 소개 영역·인라인 지표·그룹화된 아카이브에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 연락 동작·페이지 소개 영역·인라인 지표·그룹별 아카이브` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:16336e1dc469:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **연락처 동작·페이지 소개 영역·인라인 지표·그룹화된 아카이브에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 연락처 동작과 프로젝트 아카이브 공용 구조를 시작합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 연락처 동작·페이지 소개 영역·인라인 지표·그룹화된 아카이브에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:16336e1dc469:END -->

### 9. `2660465c0904` — style(brutalist): 교차 group과 상세 lead 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 alternating 그룹과 프로젝트 사례 대표

#### 커밋별 확인 사항

- `2660465c0904^`와 `2660465c0904`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **alternating 그룹과 프로젝트 사례 대표에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 교차 배치 그룹과 프로젝트 사례 대표 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:2660465c0904:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **alternating 그룹과 프로젝트 사례 대표에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 아카이브 그룹의 교차 배경과 상세 도입 그리드를 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 alternating 그룹과 프로젝트 사례 대표에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:2660465c0904:END -->

### 10. `4621b0a3cb1f` — style(brutalist): 상세 fact와 소개 본문 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 상세 핵심 정보·미디어 프레임·자리표시자·도입부 영역

#### 커밋별 확인 사항

- `4621b0a3cb1f^`와 `4621b0a3cb1f`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **상세 핵심 정보·미디어 프레임·자리표시자·도입부 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 상세 정보·미디어 프레임·자리표시자·도입부 띠` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4621b0a3cb1f:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **상세 핵심 정보·미디어 프레임·자리표시자·도입부 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 프로젝트 상세의 핵심 정보·미디어·도입부 기초를 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 상세 핵심 정보·미디어 프레임·자리표시자·도입부 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4621b0a3cb1f:END -->

### 11. `1d5445cc6f4a` — style(brutalist): 상세 본문과 gallery grid 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 labeled 근거 섹션·번호가 있는 목록·2열 갤러리

#### 커밋별 확인 사항

- `1d5445cc6f4a^`와 `1d5445cc6f4a`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **labeled 근거 섹션·번호가 있는 목록·2열 갤러리에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 이름이 있는 근거 섹션·번호 목록·2열 갤러리` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1d5445cc6f4a:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **labeled 근거 섹션·번호가 있는 목록·2열 갤러리에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 장문 프로젝트 사례 근거와 갤러리를 구조화합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 labeled 근거 섹션·번호가 있는 목록·2열 갤러리에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1d5445cc6f4a:END -->

### 12. `bb5008a8c7b3` — style(brutalist): 다음 프로젝트와 focus card 구성

- **중요도:** B
- **태그:** RENDERER, A11Y
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 다음 프로젝트·찾을 수 없음 복구·소개 프로필 사진·기술 목록

#### 커밋별 확인 사항

- `bb5008a8c7b3^`와 `bb5008a8c7b3`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **다음 프로젝트·찾을 수 없음 복구·소개 프로필 사진·기술 목록에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 다음 프로젝트·찾을 수 없음 화면의 복귀 링크·소개 프로필 사진과 기술` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `동작 줄이기·포커스·의미 구조 보완은 CSS와 DOM 규칙의 일부만 다룹니다. 실제 WCAG 적합성이나 모든 보조기기에서의 동작까지 단독으로 보장하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:bb5008a8c7b3:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **다음 프로젝트·찾을 수 없음 복구·소개 프로필 사진·기술 목록에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 상세 후속 내용·복구 상태와 소개 시작 지면을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 다음 프로젝트·찾을 수 없음 복구·소개 프로필 사진·기술 목록에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 동작 줄이기 설정·포커스·의미상 보조는 CSS·DOM 규칙의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:bb5008a8c7b3:END -->

### 13. `8de2180bcc58` — style(brutalist): focus card와 criteria grid 구성

- **중요도:** B
- **태그:** RENDERER, A11Y
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 포커스·기술 카드와 어두운 선별 기록 기준

#### 커밋별 확인 사항

- `8de2180bcc58^`와 `8de2180bcc58`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **포커스·기술 카드와 어두운 선별 기록 기준에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 집중 분야·기술 카드와 어두운 선별 기준 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `동작 줄이기·포커스·의미 구조 보완은 CSS와 DOM 규칙의 일부만 다룹니다. 실제 WCAG 적합성이나 모든 보조기기에서의 동작까지 단독으로 보장하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:8de2180bcc58:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **포커스·기술 카드와 어두운 선별 기록 기준에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 소개 기술 카드와 번호가 있는 선별 기록 기준을 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 포커스·기술 카드와 어두운 선별 기록 기준에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 동작 줄이기 설정·포커스·의미상 보조는 CSS·DOM 규칙의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:8de2180bcc58:END -->

### 14. `a34cd7cd88bf` — style(brutalist): criteria 본문과 재검토 영역 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 선별 기록 분류·제외·다음 검토

#### 커밋별 확인 사항

- `a34cd7cd88bf^`와 `a34cd7cd88bf`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **선별 기록 분류·제외·다음 검토에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 선별 분류·제외 항목·다음 검토` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:a34cd7cd88bf:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **선별 기록 분류·제외·다음 검토에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 선별 기록 화면 구성을 분류·제외·검토까지 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 선별 기록 분류·제외·다음 검토에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:a34cd7cd88bf:END -->

### 15. `5ca14417cf22` — style(brutalist): 재검토와 resume entry 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 검토 패널 마감과 이력서 반복 섹션

#### 커밋별 확인 사항

- `5ca14417cf22^`와 `5ca14417cf22`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **검토 패널 마감과 이력서 반복 섹션에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 검토 패널 마감과 이력서 반복 섹션` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5ca14417cf22:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **검토 패널 마감과 이력서 반복 섹션에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 선별 기록 종료와 이력서 진입점 문법을 연결합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 검토 패널 마감과 이력서 반복 섹션에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5ca14417cf22:END -->

### 16. `fad7f0216645` — style(brutalist): resume 본문과 contact hero 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 이력서 프로젝트 행·안내와 연락처 blue 소개 영역

#### 커밋별 확인 사항

- `fad7f0216645^`와 `fad7f0216645`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **이력서 프로젝트 행·안내와 연락처 blue 소개 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 이력서 프로젝트 행·안내와 Contact 파란색 소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:fad7f0216645:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **이력서 프로젝트 행·안내와 연락처 blue 소개 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 이력서 후반과 연락처 시작을 구성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 이력서 프로젝트 행·안내와 연락처 blue 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:fad7f0216645:END -->

### 17. `8175392db042` — style(brutalist): contact 상태와 note 목록 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 사용 가능 여부 배지·channel 그리드·안내 목록

#### 커밋별 확인 사항

- `8175392db042^`와 `8175392db042`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **사용 가능 여부 배지·channel 그리드·안내 목록에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 응답 가능 상태 배지·연락 채널 그리드·안내 목록` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:8175392db042:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **사용 가능 여부 배지·channel 그리드·안내 목록에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 연락처 상태·수단과 반복 안내 기반을 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 사용 가능 여부 배지·channel 그리드·안내 목록에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:8175392db042:END -->

### 18. `6fa3a9dc8665` — style(brutalist): note 목록과 anchor link 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 안내·근거 부족한 부분과 여정 주요 시점 카드

#### 커밋별 확인 사항

- `6fa3a9dc8665^`와 `6fa3a9dc8665`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **안내·근거 부족한 부분과 여정 주요 시점 카드에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 안내·근거 공백과 Journey 주요 시점 카드` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:6fa3a9dc8665:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **안내·근거 부족한 부분과 여정 주요 시점 카드에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 연락처 안내를 마감하고 여정 기준 카드를 엽니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 안내·근거 부족한 부분과 여정 주요 시점 카드에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:6fa3a9dc8665:END -->

### 19. `242ba8e66e0b` — style(brutalist): archive timeline과 track navigation 구성

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 여정 아카이브·현재 callout와 인터뷰 트랙 셸

#### 커밋별 확인 사항

- `242ba8e66e0b^`와 `242ba8e66e0b`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **여정 아카이브·현재 callout와 인터뷰 트랙 셸에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 여정 아카이브·현재 상태 강조와 인터뷰 트랙 셸` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:242ba8e66e0b:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **여정 아카이브·현재 callout와 인터뷰 트랙 셸에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 여정 아카이브와 인터뷰 페이지 내부 탐색 기반을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 여정 아카이브·현재 callout와 인터뷰 트랙 셸에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:242ba8e66e0b:END -->

### 20. `95e55eda6c51` — style(brutalist): track 목록과 question prompt 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 트랙 목록·트랙·질문 배치 순서

#### 커밋별 확인 사항

- `95e55eda6c51^`와 `95e55eda6c51`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **트랙 목록·트랙·질문 배치 순서에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 트랙 목록과 트랙·질문 계층` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:95e55eda6c51:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **트랙 목록·트랙·질문 배치 순서에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 인터뷰 질문 배치 순서와 트랙 목록을 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 트랙 목록·트랙·질문 배치 순서에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:95e55eda6c51:END -->

### 21. `11f229d630e9` — style(brutalist): 답변 근거와 footer lead 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 질문 참조·답변 근거·빈 답변·푸터 대표

#### 커밋별 확인 사항

- `11f229d630e9^`와 `11f229d630e9`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **질문 참조·답변 근거·빈 답변·푸터 대표에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 질문 참조·답변 근거·빈 답변·푸터 대표 문구` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:11f229d630e9:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **질문 참조·답변 근거·빈 답변·푸터 대표에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 답변 근거와 빈 답변 표현, 푸터 도입부를 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 질문 참조·답변 근거·빈 답변·푸터 대표에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:11f229d630e9:END -->

### 22. `b170c73a36d0` — style(brutalist): footer metadata와 blink 동작 구성

- **중요도:** B
- **태그:** RENDERER, SEO
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 푸터 메타데이터·dashed 빈 상태·흐르는·깜박이는 애니메이션

#### 커밋별 확인 사항

- `b170c73a36d0^`와 `b170c73a36d0`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **푸터 메타데이터·dashed 빈 상태·흐르는·깜박이는 애니메이션에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 푸터 메타데이터·점선 빈 상태·흐르기·깜박임 애니메이션` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b170c73a36d0:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **푸터 메타데이터·dashed 빈 상태·흐르는·깜박이는 애니메이션에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 푸터와 공용 빈 블록, 두 애니메이션을 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 푸터 메타데이터·dashed 빈 상태·흐르는·깜박이는 애니메이션에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b170c73a36d0:END -->

### 23. `f810c49022be` — style(brutalist): tablet grid 재배치

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 980px 태블릿 그리드 재배치

#### 커밋별 확인 사항

- `f810c49022be^`와 `f810c49022be`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **980px 태블릿 그리드 재배치에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 980px 태블릿 그리드 재배치` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f810c49022be:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **980px 태블릿 그리드 재배치에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 데스크톱 그리드를 태블릿 열·읽기 순서로 재배치합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 980px 태블릿 그리드 재배치에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:f810c49022be:END -->

### 24. `b57da6a41419` — style(brutalist): mobile header와 hero 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 720px 브라우저 기본 `<details>` 메뉴와 세로로 쌓인 헤더·소개 영역

#### 커밋별 확인 사항

- `b57da6a41419^`와 `b57da6a41419`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **720px 이하의 기본 `<details>` 메뉴와 세로로 쌓이는 헤더·소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가 및 기존 선언의 이동** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 720px 브라우저 기본 펼침 메뉴와 세로 배치 헤더·소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b57da6a41419:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **부모 커밋과 비교해 720px 이하의 기본 `<details>` 메뉴와 세로로 쌓이는 헤더·소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가 및 기존 선언의 이동**를 완결하는 선택자·미디어 규칙 묶음이 없거나 앞 단계의 일부만 존재했습니다.
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 데스크톱 탐색을 `<details>`로 바꾸고 상태·디버그·소개 영역을 기술 쌓아 배치합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모 커밋과 비교해 720px 이하의 기본 `<details>` 메뉴와 세로로 쌓이는 헤더·소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가 및 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b57da6a41419:END -->

### 25. `8168bc76c3e3` — style(brutalist): mobile 프로젝트와 상세 화면 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 모바일 지표·프로젝트 행·상세·갤러리 재배치

#### 커밋별 확인 사항

- `8168bc76c3e3^`와 `8168bc76c3e3`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **모바일 지표·프로젝트 행·상세·갤러리 재배치에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 모바일 지표·프로젝트 행·상세·갤러리 재배치` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:8168bc76c3e3:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **모바일 지표·프로젝트 행·상세·갤러리 재배치에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 홈·프로젝트·상세·선별 기록 그리드를 좁은 화면 읽기 순서로 바꿉니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 모바일 지표·프로젝트 행·상세·갤러리 재배치에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:8168bc76c3e3:END -->

### 26. `5551f3fdbb94` — style(brutalist): mobile profile과 resume 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 모바일 프로필·선별 기록·이력서·연락처·현재 위치

#### 커밋별 확인 사항

- `5551f3fdbb94^`와 `5551f3fdbb94`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **모바일 프로필·선별 기록·이력서·연락처·현재 위치에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 모바일 프로필·선별 기록·이력서·연락처·현재 위치` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5551f3fdbb94:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **모바일 프로필·선별 기록·이력서·연락처·현재 위치에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 소개·이력서·연락처 계열의 좁은 화면 배치를 이어서 완성합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 모바일 프로필·선별 기록·이력서·연락처·현재 위치에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5551f3fdbb94:END -->

### 27. `7c08aea7a2f7` — style(brutalist): mobile 여정과 interview 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 모바일 여정·인터뷰·푸터·누락된 페이지

#### 커밋별 확인 사항

- `7c08aea7a2f7^`와 `7c08aea7a2f7`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **모바일 여정·인터뷰·푸터·누락된 페이지에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 모바일 여정·인터뷰·푸터·찾을 수 없음 페이지` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:7c08aea7a2f7:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **모바일 여정·인터뷰·푸터·누락된 페이지에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 여정·인터뷰·푸터·복구 상태의 모바일 레이아웃을 마감합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 모바일 여정·인터뷰·푸터·누락된 페이지에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:7c08aea7a2f7:END -->

### 28. `077ff3d49f30` — style(brutalist): 소형 화면과 인쇄 경계 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Brutalist 시각 표현 규칙의 430px 강화·동작 줄이기 설정·print

#### 커밋별 확인 사항

- `077ff3d49f30^`와 `077ff3d49f30`를 비교하고 `src/designs/brutalist/brutalist.module.css`에서 **430px 강화·동작 줄이기 설정·print에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 시각 표현 규칙의 430px 화면 보강·동작 줄이기·인쇄` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `동작 줄이기·포커스·의미 구조 보완은 CSS와 DOM 규칙의 일부만 다룹니다. 실제 WCAG 적합성이나 모든 보조기기에서의 동작까지 단독으로 보장하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:077ff3d49f30:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **430px 강화·동작 줄이기 설정·print에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/brutalist/brutalist.module.css`에 추가된 CSS가 최소 화면 크기, 애니메이션 제거, 인쇄용 배경과 내비게이션 처리 규칙을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Brutalist 스타일시트에 남습니다.
- **정적 검토:** `src/designs/brutalist/brutalist.module.css`의 부모와의 변경 차이에서 430px 강화·동작 줄이기 설정·print에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/brutalist/brutalist.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 동작 줄이기 설정·포커스·의미상 보조는 CSS·DOM 규칙의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:077ff3d49f30:END -->

### 29. `3e6ec5262bdd` — feat(brutalist): 콘텐츠와 탐색 조회 도우미 추가

- **중요도:** B
- **태그:** CONTENT, ROUTING, RENDERER
- **개발 과정에서의 역할:** 콘텐츠·탐색 변환 함수 레이어

#### 커밋별 확인 사항

- `3e6ec5262bdd^`와 `3e6ec5262bdd`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **렌더러 보존 href, 템플릿·태그·그룹·지표·탐색 도우미 함수** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `콘텐츠·탐색 변환 코드` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:3e6ec5262bdd:BEGIN -->
- **직전 상태:** 직전 상태에는 **렌더러 보존 href, 템플릿·태그·그룹·지표·탐색 도우미 함수** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** Brutalist가 원본 콘텐츠를 반복 탐색하지 않도록 링크·프로젝트 그룹·지표·탐색 조회 도우미 함수를 한 모듈에 모은다. 이 시점 도우미 함수는 향후 view들이 공유합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 렌더러 보존 href, 템플릿·태그·그룹·지표·탐색 도우미 함수를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:3e6ec5262bdd:END -->

### 30. `08a2b0c0998f` — feat(brutalist): route 레이블과 기본 shell 구성

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** 공용 셸 시작

#### 커밋별 확인 사항

- `08a2b0c0998f^`와 `08a2b0c0998f`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **누락 없는 라우트 문구 해석 함수와 `BrutalistShell` 최상위·건너뛰기·헤더·main** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `공용 셸 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:08a2b0c0998f:BEGIN -->
- **직전 상태:** 직전 상태에는 **누락 없는 라우트 문구 해석 함수와 `BrutalistShell` 최상위·건너뛰기·헤더·main** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 여덟 라우트에 대한 문구 해석 함수를 전환로 닫고 최상위, 건너뛰기 링크, 헤더, 주요 콘텐츠 영역을 만듭니다. 아직 탐색·푸터·본문은 후속 커밋에서 채워진다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 누락 없는 라우트 문구 해석 함수와 `BrutalistShell` 최상위·건너뛰기·헤더·main를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:08a2b0c0998f:END -->

### 31. `cf2fdb36f9fc` — feat(brutalist): 주 탐색과 모바일 메뉴 추가

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** 기준 탐색

#### 커밋별 확인 사항

- `cf2fdb36f9fc^`와 `cf2fdb36f9fc`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **데스크톱·모바일 탐색, 현재 상태, 디버그, `ActionLink` 내부·외부·mailto 분기** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `기준 탐색 구성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:cf2fdb36f9fc:BEGIN -->
- **직전 상태:** 직전 상태에는 **데스크톱·모바일 탐색, 현재 상태, 디버그, `ActionLink` 내부·외부·mailto 분기** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 기준 탐색을 데스크톱과 브라우저 기본 펼침 요소 양쪽에 연결하고 내부 라우트에는 렌더러·디버그 쿼리를 보존합니다. 외부·mailto는 기준 속성을 사용합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 데스크톱·모바일 탐색, 현재 상태, 디버그, `ActionLink` 내부·외부·mailto 분기를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:cf2fdb36f9fc:END -->

### 32. `5b44afbc46ef` — feat(brutalist): footer와 홈 히어로 연결

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 푸터 + 홈 시작

#### 커밋별 확인 사항

- `5b44afbc46ef^`와 `5b44afbc46ef`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **노출 위치로 필터링한 `footerLinks`, `HomeRoute` 소개 영역, 현재 연도·지표·설정된 섹션 loop** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `푸터와 홈 화면 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5b44afbc46ef:BEGIN -->
- **직전 상태:** 직전 상태에는 **노출 위치로 필터링한 `footerLinks`, `HomeRoute` 소개 영역, 현재 연도·지표·설정된 섹션 loop** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 푸터 링크를 노출 위치로 필터링하고 홈 소개 영역을 기준 프로필·화면 구성에 연결합니다. 홈은 설정된 섹션 순서를 순회하지만 일부 섹션은 아직 후속 구현 전입니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 노출 위치로 필터링한 `footerLinks`, `HomeRoute` 소개 영역, 현재 연도·지표·설정된 섹션 loop를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5b44afbc46ef:END -->

### 33. `b477ba477127` — feat(brutalist): 홈 섹션 공용 프리미티브 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈·아카이브 공용 컴포넌트

#### 커밋별 확인 사항

- `b477ba477127^`와 `b477ba477127`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **신호 표시줄, 번호가 있는 섹션 헤더, 렌더러 보존 프로젝트 행, 연락처 공용 컴포넌트** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈·아카이브 공용 요소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b477ba477127:BEGIN -->
- **직전 상태:** 직전 상태에는 **신호 표시줄, 번호가 있는 섹션 헤더, 렌더러 보존 프로젝트 행, 연락처 공용 컴포넌트** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 홈과 아카이브가 반복 사용할 시각·링크 단위를 내부 컴포넌트로 추출해 라우트별 마크업 중복을 줄입니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 신호 표시줄, 번호가 있는 섹션 헤더, 렌더러 보존 프로젝트 행, 연락처 공용 컴포넌트를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b477ba477127:END -->

### 34. `b30b9b1c3505` — feat(brutalist): 대표 작업과 작업 원칙 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 중간 + 프로젝트 소개 영역

#### 커밋별 확인 사항

- `b30b9b1c3505^`와 `b30b9b1c3505`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **신호·대표·시스템 섹션과 프로젝트 라우트 소개 영역** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈 중간 영역과 프로젝트 페이지 소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b30b9b1c3505:BEGIN -->
- **직전 상태:** 직전 상태에는 **신호·대표·시스템 섹션과 프로젝트 라우트 소개 영역** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 설정된 홈 순서에 대표 프로젝트와 작업 원칙을 추가하고 프로젝트 아카이브의 소개 영역·지표 경계를 시작합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 신호·대표·시스템 섹션과 프로젝트 라우트 소개 영역을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b30b9b1c3505:END -->

### 35. `85ea663aaf19` — feat(brutalist): 홈 여정과 프로젝트 archive 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 완료 + 프로젝트 아카이브

#### 커밋별 확인 사항

- `85ea663aaf19^`와 `85ea663aaf19`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`slice(-4).reverse()` 최근 여정 복사, 연락처 영역, 프로젝트 그룹, 명시적인 빈 아카이브** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈 완성과 프로젝트 아카이브` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:85ea663aaf19:BEGIN -->
- **직전 상태:** 직전 상태에는 **`slice(-4).reverse()` 최근 여정 복사, 연락처 영역, 프로젝트 그룹, 명시적인 빈 아카이브** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 최근 네 여정을 복사한 배열에서 역순으로 표시해 원본을 mutate하지 않습니다. 프로젝트 라우트는 기준 그룹을 순회하고 비면 명시적 빈 블록을 보인다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 `slice(-4).reverse()` 최근 여정 복사, 연락처 영역, 프로젝트 그룹, 명시적인 빈 아카이브를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:85ea663aaf19:END -->

### 36. `d6b9a99e11ae` — feat(brutalist): 프로젝트 상세 표시 프리미티브 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 공용 컴포넌트

#### 커밋별 확인 사항

- `d6b9a99e11ae^`와 `d6b9a99e11ae`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **optimized 미디어, 순서가 지정된 동작, 문구·목록 섹션 셸, 페이지 문구, 선별 기록 제목** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 공용 요소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:d6b9a99e11ae:BEGIN -->
- **직전 상태:** 직전 상태에는 **optimized 미디어, 순서가 지정된 동작, 문구·목록 섹션 셸, 페이지 문구, 선별 기록 제목** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 상세 라우트가 사용할 미디어·동작·근거 공용 컴포넌트를 먼저 정의합니다. 동작 순서와 선택적 미디어 처리 규칙이 라우트 본문 밖으로 분리됩니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 optimized 미디어, 순서가 지정된 동작, 문구·목록 섹션 셸, 페이지 문구, 선별 기록 제목을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:d6b9a99e11ae:END -->

### 37. `b8268f47e89a` — feat(brutalist): 프로젝트 상세 hero와 소개 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 유효한·누락된 구분 지점

#### 커밋별 확인 사항

- `b8268f47e89a^`와 `b8268f47e89a`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`ProjectDetailRoute` 해석되지 않은 검사, 소개 영역, 핵심 정보, 동작, 도입부** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세의 정상·누락 처리` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `05b838d52a8b`와 `1ea2a1345b76`가 본문·결과를 완성합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b8268f47e89a:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectDetailRoute` 해석되지 않은 검사, 소개 영역, 핵심 정보, 동작, 도입부** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 프로젝트가 없으면 모든 필드 접근 전에 누락된 뷰를 반환합니다. 유효하면 소개 영역·핵심 정보·동작·도입부를 구성하고 아카이브 복귀 경로를 보존합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 `ProjectDetailRoute` 해석되지 않은 검사, 소개 영역, 핵심 정보, 동작, 도입부를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** `05b838d52a8b`와 `1ea2a1345b76`가 본문·결과를 완성합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b8268f47e89a:END -->

### 38. `05b838d52a8b` — feat(brutalist): 프로젝트 상세 본문과 gallery 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 근거 본문

#### 커밋별 확인 사항

- `05b838d52a8b^`와 `05b838d52a8b`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **문제·해결책·설계·화면 캡처·해석된 기술 스택** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 근거 본문` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:05b838d52a8b:BEGIN -->
- **직전 상태:** 직전 상태에는 **문제·해결책·설계·화면 캡처·해석된 기술 스택** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 상세 본문과 갤러리를 구성하며 기술 스택 ID는 이미 해석된 표시 데이터를 사용합니다. 선택적 화면 캡처 배열은 존재하는 항목만 렌더링합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 문제·해결책·설계·화면 캡처·해석된 기술 스택을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:05b838d52a8b:END -->

### 39. `80724a26820b` — feat(brutalist): 프로필과 기술 소개 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 소개 식별 정보

#### 커밋별 확인 사항

- `80724a26820b^`와 `80724a26820b`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`AboutRoute`, 선택적 사진, 원칙, 포커스 분야, 기술 그룹** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `소개 페이지의 프로필 정보` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:80724a26820b:BEGIN -->
- **직전 상태:** 직전 상태에는 **`AboutRoute`, 선택적 사진, 원칙, 포커스 분야, 기술 그룹** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 프로필 식별 정보와 기술 영역을 기준 콘텐츠에서 구성하고 사진이 없으면 미디어 영역을 생략합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 `AboutRoute`, 선택적 사진, 원칙, 포커스 분야, 기술 그룹을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:80724a26820b:END -->

### 40. `3399b55c3aee` — feat(brutalist): 큐레이션과 경력 소개 구성

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** 소개 경력 + 기능 플래그로 제어되는 선별 기록

#### 커밋별 확인 사항

- `3399b55c3aee^`와 `3399b55c3aee`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **경력, `isSitePageEnabled`, 분류 프로젝트 해석·필터** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `소개 페이지 경력과 기능 설정에 따른 선별 기록` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:3399b55c3aee:BEGIN -->
- **직전 상태:** 직전 상태에는 **경력, `isSitePageEnabled`, 분류 프로젝트 해석·필터** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 경력 아카이브를 추가하고 선별 기록 지원 여부가 켜진 경우에만 기준·분류·제외·검토를 렌더링합니다. 잘못된 프로젝트 ID는 링크 없이 제외합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 경력, `isSitePageEnabled`, 분류 프로젝트 해석·필터를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:3399b55c3aee:END -->

### 41. `70cf13ef1715` — feat(brutalist): 이력 hero와 경력 요약 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 이력서 시작

#### 커밋별 확인 사항

- `70cf13ef1715^`와 `70cf13ef1715`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`ResumeRoute`, 식별 정보·사용 가능 여부, 선택적 다운로드, 요약, 경력** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `이력서 화면 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:70cf13ef1715:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ResumeRoute`, 식별 정보·사용 가능 여부, 선택적 다운로드, 요약, 경력** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 다운로드 URL 유무를 분기하고 번호가 있는 요약과 날짜가 있는 경력을 구성합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 `ResumeRoute`, 식별 정보·사용 가능 여부, 선택적 다운로드, 요약, 경력을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:70cf13ef1715:END -->

### 42. `1ea2a1345b76` — feat(brutalist): 프로젝트 결과와 의사결정 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 완료

#### 커밋별 확인 사항

- `1ea2a1345b76^`와 `1ea2a1345b76`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **주요 내용·결정·절충안·결과와 명시적인 빈 문구** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1ea2a1345b76:BEGIN -->
- **직전 상태:** 직전 상태에는 **주요 내용·결정·절충안·결과와 명시적인 빈 문구** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 각 근거 배열을 별도 섹션으로 표시하고 비어 있을 때 라우트 문구가 지정한 빈 메시지를 사용합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 주요 내용·결정·절충안·결과와 명시적인 빈 문구를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1ea2a1345b76:END -->

### 43. `5fa378250d64` — feat(brutalist): 선택 프로젝트와 이력 세부 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 이력서 완료

#### 커밋별 확인 사항

- `5fa378250d64^`와 `5fa378250d64`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **이력서 프로젝트 ID 해석·필터, 교육·학력·안내 빈 대체 처리** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `이력서 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:5fa378250d64:BEGIN -->
- **직전 상태:** 직전 상태에는 **이력서 프로젝트 ID 해석·필터, 교육·학력·안내 빈 대체 처리** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 선택 프로젝트 ID를 기준 프로젝트로 해석해 유효 항목만 남기고 교육·학력·안내를 추가합니다. 비어 있는 목록에는 명시적 대체 처리가 적용됩니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 이력서 프로젝트 ID 해석·필터, 교육·학력·안내 빈 대체 처리를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:5fa378250d64:END -->

### 44. `b535539ae016` — feat(brutalist): 연락 수단과 안내 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 연락처 라우트

#### 커밋별 확인 사항

- `b535539ae016^`와 `b535539ae016`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **우선 ID 우선, 노출 위치 대체 처리, 명시적인 빈 연락 채널, 안내** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `연락처 라우트` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b535539ae016:BEGIN -->
- **직전 상태:** 직전 상태에는 **우선 ID 우선, 노출 위치 대체 처리, 명시적인 빈 연락 채널, 안내** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 우선 연락처가 해석되지 않으면 노출 위치 기반 링크로 후퇴하고 그래도 비면 빈 블록을 렌더링합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 우선 ID 우선, 노출 위치 대체 처리, 명시적인 빈 연락 채널, 안내를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b535539ae016:END -->

### 45. `15a765ecb2aa` — feat(brutalist): 여정 milestone 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 여정 설명 시작

#### 커밋별 확인 사항

- `15a765ecb2aa^`와 `15a765ecb2aa`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **주요 시점, anchorProjectIds 해석·필터, 빈 설명** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `여정 설명 화면 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:15a765ecb2aa:BEGIN -->
- **직전 상태:** 직전 상태에는 **주요 시점, anchorProjectIds 해석·필터, 빈 설명** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 주요 시점의 프로젝트 참조를 유효 프로젝트로만 연결하고 주요 시점이 없으면 여정 빈 상태를 표시합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 주요 시점, anchorProjectIds 해석·필터, 빈 설명을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:15a765ecb2aa:END -->

### 46. `388446b1a982` — feat(brutalist): 여정 archive와 인터뷰 map 머리말 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 여정 완료 + 인터뷰 시작

#### 커밋별 확인 사항

- `388446b1a982^`와 `388446b1a982`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **날짜가 있는 여정 아카이브·현재 위치, 인터뷰 도입부·참조·트랙 프래그먼트 탐색, 프로젝트 맵** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `여정 완성과 인터뷰 시작` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:388446b1a982:BEGIN -->
- **직전 상태:** 직전 상태에는 **날짜가 있는 여정 아카이브·현재 위치, 인터뷰 도입부·참조·트랙 프래그먼트 탐색, 프로젝트 맵** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 여정을 전체 아카이브와 현재 상태까지 완성하고 인터뷰 소개 영역과 트랙 목록을 엽니다. 인터뷰 답변 결합용 프로젝트 맵을 준비합니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 날짜가 있는 여정 아카이브·현재 위치, 인터뷰 도입부·참조·트랙 프래그먼트 탐색, 프로젝트 맵을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:388446b1a982:END -->

### 47. `f3fc6200a45b` — feat(brutalist): 인터뷰 근거 archive 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 인터뷰 근거

#### 커밋별 확인 사항

- `f3fc6200a45b^`와 `f3fc6200a45b`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **트랙·질문·참조, 유효한 프로젝트 답변, 연결된 근거가 없는 대체 처리** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `인터뷰 근거` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f3fc6200a45b:BEGIN -->
- **직전 상태:** 직전 상태에는 **트랙·질문·참조, 유효한 프로젝트 답변, 연결된 근거가 없는 대체 처리** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 각 답변의 프로젝트를 맵에서 찾아 유효하면 상세 링크를 만들고, 없거나 답변 배열이 비면 명시적 근거 없음 문구를 보인다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 트랙·질문·참조, 유효한 프로젝트 답변, 연결된 근거가 없는 대체 처리를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:f3fc6200a45b:END -->

### 48. `da8e59d56783` — feat(brutalist): 인터뷰 근거 공백 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 인터뷰 부족한 부분

#### 커밋별 확인 사항

- `da8e59d56783^`와 `da8e59d56783`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **부족한 부분 제목·본문·항목과 빈 목록 처리** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `인터뷰의 미해결 항목` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:da8e59d56783:BEGIN -->
- **직전 상태:** 직전 상태에는 **부족한 부분 제목·본문·항목과 빈 목록 처리** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 인터뷰 맵 마지막에 선언된 근거 부족한 부분을 별도 섹션으로 추가해 미지원 영역을 숨기지 않습니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 부족한 부분 제목·본문·항목과 빈 목록 처리를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:da8e59d56783:END -->

### 49. `e6268c4b7c74` — refactor(brutalist): 내부 helper 공개 범위 정리

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** 모듈 공개 범위 축소

#### 커밋별 확인 사항

- `e6268c4b7c74^`와 `e6268c4b7c74`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **콘텐츠 변환 함수, 공용 컴포넌트, 각 뷰의 `export` 제거** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `모듈 공개 API 축소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `caa7df81d899`가 유일한 공개 라우트 진입점을 추가합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:e6268c4b7c74:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **콘텐츠 변환 함수, 공용 컴포넌트, 각 뷰의 `export` 제거**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** 등록부가 알 필요 없는 도우미 함수·뷰를 모듈 내부로 바꿉니다. 실행 시점 출력을 바꾸는 새 분기보다 소유 주체를 외부 진입점 하나로 수렴시키는 변화입니다.
- **정적 검토:** `src/designs/brutalist/brutalist-route.tsx`의 콘텐츠 변환 함수, 공용 컴포넌트, 각 뷰의 `export` 제거를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/designs/brutalist/brutalist-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** `caa7df81d899`가 유일한 공개 라우트 진입점을 추가합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:e6268c4b7c74:END -->

### 50. `caa7df81d899` — feat(brutalist): 모든 route를 renderer에 통합

- **중요도:** A
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** Brutalist 공개 API와 여덟 라우트 분기의 최종 경계

#### 커밋별 확인 사항

- `caa7df81d899^`와 `caa7df81d899`를 비교하고 `src/designs/brutalist/brutalist-route.tsx`에서 **`BrutalistRoute`, 누락 없는 전환, 공용 셸 줄바꿈** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Brutalist 공개 API와 8개 라우트 분기의 최종 진입점` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `실행 시점에 알 수 없는 라우트를 위한 별도 기본 화면은 없으며, 닫힌 유니언의 컴파일 시점 누락 검사에 의존합니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 1의 `dd71d28143a8` 활성화가 이 진입점을 지연 로딩 등록부에 연결합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:caa7df81d899:BEGIN -->
- **직전 상태:** 직전 상태에는 **`BrutalistRoute`, 누락 없는 전환, 공용 셸 줄바꿈** 관련 라우트·컴포넌트가 아직 없었습니다.
- **구현 결정:** 모듈 내부 view들이 완성된 뒤 exported `BrutalistRoute` 하나가 종류가 구분된 라우트를 여덟 view로 매핑하고 결과를 공용 셸에 넣는다. 외부 등록부는 도우미 함수·view·콘텐츠 변환 함수를 가져오기할 수 없으며 사이트 전체 셸이 모든 라우트에 적용됩니다.
- **파일·심볼:** `src/designs/brutalist/brutalist-route.tsx`에서 `BrutalistRoute`, 누락 없는 전환, 공용 셸 줄바꿈를 확인했습니다.
- **소유권:** `src/designs/brutalist/brutalist-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `Brutalist 공개 API와 8개 라우트 분기의 최종 진입점`. 실행 중 알 수 없는 라우트에 대한 별도 기본 화면는 없고 닫힌 유니언의 컴파일 시점 누락 검사에 의존합니다.
- **역사적 연결:** 개발 과정 1의 `dd71d28143a8` 활성화가 이 진입점을 지연 로딩 등록부에 연결합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

```tsx
// caa7df81d899 · src/designs/brutalist/brutalist-route.tsx
export function BrutalistRoute(props: BrutalistRouteProps) {
  return <BrutalistShell {...props}>{renderRoute(props)}</BrutalistShell>;
}
```
<!-- LEARNER-ANSWER:commit:caa7df81d899:END -->

## 5. 불변 조건 변화

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:invariant:BEGIN -->
최종 불변 조건은 Brutalist 모듈이 라우트 진입점 하나만 공개하고, 셸·탐색·링크 규칙·콘텐츠 변환 함수·view가 같은 모듈의 내부 구현으로 유지되며, 모바일·동작 줄이기 설정·print와 누락되거나 빈 상태가 시각 표현 규칙에 포함된다는 것입니다.

- 도입·확장·폐쇄의 순서는 커밋 목록에 고정했습니다.
- 중요도 B 구축은 라우트·스타일 표시 영역을 단계적으로 넓히고, A·중요도 S 커밋은 소유 주체·분기·검증 불변 조건을 바꿉니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:invariant:END -->

## 6. 실패 → 수정 → 테스트와 소유 주체 변화

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:relations:BEGIN -->
스타일은 라우트 전체의 데스크톱 문법을 순차 구축하고 980/720/모바일/430/동작 줄이기 설정·print로 좁힙니다. TSX는 변환 함수·셸·탐색 뒤 홈→프로젝트→Detail→소개→이력서→연락처→여정→인터뷰를 완성합니다. `e6268c4b7c74`가 도우미 함수 공개를 회수하고 `caa7df81d899`가 공개 라우트 진입점을 확정합니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:relations:END -->

## 7. 최종 구성과 실행 순서

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:flow:BEGIN -->
등록부 진입점 → `BrutalistRoute` → 닫힌 라우트 전환 → 라우트 view → 내부 콘텐츠·탐색 도우미 함수와 시각 공용 컴포넌트 → `BrutalistShell` 최상위·헤더·main·푸터 순서입니다.

각 단계에서 선택적·빈·누락된 참조가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 커밋의 보장하지 않는 범위에 남겼습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:flow:END -->

## 8. 실행 및 검증 근거

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:runtime:BEGIN -->
- **실행한 저장소 테스트·빌드:** 없음.
- **정적 확인:** 지정 브랜치의 커밋 분류, 커밋 bodies, 해당 커밋의 변경 내용과 변경 이력의 파일 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 컨테이너에서 직접 복제본 시 DNS가 `github.com`을 해석하지 못해 변경 이력의 worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next 빌드 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** 코드·테스트 구현의 존재와 범위는 정적 검토로 확인했고, 실행 성공·실패는 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:runtime:END -->

## 9. 학습 완료 확인

<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·제목·중요도·태그를 커밋 목록과 커밋 섹션에서 동일하게 유지했습니다.
- [x] 각 SHA에 구체적인 파일과 심볼·선택자·라우트 포커스를 기록했습니다.
- [x] 이전 상태, 소유 주체, 누락·대체 처리, 보장 범위와 보장하지 않는 범위, 후속 관계를 채웠습니다.
- [x] S/A/B 깊이를 구분했습니다.
- [x] 실행하지 않은 테스트를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:03-brutalist-design-system-construction.md:checks:END -->
===== END FILE: 03-brutalist-design-system-construction.md =====

===== BEGIN FILE: 04-cinematic-design-system-construction.md =====
# 개발 과정: Cinematic 디자인 구성

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `05-full-site-visual-systems`
>
> 1단계 검토에서 확정한 기준 문서 틀입니다. 2단계에서는 답변 식별 속성 내부만 채웁니다.

## 0. 범위와 기준

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md`와 해당 커밋의 정확한 메타데이터를 기준으로 고정했습니다.
- **개발 과정 범위:** Cinematic CSS 모듈과 라우트 모듈 구축을 포함합니다. 등록부 활성화·API 연결은 개발 과정 1에 둡니다.
- 다른 브랜치, 최종 HEAD의 후대 구현, 실행하지 않은 명령 결과를 사용하지 않습니다.

## 1. 개발 과정 목표

Cinematic의 암실 색상표와 이미지 중심 장 문법, 공용 프레임·미디어, 사이트 모든 라우트 조립과 참조·빈 상태 경계를 복원합니다.

### 고정된 불변 조건

최종 불변 조건은 라우트가 공통 `Frame`과 `Media` 구분 지점을 사용하고 내부 링크가 선택 상태를 보존하며, 라우트별 콘텐츠 결합과 누락은 명시적으로 처리하되 각 라우트가 보장하지 않는 빈·참조 상태도 그대로 남긴다는 것입니다.

## 2. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `74a27c95eb1c` | style(cinematic): 암실 palette와 shell 기초 구성 | B | ROUTING, RENDERER | Cinematic 시각 표현 규칙의 어두운 범위가 제한된 tokens·선택·포커스·건너뛰기 링크·고정형 반투명 헤더 |
| 2 | `197c0781f1b9` | feat(cinematic): 링크와 chapter 표기 프리미티브 추가 | B | RENDERER | 탐색 공용 컴포넌트 |
| 3 | `3b72294a0fd7` | style(cinematic): 모바일 탐색과 hero 매체 구성 | B | ROUTING, RENDERER | Cinematic 시각 표현 규칙의 브라우저 기본 모바일 펼침 요소와 이미지 중심 소개 영역 |
| 4 | `e2dbb1b7c7d0` | feat(cinematic): 공용 frame과 media 추가 | B | RENDERER | 공용 사이트 전체 프레임 |
| 5 | `22c4593809bf` | feat(cinematic): 프로젝트 chapter 추가 | B | RENDERER | reusable 프로젝트 장 |
| 6 | `bb7a742122fd` | style(cinematic): chapter와 archive 지면 구성 | B | RENDERER | Cinematic 시각 표현 규칙의 장문 장·아카이브·고정 복사·미디어 마우스 오버·두 갈래 패널 |
| 7 | `29430d7dfe67` | feat(cinematic-home): 소개와 대표 프로젝트 구성 | B | RENDERER | 홈 라우트 |
| 8 | `f417e3e70b1f` | feat(cinematic-projects): 프로젝트 archive 구성 | B | RENDERER | 프로젝트 아카이브 |
| 9 | `1f4c35853502` | style(cinematic): 상세와 이력 grid 구성 | B | RENDERER | Cinematic 시각 표현 규칙의 프로젝트 근거·프로필 essays·이력서 그리드 |
| 10 | `2e9f70067daf` | feat(cinematic-project): 상세 hero와 매체 구성 | B | RENDERER | 프로젝트 상세 구분 지점 |
| 11 | `2f404402a2ea` | feat(cinematic-project): 상세 서사와 gallery 구성 | B | RENDERER | 프로젝트 상세 완료 |
| 12 | `95ee01decc8f` | style(cinematic): 프로필과 콘텐츠 section 구성 | B | CONTENT, RENDERER | Cinematic 시각 표현 규칙의 프로필 핵심 정보·장문 섹션·시간순 기록·근거·연락처·부족한 부분 |
| 13 | `4eefc512d05c` | feat(cinematic-about): 프로필과 경력 소개 구성 | B | RENDERER | 소개 라우트 |
| 14 | `ee692d893a11` | feat(cinematic-about): 큐레이션 archive 구성 | B | CONTENT, RENDERER | 기능 플래그로 제어되는 선별 기록 |
| 15 | `52f13fcc5a12` | style(cinematic): 여정 timeline과 답변 근거 구성 | B | RENDERER | Cinematic 시각 표현 규칙의 주요 시점·타임라인·현재 위치·인터뷰 근거 문법 |
| 16 | `7cc23349f59f` | feat(cinematic): 이력과 연락 route 구성 | B | ROUTING, RENDERER | 이력서 및 연락처 |
| 17 | `c3aba5da6a10` | style(cinematic): 인터뷰 근거와 반응형 동작 구성 | B | RENDERER | Cinematic 시각 표현 규칙의 인터뷰 근거 완료·980/640 재배치·동작 줄이기 설정 |
| 18 | `bddb3cc18eed` | feat(cinematic-journey): 여정 archive 구성 | B | RENDERER | 여정 라우트 |
| 19 | `2a0f0aadee1c` | feat(cinematic-interview): 인터뷰 근거 map 구성 | B | RENDERER | 인터뷰 라우트 |

## 3. 변경 전 기준

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:baseline:BEGIN -->
- **직전 상태:** 공통 위임은 있었지만 Cinematic 전용 프레임, 미디어, 링크, 장, 반응형 레이아웃과 라우트 view가 없었습니다.
- **경계 판단:** Cinematic CSS 모듈과 라우트 모듈 구축을 포함합니다. 등록부 활성화·API 연결은 개발 과정 1에 둡니다.
- **복원 기준:** 각 커밋의 부모 커밋과 해당 SHA 파일 트리만 사용하고 최종 HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:baseline:END -->

## 4. 커밋별 복원 기록

### 1. `74a27c95eb1c` — style(cinematic): 암실 palette와 shell 기초 구성

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** Cinematic 시각 표현 규칙의 어두운 범위가 제한된 tokens·선택·포커스·건너뛰기 링크·고정형 반투명 헤더

#### 커밋별 확인 사항

- `74a27c95eb1c^`와 `74a27c95eb1c`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **어두운 범위가 제한된 tokens·선택·포커스·건너뛰기 링크·고정형 반투명 헤더에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic 시각 표현 규칙의 디자인 범위에 한정된 어두운 토큰·선택 영역·포커스·본문 건너뛰기 링크·고정 유리 효과 헤더` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:74a27c95eb1c:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **어두운 범위가 제한된 tokens·선택·포커스·건너뛰기 링크·고정형 반투명 헤더에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/cinematic/cinematic.module.css`에 추가된 CSS가 암실 색상표와 셸 최상위·접근성 계약을 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Cinematic 스타일시트에 남습니다.
- **정적 검토:** `src/designs/cinematic/cinematic.module.css`의 부모와의 변경 차이에서 어두운 범위가 제한된 tokens·선택·포커스·건너뛰기 링크·고정형 반투명 헤더에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/cinematic/cinematic.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:74a27c95eb1c:END -->

### 2. `197c0781f1b9` — feat(cinematic): 링크와 chapter 표기 프리미티브 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 탐색 공용 컴포넌트

#### 커밋별 확인 사항

- `197c0781f1b9^`와 `197c0781f1b9`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`routeHref`, `isCurrentNavigation`, `CinematicLink`, `ChapterLabel`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `탐색 공용 요소` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:197c0781f1b9:BEGIN -->
- **직전 상태:** 직전 상태에는 **`routeHref`, `isCurrentNavigation`, `CinematicLink`, `ChapterLabel`** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 내부 라우트는 cinematic·디버그 상태를 보존하고 외부·mailto는 기준로 분기합니다. 장 문구를 반복 가능한 DOM 단위로 만듭니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `routeHref`, `isCurrentNavigation`, `CinematicLink`, `ChapterLabel`를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:197c0781f1b9:END -->

### 3. `3b72294a0fd7` — style(cinematic): 모바일 탐색과 hero 매체 구성

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** Cinematic 시각 표현 규칙의 브라우저 기본 모바일 펼침 요소와 이미지 중심 소개 영역

#### 커밋별 확인 사항

- `3b72294a0fd7^`와 `3b72294a0fd7`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **브라우저 기본 모바일 펼침 요소와 이미지 중심 소개 영역에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic 시각 표현 규칙의 브라우저 기본 모바일 펼침 메뉴와 이미지 중심 소개 영역` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:3b72294a0fd7:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **브라우저 기본 모바일 펼침 요소와 이미지 중심 소개 영역에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/cinematic/cinematic.module.css`에 추가된 CSS가 모바일 탐색과 2열 소개 영역·미디어 조립을 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Cinematic 스타일시트에 남습니다.
- **정적 검토:** `src/designs/cinematic/cinematic.module.css`의 부모와의 변경 차이에서 브라우저 기본 모바일 펼침 요소와 이미지 중심 소개 영역에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/cinematic/cinematic.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:3b72294a0fd7:END -->

### 4. `e2dbb1b7c7d0` — feat(cinematic): 공용 frame과 media 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 공용 사이트 전체 프레임

#### 커밋별 확인 사항

- `e2dbb1b7c7d0^`와 `e2dbb1b7c7d0`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`Frame`, `Media`, 기준 탐색·푸터, 최상위·main 셸** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `사이트 전체 공용 틀` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 후속 라우트 view들이 `Frame` 안에 본문만 제공합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:e2dbb1b7c7d0:BEGIN -->
- **직전 상태:** 직전 상태에는 **`Frame`, `Media`, 기준 탐색·푸터, 최상위·main 셸** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 모든 라우트가 사용할 최상위 프레임과 미디어 구분 지점을 만듭니다. 탐색·푸터는 기준 셸 콘텐츠를 읽고 현재 경로·템플릿·디버그 상태를 링크에 보존합니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `Frame`, `Media`, 기준 탐색·푸터, 최상위·main 셸을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** 후속 라우트 view들이 `Frame` 안에 본문만 제공합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:e2dbb1b7c7d0:END -->

### 5. `22c4593809bf` — feat(cinematic): 프로젝트 chapter 추가

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** reusable 프로젝트 장

#### 커밋별 확인 사항

- `22c4593809bf^`와 `22c4593809bf`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`ProjectChapter`, 고정 근거 복사, 미디어 링크, 접근 가능한 ARIA 문구** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `재사용 가능한 프로젝트 장` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:22c4593809bf:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectChapter`, 고정 근거 복사, 미디어 링크, 접근 가능한 ARIA 문구** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 프로젝트 요약·핵심 정보·동작과 미디어를 하나의 장으로 묶습니다. 아카이브와 홈이 같은 표현을 재사용하며 링크의 문구를 명시합니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `ProjectChapter`, 고정 근거 복사, 미디어 링크, 접근 가능한 ARIA 문구를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:22c4593809bf:END -->

### 6. `bb7a742122fd` — style(cinematic): chapter와 archive 지면 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Cinematic 시각 표현 규칙의 장문 장·아카이브·고정 복사·미디어 마우스 오버·두 갈래 패널

#### 커밋별 확인 사항

- `bb7a742122fd^`와 `bb7a742122fd`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **장문 장·아카이브·고정 복사·미디어 마우스 오버·두 갈래 패널에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic 시각 표현 규칙의 장문 장·아카이브·고정 문구·미디어 가리키기 효과·2개 패널` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:bb7a742122fd:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **장문 장·아카이브·고정 복사·미디어 마우스 오버·두 갈래 패널에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/cinematic/cinematic.module.css`에 추가된 CSS가 ProjectChapter와 아카이브가 사용할 장문 지면·고정 관계를 만듭니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Cinematic 스타일시트에 남습니다.
- **정적 검토:** `src/designs/cinematic/cinematic.module.css`의 부모와의 변경 차이에서 장문 장·아카이브·고정 복사·미디어 마우스 오버·두 갈래 패널에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/cinematic/cinematic.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:bb7a742122fd:END -->

### 7. `29430d7dfe67` — feat(cinematic-home): 소개와 대표 프로젝트 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 홈 라우트

#### 커밋별 확인 사항

- `29430d7dfe67^`와 `29430d7dfe67`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`HomeView`, 화면 구성 섹션 순서, 대표→모든 대체 처리, `slice(0, 4)`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈 라우트` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:29430d7dfe67:BEGIN -->
- **직전 상태:** 직전 상태에는 **`HomeView`, 화면 구성 섹션 순서, 대표→모든 대체 처리, `slice(0, 4)`** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 화면 구성에 정의된 섹션 ID 순서대로 노드를 순서대로 렌더링합니다. 대표가 비면 전체 프로젝트로 후퇴하고 최대 네 개를 장로 보여 줍니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `HomeView`, 화면 구성 섹션 순서, 대표→모든 대체 처리, `slice(0, 4)`를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:29430d7dfe67:END -->

### 8. `f417e3e70b1f` — feat(cinematic-projects): 프로젝트 archive 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 아카이브

#### 커밋별 확인 사항

- `f417e3e70b1f^`와 `f417e3e70b1f`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`ProjectsView`, 모든 프로젝트를 `ProjectChapter`로 표현하고 자릿수를 맞춘 개수** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 아카이브` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `프로젝트 아카이브가 비면 단순한 빈 목록이 되며, 별도의 복귀 안내 문구는 보장하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:f417e3e70b1f:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectsView`, 모든 프로젝트를 `ProjectChapter`로 표현하고 자릿수를 맞춘 개수** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 기준 프로젝트 전체를 장 형태로 순회하고 숫자 문구의 자릿수를 맞춥니다. 이 SHA에는 프로젝트 배열이 비었을 때 별도 빈 메시지가 없습니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `ProjectsView`, 모든 프로젝트를 `ProjectChapter`로 표현하고 자릿수를 맞춘 개수를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 빈 프로젝트 아카이브는 단순 빈 목록이 되며 명시적 복구 복사를 보장하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:f417e3e70b1f:END -->

### 9. `1f4c35853502` — style(cinematic): 상세와 이력 grid 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Cinematic 시각 표현 규칙의 프로젝트 근거·프로필 essays·이력서 그리드

#### 커밋별 확인 사항

- `1f4c35853502^`와 `1f4c35853502`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **프로젝트 근거·프로필 essays·이력서 그리드에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic 시각 표현 규칙의 프로젝트 근거·프로필 장문 설명·이력서 그리드` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:1f4c35853502:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **프로젝트 근거·프로필 essays·이력서 그리드에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/cinematic/cinematic.module.css`에 추가된 CSS가 상세·프로필·이력 장문 콘텐츠의 공용 그리드를 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Cinematic 스타일시트에 남습니다.
- **정적 검토:** `src/designs/cinematic/cinematic.module.css`의 부모와의 변경 차이에서 프로젝트 근거·프로필 essays·이력서 그리드에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/cinematic/cinematic.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:1f4c35853502:END -->

### 10. `2e9f70067daf` — feat(cinematic-project): 상세 hero와 매체 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 구분 지점

#### 커밋별 확인 사항

- `2e9f70067daf^`와 `2e9f70067daf`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`ProjectDetailView`, 해석되지 않은 검사, 아카이브 back 링크, 핵심 정보, 소개 영역 미디어** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 처리` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: `2f404402a2ea`가 전체 설명과 galleries를 추가합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:2e9f70067daf:BEGIN -->
- **직전 상태:** 직전 상태에는 **`ProjectDetailView`, 해석되지 않은 검사, 아카이브 back 링크, 핵심 정보, 소개 영역 미디어** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 프로젝트가 없으면 필드 접근 전에 누락된 뷰를 반환하고, 유효하면 프로젝트 핵심 정보와 소개 영역 미디어를 구성합니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `ProjectDetailView`, 해석되지 않은 검사, 아카이브 back 링크, 핵심 정보, 소개 영역 미디어를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** `2f404402a2ea`가 전체 설명과 galleries를 추가합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:2e9f70067daf:END -->

### 11. `2f404402a2ea` — feat(cinematic-project): 상세 서사와 gallery 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 프로젝트 상세 완료

#### 커밋별 확인 사항

- `2f404402a2ea^`와 `2f404402a2ea`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **선택적 설명·근거 배열, 해석된 기술 스택 대체 처리, 상세 링크, 소개 영역 화면 캡처 중복 제거** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 화면 완성` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:2f404402a2ea:BEGIN -->
- **직전 상태:** 직전 상태에는 **선택적 설명·근거 배열, 해석된 기술 스택 대체 처리, 상세 링크, 소개 영역 화면 캡처 중복 제거** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 문제·해결책·설계·결정·주요 내용·절충안·결과와 갤러리를 조건부로 구성합니다. 소개 영역 화면 캡처는 보조 이미지에서 제거해 중복 표시를 피하고, 기술 조회 실패 시 원본 ID 문구를 보존합니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 선택적 설명·근거 배열, 해석된 기술 스택 대체 처리, 상세 링크, 소개 영역 화면 캡처 중복 제거를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:2f404402a2ea:END -->

### 12. `95ee01decc8f` — style(cinematic): 프로필과 콘텐츠 section 구성

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** Cinematic 시각 표현 규칙의 프로필 핵심 정보·장문 섹션·시간순 기록·근거·연락처·부족한 부분

#### 커밋별 확인 사항

- `95ee01decc8f^`와 `95ee01decc8f`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **프로필 핵심 정보·장문 섹션·시간순 기록·근거·연락처·부족한 부분에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic 시각 표현 규칙의 프로필 사실·장문 섹션·연대기·근거·연락처·공백` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:95ee01decc8f:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **프로필 핵심 정보·장문 섹션·시간순 기록·근거·연락처·부족한 부분에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/cinematic/cinematic.module.css`에 추가된 CSS가 소개·이력서·연락처·여정·인터뷰가 공유할 콘텐츠 문법을 추가합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Cinematic 스타일시트에 남습니다.
- **정적 검토:** `src/designs/cinematic/cinematic.module.css`의 부모와의 변경 차이에서 프로필 핵심 정보·장문 섹션·시간순 기록·근거·연락처·부족한 부분에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/cinematic/cinematic.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:95ee01decc8f:END -->

### 13. `4eefc512d05c` — feat(cinematic-about): 프로필과 경력 소개 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 소개 라우트

#### 커밋별 확인 사항

- `4eefc512d05c^`와 `4eefc512d05c`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`AboutView`, 선택적 프로필 사진, 핵심 정보, 원칙, 기술 목록·그룹, 경력** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `소개 라우트` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:4eefc512d05c:BEGIN -->
- **직전 상태:** 직전 상태에는 **`AboutView`, 선택적 프로필 사진, 핵심 정보, 원칙, 기술 목록·그룹, 경력** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 기준 프로필·기술 목록·경력을 장문 cinematic 섹션으로 구성하고 사진 유무를 분기합니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `AboutView`, 선택적 프로필 사진, 핵심 정보, 원칙, 기술 목록·그룹, 경력을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:4eefc512d05c:END -->

### 14. `ee692d893a11` — feat(cinematic-about): 큐레이션 archive 구성

- **중요도:** B
- **태그:** CONTENT, RENDERER
- **개발 과정에서의 역할:** 기능 플래그로 제어되는 선별 기록

#### 커밋별 확인 사항

- `ee692d893a11^`와 `ee692d893a11`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`isSitePageEnabled`, 분류 프로젝트 ID 해석·필터, 제외 항목·nextReview** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `기능 설정에 따른 선별 기록` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:ee692d893a11:BEGIN -->
- **직전 상태:** 직전 상태에는 **`isSitePageEnabled`, 분류 프로젝트 ID 해석·필터, 제외 항목·nextReview** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 선별 기록 지원 여부가 켜졌을 때만 아카이브를 추가하고 분류 ID를 기준 프로젝트로 해석해 유효한 링크만 남깁니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `isSitePageEnabled`, 분류 프로젝트 ID 해석·필터, 제외 항목·nextRe뷰를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:ee692d893a11:END -->

### 15. `52f13fcc5a12` — style(cinematic): 여정 timeline과 답변 근거 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Cinematic 시각 표현 규칙의 주요 시점·타임라인·현재 위치·인터뷰 근거 문법

#### 커밋별 확인 사항

- `52f13fcc5a12^`와 `52f13fcc5a12`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **주요 시점·타임라인·현재 위치·인터뷰 근거 문법에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic 시각 표현 규칙의 주요 시점·타임라인·현재 위치·인터뷰 근거 표현 규칙` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 CSS만으로 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 보장할 수 없습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:52f13fcc5a12:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **주요 시점·타임라인·현재 위치·인터뷰 근거 문법에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/cinematic/cinematic.module.css`에 추가된 CSS가 여정과 인터뷰 라우트의 장문 타임라인·근거 지면을 정의합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Cinematic 스타일시트에 남습니다.
- **정적 검토:** `src/designs/cinematic/cinematic.module.css`의 부모와의 변경 차이에서 주요 시점·타임라인·현재 위치·인터뷰 근거 문법에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/cinematic/cinematic.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 해당 DOM 클래스가 실제 라우트에서 사용되는지, 브라우저마다 레이아웃이 정확한지는 이 CSS만으로 보장할 수 없습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:52f13fcc5a12:END -->

### 16. `7cc23349f59f` — feat(cinematic): 이력과 연락 route 구성

- **중요도:** B
- **태그:** ROUTING, RENDERER
- **개발 과정에서의 역할:** 이력서 및 연락처

#### 커밋별 확인 사항

- `7cc23349f59f^`와 `7cc23349f59f`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **이력서 선택한 프로젝트 ID 해석·필터, 선택적 다운로드, 연락처 우선→노출 위치 대체 처리, 빈 연락 채널** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `이력서와 연락처` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `Resume의 모든 선택 배열이 빈 경우 각각 별도 빈 상태를 제공하는 것은 아니며 안내 문구에만 명시적 대체 처리가 있습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:7cc23349f59f:BEGIN -->
- **직전 상태:** 직전 상태에는 **이력서 선택한 프로젝트 ID 해석·필터, 선택적 다운로드, 연락처 우선→노출 위치 대체 처리, 빈 연락 채널** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 이력서는 프로젝트 ID를 해석해 유효한 선택만 표시하고 다운로드를 조건부로 렌더링합니다. 연락처는 우선 링크가 없으면 노출 위치 기반 링크로 후퇴하고 그래도 없으면 명시적 빈 복사를 보인다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 이력서 선택한 프로젝트 ID 해석·필터, 선택적 다운로드, 연락처 우선→노출 위치 대체 처리, 빈 연락 채널를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이력서의 모든 선택 배열이 빈 경우 각각 별도 빈 상태를 제공하는 것은 아니며 안내에만 명시적 대체 처리가 있습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:7cc23349f59f:END -->

### 17. `c3aba5da6a10` — style(cinematic): 인터뷰 근거와 반응형 동작 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** Cinematic 시각 표현 규칙의 인터뷰 근거 완료·980/640 재배치·동작 줄이기 설정

#### 커밋별 확인 사항

- `c3aba5da6a10^`와 `c3aba5da6a10`를 비교하고 `src/designs/cinematic/cinematic.module.css`에서 **인터뷰 근거 완료·980/640 재배치·동작 줄이기 설정에 필요한 선택자·미디어 규칙·토큰이 추가되거나 기존 선언이 이동한 위치**를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Cinematic 시각 표현 규칙의 인터뷰 근거 완성·980px/640px 재배치·동작 줄이기` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `동작 줄이기·포커스·의미 구조 보완은 CSS와 DOM 규칙의 일부만 다룹니다. 실제 WCAG 적합성이나 모든 보조기기에서의 동작까지 단독으로 보장하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:c3aba5da6a10:BEGIN -->
- **직전 상태:** 직전 스타일시트에는 **인터뷰 근거 완료·980/640 재배치·동작 줄이기 설정에 필요한 선택자·미디어 규칙·토큰 묶음이 없거나 일부만 구현되어 있었습니다.**
- **변경:** 해당 SHA에서 `src/designs/cinematic/cinematic.module.css`에 추가된 CSS가 고정 배치와 그리드를 좁은 화면에서 정적 배치와 세로 쌓기로 바꾸고 애니메이션·전환 효과를 억제합니다. 이전 상태에는 이 선택자 묶음 또는 화면 너비 기준 재배치가 없었고, 이후 DOM에 적용할 배치 규칙을 이 시점부터 제공합니다. 이 시각 규칙은 콘텐츠나 라우트가 아니라 Cinematic 스타일시트에 남습니다.
- **정적 검토:** `src/designs/cinematic/cinematic.module.css`의 부모와의 변경 차이에서 인터뷰 근거 완료·980/640 재배치·동작 줄이기 설정에 필요한 선택자·미디어 규칙·토큰의 추가와 기존 선언의 이동을 확인했습니다.
- **담당 코드와 입력 범위:** DOM은 라우트 모듈이 계속 구성하고, 이 SHA에서 추가한 시각·레이아웃 규칙은 `src/designs/cinematic/cinematic.module.css` 스타일시트가 해당 디자인에 적용합니다.
- **비보장:** 동작 줄이기 설정·포커스·의미상 보조는 CSS·DOM 규칙의 일부만 다루며, 실제 WCAG 적합성이나 모든 보조기기 동작을 단독으로 보장하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:c3aba5da6a10:END -->

### 18. `bddb3cc18eed` — feat(cinematic-journey): 여정 archive 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 여정 라우트

#### 커밋별 확인 사항

- `bddb3cc18eed^`와 `bddb3cc18eed`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **주요 시점 기준 프로젝트 해석·필터, 날짜가 있는 아카이브, 직접 `item.projectId` URL, 현재 위치** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `여정 라우트` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `archive `projectId`가 기준 프로젝트 목록에 존재하는지는 이 렌더러가 검증하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:bddb3cc18eed:BEGIN -->
- **직전 상태:** 직전 상태에는 **주요 시점 기준 프로젝트 해석·필터, 날짜가 있는 아카이브, 직접 `item.projectId` URL, 현재 위치** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 주요 시점 기준 ID는 실제 프로젝트로 해석한 뒤 링크하지만 더 넓은 아카이브의 `item.projectId`는 존재 여부를 검사하지 않고 URL로 변환합니다. 설명과 chronological 이력·현재 상태를 세 구획으로 구성합니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 주요 시점 기준 프로젝트 해석·필터, 날짜가 있는 아카이브, 직접 `item.projectId` URL, 현재 위치를 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 아카이브 `projectId`가 기준 프로젝트에 존재한다는 보장은 이 렌더러에서 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:bddb3cc18eed:END -->

### 19. `2a0f0aadee1c` — feat(cinematic-interview): 인터뷰 근거 map 구성

- **중요도:** B
- **태그:** RENDERER
- **개발 과정에서의 역할:** 인터뷰 라우트

#### 커밋별 확인 사항

- `2a0f0aadee1c^`와 `2a0f0aadee1c`를 비교하고 `src/designs/cinematic/cinematic-route.tsx`에서 **`projectsById` 맵, 외부 참조, 트랙·질문 답변, 누락되거나 빈 근거, 부족한 부분** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `인터뷰 라우트` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 1의 `b8de57f130eb`가 완성된 라우트 진입점을 등록부에 활성화합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:2a0f0aadee1c:BEGIN -->
- **직전 상태:** 직전 상태에는 **`projectsById` 맵, 외부 참조, 트랙·질문 답변, 누락되거나 빈 근거, 부족한 부분** 관련 라우트·컴포넌트가 아직 없었습니다.
- **변경:** 답변과 프로젝트를 맵으로 해석하고 누락이면 `noMappedEvidence`를 표시합니다. 답변 배열과 부족한 부분 배열이 비어 있는 경우도 각각 대체 처리 복사를 사용합니다.
- **정적 검토:** `src/designs/cinematic/cinematic-route.tsx`의 `projectsById` 맵, 외부 참조, 트랙·질문 답변, 누락되거나 빈 근거, 부족한 부분을 확인했습니다.
- **담당 코드와 입력 범위:** `src/designs/cinematic/cinematic-route.tsx`가 이 DOM 조립·분기·링크·상태 표현을 담당하며 원본 콘텐츠의 수명은 변경하지 않습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** 개발 과정 1의 `b8de57f130eb`가 완성된 라우트 진입점을 등록부에 활성화합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:2a0f0aadee1c:END -->

## 5. 불변 조건 변화

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:invariant:BEGIN -->
최종 불변 조건은 라우트가 공통 `Frame`과 `Media` 구분 지점을 사용하고 내부 링크가 선택 상태를 보존하며, 라우트별 콘텐츠 결합과 누락은 명시적으로 처리하되 각 라우트가 보장하지 않는 빈·참조 상태도 그대로 남긴다는 것입니다.

- 도입·확장·폐쇄의 순서는 커밋 목록에 고정했습니다.
- 중요도 B 구축은 라우트·스타일 표시 영역을 단계적으로 넓히고, A·중요도 S 커밋은 소유 주체·분기·검증 불변 조건을 바꿉니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:invariant:END -->

## 6. 실패 → 수정 → 테스트와 소유 주체 변화

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:relations:BEGIN -->
CSS와 TSX가 교차하며 셸·장·아카이브·상세·프로필·이력서·여정·인터뷰 지면을 만든 뒤 반응형·동작 줄이기 설정을 마감합니다. 프로젝트 상세와 인터뷰는 해석되지 않은 참조를 명시적으로 처리하지만 프로젝트 아카이브의 빈 상태와 여정 아카이브의 직접 projectId URL은 별도 검증이 없어 보장하지 않는 범위로 남습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:relations:END -->

## 7. 최종 구성과 실행 순서

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:flow:BEGIN -->
등록부 진입점 → Cinematic 라우트 진입점 → 라우트 전용 view → `Frame`의 탐색·main·푸터 → `CinematicLink`/`ProjectChapter`/`Media` 공용 컴포넌트 순서입니다.

각 단계에서 선택적·빈·누락된 참조가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 커밋의 보장하지 않는 범위에 남겼습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:flow:END -->

## 8. 실행 및 검증 근거

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:runtime:BEGIN -->
- **실행한 저장소 테스트·빌드:** 없음.
- **정적 확인:** 지정 브랜치의 커밋 분류, 커밋 bodies, 해당 커밋의 변경 내용과 변경 이력의 파일 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 컨테이너에서 직접 복제본 시 DNS가 `github.com`을 해석하지 못해 변경 이력의 worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next 빌드 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** 코드·테스트 구현의 존재와 범위는 정적 검토로 확인했고, 실행 성공·실패는 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:runtime:END -->

## 9. 학습 완료 확인

<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·제목·중요도·태그를 커밋 목록과 커밋 섹션에서 동일하게 유지했습니다.
- [x] 각 SHA에 구체적인 파일과 심볼·선택자·라우트 포커스를 기록했습니다.
- [x] 이전 상태, 소유 주체, 누락·대체 처리, 보장 범위와 보장하지 않는 범위, 후속 관계를 채웠습니다.
- [x] S/A/B 깊이를 구분했습니다.
- [x] 실행하지 않은 테스트를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:04-cinematic-design-system-construction.md:checks:END -->
===== END FILE: 04-cinematic-design-system-construction.md =====

===== BEGIN FILE: 05-design-and-classic-renderer-extraction.md =====
# 개발 과정: Design·Classic 렌더러 분리

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `05-full-site-visual-systems`
>
> 1단계 검토에서 확정한 기준 문서 틀입니다. 2단계에서는 답변 식별 속성 내부만 채웁니다.

## 0. 범위와 기준

- 커밋 SHA·제목·중요도·태그는 브랜치의 `commit/commit-importance.md`와 해당 커밋의 정확한 메타데이터를 기준으로 고정했습니다.
- **개발 과정 범위:** 라우트별 분리와 최종 Design/Classic 분기 함수만 포함합니다. 새로운 시각 기능 조립이나 디자인 공통 등록부 unification은 각각 다른 분류·개발 과정에 둡니다.
- 다른 브랜치, 최종 HEAD의 후대 구현, 실행하지 않은 명령 결과를 사용하지 않습니다.

## 1. 개발 과정 목표

App Router와 공용 컴포넌트에 남아 있던 Design·Classic 화면 구성을 라우트 모듈로 옮기고, 각 디자인을 단일 분기 함수 아래 닫는 소유 주체 이전을 복원합니다.

### 고정된 불변 조건

최종 불변 조건은 App Router가 콘텐츠·메타데이터·notFound·구조화 데이터·뷰 모델 생성만 소유하고, Design·Classic 모듈이 라우트 검사·셸·화면 구성·내부 도우미 함수를 소유하며, 외부는 각 `index.tsx` 분기 함수 하나만 호출한다는 것입니다.

## 2. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `b8d35db40ed1` | refactor(routes): Design 홈 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 홈 소유 주체 이전 |
| 2 | `29943a185465` | refactor(routes): Design 프로젝트 목록 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 프로젝트 소유 주체 이전 |
| 3 | `b76fa3f1d6be` | refactor(routes): Design 프로젝트 상세 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 프로젝트 상세 소유 주체 이전 |
| 4 | `e4feecdc7a04` | refactor(routes): Design 소개 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 소개 소유 주체 이전 |
| 5 | `43fac33fdda3` | refactor(routes): Design 이력 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 이력서 소유 주체 이전 |
| 6 | `b11a4e4a3d72` | refactor(routes): Design 연락 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 연락처 소유 주체 이전 |
| 7 | `8a0bd21f7557` | refactor(routes): Design 여정 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 여정 소유 주체 이전 |
| 8 | `2fee9efda711` | refactor(routes): Design 인터뷰 renderer로 위임 | B | ROUTING, RENDERER, REFACTOR | 인터뷰 소유 주체 이전 |
| 9 | `05e1cebd0b70` | refactor(design): Design route dispatcher 추가 | A | ARCH, ROUTING, RENDERER | Design의 여덟 독립 모듈을 하나의 공개 분기 함수로 통합 |
| 10 | `15ab994dabfd` | refactor(classic-home): 홈 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic 홈 소유 주체 이전 및 공용 컴포넌트 정리 |
| 11 | `91e44a4de72c` | refactor(classic-projects): 프로젝트 목록 renderer를 이동 | B | RENDERER, REFACTOR | Classic 프로젝트 소유 주체 이전 |
| 12 | `7a65f0522061` | refactor(classic-project): 상세 renderer를 독립 모듈로 완성 | B | RENDERER, REFACTOR | Classic 프로젝트 상세 소유 주체 이전 |
| 13 | `25fa6b575c31` | refactor(classic-about): 소개 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic 소개 큰 화면 구성 이전 |
| 14 | `88fb0a09db5e` | refactor(classic-resume): 이력 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic 이력서 큰 화면 구성 이전 |
| 15 | `17e0a9ad0acb` | refactor(classic-contact): 연락 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic 연락처 이전 |
| 16 | `c44f91b0d40d` | refactor(classic-journey): 여정 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic 여정 이전 및 공용 준비된 모델 |
| 17 | `a5d8f288baa2` | refactor(classic-interview): 인터뷰 renderer를 독립 모듈로 이동 | B | RENDERER, REFACTOR | Classic 인터뷰 이전 및 공용 준비된 모델 |
| 18 | `6b193d084e69` | refactor(classic): Classic route dispatcher 추가 | A | ARCH, ROUTING, RENDERER | Classic의 여덟 독립 라우트 모듈을 하나의 공개 진입점으로 통합 |

## 3. 변경 전 기준

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:baseline:BEGIN -->
- **직전 상태:** Design과 Classic은 사이트의 초기 렌더러였기 때문에 App Router 페이지와 공용 컴포넌트가 셸·파생된 속성·마크업을 직접 소유했습니다. 전용 세 디자인과 달리 공개 단일 진입점 모듈 경계가 없었습니다.
- **경계 판단:** 라우트별 분리와 최종 Design/Classic 분기 함수만 포함합니다. 새로운 시각 기능 조립이나 디자인 공통 등록부 unification은 각각 다른 분류·개발 과정에 둡니다.
- **복원 기준:** 각 커밋의 부모 커밋과 해당 SHA 파일 트리만 사용하고 최종 HEAD를 이전 상태에 소급하지 않았습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:baseline:END -->

## 4. 커밋별 복원 기록

### 1. `b8d35db40ed1` — refactor(routes): Design 홈 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 홈 소유 주체 이전

#### 커밋별 확인 사항

- `b8d35db40ed1^`와 `b8d35db40ed1`를 비교하고 `src/app/page.tsx, src/designs/design/home-route.tsx`에서 **App 페이지의 Design 브랜치, `HomeRoute` 라우트 검사, `createDesignShellProps`, 내부 `HomeView`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `홈 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 후속 `05e1cebd0b70` 분기 함수가 이 모듈을 단일 Design 진입점 아래 묶는다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b8d35db40ed1:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 Design 브랜치, `HomeRoute` 라우트 검사, `createDesignShellProps`, 내부 `HomeView`**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** App Router가 Design 홈의 셸·섹션 조립을 직접 조립하던 책임을 렌더러 모듈로 넘긴다. 페이지는 콘텐츠·템플릿·디버그를 준비하고 라우트 속성만 전달하며 모듈이 라우트 구분값과 셸을 소유합니다.
- **정적 검토:** `src/app/page.tsx, src/designs/design/home-route.tsx`의 App 페이지의 Design 브랜치, `HomeRoute` 라우트 검사, `createDesignShellProps`, 내부 `HomeView`를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/page.tsx, src/designs/design/home-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **다음 관계:** 후속 `05e1cebd0b70` 분기 함수가 이 모듈을 단일 Design 진입점 아래 묶는다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b8d35db40ed1:END -->

### 2. `29943a185465` — refactor(routes): Design 프로젝트 목록 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 프로젝트 소유 주체 이전

#### 커밋별 확인 사항

- `29943a185465^`와 `29943a185465`를 비교하고 `src/app/projects/page.tsx, src/designs/design/projects-route.tsx`에서 **App 페이지의 파생된 pageCopy·지표·그룹 제거, 렌더러 라우트 검사와 셸·파생된 필드 사용** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 목록 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:29943a185465:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 파생된 pageCopy·지표·그룹 제거, 렌더러 라우트 검사와 셸·파생된 필드 사용**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** 프로젝트 페이지에서 셸·복사·대표·그룹·지표 조립을 제거하고 준비된 프로젝트 뷰 모델 전체를 렌더러에 넘긴다. 렌더러가 라우트 검사와 Design 셸을 소유하고 내부 view는 내부가 됩니다.
- **정적 검토:** `src/app/projects/page.tsx, src/designs/design/projects-route.tsx`의 App 페이지의 파생된 pageCopy·지표·그룹 제거, 렌더러 라우트 검사와 셸·파생된 필드 사용을 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/projects/page.tsx, src/designs/design/projects-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:29943a185465:END -->

### 3. `b76fa3f1d6be` — refactor(routes): Design 프로젝트 상세 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 프로젝트 상세 소유 주체 이전

#### 커밋별 확인 사항

- `b76fa3f1d6be^`와 `b76fa3f1d6be`를 비교하고 `src/app/projects/[projectId]/page.tsx, src/designs/design/project-detail-route.tsx`에서 **App 페이지의 StructuredData/notFound 유지, 렌더러 검사·셸·소개 영역·본문, 도우미 함수 내부화** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `프로젝트 상세 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b76fa3f1d6be:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 StructuredData/notFound 유지, 렌더러 검사·셸·소개 영역·본문, 도우미 함수 내부화**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** 프레임워크 메타데이터·notFound·구조화 데이터는 App 페이지에 남기고 시각 소개 영역·본문·셸은 Design 모듈로 이동합니다. 라우트 구분값이 `project-detail`이 아니면 null을 반환하며 내부 섹션 도우미 함수는 공개되지 않습니다.
- **정적 검토:** `src/app/projects/[projectId]/page.tsx, src/designs/design/project-detail-route.tsx`의 App 페이지의 StructuredData/notFound 유지, 렌더러 검사·셸·소개 영역·본문, 도우미 함수 내부화를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/projects/[projectId]/page.tsx, src/designs/design/project-detail-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b76fa3f1d6be:END -->

### 4. `e4feecdc7a04` — refactor(routes): Design 소개 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 소개 소유 주체 이전

#### 커밋별 확인 사항

- `e4feecdc7a04^`와 `e4feecdc7a04`를 비교하고 `src/app/about/page.tsx, src/designs/design/about-route.tsx`에서 **App 페이지의 Design 브랜치, 렌더러 셸, 경력·선별 기록 완료, 내부 선별 기록 도우미 함수** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `소개 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:e4feecdc7a04:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 Design 브랜치, 렌더러 셸, 경력·선별 기록 완료, 내부 선별 기록 도우미 함수**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** App 페이지는 콘텐츠 로딩·페이지 활성화 여부·템플릿 분기만 남기고 소개 식별 정보·기술 목록·경력·기능 플래그로 제어되는 선별 기록 조립을 모듈로 넘긴다. Curation 도우미 함수도 모듈 내부가 됩니다.
- **정적 검토:** `src/app/about/page.tsx, src/designs/design/about-route.tsx`의 App 페이지의 Design 브랜치, 렌더러 셸, 경력·선별 기록 완료, 내부 선별 기록 도우미 함수를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/about/page.tsx, src/designs/design/about-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:e4feecdc7a04:END -->

### 5. `43fac33fdda3` — refactor(routes): Design 이력 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 이력서 소유 주체 이전

#### 커밋별 확인 사항

- `43fac33fdda3^`와 `43fac33fdda3`를 비교하고 `src/app/resume/page.tsx, src/designs/design/resume-route.tsx`에서 **App 브랜치 제거, 렌더러 셸, 선택적 경력·학력·안내** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `이력서 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:43fac33fdda3:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 브랜치 제거, 렌더러 셸, 선택적 경력·학력·안내**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** 준비된 이력서 뷰 모델을 Design 렌더러가 받아 셸과 전체 섹션 순서를 소유합니다. 선택적 배열은 길이가 있을 때만 섹션을 추가하고 App 페이지는 화면 구성 마크업에서 벗어난다.
- **정적 검토:** `src/app/resume/page.tsx, src/designs/design/resume-route.tsx`의 App 브랜치 제거, 렌더러 셸, 선택적 경력·학력·안내를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/resume/page.tsx, src/designs/design/resume-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:43fac33fdda3:END -->

### 6. `b11a4e4a3d72` — refactor(routes): Design 연락 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 연락처 소유 주체 이전

#### 커밋별 확인 사항

- `b11a4e4a3d72^`와 `b11a4e4a3d72`를 비교하고 `src/app/contact/page.tsx, src/designs/design/contact-route.tsx`에서 **페이지 사용 가능 여부·맥락 해석 이후 Design 라우트 진입점 호출** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `연락처 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:b11a4e4a3d72:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **페이지 사용 가능 여부·맥락 해석 이후 Design 라우트 진입점 호출**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** 연락처 App 페이지는 활성화 여부·콘텐츠·템플릿·디버그를 준비한 뒤 렌더러를 호출합니다. 사용 가능 여부·우선 링크·안내와 셸 조립은 Design 모듈 책임이 됩니다.
- **정적 검토:** `src/app/contact/page.tsx, src/designs/design/contact-route.tsx`의 페이지 사용 가능 여부·맥락 해석 이후 Design 라우트 진입점 호출을 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/contact/page.tsx, src/designs/design/contact-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:b11a4e4a3d72:END -->

### 7. `8a0bd21f7557` — refactor(routes): Design 여정 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 여정 소유 주체 이전

#### 커밋별 확인 사항

- `8a0bd21f7557^`와 `8a0bd21f7557`를 비교하고 `src/app/journey/page.tsx, src/designs/design/journey-route.tsx, src/lib/portfolio/view-models.ts`에서 **`createJourneyViewModel`, 렌더러 라우트 검사·셸·타임라인·현재, 내부 `MilestoneCard`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `여정 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:8a0bd21f7557:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createJourneyViewModel`, 렌더러 라우트 검사·셸·타임라인·현재, 내부 `MilestoneCard`**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** App 페이지가 원본 콘텐츠가 아니라 여정 전용 뷰 모델을 명시적으로 만들고 렌더러에 전달합니다. 주요 시점 프로젝트 해석은 뷰 모델에, 시각 순서와 셸은 모듈에 위치합니다.
- **정적 검토:** `src/app/journey/page.tsx, src/designs/design/journey-route.tsx, src/lib/portfolio/view-models.ts`의 `createJourneyViewModel`, 렌더러 라우트 검사·셸·타임라인·현재, 내부 `MilestoneCard`를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/journey/page.tsx, src/designs/design/journey-route.tsx, src/lib/portfolio/view-models.ts` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:8a0bd21f7557:END -->

### 8. `2fee9efda711` — refactor(routes): Design 인터뷰 renderer로 위임

- **중요도:** B
- **태그:** ROUTING, RENDERER, REFACTOR
- **개발 과정에서의 역할:** 인터뷰 소유 주체 이전

#### 커밋별 확인 사항

- `2fee9efda711^`와 `2fee9efda711`를 비교하고 `src/app/interview-map/page.tsx, src/designs/design/interview-map-route.tsx, src/lib/portfolio/view-models.ts`에서 **`createInterviewMapViewModel`, 렌더러 라우트 검사·셸·부족한 부분, 내부 `TrackSection`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `인터뷰 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:2fee9efda711:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`createInterviewMapViewModel`, 렌더러 라우트 검사·셸·부족한 부분, 내부 `TrackSection`**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** App 페이지가 인터뷰 특정 데이터 구성을 만들고 Design 모듈이 도입부·트랙 표·부족한 부분·셸을 소유합니다. 답변과 프로젝트 해석은 뷰 모델에 남고 TrackSection은 모듈 내부다.
- **정적 검토:** `src/app/interview-map/page.tsx, src/designs/design/interview-map-route.tsx, src/lib/portfolio/view-models.ts`의 `createInterviewMapViewModel`, 렌더러 라우트 검사·셸·부족한 부분, 내부 `TrackSection`를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/interview-map/page.tsx, src/designs/design/interview-map-route.tsx, src/lib/portfolio/view-models.ts` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:2fee9efda711:END -->

### 9. `05e1cebd0b70` — refactor(design): Design route dispatcher 추가

- **중요도:** A
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** Design의 여덟 독립 모듈을 하나의 공개 분기 함수로 통합

#### 커밋별 확인 사항

- `05e1cebd0b70^`와 `05e1cebd0b70`를 비교하고 `src/designs/design/index.tsx`에서 **`DesignRoute` 전환과 홈·프로젝트·프로젝트 상세·소개·이력서·연락처·여정·인터뷰 가져오기** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Design의 8개 독립 모듈을 하나의 공개 분기 함수로 통합` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `각 모듈 내부의 빈 상태와 시각적 정확성은 분기 함수가 검증하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 1의 `380b2a025070`이 이 분기 함수를 등록부의 Design 로더로 호출합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:05e1cebd0b70:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`DesignRoute` 전환과 홈·프로젝트·프로젝트 상세·소개·이력서·연락처·여정·인터뷰 가져오기**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **구현 결정:** 라우트별 모듈 분리가 끝난 뒤 `DesignRoute` 하나가 준비된 종류가 구분된 속성을 정확한 모듈로 전달합니다. 등록부나 App Router는 개별 Design 파일 경로를 알 필요가 없고, 라우트 추가 시 분기 함수 유니언과 모듈 집합이 명시적으로 함께 바뀌어야 합니다.
- **파일·심볼:** `src/designs/design/index.tsx`에서 `DesignRoute` 전환과 홈·프로젝트·프로젝트 상세·소개·이력서·연락처·여정·인터뷰 가져오기를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/designs/design/index.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `Design의 8개 독립 모듈을 하나의 공개 분기 함수로 통합`. 각 모듈 내부 빈 상태·시각적 정확성은 분기 함수가 검증하지 않습니다.
- **역사적 연결:** 개발 과정 1의 `380b2a025070`이 이 분기 함수를 등록부의 Design 로더로 호출합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

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

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 홈 소유 주체 이전 및 공용 컴포넌트 정리

#### 커밋별 확인 사항

- `15ab994dabfd^`와 `15ab994dabfd`를 비교하고 `src/app/page.tsx, src/designs/classic/home-route.tsx, 삭제된 src/components/portfolio/home-*.tsx`에서 **App 페이지의 Classic 브랜치, 라우트 검사·셸, 설정된 섹션 순서, Classic-only 공용 컴포넌트 삭제·흡수** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 홈 화면 조립 코드 이전과 공용 컴포넌트 정리` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:15ab994dabfd:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 Classic 브랜치, 라우트 검사·셸, 설정된 섹션 순서, Classic-only 공용 컴포넌트 삭제·흡수**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** Classic 전용 홈 섹션들이 공용 컴포넌트 폴더에 흩어진 상태를 모듈 안으로 흡수하고 원래 파일을 삭제합니다. App 페이지는 준비된 속성을 넘기고 Classic 모듈이 셸·섹션 순서·시각 도우미 함수를 소유합니다.
- **정적 검토:** `src/app/page.tsx, src/designs/classic/home-route.tsx, 삭제된 src/components/portfolio/home-*.tsx`의 App 페이지의 Classic 브랜치, 라우트 검사·셸, 설정된 섹션 순서, Classic-only 공용 컴포넌트 삭제·흡수를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/page.tsx, src/designs/classic/home-route.tsx, 삭제된 src/components/portfolio/home-*.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:15ab994dabfd:END -->

### 11. `91e44a4de72c` — refactor(classic-projects): 프로젝트 목록 renderer를 이동

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 프로젝트 소유 주체 이전

#### 커밋별 확인 사항

- `91e44a4de72c^`와 `91e44a4de72c`를 비교하고 `src/app/projects/page.tsx, src/designs/classic/projects-route.tsx`에서 **App 페이지의 PageShell·지표·그룹 제거, 렌더러 라우트 검사·셸, 준비된 튜플** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 프로젝트 목록 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:91e44a4de72c:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 PageShell·지표·그룹 제거, 렌더러 라우트 검사·셸, 준비된 튜플**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** App 페이지에서 pageCopy·대표·그룹화된 프로젝트·지표 값을 해석하던 코드를 제거합니다. Classic 렌더러가 준비된 모델과 셸을 소비하고 내부 view는 내부가 됩니다.
- **정적 검토:** `src/app/projects/page.tsx, src/designs/classic/projects-route.tsx`의 App 페이지의 PageShell·지표·그룹 제거, 렌더러 라우트 검사·셸, 준비된 튜플를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/projects/page.tsx, src/designs/classic/projects-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:91e44a4de72c:END -->

### 12. `7a65f0522061` — refactor(classic-project): 상세 renderer를 독립 모듈로 완성

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 프로젝트 상세 소유 주체 이전

#### 커밋별 확인 사항

- `7a65f0522061^`와 `7a65f0522061`를 비교하고 `src/app/projects/[projectId]/page.tsx, src/designs/classic/project-detail-route.tsx`에서 **StructuredData는 페이지에 유지, 렌더러 검사·셸·소개 영역·본문과 내부 섹션 도우미 함수** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 프로젝트 상세 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:7a65f0522061:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **StructuredData는 페이지에 유지, 렌더러 검사·셸·소개 영역·본문과 내부 섹션 도우미 함수**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** App 페이지는 구조화 데이터와 프레임워크 경계를 유지하고 Classic 모듈이 라우트 구분값, 셸, 프로젝트 소개 영역·본문을 소유합니다. 시각 도우미 함수는 외부 API에서 제거됩니다.
- **정적 검토:** `src/app/projects/[projectId]/page.tsx, src/designs/classic/project-detail-route.tsx`의 StructuredData는 페이지에 유지, 렌더러 검사·셸·소개 영역·본문과 내부 섹션 도우미 함수를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/projects/[projectId]/page.tsx, src/designs/classic/project-detail-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:7a65f0522061:END -->

### 13. `25fa6b575c31` — refactor(classic-about): 소개 renderer를 독립 모듈로 이동

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 소개 큰 화면 구성 이전

#### 커밋별 확인 사항

- `25fa6b575c31^`와 `25fa6b575c31`를 비교하고 `src/app/about/page.tsx, src/designs/classic/about-route.tsx`에서 **App 페이지에서 제거된 300여 줄 식별 정보·원칙·여정·기술 목록·경력·선별 기록 마크업, 라우트 모듈 셸** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 소개 화면의 대규모 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:25fa6b575c31:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지에서 제거된 300여 줄 식별 정보·원칙·여정·기술 목록·경력·선별 기록 마크업, 라우트 모듈 셸**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** 소개 페이지에 직접 있던 Classic 전체 화면 구성과 선별 기록 도우미 함수를 전용 모듈로 옮긴다. App 페이지는 Classic/Design 구현을 선택하고 동일 준비된 속성을 전달하는 얇은 변환 함수가 됩니다.
- **정적 검토:** `src/app/about/page.tsx, src/designs/classic/about-route.tsx`의 App 페이지에서 제거된 300여 줄 식별 정보·원칙·여정·기술 목록·경력·선별 기록 마크업, 라우트 모듈 셸을 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/about/page.tsx, src/designs/classic/about-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:25fa6b575c31:END -->

### 14. `88fb0a09db5e` — refactor(classic-resume): 이력 renderer를 독립 모듈로 이동

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 이력서 큰 화면 구성 이전

#### 커밋별 확인 사항

- `88fb0a09db5e^`와 `88fb0a09db5e`를 비교하고 `src/app/resume/page.tsx, src/designs/classic/resume-route.tsx`에서 **App 페이지에서 제거된 소개 영역·요약·프로젝트·교육·경력·학력·안내, 모듈 셸** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 이력서 화면의 대규모 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:88fb0a09db5e:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지에서 제거된 소개 영역·요약·프로젝트·교육·경력·학력·안내, 모듈 셸**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** 이력서의 200여 줄 화면 구성을 Classic 모듈로 이동해 준비된 이력서 모델과 셸 조립을 한 소유 주체에 둡니다. App 페이지는 활성화 여부·맥락·템플릿 선택만 남깁니다.
- **정적 검토:** `src/app/resume/page.tsx, src/designs/classic/resume-route.tsx`의 App 페이지에서 제거된 소개 영역·요약·프로젝트·교육·경력·학력·안내, 모듈 셸을 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/resume/page.tsx, src/designs/classic/resume-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:88fb0a09db5e:END -->

### 15. `17e0a9ad0acb` — refactor(classic-contact): 연락 renderer를 독립 모듈로 이동

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 연락처 이전

#### 커밋별 확인 사항

- `17e0a9ad0acb^`와 `17e0a9ad0acb`를 비교하고 `src/app/contact/page.tsx, src/designs/classic/contact-route.tsx`에서 **App 페이지의 소개 영역·사용 가능 여부·우선 링크·빈·안내 제거와 모듈 라우트 검사** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 연락처 화면 조립 코드 이전` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:17e0a9ad0acb:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 소개 영역·사용 가능 여부·우선 링크·빈·안내 제거와 모듈 라우트 검사**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** Classic 연락처 마크업과 빈 연락처 링크 브랜치를 모듈로 옮긴다. 페이지는 Design/Classic 라우트 컴포넌트를 선택하고 준비된 속성만 전달합니다.
- **정적 검토:** `src/app/contact/page.tsx, src/designs/classic/contact-route.tsx`의 App 페이지의 소개 영역·사용 가능 여부·우선 링크·빈·안내 제거와 모듈 라우트 검사를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/contact/page.tsx, src/designs/classic/contact-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:17e0a9ad0acb:END -->

### 16. `c44f91b0d40d` — refactor(classic-journey): 여정 renderer를 독립 모듈로 이동

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 여정 이전 및 공용 준비된 모델

#### 커밋별 확인 사항

- `c44f91b0d40d^`와 `c44f91b0d40d`를 비교하고 `src/app/journey/page.tsx, src/designs/classic/journey-route.tsx`에서 **App 페이지의 원본 주요 시점 해석 제거, `createJourneyViewModel`, 모듈 `MilestoneCard`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 여정 화면 조립 코드 이전과 공용 준비 모델 적용` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:c44f91b0d40d:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 원본 주요 시점 해석 제거, `createJourneyViewModel`, 모듈 `MilestoneCard`**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** Classic/Design 모두 같은 준비된 여정 모델을 받도록 App 페이지에서 데이터 구성을 한 번 만듭니다. Classic 모듈은 셸·주요 시점·타임라인·현재 화면 구성을 소유하고 원본 프로젝트 조회를 하지 않습니다.
- **정적 검토:** `src/app/journey/page.tsx, src/designs/classic/journey-route.tsx`의 App 페이지의 원본 주요 시점 해석 제거, `createJourneyViewModel`, 모듈 `MilestoneCard`를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/journey/page.tsx, src/designs/classic/journey-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:c44f91b0d40d:END -->

### 17. `a5d8f288baa2` — refactor(classic-interview): 인터뷰 renderer를 독립 모듈로 이동

- **중요도:** B
- **태그:** RENDERER, REFACTOR
- **개발 과정에서의 역할:** Classic 인터뷰 이전 및 공용 준비된 모델

#### 커밋별 확인 사항

- `a5d8f288baa2^`와 `a5d8f288baa2`를 비교하고 `src/app/interview-map/page.tsx, src/designs/classic/interview-map-route.tsx`에서 **App 페이지의 프로젝트 맵·표 마크업 제거, `createInterviewMapViewModel`, 모듈 `TrackSection`** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic 인터뷰 화면 조립 코드 이전과 공용 준비 모델 적용` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `이 커밋은 해당 라우트·섹션의 정적 구성을 추가하지만, 다른 라우트와 모든 빈 상태, 브라우저 레이아웃까지 자동으로 검증하지는 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:a5d8f288baa2:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **App 페이지의 프로젝트 맵·표 마크업 제거, `createInterviewMapViewModel`, 모듈 `TrackSection`**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **변경:** Classic/Design이 동일 인터뷰 데이터 구성을 소비하도록 하고 App 페이지에서 200여 줄 표 화면 구성과 프로젝트 조회를 제거합니다. 모듈은 셸과 의미상 표·부족한 부분을 소유합니다.
- **정적 검토:** `src/app/interview-map/page.tsx, src/designs/classic/interview-map-route.tsx`의 App 페이지의 프로젝트 맵·표 마크업 제거, `createInterviewMapViewModel`, 모듈 `TrackSection`를 확인했습니다.
- **담당 코드와 입력 범위:** 표현·조립 책임은 `src/app/interview-map/page.tsx, src/designs/classic/interview-map-route.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **비보장:** 이 커밋은 해당 라우트·섹션의 정적 화면 구성을 추가하지만 다른 라우트, 모든 빈 상태, 브라우저 레이아웃을 자동으로 검증하지 않습니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.
<!-- LEARNER-ANSWER:commit:a5d8f288baa2:END -->

### 18. `6b193d084e69` — refactor(classic): Classic route dispatcher 추가

- **중요도:** A
- **태그:** ARCH, ROUTING, RENDERER
- **개발 과정에서의 역할:** Classic의 여덟 독립 라우트 모듈을 하나의 공개 진입점으로 통합

#### 커밋별 확인 사항

- `6b193d084e69^`와 `6b193d084e69`를 비교하고 `src/designs/classic/index.tsx`에서 **`ClassicRoute` 누락 없는 전환과 8 모듈 가져오기** 관련 항목이 처음 추가되거나 이동한 위치를 확인합니다.
- 직전 소유 주체와 이 SHA 이후 소유 주체를 구분하고, `Classic의 8개 독립 라우트 모듈을 하나의 공개 진입점으로 통합` 역할이 호출자와 호출 대상·DOM·콘텐츠 참조 지점에 어떤 변화를 주는지 적습니다.
- 누락된 값, 지원하지 않는 라우트, 찾을 수 없는 참조, 빈 목록, 선택적 미디어·링크, 반응형 대체 처리 중 이 SHA에 실제 존재하는 분기만 기록합니다.
- `분기 함수는 모듈 반환값의 내용이나 실행 시점의 알 수 없는 문자열을 별도로 검증하지 않습니다.`라는 한계를 코드와 테스트 범위에서 확인합니다.
- 후속 관계: 개발 과정 1의 `380b2a025070`이 Design과 함께 등록부 경로로 통일합니다.
- 같은 규칙을 소비하는 다른 라우트·디자인과 비교하되, 이 SHA 이후 코드를 현재 커밋의 구현으로 소급하지 않습니다.

#### 학습 기록

<!-- LEARNER-ANSWER:commit:6b193d084e69:BEGIN -->
- **직전 상태:** 동작 자체는 앞선 구현에 존재했지만 **`ClassicRoute` 누락 없는 전환과 8 모듈 가져오기**의 조립·조회·공개 책임이 App Router와 원본 콘텐츠 소비자 또는 여러 모듈에 분산돼 있었습니다.
- **구현 결정:** 라우트별 분리 후 `ClassicRoute`가 준비된 종류가 구분된 속성을 각 모듈로 전달합니다. 외부 등록부는 Classic 내부 파일 구조를 알지 않고 single 진입점만 지연 로딩할 수 있습니다.
- **파일·심볼:** `src/designs/classic/index.tsx`에서 `ClassicRoute` 누락 없는 전환과 8 모듈 가져오기를 확인했습니다.
- **소유권:** 표현·조립 책임은 `src/designs/classic/index.tsx` 쪽으로 이동하고, 이전 호출자는 프레임워크·맥락 준비 또는 공개 진입점 호출만 남습니다.
- **보장·비보장:** 이 SHA는 다음 내용을 기준 동작으로 확정합니다: `Classic의 8개 독립 라우트 모듈을 하나의 공개 진입점으로 통합`. 분기 함수는 모듈 반환값의 내용이나 실행 중 알 수 없는 문자열을 별도 검증하지 않습니다.
- **역사적 연결:** 개발 과정 1의 `380b2a025070`이 Design과 함께 등록부 경로로 통일합니다.
- **실행 증거:** 이 SHA의 저장소 테스트·실행 명령은 실행하지 않았습니다. 기록은 브랜치에서 해당 커밋의 변경 내용과 변경 후 파일 트리를 정적으로 검토한 결과입니다.

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

## 5. 불변 조건 변화

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:invariant:BEGIN -->
최종 불변 조건은 App Router가 콘텐츠·메타데이터·notFound·구조화 데이터·뷰 모델 생성만 소유하고, Design·Classic 모듈이 라우트 검사·셸·화면 구성·내부 도우미 함수를 소유하며, 외부는 각 `index.tsx` 분기 함수 하나만 호출한다는 것입니다.

- 도입·확장·폐쇄의 순서는 커밋 목록에 고정했습니다.
- 중요도 B 구축은 라우트·스타일 표시 영역을 단계적으로 넓히고, A·중요도 S 커밋은 소유 주체·분기·검증 불변 조건을 바꿉니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:invariant:END -->

## 6. 실패 → 수정 → 테스트와 소유 주체 변화

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:relations:BEGIN -->
Design은 라우트별 전용 모듈을 연결한 뒤 `05e1cebd0b70`에서 분기 함수를 만들고, Classic은 페이지·공용 컴포넌트의 대규모 화면 구성을 이동한 뒤 `6b193d084e69`에서 동일한 공개 진입점을 만듭니다. 이후 개발 과정 1의 `380b2a025070`이 둘을 등록부 경로로 통일합니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:relations:END -->

## 7. 최종 구성과 실행 순서

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:flow:BEGIN -->
App Router 페이지 → 라우트 전용 뷰 모델·맥락 → Design 또는 Classic 분기 함수 → 전용 라우트 모듈 → `createDesignShellProps` → 디자인이 소유한 본문 순서입니다.

각 단계에서 선택적·빈·누락된 참조가 처리되지 않는 경우도 보장으로 포장하지 않았으며, 해당 커밋의 보장하지 않는 범위에 남겼습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:flow:END -->

## 8. 실행 및 검증 근거

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:runtime:BEGIN -->
- **실행한 저장소 테스트·빌드:** 없음.
- **정적 확인:** 지정 브랜치의 커밋 분류, 커밋 bodies, 해당 커밋의 변경 내용과 변경 이력의 파일 변경을 GitHub 연결을 통해 확인했습니다.
- **실행하지 못한 이유:** 작업 컨테이너에서 직접 복제본 시 DNS가 `github.com`을 해석하지 못해 변경 이력의 worktree를 만들 수 없었습니다. 따라서 Vitest, Playwright, Next 빌드 결과를 성공으로 기록하지 않았습니다.
- **검증 수준:** 코드·테스트 구현의 존재와 범위는 정적 검토로 확인했고, 실행 성공·실패는 주장하지 않습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:runtime:END -->

## 9. 학습 완료 확인

<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:checks:BEGIN -->
- [x] 모든 고정 SHA·제목·중요도·태그를 커밋 목록과 커밋 섹션에서 동일하게 유지했습니다.
- [x] 각 SHA에 구체적인 파일과 심볼·선택자·라우트 포커스를 기록했습니다.
- [x] 이전 상태, 소유 주체, 누락·대체 처리, 보장 범위와 보장하지 않는 범위, 후속 관계를 채웠습니다.
- [x] S/A/B 깊이를 구분했습니다.
- [x] 실행하지 않은 테스트를 통과로 표시하지 않았습니다.
<!-- LEARNER-ANSWER:thread:05-design-and-classic-renderer-extraction.md:checks:END -->
===== END FILE: 05-design-and-classic-renderer-extraction.md =====

===== BEGIN FILE: README.md =====
# 사이트 전체 시각 체계

> 저장소: `seungwoo7050/42-archive`
>
> 브랜치: `web/portfolio`
>
> 분류: `05-full-site-visual-systems`
>
> 검토한 브랜치 범위: `cce7dd020563` → `aff0acdd4cf9`

## 분류 범위

이 분류는 사이트 전체를 아우르는 다섯 가지 시각 체계를 독립 렌더러로 구축하고, 이를 공통 라우트·셸·입력·테스트 규칙에 연결한 이력을 다룹니다.

- 포함: 디자인 공통 등록부·위임·준비된 렌더러 입력·토큰·셸 조립·활성화·회귀 검증, Editorial·Brutalist·Cinematic 전체 구축, Design·Classic 라우트 모듈 분리.
- 제외: 콘텐츠 스키마 자체의 최초 설계, 일반 App Router 실행 주기, 공용 상호작용 공용 컴포넌트의 독립 이력, 배포 상태·CI, 서식만 변경한 C 커밋.
- 예외적으로 `8a48460df4c3`와 `aef265b9bd01`은 콘텐츠 뷰 모델 구축이 아니라 **모든 렌더러에 적용되는 소유 주체 이전**이므로 개발 과정 1에 포함했습니다.

## 1단계 검토 결과

- 개발 과정 수는 5개로 유지했습니다. 각 시각 시스템의 독립 개발 과정이 분명해 분리·통합이 필요하지 않았습니다.
- 기존 문서 틀의 일반적인 확인 안내를 정확한 파일·함수·선택자 그룹·실패·빈·참조·테스트 확인 사항으로 교체했습니다.
- Editorial과 Brutalist의 누락된 데스크톱→태블릿→모바일 구축 및 라우트 조립 커밋을 복구했습니다.
- Cinematic의 `1f4c35853502`, `52f13fcc5a12`처럼 상세·이력서·여정·인터뷰 지면을 실제로 연결하는 중간 커밋을 복구했습니다.
- `dc2cf72a768d`, `8a48460df4c3`, `aef265b9bd01`, `f8b0ab7b08aa`, `380b2a025070`처럼 구성 완성에 필요한 S/A 커밋을 디자인 공통 개발 과정에 추가했습니다.
- 활성화 커밋 `c6acfe562694`, `dd71d28143a8`, `b8de57f130eb`은 렌더러 내부 구축 개발 과정에서 제거하고 공통 등록부 소유 주체를 다루는 개발 과정 1로 이동했습니다.
- 서식만 변경한 C 커밋 `ea073db5f785`, `4e54f8fef892`, `6fe74d2dd94d`, `87ac2ce7b285`은 실질적인 이력을 바꾸지 않아 포함하지 않았습니다.
- 고정 커밋 수: **158개 고유 SHA** (1: 16, 2: 55, 3: 50, 4: 19, 5: 18).

## 고정된 개발 과정 순서

1. [디자인 공통 토큰·셸·회귀 규칙](./01-디자인 공통-token-shell-and-regression-contracts.md)
2. [Editorial 디자인 체계 구축](./02-editorial-design-system-construction.md)
3. [Brutalist 디자인 체계 구축](./03-brutalist-design-system-construction.md)
4. [Cinematic 디자인 체계 구축](./04-cinematic-design-system-construction.md)
5. [Design·Classic 렌더러 분리](./05-design-and-classic-renderer-extraction.md)

## 학습 원칙

- 각 커밋은 해당 SHA의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 읽습니다.
- 최종 HEAD의 도우미 함수·파일 레이아웃·테스트를 이전 SHA에 소급하지 않습니다.
- 정적 검토와 실제 명령 실행을 분리합니다.
- 라우트별 누락되거나 빈·지원하지 않는·참조 실패와 **보장하지 않는 것**을 반드시 남깁니다.
- 중요도 S·A·B의 설명 깊이를 동일하게 만들지 않습니다.

## 2단계 완료 및 검증

<!-- LEARNER-ANSWER:readme:completion:BEGIN -->
- 문서 틀과 완료본은 각각 `README.md`를 포함한 6개 파일이며 상대 경로가 정확히 일치합니다.
- 검토 후 고정한 문서 틀 파일 트리 SHA-256: `857ec3ae46a37ca14339974872c258bfefb9182deddc8cf23f340b85e730aacf`.
- 2단계 후 문서 틀 명세 파일을 다시 계산해 고정한 해시가 변하지 않았음을 확인했습니다.
- 158개 SHA는 중복이 없고, 브랜치의 `commit/commit-importance.md` 선형 이력에 존재하며 선택 커밋의 메타데이터와 대조했습니다.
- 완료본의 답변 식별 속성 바깥 문구는 문서 틀과 바이트 단위로 같은 기본 골격인지 검증했습니다.
- 모든 학습자용 식별 속성은 비어 있지 않으며 중요도 S·A·B에 따라 설명 깊이가 다릅니다.
- 저장소 테스트·빌드·Playwright는 실행하지 않았습니다. 컨테이너 DNS 제한으로 체크아웃·worktree를 만들 수 없었고, 통과 결과를 작성하지 않았습니다.
- 로컬 파일 형태 검증: `PASS`; 패키지 파일 수 `12`; 추가 파일 `0`.

<!-- LEARNER-ANSWER:readme:completion:END -->
===== END FILE: README.md =====

