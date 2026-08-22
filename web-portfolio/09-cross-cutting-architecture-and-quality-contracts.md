===== BEGIN FILE: 01-fail-closed-content-ingestion.md =====
# Thread: Fail-closed content ingestion

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 source에서 확정된 thread 구조와 commit 역할만 미리 제공합니다. 실제 구현 해석, code evidence, failure 재현, test 결과, 최종 설명은 해당 SHA의 코드를 직접 확인해 작성합니다.

## 1. Thread 목표

타입 단언에 의존하던 JSON 입력이 파일별 schema 파싱, 누적 진단, 저장소 전체 참조 검증, 검증된 facade, 자산 경계, prebuild 차단으로 발전하는 과정을 복원합니다.

**Source-defined significance**

> The project evolves from typed-but-asserted JSON to a fail-closed ingestion system. File schemas, aggregate diagnostics, global identity rules, facade integration, asset containment, and prebuild enforcement successively remove places where invalid editorial data could survive into rendering.

### 이 Thread에 직접 연결되는 Critical Invariants

- 모든 authoritative content source는 consumer 이전에 schema parsing을 통과합니다. — `a944c73f0557 → d50870c8b8c4 → 03d2c9be0a43`
- Validation issue는 source file과 JSON path를 잃지 않고 누적됩니다. — `70e49ea34194 → d50870c8b8c4`
- Identifier와 ordering key는 repository 전체에서 결정적으로 해석됩니다. — `b9d74d8ccf08`
- Selectors와 routes는 검증된 facade만 소비합니다. — `508e0b71024b`
- Local asset은 public root 내부에 존재해야 하며 경계를 벗어날 수 없습니다. — `ff2ecadf3489`
- 잘못된 content는 production compilation 전에 build를 중단합니다. — `28b0db56190f`

### 연결되는 Major Engineering Difficulty

- Unchecked JSON과 broad application model을 strict schemas, aggregate validation, validated facade로 이동하면서 기존 route를 깨뜨리지 않는 문제
- Malformed value, duplicate identity, unresolved reference, unsupported route, missing/escaping asset를 하나의 actionable report로 축적하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 어느 시점부터 원시 JSON이 application data가 되기 전에 반드시 거치는 단일 경계가 생겼는가?
- 파일 내부 shape 오류와 파일 간 identifier/reference 오류는 각각 어느 계층이 소유하는가?
- Source file과 JSON path를 유지한 채 여러 문제를 모으는 흐름은 어떻게 구성되는가?
- 검증된 source와 portfolio facade 사이에 병렬 ingestion 경로가 남지 않았음을 어떤 코드로 확인할 수 있는가?
- Local asset과 production build가 같은 trust boundary에 포함되는 시점은 언제인가?

## 3. 완료 기준

- `raw JSON → schema output → repository-wide checks → portfolio facade → selectors/routes` 흐름을 실제 symbol과 경로로 설명할 수 있습니다.
- Schema failure, duplicate identifier, unresolved reference, escaping/missing asset의 owner를 구분했습니다.
- `03d2c9be0a43`과 `508e0b71024b`의 S-level 경계를 구분해 설명할 수 있습니다.
- `28b0db56190f`에서 validation failure가 Next.js compilation 이전 build failure로 전파되는 command chain을 기록했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a1977dc7f026` | build(content): runtime 콘텐츠 검증 의존성 추가 | B | CONTENT, VALIDATION, DEPLOY | Introduces the executable schema/tooling prerequisites. |
| 2 | `a944c73f0557` | feat(content): 프로젝트 사례 schema 추가 | A | CONTENT, VALIDATION | Captures the most complete case-study source contract. |
| 3 | `70e49ea34194` | feat(content): 콘텐츠 validation 오류 모델 추가 | A | CONTENT, VALIDATION | Creates accumulated, source-located validation failures. |
| 4 | `d50870c8b8c4` | feat(content): JSON schema 파싱 경계 추가 | A | CONTENT, VALIDATION | Turns raw JSON into schema output at a single parsing boundary. |
| 5 | `03d2c9be0a43` | feat(content): 콘텐츠 파일 schema 파싱 연결 | S | ARCH, CONTENT, VALIDATION | Makes that boundary authoritative for every source file. |
| 6 | `b9d74d8ccf08` | feat(content): 콘텐츠 식별자 중복 검증 추가 | A | CONTENT, VALIDATION | Makes repository-wide references deterministic. |
| 7 | `508e0b71024b` | refactor(content): 검증된 콘텐츠를 portfolio facade에 연결 | S | ARCH, CONTENT, VALIDATION | Moves all application consumers onto the validated pipeline. |
| 8 | `ff2ecadf3489` | feat(content): 저장소 자산 참조 경계 검증 | A | CONTENT, VALIDATION | Extends integrity to repository-hosted assets. |
| 9 | `28b0db56190f` | build(content): 콘텐츠 검사를 prebuild에 연결 | A | CONTENT, DEPLOY | Makes invalid content a production-build failure. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA를 checkout한 상태에서 작성합니다. Thread 내 이전 commit은 비교 대상으로 사용할 수 있지만 final HEAD를 정답처럼 소급하지 않습니다.

### 1. `a1977dc7f026` — build(content): runtime 콘텐츠 검증 의존성 추가

- **Importance:** B
- **Tags:** CONTENT, VALIDATION, DEPLOY
- **Source-defined thread role:** Introduces the executable schema/tooling prerequisites.
- **Source classification summary:** Add Zod as a runtime dependency and `tsx` as a development-time TypeScript runner, with the generated lockfile updated to capture their complete platform-specific resolution graph.
- **Source classification reason:** Supporting build or maintenance work inside the established release process; useful, but not a defining architectural or correctness decision.

#### Source에서 확정된 구현 의도와 상태 변화

Zod와 TypeScript 실행 도구를 추가해 runtime schema와 standalone validation tooling을 구현할 수 있는 기반을 마련합니다. 핵심 결정은 generated lockfile이 아니라 manifest의 dependency 경계입니다.

#### 해당 SHA에서 확인할 실제 코드

- Package manifest에서 Zod와 `tsx`가 각각 runtime/development dependency로 들어간 위치를 확인합니다.
- 이 SHA의 scripts를 확인해 아직 어떤 validation entry point가 존재하지 않는지 구분합니다.
- Lockfile의 기계적 dependency resolution과 사람이 검토해야 할 manifest 결정을 나눕니다.
- 직전 SHA와 비교해 application runtime behavior에는 아직 schema parsing이 추가되지 않았음을 확인합니다.

확인 원칙:

- 먼저 `a1977dc7f026^`와 `a1977dc7f026`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 이 dependency commit이 가능하게 한 후속 작업 | `a944c73f0557`이 실제 project source schema를 추가해 의존성을 첫 runtime contract로 사용합니다. |
| Runtime dependency와 dev-only runner의 lifecycle 차이 | Zod는 application/runtime validation이 소유하고 `tsx`는 repository command 실행 시점에만 필요합니다. |
| Lockfile이 제공하는 reproducibility evidence와 제공하지 않는 architecture 설명 | schema 정의, parsing, diagnostics, build gate는 보장하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** `package.json`에는 runtime schema library와 TypeScript validation runner가 없었고, content check script도 없었습니다.
- **해당 SHA 핵심 코드:** `package.json`의 `dependencies.zod`, `devDependencies.tsx`가 사람이 결정한 변경이고 `package-lock.json`은 그 해상도 결과입니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `a944c73f0557`이 실제 project source schema를 추가해 의존성을 첫 runtime contract로 사용합니다.

#### SHA별 복원 결론

- **Thread 내 역할:** Zod를 runtime dependency로, `tsx`를 development dependency로 추가했습니다.
- **실제 변경:** `package.json`의 `dependencies.zod`, `devDependencies.tsx`가 사람이 결정한 변경이고 `package-lock.json`은 그 해상도 결과입니다.
- **현재 보장:** pinned manifest와 lockfile이 이후 schema/tooling을 재현 가능하게 설치할 기반을 제공합니다.
- **남은 범위:** schema 정의, parsing, diagnostics, build gate는 보장하지 않습니다. `a944c73f0557`이 실제 project source schema를 추가해 의존성을 첫 runtime contract로 사용합니다.

### 2. `a944c73f0557` — feat(content): 프로젝트 사례 schema 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Captures the most complete case-study source contract.
- **Source classification summary:** Define the complete runtime schema for project case-study sources and the enclosing project catalog.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Project case-study source와 enclosing catalog를 strict runtime schema로 정의합니다. Group, tag, deployment, media, stack, links, architecture, decisions, trade-offs, results를 실행 가능한 input contract로 묶습니다.

#### 해당 SHA에서 확인할 실제 코드

- Project group, project item, enclosing document schema의 중첩 관계와 strict object 경계를 찾습니다.
- Stable ID, order, non-empty group/project collection, optional enablement/featured field를 실제 schema로 확인합니다.
- Screenshot, stack reference, placement-aware link, deployment 상태가 어느 schema를 재사용하는지 추적합니다.
- Schema-derived TypeScript type 또는 기존 hand-written type과의 연결 상태를 확인합니다.
- 이 schema가 shape는 보장하지만 cross-file reference는 아직 보장하지 않는다는 근거를 기록합니다.

확인 원칙:

- 먼저 `a944c73f0557^`와 `a944c73f0557`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Project list와 detail renderer가 공통으로 의존하는 field 집합 | `src/lib/content-schema.ts`의 project 관련 schemas가 group ID/order, project ID/order, screenshot, links, stack, architecture, decisions, trade-offs, results를 중첩 검증합니다. |
| 필수/선택/default field 구분 | project group/item/catalog를 strict Zod object로 정의하고 collection의 non-empty 조건과 필수·선택 field를 명시했습니다. |
| Schema가 보장하는 것과 아직 보장하지 않는 reference | cross-file reference, uniqueness, asset existence와 모든 consumer의 사용은 보장하지 않습니다. |
| 직전 관련 schema commit과 비교한 확장 지점 | Project JSON은 TypeScript 측 타입과 소비자 가정에 의존했고 runtime에서 중첩 shape를 검사하지 않았습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** Project JSON은 TypeScript 측 타입과 소비자 가정에 의존했고 runtime에서 중첩 shape를 검사하지 않았습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-schema.ts`의 project 관련 schemas가 group ID/order, project ID/order, screenshot, links, stack, architecture, decisions, trade-offs, results를 중첩 검증합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `70e49ea34194`와 `d50870c8b8c4`가 실패 위치를 보존하고 schema를 실제 parsing boundary로 만듭니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** Project JSON은 TypeScript 측 타입과 소비자 가정에 의존했고 runtime에서 중첩 shape를 검사하지 않았습니다. case-study의 group, media, deployment, stack, decision, result 중 하나만 틀려도 renderer까지 잘못된 값이 도달할 수 있었습니다.
- **구현 결정과 경로:** project group/item/catalog를 strict Zod object로 정의하고 collection의 non-empty 조건과 필수·선택 field를 명시했습니다. `src/lib/content-schema.ts`의 project 관련 schemas가 group ID/order, project ID/order, screenshot, links, stack, architecture, decisions, trade-offs, results를 중첩 검증합니다.
- **소유권·실패 처리:** schema는 한 project source의 구조를 소유하지만 다른 파일의 ID가 실제 존재하는지는 소유하지 않습니다. shape, enum, required/non-empty 조건 위반은 schema parse failure가 되도록 준비됐지만 이 SHA에는 전체 loader 연결이 없습니다.
- **보장:** 해당 schema를 통과한 project source의 파일 내부 runtime shape를 표현할 수 있습니다.
- **보장하지 않는 범위:** cross-file reference, uniqueness, asset existence와 모든 consumer의 사용은 보장하지 않습니다.
- **후속 연결:** `70e49ea34194`와 `d50870c8b8c4`가 실패 위치를 보존하고 schema를 실제 parsing boundary로 만듭니다.

### 3. `70e49ea34194` — feat(content): 콘텐츠 validation 오류 모델 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Creates accumulated, source-located validation failures.
- **Source classification summary:** Introduced the structured error model and source inventory for runtime content validation.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Validation issue에 source file, JSON path, message를 보존하고 여러 issue를 한 `PortfolioContentError`로 표현하는 오류 모델과 source inventory를 도입합니다.

#### 해당 SHA에서 확인할 실제 코드

- Issue type과 aggregate error class의 fields, constructor, message formatting을 확인합니다.
- Complete issue array와 human-readable message가 어떻게 분리되는지 확인합니다.
- Schema-backed source 이름과 override 가능한 input inventory를 찾습니다.
- Supported design과 navigable page 목록이 후속 validation을 위해 어떻게 표현되는지 확인합니다.
- 이 commit에서는 실제 schema parsing이 아직 연결되지 않은 경계를 기록합니다.

확인 원칙:

- 먼저 `70e49ea34194^`와 `70e49ea34194`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Throw-on-first-error 대신 issue accumulation을 선택한 이유 | `PortfolioContentValidationIssue`와 issue 배열을 보존하는 `PortfolioContentError`를 도입했습니다. |
| Source inventory가 후속 validator에 제공하는 context | `d50870c8b8c4`가 Zod issue를 이 모델로 변환합니다. |
| Machine-readable issue와 human-readable error text 구분 | `src/lib/content-loader.ts`의 issue type/error class, 14개 source inventory, supported design/navigation inventory가 후속 validator가 사용할 context를 만듭니다. |
| 아직 실제 issue를 생성하지 않는 부분 | 실제 parsing, file 간 누적 범위, uniqueness 검사는 보장하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 검증 실패를 source file과 JSON path 단위로 보존하는 공통 오류 모델이 없었습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-loader.ts`의 issue type/error class, 14개 source inventory, supported design/navigation inventory가 후속 validator가 사용할 context를 만듭니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `d50870c8b8c4`가 Zod issue를 이 모델로 변환합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** 검증 실패를 source file과 JSON path 단위로 보존하는 공통 오류 모델이 없었습니다. 일반 예외나 즉시 throw만 사용하면 여러 편집 오류를 한 번에 고치기 어렵고 발생 위치도 잃기 쉽습니다.
- **구현 결정과 경로:** `PortfolioContentValidationIssue`와 issue 배열을 보존하는 `PortfolioContentError`를 도입했습니다. `src/lib/content-loader.ts`의 issue type/error class, 14개 source inventory, supported design/navigation inventory가 후속 validator가 사용할 context를 만듭니다.
- **소유권·실패 처리:** machine-readable issue 배열은 validator가 만들고 human-readable message는 aggregate error가 조립합니다. 아직 schema parser가 호출되지 않아 이 모델 자체는 실제 issue를 생성하지 않습니다.
- **보장:** 후속 실패가 `file/path/message`를 잃지 않고 한 error에 담길 protocol을 제공합니다.
- **보장하지 않는 범위:** 실제 parsing, file 간 누적 범위, uniqueness 검사는 보장하지 않습니다.
- **후속 연결:** `d50870c8b8c4`가 Zod issue를 이 모델로 변환합니다.

### 4. `d50870c8b8c4` — feat(content): JSON schema 파싱 경계 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Turns raw JSON into schema output at a single parsing boundary.
- **Source classification summary:** Added the schema-parsing boundary that converts raw JSON into typed application data.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

`parseContentFile`이 raw JSON과 schema를 받아 `safeParse`를 실행하고, Zod issue를 source-aware issue로 변환하거나 schema output을 반환하는 ingestion primitive를 만듭니다.

#### 해당 SHA에서 확인할 실제 코드

- `parseContentFile`의 generic input/output type과 schema output 반환을 확인합니다.
- `safeParse` success/failure branch의 control flow를 추적합니다.
- Zod path를 JSON-style path로 바꾸고 source filename과 결합하는 호출을 찾습니다.
- 한 파일의 여러 issue가 aggregate error로 바뀌는 순서와 throw 지점을 확인합니다.
- Assertion/cast 없이 consumer가 받는 validated value의 type을 기록합니다.

확인 원칙:

- 먼저 `d50870c8b8c4^`와 `d50870c8b8c4`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Raw input ownership에서 validated output ownership으로 넘어가는 함수 경계 | 함수 호출 전 raw value는 caller 소유이고 성공 후에는 schema transformation/default가 반영된 output만 반환됩니다. |
| Schema transformation/default가 output에 반영되는 방식 | `parseContentFile`을 단일 file-local parsing primitive로 만들었습니다. `src/lib/content-loader.ts::parseContentFile`은 `schema.safeParse(raw)`를 호출하고 성공 시 schema output, 실패 시 Zod path를 JSON-style path로 바꾼 issue 배열을 `PortfolioContentError`로 throw합니다. |
| Failure 시 반환되지 않는 값과 보존되는 diagnostics | 모든 authoritative file이 이 함수를 사용한다는 사실은 아직 보장하지 않습니다. |
| 아직 모든 source가 이 함수를 반드시 사용한다고 보장하지 않는 이유 | 모든 authoritative file이 이 함수를 사용한다는 사실은 아직 보장하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** schema와 error model이 존재했지만 raw JSON을 반드시 통과시키는 공통 함수는 없었습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-loader.ts::parseContentFile`은 `schema.safeParse(raw)`를 호출하고 성공 시 schema output, 실패 시 Zod path를 JSON-style path로 바꾼 issue 배열을 `PortfolioContentError`로 throw합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `03d2c9be0a43`이 14개 source를 모두 이 함수에 연결합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** schema와 error model이 존재했지만 raw JSON을 반드시 통과시키는 공통 함수는 없었습니다. 소비자가 schema를 선택적으로 사용하면 unchecked JSON path가 계속 남을 수 있었습니다.
- **구현 결정과 경로:** `parseContentFile`을 단일 file-local parsing primitive로 만들었습니다. `src/lib/content-loader.ts::parseContentFile`은 `schema.safeParse(raw)`를 호출하고 성공 시 schema output, 실패 시 Zod path를 JSON-style path로 바꾼 issue 배열을 `PortfolioContentError`로 throw합니다.
- **소유권·실패 처리:** 함수 호출 전 raw value는 caller 소유이고 성공 후에는 schema transformation/default가 반영된 output만 반환됩니다. 한 파일에서 발생한 모든 Zod issue를 모은 뒤 throw하며 실패한 raw/output은 반환하지 않습니다.
- **보장:** 이 함수를 통과한 단일 파일의 output은 해당 schema에 의해 runtime 검증됩니다.
- **보장하지 않는 범위:** 모든 authoritative file이 이 함수를 사용한다는 사실은 아직 보장하지 않습니다.
- **후속 연결:** `03d2c9be0a43`이 14개 source를 모두 이 함수에 연결합니다.

### 5. `03d2c9be0a43` — feat(content): 콘텐츠 파일 schema 파싱 연결

- **Importance:** S
- **Tags:** ARCH, CONTENT, VALIDATION
- **Source-defined thread role:** Makes that boundary authoritative for every source file.
- **Source classification summary:** Connected every portfolio content file to its corresponding runtime schema through a single loader.
- **Source classification reason:** Critical because it makes one loader the authoritative trust boundary for every content file; without it, the later integrity, route, and build guarantees would still rest on unchecked JSON.

#### Source에서 확정된 구현 의도와 상태 변화

모든 portfolio content file을 각 schema와 source name으로 파싱하는 `loadPortfolioSource`를 만들고 validated value만 application model에 노출합니다. Targeted override도 같은 ingestion 경계를 사용합니다.

#### S-level source profile

- **Problem:** Typed JSON file이 많아도 TypeScript type만으로 malformed runtime data가 selector와 route에 도달하는 것을 막을 수 없었습니다.
- **Decision:** 모든 authoritative file을 각 schema와 source name으로 `loadPortfolioSource`에서 파싱하고 schema output만 반환합니다.
- **Why it mattered:** Schema 정의를 선택적 도구가 아니라 유일한 ingestion path로 바꾸며 이후 uniqueness, reference, asset, readiness, build guarantee의 기반이 됩니다.
- **What changed:** Default JSON modules와 targeted override를 한 loader에서 조립하고 file-local error를 consumption 이전에 보고합니다.

#### 해당 SHA에서 확인할 실제 코드

- `loadPortfolioSource`가 default JSON modules와 targeted overrides를 조립하는 순서를 추적합니다.
- 각 file name이 어떤 schema와 `parseContentFile` 호출에 연결되는지 전체 목록을 작성합니다.
- 여러 file parse가 실패할 때 issue가 파일 경계를 넘어 어떻게 합쳐지는지 확인합니다.
- Successful return type이 schema output에서 파생되는지 확인합니다.
- Exported singleton 또는 authoritative source가 어디서 만들어지는지 찾습니다.
- 이 SHA에서 raw JSON direct import 우회가 남아 있는지 검색하되 후속 facade migration 전 상태와 구분합니다.
- Override input의 lifetime과 production source 격리를 확인합니다.

확인 원칙:

- 먼저 `03d2c9be0a43^`와 `03d2c9be0a43`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태: schema와 parser가 있었지만 all-file authoritative loader가 없었던 call graph | `parseContentFile`은 존재했지만 raw JSON direct import를 대체하는 all-file loader가 없어 선택적 도구에 머물렀습니다. |
| 어떤 source가 validation을 건너뛸 수 있었는지 | `parseContentFile`은 존재했지만 raw JSON direct import를 대체하는 all-file loader가 없어 선택적 도구에 머물렀습니다. |
| File-to-schema mapping과 single loader decision | `loadPortfolioSource(overrides = {})`에서 모든 authoritative JSON과 schema/source name의 mapping을 고정하고 `portfolioSource` singleton을 만들었습니다. |
| 핵심 loader/parser/output type/export code | `src/lib/content-loader.ts::loadPortfolioSource`가 site/profile/projects/presentation/skills/techStack/experience/journey/journeyNarrative/interviewMap/curation/links/contact/resume를 각각 `parseContentFile`로 처리합니다. Override도 같은 parser를 통과합니다. |
| 한 파일/여러 파일 malformed일 때 failure path | 중요한 실제 한계가 있습니다. `parseContentFile`이 파일별 실패에서 즉시 throw하므로 한 파일 내부 issue는 누적되지만 여러 파일이 동시에 malformed이면 첫 실패 파일에서 loader가 중단됩니다. |
| 보장하는 것: loader output의 runtime shape validation | loader output에 포함된 모든 source의 파일 내부 runtime shape가 검증됩니다. |
| 아직 보장하지 않는 것: uniqueness/reference/asset/facade consumer migration | repository-wide uniqueness/reference, asset, facade consumer migration과 cross-file parse-error aggregation은 아직 보장하지 않습니다. |
| 다음 commit과의 연결 | `b9d74d8ccf08`이 repository-wide key validation을 추가하고 `508e0b71024b`가 consumer를 이 output으로 이동합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** `parseContentFile`은 존재했지만 raw JSON direct import를 대체하는 all-file loader가 없어 선택적 도구에 머물렀습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-loader.ts::loadPortfolioSource`가 site/profile/projects/presentation/skills/techStack/experience/journey/journeyNarrative/interviewMap/curation/links/contact/resume를 각각 `parseContentFile`로 처리합니다. Override도 같은 parser를 통과합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `b9d74d8ccf08`이 repository-wide key validation을 추가하고 `508e0b71024b`가 consumer를 이 output으로 이동합니다.

#### SHA별 복원 결론

- **이전 상태:** `parseContentFile`은 존재했지만 raw JSON direct import를 대체하는 all-file loader가 없어 선택적 도구에 머물렀습니다.
- **문제와 위험:** schema가 있어도 source 하나가 parser를 우회하면 전체 application trust boundary가 성립하지 않습니다.
- **핵심 결정:** `loadPortfolioSource(overrides = {})`에서 모든 authoritative JSON과 schema/source name의 mapping을 고정하고 `portfolioSource` singleton을 만들었습니다.
- **실제 구현 경로:** `src/lib/content-loader.ts::loadPortfolioSource`가 site/profile/projects/presentation/skills/techStack/experience/journey/journeyNarrative/interviewMap/curation/links/contact/resume를 각각 `parseContentFile`로 처리합니다. Override도 같은 parser를 통과합니다.
- **소유권·상태 변화:** default JSON module 또는 targeted override는 loader input이고, 성공한 `PortfolioSource`만 이후 계층으로 넘어갑니다.
- **실패 경계:** 중요한 실제 한계가 있습니다. `parseContentFile`이 파일별 실패에서 즉시 throw하므로 한 파일 내부 issue는 누적되지만 여러 파일이 동시에 malformed이면 첫 실패 파일에서 loader가 중단됩니다.
- **보장:** loader output에 포함된 모든 source의 파일 내부 runtime shape가 검증됩니다.
- **보장하지 않는 범위:** repository-wide uniqueness/reference, asset, facade consumer migration과 cross-file parse-error aggregation은 아직 보장하지 않습니다.
- **후속 fix/test:** `b9d74d8ccf08`이 repository-wide key validation을 추가하고 `508e0b71024b`가 consumer를 이 output으로 이동합니다.

### 6. `b9d74d8ccf08` — feat(content): 콘텐츠 식별자 중복 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Makes repository-wide references deterministic.
- **Source classification summary:** Added whole-repository uniqueness checks for the identifiers and ordering keys that other content records depend on.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Project group/order, metric, project/order, technology, link, journey, interview, curation, design, navigation identifiers와 ordering key 중복을 repository 전체에서 검사합니다.

#### 해당 SHA에서 확인할 실제 코드

- Duplicate helper가 repeated value를 한 번만 issue로 기록하는 방식을 확인합니다.
- 검사 대상 collection과 key selector를 source 순서대로 목록화합니다.
- Uniqueness validation이 reference validation보다 먼저 실행되는 call order를 찾습니다.
- 동일 order와 동일 ID가 어떤 source path/message로 보고되는지 비교합니다.
- 여러 collection의 collision이 한 aggregate error에 누적되는지 확인합니다.

확인 원칙:

- 먼저 `b9d74d8ccf08^`와 `b9d74d8ccf08`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Identifier가 하나의 semantic record로 결정된다는 invariant | 검사된 identifier와 ordering key가 하나의 record/position으로 결정됩니다. |
| Order collision과 identity collision 차이 | whole-repository duplicate checks를 loader 후단에 추가했습니다. |
| 세 번 이상 반복된 값의 issue cardinality | `src/lib/content-loader.ts`의 `findDuplicates` 계열 검사가 project group ID/order, metric ID, project ID/order, technology/link/milestone/interview/curation/design IDs와 navigation href를 검사합니다. |
| 아직 unresolved reference를 직접 검사하지 않는 범위 | ID가 다른 파일에서 실제로 참조되는지와 asset 존재 여부는 이 commit만으로 보장하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 파일별 schema가 통과해도 서로 다른 record가 같은 ID나 order를 사용할 수 있었습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-loader.ts`의 `findDuplicates` 계열 검사가 project group ID/order, metric ID, project ID/order, technology/link/milestone/interview/curation/design IDs와 navigation href를 검사합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `508e0b71024b`이 이 validated source를 facade의 canonical input으로 사용합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** 파일별 schema가 통과해도 서로 다른 record가 같은 ID나 order를 사용할 수 있었습니다. 중복 identity/order는 lookup, sort, cross-reference 결과를 입력 순서나 마지막 write에 의존하게 만듭니다.
- **구현 결정과 경로:** whole-repository duplicate checks를 loader 후단에 추가했습니다. `src/lib/content-loader.ts`의 `findDuplicates` 계열 검사가 project group ID/order, metric ID, project ID/order, technology/link/milestone/interview/curation/design IDs와 navigation href를 검사합니다.
- **소유권·실패 처리:** seen set이 첫 값을 소유하고 duplicates set이 중복 value를 한 번만 report 대상으로 보존합니다. 같은 값이 세 번 이상 나와도 duplicate value당 issue 하나가 생성되며 여러 collection의 issue는 최종 array에 누적됩니다.
- **보장:** 검사된 identifier와 ordering key가 하나의 record/position으로 결정됩니다.
- **보장하지 않는 범위:** ID가 다른 파일에서 실제로 참조되는지와 asset 존재 여부는 이 commit만으로 보장하지 않습니다.
- **후속 연결:** `508e0b71024b`이 이 validated source를 facade의 canonical input으로 사용합니다.

### 7. `508e0b71024b` — refactor(content): 검증된 콘텐츠를 portfolio facade에 연결

- **Importance:** S
- **Tags:** ARCH, CONTENT, VALIDATION
- **Source-defined thread role:** Moves all application consumers onto the validated pipeline.
- **Source classification summary:** Moved the portfolio facade from direct JSON imports and unchecked type assertions to the validated `portfolioSource`.
- **Source classification reason:** Critical because it completes the single validated data pipeline used by selectors and routes, replacing parallel imports and duplicated relationships with one canonical application model.

#### Source에서 확정된 구현 의도와 상태 변화

Portfolio facade가 direct JSON import와 unchecked assertion을 버리고 validated `portfolioSource`를 사용합니다. Group ID를 canonical relationship으로 삼고 metrics, links, journey, interview, curation을 한 aggregate로 노출합니다.

#### S-level source profile

- **Problem:** Schema parsing이 있어도 facade가 JSON을 직접 import해 병렬 ingestion path가 남아 있었습니다.
- **Decision:** Facade를 `portfolioSource`로 이동하고 group identifier를 canonical relationship으로 사용합니다.
- **Why it mattered:** List, detail, metrics, links, journey, interview, curation이 같은 source of truth를 사용하게 합니다.
- **What changed:** Direct JSON import와 presentation-layer environment substitution을 validated relationship derivation으로 대체합니다.

#### 해당 SHA에서 확인할 실제 코드

- Facade import에서 raw JSON이 제거되고 `portfolioSource`로 대체되는 diff를 확인합니다.
- Project group sorting이 한 번 수행되고 category/group copy에 재사용되는 흐름을 추적합니다.
- Environment URL substitution이 제거되거나 legacy argument가 의도적으로 무시되는 branch를 찾습니다.
- Selectors와 route-facing `PortfolioContent`가 facade output을 받는 call graph를 작성합니다.
- Journey narrative, interview map, curation, metrics의 exposure ownership을 확인합니다.
- Repository 전체에서 direct JSON import 또는 별도 assertion path가 남는지 이 SHA 기준으로 검색합니다.
- Canonical collection과 derived collection의 referential identity를 구분합니다.

확인 원칙:

- 먼저 `508e0b71024b^`와 `508e0b71024b`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 loader를 facade가 bypass하던 import/relationship | authoritative loader가 있어도 portfolio facade는 raw JSON import와 unchecked cast를 사용해 병렬 ingestion path를 유지했습니다. |
| 두 ingestion path가 다른 interpretation을 만들 수 있었던 이유 | `src/lib/portfolio/content.ts`, `src/lib/portfolio/selectors.ts`, `src/lib/portfolio/types.ts`가 validated source를 받아 group/project/journey를 정렬하고 enabled project/link를 필터링하며 metrics, narrative, interview, curation을 aggregate에 포함합니다. |
| `portfolioSource`와 group ID 중심 aggregate decision | facade input을 `portfolioSource` 하나로 바꾸고 group ID를 canonical relationship으로 사용했습니다. |
| Facade construction, sort/derive, exported API | facade input을 `portfolioSource` 하나로 바꾸고 group ID를 canonical relationship으로 사용했습니다. `src/lib/portfolio/content.ts`, `src/lib/portfolio/selectors.ts`, `src/lib/portfolio/types.ts`가 validated source를 받아 group/project/journey를 정렬하고 enabled project/link를 필터링하며 metrics, narrative, interview, curation을 aggregate에 포함합니다. |
| Raw source, canonical validated collection, derived collection ownership | loader는 canonical validated source를 소유하고 facade는 sort/filter/derived collection을 소유하며 routes/selectors는 facade만 소비합니다. |
| Disabled/missing 관계의 absence 처리 | disabled record는 derived public collection에서 제외되고 missing optional relationship은 selector/facade 정책에 따라 absence로 남습니다. Presentation-layer environment URL substitution도 제거됐습니다. |
| 보장하는 것: application consumer의 single validated pipeline | application consumer가 하나의 validated pipeline을 사용합니다. |
| 아직 보장하지 않는 것: asset containment와 prebuild enforcement | public asset containment, actual file existence와 automatic build enforcement는 아직 보장하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** authoritative loader가 있어도 portfolio facade는 raw JSON import와 unchecked cast를 사용해 병렬 ingestion path를 유지했습니다.
- **해당 SHA 핵심 코드:** `src/lib/portfolio/content.ts`, `src/lib/portfolio/selectors.ts`, `src/lib/portfolio/types.ts`가 validated source를 받아 group/project/journey를 정렬하고 enabled project/link를 필터링하며 metrics, narrative, interview, curation을 aggregate에 포함합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `ff2ecadf3489`가 repository asset boundary를, `28b0db56190f`가 build gate를 닫습니다.

#### SHA별 복원 결론

- **이전 상태:** authoritative loader가 있어도 portfolio facade는 raw JSON import와 unchecked cast를 사용해 병렬 ingestion path를 유지했습니다.
- **문제와 위험:** loader와 facade가 같은 관계를 다르게 해석하면 route마다 검증 수준과 group membership이 달라질 수 있었습니다.
- **핵심 결정:** facade input을 `portfolioSource` 하나로 바꾸고 group ID를 canonical relationship으로 사용했습니다.
- **실제 구현 경로:** `src/lib/portfolio/content.ts`, `src/lib/portfolio/selectors.ts`, `src/lib/portfolio/types.ts`가 validated source를 받아 group/project/journey를 정렬하고 enabled project/link를 필터링하며 metrics, narrative, interview, curation을 aggregate에 포함합니다.
- **소유권·상태 변화:** loader는 canonical validated source를 소유하고 facade는 sort/filter/derived collection을 소유하며 routes/selectors는 facade만 소비합니다.
- **실패 경계:** disabled record는 derived public collection에서 제외되고 missing optional relationship은 selector/facade 정책에 따라 absence로 남습니다. Presentation-layer environment URL substitution도 제거됐습니다.
- **보장:** application consumer가 하나의 validated pipeline을 사용합니다.
- **보장하지 않는 범위:** public asset containment, actual file existence와 automatic build enforcement는 아직 보장하지 않습니다.
- **후속 fix/test:** `ff2ecadf3489`가 repository asset boundary를, `28b0db56190f`가 build gate를 닫습니다.

### 8. `ff2ecadf3489` — feat(content): 저장소 자산 참조 경계 검증

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Extends integrity to repository-hosted assets.
- **Source classification summary:** Added repository-boundary validation for every local asset referenced by portfolio content.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Social image, profile portrait, résumé download, project screenshots의 local asset reference를 source file/path와 함께 수집하고 public root containment와 실제 file existence를 검증합니다.

#### 해당 SHA에서 확인할 실제 코드

- 각 asset category와 originating JSON path를 수집하는 code를 찾습니다.
- Public URL이 filesystem path로 변환되는 순서와 public root 주입 방식을 추적합니다.
- Absolute path, root escape/path traversal, missing file을 각각 거부하는 branch를 확인합니다.
- Failure를 기존 `PortfolioContentError` issue로 누적하는 흐름을 확인합니다.
- Committed portrait placeholder가 어떤 content reference를 만족시키는지 해당 SHA에서 확인합니다.

확인 원칙:

- 먼저 `ff2ecadf3489^`와 `ff2ecadf3489`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| URL namespace와 filesystem namespace 변환 규칙 | content에서 local asset reference를 수집하고 주입된 public root 아래의 실제 file로 resolve한 뒤 containment/existence를 검사했습니다. |
| Containment 검사 전후 normalization 순서 | `src/lib/content-assets.ts`가 site social image, profile portrait, resume download, project screenshots를 source path와 함께 수집하고 `resolve`, `relative`, `existsSync`로 검사합니다. `public/template/profile/portrait-placeholder.svg`도 추가됐습니다. |
| 여러 asset issue가 한 번에 보고되는 증거 | content에서 local asset reference를 수집하고 주입된 public root 아래의 실제 file로 resolve한 뒤 containment/existence를 검사했습니다. `src/lib/content-assets.ts`가 site social image, profile portrait, resume download, project screenshots를 source path와 함께 수집하고 `resolve`, `relative`, `existsSync`로 검사합니다. `public/template/profile/portrait-placeholder.svg`도 추가됐습니다. |
| Schema path validation만으로 existence를 보장할 수 없었던 이유 | `src/lib/content-assets.ts`가 site social image, profile portrait, resume download, project screenshots를 source path와 함께 수집하고 `resolve`, `relative`, `existsSync`로 검사합니다. `public/template/profile/portrait-placeholder.svg`도 추가됐습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** schema가 local-looking path 문자열을 허용해도 실제 public file 존재와 root containment는 확인하지 않았습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-assets.ts`가 site social image, profile portrait, resume download, project screenshots를 source path와 함께 수집하고 `resolve`, `relative`, `existsSync`로 검사합니다. `public/template/profile/portrait-placeholder.svg`도 추가됐습니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `28b0db56190f`가 schema와 asset 검사를 normal build 전에 실행합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** schema가 local-looking path 문자열을 허용해도 실제 public file 존재와 root containment는 확인하지 않았습니다. 누락 파일이나 `..` 경로가 production render/build 시점까지 살아남을 수 있었습니다.
- **구현 결정과 경로:** content에서 local asset reference를 수집하고 주입된 public root 아래의 실제 file로 resolve한 뒤 containment/existence를 검사했습니다. `src/lib/content-assets.ts`가 site social image, profile portrait, resume download, project screenshots를 source path와 함께 수집하고 `resolve`, `relative`, `existsSync`로 검사합니다. `public/template/profile/portrait-placeholder.svg`도 추가됐습니다.
- **소유권·실패 처리:** content는 URL-like reference를 소유하고 validator는 filesystem translation과 public-root boundary를 소유합니다. relative 결과가 root 밖이거나 target이 없으면 issue를 누적해 `PortfolioContentError`로 throw합니다. Scaffold 표현과 달리 이 SHA는 URL의 선행 `/` 자체를 별도 branch로 거부하지 않고 `resolve` 결과의 containment/existence로 판단합니다.
- **보장:** 수집 대상 local asset이 public root를 벗어나지 않고 검사 시점에 존재함을 보장합니다.
- **보장하지 않는 범위:** MIME, decode 가능성, production readiness와 실제 HTTP serving은 보장하지 않습니다.
- **후속 연결:** `28b0db56190f`가 schema와 asset 검사를 normal build 전에 실행합니다.

### 9. `28b0db56190f` — build(content): 콘텐츠 검사를 prebuild에 연결

- **Importance:** A
- **Tags:** CONTENT, DEPLOY
- **Source-defined thread role:** Makes invalid content a production-build failure.
- **Source classification summary:** Connected the content validation command to the package's `prebuild` lifecycle.
- **Source classification reason:** Significant because it changes or hardens the reproducible production-build and runtime boundary rather than merely adjusting local tooling.

#### Source에서 확정된 구현 의도와 상태 변화

`content:check`를 package `prebuild` lifecycle에 연결해 structured content가 잘못되면 Next.js compilation 전에 production build를 중단합니다.

#### 해당 SHA에서 확인할 실제 코드

- Package scripts에서 `prebuild`, `build`, `content:check`의 exact command chain을 확인합니다.
- `content:check`가 production loader와 asset validator를 재사용하는지 추적합니다.
- 직전 SHA에서 manual command였지만 build가 강제하지 않았던 차이를 비교합니다.
- Intentional failure에서 Next compiler output 전 process가 종료되는지 실행 결과를 기록합니다.
- Success status의 project/design count와 실제 build guarantee를 구분합니다.

확인 원칙:

- 먼저 `28b0db56190f^`와 `28b0db56190f`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Build lifecycle 순서와 failure status | loader 또는 asset validator throw가 uncaught 상태로 command를 non-zero 종료시키므로 뒤의 Next compilation이 시작되지 않습니다. |
| Manual check와 automatic gate 차이 | `scripts/validate-content.ts`는 `loadPortfolioSource()`와 `validatePortfolioAssets(..., resolve(cwd, "public"))`를 호출하고 `package.json`의 `prebuild`가 `npm run content:check`를 실행합니다. |
| Structural/asset integrity는 강제하지만 production readiness는 아직 아닌 범위 | template marker, public origin, usable project/contact 같은 production publication readiness는 아직 검사하지 않습니다. |
| 재현 command와 실제 exit code/output | `scripts/validate-content.ts`는 `loadPortfolioSource()`와 `validatePortfolioAssets(..., resolve(cwd, "public"))`를 호출하고 `package.json`의 `prebuild`가 `npm run content:check`를 실행합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** content/asset validator는 수동 실행할 수 있었지만 `next build`와 강제 연결되지 않았습니다.
- **해당 SHA 핵심 코드:** `scripts/validate-content.ts`는 `loadPortfolioSource()`와 `validatePortfolioAssets(..., resolve(cwd, "public"))`를 호출하고 `package.json`의 `prebuild`가 `npm run content:check`를 실행합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** 다음 thread의 `37c0dbc079ff`이 stricter readiness를 같은 prebuild chain에 추가합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** content/asset validator는 수동 실행할 수 있었지만 `next build`와 강제 연결되지 않았습니다. 개발자가 check command를 생략하면 invalid editorial data가 production compilation에 들어갈 수 있었습니다.
- **구현 결정과 경로:** `content:check`를 만들고 package `prebuild`에서 호출하도록 연결했습니다. `scripts/validate-content.ts`는 `loadPortfolioSource()`와 `validatePortfolioAssets(..., resolve(cwd, "public"))`를 호출하고 `package.json`의 `prebuild`가 `npm run content:check`를 실행합니다.
- **소유권·실패 처리:** validation script가 production source/public root를 소유하고 npm lifecycle이 failure propagation을 소유합니다. loader 또는 asset validator throw가 uncaught 상태로 command를 non-zero 종료시키므로 뒤의 Next compilation이 시작되지 않습니다.
- **보장:** normal `npm run build`가 structural content와 local asset integrity를 선행 조건으로 가집니다.
- **보장하지 않는 범위:** template marker, public origin, usable project/contact 같은 production publication readiness는 아직 검사하지 않습니다.
- **후속 연결:** 다음 thread의 `37c0dbc079ff`이 stricter readiness를 같은 prebuild chain에 추가합니다.

## 6. Invariant ledger

| Invariant | Source에서 확인된 변화 지점 | 해당 SHA의 실제 code/test evidence | 부족함이 드러난 시점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| 모든 authoritative content source는 consumer 이전에 schema parsing을 통과합니다. | `a944c73f0557 → d50870c8b8c4 → 03d2c9be0a43` | `a944c73f0557`의 strict project schemas, `d50870c8b8c4::parseContentFile`, `03d2c9be0a43::loadPortfolioSource`의 14개 file-to-schema mapping과 exported `portfolioSource`. | `d50870c8b8c4`까지 parser는 선택적이었고 `03d2c9be0a43`에서 authoritative loader가 됐습니다. 단, 여러 파일 동시 parse failure는 첫 failing file에서 중단됩니다. | loader가 반환한 모든 authoritative source는 해당 file schema를 통과합니다. 이후 facade consumer도 이 output만 사용합니다. |
| Validation issue는 source file과 JSON path를 잃지 않고 누적됩니다. | `70e49ea34194 → d50870c8b8c4` | `PortfolioContentError.issues`, Zod issue의 `file/path/message` 변환과 JSON-style path formatter. | `70e49ea34194`에서는 protocol만 있었고 `d50870c8b8c4`부터 한 파일의 모든 Zod issue를 실제로 누적합니다. | 한 failing file의 diagnostics는 source/path를 보존합니다. File 간 shape issue까지 한 번에 모으는 것은 최종 구현도 보장하지 않습니다. |
| Identifier와 ordering key는 repository 전체에서 결정적으로 해석됩니다. | `b9d74d8ccf08` | `b9d74d8ccf08`의 seen/duplicate set과 ID/order별 issue collection. | 파일 내부 schema만 통과한 상태에서는 duplicate ID/order가 허용됐습니다. | 명시된 group/metric/project/technology/link/journey/interview/curation/design/navigation key는 duplicate value당 하나의 issue로 거부됩니다. |
| Selectors와 routes는 검증된 facade만 소비합니다. | `508e0b71024b` | `508e0b71024b`에서 facade가 raw JSON/cast 대신 `portfolioSource`를 import하고 routes/selectors가 facade output을 사용합니다. | loader가 생긴 뒤에도 facade direct imports가 병렬 path로 남아 있었습니다. | application aggregate, selectors와 routes가 하나의 validated source 및 canonical group relationship을 소비합니다. |
| Local asset은 public root 내부에 존재해야 하며 경계를 벗어날 수 없습니다. | `ff2ecadf3489` | `ff2ecadf3489::validatePortfolioAssets`의 content-derived refs, `resolve`/`relative` containment, `existsSync`와 aggregate issues. | schema path 문자열만으로는 root escape와 missing file을 알 수 없었습니다. | 수집 대상 asset은 검사 시점 public root 내부에 존재해야 합니다. MIME/HTTP serving은 별도 thread 범위입니다. |
| 잘못된 content는 production compilation 전에 build를 중단합니다. | `28b0db56190f` | `28b0db56190f`의 `content:check` script와 npm `prebuild` lifecycle. | validator가 수동 command일 때는 normal build에서 생략할 수 있었습니다. | `npm run build`는 Next compilation 전에 structural source와 asset check를 통과해야 합니다. Stricter production readiness는 후속 thread에서 추가됩니다. |

Ledger 작성 원칙:

- Source가 명시한 invariant만 사용하고 새 invariant를 확정 사실처럼 추가하지 않습니다.
- 도입, 강화, 부족함 노출, fix, regression test가 서로 다른 commit이면 각 열에 분리해 기록합니다.
- Code evidence에는 실제 field, function, branch, selector, command 또는 assertion을 적습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 위험 | 대응 commit | 실제 수정/강화 code에서 확인할 것 | Test 또는 실행 증거 |
| --- | --- | --- | --- |
| 타입은 맞는다고 단언했지만 runtime JSON shape가 잘못됨 | d50870c8b8c4, 03d2c9be0a43 | 각 schema의 `safeParse` 결과와 structured issue 변환을 확인합니다. | Override input을 사용하는 validation scenario를 해당 SHA에서 찾습니다. |
| 서로 다른 파일이 같은 identifier/order를 선언함 | b9d74d8ccf08 | 중복 검사를 reference 검사보다 먼저 수행하는 순서와 누적 issue를 확인합니다. | 세 번 이상 반복된 값의 issue cardinality를 기록합니다. |
| Content가 public root 밖을 가리키거나 실제 파일이 없음 | ff2ecadf3489 | URL 해석, containment, absolute/escape 거부, existence check를 추적합니다. | 여러 asset failure가 한 번에 보고되는지 확인합니다. |
| 검증 command를 생략해 invalid content가 build에 포함됨 | 28b0db56190f | `prebuild`에서 동일 loader/validator와 exit status 전파를 확인합니다. | Malformed source에서 Next compilation이 시작되지 않는지 기록합니다. |

### 실제 연결 기록

- Shape failure는 `parseContentFile`, duplicate identity/order는 repository validator, asset escape/missing은 `validatePortfolioAssets`, build propagation은 npm `prebuild`가 각각 소유합니다.
- `03d2c9be0a43`의 실제 구현은 파일 내부 issue는 누적하지만 여러 malformed 파일을 모두 parse한 뒤 합치지는 않습니다. 이 차이를 scaffold의 기대와 구분했습니다.
- `ff2ecadf3489`은 normalized target containment/existence를 검사하며 absolute-looking URL을 별도 error category로 나누지 않습니다.

## 8. Ownership / state / responsibility 변화

| Concern | Thread 초기 owner/state | Thread 최종 owner/state | 실제 symbol과 호출 경로 |
| --- | --- | --- | --- |
| 원시 JSON shape | TypeScript assertion과 개별 import | 파일별 schema와 `parseContentFile` | `src/lib/content-loader.ts::parseContentFile`: raw JSON + schema → validated output 또는 `PortfolioContentError`. |
| 오류 위치와 집계 | 일반 예외 또는 분산 방어 | `PortfolioContentError`와 source-aware issue collection | `PortfolioContentValidationIssue`/`PortfolioContentError`: validator issue array → source-aware formatted error. |
| Repository identity/order | Consumer의 암묵적 해석 | Whole-repository validation | `loadPortfolioSource` 후단의 duplicate validators: validated collections → deterministic ID/order set. |
| Application aggregate | 직접 JSON import와 중복 derivation | Validated `portfolioSource` 기반 facade | `loadPortfolioSource` → exported `portfolioSource` → `src/lib/portfolio/content.ts` facade → selectors/App Router pages. |
| Local asset validity | Render/build에서 뒤늦은 실패 | Repository-boundary asset validator | `src/lib/content-assets.ts::validatePortfolioAssets`: content asset refs + injected public root → containment/existence issues. |
| Build 차단 | 수동 검사 가능성 | `prebuild` lifecycle | `package.json::prebuild` → `scripts/validate-content.ts` → loader + asset validator; throw가 npm status로 전파됩니다. |

## 9. Thread 최종 상태

### Source에서 확정된 최종 상태

타입 단언에 의존하던 JSON 입력이 파일별 schema 파싱, 누적 진단, 저장소 전체 참조 검증, 검증된 facade, 자산 경계, prebuild 차단으로 발전하는 과정을 복원합니다.

### 학습자가 완성할 최종 설명

- **Thread 시작 시점의 설계와 위험:** 시작 시점에는 runtime JSON이 TypeScript assertion과 direct import를 통해 renderer까지 갈 수 있었고 file location을 보존하는 공통 failure report도 없었습니다.
- **핵심 architecture/decision이 형성된 순서:** schema 정의 → source-aware parser → all-file loader → repository duplicate checks → validated facade → asset validator → prebuild 순서로 trust boundary가 형성됐습니다.
- **실제 failure 또는 부족함이 드러난 지점:** 중간에 parser가 있어도 facade가 raw JSON을 계속 import해 병렬 ingestion path가 남았고, schema-valid path가 실제 file 존재를 뜻하지 않는 공백도 드러났습니다.
- **Fix 또는 boundary 강화가 바꾼 invariant:** `508e0b71024b`이 consumer 우회를 제거하고 `ff2ecadf3489`/`28b0db56190f`이 filesystem과 build lifecycle까지 fail-closed로 확장했습니다.
- **Test/build/browser evidence가 보장한 범위:** 정적 검토로 file-to-schema mapping, duplicate branches, asset containment와 npm command chain을 확인했습니다. 환경 제한 때문에 command exit/result는 실행 증거로 만들지 않았습니다.
- **Thread 종료 시점에도 보장하지 않는 범위:** 여러 malformed 파일의 parse issue를 하나의 report로 합치는 것, MIME/HTTP serving, production placeholder/origin completeness는 이 thread 종료 시점의 보장이 아닙니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 source-backed flow의 각 단계에 실제 file path, symbol, input/output, failure branch를 추가합니다.

1. Authoritative JSON source와 targeted test override를 조립합니다.
   - 실제 코드 위치: `src/lib/content-loader.ts::loadPortfolioSource`
   - 입력과 출력: 14개 imported JSON module과 optional per-key override를 받아 하나의 loader input set을 구성합니다.
   - 실패/absence 처리: override가 있어도 schema를 우회하지 않으며 first failing file의 `PortfolioContentError`가 호출을 중단합니다.
2. 각 source file에 대응하는 schema와 source name을 선택합니다.
   - 실제 코드 위치: `contentFiles`/각 explicit `parseContentFile` call in `src/lib/content-loader.ts`
   - 입력과 출력: source key별 raw value, schema, `src/content/*.json` filename을 결합합니다.
   - 실패/absence 처리: mapping 누락은 type/source inventory 검토 대상이며 schema mismatch는 parser failure가 됩니다.
3. `safeParse` 결과를 typed output 또는 source/path/message issue로 변환합니다.
   - 실제 코드 위치: `parseContentFile`
   - 입력과 출력: `safeParse` 성공 시 transformed/defaulted output, 실패 시 Zod issue를 JSON path가 있는 issue array로 변환합니다.
   - 실패/absence 처리: issue가 하나라도 있으면 output을 반환하지 않고 aggregate error를 throw합니다.
4. 파일별 output 위에 uniqueness와 cross-file policy를 적용합니다.
   - 실제 코드 위치: `loadPortfolioSource`의 repository validators
   - 입력과 출력: validated collections에서 ID/order/href values를 추출해 duplicates를 찾습니다.
   - 실패/absence 처리: duplicate value당 source-aware issue를 누적하고 validation error로 중단합니다.
5. Asset path를 public root 아래로 제한하고 실제 존재를 확인합니다.
   - 실제 코드 위치: `src/lib/content-assets.ts::validatePortfolioAssets`
   - 입력과 출력: social/profile/resume/project asset URL과 public root를 filesystem target으로 변환합니다.
   - 실패/absence 처리: root 밖 relative path 또는 missing target을 issue로 누적합니다. 선행 `/` 자체는 별도 reject branch가 아닙니다.
6. Validated source를 portfolio facade와 selectors/routes가 소비합니다.
   - 실제 코드 위치: `src/lib/portfolio/content.ts` 및 selectors/pages
   - 입력과 출력: `portfolioSource`를 canonical input으로 받아 sorted/filtered/derived portfolio aggregate를 만듭니다.
   - 실패/absence 처리: disabled/absent data는 facade/selector policy로 제외되며 raw JSON fallback은 없습니다.
7. `content:check`와 `prebuild`가 동일 경계를 실행해 failure를 build에 전달합니다.
   - 실제 코드 위치: `package.json` → `scripts/validate-content.ts`
   - 입력과 출력: `prebuild`가 content loader와 asset validator를 실행한 뒤에만 Next build로 진행합니다.
   - 실패/absence 처리: throw/non-zero status면 compilation 이전에 build가 종료됩니다.

### 코드 없이 설명하기

> 이 Thread의 최종 실행 흐름을 code snippet 없이 자신의 말로 작성합니다. 설계 → 구현 → failure/risk → 수정/강화 → 검증 순서가 드러나야 합니다.

처음에는 JSON module을 TypeScript 타입으로 취급했지만 runtime 값은 검증되지 않았습니다. 프로젝트 schema와 source-aware error model을 만든 뒤 `parseContentFile`과 `loadPortfolioSource`가 모든 파일을 통과시키는 단일 입력 경계가 됐습니다. Repository-wide duplicate checks가 identity를 결정적으로 만들고 facade migration이 raw import 우회를 없앴습니다. 마지막으로 local asset을 public root에 묶고 같은 loader/validator를 `prebuild`에서 실행해 invalid content가 compilation 전에 실패하도록 했습니다. 다만 이 단계는 publication readiness와 실제 HTTP serving까지는 다루지 않습니다.

> **실행 증거 구분:** 참조 SHA의 diff와 해당 SHA 파일은 GitHub connector로 확인했습니다. 로컬 clone은 실행 환경의 DNS 차단으로 실패해 build, unit/E2E, browser, Lighthouse, Docker command는 실행하지 않았습니다. 따라서 위 test 결과는 구현된 test technique과 assertion의 정적 검토이며 실제 통과 결과가 아닙니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 실제로 checkout하거나 diff로 확인했습니다.
- [x] Thread 내 commit 순서를 source와 동일하게 유지했습니다.
- [x] 각 A/S commit에서 직전 상태, decision, failure boundary, guarantee와 non-guarantee를 구분했습니다.
- [x] B commit은 thread 흐름에서 필요한 구현 역할과 state change를 확인했습니다.
- [x] Fix commit을 독립 feature가 아니라 기존 가정 → failure → root cause → corrected invariant로 설명했습니다.
- [x] Test commit에서 production invariant, failure injection, technique, traversed production path, proves/does-not-prove를 구분했습니다.
- [x] Invariant ledger의 모든 주장에 해당 SHA의 code/test evidence가 있습니다.
- [x] Final HEAD의 code를 과거 commit 설명에 소급하지 않았습니다.
- [x] Thread 최종 흐름을 code 없이 설명할 수 있습니다.
===== END FILE: 01-fail-closed-content-ingestion.md =====

===== BEGIN FILE: 02-full-site-renderer-architecture.md =====
# Thread: Full-site renderer architecture

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 source에서 확정된 thread 구조와 commit 역할만 미리 제공합니다. 실제 구현 해석, code evidence, failure 재현, test 결과, 최종 설명은 해당 SHA의 코드를 직접 확인해 작성합니다.

## 1. Thread 목표

다섯 시각 시스템이 전체 route composition을 독립적으로 소유하면서도 App Router의 loading, query, availability, not-found 책임을 복제하지 않도록 registry와 delegation 경계가 형성되는 과정을 복원합니다.

**Source-defined significance**

> This thread is not merely a sequence of visual additions. It defines the architectural mechanism that lets five designs own complete route composition while App Router pages retain framework responsibilities. Registry metadata, validation, lazy loading, exhaustive route contracts, and final unified dispatch must agree.

### 이 Thread에 직접 연결되는 Critical Invariants

- Content가 광고하는 design은 code registry가 지원하고 lazy loading할 수 있습니다. — `418e7bc1d8bb → 6fc28f4c6586 → c6acfe562694/dd71d28143a8/b8de57f130eb`
- Renderer input은 route discriminator, current path, debug state, detail context를 명시합니다. — `e14202198948`
- App Router page는 framework concern을, renderer는 complete composition을 소유합니다. — `dc2cf72a768d`
- 모든 route와 design은 하나의 registry dispatch path를 사용합니다. — `380b2a025070`

### 연결되는 Major Engineering Difficulty

- 다섯 개의 시각적으로 독립적인 renderer가 모든 route에서 동일 semantic behavior를 유지하도록 framework, selection, relationship logic 중복을 막는 문제
- 이미 구현된 route를 유지하면서 lazy registry와 full-page delegation architecture로 이동하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Design catalog, content validation, lazy loader, route dispatcher가 같은 identifier를 공유한다는 것을 코드에서 어떻게 보장하는가?
- App Router page와 full-site renderer의 책임 분리는 어느 commit에서 시작되고 최종적으로 어디서 예외가 제거되는가?
- Renderer가 URL을 직접 해석하지 않고 resolved route context를 받는 이유는 무엇인가?
- Dedicated renderer가 없는 design은 migration 중 어떤 fallback을 거치며 최종 구조에서 무엇이 사라지는가?
- Editorial, Brutalist, Cinematic 활성화가 단순 UI 추가가 아니라 registry invariant 검증인 이유는 무엇인가?

## 3. 완료 기준

- `SITE_DESIGNS`, supported IDs, presentation registry, lazy loader map, route dispatcher의 연결 지점을 모두 찾았습니다.
- 모든 public page에서 framework concern과 renderer concern을 실제 call graph로 구분했습니다.
- `dc2cf72a768d`의 migration delegation과 `380b2a025070`의 final unified dispatch 차이를 설명할 수 있습니다.
- Advertised-but-unloadable 또는 loadable-but-unvalidated design을 만드는 누락 지점을 열거할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `418e7bc1d8bb` | feat(designs): site design 정의 registry 추가 | A | ARCH, RENDERER | Defines one authoritative design catalog. |
| 2 | `e14202198948` | feat(designs): route renderer 계약 추가 | A | ARCH, ROUTING, RENDERER | Defines the route-discriminated full-site renderer input. |
| 3 | `6fc28f4c6586` | refactor(designs): 확장 renderer lazy registry 추가 | A | ARCH, RENDERER, REFACTOR | Introduces lazy capability detection and loading. |
| 4 | `dc2cf72a768d` | refactor(routes): 확장 디자인 renderer 위임 경계 추가 | S | ARCH, ROUTING, RENDERER | Moves complete page composition behind one route delegation boundary. |
| 5 | `c6acfe562694` | feat(editorial): renderer를 디자인 registry에 활성화 | A | ARCH, RENDERER | Proves that a complete external renderer can be selected safely. |
| 6 | `dd71d28143a8` | feat(designs): Brutalist renderer 활성화 | A | ARCH, RENDERER | Applies the same registry invariant to a second expanded renderer. |
| 7 | `b8de57f130eb` | feat(designs): Cinematic renderer 활성화 | A | ARCH, RENDERER | Completes the five-design registry. |
| 8 | `380b2a025070` | refactor(designs): 모든 route를 registry renderer로 위임 | S | ARCH, ROUTING, RENDERER | Eliminates direct template special cases and establishes one dispatch path. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA를 checkout한 상태에서 작성합니다. Thread 내 이전 commit은 비교 대상으로 사용할 수 있지만 final HEAD를 정답처럼 소급하지 않습니다.

### 1. `418e7bc1d8bb` — feat(designs): site design 정의 registry 추가

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Source-defined thread role:** Defines one authoritative design catalog.
- **Source classification summary:** Established a single registry for the site's available visual designs and their preview palettes.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Available visual design과 preview palette를 ordered `SITE_DESIGNS` registry로 정의하고 validation용 identifier와 deterministic fallback lookup을 파생합니다.

#### 해당 SHA에서 확인할 실제 코드

- `SITE_DESIGNS` element shape, ordering, ID, palette fields를 확인합니다.
- Identifier list가 별도 수동 enum이 아니라 registry에서 파생되는지 확인합니다.
- `getSiteDesignDefinition`이 unknown ID에서 첫 item으로 fallback하는 branch를 추적합니다.
- Design switcher와 validation이 registry의 어떤 field를 소비하는지 caller를 찾습니다.

확인 원칙:

- 먼저 `418e7bc1d8bb^`와 `418e7bc1d8bb`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Authoritative design ordering과 metadata ownership | registry가 machine identity/order/palette를 소유하고 presentation content는 이후 user-facing copy를 소유합니다. |
| Unknown ID의 deterministic fallback | ordered `SITE_DESIGNS`를 authoritative catalog로 만들고 ID list와 fallback lookup을 파생했습니다. |
| 이 commit만으로는 full-route renderer loadability를 보장하지 않는 이유 | 해당 ID의 full-route module이 존재하거나 lazy-load 가능하다는 것은 아직 보장하지 않습니다. |
| 새 design 추가 시 아직 변경해야 하는 다른 boundary | 해당 ID의 full-route module이 존재하거나 lazy-load 가능하다는 것은 아직 보장하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** design identity, display order, palette가 여러 consumer에 흩어질 가능성이 있었습니다.
- **해당 SHA 핵심 코드:** `src/designs/config.ts`의 registry는 이 SHA에서 `design`, `classic`과 palette metadata를 보유하며 `SITE_DESIGN_IDS`, `getSiteDesignDefinition`을 파생합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `e14202198948`가 full-route contract를, `6fc28f4c6586`가 loader registry를 추가합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** design identity, display order, palette가 여러 consumer에 흩어질 가능성이 있었습니다. selector와 validation이 서로 다른 design 집합·순서를 쓰면 advertised state와 code capability가 어긋납니다.
- **구현 결정과 경로:** ordered `SITE_DESIGNS`를 authoritative catalog로 만들고 ID list와 fallback lookup을 파생했습니다. `src/designs/config.ts`의 registry는 이 SHA에서 `design`, `classic`과 palette metadata를 보유하며 `SITE_DESIGN_IDS`, `getSiteDesignDefinition`을 파생합니다.
- **소유권·실패 처리:** registry가 machine identity/order/palette를 소유하고 presentation content는 이후 user-facing copy를 소유합니다. unknown ID는 exception이 아니라 registry 첫 항목으로 deterministic fallback합니다.
- **보장:** catalog 내부의 ID/order/metadata source가 하나로 정리됩니다.
- **보장하지 않는 범위:** 해당 ID의 full-route module이 존재하거나 lazy-load 가능하다는 것은 아직 보장하지 않습니다.
- **후속 연결:** `e14202198948`가 full-route contract를, `6fc28f4c6586`가 loader registry를 추가합니다.

### 2. `e14202198948` — feat(designs): route renderer 계약 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Source-defined thread role:** Defines the route-discriminated full-site renderer input.
- **Source classification summary:** Defined the shared input contract for designs that render complete portfolio routes.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Full-site design renderer가 받을 closed route union과 shared props를 정의합니다. Route kind, content, debug state, current path, optional project를 명시해 renderer가 URL parsing이나 loading을 소유하지 않게 합니다.

#### 해당 SHA에서 확인할 실제 코드

- `PortfolioRouteId`의 모든 route literal과 dynamic project route 표현을 확인합니다.
- `DesignRouteProps` field별 owner가 App Router인지 renderer인지 표시합니다.
- Optional project가 detail route에서만 의미 있다는 제약이 type으로 완전히 강제되는지 확인합니다.
- Renderer가 current path를 받는 이유를 navigation/design switching caller에서 추적합니다.
- 이 초기 contract의 `content` input이 후속 view-model refactor 전 얼마나 broad한지 기록합니다.

확인 원칙:

- 먼저 `e14202198948^`와 `e14202198948`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Route discriminator와 exhaustive switch 기반 | 8개 route literal과 shared renderer props를 closed contract로 정의했습니다. `src/designs/types.ts`의 `PortfolioRouteId`와 `DesignRouteProps`가 `route`, broad `content`, `contentDebug`, `currentPath`, optional `project`를 명시합니다. |
| Props 밖에 남는 framework responsibility | App Router는 loading/query/availability/project lookup을 소유하고 renderer는 전달받은 context로 composition을 소유하도록 경계를 준비합니다. |
| Optional project의 valid/invalid 조합 | `project`가 optional field라 이 SHA의 type만으로는 non-detail route에 project를 주거나 detail route에서 빼는 invalid 조합을 완전히 막지 못합니다. |
| 후속 projection thread에서 좁혀질 broad input | `dc2cf72a768d`가 모든 public page에서 이 request를 실제 delegation에 사용합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** design renderer가 page URL과 framework state를 자체 추론할 수 있는 느슨한 입력 상태였습니다.
- **해당 SHA 핵심 코드:** `src/designs/types.ts`의 `PortfolioRouteId`와 `DesignRouteProps`가 `route`, broad `content`, `contentDebug`, `currentPath`, optional `project`를 명시합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `dc2cf72a768d`가 모든 public page에서 이 request를 실제 delegation에 사용합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** design renderer가 page URL과 framework state를 자체 추론할 수 있는 느슨한 입력 상태였습니다. 각 renderer가 route parsing, debug query, current path, detail lookup을 다시 구현할 위험이 있었습니다.
- **구현 결정과 경로:** 8개 route literal과 shared renderer props를 closed contract로 정의했습니다. `src/designs/types.ts`의 `PortfolioRouteId`와 `DesignRouteProps`가 `route`, broad `content`, `contentDebug`, `currentPath`, optional `project`를 명시합니다.
- **소유권·실패 처리:** App Router는 loading/query/availability/project lookup을 소유하고 renderer는 전달받은 context로 composition을 소유하도록 경계를 준비합니다. `project`가 optional field라 이 SHA의 type만으로는 non-detail route에 project를 주거나 detail route에서 빼는 invalid 조합을 완전히 막지 못합니다.
- **보장:** renderer가 처리해야 할 route vocabulary와 shared context가 명시됩니다.
- **보장하지 않는 범위:** route-model correlation과 narrow payload는 아직 보장하지 않습니다.
- **후속 연결:** `dc2cf72a768d`가 모든 public page에서 이 request를 실제 delegation에 사용합니다.

### 3. `6fc28f4c6586` — refactor(designs): 확장 renderer lazy registry 추가

- **Importance:** A
- **Tags:** ARCH, RENDERER, REFACTOR
- **Source-defined thread role:** Introduces lazy capability detection and loading.
- **Source classification summary:** Introduced a registry boundary for design-specific route renderers.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Design identifier를 asynchronous module loader에 매핑하고 capability detection과 rendering operation을 분리하는 lazy registry boundary를 만듭니다. Loader table은 비어 있어 extension contract만 먼저 확정합니다.

#### 해당 SHA에서 확인할 실제 코드

- Loader map의 key/value type과 dynamic import return contract를 확인합니다.
- Capability check와 actual render function이 분리된 control flow를 추적합니다.
- Unsupported design 또는 loader 부재 시 legacy renderer로 돌아가는 branch를 찾습니다.
- 빈 loader table에서도 existing route가 동작하는 fallback path를 확인합니다.
- Eager import가 제거되거나 아직 존재하는 지점을 bundle boundary 관점에서 기록합니다.

확인 원칙:

- 먼저 `6fc28f4c6586^`와 `6fc28f4c6586`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Registry가 정의하는 extension point | design ID에서 asynchronous module loader로 가는 registry와 capability check를 도입했습니다. `src/designs/registry.tsx`의 partial loader map과 `hasDedicatedRouteRenderer`/`renderDesignRoute`가 detection과 import/render를 분리합니다. 이 SHA의 loader table은 아직 비어 있습니다. |
| Detection과 loading 부재/failure 처리 | loader가 없으면 dedicated render path를 선택하지 않으며 기존 route는 fallback으로 계속 동작합니다. |
| Lazy loading이 보장하는 것과 module completeness를 보장하지 않는 것 | 등록 module의 route completeness나 advertised/validated ID 일치는 아직 보장하지 않습니다. |
| 첫 concrete activation에서 채워져야 할 항목 | design ID에서 asynchronous module loader로 가는 registry와 capability check를 도입했습니다. `src/designs/registry.tsx`의 partial loader map과 `hasDedicatedRouteRenderer`/`renderDesignRoute`가 detection과 import/render를 분리합니다. 이 SHA의 loader table은 아직 비어 있습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** App Router page가 renderer를 직접 import하거나 template conditional을 소유했습니다.
- **해당 SHA 핵심 코드:** `src/designs/registry.tsx`의 partial loader map과 `hasDedicatedRouteRenderer`/`renderDesignRoute`가 detection과 import/render를 분리합니다. 이 SHA의 loader table은 아직 비어 있습니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `dc2cf72a768d`가 page delegation을 연결하고 이후 세 activation commit이 loader table을 채웁니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** App Router page가 renderer를 직접 import하거나 template conditional을 소유했습니다. 새 complete renderer를 추가할 때 page마다 eager import/branch를 복제할 가능성이 있었습니다.
- **구현 결정과 경로:** design ID에서 asynchronous module loader로 가는 registry와 capability check를 도입했습니다. `src/designs/registry.tsx`의 partial loader map과 `hasDedicatedRouteRenderer`/`renderDesignRoute`가 detection과 import/render를 분리합니다. 이 SHA의 loader table은 아직 비어 있습니다.
- **소유권·실패 처리:** registry는 module capability/loading을 소유하고 page는 unsupported design에서 기존 local renderer fallback을 유지합니다. loader가 없으면 dedicated render path를 선택하지 않으며 기존 route는 fallback으로 계속 동작합니다.
- **보장:** lazy module extension point와 unsupported capability 판정이 생깁니다.
- **보장하지 않는 범위:** 등록 module의 route completeness나 advertised/validated ID 일치는 아직 보장하지 않습니다.
- **후속 연결:** `dc2cf72a768d`가 page delegation을 연결하고 이후 세 activation commit이 loader table을 채웁니다.

### 4. `dc2cf72a768d` — refactor(routes): 확장 디자인 renderer 위임 경계 추가

- **Importance:** S
- **Tags:** ARCH, ROUTING, RENDERER
- **Source-defined thread role:** Moves complete page composition behind one route delegation boundary.
- **Source classification summary:** Added a common delegation boundary from every public route to design-specific renderers registered outside the App Router pages.
- **Source classification reason:** Critical because it separates framework responsibilities from full-site presentation across every public route, enabling independent visual systems without duplicating loading, routing, or not-found policy.

#### Source에서 확정된 구현 의도와 상태 변화

모든 public App Router page가 content, template, debug, route validity를 해결한 뒤 compact context를 `renderDesignRoute`에 넘기는 공통 delegation boundary를 도입합니다. Dedicated renderer가 없는 design은 migration 동안 legacy fallback을 사용합니다.

#### S-level source profile

- **Problem:** 각 App Router page 안에 complete visual system을 추가하면 query parsing, loading, route validation, not-found behavior가 design마다 복제될 수 있었습니다.
- **Decision:** Framework concern은 page에 남기고 compact route context를 registered full-page renderer에 위임합니다.
- **Why it mattered:** Expanded design이 alternate routing implementation이 되지 않고 composition만 독립적으로 소유할 수 있게 합니다.
- **What changed:** 모든 public route가 route kind, path, debug state, content, optional project를 `renderDesignRoute`에 전달합니다.

#### 해당 SHA에서 확인할 실제 코드

- Home, projects, project detail, about, resume, contact, journey, interview page에서 새 delegation call을 찾습니다.
- Delegation 전에 page가 수행하는 content loading, query resolution, page enablement, project lookup, `notFound`를 목록화합니다.
- Request에 current path, route kind, debug state, content, optional project가 어떻게 들어가는지 route별로 비교합니다.
- `renderDesignRoute`가 capability를 확인하고 dedicated output 또는 fallback을 선택하는 branch를 추적합니다.
- Project detail에서 unresolved project를 renderer 전에 차단하는지 확인합니다.
- Dedicated renderer가 metadata/static params/not-found를 다시 구현하지 않는다는 module boundary를 확인합니다.
- 직전 template conditionals와 비교해 migration이 아직 완결되지 않은 부분을 표시합니다.

확인 원칙:

- 먼저 `dc2cf72a768d^`와 `dc2cf72a768d`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 page가 composition과 framework concern을 함께 소유한 구조 | 각 App Router page가 framework concern과 complete visual composition을 함께 소유했고 expanded design을 넣으려면 route logic 복제가 필요했습니다. |
| Expanded design마다 route concern을 복제할 위험 | unsupported loader는 local legacy branch로 내려가고 disabled route/unknown project는 page에서 차단됩니다. |
| Page prepares, renderer composes, legacy fallback decision | page가 context를 준비하고 dedicated renderer가 complete composition을 담당하며 기존 Design/Classic은 migration fallback으로 남는 boundary를 만들었습니다. |
| Route request type, `renderDesignRoute`, 각 page call site | page가 context를 준비하고 dedicated renderer가 complete composition을 담당하며 기존 Design/Classic은 migration fallback으로 남는 boundary를 만들었습니다. 8개 `src/app/**/page.tsx`가 `hasDedicatedRouteRenderer`를 먼저 확인하고 `renderDesignRoute`에 route/path/debug/content/project를 전달합니다. Project detail은 위임 전에 lookup 실패를 `notFound()`로 처리합니다. |
| Unsupported loader, missing project, disabled route의 owner | unsupported loader는 local legacy branch로 내려가고 disabled route/unknown project는 page에서 차단됩니다. |
| 보장하는 것: 모든 public route의 동일 delegation entry | 모든 public route에 expanded renderer로 들어가는 동일 delegation entry가 생깁니다. |
| 아직 보장하지 않는 것: original templates의 direct special case 제거 | Design/Classic direct imports와 template conditionals가 남아 있어 단일 dispatch architecture는 아직 아닙니다. |
| 후속 activation과 `380b2a025070` 연결 | `c6acfe562694`, `dd71d28143a8`, `b8de57f130eb`가 concrete modules를 활성화하고 `380b2a025070`이 fallback 특례를 제거합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 각 App Router page가 framework concern과 complete visual composition을 함께 소유했고 expanded design을 넣으려면 route logic 복제가 필요했습니다.
- **해당 SHA 핵심 코드:** 8개 `src/app/**/page.tsx`가 `hasDedicatedRouteRenderer`를 먼저 확인하고 `renderDesignRoute`에 route/path/debug/content/project를 전달합니다. Project detail은 위임 전에 lookup 실패를 `notFound()`로 처리합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `c6acfe562694`, `dd71d28143a8`, `b8de57f130eb`가 concrete modules를 활성화하고 `380b2a025070`이 fallback 특례를 제거합니다.

#### SHA별 복원 결론

- **이전 상태:** 각 App Router page가 framework concern과 complete visual composition을 함께 소유했고 expanded design을 넣으려면 route logic 복제가 필요했습니다.
- **문제와 위험:** design마다 query parsing, page availability, project lookup, `notFound`까지 재구현하면 semantic route behavior가 갈라집니다.
- **핵심 결정:** page가 context를 준비하고 dedicated renderer가 complete composition을 담당하며 기존 Design/Classic은 migration fallback으로 남는 boundary를 만들었습니다.
- **실제 구현 경로:** 8개 `src/app/**/page.tsx`가 `hasDedicatedRouteRenderer`를 먼저 확인하고 `renderDesignRoute`에 route/path/debug/content/project를 전달합니다. Project detail은 위임 전에 lookup 실패를 `notFound()`로 처리합니다.
- **소유권·상태 변화:** page는 content/query/availability/not-found를, registry는 capability/module selection을, design route renderer는 shell과 page composition을 소유합니다.
- **실패 경계:** unsupported loader는 local legacy branch로 내려가고 disabled route/unknown project는 page에서 차단됩니다.
- **보장:** 모든 public route에 expanded renderer로 들어가는 동일 delegation entry가 생깁니다.
- **보장하지 않는 범위:** Design/Classic direct imports와 template conditionals가 남아 있어 단일 dispatch architecture는 아직 아닙니다.
- **후속 fix/test:** `c6acfe562694`, `dd71d28143a8`, `b8de57f130eb`가 concrete modules를 활성화하고 `380b2a025070`이 fallback 특례를 제거합니다.

### 5. `c6acfe562694` — feat(editorial): renderer를 디자인 registry에 활성화

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Source-defined thread role:** Proves that a complete external renderer can be selected safely.
- **Source classification summary:** Promote the completed Editorial renderer into the site's selectable design contract.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Completed Editorial renderer를 selectable design contract에 등록하고 metadata, swatch, lazy loader, public export, validation acceptance를 함께 갱신하며 default template로 설정합니다.

#### 해당 SHA에서 확인할 실제 코드

- Editorial ID가 `SITE_DESIGNS`, presentation templates, supported IDs, loader map에 동일하게 추가되는지 대조합니다.
- Editorial module의 public entry가 exhaustive route dispatcher 하나인지 확인합니다.
- Default design 변경이 query resolver와 selector fallback에 미치는 caller를 추적합니다.
- 한 registration 지점을 빠뜨렸을 때 예상되는 compile/runtime/content validation failure를 기록합니다.
- Lazy import가 route module을 언제 load하는지 확인합니다.

확인 원칙:

- 먼저 `c6acfe562694^`와 `c6acfe562694`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Advertised/supported/loadable/renderable 네 조건의 source locations | `src/designs/config.ts`, `src/content/presentation.json`, `src/lib/content-loader.ts`, `src/designs/registry.tsx`, Editorial module entry가 같은 `editorial` ID를 공유합니다. |
| Editorial complete route set의 dispatcher 증거 | Editorial을 metadata, presentation, schema-supported ID, lazy loader, public route entry 전부에 동시에 등록하고 default로 선택했습니다. `src/designs/config.ts`, `src/content/presentation.json`, `src/lib/content-loader.ts`, `src/designs/registry.tsx`, Editorial module entry가 같은 `editorial` ID를 공유합니다. |
| Default 변경과 explicit query selection 관계 | `src/designs/config.ts`, `src/content/presentation.json`, `src/lib/content-loader.ts`, `src/designs/registry.tsx`, Editorial module entry가 같은 `editorial` ID를 공유합니다. |
| Registry invariant를 처음 실제로 증명한 지점 | Editorial이 advertised·validated·loadable·renderable인 complete selectable renderer가 됩니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** lazy registry와 delegation은 있었지만 실제 selectable expanded renderer가 등록되지 않았습니다.
- **해당 SHA 핵심 코드:** `src/designs/config.ts`, `src/content/presentation.json`, `src/lib/content-loader.ts`, `src/designs/registry.tsx`, Editorial module entry가 같은 `editorial` ID를 공유합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `dd71d28143a8`과 `b8de57f130eb`이 같은 checklist를 재사용합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** lazy registry와 delegation은 있었지만 실제 selectable expanded renderer가 등록되지 않았습니다. presentation에서 design을 광고하면서 loader/schema/config 중 하나가 빠지면 선택은 가능하지만 render/validation은 실패할 수 있습니다.
- **구현 결정과 경로:** Editorial을 metadata, presentation, schema-supported ID, lazy loader, public route entry 전부에 동시에 등록하고 default로 선택했습니다. `src/designs/config.ts`, `src/content/presentation.json`, `src/lib/content-loader.ts`, `src/designs/registry.tsx`, Editorial module entry가 같은 `editorial` ID를 공유합니다.
- **소유권·실패 처리:** catalog는 identity/palette, content는 copy/default, registry는 loading, `EditorialRoute`는 exhaustive route composition을 소유합니다. 등록 지점 누락은 validation reject, unavailable loader, missing copy 또는 selection fallback 중 하나로 드러납니다.
- **보장:** Editorial이 advertised·validated·loadable·renderable인 complete selectable renderer가 됩니다.
- **보장하지 않는 범위:** original Design/Classic의 별도 path와 다른 expanded renderer의 활성화는 남아 있습니다.
- **후속 연결:** `dd71d28143a8`과 `b8de57f130eb`이 같은 checklist를 재사용합니다.

### 6. `dd71d28143a8` — feat(designs): Brutalist renderer 활성화

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Source-defined thread role:** Applies the same registry invariant to a second expanded renderer.
- **Source classification summary:** Activate Brutalist across every registry boundary required for a selectable renderer.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Brutalist renderer를 metadata, module entry, palette, lazy loader, content-loader support까지 모든 registry boundary에 활성화합니다.

#### 해당 SHA에서 확인할 실제 코드

- Brutalist ID가 각 registry/validation/presentation 위치에 동일하게 등록되는지 Editorial과 비교합니다.
- `BrutalistRoute`가 route switch 뒤 shared shell을 적용하는 구조를 확인합니다.
- Internal helper가 module-private이고 public API가 renderer entry로 제한되는지 확인합니다.
- Design switcher가 ordered registry에서 Brutalist position/palette를 얻는 경로를 추적합니다.

확인 원칙:

- 먼저 `dd71d28143a8^`와 `dd71d28143a8`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 두 번째 expanded renderer에도 같은 activation checklist가 적용된 증거 | Brutalist를 config/presentation/validation/lazy loader/module entry에 동일하게 등록했습니다. `BrutalistRoute` public entry가 route switch와 shared shell을 소유하고 private helpers는 module 밖으로 노출하지 않습니다. |
| Complete route coverage와 shell uniformity | `BrutalistRoute` public entry가 route switch와 shared shell을 소유하고 private helpers는 module 밖으로 노출하지 않습니다. |
| Registration 누락이 만들 수 있는 불일치 | 어느 registration boundary든 누락되면 selector·schema·loader 중 한 단계에서 ID 불일치가 발생합니다. |
| 다른 visual language와 같은 semantic contract의 경계 | registry가 module 선택을, Brutalist route entry가 route-specific composition과 shell uniformity를 소유합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** Editorial만 expanded registry invariant를 실제로 만족했습니다.
- **해당 SHA 핵심 코드:** `BrutalistRoute` public entry가 route switch와 shared shell을 소유하고 private helpers는 module 밖으로 노출하지 않습니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `b8de57f130eb`이 마지막 expanded renderer를 등록합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** Editorial만 expanded registry invariant를 실제로 만족했습니다. 두 번째 renderer에서 같은 activation checklist가 유지되지 않으면 architecture가 일회성 integration에 머뭅니다.
- **구현 결정과 경로:** Brutalist를 config/presentation/validation/lazy loader/module entry에 동일하게 등록했습니다. `BrutalistRoute` public entry가 route switch와 shared shell을 소유하고 private helpers는 module 밖으로 노출하지 않습니다.
- **소유권·실패 처리:** registry가 module 선택을, Brutalist route entry가 route-specific composition과 shell uniformity를 소유합니다. 어느 registration boundary든 누락되면 selector·schema·loader 중 한 단계에서 ID 불일치가 발생합니다.
- **보장:** 서로 다른 visual language가 동일 route contract로 확장될 수 있음을 두 번째 사례로 확인합니다.
- **보장하지 않는 범위:** five-design completeness와 unified dispatch는 아직 아닙니다.
- **후속 연결:** `b8de57f130eb`이 마지막 expanded renderer를 등록합니다.

### 7. `b8de57f130eb` — feat(designs): Cinematic renderer 활성화

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Source-defined thread role:** Completes the five-design registry.
- **Source classification summary:** Activate Cinematic as a complete selectable renderer and narrow its module API to the route entry point.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Cinematic renderer를 complete selectable renderer로 활성화하고 module API를 `CinematicRoute` entry point로 제한해 five-design registry를 완성합니다.

#### 해당 SHA에서 확인할 실제 코드

- Cinematic ID의 metadata, palette, presentation, loader, content validation 등록을 모두 확인합니다.
- `CinematicRoute`가 모든 supported route를 shared frame 안에서 dispatch하는 exhaustive switch인지 확인합니다.
- Module export에서 incidental helper가 노출되지 않는지 확인합니다.
- Registry가 정확히 다섯 design을 포함한다는 derived IDs와 selector rendering을 기록합니다.
- 다섯 design이 동일 current path/debug propagation contract를 받는지 확인합니다.

확인 원칙:

- 먼저 `b8de57f130eb^`와 `b8de57f130eb`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Five-design completeness의 source-of-truth | `CinematicRoute`가 shared Frame 안에서 8개 route를 exhaustive switch하고 config/presentation/loader/validation이 같은 `cinematic` ID를 사용합니다. |
| Cinematic activation이 architecture 수정 없이 확장되는 방식 | Cinematic을 모든 registry boundary에 등록하고 public API를 `CinematicRoute` 하나로 좁혔습니다. `CinematicRoute`가 shared Frame 안에서 8개 route를 exhaustive switch하고 config/presentation/loader/validation이 같은 `cinematic` ID를 사용합니다. |
| Route 누락을 compiler/exhaustive switch가 드러내는지 여부 | 지원 route 누락은 exhaustive switch/type 수준에서 드러날 기반이 있고 invalid query는 page resolver에서 fallback합니다. |
| Final unified dispatch 전에 남은 original template special case | Cinematic을 모든 registry boundary에 등록하고 public API를 `CinematicRoute` 하나로 좁혔습니다. `CinematicRoute`가 shared Frame 안에서 8개 route를 exhaustive switch하고 config/presentation/loader/validation이 같은 `cinematic` ID를 사용합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** Cinematic이 완성되어도 registry의 advertised/supported/loadable contract에 포함되지 않았습니다.
- **해당 SHA 핵심 코드:** `CinematicRoute`가 shared Frame 안에서 8개 route를 exhaustive switch하고 config/presentation/loader/validation이 같은 `cinematic` ID를 사용합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `380b2a025070`이 다섯 design을 하나의 registry dispatch로 통합합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** Cinematic이 완성되어도 registry의 advertised/supported/loadable contract에 포함되지 않았습니다. module 구현과 selection contract가 분리되면 code는 존재하지만 사용 불가능한 renderer가 됩니다.
- **구현 결정과 경로:** Cinematic을 모든 registry boundary에 등록하고 public API를 `CinematicRoute` 하나로 좁혔습니다. `CinematicRoute`가 shared Frame 안에서 8개 route를 exhaustive switch하고 config/presentation/loader/validation이 같은 `cinematic` ID를 사용합니다.
- **소유권·실패 처리:** module entry가 route dispatch를 소유하고 incidental helper는 private으로 남습니다. 지원 route 누락은 exhaustive switch/type 수준에서 드러날 기반이 있고 invalid query는 page resolver에서 fallback합니다.
- **보장:** registry는 Design, Classic, Editorial, Brutalist, Cinematic 다섯 design의 selection metadata와 modules를 갖습니다.
- **보장하지 않는 범위:** App Router page에는 original template의 direct branches가 아직 남아 있습니다.
- **후속 연결:** `380b2a025070`이 다섯 design을 하나의 registry dispatch로 통합합니다.

### 8. `380b2a025070` — refactor(designs): 모든 route를 registry renderer로 위임

- **Importance:** S
- **Tags:** ARCH, ROUTING, RENDERER
- **Source-defined thread role:** Eliminates direct template special cases and establishes one dispatch path.
- **Source classification summary:** Route every application page through the design registry after resolving its template and building the corresponding view model.
- **Source classification reason:** Critical because it completes one dispatch architecture for all routes and designs, leaving App Router pages with framework concerns and renderers with presentation ownership.

#### Source에서 확정된 구현 의도와 상태 변화

모든 App Router page가 template resolution과 route view-model construction 후 design registry를 호출하도록 바꾸고 Classic/Design direct imports와 special-case selection을 제거합니다. Project detail의 `notFound`와 structured-data framing은 page에 남깁니다.

#### S-level source profile

- **Problem:** Expanded design은 registry를 사용했지만 Classic/Design은 direct import를 유지해 두 dispatch architecture가 공존했습니다.
- **Decision:** 모든 page가 view model을 만든 뒤 같은 registry를 호출하도록 합니다.
- **Why it mattered:** 모든 design과 route에 하나의 integration path를 제공하고 original template의 예외를 제거합니다.
- **What changed:** Direct template branch를 제거하고 discriminated projection을 registry에 전달합니다.

#### 해당 SHA에서 확인할 실제 코드

- 모든 public page에서 direct Classic/Design renderer import가 제거됐는지 검색합니다.
- Page별로 route view model을 만든 뒤 동일 registry function을 호출하는 순서를 확인합니다.
- Registry가 original Design/Classic까지 포함해 하나의 entry point로 dispatch하는 구현을 추적합니다.
- Project detail에서 project existence, `notFound`, metadata/JSON-LD wrapper가 renderer 밖에 남는 이유를 확인합니다.
- Registry design switch와 renderer route switch의 두 단계를 구분합니다.
- Legacy compatibility request 또는 fallback branch가 제거됐는지 확인합니다.
- Invalid/default template query가 registry selection 전에 해결되는지 확인합니다.

확인 원칙:

- 먼저 `380b2a025070^`와 `380b2a025070`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 expanded registry와 original direct branch의 이중 architecture | expanded designs는 registry를 사용하지만 Design/Classic은 page의 direct import/branch를 사용해 두 dispatch architecture가 공존했습니다. |
| Template class에 따라 route behavior/verification가 달라진 문제 | template class에 따라 integration path와 검증 범위가 달라져 original design이 privileged special case가 됐습니다. |
| All pages build projection then one registry dispatch decision | 모든 page가 route view model을 만든 뒤 동일 `renderDesignRoute` registry를 무조건 호출하도록 바꿨습니다. |
| Page calls, registry selection, renderer route dispatch | 모든 page가 route view model을 만든 뒤 동일 `renderDesignRoute` registry를 무조건 호출하도록 바꿨습니다. `src/designs/registry.tsx`는 다섯 ID의 complete `Record` loader/renderer entry를 가지며 8개 page에서 Design/Classic direct imports와 capability branch가 제거됩니다. Renderer 내부에서 design 선택 후 route discriminant로 두 단계 dispatch합니다. |
| App Router, registry, renderer, shell responsibility split | App Router는 lookup/not-found/metadata/JSON-LD를, registry는 design selection을, renderer는 route composition/shell을 소유합니다. |
| Invalid design, disabled route, unknown project failure handling | invalid design query는 registry 이전 resolver의 deterministic fallback으로 정리되고 disabled route/unknown project는 page에서 실패합니다. Project structured data는 page에 남습니다. |
| 보장하는 것: privileged template 없는 단일 path | 모든 public route와 다섯 design이 privileged template 없이 하나의 dispatch path를 사용합니다. |
| Route model input 제한 thread와의 연결 | 다음 thread의 `f8b0ab7b08aa`와 `5897b4b024da`가 registry input과 runtime payload를 좁힙니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** expanded designs는 registry를 사용하지만 Design/Classic은 page의 direct import/branch를 사용해 두 dispatch architecture가 공존했습니다.
- **해당 SHA 핵심 코드:** `src/designs/registry.tsx`는 다섯 ID의 complete `Record` loader/renderer entry를 가지며 8개 page에서 Design/Classic direct imports와 capability branch가 제거됩니다. Renderer 내부에서 design 선택 후 route discriminant로 두 단계 dispatch합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** 다음 thread의 `f8b0ab7b08aa`와 `5897b4b024da`가 registry input과 runtime payload를 좁힙니다.

#### SHA별 복원 결론

- **이전 상태:** expanded designs는 registry를 사용하지만 Design/Classic은 page의 direct import/branch를 사용해 두 dispatch architecture가 공존했습니다.
- **문제와 위험:** template class에 따라 integration path와 검증 범위가 달라져 original design이 privileged special case가 됐습니다.
- **핵심 결정:** 모든 page가 route view model을 만든 뒤 동일 `renderDesignRoute` registry를 무조건 호출하도록 바꿨습니다.
- **실제 구현 경로:** `src/designs/registry.tsx`는 다섯 ID의 complete `Record` loader/renderer entry를 가지며 8개 page에서 Design/Classic direct imports와 capability branch가 제거됩니다. Renderer 내부에서 design 선택 후 route discriminant로 두 단계 dispatch합니다.
- **소유권·상태 변화:** App Router는 lookup/not-found/metadata/JSON-LD를, registry는 design selection을, renderer는 route composition/shell을 소유합니다.
- **실패 경계:** invalid design query는 registry 이전 resolver의 deterministic fallback으로 정리되고 disabled route/unknown project는 page에서 실패합니다. Project structured data는 page에 남습니다.
- **보장:** 모든 public route와 다섯 design이 privileged template 없이 하나의 dispatch path를 사용합니다.
- **보장하지 않는 범위:** 이 시점의 renderer input은 여전히 broad content일 수 있어 semantic data ownership 최소성은 보장하지 않습니다.
- **후속 fix/test:** 다음 thread의 `f8b0ab7b08aa`와 `5897b4b024da`가 registry input과 runtime payload를 좁힙니다.

## 6. Invariant ledger

| Invariant | Source에서 확인된 변화 지점 | 해당 SHA의 실제 code/test evidence | 부족함이 드러난 시점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Content가 광고하는 design은 code registry가 지원하고 lazy loading할 수 있습니다. | `418e7bc1d8bb → 6fc28f4c6586 → c6acfe562694/dd71d28143a8/b8de57f130eb` | `SITE_DESIGNS`/derived IDs, partial lazy loader registry, Editorial·Brutalist·Cinematic activation diff에서 같은 identifiers가 config/presentation/validation/module loader에 등록됩니다. | `418e7bc1d8bb`에서는 catalog만, `6fc28f4c6586`에서는 빈 extension point만 존재했습니다. | `380b2a025070` 시점에는 다섯 advertised designs 모두 complete registry entry와 route dispatcher를 가집니다. |
| Renderer input은 route discriminator, current path, debug state, detail context를 명시합니다. | `e14202198948` | `e14202198948::PortfolioRouteId`와 renderer props의 `route/currentPath/contentDebug/project` fields. | optional project는 route-correlated union이 아니며 broad `content`도 남았습니다. | framework가 resolved route context를 명시적으로 전달한다는 contract가 생겼고 payload correlation/minimization은 다음 thread에서 강화됩니다. |
| App Router page는 framework concern을, renderer는 complete composition을 소유합니다. | `dc2cf72a768d` | `dc2cf72a768d`의 8개 page delegation: page에서 loading/query/availability/project lookup/`notFound`, renderer에서 shell/composition. | 이 commit에는 Design/Classic local branch가 migration fallback으로 남았습니다. | framework concern은 App Router page에 유지되고 complete visual system은 external route renderer가 소유합니다. |
| 모든 route와 design은 하나의 registry dispatch path를 사용합니다. | `380b2a025070` | `380b2a025070`에서 8개 page의 direct Design/Classic imports와 capability branch가 제거되고 complete registry를 무조건 호출합니다. | 직전에는 original templates와 expanded designs가 서로 다른 dispatch path를 사용했습니다. | 모든 route와 five designs가 registry design selection → renderer route switch의 동일 two-stage path를 사용합니다. |

Ledger 작성 원칙:

- Source가 명시한 invariant만 사용하고 새 invariant를 확정 사실처럼 추가하지 않습니다.
- 도입, 강화, 부족함 노출, fix, regression test가 서로 다른 commit이면 각 열에 분리해 기록합니다.
- Code evidence에는 실제 field, function, branch, selector, command 또는 assertion을 적습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 위험 | 대응 commit | 실제 수정/강화 code에서 확인할 것 | Test 또는 실행 증거 |
| --- | --- | --- | --- |
| Design ID는 selector에 보이지만 loader나 validation이 모름 | c6acfe562694, dd71d28143a8, b8de57f130eb | Metadata, swatch, schema acceptance, module export, loader map을 같은 SHA에서 대조합니다. | 각 activation에서 full-route entry point가 실제 등록되는지 기록합니다. |
| 각 page가 design별 분기와 route concern을 중복 구현함 | dc2cf72a768d | `renderDesignRoute` 전후로 page가 유지하는 책임과 넘기는 context를 구분합니다. | Legacy fallback이 어떤 design에 남아 있는지 기록합니다. |
| Classic/Design만 direct import되어 두 dispatch 체계가 공존함 | 380b2a025070 | Direct template branch 제거와 모든 page의 registry request 생성을 확인합니다. | Project detail의 `notFound`와 structured data가 page에 남는 이유를 설명합니다. |

### 실제 연결 기록

- Advertised/supported/loadable/renderable 불일치는 각 activation commit에서 config, presentation, loader, content support와 module entry를 같은 SHA에서 대조해 닫았습니다.
- `dc2cf72a768d`은 migration boundary이며 original templates의 direct fallback을 의도적으로 유지합니다. `380b2a025070`이 그 특례를 제거한 commit입니다.
- Unknown project/disabled route는 renderer 내부 fallback이 아니라 App Router page의 `notFound` owner에 남습니다.

## 8. Ownership / state / responsibility 변화

| Concern | Thread 초기 owner/state | Thread 최종 owner/state | 실제 symbol과 호출 경로 |
| --- | --- | --- | --- |
| Design 목록과 순서 | 분산 상수 가능성 | `SITE_DESIGNS`와 파생 identifier | `src/designs/config.ts::SITE_DESIGNS` → derived IDs/lookup → switcher/validation callers. |
| Route shape | Renderer가 URL/page를 추론 | `PortfolioRouteId`와 renderer props | `src/designs/types.ts::PortfolioRouteId`와 request props; page가 값들을 준비하고 renderer가 discriminator를 소비합니다. |
| Module loading | Static 또는 page별 import | Lazy renderer registry | `src/designs/registry.tsx`의 loader record와 dynamic imports가 module loading을 소유합니다. |
| Framework concern | Design 구현에 섞일 위험 | App Router page | `src/app/**/page.tsx`: content/query/availability/project lookup/`notFound`/metadata framing. |
| Complete composition | Page 내부 template conditional | Design-specific route renderer | 각 design의 `*Route` entry: exhaustive route switch, shared shell과 full composition. |
| Selection | Original/expanded design별 별도 경로 | 단일 registry dispatch | 최종 page → route view-model factory → `renderDesignRoute` → selected design route entry. |

## 9. Thread 최종 상태

### Source에서 확정된 최종 상태

다섯 시각 시스템이 전체 route composition을 독립적으로 소유하면서도 App Router의 loading, query, availability, not-found 책임을 복제하지 않도록 registry와 delegation 경계가 형성되는 과정을 복원합니다.

### 학습자가 완성할 최종 설명

- **Thread 시작 시점의 설계와 위험:** 초기에는 design catalog와 page-local templates가 있었지만 complete alternate renderer를 추가하면 framework route logic까지 복제할 위험이 있었습니다.
- **핵심 architecture/decision이 형성된 순서:** authoritative catalog, route input contract, lazy registry, all-page delegation, 세 expanded renderer activation, original template 통합 순서로 architecture가 완성됐습니다.
- **실제 failure 또는 부족함이 드러난 지점:** `dc2cf72a768d` 이후에도 Design/Classic direct branch가 남아 두 dispatch 체계가 공존한 것이 핵심 미완성 지점이었습니다.
- **Fix 또는 boundary 강화가 바꾼 invariant:** `380b2a025070`이 direct special cases를 제거해 page는 framework concern만, registry/renderer는 design과 route composition만 소유하도록 강화했습니다.
- **Test/build/browser evidence가 보장한 범위:** 각 activation SHA의 changed files와 final page/registry call graph를 정적 검토했습니다. 실제 browser render matrix는 이 thread의 해당 commit에서 실행하지 않았습니다.
- **Thread 종료 시점에도 보장하지 않는 범위:** route payload의 semantic 최소성, visual equality, runtime performance와 accessibility는 이 architecture 자체가 보장하지 않습니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 source-backed flow의 각 단계에 실제 file path, symbol, input/output, failure branch를 추가합니다.

1. App Router page가 content와 query state를 해결합니다.
   - 실제 코드 위치: `src/app/**/page.tsx`와 portfolio content/query helpers
   - 입력과 출력: validated content와 search params에서 selected design, debug state, current path를 계산합니다.
   - 실패/absence 처리: invalid design은 deterministic fallback, malformed/disabled state는 page/helper policy로 정리됩니다.
2. Page enablement와 project existence를 검사하고 필요한 `notFound`를 처리합니다.
   - 실제 코드 위치: 각 public page의 availability/project lookup branch
   - 입력과 출력: optional page enablement와 dynamic project ID를 확인합니다.
   - 실패/absence 처리: disabled page나 unknown project는 renderer 호출 전 `notFound()` 처리합니다.
3. Route kind, path, debug state, detail context를 renderer request로 구성합니다.
   - 실제 코드 위치: route view-model factory 및 registry request construction
   - 입력과 출력: route literal, current path, debug state와 route-specific detail context를 묶습니다.
   - 실패/absence 처리: required detail context가 없으면 page/model factory에서 absence가 드러납니다.
4. Design registry가 identifier의 capability와 lazy module을 선택합니다.
   - 실제 코드 위치: `src/designs/registry.tsx::renderDesignRoute`
   - 입력과 출력: selected design ID로 five-design registry entry를 고르고 lazy module/renderer를 호출합니다.
   - 실패/absence 처리: final structure에는 advertised design의 null fallback이 없고 invalid ID는 upstream resolver가 처리합니다.
5. 선택된 design의 exhaustive route dispatcher가 전용 route renderer를 호출합니다.
   - 실제 코드 위치: 각 `DesignRoute`/`ClassicRoute`/`EditorialRoute`/`BrutalistRoute`/`CinematicRoute`
   - 입력과 출력: discriminated route request를 exhaustive switch해 exact route component로 전달합니다.
   - 실패/absence 처리: unsupported route literal은 closed union/exhaustive switch 수준에서 드러납니다.
6. Design shell과 route composition을 렌더링하고 metadata/JSON-LD는 page 경계에 유지합니다.
   - 실제 코드 위치: App Router page wrapper + selected design shell
   - 입력과 출력: renderer output을 반환하고 project JSON-LD/metadata/static params 같은 framework framing은 page에 유지합니다.
   - 실패/absence 처리: rendering exception은 normal server render failure로 전파되며 renderer가 `notFound` policy를 재구현하지 않습니다.

### 코드 없이 설명하기

> 이 Thread의 최종 실행 흐름을 code snippet 없이 자신의 말로 작성합니다. 설계 → 구현 → failure/risk → 수정/강화 → 검증 순서가 드러나야 합니다.

design 목록과 route input contract를 먼저 고정한 뒤 lazy registry를 만들었습니다. 모든 App Router page가 framework 상태를 해결하고 dedicated renderer에 complete composition을 넘기는 migration boundary를 도입했으며, Editorial·Brutalist·Cinematic을 같은 registration checklist로 활성화했습니다. 마지막에는 Design과 Classic의 direct page branches까지 registry로 이동해 다섯 design 모두 동일한 design dispatch와 route dispatch를 사용합니다. 따라서 renderer는 시각 시스템을 독립적으로 구성하지만 query, availability, project lookup과 not-found를 다시 구현하지 않습니다.

> **실행 증거 구분:** 참조 SHA의 diff와 해당 SHA 파일은 GitHub connector로 확인했습니다. 로컬 clone은 실행 환경의 DNS 차단으로 실패해 build, unit/E2E, browser, Lighthouse, Docker command는 실행하지 않았습니다. 따라서 위 test 결과는 구현된 test technique과 assertion의 정적 검토이며 실제 통과 결과가 아닙니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 실제로 checkout하거나 diff로 확인했습니다.
- [x] Thread 내 commit 순서를 source와 동일하게 유지했습니다.
- [x] 각 A/S commit에서 직전 상태, decision, failure boundary, guarantee와 non-guarantee를 구분했습니다.
- [x] B commit은 thread 흐름에서 필요한 구현 역할과 state change를 확인했습니다.
- [x] Fix commit을 독립 feature가 아니라 기존 가정 → failure → root cause → corrected invariant로 설명했습니다.
- [x] Test commit에서 production invariant, failure injection, technique, traversed production path, proves/does-not-prove를 구분했습니다.
- [x] Invariant ledger의 모든 주장에 해당 SHA의 code/test evidence가 있습니다.
- [x] Final HEAD의 code를 과거 commit 설명에 소급하지 않았습니다.
- [x] Thread 최종 흐름을 code 없이 설명할 수 있습니다.
===== END FILE: 02-full-site-renderer-architecture.md =====

===== BEGIN FILE: 03-route-projections-and-renderer-data-ownership.md =====
# Thread: Route projections and renderer data ownership

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 source에서 확정된 thread 구조와 commit 역할만 미리 제공합니다. 실제 구현 해석, code evidence, failure 재현, test 결과, 최종 설명은 해당 SHA의 코드를 직접 확인해 작성합니다.

## 1. Thread 목표

Renderer가 전체 `PortfolioContent`를 받아 관계를 다시 해석하던 상태에서 route별 view model이 join, ordering, fallback, missing-reference policy를 한 번만 해결하고 필요한 데이터만 노출하는 상태로 발전하는 과정을 복원합니다.

**Source-defined significance**

> The initial multi-design system still allowed renderers to reinterpret a broad content graph. Route projections progressively centralize joins, ordering, fallbacks, and time-dependent values. The final type and runtime restrictions make route ownership enforceable rather than conventional.

### 이 Thread에 직접 연결되는 Critical Invariants

- Cross-content join, ordering, fallback은 route projection이 한 번만 수행합니다. — `4d6b4e6d564e → 44e3b80b297f → d4ad7ecd0d08`
- Route literal은 정확한 view-model variant와 상관관계를 가집니다. — `d7eaa1ac401d`
- Journey와 interview renderer는 raw project collection을 조회하지 않습니다. — `bc0a718e2052 → 98f07b1c211d`
- Registry renderer의 유일한 입력은 discriminated route view model입니다. — `f8b0ab7b08aa`
- Common route base는 site/profile/presentation/footerLinks로 제한됩니다. — `5897b4b024da`
- 허용 field와 missing-reference policy는 runtime regression test로 고정됩니다. — `527b9f872333`

### 연결되는 Major Engineering Difficulty

- Broad portfolio object를 scoped route projection으로 이동하면서 renderer behavior를 보존하는 문제
- Cross-reference resolution과 missing-reference policy를 design마다 재구현하지 않도록 한 owner로 제한하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Home, projects, detail, about, journey, interview route마다 projection이 소유하는 derived value와 cross-reference는 무엇인가?
- Unknown reference를 omit하는 경우와 `null`로 보존하는 경우는 어떻게 구분되는가?
- Route literal과 view-model variant의 상관관계가 TypeScript와 registry에서 어떻게 유지되는가?
- Type-level `never`와 runtime object shape가 각각 어떤 우회를 막는가?
- Renderer가 raw project collection을 다시 얻지 못한다는 사실을 test가 어떻게 검증하는가?

## 3. 완료 기준

- 각 route projection의 input, output, derived fields, missing-reference policy를 표로 정리했습니다.
- `f8b0ab7b08aa`의 registry input 제한과 `5897b4b024da`의 actual runtime payload 축소 차이를 설명할 수 있습니다.
- Renderer에서 제거된 lookup/filter/grouping code와 새 owner인 factory를 직전 관련 SHA와 비교했습니다.
- `527b9f872333`이 type declaration뿐 아니라 runtime field set과 incomplete reference behavior를 고정하는 방식을 기록했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4d6b4e6d564e` | refactor(content): 홈 route view model 경계 추가 | A | ARCH, CONTENT, ROUTING | Introduces centralized presentation-ready derivation and the route discriminant. |
| 2 | `44e3b80b297f` | refactor(content): 프로젝트 목록 파생 모델 추가 | A | ARCH, CONTENT, REFACTOR | Moves project partitioning and fallback grouping into content projection. |
| 3 | `d4ad7ecd0d08` | refactor(content): 상세와 소개 파생 모델 추가 | A | ARCH, CONTENT, REFACTOR | Resolves project and curation relationships before rendering. |
| 4 | `d7eaa1ac401d` | refactor(routes): renderer view model 요청 타입 추가 | A | ARCH, ROUTING, RENDERER | Correlates each route literal with its exact projection. |
| 5 | `bc0a718e2052` | refactor(content): 여정 근거 view model 추가 | A | ARCH, CONTENT, REFACTOR | Removes raw project lookup from journey renderers. |
| 6 | `98f07b1c211d` | refactor(content): 인터뷰 근거 view model 추가 | A | ARCH, CONTENT, REFACTOR | Preserves unresolved evidence explicitly while resolving valid answers once. |
| 7 | `f8b0ab7b08aa` | refactor(designs): renderer 입력을 route view model로 제한 | S | ARCH, ROUTING, RENDERER | Makes projected route data the only registry input. |
| 8 | `5897b4b024da` | refactor(content): route view model 공용 경계 제한 | S | ARCH, CONTENT, ROUTING | Removes the global content spread and finalizes a small shared base. |
| 9 | `527b9f872333` | test(content): scoped view model과 연락처 회귀 검증 | A | CONTENT, VALIDATION, TEST | Verifies allowed runtime fields and missing-reference behavior. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA를 checkout한 상태에서 작성합니다. Thread 내 이전 commit은 비교 대상으로 사용할 수 있지만 final HEAD를 정답처럼 소급하지 않습니다.

### 1. `4d6b4e6d564e` — refactor(content): 홈 route view model 경계 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, ROUTING
- **Source-defined thread role:** Introduces centralized presentation-ready derivation and the route discriminant.
- **Source classification summary:** Introduce a dedicated home-route view model that computes presentation-ready selections once at the content boundary.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Home route 전용 view model이 featured fallback, lead project, metrics, current year, recent journey, placed links, preferred contacts를 content boundary에서 한 번 계산합니다. Route discriminant와 unrelated collection 제거도 시작합니다.

#### 해당 SHA에서 확인할 실제 코드

- Home view-model type, factory input/output, route discriminant를 확인합니다.
- Featured project 부재 시 fallback selection과 lead 결정 순서를 추적합니다.
- Metric, current year, recent journey, hero/footer/contact link selection helper를 찾습니다.
- Time-dependent year가 직접 `Date`를 읽는지 주입 가능한지 이 SHA에서 확인합니다.
- Renderer가 이전에 수행하던 derivation code와 비교해 owner 이동을 표시합니다.
- Unrelated collection을 빈 값으로 두거나 제거하는 초기 isolation 방식의 한계를 기록합니다.

확인 원칙:

- 먼저 `4d6b4e6d564e^`와 `4d6b4e6d564e`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Source field와 derived field 구분 | `src/lib/portfolio/view-models.ts`의 home type/factory가 featured fallback→lead project, metrics, recent journey, hero/footer/contact links와 `new Date().getFullYear()` 값을 계산합니다. |
| Fallback과 ordering 규칙 | home route discriminant와 presentation-ready fields를 만드는 factory를 도입했습니다. |
| Renderer에서 제거된 계산 | `src/lib/portfolio/view-models.ts`의 home type/factory가 featured fallback→lead project, metrics, recent journey, hero/footer/contact links와 `new Date().getFullYear()` 값을 계산합니다. |
| Route discriminant가 union에 제공하는 기반 | home route discriminant와 presentation-ready fields를 만드는 factory를 도입했습니다. `src/lib/portfolio/view-models.ts`의 home type/factory가 featured fallback→lead project, metrics, recent journey, hero/footer/contact links와 `new Date().getFullYear()` 값을 계산합니다. |
| 아직 broad compatibility가 남은 부분 | `RouteViewModelBase = PortfolioContent & ...`, `...content`, synthetic empty collections가 남아 실제 runtime isolation은 아직 아닙니다. Year도 주입 가능 clock이 아니라 직접 `Date`를 읽습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 각 home renderer가 featured fallback, metrics, contact/link placement, recent journey와 year를 직접 계산했습니다.
- **해당 SHA 핵심 코드:** `src/lib/portfolio/view-models.ts`의 home type/factory가 featured fallback→lead project, metrics, recent journey, hero/footer/contact links와 `new Date().getFullYear()` 값을 계산합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `44e3b80b297f`과 `d4ad7ecd0d08`이 다른 route projection으로 확장하고 `5897b4b024da`가 broad base를 제거합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** 각 home renderer가 featured fallback, metrics, contact/link placement, recent journey와 year를 직접 계산했습니다. 다섯 renderer가 같은 source graph를 다르게 해석하거나 시간 값과 fallback을 중복 계산할 수 있었습니다.
- **구현 결정과 경로:** home route discriminant와 presentation-ready fields를 만드는 factory를 도입했습니다. `src/lib/portfolio/view-models.ts`의 home type/factory가 featured fallback→lead project, metrics, recent journey, hero/footer/contact links와 `new Date().getFullYear()` 값을 계산합니다.
- **소유권·실패 처리:** factory가 semantic selection을 소유하기 시작하고 renderer는 prepared fields를 소비합니다. featured가 없으면 enabled projects에서 fallback하고 lead는 그 결과의 첫 항목으로 결정됩니다.
- **보장:** home derivation과 route discriminant가 한 위치에 생깁니다.
- **보장하지 않는 범위:** `RouteViewModelBase = PortfolioContent & ...`, `...content`, synthetic empty collections가 남아 실제 runtime isolation은 아직 아닙니다. Year도 주입 가능 clock이 아니라 직접 `Date`를 읽습니다.
- **후속 연결:** `44e3b80b297f`과 `d4ad7ecd0d08`이 다른 route projection으로 확장하고 `5897b4b024da`가 broad base를 제거합니다.

### 2. `44e3b80b297f` — refactor(content): 프로젝트 목록 파생 모델 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, REFACTOR
- **Source-defined thread role:** Moves project partitioning and fallback grouping into content projection.
- **Source classification summary:** Add a projects-route view model that resolves featured and archive partitions, project groups, and metric values before rendering.
- **Source classification reason:** Significant because it materially narrows ownership or coupling at a boundary used by multiple routes and renderers while preserving established behavior.

#### Source에서 확정된 구현 의도와 상태 변화

Projects route view model이 featured/archive partition, group ordering, fallback groups, metric values를 rendering 전에 해결합니다. Empty configured group은 생략하지만 unknown group의 project는 보존합니다.

#### 해당 SHA에서 확인할 실제 코드

- Featured와 archive collection을 나누는 predicate와 duplicate 방지 방식을 확인합니다.
- Configured group order/metadata를 적용하고 empty group을 생략하는 loop를 추적합니다.
- Unknown group ID를 deterministic fallback group으로 보존하는 규칙을 확인합니다.
- Metric values가 selector를 통해 한 번 계산되어 model에 저장되는지 확인합니다.
- Design renderer의 이전 local grouping code와 비교합니다.

확인 원칙:

- 먼저 `44e3b80b297f^`와 `44e3b80b297f`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Configured taxonomy와 unknown data 보존의 균형 | projects route projection에서 partition/grouping/order/metric을 한 번 해결했습니다. `createProjectsViewModel`은 featured predicate로 한 collection을 분리하고 archive는 featured가 아닌 나머지로 만들어 중복을 막습니다. Configured group 순서대로 non-empty group만 넣고 unknown group IDs는 deterministic fallback groups로 뒤에 보존합니다. |
| Empty group omission과 project omission 차이 | `createProjectsViewModel`은 featured predicate로 한 collection을 분리하고 archive는 featured가 아닌 나머지로 만들어 중복을 막습니다. Configured group 순서대로 non-empty group만 넣고 unknown group IDs는 deterministic fallback groups로 뒤에 보존합니다. |
| Group/order/metric owner 이동 | projection이 taxonomy/ordering/metric value를 소유하고 renderer는 group collection을 배치합니다. |
| Featured와 archive 중복 방지 증거 | projects route projection에서 partition/grouping/order/metric을 한 번 해결했습니다. `createProjectsViewModel`은 featured predicate로 한 collection을 분리하고 archive는 featured가 아닌 나머지로 만들어 중복을 막습니다. Configured group 순서대로 non-empty group만 넣고 unknown group IDs는 deterministic fallback groups로 뒤에 보존합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** projects renderer들이 featured/archive partition, group order, metrics와 unknown taxonomy fallback을 각자 계산했습니다.
- **해당 SHA 핵심 코드:** `createProjectsViewModel`은 featured predicate로 한 collection을 분리하고 archive는 featured가 아닌 나머지로 만들어 중복을 막습니다. Configured group 순서대로 non-empty group만 넣고 unknown group IDs는 deterministic fallback groups로 뒤에 보존합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `d4ad7ecd0d08`이 detail/about 관계 해석을 projection으로 이동합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** projects renderer들이 featured/archive partition, group order, metrics와 unknown taxonomy fallback을 각자 계산했습니다. configured empty group을 보여 주거나 unknown group project를 버리는 등 design별 behavior divergence가 가능했습니다.
- **구현 결정과 경로:** projects route projection에서 partition/grouping/order/metric을 한 번 해결했습니다. `createProjectsViewModel`은 featured predicate로 한 collection을 분리하고 archive는 featured가 아닌 나머지로 만들어 중복을 막습니다. Configured group 순서대로 non-empty group만 넣고 unknown group IDs는 deterministic fallback groups로 뒤에 보존합니다.
- **소유권·실패 처리:** projection이 taxonomy/ordering/metric value를 소유하고 renderer는 group collection을 배치합니다. configured empty group은 omission되지만 그 group이 unknown이라는 이유로 project 자체를 버리지는 않습니다.
- **보장:** 모든 design이 동일 featured/archive/group/metric 결과를 받습니다.
- **보장하지 않는 범위:** detail/about/journey/interview relationship은 아직 renderer 또는 다른 route에서 해결합니다.
- **후속 연결:** `d4ad7ecd0d08`이 detail/about 관계 해석을 projection으로 이동합니다.

### 3. `d4ad7ecd0d08` — refactor(content): 상세와 소개 파생 모델 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, REFACTOR
- **Source-defined thread role:** Resolves project and curation relationships before rendering.
- **Source classification summary:** Extend the route-model boundary to project detail and about pages.
- **Source classification reason:** Significant because it materially narrows ownership or coupling at a boundary used by multiple routes and renderers while preserving established behavior.

#### Source에서 확정된 구현 의도와 상태 변화

Project detail과 About projection을 추가합니다. Detail은 unknown project에 `null`, action links, stack fallback, secondary screenshots를 준비하고 About은 curation project reference를 indexed lookup으로 해결해 missing reference를 생략합니다.

#### 해당 SHA에서 확인할 실제 코드

- `createProjectDetailViewModel`의 lookup 실패 `null` branch를 확인합니다.
- Detail action links, technology fallback, lead image 제외 secondary screenshot 계산을 추적합니다.
- About curation project ID가 indexed lookup으로 해결되고 missing item이 filter되는 지점을 찾습니다.
- Renderer의 이전 unsafe non-null stack lookup과 raw reference resolution을 비교합니다.
- Detail null과 About omission이라는 서로 다른 absence policy를 구분합니다.

확인 원칙:

- 먼저 `d4ad7ecd0d08^`와 `d4ad7ecd0d08`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Detail null이 page-level 404로 이어질 준비 경계 | detail factory가 page-level absence를 표현하고 about factory가 curation join/order를 소유합니다. |
| Stack fallback과 screenshot exclusion 규칙 | project detail과 about projection을 추가하고 route semantics별 absence policy를 명시했습니다. |
| Curation reference order 보존 여부 | project detail과 about projection을 추가하고 route semantics별 absence policy를 명시했습니다. |
| Route semantics에 따른 missing-reference 정책 차이 | detail은 model 전체를 `null`로 반환할 수 있어 caller의 404 경계를 준비하고, about은 일부 missing curation reference만 omit합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** detail renderer는 project/stack/link/screenshot을 직접 찾고 about renderer는 curation project IDs를 raw collection에서 해결했습니다.
- **해당 SHA 핵심 코드:** `createProjectDetailViewModel`은 unknown ID에 `null`, valid project에 enabled action links, technology stack fallback, lead image를 제외한 secondary screenshots를 만듭니다. About factory는 indexed project lookup으로 curation IDs를 source order대로 resolve하고 missing ID는 filter합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `d7eaa1ac401d`이 route/model correlation을 추가합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** detail renderer는 project/stack/link/screenshot을 직접 찾고 about renderer는 curation project IDs를 raw collection에서 해결했습니다. unsafe non-null lookup과 design별 missing-reference 처리 차이가 있었습니다.
- **구현 결정과 경로:** project detail과 about projection을 추가하고 route semantics별 absence policy를 명시했습니다. `createProjectDetailViewModel`은 unknown ID에 `null`, valid project에 enabled action links, technology stack fallback, lead image를 제외한 secondary screenshots를 만듭니다. About factory는 indexed project lookup으로 curation IDs를 source order대로 resolve하고 missing ID는 filter합니다.
- **소유권·실패 처리:** detail factory가 page-level absence를 표현하고 about factory가 curation join/order를 소유합니다. detail은 model 전체를 `null`로 반환할 수 있어 caller의 404 경계를 준비하고, about은 일부 missing curation reference만 omit합니다.
- **보장:** stack/link/screenshot/curation relationship이 renderer 전에 확정됩니다.
- **보장하지 않는 범위:** route literal과 exact model type correlation, journey/interview joins는 아직 남습니다.
- **후속 연결:** `d7eaa1ac401d`이 route/model correlation을 추가합니다.

### 4. `d7eaa1ac401d` — refactor(routes): renderer view model 요청 타입 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, RENDERER
- **Source-defined thread role:** Correlates each route literal with its exact projection.
- **Source classification summary:** Introduce a typed migration request that pairs each route literal with its corresponding view-model variant.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

각 route literal을 정확한 view-model variant와 연결하는 typed migration request를 도입합니다. Registry는 새 discriminated request와 legacy props를 임시로 모두 받아 adapter합니다.

#### 해당 SHA에서 확인할 실제 코드

- Route literal에서 view-model type으로 매핑하는 generic/conditional type을 확인합니다.
- 새 request union과 legacy props의 discriminant 차이를 찾습니다.
- Adapter가 project detail에서만 project를 추출하는 branch를 확인합니다.
- Invalid route/model 조합이 compile time에 거부되는 예시를 작성합니다.
- Compatibility layer가 migration을 가능하게 하지만 broad input을 유지하는 부분을 기록합니다.

확인 원칙:

- 먼저 `d7eaa1ac401d^`와 `d7eaa1ac401d`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Route-to-model correlation의 TypeScript 표현 | design request union이 `{ route, viewModel: ViewModelForRoute<R> }` 형태의 correlation을 표현하고 registry adapter는 새 request와 legacy props를 모두 받아 detail일 때만 project를 추출합니다. |
| Legacy adapter lifecycle과 제거 조건 | `f8b0ab7b08aa`이 legacy content/request surface를 제거합니다. |
| Compile-time guarantee와 runtime guard 역할 | route literal을 conditional/generic type으로 model variant에 연결한 typed migration request를 추가했습니다. design request union이 `{ route, viewModel: ViewModelForRoute<R> }` 형태의 correlation을 표현하고 registry adapter는 새 request와 legacy props를 모두 받아 detail일 때만 project를 추출합니다. |
| 후속 `f8b0ab7b08aa`에서 제거될 surface | `f8b0ab7b08aa`이 legacy content/request surface를 제거합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** route model factories가 생겼지만 registry request가 route literal과 exact model variant를 강하게 묶지 않았습니다.
- **해당 SHA 핵심 코드:** design request union이 `{ route, viewModel: ViewModelForRoute<R> }` 형태의 correlation을 표현하고 registry adapter는 새 request와 legacy props를 모두 받아 detail일 때만 project를 추출합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `f8b0ab7b08aa`이 legacy content/request surface를 제거합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** route model factories가 생겼지만 registry request가 route literal과 exact model variant를 강하게 묶지 않았습니다. 잘못된 route/model 조합이 adapter나 renderer까지 도달할 수 있고 migration 동안 broad legacy props가 남았습니다.
- **구현 결정과 경로:** route literal을 conditional/generic type으로 model variant에 연결한 typed migration request를 추가했습니다. design request union이 `{ route, viewModel: ViewModelForRoute<R> }` 형태의 correlation을 표현하고 registry adapter는 새 request와 legacy props를 모두 받아 detail일 때만 project를 추출합니다.
- **소유권·실패 처리:** type system이 새 request의 correlation을 소유하지만 legacy adapter가 broad input lifecycle을 임시로 유지합니다. 새 typed request는 invalid pair를 compile time에 막지만 runtime/legacy caller에는 adapter guard가 필요합니다.
- **보장:** migration caller는 route와 맞는 projection을 요청할 수 있습니다.
- **보장하지 않는 범위:** complete content input과 compatibility surface가 남아 strict renderer-only contract는 아닙니다.
- **후속 연결:** `f8b0ab7b08aa`이 legacy content/request surface를 제거합니다.

### 5. `bc0a718e2052` — refactor(content): 여정 근거 view model 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, REFACTOR
- **Source-defined thread role:** Removes raw project lookup from journey renderers.
- **Source classification summary:** Introduce a journey-specific view model that resolves content references before they reach a renderer.
- **Source classification reason:** Significant because it materially narrows ownership or coupling at a boundary used by multiple routes and renderers while preserving established behavior.

#### Source에서 확정된 구현 의도와 상태 변화

Journey 전용 view model이 milestone anchor ID를 existing project로 해결하고 unknown ID는 생략합니다. Timeline project는 resolved object 또는 explicit `null`로 표현하며 raw project collection은 제외합니다.

#### 해당 SHA에서 확인할 실제 코드

- Project lookup index와 milestone anchor mapping/filter 순서를 확인합니다.
- Timeline project field가 absent, valid, unknown일 때 각각 어떤 model value가 되는지 추적합니다.
- Full projects collection이 journey model output에서 제외되는지 runtime/type 모두 확인합니다.
- 각 renderer의 이전 anchor lookup code와 비교해 중복 제거를 표시합니다.
- Source order가 resolution/filter 후에도 유지되는지 확인합니다.

확인 원칙:

- 먼저 `bc0a718e2052^`와 `bc0a718e2052`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Milestone unknown reference omission 정책 | journey-specific projection이 project index를 한 번 만들고 evidence relationship을 resolve하게 했습니다. |
| Timeline explicit `null` 정책 | journey-specific projection이 project index를 한 번 만들고 evidence relationship을 resolve하게 했습니다. |
| Renderer에 필요한 evidence relationship만 남긴 payload | factory는 milestone anchor IDs를 map한 뒤 unresolved item을 filter해 source order를 보존하고, timeline project는 valid object 또는 explicit `null`로 저장합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| Project lookup의 단일 owner | projection이 project lookup을 단독 소유하고 renderer payload에는 필요한 milestone/timeline relationship만 남습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** journey renderer마다 milestone anchor와 timeline project를 raw project collection에서 lookup했습니다.
- **해당 SHA 핵심 코드:** factory는 milestone anchor IDs를 map한 뒤 unresolved item을 filter해 source order를 보존하고, timeline project는 valid object 또는 explicit `null`로 저장합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `98f07b1c211d`이 같은 원칙을 interview hierarchy에 적용합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** journey renderer마다 milestone anchor와 timeline project를 raw project collection에서 lookup했습니다. unknown ID를 omit할지 placeholder로 보일지 design마다 달라질 수 있었습니다.
- **구현 결정과 경로:** journey-specific projection이 project index를 한 번 만들고 evidence relationship을 resolve하게 했습니다. factory는 milestone anchor IDs를 map한 뒤 unresolved item을 filter해 source order를 보존하고, timeline project는 valid object 또는 explicit `null`로 저장합니다.
- **소유권·실패 처리:** projection이 project lookup을 단독 소유하고 renderer payload에는 필요한 milestone/timeline relationship만 남습니다. unknown milestone anchor는 omit하고 absent/unknown timeline project는 `null`로 표현합니다.
- **보장:** journey renderer가 raw projects collection을 다시 조회하지 않아도 됩니다.
- **보장하지 않는 범위:** interview answer evidence는 아직 별도 projection이 필요합니다.
- **후속 연결:** `98f07b1c211d`이 같은 원칙을 interview hierarchy에 적용합니다.

### 6. `98f07b1c211d` — refactor(content): 인터뷰 근거 view model 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, REFACTOR
- **Source-defined thread role:** Preserves unresolved evidence explicitly while resolving valid answers once.
- **Source classification summary:** Introduce an interview-map view model that preserves the track, question, and answer hierarchy while resolving each answer’s project identifier to a project object or `null`.
- **Source classification reason:** Significant because it materially narrows ownership or coupling at a boundary used by multiple routes and renderers while preserving established behavior.

#### Source에서 확정된 구현 의도와 상태 변화

Interview-map view model이 track/question/answer hierarchy를 보존하면서 각 answer project ID를 project object 또는 `null`로 해결합니다. Missing evidence는 presentation boundary에 명시적으로 남습니다.

#### 해당 SHA에서 확인할 실제 코드

- Nested mapping에서 source identifier가 보존되는지 확인합니다.
- Project lookup이 한 번 만들어져 각 answer에 재사용되는지 확인합니다.
- Valid와 unknown reference가 object/`null`로 표현되는 branch를 추적합니다.
- Raw project collection이 model에 포함되지 않는지 확인합니다.
- Renderer가 unresolved ID를 visible no-evidence state로 표시할 field를 확인합니다.

확인 원칙:

- 먼저 `98f07b1c211d^`와 `98f07b1c211d`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Hierarchy 보존과 join resolution 분리 | track/question/answer hierarchy와 identifiers를 보존하면서 answer evidence만 object 또는 `null`로 resolve했습니다. interview factory는 project index를 한 번 만든 뒤 nested map에서 answer ID/depth note를 유지하고 `projectId`가 valid하면 object, 아니면 `project: null`을 넣습니다. |
| Missing evidence를 omission하지 않는 이유 | unresolved answer를 omit하지 않고 identifier와 `null`을 남겨 source의 evidence gap을 보존합니다. |
| Identifier, resolved project, depth note의 상관관계 | interview factory는 project index를 한 번 만든 뒤 nested map에서 answer ID/depth note를 유지하고 `projectId`가 valid하면 object, 아니면 `project: null`을 넣습니다. |
| Template별 project map 제거 전후 | track/question/answer hierarchy와 identifiers를 보존하면서 answer evidence만 object 또는 `null`로 resolve했습니다. interview factory는 project index를 한 번 만든 뒤 nested map에서 answer ID/depth note를 유지하고 `projectId`가 valid하면 object, 아니면 `project: null`을 넣습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** interview renderer마다 answer의 project ID를 map으로 다시 해결했습니다.
- **해당 SHA 핵심 코드:** interview factory는 project index를 한 번 만든 뒤 nested map에서 answer ID/depth note를 유지하고 `projectId`가 valid하면 object, 아니면 `project: null`을 넣습니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `f8b0ab7b08aa`이 route model을 registry의 유일한 input으로 만듭니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** interview renderer마다 answer의 project ID를 map으로 다시 해결했습니다. missing evidence를 삭제하면 질문/답변 구조 자체가 바뀌고 design마다 표시가 달라질 수 있었습니다.
- **구현 결정과 경로:** track/question/answer hierarchy와 identifiers를 보존하면서 answer evidence만 object 또는 `null`로 resolve했습니다. interview factory는 project index를 한 번 만든 뒤 nested map에서 answer ID/depth note를 유지하고 `projectId`가 valid하면 object, 아니면 `project: null`을 넣습니다.
- **소유권·실패 처리:** projection이 join을 소유하고 renderer는 explicit no-evidence state를 표시할 수 있습니다. unresolved answer를 omit하지 않고 identifier와 `null`을 남겨 source의 evidence gap을 보존합니다.
- **보장:** 모든 design이 같은 hierarchy와 evidence resolution을 받습니다.
- **보장하지 않는 범위:** registry가 complete content를 함께 받을 수 있는 우회는 아직 남습니다.
- **후속 연결:** `f8b0ab7b08aa`이 route model을 registry의 유일한 input으로 만듭니다.

### 7. `f8b0ab7b08aa` — refactor(designs): renderer 입력을 route view model로 제한

- **Importance:** S
- **Tags:** ARCH, ROUTING, RENDERER
- **Source-defined thread role:** Makes projected route data the only registry input.
- **Source classification summary:** Change the common design contract from the complete portfolio content object to the discriminated union of route view models.
- **Source classification reason:** Critical because it forbids renderers from consuming the global content graph and makes prepared, discriminated route data the only legal design input.

#### Source에서 확정된 구현 의도와 상태 변화

Common design contract를 complete `PortfolioContent`에서 discriminated route-view-model union으로 바꾸고 shell도 projected `footerLinks`를 받도록 합니다. Journey/interview temporary request와 compatibility 경계를 제거합니다.

#### S-level source profile

- **Problem:** Full-site renderer가 complete portfolio aggregate를 받아 design별로 join, filter, fallback을 다시 수행할 수 있었습니다.
- **Decision:** Renderer props를 discriminated route-view-model union으로 교체하고 shell data도 명시적으로 projection합니다.
- **Why it mattered:** Composition의 자유는 유지하되 project membership, evidence resolution, contact precedence를 재해석할 우회를 막습니다.
- **What changed:** Registry contract와 shell adapter를 route-owned projections로 좁힙니다.

#### 해당 SHA에서 확인할 실제 코드

- Registry render function input type이 route-view-model union으로 바뀌는 diff를 확인합니다.
- 각 renderer entry가 route discriminant를 좁혀 exact model field만 접근하는지 확인합니다.
- Shell adapter가 raw links 대신 `footerLinks`를 소비하는 모든 design을 찾습니다.
- Legacy content props와 alternate journey/interview request가 제거됐는지 검색합니다.
- Renderer의 raw project lookup/grouping/contact filtering이 type error로 드러나는지 확인합니다.
- Route page가 model 없이 registry를 호출할 수 없는 compile-time path를 추적합니다.
- Visual composition은 바꿀 수 있지만 semantic selection을 재해석할 수 없는 concrete example을 입증합니다.

확인 원칙:

- 먼저 `f8b0ab7b08aa^`와 `f8b0ab7b08aa`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| View model이 있어도 complete content를 허용하던 우회 | registry와 모든 renderer props를 discriminated `PortfolioRouteViewModel` union 하나로 교체했습니다. `src/designs/types.ts`, `src/designs/registry.tsx`와 다섯 renderer entry가 route discriminant로 exact fields를 좁히며 shell은 raw links 대신 projected `footerLinks`를 받습니다. Journey/interview temporary request와 legacy content props가 제거됐습니다. |
| Design별 join/filter/fallback 재구현 문제 | registry와 모든 renderer props를 discriminated `PortfolioRouteViewModel` union 하나로 교체했습니다. |
| Discriminated route models only decision | registry와 모든 renderer props를 discriminated `PortfolioRouteViewModel` union 하나로 교체했습니다. |
| Registry contract, renderer props, shell adapters, removed compatibility | registry와 모든 renderer props를 discriminated `PortfolioRouteViewModel` union 하나로 교체했습니다. `src/designs/types.ts`, `src/designs/registry.tsx`와 다섯 renderer entry가 route discriminant로 exact fields를 좁히며 shell은 raw links 대신 projected `footerLinks`를 받습니다. Journey/interview temporary request와 legacy content props가 제거됐습니다. |
| Content graph, projection, renderer composition ownership | content graph는 loader/facade, semantic projection은 route factory, visual composition은 renderer가 소유합니다. |
| Unresolved references가 model에서 표현되는 방식 | `src/designs/types.ts`, `src/designs/registry.tsx`와 다섯 renderer entry가 route discriminant로 exact fields를 좁히며 shell은 raw links 대신 projected `footerLinks`를 받습니다. Journey/interview temporary request와 legacy content props가 제거됐습니다. |
| 보장하는 것: registry input 수준 semantic consistency | registry input 수준에서 route/model correlation과 semantic selection consistency를 보장합니다. |
| 아직 보장하지 않는 것: actual runtime payload 최소성 | 각 factory가 runtime에 실제로 full content를 spread하는지까지는 막지 못합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** view model이 있어도 common design contract와 compatibility request가 complete `PortfolioContent`를 허용했습니다.
- **해당 SHA 핵심 코드:** `src/designs/types.ts`, `src/designs/registry.tsx`와 다섯 renderer entry가 route discriminant로 exact fields를 좁히며 shell은 raw links 대신 projected `footerLinks`를 받습니다. Journey/interview temporary request와 legacy content props가 제거됐습니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `5897b4b024da`가 runtime payload의 broad spread와 synthetic empty collections를 제거합니다.

#### SHA별 복원 결론

- **이전 상태:** view model이 있어도 common design contract와 compatibility request가 complete `PortfolioContent`를 허용했습니다.
- **문제와 위험:** renderer가 raw projects/links/groups를 다시 얻어 join, fallback, contact precedence를 재구현할 수 있었습니다.
- **핵심 결정:** registry와 모든 renderer props를 discriminated `PortfolioRouteViewModel` union 하나로 교체했습니다.
- **실제 구현 경로:** `src/designs/types.ts`, `src/designs/registry.tsx`와 다섯 renderer entry가 route discriminant로 exact fields를 좁히며 shell은 raw links 대신 projected `footerLinks`를 받습니다. Journey/interview temporary request와 legacy content props가 제거됐습니다.
- **소유권·상태 변화:** content graph는 loader/facade, semantic projection은 route factory, visual composition은 renderer가 소유합니다.
- **실패 경계:** unresolved relationship은 journey omission, timeline/interview `null`, detail `null`처럼 model에 이미 표현돼 renderer가 policy를 다시 정하지 않습니다.
- **보장:** registry input 수준에서 route/model correlation과 semantic selection consistency를 보장합니다.
- **보장하지 않는 범위:** 각 factory가 runtime에 실제로 full content를 spread하는지까지는 막지 못합니다.
- **후속 fix/test:** `5897b4b024da`가 runtime payload의 broad spread와 synthetic empty collections를 제거합니다.

### 8. `5897b4b024da` — refactor(content): route view model 공용 경계 제한

- **Importance:** S
- **Tags:** ARCH, CONTENT, ROUTING
- **Source-defined thread role:** Removes the global content spread and finalizes a small shared base.
- **Source classification summary:** Finish the route-model isolation by reducing the common base to presentation, profile, site, and prepared footer links.
- **Source classification reason:** Critical because it finalizes route-level data ownership: the shared payload is intentionally small and unrelated source collections are unavailable both at runtime and by type.

#### Source에서 확정된 구현 의도와 상태 변화

Route-model common base를 site, profile, presentation, prepared footer links로 축소하고 factory에서 complete content spread와 synthetic empty collections를 제거합니다. Unavailable key는 `never`로만 남깁니다.

#### S-level source profile

- **Problem:** Earlier view model이 broad common object와 synthetic empty collection을 유지해 actual runtime ownership을 숨길 수 있었습니다.
- **Decision:** Common base를 site, profile, presentation, footerLinks로 축소하고 unavailable key는 runtime에 복사하지 않습니다.
- **Why it mattered:** Route isolation을 완결해 renderer가 unrelated source를 복구하지 못하게 합니다.
- **What changed:** Full content spread와 empty compatibility arrays를 제거합니다.

#### 해당 SHA에서 확인할 실제 코드

- Common base type의 허용 field와 `never` 처리된 unavailable key를 확인합니다.
- Factory object construction에서 `...content` spread가 제거되는 diff를 찾습니다.
- Links, groups, projects, metrics용 synthetic empty array가 제거됐는지 확인합니다.
- 각 route factory가 필요한 source field만 명시적으로 copy하는지 runtime key로 확인합니다.
- `never` field가 compile-time 접근을 막지만 runtime key를 만들지 않는지 확인합니다.
- Footer links가 common prepared field로 남는 이유와 selection owner를 추적합니다.
- Legacy helper signature와 actual payload가 어떻게 공존하는지 확인합니다.

확인 원칙:

- 먼저 `5897b4b024da^`와 `5897b4b024da`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Broad common object와 empty collection이 ownership을 숨긴 방식 | shared shell fields는 small base가, route semantics는 각 return literal이 소유하며 runtime object에는 compatibility key가 만들어지지 않습니다. |
| Small shared base, explicit route fields, no runtime compatibility payload decision | common base를 `site/profile/presentation/footerLinks`로 제한하고 각 factory가 route field만 explicit copy하도록 바꿨습니다. |
| Base type, factory return literals, `never` declarations | common base를 `site/profile/presentation/footerLinks`로 제한하고 각 factory가 route field만 explicit copy하도록 바꿨습니다. `src/lib/portfolio/view-models.ts`에서 full spread와 synthetic links/groups/projects/metrics arrays가 제거되고 unavailable source keys는 type-level `never`로만 선언됩니다. |
| Runtime ownership과 type-level compatibility 차이 | shared shell fields는 small base가, route semantics는 각 return literal이 소유하며 runtime object에는 compatibility key가 만들어지지 않습니다. |
| Payload field 변화 | `src/lib/portfolio/view-models.ts`에서 full spread와 synthetic links/groups/projects/metrics arrays가 제거되고 unavailable source keys는 type-level `never`로만 선언됩니다. |
| 보장하는 것: unrelated collection 복구 불가 | renderer가 unrelated collection을 model에서 복구할 수 없는 type/runtime 경계를 함께 제공합니다. |
| 후속 test와 연결 | `527b9f872333`이 allowed runtime keys와 source-structure prohibition을 regression test로 고정합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** type-level renderer input은 좁아졌지만 factory base가 `PortfolioContent` intersection과 `...content`를 사용하고 unrelated collections를 빈 배열로 덮었습니다.
- **해당 SHA 핵심 코드:** `src/lib/portfolio/view-models.ts`에서 full spread와 synthetic links/groups/projects/metrics arrays가 제거되고 unavailable source keys는 type-level `never`로만 선언됩니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `527b9f872333`이 allowed runtime keys와 source-structure prohibition을 regression test로 고정합니다.

#### SHA별 복원 결론

- **이전 상태:** type-level renderer input은 좁아졌지만 factory base가 `PortfolioContent` intersection과 `...content`를 사용하고 unrelated collections를 빈 배열로 덮었습니다.
- **문제와 위험:** 실제 object에 global graph가 남으면 casting, key enumeration, helper signature를 통해 ownership을 복구하거나 숨길 수 있었습니다.
- **핵심 결정:** common base를 `site/profile/presentation/footerLinks`로 제한하고 각 factory가 route field만 explicit copy하도록 바꿨습니다.
- **실제 구현 경로:** `src/lib/portfolio/view-models.ts`에서 full spread와 synthetic links/groups/projects/metrics arrays가 제거되고 unavailable source keys는 type-level `never`로만 선언됩니다.
- **소유권·상태 변화:** shared shell fields는 small base가, route semantics는 각 return literal이 소유하며 runtime object에는 compatibility key가 만들어지지 않습니다.
- **실패 경계:** `never`는 compile-time access를 막고 explicit object construction은 runtime key 자체를 없앱니다.
- **보장:** renderer가 unrelated collection을 model에서 복구할 수 없는 type/runtime 경계를 함께 제공합니다.
- **보장하지 않는 범위:** serialization byte나 visual behavior는 이 refactor만으로 측정하지 않습니다.
- **후속 fix/test:** `527b9f872333`이 allowed runtime keys와 source-structure prohibition을 regression test로 고정합니다.

### 9. `527b9f872333` — test(content): scoped view model과 연락처 회귀 검증

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, TEST
- **Source-defined thread role:** Verifies allowed runtime fields and missing-reference behavior.
- **Source classification summary:** Lock down the route-view-model boundary at runtime rather than relying only on TypeScript declarations.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

각 route model이 runtime에 보유해도 되는 source field를 열거하고 full `PortfolioContent` spread/intersection 재구성을 거부합니다. Contact fallback, journey omission, interview null behavior도 회귀 계약으로 고정합니다.

#### 해당 SHA에서 확인할 실제 코드

- Route별 allowed field fixture와 shared shell field assertion을 확인합니다.
- Runtime key enumeration 또는 source structural check가 full spread/intersection을 탐지하는 방식을 추적합니다.
- Projects/About model이 실제 필요한 contact data를 유지하는 test를 확인합니다.
- Journey unresolved anchor drop과 Interview identifier+`null` 유지 fixture를 비교합니다.
- Contact preferred link precedence와 placement fallback test를 확인합니다.
- Test가 호출하는 production factory와 isolated helper 범위를 구분합니다.

확인 원칙:

- 먼저 `527b9f872333^`와 `527b9f872333`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | route payload scope, contact selection과 omission/null policy가 runtime regression으로 고정됩니다. |
| 주입한 incomplete reference와 expected model shape | runtime key whitelist, source structural assertion, semantic incomplete-reference fixtures와 renderer matrix를 추가했습니다. `src/lib/portfolio/view-models.test.ts`는 8개 model의 allowed keys, shared fields, contact fallback, journey unresolved anchor omission, interview identifier+`project:null`을 검사하고 source text에서 `PortfolioContent &`와 `...content`를 거부합니다. `src/designs/route-view-models.test.tsx`는 5 design×8 route의 design root와 non-empty `h1`을 검사합니다. |
| Runtime key inspection, structural source assertion, semantic fixture | `src/lib/portfolio/view-models.test.ts`는 8개 model의 allowed keys, shared fields, contact fallback, journey unresolved anchor omission, interview identifier+`project:null`을 검사하고 source text에서 `PortfolioContent &`와 `...content`를 거부합니다. `src/designs/route-view-models.test.tsx`는 5 design×8 route의 design root와 non-empty `h1`을 검사합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| 통과하는 production factory path | `src/lib/portfolio/view-models.test.ts`는 8개 model의 allowed keys, shared fields, contact fallback, journey unresolved anchor omission, interview identifier+`project:null`을 검사하고 source text에서 `PortfolioContent &`와 `...content`를 거부합니다. `src/designs/route-view-models.test.tsx`는 5 design×8 route의 design root와 non-empty `h1`을 검사합니다. |
| 증명하는 것: route payload scope와 resolution policy | route payload scope, contact selection과 omission/null policy가 runtime regression으로 고정됩니다. |
| 증명하지 않는 것: visual behavior 또는 network payload size | visual fidelity, network payload byte, browser accessibility와 모든 source permutation은 증명하지 않습니다. |
| 후속 변경에서 막는 회귀 | 이 thread의 최종 boundary를 보호하며 이후 renderer/SEO/performance work가 같은 scoped model을 전제로 진행됩니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** route payload isolation과 missing-reference policy가 구현 convention과 TypeScript declaration에만 의존했습니다.
- **해당 SHA 핵심 코드:** `src/lib/portfolio/view-models.test.ts`는 8개 model의 allowed keys, shared fields, contact fallback, journey unresolved anchor omission, interview identifier+`project:null`을 검사하고 source text에서 `PortfolioContent &`와 `...content`를 거부합니다. `src/designs/route-view-models.test.tsx`는 5 design×8 route의 design root와 non-empty `h1`을 검사합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** 이 thread의 최종 boundary를 보호하며 이후 renderer/SEO/performance work가 같은 scoped model을 전제로 진행됩니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** route payload isolation과 missing-reference policy가 구현 convention과 TypeScript declaration에만 의존했습니다. 후속 refactor가 `...content`, intersection, extra key, omission/null policy를 조용히 되돌릴 수 있었습니다.
- **구현 결정과 경로:** runtime key whitelist, source structural assertion, semantic incomplete-reference fixtures와 renderer matrix를 추가했습니다. `src/lib/portfolio/view-models.test.ts`는 8개 model의 allowed keys, shared fields, contact fallback, journey unresolved anchor omission, interview identifier+`project:null`을 검사하고 source text에서 `PortfolioContent &`와 `...content`를 거부합니다. `src/designs/route-view-models.test.tsx`는 5 design×8 route의 design root와 non-empty `h1`을 검사합니다.
- **소유권·실패 처리:** production factories를 직접 호출해 payload shape/semantic resolution을 검사하며 renderer matrix는 registry→renderer integration을 소비합니다. incomplete reference를 fixture에 주입해 expected object shape를 비교합니다. 별도 allocator-style failure injection은 해당되지 않습니다.
- **보장:** route payload scope, contact selection과 omission/null policy가 runtime regression으로 고정됩니다.
- **보장하지 않는 범위:** visual fidelity, network payload byte, browser accessibility와 모든 source permutation은 증명하지 않습니다.
- **후속 연결:** 이 thread의 최종 boundary를 보호하며 이후 renderer/SEO/performance work가 같은 scoped model을 전제로 진행됩니다.

## 6. Invariant ledger

| Invariant | Source에서 확인된 변화 지점 | 해당 SHA의 실제 code/test evidence | 부족함이 드러난 시점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Cross-content join, ordering, fallback은 route projection이 한 번만 수행합니다. | `4d6b4e6d564e → 44e3b80b297f → d4ad7ecd0d08` | home/projects/detail/about factories의 featured fallback, grouping/order, metric, links/stack/screenshots와 curation joins. | 초기 home projection은 broad content spread와 empty compatibility collections를 유지했고 다른 routes의 joins도 renderer에 남았습니다. | 각 route factory가 semantic selection/join/order/fallback을 한 번 수행하고 renderer에는 prepared data만 줍니다. |
| Route literal은 정확한 view-model variant와 상관관계를 가집니다. | `d7eaa1ac401d` | `d7eaa1ac401d`의 route-to-view-model conditional/generic request. | legacy props와 adapter가 migration 동안 broad input을 허용했습니다. | `f8b0ab7b08aa` 이후 registry input은 discriminated route model union 하나이며 route literal과 variant가 상관됩니다. |
| Journey와 interview renderer는 raw project collection을 조회하지 않습니다. | `bc0a718e2052 → 98f07b1c211d` | journey factory의 project index/anchor filtering/timeline `null`, interview factory의 nested project object 또는 `null`. | 직전 renderer마다 raw project map을 재구성했습니다. | journey/interview renderer는 model에 포함된 evidence relationship만 소비하고 raw projects collection을 받지 않습니다. |
| Registry renderer의 유일한 입력은 discriminated route view model입니다. | `f8b0ab7b08aa` | `f8b0ab7b08aa`의 common design props/registry renderer signature 변경과 legacy request 제거. | view models가 있어도 complete content를 같이 받을 수 있던 우회가 있었습니다. | registry entry와 모든 design route renderer의 legal input이 route model union으로 제한됩니다. |
| Common route base는 site/profile/presentation/footerLinks로 제한됩니다. | `5897b4b024da` | `5897b4b024da`의 base fields, explicit return literals, removed `...content`/synthetic arrays와 unavailable-key `never` declarations. | `f8b0ab7b08aa` 직후에도 actual model object는 broad spread를 가질 수 있었습니다. | runtime common payload는 site/profile/presentation/footerLinks와 route-owned fields만 포함합니다. |
| 허용 field와 missing-reference policy는 runtime regression test로 고정됩니다. | `527b9f872333` | `527b9f872333`의 runtime key whitelist, source text prohibitions, contact and unresolved-reference fixtures, 5×8 renderer matrix. | type declarations/convention만으로는 broad spread와 omission/null policy 회귀를 막지 못했습니다. | allowed runtime field set과 journey omission/interview null/contact fallback이 regression contract로 고정됩니다. |

Ledger 작성 원칙:

- Source가 명시한 invariant만 사용하고 새 invariant를 확정 사실처럼 추가하지 않습니다.
- 도입, 강화, 부족함 노출, fix, regression test가 서로 다른 commit이면 각 열에 분리해 기록합니다.
- Code evidence에는 실제 field, function, branch, selector, command 또는 assertion을 적습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 위험 | 대응 commit | 실제 수정/강화 code에서 확인할 것 | Test 또는 실행 증거 |
| --- | --- | --- | --- |
| Design별로 featured fallback, group order, metric, contact precedence를 다르게 계산함 | 4d6b4e6d564e, 44e3b80b297f, d4ad7ecd0d08 | 동일 derivation이 renderer에서 factory로 이동하는 diff를 확인합니다. | 여러 design이 같은 prepared collection을 받는 caller를 기록합니다. |
| Journey/Interview가 raw project map을 재구성해 missing reference를 다르게 처리함 | bc0a718e2052, 98f07b1c211d | Omit과 explicit `null` 정책을 실제 field와 renderer branch로 구분합니다. | `527b9f872333`에서 identifier 보존과 null/omission test를 연결합니다. |
| Type은 제한됐지만 factory가 full content를 spread하거나 synthetic empty array를 넣음 | 5897b4b024da | Factory return object에서 spread/empty compatibility collection 제거를 확인합니다. | Runtime key enumeration test가 우회를 막는지 확인합니다. |

### 실제 연결 기록

- Journey missing anchor는 omit하지만 interview missing evidence는 identifier와 `project: null`을 유지합니다. Detail lookup failure는 entire model `null`로 page 404 경계를 준비합니다.
- `f8b0ab7b08aa`은 legal renderer input을 좁혔고 `5897b4b024da`는 실제 runtime object shape를 좁혔습니다. 둘은 같은 작업이 아닙니다.
- `527b9f872333`은 source structure와 runtime keys를 함께 검사하지만 visual/network-size 증거는 아닙니다.

## 8. Ownership / state / responsibility 변화

| Concern | Thread 초기 owner/state | Thread 최종 owner/state | 실제 symbol과 호출 경로 |
| --- | --- | --- | --- |
| Home selection과 시간 값 | 각 renderer | Home view-model factory | `createHomeViewModel`: portfolio content → featured/lead/metrics/recent journey/placed links/current year. |
| Projects partition/group/metric | Route와 design별 helper | Projects view model | `createProjectsViewModel`: enabled projects + configured groups → featured/archive/groups/metric values. |
| Detail links/stack/screenshots와 About curation refs | Renderer lookup | Detail/About projection | `createProjectDetailViewModel`/about factory: ID lookup, action links, stack fallback, secondary media와 curation resolution. |
| Journey anchor/timeline project | Renderer raw project lookup | Journey projection | journey factory: project index → milestone anchors/nullable timeline evidence. |
| Interview answer evidence | Renderer별 project map | Interview projection | interview factory: track/question/answer hierarchy → resolved project 또는 explicit `null`. |
| Renderer input | Complete content 또는 migration union | Discriminated route-view-model union | page factory call → `PortfolioRouteViewModel` → `renderDesignRoute` → route-discriminated renderer. |
| Shared runtime payload | Full content spread와 빈 배열 | Small common base + route-owned fields | `RouteViewModelBase` small fields + explicit route return literals; no runtime global content spread. |

## 9. Thread 최종 상태

### Source에서 확정된 최종 상태

Renderer가 전체 `PortfolioContent`를 받아 관계를 다시 해석하던 상태에서 route별 view model이 join, ordering, fallback, missing-reference policy를 한 번만 해결하고 필요한 데이터만 노출하는 상태로 발전하는 과정을 복원합니다.

### 학습자가 완성할 최종 설명

- **Thread 시작 시점의 설계와 위험:** 초기 multi-design renderer는 complete portfolio graph를 받아 featured selection, grouping, joins와 missing-reference policy를 각각 재해석할 수 있었습니다.
- **핵심 architecture/decision이 형성된 순서:** home/projects/detail/about projection부터 시작해 route/model correlation, journey/interview evidence resolution, registry input 제한, runtime base 축소 순으로 ownership을 이동했습니다.
- **실제 failure 또는 부족함이 드러난 지점:** `f8b0ab7b08aa`에서 type/registry input은 좁아졌지만 factory가 `...content`와 empty collections를 유지해 actual payload isolation이 아직 불완전했습니다.
- **Fix 또는 boundary 강화가 바꾼 invariant:** `5897b4b024da`가 full spread를 제거하고 `527b9f872333`이 runtime keys와 semantic absence policy를 회귀 test로 고정했습니다.
- **Test/build/browser evidence가 보장한 범위:** factory 및 test source를 정적 검토해 caller/callee와 expected shapes를 확인했습니다. Vitest/renderer matrix는 네트워크 제한으로 실행하지 않았습니다.
- **Thread 종료 시점에도 보장하지 않는 범위:** visual behavior, serialized network bytes, all possible incomplete content와 time determinism은 thread 종료 시점에도 별도 보장입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 source-backed flow의 각 단계에 실제 file path, symbol, input/output, failure branch를 추가합니다.

1. App Router page가 validated portfolio content를 로드합니다.
   - 실제 코드 위치: `src/lib/portfolio/content.ts`와 App Router pages
   - 입력과 출력: validated facade가 site/profile/projects/links/journey/interview/curation aggregate를 제공합니다.
   - 실패/absence 처리: invalid raw source는 이전 ingestion boundary에서 실패하고 disabled/absent records는 facade policy로 제외됩니다.
2. Route-specific factory가 route literal에 맞는 projection을 만듭니다.
   - 실제 코드 위치: `src/lib/portfolio/view-models.ts::create*ViewModel`
   - 입력과 출력: route literal에 맞는 source subset과 detail ID/current time context를 받아 exact model variant를 만듭니다.
   - 실패/absence 처리: detail lookup 실패는 `null`; other route factories는 route-specific absence 표현을 사용합니다.
3. Factory가 ordering, filtering, reference resolution, fallback, time 값을 확정합니다.
   - 실제 코드 위치: 각 route factory의 selectors/indexes
   - 입력과 출력: sorting, featured/archive partition, link placement, stack fallback, project reference resolution과 year를 계산합니다.
   - 실패/absence 처리: unknown configured relationship은 omission/fallback/null 중 해당 route policy로 변환합니다.
4. Unresolved reference를 route policy에 따라 omit 또는 explicit `null`로 표현합니다.
   - 실제 코드 위치: journey/interview/detail/about mapping branches
   - 입력과 출력: milestone missing project는 omit, timeline/interview evidence는 explicit `null`, detail unknown ID는 whole model `null`입니다.
   - 실패/absence 처리: renderer가 raw ID를 다시 lookup하지 않도록 absence가 output shape에 남습니다.
5. Common shell field와 route-owned field만 runtime model에 복사합니다.
   - 실제 코드 위치: small `RouteViewModelBase`와 explicit object literals
   - 입력과 출력: site/profile/presentation/footerLinks 및 route fields만 runtime object에 복사합니다.
   - 실패/absence 처리: unavailable source keys는 runtime에 생성되지 않고 type-level `never`가 access를 막습니다.
6. Registry가 discriminated model을 선택된 renderer에 전달합니다.
   - 실제 코드 위치: `src/designs/registry.tsx::renderDesignRoute`
   - 입력과 출력: discriminated model을 selected design route entry에 전달합니다.
   - 실패/absence 처리: route/model mismatch는 type contract와 renderer switch에서 드러납니다.
7. Renderer는 prepared data를 배치할 뿐 global content graph를 다시 조회하지 않습니다.
   - 실제 코드 위치: 각 design renderer
   - 입력과 출력: prepared collections/relationships를 visual hierarchy에 배치합니다.
   - 실패/absence 처리: global content graph가 input에 없어 join/filter/fallback을 독자 재구성할 수 없습니다.

### 코드 없이 설명하기

> 이 Thread의 최종 실행 흐름을 code snippet 없이 자신의 말로 작성합니다. 설계 → 구현 → failure/risk → 수정/강화 → 검증 순서가 드러나야 합니다.

처음에는 다섯 renderer가 broad portfolio object에서 featured project, groups, links와 evidence 관계를 직접 계산할 수 있었습니다. Route factories가 home부터 projects, detail/about, journey, interview 순으로 joins와 fallback을 소유했습니다. Typed route request가 literal과 model variant를 묶고, registry contract는 complete content를 제거했습니다. 마지막으로 common base에서 global spread와 fake empty arrays까지 없애 실제 runtime payload를 좁혔고 tests가 allowed keys와 missing-reference 정책을 고정했습니다. Renderer는 이제 의미를 다시 계산하지 않고 준비된 route data를 배치합니다.

> **실행 증거 구분:** 참조 SHA의 diff와 해당 SHA 파일은 GitHub connector로 확인했습니다. 로컬 clone은 실행 환경의 DNS 차단으로 실패해 build, unit/E2E, browser, Lighthouse, Docker command는 실행하지 않았습니다. 따라서 위 test 결과는 구현된 test technique과 assertion의 정적 검토이며 실제 통과 결과가 아닙니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 실제로 checkout하거나 diff로 확인했습니다.
- [x] Thread 내 commit 순서를 source와 동일하게 유지했습니다.
- [x] 각 A/S commit에서 직전 상태, decision, failure boundary, guarantee와 non-guarantee를 구분했습니다.
- [x] B commit은 thread 흐름에서 필요한 구현 역할과 state change를 확인했습니다.
- [x] Fix commit을 독립 feature가 아니라 기존 가정 → failure → root cause → corrected invariant로 설명했습니다.
- [x] Test commit에서 production invariant, failure injection, technique, traversed production path, proves/does-not-prove를 구분했습니다.
- [x] Invariant ledger의 모든 주장에 해당 SHA의 code/test evidence가 있습니다.
- [x] Final HEAD의 code를 과거 commit 설명에 소급하지 않았습니다.
- [x] Thread 최종 흐름을 code 없이 설명할 수 있습니다.
===== END FILE: 03-route-projections-and-renderer-data-ownership.md =====

===== BEGIN FILE: 04-template-preview-to-production-publication.md =====
# Thread: Template preview to production publication

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 source에서 확정된 thread 구조와 commit 역할만 미리 제공합니다. 실제 구현 해석, code evidence, failure 재현, test 결과, 최종 설명은 해당 SHA의 코드를 직접 확인해 작성합니다.

## 1. Thread 목표

Schema-valid starter content와 실제 공개 가능한 production content를 구분하고 strict readiness 결과를 build, metadata, robots, sitemap에 일관되게 적용하는 publication trust boundary를 복원합니다.

**Source-defined significance**

> Schema-valid starter data is not necessarily safe to publish. This progression adds a second, stricter publication contract and then reuses it for canonical metadata, robots policy, and sitemap discovery. It prevents template identities and incomplete evidence from becoming indexed production claims.

### 이 Thread에 직접 연결되는 Critical Invariants

- Missing/empty/explicit template mode는 template이며 exact production만 production을 요청합니다. — `b3bd671a3243`
- Production origin은 public HTTP(S)이고 local/reserved/credential-bearing 값이 아닙니다. — `47b99d6256ef`
- Production result는 complete readiness 성공 뒤에만 verified URL을 포함합니다. — `002b642d52a3`
- Published project는 실제 자산과 enabled public exit를 가지며 usable contact가 존재합니다. — `bcd87ed856bf → 71e7ece7208f`
- Readiness는 normal production build의 mandatory gate입니다. — `37c0dbc079ff`
- Template는 non-indexable이고 crawler output은 enabled publication surface만 반영합니다. — `55b6061e0052 → cb61450ad922 → 70b69f04e8c7`

### 연결되는 Major Engineering Difficulty

- 편집 가능한 template content와 실제 publishable content를 구분해 local preview 편의와 production fail-closed를 동시에 유지하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Content mode는 어떤 입력에서 template, production, invalid로 결정되며 추측하지 않는 정책은 어디에 표현되는가?
- Structural schema validation과 production readiness validation은 무엇을 다르게 보장하는가?
- Public origin, asset, project exit, contact method가 production result를 얻기 위해 모두 필요한 이유는 무엇인가?
- Build gate와 crawler-visible output이 같은 mode/result를 소비한다는 것을 어떤 call relation으로 확인하는가?
- Disabled page/project와 template deployment가 metadata, robots, sitemap에서 제외되는 경로는 무엇인가?

## 3. 완료 기준

- Template/production mode의 input, return type, required checks, indexing 결과를 하나의 표로 정리했습니다.
- `002b642d52a3`의 discriminated result가 verified `URL`을 포함하는 success path와 failure branch를 추적했습니다.
- `37c0dbc079ff`에서 schema check 뒤 readiness check가 build status로 전파되는 순서를 기록했습니다.
- Metadata, robots, sitemap이 같은 publication decision을 사용하며 모순되지 않는지 확인했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b3bd671a3243` | feat(content): 콘텐츠 mode와 readiness 오류 모델 추가 | A | CONTENT, VALIDATION | Defines conservative template mode and explicit production mode. |
| 2 | `47b99d6256ef` | feat(content): public origin과 자산 경계 검증 추가 | A | CONTENT, VALIDATION | Rejects local, reserved, credential-bearing, and misplaced publication inputs. |
| 3 | `002b642d52a3` | feat(content): production readiness 기본 검사 추가 | S | ARCH, CONTENT, VALIDATION | Aggregates origin and placeholder checks into a discriminated production result. |
| 4 | `bcd87ed856bf` | feat(content): 필수 자산과 프로젝트 readiness 추가 | A | CONTENT, VALIDATION | Requires real public evidence and exits for every published project. |
| 5 | `71e7ece7208f` | feat(content): 연락 수단과 build readiness 연결 | A | CONTENT, VALIDATION, DEPLOY | Requires a usable contact method and exposes one mode-aware gate. |
| 6 | `37c0dbc079ff` | build(content): readiness 검사를 prebuild에 연결 | A | CONTENT, VALIDATION, DEPLOY | Makes publication completeness mandatory during normal production builds. |
| 7 | `55b6061e0052` | feat(seo): 콘텐츠 mode별 metadata 정책 추가 | A | CONTENT, SEO | Aligns page indexing with the validated content mode. |
| 8 | `cb61450ad922` | feat(seo): 콘텐츠 mode별 robots 정책 추가 | A | CONTENT, SEO | Aligns crawler policy with the same mode contract. |
| 9 | `70b69f04e8c7` | feat(seo): 공개 route sitemap 생성 | A | ARCH, ROUTING, SEO | Publishes only enabled production routes and projects. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA를 checkout한 상태에서 작성합니다. Thread 내 이전 commit은 비교 대상으로 사용할 수 있지만 final HEAD를 정답처럼 소급하지 않습니다.

### 1. `b3bd671a3243` — feat(content): 콘텐츠 mode와 readiness 오류 모델 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Defines conservative template mode and explicit production mode.
- **Source classification summary:** Introduce the type and error model for distinguishing template content from production-ready content.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Template content와 production-ready content를 구분하는 mode/error protocol을 정의합니다. Missing, empty, explicit `template`은 template로, exact `production`만 production으로 처리하며 unsupported 값은 즉시 실패합니다.

#### 해당 SHA에서 확인할 실제 코드

- Content mode union과 resolver의 input normalization/branch를 확인합니다.
- Unsupported string이 fallback되지 않고 어떤 error로 변환되는지 추적합니다.
- Readiness issue의 source file/path/message와 aggregate error formatting을 확인합니다.
- Production branch만 parsed site `URL`을 요구하는 discriminated result type을 확인합니다.
- 이 commit에는 actual readiness check가 아직 없다는 protocol/implementation 경계를 기록합니다.

확인 원칙:

- 먼저 `b3bd671a3243^`와 `b3bd671a3243`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Conservative template default의 정확한 입력 집합 | `src/lib/content-readiness.ts::resolvePortfolioContentMode`는 `undefined`, 빈 문자열, exact `template`을 template로, exact `production`만 production으로 반환하고 나머지는 throw합니다. Result는 template일 때 `siteUrl: undefined`, production일 때 `siteUrl: URL`입니다. |
| Exact production activation과 invalid mode | unsupported 문자열은 fallback하지 않고 descriptive `Error`가 되며 이 SHA에는 아직 actual readiness issue를 수집하는 validator가 없습니다. |
| Structured issue/error model | conservative resolver와 discriminated readiness result/error types를 도입했습니다. `src/lib/content-readiness.ts::resolvePortfolioContentMode`는 `undefined`, 빈 문자열, exact `template`을 template로, exact `production`만 production으로 반환하고 나머지는 throw합니다. Result는 template일 때 `siteUrl: undefined`, production일 때 `siteUrl: URL`입니다. |
| 후속 validator가 따라야 하는 return shape | `47b99d6256ef`이 production-specific helpers를 추가하고 `002b642d52a3`이 aggregate validator를 구현합니다. |
| 아직 생성되지 않는 readiness issues | public origin, placeholder, asset, project, contact를 실제 검사하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** schema-valid content를 template preview와 production publication으로 구분하는 공통 mode/error protocol이 없었습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-readiness.ts::resolvePortfolioContentMode`는 `undefined`, 빈 문자열, exact `template`을 template로, exact `production`만 production으로 반환하고 나머지는 throw합니다. Result는 template일 때 `siteUrl: undefined`, production일 때 `siteUrl: URL`입니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `47b99d6256ef`이 production-specific helpers를 추가하고 `002b642d52a3`이 aggregate validator를 구현합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** schema-valid content를 template preview와 production publication으로 구분하는 공통 mode/error protocol이 없었습니다. 누락되거나 오타인 environment 값을 production으로 추정하면 starter identity가 공개될 수 있고, 반대로 local preview를 strict production rule로 막을 수 있었습니다.
- **구현 결정과 경로:** conservative resolver와 discriminated readiness result/error types를 도입했습니다. `src/lib/content-readiness.ts::resolvePortfolioContentMode`는 `undefined`, 빈 문자열, exact `template`을 template로, exact `production`만 production으로 반환하고 나머지는 throw합니다. Result는 template일 때 `siteUrl: undefined`, production일 때 `siteUrl: URL`입니다.
- **소유권·실패 처리:** resolver가 mode interpretation을 소유하고 후속 validator가 issue/result protocol을 따르게 됩니다. unsupported 문자열은 fallback하지 않고 descriptive `Error`가 되며 이 SHA에는 아직 actual readiness issue를 수집하는 validator가 없습니다.
- **보장:** production activation이 explicit opt-in이고 result type이 verified URL의 유무를 mode와 연결합니다.
- **보장하지 않는 범위:** public origin, placeholder, asset, project, contact를 실제 검사하지 않습니다.
- **후속 연결:** `47b99d6256ef`이 production-specific helpers를 추가하고 `002b642d52a3`이 aggregate validator를 구현합니다.

### 2. `47b99d6256ef` — feat(content): public origin과 자산 경계 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Rejects local, reserved, credential-bearing, and misplaced publication inputs.
- **Source classification summary:** Add production-specific validation for public origins and locally served assets.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Production `SITE_URL`과 locally served asset의 publication boundary를 정의합니다. Origin은 public absolute HTTP(S)여야 하며 local, credential-bearing, reserved example host를 거부하고 asset은 `public/content` 아래에 있어야 합니다.

#### 해당 SHA에서 확인할 실제 코드

- `SITE_URL` parsing과 protocol/origin validation 순서를 확인합니다.
- Local/reserved host 판정 helper와 credential 검사 branch를 찾습니다.
- Origin만 허용하는지 path/query/hash가 있는 URL을 어떻게 처리하는지 확인합니다.
- Asset URL이 `public/content` namespace 아래인지 확인하는 boundary check를 추적합니다.
- Schema-level local path와 production-specific placement의 차이를 비교합니다.

확인 원칙:

- 먼저 `47b99d6256ef^`와 `47b99d6256ef`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Accept/reject URL examples와 branch | public site URL parser와 `/content/` asset namespace check를 추가했습니다. `src/lib/content-readiness.ts::parsePublicSiteUrl`은 absolute URL parsing 후 protocol이 HTTP(S)인지, localhost/127.0.0.1/::1/.localhost인지, example.com/net/org 및 `.example/.invalid/.test`인지, username/password가 비어 있는지 검사합니다. `addProductionAssetIssue`는 `/content/` prefix를 요구합니다. |
| Origin validation과 deployable public link validation 구분 | `src/lib/content-readiness.ts::parsePublicSiteUrl`은 absolute URL parsing 후 protocol이 HTTP(S)인지, localhost/127.0.0.1/::1/.localhost인지, example.com/net/org 및 `.example/.invalid/.test`인지, username/password가 비어 있는지 검사합니다. `addProductionAssetIssue`는 `/content/` prefix를 요구합니다. |
| Asset namespace와 publication 의미 | public site URL parser와 `/content/` asset namespace check를 추가했습니다. `src/lib/content-readiness.ts::parsePublicSiteUrl`은 absolute URL parsing 후 protocol이 HTTP(S)인지, localhost/127.0.0.1/::1/.localhost인지, example.com/net/org 및 `.example/.invalid/.test`인지, username/password가 비어 있는지 검사합니다. `addProductionAssetIssue`는 `/content/` prefix를 요구합니다. |
| 이 commit만으로 complete readiness가 되지 않는 이유 | complete content scan, project/contact completeness, actual asset existence와 build enforcement는 아직 없습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** mode protocol은 있었지만 `SITE_URL`과 production asset namespace를 판정하는 helper가 없었습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-readiness.ts::parsePublicSiteUrl`은 absolute URL parsing 후 protocol이 HTTP(S)인지, localhost/127.0.0.1/::1/.localhost인지, example.com/net/org 및 `.example/.invalid/.test`인지, username/password가 비어 있는지 검사합니다. `addProductionAssetIssue`는 `/content/` prefix를 요구합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `002b642d52a3`이 origin helper와 placeholder scan을 한 production result로 묶습니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** mode protocol은 있었지만 `SITE_URL`과 production asset namespace를 판정하는 helper가 없었습니다. localhost, reserved example host, credentials 또는 template asset path가 production identity/evidence로 사용될 수 있었습니다.
- **구현 결정과 경로:** public site URL parser와 `/content/` asset namespace check를 추가했습니다. `src/lib/content-readiness.ts::parsePublicSiteUrl`은 absolute URL parsing 후 protocol이 HTTP(S)인지, localhost/127.0.0.1/::1/.localhost인지, example.com/net/org 및 `.example/.invalid/.test`인지, username/password가 비어 있는지 검사합니다. `addProductionAssetIssue`는 `/content/` prefix를 요구합니다.
- **소유권·실패 처리:** URL parser가 publication origin 후보를, asset helper가 production-hosted local namespace를 판정합니다. missing/malformed URL과 public-origin predicate 실패는 environment issue로 누적됩니다. 관찰된 차이로 path/query/hash는 이 SHA에서 별도 거부하지 않고 parsed `URL`을 그대로 반환합니다.
- **보장:** local/reserved/credential-bearing/non-HTTP(S) URL과 `/content/` 밖 asset을 거부할 수 있습니다.
- **보장하지 않는 범위:** complete content scan, project/contact completeness, actual asset existence와 build enforcement는 아직 없습니다.
- **후속 연결:** `002b642d52a3`이 origin helper와 placeholder scan을 한 production result로 묶습니다.

### 3. `002b642d52a3` — feat(content): production readiness 기본 검사 추가

- **Importance:** S
- **Tags:** ARCH, CONTENT, VALIDATION
- **Source-defined thread role:** Aggregates origin and placeholder checks into a discriminated production result.
- **Source classification summary:** Introduce the aggregate production-readiness validator.
- **Source classification reason:** Critical because it establishes the fail-closed publication boundary: production mode exists only after origin and content readiness are verified across the complete source set.

#### Source에서 확정된 구현 의도와 상태 변화

Public origin validation과 모든 authoritative content file의 template marker scan을 aggregate production-readiness validator로 묶습니다. 모든 issue가 없을 때만 verified URL을 가진 production result를 반환합니다.

#### S-level source profile

- **Problem:** Schema-valid starter content도 template marker나 non-public origin을 포함해 실제 production claim으로 게시하기에는 부적합할 수 있었습니다.
- **Decision:** Public origin과 모든 authoritative source의 placeholder를 누적 검사하고 complete success 뒤에만 verified URL을 반환합니다.
- **Why it mattered:** Development validity와 publication validity를 분리해 build와 crawler output이 신뢰할 두 번째 trust level을 만듭니다.
- **What changed:** Readiness issue를 actionable set으로 보고하고 production branch를 discriminated result로 표현합니다.

#### 해당 SHA에서 확인할 실제 코드

- Aggregate validator의 input source, environment, issue collection, call order를 확인합니다.
- Origin failure가 issue로 누적되고 즉시 return하지 않는지 확인합니다.
- Source-to-file mapping과 recursive placeholder scanner가 JSON path를 만드는 흐름을 추적합니다.
- Issue가 하나라도 있을 때 aggregate error를 throw하고 result를 만들지 않는 branch를 확인합니다.
- Success branch에서 parsed `URL`과 discriminant가 반환되는지 확인합니다.
- Template mode handling과 production-only validator의 public API 관계를 구분합니다.
- 서로 다른 file/category failure를 둘 이상 주입해 ordering과 completeness를 기록합니다.

확인 원칙:

- 먼저 `002b642d52a3^`와 `002b642d52a3`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 mode protocol과 개별 helper가 분리된 구조 | mode protocol과 개별 origin/asset helpers는 있었지만 production 전체 source를 통과시키는 aggregate decision이 없었습니다. |
| Schema-valid starter가 production claim으로 노출될 수 있었던 문제 | schema-valid starter marker와 non-public origin이 그대로 metadata/indexing의 production claim이 될 수 있었습니다. |
| Aggregate fail-closed readiness와 verified URL decision | 14개 authoritative source의 recursive placeholder scan과 origin validation을 누적한 뒤 issue가 없을 때만 production result를 반환했습니다. |
| Validator, scanner traversal, issue aggregation, success result | 14개 authoritative source의 recursive placeholder scan과 origin validation을 누적한 뒤 issue가 없을 때만 production result를 반환했습니다. `src/lib/content-readiness.ts::validateProductionReadiness`는 `parsePublicSiteUrl` failure를 issue에 남긴 뒤 `contentFiles` 순서로 `collectPlaceholderIssues`를 재귀 호출합니다. `appendPath`가 object/array JSON path를 만들고 하나라도 issue가 있으면 `PortfolioReadinessError`, 없으면 `{ mode: "production", siteUrl }`입니다. |
| Bad origin와 multiple placeholders 동시 failure | bad origin이 있어도 placeholder scan을 계속하므로 서로 다른 category issue가 한 error에 함께 남습니다. Marker는 문자열당 첫 matching pattern을 report합니다. |
| 보장하는 것: origin/content-marker trust level | production result가 public-origin과 template-marker trust level을 통과했음을 보장합니다. |
| 아직 보장하지 않는 것: required assets/project exits/contact/build | required social/profile/resume assets, enabled project exits, contact, actual filesystem file, build gate와 crawler policy는 아직 보장하지 않습니다. |
| 후속 commit과 indexing policy 연결 | `bcd87ed856bf`와 `71e7ece7208f`이 portfolio-specific completeness를 강화하고 이후 build/SEO가 result를 소비합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** mode protocol과 개별 origin/asset helpers는 있었지만 production 전체 source를 통과시키는 aggregate decision이 없었습니다.
- **해당 SHA 핵심 코드:** `src/lib/content-readiness.ts::validateProductionReadiness`는 `parsePublicSiteUrl` failure를 issue에 남긴 뒤 `contentFiles` 순서로 `collectPlaceholderIssues`를 재귀 호출합니다. `appendPath`가 object/array JSON path를 만들고 하나라도 issue가 있으면 `PortfolioReadinessError`, 없으면 `{ mode: "production", siteUrl }`입니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `bcd87ed856bf`와 `71e7ece7208f`이 portfolio-specific completeness를 강화하고 이후 build/SEO가 result를 소비합니다.

#### SHA별 복원 결론

- **이전 상태:** mode protocol과 개별 origin/asset helpers는 있었지만 production 전체 source를 통과시키는 aggregate decision이 없었습니다.
- **문제와 위험:** schema-valid starter marker와 non-public origin이 그대로 metadata/indexing의 production claim이 될 수 있었습니다.
- **핵심 결정:** 14개 authoritative source의 recursive placeholder scan과 origin validation을 누적한 뒤 issue가 없을 때만 production result를 반환했습니다.
- **실제 구현 경로:** `src/lib/content-readiness.ts::validateProductionReadiness`는 `parsePublicSiteUrl` failure를 issue에 남긴 뒤 `contentFiles` 순서로 `collectPlaceholderIssues`를 재귀 호출합니다. `appendPath`가 object/array JSON path를 만들고 하나라도 issue가 있으면 `PortfolioReadinessError`, 없으면 `{ mode: "production", siteUrl }`입니다.
- **소유권·상태 변화:** validator가 source traversal과 aggregate decision을 소유하고 success result의 `URL`은 모든 현재 check가 완료된 뒤에만 생성됩니다.
- **실패 경계:** bad origin이 있어도 placeholder scan을 계속하므로 서로 다른 category issue가 한 error에 함께 남습니다. Marker는 문자열당 첫 matching pattern을 report합니다.
- **보장:** production result가 public-origin과 template-marker trust level을 통과했음을 보장합니다.
- **보장하지 않는 범위:** required social/profile/resume assets, enabled project exits, contact, actual filesystem file, build gate와 crawler policy는 아직 보장하지 않습니다.
- **후속 fix/test:** `bcd87ed856bf`와 `71e7ece7208f`이 portfolio-specific completeness를 강화하고 이후 build/SEO가 result를 소비합니다.

### 4. `bcd87ed856bf` — feat(content): 필수 자산과 프로젝트 readiness 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Source-defined thread role:** Requires real public evidence and exits for every published project.
- **Source classification summary:** Extend production readiness from generic placeholder detection to portfolio-specific completeness.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Readiness에 portfolio-specific completeness를 추가합니다. Social/profile/résumé asset, enabled project, production-hosted screenshots, 각 published project의 enabled public link를 요구하며 disabled project는 제외합니다.

#### 해당 SHA에서 확인할 실제 코드

- 필수 site/profile/resume asset field와 `public/content` check를 확인합니다.
- Enabled project collection과 최소 한 개 요구 branch를 추적합니다.
- Project screenshots의 production-hosted check와 issue path를 찾습니다.
- 각 enabled project에서 usable public link를 선택하는 protocol/enablement 조건을 확인합니다.
- Disabled project가 검사에서 제외되는 위치와 범위를 기록합니다.

확인 원칙:

- 먼저 `bcd87ed856bf^`와 `bcd87ed856bf`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Visible publication evidence의 최소 조건 | 공개 대상으로 선택된 project가 production-hosted evidence와 최소 하나의 public exit를 가집니다. |
| Project-level exit의 의미와 accepted protocol | visible publication evidence와 enabled project별 exit를 production readiness에 추가했습니다. `validateProductionReadiness`가 site.socialImage, profile.photo.src, resume.downloadUrl을 `/content/`로 검사하고 enabled projects가 하나 이상인지 확인합니다. Enabled project의 lead/secondary screenshots와 enabled HTTP(S) non-reserved URL link를 검사하며 `enabled === false` project는 skip합니다. |
| Disabled staging exemption | visible publication evidence와 enabled project별 exit를 production readiness에 추가했습니다. `validateProductionReadiness`가 site.socialImage, profile.photo.src, resume.downloadUrl을 `/content/`로 검사하고 enabled projects가 하나 이상인지 확인합니다. Enabled project의 lead/secondary screenshots와 enabled HTTP(S) non-reserved URL link를 검사하며 `enabled === false` project는 skip합니다. |
| 여러 completeness issue의 accumulation | visible publication evidence와 enabled project별 exit를 production readiness에 추가했습니다. `validateProductionReadiness`가 site.socialImage, profile.photo.src, resume.downloadUrl을 `/content/`로 검사하고 enabled projects가 하나 이상인지 확인합니다. Enabled project의 lead/secondary screenshots와 enabled HTTP(S) non-reserved URL link를 검사하며 `enabled === false` project는 skip합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** origin과 placeholder만 통과하면 evidence가 없는 portfolio도 production result를 얻을 수 있었습니다.
- **해당 SHA 핵심 코드:** `validateProductionReadiness`가 site.socialImage, profile.photo.src, resume.downloadUrl을 `/content/`로 검사하고 enabled projects가 하나 이상인지 확인합니다. Enabled project의 lead/secondary screenshots와 enabled HTTP(S) non-reserved URL link를 검사하며 `enabled === false` project는 skip합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `71e7ece7208f`이 contact와 mode-aware build entry를 추가합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** origin과 placeholder만 통과하면 evidence가 없는 portfolio도 production result를 얻을 수 있었습니다. social/profile/resume asset, enabled project, screenshot, public exit가 없으면 index 가능한 portfolio claim으로서 불완전했습니다.
- **구현 결정과 경로:** visible publication evidence와 enabled project별 exit를 production readiness에 추가했습니다. `validateProductionReadiness`가 site.socialImage, profile.photo.src, resume.downloadUrl을 `/content/`로 검사하고 enabled projects가 하나 이상인지 확인합니다. Enabled project의 lead/secondary screenshots와 enabled HTTP(S) non-reserved URL link를 검사하며 `enabled === false` project는 skip합니다.
- **소유권·실패 처리:** readiness validator가 publication completeness를 소유하고 disabled project는 staging data로 남을 수 있습니다. 누락 asset, zero enabled project, project별 usable link 부재가 각각 source/path issue로 누적됩니다.
- **보장:** 공개 대상으로 선택된 project가 production-hosted evidence와 최소 하나의 public exit를 가집니다.
- **보장하지 않는 범위:** 실제 file existence, contact method, build enforcement와 crawler consistency는 아직 보장하지 않습니다.
- **후속 연결:** `71e7ece7208f`이 contact와 mode-aware build entry를 추가합니다.

### 5. `71e7ece7208f` — feat(content): 연락 수단과 build readiness 연결

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, DEPLOY
- **Source-defined thread role:** Requires a usable contact method and exposes one mode-aware gate.
- **Source classification summary:** Require at least one enabled, non-placeholder contact method in production and expose a single mode-aware build-readiness entry point.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Production에 enabled non-placeholder contact method를 요구하고 template mode는 strict publication check를 생략하며 production mode는 aggregate validator로 위임하는 하나의 mode-aware build-readiness entry point를 제공합니다.

#### 해당 SHA에서 확인할 실제 코드

- Preferred contact IDs와 enabled global links를 어떻게 확인하는지 추적합니다.
- `mailto:`, `tel:`, HTTP(S) 허용과 placeholder rejection을 확인합니다.
- Template branch가 publication requirements 없이 반환하는 result를 확인합니다.
- Production branch가 full validator에 delegate하고 verified URL을 보존하는지 확인합니다.
- Public exports가 core contract로 축소되는 diff를 확인합니다.

확인 원칙:

- 먼저 `71e7ece7208f^`와 `71e7ece7208f`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Usable contact method의 exact predicate | usable contact predicate와 `validateBuildReadiness` mode-aware entry를 추가했습니다. |
| Template/production execution path 차이 | `isUsableContactHref`는 placeholder를 거부하고 `mailto:`, `tel:` 또는 usable HTTP(S) URL을 허용합니다. 실제 contact record는 enabled, `placements`에 `contact`, type이 `email\|github\|website` 중 하나여야 합니다. `validateBuildReadiness`는 template이면 `{mode, siteUrl:undefined}`, production이면 full validator로 delegate합니다. |
| Mode-aware public API와 private helper | usable contact predicate와 `validateBuildReadiness` mode-aware entry를 추가했습니다. `isUsableContactHref`는 placeholder를 거부하고 `mailto:`, `tel:` 또는 usable HTTP(S) URL을 허용합니다. 실제 contact record는 enabled, `placements`에 `contact`, type이 `email\|github\|website` 중 하나여야 합니다. `validateBuildReadiness`는 template이면 `{mode, siteUrl:undefined}`, production이면 full validator로 delegate합니다. |
| Build script가 호출해야 하는 단일 entry point | `isUsableContactHref`는 placeholder를 거부하고 `mailto:`, `tel:` 또는 usable HTTP(S) URL을 허용합니다. 실제 contact record는 enabled, `placements`에 `contact`, type이 `email\|github\|website` 중 하나여야 합니다. `validateBuildReadiness`는 template이면 `{mode, siteUrl:undefined}`, production이면 full validator로 delegate합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** portfolio-specific project evidence는 검사했지만 사용자가 연락할 수 있는 public method와 template/production 공용 entry가 없었습니다.
- **해당 SHA 핵심 코드:** `isUsableContactHref`는 placeholder를 거부하고 `mailto:`, `tel:` 또는 usable HTTP(S) URL을 허용합니다. 실제 contact record는 enabled, `placements`에 `contact`, type이 `email|github|website` 중 하나여야 합니다. `validateBuildReadiness`는 template이면 `{mode, siteUrl:undefined}`, production이면 full validator로 delegate합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `37c0dbc079ff`이 이 단일 entry를 `prebuild`에 연결합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** portfolio-specific project evidence는 검사했지만 사용자가 연락할 수 있는 public method와 template/production 공용 entry가 없었습니다. 프로젝트는 공개됐지만 contact link가 disabled, wrong placement/type, placeholder 또는 unusable URL일 수 있었습니다.
- **구현 결정과 경로:** usable contact predicate와 `validateBuildReadiness` mode-aware entry를 추가했습니다. `isUsableContactHref`는 placeholder를 거부하고 `mailto:`, `tel:` 또는 usable HTTP(S) URL을 허용합니다. 실제 contact record는 enabled, `placements`에 `contact`, type이 `email|github|website` 중 하나여야 합니다. `validateBuildReadiness`는 template이면 `{mode, siteUrl:undefined}`, production이면 full validator로 delegate합니다.
- **소유권·실패 처리:** private helpers가 URL/contact details를 소유하고 public caller는 mode-aware gate 하나를 호출합니다. usable contact가 하나도 없으면 `src/content/links.json:$` issue가 누적됩니다. Template path는 strict publication checks를 의도적으로 생략합니다.
- **보장:** production은 project evidence와 usable contact를 포함한 full current readiness를 통과하고 template preview는 계속 가능합니다.
- **보장하지 않는 범위:** actual build lifecycle 연결과 indexing output 일치는 아직 보장하지 않습니다.
- **후속 연결:** `37c0dbc079ff`이 이 단일 entry를 `prebuild`에 연결합니다.

### 6. `37c0dbc079ff` — build(content): readiness 검사를 prebuild에 연결

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, DEPLOY
- **Source-defined thread role:** Makes publication completeness mandatory during normal production builds.
- **Source classification summary:** Make content readiness a mandatory prebuild gate after schema validation.
- **Source classification reason:** Significant because it strengthens the shared content trust boundary or a cross-file invariant used by every route, rather than validating one local component.

#### Source에서 확정된 구현 의도와 상태 변화

Schema validation 뒤 content readiness를 수행하는 dedicated script를 `prebuild`에 연결합니다. Selected mode와 production origin을 출력하고 readiness issue는 non-zero process status로 변환합니다.

#### 해당 SHA에서 확인할 실제 코드

- Package `prebuild`에서 structural `content:check`와 readiness check 순서를 확인합니다.
- Readiness CLI가 authoritative source와 environment를 어떻게 전달하는지 추적합니다.
- Template success와 production success의 output 차이를 확인합니다.
- Aggregate error가 stderr/exit code로 변환되는 catch branch를 찾습니다.
- Failure에서 Next build가 시작되지 않는 실행 결과를 기록합니다.

확인 원칙:

- 먼저 `37c0dbc079ff^`와 `37c0dbc079ff`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Schema validity와 publication readiness gate 순서 | `scripts/validate-content-readiness.ts`가 authoritative source와 `PORTFOLIO_CONTENT_MODE`/`SITE_URL`을 `validateBuildReadiness`에 전달합니다. `package.json`의 `prebuild`는 `content:check && content:ready` 순서입니다. |
| Mode/env input source | `scripts/validate-content-readiness.ts`가 authoritative source와 `PORTFOLIO_CONTENT_MODE`/`SITE_URL`을 `validateBuildReadiness`에 전달합니다. `package.json`의 `prebuild`는 `content:check && content:ready` 순서입니다. |
| Success/failure output과 exit code | template은 indexing-disabled/strict-check-skip message, production은 verified origin을 출력합니다. `PortfolioReadinessError`는 stderr와 `process.exitCode = 1`로 변환되고 unknown error는 rethrow되어 Next build가 시작되지 않습니다. |
| Normal build lifecycle에서 우회할 수 없는 이유 | npm lifecycle이 order/failure propagation을, readiness CLI가 environment input과 user-facing output을 소유합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** readiness API는 존재했지만 normal build가 호출하지 않으면 production completeness를 우회할 수 있었습니다.
- **해당 SHA 핵심 코드:** `scripts/validate-content-readiness.ts`가 authoritative source와 `PORTFOLIO_CONTENT_MODE`/`SITE_URL`을 `validateBuildReadiness`에 전달합니다. `package.json`의 `prebuild`는 `content:check && content:ready` 순서입니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `55b6061e0052`부터 동일 mode/result가 crawler-visible policy에 사용됩니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** readiness API는 존재했지만 normal build가 호출하지 않으면 production completeness를 우회할 수 있었습니다. schema/asset check만 통과한 starter content가 Next compilation에 들어갈 수 있었습니다.
- **구현 결정과 경로:** structural `content:check` 뒤 mode-aware `content:ready`를 실행하는 mandatory prebuild chain을 만들었습니다. `scripts/validate-content-readiness.ts`가 authoritative source와 `PORTFOLIO_CONTENT_MODE`/`SITE_URL`을 `validateBuildReadiness`에 전달합니다. `package.json`의 `prebuild`는 `content:check && content:ready` 순서입니다.
- **소유권·실패 처리:** npm lifecycle이 order/failure propagation을, readiness CLI가 environment input과 user-facing output을 소유합니다. template은 indexing-disabled/strict-check-skip message, production은 verified origin을 출력합니다. `PortfolioReadinessError`는 stderr와 `process.exitCode = 1`로 변환되고 unknown error는 rethrow되어 Next build가 시작되지 않습니다.
- **보장:** normal production build가 structural check 후 publication readiness를 반드시 통과합니다.
- **보장하지 않는 범위:** 직접 `next build` binary를 lifecycle 밖에서 호출하는 외부 우회나 deployment infrastructure 자체는 보장하지 않습니다.
- **후속 연결:** `55b6061e0052`부터 동일 mode/result가 crawler-visible policy에 사용됩니다.

### 7. `55b6061e0052` — feat(seo): 콘텐츠 mode별 metadata 정책 추가

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Source-defined thread role:** Aligns page indexing with the validated content mode.
- **Source classification summary:** Add a pure metadata factory driven by validated site content and the selected content mode.
- **Source classification reason:** Significant because it governs publication identity, indexing, or safe machine-readable output across routes, keeping crawler-visible state aligned with validated content.

#### Source에서 확정된 구현 의도와 상태 변화

Validated site content와 content mode를 입력으로 canonical, Open Graph, Twitter, optional social image metadata를 만드는 pure factory를 추가하고 indexing은 production에서만 허용합니다.

#### 해당 SHA에서 확인할 실제 코드

- Factory input에 site data, mode, base URL이 어떻게 표현되는지 확인합니다.
- Template와 production robots/indexing field 차이를 추적합니다.
- Canonical/Open Graph/Twitter URL과 image가 같은 origin에서 파생되는지 확인합니다.
- Optional social image absent branch가 unsupported claim을 만들지 않는지 확인합니다.
- Pure helper가 request headers/filesystem을 직접 읽지 않는지 확인합니다.

확인 원칙:

- 먼저 `55b6061e0052^`와 `55b6061e0052`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Mode별 metadata/robots 결과 | validated site content, content mode, base URL을 입력으로 하는 pure metadata factory를 추가했습니다. metadata helper는 canonical, Open Graph, Twitter와 optional social image absolute URL을 한 base에서 만들고 robots `index/follow`를 production에서만 true로 설정합니다. Social image가 없으면 image field를 만들지 않습니다. |
| Canonical identity와 social URL source | metadata helper는 canonical, Open Graph, Twitter와 optional social image absolute URL을 한 base에서 만들고 robots `index/follow`를 production에서만 true로 설정합니다. Social image가 없으면 image field를 만들지 않습니다. |
| Optional field omission policy | validated site content, content mode, base URL을 입력으로 하는 pure metadata factory를 추가했습니다. |
| Factory testability와 readiness dependency | metadata helper는 canonical, Open Graph, Twitter와 optional social image absolute URL을 한 base에서 만들고 robots `index/follow`를 production에서만 true로 설정합니다. Social image가 없으면 image field를 만들지 않습니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** build가 production readiness를 알지만 page metadata가 template/production을 별도로 추정할 수 있었습니다.
- **해당 SHA 핵심 코드:** metadata helper는 canonical, Open Graph, Twitter와 optional social image absolute URL을 한 base에서 만들고 robots `index/follow`를 production에서만 true로 설정합니다. Social image가 없으면 image field를 만들지 않습니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `cb61450ad922`과 `70b69f04e8c7`이 같은 contract를 robots/sitemap에 적용합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** build가 production readiness를 알지만 page metadata가 template/production을 별도로 추정할 수 있었습니다. template preview가 canonical/robots index claim을 생성하거나 social URL이 다른 origin을 사용할 위험이 있었습니다.
- **구현 결정과 경로:** validated site content, content mode, base URL을 입력으로 하는 pure metadata factory를 추가했습니다. metadata helper는 canonical, Open Graph, Twitter와 optional social image absolute URL을 한 base에서 만들고 robots `index/follow`를 production에서만 true로 설정합니다. Social image가 없으면 image field를 만들지 않습니다.
- **소유권·실패 처리:** factory input caller가 mode/base URL을 제공하고 pure helper는 request headers/filesystem을 직접 읽지 않습니다. optional field absence는 omission으로 처리하며 template은 canonical metadata가 있어도 index/follow를 허용하지 않습니다.
- **보장:** page metadata의 publication identity와 indexing flag가 같은 validated mode를 따릅니다.
- **보장하지 않는 범위:** `robots.txt`와 sitemap discovery까지의 외부 crawler policy는 아직 분리돼 있습니다.
- **후속 연결:** `cb61450ad922`과 `70b69f04e8c7`이 같은 contract를 robots/sitemap에 적용합니다.

### 8. `cb61450ad922` — feat(seo): 콘텐츠 mode별 robots 정책 추가

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Source-defined thread role:** Aligns crawler policy with the same mode contract.
- **Source classification summary:** Generate `robots.txt` from the same content-mode contract used by readiness and metadata.
- **Source classification reason:** Significant because it governs publication identity, indexing, or safe machine-readable output across routes, keeping crawler-visible state aligned with validated content.

#### Source에서 확정된 구현 의도와 상태 변화

같은 content-mode contract로 `robots.txt`를 생성합니다. Template은 모든 crawler를 차단하고 production은 validated host를 명시해 indexing을 허용하며 production URL 누락은 programming error입니다.

#### 해당 SHA에서 확인할 실제 코드

- Robots generator의 template/production branch와 returned rule shape를 확인합니다.
- Production host가 verified URL에서 추출되는지 확인합니다.
- Production mode인데 URL이 없는 impossible state를 어떤 error로 막는지 추적합니다.
- Metadata indexing branch와 robots 결과를 같은 input으로 비교합니다.
- Sitemap advertisement는 후속 commit과 구분합니다.

확인 원칙:

- 먼저 `cb61450ad922^`와 `cb61450ad922`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Template disallow-all 정책 | content mode와 verified production URL로 robots output을 생성했습니다. |
| Production allow/host 정책 | content mode와 verified production URL로 robots output을 생성했습니다. |
| Impossible state 방어 | readiness result가 production URL ownership을, robots route가 crawler rule serialization을 소유합니다. |
| Metadata와 crawler policy 일관성 | content mode와 verified production URL로 robots output을 생성했습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** metadata robots flag는 정리됐지만 `/robots.txt`가 같은 mode/result를 소비하지 않았습니다.
- **해당 SHA 핵심 코드:** robots generator는 template에서 user-agent `*`, disallow `/`; production에서 allow `/`와 verified URL의 host를 반환합니다. Production인데 URL이 없으면 impossible programming state로 error를 throw합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `70b69f04e8c7`이 enabled publication surface만 sitemap으로 노출합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** metadata robots flag는 정리됐지만 `/robots.txt`가 같은 mode/result를 소비하지 않았습니다. page metadata는 non-indexable인데 crawler policy는 허용하거나 반대 상태가 될 수 있었습니다.
- **구현 결정과 경로:** content mode와 verified production URL로 robots output을 생성했습니다. robots generator는 template에서 user-agent `*`, disallow `/`; production에서 allow `/`와 verified URL의 host를 반환합니다. Production인데 URL이 없으면 impossible programming state로 error를 throw합니다.
- **소유권·실패 처리:** readiness result가 production URL ownership을, robots route가 crawler rule serialization을 소유합니다. template는 fail-closed disallow-all이고 impossible discriminant/URL mismatch는 조용한 fallback 없이 실패합니다.
- **보장:** metadata와 robots가 template/production indexing decision에서 일치합니다.
- **보장하지 않는 범위:** enabled route discovery와 sitemap advertisement는 아직 추가되지 않았습니다.
- **후속 연결:** `70b69f04e8c7`이 enabled publication surface만 sitemap으로 노출합니다.

### 9. `70b69f04e8c7` — feat(seo): 공개 route sitemap 생성

- **Importance:** A
- **Tags:** ARCH, ROUTING, SEO
- **Source-defined thread role:** Publishes only enabled production routes and projects.
- **Source classification summary:** Generate `sitemap.xml` from the validated production origin and the routes that the current content configuration actually exposes.
- **Source classification reason:** Significant because it governs publication identity, indexing, or safe machine-readable output across routes, keeping crawler-visible state aligned with validated content.

#### Source에서 확정된 구현 의도와 상태 변화

Validated production origin과 content configuration이 실제로 노출하는 route만으로 `sitemap.xml`을 생성합니다. Template에서는 entry가 없고 disabled page/project는 제외하며 robots가 sitemap을 광고합니다.

#### 해당 SHA에서 확인할 실제 코드

- Template mode에서 sitemap entries가 빈 collection이 되는 branch를 확인합니다.
- Core/optional page URL과 availability check를 추적합니다.
- Enabled project ID로 detail URL을 생성하는 loop와 URL join을 확인합니다.
- Disabled route/project가 제외되는 predicate를 확인합니다.
- Robots output에 sitemap URL을 추가하는 integration 지점을 찾습니다.

확인 원칙:

- 먼저 `70b69f04e8c7^`와 `70b69f04e8c7`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Published route set의 source-of-truth | `src/app/sitemap.ts`는 template에서 `[]`, production에서 core route와 enabled optional pages, enabled project IDs의 detail URLs를 만듭니다. `src/app/robots.ts`는 production에서 sitemap URL도 광고합니다. |
| Template/production sitemap 차이 | `src/app/sitemap.ts`는 template에서 `[]`, production에서 core route와 enabled optional pages, enabled project IDs의 detail URLs를 만듭니다. `src/app/robots.ts`는 production에서 sitemap URL도 광고합니다. |
| Optional page와 project detail inclusion rules | validated production origin과 current content availability에서 sitemap entries를 동적으로 생성했습니다. `src/app/sitemap.ts`는 template에서 `[]`, production에서 core route와 enabled optional pages, enabled project IDs의 detail URLs를 만듭니다. `src/app/robots.ts`는 production에서 sitemap URL도 광고합니다. |
| Rendering availability와 crawl discovery의 동일성 | validated production origin과 current content availability에서 sitemap entries를 동적으로 생성했습니다. `src/app/sitemap.ts`는 template에서 `[]`, production에서 core route와 enabled optional pages, enabled project IDs의 detail URLs를 만듭니다. `src/app/robots.ts`는 production에서 sitemap URL도 광고합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** robots는 production indexing을 허용하지만 실제 enabled route/project set을 알리는 sitemap이 없었습니다.
- **해당 SHA 핵심 코드:** `src/app/sitemap.ts`는 template에서 `[]`, production에서 core route와 enabled optional pages, enabled project IDs의 detail URLs를 만듭니다. `src/app/robots.ts`는 production에서 sitemap URL도 광고합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** 이 commit으로 build, metadata, robots, sitemap의 publication decision chain이 완성됩니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** robots는 production indexing을 허용하지만 실제 enabled route/project set을 알리는 sitemap이 없었습니다. disabled optional page나 project가 고정 sitemap에 남으면 rendering availability와 crawler discovery가 어긋납니다.
- **구현 결정과 경로:** validated production origin과 current content availability에서 sitemap entries를 동적으로 생성했습니다. `src/app/sitemap.ts`는 template에서 `[]`, production에서 core route와 enabled optional pages, enabled project IDs의 detail URLs를 만듭니다. `src/app/robots.ts`는 production에서 sitemap URL도 광고합니다.
- **소유권·실패 처리:** content configuration이 route availability를 소유하고 sitemap은 그 predicate를 재사용해 absolute URLs를 직렬화합니다. disabled page/project는 제외되고 production URL이 없는 impossible state는 upstream result contract로 막힙니다.
- **보장:** crawler discovery가 실제 enabled publication surface와 동일한 mode/origin을 사용합니다.
- **보장하지 않는 범위:** 검색 엔진의 실제 crawl/index 결과나 external canonical correctness는 보장하지 않습니다.
- **후속 연결:** 이 commit으로 build, metadata, robots, sitemap의 publication decision chain이 완성됩니다.

## 6. Invariant ledger

| Invariant | Source에서 확인된 변화 지점 | 해당 SHA의 실제 code/test evidence | 부족함이 드러난 시점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Missing/empty/explicit template mode는 template이며 exact production만 production을 요청합니다. | `b3bd671a3243` | `resolvePortfolioContentMode`: `undefined\|""\|"template"` → template, exact `"production"` → production, else throw. | 이전에는 consumer가 environment string을 각자 추정할 수 있었습니다. | production은 explicit opt-in이며 mode/result discriminant가 URL 유무와 연결됩니다. |
| Production origin은 public HTTP(S)이고 local/reserved/credential-bearing 값이 아닙니다. | `47b99d6256ef` | `parsePublicSiteUrl`의 URL parse, HTTP(S), local/reserved host, credentials branches와 `/content/` asset helper. | schema-valid URL/string만으로 public production origin/asset placement를 보장하지 못했습니다. | 현재 validator는 public-looking origin과 production asset namespace를 요구합니다. Path/query/hash는 이 SHA에서 별도 거부하지 않습니다. |
| Production result는 complete readiness 성공 뒤에만 verified URL을 포함합니다. | `002b642d52a3` | `validateProductionReadiness`: origin issue accumulation, 14-file recursive placeholder scan, issue-empty success에서만 `{mode:"production", siteUrl}`. | mode/error protocol과 helpers만 있을 때 complete publication decision은 없었습니다. | success result의 URL은 현재 모든 production readiness check를 통과한 뒤에만 노출됩니다. |
| Published project는 실제 자산과 enabled public exit를 가지며 usable contact가 존재합니다. | `bcd87ed856bf → 71e7ece7208f` | required social/profile/resume assets, enabled project count/screenshots/public links와 enabled contact predicate. | origin/placeholder만 통과해도 evidence/contact가 없는 portfolio가 production이 될 수 있었습니다. | enabled publication records만 검사하며 published project와 contact의 minimum evidence/exit를 요구합니다. |
| Readiness는 normal production build의 mandatory gate입니다. | `37c0dbc079ff` | `package.json::prebuild` → `content:check && content:ready`; CLI의 mode/env 전달 및 non-zero handling. | readiness API가 manual command이면 normal build에서 우회할 수 있었습니다. | npm normal build lifecycle은 schema/asset check 후 mode-aware readiness를 통과해야 합니다. |
| Template는 non-indexable이고 crawler output은 enabled publication surface만 반영합니다. | `55b6061e0052 → cb61450ad922 → 70b69f04e8c7` | metadata factory, robots route, sitemap route가 같은 mode/verified URL/content availability를 사용합니다. | build decision과 crawler-visible outputs가 서로 다른 조건을 사용할 수 있었습니다. | template는 noindex/disallow/empty sitemap이며 production은 enabled routes/projects만 canonical/crawl discovery에 포함합니다. |

Ledger 작성 원칙:

- Source가 명시한 invariant만 사용하고 새 invariant를 확정 사실처럼 추가하지 않습니다.
- 도입, 강화, 부족함 노출, fix, regression test가 서로 다른 commit이면 각 열에 분리해 기록합니다.
- Code evidence에는 실제 field, function, branch, selector, command 또는 assertion을 적습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 위험 | 대응 commit | 실제 수정/강화 code에서 확인할 것 | Test 또는 실행 증거 |
| --- | --- | --- | --- |
| Environment 값이 비어 있거나 오타인데 production으로 추정함 | b3bd671a3243 | Exact-value resolver와 unsupported-value error를 확인합니다. | Production branch만 URL을 요구하는 discriminated type을 기록합니다. |
| Schema는 통과하지만 example host, localhost, credential URL, starter marker가 남음 | 47b99d6256ef, 002b642d52a3 | Origin predicate, placeholder traversal, issue accumulation을 확인합니다. | 여러 category가 동시에 실패할 때 aggregate error를 기록합니다. |
| Published project에 screenshot/exit가 없거나 contact가 없음 | bcd87ed856bf, 71e7ece7208f | Enabled/disabled 분기와 link protocol/placeholder 판정을 추적합니다. | Disabled staging record가 exempt되는 branch를 확인합니다. |
| Template preview가 index되거나 disabled route가 sitemap에 남음 | 55b6061e0052, cb61450ad922, 70b69f04e8c7 | Metadata, `robots.txt`, sitemap을 같은 mode로 대조합니다. | Template에서 sitemap이 비고 crawler가 disallow되는지 기록합니다. |

### 실제 연결 기록

- Unsupported mode는 template fallback이 아니라 immediate error입니다. Production origin issue와 content placeholders는 한 aggregate report에 함께 들어갑니다.
- `47b99d6256ef`의 parser는 path/query/hash를 거부하거나 origin으로 normalize하지 않고 parsed URL을 그대로 반환합니다. 문서에서는 이를 실제 구현 범위로 제한했습니다.
- Contact URL helper는 `mailto:`, `tel:`, usable HTTP(S)를 허용하지만 contact record type filter는 `email|github|website`와 contact placement를 별도로 요구합니다.

## 8. Ownership / state / responsibility 변화

| Concern | Thread 초기 owner/state | Thread 최종 owner/state | 실제 symbol과 호출 경로 |
| --- | --- | --- | --- |
| Content mode 해석 | Consumer별 환경 문자열 해석 | 공용 mode resolver와 error model | `src/lib/content-readiness.ts::resolvePortfolioContentMode` → discriminated template/production path. |
| Public origin/asset policy | Schema 또는 metadata의 추정 | Production-specific readiness validator | `parsePublicSiteUrl`, `addProductionAssetIssue`, `validateProductionReadiness`가 production-specific trust를 소유합니다. |
| Production completeness | Page/SEO별 개별 방어 | Aggregate readiness result | `validateBuildReadiness`가 template short-circuit와 production aggregate validator를 하나의 public entry로 묶습니다. |
| Build enforcement | Optional command | `prebuild` readiness script | `package.json::prebuild` → readiness CLI → `validateBuildReadiness`; issue error가 process status로 전파됩니다. |
| Indexing policy | Metadata/robots/sitemap별 조건 | 공통 content-mode contract | layout metadata helper, `src/app/robots.ts`, `src/app/sitemap.ts`가 같은 mode/siteUrl/availability result를 직렬화합니다. |

## 9. Thread 최종 상태

### Source에서 확정된 최종 상태

Schema-valid starter content와 실제 공개 가능한 production content를 구분하고 strict readiness 결과를 build, metadata, robots, sitemap에 일관되게 적용하는 publication trust boundary를 복원합니다.

### 학습자가 완성할 최종 설명

- **Thread 시작 시점의 설계와 위험:** 시작 시점에는 schema-valid starter data와 실제로 공개 가능한 identity/evidence를 구분하지 않아 template placeholder가 production claim이 될 위험이 있었습니다.
- **핵심 architecture/decision이 형성된 순서:** conservative mode resolver, public-origin/asset helpers, aggregate placeholder scan, project/contact completeness, prebuild gate, metadata/robots/sitemap policy 순서로 publication boundary를 만들었습니다.
- **실제 failure 또는 부족함이 드러난 지점:** origin과 marker만 검사한 초기 aggregate validator는 required assets, enabled exits와 contact를 보장하지 않아 후속 commits에서 completeness가 강화됐습니다.
- **Fix 또는 boundary 강화가 바꾼 invariant:** `validateBuildReadiness`와 `prebuild`가 우회를 줄이고 crawler outputs가 같은 result를 소비해 build state와 indexing state의 모순을 막습니다.
- **Test/build/browser evidence가 보장한 범위:** 각 exact SHA의 predicates, issue accumulation과 command wiring을 정적으로 검토했습니다. build/robots/sitemap command는 실행하지 않았습니다.
- **Thread 종료 시점에도 보장하지 않는 범위:** 실제 asset filesystem/HTTP serving, search engine indexing 결과, external proxy/canonical configuration과 human content quality는 별도 보장입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 source-backed flow의 각 단계에 실제 file path, symbol, input/output, failure branch를 추가합니다.

1. Environment에서 content mode를 exact policy로 해석합니다.
   - 실제 코드 위치: `resolvePortfolioContentMode`
   - 입력과 출력: `PORTFOLIO_CONTENT_MODE` raw string을 template 또는 production discriminant로 변환합니다.
   - 실패/absence 처리: unsupported value는 fallback 없이 throw합니다.
2. Template mode는 publication requirements 없이 non-indexable preview 결과를 반환합니다.
   - 실제 코드 위치: `validateBuildReadiness` template branch + metadata/robots/sitemap consumers
   - 입력과 출력: template result는 `siteUrl: undefined`; preview render는 허용하되 indexing은 false, robots disallow, sitemap empty입니다.
   - 실패/absence 처리: publication requirements를 의도적으로 실행하지 않으며 crawler-visible claim을 만들지 않습니다.
3. Production mode는 `SITE_URL`, placeholder, asset, project, contact 검사를 누적합니다.
   - 실제 코드 위치: `validateProductionReadiness`와 helper predicates
   - 입력과 출력: `SITE_URL`, 14 content files, required assets, enabled projects/screenshots/links/contact를 issue array에 누적합니다.
   - 실패/absence 처리: 각 invalid field는 source/path/message issue가 되며 bad origin이 있어도 나머지 scan을 계속합니다.
4. Issue가 없을 때만 verified public `URL`을 포함한 production result를 만듭니다.
   - 실제 코드 위치: `validateProductionReadiness` success branch
   - 입력과 출력: issue가 없고 parsed URL이 있을 때만 `{mode:"production", siteUrl: URL}`을 반환합니다.
   - 실패/absence 처리: URL이 없거나 issue가 하나라도 있으면 `PortfolioReadinessError`를 throw합니다.
5. Prebuild가 schema validation 뒤 readiness를 실행하고 failure를 전파합니다.
   - 실제 코드 위치: `package.json::prebuild`, `scripts/validate-content-readiness.ts`
   - 입력과 출력: schema/asset check 다음 readiness CLI가 mode/env와 authoritative source를 검사합니다.
   - 실패/absence 처리: known readiness error는 stderr + exitCode 1, unknown error는 rethrow되어 Next build를 중단합니다.
6. Layout metadata, robots, sitemap이 같은 mode/URL/availability를 소비합니다.
   - 실제 코드 위치: metadata factory, `src/app/robots.ts`, `src/app/sitemap.ts`
   - 입력과 출력: verified mode/site URL과 page/project availability로 canonical/social/robots/sitemap outputs를 만듭니다.
   - 실패/absence 처리: production URL 없는 impossible state는 error이고 optional/disabled records는 omission됩니다.
7. Template와 disabled publication surface는 crawler-visible output에서 제외됩니다.
   - 실제 코드 위치: crawler-visible template/disabled branches
   - 입력과 출력: template는 non-indexable, disabled pages/projects는 production sitemap에서 제외됩니다.
   - 실패/absence 처리: rendering availability와 discovery set이 다를 경우 shared predicate/test가 regression을 드러냅니다.

### 코드 없이 설명하기

> 이 Thread의 최종 실행 흐름을 code snippet 없이 자신의 말로 작성합니다. 설계 → 구현 → failure/risk → 수정/강화 → 검증 순서가 드러나야 합니다.

runtime schema를 통과한 starter content라도 공개 가능한 portfolio라는 뜻은 아닙니다. 그래서 mode를 conservative하게 해석하고 production에서만 public URL, placeholder, assets, enabled project exits와 contact를 누적 검사했습니다. 모든 조건이 성공한 뒤에만 verified URL이 있는 production result를 만들고 normal prebuild에서 이 gate를 실행합니다. Metadata, robots와 sitemap도 같은 mode/result 및 enabled content를 사용하므로 template는 preview 가능하지만 index되지 않고 disabled publication surface는 discovery되지 않습니다.

> **실행 증거 구분:** 참조 SHA의 diff와 해당 SHA 파일은 GitHub connector로 확인했습니다. 로컬 clone은 실행 환경의 DNS 차단으로 실패해 build, unit/E2E, browser, Lighthouse, Docker command는 실행하지 않았습니다. 따라서 위 test 결과는 구현된 test technique과 assertion의 정적 검토이며 실제 통과 결과가 아닙니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 실제로 checkout하거나 diff로 확인했습니다.
- [x] Thread 내 commit 순서를 source와 동일하게 유지했습니다.
- [x] 각 A/S commit에서 직전 상태, decision, failure boundary, guarantee와 non-guarantee를 구분했습니다.
- [x] B commit은 thread 흐름에서 필요한 구현 역할과 state change를 확인했습니다.
- [x] Fix commit을 독립 feature가 아니라 기존 가정 → failure → root cause → corrected invariant로 설명했습니다.
- [x] Test commit에서 production invariant, failure injection, technique, traversed production path, proves/does-not-prove를 구분했습니다.
- [x] Invariant ledger의 모든 주장에 해당 SHA의 code/test evidence가 있습니다.
- [x] Final HEAD의 code를 과거 commit 설명에 소급하지 않았습니다.
- [x] Thread 최종 흐름을 code 없이 설명할 수 있습니다.
===== END FILE: 04-template-preview-to-production-publication.md =====

===== BEGIN FILE: 05-native-design-switcher-and-server-first-interaction.md =====
# Thread: Native design switcher and server-first interaction

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 source에서 확정된 thread 구조와 commit 역할만 미리 제공합니다. 실제 구현 해석, code evidence, failure 재현, test 결과, 최종 설명은 해당 SHA의 코드를 직접 확인해 작성합니다.

## 1. Thread 목표

Native `<details>` 기반 design switcher가 client wrapper로 시작해 pre-hydration race를 드러내고, fix와 deterministic test를 거쳐 server-rendered markup과 최소 client action으로 축소되는 과정을 복원합니다.

**Source-defined significance**

> The switcher begins as a client component built around native semantics, exposes a hydration race, and is then reduced to server markup plus the smallest required client action. The thread shows a concrete progression from behavior implementation to root-cause correction and boundary simplification.

### 이 Thread에 직접 연결되는 Critical Invariants

- Design order는 `SITE_DESIGNS`, 표시 문구는 presentation content가 소유합니다. — `e43e8addd7f3`
- Explicit close는 owning `<details>`를 닫고 summary로 focus를 복원합니다. — `c69ef85c98b2`
- Hydration 전 변경된 native `open` state는 유효한 사용자/browser state입니다. — `c702b870d57a → b6c0238ab8b8`
- Static switcher markup은 server에서 렌더링되고 imperative close만 client boundary가 소유합니다. — `a37cb8596733`
- Selector module은 다시 top-level client component나 `useRef` owner가 되지 않습니다. — `1ac7813155c6`

### 연결되는 Major Engineering Difficulty

- Native `<details>` behavior를 server rendering, pre-hydration interaction, explicit closure, keyboard-focus restoration과 함께 보존하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Native disclosure semantics를 쓰면서도 처음에는 어떤 client state/ref 책임이 component 전체를 hydration boundary로 만들었는가?
- Close 동작은 왜 `open=false`만이 아니라 summary focus restoration까지 포함해야 하는가?
- Hydration 전에 사용자가 `<details>`를 연 경우 React의 어떤 가정이 깨지는가?
- Hydration warning suppression fix와 server-first refactor는 각각 무엇을 해결하는가?
- 마지막 structural test는 client boundary가 다시 커지는 회귀를 어떻게 막는가?

## 3. 완료 기준

- 초기 client state/ref, native DOM state, hydration, explicit close 흐름을 시간 순서로 그렸습니다.
- `c702b870d57a`의 기존 가정 → race → root cause → 최소 fix를 element/property 수준에서 설명할 수 있습니다.
- `b6c0238ab8b8`의 exact SSR/pre-hydration/hydration 재현 순서를 기록했습니다.
- `a37cb8596733` 이후 server component와 `DesignSwitcherClose`의 책임을 확인했습니다.
- `1ac7813155c6`이 증명하는 것과 증명하지 않는 것을 구분했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `e43e8addd7f3` | feat(designs): 디자인 선택기 상태와 trigger 추가 | B | RENDERER | Builds the initial hydrated native disclosure wrapper. |
| 2 | `c69ef85c98b2` | feat(designs): 디자인 선택 목록과 닫기 동작 추가 | A | ARCH, RENDERER | Adds explicit closure and keyboard-focus restoration. |
| 3 | `c702b870d57a` | fix(ui): hydration 중 native details 상태 보존 | A | DEBUG | Recognizes pre-hydration native state as legitimate. |
| 4 | `b6c0238ab8b8` | test(ui): details hydration 경쟁 조건 검증 | A | VALIDATION, TEST | Reproduces the server/browser/React race. |
| 5 | `a37cb8596733` | refactor(ui): 디자인 선택기를 server markup으로 전환 | A | ARCH, REFACTOR | Moves static disclosure markup back to the server and isolates imperative closure. |
| 6 | `1ac7813155c6` | test(ui): server 선택기와 focus 복원 검증 | A | VALIDATION, A11Y, TEST | Prevents the server boundary from silently becoming a hydrated component again. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA를 checkout한 상태에서 작성합니다. Thread 내 이전 commit은 비교 대상으로 사용할 수 있지만 final HEAD를 정답처럼 소급하지 않습니다.

### 1. `e43e8addd7f3` — feat(designs): 디자인 선택기 상태와 trigger 추가

- **Importance:** B
- **Tags:** RENDERER
- **Source-defined thread role:** Builds the initial hydrated native disclosure wrapper.
- **Source classification summary:** Introduced the client-side state and trigger for a route-preserving design switcher.
- **Source classification reason:** Competent implementation within an established design; it advances the finished product but contains limited branch-wide architectural judgment.

#### Source에서 확정된 구현 의도와 상태 변화

Client-side design switcher의 native `<details>/<summary>` trigger와 state context를 만들고 `SITE_DESIGNS`를 order source로, presentation templates를 label/description source로 사용합니다.

#### 해당 SHA에서 확인할 실제 코드

- Component의 top-level client directive와 state/ref/hook 사용을 확인합니다.
- `<details>/<summary>` markup과 current design fallback 계산을 추적합니다.
- Registry order와 presentation label을 join하는 key를 확인합니다.
- Position/total summary label의 interpolation template을 찾습니다.
- Selection panel과 explicit close가 아직 없는 초기 boundary를 기록합니다.

확인 원칙:

- 먼저 `e43e8addd7f3^`와 `e43e8addd7f3`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Native semantics와 React client ownership의 조합 | React client component가 disclosure markup/ref ownership을 갖고 registry는 design identity/order, presentation content는 user-facing copy를 갖습니다. |
| Design identity와 user-facing copy의 서로 다른 source | `src/components/portfolio/design-switcher.tsx`가 `"use client"`, details/summary refs, `SITE_DESIGNS` order, presentation template label/description, current design fallback과 position/total summary text를 사용합니다. |
| Unknown active ID fallback | top-level client `DesignSwitcher`에 `<details>/<summary>`와 ref/state context를 만들었습니다. |
| 이 시점의 미완성 interaction 범위 | option list, route navigation, explicit close/focus와 hydration race 처리는 아직 없습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** design selection UI가 complete ordered native disclosure로 존재하지 않았습니다.
- **해당 SHA 핵심 코드:** `src/components/portfolio/design-switcher.tsx`가 `"use client"`, details/summary refs, `SITE_DESIGNS` order, presentation template label/description, current design fallback과 position/total summary text를 사용합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `c69ef85c98b2`가 full option sheet와 close behavior를 추가합니다.

#### SHA별 복원 결론

- **Thread 내 역할:** top-level client `DesignSwitcher`에 `<details>/<summary>`와 ref/state context를 만들었습니다.
- **실제 변경:** `src/components/portfolio/design-switcher.tsx`가 `"use client"`, details/summary refs, `SITE_DESIGNS` order, presentation template label/description, current design fallback과 position/total summary text를 사용합니다.
- **현재 보장:** native disclosure trigger와 current-state summary가 렌더링됩니다.
- **남은 범위:** option list, route navigation, explicit close/focus와 hydration race 처리는 아직 없습니다. `c69ef85c98b2`가 full option sheet와 close behavior를 추가합니다.

### 2. `c69ef85c98b2` — feat(designs): 디자인 선택 목록과 닫기 동작 추가

- **Importance:** A
- **Tags:** ARCH, RENDERER
- **Source-defined thread role:** Adds explicit closure and keyboard-focus restoration.
- **Source classification summary:** Completed the design-switcher sheet with an ordered list of registered designs, active-state semantics, palette previews, and explicit closing behavior.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Ordered design option list, active `aria-current`, palette preview, route-preserving links, close button을 완성합니다. Explicit close는 `<details>`의 open state를 제거하고 summary trigger로 focus를 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- Current path와 debug state를 보존한 design URL 생성 호출을 확인합니다.
- Option order, active item, `aria-current`, palette swatch를 확인합니다.
- Close handler가 owning `<details>`와 summary를 찾는 ref/DOM path를 추적합니다.
- Open state 변경 후 focus restoration 순서를 확인합니다.
- Design link navigation과 explicit close의 역할 차이를 확인합니다.
- Keyboard-only로 open, option navigation, close, focus return을 실행합니다.

확인 원칙:

- 먼저 `c69ef85c98b2^`와 `c69ef85c98b2`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Disclosure state와 route navigation state 구분 | native `<details>`가 disclosure state를, route link가 navigation state를, explicit close handler가 DOM mutation/focus를 소유합니다. |
| Active-state semantics | native `<details>`가 disclosure state를, route link가 navigation state를, explicit close handler가 DOM mutation/focus를 소유합니다. |
| Close와 focus restoration invariant | close 후 trigger focus가 복원되고 current design/route/debug state가 보존됩니다. |
| Mobile sheet에서도 동일 DOM owner를 쓰는지 | native `<details>`가 disclosure state를, route link가 navigation state를, explicit close handler가 DOM mutation/focus를 소유합니다. |
| 아직 hydration race를 고려하지 않은 가정 | SSR 후 hydration 전 browser가 `open`을 바꾸는 race는 고려하지 않았습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 초기 switcher는 trigger만 있고 design links와 explicit dismissal가 없었습니다.
- **해당 SHA 핵심 코드:** 같은 component에서 `SITE_DESIGNS` 순서로 option을 만들고 active item에 `aria-current`, palette swatches, current path/debug를 보존한 URL을 제공합니다. Close handler는 owning details의 `open`을 제거한 뒤 summary에 `focus()`합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `c702b870d57a`이 pre-hydration native state를 legitimate state로 인정합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** 초기 switcher는 trigger만 있고 design links와 explicit dismissal가 없었습니다. sheet를 닫은 뒤 keyboard focus가 body로 사라지거나 route state가 유실될 수 있었습니다.
- **구현 결정과 경로:** ordered options, active semantics, state-preserving links와 close→summary focus invariant를 추가했습니다. 같은 component에서 `SITE_DESIGNS` 순서로 option을 만들고 active item에 `aria-current`, palette swatches, current path/debug를 보존한 URL을 제공합니다. Close handler는 owning details의 `open`을 제거한 뒤 summary에 `focus()`합니다.
- **소유권·실패 처리:** native `<details>`가 disclosure state를, route link가 navigation state를, explicit close handler가 DOM mutation/focus를 소유합니다. details/summary refs가 동일 mobile sheet DOM owner를 가리키며 link navigation과 explicit close는 별도 동작입니다.
- **보장:** close 후 trigger focus가 복원되고 current design/route/debug state가 보존됩니다.
- **보장하지 않는 범위:** SSR 후 hydration 전 browser가 `open`을 바꾸는 race는 고려하지 않았습니다.
- **후속 연결:** `c702b870d57a`이 pre-hydration native state를 legitimate state로 인정합니다.

### 3. `c702b870d57a` — fix(ui): hydration 중 native details 상태 보존

- **Importance:** A
- **Tags:** DEBUG
- **Source-defined thread role:** Recognizes pre-hydration native state as legitimate.
- **Source classification summary:** Mark the native design-switcher `details` element as an intentional hydration boundary.
- **Source classification reason:** Significant because it corrects a non-obvious build, integration, or runtime failure at the layer that owns the violated invariant.

#### Source에서 확정된 구현 의도와 상태 변화

SSR markup 도착 후 hydration 전에 browser/user가 native `<details>`의 `open`을 바꿀 수 있음을 인정하고 owning element에서 transient mismatch warning을 의도적으로 억제해 current open state를 보존합니다.

#### 해당 SHA에서 확인할 실제 코드

- Fix 전 React가 server/client markup 동일성을 가정하던 owning element를 확인합니다.
- `suppressHydrationWarning`이 정확히 `<details>`에 적용되는지 확인합니다.
- Hydration 시 open state를 강제로 reset하는 code가 없는지 확인합니다.
- Post-hydration close/focus logic이 유지되는지 비교합니다.
- Suppression 범위가 element-level인지 기록합니다.

확인 원칙:

- 먼저 `c702b870d57a^`와 `c702b870d57a`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 기존 가정: hydration 전 DOM state는 변하지 않음 | React는 server HTML의 closed `<details>`와 hydration 시 DOM이 동일하다고 가정했습니다. |
| 실제 risk: native interaction이 open attribute를 변경 | root cause는 uncontrolled browser-owned `open`과 React comparison의 충돌이며 suppression 범위는 details element 하나입니다. |
| Root cause: browser-owned state와 React comparison 충돌 | 사용자가 hydration 전에 native summary를 열면 browser가 `open` attribute를 추가해 mismatch diagnostic이 발생할 수 있었습니다. |
| 수정된 invariant: current native open state 보존 | hydration 전 변경된 current native open state를 강제로 덮어쓰지 않습니다. |
| 실제 수정 code와 영향받지 않는 close/focus path | `src/components/portfolio/design-switcher.tsx`의 `<details suppressHydrationWarning>` 한 지점이 수정됐고 기존 close handler와 focus path는 그대로입니다. |
| Server-first boundary까지는 달성하지 않은 범위 | whole selector는 여전히 client component이고 모든 browser timing/focus behavior를 검증한 것은 아닙니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** React는 server HTML의 closed `<details>`와 hydration 시 DOM이 동일하다고 가정했습니다.
- **해당 SHA 핵심 코드:** `src/components/portfolio/design-switcher.tsx`의 `<details suppressHydrationWarning>` 한 지점이 수정됐고 기존 close handler와 focus path는 그대로입니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `b6c0238ab8b8`이 SSR→mutation→hydrate 순서를 deterministic test로 재현합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** React는 server HTML의 closed `<details>`와 hydration 시 DOM이 동일하다고 가정했습니다. 사용자가 hydration 전에 native summary를 열면 browser가 `open` attribute를 추가해 mismatch diagnostic이 발생할 수 있었습니다.
- **구현 결정과 경로:** owning `<details>`에 element-level `suppressHydrationWarning`을 추가하고 open state를 reset하지 않았습니다. `src/components/portfolio/design-switcher.tsx`의 `<details suppressHydrationWarning>` 한 지점이 수정됐고 기존 close handler와 focus path는 그대로입니다.
- **소유권·실패 처리:** browser/user가 transient native open state를 소유하고 React는 이 element의 hydration attribute mismatch를 경고 대상으로 취급하지 않습니다. root cause는 uncontrolled browser-owned `open`과 React comparison의 충돌이며 suppression 범위는 details element 하나입니다.
- **보장:** hydration 전 변경된 current native open state를 강제로 덮어쓰지 않습니다.
- **보장하지 않는 범위:** whole selector는 여전히 client component이고 모든 browser timing/focus behavior를 검증한 것은 아닙니다.
- **후속 연결:** `b6c0238ab8b8`이 SSR→mutation→hydrate 순서를 deterministic test로 재현합니다.

### 4. `b6c0238ab8b8` — test(ui): details hydration 경쟁 조건 검증

- **Importance:** A
- **Tags:** VALIDATION, TEST
- **Source-defined thread role:** Reproduces the server/browser/React race.
- **Source classification summary:** Reproduce the design-switcher hydration race directly and lock down the intended invariant.
- **Source classification reason:** Significant because it locks down a cross-cutting contract or production-relevant regression rather than adding routine component coverage.

#### Source에서 확정된 구현 의도와 상태 변화

Switcher를 server-render하고 hydration 전에 native `<details>`를 연 뒤 같은 tree를 hydrate하는 race를 직접 재현합니다. Open attribute 유지와 hydration diagnostic 부재를 검증하고 cleanup을 명시합니다.

#### 해당 SHA에서 확인할 실제 코드

- Server render API와 생성 HTML을 DOM에 설치하는 순서를 확인합니다.
- Hydration 전에 `<details>.open` 또는 attribute를 변경하는 line을 찾습니다.
- Console/error diagnostic spy가 어떤 channel을 capture하는지 확인합니다.
- Hydration 후 open state와 diagnostic assertions를 확인합니다.
- Unmount, spy restoration, DOM cleanup을 확인합니다.

확인 원칙:

- 먼저 `b6c0238ab8b8^`와 `b6c0238ab8b8`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | 해당 fixture에서 open state 보존과 hydration diagnostic 부재를 증명합니다. |
| 주입한 race timing과 native state mutation | test fixture가 race timing과 DOM ownership을 통제하고 production component가 hydration path를 제공합니다. |
| SSR + pre-hydration DOM mutation + hydrate technique | test는 `renderToString`→DOM install→native open mutation→`hydrateRoot` 순서로 실제 component path를 실행하고 `console.error`의 hydration diagnostic을 spy합니다. Hydration 후 open 유지/no captured errors를 assert하고 root unmount, spy restore, DOM remove를 cleanup합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| 실제 production component path | test는 `renderToString`→DOM install→native open mutation→`hydrateRoot` 순서로 실제 component path를 실행하고 `console.error`의 hydration diagnostic을 spy합니다. Hydration 후 open 유지/no captured errors를 assert하고 root unmount, spy restore, DOM remove를 cleanup합니다. |
| 증명하는 것: open 보존과 hydration error 부재 | 해당 fixture에서 open state 보존과 hydration diagnostic 부재를 증명합니다. |
| 증명하지 않는 것: 모든 browser timing, focus restoration, server-only boundary | 모든 browser scheduler/timing, explicit close focus, server-only boundary와 visual behavior는 증명하지 않습니다. |
| 후속 refactor에서 유지할 contract | `a37cb8596733`이 behavior를 보존하면서 whole selector hydration boundary를 제거합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** hydration fix가 code comment/convention에만 의존했고 race 재현이 없었습니다.
- **해당 SHA 핵심 코드:** test는 `renderToString`→DOM install→native open mutation→`hydrateRoot` 순서로 실제 component path를 실행하고 `console.error`의 hydration diagnostic을 spy합니다. Hydration 후 open 유지/no captured errors를 assert하고 root unmount, spy restore, DOM remove를 cleanup합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `a37cb8596733`이 behavior를 보존하면서 whole selector hydration boundary를 제거합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** hydration fix가 code comment/convention에만 의존했고 race 재현이 없었습니다. 향후 warning suppression 제거나 controlled state 도입이 same mismatch를 되살릴 수 있었습니다.
- **구현 결정과 경로:** production switcher를 server-render하고 hydration 전에 `details.open = true`로 바꾼 뒤 hydrate하는 deterministic regression을 추가했습니다. test는 `renderToString`→DOM install→native open mutation→`hydrateRoot` 순서로 실제 component path를 실행하고 `console.error`의 hydration diagnostic을 spy합니다. Hydration 후 open 유지/no captured errors를 assert하고 root unmount, spy restore, DOM remove를 cleanup합니다.
- **소유권·실패 처리:** test fixture가 race timing과 DOM ownership을 통제하고 production component가 hydration path를 제공합니다. 실패 주입은 allocation이 아니라 hydration 직전 native property mutation입니다.
- **보장:** 해당 fixture에서 open state 보존과 hydration diagnostic 부재를 증명합니다.
- **보장하지 않는 범위:** 모든 browser scheduler/timing, explicit close focus, server-only boundary와 visual behavior는 증명하지 않습니다.
- **후속 연결:** `a37cb8596733`이 behavior를 보존하면서 whole selector hydration boundary를 제거합니다.

### 5. `a37cb8596733` — refactor(ui): 디자인 선택기를 server markup으로 전환

- **Importance:** A
- **Tags:** ARCH, REFACTOR
- **Source-defined thread role:** Moves static disclosure markup back to the server and isolates imperative closure.
- **Source classification summary:** Convert the design selector itself from a client component to server-rendered markup and isolate the only imperative behavior in a small `DesignSwitcherClose` client component.
- **Source classification reason:** Significant because it materially narrows ownership or coupling at a boundary used by multiple routes and renderers while preserving established behavior.

#### Source에서 확정된 구현 의도와 상태 변화

Design selector의 static `<details>`, option list, links를 server-rendered component로 바꾸고 DOM mutation과 focus가 필요한 `DesignSwitcherClose`만 client component로 격리합니다.

#### 해당 SHA에서 확인할 실제 코드

- Selector module에서 top-level client directive와 ref/state가 제거되는 diff를 확인합니다.
- Server component props가 serializable한지 확인합니다.
- `DesignSwitcherClose`의 client directive, event target traversal, details lookup을 추적합니다.
- Handler가 open attribute 제거 후 summary focus를 복원하는 순서를 확인합니다.
- Design link의 `onClick`이 제거되고 navigation이 page replacement를 담당하는지 확인합니다.
- Hydration되는 JavaScript boundary가 어디까지 축소됐는지 import graph로 기록합니다.

확인 원칙:

- 먼저 `a37cb8596733^`와 `a37cb8596733`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 client component의 static/imperative 책임 분해 | 정적 details/summary/list/links 전체가 top-level client component와 refs 때문에 hydration 대상이었습니다. |
| Server markup의 immediate native behavior | static selector를 server component로 되돌리고 DOM mutation/focus만 작은 `DesignSwitcherClose` client island로 분리했습니다. `src/components/portfolio/design-switcher.tsx`에서 `"use client"`, refs와 link `onClick`이 제거되고 serializable props로 markup을 출력합니다. 새 `design-switcher-close.tsx`는 click target에서 closest details를 찾고 `open` 제거 후 direct child summary에 focus합니다. |
| Client-only action의 최소 input과 DOM ownership | server component가 list/link/disclosure markup을, browser가 native open/navigation을, close island만 imperative DOM/focus를 소유합니다. |
| Hydration race fix와 함께 유지되는 policy | hydration 전 native behavior를 즉시 사용할 수 있고 only explicit close에 client JavaScript가 필요합니다. |
| Explicit close에는 JS가 필요하지만 list/navigation에는 필요하지 않은 trade-off | close action에는 여전히 JS가 필요하고 source boundary가 향후 되돌아가지 않는다는 regression proof는 아직 없습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 정적 details/summary/list/links 전체가 top-level client component와 refs 때문에 hydration 대상이었습니다.
- **해당 SHA 핵심 코드:** `src/components/portfolio/design-switcher.tsx`에서 `"use client"`, refs와 link `onClick`이 제거되고 serializable props로 markup을 출력합니다. 새 `design-switcher-close.tsx`는 click target에서 closest details를 찾고 `open` 제거 후 direct child summary에 focus합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `1ac7813155c6`이 source structure와 focus behavior를 회귀 계약으로 고정합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** 정적 details/summary/list/links 전체가 top-level client component와 refs 때문에 hydration 대상이었습니다. native semantics로 가능한 markup까지 JavaScript가 소유해 bundle/ownership이 넓고 race surface가 컸습니다.
- **구현 결정과 경로:** static selector를 server component로 되돌리고 DOM mutation/focus만 작은 `DesignSwitcherClose` client island로 분리했습니다. `src/components/portfolio/design-switcher.tsx`에서 `"use client"`, refs와 link `onClick`이 제거되고 serializable props로 markup을 출력합니다. 새 `design-switcher-close.tsx`는 click target에서 closest details를 찾고 `open` 제거 후 direct child summary에 focus합니다.
- **소유권·실패 처리:** server component가 list/link/disclosure markup을, browser가 native open/navigation을, close island만 imperative DOM/focus를 소유합니다. details가 없거나 summary를 찾지 못하면 handler는 해당 DOM path에서 더 진행하지 않으며 navigation은 normal page replacement가 처리합니다.
- **보장:** hydration 전 native behavior를 즉시 사용할 수 있고 only explicit close에 client JavaScript가 필요합니다.
- **보장하지 않는 범위:** close action에는 여전히 JS가 필요하고 source boundary가 향후 되돌아가지 않는다는 regression proof는 아직 없습니다.
- **후속 연결:** `1ac7813155c6`이 source structure와 focus behavior를 회귀 계약으로 고정합니다.

### 6. `1ac7813155c6` — test(ui): server 선택기와 focus 복원 검증

- **Importance:** A
- **Tags:** VALIDATION, A11Y, TEST
- **Source-defined thread role:** Prevents the server boundary from silently becoming a hydrated component again.
- **Source classification summary:** Add a structural regression check that reads the design-switcher source and rejects either a top-level `"use client"` directive or `useRef` state in the selector component.
- **Source classification reason:** Significant because it restores or verifies an accessibility invariant across designs or routes, where a local presentation defect would otherwise become a site-wide failure.

#### Source에서 확정된 구현 의도와 상태 변화

Selector source를 검사해 top-level `"use client"`와 `useRef`가 다시 들어오는 것을 거부하고 existing explicit-close focus tests와 함께 server/client split을 회귀 계약으로 만듭니다.

#### 해당 SHA에서 확인할 실제 코드

- Test가 selector source file을 읽는 방식과 exact token/pattern을 확인합니다.
- Top-level directive와 incidental string을 어떻게 구분하는지 확인합니다.
- `useRef` 금지가 어떤 ownership 회귀를 뜻하는지 production source와 대조합니다.
- Explicit close/focus behavior test가 어떤 production path를 실행하는지 확인합니다.
- Lockfile movement이 behavior source가 아닌 transitive refresh인지 구분합니다.

확인 원칙:

- 먼저 `1ac7813155c6^`와 `1ac7813155c6`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 대상 invariant: static selector는 server component | static selector의 server ownership 재도입 방지와 explicit close focus behavior를 보호합니다. |
| Source-structure regression + interaction regression technique | test가 `design-switcher.tsx` source에서 top-level `"use client"` directive와 `useRef` token을 거부하고, 허용된 `DesignSwitcherClose`의 click path가 `open` 제거와 summary focus를 수행하는지 검사합니다. Lockfile 변경은 transitive refresh로 behavior decision과 구분됩니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| 증명하는 것: whole selector hydration 재도입 방지와 focus behavior | static selector의 server ownership 재도입 방지와 explicit close focus behavior를 보호합니다. |
| 증명하지 않는 것: bundle byte, 모든 hook, 모든 browser race | bundle byte, 모든 React hook, 모든 browser hydration race와 실제 assistive-technology behavior는 증명하지 않습니다. |
| 허용된 `DesignSwitcherClose` boundary와 금지 owner 구분 | selector source structure는 structural test가, imperative behavior는 small client component test가 소유합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** server/client split은 구현됐지만 selector에 top-level client directive/ref가 다시 들어오는 것을 자동 차단하지 않았습니다.
- **해당 SHA 핵심 코드:** test가 `design-switcher.tsx` source에서 top-level `"use client"` directive와 `useRef` token을 거부하고, 허용된 `DesignSwitcherClose`의 click path가 `open` 제거와 summary focus를 수행하는지 검사합니다. Lockfile 변경은 transitive refresh로 behavior decision과 구분됩니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** 이 commit이 thread의 server-first boundary와 focus invariant를 최종 regression contract로 만듭니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** server/client split은 구현됐지만 selector에 top-level client directive/ref가 다시 들어오는 것을 자동 차단하지 않았습니다. 기능 test만 통과하면서 whole selector hydration owner가 조용히 재도입될 수 있었습니다.
- **구현 결정과 경로:** selector source를 직접 읽는 structural regression과 existing close/focus interaction test를 결합했습니다. test가 `design-switcher.tsx` source에서 top-level `"use client"` directive와 `useRef` token을 거부하고, 허용된 `DesignSwitcherClose`의 click path가 `open` 제거와 summary focus를 수행하는지 검사합니다. Lockfile 변경은 transitive refresh로 behavior decision과 구분됩니다.
- **소유권·실패 처리:** selector source structure는 structural test가, imperative behavior는 small client component test가 소유합니다. 금지 대상은 whole selector owner이고 `DesignSwitcherClose` client boundary 자체는 허용됩니다.
- **보장:** static selector의 server ownership 재도입 방지와 explicit close focus behavior를 보호합니다.
- **보장하지 않는 범위:** bundle byte, 모든 React hook, 모든 browser hydration race와 실제 assistive-technology behavior는 증명하지 않습니다.
- **후속 연결:** 이 commit이 thread의 server-first boundary와 focus invariant를 최종 regression contract로 만듭니다.

## 6. Invariant ledger

| Invariant | Source에서 확인된 변화 지점 | 해당 SHA의 실제 code/test evidence | 부족함이 드러난 시점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Design order는 `SITE_DESIGNS`, 표시 문구는 presentation content가 소유합니다. | `e43e8addd7f3` | `e43e8addd7f3`의 `SITE_DESIGNS` iteration과 presentation template lookup/summary interpolation. | 초기에는 trigger만 있고 complete options/close는 없었습니다. | design ID/order/palette는 registry, label/description/summary copy는 presentation content에서 결합됩니다. |
| Explicit close는 owning `<details>`를 닫고 summary로 focus를 복원합니다. | `c69ef85c98b2` | `c69ef85c98b2` close handler: owning details의 `open` 제거 후 summary `focus()`. | 처음에는 explicit dismissal와 keyboard focus return이 없었습니다. | close action은 disclosure를 닫고 trigger focus를 복원하는 하나의 invariant로 구현됩니다. |
| Hydration 전 변경된 native `open` state는 유효한 사용자/browser state입니다. | `c702b870d57a → b6c0238ab8b8` | `c702b870d57a`의 `<details suppressHydrationWarning>`와 `b6c0238ab8b8`의 SSR→pre-open→hydrate assertions. | React/server markup 동일성 가정이 hydration 전 native mutation을 고려하지 못했습니다. | current browser-owned open state를 reset하지 않고 해당 element mismatch를 허용하며 deterministic regression이 이를 보호합니다. |
| Static switcher markup은 server에서 렌더링되고 imperative close만 client boundary가 소유합니다. | `a37cb8596733` | `a37cb8596733`에서 selector의 client directive/refs/handlers 제거, `DesignSwitcherClose`만 client island. | static list/link까지 whole client component가 소유했습니다. | native disclosure/list/navigation은 server markup+browser가, explicit close/focus만 small client action이 소유합니다. |
| Selector module은 다시 top-level client component나 `useRef` owner가 되지 않습니다. | `1ac7813155c6` | `1ac7813155c6` source test가 selector의 top-level `use client`와 `useRef`를 거부하고 close/focus interaction을 유지합니다. | server-first split이 convention만으로 되돌아갈 수 있었습니다. | whole selector hydration owner 재도입을 structural regression으로 차단합니다. |

Ledger 작성 원칙:

- Source가 명시한 invariant만 사용하고 새 invariant를 확정 사실처럼 추가하지 않습니다.
- 도입, 강화, 부족함 노출, fix, regression test가 서로 다른 commit이면 각 열에 분리해 기록합니다.
- Code evidence에는 실제 field, function, branch, selector, command 또는 assertion을 적습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 위험 | 대응 commit | 실제 수정/강화 code에서 확인할 것 | Test 또는 실행 증거 |
| --- | --- | --- | --- |
| Initial behavior | e43e8addd7f3 → c69ef85c98b2 | Native disclosure, active item, state-preserving links, close/focus를 구현합니다. | 일반 interaction은 동작하지만 pre-hydration mutation은 아직 고려되지 않습니다. |
| 실제 failure/risk | c702b870d57a | SSR markup과 hydration 사이에 `open`이 바뀌어 mismatch diagnostic이 발생합니다. | Owning `<details>`에서 transient state를 의도된 boundary로 표시합니다. |
| Deterministic regression | b6c0238ab8b8 | Server render → hydration 전 open → hydrate → diagnostic capture를 수행합니다. | Open state 유지와 hydration error 부재를 검증합니다. |
| Boundary simplification | a37cb8596733 → 1ac7813155c6 | Static panel/list/link는 server, close/focus만 client action으로 이동합니다. | Source-structure test로 whole-selector hydration 회귀를 차단합니다. |

### 실제 연결 기록

- Fix는 `open`을 React state로 덮어쓰는 것이 아니라 owning `<details>`에서 transient mismatch warning을 억제하고 browser state를 그대로 둡니다.
- Deterministic test의 failure injection은 hydration 전에 `details.open = true`를 설정하는 것입니다. Console diagnostic spy, unmount, spy restore와 DOM cleanup을 확인했습니다.
- Server-first refactor 뒤에는 link/list가 client behavior를 요구하지 않고 explicit close/focus만 작은 island에 남습니다.

## 8. Ownership / state / responsibility 변화

| Concern | Thread 초기 owner/state | Thread 최종 owner/state | 실제 symbol과 호출 경로 |
| --- | --- | --- | --- |
| Disclosure markup/list/link | Client component | Server component | `src/components/portfolio/design-switcher.tsx`: server-rendered details/summary/options/links. |
| Native open state | React hydration의 동일 markup 가정 | Browser/user state를 보존하는 owning `<details>` | native `<details>` element/browser: user가 hydration 전후 변경하는 `open` state; owning element에 warning suppression. |
| Explicit close | Component-wide ref/state | 작은 `DesignSwitcherClose` handler | `design-switcher-close.tsx::DesignSwitcherClose` click handler: closest details lookup과 `open` removal. |
| Focus restoration | Close와 분리될 위험 | Close handler가 summary를 찾아 복원 | same close handler: direct summary lookup 후 focus restoration. |
| Regression proof | 일반 interaction test | Hydration race test + source-boundary test | hydration race test + selector source-structure test + close/focus interaction test. |

## 9. Thread 최종 상태

### Source에서 확정된 최종 상태

Native `<details>` 기반 design switcher가 client wrapper로 시작해 pre-hydration race를 드러내고, fix와 deterministic test를 거쳐 server-rendered markup과 최소 client action으로 축소되는 과정을 복원합니다.

### 학습자가 완성할 최종 설명

- **Thread 시작 시점의 설계와 위험:** 초기 switcher는 native details를 사용했지만 top-level client component와 refs가 static markup 전체를 hydration boundary로 만들었습니다.
- **핵심 architecture/decision이 형성된 순서:** trigger/context → option list/navigation/close-focus → hydration mismatch fix → deterministic race test → server markup/client island split → structural regression 순서로 발전했습니다.
- **실제 failure 또는 부족함이 드러난 지점:** SSR closed markup 뒤 사용자가 hydration 전에 details를 열면 browser `open`과 React comparison이 충돌하는 non-obvious race가 드러났습니다.
- **Fix 또는 boundary 강화가 바꾼 invariant:** element-level suppression은 current native state를 보존했고 server-first refactor는 race surface와 hydrated responsibility 자체를 줄였습니다.
- **Test/build/browser evidence가 보장한 범위:** test code의 exact timing, assertions와 cleanup을 정적으로 검토했습니다. JSDOM/browser test를 실제 실행하지 않았습니다.
- **Thread 종료 시점에도 보장하지 않는 범위:** 모든 browser timing, bundle byte, explicit close의 JS-free 동작과 assistive technology 전반은 보장하지 않습니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 source-backed flow의 각 단계에 실제 file path, symbol, input/output, failure branch를 추가합니다.

1. Server가 native `<details>/<summary>`와 design link 목록을 출력합니다.
   - 실제 코드 위치: `src/components/portfolio/design-switcher.tsx` server component
   - 입력과 출력: serializable props, `SITE_DESIGNS`, presentation copy와 current path/debug를 받아 native details/summary/options/links HTML을 출력합니다.
   - 실패/absence 처리: unknown design은 registry fallback을 사용하고 content copy가 없으면 해당 data contract failure가 upstream에 드러납니다.
2. Browser/user가 hydration 전에도 native disclosure state를 바꿀 수 있습니다.
   - 실제 코드 위치: browser native `<details>` behavior
   - 입력과 출력: summary activation이 JavaScript/hydration 전에도 element `open` state를 토글합니다.
   - 실패/absence 처리: native behavior 자체가 없으면 platform support 범위이며 React controlled reset code는 없습니다.
3. Hydration은 transient `open` 차이를 owning boundary에서 유효하게 취급합니다.
   - 실제 코드 위치: same `<details suppressHydrationWarning>`
   - 입력과 출력: hydration이 server closed attribute와 current browser open attribute 차이를 element-level transient state로 허용합니다.
   - 실패/absence 처리: suppression은 다른 descendants/global mismatch를 숨기는 범위가 아닙니다.
4. Design link는 state-preserving URL로 navigation합니다.
   - 실제 코드 위치: design URL builder와 server-rendered links
   - 입력과 출력: current path, selected design와 debug query를 결합한 href로 normal navigation합니다.
   - 실패/absence 처리: link click handler로 disclosure를 수동 닫지 않으며 navigation failure는 route layer로 전파됩니다.
5. Explicit close button만 client handler를 호출해 `open`을 제거합니다.
   - 실제 코드 위치: `DesignSwitcherClose` client click handler
   - 입력과 출력: event target에서 closest details를 찾아 `open` attribute를 제거합니다.
   - 실패/absence 처리: owning details를 찾지 못하면 mutation을 수행하지 않습니다.
6. Handler가 summary에 focus를 복원합니다.
   - 실제 코드 위치: same handler의 summary lookup/focus
   - 입력과 출력: details direct child summary를 찾아 `.focus()`로 keyboard position을 복구합니다.
   - 실패/absence 처리: summary가 없으면 focus restoration을 수행할 target이 없습니다.
7. Regression tests가 hydration race와 server/client boundary를 각각 고정합니다.
   - 실제 코드 위치: hydration test와 source/interaction tests
   - 입력과 출력: SSR→mutation→hydrate 및 source token/close-focus assertions로 두 regression class를 분리합니다.
   - 실패/absence 처리: fixture 밖 browser scheduling, bundle size와 visual layout은 test 범위가 아닙니다.

### 코드 없이 설명하기

> 이 Thread의 최종 실행 흐름을 code snippet 없이 자신의 말로 작성합니다. 설계 → 구현 → failure/risk → 수정/강화 → 검증 순서가 드러나야 합니다.

native details 기반 switcher는 처음에 전체 client component였고 option links와 explicit close/focus를 구현했습니다. 그러나 server HTML이 도착한 뒤 hydration 전에 사용자가 details를 열 수 있어 React가 예상한 closed markup과 browser state가 충돌했습니다. Owning element에서 그 transient mismatch를 허용하고 exact race를 test로 고정했습니다. 이후 static details, options와 links는 server component로 옮기고 open 제거와 summary focus만 작은 client button에 남겼습니다. 마지막 structural test는 selector가 다시 client/ref owner가 되는 회귀를 막습니다.

> **실행 증거 구분:** 참조 SHA의 diff와 해당 SHA 파일은 GitHub connector로 확인했습니다. 로컬 clone은 실행 환경의 DNS 차단으로 실패해 build, unit/E2E, browser, Lighthouse, Docker command는 실행하지 않았습니다. 따라서 위 test 결과는 구현된 test technique과 assertion의 정적 검토이며 실제 통과 결과가 아닙니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 실제로 checkout하거나 diff로 확인했습니다.
- [x] Thread 내 commit 순서를 source와 동일하게 유지했습니다.
- [x] 각 A/S commit에서 직전 상태, decision, failure boundary, guarantee와 non-guarantee를 구분했습니다.
- [x] B commit은 thread 흐름에서 필요한 구현 역할과 state change를 확인했습니다.
- [x] Fix commit을 독립 feature가 아니라 기존 가정 → failure → root cause → corrected invariant로 설명했습니다.
- [x] Test commit에서 production invariant, failure injection, technique, traversed production path, proves/does-not-prove를 구분했습니다.
- [x] Invariant ledger의 모든 주장에 해당 SHA의 code/test evidence가 있습니다.
- [x] Final HEAD의 code를 과거 commit 설명에 소급하지 않았습니다.
- [x] Thread 최종 흐름을 code 없이 설명할 수 있습니다.
===== END FILE: 05-native-design-switcher-and-server-first-interaction.md =====

===== BEGIN FILE: 06-production-artifact-and-performance-enforcement.md =====
# Thread: Production artifact and performance enforcement

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 source에서 확정된 thread 구조와 commit 역할만 미리 제공합니다. 실제 구현 해석, code evidence, failure 재현, test 결과, 최종 설명은 해당 SHA의 코드를 직접 확인해 작성합니다.

## 1. Thread 목표

Source-level green 상태를 넘어 실제 Next.js production output, route별 client asset, Lighthouse 결과, standalone package, non-root container와 public asset serving을 fail-closed release contract로 만드는 과정을 복원합니다.

**Source-defined significance**

> The release story moves from source validation to verification of the actual emitted artifact. Standalone checks, route-size accounting, Lighthouse thresholds, CI enforcement, and container HTTP tests close different gaps; none can be substituted by a development-server smoke test.

### 이 Thread에 직접 연결되는 Critical Invariants

- CI는 pinned toolchain과 reproducible install로 production path를 검사합니다. — `9fd3541c11dc`
- Standalone server entry와 static directory는 explicit artifact입니다. — `29508f4668ea → c0f7434467a0`
- Route budget은 emitted JS와 non-inlined CSS의 actual byte를 기준으로 합니다. — `605b64512edf`
- Baseline route coverage와 5% growth limit은 fail-closed입니다. — `518ff5b51ec5`
- Lighthouse는 production server와 five-design representative routes를 반복 측정합니다. — `1529ccf225c1`
- Release CI는 standalone, route budget, Lighthouse gate를 실제 build에 적용합니다. — `abbd530368a0`
- Final image는 public assets를 포함하고 non-root로 실행되며 real HTTP로 검증됩니다. — `b87a2b453741 → b94fa6dd0118`

### 연결되는 Major Engineering Difficulty

- Compiler-specific manifest, standalone packaging, route bundle cost, Lighthouse threshold, Docker asset serving을 actual production output에서 검증하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 CI gate와 later artifact/performance gate는 각각 어떤 검증 공백을 닫는가?
- Standalone output의 필수 file을 source config가 아니라 build artifact에서 어떻게 확인하는가?
- Route별 JS/CSS 측정은 어떤 manifest를 읽고 shared chunk, duplicate asset, inlined CSS를 어떻게 처리하는가?
- Committed baseline과 5% policy가 normal check에서 자동 갱신되지 않는 구조는 무엇인가?
- Lighthouse와 Docker HTTP test가 development-server smoke test로 대체될 수 없는 이유는 무엇인가?

## 3. 완료 기준

- Pinned install부터 CI build, standalone check, bundle check, Lighthouse, Docker test까지 command/artifact 흐름을 기록했습니다.
- Route asset collector의 manifest, route key, deduplication, byte-count 규칙을 실제 code로 추적했습니다.
- Budget evaluator의 missing/new route, JS/CSS overage branch를 확인했습니다.
- Lighthouse route/design coverage, run count, median, threshold, production server 시작 방식을 기록했습니다.
- Container copy boundary, runtime user, readiness, HTTP/MIME, cleanup path를 확인했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `9fd3541c11dc` | ci: 기본 배포 품질 검사 추가 | A | DEPLOY, TEST | Establishes reproducible production checks in automation. |
| 2 | `29508f4668ea` | build: standalone server 산출물 생성 | B | DEPLOY | Defines the traced server artifact boundary. |
| 3 | `c0f7434467a0` | test(build): standalone 산출물 완전성 검증 | A | VALIDATION, DEPLOY, TEST | Makes required server and static outputs explicit. |
| 4 | `605b64512edf` | build(perf): route별 client asset 측정 추가 | A | ARCH, ROUTING, PERF | Measures emitted JS and non-inlined CSS per route. |
| 5 | `518ff5b51ec5` | build(perf): route bundle 성장 예산 평가 추가 | A | ARCH, ROUTING, PERF | Turns measurements into fail-closed growth limits. |
| 6 | `1529ccf225c1` | build(perf): desktop Lighthouse 실행 경계 추가 | A | PERF, DEPLOY | Defines repeatable route/design laboratory thresholds. |
| 7 | `abbd530368a0` | ci: 검증된 bundle과 Lighthouse gate 활성화 | A | VALIDATION, PERF, DEPLOY | Promotes artifact, size, and audit limits into the release path. |
| 8 | `b87a2b453741` | build(docker): public 자산을 포함한 비루트 standalone image 추가 | A | DEPLOY | Packages the verified artifact with explicit public assets and non-root runtime. |
| 9 | `b94fa6dd0118` | test(docker): runtime route와 public 자산 검증 자동화 | A | ARCH, VALIDATION, ROUTING | Exercises the final container and derives asset coverage from authoritative content. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA를 checkout한 상태에서 작성합니다. Thread 내 이전 commit은 비교 대상으로 사용할 수 있지만 final HEAD를 정답처럼 소급하지 않습니다.

### 1. `9fd3541c11dc` — ci: 기본 배포 품질 검사 추가

- **Importance:** A
- **Tags:** DEPLOY, TEST
- **Source-defined thread role:** Establishes reproducible production checks in automation.
- **Source classification summary:** Establish a deployment-quality CI gate using the repository's pinned toolchain.
- **Source classification reason:** Significant because it verifies the production artifact or release path itself, closing a gap that source-level and development-server tests cannot cover.

#### Source에서 확정된 구현 의도와 상태 변화

Pinned toolchain을 사용하는 deployment-quality CI를 만들고 `npm ci`, lint, type check, content validation, production build, Chromium E2E를 required path로 구성합니다.

#### 해당 SHA에서 확인할 실제 코드

- Workflow trigger, concurrency/cancellation, read-only permission, timeout을 확인합니다.
- Node/npm setup이 repository pin과 일치하는지 확인합니다.
- `npm ci` 이후 lint/type/content/build/E2E command 순서와 failure propagation을 추적합니다.
- Playwright browser setup이 어느 단계에서 수행되는지 확인합니다.
- 아직 standalone/budget/Lighthouse/Docker가 없는 초기 gate 범위를 기록합니다.

확인 원칙:

- 먼저 `9fd3541c11dc^`와 `9fd3541c11dc`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Reproducible CI environment | pinned Node/npm, reproducible install과 production checks를 GitHub Actions required path로 만들었습니다. `.github/workflows/ci.yml`은 push/PR trigger, contents read-only permission, concurrency cancel, 30-minute timeout을 두고 `.nvmrc`, npm 11.16.0, `npm ci`, lint, typecheck, content check, Playwright Chromium install, production E2E 순으로 실행합니다. |
| Required checks와 실행 순서 | `.github/workflows/ci.yml`은 push/PR trigger, contents read-only permission, concurrency cancel, 30-minute timeout을 두고 `.nvmrc`, npm 11.16.0, `npm ci`, lint, typecheck, content check, Playwright Chromium install, production E2E 순으로 실행합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| Workflow 권한/자원 경계 | workflow가 environment/order/failure propagation을 소유하고 repository manifest/lockfile이 dependency resolution을 소유합니다. |
| 초기 gate가 검증하지 않는 final artifact 속성 | 배포 품질 검사가 개발자 로컬 실행과 환경 차이에 의존했습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 배포 품질 검사가 개발자 로컬 실행과 환경 차이에 의존했습니다.
- **해당 SHA 핵심 코드:** `.github/workflows/ci.yml`은 push/PR trigger, contents read-only permission, concurrency cancel, 30-minute timeout을 두고 `.nvmrc`, npm 11.16.0, `npm ci`, lint, typecheck, content check, Playwright Chromium install, production E2E 순으로 실행합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `29508f4668ea`부터 emitted artifact boundary를 명시적으로 다룹니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** 배포 품질 검사가 개발자 로컬 실행과 환경 차이에 의존했습니다. lockfile/toolchain과 production E2E를 자동화하지 않으면 source-level green이 release path를 대표하지 못합니다.
- **구현 결정과 경로:** pinned Node/npm, reproducible install과 production checks를 GitHub Actions required path로 만들었습니다. `.github/workflows/ci.yml`은 push/PR trigger, contents read-only permission, concurrency cancel, 30-minute timeout을 두고 `.nvmrc`, npm 11.16.0, `npm ci`, lint, typecheck, content check, Playwright Chromium install, production E2E 순으로 실행합니다.
- **소유권·실패 처리:** workflow가 environment/order/failure propagation을 소유하고 repository manifest/lockfile이 dependency resolution을 소유합니다. 각 shell step의 non-zero status가 job failure로 전파되고 stale run은 concurrency로 취소됩니다.
- **보장:** 동일 pinned environment에서 basic production build/E2E quality path를 재현합니다.
- **보장하지 않는 범위:** standalone artifact completeness, route budget, Lighthouse와 final Docker runtime은 아직 검사하지 않습니다.
- **후속 연결:** `29508f4668ea`부터 emitted artifact boundary를 명시적으로 다룹니다.

### 2. `29508f4668ea` — build: standalone server 산출물 생성

- **Importance:** B
- **Tags:** DEPLOY
- **Source-defined thread role:** Defines the traced server artifact boundary.
- **Source classification summary:** Configure Next.js to emit a standalone server bundle.
- **Source classification reason:** Supporting build or maintenance work inside the established release process; useful, but not a defining architectural or correctness decision.

#### Source에서 확정된 구현 의도와 상태 변화

Next.js가 traced runtime file을 포함한 standalone server bundle을 생성하도록 output mode를 설정합니다. 이후 container와 artifact check가 사용할 deployment boundary입니다.

#### 해당 SHA에서 확인할 실제 코드

- Next configuration에서 standalone output을 선택하는 setting을 확인합니다.
- Production build 후 standalone tree와 server entry를 filesystem에서 확인합니다.
- Development dependency tree 없이 runtime traced file이 포함되는지 구조를 기록합니다.
- Public/static asset이 standalone bundle에 자동 포함되는지 이 SHA output으로 확인합니다.
- Artifact를 생성하지만 existence/completeness를 아직 fail-closed로 검사하지 않는 점을 기록합니다.

확인 원칙:

- 먼저 `29508f4668ea^`와 `29508f4668ea`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Standalone artifact directory/file boundary | Next trace output이 runtime dependency boundary를 소유하고 source/development tree는 artifact 밖에 남을 수 있습니다. |
| Traced runtime과 development tree 차이 | `next.config.ts`의 standalone setting이 build 시 `.next/standalone/server.js`와 traced runtime tree를 생성하도록 요청합니다. |
| Static/public asset packaging 상태 | Next config에서 `output: "standalone"`을 선택했습니다. `next.config.ts`의 standalone setting이 build 시 `.next/standalone/server.js`와 traced runtime tree를 생성하도록 요청합니다. |
| 후속 verifier가 필요한 implicit assumption | `c0f7434467a0`이 required output existence를 fail-closed check로 만듭니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** Next build output은 일반 server deployment tree였고 container가 사용할 traced runtime boundary가 명시되지 않았습니다.
- **해당 SHA 핵심 코드:** `next.config.ts`의 standalone setting이 build 시 `.next/standalone/server.js`와 traced runtime tree를 생성하도록 요청합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `c0f7434467a0`이 required output existence를 fail-closed check로 만듭니다.

#### SHA별 복원 결론

- **Thread 내 역할:** Next config에서 `output: "standalone"`을 선택했습니다.
- **실제 변경:** `next.config.ts`의 standalone setting이 build 시 `.next/standalone/server.js`와 traced runtime tree를 생성하도록 요청합니다.
- **현재 보장:** 후속 verifier/container가 사용할 standalone artifact convention을 정의합니다.
- **남은 범위:** 실제 build 결과, `.next/static`, `public` 포함과 runtime serving은 보장하지 않습니다. `c0f7434467a0`이 required output existence를 fail-closed check로 만듭니다.

### 3. `c0f7434467a0` — test(build): standalone 산출물 완전성 검증

- **Importance:** A
- **Tags:** VALIDATION, DEPLOY, TEST
- **Source-defined thread role:** Makes required server and static outputs explicit.
- **Source classification summary:** Add an explicit post-build check for the standalone server entry point and static asset directory.
- **Source classification reason:** Significant because it verifies the production artifact or release path itself, closing a gap that source-level and development-server tests cannot cover.

#### Source에서 확정된 구현 의도와 상태 변화

Production build 뒤 standalone server entry와 static asset directory가 실제로 존재하는지 검사하고 누락 시 실패하는 post-build contract를 추가합니다.

#### 해당 SHA에서 확인할 실제 코드

- Verifier가 기대하는 exact output paths를 확인합니다.
- File/directory existence/type 검사와 error/exit branch를 추적합니다.
- Package command가 build 후 verifier를 실행하는 순서를 확인합니다.
- Standalone config만 있을 때와 verifier 이후의 explicit contract를 비교합니다.
- Temporary output 누락으로 failure를 재현합니다.

확인 원칙:

- 먼저 `c0f7434467a0^`와 `c0f7434467a0`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 대상 production invariant | standalone server entry와 static path가 검사 시점에 존재함을 보장합니다. |
| Failure injection과 diagnostic | missing path를 temporary removal로 주입할 수 있으며 script throw/non-zero가 diagnostic을 제공합니다. |
| Post-build filesystem assertion technique | build verification script는 required paths array를 `existsSync`로 검사해 missing list를 error로 throw하고 `package.json`에 `build:verify` command를 추가합니다. 실제로 file/directory type까지 구분하지는 않습니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| 증명하는 artifact와 아직 확인하지 않는 runtime serving/public asset | server가 실제 기동하거나 `public` asset을 serve하는지는 검사하지 않으며 이 SHA에서는 CI gate도 아직 아닙니다. |
| CI 연결 전 command usage | `abbd530368a0`이 verifier를 CI에 올리고 `b94fa6dd0118`이 actual runtime serving을 검사합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** standalone config만 존재해 compiler/version/config 변화로 required output이 누락돼도 별도 diagnostic이 없었습니다.
- **해당 SHA 핵심 코드:** build verification script는 required paths array를 `existsSync`로 검사해 missing list를 error로 throw하고 `package.json`에 `build:verify` command를 추가합니다. 실제로 file/directory type까지 구분하지는 않습니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `abbd530368a0`이 verifier를 CI에 올리고 `b94fa6dd0118`이 actual runtime serving을 검사합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** standalone config만 존재해 compiler/version/config 변화로 required output이 누락돼도 별도 diagnostic이 없었습니다. build 성공 status가 deployable server entry/static directory의 존재를 뜻하지 않을 수 있었습니다.
- **구현 결정과 경로:** post-build verifier에서 `.next/standalone/server.js`와 `.next/static` existence를 explicit contract로 검사했습니다. build verification script는 required paths array를 `existsSync`로 검사해 missing list를 error로 throw하고 `package.json`에 `build:verify` command를 추가합니다. 실제로 file/directory type까지 구분하지는 않습니다.
- **소유권·실패 처리:** Next build가 artifact를 생성하고 verifier가 release-required path 존재를 소유합니다. missing path를 temporary removal로 주입할 수 있으며 script throw/non-zero가 diagnostic을 제공합니다.
- **보장:** standalone server entry와 static path가 검사 시점에 존재함을 보장합니다.
- **보장하지 않는 범위:** server가 실제 기동하거나 `public` asset을 serve하는지는 검사하지 않으며 이 SHA에서는 CI gate도 아직 아닙니다.
- **후속 연결:** `abbd530368a0`이 verifier를 CI에 올리고 `b94fa6dd0118`이 actual runtime serving을 검사합니다.

### 4. `605b64512edf` — build(perf): route별 client asset 측정 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, PERF
- **Source-defined thread role:** Measures emitted JS and non-inlined CSS per route.
- **Source classification summary:** Add production-build measurement that reports the client JavaScript and non-inlined CSS attributable to each application route.
- **Source classification reason:** Significant because it removes a systematic loading cost or converts production performance into measured, reviewable, and enforceable output constraints.

#### Source에서 확정된 구현 의도와 상태 변화

Production build manifest를 읽어 각 public route의 client JavaScript와 non-inlined CSS actual uncompressed byte를 측정합니다. Shared chunk와 duplicate asset은 route 안에서 한 번만 계산합니다.

#### 해당 SHA에서 확인할 실제 코드

- `app-paths-manifest.json`에서 public route를 파생하고 framework internal entry를 제외하는 predicate를 확인합니다.
- Page output에서 client-reference manifest를 찾는 path/key mapping을 추적합니다.
- Route-specific JS와 root shared chunks를 합치는 순서를 확인합니다.
- Asset deduplication key와 actual filesystem byte read를 확인합니다.
- CSS inlining flag가 separate transfer count에서 제외되는 branch를 확인합니다.
- Dynamic route key와 generated manifest syntax를 처리하는 parser dependency를 기록합니다.
- 한 route result를 실제 file size 합으로 수동 검산합니다.

확인 원칙:

- 먼저 `605b64512edf^`와 `605b64512edf`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Authoritative measurement input과 source size 차이 | `scripts/route-budgets.mjs`는 `app-paths-manifest.json`에서 page route를 찾고 internal entries를 제외하며 build manifest root chunks와 route client-reference assets를 합칩니다. `Set`으로 route 내 asset을 deduplicate하고 `stat.size`의 uncompressed bytes를 더하며 inlined CSS는 separate transfer count에서 제외합니다. |
| Public route discovery 규칙 | production manifests와 filesystem stat에서 public route별 JS/non-inlined CSS bytes를 계산했습니다. |
| Shared/deduplicated asset accounting | production manifests와 filesystem stat에서 public route별 JS/non-inlined CSS bytes를 계산했습니다. `scripts/route-budgets.mjs`는 `app-paths-manifest.json`에서 page route를 찾고 internal entries를 제외하며 build manifest root chunks와 route client-reference assets를 합칩니다. `Set`으로 route 내 asset을 deduplicate하고 `stat.size`의 uncompressed bytes를 더하며 inlined CSS는 separate transfer count에서 제외합니다. |
| Inlined/non-inlined CSS 구분 | `scripts/route-budgets.mjs`는 `app-paths-manifest.json`에서 page route를 찾고 internal entries를 제외하며 build manifest root chunks와 route client-reference assets를 합칩니다. `Set`으로 route 내 asset을 deduplicate하고 `stat.size`의 uncompressed bytes를 더하며 inlined CSS는 separate transfer count에서 제외합니다. |
| Malformed/missing manifest failure path | missing/malformed manifest, unresolved file 또는 unsupported generated syntax는 parse/read failure로 command를 중단합니다. |
| 아직 policy failure를 만들지 않는 범위 | threshold/policy failure를 만들지 않고 compressed network cost나 runtime execution cost도 측정하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** client cost를 source size나 generic build 성공으로만 판단했고 route별 emitted asset 비용을 알 수 없었습니다.
- **해당 SHA 핵심 코드:** `scripts/route-budgets.mjs`는 `app-paths-manifest.json`에서 page route를 찾고 internal entries를 제외하며 build manifest root chunks와 route client-reference assets를 합칩니다. `Set`으로 route 내 asset을 deduplicate하고 `stat.size`의 uncompressed bytes를 더하며 inlined CSS는 separate transfer count에서 제외합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `518ff5b51ec5`가 measurement를 committed growth policy로 전환합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** client cost를 source size나 generic build 성공으로만 판단했고 route별 emitted asset 비용을 알 수 없었습니다. shared chunk, duplicate assets와 CSS inlining 때문에 source-level estimate는 실제 transfer boundary와 다를 수 있었습니다.
- **구현 결정과 경로:** production manifests와 filesystem stat에서 public route별 JS/non-inlined CSS bytes를 계산했습니다. `scripts/route-budgets.mjs`는 `app-paths-manifest.json`에서 page route를 찾고 internal entries를 제외하며 build manifest root chunks와 route client-reference assets를 합칩니다. `Set`으로 route 내 asset을 deduplicate하고 `stat.size`의 uncompressed bytes를 더하며 inlined CSS는 separate transfer count에서 제외합니다.
- **소유권·실패 처리:** Next emitted manifests가 route/asset membership을, filesystem artifact가 byte truth를 소유합니다. missing/malformed manifest, unresolved file 또는 unsupported generated syntax는 parse/read failure로 command를 중단합니다.
- **보장:** 각 emitted public route의 counted JS와 non-inlined CSS actual artifact bytes를 report합니다.
- **보장하지 않는 범위:** threshold/policy failure를 만들지 않고 compressed network cost나 runtime execution cost도 측정하지 않습니다.
- **후속 연결:** `518ff5b51ec5`가 measurement를 committed growth policy로 전환합니다.

### 5. `518ff5b51ec5` — build(perf): route bundle 성장 예산 평가 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, PERF
- **Source-defined thread role:** Turns measurements into fail-closed growth limits.
- **Source classification summary:** Define route bundle growth as a comparison against committed per-route JavaScript and CSS baselines, with a fixed five-percent allowance.
- **Source classification reason:** Significant because it removes a systematic loading cost or converts production performance into measured, reviewable, and enforceable output constraints.

#### Source에서 확정된 구현 의도와 상태 변화

Committed per-route baseline과 fixed 5% allowance를 비교해 missing route, new route, JavaScript/CSS growth를 structured violation으로 보고하는 fail-closed evaluator를 정의합니다.

#### 해당 SHA에서 확인할 실제 코드

- Baseline schema, route key, JS/CSS byte field, policy literal을 확인합니다.
- Allowed limit의 rounding과 정확히 5% 경계를 확인합니다.
- Expected route missing, emitted new route, JS overage, CSS overage branch를 각각 추적합니다.
- Violation record가 baseline/allowed/actual/asset class를 보존하는지 확인합니다.
- Baseline generation과 routine check가 별도 command가 되는 구조를 확인합니다.
- Evaluator가 current result로 baseline을 자동 rewrite하지 않는지 확인합니다.

확인 원칙:

- 먼저 `518ff5b51ec5^`와 `518ff5b51ec5`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 5% policy 계산식과 rounding | committed per-route baseline과 fixed 5% allowance를 비교하는 fail-closed evaluator를 추가했습니다. |
| Route coverage와 size growth failure 구분 | 정확히 floor limit은 통과하고 limit+1 byte는 실패합니다. Routine check는 current result로 baseline을 자동 rewrite하지 않습니다. |
| Structured diagnostic fields | `BUDGET_GROWTH_FACTOR = 1.05`, allowed limit는 `Math.floor(baseline * 1.05)`이고 actual이 allowed보다 클 때 위반입니다. Expected route missing, emitted new route, JS overage, CSS overage를 별도 structured record로 저장하며 baseline/allowed/actual/asset class를 보존합니다. |
| Exactly limit / limit+1 behavior | committed per-route baseline과 fixed 5% allowance를 비교하는 fail-closed evaluator를 추가했습니다. `BUDGET_GROWTH_FACTOR = 1.05`, allowed limit는 `Math.floor(baseline * 1.05)`이고 actual이 allowed보다 클 때 위반입니다. Expected route missing, emitted new route, JS overage, CSS overage를 별도 structured record로 저장하며 baseline/allowed/actual/asset class를 보존합니다. |
| Reviewable baseline update가 필요한 이유 | `abbd530368a0`이 evaluator를 release CI에 연결합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** route measurement는 가능했지만 현재 값을 그대로 받아들이면 growth regression을 차단할 기준이 없었습니다.
- **해당 SHA 핵심 코드:** `BUDGET_GROWTH_FACTOR = 1.05`, allowed limit는 `Math.floor(baseline * 1.05)`이고 actual이 allowed보다 클 때 위반입니다. Expected route missing, emitted new route, JS overage, CSS overage를 별도 structured record로 저장하며 baseline/allowed/actual/asset class를 보존합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `abbd530368a0`이 evaluator를 release CI에 연결합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** route measurement는 가능했지만 현재 값을 그대로 받아들이면 growth regression을 차단할 기준이 없었습니다. route가 사라지거나 새 route가 생기고 JS/CSS가 커져도 report만 생성하면 build는 성공했습니다.
- **구현 결정과 경로:** committed per-route baseline과 fixed 5% allowance를 비교하는 fail-closed evaluator를 추가했습니다. `BUDGET_GROWTH_FACTOR = 1.05`, allowed limit는 `Math.floor(baseline * 1.05)`이고 actual이 allowed보다 클 때 위반입니다. Expected route missing, emitted new route, JS overage, CSS overage를 별도 structured record로 저장하며 baseline/allowed/actual/asset class를 보존합니다.
- **소유권·실패 처리:** committed baseline이 reviewable policy state를 소유하고 routine checker는 current result만 읽어 비교합니다. 정확히 floor limit은 통과하고 limit+1 byte는 실패합니다. Routine check는 current result로 baseline을 자동 rewrite하지 않습니다.
- **보장:** route coverage와 5% growth limit이 fail-closed violation으로 변환됩니다.
- **보장하지 않는 범위:** baseline 자체가 최적이라는 사실, compressed bytes와 runtime UX는 보장하지 않습니다.
- **후속 연결:** `abbd530368a0`이 evaluator를 release CI에 연결합니다.

### 6. `1529ccf225c1` — build(perf): desktop Lighthouse 실행 경계 추가

- **Importance:** A
- **Tags:** PERF, DEPLOY
- **Source-defined thread role:** Defines repeatable route/design laboratory thresholds.
- **Source classification summary:** Add a reproducible desktop Lighthouse CI matrix for the home page and one enabled project detail page in all five designs.
- **Source classification reason:** Significant because it removes a systematic loading cost or converts production performance into measured, reviewable, and enforceable output constraints.

#### Source에서 확정된 구현 의도와 상태 변화

Production server를 대상으로 home과 enabled project detail을 five designs에서 세 번씩 측정하고 median으로 performance/accessibility/LCP/TBT/CLS threshold를 평가하는 desktop Lighthouse CI matrix를 정의합니다.

#### 해당 SHA에서 확인할 실제 코드

- Project identifier를 authoritative content에서 파생하는 config를 확인합니다.
- Production server start command, dedicated port, readiness/reuse behavior를 확인합니다.
- Five-design home/detail URL 생성과 query state를 확인합니다.
- Number of runs와 median aggregation을 확인합니다.
- Performance 0.90, accessibility 0.95, LCP 2.5s, TBT 200ms, CLS 0.1을 exact config로 기록합니다.
- Headless Chrome binary/environment dependency를 확인합니다.

확인 원칙:

- 먼저 `1529ccf225c1^`와 `1529ccf225c1`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Route/design coverage matrix | Lighthouse config는 authoritative content의 첫 enabled project ID를 사용하고 dedicated port 3300의 production server를 시작합니다. 각 URL은 3 runs, median이고 thresholds는 performance 0.90, accessibility 0.95, LCP 2500ms, TBT 200ms, CLS 0.1입니다. |
| Production server lifecycle | content가 representative detail route를, config가 matrix/run/threshold/server lifecycle을 소유합니다. |
| Run count와 aggregation policy | production server에서 5 design×home/detail 10 URLs를 3회 측정하고 median threshold를 적용하는 desktop Lighthouse matrix를 정의했습니다. |
| 각 threshold와 failure condition | server readiness 또는 Chrome binary가 없거나 median assertion을 위반하면 audit command가 실패합니다. |
| Lab result가 real-user field data를 증명하지 않는 범위 | real-user field data, 네트워크/기기 전체 분포와 모든 route 성능은 증명하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** bundle byte gate만으로 rendered production page의 lab performance/accessibility를 판단할 수 없었습니다.
- **해당 SHA 핵심 코드:** Lighthouse config는 authoritative content의 첫 enabled project ID를 사용하고 dedicated port 3300의 production server를 시작합니다. 각 URL은 3 runs, median이고 thresholds는 performance 0.90, accessibility 0.95, LCP 2500ms, TBT 200ms, CLS 0.1입니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `abbd530368a0`이 Playwright Chromium을 제공하고 audit를 CI gate로 승격합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** bundle byte gate만으로 rendered production page의 lab performance/accessibility를 판단할 수 없었습니다. 개발 서버 한 번의 수동 audit는 production mode, route/design variation과 measurement noise를 반영하지 못했습니다.
- **구현 결정과 경로:** production server에서 5 design×home/detail 10 URLs를 3회 측정하고 median threshold를 적용하는 desktop Lighthouse matrix를 정의했습니다. Lighthouse config는 authoritative content의 첫 enabled project ID를 사용하고 dedicated port 3300의 production server를 시작합니다. 각 URL은 3 runs, median이고 thresholds는 performance 0.90, accessibility 0.95, LCP 2500ms, TBT 200ms, CLS 0.1입니다.
- **소유권·실패 처리:** content가 representative detail route를, config가 matrix/run/threshold/server lifecycle을 소유합니다. server readiness 또는 Chrome binary가 없거나 median assertion을 위반하면 audit command가 실패합니다.
- **보장:** 고정된 lab 환경에서 대표 route/design의 configured threshold를 반복 평가합니다.
- **보장하지 않는 범위:** real-user field data, 네트워크/기기 전체 분포와 모든 route 성능은 증명하지 않습니다.
- **후속 연결:** `abbd530368a0`이 Playwright Chromium을 제공하고 audit를 CI gate로 승격합니다.

### 7. `abbd530368a0` — ci: 검증된 bundle과 Lighthouse gate 활성화

- **Importance:** A
- **Tags:** VALIDATION, PERF, DEPLOY
- **Source-defined thread role:** Promotes artifact, size, and audit limits into the release path.
- **Source classification summary:** Promote the production bundle and Lighthouse checks from local tooling into the CI release path.
- **Source classification reason:** Significant because it removes a systematic loading cost or converts production performance into measured, reviewable, and enforceable output constraints.

#### Source에서 확정된 구현 의도와 상태 변화

Standalone verification, route bundle budget, Lighthouse gate를 CI release path에 올립니다. Production E2E build를 한 번 사용하고 Playwright Chromium을 Lighthouse browser로 재사용하며 visual snapshots는 일반 CI E2E에서 제외합니다.

#### 해당 SHA에서 확인할 실제 코드

- Workflow가 production build를 한 번 만들고 이후 단계가 같은 artifact를 쓰는지 확인합니다.
- Standalone check, bundle check, Lighthouse command 순서를 추적합니다.
- Playwright-installed Chromium path를 Lighthouse에 전달하는 environment를 확인합니다.
- Visual snapshot test가 general CI command에서 제외되는 filter를 확인합니다.
- 각 gate failure가 job status로 전파되는지 확인합니다.
- 이전 기본 CI와 비교해 확장된 boundary를 목록화합니다.

확인 원칙:

- 먼저 `abbd530368a0^`와 `abbd530368a0`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Build-once artifact reuse | production E2E build를 한 번 만들고 같은 `.next` artifact에 verifier, budget, Lighthouse를 순서대로 적용했습니다. CI의 `test:e2e:ci`가 build+production E2E를 수행하며 `@visual` snapshot은 일반 command에서 제외합니다. 뒤이어 `build:verify`, bundle check, Playwright-installed Chromium path를 `CHROME_PATH`로 export한 Lighthouse audit가 같은 artifact를 사용합니다. |
| CI release gate sequence | production E2E build를 한 번 만들고 같은 `.next` artifact에 verifier, budget, Lighthouse를 순서대로 적용했습니다. CI의 `test:e2e:ci`가 build+production E2E를 수행하며 `@visual` snapshot은 일반 command에서 제외합니다. 뒤이어 `build:verify`, bundle check, Playwright-installed Chromium path를 `CHROME_PATH`로 export한 Lighthouse audit가 같은 artifact를 사용합니다. |
| Browser binary source | CI의 `test:e2e:ci`가 build+production E2E를 수행하며 `@visual` snapshot은 일반 command에서 제외합니다. 뒤이어 `build:verify`, bundle check, Playwright-installed Chromium path를 `CHROME_PATH`로 export한 Lighthouse audit가 같은 artifact를 사용합니다. |
| Visual regression 분리 이유 | production E2E build를 한 번 만들고 같은 `.next` artifact에 verifier, budget, Lighthouse를 순서대로 적용했습니다. CI의 `test:e2e:ci`가 build+production E2E를 수행하며 `@visual` snapshot은 일반 command에서 제외합니다. 뒤이어 `build:verify`, bundle check, Playwright-installed Chromium path를 `CHROME_PATH`로 export한 Lighthouse audit가 같은 artifact를 사용합니다. |
| Failure propagation과 diagnostic artifact | 어느 gate든 non-zero면 job/release path가 실패하며 browser path 부재도 명시적 failure가 됩니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** standalone/budget/Lighthouse tooling은 로컬 command였고 기본 CI release path가 실행하지 않았습니다.
- **해당 SHA 핵심 코드:** CI의 `test:e2e:ci`가 build+production E2E를 수행하며 `@visual` snapshot은 일반 command에서 제외합니다. 뒤이어 `build:verify`, bundle check, Playwright-installed Chromium path를 `CHROME_PATH`로 export한 Lighthouse audit가 같은 artifact를 사용합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `b87a2b453741`과 `b94fa6dd0118`이 final image와 real HTTP contract를 닫습니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** standalone/budget/Lighthouse tooling은 로컬 command였고 기본 CI release path가 실행하지 않았습니다. 검사 도구가 있어도 required automation에서 빠지면 regression을 merge 전에 막지 못합니다.
- **구현 결정과 경로:** production E2E build를 한 번 만들고 같은 `.next` artifact에 verifier, budget, Lighthouse를 순서대로 적용했습니다. CI의 `test:e2e:ci`가 build+production E2E를 수행하며 `@visual` snapshot은 일반 command에서 제외합니다. 뒤이어 `build:verify`, bundle check, Playwright-installed Chromium path를 `CHROME_PATH`로 export한 Lighthouse audit가 같은 artifact를 사용합니다.
- **소유권·실패 처리:** CI가 build-once artifact lifecycle과 gate order를 소유하고 visual snapshot은 별도 환경-sensitive suite로 분리됩니다. 어느 gate든 non-zero면 job/release path가 실패하며 browser path 부재도 명시적 failure가 됩니다.
- **보장:** standalone existence, route growth와 Lighthouse thresholds가 실제 release CI에 적용됩니다.
- **보장하지 않는 범위:** Docker packaging/runtime, external deployment, visual snapshots의 일반 CI 실행은 아직 보장하지 않습니다.
- **후속 연결:** `b87a2b453741`과 `b94fa6dd0118`이 final image와 real HTTP contract를 닫습니다.

### 8. `b87a2b453741` — build(docker): public 자산을 포함한 비루트 standalone image 추가

- **Importance:** A
- **Tags:** DEPLOY
- **Source-defined thread role:** Packages the verified artifact with explicit public assets and non-root runtime.
- **Source classification summary:** Add a multi-stage container build around the verified Next.js standalone artifact.
- **Source classification reason:** Significant because it changes or hardens the reproducible production-build and runtime boundary rather than merely adjusting local tooling.

#### Source에서 확정된 구현 의도와 상태 변화

Verified standalone artifact를 multi-stage Docker image로 package합니다. Builder는 pinned Node/npm과 build-time content mode/origin을 사용하고 runtime image는 unprivileged `node` user로 standalone, static, public만 포함합니다.

#### 해당 SHA에서 확인할 실제 코드

- Dependency, builder, runtime stage의 base image와 copy boundary를 확인합니다.
- Pinned Node/npm version과 install command가 repository contract와 일치하는지 확인합니다.
- Content mode와 public origin build args/env가 readiness에 전달되는지 추적합니다.
- Builder가 production build와 standalone verifier를 통과해야 runtime stage가 만들어지는지 확인합니다.
- Runtime stage의 `.next/standalone`, `.next/static`, `public` copy를 각각 확인합니다.
- Final `USER node`, port, startup command, filesystem ownership을 확인합니다.
- Docker ignore가 git, local deps, tests, logs, env files를 제외하는지 확인합니다.

확인 원칙:

- 먼저 `b87a2b453741^`와 `b87a2b453741`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| Stage별 artifact ownership | deps stage는 installed dependency, builder는 verified build output, runner는 minimal runtime files와 ownership을 소유합니다. |
| Build-time publication inputs | pinned multi-stage Docker build로 dependency/builder/runtime를 나누고 standalone/static/public만 non-root image에 복사했습니다. `Dockerfile`은 `node:24.18.0-bookworm-slim`, npm 11.16.0, `npm ci`, build args/env `PORTFOLIO_CONTENT_MODE`/`SITE_URL`, build+verify를 사용합니다. Runner는 `.next/standalone`, `.next/static`, `public`을 복사하고 `USER node`, port 3100, `node server.js`로 시작합니다. `.dockerignore`는 git, deps, tests, logs, env 등을 제외합니다. |
| Public asset을 explicit copy해야 하는 이유 | pinned multi-stage Docker build로 dependency/builder/runtime를 나누고 standalone/static/public만 non-root image에 복사했습니다. `Dockerfile`은 `node:24.18.0-bookworm-slim`, npm 11.16.0, `npm ci`, build args/env `PORTFOLIO_CONTENT_MODE`/`SITE_URL`, build+verify를 사용합니다. Runner는 `.next/standalone`, `.next/static`, `public`을 복사하고 `USER node`, port 3100, `node server.js`로 시작합니다. `.dockerignore`는 git, deps, tests, logs, env 등을 제외합니다. |
| Non-root runtime boundary | deps stage는 installed dependency, builder는 verified build output, runner는 minimal runtime files와 ownership을 소유합니다. |
| Image에 포함되지 않는 development/source material | 실제 image build, process startup, route/asset HTTP/MIME은 아직 실행 검증하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** verified `.next` artifact가 있어도 runtime image의 copy boundary, publication env와 user가 명시되지 않았습니다.
- **해당 SHA 핵심 코드:** `Dockerfile`은 `node:24.18.0-bookworm-slim`, npm 11.16.0, `npm ci`, build args/env `PORTFOLIO_CONTENT_MODE`/`SITE_URL`, build+verify를 사용합니다. Runner는 `.next/standalone`, `.next/static`, `public`을 복사하고 `USER node`, port 3100, `node server.js`로 시작합니다. `.dockerignore`는 git, deps, tests, logs, env 등을 제외합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `b94fa6dd0118`이 container를 실제 build/start하고 contract를 검사합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** verified `.next` artifact가 있어도 runtime image의 copy boundary, publication env와 user가 명시되지 않았습니다. public assets 누락, root 실행 또는 source/development material 과포장이 가능했습니다.
- **구현 결정과 경로:** pinned multi-stage Docker build로 dependency/builder/runtime를 나누고 standalone/static/public만 non-root image에 복사했습니다. `Dockerfile`은 `node:24.18.0-bookworm-slim`, npm 11.16.0, `npm ci`, build args/env `PORTFOLIO_CONTENT_MODE`/`SITE_URL`, build+verify를 사용합니다. Runner는 `.next/standalone`, `.next/static`, `public`을 복사하고 `USER node`, port 3100, `node server.js`로 시작합니다. `.dockerignore`는 git, deps, tests, logs, env 등을 제외합니다.
- **소유권·실패 처리:** deps stage는 installed dependency, builder는 verified build output, runner는 minimal runtime files와 ownership을 소유합니다. readiness/build/verify가 실패하면 runtime stage가 완성되지 않고 non-root user가 final process identity가 됩니다.
- **보장:** image definition이 explicit public assets와 non-root standalone runtime을 포함합니다.
- **보장하지 않는 범위:** 실제 image build, process startup, route/asset HTTP/MIME은 아직 실행 검증하지 않습니다.
- **후속 연결:** `b94fa6dd0118`이 container를 실제 build/start하고 contract를 검사합니다.

### 9. `b94fa6dd0118` — test(docker): runtime route와 public 자산 검증 자동화

- **Importance:** A
- **Tags:** ARCH, VALIDATION, ROUTING
- **Source-defined thread role:** Exercises the final container and derives asset coverage from authoritative content.
- **Source classification summary:** Add an end-to-end container contract that builds the image, starts it on an isolated ephemeral port, waits for readiness, and verifies that the configured runtime user is `node`.
- **Source classification reason:** Significant because it standardizes a cross-route design, navigation, shell, or dispatch boundary instead of adding isolated page markup.

#### Source에서 확정된 구현 의도와 상태 변화

Container image를 실제 build/start하고 isolated port에서 readiness를 기다린 뒤 runtime user, home/detail HTML, 모든 content-derived public asset의 body와 MIME을 검증합니다. Unique names, failure logs, `finally` cleanup도 포함합니다.

#### 해당 SHA에서 확인할 실제 코드

- Image/container name과 ephemeral host port 생성 방식을 확인합니다.
- Docker build/run, readiness polling, timeout/failure log collection을 추적합니다.
- Configured user가 `node`인지 inspect하는 command를 확인합니다.
- Home와 enabled project detail route의 non-empty successful HTML assertion을 확인합니다.
- Authoritative JSON을 recursive traversal해 asset URL set을 만드는 code와 dedup을 확인합니다.
- Extension별 expected MIME과 response body assertion을 확인합니다.
- `finally` cleanup이 성공/실패 모두에서 container/image를 제거하는지 확인합니다.

확인 원칙:

- 먼저 `b94fa6dd0118^`와 `b94fa6dd0118`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 대상 invariant: deployable image의 user/routes/assets | packaged standalone/public serving, configured non-root user와 selected routes/assets의 actual HTTP/MIME contract를 검증하도록 구현됐습니다. |
| Docker build + real process + HTTP/MIME technique | container test script는 PID+random 이름, Docker build/run, 60회×1초 readiness polling, `.Config.User === "node"`, `/`와 `/projects/example-project?view=classic`의 200/non-empty/text-html을 검사합니다. 모든 `src/content/*.json`을 재귀 순회해 `content\|template/` asset 문자열을 Set으로 deduplicate/sort하고 extension별 MIME, 200, non-empty body를 검사합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| Production path와 content source traversal | container test script는 PID+random 이름, Docker build/run, 60회×1초 readiness polling, `.Config.User === "node"`, `/`와 `/projects/example-project?view=classic`의 200/non-empty/text-html을 검사합니다. 모든 `src/content/*.json`을 재귀 순회해 `content\|template/` asset 문자열을 Set으로 deduplicate/sort하고 extension별 MIME, 200, non-empty body를 검사합니다. |
| 증명하는 것: packaged standalone/public serving/non-root | packaged standalone/public serving, configured non-root user와 selected routes/assets의 actual HTTP/MIME contract를 검증하도록 구현됐습니다. |
| 증명하지 않는 것: orchestration, TLS, external proxy, production load | orchestration, TLS, reverse proxy, production load, all route/content combinations는 증명하지 않습니다. |
| Failure logs와 cleanup lifecycle | 실패 시 started container logs를 수집하고 main failure 여부와 별개로 `finally`에서 container 후 image를 제거합니다. Cleanup error는 기존 main failure가 없을 때만 rethrow합니다. |
| CI에서 막는 회귀 | unique image/container를 build/start하고 ephemeral loopback port에서 runtime user, routes, content-derived assets와 cleanup을 검증했습니다. container test script는 PID+random 이름, Docker build/run, 60회×1초 readiness polling, `.Config.User === "node"`, `/`와 `/projects/example-project?view=classic`의 200/non-empty/text-html을 검사합니다. 모든 `src/content/*.json`을 재귀 순회해 `content\|template/` asset 문자열을 Set으로 deduplicate/sort하고 extension별 MIME, 200, non-empty body를 검사합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** Dockerfile의 copy/user 설정은 정적 선언이라 실제 process와 HTTP serving이 성공한다는 증거가 없었습니다.
- **해당 SHA 핵심 코드:** container test script는 PID+random 이름, Docker build/run, 60회×1초 readiness polling, `.Config.User === "node"`, `/`와 `/projects/example-project?view=classic`의 200/non-empty/text-html을 검사합니다. 모든 `src/content/*.json`을 재귀 순회해 `content|template/` asset 문자열을 Set으로 deduplicate/sort하고 extension별 MIME, 200, non-empty body를 검사합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** 이 commit으로 source checks에서 final runnable image까지 release contract가 연결됩니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** Dockerfile의 copy/user 설정은 정적 선언이라 실제 process와 HTTP serving이 성공한다는 증거가 없었습니다. image가 build돼도 route가 비거나 asset MIME/body가 잘못되고 cleanup이 누락될 수 있었습니다.
- **구현 결정과 경로:** unique image/container를 build/start하고 ephemeral loopback port에서 runtime user, routes, content-derived assets와 cleanup을 검증했습니다. container test script는 PID+random 이름, Docker build/run, 60회×1초 readiness polling, `.Config.User === "node"`, `/`와 `/projects/example-project?view=classic`의 200/non-empty/text-html을 검사합니다. 모든 `src/content/*.json`을 재귀 순회해 `content|template/` asset 문자열을 Set으로 deduplicate/sort하고 extension별 MIME, 200, non-empty body를 검사합니다.
- **소유권·실패 처리:** authoritative JSON이 asset coverage를, running container가 user/process/HTTP behavior를, script `finally`가 container/image cleanup을 소유합니다. 실패 시 started container logs를 수집하고 main failure 여부와 별개로 `finally`에서 container 후 image를 제거합니다. Cleanup error는 기존 main failure가 없을 때만 rethrow합니다.
- **보장:** packaged standalone/public serving, configured non-root user와 selected routes/assets의 actual HTTP/MIME contract를 검증하도록 구현됐습니다.
- **보장하지 않는 범위:** orchestration, TLS, reverse proxy, production load, all route/content combinations는 증명하지 않습니다.
- **후속 연결:** 이 commit으로 source checks에서 final runnable image까지 release contract가 연결됩니다.

## 6. Invariant ledger

| Invariant | Source에서 확인된 변화 지점 | 해당 SHA의 실제 code/test evidence | 부족함이 드러난 시점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| CI는 pinned toolchain과 reproducible install로 production path를 검사합니다. | `9fd3541c11dc` | `.github/workflows/ci.yml`: `.nvmrc`, npm 11.16.0, `npm ci`, ordered lint/type/content/build/E2E, read-only/concurrency/timeout. | 개발자 로컬 품질 실행은 environment/order가 재현되지 않았습니다. | pinned install과 basic production path가 automation failure status로 연결됩니다. |
| Standalone server entry와 static directory는 explicit artifact입니다. | `29508f4668ea → c0f7434467a0` | `next.config.ts output:"standalone"` 및 post-build required paths `.next/standalone/server.js`, `.next/static`. | config만으로 actual artifact 존재를 알 수 없었습니다. | verifier가 두 path의 existence를 explicit release contract로 만듭니다. Type/content까지는 검사하지 않습니다. |
| Route budget은 emitted JS와 non-inlined CSS의 actual byte를 기준으로 합니다. | `605b64512edf` | `route-budgets.mjs`의 app/build/client-reference manifests, route discovery, Set dedup, `stat.size`, non-inlined CSS branch. | source size나 build success가 route transfer assets를 나타내지 못했습니다. | emitted artifact membership과 uncompressed actual bytes를 route별 measurement truth로 사용합니다. |
| Baseline route coverage와 5% growth limit은 fail-closed입니다. | `518ff5b51ec5` | baseline schema/evaluator의 `Math.floor(baseline*1.05)`, missing/new route, JS/CSS violation records. | measurement만 보고 current result를 자동 수용할 위험이 있었습니다. | committed route set과 exact 5% floor limit을 초과하거나 coverage가 달라지면 fail-closed입니다. |
| Lighthouse는 production server와 five-design representative routes를 반복 측정합니다. | `1529ccf225c1` | Lighthouse config의 5 design×home/detail 10 URLs, 3 runs median, production port와 five thresholds. | one-off dev-server audit는 production artifact와 variation/noise를 대표하지 못했습니다. | representative production routes를 repeatable desktop lab policy로 평가합니다. |
| Release CI는 standalone, route budget, Lighthouse gate를 실제 build에 적용합니다. | `abbd530368a0` | CI에서 build-once production E2E 후 artifact verify, bundle check, Playwright Chromium Lighthouse gate. | tooling이 local-only면 merge/release를 차단하지 못했습니다. | 동일 emitted artifact에 세 release gates가 순차 적용되고 non-zero가 job failure가 됩니다. |
| Final image는 public assets를 포함하고 non-root로 실행되며 real HTTP로 검증됩니다. | `b87a2b453741 → b94fa6dd0118` | Dockerfile standalone/static/public copies, `USER node`, runtime test의 inspect/routes/content-derived assets/MIME/finally cleanup. | 정적 Dockerfile만으로 actual process/serving을 증명하지 못했습니다. | final image의 configured user와 selected real HTTP routes/assets를 running container contract로 검사하도록 구현됐습니다. |

Ledger 작성 원칙:

- Source가 명시한 invariant만 사용하고 새 invariant를 확정 사실처럼 추가하지 않습니다.
- 도입, 강화, 부족함 노출, fix, regression test가 서로 다른 commit이면 각 열에 분리해 기록합니다.
- Code evidence에는 실제 field, function, branch, selector, command 또는 assertion을 적습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 위험 | 대응 commit | 실제 수정/강화 code에서 확인할 것 | Test 또는 실행 증거 |
| --- | --- | --- | --- |
| Source checks는 통과하지만 server entry/static asset이 없음 | 29508f4668ea → c0f7434467a0 | Standalone output과 post-build filesystem check를 확인합니다. | 필수 path 누락의 message와 exit를 기록합니다. |
| Route chunk가 커졌거나 route가 사라졌지만 build는 성공 | 605b64512edf → 518ff5b51ec5 | Manifest/asset byte를 측정해 committed baseline과 비교합니다. | 정확히 5%와 5%+1 byte 경계를 확인합니다. |
| Development server는 빠르지만 production route/design이 기준 미달 | 1529ccf225c1 → abbd530368a0 | Production server, 3 runs, median, explicit thresholds를 확인합니다. | CI가 audit failure를 release failure로 전달하는 chain을 기록합니다. |
| Image가 public asset을 누락하거나 root로 실행 | b87a2b453741 → b94fa6dd0118 | Docker copy/user contract와 runtime HTTP/MIME/user 검증을 확인합니다. | Failure logs와 `finally` cleanup을 확인합니다. |

### 실제 연결 기록

- Standalone verifier는 `existsSync`를 사용하므로 이름이 맞는 path의 file/directory type 또는 runtime serving은 증명하지 않습니다. Container test가 다른 계층의 공백을 닫습니다.
- Budget limit은 `floor(baseline × 1.05)`이고 actual이 정확히 limit이면 통과, limit+1이면 실패합니다. Baseline update는 normal check와 분리됩니다.
- Container asset set은 hard-coded list가 아니라 all content JSON의 `content|template/` strings를 재귀 수집하며 failure 여부와 무관하게 cleanup합니다.

## 8. Ownership / state / responsibility 변화

| Concern | Thread 초기 owner/state | Thread 최종 owner/state | 실제 symbol과 호출 경로 |
| --- | --- | --- | --- |
| General quality gate | Developer local execution | CI workflow | `.github/workflows/ci.yml`: pinned environment, install/check order와 release status. |
| Runtime package boundary | Implicit Next build output | Standalone output + verifier | `next.config.ts` output + build verifier script: traced runtime convention과 required path existence. |
| Client cost measurement | Source size/estimate | Emitted manifests와 asset bytes | `scripts/route-budgets.mjs`: Next manifests/assets → route JS/CSS byte measurement. |
| Growth policy | Check가 current value를 수용할 위험 | Committed baseline + separate baseline/check | committed baseline + evaluator/check command: reviewable route set와 5% growth policy. |
| Lab performance | One-off manual audit | Repeatable Lighthouse matrix | Lighthouse config/CI: production server lifecycle, URL matrix, browser path, 3-run median thresholds. |
| Deployable image | Artifact copy assumption | Multi-stage Dockerfile + HTTP contract test | `Dockerfile` → container contract script: minimal image assembly → non-root process/readiness/routes/assets/MIME → cleanup. |

## 9. Thread 최종 상태

### Source에서 확정된 최종 상태

Source-level green 상태를 넘어 실제 Next.js production output, route별 client asset, Lighthouse 결과, standalone package, non-root container와 public asset serving을 fail-closed release contract로 만드는 과정을 복원합니다.

### 학습자가 완성할 최종 설명

- **Thread 시작 시점의 설계와 위험:** 초기 CI는 reproducible source/build/E2E checks를 제공했지만 emitted standalone tree, route cost와 final image behavior까지는 확인하지 않았습니다.
- **핵심 architecture/decision이 형성된 순서:** standalone output/verification, route measurement/baseline, production Lighthouse, CI promotion, multi-stage image, real container HTTP test 순서로 release boundary가 확장됐습니다.
- **실제 failure 또는 부족함이 드러난 지점:** build success가 deployable entry/static path를 뜻하지 않고 source size가 emitted route cost를 뜻하지 않으며 Dockerfile 선언이 실제 serving을 뜻하지 않는 각각의 공백이 드러났습니다.
- **Fix 또는 boundary 강화가 바꾼 invariant:** 각 공백을 artifact existence, manifest byte accounting, committed policy, repeated audit, actual process/HTTP assertion으로 별도 강화했습니다.
- **Test/build/browser evidence가 보장한 범위:** 스크립트와 workflow의 exact commands, predicates, thresholds, cleanup을 정적으로 확인했습니다. Node/Chromium/Docker command는 환경 제한 때문에 실행하지 않았습니다.
- **Thread 종료 시점에도 보장하지 않는 범위:** field performance, orchestration/TLS/proxy, production load, 모든 route와 compressed/runtime execution cost는 최종 contract 범위 밖입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 source-backed flow의 각 단계에 실제 file path, symbol, input/output, failure branch를 추가합니다.

1. Pinned Node/npm 환경에서 reproducible dependency install을 수행합니다.
   - 실제 코드 위치: `.github/workflows/ci.yml`, `.nvmrc`, `package.json`
   - 입력과 출력: pinned Node/npm과 lockfile에서 `npm ci`로 dependency tree를 재현합니다.
   - 실패/absence 처리: install/version mismatch는 첫 단계에서 job을 실패시킵니다.
2. Lint, type, content/readiness, production build를 실행합니다.
   - 실제 코드 위치: CI/package scripts
   - 입력과 출력: lint, typecheck, content check/readiness, production build/E2E를 ordered shell steps로 실행합니다.
   - 실패/absence 처리: 어느 command든 non-zero면 뒤 gate로 진행하지 않습니다.
3. Build output에서 standalone server entry와 static directory를 확인합니다.
   - 실제 코드 위치: standalone verifier script
   - 입력과 출력: `.next/standalone/server.js`와 `.next/static` existence를 검사합니다.
   - 실패/absence 처리: missing paths를 목록으로 throw합니다. Path type/runtime serving은 후속 test가 담당합니다.
4. Manifest를 읽어 route별 emitted JS와 non-inlined CSS byte를 계산합니다.
   - 실제 코드 위치: `scripts/route-budgets.mjs` measurement
   - 입력과 출력: manifests에서 public route와 shared/route assets를 수집하고 deduplicated JS/CSS filesystem bytes를 계산합니다.
   - 실패/absence 처리: missing/malformed manifest 또는 asset read failure는 command failure가 됩니다.
5. Committed baseline과 5% policy로 route coverage와 growth를 평가합니다.
   - 실제 코드 위치: route baseline evaluator/check
   - 입력과 출력: measured routes를 committed JS/CSS baselines와 floor 5% limits로 비교합니다.
   - 실패/absence 처리: missing expected route, new route, actual > allowed가 structured violation/non-zero status가 됩니다.
6. Production server에서 five-design home/detail Lighthouse matrix를 실행합니다.
   - 실제 코드 위치: Lighthouse CI config
   - 입력과 출력: production server에서 five-design home/detail URLs를 3회 audit하고 median categories/metrics를 threshold와 비교합니다.
   - 실패/absence 처리: server/browser/readiness 또는 threshold failure가 audit failure로 전파됩니다.
7. Verified standalone/static/public을 non-root image로 조립합니다.
   - 실제 코드 위치: `Dockerfile`
   - 입력과 출력: verified standalone, static, public을 `node` owner의 runtime image로 복사하고 port 3100 server를 정의합니다.
   - 실패/absence 처리: builder build/verify 실패면 runner image가 완성되지 않습니다.
8. Container를 기동하고 user, routes, content-derived assets, MIME, cleanup을 검증합니다.
   - 실제 코드 위치: container contract script
   - 입력과 출력: unique image/container를 시작하고 inspect user, readiness, routes, JSON-derived assets/body/MIME를 확인합니다.
   - 실패/absence 처리: 실패 로그를 수집하고 성공/실패 모두 `finally`에서 container와 image를 정리합니다.

### 코드 없이 설명하기

> 이 Thread의 최종 실행 흐름을 code snippet 없이 자신의 말로 작성합니다. 설계 → 구현 → failure/risk → 수정/강화 → 검증 순서가 드러나야 합니다.

basic CI로 pinned install과 production E2E를 재현한 뒤 Next standalone output을 만들고 required entry/static path를 검사했습니다. Compiler manifests와 actual files에서 route별 JS와 non-inlined CSS를 계산하고 committed baseline의 5% limit으로 fail-closed policy를 만들었습니다. Production server의 five-design home/detail을 Lighthouse로 세 번씩 측정해 median thresholds를 적용하고 이 gates를 CI에 올렸습니다. 마지막에는 verified standalone/static/public만 non-root image로 조립하고 real container에서 user, HTML routes, content-derived assets와 MIME를 검사한 뒤 항상 정리합니다.

> **실행 증거 구분:** 참조 SHA의 diff와 해당 SHA 파일은 GitHub connector로 확인했습니다. 로컬 clone은 실행 환경의 DNS 차단으로 실패해 build, unit/E2E, browser, Lighthouse, Docker command는 실행하지 않았습니다. 따라서 위 test 결과는 구현된 test technique과 assertion의 정적 검토이며 실제 통과 결과가 아닙니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 실제로 checkout하거나 diff로 확인했습니다.
- [x] Thread 내 commit 순서를 source와 동일하게 유지했습니다.
- [x] 각 A/S commit에서 직전 상태, decision, failure boundary, guarantee와 non-guarantee를 구분했습니다.
- [x] B commit은 thread 흐름에서 필요한 구현 역할과 state change를 확인했습니다.
- [x] Fix commit을 독립 feature가 아니라 기존 가정 → failure → root cause → corrected invariant로 설명했습니다.
- [x] Test commit에서 production invariant, failure injection, technique, traversed production path, proves/does-not-prove를 구분했습니다.
- [x] Invariant ledger의 모든 주장에 해당 SHA의 code/test evidence가 있습니다.
- [x] Final HEAD의 code를 과거 commit 설명에 소급하지 않았습니다.
- [x] Thread 최종 흐름을 code 없이 설명할 수 있습니다.
===== END FILE: 06-production-artifact-and-performance-enforcement.md =====

===== BEGIN FILE: 07-accessibility-policy-to-cross-design-regression-evidence.md =====
# Thread: Accessibility policy to cross-design regression evidence

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> 이 문서는 source에서 확정된 thread 구조와 commit 역할만 미리 제공합니다. 실제 구현 해석, code evidence, failure 재현, test 결과, 최종 설명은 해당 SHA의 코드를 직접 확인해 작성합니다.

## 1. Thread 목표

국소 reduced-motion 처리에서 시작해 global motion policy, contrast, skip-link focus, semantic definition list를 수정하고 다섯 design과 모든 enabled route를 browser-level WCAG matrix로 검증하는 과정을 복원합니다.

**Source-defined significance**

> Accessibility develops from local motion rules into cross-design invariants backed by browser evidence. Later fixes show that visual independence can produce semantic and contrast regressions, so the final matrix tests landmarks, Axe rules, and keyboard focus across the full route/design product.

### 이 Thread에 직접 연결되는 Critical Invariants

- Reduced-motion에서는 nonessential animation, transition, smooth scroll이 기능 전제가 아닙니다. — `29bb40579cb2 → af9191fc15ad`
- Accent text는 light와 dark surface 모두에서 구분 가능한 token을 사용합니다. — `5a6fd8a802ff`
- Skip link는 main landmark로 programmatic focus를 이동합니다. — `a15e117cb51b → e1aac08e0e9e`
- Definition-list 값은 `<dd>`이고 landmarks는 route마다 유일합니다. — `e1aac08e0e9e → 84c71d027630`
- 모든 enabled route와 five designs는 configured WCAG A/AA scan과 keyboard skip path를 통과합니다. — `84c71d027630`

### 연결되는 Major Engineering Difficulty

- 서로 다른 다섯 renderer에서 motion, contrast, focus, semantics, landmarks를 하나의 cross-design invariant로 유지하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- Selector-by-selector motion override가 왜 새 animation을 놓칠 수 있으며 global policy는 이를 어떻게 막는가?
- Light/dark surface에서 같은 accent token을 공유한 것이 어떤 contrast trade-off를 만들었는가?
- Skip link가 viewport만 이동하고 keyboard focus를 옮기지 못한 root cause는 무엇인가?
- Brutalist metric의 `<dl>` 내부 element가 visual output은 유지하면서 semantic contract를 깨는 이유는 무엇인가?
- Final Axe/landmark/keyboard matrix가 보장하는 범위와 보장하지 않는 범위는 무엇인가?

## 3. 완료 기준

- Local motion override와 global `prefers-reduced-motion` policy의 scope/specificity를 비교했습니다.
- Contrast token fix의 light/dark surface 분리와 consumer selector를 기록했습니다.
- 각 shell의 skip link target, main landmark, `tabIndex`, focus assertion을 추적했습니다.
- Brutalist `<dl>/<dt>/<dd>` 구조와 CSS selector를 수정 전후로 비교했습니다.
- `84c71d027630`의 route×design matrix, Axe rules, landmark count, keyboard path를 기록했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `29bb40579cb2` | style(a11y): 동적 목록의 모션 감소 지원 | B | RENDERER, A11Y | Extends the initial reduced-motion coverage to repeated elements. |
| 2 | `af9191fc15ad` | style(a11y): 모바일 헤더와 동작 감소 보강 | A | ARCH, RENDERER, A11Y | Replaces selector-by-selector coverage with a global motion contract. |
| 3 | `5a6fd8a802ff` | fix(a11y): 디자인별 색상 대비 보정 | A | A11Y, DEBUG | Corrects shared and Editorial contrast tokens. |
| 4 | `a15e117cb51b` | fix(a11y): skip link focus target 복원 | A | A11Y, DEBUG | Restores programmatic focus transfer in shared shells. |
| 5 | `e1aac08e0e9e` | fix(a11y): Brutalist 지표의 definition semantics 수정 | A | ARCH, RENDERER, A11Y | Repairs definition-list semantics and the remaining shell focus boundary. |
| 6 | `84c71d027630` | test(a11y): 디자인×route WCAG 행렬 추가 | A | ARCH, ROUTING, A11Y | Verifies every enabled route under every design and a keyboard skip path. |

## 5. Commit별 학습 기록

각 section은 반드시 해당 SHA를 checkout한 상태에서 작성합니다. Thread 내 이전 commit은 비교 대상으로 사용할 수 있지만 final HEAD를 정답처럼 소급하지 않습니다.

### 1. `29bb40579cb2` — style(a11y): 동적 목록의 모션 감소 지원

- **Importance:** B
- **Tags:** RENDERER, A11Y
- **Source-defined thread role:** Extends the initial reduced-motion coverage to repeated elements.
- **Source classification summary:** Extend the reduced-motion override to technology chips and the animated guide elements used by experience and journey lists.
- **Source classification reason:** Necessary renderer and responsive implementation within an established visual system; it completes presentation behavior without redefining project-wide architecture or correctness.

#### Source에서 확정된 구현 의도와 상태 변화

Technology chip과 experience/journey guide 같은 반복 요소의 transform/transition을 기존 reduced-motion override에 포함해 hero 밖의 motion gap을 줄입니다.

#### 해당 SHA에서 확인할 실제 코드

- 기존 `prefers-reduced-motion` block과 새 chip/guide selector를 비교합니다.
- 각 selector에서 transform, transition, animation 중 무엇을 disable하는지 확인합니다.
- Motion 없이 content visibility/order가 동일한지 markup과 함께 확인합니다.
- Selector list 방식이 future motion을 자동 포함하지 않는 한계를 기록합니다.

확인 원칙:

- 먼저 `29bb40579cb2^`와 `29bb40579cb2`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 초기 motion policy coverage | 기존 reduced-motion block은 `.reveal-item`, `.motion-card`만 transform/transition을 껐습니다. |
| 새로 닫힌 repeated-list/timeline gap | motion만 제거하므로 information visibility와 interaction target은 유지됩니다. |
| Motion 제거 후 유지되는 information/interaction | hero 밖 반복 list/timeline의 현재 transform/transition gap을 닫습니다. |
| 후속 global policy가 필요한 이유 | `af9191fc15ad`가 selector별 방식 대신 global reduced-motion contract를 추가합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 기존 reduced-motion block은 `.reveal-item`, `.motion-card`만 transform/transition을 껐습니다.
- **해당 SHA 핵심 코드:** `src/app/globals.css`의 `prefers-reduced-motion` block이 `.tech-chip`, `.experience-row::before`, `.paired-timeline::before`, `.timeline-list::before`까지 `transform: none; transition: none;`을 적용합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `af9191fc15ad`가 selector별 방식 대신 global reduced-motion contract를 추가합니다.

#### SHA별 복원 결론

- **Thread 내 역할:** 같은 local media block의 selector list에 repeated elements를 추가했습니다.
- **실제 변경:** `src/app/globals.css`의 `prefers-reduced-motion` block이 `.tech-chip`, `.experience-row::before`, `.paired-timeline::before`, `.timeline-list::before`까지 `transform: none; transition: none;`을 적용합니다.
- **현재 보장:** hero 밖 반복 list/timeline의 현재 transform/transition gap을 닫습니다.
- **남은 범위:** 새 animation selector를 자동 포함하지 않는 allow-list 방식의 한계가 남습니다. `af9191fc15ad`가 selector별 방식 대신 global reduced-motion contract를 추가합니다.

### 2. `af9191fc15ad` — style(a11y): 모바일 헤더와 동작 감소 보강

- **Importance:** A
- **Tags:** ARCH, RENDERER, A11Y
- **Source-defined thread role:** Replaces selector-by-selector coverage with a global motion contract.
- **Source classification summary:** Strengthened the global reduced-motion contract and simplified the mobile header effect.
- **Source classification reason:** Significant because it restores or verifies an accessibility invariant across designs or routes, where a local presentation defect would otherwise become a site-wide failure.

#### Source에서 확정된 구현 의도와 상태 변화

Reduced-motion에서 모든 animation/transition을 near-zero single iteration으로 축소하고 smooth scrolling을 끄는 global contract로 강화합니다. Terminal wrapper와 hover card를 포괄하고 mobile header backdrop filter도 제거합니다.

#### 해당 SHA에서 확인할 실제 코드

- Global selector scope가 renderer/component subtree에 적용되는지 확인합니다.
- Animation duration, iteration count, transition duration, scroll behavior override를 기록합니다.
- Terminal wrapper와 hover transform이 general/explicit rule로 처리되는지 확인합니다.
- Specificity 또는 `!important`가 design-scoped CSS를 이기는지 확인합니다.
- Mobile breakpoint의 backdrop filter 제거를 별도 변경으로 구분합니다.
- 새 animation을 가정해 local list 방식과 global 방식 차이를 설명합니다.

확인 원칙:

- 먼저 `af9191fc15ad^`와 `af9191fc15ad`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 local policy와 global policy 차이 | reduced-motion coverage가 known selector list에 의존해 새 animation/transition을 놓칠 수 있었습니다. |
| Near-zero 처리와 `none` 처리의 실제 선택 | media query 안에서 global elements/pseudo-elements를 near-zero duration, single iteration, auto scroll로 강제했습니다. |
| Smooth scroll, hover, terminal coverage | `src/app/globals.css`의 `*, *::before, *::after`에 animation-duration/transition-duration `.01ms !important`, animation-iteration-count `1 !important`, scroll-behavior `auto !important`를 적용하고 terminal wrapper/`.motion-card:hover` transform도 제거합니다. <=640px header backdrop-filter 제거는 별도 mobile effect simplification입니다. |
| Reduced motion에서도 유지되는 functionality | reduced-motion preference에서 animation/transition/smooth scroll이 기능 전제가 되지 않습니다. |
| Global rule의 potential side effect와 trade-off | 브라우저/OS별 실제 사용자 경험과 custom script-driven motion 전체는 이 CSS만으로 증명하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** reduced-motion coverage가 known selector list에 의존해 새 animation/transition을 놓칠 수 있었습니다.
- **해당 SHA 핵심 코드:** `src/app/globals.css`의 `*, *::before, *::after`에 animation-duration/transition-duration `.01ms !important`, animation-iteration-count `1 !important`, scroll-behavior `auto !important`를 적용하고 terminal wrapper/`.motion-card:hover` transform도 제거합니다. <=640px header backdrop-filter 제거는 별도 mobile effect simplification입니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `84c71d027630`의 browser matrix는 구조/keyboard/Axe를 검사하지만 motion preference visual timing 자체는 별도 측정하지 않습니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** reduced-motion coverage가 known selector list에 의존해 새 animation/transition을 놓칠 수 있었습니다. hover, terminal wrapper, smooth scroll 등 다른 motion source가 local overrides를 우회할 수 있었습니다.
- **구현 결정과 경로:** media query 안에서 global elements/pseudo-elements를 near-zero duration, single iteration, auto scroll로 강제했습니다. `src/app/globals.css`의 `*, *::before, *::after`에 animation-duration/transition-duration `.01ms !important`, animation-iteration-count `1 !important`, scroll-behavior `auto !important`를 적용하고 terminal wrapper/`.motion-card:hover` transform도 제거합니다. <=640px header backdrop-filter 제거는 별도 mobile effect simplification입니다.
- **소유권·실패 처리:** global policy가 future CSS motion baseline을 소유하고 design-scoped rules는 `!important`에 의해 override됩니다. near-zero는 lifecycle event를 완전히 없애지 않으면서 체감 motion을 제거하지만 broad rule이 의도된 micro-transition도 축소하는 trade-off가 있습니다.
- **보장:** reduced-motion preference에서 animation/transition/smooth scroll이 기능 전제가 되지 않습니다.
- **보장하지 않는 범위:** 브라우저/OS별 실제 사용자 경험과 custom script-driven motion 전체는 이 CSS만으로 증명하지 않습니다.
- **후속 연결:** `84c71d027630`의 browser matrix는 구조/keyboard/Axe를 검사하지만 motion preference visual timing 자체는 별도 측정하지 않습니다.

### 3. `5a6fd8a802ff` — fix(a11y): 디자인별 색상 대비 보정

- **Importance:** A
- **Tags:** A11Y, DEBUG
- **Source-defined thread role:** Corrects shared and Editorial contrast tokens.
- **Source classification summary:** Adjust shared and Editorial color tokens so accent text remains distinguishable on both light and dark surfaces.
- **Source classification reason:** Significant because it restores or verifies an accessibility invariant across designs or routes, where a local presentation defect would otherwise become a site-wide failure.

#### Source에서 확정된 구현 의도와 상태 변화

Shared와 Editorial color token을 조정해 accent text가 light/dark surface 모두에서 구분되도록 하고 dark-surface vermilion을 normal text token과 분리합니다.

#### 해당 SHA에서 확인할 실제 코드

- Fix 전 하나의 accent token이 적용되던 light/dark selector를 찾습니다.
- 새 token definition과 surface-specific consumer 변경을 비교합니다.
- Evidence, architecture, curation, gap label 적용 영역을 확인합니다.
- 한 context 개선이 다른 context를 악화시키던 기존 color/background 조합을 기록합니다.
- Axe/contrast 검사에서 해당 route/design fixture를 찾습니다.

확인 원칙:

- 먼저 `5a6fd8a802ff^`와 `5a6fd8a802ff`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 기존 가정: shared accent 하나로 모든 surface 대응 | 같은 accent literal/token을 light와 dark surfaces에 재사용했습니다. |
| 실제 risk: dark/light 중 한쪽 대비 부족 | root cause는 role이 다른 surfaces에 literal identity 하나를 강제한 것이며 수정은 context-specific value로 분리합니다. |
| Root cause: semantic role보다 literal visual identity 공유 | 한 surface에서 충분한 색이 다른 background에서는 contrast를 잃어 visual identity와 semantic readability가 충돌했습니다. |
| 수정 decision: surface-specific token | shared accent를 조정하고 Editorial에 dark-surface 전용 vermilion token을 분리했습니다. |
| Regression evidence와 아직 보장하지 않는 color combination | 모든 사용자 content/background 조합과 human contrast perception을 정적 token 변경만으로 보장하지 않습니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** 같은 accent literal/token을 light와 dark surfaces에 재사용했습니다.
- **해당 SHA 핵심 코드:** `src/app/globals.css`의 accent가 `#008c89`에서 `#007b78`로 조정되고 dark chip text는 `#f2efe7`을 사용합니다. Editorial stylesheet는 normal vermilion을 `#c7432e`, `--vermilion-on-dark: #e15b43`으로 나눠 evidence/architecture/curation/gap dark contexts에 적용합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `84c71d027630`의 Axe matrix가 configured route/design fixture에서 contrast rule 회귀를 탐지합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** 같은 accent literal/token을 light와 dark surfaces에 재사용했습니다. 한 surface에서 충분한 색이 다른 background에서는 contrast를 잃어 visual identity와 semantic readability가 충돌했습니다.
- **구현 결정과 경로:** shared accent를 조정하고 Editorial에 dark-surface 전용 vermilion token을 분리했습니다. `src/app/globals.css`의 accent가 `#008c89`에서 `#007b78`로 조정되고 dark chip text는 `#f2efe7`을 사용합니다. Editorial stylesheet는 normal vermilion을 `#c7432e`, `--vermilion-on-dark: #e15b43`으로 나눠 evidence/architecture/curation/gap dark contexts에 적용합니다.
- **소유권·실패 처리:** token layer가 semantic surface role을 소유하고 consumer selectors가 light/dark context에 맞는 token을 선택합니다. root cause는 role이 다른 surfaces에 literal identity 하나를 강제한 것이며 수정은 context-specific value로 분리합니다.
- **보장:** 수정된 known light/dark consumers에서 accent text 구분 가능성을 강화합니다.
- **보장하지 않는 범위:** 모든 사용자 content/background 조합과 human contrast perception을 정적 token 변경만으로 보장하지 않습니다.
- **후속 연결:** `84c71d027630`의 Axe matrix가 configured route/design fixture에서 contrast rule 회귀를 탐지합니다.

### 4. `a15e117cb51b` — fix(a11y): skip link focus target 복원

- **Importance:** A
- **Tags:** A11Y, DEBUG
- **Source-defined thread role:** Restores programmatic focus transfer in shared shells.
- **Source classification summary:** Make each main-content landmark programmatically focusable with `tabIndex={-1}`.
- **Source classification reason:** Significant because it restores or verifies an accessibility invariant across designs or routes, where a local presentation defect would otherwise become a site-wide failure.

#### Source에서 확정된 구현 의도와 상태 변화

Shared, Cinematic, Editorial shell의 main landmark에 `tabIndex={-1}`을 추가해 skip link가 viewport 이동뿐 아니라 keyboard focus까지 전달하도록 복원합니다.

#### 해당 SHA에서 확인할 실제 코드

- 각 shell의 skip link `href`와 main `id`가 일치하는지 확인합니다.
- Fix 전 main이 programmatic focus target이 될 수 없었던 markup을 비교합니다.
- `tabIndex={-1}`이 normal tab order에는 없지만 focus를 허용하는 browser behavior를 확인합니다.
- Shared, Cinematic, Editorial의 동일 수정 지점을 찾습니다.
- Brutalist gap은 다음 commit과 연결합니다.

확인 원칙:

- 먼저 `a15e117cb51b^`와 `a15e117cb51b`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 기존 가정: fragment navigation으로 keyboard focus도 이동 | skip link `href`가 main fragment로 scroll했지만 대상 `<main>`이 programmatic focusable하지 않았습니다. |
| 실제 failure: visual scroll과 focus position 불일치 | `-1`은 normal sequential tab order에 새 stop을 만들지 않으면서 fragment/explicit focus를 허용합니다. |
| 수정 code와 shell coverage | 각 shell의 skip link `href`와 main `id` (`main-content`, `cinematic-content`, `editorial-main`)가 일치하고 main에 `tabIndex={-1}`이 추가됩니다. |
| Normal tab order에 새 stop을 추가하지 않는 이유 | Shared, Cinematic, Editorial main landmarks에 `tabIndex={-1}`을 추가했습니다. |
| 다음 commit/test와 연결 | `e1aac08e0e9e`이 Brutalist main focusability를 추가하고 `84c71d027630`이 keyboard path를 검사합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** skip link `href`가 main fragment로 scroll했지만 대상 `<main>`이 programmatic focusable하지 않았습니다.
- **해당 SHA 핵심 코드:** 각 shell의 skip link `href`와 main `id` (`main-content`, `cinematic-content`, `editorial-main`)가 일치하고 main에 `tabIndex={-1}`이 추가됩니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `e1aac08e0e9e`이 Brutalist main focusability를 추가하고 `84c71d027630`이 keyboard path를 검사합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** skip link `href`가 main fragment로 scroll했지만 대상 `<main>`이 programmatic focusable하지 않았습니다. visual viewport는 이동해도 keyboard focus는 header/skip link에 남아 다음 Tab 순서가 content 시작과 어긋날 수 있었습니다.
- **구현 결정과 경로:** Shared, Cinematic, Editorial main landmarks에 `tabIndex={-1}`을 추가했습니다. 각 shell의 skip link `href`와 main `id` (`main-content`, `cinematic-content`, `editorial-main`)가 일치하고 main에 `tabIndex={-1}`이 추가됩니다.
- **소유권·실패 처리:** anchor가 navigation trigger를, main landmark가 programmatic focus target을 소유합니다. `-1`은 normal sequential tab order에 새 stop을 만들지 않으면서 fragment/explicit focus를 허용합니다.
- **보장:** 세 shell에서 skip-link activation이 viewport 이동뿐 아니라 main focus transfer로 이어질 기반을 복원합니다.
- **보장하지 않는 범위:** Brutalist shell은 이 SHA에서 누락돼 cross-design complete invariant는 아직 아닙니다.
- **후속 연결:** `e1aac08e0e9e`이 Brutalist main focusability를 추가하고 `84c71d027630`이 keyboard path를 검사합니다.

### 5. `e1aac08e0e9e` — fix(a11y): Brutalist 지표의 definition semantics 수정

- **Importance:** A
- **Tags:** ARCH, RENDERER, A11Y
- **Source-defined thread role:** Repairs definition-list semantics and the remaining shell focus boundary.
- **Source classification summary:** Correct the Brutalist metric block so every descriptive value is represented by a `<dd>` inside its definition list instead of an unrelated paragraph.
- **Source classification reason:** Significant because it restores or verifies an accessibility invariant across designs or routes, where a local presentation defect would otherwise become a site-wide failure.

#### Source에서 확정된 구현 의도와 상태 변화

Brutalist metric `<dl>`에서 descriptive value를 paragraph 대신 `<dd>`로 수정하고 CSS selector를 맞춥니다. Brutalist main landmark도 focusable하게 만들어 남은 skip-link gap을 닫습니다.

#### 해당 SHA에서 확인할 실제 코드

- Metric block의 `<dl>`, `<dt>`, value, description markup을 수정 전후로 비교합니다.
- Paragraph에서 `<dd>`로 바뀌며 accessible relationship이 어떻게 달라지는지 확인합니다.
- Description이 strong value typography를 상속하지 않도록 CSS selector가 분리되는지 확인합니다.
- Brutalist main의 `tabIndex={-1}`과 skip link target을 확인합니다.
- Visual output과 accessibility tree 차이를 기록합니다.

확인 원칙:

- 먼저 `e1aac08e0e9e^`와 `e1aac08e0e9e`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 기존 malformed definition-list assumption | Brutalist metric `<dl>` 안에서 description이 `<p>`로 렌더링됐고 Brutalist main은 focus target이 아니었습니다. |
| Root cause와 semantic correction | visual layout은 보이지만 definition relationship이 accessibility tree에서 끊기고 skip link focus gap이 한 design에 남았습니다. |
| CSS adaptation | description을 두 번째 `<dd>`로 바꾸고 CSS selector를 분리하며 Brutalist main에 `tabIndex={-1}`을 추가했습니다. Brutalist metric component의 `<dt>` 뒤 value `<dd>`와 description `<dd className="metricDescription">`가 semantic pair를 이루고 stylesheet가 strong value style과 description style을 구분합니다. Main landmark는 matching id와 `tabIndex={-1}`을 가집니다. |
| 남은 shell focus boundary 복구 | HTML definition list가 term/value relationship을 소유하고 CSS는 visual distinction만 소유합니다. |
| 다음 broad matrix와 연결 | `84c71d027630`이 semantic/focus fixes를 모든 enabled route×design integration에서 검사합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** Brutalist metric `<dl>` 안에서 description이 `<p>`로 렌더링됐고 Brutalist main은 focus target이 아니었습니다.
- **해당 SHA 핵심 코드:** Brutalist metric component의 `<dt>` 뒤 value `<dd>`와 description `<dd className="metricDescription">`가 semantic pair를 이루고 stylesheet가 strong value style과 description style을 구분합니다. Main landmark는 matching id와 `tabIndex={-1}`을 가집니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** `84c71d027630`이 semantic/focus fixes를 모든 enabled route×design integration에서 검사합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** Brutalist metric `<dl>` 안에서 description이 `<p>`로 렌더링됐고 Brutalist main은 focus target이 아니었습니다. visual layout은 보이지만 definition relationship이 accessibility tree에서 끊기고 skip link focus gap이 한 design에 남았습니다.
- **구현 결정과 경로:** description을 두 번째 `<dd>`로 바꾸고 CSS selector를 분리하며 Brutalist main에 `tabIndex={-1}`을 추가했습니다. Brutalist metric component의 `<dt>` 뒤 value `<dd>`와 description `<dd className="metricDescription">`가 semantic pair를 이루고 stylesheet가 strong value style과 description style을 구분합니다. Main landmark는 matching id와 `tabIndex={-1}`을 가집니다.
- **소유권·실패 처리:** HTML definition list가 term/value relationship을 소유하고 CSS는 visual distinction만 소유합니다. malformed child element를 semantic `<dd>`로 교체해 visual output을 유지하면서 accessibility tree를 수정합니다.
- **보장:** definition values가 `<dd>`이고 남은 shell skip target도 programmatic focusable합니다.
- **보장하지 않는 범위:** route별 landmark cardinality와 full design matrix는 아직 broad test로 고정되지 않았습니다.
- **후속 연결:** `84c71d027630`이 semantic/focus fixes를 모든 enabled route×design integration에서 검사합니다.

### 6. `84c71d027630` — test(a11y): 디자인×route WCAG 행렬 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, A11Y
- **Source-defined thread role:** Verifies every enabled route under every design and a keyboard skip path.
- **Source classification summary:** Add an end-to-end accessibility matrix that exercises every enabled route under all five designs.
- **Source classification reason:** Significant because it restores or verifies an accessibility invariant across designs or routes, where a local presentation defect would otherwise become a site-wide failure.

#### Source에서 확정된 구현 의도와 상태 변화

모든 enabled route를 five designs에서 실행하는 browser-level accessibility matrix를 추가합니다. Response, selected design root, banner/main/content-info 각 하나, Axe WCAG A/AA clean scan과 skip-link focus transfer를 검증합니다.

#### 해당 SHA에서 확인할 실제 코드

- Enabled route fixture와 five-design fixture가 기존 matrix와 공유되는지 확인합니다.
- Route×design case 생성과 dynamic project route 선택을 확인합니다.
- Response status, design root, landmark cardinality assertion을 추적합니다.
- Axe integration의 included/excluded rules와 WCAG 2.x A/AA tags를 확인합니다.
- Keyboard test에서 first Tab, skip activation, main focus 순서를 확인합니다.
- Failure report가 route/design/rule을 식별하는지 확인합니다.
- 이전 fixes를 제거했을 때 어떤 case가 실패할지 연결합니다.

확인 원칙:

- 먼저 `84c71d027630^`와 `84c71d027630`를 비교하고, thread 흐름이 필요한 경우에만 이전 thread commit과 추가 비교합니다.
- Final HEAD의 함수명, file layout, test 결과를 이 commit에 소급하지 않습니다.
- 실제 file path, symbol, caller/callee, state mutation 순서, failure branch를 확인한 뒤 기록합니다.

#### 학습자가 남길 증거

| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 대상 production invariants | configured WCAG scan의 violation 부재, single landmark cardinality와 keyboard skip focus contract를 모든 enabled route×5 designs에 적용합니다. |
| Test matrix의 route/design 범위 | `tests/e2e/site-matrix.ts`는 content에서 첫 enabled project와 enabled optional pages를 포함한 8 route definitions를 만들고 5 design IDs를 공유합니다. `tests/e2e/accessibility.spec.ts`는 각 design×enabled route에서 response OK, selected design root visible, banner/main/contentinfo 각 1개, Axe tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`의 zero violations를 검사합니다. 별도 case는 home에서 Tab→skip link label→Enter→main focus를 확인합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| Real browser, Axe, landmark count, keyboard path technique | `tests/e2e/site-matrix.ts`는 content에서 첫 enabled project와 enabled optional pages를 포함한 8 route definitions를 만들고 5 design IDs를 공유합니다. `tests/e2e/accessibility.spec.ts`는 각 design×enabled route에서 response OK, selected design root visible, banner/main/contentinfo 각 1개, Axe tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`의 zero violations를 검사합니다. 별도 case는 home에서 Tab→skip link label→Enter→main focus를 확인합니다. 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다. |
| 통과하는 production route/shell/renderer path | `tests/e2e/site-matrix.ts`는 content에서 첫 enabled project와 enabled optional pages를 포함한 8 route definitions를 만들고 5 design IDs를 공유합니다. `tests/e2e/accessibility.spec.ts`는 각 design×enabled route에서 response OK, selected design root visible, banner/main/contentinfo 각 1개, Axe tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`의 zero violations를 검사합니다. 별도 case는 home에서 Tab→skip link label→Enter→main focus를 확인합니다. |
| 증명하는 것: configured violations 부재와 structural/focus contract | configured WCAG scan의 violation 부재, single landmark cardinality와 keyboard skip focus contract를 모든 enabled route×5 designs에 적용합니다. |
| 증명하지 않는 것: 모든 assistive technology와 human evaluation | 모든 assistive technology, screen-reader UX, human review, reduced-motion timing과 WCAG 전체 적합성을 증명하지 않습니다. |
| Broad integration regression으로서 막는 범위 | `tests/e2e/site-matrix.ts`는 content에서 첫 enabled project와 enabled optional pages를 포함한 8 route definitions를 만들고 5 design IDs를 공유합니다. `tests/e2e/accessibility.spec.ts`는 각 design×enabled route에서 response OK, selected design root visible, banner/main/contentinfo 각 1개, Axe tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`의 zero violations를 검사합니다. 별도 case는 home에서 Tab→skip link label→Enter→main focus를 확인합니다. |

#### 코드 발췌 기록

- **변경 전 대응 코드:** motion/contrast/focus/semantics fixes가 개별 source 변경과 일부 local checks에 의존했습니다.
- **해당 SHA 핵심 코드:** `tests/e2e/site-matrix.ts`는 content에서 첫 enabled project와 enabled optional pages를 포함한 8 route definitions를 만들고 5 design IDs를 공유합니다. `tests/e2e/accessibility.spec.ts`는 각 design×enabled route에서 response OK, selected design root visible, banner/main/contentinfo 각 1개, Axe tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`의 zero violations를 검사합니다. 별도 case는 home에서 Tab→skip link label→Enter→main focus를 확인합니다.
- **실행·테스트 증거:** 실행하지 못했습니다. 로컬 Git clone 단계에서 실행 환경의 GitHub DNS 해석이 차단되어 build/test/browser/Docker command를 수행할 수 없었습니다. 문서의 구현 설명은 해당 SHA의 diff와 파일을 GitHub connector로 정적 검토한 결과이며 실행 성공으로 표시하지 않습니다.
- **다음 commit 연결:** 이 commit이 이전 local fixes를 broad production-browser integration regression으로 연결합니다.

#### SHA별 복원 결론

- **이전 상태와 문제:** motion/contrast/focus/semantics fixes가 개별 source 변경과 일부 local checks에 의존했습니다. 다섯 renderer와 optional/dynamic routes 중 하나가 banner/main/footer, contrast 또는 keyboard skip behavior를 다시 깨뜨릴 수 있었습니다.
- **구현 결정과 경로:** shared route fixture와 5-design browser accessibility matrix를 추가했습니다. `tests/e2e/site-matrix.ts`는 content에서 첫 enabled project와 enabled optional pages를 포함한 8 route definitions를 만들고 5 design IDs를 공유합니다. `tests/e2e/accessibility.spec.ts`는 각 design×enabled route에서 response OK, selected design root visible, banner/main/contentinfo 각 1개, Axe tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`, `wcag22aa`의 zero violations를 검사합니다. 별도 case는 home에서 Tab→skip link label→Enter→main focus를 확인합니다.
- **소유권·실패 처리:** content/config fixture가 matrix coverage를, Playwright page가 real browser route path를, Axe/landmark/keyboard assertions가 regression result를 소유합니다. dynamic project는 첫 enabled ID를 사용하며 violation formatter가 route/design/rule context를 남깁니다.
- **보장:** configured WCAG scan의 violation 부재, single landmark cardinality와 keyboard skip focus contract를 모든 enabled route×5 designs에 적용합니다.
- **보장하지 않는 범위:** 모든 assistive technology, screen-reader UX, human review, reduced-motion timing과 WCAG 전체 적합성을 증명하지 않습니다.
- **후속 연결:** 이 commit이 이전 local fixes를 broad production-browser integration regression으로 연결합니다.

## 6. Invariant ledger

| Invariant | Source에서 확인된 변화 지점 | 해당 SHA의 실제 code/test evidence | 부족함이 드러난 시점 | 최종 보장 범위 |
| --- | --- | --- | --- | --- |
| Reduced-motion에서는 nonessential animation, transition, smooth scroll이 기능 전제가 아닙니다. | `29bb40579cb2 → af9191fc15ad` | `29bb40579cb2` local selector additions과 `af9191fc15ad` global `*/*::before/*::after` near-zero duration, single iteration, auto scroll. | selector allow-list는 future/new motion source를 자동 포함하지 못했습니다. | reduced-motion preference에서 known transforms와 future CSS animation/transition/smooth scroll이 core functionality의 전제가 되지 않습니다. |
| Accent text는 light와 dark surface 모두에서 구분 가능한 token을 사용합니다. | `5a6fd8a802ff` | shared accent `#007b78`, dark chip text, Editorial normal/dark-surface vermilion tokens와 consumers. | 하나의 literal accent를 light/dark surfaces에 공유해 한쪽 contrast가 부족했습니다. | known light/dark contexts가 surface-specific tokens를 사용하고 Axe matrix가 configured fixtures의 contrast regression을 검사합니다. |
| Skip link는 main landmark로 programmatic focus를 이동합니다. | `a15e117cb51b → e1aac08e0e9e` | Shared/Cinematic/Editorial main `tabIndex={-1}`, Brutalist follow-up와 keyboard Tab→skip→Enter→main assertion. | fragment scroll만으로 focus가 이동한다는 가정과 Brutalist 누락이 있었습니다. | 모든 five-design shells의 main은 normal tab order를 늘리지 않으면서 programmatic focus target이 됩니다. |
| Definition-list 값은 `<dd>`이고 landmarks는 route마다 유일합니다. | `e1aac08e0e9e → 84c71d027630` | Brutalist metric의 second `<dd className="metricDescription">`; matrix의 banner/main/contentinfo exact-one assertions. | `<dl>` 안 paragraph와 untested landmark duplicates가 semantic structure를 깰 수 있었습니다. | definition values는 `<dd>`로 연결되고 enabled route마다 three landmark categories가 하나씩 있어야 합니다. |
| 모든 enabled route와 five designs는 configured WCAG A/AA scan과 keyboard skip path를 통과합니다. | `84c71d027630` | `tests/e2e/site-matrix.ts` + `accessibility.spec.ts`: enabled route×5 design, Axe WCAG tags, zero violations와 keyboard skip test. | 개별 fix만으로 cross-design/route regression을 알 수 없었습니다. | configured browser scan과 structural/focus contract를 full enabled product matrix에 적용합니다. |

Ledger 작성 원칙:

- Source가 명시한 invariant만 사용하고 새 invariant를 확정 사실처럼 추가하지 않습니다.
- 도입, 강화, 부족함 노출, fix, regression test가 서로 다른 commit이면 각 열에 분리해 기록합니다.
- Code evidence에는 실제 field, function, branch, selector, command 또는 assertion을 적습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 위험 | 대응 commit | 실제 수정/강화 code에서 확인할 것 | Test 또는 실행 증거 |
| --- | --- | --- | --- |
| 반복 list/timeline animation이 reduced-motion override 밖에 남음 | 29bb40579cb2 | Chip/guide transform과 transition을 기존 override에 포함합니다. | Future selector를 자동 포함하지 못하는 한계를 기록합니다. |
| 새 animation이 selector list를 우회함 | af9191fc15ad | Global near-zero duration/iteration과 smooth-scroll off로 강화합니다. | Terminal wrapper와 hover card 적용을 확인합니다. |
| Shared accent가 한 surface에서는 충분하지만 다른 surface에서는 대비 부족 | 5a6fd8a802ff | Normal/dark-surface token 분리와 consumer selector를 확인합니다. | 어느 route/design에서 contrast failure가 재현되는지 기록합니다. |
| Skip link가 scroll만 하고 focus는 header에 남음 | a15e117cb51b | Main에 `tabIndex={-1}`을 추가해 programmatic focusable하게 만듭니다. | Shared/Cinematic/Editorial shell을 확인합니다. |
| Brutalist definition semantics와 focus gap | e1aac08e0e9e | Paragraph를 `<dd>`로 바꾸고 CSS와 main focusability를 보정합니다. | 다음 matrix와 연결합니다. |
| Cross-design regression | 84c71d027630 | Enabled route×5 designs에 response, root, landmark, Axe, skip focus를 적용합니다. | Broad matrix가 증명하지 않는 screen-reader별 경험을 구분합니다. |

### 실제 연결 기록

- `29bb40579cb2`은 known repeated selectors를 추가한 보완이고 `af9191fc15ad`은 future CSS motion까지 포괄하는 global policy입니다.
- `a15e117cb51b`은 Shared/Cinematic/Editorial만 수정했고 Brutalist gap은 `e1aac08e0e9e`에서 닫혔습니다.
- `84c71d027630`은 real-browser Axe/landmark/keyboard integration을 정의하지만 human evaluation이나 모든 assistive technology의 적합성 증명은 아닙니다.

## 8. Ownership / state / responsibility 변화

| Concern | Thread 초기 owner/state | Thread 최종 owner/state | 실제 symbol과 호출 경로 |
| --- | --- | --- | --- |
| Motion coverage | 개별 selector | Global reduced-motion contract | `src/app/globals.css` global `prefers-reduced-motion` block이 CSS motion baseline을 소유합니다. |
| Contrast | 하나의 shared accent token | Surface-specific tokens | shared/Editorial token definitions과 surface consumer selectors가 contrast 역할을 소유합니다. |
| Skip navigation | Anchor scroll behavior | Focusable main + keyboard assertion | 각 design shell의 skip link href + matching focusable main; Playwright keyboard assertion이 integration proof를 소유합니다. |
| Metric semantics | Paragraph inside `<dl>` | Semantic `<dd>` | Brutalist metric component의 `<dt>/<dd>/<dd>` markup과 description CSS selector. |
| Regression proof | 개별 수동 확인 | Route×design browser matrix | `tests/e2e/site-matrix.ts` route/design fixtures → Playwright navigation → Axe/landmark/skip-focus assertions. |

## 9. Thread 최종 상태

### Source에서 확정된 최종 상태

국소 reduced-motion 처리에서 시작해 global motion policy, contrast, skip-link focus, semantic definition list를 수정하고 다섯 design과 모든 enabled route를 browser-level WCAG matrix로 검증하는 과정을 복원합니다.

### 학습자가 완성할 최종 설명

- **Thread 시작 시점의 설계와 위험:** 초기 reduced-motion support는 일부 reveal/card selector만 다뤄 repeated list/timeline과 future animation을 놓칠 수 있었습니다.
- **핵심 architecture/decision이 형성된 순서:** local motion gap 보완 → global policy → surface-specific contrast → shared shell focus → Brutalist semantics/focus → full browser matrix 순서로 cross-design invariant가 형성됐습니다.
- **실제 failure 또는 부족함이 드러난 지점:** 같은 accent token의 surface mismatch, fragment scroll과 keyboard focus 불일치, visually acceptable paragraph inside `<dl>`이 실제 부족함으로 확인됐습니다.
- **Fix 또는 boundary 강화가 바꾼 invariant:** CSS token/policy와 semantic markup/focus target을 수정하고 five-design enabled-route matrix에서 Axe, landmarks와 keyboard path를 함께 검사하도록 강화했습니다.
- **Test/build/browser evidence가 보장한 범위:** commit diffs와 test source의 exact fixtures/assertions를 정적으로 확인했습니다. Playwright/Axe browser run은 실행하지 않았으므로 “통과”라는 runtime claim은 기록하지 않았습니다.
- **Thread 종료 시점에도 보장하지 않는 범위:** 모든 assistive technology, screen-reader experience, human WCAG review, motion timing perception와 arbitrary user content 조합은 보장하지 않습니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 source-backed flow의 각 단계에 실제 file path, symbol, input/output, failure branch를 추가합니다.

1. User preference가 `prefers-reduced-motion` media query에 전달됩니다.
   - 실제 코드 위치: `src/app/globals.css @media (prefers-reduced-motion: reduce)`
   - 입력과 출력: OS/browser preference가 matching media query를 활성화합니다.
   - 실패/absence 처리: preference가 없으면 normal design motion rules가 유지됩니다.
2. Global policy가 animation/transition/smooth scroll을 기능과 분리합니다.
   - 실제 코드 위치: same global block + explicit transform rules
   - 입력과 출력: animation/transition duration을 `.01ms`, iteration을 1, scroll을 auto로 바꾸고 known transforms를 none으로 만듭니다.
   - 실패/absence 처리: script-driven motion이나 외부 styles가 scope 밖이면 이 CSS만으로 제어하지 않습니다.
3. Design-scoped token이 각 surface에서 contrast-safe 값을 제공합니다.
   - 실제 코드 위치: shared and design-scoped CSS variables/selectors
   - 입력과 출력: surface context가 normal 또는 dark-safe accent token을 선택해 text/background 조합을 출력합니다.
   - 실패/absence 처리: 새 consumer가 잘못된 token을 쓰면 Axe matrix의 covered route에서 contrast violation으로 드러날 수 있습니다.
4. Skip link가 first tab stop으로 나타나 main target에 focus를 이동합니다.
   - 실제 코드 위치: 각 shell의 skip link와 `<main id=... tabIndex={-1}>`
   - 입력과 출력: 첫 Tab에 skip link가 나타나고 Enter fragment activation이 focusable main으로 이동합니다.
   - 실패/absence 처리: id mismatch나 missing tabindex이면 Playwright focus assertion이 실패합니다.
5. 각 renderer가 banner/main/content-info와 semantic content structure를 출력합니다.
   - 실제 코드 위치: five route renderers/shells 및 Brutalist metric markup
   - 입력과 출력: 각 page가 banner/main/contentinfo와 semantic `<dl>/<dt>/<dd>` content를 출력합니다.
   - 실패/absence 처리: duplicate/missing landmarks 또는 invalid semantic relations는 structural/Axe checks 대상입니다.
6. Playwright matrix가 route×design을 열고 Axe와 landmark cardinality를 검사합니다.
   - 실제 코드 위치: `tests/e2e/accessibility.spec.ts` route×design loop
   - 입력과 출력: shared fixture의 enabled routes를 five explicit design query로 열고 response/root/landmarks/Axe를 검사합니다.
   - 실패/absence 처리: HTTP failure, wrong renderer root, landmark count 또는 configured Axe violation이 route/design context와 함께 실패합니다.
7. 별도 keyboard path가 skip-link focus transfer를 실제 browser에서 검증합니다.
   - 실제 코드 위치: same spec의 keyboard test
   - 입력과 출력: 각 design home에서 Tab, skip-link accessible name, Enter, main focus 순서를 real browser API로 확인합니다.
   - 실패/absence 처리: 이 path는 broader screen-reader navigation이나 all keyboard controls를 평가하지 않습니다.

### 코드 없이 설명하기

> 이 Thread의 최종 실행 흐름을 code snippet 없이 자신의 말로 작성합니다. 설계 → 구현 → failure/risk → 수정/강화 → 검증 순서가 드러나야 합니다.

처음에는 reduced-motion 대상 selector를 하나씩 늘렸지만 새 animation을 놓칠 수 있어 global near-zero duration, single iteration과 smooth-scroll off 정책으로 바꿨습니다. 이후 light/dark surface의 accent token을 분리하고 skip link target main을 programmatic focusable하게 만들었습니다. Brutalist의 잘못된 definition-list child를 `<dd>`로 고치고 남은 focus gap도 닫았습니다. 마지막 browser matrix는 content에서 enabled routes를 만들고 다섯 design 각각에서 response, design root, single landmarks, configured Axe rules와 실제 keyboard skip focus를 검사하도록 구현됐습니다.

> **실행 증거 구분:** 참조 SHA의 diff와 해당 SHA 파일은 GitHub connector로 확인했습니다. 로컬 clone은 실행 환경의 DNS 차단으로 실패해 build, unit/E2E, browser, Lighthouse, Docker command는 실행하지 않았습니다. 따라서 위 test 결과는 구현된 test technique과 assertion의 정적 검토이며 실제 통과 결과가 아닙니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 실제로 checkout하거나 diff로 확인했습니다.
- [x] Thread 내 commit 순서를 source와 동일하게 유지했습니다.
- [x] 각 A/S commit에서 직전 상태, decision, failure boundary, guarantee와 non-guarantee를 구분했습니다.
- [x] B commit은 thread 흐름에서 필요한 구현 역할과 state change를 확인했습니다.
- [x] Fix commit을 독립 feature가 아니라 기존 가정 → failure → root cause → corrected invariant로 설명했습니다.
- [x] Test commit에서 production invariant, failure injection, technique, traversed production path, proves/does-not-prove를 구분했습니다.
- [x] Invariant ledger의 모든 주장에 해당 SHA의 code/test evidence가 있습니다.
- [x] Final HEAD의 code를 과거 commit 설명에 소급하지 않았습니다.
- [x] Thread 최종 흐름을 code 없이 설명할 수 있습니다.
===== END FILE: 07-accessibility-policy-to-cross-design-regression-evidence.md =====

===== BEGIN FILE: README.md =====
# portfolio-project Development Thread 학습 골격

## 목적

이 문서 세트는 42 Archive Portfolio (`web/portfolio`)의 실제 commit history와 각 SHA의 코드를 직접 읽으며 설계, 구현, 실패 처리, 수정, 검증의 발전 과정을 복원하기 위한 학습 골격입니다.

완성형 프로젝트 해설서가 아닙니다. Source에서 확정된 thread 구조, commit metadata, 역할, 중요도, invariant와 engineering difficulty만 미리 제공하며 실제 코드 해석과 실행 결과는 학습자가 채웁니다.

## 권장 학습 순서

1. [Fail-closed content ingestion](01-fail-closed-content-ingestion.md)
2. [Full-site renderer architecture](02-full-site-renderer-architecture.md)
3. [Route projections and renderer data ownership](03-route-projections-and-renderer-data-ownership.md)
4. [Template preview to production publication](04-template-preview-to-production-publication.md)
5. [Native design switcher and server-first interaction](05-native-design-switcher-and-server-first-interaction.md)
6. [Production artifact and performance enforcement](06-production-artifact-and-performance-enforcement.md)
7. [Accessibility policy to cross-design regression evidence](07-accessibility-policy-to-cross-design-regression-evidence.md)

Source의 Development Thread 순서를 그대로 따릅니다. 동일 commit이 여러 thread에 등장하는 경우 중복을 제거하지 않고 각 관점에서 다시 확인합니다.

## Thread 문서 사용법

1. Thread 목표, 핵심 질문, 완료 기준을 먼저 읽습니다.
2. Commit map 순서대로 각 SHA를 checkout합니다.
3. 해당 commit의 first-parent diff와 resulting code를 확인합니다.
4. 문서가 지정한 symbol, caller/callee, state, reference, failure branch, cleanup, command, test path를 찾아 기록합니다.
5. 필요한 경우에만 직전 관련 thread SHA와 비교합니다.
6. Invariant ledger와 Failure → Fix → Test 연결을 commit evidence로 채웁니다.
7. 마지막에 code 없이 thread의 최종 architecture 또는 execution flow를 설명합니다.

## 해당 SHA 코드 확인 원칙

- 모든 판단은 현재 학습 중인 SHA의 tree와 그 commit의 diff를 기준으로 합니다.
- File path, function/type/component 이름, caller/callee, state mutation 순서, error branch를 구체적으로 기록합니다.
- Source에서 명시하지 않은 architecture나 invariant를 추측해 확정 사실로 추가하지 않습니다.
- Commit subject나 body만 옮기지 말고 그 역할을 actual code evidence와 연결합니다.
- Generated lockfile, snapshot, measurement는 evidence일 수 있지만 decision 자체와 구분합니다.

## Final HEAD 소급 사용 금지

- Final HEAD의 file layout, function, test, behavior를 과거 commit에 소급하지 않습니다.
- 현재 SHA에 없는 helper나 fix를 사용해 해당 commit을 설명하지 않습니다.
- 이후 commit에서 해결된 failure는 해당 commit 시점에는 미해결 상태로 기록합니다.
- 비교가 필요하면 현재 SHA의 parent 또는 문서가 연결한 이전 관련 SHA를 사용합니다.

## S/A/B/C별 학습 깊이

- **S:** Project-wide architecture/invariant로 취급합니다. Problem, 직전 상태, failure 가능성, decision, 핵심 code, ownership/lifecycle/state transition, 후속 fix/test, guarantee/non-guarantee를 모두 추적합니다.
- **A:** 주요 subsystem, boundary, integration point, failure path를 이해합니다. 핵심 code와 design judgment, caller/callee, state/reference 처리, regression evidence를 확인합니다.
- **B:** Thread 흐름에서 맡는 구현 역할과 필요한 code/state 변화를 확인합니다. Project-wide 결론을 과도하게 부여하지 않습니다.
- **C:** Thread 이해에 필요한 맥락으로만 사용합니다. 동일 깊이의 분석란을 억지로 채우지 않습니다.

## 실제 코드 삽입 기준

- Decision, invariant, ownership transfer, state transition, failure branch, cleanup 또는 test injection을 직접 보여 주는 최소 code만 삽입합니다.
- Code block 앞에 SHA, file path, symbol, 확인 목적을 적습니다.
- 변경 전/후를 비교할 때는 각각 어느 SHA인지 명시합니다.
- 대규모 diff, generated file 전체, 단순 markup 반복은 삽입하지 않습니다.
- Code 없이 설명 가능한 부분은 자신의 말로 정리합니다.

## Test commit 학습 방법

각 test commit에서 다음을 분리해 기록합니다.

- 대상 production invariant
- 재현하는 failure 또는 boundary
- 사용한 test technique과 fixture
- 통과하는 actual production code path
- Test가 증명하는 것
- Test가 증명하지 않는 것
- Broad integration test인지 deterministic regression인지
- 후속 변경에서 막는 회귀

## 문서 완료 기준

- 모든 thread 문서의 commit section을 해당 SHA evidence로 채웠습니다.
- 모든 SHA, importance, tags, thread order를 변경하지 않았습니다.
- S/A/B/C별 기록 깊이가 구분됩니다.
- Invariant의 도입, 강화, 부족함, fix, regression evidence가 ledger에 연결됩니다.
- Fix는 기존 가정 → failure/risk → root cause → corrected decision/invariant → code → test로 설명됩니다.
- Test는 production path와 injected failure를 연결하고 보장 범위를 제한해 설명합니다.
- 각 thread 최종 architecture 또는 execution flow를 별도 재학습 없이 설명할 수 있습니다.
===== END FILE: README.md =====

