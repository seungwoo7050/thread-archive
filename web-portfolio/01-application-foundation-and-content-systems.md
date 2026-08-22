===== BEGIN FILE: 01-runnable-next-application-boundary.md =====
# Thread: Runnable Next application boundary

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

문서뿐인 저장소가 고정된 Next.js 애플리케이션, 전역 스타일 입력점, content aggregate를 소비하는 첫 route까지 갖추는 경계를 복원합니다.

### 계획된 핵심 invariant

- 실행 경계는 package script, TypeScript/Next/PostCSS 설정, App Router root로 명시됩니다.
- 전역 스타일은 하나의 `globals.css`와 root layout import를 통해 적용됩니다.
- 첫 route는 JSON을 직접 조립하지 않고 portfolio aggregate를 호출합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 문서용 저장소와 실행 가능한 애플리케이션의 경계는 어느 commit에서 생기는가?
- `globals.css`가 추가된 시점과 실제 import된 시점을 구분하면 무엇이 보이는가?
- 초기 route가 content와 renderer 사이에서 맡은 책임과 아직 맡지 않은 책임은 무엇인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `cce7dd020563` | docs(portfolio): 프로젝트 목적과 초기 규약 정의 | C | CONTENT | 문서 기반 초기 상태 |
| 2 | `448bc2510f34` | build(next): 실행 가능한 애플리케이션 골격 구성 | A | DEPLOY | 실행 가능한 애플리케이션 기반 |
| 3 | `0a28cb050bc8` | style(theme): 포트폴리오 기본 디자인 토큰 추가 | B | RENDERER | 전역 스타일 vocabulary 도입 |
| 4 | `03c4e1f7b439` | feat(app): 콘텐츠 기반 디자인 홈 연결 | B | CONTENT | 첫 content-to-renderer 통합 |

## 5. Commit별 학습 기록

### 1. `cce7dd020563` — docs(portfolio): 프로젝트 목적과 초기 규약 정의

- **Importance:** C
- **Tags:** CONTENT
- **Thread 역할:** 문서 기반 초기 상태
- **조사 깊이:** Thread의 출발점을 이해하는 데 필요한 context와 후속 제약만 기록합니다.

#### 해당 SHA에서 확인할 실제 코드

- `README.md`의 목적, 콘텐츠 편집 위치, 코드/콘텐츠 분리 규칙을 확인합니다.
- 이 tree에 `package.json`, `src/app`, 실행 script가 없는지 확인합니다.

확인 원칙:

- 먼저 `cce7dd020563^`와 `cce7dd020563`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 저장소에는 실행 코드가 없고 포트폴리오의 목적과 향후 디렉터리 규칙만 문서화되어 있었습니다. |
| 실제 변경 file/symbol/call path | `README.md`가 `src/content`, `src/lib/portfolio`, `src/components/portfolio`를 각각 source, 조립, 표현 위치로 예고합니다. |
| Data/state/resource owner와 lifetime | 소유권은 아직 문서 규칙에만 있으며 runtime owner는 존재하지 않습니다. |
| Failure·absence·fallback 처리 | 문서가 맞아도 build·route·렌더링을 검증할 방법은 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 후속 구현이 따라야 할 편집 경계는 제시하지만 실행 가능성은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `448bc2510f34`가 이 문서 규칙 위에 실제 Next 애플리케이션 경계를 만듭니다. |

#### 코드·실행 증거

정적 근거: `cce7dd020563`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `448bc2510f34` — build(next): 실행 가능한 애플리케이션 골격 구성

- **Importance:** A
- **Tags:** DEPLOY
- **Thread 역할:** 실행 가능한 애플리케이션 기반
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `dev`, `build`, `start`, `lint`, `typecheck` script와 Next/React version을 확인합니다.
- `tsconfig.json`, `next.config.ts`, `postcss.config.mjs`, `eslint.config.mjs`의 compiler/build 경계를 확인합니다.
- `src/app/layout.tsx`와 `src/app/page.tsx`가 제공하는 최소 App Router tree를 확인합니다.

확인 원칙:

- 먼저 `448bc2510f34^`와 `448bc2510f34`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 직전 tree에는 dependency graph, compiler 설정, App Router entry가 없어 어떤 콘텐츠도 웹 애플리케이션으로 실행할 수 없었습니다. |
| 실제 변경 file/symbol/call path | Next 16.2.4·React 19.2.4 기반 package와 strict TypeScript, Tailwind PostCSS plugin, ESLint, root layout/page를 한 번에 추가합니다. 개발 서버와 production server는 포트 3100을 사용합니다. |
| Data/state/resource owner와 lifetime | `package.json`이 lifecycle command를, `src/app/layout.tsx`가 document shell을, `src/app/page.tsx`가 첫 route output을 소유합니다. |
| Failure·absence·fallback 처리 | 설정 파일이 생겨도 당시 page는 정적 placeholder이며 content schema·loader·실제 renderer 통합은 없습니다. Node/npm 재현성 pinning도 이 Thread가 아니라 category 08의 후속 책임입니다. |
| 보장하는 것과 보장하지 않는 것 | `npm run build`가 가능한 구조와 App Router root는 생기지만 실제 실행 성공은 이번 작업에서 재현하지 않았습니다. |
| 다음 commit 또는 관련 test 연결 | `0a28cb050bc8`이 styling input을 만들고 `03c4e1f7b439`가 content/render integration을 연결합니다. |

#### 코드·실행 증거

정적 근거: `448bc2510f34`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 정적 증거: exact SHA diff에서 package/config/App Router 파일 추가를 확인했습니다. 저장소 command는 로컬 checkout 부재로 실행하지 않았습니다.

### 3. `0a28cb050bc8` — style(theme): 포트폴리오 기본 디자인 토큰 추가

- **Importance:** B
- **Tags:** RENDERER
- **Thread 역할:** 전역 스타일 vocabulary 도입
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/app/globals.css`의 Tailwind import, `:root` token, `@theme inline`, body/anchor/selection 규칙을 확인합니다.
- 이 SHA에서 root layout이 파일을 import하는지와 아직 미연결인지 구분합니다.

확인 원칙:

- 먼저 `0a28cb050bc8^`와 `0a28cb050bc8`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 초기 App Router에는 전역 token과 base style을 담을 프로젝트 파일이 없었습니다. |
| 실제 변경 file/symbol/call path | `globals.css`가 색상·font token을 CSS custom property로 정의하고 Tailwind theme alias와 document base style을 제공합니다. |
| Data/state/resource owner와 lifetime | 스타일 값의 owner는 component별 class가 아니라 root stylesheet로 이동하지만, 이 SHA만으로는 layout import가 없어 소비가 시작되지 않습니다. |
| Failure·absence·fallback 처리 | 파일이 존재해도 import되지 않으면 runtime CSS bundle에 포함된다는 보장이 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 공용 token vocabulary를 보장하지만 실제 적용은 다음 commit까지 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `03c4e1f7b439`의 root layout import가 이 파일을 실제 application boundary에 연결합니다. |

#### 코드·실행 증거

정적 근거: `0a28cb050bc8`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 4. `03c4e1f7b439` — feat(app): 콘텐츠 기반 디자인 홈 연결

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 첫 content-to-renderer 통합
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`의 Geist font, `site.json`, `globals.css` import와 metadata/lang 설정을 확인합니다.
- `src/app/page.tsx`의 `getPortfolioContent()` → `DesignHomeRoute` 호출과 전달 props를 확인합니다.
- `contentDebug={false}`와 단일 Design renderer라는 초기 제한을 기록합니다.

확인 원칙:

- 먼저 `03c4e1f7b439^`와 `03c4e1f7b439`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | App Router skeleton과 stylesheet는 있었지만 site metadata, portfolio aggregate, home renderer가 연결되지 않았습니다. |
| 실제 변경 file/symbol/call path | root layout이 site source로 metadata와 language를 정하고 globals/font를 적용하며, page는 `getPortfolioContent()` 결과를 `DesignHomeRoute`에 전달합니다. |
| Data/state/resource owner와 lifetime | route는 aggregate 호출과 renderer 선택을 소유하고, content 조립은 `src/lib/portfolio`, 표현은 Design route component가 소유합니다. |
| Failure·absence·fallback 처리 | 고정 `contentDebug={false}`이고 다른 design 선택·runtime validation·route error policy는 아직 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 첫 page가 직접 JSON을 조립하지 않는 application/content boundary를 보장하지만 source의 runtime 신뢰성은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | T2가 aggregate 모델을, T3 이후가 presentation/다중 route 계약을 확장합니다. |

#### 코드·실행 증거

정적 근거: `03c4e1f7b439`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| 실행 command와 App Router root가 존재한다. | `448bc2510f34` | `package.json`, `src/app/layout.tsx`, `src/app/page.tsx` | Node/npm pin과 production smoke는 category 08에서 보강됩니다. |
| 전역 token은 root stylesheet에서 정의되고 layout이 import한다. | `0a28cb050bc8` → `03c4e1f7b439` | `src/app/globals.css`, `src/app/layout.tsx` | token 의미의 visual regression은 이 Thread가 검증하지 않습니다. |
| 첫 route는 portfolio aggregate를 renderer에 전달한다. | `03c4e1f7b439` | `src/app/page.tsx`의 `getPortfolioContent()` 호출 | aggregate 내부 값은 아직 assertion 기반입니다. |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| 문서만 있고 실행 경계가 없음 | `448bc2510f34` | package/config/App Router root 추가 | 이번 작업에서는 repository command 미실행; 후속 category 08 검증과 연결 |
| stylesheet가 존재하지만 소비되지 않을 수 있음 | `03c4e1f7b439` | root layout에서 `./globals.css` import | 후속 browser/visual tests가 실제 회귀 면을 보호 |
| route가 source를 직접 조립할 위험 | `03c4e1f7b439` | `getPortfolioContent()`를 단일 호출점으로 사용 | `3353032ba23b` 이후 content test가 aggregate 경로를 검증 |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| 실행 lifecycle | 없음 | `package.json` scripts | `npm run dev/build/start/lint/typecheck` |
| document shell | 없음 | `src/app/layout.tsx` | metadata, language, font, globals |
| content 조립 | 없음 | `src/lib/portfolio` | `getPortfolioContent()` |
| home 표현 | 정적 placeholder | `DesignHomeRoute` | `src/app/page.tsx`가 aggregate를 전달 |

## 9. Thread 최종 상태

Thread 종료 시점에는 Next App Router가 실행 구조를 갖고 전역 스타일과 site metadata를 root에서 적용하며, 첫 page가 portfolio aggregate를 Design home renderer에 전달합니다. 다만 toolchain pin, production smoke, runtime content validation과 다중 design routing은 별도 후속 책임입니다.

### 최종 설명

- 문서 규칙만 있던 root에 Next/React/TypeScript/PostCSS/ESLint lifecycle을 추가했습니다.
- 전역 CSS token을 별도 파일에 만들고 root layout import로 실제 소비를 연결했습니다.
- page는 content 조립을 소유하지 않고 `getPortfolioContent()` 결과를 renderer에 넘기는 얇은 경계가 되었습니다.
- 실행 성공과 production server 상태는 이번 정적 조사로 증명하지 않았습니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| 애플리케이션 lifecycle을 선택합니다. | `package.json` | script와 dependency graph | command 자체의 환경 오류는 이 Thread에서 실행 검증하지 않음 |
| root document를 구성합니다. | `src/app/layout.tsx` | site metadata, language, fonts, global CSS | 잘못된 source 값은 당시 runtime parse 없이 소비 |
| portfolio aggregate를 요청합니다. | `src/app/page.tsx` → `getPortfolioContent()` | 한 개 aggregate | loader/schema failure path는 후속 Thread |
| Design home을 렌더링합니다. | `DesignHomeRoute` | content와 fixed debug flag | 다른 design/route 선택 없음 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. 후속 category 08의 toolchain/production smoke와 category 07의 integration/visual tests가 실행·회귀 증거를 추가합니다. 이 Thread에서 repository command는 실행하지 않았습니다.
===== END FILE: 01-runnable-next-application-boundary.md =====

===== BEGIN FILE: 02-portfolio-domain-and-aggregate-model.md =====
# Thread: Portfolio domain and aggregate model

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

분산 JSON source와 수동 TypeScript shape가 하나의 `PortfolioContent` aggregate 및 파생 map/filter 흐름으로 조립되는 초기 domain 모델을 복원합니다.

### 계획된 핵심 invariant

- 콘텐츠 source는 JSON에 있고 renderer-facing aggregate는 `src/lib/portfolio`가 구성합니다.
- 정렬·enabled filtering·environment href resolution은 호출자마다 재구현하지 않습니다.
- 이 단계의 TypeScript assertion은 runtime validation이 아니라 정적 편의에 불과합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 각 JSON source와 대응 TypeScript type은 어떤 순서로 확장되는가?
- `getPortfolioContent()`가 새 aggregate를 만들면서 공유하는 객체와 복사하는 배열은 무엇인가?
- disabled 항목과 environment override는 어디에서 제거·적용되는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `efb1e2e26b74` | feat(content): 사이트와 프로필 콘텐츠 기반 추가 | B | CONTENT | 기본 identity source와 type 도입 |
| 2 | `5eb01dfecabb` | feat(content): 링크와 프로젝트 도메인 정의 | B | CONTENT | project/link core vocabulary |
| 3 | `0d891d41cf4c` | feat(content): 기술과 여정 콘텐츠 모델 추가 | B | CONTENT | 경력/기술/여정 source 확장 |
| 4 | `8661cc00c45d` | feat(content): 연락과 이력 집계 모델 완성 | B | CONTENT | aggregate type 완성 |
| 5 | `a365b3d19118` | feat(content): 정적 포트폴리오 콘텐츠 로딩 | B | CONTENT | 직접 JSON import 기반 조립 |
| 6 | `0b134b1a6cf6` | feat(content): 여정 정렬과 콘텐츠 인덱스 구성 | B | CONTENT | 전체 source와 파생 index 연결 |
| 7 | `7c95a6f387b4` | feat(content): 환경 링크를 반영한 콘텐츠 집계 | B | CONTENT | 초기 aggregate policy 완성 |

## 5. Commit별 학습 기록

### 1. `efb1e2e26b74` — feat(content): 사이트와 프로필 콘텐츠 기반 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 기본 identity source와 type 도입
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/content/site.json`, `profile.json`의 초기 필드를 확인합니다.
- `src/lib/portfolio/types.ts`의 `SiteContent`, `ProfileContent`와 JSON 사이에 runtime parse가 없는지 확인합니다.

확인 원칙:

- 먼저 `efb1e2e26b74^`와 `efb1e2e26b74`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | site/profile 값이 route 또는 component에 하드코딩될 수 있는 상태였습니다. |
| 실제 변경 file/symbol/call path | site metadata/navigation/footer와 profile identity/summary/principles를 JSON source와 TypeScript shape로 분리합니다. |
| Data/state/resource owner와 lifetime | 편집 가능한 값은 JSON이, compile-time shape는 `types.ts`가 소유합니다. |
| Failure·absence·fallback 처리 | JSON이 type을 만족한다는 보장은 runtime에 없고, assertion을 쓰면 malformed source도 import될 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | 기본 identity vocabulary를 제공하지만 aggregate·validation은 아직 없습니다. |
| 다음 commit 또는 관련 test 연결 | `5eb01dfecabb`부터 project/link domain이 추가됩니다. |

#### 코드·실행 증거

정적 근거: `efb1e2e26b74`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `5eb01dfecabb` — feat(content): 링크와 프로젝트 도메인 정의

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** project/link core vocabulary
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/content/projects.json`, `links.json`의 초기 빈 collection을 확인합니다.
- `ContentLink`, deployment/project model, environment key 정의를 확인합니다.

확인 원칙:

- 먼저 `5eb01dfecabb^`와 `5eb01dfecabb`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | site/profile 외에 작업 결과와 외부 행동을 표현할 domain type이 없었습니다. |
| 실제 변경 file/symbol/call path | 프로젝트 상세 필드, deployment 상태, screenshot, link type 및 environment key shape를 추가합니다. |
| Data/state/resource owner와 lifetime | project/link record의 shape는 types가 소유하지만 실제 collection은 비어 있어 consumer evidence는 없습니다. |
| Failure·absence·fallback 처리 | 빈 배열도 유효하게 보이며 ID uniqueness·link 안전성·참조 존재성은 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 향후 renderer가 사용할 기본 project/link vocabulary만 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `0d891d41cf4c`이 기술·경험·여정 domain을 추가합니다. |

#### 코드·실행 증거

정적 근거: `5eb01dfecabb`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 3. `0d891d41cf4c` — feat(content): 기술과 여정 콘텐츠 모델 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 경력/기술/여정 source 확장
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `tech-stack.json`, `skills.json`, `experience.json`, `journey.json`과 대응 type을 확인합니다.
- journey의 `projectId` nullability와 technology ID reference가 단순 문자열인지 확인합니다.

확인 원칙:

- 먼저 `0d891d41cf4c^`와 `0d891d41cf4c`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | project 외의 역량·시간 흐름을 표현할 source가 없었습니다. |
| 실제 변경 file/symbol/call path | 기술 stack, skill group, experience, journey item의 JSON과 TypeScript shape를 추가합니다. |
| Data/state/resource owner와 lifetime | 각 파일이 raw record를 소유하며 cross-file ID 관계는 문자열 약속에 머뭅니다. |
| Failure·absence·fallback 처리 | 존재하지 않는 기술/project ID도 compile-time에는 걸러지지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | aggregate에 포함될 domain 범위가 넓어지지만 참조 무결성은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `8661cc00c45d`가 contact/resume와 전체 aggregate shape를 만듭니다. |

#### 코드·실행 증거

정적 근거: `0d891d41cf4c`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 4. `8661cc00c45d` — feat(content): 연락과 이력 집계 모델 완성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** aggregate type 완성
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `contact.json`, `resume.json`과 `ContactContent`, `ResumeContent`를 확인합니다.
- `PortfolioContent`, `PortfolioEnv`, `RouteSearchParams`가 어떤 하위 source를 묶는지 확인합니다.

확인 원칙:

- 먼저 `8661cc00c45d^`와 `8661cc00c45d`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 여러 source type은 존재했지만 renderer가 받을 단일 aggregate 계약이 없었습니다. |
| 실제 변경 file/symbol/call path | contact/resume source와 함께 모든 domain을 묶는 `PortfolioContent`, public env shape, route search params를 정의합니다. |
| Data/state/resource owner와 lifetime | aggregate의 정적 계약은 `types.ts`가 소유하지만 실제 construction은 아직 없습니다. |
| Failure·absence·fallback 처리 | type alias만으로 disabled filtering, ordering, env resolution 또는 runtime 검증은 발생하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | consumer가 기대할 전체 필드 집합은 정의하지만 값의 신뢰성은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `a365b3d19118`이 JSON imports를 실제 aggregate module에 연결합니다. |

#### 코드·실행 증거

정적 근거: `8661cc00c45d`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 5. `a365b3d19118` — feat(content): 정적 포트폴리오 콘텐츠 로딩

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 직접 JSON import 기반 조립
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/content.ts`의 JSON imports와 `as` assertions를 확인합니다.
- 어떤 source가 module-level singleton으로 저장되고 `getPortfolioContent()`가 무엇을 반환하는지 확인합니다.

확인 원칙:

- 먼저 `a365b3d19118^`와 `a365b3d19118`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 정적 aggregate type은 있었지만 JSON을 한 곳에서 읽고 반환하는 구현이 없었습니다. |
| 실제 변경 file/symbol/call path | `content.ts`가 source files를 import하고 수동 assertion으로 typed module values를 만든 뒤 aggregate 반환점을 제공합니다. |
| Data/state/resource owner와 lifetime | module-level imports가 source lifetime을 소유하고 호출자는 반환 aggregate를 소비합니다. |
| Failure·absence·fallback 처리 | `as`는 검증이 아니므로 잘못된 JSON이 fail-closed 되지 않습니다. 파생 정렬/filter도 아직 분산될 여지가 있습니다. |
| 보장하는 것과 보장하지 않는 것 | JSON direct import를 한 module로 모으지만 안전한 trust boundary는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `0b134b1a6cf6`이 나머지 source와 파생 map/sort를 연결합니다. |

#### 코드·실행 증거

정적 근거: `a365b3d19118`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 6. `0b134b1a6cf6` — feat(content): 여정 정렬과 콘텐츠 인덱스 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 전체 source와 파생 index 연결
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `content.ts`에서 skills/tech/experience/journey/links/contact/resume import를 확인합니다.
- journey의 date→title 정렬, `portfolioTechStackById` Map, `getEnabledLinks()`를 확인합니다.

확인 원칙:

- 먼저 `0b134b1a6cf6^`와 `0b134b1a6cf6`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 초기 조립 모듈은 일부 source만 반환하고 반복 lookup·정렬 정책이 정해지지 않았습니다. |
| 실제 변경 file/symbol/call path | 나머지 source를 aggregate에 넣고 journey를 copy 후 정렬하며 technology Map과 enabled link filter를 중앙화합니다. |
| Data/state/resource owner와 lifetime | module이 파생 collection과 lookup index를 소유하고 renderer는 정렬/filter 구현을 알 필요가 없어집니다. |
| Failure·absence·fallback 처리 | Map은 unknown ID를 `undefined`로 반환할 뿐 누락을 오류로 바꾸지 않습니다. source assertion 문제도 남습니다. |
| 보장하는 것과 보장하지 않는 것 | 일관된 journey order와 enabled top-level links를 보장하지만 project enablement/env resolution은 아직 불완전합니다. |
| 다음 commit 또는 관련 test 연결 | `7c95a6f387b4`가 project/link 활성화와 env href를 완성합니다. |

#### 코드·실행 증거

정적 근거: `0b134b1a6cf6`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 7. `7c95a6f387b4` — feat(content): 환경 링크를 반영한 콘텐츠 집계

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 초기 aggregate policy 완성
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `withEnvHref`, `getPortfolioContent`의 env default와 project/link mapping을 확인합니다.
- project `enabled !== false`, nested link `enabled !== false`, top-level `getEnabledLinks()`의 차이를 확인합니다.

확인 원칙:

- 먼저 `7c95a6f387b4^`와 `7c95a6f387b4`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | aggregate는 source를 모았지만 disabled project/link와 environment-specific href를 일관되게 해석하지 않았습니다. |
| 실제 변경 file/symbol/call path | public env 값을 trim해 non-empty일 때 href를 덮어쓰고, disabled project와 nested/top-level links를 제거합니다. |
| Data/state/resource owner와 lifetime | `getPortfolioContent()`가 활성화·href resolution policy를 소유하며 호출자는 filtered aggregate만 받습니다. |
| Failure·absence·fallback 처리 | environment 값 자체의 URL 안전성, cross-file 참조, schema validation은 보장하지 않습니다. module-level source 객체 일부는 계속 공유됩니다. |
| 보장하는 것과 보장하지 않는 것 | 초기 domain aggregate의 호출 계약을 완성하지만 후속 validated facade 이전에는 source를 신뢰합니다. |
| 다음 commit 또는 관련 test 연결 | T4가 selector policy를 확장하고 T8–T10이 runtime validation과 validated facade로 교체합니다. |

#### 코드·실행 증거

정적 근거: `7c95a6f387b4`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| JSON source와 renderer aggregate를 분리한다. | `efb1e2e26b74` → `a365b3d19118` | `src/content/*.json`, `src/lib/portfolio/content.ts` | 초기 연결은 assertion 기반 |
| journey order와 lookup/filter 정책은 aggregate module이 소유한다. | `0b134b1a6cf6` | date/title sort, `portfolioTechStackById`, `getEnabledLinks` | unknown references는 오류가 아님 |
| disabled project/link와 env href를 호출 전에 해석한다. | `7c95a6f387b4` | `getPortfolioContent`, `withEnvHref` | URL/schema/cross-file integrity는 후속 loader 책임 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| 각 component가 JSON을 직접 import할 위험 | `a365b3d19118` | 한 aggregate module로 import 집중 | `508e0b71024b`에서 validated source로 교체 |
| journey 순서와 tech lookup이 consumer마다 달라질 위험 | `0b134b1a6cf6` | 정렬과 Map 중앙화 | `b77b386b344e`의 view-model regression이 후속 보호 |
| disabled/env link가 그대로 노출될 위험 | `7c95a6f387b4` | explicit false filter와 non-empty env override | 후속 schema/route/link tests가 허용 범위를 검증 |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| 원본 값 | component 하드코딩 가능 | `src/content/*.json` | 각 JSON file |
| aggregate shape | 분산 type | `PortfolioContent` | `types.ts` |
| 파생 정렬/index/filter | 호출자 책임 | `content.ts` | journey sort, tech Map, enabled filters |
| runtime 신뢰 | 없음 | 여전히 없음 | `as` assertion; T8 이후 loader가 인수 |

## 9. Thread 최종 상태

Thread 종료 시점에는 `getPortfolioContent()`가 모든 JSON source를 하나의 aggregate로 묶고 journey 정렬, technology lookup, enabled filtering, environment href resolution을 수행합니다. 그러나 이 aggregate는 runtime schema나 cross-file integrity 검증 없이 source를 assertion으로 신뢰합니다.

### 최종 설명

- site/profile에서 project, technology, experience, journey, contact, resume로 domain source 범위를 확장했습니다.
- 단일 `PortfolioContent`와 `content.ts`가 renderer-facing aggregate를 소유하게 했습니다.
- 정렬·lookup·enabled/env 정책을 consumer 밖으로 이동했습니다.
- 이 단계의 핵심 비보장은 `as` 기반 source 신뢰이며 T8–T10이 이를 제거합니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| JSON modules를 import합니다. | `src/lib/portfolio/content.ts` | module-level source objects | malformed JSON shape를 runtime에서 거부하지 않음 |
| 파생 collection을 준비합니다. | journey sort, tech Map, enabled links | 정렬된 배열과 lookup/filter | unknown ID는 `undefined` 또는 omission |
| 환경 링크와 project 활성화를 해석합니다. | `getPortfolioContent()` | filtered projects/links | invalid URL은 별도 검증 없음 |
| aggregate를 route에 반환합니다. | `PortfolioContent` | renderer-facing object | nested 공유/복사 경계는 후속 test에서 고정 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b`, `dc07871c4d24`, `b77b386b344e`, `527b9f872333`이 이후 public export, clone boundary, selector/view-model 결과를 테스트합니다. 이 Thread에서는 테스트 command를 실행하지 않았습니다.
===== END FILE: 02-portfolio-domain-and-aggregate-model.md =====

===== BEGIN FILE: 03-presentation-contracts-for-multi-route-ui.md =====
# Thread: Presentation contracts for multi-route UI

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

한 개 home placeholder에서 다섯 design과 여러 route가 공유·분리하는 copy, section order, metric key, ARIA/empty-state vocabulary까지 `presentation.json` 계약으로 성장하는 과정을 복원합니다.

### 계획된 핵심 invariant

- route/design별 표시 문구와 section order는 renderer 코드가 아니라 presentation source가 소유합니다.
- count key, section ID, empty/ARIA label은 제한된 vocabulary로 소비됩니다.
- placeholder 추가와 실제 copy 완성을 구분하고, JSON 변경만으로 renderer 지원을 과장하지 않습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Design/Classic의 초기 home 계약에서 다섯 design·다중 route로 어떻게 확장되는가?
- 새 copy가 추가된 commit과 실제 placeholder가 완성된 commit은 각각 무엇인가?
- 공용 vocabulary와 design-specific vocabulary의 경계는 어디인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7f5017b21d37` | feat(content): 디자인 홈 표현 모델 추가 | B | CONTENT | Design home 계약 시작 |
| 2 | `04a810bb0ab4` | feat(content): 클래식과 공용 홈 표현 추가 | B | CONTENT | Classic/shared home 계약 |
| 3 | `d21d53591b5c` | feat(content): 프로젝트 목록 표현 계약 정의 | B | CONTENT | projects route type 계약 |
| 4 | `d55a2017e725` | feat(content): 프로젝트 목록 화면 문구 추가 | B | CONTENT | projects route placeholder source |
| 5 | `d6468cbea9e2` | feat(content): 보조 페이지 표현 계약 정의 | B | CONTENT | detail/about/resume/contact 계약 |
| 6 | `da3941184155` | feat(content): 상세 소개 이력 연락 문구 추가 | B | CONTENT | 초기 route copy source |
| 7 | `96c8ba5733f5` | feat(content): 공용 UI 표현 콘텐츠 구성 | B | CONTENT | 공용 ARIA/empty-state vocabulary |
| 8 | `9a7d41edfad0` | feat(content): Design과 Classic 홈 표현 콘텐츠 구성 | B | CONTENT, RENDERER | 초기 두 home의 의미 있는 구성 |
| 9 | `2b9b35d4b8de` | feat(content): 확장 디자인 홈 표현 콘텐츠 구성 | B | CONTENT | Editorial/Brutalist/Cinematic 계약 |
| 10 | `8886459d1b0d` | feat(content): 공용 홈 섹션 표현 콘텐츠 구성 | B | CONTENT | shared home copy 완성 |
| 11 | `61d1976cde0d` | feat(content): 프로젝트 목록 표현 콘텐츠 구성 | B | CONTENT | 다섯 design projects route copy |
| 12 | `a6c72a6b3b34` | feat(content): 프로젝트 상세 표현 콘텐츠 구성 | B | CONTENT | project detail copy 완성 |
| 13 | `20dfc298375c` | feat(content): About과 Journey 표현 콘텐츠 구성 | B | CONTENT, RENDERER | About/Journey narrative 계약 |
| 14 | `13c8c52c54d9` | feat(content): Interview Map과 Resume 표현 콘텐츠 구성 | B | CONTENT, RENDERER | Resume/Interview Map 계약 완성 |
| 15 | `a7a2000ff462` | feat(content): Contact 표현 콘텐츠와 최종 문서 형식 구성 | B | CONTENT, RENDERER | 문서형 route 최종 구성 |

## 5. Commit별 학습 기록

### 1. `7f5017b21d37` — feat(content): 디자인 홈 표현 모델 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** Design home 계약 시작
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json`의 `defaultHomeTemplate`, templates, `home.design`을 확인합니다.
- `types.ts`의 home section/count key/Design hero type을 확인합니다.

확인 원칙:

- 먼저 `7f5017b21d37^`와 `7f5017b21d37`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | home 문구와 section order가 renderer 내부에 머물 수 있었습니다. |
| 실제 변경 file/symbol/call path | Design/Classic template ID와 Design hero stats, section order, featured copy를 presentation source/type으로 이동합니다. |
| Data/state/resource owner와 lifetime | `presentation.json`이 표시 순서와 문구를, renderer가 구조/스타일을 소유하도록 경계를 시작합니다. |
| Failure·absence·fallback 처리 | Classic 세부 계약과 다른 route는 없고 runtime schema도 없습니다. |
| 보장하는 것과 보장하지 않는 것 | Design home의 데이터 기반 표현 계약을 보장하지만 multi-route 지원은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `04a810bb0ab4`가 Classic terminal과 shared home copy를 추가합니다. |

#### 코드·실행 증거

정적 근거: `7f5017b21d37`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `04a810bb0ab4` — feat(content): 클래식과 공용 홈 표현 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** Classic/shared home 계약
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `home.classic.terminal`, shared `workMap`, `technicalFocus`, `stack`, `journey`, `contact`를 확인합니다.
- terminal command/output 배열의 소유 위치를 확인합니다.

확인 원칙:

- 먼저 `04a810bb0ab4^`와 `04a810bb0ab4`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Design home만 source-driven이고 Classic terminal과 공용 section copy는 코드에 남을 수 있었습니다. |
| 실제 변경 file/symbol/call path | Classic hero/featured/terminal과 두 renderer가 공유할 home section copy를 presentation source에 추가합니다. |
| Data/state/resource owner와 lifetime | terminal script-like text와 shared section copy는 JSON이 소유하고 component는 형식만 렌더링합니다. |
| Failure·absence·fallback 처리 | 값은 대부분 placeholder이고 section ID 유일성이나 template consistency는 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 두 초기 design의 공용/전용 copy 경계를 정의하지만 완성된 editorial content는 아닙니다. |
| 다음 commit 또는 관련 test 연결 | `d21d53591b5c`부터 projects route 계약이 분리됩니다. |

#### 코드·실행 증거

정적 근거: `04a810bb0ab4`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 3. `d21d53591b5c` — feat(content): 프로젝트 목록 표현 계약 정의

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** projects route type 계약
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `ProjectPageContent`와 Design/Classic hero stats, terminal, selected/grouped shape를 확인합니다.
- countKey가 renderer 계산 결과와 연결될 단순 문자열 union인지 확인합니다.

확인 원칙:

- 먼저 `d21d53591b5c^`와 `d21d53591b5c`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | home 외 projects index의 문구·통계 label·terminal 설정이 code-local일 수 있었습니다. |
| 실제 변경 file/symbol/call path | projects list page의 Design/Classic presentation type을 추가하여 route-level copy contract를 분리합니다. |
| Data/state/resource owner와 lifetime | presentation type이 route copy shape를 소유하지만 JSON 값은 다음 commit까지 없습니다. |
| Failure·absence·fallback 처리 | 타입만 추가되어 runtime 데이터 존재나 count key 유효 계산은 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | projects route가 요구할 shape를 명시하지만 실제 source 연결은 다음 commit 책임입니다. |
| 다음 commit 또는 관련 test 연결 | `d55a2017e725`가 placeholder source를 추가합니다. |

#### 코드·실행 증거

정적 근거: `d21d53591b5c`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 4. `d55a2017e725` — feat(content): 프로젝트 목록 화면 문구 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** projects route placeholder source
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json.pages.projects`의 Design/Classic 구조를 확인합니다.
- 빈 `groups`, placeholder body, stats/countKey, terminal `maxGroups`를 구분합니다.

확인 원칙:

- 먼저 `d55a2017e725^`와 `d55a2017e725`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | projects route type은 있었지만 source 값이 없어 renderer가 계약을 소비할 수 없었습니다. |
| 실제 변경 file/symbol/call path | Design/Classic projects page의 hero, stats, featured, terminal, selected/grouped 문구를 placeholder 형태로 추가합니다. |
| Data/state/resource owner와 lifetime | route copy는 JSON이 소유하지만 groups는 빈 배열이고 실제 group description은 아직 연결되지 않습니다. |
| Failure·absence·fallback 처리 | 구조 존재만 보장하며 의미 있는 copy와 모든 design 지원은 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | projects route source contract의 최소 값을 제공하지만 완성 상태는 아닙니다. |
| 다음 commit 또는 관련 test 연결 | `d6468cbea9e2`가 다른 route type/source 계약을 확장합니다. |

#### 코드·실행 증거

정적 근거: `d55a2017e725`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 5. `d6468cbea9e2` — feat(content): 보조 페이지 표현 계약 정의

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** detail/about/resume/contact 계약
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `types.ts`의 project detail section, About, Resume, Contact page content와 `PresentationContent.pages`를 확인합니다.
- project detail section key 집합과 route별 필수 field를 확인합니다.

확인 원칙:

- 먼저 `d6468cbea9e2^`와 `d6468cbea9e2`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | projects index 외 상세/정보 route는 code-local copy에 의존할 수 있었습니다. |
| 실제 변경 file/symbol/call path | project detail, About, Resume, Contact의 page-level presentation type을 aggregate에 추가합니다. |
| Data/state/resource owner와 lifetime | `PresentationContent.pages`가 multi-route copy contract를 소유합니다. |
| Failure·absence·fallback 처리 | Journey/Interview Map과 다섯 design-specific 확장은 없고 JSON values도 placeholder 단계입니다. |
| 보장하는 것과 보장하지 않는 것 | 초기 핵심 route의 type contract를 보장하지만 complete copy와 runtime parse는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `da3941184155`가 실제 route copy를 채웁니다. |

#### 코드·실행 증거

정적 근거: `d6468cbea9e2`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 6. `da3941184155` — feat(content): 상세 소개 이력 연락 문구 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 초기 route copy source
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json.pages`의 projectDetail/about/resume/contact 값을 확인합니다.
- placeholder와 실제 label을 구분하고 renderer가 아직 소비하지 않는 field가 있는지 확인합니다.

확인 원칙:

- 먼저 `da3941184155^`와 `da3941184155`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | route type은 있었지만 JSON에 대응 값이 없어 source-driven rendering이 불완전했습니다. |
| 실제 변경 file/symbol/call path | 상세/About/Resume/Contact의 heading, label, action copy를 presentation source에 추가합니다. |
| Data/state/resource owner와 lifetime | route copy owner가 component에서 JSON으로 이동할 기반이 생깁니다. |
| Failure·absence·fallback 처리 | 여러 값은 placeholder이고 모든 component가 즉시 이 source를 소비한다는 보장은 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 필수 route copy 구조를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `96c8ba5733f5`가 공용 UI vocabulary와 template description을 추가합니다. |

#### 코드·실행 증거

정적 근거: `da3941184155`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 7. `96c8ba5733f5` — feat(content): 공용 UI 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 공용 ARIA/empty-state vocabulary
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json.ui`의 debug/skip/nav/switcher/ARIA/emptyStates를 확인합니다.
- template description과 public UI label이 component-local literal을 대체할 범위를 확인합니다.

확인 원칙:

- 먼저 `96c8ba5733f5^`와 `96c8ba5733f5`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | navigation, accessibility label, empty-state 문구가 여러 component에 중복될 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | 공용 UI label과 ARIA template, empty-state vocabulary를 presentation source에 중앙화합니다. |
| Data/state/resource owner와 lifetime | 문구의 owner는 공용 JSON node로 이동하고 component는 format/placement만 담당합니다. |
| Failure·absence·fallback 처리 | 문구가 존재해도 실제 semantic markup과 screen-reader 동작을 검증하지는 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 공용 vocabulary 일관성을 제공하지만 accessibility behavior 자체는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `9a7d41edfad0`이 Design/Classic home placeholder를 실제 copy와 section order로 완성합니다. |

#### 코드·실행 증거

정적 근거: `96c8ba5733f5`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 8. `9a7d41edfad0` — feat(content): Design과 Classic 홈 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** 초기 두 home의 의미 있는 구성
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json.home.design/classic.sections`가 여섯 section으로 채워지는 diff를 확인합니다.
- Design action/featured copy와 Classic terminal commands/output placeholder 교체를 확인합니다.

확인 원칙:

- 먼저 `9a7d41edfad0^`와 `9a7d41edfad0`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 두 home 계약은 있었지만 빈 section order와 placeholder terminal/copy 때문에 실제 정보 구조가 정해지지 않았습니다. |
| 실제 변경 file/symbol/call path | Design/Classic의 section order를 채우고 action label, featured copy, terminal command sequence를 실제 콘텐츠로 교체합니다. |
| Data/state/resource owner와 lifetime | home 정보 구조와 terminal narrative는 JSON이 소유합니다. |
| Failure·absence·fallback 처리 | 다른 세 design과 shared section body는 아직 완성되지 않았고 renderer behavior는 이 JSON diff만으로 증명되지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 초기 두 home의 완성된 순서/문구 계약을 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `2b9b35d4b8de`가 추가 design 계약을 만들고 `8886459d1b0d`이 shared copy를 완성합니다. |

#### 코드·실행 증거

정적 근거: `9a7d41edfad0`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 9. `2b9b35d4b8de` — feat(content): 확장 디자인 홈 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** Editorial/Brutalist/Cinematic 계약
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json`과 type에서 editorial/brutalist/cinematic shell/home nodes를 확인합니다.
- 각 design의 section ID vocabulary와 action label 차이를 확인합니다.

확인 원칙:

- 먼저 `2b9b35d4b8de^`와 `2b9b35d4b8de`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Design/Classic 외 visual system은 공용 contract에 표현할 위치가 없었습니다. |
| 실제 변경 file/symbol/call path | 세 추가 design의 shell/home section order, hero/action, design-specific copy shape를 추가합니다. |
| Data/state/resource owner와 lifetime | design별 정보 구조는 JSON이, 실제 layout/visual language는 각 renderer가 소유합니다. |
| Failure·absence·fallback 처리 | JSON node 추가만으로 renderer 구현·lazy loading·visual regression은 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 다섯 design을 표현할 source contract를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `8886459d1b0d`과 후속 route copy commits가 placeholder를 실제 콘텐츠로 바꿉니다. |

#### 코드·실행 증거

정적 근거: `2b9b35d4b8de`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 10. `8886459d1b0d` — feat(content): 공용 홈 섹션 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** shared home copy 완성
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `home.shared.workMap` card IDs/labels/body/countKey와 technicalFocus/stack/journey/contact diff를 확인합니다.
- `curriculum` card ID가 `archive`로 바뀐 의미를 확인합니다.

확인 원칙:

- 먼저 `8886459d1b0d^`와 `8886459d1b0d`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 공용 home sections는 placeholder body와 임시 card vocabulary를 사용했습니다. |
| 실제 변경 file/symbol/call path | work-map 분류를 product/archive/reliability로 정리하고 각 shared section의 실제 설명과 contact title을 채웁니다. |
| Data/state/resource owner와 lifetime | 공용 cross-design narrative는 shared node가 소유합니다. |
| Failure·absence·fallback 처리 | count 값 계산과 card ID 참조 무결성은 selector/schema/loader가 별도로 책임집니다. |
| 보장하는 것과 보장하지 않는 것 | 공용 문구 일관성을 제공하지만 계산 결과를 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `61d1976cde0d`부터 다섯 design의 projects route copy가 완성됩니다. |

#### 코드·실행 증거

정적 근거: `8886459d1b0d`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 11. `61d1976cde0d` — feat(content): 프로젝트 목록 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 다섯 design projects route copy
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json.pages.projects`의 groups와 editorial/brutalist/cinematic nodes를 확인합니다.
- 초기 Design/Classic placeholder가 실제 archive copy로 교체되는지 확인합니다.

확인 원칙:

- 먼저 `61d1976cde0d^`와 `61d1976cde0d`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | projects route는 초기 두 design placeholder만 갖고 추가 design 표현 계약이 비어 있었습니다. |
| 실제 변경 file/symbol/call path | group copy와 다섯 design의 hero/archive vocabulary를 실제 콘텐츠로 구성합니다. |
| Data/state/resource owner와 lifetime | projects route의 design-specific copy는 presentation source가 소유합니다. |
| Failure·absence·fallback 처리 | group/project 실제 데이터 결합과 route rendering은 별도 selector/view-model 책임입니다. |
| 보장하는 것과 보장하지 않는 것 | 모든 design이 projects route에 필요한 copy를 얻습니다. |
| 다음 commit 또는 관련 test 연결 | `a6c72a6b3b34`가 project detail copy를 확장합니다. |

#### 코드·실행 증거

정적 근거: `61d1976cde0d`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 12. `a6c72a6b3b34` — feat(content): 프로젝트 상세 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** project detail copy 완성
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `pages.projectDetail`의 missing/facts/outro/frame/editorial/sections를 확인합니다.
- highlights section과 기존 section label의 최종 집합을 확인합니다.

확인 원칙:

- 먼저 `a6c72a6b3b34^`와 `a6c72a6b3b34`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | project detail에는 기본 heading만 있고 missing state, facts, outro, design-specific label이 불완전했습니다. |
| 실제 변경 file/symbol/call path | 없는 프로젝트 fallback과 상세 facts/outro/frame, highlights 포함 section vocabulary를 채웁니다. |
| Data/state/resource owner와 lifetime | 상세 상태 문구는 presentation source가, project 존재 판단은 route/view-model이 소유합니다. |
| Failure·absence·fallback 처리 | 문구는 missing state를 표현하지만 404 status나 route generation behavior는 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | project detail renderer가 사용할 완전한 copy contract를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `20dfc298375c`이 About/Journey route 구조를 완성합니다. |

#### 코드·실행 증거

정적 근거: `a6c72a6b3b34`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 13. `20dfc298375c` — feat(content): About과 Journey 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** About/Journey narrative 계약
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `pages.about.curation`과 `pages.journey` hero/narrative/timeline/now를 확인합니다.
- state/reason/result labels와 anchor label을 확인합니다.

확인 원칙:

- 먼저 `20dfc298375c^`와 `20dfc298375c`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | About curation과 Journey narrative가 독립 source를 표시할 route copy 계약이 부족했습니다. |
| 실제 변경 file/symbol/call path | curation 설명/section label과 Journey의 현재 상태·이유·결과·timeline/current-position vocabulary를 추가합니다. |
| Data/state/resource owner와 lifetime | route copy는 presentation source가, 실제 milestone/project resolution은 domain/view-model이 소유합니다. |
| Failure·absence·fallback 처리 | 참조 project 존재성과 timeline chronology는 이 JSON 변경이 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | About/Journey route가 source-driven section structure를 갖습니다. |
| 다음 commit 또는 관련 test 연결 | `13c8c52c54d9`가 Resume/Interview Map을 완성합니다. |

#### 코드·실행 증거

정적 근거: `20dfc298375c`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 14. `13c8c52c54d9` — feat(content): Interview Map과 Resume 표현 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** Resume/Interview Map 계약 완성
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `pages.resume`의 experience/education/notes/identity/editorial/brutalist를 확인합니다.
- `pages.interviewMap`의 hero/tracks/gaps label과 empty label을 확인합니다.

확인 원칙:

- 먼저 `13c8c52c54d9^`와 `13c8c52c54d9`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Resume는 기본 summary/projects/training만, Interview Map은 route-level presentation copy가 없었습니다. |
| 실제 변경 file/symbol/call path | Resume의 identity·experience·education·notes와 design-specific eyebrow, Interview Map의 evidence index/empty/gaps vocabulary를 추가합니다. |
| Data/state/resource owner와 lifetime | 문서 구조와 label은 presentation source가 소유하고 실제 project evidence mapping은 domain content가 소유합니다. |
| Failure·absence·fallback 처리 | download asset 존재성이나 answer project 참조는 이 commit이 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 두 route의 presentation contract를 완성합니다. |
| 다음 commit 또는 관련 test 연결 | `a7a2000ff462`가 Contact/Journey/Interview Map의 최종 문서 순서와 copy를 다듬습니다. |

#### 코드·실행 증거

정적 근거: `13c8c52c54d9`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 15. `a7a2000ff462` — feat(content): Contact 표현 콘텐츠와 최종 문서 형식 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** 문서형 route 최종 구성
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentation.json`의 Contact copy와 Journey/Interview Map grouping/reordering diff를 확인합니다.
- 삭제·재배치된 field가 단순 copy 수정인지 contract shape 변경인지 구분합니다.

확인 원칙:

- 먼저 `a7a2000ff462^`와 `a7a2000ff462`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 문서형 route의 copy는 존재했지만 Contact와 evidence-oriented 화면의 읽기 순서가 최종 정보 구조와 맞지 않았습니다. |
| 실제 변경 file/symbol/call path | Contact 문구를 완성하고 Journey/Interview Map nodes를 renderer가 소비할 최종 문서 순서로 재구성합니다. |
| Data/state/resource owner와 lifetime | presentation source가 route의 문서 계층을 소유합니다. |
| Failure·absence·fallback 처리 | 실제 DOM heading hierarchy, responsive layout, accessibility behavior는 renderer/test 책임입니다. |
| 보장하는 것과 보장하지 않는 것 | 다중 route presentation source의 최종 shape를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | T6가 같은 shape를 runtime schema로 제한하고 후속 renderer/view-model tests가 소비 경로를 보호합니다. |

#### 코드·실행 증거

정적 근거: `a7a2000ff462`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| 표시 문구와 section order는 JSON source가 소유한다. | `7f5017b21d37` → `a7a2000ff462` | `src/content/presentation.json` | renderer는 구조·스타일·interaction을 유지 |
| 공용 UI/empty/ARIA vocabulary는 한 node에서 공유한다. | `96c8ba5733f5` | `presentation.ui` | semantic behavior 자체는 별도 검증 |
| 다섯 design과 핵심 routes가 명시적 copy 계약을 가진다. | `2b9b35d4b8de` → `13c8c52c54d9` | `home.*`, `pages.*` | runtime validity는 T6가 인수 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| 문구·순서가 renderer마다 하드코딩될 위험 | `7f5017b21d37` 이후 | presentation source와 type으로 이동 | 후속 route/view-model tests가 source consumption을 검증 |
| placeholder/빈 section이 정상 콘텐츠처럼 노출될 위험 | `9a7d41edfad0`, `8886459d1b0d`, route copy commits | 실제 section order와 body로 교체 | visual/content completeness 자체는 자동 증명하지 않음 |
| 추가 design/route가 계약 밖 field를 요구할 위험 | `2b9b35d4b8de`와 route 확장 | design/page별 node 추가 | T6 runtime schema가 known fields를 검사 |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| 표시 source | component literal | `presentation.json` | home/pages/ui nodes |
| section order | renderer 고정 | design별 arrays | `home.<design>.sections` |
| 공용 label | 여러 component | `presentation.ui`, `home.shared` | ARIA/empty/work-map copy |
| 실제 data resolution | 혼재 가능 | presentation 밖 selector/view-model | T4·후속 view-model Thread |

## 9. Thread 최종 상태

Thread 종료 시점에는 다섯 design과 Projects, Project Detail, About, Resume, Contact, Journey, Interview Map이 presentation source의 명시적 copy/section 계약을 공유합니다. JSON은 무엇을 말하고 어떤 순서로 보여 줄지를 소유하지만 실제 data resolution, DOM semantics, runtime schema와 visual behavior는 별도 계층이 책임집니다.

### 최종 설명

- Design/Classic home에서 시작해 공용 section과 다섯 design-specific 계약으로 확장했습니다.
- Projects와 문서형 routes의 placeholder를 실제 copy와 section order로 교체했습니다.
- ARIA/empty/count-key vocabulary를 공용 source로 이동했습니다.
- 이 Thread는 presentation 데이터의 의미를 다루며 runtime parsing과 renderer 동작을 직접 보장하지 않습니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| route가 active design과 page copy key를 선택합니다. | `presentation.defaultHomeTemplate`, `templates`, `pages` | design/page node | 없는 key는 당시 type assertion 단계에서 runtime 거부 없음 |
| presentation copy를 읽습니다. | `presentation.json` | section order, labels, templates | placeholder 여부는 자동 판별하지 않음 |
| domain/view model 값과 결합합니다. | T4 selectors 및 route view models | count/project/evidence와 copy | unknown references는 T9 loader가 후속 차단 |
| design renderer가 구조와 스타일을 적용합니다. | 각 design route renderer | DOM/UI | visual/ARIA behavior는 category 05/07 검증 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b`은 다섯 design과 presentation validation을, `b77b386b344e`·`527b9f872333`은 route별 view-model/scoped payload를 후속 검증합니다. 이번 작업에서 테스트는 실행하지 않았습니다.
===== END FILE: 03-presentation-contracts-for-multi-route-ui.md =====

===== BEGIN FILE: 04-selectors-links-and-derived-content-policy.md =====
# Thread: Selectors, links, and derived content policy

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

renderer와 route에 흩어질 수 있는 lookup, fallback, enabled page, link placement, project metric 계산을 pure selector policy로 중앙화하고 실제 consumer가 이를 사용하도록 전환하는 과정을 복원합니다.

### 계획된 핵심 invariant

- lookup/fallback/filter/metric 계산은 selector가 소유하고 renderer는 결과를 표현합니다.
- page enablement는 `false`만 차단하는 명시적 fail-open 정책입니다.
- link placement와 live deployment 조건은 같은 selector 경로에서 함께 적용됩니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- unknown technology, missing reference, disabled page는 각각 fallback·omission·false 중 무엇으로 처리되는가?
- metric filter의 여러 조건은 AND인가 OR인가?
- placement vocabulary가 도입된 뒤 어떤 consumer의 임의 type filter가 제거되는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `eb988f5e09e4` | feat(portfolio): 기술과 프로젝트 조회기 추가 | B | CONTENT | 기본 lookup/fallback selectors |
| 2 | `ba8da56d3fcf` | feat(portfolio): 연락과 프로젝트 링크 선택기 추가 | A | CONTENT | link/deployment 정책 중앙화 |
| 3 | `3e2e95a3a28c` | feat(content): 페이지 활성화 selector 추가 | B | CONTENT, ROUTING | page enablement selector |
| 4 | `7c539b142d6d` | feat(content): 프로젝트 지표 selector 추가 | B | CONTENT | declarative metric evaluator |
| 5 | `daa6815a6dfa` | feat(project): 카드 링크를 콘텐츠 배치 기준으로 선택 | B | CONTENT, RENDERER | card placement 최초 소비 |
| 6 | `383a3b86e119` | feat(content): 프로젝트 지표를 화면에 적용 | B | CONTENT | metric selector consumer migration |
| 7 | `119ff9a92090` | feat(content): 링크 배치 selector 추가 | B | CONTENT | 공용 LinkPlacement vocabulary |
| 8 | `2d87b62dcce8` | refactor(project): 상세 링크를 배치 기준으로 선택 | B | RENDERER, REFACTOR | detail component consumer migration |
| 9 | `ee2c118a76d6` | feat(content): 홈 링크를 배치 기준으로 선택 | B | CONTENT | home hero consumer migration |

## 5. Commit별 학습 기록

### 1. `eb988f5e09e4` — feat(portfolio): 기술과 프로젝트 조회기 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 기본 lookup/fallback selectors
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/selectors.ts`의 technology fallback, featured/project-by-id, resume project selectors를 확인합니다.
- unknown tech와 missing project ID가 각각 어떤 결과를 내는지 확인합니다.

확인 원칙:

- 먼저 `eb988f5e09e4^`와 `eb988f5e09e4`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | renderer가 Map lookup, featured filtering, resume ID resolution을 직접 반복할 수 있었습니다. |
| 실제 변경 file/symbol/call path | unknown technology를 `tool` fallback과 기본 색으로 정규화하고, project lookup/featured/resume resolution을 함수로 모읍니다. |
| Data/state/resource owner와 lifetime | selector가 파생 정책을 소유하며 component는 반환된 record만 소비합니다. |
| Failure·absence·fallback 처리 | missing resume/project reference는 omission 또는 `undefined`가 되어 오류가 아닙니다. cross-file validation은 아직 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 일관된 fallback/omission을 보장하지만 source integrity를 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `ba8da56d3fcf`가 link/deployment policy를 확장합니다. |

#### 코드·실행 증거

정적 근거: `eb988f5e09e4`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `ba8da56d3fcf` — feat(portfolio): 연락과 프로젝트 링크 선택기 추가

- **Importance:** A
- **Tags:** CONTENT
- **Thread 역할:** link/deployment 정책 중앙화
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `getPreferredContactLinks`, project link selectors, `isProjectLive`, `getProjectCardLinks`, external link props를 확인합니다.
- demo link가 live deployment 조건을 통과해야 하는지와 source/github/case-study 처리 차이를 확인합니다.
- barrel export에서 public selector surface를 확인합니다.

확인 원칙:

- 먼저 `ba8da56d3fcf^`와 `ba8da56d3fcf`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | contact preferred IDs, project action links, live badge/external props를 component마다 type과 deployment 상태로 판단할 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | preferred contact ID order를 보존해 enabled links를 resolve하고, project link type별 선택, live 상태, card action, external target/rel props를 selector로 중앙화합니다. |
| Data/state/resource owner와 lifetime | `selectors.ts`가 선택 정책을 소유하고 components는 link record/props를 표현합니다. missing preferred IDs는 생략되고 demo는 live일 때만 노출됩니다. |
| Failure·absence·fallback 처리 | 아직 `placements`가 없어 card/detail/hero 문맥을 link type으로 추정합니다. URL route integrity도 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | link/deployment policy의 단일 호출 경로를 보장하지만 문맥별 author intent는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `3e2e95a3a28c`가 page policy를, `119ff9a92090` 이후가 placement vocabulary를 추가합니다. |

#### 코드·실행 증거

정적 근거: `ba8da56d3fcf`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 후속 `dc07871c4d24`가 public export surface를, `b77b386b344e`가 selector 결과를 테스트합니다.

### 3. `3e2e95a3a28c` — feat(content): 페이지 활성화 selector 추가

- **Importance:** B
- **Tags:** CONTENT, ROUTING
- **Thread 역할:** page enablement selector
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `isPortfolioPageEnabled` 또는 대응 selector가 `site.pages?.[pageId] !== false`를 사용하는지 확인합니다.
- missing pages map과 missing page key의 결과를 확인합니다.

확인 원칙:

- 먼저 `3e2e95a3a28c^`와 `3e2e95a3a28c`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | route가 page availability를 각자 판단하거나 optional config 누락을 disabled로 오해할 수 있었습니다. |
| 실제 변경 file/symbol/call path | 명시적 `false`만 비활성으로 보고 누락/true는 활성으로 해석하는 selector를 추가합니다. |
| Data/state/resource owner와 lifetime | site config가 policy input을, selector가 fail-open 판단을 소유합니다. |
| Failure·absence·fallback 처리 | 오타 난 page ID는 type/runtime schema가 별도 차단해야 하며, route가 selector를 사용하지 않으면 정책은 적용되지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | optional page config의 backward-compatible 활성화 규칙을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | T9 loader가 disabled page를 가리키는 internal route를 후속 거부합니다. |

#### 코드·실행 증거

정적 근거: `3e2e95a3a28c`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 4. `7c539b142d6d` — feat(content): 프로젝트 지표 selector 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** declarative metric evaluator
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `getProjectMetricValue`와 filter predicate를 확인합니다.
- `projectIds`, `groupIds`, `tags`, `featured`, `deploymentStatuses` 조건이 함께 있을 때 AND로 적용되는지 확인합니다.
- `aggregate: count`와 `sum-highlights`, unknown metric ID의 0 fallback을 확인합니다.

확인 원칙:

- 먼저 `7c539b142d6d^`와 `7c539b142d6d`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 프로젝트 통계가 featured, path prefix, 특정 ID 같은 화면별 heuristic으로 계산될 수 있었습니다. |
| 실제 변경 file/symbol/call path | metric definition을 ID로 찾고 여러 filter 조건을 모두 만족하는 project만 선택한 뒤 count 또는 highlights 합계를 계산합니다. |
| Data/state/resource owner와 lifetime | metric 정의는 content가, 평가 알고리즘은 selector가 소유합니다. |
| Failure·absence·fallback 처리 | unknown metric은 0으로 fallback하므로 구성 누락을 오류로 만들지 않습니다. project/tag/group reference integrity는 loader 전까지 보장되지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 동일 metric ID에 대한 일관된 계산을 보장하지만 정의의 비즈니스 타당성은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `383a3b86e119`이 실제 projects/work-map consumers의 heuristic을 제거합니다. |

#### 코드·실행 증거

정적 근거: `7c539b142d6d`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 5. `daa6815a6dfa` — feat(project): 카드 링크를 콘텐츠 배치 기준으로 선택

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** card placement 최초 소비
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `getProjectCardLinks`의 `placements?.includes("card")` 선행 조건을 확인합니다.
- demo link에만 `isProjectLive`가 추가로 적용되고 다른 type은 placement만 통과하면 반환되는지 확인합니다.

확인 원칙:

- 먼저 `daa6815a6dfa^`와 `daa6815a6dfa`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | card action은 github/case-study type을 암묵적으로 허용해 author가 지정한 문맥을 표현할 수 없었습니다. |
| 실제 변경 file/symbol/call path | card placement가 없는 link를 먼저 제거하고 demo에는 live 조건을 유지하며 나머지 type은 배치 의도를 따릅니다. |
| Data/state/resource owner와 lifetime | content author가 placement를 소유하고 selector가 deployment guard와 함께 해석합니다. |
| Failure·absence·fallback 처리 | placement 누락은 omission이며 오류가 아닙니다. 다른 hero/detail/contact consumers는 아직 이전 정책일 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | project card가 explicit placement를 따르는 것을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `119ff9a92090`이 공용 placement vocabulary/selectors를 정리합니다. |

#### 코드·실행 증거

정적 근거: `daa6815a6dfa`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 6. `383a3b86e119` — feat(content): 프로젝트 지표를 화면에 적용

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** metric selector consumer migration
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/app/projects/page.tsx`와 `work-map-section.tsx`에서 path/ID/featured heuristic이 `getProjectMetricValue`로 교체되는지 확인합니다.
- `project-detail-view.tsx`에 highlights section이 추가되는 별도 표현 변경을 구분합니다.

확인 원칙:

- 먼저 `383a3b86e119^`와 `383a3b86e119`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | projects page와 work map이 screenshot path, 특정 project ID, `featured`를 직접 해석해 metric source와 중복되었습니다. |
| 실제 변경 file/symbol/call path | 두 consumer가 named metric selector를 호출하도록 바꾸고 project detail에 이미 계약화된 highlights section을 표시합니다. |
| Data/state/resource owner와 lifetime | 통계 정책 owner는 selector/content metric으로 이동하고 route/component는 key 선택과 표시만 담당합니다. |
| Failure·absence·fallback 처리 | metric ID 오타는 0으로 보일 수 있으며 이 commit의 highlights 추가는 metric migration과 독립적인 표현 변화입니다. |
| 보장하는 것과 보장하지 않는 것 | 주요 화면이 동일 metric evaluator를 소비함을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `119ff9a92090` 이후 link placement policy도 공용화됩니다. |

#### 코드·실행 증거

정적 근거: `383a3b86e119`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 7. `119ff9a92090` — feat(content): 링크 배치 selector 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** 공용 LinkPlacement vocabulary
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `LinkPlacement` type과 placement 기반 selector들을 확인합니다.
- hero/contact/card/detail/footer별 selection과 demo live guard가 어디에서 공유되는지 확인합니다.

확인 원칙:

- 먼저 `119ff9a92090^`와 `119ff9a92090`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | card만 placement를 사용하고 다른 문맥은 type/수동 filter에 남아 정책이 비대칭이었습니다. |
| 실제 변경 file/symbol/call path | 공용 `LinkPlacement` vocabulary와 context별 selectors를 추가합니다. |
| Data/state/resource owner와 lifetime | selector module이 모든 placement 해석을 소유하고 content link가 author intent를 담습니다. |
| Failure·absence·fallback 처리 | 빈/누락 placements를 오류로 만들지 않으며 schema/loader가 placement enum만 별도 검증합니다. |
| 보장하는 것과 보장하지 않는 것 | 문맥별 link 선택의 단일 정책을 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `2d87b62dcce8`과 `ee2c118a76d6`이 실제 components/renderers를 migration합니다. |

#### 코드·실행 증거

정적 근거: `119ff9a92090`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 8. `2d87b62dcce8` — refactor(project): 상세 링크를 배치 기준으로 선택

- **Importance:** B
- **Tags:** RENDERER, REFACTOR
- **Thread 역할:** detail component consumer migration
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- project detail links component의 local `link.type` filter 제거를 확인합니다.
- 새 detail selector 호출과 렌더링 props가 behavior-preserving인지 비교합니다.

확인 원칙:

- 먼저 `2d87b62dcce8^`와 `2d87b62dcce8`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | project detail component가 허용 link type을 자체 filter하여 placement policy와 중복되었습니다. |
| 실제 변경 file/symbol/call path | local filter를 제거하고 detail placement selector 결과만 렌더링하도록 refactor합니다. |
| Data/state/resource owner와 lifetime | 선택 책임은 selector로, markup/action 표현은 component에 남습니다. |
| Failure·absence·fallback 처리 | DOM/action behavior가 동일하다는 실행 증거는 이 commit 자체에 없고 source placements가 비어 있으면 링크가 사라질 수 있습니다. |
| 보장하는 것과 보장하지 않는 것 | detail link 선택이 공용 policy를 사용함을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `ee2c118a76d6`이 home hero consumers를 같은 policy로 전환합니다. |

#### 코드·실행 증거

정적 근거: `2d87b62dcce8`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 9. `ee2c118a76d6` — feat(content): 홈 링크를 배치 기준으로 선택

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** home hero consumer migration
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Design/Classic home renderer에서 link type 기반 selection이 hero placement selector로 바뀌는지 확인합니다.
- live demo guard와 fallback action이 유지되는지 확인합니다.

확인 원칙:

- 먼저 `ee2c118a76d6^`와 `ee2c118a76d6`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | home hero가 link type을 직접 해석해 detail/card와 다른 규칙을 가질 수 있었습니다. |
| 실제 변경 file/symbol/call path | Design/Classic home action selection을 placement selector로 전환합니다. |
| Data/state/resource owner와 lifetime | selector가 author intent/deployment를 결정하고 renderer는 반환 action을 표현합니다. |
| Failure·absence·fallback 처리 | 다른 design renderer 소비와 visual 결과는 별도 commits/tests가 책임집니다. |
| 보장하는 것과 보장하지 않는 것 | 초기 두 home이 공용 placement policy를 따릅니다. |
| 다음 commit 또는 관련 test 연결 | 후속 design/view-model tests가 모든 renderer와 scoped payload의 일관성을 보호합니다. |

#### 코드·실행 증거

정적 근거: `ee2c118a76d6`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| fallback/omission/selection 정책은 selector가 소유한다. | `eb988f5e09e4` → `ba8da56d3fcf` | `src/lib/portfolio/selectors.ts` | unknown tech는 fallback, missing refs는 omission |
| page는 명시적 false만 비활성이다. | `3e2e95a3a28c` | page enablement selector | disabled route references는 T9에서 오류 |
| metric은 content definition을 AND filter로 평가한다. | `7c539b142d6d` → `383a3b86e119` | `getProjectMetricValue`와 consumers | unknown metric은 0 |
| link 문맥은 explicit placement로 결정한다. | `daa6815a6dfa` → `119ff9a92090` → consumers | placement selectors | placement 누락은 omission |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| renderer마다 lookup/fallback이 달라질 위험 | `eb988f5e09e4` | technology/project/resume selectors | 후속 selector/view-model tests |
| link type이 문맥을 암묵적으로 결정 | `daa6815a6dfa`, `119ff9a92090` | placement-first selectors | `3353032ba23b`의 card/live selector regression |
| 화면이 ID/path heuristic으로 metric 계산 | `383a3b86e119` | 공용 metric evaluator 소비 | `3353032ba23b`의 metric differential test |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| raw source | renderer/component | portfolio aggregate | `PortfolioContent` |
| lookup/fallback | 각 consumer | `selectors.ts` | pure lookup/filter functions |
| metric 계산 | route/component heuristic | `getProjectMetricValue` | content metrics + AND filters |
| link 문맥 | type 추정 | content placements + selectors | consumer는 결과만 렌더링 |

## 9. Thread 최종 상태

Thread 종료 시점에는 technology/project/contact/resume lookup, page enablement, metric 계산, deployment/live link와 placement 정책이 selector로 집중됩니다. Consumer는 domain rule을 재해석하지 않고 selector 결과를 표시하지만, missing configuration의 상당수는 error가 아니라 fallback/omission/0으로 처리됩니다.

### 최종 설명

- 조회·fallback·enabled/deployment 정책을 pure selectors로 모았습니다.
- hard-coded metric heuristics를 declarative metric evaluator로 교체했습니다.
- link type 추정에서 explicit placement vocabulary로 이동하고 실제 components를 migration했습니다.
- fail-open/omission/0 fallback이 의도된 비보장임을 runtime integrity 오류와 구분해야 합니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| aggregate와 context를 입력합니다. | `PortfolioContent`, project/link/page ID | selector input | unknown/missing 값은 각 selector 정책으로 처리 |
| lookup/filter/metric을 평가합니다. | `selectors.ts` | records, booleans, counts, links | unknown metric 0; missing refs omission |
| route/view model이 결과를 조합합니다. | projects/home/detail consumers | scoped values | consumer가 local heuristic을 다시 넣으면 invariant가 깨짐 |
| renderer가 표시합니다. | components/design routes | actions/stats/content | DOM/visual behavior는 별도 test |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b`은 metric differential, featured/resume/card/live selectors를, `b77b386b344e`와 `527b9f872333`은 route별 derived/scoped view models를 후속 테스트합니다. 이 작업에서는 실행하지 않았습니다.
===== END FILE: 04-selectors-links-and-derived-content-policy.md =====

===== BEGIN FILE: 05-runtime-schema-vocabulary.md =====
# Thread: Runtime schema vocabulary

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

TypeScript assertion만 존재하던 domain source에 Zod dependency와 공용 primitive를 도입하고 site, profile, links, projects, technology, experience, journey, contact, résumé, interview map, curation의 runtime shape를 단계적으로 고정하는 과정을 복원합니다.

### 계획된 핵심 invariant

- 공백뿐인 문자열, 허용하지 않은 ID·URL·asset path는 공용 primitive에서 거부합니다.
- 알려진 domain object는 대체로 `strict()`로 예상 밖 key를 거부하되 optional field는 schema에 명시합니다.
- Schema 정의와 실제 source parsing, cross-file reference validation은 서로 다른 책임입니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 정적 TypeScript type이 runtime source 신뢰를 제공하지 못하는 이유는 무엇인가?
- `strict()`, `optional()`, `nullable()`, `min(1)`이 각 source의 허용 범위를 어떻게 바꾸는가?
- 이 Thread가 검증하는 단일 파일 shape와 후속 loader가 검증하는 참조 무결성은 어디서 갈리는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a1977dc7f026` | build(content): runtime 콘텐츠 검증 의존성 추가 | B | CONTENT, VALIDATION, DEPLOY | runtime validation 도구 기반 |
| 2 | `51ceb76ad88a` | feat(content): 콘텐츠 경로와 기본 식별자 schema 추가 | B | CONTENT, VALIDATION | 공용 primitive vocabulary |
| 3 | `c2f3d376e96b` | feat(content): 사이트와 프로필 schema 추가 | B | CONTENT, VALIDATION | site/profile runtime shape |
| 4 | `857fa82a2030` | feat(content): 링크와 배포 상태 schema 추가 | B | CONTENT, VALIDATION | link/deployment/image schema |
| 5 | `f1163dc120bc` | feat(content): 프로젝트 분류와 지표 schema 추가 | B | CONTENT, VALIDATION | project group/metric vocabulary |
| 6 | `a944c73f0557` | feat(content): 프로젝트 사례 schema 추가 | A | CONTENT, VALIDATION | 완전한 project catalog schema |
| 7 | `d93ec9730edd` | feat(content): 기술과 경력 schema 추가 | B | CONTENT, VALIDATION | technology/skills/experience schemas |
| 8 | `6fc79f058744` | feat(content): 여정과 연락 schema 추가 | B | CONTENT, VALIDATION | journey/link-list/contact schemas |
| 9 | `03aacfddc364` | feat(content): Resume 콘텐츠 schema 추가 | B | CONTENT, VALIDATION, RENDERER | résumé schema |
| 10 | `80152dae761f` | feat(content): 여정 narrative schema 추가 | B | CONTENT, VALIDATION | journey narrative schema |
| 11 | `51ce1c15a0e5` | feat(content): Interview Map 콘텐츠 schema 추가 | B | CONTENT, VALIDATION, RENDERER | interview evidence schema |
| 12 | `d0a62a7da4bd` | feat(content): 큐레이션 schema와 타입 export 추가 | A | CONTENT, VALIDATION | curation schema 및 schema-derived source types |

## 5. Commit별 학습 기록

### 1. `a1977dc7f026` — build(content): runtime 콘텐츠 검증 의존성 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, DEPLOY
- **Thread 역할:** runtime validation 도구 기반
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`과 lockfile에서 `zod`와 `tsx` 추가를 확인합니다.
- `zod`가 runtime dependency이고 `tsx`가 development command 실행 도구인지 구분합니다.

확인 원칙:

- 먼저 `a1977dc7f026^`와 `a1977dc7f026`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 콘텐츠 type은 TypeScript에만 있었고 JSON import 결과를 runtime에서 검사할 library나 TypeScript script runner가 없었습니다. |
| 실제 변경 file/symbol/call path | `zod` 4.x와 `tsx`를 dependency graph에 넣어 schema 및 후속 validation command를 구현할 기반을 만듭니다. |
| Data/state/resource owner와 lifetime | package manifest가 validation toolchain의 version range를 소유합니다. |
| Failure·absence·fallback 처리 | Dependency만 추가되었으므로 source가 자동으로 parse되거나 build가 fail-closed 되지는 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 후속 schema/command를 구현할 수 있는 도구 기반만 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `51ceb76ad88a`가 첫 공용 primitive를 추가합니다. |

#### 코드·실행 증거

정적 근거: `a1977dc7f026`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `51ceb76ad88a` — feat(content): 콘텐츠 경로와 기본 식별자 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** 공용 primitive vocabulary
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `nonEmptyString`, `contentId`, href/asset path schema의 구현을 확인합니다.
- trim 후 `min(1)`, ID 정규식, 허용 protocol과 `/template`·`/assets` prefix 범위를 기록합니다.

확인 원칙:

- 먼저 `51ceb76ad88a^`와 `51ceb76ad88a`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 각 source가 임의 문자열을 받아 공백 값, 불안정한 ID, 위험하거나 지원하지 않는 경로를 구분하지 못했습니다. |
| 실제 변경 file/symbol/call path | trimmed non-empty string, 제한된 content ID, 외부 URL·mailto·tel·root asset path의 공용 schema를 정의합니다. |
| Data/state/resource owner와 lifetime | `content-schema.ts`의 primitive가 후속 모든 object schema의 입력 경계를 소유합니다. |
| Failure·absence·fallback 처리 | 문자열 형식만 확인하며 실제 URL endpoint나 asset 파일 존재성은 확인하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 공통 문자열·식별자·경로 형식을 일관되게 거부/수용합니다. |
| 다음 commit 또는 관련 test 연결 | `c2f3d376e96b`부터 domain object가 이 primitive를 조합합니다. |

#### 코드·실행 증거

정적 근거: `51ceb76ad88a`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 3. `c2f3d376e96b` — feat(content): 사이트와 프로필 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** site/profile runtime shape
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `siteContentSchema`, `profileContentSchema`와 하위 navigation/footer/photo/principle schema를 확인합니다.
- `siteContentSchema`의 `passthrough()`와 `pages` 하위 object의 strictness 차이를 기록합니다.

확인 원칙:

- 먼저 `c2f3d376e96b^`와 `c2f3d376e96b`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | site/profile JSON은 수동 type assertion만 통과해 공백 title이나 잘못된 nested shape를 runtime에서 거부하지 못했습니다. |
| 실제 변경 file/symbol/call path | site metadata/navigation/footer와 profile identity/photo/principles를 Zod object로 정의합니다. |
| Data/state/resource owner와 lifetime | schema가 known field의 shape를 소유하며 site 최상위는 확장 key를 허용합니다. |
| Failure·absence·fallback 처리 | 최상위 `passthrough()` 때문에 알려지지 않은 site key를 모두 거부하지 않으며 navigation route 존재성도 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | known site/profile field의 runtime 형식은 고정하지만 파일 간 의미 관계는 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `857fa82a2030`이 link/deployment vocabulary를 추가합니다. |

#### 코드·실행 증거

정적 근거: `c2f3d376e96b`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 4. `857fa82a2030` — feat(content): 링크와 배포 상태 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** link/deployment/image schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `contentLinkSchema`, `deploymentStatusSchema`, `projectImageSchema`를 확인합니다.
- link type, env key, external/enabled/placements optional field와 object `strict()`를 기록합니다.

확인 원칙:

- 먼저 `857fa82a2030^`와 `857fa82a2030`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | link와 deployment record는 임의 string union 및 assertion에만 의존했습니다. |
| 실제 변경 file/symbol/call path | 허용 link type, deployment status, project image, optional environment/placement metadata를 strict object로 정의합니다. |
| Data/state/resource owner와 lifetime | schema가 link record의 vocabulary와 nested object shape를 소유합니다. |
| Failure·absence·fallback 처리 | href가 실제 route나 enabled target을 가리키는지는 확인하지 않고, placement 누락도 허용합니다. |
| 보장하는 것과 보장하지 않는 것 | 개별 link/deployment/image record의 runtime shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `f1163dc120bc`이 project grouping과 metric schema를 추가합니다. |

#### 코드·실행 증거

정적 근거: `857fa82a2030`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 5. `f1163dc120bc` — feat(content): 프로젝트 분류와 지표 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** project group/metric vocabulary
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `projectGroupSchema`, `projectMetricFilterSchema`, `projectMetricSchema`를 확인합니다.
- non-negative order, optional project/group/tag/featured/deployment filters, aggregate enum을 기록합니다.

확인 원칙:

- 먼저 `f1163dc120bc^`와 `f1163dc120bc`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | group과 metric definition이 JSON convention에만 의존해 잘못된 aggregate나 음수 order를 허용할 수 있었습니다. |
| 실제 변경 file/symbol/call path | project group과 metric filter/aggregate를 strict schema로 정의합니다. |
| Data/state/resource owner와 lifetime | metric source가 선언을 소유하고 schema는 허용된 filter key와 aggregate를 제한합니다. |
| Failure·absence·fallback 처리 | 참조하는 group/project/tag가 실제 존재하는지와 여러 filter의 계산 의미는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | metric 정의의 단일 파일 shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `a944c73f0557`이 전체 project catalog schema를 닫습니다. |

#### 코드·실행 증거

정적 근거: `f1163dc120bc`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 6. `a944c73f0557` — feat(content): 프로젝트 사례 schema 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** 완전한 project catalog schema
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `portfolioProjectSourceSchema`의 ID/order/group/tags/deployment/images/stack/links/narrative fields를 전부 확인합니다.
- `projectsContentSchema`의 groups/items `min(1)`, metrics 배열, root `strict()`를 확인합니다.
- nested architecture/deployment objects와 arrays가 허용하는 empty state를 구분합니다.

확인 원칙:

- 먼저 `a944c73f0557^`와 `a944c73f0557`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | group/metric schema는 있었지만 실제 project item과 `{groups, metrics, items}` catalog 전체를 runtime에서 검증할 계약이 없었습니다. |
| 실제 변경 file/symbol/call path | 한 project의 identity, classification, deployment, media, stack, links, problem/solution/architecture/decisions/tradeoffs/results를 strict schema로 정의하고 groups/items가 최소 한 개 있는 catalog를 만듭니다. |
| Data/state/resource owner와 lifetime | `projectsContentSchema`가 project source document의 구조를 소유하며 item 내부 예상 밖 key를 거부합니다. |
| Failure·absence·fallback 처리 | arrays 중 tags/stack/screenshots/highlights 등은 빈 배열이 허용될 수 있고, ID uniqueness·group/stack/link 참조·asset 존재는 아직 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | project catalog가 비어 있지 않고 각 item이 완전한 known shape를 갖는다는 runtime 계약을 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `d93ec9730edd` 이후 나머지 domain file schema가 확장되고 T9가 참조 무결성을 추가합니다. |

#### 코드·실행 증거

정적 근거: `a944c73f0557`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: 단일 project record가 아닌 catalog 전체의 필수 필드와 최소 cardinality를 고정해 이후 loader/facade의 핵심 입력 계약이 됩니다.

### 7. `d93ec9730edd` — feat(content): 기술과 경력 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** technology/skills/experience schemas
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- technology item icon/color, skills group, experience item schema를 확인합니다.
- 각 record의 strictness와 array cardinality를 기록합니다.

확인 원칙:

- 먼저 `d93ec9730edd^`와 `d93ec9730edd`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 기술·skills·experience source가 assertion만 사용했습니다. |
| 실제 변경 file/symbol/call path | technology metadata, grouped skills, experience timeline을 공용 primitive 기반 schema로 정의합니다. |
| Data/state/resource owner와 lifetime | 각 source file의 record shape는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | technology ID의 project stack 참조 여부와 날짜 의미/정렬은 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 세 domain source의 개별 구조를 runtime에서 고정합니다. |
| 다음 commit 또는 관련 test 연결 | `6fc79f058744`가 journey/links/contact를 추가합니다. |

#### 코드·실행 증거

정적 근거: `d93ec9730edd`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 8. `6fc79f058744` — feat(content): 여정과 연락 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** journey/link-list/contact schemas
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- journey item의 nullable `projectId`, links collection, contact preferred ID 배열을 확인합니다.
- nullable과 optional을 혼동하지 않고 기록합니다.

확인 원칙:

- 먼저 `6fc79f058744^`와 `6fc79f058744`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 여정과 연락 source는 project/link ID를 임의 문자열로 보유하고 shape 오류를 거부하지 못했습니다. |
| 실제 변경 file/symbol/call path | journey item, top-level links collection, contact availability/notes/preferred IDs를 schema로 정의합니다. |
| Data/state/resource owner와 lifetime | schema가 null 허용과 field shape를 소유합니다. |
| Failure·absence·fallback 처리 | nullable project ID가 non-null일 때 실제 project를 가리키는지, preferred ID가 enabled link인지 확인하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 각 파일의 runtime 형식을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `03aacfddc364`가 résumé 구조를 추가합니다. |

#### 코드·실행 증거

정적 근거: `6fc79f058744`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 9. `03aacfddc364` — feat(content): Resume 콘텐츠 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** résumé schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- summary, projectIds, education/training/experience/notes, nullable `downloadUrl` schema를 확인합니다.
- asset path nullable 의미와 projectIds 배열의 허용 상태를 기록합니다.

확인 원칙:

- 먼저 `03aacfddc364^`와 `03aacfddc364`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | résumé source는 route가 기대하는 section shape와 download URL 경계를 runtime에서 보장하지 못했습니다. |
| 실제 변경 file/symbol/call path | résumé summary와 참조 project IDs, 여러 이력 section, nullable download asset을 schema로 정의합니다. |
| Data/state/resource owner와 lifetime | résumé document shape는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | `downloadUrl`이 null이면 허용되며 non-null asset 파일 존재와 project reference는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | résumé source의 known field 형식을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `80152dae761f`이 narrative journey schema를 추가합니다. |

#### 코드·실행 증거

정적 근거: `03aacfddc364`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 10. `80152dae761f` — feat(content): 여정 narrative schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** journey narrative schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- intro, milestones, state/reason/result, anchorProjectIds, currentPosition schema를 확인합니다.
- milestone ID uniqueness와 date 의미가 아직 없는지 확인합니다.

확인 원칙:

- 먼저 `80152dae761f^`와 `80152dae761f`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 간단한 journey list와 별도로 설명형 milestone source를 검증할 계약이 없었습니다. |
| 실제 변경 file/symbol/call path | narrative intro, milestone identity/date/state/reason/result/project anchors, current position을 schema로 정의합니다. |
| Data/state/resource owner와 lifetime | journey narrative file의 구조는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | milestone ID uniqueness, chronology, anchor project 존재는 확인하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | narrative 문서의 단일 파일 shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `51ce1c15a0e5`가 interview evidence schema를 추가합니다. |

#### 코드·실행 증거

정적 근거: `80152dae761f`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 11. `51ce1c15a0e5` — feat(content): Interview Map 콘텐츠 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** interview evidence schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- reference repo, tracks/items/answers/gaps schema를 확인합니다.
- answer의 `projectId`와 depth field, track ID가 어떤 primitive를 사용하는지 기록합니다.

확인 원칙:

- 먼저 `51ce1c15a0e5^`와 `51ce1c15a0e5`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Interview Map source가 nested evidence 구조를 runtime에서 검증하지 못했습니다. |
| 실제 변경 file/symbol/call path | reference, tracks, topics, project answers, gaps를 strict nested schema로 정의합니다. |
| Data/state/resource owner와 lifetime | evidence document의 구조는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | answer가 enabled project를 가리키는지와 track ID uniqueness는 아직 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Interview Map 단일 파일의 known shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `d0a62a7da4bd`가 curation과 schema-derived type export를 추가합니다. |

#### 코드·실행 증거

정적 근거: `51ce1c15a0e5`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 12. `d0a62a7da4bd` — feat(content): 큐레이션 schema와 타입 export 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** curation schema 및 schema-derived source types
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- curation intro/criteria/categories/omissions/nextReview schema를 확인합니다.
- `z.infer` 또는 schema output 기반으로 export되는 source types를 확인하고 수동 type 중복이 줄어드는 범위를 기록합니다.

확인 원칙:

- 먼저 `d0a62a7da4bd^`와 `d0a62a7da4bd`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | curation source의 runtime 계약이 없고 schema와 별도 수동 source type이 쉽게 어긋날 수 있었습니다. |
| 실제 변경 file/symbol/call path | curation 문서의 strict nested schema를 추가하고 주요 domain source type을 schema에서 추론해 export합니다. |
| Data/state/resource owner와 lifetime | runtime schema가 source type의 canonical owner가 되기 시작합니다. |
| Failure·absence·fallback 처리 | 모든 renderer-facing type이 즉시 schema-derived 되는 것은 아니며 category/project references와 source parsing은 여전히 후속 책임입니다. |
| 보장하는 것과 보장하지 않는 것 | curation shape와 주요 source schema/type의 동기화를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | T6가 presentation schema를 별도로 확장하고 T8이 실제 JSON parsing을 연결합니다. |

#### 코드·실행 증거

정적 근거: `d0a62a7da4bd`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: schema가 runtime 검증과 정적 source type을 함께 소유하는 방향으로 전환되어 후속 facade의 중복 type 제거 기반이 됩니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| 공용 primitive가 공백 문자열, ID, URL/asset path 형식을 제한한다. | `51ceb76ad88a` | `src/lib/content-schema.ts` primitives | endpoint/asset 존재는 검사하지 않음 |
| domain object는 알려진 shape와 optional/nullability를 schema로 표현한다. | `c2f3d376e96b` → `51ce1c15a0e5` | 개별 source schemas | 일부 최상위 확장 지점은 passthrough |
| project catalog는 groups/items 최소 한 개와 strict project shape를 요구한다. | `a944c73f0557` | `projectsContentSchema` | ID uniqueness/reference는 T9 |
| source type은 schema에서 추론하기 시작한다. | `d0a62a7da4bd` | schema-derived exports | renderer-facing 수동 type 일부는 남음 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| TypeScript assertion이 malformed JSON을 통과시킴 | `51ceb76ad88a` 이후 schema sequence | runtime Zod vocabulary 구축 | `d50870c8b8c4`/`03d2c9be0a43`에서 실제 parse |
| project record가 부분 shape로 들어올 위험 | `a944c73f0557` | strict full project schema + catalog cardinality | 후속 invalid-source tests가 regression 보호 |
| schema와 source type이 어긋날 위험 | `d0a62a7da4bd` | schema-derived source type export | `85df59454b46`에서 facade type 연결 확대 |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| 문자열/ID/path 규칙 | 각 type/consumer | schema primitives | `content-schema.ts` |
| domain file shape | 수동 TypeScript type | Zod schemas | 각 source schema |
| source type | 수동 선언 | 일부 `z.infer` export | schema가 canonical source 계약 |
| cross-file semantics | 없음 | 여전히 없음 | T9 loader integrity가 인수 |

## 9. Thread 최종 상태

Thread 종료 시점에는 모든 주요 domain JSON file의 known shape가 Zod schema로 표현되고 project catalog의 최소 cardinality까지 고정됩니다. 그러나 schema는 아직 source import에 연결되지 않았고, ID uniqueness·cross-file reference·internal route·asset 존재는 보장하지 않습니다.

### 최종 설명

- Zod/tsx 기반을 추가하고 공통 문자열·ID·경로 vocabulary를 만들었습니다.
- site/profile에서 project, résumé, journey, interview map, curation까지 domain file schema를 확장했습니다.
- 전체 project catalog의 strict shape와 최소 항목 수를 고정했습니다.
- Schema 정의와 실제 parsing/integrity 검증을 의도적으로 분리했습니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| 원시 JSON 값을 schema에 입력할 준비를 합니다. | `content-schema.ts` exports | Zod schema objects | 이 Thread에서는 아직 호출되지 않음 |
| 공용 primitive를 먼저 적용합니다. | non-empty/ID/path schema | 정규화되거나 거부되는 scalar | 실제 endpoint/asset은 미검사 |
| nested domain object를 검사합니다. | site/project/etc. schemas | known-shape output | cross-file reference는 미검사 |
| schema-derived type을 export합니다. | `z.infer` source types | 정적 source contract | 모든 facade type 대체는 후속 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b` 이후 invalid schema, duplicate/reference, asset cases가 테스트됩니다. 이 작업에서는 repository test command를 실행하지 않았습니다.
===== END FILE: 05-runtime-schema-vocabulary.md =====

===== BEGIN FILE: 06-runtime-presentation-schema-contracts.md =====
# Thread: Runtime presentation schema contracts

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

다섯 design과 home/projects/detail/About/Contact/Interview Map/Journey/Resume route의 presentation source를 공용 ID vocabulary, strict nested object, 의도된 passthrough 확장점으로 runtime schema에 단계적으로 연결하는 과정을 복원합니다.

### 계획된 핵심 invariant

- Design/route별 알려진 필드는 schema가 검증하고 section order 배열은 비어 있지 않으며 중복될 수 없습니다.
- Nested design contract는 주로 `strict()`이지만 presentation root, home/pages의 선택된 확장 지점은 `passthrough()`를 유지합니다.
- Presentation schema는 copy/ordering 형식을 검증할 뿐 cross-file project/link/route 존재를 검증하지 않습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- section ID·count key·design ID는 어떤 enum/refine 조합으로 제한되는가?
- `strict()`와 `passthrough()`가 공존하는 정확한 계층은 어디인가?
- route별 schema가 추가되어도 default template·supported design completeness는 왜 후속 loader 책임인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `807214624c87` | feat(content): 홈 표현 식별자 schema 추가 | B | CONTENT, VALIDATION | presentation primitive와 section uniqueness |
| 2 | `97ff48de55b8` | feat(content): 프로젝트 목록 표현 schema 추가 | B | CONTENT, VALIDATION | five-design projects page schema |
| 3 | `42a81197af82` | feat(content): 표현 공용 UI schema 추가 | B | CONTENT, VALIDATION | presentation root/UI schema |
| 4 | `a3825a49a055` | feat(content): Design과 Classic 홈 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | initial home design schemas |
| 5 | `2a02781859b8` | feat(content): Editorial 홈 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | Editorial schema |
| 6 | `23d5297b42bd` | feat(content): Brutalist 홈 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | Brutalist schema |
| 7 | `ad07ab4b31a9` | feat(content): Cinematic 홈 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | Cinematic schema |
| 8 | `3c873e373bbb` | feat(content): 공용 홈 섹션 schema 추가 | B | CONTENT, VALIDATION | shared home schema |
| 9 | `edcf1eaa6f71` | feat(content): About과 Contact 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | About/Contact schemas |
| 10 | `4eb0db7b9656` | feat(content): Interview Map 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | Interview Map schema |
| 11 | `7f3c16b50990` | feat(content): Journey 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | Journey schema |
| 12 | `50b78a557344` | feat(content): 프로젝트 상세 표현 schema 추가 | B | CONTENT, VALIDATION | project detail and projects route schema closure |
| 13 | `4e2454cfc9c4` | feat(content): Resume 표현 schema 추가 | B | CONTENT, VALIDATION, RENDERER | Resume presentation schema closure |

## 5. Commit별 학습 기록

### 1. `807214624c87` — feat(content): 홈 표현 식별자 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** presentation primitive와 section uniqueness
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `sectionCopySchema`, home section ID enums, five design IDs, work-map/project count key enums를 확인합니다.
- Editorial/Brutalist/Cinematic section 배열의 `min(1)`과 Set-size `refine`을 확인합니다.

확인 원칙:

- 먼저 `807214624c87^`와 `807214624c87`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | presentation source의 section/order/count key는 임의 문자열 배열이어서 중복·지원하지 않는 값이 들어갈 수 있었습니다. |
| 실제 변경 file/symbol/call path | 공용 section copy와 design별 section ID, site design ID, count key를 enum으로 제한하고 추가 design section order의 비어 있음/중복을 거부합니다. |
| Data/state/resource owner와 lifetime | `content-schema.ts`가 presentation vocabulary와 순서 배열 invariant를 소유합니다. |
| Failure·absence·fallback 처리 | 배열의 업무상 최적 순서나 모든 supported design의 실제 구성 여부는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 허용 ID와 section 배열의 최소·유일성을 runtime에서 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `97ff48de55b8`이 projects route 구조를 추가합니다. |

#### 코드·실행 증거

정적 근거: `807214624c87`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `97ff48de55b8` — feat(content): 프로젝트 목록 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** five-design projects page schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `projectPageContentSchema`의 groups와 Design/Classic/Editorial/Brutalist/Cinematic nested fields를 확인합니다.
- Classic `maxGroups` positive integer, count key enum, nested `strict()`와 root `passthrough()`를 기록합니다.

확인 원칙:

- 먼저 `97ff48de55b8^`와 `97ff48de55b8`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | projects route presentation 값은 type/JSON에 있었지만 runtime shape를 확인하지 못했습니다. |
| 실제 변경 file/symbol/call path | 다섯 design의 hero/stats/terminal/group copy와 group descriptions를 schema로 정의합니다. |
| Data/state/resource owner와 lifetime | route-specific nested object는 schema가 소유하고 outer project page는 확장 key를 허용합니다. |
| Failure·absence·fallback 처리 | `passthrough()` 때문에 unknown outer key를 거부하지 않으며 group category가 실제 project group과 대응하는지 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | known projects presentation field의 runtime 형식을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `42a81197af82`가 presentation root와 공용 UI를 연결합니다. |

#### 코드·실행 증거

정적 근거: `97ff48de55b8`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 3. `42a81197af82` — feat(content): 표현 공용 UI schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** presentation root/UI schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `presentationContentSchema`의 default/template/ui/emptyStates를 확인합니다.
- template/root `passthrough()`와 UI/emptyStates `strict()`를 구분합니다.

확인 원칙:

- 먼저 `42a81197af82^`와 `42a81197af82`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 개별 projects schema는 있었지만 `presentation.json` root와 공용 UI copy를 한 문서로 검사할 schema가 없었습니다. |
| 실제 변경 file/symbol/call path | default design, templates, 공용 UI/ARIA/empty-state 필드를 root schema에 추가합니다. |
| Data/state/resource owner와 lifetime | presentation root가 전체 문서 entry contract를 소유합니다. |
| Failure·absence·fallback 처리 | root와 template는 확장 key를 허용하고 default design이 templates에 실제 포함되는지 아직 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 공용 UI known fields와 design ID 형식을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `a3825a49a055`부터 home design subtree를 채웁니다. |

#### 코드·실행 증거

정적 근거: `42a81197af82`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 4. `a3825a49a055` — feat(content): Design과 Classic 홈 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** initial home design schemas
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Design/Classic shell, hero, sections, featured, terminal command/output schema를 확인합니다.
- nested strictness와 `home` outer passthrough를 확인합니다.

확인 원칙:

- 먼저 `a3825a49a055^`와 `a3825a49a055`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | presentation root는 존재했지만 초기 두 home design subtree가 runtime 검증되지 않았습니다. |
| 실제 변경 file/symbol/call path | Design/Classic의 shell/home copy, stats/count keys, section order, terminal structure를 strict nested schema로 추가합니다. |
| Data/state/resource owner와 lifetime | 각 design contract가 known field shape를 소유합니다. |
| Failure·absence·fallback 처리 | home outer node는 passthrough이고 terminal placeholder token의 의미나 renderer 지원은 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 초기 두 design의 known presentation shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `2a02781859b8`부터 세 추가 design을 확장합니다. |

#### 코드·실행 증거

정적 근거: `a3825a49a055`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 5. `2a02781859b8` — feat(content): Editorial 홈 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** Editorial schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Editorial shell/home hero/lead/featured/principles/contact fields와 section uniqueness schema 연결을 확인합니다.

확인 원칙:

- 먼저 `2a02781859b8^`와 `2a02781859b8`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Editorial JSON contract가 root passthrough 아래 무검증으로 남았습니다. |
| 실제 변경 file/symbol/call path | Editorial 전용 shell 및 home presentation fields를 strict nested schema로 추가합니다. |
| Data/state/resource owner와 lifetime | Editorial known vocabulary는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | 실제 Editorial renderer가 모든 field를 사용하거나 의미 있는 copy인지 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Editorial source shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `23d5297b42bd`가 Brutalist를 추가합니다. |

#### 코드·실행 증거

정적 근거: `2a02781859b8`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 6. `23d5297b42bd` — feat(content): Brutalist 홈 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** Brutalist schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Brutalist shell/home signal/system/journey/contact fields와 section array schema를 확인합니다.

확인 원칙:

- 먼저 `23d5297b42bd^`와 `23d5297b42bd`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Brutalist subtree가 runtime 검증되지 않았습니다. |
| 실제 변경 file/symbol/call path | Brutalist 전용 표현 fields를 strict nested schema로 추가합니다. |
| Data/state/resource owner와 lifetime | Brutalist contract는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | renderer behavior와 visual contract는 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Brutalist source shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `ad07ab4b31a9`가 Cinematic을 추가합니다. |

#### 코드·실행 증거

정적 근거: `23d5297b42bd`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 7. `ad07ab4b31a9` — feat(content): Cinematic 홈 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** Cinematic schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Cinematic shell subtitle, home sections/hero/statement/focus/contact/case-study labels를 확인합니다.
- home outer passthrough 안의 Cinematic node가 strict인지 확인합니다.

확인 원칙:

- 먼저 `ad07ab4b31a9^`와 `ad07ab4b31a9`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Cinematic subtree가 runtime 검증되지 않았습니다. |
| 실제 변경 file/symbol/call path | Cinematic shell/home presentation fields를 strict nested schema로 추가합니다. |
| Data/state/resource owner와 lifetime | Cinematic known contract는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | 알려지지 않은 sibling home key는 outer passthrough로 허용됩니다. |
| 보장하는 것과 보장하지 않는 것 | Cinematic source shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `3c873e373bbb`이 공용 home sections를 추가합니다. |

#### 코드·실행 증거

정적 근거: `ad07ab4b31a9`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 8. `3c873e373bbb` — feat(content): 공용 홈 섹션 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** shared home schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- shared workMap cards의 ID/label/body/countKey와 technicalFocus/stack/journey/contact schema를 확인합니다.
- shared node strictness를 확인합니다.

확인 원칙:

- 먼저 `3c873e373bbb^`와 `3c873e373bbb`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 다섯 design이 공유하는 home copy가 root passthrough 아래 무검증으로 남았습니다. |
| 실제 변경 file/symbol/call path | work map과 공용 home sections를 strict shared object로 정의합니다. |
| Data/state/resource owner와 lifetime | 공유 copy contract는 하나의 schema node가 소유합니다. |
| Failure·absence·fallback 처리 | count key로 계산되는 metric 값과 card ID 의미는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | cross-design shared copy의 known shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `edcf1eaa6f71`부터 auxiliary routes를 추가합니다. |

#### 코드·실행 증거

정적 근거: `3c873e373bbb`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 9. `edcf1eaa6f71` — feat(content): About과 Contact 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** About/Contact schemas
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- About hero/principles/journey/skills/curation/editorial/brutalist와 Contact availability/notes design fields를 확인합니다.
- About/Contact/pages outer `passthrough()`를 기록합니다.

확인 원칙:

- 먼저 `edcf1eaa6f71^`와 `edcf1eaa6f71`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | About과 Contact route copy가 runtime schema에 포함되지 않았습니다. |
| 실제 변경 file/symbol/call path | 두 route의 known presentation fields를 추가합니다. |
| Data/state/resource owner와 lifetime | route subtree의 known fields는 schema가 소유하되 page nodes는 확장 가능성을 위해 passthrough를 유지합니다. |
| Failure·absence·fallback 처리 | unknown page key를 모두 거부하지 않고 실제 page enablement/route 존재를 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | About/Contact known copy 형식을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `4eb0db7b9656`이 Interview Map을 추가합니다. |

#### 코드·실행 증거

정적 근거: `edcf1eaa6f71`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 10. `4eb0db7b9656` — feat(content): Interview Map 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** Interview Map schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- hero, tracks labels/templates, gaps ARIA/eyebrow schema를 확인합니다.
- nested strict와 route passthrough를 구분합니다.

확인 원칙:

- 먼저 `4eb0db7b9656^`와 `4eb0db7b9656`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Interview Map presentation copy가 root passthrough로만 통과했습니다. |
| 실제 변경 file/symbol/call path | Interview Map의 evidence index/empty/gaps vocabulary를 schema에 추가합니다. |
| Data/state/resource owner와 lifetime | known labels/templates는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | template placeholder의 실제 치환과 evidence reference 존재는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Interview Map known presentation shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `7f3c16b50990`이 Journey를 추가합니다. |

#### 코드·실행 증거

정적 근거: `4eb0db7b9656`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 11. `7f3c16b50990` — feat(content): Journey 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** Journey schema
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- hero/narrative labels/timeline/now schema와 nested strictness를 확인합니다.

확인 원칙:

- 먼저 `7f3c16b50990^`와 `7f3c16b50990`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Journey presentation copy가 runtime 검증되지 않았습니다. |
| 실제 변경 file/symbol/call path | Journey narrative state/reason/result labels와 timeline/current-position fields를 schema에 추가합니다. |
| Data/state/resource owner와 lifetime | Journey route copy contract는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | milestone chronology나 anchor project 참조는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Journey known presentation shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `50b78a557344`가 project detail/projects page linkage를 추가합니다. |

#### 코드·실행 증거

정적 근거: `7f3c16b50990`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 12. `50b78a557344` — feat(content): 프로젝트 상세 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** project detail and projects route schema closure
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- projectDetail back/case/missing/facts/outro/frame/editorial/sections schema를 확인합니다.
- `pages.projects: projectPageContentSchema` 연결을 확인합니다.

확인 원칙:

- 먼저 `50b78a557344^`와 `50b78a557344`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | project detail route와 이미 정의한 projects page schema가 `presentationContentSchema.pages`에 완전히 연결되지 않았습니다. |
| 실제 변경 file/symbol/call path | project detail known fields와 section key 집합을 추가하고 projects route schema를 pages node에 연결합니다. |
| Data/state/resource owner와 lifetime | 두 project route contract를 presentation schema가 소유합니다. |
| Failure·absence·fallback 처리 | project ID 존재, 404 status, section content 존재는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | project index/detail known presentation shape를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `4e2454cfc9c4`가 Resume route를 추가합니다. |

#### 코드·실행 증거

정적 근거: `50b78a557344`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 13. `4e2454cfc9c4` — feat(content): Resume 표현 schema 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** Resume presentation schema closure
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Resume hero/summary/projects/training/experience/education/notes/identity/editorial/brutalist fields를 확인합니다.
- route node passthrough와 nested strictness를 기록합니다.

확인 원칙:

- 먼저 `4e2454cfc9c4^`와 `4e2454cfc9c4`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Resume presentation route가 schema coverage에서 빠져 있었습니다. |
| 실제 변경 file/symbol/call path | Resume의 known section/title/identity/design-specific fields를 schema에 추가합니다. |
| Data/state/resource owner와 lifetime | Resume copy contract는 schema가 소유합니다. |
| Failure·absence·fallback 처리 | download asset, project refs, 실제 section rendering은 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 주요 presentation routes와 다섯 design의 known field coverage를 완성합니다. |
| 다음 commit 또는 관련 test 연결 | T8 parser가 이 schema를 실제 `presentation.json`에 적용하고 T9가 design completeness를 검사합니다. |

#### 코드·실행 증거

정적 근거: `4e2454cfc9c4`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| section/design/count key vocabulary는 enum으로 제한한다. | `807214624c87` | presentation primitives | 업무상 순서의 타당성은 미검사 |
| section order는 최소 한 개이며 중복될 수 없다. | `807214624c87` | design-specific arrays + refine | 지원 design completeness는 T9 |
| known nested fields는 주로 strict하게 검사한다. | `97ff48de55b8` → `4e2454cfc9c4` | design/route schemas | 선택된 outer nodes는 passthrough |
| presentation root는 확장 가능성을 유지한다. | `42a81197af82` 이후 | `presentationContentSchema` | unknown top-level/page key를 모두 거부하지 않음 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| presentation JSON이 임의 section/count key를 수용 | `807214624c87` | enum + unique section arrays | 후속 invalid-presentation tests |
| route/design known field가 runtime 검증 밖에 있음 | T6 sequence | 각 subtree schema를 점진 연결 | `03d2c9be0a43`에서 실제 source parse |
| strictness를 전체 문서에 과장할 위험 | 의도된 `passthrough()` 유지 | known nested contract만 strict | unknown extension key 허용을 비보장으로 기록 |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| presentation ID vocabulary | 수동 union | Zod enums/refine | `content-schema.ts` |
| design/route known fields | TypeScript/JSON convention | nested schemas | 각 design/page node |
| extension keys | 암묵적 허용 | 명시적 passthrough 지점 | root/home/pages/선택 route |
| cross-file design/reference integrity | 없음 | 여전히 없음 | T9 loader가 인수 |

## 9. Thread 최종 상태

Thread 종료 시점에는 다섯 design과 주요 route의 known presentation fields, 공용 UI/empty-state vocabulary, section ID·count key·ordering cardinality가 runtime schema로 표현됩니다. 다만 root와 선택된 outer page nodes는 확장 key를 허용하며 default design completeness, route/project reference, renderer behavior는 보장하지 않습니다.

### 최종 설명

- 공용 design/section/count-key vocabulary와 section 유일성 invariant를 정의했습니다.
- 다섯 home design과 shared sections를 strict nested contract로 확장했습니다.
- projects/detail/About/Contact/Interview Map/Journey/Resume route를 schema에 연결했습니다.
- 문서 전체 strictness를 과장하지 않고 의도된 passthrough 지점을 남겼습니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| presentation JSON을 입력합니다. | `presentationContentSchema` | root object | parser 연결 전에는 호출되지 않음 |
| design/section IDs를 검사합니다. | enum/refine schemas | 허용·유일한 IDs | 순서 의미는 미검사 |
| known nested route/design fields를 검사합니다. | strict child schemas | typed parsed subtrees | outer extension keys는 보존 |
| 후속 integrity 검사로 넘깁니다. | T8/T9 loader | parsed presentation source | default/templates completeness는 후속 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b`은 다섯 design과 invalid/unsupported design cases를 후속 테스트합니다. 이 작업에서는 실행하지 않았습니다.
===== END FILE: 06-runtime-presentation-schema-contracts.md =====

===== BEGIN FILE: 07-starter-catalog-migration.md =====
# Thread: Starter catalog migration

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

빈 collection과 generic placeholder를 실제 schema 전체를 행사하는 coherent starter catalog로 바꾸고, project source를 legacy 배열에서 `{groups, metrics, items}` document로 이행하는 과정을 복원합니다.

### 계획된 핵심 invariant

- Migration 중에는 legacy 배열과 새 catalog object를 모두 읽지만 이 compatibility branch는 runtime validation이 아닙니다.
- Starter source의 cross-file IDs는 서로 맞물려야 하지만 이 Thread 시점에는 관례로만 유지됩니다.
- 중간 commit의 일시적으로 불완전한 catalog와 최종 coherent starter 상태를 구분합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 호환 loader가 두 project source shape를 수용하는 방식과 한계는 무엇인가?
- groups/metrics가 먼저 추가되고 items가 뒤따르는 중간 상태는 왜 그대로 기록해야 하는가?
- starter content가 schema coverage를 넓혀도 runtime parse/reference validation을 대신하지 못하는 이유는 무엇인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `d717c35cf80c` | refactor(content): 프로젝트 컬렉션 migration 경계 추가 | B | CONTENT, REFACTOR | legacy/new project source 호환 경계 |
| 2 | `6caa6debdb01` | feat(content): 사이트와 프로필 starter 콘텐츠 구성 | B | CONTENT | identity starter source |
| 3 | `f8ea6376c65c` | feat(content): 링크와 기술 starter 콘텐츠 구성 | B | CONTENT | link/technology starter graph |
| 4 | `247ad421101a` | feat(content): 프로젝트 starter 분류와 지표 구성 | B | CONTENT | catalog envelope와 metric starter |
| 5 | `c58a1be43009` | feat(content): 프로젝트 starter 상세 구성 | B | CONTENT | full starter project case |
| 6 | `83bd41a353ce` | feat(content): Resume와 여정 starter 콘텐츠 구성 | B | CONTENT, RENDERER | chronology/résumé starter references |
| 7 | `7ba62b311776` | feat(content): Interview Map과 큐레이션 starter 콘텐츠 구성 | B | CONTENT, RENDERER | evidence/curation starter graph |

## 5. Commit별 학습 기록

### 1. `d717c35cf80c` — refactor(content): 프로젝트 컬렉션 migration 경계 추가

- **Importance:** B
- **Tags:** CONTENT, REFACTOR
- **Thread 역할:** legacy/new project source 호환 경계
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `ProjectContentFile = PortfolioProject[] | { items: PortfolioProject[] }`와 `Array.isArray` branch를 확인합니다.
- 두 branch 모두 `as unknown as` assertion을 사용하는지와 groups/metrics를 아직 소비하지 않는지 확인합니다.

확인 원칙:

- 먼저 `d717c35cf80c^`와 `d717c35cf80c`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | project source는 legacy 배열이었고 새 catalog object로 바꾸면 `content.ts` import가 즉시 깨질 수 있었습니다. |
| 실제 변경 file/symbol/call path | 배열이면 그대로, object면 `.items`를 사용하도록 임시 compatibility branch를 추가합니다. |
| Data/state/resource owner와 lifetime | `content.ts`가 migration 기간의 shape normalization을 소유합니다. |
| Failure·absence·fallback 처리 | 이 코드는 parse가 아니라 assertion이므로 `{}` 같은 malformed object에서 `.items`가 `undefined`가 될 수 있고 groups/metrics를 보존하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 배열→object 전환 중 기존 project consumers의 최소 호환만 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `247ad421101a`와 `c58a1be43009`에서 새 catalog source가 단계적으로 채워지고 `508e0b71024b`에서 이 임시 branch가 제거됩니다. |

#### 코드·실행 증거

정적 근거: `d717c35cf80c`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `6caa6debdb01` — feat(content): 사이트와 프로필 starter 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** identity starter source
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- site title/brand/navigation/footer/pages/socialImage와 profile identity/photo/principles를 확인합니다.
- contact/experience starter additions가 같은 commit에 포함되는지 확인합니다.

확인 원칙:

- 먼저 `6caa6debdb01^`와 `6caa6debdb01`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | site/profile/contact/experience source가 빈 값 또는 generic placeholder여서 실제 route와 schema field를 충분히 행사하지 못했습니다. |
| 실제 변경 file/symbol/call path | portfolio template에 맞는 identity, navigation, page toggles, profile principles, contact availability, example experience를 채웁니다. |
| Data/state/resource owner와 lifetime | starter defaults는 각 JSON file이 소유합니다. |
| Failure·absence·fallback 처리 | 실제 사용자 정보가 아니며 navigation target/asset 존재는 아직 검증하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | site/profile 기반 route가 사용할 coherent starter vocabulary를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `f8ea6376c65c`가 link/technology ID 집합을 구성합니다. |

#### 코드·실행 증거

정적 근거: `6caa6debdb01`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 3. `f8ea6376c65c` — feat(content): 링크와 기술 starter 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** link/technology starter graph
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `links.json`의 IDs, enabled flag, placements, href를 확인합니다.
- `skills.json`, `tech-stack.json`의 IDs와 project stack에서 사용할 vocabulary를 확인합니다.

확인 원칙:

- 먼저 `f8ea6376c65c^`와 `f8ea6376c65c`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | link/skills/technology collection이 비어 있어 selector·schema·renderer가 실제 ID/placement를 행사하지 못했습니다. |
| 실제 변경 file/symbol/call path | source/repository/contact link와 technology/skill examples를 추가하고 email link는 disabled 상태로 남깁니다. |
| Data/state/resource owner와 lifetime | content author가 link placement와 technology ID vocabulary를 소유합니다. |
| Failure·absence·fallback 처리 | ID uniqueness, enabled preferred link, 실제 URL 응답은 아직 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 후속 project/contact source가 참조할 starter IDs를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `247ad421101a`가 project groups/metrics를 먼저 추가합니다. |

#### 코드·실행 증거

정적 근거: `f8ea6376c65c`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 4. `247ad421101a` — feat(content): 프로젝트 starter 분류와 지표 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** catalog envelope와 metric starter
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `projects.json`이 배열에서 `{groups, metrics, items}` object로 바뀌는 diff를 확인합니다.
- groups/metrics가 실제 값을 갖지만 `items`가 빈 중간 상태인지 확인합니다.

확인 원칙:

- 먼저 `247ad421101a^`와 `247ad421101a`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | project source는 분류/metric vocabulary 없이 legacy item 배열만 표현했습니다. |
| 실제 변경 file/symbol/call path | starter group descriptions와 declarative metrics를 추가하고 새 catalog envelope를 도입합니다. |
| Data/state/resource owner와 lifetime | project catalog document가 grouping/metric definitions를 소유합니다. |
| Failure·absence·fallback 처리 | 이 exact SHA에서는 `items`가 비어 있어 훗날의 `projectsContentSchema.min(1)`을 만족하지 않는 중간 상태입니다. 당시 compatibility loader는 `.items`만 읽습니다. |
| 보장하는 것과 보장하지 않는 것 | 새 document shape와 분류/metric vocabulary를 도입하지만 usable starter catalog 완성은 보장하지 않습니다. |
| 다음 commit 또는 관련 test 연결 | `c58a1be43009`가 실제 project item을 추가해 중간 공백을 닫습니다. |

#### 코드·실행 증거

정적 근거: `247ad421101a`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 5. `c58a1be43009` — feat(content): 프로젝트 starter 상세 구성

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** full starter project case
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- example project의 groupId/tags/deployment/screenshots/stack/links/highlights/problem/solution/architecture/decisions/tradeoffs/results를 확인합니다.
- 이 IDs가 앞선 groups/tech/link vocabulary와 맞물리는지 정적으로 추적합니다.

확인 원칙:

- 먼저 `c58a1be43009^`와 `c58a1be43009`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 새 catalog envelope에는 project item이 없어 route와 selector가 실제 상세 사례를 렌더링할 수 없었습니다. |
| 실제 변경 file/symbol/call path | schema 전체를 행사하는 example project item을 추가합니다. |
| Data/state/resource owner와 lifetime | project source가 starter case study의 완전한 narrative와 references를 소유합니다. |
| Failure·absence·fallback 처리 | 참조가 코드상 맞아 보여도 loader integrity가 아직 실행되지 않아 typo를 fail-closed 하지 않습니다. asset 파일 존재도 별도입니다. |
| 보장하는 것과 보장하지 않는 것 | starter catalog가 최소 한 개의 완전한 project를 갖습니다. |
| 다음 commit 또는 관련 test 연결 | `83bd41a353ce`가 résumé/journey에서 이 project를 참조합니다. |

#### 코드·실행 증거

정적 근거: `c58a1be43009`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 6. `83bd41a353ce` — feat(content): Resume와 여정 starter 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** chronology/résumé starter references
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `journey.json`, `journey-narrative.json`, `resume.json`의 project IDs와 narrative fields를 확인합니다.
- nullable journey item과 non-null project references를 구분합니다.

확인 원칙:

- 먼저 `83bd41a353ce^`와 `83bd41a353ce`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | starter project가 résumé와 chronology에 연결되지 않아 cross-route narrative가 비어 있었습니다. |
| 실제 변경 file/symbol/call path | journey items, narrative milestones/current position, résumé summary/training/education/notes/projectIds를 채웁니다. |
| Data/state/resource owner와 lifetime | 각 document가 자신의 narrative와 project references를 소유합니다. |
| Failure·absence·fallback 처리 | project ID 존재, chronology order, resume asset 존재는 아직 runtime에서 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | starter project가 résumé와 journey route에서 재사용될 source graph를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `7ba62b311776`가 interview evidence와 curation references를 완성합니다. |

#### 코드·실행 증거

정적 근거: `83bd41a353ce`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 7. `7ba62b311776` — feat(content): Interview Map과 큐레이션 starter 콘텐츠 구성

- **Importance:** B
- **Tags:** CONTENT, RENDERER
- **Thread 역할:** evidence/curation starter graph
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Interview Map tracks/items/answers와 curation categories/projectIds를 확인합니다.
- reference repo, gaps, omissions, nextReview 값을 확인합니다.

확인 원칙:

- 먼저 `7ba62b311776^`와 `7ba62b311776`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Interview Map과 curation source가 비어 있어 project evidence의 선택 이유와 부족한 영역을 표현하지 못했습니다. |
| 실제 변경 file/symbol/call path | starter project를 evidence answer와 curation category에 연결하고 gaps/criteria/omissions를 채웁니다. |
| Data/state/resource owner와 lifetime | evidence/curation documents가 project selection narrative를 소유합니다. |
| Failure·absence·fallback 처리 | answer/category project ID가 enabled project인지와 track/category ID uniqueness는 아직 loader가 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 주요 domain source가 하나의 coherent starter project graph를 공유합니다. |
| 다음 commit 또는 관련 test 연결 | T8/T9가 이 starter graph를 실제 parse하고 reference integrity로 검증합니다. |

#### 코드·실행 증거

정적 근거: `7ba62b311776`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| legacy 배열과 새 object를 migration 중 함께 읽는다. | `d717c35cf80c` | `content.ts` compatibility branch | assertion 기반이고 groups/metrics 손실 |
| 새 project catalog는 groups/metrics/items를 분리한다. | `247ad421101a` → `c58a1be43009` | `projects.json` | 첫 commit은 items가 빈 중간 상태 |
| starter IDs는 여러 source가 같은 project graph를 참조한다. | `f8ea6376c65c` → `7ba62b311776` | content JSON files | runtime integrity는 T9 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| source shape 전환이 기존 import를 깨뜨림 | `d717c35cf80c` | dual-shape compatibility branch | `508e0b71024b` validated facade에서 제거 |
| catalog envelope만 있고 item이 없음 | `c58a1be43009` | full starter project 추가 | 후속 schema parse가 min(1) 보호 |
| cross-file starter references에 typo가 생길 위험 | T9 loader sequence | enabled ID sets와 missing-reference issues | `3353032ba23b` invalid source tests |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| project source normalization | legacy array assumption | `content.ts` dual branch | 임시 migration owner |
| catalog grouping/metrics | 없음 | `projects.json` root | content author |
| starter narrative graph | 분산 placeholder | 각 content JSON | shared IDs로 연결 |
| runtime trust | 없음 | 여전히 없음 | T8/T9가 인수 |

## 9. Thread 최종 상태

Thread 종료 시점에는 project source가 `{groups, metrics, items}` document로 전환되고 하나의 full starter case가 technology, résumé, journey, interview map, curation, contact/link source와 coherent ID graph를 이룹니다. 그러나 migration branch는 assertion 기반이고 runtime parse/reference/asset validation은 아직 없습니다.

### 최종 설명

- 배열→catalog object 전환을 임시 dual-shape reader로 분리했습니다.
- groups/metrics를 먼저 추가한 불완전 중간 SHA와 item을 추가한 완료 SHA를 구분했습니다.
- starter source가 대부분의 domain schema field와 cross-route reference를 실제 값으로 행사하도록 확장했습니다.
- coherent-looking fixture와 fail-closed validation을 동일시하지 않았습니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| legacy/new project JSON을 import합니다. | `content.ts` compatibility branch | items array | malformed object는 assertion으로 통과 가능 |
| starter identity/link/technology vocabulary를 읽습니다. | content JSON files | shared IDs/values | uniqueness 미검사 |
| catalog group/metric/item을 구성합니다. | `projects.json` | starter project document | 중간 SHA에는 items가 비어 있음 |
| résumé/journey/evidence/curation에서 참조합니다. | 각 source projectIds | cross-route narrative graph | T9 이전에는 dangling ref 미거부 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. 후속 `03d2c9be0a43`와 T9 integrity commits가 이 starter source를 fail-closed boundary에 연결합니다. 이 작업에서는 source 실행 검증을 수행하지 않았습니다.
===== END FILE: 07-starter-catalog-migration.md =====

===== BEGIN FILE: 08-source-aware-schema-parsing-boundary.md =====
# Thread: Source-aware schema parsing boundary

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

14개 JSON source를 단순 assertion에서 파일명·JSON path가 포함된 structured validation error와 단일 `loadPortfolioSource()` parsing 경계로 전환하는 과정을 복원합니다.

### 계획된 핵심 invariant

- 각 file은 대응 schema로 `safeParse`되고 성공한 `parsed.data`만 반환됩니다.
- Validation failure는 source filename, JSON-style path, message를 보존한 `PortfolioContentError`로 노출됩니다.
- 기본 `portfolioSource`는 module import 시 load되어 malformed source를 consumer 전에 fail-closed 합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Zod issue path를 사용자 편집 가능한 JSON path로 바꾸는 규칙은 무엇인가?
- Schema parse failure와 cross-file integrity issue는 같은 error model을 어떻게 공유하는가?
- `overrides`가 테스트 가능성을 만들지만 production source ownership을 흐리지 않는 이유는 무엇인가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `70e49ea34194` | feat(content): 콘텐츠 validation 오류 모델 추가 | A | CONTENT, VALIDATION | loader module과 structured error model |
| 2 | `830f02688d63` | feat(content): JSON 경로 진단 추가 | B | CONTENT, VALIDATION | deterministic JSON path formatter |
| 3 | `d50870c8b8c4` | feat(content): JSON schema 파싱 경계 추가 | A | CONTENT, VALIDATION | generic per-file parser |
| 4 | `03d2c9be0a43` | feat(content): 콘텐츠 파일 schema 파싱 연결 | S | ARCH, CONTENT, VALIDATION | 14-source fail-closed loader |

## 5. Commit별 학습 기록

### 1. `70e49ea34194` — feat(content): 콘텐츠 validation 오류 모델 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** loader module과 structured error model
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/content-loader.ts`가 import하는 14 JSON/schema 목록을 확인합니다.
- `ContentValidationIssue`, `PortfolioSourceOverrides`, `PortfolioContentError`의 fields/message assembly를 확인합니다.
- supported design IDs와 internal route map이 아직 사용되기 전 scaffold 상태를 기록합니다.

확인 원칙:

- 먼저 `70e49ea34194^`와 `70e49ea34194`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Schema는 정의되어 있었지만 source imports와 연결되지 않았고 오류가 어느 JSON file/path에서 발생했는지 표현할 공통 모델이 없었습니다. |
| 실제 변경 file/symbol/call path | 모든 source/schema imports와 override key 집합을 한 module에 모으고 `{file, path, message}` issue와 aggregated error class를 추가합니다. |
| Data/state/resource owner와 lifetime | `content-loader.ts`가 validation boundary 및 diagnostic model의 owner가 됩니다. |
| Failure·absence·fallback 처리 | 아직 parse function이나 `loadPortfolioSource()`가 없어 source가 실제로 거부되지는 않습니다. supported design/internal route 상수도 후속 integrity를 위한 준비 상태입니다. |
| 보장하는 것과 보장하지 않는 것 | 후속 schema/integrity failure가 공유할 source-aware error contract를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `830f02688d63`이 Zod path를 JSON-style path로 변환합니다. |

#### 코드·실행 증거

정적 근거: `70e49ea34194`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: 이후 모든 validation 경로와 테스트가 의존하는 public failure type과 source override seam을 정의합니다.

### 2. `830f02688d63` — feat(content): JSON 경로 진단 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** deterministic JSON path formatter
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `jsonPath(PropertyKey[])`의 empty path `$`, number `[index]`, identifier `.key`, unusual key bracket+JSON.stringify branch를 확인합니다.
- 실제 source file path는 이 helper가 아니라 caller가 붙인다는 점을 기록합니다.

확인 원칙:

- 먼저 `830f02688d63^`와 `830f02688d63`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Zod path segment 배열을 그대로 노출하면 편집자가 어느 JSON 위치를 고쳐야 하는지 일관되게 읽기 어렵습니다. |
| 실제 변경 file/symbol/call path | root, array index, 일반 property, 특수 property를 JSON-style 문자열로 변환하는 helper를 추가합니다. |
| Data/state/resource owner와 lifetime | diagnostic path formatting은 loader helper가 소유합니다. |
| Failure·absence·fallback 처리 | path가 source line number를 제공하지 않고 JSON parser syntax error를 다루는 것도 아닙니다. |
| 보장하는 것과 보장하지 않는 것 | Zod issue path의 결정적이고 읽을 수 있는 표현을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `d50870c8b8c4`가 이 formatter를 parse failure mapping에 사용합니다. |

#### 코드·실행 증거

정적 근거: `830f02688d63`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 3. `d50870c8b8c4` — feat(content): JSON schema 파싱 경계 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** generic per-file parser
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `parseContentFile<Schema extends z.ZodType>`의 `safeParse`, failure mapping, `parsed.data` 반환을 확인합니다.
- throw가 첫 issue만이 아니라 해당 file의 모든 Zod issues를 보존하는지 확인합니다.

확인 원칙:

- 먼저 `d50870c8b8c4^`와 `d50870c8b8c4`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Error model과 path formatter는 있었지만 각 schema call이 반복되거나 assertion으로 우회될 수 있었습니다. |
| 실제 변경 file/symbol/call path | file label, schema, unknown input을 받는 generic parser를 만들고 Zod issues를 source-aware issues로 변환해 `PortfolioContentError`를 던집니다. |
| Data/state/resource owner와 lifetime | per-file shape validation과 output typing은 `parseContentFile`이 소유합니다. |
| Failure·absence·fallback 처리 | 한 file parse가 실패하면 다음 file이나 cross-file integrity 단계로 진행하지 않으며 JSON text syntax는 bundler import 단계의 책임입니다. |
| 보장하는 것과 보장하지 않는 것 | 성공한 schema output만 caller에 반환하고 실패 시 file/path/message를 보존합니다. |
| 다음 commit 또는 관련 test 연결 | 여러 integrity helper commits 뒤 `03d2c9be0a43`가 모든 source에 parser를 연결합니다. |

#### 코드·실행 증거

정적 근거: `d50870c8b8c4`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: assertion과 runtime trust 사이의 실제 변환 함수이며 downstream loader invariant의 핵심입니다.

### 4. `03d2c9be0a43` — feat(content): 콘텐츠 파일 schema 파싱 연결

- **Importance:** S
- **Tags:** ARCH, CONTENT, VALIDATION
- **Thread 역할:** 14-source fail-closed loader
- **조사 깊이:** Architecture 전환, 이전 trust/ownership 모델, failure path, lifecycle, downstream regression까지 깊게 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `loadPortfolioSource(overrides = {})`가 default imports와 overrides를 merge하는 순서를 확인합니다.
- site/profile/projects/presentation/skills/techStack/experience/journey/links/contact/resume/journeyNarrative/interviewMap/curation 각각의 exact file label/schema call을 추적합니다.
- 반환 object, `portfolioSource = loadPortfolioSource()`, `PortfolioSource = ReturnType<...>`를 확인합니다.
- 이 SHA에 tests가 추가되지 않았음을 구분합니다.

확인 원칙:

- 먼저 `03d2c9be0a43^`와 `03d2c9be0a43`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 개별 parser는 있었지만 production imports는 여전히 직접 assertion path를 사용할 수 있었고 14개 source를 빠짐없이 검사하는 단일 entry가 없었습니다. |
| 실제 변경 file/symbol/call path | default JSON inputs에 caller overrides를 덮어쓴 뒤 각 source를 exact schema와 source path로 `parseContentFile`에 통과시키고 parsed outputs만 aggregate합니다. module-level `portfolioSource`가 import 시 기본 source를 즉시 검증합니다. |
| Data/state/resource owner와 lifetime | `loadPortfolioSource`가 raw source→validated source의 유일한 loader boundary를 소유하고 overrides는 특정 source만 deterministic하게 교체할 test seam이 됩니다. parsed values의 lifetime은 returned source object와 module singleton이 소유합니다. |
| Failure·absence·fallback 처리 | Shape parse가 성공해도 duplicate ID, dangling reference, supported design completeness, internal route, asset 존재는 아직 모두 보장되지 않습니다. 한 file의 shape failure는 즉시 throw하므로 다른 files의 shape issue를 함께 누적하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 모든 14 source가 consumer에 도달하기 전에 대응 runtime schema를 통과하고 malformed field는 source-aware error로 fail-closed 된다는 핵심 invariant를 도입합니다. |
| 다음 commit 또는 관련 test 연결 | T9가 parsed outputs 사이의 repository-wide integrity를 누적 검사하고 T10이 기존 portfolio facade를 이 validated singleton으로 교체합니다. |

#### 코드·실행 증거

정적 근거: `03d2c9be0a43`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 S 근거: source 신뢰 모델이 “JSON import + `as`”에서 “unknown input + exact schema parse + structured failure + validated aggregate”로 바뀌며 이후 content architecture 전체의 입력 경계가 됩니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| 모든 validation issue는 file/path/message를 갖는다. | `70e49ea34194` | `ContentValidationIssue`, `PortfolioContentError` | line/column 정보는 없음 |
| Zod path는 결정적 JSON-style 경로로 변환된다. | `830f02688d63` | `jsonPath` | source syntax error는 별도 |
| schema output만 loader를 통과한다. | `d50870c8b8c4` → `03d2c9be0a43` | `parseContentFile`, `loadPortfolioSource` | cross-file integrity는 T9 |
| 기본 source는 import 시 fail-closed 된다. | `03d2c9be0a43` | `portfolioSource = loadPortfolioSource()` | 테스트 실행 증거는 후속 category 07 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| JSON `as` assertion이 malformed source를 신뢰 | `d50870c8b8c4`, `03d2c9be0a43` | unknown input을 schema `safeParse` | `3353032ba23b` invalid-source regression |
| 오류가 어느 file/path인지 알기 어려움 | `70e49ea34194`, `830f02688d63` | structured issue + JSON path | 후속 tests가 error issue shape를 확인 |
| production source만 고정돼 failure case 주입이 어려움 | `PortfolioSourceOverrides` + loader merge | single-file deterministic overrides | 후속 tests가 malformed/duplicate/route cases 주입 |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| raw JSON imports | 여러 aggregate module | `content-loader.ts` | 14 default inputs |
| per-file parse | 없음 | `parseContentFile` | schema output 또는 structured throw |
| source aggregate | asserted `PortfolioContent` | `PortfolioSource` | validated raw-domain aggregate |
| failure ownership | import/consumer별 | `PortfolioContentError` | loader가 source context를 보존 |

## 9. Thread 최종 상태

Thread 종료 시점에는 14개 JSON source가 각각 exact Zod schema를 통과한 `PortfolioSource`로만 반환되고, 기본 source는 module import 시 fail-closed 됩니다. 오류는 source file과 JSON-style path를 보존합니다. 그러나 duplicate/reference/route/design/asset integrity는 후속 Thread 책임입니다.

### 최종 설명

- 공통 issue/error 모델과 JSON path formatter를 만들었습니다.
- Generic per-file parser가 unknown input을 schema output으로 전환하거나 structured error를 던집니다.
- `loadPortfolioSource()`가 14개 source와 override seam을 한 경계에 모았습니다.
- S-level 전환은 source 신뢰 모델의 변경이며 단순 helper 추가가 아닙니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| Default JSON과 overrides를 합칩니다. | `loadPortfolioSource` input object | 14 unknown inputs | override는 지정 key만 대체 |
| 각 file을 exact schema로 parse합니다. | `parseContentFile` | typed parsed output | 한 file 실패 시 즉시 `PortfolioContentError` |
| parsed outputs를 aggregate합니다. | `loadPortfolioSource` return | `PortfolioSource` | cross-file issues는 아직 검사하지 않음 |
| 기본 singleton을 초기화합니다. | `portfolioSource` | module-lifetime validated source | malformed source면 import 실패 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b`이 override seam으로 malformed source와 structured issue를 후속 검증합니다. 이 작업에서는 Vitest나 content command를 실행하지 않았습니다.
===== END FILE: 08-source-aware-schema-parsing-boundary.md =====

===== BEGIN FILE: 09-repository-wide-content-integrity.md =====
# Thread: Repository-wide content integrity

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

개별 JSON file이 schema를 통과한 뒤에도 남는 duplicate ID, unsupported/disabled internal route, design completeness, project/group/technology/link/project-reference 문제를 하나의 누적 integrity phase에서 fail-closed 하는 과정을 복원합니다.

### 계획된 핵심 invariant

- Shape parsing 뒤 cross-file issues를 별도 배열에 누적하고 가능한 한 여러 문제를 한 번에 보고합니다.
- 참조 검증은 enabled target 집합을 사용하므로 존재하지만 disabled인 project/link도 유효한 공개 참조가 아닙니다.
- 외부 URL은 internal route validator의 책임 밖이며 root-relative internal route만 route map과 page enablement에 대조합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Schema-valid 문자열과 repository-valid reference 사이에는 어떤 차이가 있는가?
- Duplicate/reference issues는 어느 시점에 누적되고 언제 하나의 error로 던져지는가?
- Internal route와 enabled project/page policy가 어떻게 결합되는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f8a4aa2109e8` | feat(content): 중복과 참조 진단 helper 추가 | B | CONTENT, VALIDATION | integrity issue primitives |
| 2 | `b380f56f5d90` | feat(content): 내부 route 참조 검증 추가 | A | ARCH, CONTENT, VALIDATION | internal route policy helper |
| 3 | `b9d74d8ccf08` | feat(content): 콘텐츠 식별자 중복 검증 추가 | A | CONTENT, VALIDATION | global identifier/order uniqueness |
| 4 | `b87da7ca505c` | feat(content): 지원 디자인 구성 검증 추가 | A | CONTENT, VALIDATION | design registry completeness |
| 5 | `6b9e10289b64` | feat(content): 사이트와 링크 route 참조 검증 추가 | A | ARCH, CONTENT, VALIDATION | navigation/top-level link route integration |
| 6 | `08b4ac81739f` | feat(content): 프로젝트 내부 참조 검증 추가 | A | CONTENT, VALIDATION | project group/stack/link integrity |
| 7 | `6514b4e0bcff` | feat(content): 지표와 Resume 참조 검증 추가 | B | CONTENT, VALIDATION, RENDERER | metric/resume project references |
| 8 | `805072d7b610` | feat(content): 여정과 Interview 참조 검증 추가 | B | CONTENT, VALIDATION, RENDERER | journey/evidence project references |
| 9 | `d5afc69ae9da` | feat(content): 큐레이션과 연락 참조 검증 추가 | B | CONTENT, VALIDATION | curation/contact reference closure |

## 5. Commit별 학습 기록

### 1. `f8a4aa2109e8` — feat(content): 중복과 참조 진단 helper 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** integrity issue primitives
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `findDuplicates`, `addDuplicateIssues`, `addMissingReferenceIssue`를 확인합니다.
- Set을 이용해 duplicate value를 한 번만 issue로 추가하는지와 caller가 file/path/label을 공급하는지 확인합니다.

확인 원칙:

- 먼저 `f8a4aa2109e8^`와 `f8a4aa2109e8`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Schema는 record shape만 확인하므로 중복 ID와 존재하지 않는 reference를 표현할 공통 integrity helper가 없었습니다. |
| 실제 변경 file/symbol/call path | 중복 값을 찾는 Set 기반 helper와 duplicate/missing-reference issue accumulator를 추가합니다. |
| Data/state/resource owner와 lifetime | loader의 shared `issues` 배열이 integrity diagnostic lifetime을 소유하고 helper는 throw하지 않고 issue를 추가합니다. |
| Failure·absence·fallback 처리 | 아직 실제 source collection에 호출되지 않으며 duplicate occurrence의 각 index가 아니라 collection-level path를 기록하는 경우가 있습니다. |
| 보장하는 것과 보장하지 않는 것 | 후속 integrity rules가 같은 source-aware issue 형식을 재사용할 수 있게 합니다. |
| 다음 commit 또는 관련 test 연결 | `b380f56f5d90`이 internal route 판단 helper를 추가합니다. |

#### 코드·실행 증거

정적 근거: `f8a4aa2109e8`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 2. `b380f56f5d90` — feat(content): 내부 route 참조 검증 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, VALIDATION
- **Thread 역할:** internal route policy helper
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `addInternalRouteIssue`의 root-relative/`//` early return, `new URL` pathname extraction, `/` special case를 확인합니다.
- `/projects/:id` match, internal page map, disabled page, unknown/disabled project issue branches를 추적합니다.
- query/hash가 pathname 검사에서 어떻게 처리되는지 확인합니다.

확인 원칙:

- 먼저 `b380f56f5d90^`와 `b380f56f5d90`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Link schema가 href 문자열 형식은 확인해도 portfolio 내부 route가 실제 지원·활성 page/project를 가리키는지는 알 수 없었습니다. |
| 실제 변경 file/symbol/call path | root-relative internal href만 route policy 대상으로 삼고 pathname을 canonicalize한 뒤 home, known pages, project detail route를 site config와 enabled project ID set에 대조하는 helper를 추가합니다. |
| Data/state/resource owner와 lifetime | loader가 internal routing integrity를 소유하고 caller는 source file/path 및 route kind를 전달합니다. |
| Failure·absence·fallback 처리 | 외부 URL과 protocol-relative URL은 이 helper가 검사하지 않습니다. 동적 route는 `/projects/<single-segment>`만 지원하며 route handler 실제 존재를 filesystem에서 탐색하지 않고 고정 map에 의존합니다. |
| 보장하는 것과 보장하지 않는 것 | 지원하지 않는 내부 route, disabled page, unknown/disabled project detail link를 일관된 issue로 만들 수 있습니다. |
| 다음 commit 또는 관련 test 연결 | `6b9e10289b64`와 `08b4ac81739f`가 site/link/project consumers에 helper를 적용합니다. |

#### 코드·실행 증거

정적 근거: `b380f56f5d90`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: content authoring과 routing/page enablement를 연결하는 architecture-level policy이며 여러 source의 href가 이 한 helper를 공유합니다.

### 3. `b9d74d8ccf08` — feat(content): 콘텐츠 식별자 중복 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** global identifier/order uniqueness
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `loadPortfolioSource`의 `issues` 배열과 group/enabled project/stack/tag/enabled link ID Sets를 확인합니다.
- group ID/order, metric ID, project ID/order, technology ID, link ID, milestone ID, track ID, category ID, template ID, navigation href duplicate checks를 모두 열거합니다.
- issue가 끝에서 한 번 throw되는 위치를 확인합니다.

확인 원칙:

- 먼저 `b9d74d8ccf08^`와 `b9d74d8ccf08`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 각 file의 ID/order가 schema 형식은 만족해도 duplicate이면 lookup Map, ordering, route/view-model 결과가 비결정적이거나 덮어써질 수 있었습니다. |
| 실제 변경 file/symbol/call path | parsed source에서 reference Sets를 만들고 repository 전반의 주요 identifiers/order/navigation href 중복을 한 `issues` 배열에 누적한 뒤 validation phase 끝에서 `PortfolioContentError`로 던집니다. |
| Data/state/resource owner와 lifetime | `loadPortfolioSource`가 repository-wide uniqueness와 enabled reference universe를 소유합니다. |
| Failure·absence·fallback 처리 | 현재 duplicate message는 동일 값당 한 issue이며 모든 array item index를 지목하지 않을 수 있습니다. Tag vocabulary 자체는 projects에서 관찰된 enabled tags 집합으로 파생됩니다. |
| 보장하는 것과 보장하지 않는 것 | 주요 lookup/order keys의 uniqueness와 여러 issue의 batch reporting을 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `b87da7ca505c`가 design registry completeness를 같은 phase에 추가합니다. |

#### 코드·실행 증거

정적 근거: `b9d74d8ccf08`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: source별 shape 검증을 repository-wide namespace invariant로 확장하고 모든 후속 reference checks의 기준 Sets를 만듭니다.

### 4. `b87da7ca505c` — feat(content): 지원 디자인 구성 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** design registry completeness
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- defaultHomeTemplate가 templates에 있는지, 모든 configured ID가 supported인지, supported five IDs가 모두 configured인지 세 방향 검사를 확인합니다.
- issue file/path가 `presentation.json`의 default/templates 위치를 가리키는지 확인합니다.

확인 원칙:

- 먼저 `b87da7ca505c^`와 `b87da7ca505c`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Presentation schema는 design ID enum을 제한하지만 default가 templates에 포함되는지, 지원하는 다섯 design이 빠짐없이 등록됐는지 전체 문서 관계를 보장하지 못했습니다. |
| 실제 변경 file/symbol/call path | default membership, configured⊆supported, supported⊆configured를 각각 issue로 검사합니다. |
| Data/state/resource owner와 lifetime | loader가 renderer-supported design registry와 content configuration의 완전성을 소유합니다. |
| Failure·absence·fallback 처리 | Template ID uniqueness는 앞 commit이, 실제 renderer module 존재·lazy import 성공은 다른 architecture/tests가 책임집니다. |
| 보장하는 것과 보장하지 않는 것 | default design과 templates registry가 지원 design 집합과 정확히 대응하도록 fail-closed 합니다. |
| 다음 commit 또는 관련 test 연결 | `6b9e10289b64`가 실제 href collections를 route policy에 연결합니다. |

#### 코드·실행 증거

정적 근거: `b87da7ca505c`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: 다섯 renderer의 content registry가 부분 구성이나 unsupported entry로 배포되는 것을 source boundary에서 차단합니다.

### 5. `6b9e10289b64` — feat(content): 사이트와 링크 route 참조 검증 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, VALIDATION
- **Thread 역할:** navigation/top-level link route integration
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `site.navigation.forEach`와 `links.forEach`가 `addInternalRouteIssue`에 전달하는 file/path/routeKind/site/enabledProjectIds를 확인합니다.
- external href가 helper early return으로 통과하는지 확인합니다.

확인 원칙:

- 먼저 `6b9e10289b64^`와 `6b9e10289b64`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Internal route helper는 존재했지만 실제 navigation과 top-level links에 적용되지 않아 dangling route가 loader를 통과했습니다. |
| 실제 변경 file/symbol/call path | site navigation과 top-level link href를 source index별 path로 route validator에 연결합니다. |
| Data/state/resource owner와 lifetime | loader가 두 public navigation source의 internal route consistency를 소유합니다. |
| Failure·absence·fallback 처리 | 외부 destination의 접근 가능성·보안 헤더는 검사하지 않고, page route map은 코드 상수에 의존합니다. |
| 보장하는 것과 보장하지 않는 것 | navigation/link가 unsupported route, disabled page, unknown/disabled project를 가리키면 source load가 실패합니다. |
| 다음 commit 또는 관련 test 연결 | `08b4ac81739f`가 project record 내부 links와 group/stack refs를 추가합니다. |

#### 코드·실행 증거

정적 근거: `6b9e10289b64`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: site shell과 global links의 공개 routing surface를 content integrity boundary에 연결합니다.

### 6. `08b4ac81739f` — feat(content): 프로젝트 내부 참조 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** project group/stack/link integrity
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- 각 project의 groupId missing reference, duplicate tags/stack refs, stack ID existence, project internal links route validation을 확인합니다.
- enabled 여부와 무관하게 items 전체를 검사하는지, target project set은 enabled만인지 구분합니다.

확인 원칙:

- 먼저 `08b4ac81739f^`와 `08b4ac81739f`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 완전한 project schema가 있어도 존재하지 않는 group/technology를 참조하거나 같은 tag/stack을 중복하고 dangling internal link를 가질 수 있었습니다. |
| 실제 변경 file/symbol/call path | 각 project item을 순회해 group/stack references와 per-project duplicates를 검사하고 nested links를 route validator에 연결합니다. |
| Data/state/resource owner와 lifetime | project catalog 내부 관계는 loader integrity phase가 소유합니다. |
| Failure·absence·fallback 처리 | Project 자체가 disabled이어도 source record의 refs를 검사하는 반면 다른 source가 참조할 수 있는 target set은 enabled projects만입니다. External project links의 원격 상태는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | project group/technology/internal-route graph의 기본 integrity를 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `6514b4e0bcff` 이후 project IDs를 사용하는 다른 documents를 검증합니다. |

#### 코드·실행 증거

정적 근거: `08b4ac81739f`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: 가장 큰 domain record의 internal graph와 public links를 한 번에 fail-closed 합니다.

### 7. `6514b4e0bcff` — feat(content): 지표와 Resume 참조 검증 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** metric/resume project references
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- metrics의 projectIds/groupIds/tags filter refs와 résumé projectIds 검사를 확인합니다.
- project/tag target이 enabled project universe에서 파생되는지 기록합니다.

확인 원칙:

- 먼저 `6514b4e0bcff^`와 `6514b4e0bcff`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Metric filter나 résumé가 schema-valid하지만 존재하지 않거나 disabled인 project/group/tag를 가리키면 selector가 0/omission으로 조용히 실패할 수 있었습니다. |
| 실제 변경 file/symbol/call path | Metric filter references와 résumé project list를 known enabled project/group/tag Sets에 대조합니다. |
| Data/state/resource owner와 lifetime | loader가 declarative metric 및 résumé evidence references의 validity를 소유합니다. |
| Failure·absence·fallback 처리 | Metric 계산의 논리/숫자 타당성과 résumé의 원하는 순서/중복은 별도 규칙이 없는 범위에서 보장하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 이 source들이 selector fallback에 의존하기 전에 dangling refs를 fail-closed 합니다. |
| 다음 commit 또는 관련 test 연결 | `805072d7b610`이 journey/interview references를 추가합니다. |

#### 코드·실행 증거

정적 근거: `6514b4e0bcff`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 8. `805072d7b610` — feat(content): 여정과 Interview 참조 검증 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, RENDERER
- **Thread 역할:** journey/evidence project references
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- journey nullable projectId, narrative milestone anchorProjectIds, interview answers projectId loops와 exact JSON paths를 확인합니다.

확인 원칙:

- 먼저 `805072d7b610^`와 `805072d7b610`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Journey와 Interview Map에서 잘못된 project reference가 route view model에서 omission/undefined로 나타날 수 있었습니다. |
| 실제 변경 file/symbol/call path | non-null journey project, milestone anchors, nested interview answers를 enabled project set에 대조합니다. |
| Data/state/resource owner와 lifetime | loader가 chronology/evidence documents의 project references를 소유합니다. |
| Failure·absence·fallback 처리 | Chronology date ordering, duplicate anchors/answers, evidence depth의 의미는 검사하지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Journey와 Interview Map이 공개 가능한 enabled project만 참조하도록 보장합니다. |
| 다음 commit 또는 관련 test 연결 | `d5afc69ae9da`가 curation/contact references로 integrity 범위를 닫습니다. |

#### 코드·실행 증거

정적 근거: `805072d7b610`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 9. `d5afc69ae9da` — feat(content): 큐레이션과 연락 참조 검증 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** curation/contact reference closure
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- curation categories projectIds와 contact preferred linkIds 검사를 확인합니다.
- contact target set이 ID가 있고 `enabled !== false`인 links만 포함하는지 확인합니다.

확인 원칙:

- 먼저 `d5afc69ae9da^`와 `d5afc69ae9da`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Curation과 contact preferred 목록이 dangling/disabled project·link를 가리켜 selector가 조용히 생략할 수 있었습니다. |
| 실제 변경 file/symbol/call path | Curation project references를 enabled project set에, preferred contact IDs를 enabled link ID set에 대조합니다. |
| Data/state/resource owner와 lifetime | loader가 마지막 cross-file public references의 validity를 소유합니다. |
| Failure·absence·fallback 처리 | Category/project ID 중복은 앞 단계가 다루지만 per-category projectIds 중복이나 contact order의 업무 의미는 별도 검사가 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 주요 content documents의 cross-file public reference graph가 load 전에 fail-closed 됩니다. |
| 다음 commit 또는 관련 test 연결 | T10 validated facade가 이 integrity-checked `portfolioSource`만 소비합니다. |

#### 코드·실행 증거

정적 근거: `d5afc69ae9da`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| Integrity issues는 shape parse 뒤 하나의 배열에 누적된다. | `b9d74d8ccf08` 이후 | `loadPortfolioSource` issues | per-file schema failure는 그 이전에 즉시 throw |
| 주요 identifiers/order/navigation href는 unique하다. | `b9d74d8ccf08` | duplicate helpers + repository collections | 모든 possible array 중복을 포괄하지는 않음 |
| Design registry는 default/list/supported 집합이 일치한다. | `b87da7ca505c` | presentation integrity rules | renderer module 존재는 별도 |
| Internal route는 enabled page/project만 가리킨다. | `b380f56f5d90` → `6b9e10289b64`/`08b4ac81739f` | route helper and consumers | 외부 URL reachability 미검사 |
| Cross-file public references는 enabled target universe에 존재한다. | `08b4ac81739f` → `d5afc69ae9da` | Sets + missing-reference issues | 일부 per-array duplicates/semantic order 미검사 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| schema-valid ID가 duplicate/dangling일 수 있음 | `b9d74d8ccf08`과 reference sequence | repository-wide Sets와 batch issues | `3353032ba23b` duplicate/missing-reference tests |
| internal href가 unsupported/disabled target을 가리킴 | `b380f56f5d90`, `6b9e10289b64`, `08b4ac81739f` | route policy + source loops | 후속 unsupported/disabled route tests |
| presentation registry가 partial/unsupported일 수 있음 | `b87da7ca505c` | bidirectional supported design completeness | 후속 missing/unsupported design tests |
| selector omission/0이 authoring error를 숨김 | reference checks before facade | dangling refs를 load error로 승격 | selector fallback은 runtime query semantics로만 남음 |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| per-file shape | schema/parser | T8 parser | 즉시 failure |
| repository namespaces | 없음 | T9 loader issues/Sets | duplicate IDs/order/hrefs |
| routing integrity | renderer/route fallback | `addInternalRouteIssue` | enabled page/project policy |
| cross-file references | selector omission/undefined | loader missing-reference rules | enabled target sets |
| batch failure | 첫 consumer failure | `PortfolioContentError(issues)` | 가능한 integrity issues를 함께 보고 |

## 9. Thread 최종 상태

Thread 종료 시점에는 schema-parsed source가 duplicate identifiers/order, supported design registry, internal routes, project group/stack/link, metric/résumé/journey/interview/curation/contact references를 통과해야만 반환됩니다. 가능한 integrity 문제는 한 배열에 누적되지만 per-file schema parse failure는 앞 단계에서 즉시 중단됩니다. 외부 URL, asset 존재, 모든 semantic ordering/duplicate case는 범위 밖입니다.

### 최종 설명

- Shape-valid와 repository-valid를 분리하고 두 번째 integrity phase를 추가했습니다.
- 중복/참조 helper를 공유해 source file과 JSON path를 보존한 batch diagnostic을 만들었습니다.
- Internal route를 page enablement와 enabled project universe에 연결했습니다.
- Selector의 의도된 fallback/omission이 authoring error를 숨기지 않도록 주요 cross-file references를 loader에서 차단했습니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| Parsed source에서 namespace Sets를 구성합니다. | `loadPortfolioSource` | group/project/stack/tag/link ID sets | enabled target과 all records를 구분 |
| Duplicate/design registry를 검사합니다. | duplicate helpers + supported design sets | issues 배열 | 한 값당 collection-level issue 가능 |
| Internal routes를 검사합니다. | `addInternalRouteIssue` | navigation/link route issues | 외부 URL은 건너뜀 |
| Cross-file references를 검사합니다. | missing-reference helper loops | project/group/stack/link issues | enabled target만 유효 |
| Issues가 있으면 한 번에 실패합니다. | `PortfolioContentError(issues)` | structured batch error | 없으면 parsed source 반환 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b`은 duplicate, missing design, unsupported/disabled route, disabled reference 등의 deterministic override cases를 후속 테스트합니다. 이 작업에서는 해당 tests를 실행하지 않았습니다.
===== END FILE: 09-repository-wide-content-integrity.md =====

===== BEGIN FILE: 10-validated-facade-assets-and-build-gate.md =====
# Thread: Validated facade, assets, and build gate

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`  
> Category: `01-application-foundation-and-content-systems`

## 0. 분류 출처와 변경 가능 범위

- Commit SHA, subject, importance, tags는 target branch의 `commit/commit-importance.md` 분류와 exact commit metadata를 사용합니다.
- 이 문서의 Thread grouping, 목표, 역할, 조사 지점은 Phase 1 category audit에서 repository evidence를 기준으로 확정했습니다.
- Phase 2에서는 이 fixed information을 바꾸지 않고 learner-facing 기록만 채웠습니다.
- 다른 branch나 final HEAD 구현을 과거 SHA 설명에 소급하지 않습니다.

## 1. Thread 목표

기존 JSON-direct portfolio facade를 validated `portfolioSource`로 교체하고 schema-derived type 연결을 확대한 뒤, repository-local asset 존재 검증과 `content:check`/`prebuild` gate로 source trust를 build lifecycle까지 확장하는 과정을 복원합니다.

### 계획된 핵심 invariant

- Portfolio facade는 raw JSON을 직접 import하지 않고 validated source만 소비합니다.
- Facade는 group label 파생, journey 정렬, enabled filtering처럼 renderer-facing transformation만 소유합니다.
- Content build gate는 schema/integrity와 public asset 존재를 모두 통과해야 성공합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Schema-derived source types와 기존 renderer-facing types의 경계는 어떻게 줄어드는가?
- Validated source로 전환하면서 environment href behavior와 migration branch는 어떻게 제거되는가?
- Asset path traversal/absence 검사와 build lifecycle 연결이 무엇을 보장하고 무엇을 보장하지 않는가?

## 3. 완료 기준

- 각 SHA의 parent diff와 resulting tree에서 실제 file/symbol을 확인합니다.
- 이전 상태, implementation decision, owner/lifetime, absence/failure/fallback, guarantee/non-guarantee를 분리합니다.
- Fix·refactor·integration은 바로 앞의 assumption이나 duplicated responsibility와 연결합니다.
- 테스트나 command는 실제 실행 여부를 정적 검토와 명확히 구분합니다.
- Thread 종료 시 invariant evolution과 최종 flow를 코드 없이 설명합니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `85df59454b46` | refactor(content): schema 기반 핵심 콘텐츠 타입 연결 | A | ARCH, CONTENT, VALIDATION | schema-derived facade type bridge |
| 2 | `16bdf03ce979` | feat(content): 여정과 큐레이션 콘텐츠 타입 추가 | B | CONTENT | new narrative facade types |
| 3 | `508e0b71024b` | refactor(content): 검증된 콘텐츠를 portfolio facade에 연결 | S | ARCH, CONTENT, VALIDATION | raw imports 제거와 validated facade cutover |
| 4 | `ff2ecadf3489` | feat(content): 저장소 자산 참조 경계 검증 | A | CONTENT, VALIDATION | public asset filesystem integrity |
| 5 | `0e0ed9e50323` | build(content): 콘텐츠 검사 명령 추가 | B | CONTENT, DEPLOY | explicit content validation command |
| 6 | `28b0db56190f` | build(content): 콘텐츠 검사를 prebuild에 연결 | A | CONTENT, DEPLOY | build fail-closed gate |

## 5. Commit별 학습 기록

### 1. `85df59454b46` — refactor(content): schema 기반 핵심 콘텐츠 타입 연결

- **Importance:** A
- **Tags:** ARCH, CONTENT, VALIDATION
- **Thread 역할:** schema-derived facade type bridge
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/types.ts`에서 `PresentationContentSource`, project group/metric/source types import/export를 확인합니다.
- Home/ProjectPage/Detail/About/Journey/InterviewMap/Resume/Contact/Presentation types가 indexed schema source types로 바뀌는 범위를 확인합니다.
- 여전히 수동으로 남는 Site/Profile/ContentLink/PortfolioContent types를 기록합니다.

확인 원칙:

- 먼저 `85df59454b46^`와 `85df59454b46`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Schema가 source shape와 inferred types를 제공해도 portfolio facade는 다수의 수동 presentation/project type을 별도로 유지해 drift 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | 핵심 project source types를 schema module에서 재export하고 presentation route types를 `PresentationContentSource[...]` indexed types로 연결합니다. 다섯 `SiteDesignId`와 새 project fields도 facade type에 반영합니다. |
| Data/state/resource owner와 lifetime | Runtime schema가 source/presentation type의 canonical owner가 되고 facade types는 consumer용 aliases를 제공합니다. |
| Failure·absence·fallback 처리 | 모든 type이 schema-derived 되는 것은 아니며 일부 manual type과 assertion이 남습니다. Runtime behavior도 이 refactor 하나로 바뀌지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 주요 schema/presentation source와 facade 정적 계약의 drift를 줄입니다. |
| 다음 commit 또는 관련 test 연결 | `16bdf03ce979`가 새 narrative domain types를 채우고 `508e0b71024b`가 production facade source를 교체합니다. |

#### 코드·실행 증거

정적 근거: `85df59454b46`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: schema module과 public portfolio type surface 사이의 ownership을 재정렬하는 architecture-level refactor입니다.

### 2. `16bdf03ce979` — feat(content): 여정과 큐레이션 콘텐츠 타입 추가

- **Importance:** B
- **Tags:** CONTENT
- **Thread 역할:** new narrative facade types
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- JourneyMilestone/Narrative, InterviewMap reference/answer/item/track/content, Curation category/criteria/omission/content types를 확인합니다.
- 이 types가 아직 schema inferred aliases가 아니라 manual declarations인지 기록합니다.

확인 원칙:

- 먼저 `16bdf03ce979^`와 `16bdf03ce979`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Validated source에 journey narrative, interview map, curation이 있어도 portfolio aggregate가 노출할 renderer-facing types가 없었습니다. |
| 실제 변경 file/symbol/call path | 세 문서의 consumer-facing nested types를 `types.ts`에 추가합니다. |
| Data/state/resource owner와 lifetime | Portfolio facade type module이 renderer-facing aliases를 소유합니다. |
| Failure·absence·fallback 처리 | 수동 declarations이므로 schema와 완전히 자동 동기화되지는 않으며 production aggregate 연결은 다음 commit까지 없습니다. |
| 보장하는 것과 보장하지 않는 것 | 새 routes/view models가 사용할 정적 contracts를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `508e0b71024b`가 실제 aggregate fields와 validated source를 연결합니다. |

#### 코드·실행 증거

정적 근거: `16bdf03ce979`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 3. `508e0b71024b` — refactor(content): 검증된 콘텐츠를 portfolio facade에 연결

- **Importance:** S
- **Tags:** ARCH, CONTENT, VALIDATION
- **Thread 역할:** raw imports 제거와 validated facade cutover
- **조사 깊이:** Architecture 전환, 이전 trust/ownership 모델, failure path, lifecycle, downstream regression까지 깊게 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio/content.ts`에서 11개 direct JSON imports와 legacy project union이 제거되고 `portfolioSource` import 하나로 바뀌는지 확인합니다.
- projectGroups sort, groupId→category label mapping, projectMetrics, presentation project groups derivation, journey sort를 추적합니다.
- `withEnvHref`, `PortfolioEnv`/`EnvKey` 제거와 `_legacyEnvironment` 무시를 확인합니다.
- `PortfolioContent`에 groups/metrics/journeyNarrative/interviewMap/curation이 추가되는지 확인합니다.
- public exports와 call sites가 raw source를 우회하지 않는지 resulting tree에서 확인합니다.

확인 원칙:

- 먼저 `508e0b71024b^`와 `508e0b71024b`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 기존 facade는 JSON을 직접 import해 `as` assertion과 임시 dual-shape branch를 사용했으므로 T8/T9 loader가 있어도 production consumer가 이를 우회했습니다. Environment href override도 unvalidated runtime mutation을 추가했습니다. |
| 실제 변경 file/symbol/call path | 모든 raw imports를 `portfolioSource`로 교체하고 validated projects/groups/metrics/presentation/narrative source를 renderer aggregate로 변환합니다. Group order를 copy-sort하고 group label을 project category로 파생하며 project groups를 presentation page에 투영합니다. Disabled project/link filtering은 유지하되 environment href mutation은 제거하고 legacy parameter는 무시합니다. |
| Data/state/resource owner와 lifetime | `content-loader.ts`가 source trust와 integrity를 소유하고 `portfolio/content.ts`는 validated data의 renderer-facing transformation/selection만 소유합니다. Module singleton source는 공유되고 `getPortfolioContent()`는 project/link arrays를 새로 구성하는 경계를 유지합니다. |
| Failure·absence·fallback 처리 | 여러 `as` casts가 consumer aliases 때문에 일부 남고 asset 존재는 아직 검사하지 않습니다. `_legacyEnvironment` 인자는 호환을 위해 존재하지만 behavior는 없습니다. Return object의 모든 nested value를 deep clone하지도 않습니다. |
| 보장하는 것과 보장하지 않는 것 | Production portfolio facade가 raw JSON 경로를 우회하지 않고 schema+integrity-validated source만 소비한다는 핵심 invariant를 확립합니다. |
| 다음 commit 또는 관련 test 연결 | `ff2ecadf3489`가 schema로 확인할 수 없는 public asset filesystem boundary를 추가하고 category 07 tests가 public export/clone/view-model behavior를 고정합니다. |

#### 코드·실행 증거

정적 근거: `508e0b71024b`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 S 근거: 실제 consumer architecture의 trust path를 raw assertions에서 validated loader로 교체하고 migration/env mutation까지 제거하는 ownership cutover입니다.

### 4. `ff2ecadf3489` — feat(content): 저장소 자산 참조 경계 검증

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Thread 역할:** public asset filesystem integrity
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/content-assets.ts`의 `collectAssetReferences`가 site socialImage, profile photo, résumé download, primary/additional project screenshots를 수집하는지 확인합니다.
- `validatePortfolioAssets`의 `resolve(publicRoot, "." + assetPath)`, `relative`, `startsWith("..")`, `isAbsolute`, `existsSync` branches를 확인합니다.
- issues가 source file/path를 보존하고 content object를 그대로 반환하는지 확인합니다.
- 추가된 portrait placeholder SVG가 어떤 starter reference를 만족하는지 확인합니다.

확인 원칙:

- 먼저 `ff2ecadf3489^`와 `ff2ecadf3489`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Schema는 asset path prefix/형식만 검사하므로 `public/` 밖으로 탈출하거나 repository에 없는 파일을 참조해도 source parse가 성공할 수 있었습니다. |
| 실제 변경 file/symbol/call path | 모든 repository-local asset references를 수집해 `publicRoot` 아래 absolute path로 resolve하고 traversal/absolute escape/absence를 하나의 structured issue 배열로 검사합니다. 성공 시 같은 validated source object를 반환합니다. |
| Data/state/resource owner와 lifetime | Filesystem-aware asset integrity는 `content-assets.ts`가 소유하고 schema/loader는 순수 data integrity를 유지합니다. |
| Failure·absence·fallback 처리 | 파일 내용·MIME·image decode·case sensitivity across platforms·remote URL reachability는 검사하지 않습니다. Symlink escape에 대한 명시적 `realpath` 검사는 보이지 않습니다. |
| 보장하는 것과 보장하지 않는 것 | 수집된 asset references가 지정 public root 아래 존재한다는 build-time invariant를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `0e0ed9e50323`이 loader+asset validation을 한 command에 연결합니다. |

#### 코드·실행 증거

정적 근거: `ff2ecadf3489`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: data schema로 표현할 수 없는 repository filesystem 상태를 content validation contract에 통합합니다.

### 5. `0e0ed9e50323` — build(content): 콘텐츠 검사 명령 추가

- **Importance:** B
- **Tags:** CONTENT, DEPLOY
- **Thread 역할:** explicit content validation command
- **조사 깊이:** 이 commit이 맡은 실제 구현 역할, changed symbol, state/absence 처리와 다음 연결을 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `content:check` script와 `scripts/validate-content.ts`를 확인합니다.
- script가 `loadPortfolioSource()` 뒤 `validatePortfolioAssets(..., resolve(process.cwd(), "public"))`를 호출하고 project/design count를 출력하는지 확인합니다.

확인 원칙:

- 먼저 `0e0ed9e50323^`와 `0e0ed9e50323`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Loader와 asset validator는 code path로 존재했지만 개발자/CI가 독립적으로 실행할 stable command가 없었습니다. |
| 실제 변경 file/symbol/call path | `node --import tsx scripts/validate-content.ts` command를 추가해 schema/integrity와 public assets를 연속 실행하고 성공 summary를 출력합니다. |
| Data/state/resource owner와 lifetime | package script가 manual/automation entry를, validation modules가 실제 rules를 소유합니다. |
| Failure·absence·fallback 처리 | Command가 아직 build lifecycle에 자동 연결되지 않았고 출력 count는 correctness proof가 아니라 성공 summary입니다. |
| 보장하는 것과 보장하지 않는 것 | 명시적으로 실행 가능한 content validation command를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | `28b0db56190f`가 이를 `prebuild`에 연결합니다. |

#### 코드·실행 증거

정적 근거: `0e0ed9e50323`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다.

### 6. `28b0db56190f` — build(content): 콘텐츠 검사를 prebuild에 연결

- **Importance:** A
- **Tags:** CONTENT, DEPLOY
- **Thread 역할:** build fail-closed gate
- **조사 깊이:** 주요 subsystem의 결정 경로, owner, failure/non-guarantee와 integration evidence를 구체적으로 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `prebuild: npm run content:check`와 npm lifecycle 순서를 확인합니다.
- `npm run build`가 content check failure 시 Next build 전에 중단되는 결과를 command semantics로 설명하되 실행하지 않은 결과를 주장하지 않습니다.

확인 원칙:

- 먼저 `28b0db56190f^`와 `28b0db56190f`의 first-parent diff를 비교합니다. Root commit이면 parent 부재를 명시합니다.
- Resulting tree의 file/symbol만 이 SHA의 사실로 사용합니다.
- 실행하지 않은 command 결과와 후속 test evidence를 직접 실행한 결과처럼 쓰지 않습니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Content validation command는 선택적이어서 개발자나 release process가 건너뛴 채 `next build`를 수행할 수 있었습니다. |
| 실제 변경 file/symbol/call path | npm `prebuild` lifecycle에 `content:check`를 연결해 standard build path에서 schema, cross-file integrity, asset 존재를 먼저 검사합니다. |
| Data/state/resource owner와 lifetime | `package.json` lifecycle이 build gate ordering을 소유합니다. |
| Failure·absence·fallback 처리 | `next build` 외의 직접 framework invocation이나 `--ignore-scripts` 같은 우회는 차단하지 않으며 이 작업에서는 command를 실제 실행하지 않았습니다. |
| 보장하는 것과 보장하지 않는 것 | 정상적인 `npm run build`가 content validation success 없이는 build phase에 진입하지 않는 구조를 제공합니다. |
| 다음 commit 또는 관련 test 연결 | Category 08이 production toolchain/server verification을, category 07이 regression tests를 후속 보호합니다. |

#### 코드·실행 증거

정적 근거: `28b0db56190f`의 parent diff와 resulting tree에서 위 file/symbol을 확인했습니다. 실행 근거: 없음. 로컬 환경에서 GitHub 도메인 DNS가 차단되어 target branch checkout과 repository command 실행을 수행하지 못했고, GitHub commit/file 조회로만 검토했습니다. 코드 발췌 판단: 별도 code block은 넣지 않았습니다. 함수·field·분기 관계를 위 기록에 최소 단위로 직접 명시했습니다. 중요도 A 근거: validation을 선택적 도구에서 standard release build의 선행 조건으로 승격합니다.

## 6. Invariant evolution ledger

| 추적할 invariant | 도입·변화 SHA | 실제 owner/evidence | 제한·후속 보호 |
| --- | --- | --- | --- |
| 주요 facade source/presentation types는 schema와 연결된다. | `85df59454b46` | `portfolio/types.ts` schema aliases | 일부 manual types는 남음 |
| Production facade는 raw JSON을 직접 import하지 않는다. | `508e0b71024b` | `portfolioSource` single input | asset check는 별도 command |
| Facade는 validation이 아니라 renderer transformation만 소유한다. | `508e0b71024b` | group sort/label, presentation groups, journey sort, enabled filter | deep clone은 아님 |
| Repository-local assets는 public root 아래 존재해야 한다. | `ff2ecadf3489` | `validatePortfolioAssets` | content/MIME/symlink semantics 미검사 |
| Standard build 전에 content gate를 통과한다. | `0e0ed9e50323` → `28b0db56190f` | `content:check`, `prebuild` | 직접/ignore-scripts 우회 가능 |

## 7. Failure → Fix → Test 관계

| Failure 또는 risk | Fix/전환 SHA | 교정된 결정 | Regression·검증 관계 |
| --- | --- | --- | --- |
| Validated loader가 있어도 facade가 raw JSON을 우회 | `508e0b71024b` | direct imports/assertions 제거, `portfolioSource` cutover | `3353032ba23b`/`dc07871c4d24` regression |
| Legacy array/env href mutation이 trust boundary 밖에 남음 | `508e0b71024b` | migration branch/`withEnvHref` 제거 | public API compatibility는 ignored parameter로 유지 |
| Schema-valid asset path가 repository에 없음/탈출 | `ff2ecadf3489` | public-root resolve/relative/exists checks | 후속 missing-asset tests |
| Validation command를 release가 누락 | `28b0db56190f` | `prebuild` gate | category 08 CI/container/release verification |

## 8. Ownership·state·responsibility 변화

| 대상 | 이전 owner/state | 최종 owner/state | 근거 |
| --- | --- | --- | --- |
| source trust | raw imports + assertions | `content-loader.ts`/`portfolioSource` | schema+integrity parsed singleton |
| renderer aggregate | source import와 validation 혼합 | `portfolio/content.ts` | validated data transformation only |
| asset integrity | 없음 | `content-assets.ts` | filesystem-aware public asset checks |
| manual validation entry | 없음 | `content:check` | package script |
| build ordering | Next build directly | `prebuild` → `content:check` → `build` | npm lifecycle |

## 9. Thread 최종 상태

Thread 종료 시점에는 portfolio facade가 validated `portfolioSource`만 소비하고 renderer-facing group/category/presentation/journey/enablement 변환을 수행합니다. Repository-local assets는 별도 validator로 public root 안의 존재를 검사하며 standard `npm run build`는 `content:check`를 선행합니다. 다만 deep clone, remote URL/MIME/image decoding, symlink realpath, 모든 build 우회, 실제 runtime command 성공은 보장하지 않습니다.

### 최종 설명

- Schema-derived types를 public facade type surface에 연결해 중복 계약을 줄였습니다.
- S-level cutover에서 raw JSON imports, assertion-based project migration, env href mutation을 production facade에서 제거했습니다.
- Data integrity와 filesystem asset integrity를 분리한 뒤 하나의 command로 조합했습니다.
- 선택적 content check를 standard npm build gate로 승격했습니다.

## 10. 최종 실행·데이터 흐름

| 단계 | Owner/call path | 입력·출력 | Failure/non-guarantee |
| --- | --- | --- | --- |
| Module import 시 validated source를 획득합니다. | `portfolioSource` | schema+integrity checked raw-domain object | malformed source면 import 실패 |
| Facade transformation을 수행합니다. | `portfolio/content.ts` | sorted groups/journey, derived category/presentation groups, enabled items | deep clone 아님; env override 무시 |
| Content command에서 asset을 검사합니다. | `validate-content.ts` → `validatePortfolioAssets` | same source or structured error | remote/MIME/decode 미검사 |
| Standard build lifecycle을 시작합니다. | `npm run build` | `prebuild`가 `content:check` 선행 | 실제 command는 이번 작업에서 미실행 |
| Next build로 진행합니다. | `build: next build` | production bundle attempt | category 08의 toolchain/server verification 범위 |

## 11. 학습 완료 확인

완료했습니다. 모든 commit은 exact SHA의 parent diff/resulting tree를 기준으로 기록했고, direct execution evidence와 static inspection을 구분했습니다. `3353032ba23b`은 source/assets/model validation, `dc07871c4d24`는 public export와 clone boundary, `b77b386b344e`/`527b9f872333`은 route view models/scoped payload를 후속 테스트합니다. 이번 환경에서는 repository checkout이 없어 테스트·build를 실행하지 않았습니다.
===== END FILE: 10-validated-facade-assets-and-build-gate.md =====

===== BEGIN FILE: README.md =====
# 01-application-foundation-and-content-systems

> Repository: `https://github.com/seungwoo7050/42-archive`  
> Branch: `web/portfolio`

이 category는 실행 가능한 Next application boundary와 portfolio content system의 source → schema → parser → integrity → validated facade → build gate 이력을 복원합니다.

## Category boundary

포함 범위:

- Next/App Router의 최초 실행 경계와 content aggregate를 소비하는 첫 route
- Domain JSON/type/aggregate, presentation contract, selector policy
- Runtime domain/presentation schema, starter catalog migration
- Source-aware parsing, cross-file integrity, validated facade, local asset validation과 content build gate

제외 범위:

- Node/npm pinning, container/production server smoke, release delivery는 category 08에서 다룹니다.
- Content·selector·view-model·asset regression test 구현과 실행은 category 07에서 다룹니다.
- Visual system, responsive behavior, route renderer 구현 자체는 해당 UI/visual category에서 다룹니다.

## Phase 1 audit 결과

- 기존 6개 draft Thread의 범위는 핵심 trust transition을 누락했습니다. Domain schema와 presentation schema를 분리하고 parsing, repository-wide integrity, validated facade/build gate Thread를 추가해 총 10개로 보강했습니다.
- `f66b880a8f97`은 category 08의 reproducible toolchain/production verification story에 이미 속하므로 T1에서 제거했습니다.
- `0a28cb050bc8`은 `globals.css`를 만들고 다음 application integration이 이를 import하므로 T1에 추가했습니다.
- Presentation-only commits `61d1976cde0d`, `a6c72a6b3b34`, `20dfc298375c`는 starter catalog에서 presentation Thread로 이동했습니다. `d55a2017e725`, `9a7d41edfad0`, `8886459d1b0d`, `13c8c52c54d9`도 누락된 중간 계약/완성 단계로 추가했습니다.
- Selector consumers `daa6815a6dfa`, `383a3b86e119`를 추가해 policy 정의와 실제 renderer/route migration을 연결했습니다.
- 모든 commit SHA·subject·importance·tags는 target branch source classification과 exact commit metadata에 대조했습니다. 선택된 SHA는 모두 `web/portfolio` history에 속합니다.

## Thread index

| 순서 | Thread | Commit 수 |
| ---: | --- | ---: |
| 1 | [Runnable Next application boundary](./01-runnable-next-application-boundary.md) | 4 |
| 2 | [Portfolio domain and aggregate model](./02-portfolio-domain-and-aggregate-model.md) | 7 |
| 3 | [Presentation contracts for multi-route UI](./03-presentation-contracts-for-multi-route-ui.md) | 15 |
| 4 | [Selectors, links, and derived content policy](./04-selectors-links-and-derived-content-policy.md) | 9 |
| 5 | [Runtime schema vocabulary](./05-runtime-schema-vocabulary.md) | 12 |
| 6 | [Runtime presentation schema contracts](./06-runtime-presentation-schema-contracts.md) | 13 |
| 7 | [Starter catalog migration](./07-starter-catalog-migration.md) | 7 |
| 8 | [Source-aware schema parsing boundary](./08-source-aware-schema-parsing-boundary.md) | 4 |
| 9 | [Repository-wide content integrity](./09-repository-wide-content-integrity.md) | 9 |
| 10 | [Validated facade, assets, and build gate](./10-validated-facade-assets-and-build-gate.md) | 6 |

## Historical inspection discipline

- 각 commit은 exact SHA의 diff와 resulting tree를 기준으로 설명합니다.
- Later code는 earlier commit의 사실로 소급하지 않습니다.
- Shape validation, cross-file integrity, asset filesystem validation, selector fallback을 서로 다른 failure domain으로 구분합니다.
- 이번 환경에서는 GitHub 도메인 DNS 차단으로 local checkout과 repository test/build command를 실행하지 못했습니다. 실행 결과를 만들지 않았으며, GitHub connector를 통한 branch/commit/file 정적 검토만 기록했습니다.

## Workbook 상태

- `scaffold/`는 Phase 1 종료 시 동결한 authoritative investigation structure입니다.
- `completed/`는 fixed scaffold information을 그대로 보존하고 learner-facing section만 채운 counterpart입니다.
- 두 directory의 filename과 relative path는 정확히 일치합니다.
===== END FILE: README.md =====

