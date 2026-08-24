===== BEGIN FILE: 01-content-declared-internal-route-integrity.md =====
# 개발 과정: 콘텐츠에 선언된 내부 라우트 무결성

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `06-seo-security-and-machine-readable-output`
>
> 1단계 검토에서 확정한 구조입니다. 2단계는 이 문서의 고정 항목과 커밋 순서를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- 저장소·브랜치 범위는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md`의 `web/portfolio`는 이 브랜치를 `cce7dd020563`부터 `aff0acdd4cf9`까지 이어지는 독립적인 선형 476개 커밋 이력으로 설명합니다. 아래 SHA는 모두 브랜치 내부 분류와 해당 커밋 객체·변경 내용에 대조했습니다.
- Subject, 중요도, 태그는 브랜치 내부 원본 분류와 일치시켰습니다.
- 아래 역할, 확인 대상, 불변 조건은 1단계 분류 검토에서 저장소 근거에 맞춰 동결했습니다.
- 다른 브랜치 또는 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표

JSON 콘텐츠가 선언한 최상위 경로 기준 URL을 실제 App Router 공개 표시 영역과 대조하고, 비활성화된 페이지·알 수 없는 프로젝트·지원하지 않는 경로가 렌더러에 도달하기 전에 원본 위치를 포함한 오류로 누적되는 과정을 복원합니다.

### 1단계 범위 결정

기존 초안은 쿼리 상태 URL 작성, 외부 링크의 `<a>` 요소 처리, 콘텐츠 검증을 하나로 묶었습니다. 1단계에서는 분류 02/03이 소유하는 UI 이동 방식 커밋을 제거하고, 검색 수집기와 공개 대상 범위의 정확성에 직접 영향을 주는 콘텐츠 원본 라우트 무결성만 남겼습니다.

### 고정된 핵심 불변 조건

- 검증 대상은 `/`로 시작하지만 `//`로 시작하지 않는 내부 라우트 참조입니다.
- 지원되지 않는 pathname, 비활성화된 페이지, 알 수 없는·비활성화된 프로젝트는 성공으로 통과하지 않습니다.
- 오류는 해당 JSON 원본 파일과 정확한 JSON 경로를 보존한 채 집계 객체 `PortfolioContentError`에 합쳐집니다.
- 사이트 탐색, 전역 링크, 프로젝트 링크가 동일한 검증 함수를 사용하고 렌더러는 이를 재해석하지 않습니다.

### 주요 기술적 어려움

- URL 문자열의 이동 방식 분류와 실제 공개 라우트 존재 여부 검증을 분리하는 문제
- 여러 JSON 파일에서 발견되는 오류를 첫 실패에서 중단하지 않고 원본 위치를 포함한 목록으로 누적하는 문제
- 페이지 사용 가능 여부와 활성화된 프로젝트 식별 정보를 검증 함수가 일관되게 참조하도록 만드는 문제

## 2. 핵심 질문

- `addInternalRouteIssue`는 어떤 입력을 의도적으로 무시하고 어떤 pathname만 검증합니까?
- 지원되는 페이지와 프로젝트 상세 라우트를 판정하는 실제 표·regular expression은 무엇입니까?
- 도우미 함수 도입 뒤 사이트, 전역 링크, 프로젝트 링크 소비자가 어떤 순서로 연결됩니까?
- 회귀 테스트는 어떤 콘텐츠 복제본을 변형하고 어떤 오류의 파일·메시지를 확인합니까?

## 3. 완료 기준

- 각 SHA에서 `src/lib/content-loader.ts`의 도우미 함수와 호출자 loop를 부모와의 변경 차이로 확인했습니다.
- 외부·프로토콜 상대 URL이 이 개발 과정의 검증 범위 밖이라는 점을 보장과 비보장으로 구분했습니다.
- `PortfolioContentError`의 집계 객체 문제가 원본 파일과 JSON 경로를 유지하는 흐름을 설명했습니다.
- `3353032ba23b`의 결정적인 콘텐츠 변경 테스트가 무엇을 증명하고 무엇을 증명하지 않는지 기록했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `b380f56f5d90` | feat(content): 내부 route 참조 검증 추가 | A | ARCH, CONTENT, VALIDATION | 내부 라우트 검증에 재사용하는 함수 |
| 2 | `6b9e10289b64` | feat(content): 사이트와 링크 route 참조 검증 추가 | A | ARCH, CONTENT, VALIDATION | 라우트 검증 함수를 사이트 탐색과 전역 링크에 연결 |
| 3 | `08b4ac81739f` | feat(content): 프로젝트 내부 참조 검증 추가 | A | CONTENT, VALIDATION | 프로젝트 참조와 프로젝트 내부 링크까지 무결성 검사 확장 |
| 4 | `3353032ba23b` | test(content): Vitest 기반 콘텐츠 계약 검증 추가 | A | CONTENT, VALIDATION, TEST | 원본 위치가 포함된 라우트 오류를 결정적으로 재현하는 회귀 테스트 |

## 5. 커밋별 학습 기록

### `b380f56f5d90` — feat(content): 내부 route 참조 검증 추가

- **중요도:** A
- **태그:** ARCH, CONTENT, VALIDATION
- **확정된 역할:** 내부 라우트 검증에 재사용하는 함수

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/content-loader.ts`의 `addInternalRouteIssue`와 커밋 부모 커밋을 비교합니다.
- `href.startsWith("/")`, `href.startsWith("//")`, `new URL(..., "https://portfolio.invalid")` 브랜치를 추적합니다.
- 지원 대상 페이지 맵, `/projects/<id>` regular expression, `enabledPageIds`, `enabledProjectIds`의 소유 주체를 확인합니다.
- 도우미 함수가 아직 어떤 원본 순회에도 호출되지 않는 통합 누락을 확인합니다.

확인 원칙:

- `b380f56f5d90^`와 `b380f56f5d90`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-b380f56f5d90 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 스키마와 파일 간 참조 검증은 존재했지만 콘텐츠의 최상위 경로 기준 URL이 실제 공개 라우트를 가리키는지는 중앙에서 확인하지 않았습니다. 잘못된 `/not-a-route`나 비활성화된 페이지 링크가 구조상 문자열로 통과할 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `src/lib/content-loader.ts`에 `addInternalRouteIssue(issues, file, path, href, enabledPageIds, enabledProjectIds, messagePrefix)`가 추가됩니다. URL을 임시 기준 URL로 파싱한 뒤 `/`, 정적 페이지 맵, `/projects/<id>` 순서로 판정합니다. |
| 데이터·상태·소유 주체 | 검증 결과의 소유 주체는 로더의 `issues` 배열입니다. 도우미 함수는 콘텐츠를 수정하지 않고 문제만 append하며, 사용 가능 여부 집합은 호출자가 주입합니다. |
| 실패·누락·대체 처리 | 외부 URL과 `//host/path`는 검사하지 않고 반환합니다. 지원하지 않는 경로, 비활성화된 페이지, 알 수 없는·비활성화된 프로젝트는 각각 원본 위치가 포함된 검증 문제를 추가합니다. 이 커밋 자체에는 호출자가 없어 실제 원본 로딩에는 아직 영향이 없습니다. |
| 보장·비보장 | 내부 pathname 판정 허용 값 집합은 생겼지만 사이트·전역·프로젝트 원본 전체 적용은 보장하지 않습니다. 잘못된 형식의 percent-encoding을 별도 복구하는 브랜치도 변경 내용에서 확인되지 않습니다. |
| 후속 연결 | `6b9e10289b64`가 사이트 탐색과 전역 링크에, `08b4ac81739f`가 프로젝트 링크에 이 도우미 함수를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// b380f56f5d90 — src/lib/content-loader.ts — addInternalRouteIssue
if (!href.startsWith("/") || href.startsWith("//")) return;
const pathname = new URL(href, "https://portfolio.invalid").pathname;
if (pathname === "/") return;
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

도우미 함수만 추가했으며 다음 커밋 전까지는 배포 환경 호출자가 없습니다.
<!-- learner:end commit-b380f56f5d90 -->


### `6b9e10289b64` — feat(content): 사이트와 링크 route 참조 검증 추가

- **중요도:** A
- **태그:** ARCH, CONTENT, VALIDATION
- **확정된 역할:** 라우트 검증 함수를 사이트 탐색과 전역 링크에 연결

#### 해당 SHA에서 확인할 실제 코드

- `loadPortfolioSource`에서 활성화된 페이지·프로젝트 집합이 만들어지는 위치를 확인합니다.
- `source.site.navigation.forEach`와 `source.links.forEach`가 넘기는 파일·경로·messagePrefix를 비교합니다.
- 비활성화된 링크도 스키마·로딩 경로에 남아 있는지, 라우트 검증이 활성화된 설정값을 조건으로 건너뛰는지 확인합니다.
- 여러 문제가 최종 `PortfolioContentError`로 합쳐지는 기존 예외 발생 구분 지점을 추적합니다.

확인 원칙:

- `6b9e10289b64^`와 `6b9e10289b64`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-6b9e10289b64 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Validator 도우미 함수는 존재했지만 호출되지 않아 실제 콘텐츠 로딩이 잘못된 탐색·링크를 거부하지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | `loadPortfolioSource`가 `site.navigation`과 전역 `links`를 순회하며 `addInternalRouteIssue`를 호출합니다. 각 호출은 `src/content/site.json` 또는 `src/content/links.json`과 배열 인덱스 기반 JSON 경로를 전달합니다. |
| 데이터·상태·소유 주체 | 라우트 사용 가능 여부 판단은 로더가 만든 `enabledPageIds`와 `enabledProjectIds`가 소유합니다. 개별 렌더러나 선택자는 검증 규칙을 갖지 않습니다. |
| 실패·누락·대체 처리 | 지원되지 않는 탐색은 `Unsupported internal navigation route`, 전역 링크는 `Unsupported internal link route` 계열 문제가 됩니다. 문제는 즉시 예외를 발생시키지 않고 기존 집계 객체 배열에 누적됩니다. |
| 보장·비보장 | 사이트와 전역 링크 원본은 보호되지만 프로젝트 항목 내부 링크는 아직 검사하지 않습니다. 외부 URL의 처리 형식·호스트 안전성도 이 로더 도우미 함수의 책임이 아닙니다. |
| 후속 연결 | `08b4ac81739f`가 프로젝트 내부 참조와 링크까지 같은 구분 지점으로 확장합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`08b4ac81739f`가 프로젝트 내부 참조와 링크까지 같은 구분 지점으로 확장합니다.
<!-- learner:end commit-6b9e10289b64 -->


### `08b4ac81739f` — feat(content): 프로젝트 내부 참조 검증 추가

- **중요도:** A
- **태그:** CONTENT, VALIDATION
- **확정된 역할:** 프로젝트 참조와 프로젝트 내부 링크까지 무결성 검사 확장

#### 해당 SHA에서 확인할 실제 코드

- `source.projects.items.forEach` 안에서 그룹, 태그, 기술 스택, 링크 검증 순서를 확인합니다.
- 프로젝트 링크의 파일·경로가 `src/content/projects.json`과 `$.items[i].links[j].href`로 보존되는지 확인합니다.
- `/projects/<id>`가 활성화된 프로젝트 집합과 대조되는 브랜치를 다시 확인합니다.
- 이 커밋이 라우트 무결성 외에 추가한 중복된·참조 문제를 라우트 검사와 구분합니다.

확인 원칙:

- `08b4ac81739f^`와 `08b4ac81739f`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-08b4ac81739f -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Site·전역 링크는 검증됐지만 프로젝트 사례 안의 내부 링크는 별도 경로라 알 수 없는 프로젝트 상세를 가리킬 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `loadPortfolioSource`의 프로젝트 loop가 각 `project.links` 진입점에 `addInternalRouteIssue`를 호출합니다. 같은 loop에서 그룹 ID, 중복된 태그·기술 스택, 기술 참조도 검사합니다. |
| 데이터·상태·소유 주체 | 프로젝트 원본 목록이 JSON 경로 소유 주체를 제공하고, 활성화된 프로젝트 ID가 상세 라우트 유효성을 결정합니다. 변형 없이 문제만 누적됩니다. |
| 실패·누락·대체 처리 | 알 수 없는 또는 비활성화된 프로젝트 상세는 프로젝트 JSON의 정확한 링크 경로에 문제를 남깁니다. 외부·프로토콜 상대 href는 계속 검사 범위 밖입니다. |
| 보장·비보장 | 세 종류의 콘텐츠 URL 원본이 동일 라우트 허용 값 집합을 사용하게 됩니다. 실제 Next 라우트 렌더링, HTTP 404, 외부 링크 보안 속성은 이 커밋이 보장하지 않습니다. |
| 후속 연결 | `3353032ba23b`가 지원하지 않는 전역 링크와 누락된 프로젝트 라우트를 결정적인 복제본 변경으로 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 08b4ac81739f — src/lib/content-loader.ts — project link loop
project.links.forEach((link, linkIndex) =>
  addInternalRouteIssue(
    issues,
    "src/content/projects.json",
    `$.items[${projectIndex}].links[${linkIndex}].href`,
    link.href,
    enabledPageIds,
    enabledProjectIds,
    "Unsupported internal project link route",
  ),
);
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`3353032ba23b`가 지원하지 않는 전역 링크와 누락된 프로젝트 라우트를 결정적인 복제본 변경으로 검증합니다.
<!-- learner:end commit-08b4ac81739f -->


### `3353032ba23b` — test(content): Vitest 기반 콘텐츠 계약 검증 추가

- **중요도:** A
- **태그:** CONTENT, VALIDATION, TEST
- **확정된 역할:** 원본 위치가 포함된 라우트 오류를 결정적으로 재현하는 회귀 테스트

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio.test.ts`의 `captureContentError`가 예외 타입을 어떻게 고정하는지 확인합니다.
- `rejects duplicate IDs, missing designs, and unsupported navigation` 테스트의 복제본 변경을 추적합니다.
- `rejects unsupported internal links and missing project routes`가 전역·프로젝트 링크를 어떻게 바꾸는지 확인합니다.
- 단언문이 정확한 전체 문제 목록이 아닌 `arrayContaining/objectContaining`임을 기록합니다.

확인 원칙:

- `3353032ba23b^`와 `3353032ba23b`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-3353032ba23b -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 배포 검증 함수는 구현됐지만 잘못된 라우트 원본이 다시 허용되는 회귀를 자동으로 잡는 실행 가능한 검증 지점이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | Vitest/jsdom/Testing Library 실행 환경을 추가하고 `src/lib/portfolio.test.ts`가 복제한 JSON을 `loadPortfolioSource`에 주입합니다. `captureContentError`는 발생한 예외 값이 `PortfolioContentError`인지 확인합니다. |
| 데이터·상태·소유 주체 | 테스트 입력은 imported JSON을 `structuredClone`해 원본 모듈 상태를 변경하지 않습니다. 실패 상태는 반환된 값이 아니라 exception의 `issues` 배열로 관찰합니다. |
| 실패·누락·대체 처리 | `/not-a-route` 탐색·전역 링크와 `/projects/not-a-project`를 주입해 파일·메시지를 확인합니다. 비활성 페이지·프로젝트 참조도 별도 테스트에서 검사합니다. |
| 보장·비보장 | 원본 위치를 포함한 라우트 거부의 결정적인 회귀 검증 근거입니다. HTTP 라우터, 브라우저 탐색, 모든 정확한 JSON 경로·순서, 잘못된 형식의 URL 파서 동작까지 증명하지는 않습니다. |
| 후속 연결 | 이 개발 과정의 실제 코드 경로는 로더에서 종료됩니다. UI 이동 방식과 실제 404 동작은 다른 분류·개발 과정이 소유합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 3353032ba23b — src/lib/portfolio.test.ts
links[0].href = "/not-a-route";
links[0].external = false;
projectWithLinks.links[0].href = "/projects/not-a-project";
const error = captureContentError(() => loadPortfolioSource({ links, projects }));
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

이 개발 과정의 실제 코드 경로는 로더에서 종료됩니다. UI 이동 방식과 실제 404 동작은 다른 분류·개발 과정이 소유합니다.
<!-- learner:end commit-3353032ba23b -->


## 6. 불변 조건 변화

<!-- learner:start thread-invariant-evolution -->
| Commit·구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| b380f56f5d90 | 도입됨 | 내부 라우트 값 집합과 문제를 추가하는 도우미 함수는 생겼지만 아직 실제 원본 순회에 연결되지 않았습니다. |
| 6b9e10289b64 | 확장됨 | 사이트 탐색과 전역 링크가 원본 위치를 포함한 경로 정보와 함께 도우미 함수를 호출합니다. |
| 08b4ac81739f | 완료 | 프로젝트 로컬 링크 및 프로젝트 identities enter the 같은 검증 지점. |
| 3353032ba23b | Deterministically 검증된 | Cloned 유효하지 않은 원본 reproduce 지원하지 않는 및 비활성화된 참조 failures. |
<!-- learner:end thread-invariant-evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:start thread-failure-fix-test -->
| 실패·위험 | 수정·결정 | 테스트·증거 |
| --- | --- | --- |
| 라우트 문자열이 스키마에는 맞지만 실제 라우트 규칙에는 맞지 않을 수 있음 | 중앙 도우미 함수가 지원·비활성·알 수 없는 내부 경로를 구분함 | Vitest가 탐색·전역·프로젝트 URL을 바꾸고 `PortfolioContentError`를 검사함 |
| 도우미 함수를 처음 추가했을 때 호출자가 없었음 | 두 통합 커밋이 모든 콘텐츠 URL 컬렉션을 연결함 | 최종 콘텐츠 테스트는 도우미 함수를 따로 호출하지 않고 집계 로더 전체를 호출함 |
<!-- learner:end thread-failure-fix-test -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:start thread-ownership -->
| 시점 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 이전 | 개별 콘텐츠 문자열과 렌더러 | 내부 경로가 활성화된 라우트에 해당하는지 판단하는 공용 규칙이 없었습니다. |
| b380 | `addInternalRouteIssue` | 담당: 경로 분류 but no 원본 순회. |
| 6b9 → 08b | `loadPortfolioSource` | 담당: 순회, 활성화된 집합, 원본 파일·경로, 및 집계 객체 실패. |
| 335 | `src/lib/portfolio.test.ts` | 담당: 체크인된 JSON을 변경하지 않는 결정적 회귀 테스트 입력. |
<!-- learner:end thread-ownership -->

## 9. 최종 개발 과정 상태와 실행 순서

<!-- learner:start thread-final-state -->
**최종 상태**

이 개발 과정이 끝나면 사이트 탐색, 전역 링크, 프로젝트 링크에 선언된 모든 내부 라우트를 콘텐츠 로딩 중 하나의 활성 라우트 목록과 대조합니다. 지원하지 않거나 비활성화된 주소가 있으면 유효한 포트폴리오 집계 객체를 만들지 않습니다. 외부 링크 속성과 실제 브라우저 탐색은 이 개발 과정의 범위가 아닙니다.

**코드 없는 실행 순서**

1. JSON 모듈 are 스키마 파싱된 into `PortfolioSource`.
2. `loadPortfolioSource`가 활성화된 페이지와 프로젝트의 식별자 집합을 만듭니다.
3. 관련 URL 필드를 원본 파일과 JSON 경로와 함께 `addInternalRouteIssue`에 전달합니다.
4. 외부 URL과 프로토콜 상대 주소는 이 검증 함수의 범위에서 제외합니다. 내부 경로는 최상위, 지원되는 페이지, 프로젝트 상세 또는 유효하지 않은 경로로 분류합니다.
5. 발견한 문제를 모두 누적한 뒤 로더가 하나의 `PortfolioContentError`를 발생시킵니다. 문제가 없는 원본만 선택자와 렌더러에 전달됩니다.
<!-- learner:end thread-final-state -->

## 10. 학습 완료 확인

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 정확한 변경 내용·파일 트리를 GitHub 연결 도구로 정적 확인했습니다.
- [x] 보장과 비보장을 커밋별로 구분했습니다.
- [x] 테스트 방법과 증명 범위와 증명하지 않는 범위를 구분했습니다.
- [x] 최종 처리 순서를 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 01-content-declared-internal-route-integrity.md =====

===== BEGIN FILE: 02-production-origin-and-publication-url-safety.md =====
# 개발 과정: 배포 출처 URL과 공개 URL 안전성

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `06-seo-security-and-machine-readable-output`
>
> 1단계 검토에서 확정한 구조입니다. 2단계는 이 문서의 고정 항목과 커밋 순서를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- 저장소·브랜치 범위는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md`의 `web/portfolio`는 이 브랜치를 `cce7dd020563`부터 `aff0acdd4cf9`까지 이어지는 독립적인 선형 476개 커밋 이력으로 설명합니다. 아래 SHA는 모두 브랜치 내부 분류와 해당 커밋 객체·변경 내용에 대조했습니다.
- Subject, 중요도, 태그는 브랜치 내부 원본 분류와 일치시켰습니다.
- 아래 역할, 확인 대상, 불변 조건은 1단계 분류 검토에서 저장소 근거에 맞춰 동결했습니다.
- 다른 브랜치 또는 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표

Permissive 템플릿 콘텐츠와 실제 공개 가능한 배포 콘텐츠를 분리하고, 공개 출처 URL, 자리표시자 제거, 자산, 프로젝트 외부 링크, 연락 수단이 모두 충족된 경우에만 검증된 배포 결과를 반환하는 검증 실패 시 공개를 차단하는 구분 지점을 복원합니다.

### 1단계 범위 결정

기존 초안은 `428055be3e64`의 판정 함수만 링크 보안 개발 과정 끝에 두어 실제 소유 주체와 실행 주기를 잃었습니다. 1단계에서는 모드·오류 형식부터 빌드 전 검사 통합과 회귀 테스트까지 독립 개발 과정으로 분리하고, 중간 중요도 B 자리표시자 검사 함수와 중요도 S 전체 검증 함수를 복원했습니다.

### 고정된 핵심 불변 조건

- 누락되거나 빈/`template` 모드는 템플릿이며 정확한 `production`만 엄격한 공개 검증을 요청합니다.
- 배포 환경 `SITE_URL`은 절대 공개 HTTP(S)이고 로컬·예약 호스트·인증 정보가 포함된 출처 URL이 아닙니다.
- 배포 환경 결과는 모든 원본 자리표시자 검사, 필수 `/content/` 자산, 활성 프로젝트의 외부 이동 링크, 사용 가능한 연락처가 모두 성공한 뒤에만 검증된 `URL`을 포함합니다.
- 준비 상태 실패는 첫 오류가 아니라 파일·경로·메시지 문제 목록으로 누적됩니다.
- 일반 `npm run build`는 스키마 검사 뒤 배포 준비 상태 검사를 반드시 실행합니다.

### 주요 기술적 어려움

- 초기 예시 템플릿을 로컬 미리보기에서는 허용하되 실제 공개에서는 실패 시 차단하는 방식으로 바꾸는 문제
- URL 형식, 공개 호스트 규칙, 자산 경로 범위, 프로젝트·연락처의 필수 데이터 규칙을 하나의 종류를 구분한 결과로 모으는 문제
- 검증 라이브러리의 실패를 CLI 종료 상태와 빌드 실행 주기에 정확히 전달하는 문제

## 2. 핵심 질문

- 모드 해석 함수가 허용하는 정확한 입력 집합과 유효하지 않은 값 동작은 무엇입니까?
- 자리표시자 검사 함수는 어떤 원본 파일 목록과 JSON 경로 형식화 함수를 사용합니까?
- `parsePublicSiteUrl`과 `isUsablePublicUrl`의 허용·거부 규칙은 어디가 다릅니까?
- 중요도 S 전체 검증 함수가 성공하기 전후 소유 주체와 반환 타입은 어떻게 달라집니까?
- 빌드 전 검사 단계를 우회할 수 있는 호출과 테스트가 증명하지 않는 범위는 무엇입니까?

## 3. 완료 기준

- 모드 → 검사 함수 → 출처 URL·링크 판정 함수 → 전체 검증 함수 → 데이터 영역 완전성 → 빌드 차단 단계 순서를 설명했습니다.
- 중요도 S `002b642d52a3`의 이전 위험, 실패 시 차단하는 결정, 종류를 구분한 결과, 남은 부족한 부분을 깊게 기록했습니다.
- `isUsablePublicUrl`이 배포 환경 `SITE_URL` 검증 함수와 동일하지 않은 구체적 보장하지 않는 범위를 확인했습니다.
- `fb3d18fd660b`의 테스트 입력 변환과 구분 지점 테스트를 실제 코드 경로에 연결했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `b3bd671a3243` | feat(content): 콘텐츠 mode와 readiness 오류 모델 추가 | A | CONTENT, VALIDATION | 보수적인 모드 결정과 구조화된 준비 상태 오류 형식 정의 |
| 2 | `741bbb4caab7` | feat(content): template placeholder 탐색 경계 추가 | B | CONTENT, ROUTING | 원본 파일과 경로를 보존하는 재귀 자리표시자 탐색 |
| 3 | `47b99d6256ef` | feat(content): public origin과 자산 경계 검증 추가 | A | CONTENT, VALIDATION | 공개 배포 출처 URL과 `/content/` 자산 경로 범위 검증 |
| 4 | `428055be3e64` | feat(content): 공개 URL과 연락 링크 검증 추가 | A | CONTENT, VALIDATION | 배포 가능한 프로젝트·연락처 URL 판정 함수 정의 |
| 5 | `002b642d52a3` | feat(content): production readiness 기본 검사 추가 | S | ARCH, CONTENT, VALIDATION | 하나라도 실패하면 차단하는 전체 배포 검증 지점 마련 |
| 6 | `bcd87ed856bf` | feat(content): 필수 자산과 프로젝트 readiness 추가 | A | CONTENT, VALIDATION | 배포 검증 결과에 포트폴리오 근거 완전성 검사 추가 |
| 7 | `71e7ece7208f` | feat(content): 연락 수단과 build readiness 연결 | A | CONTENT, VALIDATION, DEPLOY | 대상별 준비 상태 검사를 완성하고 모드별 진입점 하나만 노출 |
| 8 | `37c0dbc079ff` | build(content): readiness 검사를 prebuild에 연결 | A | CONTENT, VALIDATION, DEPLOY | 일반 npm 빌드에서 준비 상태 검사 강제 |
| 9 | `fb3d18fd660b` | test(content): readiness와 indexing 계약 검증 | A | CONTENT, VALIDATION, SEO | 전체 준비 결과와 공개 출처 URL 검증을 회귀 테스트 |

## 5. 커밋별 학습 기록

### `b3bd671a3243` — feat(content): 콘텐츠 mode와 readiness 오류 모델 추가

- **중요도:** A
- **태그:** CONTENT, VALIDATION
- **확정된 역할:** 보수적인 모드 결정과 구조화된 준비 상태 오류 형식 정의

#### 해당 SHA에서 확인할 실제 코드

- `PortfolioContentMode`, `PortfolioReadinessEnvironment`, 문제·결과 unions를 확인합니다.
- `PortfolioReadinessError`의 메시지 조립과 retained `issues` 소유 주체를 확인합니다.
- `resolvePortfolioContentMode`가 undefined·빈·템플릿·배포 환경·other를 처리하는 브랜치를 표로 만듭니다.
- 실제 배포 환경 검사가 아직 없다는 처리 형식·구현 구분 지점을 기록합니다.

확인 원칙:

- `b3bd671a3243^`와 `b3bd671a3243`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-b3bd671a3243 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 스키마 검증을 통과한 콘텐츠와 공개 가능한 콘텐츠를 구분하는 모드, 성공 타입, 집계 객체 배포 준비 상태 오류가 없었습니다. 템플릿 자리표시자가 배포 의도와 분리되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 새 `src/lib/content-readiness.ts`가 모드·환경·문제·결과 타입, `PortfolioReadinessError`, `resolvePortfolioContentMode`를 정의합니다. |
| 데이터·상태·소유 주체 | 모드 해석 함수는 문자열 입력을 수정하지 않는 유니언으로 바꾸고, 오류 객체가 문제 배열을 보존합니다. 배포 환경 성공 분기의 결과 타입만 `URL`을 소유하도록 구분된 유니언을 설계합니다. |
| 실패·누락·대체 처리 | `undefined`, 빈 문자열, `template`은 템플릿 모드로 처리합니다. 정확히 `production`일 때만 엄격한 배포 검증을 수행하며, 그 밖의 값은 일반 `Error`를 던집니다. |
| 보장·비보장 | 프로토콜과 실패 표현만 보장합니다. 아직 `SITE_URL`, 자리표시자, 자산, 프로젝트, 연락처를 검사하거나 배포 검증 결과를 생성하지 않습니다. |
| 후속 연결 | `741bbb4caab7`부터 구체적인 문제 생성 함수가 추가되고 `002b642d52a3`에서 집계 객체 성공·실패 구분 지점이 생깁니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// b3bd671a3243 — src/lib/content-readiness.ts — resolvePortfolioContentMode
if (value === undefined || value === "" || value === "template") {
  return "template";
}
if (value === "production") {
  return "production";
}
throw new Error(
  `PORTFOLIO_CONTENT_MODE must be "template" or "production"; received ${JSON.stringify(value)}.`,
);
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`741bbb4caab7`부터 구체적인 문제 생성 함수가 추가되고 `002b642d52a3`에서 집계 객체 성공·실패 구분 지점이 생깁니다.
<!-- learner:end commit-b3bd671a3243 -->


### `741bbb4caab7` — feat(content): template placeholder 탐색 경계 추가

- **중요도:** B
- **태그:** CONTENT, ROUTING
- **확정된 역할:** 원본 파일과 경로를 보존하는 재귀 자리표시자 탐색

#### 해당 SHA에서 확인할 실제 코드

- `contentFiles`가 모든 `PortfolioSource` 키를 해당 원본 파일 이름에 매핑하는지 확인합니다.
- `placeholderMarkers`의 정규식과 거짓 양성·거짓 음성 가능성을 기록합니다.
- `appendPath`가 식별자 키, 따옴표로 감싼 키, 배열 인덱스을 어떻게 표현하는지 확인합니다.
- `collectPlaceholderIssues`의 문자열·배열·객체 recursion과 비 객체 터미널 동작을 추적합니다.

확인 원칙:

- `741bbb4caab7^`와 `741bbb4caab7`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-741bbb4caab7 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 모드와 오류 타입은 있었지만 어떤 원본을 검사하고 자리표시자 위치를 어떻게 보고할지 구현되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | `contentFiles`, `placeholderMarkers`, `appendPath`, `findPlaceholderMarker`, `collectPlaceholderIssues`가 `src/lib/content-readiness.ts`에 추가됩니다. |
| 데이터·상태·소유 주체 | 검사 함수는 원본 객체를 변경하지 않고 호출자가 소유한 문제 배열에 파일·경로·메시지를 추가합니다. 경로는 `$`에서 시작해 배열 인덱스과 속성을 재귀적으로 확장합니다. |
| 실패·누락·대체 처리 | 문자열에서 첫 matching 식별 속성을 문제로 기록하고 반환합니다. 배열·객체는 재귀 순회하며 null·숫자·불리언은 무시합니다. |
| 보장·비보장 | Declared 식별 속성 허용 값 집합 탐지는 보장하지만 arbitrary 자리표시자 의미, cyclic 객체 방어, 실제 배포 환경 검사 단계 호출은 보장하지 않습니다. 중요도 B 보조 mechanism입니다. |
| 후속 연결 | `002b642d52a3`가 모든 `contentFiles`에 검사 함수를 호출해 누적된 배포 검증 실패에 포함합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 741bbb4caab7 — src/lib/content-readiness.ts — collectPlaceholderIssues
if (typeof value === "string") {
  const marker = findPlaceholderMarker(value);
  if (marker) {
    issues.push({
      file,
      path,
      message: `Replace the template marker "${marker.label}" with production content.`,
    });
  }
  return;
}
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`002b642d52a3`가 모든 `contentFiles`에 검사 함수를 호출해 누적된 배포 검증 실패에 포함합니다.
<!-- learner:end commit-741bbb4caab7 -->


### `47b99d6256ef` — feat(content): public origin과 자산 경계 검증 추가

- **중요도:** A
- **태그:** CONTENT, VALIDATION
- **확정된 역할:** 공개 배포 출처 URL과 `/content/` 자산 경로 범위 검증

#### 해당 SHA에서 확인할 실제 코드

- `isReservedHostname`의 정확한 데이터 영역·suffix 집합을 확인합니다.
- `parsePublicSiteUrl`의 누락된, 파싱 오류, 처리 형식, 로컬, reserved, 인증 정보 분기를 확인합니다.
- `resolveProductionSiteUrl`이 문제 배열을 `PortfolioReadinessError`로 바꾸는 경로를 확인합니다.
- URL 경로·쿼리와 해시를 명시적으로 거부하거나 일반ize하는지 확인해 보장하지 않는 범위에 기록합니다.

확인 원칙:

- `47b99d6256ef^`와 `47b99d6256ef`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-47b99d6256ef -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 자리표시자 scanning만으로는 기준·robots/사이트맵이 신뢰할 공개 출처 URL을 얻을 수 없고 템플릿 자산 경로 범위가 그대로 공개될 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `addProductionAssetIssue`, `isReservedHostname`, `parsePublicSiteUrl`, `resolveProductionSiteUrl`가 추가됩니다. URL 파서는 누락되거나 형식이 잘못됐거나 안전하지 않은 값을 구조화된 문제로 바꿉니다. |
| 데이터·상태·소유 주체 | Parsed `URL`은 성공 값이고 문제 배열은 호출자가 소유합니다. 자산 검사는 `/content/` 접두사 규칙만 적용합니다. |
| 실패·누락·대체 처리 | 누락된 값, URL이 아닌 값, HTTP(S)가 아닌 주소, 로컬 호스트·루프백·`.localhost`, 예약된 example·테스트·유효하지 않은 호스트와 사용자 이름·비밀번호가 포함된 주소를 거부합니다. `resolveProductionSiteUrl`은 문제가 하나라도 있으면 누적 오류를 발생시킵니다. |
| 보장·비보장 | 공개 호스트·처리 형식·credential 구분 지점은 보장하지만 URL 경로·쿼리와 해시를 최상위 출처 URL으로 강제하지는 않습니다. 호출자가 `.origin`을 사용할 때만 경로가 제거됩니다. |
| 후속 연결 | `002b642d52a3`가 파싱된 사이트 URL을 집계 객체 결과에 포함하고, 후속 메타데이터·robots/sitemap 소비자가 이 해석 함수를 직접 사용합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`002b642d52a3`가 파싱된 사이트 URL을 집계 객체 결과에 포함하고, 후속 메타데이터·robots/sitemap 소비자가 이 해석 함수를 직접 사용합니다.
<!-- learner:end commit-47b99d6256ef -->


### `428055be3e64` — feat(content): 공개 URL과 연락 링크 검증 추가

- **중요도:** A
- **태그:** CONTENT, VALIDATION
- **확정된 역할:** 배포 가능한 프로젝트·연락처 URL 판정 함수 정의

#### 해당 SHA에서 확인할 실제 코드

- `isUsablePublicUrl`의 자리표시자, URL 파싱, 처리 형식, 예약 호스트 조건을 확인합니다.
- `isUsableContactHref`가 `mailto:`/`tel:`과 공개 URL을 어떻게 합성하는지 확인합니다.
- `parsePublicSiteUrl`과 달리 로컬 호스트·인증 정보를 재검사하는지 비교합니다.
- 이 커밋 시점에 이 판정 함수를 배포 검증에서 호출하는 코드가 있는지 확인합니다.

확인 원칙:

- `428055be3e64^`와 `428055be3e64`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-428055be3e64 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 배포 출처 URL은 검증됐지만 프로젝트 외부 링크와 연락처 href가 실제 배포 가능한 목적지인지 재사용 가능한 방식으로 판정할 도우미 함수가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `src/lib/content-readiness.ts`에 `isUsablePublicUrl`과 `isUsableContactHref`가 추가됩니다. 자리표시자 문자열을 먼저 거부하고 URL 프로토콜과 호스트 이름 또는 연락처 스킴을 판정합니다. |
| 데이터·상태·소유 주체 | 판정 함수는 불리언만 반환하고 문제를 직접 만들지 않습니다. 도메인별 검증 함수가 불리언을 소비해 원본 경로가 있는 문제로 변환할 책임을 가집니다. |
| 실패·누락·대체 처리 | 잘못된 URL, HTTP(S)가 아닌 URL, 예약 호스트는 `false`입니다. 연락처 링크는 자리표시자가 아니면 `mailto:`/`tel:`을 즉시 `true`로 처리하거나 공개 URL 판정 함수에 위임합니다. |
| 보장·비보장 | `isUsablePublicUrl`은 `parsePublicSiteUrl`과 동일하지 않습니다. 이 변경 내용에서는 로컬 호스트와 인증 정보를 명시적으로 거부하지 않으며, `mailto:`/`tel:` 페이로드 형식도 검증하지 않습니다. 이 점은 후속 테스트에서도 직접 보호되지 않습니다. |
| 후속 연결 | `bcd87ed856bf`가 프로젝트 링크에, `71e7ece7208f`가 연락처 선택에서 이 판정 함수를 사용합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 428055be3e64 — src/lib/content-readiness.ts
export function isUsableContactHref(href: string) {
  if (findPlaceholderMarker(href)) return false;
  return href.startsWith("mailto:") || href.startsWith("tel:") || isUsablePublicUrl(href);
}
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`bcd87ed856bf`가 프로젝트 링크에, `71e7ece7208f`가 연락처 선택에서 이 판정 함수를 사용합니다.
<!-- learner:end commit-428055be3e64 -->


### `002b642d52a3` — feat(content): production readiness 기본 검사 추가

- **중요도:** S
- **태그:** ARCH, CONTENT, VALIDATION
- **확정된 역할:** 하나라도 실패하면 차단하는 전체 배포 검증 지점 마련

#### 해당 SHA에서 확인할 실제 코드

- `ProductionReadinessResult`가 유니언에서 배포 환경 브랜치만 추출하는 방식을 확인합니다.
- `validateProductionReadiness`의 문제 initialization → 출처 URL 파싱 → 모든 원본 검사 → single 실패 구분 지점 → 성공 반환 순서를 추적합니다.
- 이전 도우미 함수가 서로 분리된 도우미 함수에서 하나 기준이 되는 공개 결과로 바뀌는 소유 주체 전환 효과를 기록합니다.
- 이 SHA에서 아직 자산·프로젝트·연락처 완전성이 없는 부족한 부분을 명시합니다.

확인 원칙:

- `002b642d52a3^`와 `002b642d52a3`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-002b642d52a3 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 모드, 검사 함수, 출처 URL·링크 도우미 함수는 각각 존재했지만 호출자가 임의로 일부만 사용할 수 있었습니다. `production`이라는 상태가 전체 검증 없이 선언될 위험이 남아 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `validateProductionReadiness(content, {SITE_URL})`가 하나 문제 배열을 만들고 `parsePublicSiteUrl`과 모든 `contentFiles` 진입점의 `collectPlaceholderIssues`를 실행한 뒤, any 실패에서 `PortfolioReadinessError`를 예외 발생하고 성공 시 `{mode: "production", siteUrl}`을 반환합니다. |
| 데이터·상태·소유 주체 | 이 커밋부터 검증된 `URL`의 소유 주체는 종류가 구분된 성공 결과에 있습니다. 호출자는 검증을 통과하지 않고 배포 검증 결과를 구성할 수 없으며 원본 콘텐츠는 변경되지 않습니다. |
| 실패·누락·대체 처리 | 유효하지 않은·누락된 출처 URL과 모든 자리표시자 문제가 한 목록에 공존하므로 앞선 실패가 뒤의 원본 문제를 가리지 않습니다. `!siteUrl \|\| issues.length > 0`가 단일 실패 시 차단하는 분기입니다. |
| 보장·비보장 | 중요도 S의 핵심 불변 조건은 출처 URL과 모든 원본의 자리표시자 검증을 통과한 뒤에만 배포 검증 결과를 만든다는 것입니다. 필수 자산 존재 여부, `/content/` 경로, 활성 프로젝트의 외부 이동 링크, 사용 가능한 연락 수단, 템플릿 모드 우회와 빌드 통합은 아직 포함하지 않습니다. |
| 후속 연결 | `bcd87ed856bf`와 `71e7ece7208f`가 데이터 영역 완전성을 확장하고, `37c0dbc079ff`가 일반 빌드 실행 주기에 강제하며, `fb3d18fd660b`가 집계 객체 분류를 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 002b642d52a3 — src/lib/content-readiness.ts
const issues: PortfolioReadinessIssue[] = [];
const siteUrl = parsePublicSiteUrl(environment.SITE_URL, issues);
for (const [key, file] of contentFiles) collectPlaceholderIssues(content[key], file, "$", issues);
if (!siteUrl || issues.length > 0) throw new PortfolioReadinessError(issues);
return { mode: "production", siteUrl };
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`bcd87ed856bf`와 `71e7ece7208f`가 데이터 영역 완전성을 확장하고, `37c0dbc079ff`가 일반 빌드 실행 주기에 강제하며, `fb3d18fd660b`가 집계 객체 분류를 검증합니다.
<!-- learner:end commit-002b642d52a3 -->


### `bcd87ed856bf` — feat(content): 필수 자산과 프로젝트 readiness 추가

- **중요도:** A
- **태그:** CONTENT, VALIDATION
- **확정된 역할:** 배포 검증 결과에 포트폴리오 근거 완전성 검사 추가

#### 해당 SHA에서 확인할 실제 코드

- 사이트 소셜 이미지, 프로필 사진, 이력서 다운로드의 존재 여부와 `/content/` 검사를 확인합니다.
- 활성화된 프로젝트 필터링과 0-프로젝트 실패를 확인합니다.
- 각 활성화된 프로젝트의 화면 캡처 목록과 `isUsablePublicUrl` 링크 requirement를 추적합니다.
- 비활성화된 프로젝트가 왜 건너뛰기되는지 공개 대상 범위와 연결합니다.

확인 원칙:

- `bcd87ed856bf^`와 `bcd87ed856bf`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-bcd87ed856bf -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 자리표시자가 없는 일반 콘텐츠와 공개 출처 URL과 자리표시자 검사만으로 배포 준비 성공이 가능해, 실제 이미지·이력서·프로젝트 외부 링크가 없는 내용이 내용이 비어 있는 공개 결과도 통과할 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `validateProductionReadiness`가 사이트 소셜 이미지, 프로필 사진, 이력서 URL, 활성화된 프로젝트 개수, 프로젝트 화면 캡처, 활성화된 공개 프로젝트 URL을 검사합니다. |
| 데이터·상태·소유 주체 | 공개 콘텐츠 완전성 검사는 전체 검증 함수가 담당하며, 각 실패는 원래 원본 파일과 목록 경로를 사용합니다. `enabled=false`인 프로젝트는 공개 대상에서 제외합니다. |
| 실패·누락·대체 처리 | 누락 자산은 명시적인 문제, 잘못된 namespace는 `addProductionAssetIssue`, no 활성화된 프로젝트·외부 이동 링크는 프로젝트별 문제가 됩니다. 검사는 계속 진행되어 여러 분류를 함께 보고합니다. |
| 보장·비보장 | 공개된 프로젝트마다 저장소가 소유한 시각 근거와 최소 하나의 외부 이동 링크를 요구합니다. 연락처 method는 아직 요구하지 않고, 공개 링크 판정 함수의 로컬 호스트/credential 보장하지 않는 범위는 그대로입니다. |
| 후속 연결 | `71e7ece7208f`가 연락처 requirement와 모드에 따라 동작하는 진입점을 완성합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// bcd87ed856bf — src/lib/content-readiness.ts — enabled project exit
if (
  !project.links.some(
    (link) => link.enabled !== false && isUsablePublicUrl(link.href),
  )
) {
  issues.push({
    file: "src/content/projects.json",
    path: `$.items[${projectIndex}].links`,
    message: "Add at least one enabled public project URL for production.",
  });
}
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`71e7ece7208f`가 연락처 requirement와 모드에 따라 동작하는 진입점을 완성합니다.
<!-- learner:end commit-bcd87ed856bf -->


### `71e7ece7208f` — feat(content): 연락 수단과 build readiness 연결

- **중요도:** A
- **태그:** CONTENT, VALIDATION, DEPLOY
- **확정된 역할:** 대상별 준비 상태 검사를 완성하고 모드별 진입점 하나만 노출

#### 해당 SHA에서 확인할 실제 코드

- `hasContactMethod`의 활성화된, 노출 위치, 타입, href 판정 함수 조건을 확인합니다.
- 사용 가능한 연락 수단 없음 문제의 원본·경로를 확인합니다.
- `validateBuildReadiness`의 템플릿 즉시 반환과 배포 환경 위임을 확인합니다.
- 도우미 함수 공개 항목이 내부로 축소되는 소유 주체 정리를 확인합니다.

확인 원칙:

- `71e7ece7208f^`와 `71e7ece7208f`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-71e7ece7208f -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 배포 환경 콘텐츠가 프로젝트 근거는 갖춰도 방문자가 연락할 실제 경로가 없을 수 있었고, 호출자는 모드 해석과 엄격한 검증 함수를 직접 조합해야 했습니다. |
| 실제 변경 파일·심볼·호출 경로 | `validateProductionReadiness`가 연락처 노출 위치에 포함된 활성화된 email/github/website 링크 중 사용 가능한 href가 하나 이상인지 검사합니다. `validateBuildReadiness`가 모드를 해석하고 템플릿은 즉시 성공, 배포 환경은 엄격한 검증 함수에 위임합니다. |
| 데이터·상태·소유 주체 | 모드 branching의 소유 주체가 하나 공개 함수으로 이동합니다. 내부 도우미 함수와 constant 공개 항목은 내부로 좁혀져 외부 호출자가 일부 규칙을 조합하기 어려워집니다. |
| 실패·누락·대체 처리 | 사용 가능한 연락처가 없으면 `src/content/links.json:$` 문제를 추가합니다. 템플릿 모드는 자리표시자·공개 검사를 의도적으로 건너뛰고 `{mode, siteUrl: undefined}`를 반환합니다. |
| 보장·비보장 | 모드를 해석하는 라이브러리 진입점과 최소 연락 수단 요구 사항를 보장합니다. 빌드 프로세스가 이 함수를 호출하는지는 아직 보장하지 않습니다. |
| 후속 연결 | `37c0dbc079ff`가 CLI와 `prebuild`에 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 71e7ece7208f — src/lib/content-readiness.ts
export function validateBuildReadiness(content, environment) {
  const mode = resolvePortfolioContentMode(environment.PORTFOLIO_CONTENT_MODE);
  if (mode === "template") return { mode, siteUrl: undefined };
  return validateProductionReadiness(content, environment);
}
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`37c0dbc079ff`가 CLI와 `prebuild`에 연결합니다.
<!-- learner:end commit-71e7ece7208f -->


### `37c0dbc079ff` — build(content): readiness 검사를 prebuild에 연결

- **중요도:** A
- **태그:** CONTENT, VALIDATION, DEPLOY
- **확정된 역할:** 일반 npm 빌드에서 준비 상태 검사 강제

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `prebuild`, `content:check`, `content:ready` 스크립트와 셸의 단축 평가 순서를 확인합니다.
- `scripts/validate-content-readiness.ts`의 원본 로딩, env read, 결과 logging을 확인합니다.
- 알려진 배포 준비 상태 오류는 `process.exitCode = 1`, 예상하지 못한 오류는 다시 던지는 브랜치를 확인합니다.
- `npm run build`와 직접 `next build`의 실행 주기 차이를 보장하지 않는 범위로 기록합니다.

확인 원칙:

- `37c0dbc079ff^`와 `37c0dbc079ff`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-37c0dbc079ff -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Correct 라이브러리 함수는 존재했지만 빌드가 호출하지 않으면 incomplete 배포 콘텐츠가 산출물로 만들어질 수 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `prebuild`가 `npm run content:check && npm run content:ready`로 바뀌고 새 스크립트가 로더와 `validateBuildReadiness`를 호출합니다. |
| 데이터·상태·소유 주체 | Library 오류가 프로세스 종료 상태로 전파됩니다. Template·배포 환경 성공은 로그만 만들고 원본을 변경하지 않습니다. |
| 실패·누락·대체 처리 | 알려진 누적 배포 준비 상태 오류는 형식화한 메시지를 stderr에 쓰고 종료 코드 1을 설정합니다. 예상하지 못한 오류는 숨기지 않고 예외를 던집니다. 첫 `content:check` 실패 시 셸 `&&`로 배포 준비 상태는 실행되지 않습니다. |
| 보장·비보장 | 일반 `npm run build` 실행 주기에는 필수 검사 단계가 생깁니다. Direct `next build` 또는 스크립트 검사 생략까지 기술적으로 차단하지는 않습니다. |
| 후속 연결 | `fb3d18fd660b`가 라이브러리 수준 동작을 검증하지만 이 CLI·빌드 전 검사 프로세스 자체를 spawn해 검증하지는 않습니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 37c0dbc079ff — package.json
"prebuild": "npm run content:check && npm run content:ready",
"content:ready": "node --import tsx scripts/validate-content-readiness.ts"
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`fb3d18fd660b`가 라이브러리 수준 동작을 검증하지만 이 CLI·빌드 전 검사 프로세스 자체를 spawn해 검증하지는 않습니다.
<!-- learner:end commit-37c0dbc079ff -->


### `fb3d18fd660b` — test(content): readiness와 indexing 계약 검증

- **중요도:** A
- **태그:** CONTENT, VALIDATION, SEO
- **확정된 역할:** 전체 준비 결과와 공개 출처 URL 검증을 회귀 테스트

#### 해당 SHA에서 확인할 실제 코드

- `replaceTemplateMarkers`와 `createProductionReadyContent`의 고정된 테스트 입력 구축을 확인합니다.
- 모드 기본값·유효하지 않은, 템플릿 검사 생략, 집계 객체 분류, 성공 결과, 유효하지 않은 SITE_URL 테스트를 분류합니다.
- 단언문이 정확한 문제 순서보다 분류·경로 존재 여부를 검사하는 이유와 한계를 기록합니다.
- 같은 커밋의 `site-metadata.test.ts`는 검색 노출 개발 과정에서 별도 관점으로 확인합니다.

확인 원칙:

- `fb3d18fd660b^`와 `fb3d18fd660b`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-fb3d18fd660b -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 준비 상태 검사와 빌드 차단은 있었지만 모드 기본값, 문제 누적, 유효한 배포 검증 결과와 출처 URL 거부 규칙을 고정하는 단위 테스트가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `src/lib/content-readiness.test.ts`가 저장소의 원본을 복제하고 자리표시자를 배포용 값으로 바꾼 뒤 필수 자산과 프로젝트 링크를 채웁니다. 테스트는 공개 함수인 `validateBuildReadiness`와 `validateProductionReadiness`를 호출합니다. |
| 데이터·상태·소유 주체 | 테스트 입력 도우미 함수가 수정 가능한 복제본을 소유하고 imported 원본은 보존합니다. 실패는 `captureReadinessError`가 타입이 지정된 오류와 문제 목록으로 관찰합니다. |
| 실패·누락·대체 처리 | 지원하지 않는 모드, 템플릿 모드의 검사 생략, 여러 범주의 문제 누적, 전체 성공, 형식이 잘못됐거나 FTP·로컬 호스트·예시 도메인인 출처 URL의 거부를 고정합니다. |
| 보장·비보장 | 입력 검증 지점·단위 회귀 검증 근거이며 CLI 빌드 전 검사 실행, 실제 파일 시스템 자산 존재 여부, 인증 정보가 포함된 URL, 로컬 프로젝트·연락처 href, 브라우저 검색 노출을 증명하지 않습니다. |
| 후속 연결 | 검색 노출 개발 과정에서는 같은 커밋의 `site-metadata.test.ts`가 템플릿의 `noindex`·robots 설정과 배포 출처 URL 동작을 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// fb3d18fd660b — src/lib/content-readiness.test.ts
for (const siteUrl of ["not-a-url", "ftp://portfolio.example.dev", "http://localhost:3100", "https://example.com"]) {
  const error = captureReadinessError(() => validateProductionReadiness(createProductionReadyContent(), { SITE_URL: siteUrl }));
  expect(error.issues).toEqual(expect.arrayContaining([expect.objectContaining({ path: "SITE_URL" })]));
}
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

검색 노출 개발 과정에서는 같은 커밋의 `site-metadata.test.ts`가 템플릿 noindex/robots와 배포 출처 URL 동작을 검증합니다.
<!-- learner:end commit-fb3d18fd660b -->


## 6. 불변 조건 변화

<!-- learner:start thread-invariant-evolution -->
| Commit·구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| b3bd671a3243 | 도입됨 | 보수적인 모드 해석과 구조화된 결과·오류 형식을 정의합니다. |
| 741bbb4caab7 | 확장됨 | 모든 원본에서 파일과 JSON 경로를 포함한 자리표시자 문제를 수집할 수 있습니다. |
| 47b99d6256ef → 428055be3e64 | 확장됨 | 공개 출처 URL, 자산 경로 범위, 프로젝트·연락처 URL 판정 규칙을 명시합니다. |
| 002b642d52a3 | 전체 검사로 강제됨 | 하나의 전체 검증 함수가 배포 성공 여부를 결정하고 검증된 URL을 반환합니다. |
| bcd87ed856bf → 71e7ece7208f | 완료됨 | 필수 자산, 프로젝트 외부 링크, 연락 수단과 모드별 진입점까지 검사 범위를 확장합니다. |
| 37c0dbc079ff | 통합됨 | 일반 npm 빌드가 준비 상태 검사 결과를 사용하고 실패를 프로세스 종료 상태로 전달합니다. |
| fb3d18fd660b | 결정적으로 검증됨 | 모드, 문제 누적, 성공 결과와 출처 URL 제한을 회귀 테스트로 고정합니다. |
<!-- learner:end thread-invariant-evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:start thread-failure-fix-test -->
| 실패·위험 | 수정·결정 | 테스트·증거 |
| --- | --- | --- |
| 스키마 검증을 통과한 템플릿이 그대로 공개될 수 있음 | 명시적 모드, 전체 배포 검증 함수와 필수 빌드 전 검사를 추가함 | 준비 상태 테스트가 템플릿 모드의 검사 생략과 배포 모드의 엄격한 실패를 검증함 |
| 출처 URL과 도우미 함수 규칙이 일부만 적용될 수 있음 | 중요도 S의 성공 결과가 출처 URL과 원본 검사를 한곳에 모음 | 전체 테스트 입력은 반드시 `{mode: production, siteUrl: URL}`을 반환해야 함 |
| 공개 결과에 근거·외부 이동 링크·연락 수단이 없을 수 있음 | 공개 콘텐츠 완전성 검사가 같은 문제 배열에 진단을 추가함 | 테스트가 후속 문제를 가리지 않고 모든 입력 분류의 오류를 확인함 |
<!-- learner:end thread-failure-fix-test -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:start thread-ownership -->
| 시점 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 이전 | 호출자와 환경 변수 | 미리보기 상태와 공개 가능한 상태를 타입으로 구분하지 않았습니다. |
| b3bd → 428 | `content-readiness.ts`의 도우미 함수 | 모드 허용값과 문제 생성 규칙을 담당하지만 전체 성공 여부를 정하는 함수는 아직 없습니다. |
| 002 | `validateProductionReadiness` | 검증을 통과한 배포 결과와 URL을 반환합니다. |
| 71 | `validateBuildReadiness` | 템플릿·배포 모드 분기를 담당하고 내부 도우미 함수의 공개 범위를 줄입니다. |
| 37 | npm 빌드 전 검사 + 배포 준비 상태 CLI | 라이브러리 실패를 빌드 종료 상태로 전달합니다. |
<!-- learner:end thread-ownership -->

## 9. 최종 개발 과정 상태와 실행 순서

<!-- learner:start thread-final-state -->
**최종 상태**

템플릿 모드는 로컬 미리보기에 편리하지만 공개 가능한 상태로 취급하지 않습니다. 배포 모드는 공개 `URL`을 담은 검증 결과로만 표현합니다. 자리표시자, 출처 URL, 필수 자산, 활성 프로젝트, 프로젝트 외부 이동 링크와 연락 수단을 모두 검사하기 전에는 이 결과를 반환하지 않습니다. 일반적인 npm 빌드는 Next 컴파일 전에 이 검사를 실행합니다.

**코드 없는 실행 순서**

1. 배포 준비 상태 CLI가 스키마 검증을 통과한 포트폴리오 원본을 불러오고 `PORTFOLIO_CONTENT_MODE`와 `SITE_URL`을 읽습니다.
2. `validateBuildReadiness`가 모드를 해석합니다. 템플릿 모드이면 엄격한 공개 검사를 수행하지 않고 결과를 반환합니다.
3. 배포 모드이면 `validateProductionReadiness`에 위임하여 출처 URL, 자리표시자, 자산, 프로젝트와 연락처 문제를 누적합니다.
4. 문제가 하나라도 있으면 `PortfolioReadinessError`를 던집니다. 문제가 없으면 파싱한 사이트 URL을 포함한 배포 검증 결과를 반환합니다.
5. CLI는 알려진 실패를 종료 코드 1로 바꾸므로 `npm run build`는 `next build` 실행 주기에서 멈추고 `prebuild`를 시작하지 않습니다.
<!-- learner:end thread-final-state -->

## 10. 학습 완료 확인

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 정확한 변경 내용·파일 트리를 GitHub 연결 도구로 정적 확인했습니다.
- [x] 보장과 비보장을 커밋별로 구분했습니다.
- [x] 테스트 방법과 증명 범위와 증명하지 않는 범위를 구분했습니다.
- [x] 최종 처리 순서를 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 02-production-origin-and-publication-url-safety.md =====

===== BEGIN FILE: 03-route-aware-metadata-and-canonical-identity.md =====
# 개발 과정: 라우트별 메타데이터와 기준 URL 식별 정보

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `06-seo-security-and-machine-readable-output`
>
> 1단계 검토에서 확정한 구조입니다. 2단계는 이 문서의 고정 항목과 커밋 순서를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- 저장소·브랜치 범위는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md`의 `web/portfolio`는 이 브랜치를 `cce7dd020563`부터 `aff0acdd4cf9`까지 이어지는 독립적인 선형 476개 커밋 이력으로 설명합니다. 아래 SHA는 모두 브랜치 내부 분류와 해당 커밋 객체·변경 내용에 대조했습니다.
- Subject, 중요도, 태그는 브랜치 내부 원본 분류와 일치시켰습니다.
- 아래 역할, 확인 대상, 불변 조건은 1단계 분류 검토에서 저장소 근거에 맞춰 동결했습니다.
- 다른 브랜치 또는 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표

콘텐츠에서 계산한 사이트 식별 정보를 최상위 레이아웃 메타데이터로 옮기고, 배포 환경에서는 검증된 `SITE_URL`을 기준 출처 URL로 사용하며, 각 공개 라우트가 쿼리를 제외한 기준 경로와 라우트가 소유한 제목·설명·Open Graph/Twitter 값을 공개하도록 확장하는 과정을 복원합니다.

### 1단계 범위 결정

기존 초안의 라우트 순서는 대체로 맞았지만 최초 요청 헤더 메타데이터가 후속 모드에 따라 동작하는 생성 함수로 대체되는 소유 주체 이전가 빠져 있었고, sitemap 테스트 커밋까지 한 개발 과정에 섞여 있었습니다. 1단계에서는 `55b6061e0052`와 `67aabeab1553`을 추가해 실제 전환을 복원하고, `adc392157f70`은 sitemap 개발 과정으로 이동했습니다.

### 고정된 핵심 불변 조건

- 배포 환경 메타데이터 출처 URL은 요청 호스트 추정이 아니라 검증된 `SITE_URL`에서 옵니다.
- 최상위 기준은 `/`, 비관리자 기준은 해당 라우트 경로이며 `view`, `debug` 같은 쿼리 상태를 포함하지 않습니다.
- 라우트 제목·설명은 기준이 되는 콘텐츠에서 읽고, 비관리자 제목은 사이트 브랜드와 결합합니다.
- 비활성 페이지와 알 수 없는 프로젝트는 메타데이터 공개에서도 `notFound()`로 차단됩니다.
- 공용 생성 함수가 메타데이터 형식을 소유하고 각 App Router 페이지는 라우트 사용 가능 여부와 콘텐츠 선택을 소유합니다.

### 주요 기술적 어려움

- Reverse proxy 요청 headers에서 계산한 convenient 미리보기 출처 URL과 배포 환경 기준 출처 URL을 분리하는 문제
- 전역 사이트 메타데이터와 라우트 전용 메타데이터를 중복 없이 합성하면서 기준 URL 식별 정보를 고정하는 문제
- 동적 프로젝트 메타데이터가 페이지 렌더링과 같은 사용 가능 여부·프로젝트 조회 규칙을 사용하도록 맞추는 문제

## 2. 핵심 질문

- 최초 `generateMetadata`는 호스트·처리 형식을 어떻게 추정하며 어떤 신뢰 가정을 가집니까?
- 순수 생성 함수 도입 뒤 배포 환경·템플릿 metadataBase 소유 주체가 어떻게 이동합니까?
- `createRouteMetadata`는 최상위와 비관리자 제목, 기준, Open Graph URL을 어떻게 계산합니까?
- 각 라우트 공개가 어떤 콘텐츠 필드를 제목·설명으로 사용하고 비활성화된 상태를 어디서 거부합니까?
- 라우트 공개 테스트는 공용 도우미 함수 테스트와 달리 무엇을 실제로 검증합니까?

## 3. 완료 기준

- `1f4f93ad9a0f → 55b6061e0052 → 67aabeab1553`의 transitional 출처 URL·소유 주체 변화를 설명했습니다.
- 각 라우트 SHA에서 정확한 페이지 파일의 `generateMetadata`와 `notFound` 경로를 확인했습니다.
- 기준 경로와 화면 선택용 쿼리 상태를 분리한 규칙을 기록했습니다.
- `4358bcd34f2e`가 실제 라우트 공개 항목을 호출하는 테스트 방법과 한계를 구분했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `1f4f93ad9a0f` | feat(metadata): 콘텐츠 기반 site metadata 추가 | A | CONTENT, SEO | 콘텐츠에서 만든 사이트 메타데이터를 최상위 레이아웃에 적용 |
| 2 | `55b6061e0052` | feat(seo): 콘텐츠 mode별 metadata 정책 추가 | A | CONTENT, SEO | 콘텐츠 모드에만 의존하는 사이트 메타데이터 생성 함수 분리 |
| 3 | `67aabeab1553` | feat(seo): layout metadata를 콘텐츠 mode에 연결 | A | CONTENT, SEO | 최상위 메타데이터 생성을 모드별 규칙과 검증된 배포 출처 URL으로 이전 |
| 4 | `1c40645caead` | feat(seo): route별 검색 metadata 정책 추가 | A | ARCH, ROUTING, SEO | 라우트 메타데이터 식별 규칙 공통화 |
| 5 | `844ff4d7abcb` | feat(seo): 홈과 프로젝트 route metadata 연결 | B | ROUTING, SEO | 공통 규칙을 홈·프로젝트 목록·동적 프로젝트 상세에 적용 |
| 6 | `fd5ff532bfe9` | feat(seo): 프로필 route metadata 연결 | B | ROUTING, SEO | 소개·연락처·이력서에 라우트 메타데이터 적용 |
| 7 | `5632c5df9b47` | feat(seo): 여정과 근거 route metadata 연결 | B | ROUTING, SEO | 여정과 인터뷰 근거까지 라우트 메타데이터 적용 완료 |
| 8 | `4358bcd34f2e` | test(seo): route metadata export 검증 | B | VALIDATION, ROUTING, SEO | 모든 공개 라우트가 실제로 내보내는 메타데이터를 현재 동작으로 고정 |

## 5. 커밋별 학습 기록

### `1f4f93ad9a0f` — feat(metadata): 콘텐츠 기반 site metadata 추가

- **중요도:** A
- **태그:** CONTENT, SEO
- **확정된 역할:** 콘텐츠에서 만든 사이트 메타데이터를 최상위 레이아웃에 적용

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`의 정적 메타데이터 이전 상태와 async `generateMetadata` 변경 내용을 비교합니다.
- `headers()`에서 `x-forwarded-host`/`host`, `x-forwarded-proto`, 로컬 호스트 대체 처리를 선택하는 순서를 확인합니다.
- `site.title`, `site.description`, `site.socialImage`가 제목·Open Graph/Twitter로 흐르는 경로를 추적합니다.
- Request-controlled 출처 URL과 상대 기준 `./`가 남기는 배포 환경 신뢰 부족한 부분을 기록합니다.

확인 원칙:

- `1f4f93ad9a0f^`와 `1f4f93ad9a0f`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-1f4f93ad9a0f -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 최상위 레이아웃은 고정 또는 최소 메타데이터만 제공해 JSON 콘텐츠의 제목, 설명, 소셜 이미지가 검색 수집기·소셜 출력에 일관되게 반영되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | `src/app/layout.tsx`의 `generateMetadata()`가 요청 headers로 `metadataBase`를 만들고 모듈 범위 `site` 콘텐츠를 제목, 설명, Open Graph, Twitter와 기준 `./`에 배치합니다. |
| 데이터·상태·소유 주체 | 이 시점의 메타데이터 형식과 출처 URL 추론을 최상위 레이아웃 함수 하나가 모두 소유합니다. 콘텐츠는 읽기 전용 입력이고 요청 headers가 출처 URL 입력입니다. |
| 실패·누락·대체 처리 | Forwarded 호스트가 없으면 호스트, 그것도 없으면 `localhost:3100`을 사용합니다. 프로토콜 헤더가 없으면 로컬 호스트만 http, 그 외 https로 추정합니다. 유효하지 않은 호스트가 `new URL`에서 예외 발생할 수 있으나 별도 데이터 영역 오류는 없습니다. |
| 보장·비보장 | Site 복사 기반 출력은 생기지만 요청 headers를 기준 배포 환경 기준로 신뢰하며 템플릿·배포 환경 검색 노출 규칙이 없습니다. `./` 기준은 라우트 전용 식별 정보도 표현하지 않습니다. |
| 후속 연결 | `55b6061e0052`가 메타데이터 구축을 입력에만 의존하는 생성 함수로 분리하고, `67aabeab1553`이 배포 출처 URL을 검증된 `SITE_URL`로 이전합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 1f4f93ad9a0f — src/app/layout.tsx — generateMetadata
const host = requestHeaders.get("x-forwarded-host") ?? requestHeaders.get("host") ?? "localhost:3100";
const protocol = requestHeaders.get("x-forwarded-proto") ??
  (host.startsWith("localhost") || host.startsWith("127.0.0.1") ? "http" : "https");
const metadataBase = new URL(`${protocol}://${host}`);
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`55b6061e0052`가 메타데이터 구축을 입력에만 의존하는 생성 함수로 분리하고, `67aabeab1553`이 배포 출처 URL을 검증된 `SITE_URL`로 이전합니다.
<!-- learner:end commit-1f4f93ad9a0f -->


### `55b6061e0052` — feat(seo): 콘텐츠 mode별 metadata 정책 추가

- **중요도:** A
- **태그:** CONTENT, SEO
- **확정된 역할:** 콘텐츠 모드에만 의존하는 사이트 메타데이터 생성 함수 분리

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/lib/site-metadata.ts`의 `createPortfolioMetadata` 입력·출력을 확인합니다.
- Relative 소셜 이미지를 `metadataBase`로 절대 URL로 만드는 브랜치를 확인합니다.
- `mode === "production"`만 index/follow를 활성화하는 결정을 추적합니다.
- Factory가 아직 최상위 레이아웃에 연결되지 않은 통합 상태를 기록합니다.

확인 원칙:

- `55b6061e0052^`와 `55b6061e0052`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-55b6061e0052 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 최상위 레이아웃 안에 출처 URL 해석, 메타데이터 형식, 소셜 이미지 정규화이 결합되어 모드 규칙을 독립적으로 검증하거나 robots 규칙과 공유하기 어려웠습니다. |
| 실제 변경 파일·심볼·호출 경로 | 새 `src/lib/site-metadata.ts`가 `createPortfolioMetadata({metadataBase, mode, site})`를 정의합니다. 소셜 이미지는 `new URL(site.socialImage, metadataBase)`로 정규화하고 템플릿·배포 환경 robots 지시문을 반환합니다. |
| 데이터·상태·소유 주체 | 순수 생성 함수가 사이트 수준 메타데이터 형식과 검색 노출 설정값을 소유합니다. 호출자는 이미 결정된 모드와 URL을 주입해야 합니다. |
| 실패·누락·대체 처리 | 소셜 이미지가 없으면 이미지는 `undefined`입니다. 유효하지 않은 기본·이미지 조합은 URL 생성자가 예외를 던집니다. 지원하지 않는 모드는 앞선 해석 함수 책임입니다. |
| 보장·비보장 | 부작용 없이 같은 결과를 내는 규칙은 생겼지만 이 커밋에서는 레이아웃 호출자가 아직 이전되지 않아 애플리케이션 출력이 자동으로 바뀌지 않습니다. 기준 URL은 여전히 `./`입니다. |
| 후속 연결 | `67aabeab1553`이 최상위 레이아웃을 생성 함수에 연결하고 배포 출처 URL을 배포 준비 상태 검증 함수가 소유하게 합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 55b6061e0052 — src/lib/site-metadata.ts — createPortfolioMetadata
const socialImage = site.socialImage
  ? new URL(site.socialImage, metadataBase).toString()
  : undefined;
const shouldIndex = mode === "production";
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`67aabeab1553`이 최상위 레이아웃을 생성 함수에 연결하고 배포 출처 URL을 배포 준비 상태 검증 함수가 소유하게 합니다.
<!-- learner:end commit-55b6061e0052 -->


### `67aabeab1553` — feat(seo): layout metadata를 콘텐츠 mode에 연결

- **중요도:** A
- **태그:** CONTENT, SEO
- **확정된 역할:** 최상위 메타데이터 생성을 모드별 규칙과 검증된 배포 출처 URL으로 이전

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`에서 인라인 메타데이터 객체가 제거되고 `createPortfolioMetadata` 호출로 대체되는 변경 내용을 확인합니다.
- 배포 환경 브랜치의 `resolveProductionSiteUrl(process.env.SITE_URL)`와 템플릿 브랜치의 헤더에서 계산한 URL을 비교합니다.
- 모드 해석 함수와 출처 URL 해석 함수가 예외 발생하는 실패가 `generateMetadata`까지 전파되는 경로를 확인합니다.
- 전체 집계 객체 배포 준비 상태가 아니라 모드·공개 출처 URL 도우미 함수를 직접 소비한다는 비보장을 기록합니다.

확인 원칙:

- `67aabeab1553^`와 `67aabeab1553`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-67aabeab1553 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 부작용 없는 메타데이터 규칙은 있었지만 최상위 출력은 여전히 이전 인라인 구현을 사용했고 배포 환경 기준 출처 URL도 요청 headers에 의존했습니다. |
| 실제 변경 파일·심볼·호출 경로 | `generateMetadata`가 모드를 해석합니다. 배포 환경은 `resolveProductionSiteUrl(SITE_URL)`, 템플릿은 요청 headers로 `metadataBase`를 만든 뒤 `createPortfolioMetadata({metadataBase, mode, site})`를 반환합니다. |
| 데이터·상태·소유 주체 | Publication 모드·공개 출처 URL 검증은 `content-readiness.ts`, 메타데이터 형식은 `site-metadata.ts`, 프레임워크 공개는 레이아웃이 소유하는 분리가 완성됩니다. |
| 실패·누락·대체 처리 | 배포 환경에서 SITE_URL이 누락됐거나 로컬·예약 호스트인이면 `PortfolioReadinessError`가 전파됩니다. Template은 로컬 미리보기를 위해 헤더에서 계산한 대체 처리를 유지합니다. |
| 보장·비보장 | 배포 환경 기준 기본값은 요청 호스트 spoofing에서 분리됩니다. 그러나 이 실행 시점 경로는 `validateProductionReadiness` 전체를 호출하지 않아 자산·연락처 완전성은 빌드 전 검사가 별도로 보장합니다. |
| 후속 연결 | `1c40645caead`가 이 최상위 규칙 위에 라우트 전용 기준 URL·제목 생성 함수를 추가합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`1c40645caead`가 이 최상위 규칙 위에 라우트 전용 기준 URL·제목 생성 함수를 추가합니다.
<!-- learner:end commit-67aabeab1553 -->


### `1c40645caead` — feat(seo): route별 검색 metadata 정책 추가

- **중요도:** A
- **태그:** ARCH, ROUTING, SEO
- **확정된 역할:** 라우트 메타데이터 식별 규칙 공통화

#### 해당 SHA에서 확인할 실제 코드

- `RouteMetadataInput`, `routeTitle`, `createRouteMetadata`의 정확한 타입·분기를 확인합니다.
- 최상위 `path === "/"`와 비관리자 제목 조립을 비교합니다.
- 기준, Open Graph URL, Twitter/Open Graph 이미지가 어떤 상대 값을 유지하는지 확인합니다.
- 같은 커밋에서 최상위 기준 `./ → /`와 Open Graph 최상위 URL이 보정되는 이유를 기록합니다.

확인 원칙:

- `1c40645caead^`와 `1c40645caead`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-1c40645caead -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 최상위 메타데이터는 모드에 따라 동작하는했지만 모든 라우트가 같은 사이트 제목·설명·기준을 상속해 페이지별 검색 식별 정보가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `src/lib/site-metadata.ts`에 타입이 지정된 `RouteMetadataInput`, `routeTitle`, `createRouteMetadata`가 추가됩니다. Factory는 기준 경로, 라우트 설명, OG/Twitter 제목·이미지·URL을 반환합니다. |
| 데이터·상태·소유 주체 | Factory가 라우트 메타데이터 형식과 제목 관례를 소유합니다. App 라우트는 경로와 기준이 되는 콘텐츠 필드만 선택합니다. |
| 실패·누락·대체 처리 | 선택적 `type`은 `website`로 기본값하고 소셜 이미지가 없으면 이미지 배열을 생략합니다. 실행 시점 경로 존재 여부는 호출자와 콘텐츠 검증 책임입니다. |
| 보장·비보장 | 쿼리가 없는 경로 식별 정보와 일관된 소셜 메타데이터를 보장하지만 절대 해석은 부모 커밋 `metadataBase`와 Next 메타데이터 병합에 의존합니다. 언어별 대체 URL, 페이지 나누기, 라우트별 robots는 없습니다. |
| 후속 연결 | 이어지는 세 커밋이 모든 현재 공개 라우트 공개를 생성 함수에 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 1c40645caead — src/lib/site-metadata.ts — createRouteMetadata
const resolvedTitle = path === "/" ? site.title : `${title} | ${site.brand}`;
return {
  alternates: { canonical: path },
  openGraph: { description, images, title: resolvedTitle, type, url: path },
  title: resolvedTitle,
};
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

이어지는 세 커밋이 모든 현재 공개 라우트 공개를 생성 함수에 연결합니다.
<!-- learner:end commit-1c40645caead -->


### `844ff4d7abcb` — feat(seo): 홈과 프로젝트 route metadata 연결

- **중요도:** B
- **태그:** ROUTING, SEO
- **확정된 역할:** 공통 규칙을 홈·프로젝트 목록·동적 프로젝트 상세에 적용

#### 해당 SHA에서 확인할 실제 코드

- `src/app/page.tsx`, `src/app/projects/page.tsx`, `src/app/projects/[projectId]/page.tsx`의 `generateMetadata`를 확인합니다.
- 프로젝트 상세에서 awaited 매개변수, `getProjectById`, 페이지 활성화 검사, `notFound()` 순서를 추적합니다.
- 각 라우트가 제목·설명·타입에 선택하는 콘텐츠 필드를 표로 기록합니다.

확인 원칙:

- `844ff4d7abcb^`와 `844ff4d7abcb`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-844ff4d7abcb -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 공용 라우트 생성 함수는 있었지만 실제 App Router 공개 항목이 없어 공개 페이지가 라우트 식별 정보를 사용하지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 홈은 사이트 복사, 프로젝트는 프로젝트 소개 영역, 상세는 matched 프로젝트 제목·요약과 `article` 타입을 `createRouteMetadata`에 전달합니다. |
| 데이터·상태·소유 주체 | 페이지 모듈이 라우트 사용 가능 여부와 레코드 조회를 소유하고 생성 함수가 메타데이터 형식을 소유합니다. 동적 매개변수는 메타데이터 함수에서 await됩니다. |
| 실패·누락·대체 처리 | 프로젝트 페이지가 비활성화된이거나 상세 프로젝트가 누락된이면 `notFound()`가 메타데이터 생성에서도 실행됩니다. |
| 보장·비보장 | 핵심 3 라우트의 연결을 보장하지만 소개·이력서·연락처·여정·인터뷰 라우트는 아직 최상위 메타데이터만 상속합니다. |
| 후속 연결 | `fd5ff532bfe9`와 `5632c5df9b47`이 나머지 공개 라우트를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`fd5ff532bfe9`와 `5632c5df9b47`이 나머지 공개 라우트를 연결합니다.
<!-- learner:end commit-844ff4d7abcb -->


### `fd5ff532bfe9` — feat(seo): 프로필 route metadata 연결

- **중요도:** B
- **태그:** ROUTING, SEO
- **확정된 역할:** 소개·연락처·이력서에 라우트 메타데이터 적용

#### 해당 SHA에서 확인할 실제 코드

- 각 페이지의 활성화 여부 키와 메타데이터 원본 필드를 확인합니다.
- 소개 요약, 연락처 도입부·제목, 이력서 소개 영역 본문·제목이 생성 함수 입력으로 전달되는지 추적합니다.
- 비활성 페이지가 메타데이터 공개에서 `notFound()`되는지 확인합니다.

확인 원칙:

- `fd5ff532bfe9^`와 `fd5ff532bfe9`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-fd5ff532bfe9 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 홈·프로젝트 라우트만 라우트 전용 식별 정보를 제공해 프로필 관련 페이지는 일반적인 사이트 메타데이터로 남았습니다. |
| 실제 변경 파일·심볼·호출 경로 | 소개, 연락처, 이력서 페이지가 각각 sync `generateMetadata()`를 공개하고 `getPortfolioContent`, `isSitePageEnabled`, `createRouteMetadata`를 연결합니다. |
| 데이터·상태·소유 주체 | 각 라우트가 자기 콘텐츠 원본과 페이지 사용 가능 여부를 선택합니다. 공용 생성 함수는 형식 변환만 담당합니다. |
| 실패·누락·대체 처리 | 해당 페이지 설정값이 false면 메타데이터 생성 시점에 `notFound()`합니다. 콘텐츠 필드가 스키마에 맞는하다는 전제는 로더 구분 지점에서 옵니다. |
| 보장·비보장 | 세 프로필 라우트의 식별 정보는 추가되지만 여정·인터뷰는 아직 남습니다. |
| 후속 연결 | `5632c5df9b47`가 마지막 두 근거 라우트를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`5632c5df9b47`가 마지막 두 근거 라우트를 연결합니다.
<!-- learner:end commit-fd5ff532bfe9 -->


### `5632c5df9b47` — feat(seo): 여정과 근거 route metadata 연결

- **중요도:** B
- **태그:** ROUTING, SEO
- **확정된 역할:** 여정과 인터뷰 근거까지 라우트 메타데이터 적용 완료

#### 해당 SHA에서 확인할 실제 코드

- `src/app/journey/page.tsx`와 `src/app/interview-map/page.tsx`의 메타데이터 공개 항목을 확인합니다.
- 여정 설명 도입부와 인터뷰 맵 도입부가 설명 소유 주체인지 확인합니다.
- 페이지 사용 가능 여부 설정값과 정확한 기준 경로를 기록합니다.

확인 원칙:

- `5632c5df9b47^`와 `5632c5df9b47`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-5632c5df9b47 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 여정과 인터뷰 근거 라우트만 일반적인 사이트 식별 정보를 사용해 공개 라우트 검증 범위가 불완전했습니다. |
| 실제 변경 파일·심볼·호출 경로 | 두 페이지 모듈이 콘텐츠 도입부와 화면 구성 소개 영역 제목을 선택해 `/journey`, `/interview-map` 메타데이터를 만듭니다. |
| 데이터·상태·소유 주체 | 라우트 페이지가 활성화 여부와 원본 선택을 소유하고 생성 함수가 기준·소셜 형식을 재사용합니다. |
| 실패·누락·대체 처리 | 비활성 라우트는 `notFound()`합니다. 누락된 콘텐츠는 앞선 스키마·콘텐츠 검증에서 차단됩니다. |
| 보장·비보장 | 현재 활성화된 공개 라우트 집합의 메타데이터 공개 항목이 완성됩니다. 실제 렌더링된 최종 커밋와 절대 기준 직렬화는 아직 브라우저 테스트로 검증하지 않았습니다. |
| 후속 연결 | `4358bcd34f2e`가 페이지 모듈의 실제 공개 항목을 직접 호출해 연결 회귀를 고정합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`4358bcd34f2e`가 페이지 모듈의 실제 공개 항목을 직접 호출해 연결 회귀를 고정합니다.
<!-- learner:end commit-5632c5df9b47 -->


### `4358bcd34f2e` — test(seo): route metadata export 검증

- **중요도:** B
- **태그:** VALIDATION, ROUTING, SEO
- **확정된 역할:** 모든 공개 라우트가 실제로 내보내는 메타데이터를 현재 동작으로 고정

#### 해당 SHA에서 확인할 실제 코드

- `src/app/route-metadata.test.ts`가 도우미 함수가 아니라 각 페이지의 `generateMetadata`를 불러오는지 확인합니다.
- `it.each` 라우트 조합표와 동적 프로젝트 상세 경우를 구분합니다.
- 기준 URL·제목·설명 단언문과 설정 시간 첫 번째 프로젝트 requirement를 확인합니다.
- Rendered HTML, 비활성 라우트 동작, 절대 메타데이터 URL을 증명하지 않는 범위를 기록합니다.

확인 원칙:

- `4358bcd34f2e^`와 `4358bcd34f2e`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-4358bcd34f2e -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Factory 구현은 연결됐지만 라우트 모듈이 잘못된 경로·콘텐츠를 전달하는 연결 회귀를 도우미 함수 단위 테스트만으로 잡을 수 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 테스트가 8개 라우트 모듈의 `generateMetadata`를 별칭으로 가져오고 정적 라우트 표와 첫 프로젝트 상세 호출을 실행합니다. |
| 데이터·상태·소유 주체 | 체크인된 검증 콘텐츠가 예상한 제목·설명 소유 주체입니다. 테스트는 변경 없이 실제 공개 항목의 반환 객체를 관찰합니다. |
| 실패·누락·대체 처리 | 프로젝트 테스트 입력이 없으면 설정에서 명시적 오류를 발생시킵니다. 정적 조합표는 기준 경로, 제목 포함, truthy 설명을 검사합니다. |
| 보장·비보장 | 라우트와 생성 함수 연결과 동적 프로젝트 콘텐츠 선택을 증명합니다. Next 메타데이터 최종 렌더링, 메타데이터 병합, 절대 URL, 비활성화된 경로, Open Graph/Twitter 전체 형식은 증명하지 않습니다. |
| 후속 연결 | 사이트맵·라우트 도우미 함수의 추가 규칙은 `adc392157f70`에서 검색 노출 개발 과정 관점으로 검증됩니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 4358bcd34f2e — src/app/route-metadata.test.ts
it.each(routeCases)("provides content metadata for %s", async (path, title, getMetadata) => {
  const metadata = await getMetadata();
  expect(metadata.alternates).toEqual({ canonical: path });
  expect(String(metadata.title)).toContain(title);
  expect(metadata.description).toBeTruthy();
});
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

사이트맵·라우트 도우미 함수의 추가 규칙은 `adc392157f70`에서 검색 노출 개발 과정 관점으로 검증됩니다.
<!-- learner:end commit-4358bcd34f2e -->


## 6. 불변 조건 변화

<!-- learner:start thread-invariant-evolution -->
| Commit·구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| 1f4f93ad9a0f | 도입됨 | 콘텐츠가 사이트 제목·설명·소셜 출력을 정하지만 출처 URL은 요청에서 계산합니다. |
| 55b6061e0052 | 한곳으로 모음 | 입력에만 의존하고 모드에 따라 동작하는 생성 함수가 최상위 메타데이터 형식을 담당합니다. |
| 67aabeab1553 | 수정·통합됨 | 배포 출처 URL은 검증된 `SITE_URL`이 정하고, 템플릿 모드는 현재 요청 출처 URL을 미리보기 대체값으로 유지합니다. |
| 1c40645caead | 확장됨 | 기준 URL·제목·소셜 식별 정보가 현재 라우트를 반영하며 쿼리를 포함하지 않습니다. |
| 844ff4d7abcb → 5632c5df9b47 | 통합됨 | 모든 공개 페이지가 라우트별 메타데이터를 제공합니다. |
| 4358bcd34f2e | 검증됨 | 실제 페이지의 메타데이터 진입점을 호출해 연결 상태를 회귀 테스트로 고정합니다. |
<!-- learner:end thread-invariant-evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:start thread-failure-fix-test -->
| 실패·위험 | 수정·결정 | 테스트·증거 |
| --- | --- | --- |
| 요청에서 계산한 출처 URL이 배포 환경의 사이트 식별 정보가 될 수 있음 | 배포 환경 분기가 검증된 `SITE_URL`을 해석한 뒤 입력에만 의존하는 생성 함수를 호출함 | 생성 함수·배포 준비 상태 테스트가 공개 출처 URL 규칙을 검증하고 라우트 테스트가 페이지 연결을 확인함 |
| 모든 라우트가 하나의 일반 기준 URL과 제목을 상속함 | 공용 라우트 생성 함수와 페이지별 공개 항목을 분리함 | 라우트·공개 조합표가 모든 라우트의 기준 URL·제목·설명을 검사함 |
| 동적 프로젝트 메타데이터가 페이지 사용 가능 여부와 어긋날 수 있음 | 메타데이터 생성도 페이지 활성화 여부와 프로젝트 조회 함수를 재사용함 | 프로젝트 상세 테스트가 실제 프로젝트 ID로 비동기 메타데이터 생성을 호출함 |
<!-- learner:end thread-failure-fix-test -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:start thread-ownership -->
| 시점 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 이전 | 최상위 레이아웃의 인라인 함수 | 헤더, 출처 URL, 메타데이터 형식과 사이트 콘텐츠 대응을 한곳에서 처리합니다. |
| 55b | `site-metadata.ts` | 입력에만 의존하는 최상위 메타데이터 규칙을 담당하며, 호출자는 검증된 입력을 선택해야 합니다. |
| 67a | 준비 상태 해석 함수 + 레이아웃 + 메타데이터 생성 함수 | 출처 URL 결정, 프레임워크 연결과 출력 형식 작성을 각각 분리합니다. |
| 1c 이후 | 페이지 모듈 + 공용 생성 함수 | 페이지는 라우트·콘텐츠·사용 가능 여부를, 생성 함수는 기준 URL과 소셜 메타데이터 형식 변환을 담당합니다. |
<!-- learner:end thread-ownership -->

## 9. 최종 개발 과정 상태와 실행 순서

<!-- learner:start thread-final-state -->
**최종 상태**

배포 환경의 사이트 식별 정보는 검증된 `SITE_URL`을 기준으로 만들고, 템플릿 미리보기에서는 현재 요청 출처 URL을 사용할 수 있습니다. 각 공개 라우트는 자신에게 맞는 제목과 설명을 선택하고, 쿼리가 없는 기준 경로와 일관된 소셜 메타데이터를 출력합니다. 비활성화되었거나 누락된 콘텐츠는 해당 페이지 진입점에서 거부합니다.

**코드 없는 실행 순서**

1. 최상위 레이아웃이 콘텐츠 모드를 해석하고, 검증된 배포 출처 URL 또는 요청에서 계산한 템플릿 미리보기 출처 URL을 선택합니다.
2. `createPortfolioMetadata`가 해당 입력으로 사이트 제목·설명·이미지와 검색 노출 지시문을 만듭니다.
3. 각 App Router 페이지가 사용 가능 여부를 검사하고 라우트별 콘텐츠를 선택한 뒤, 기준 경로 하나를 `createRouteMetadata`에 전달합니다.
4. Next가 라우트 메타데이터와 최상위 `metadataBase`를 결합하므로 상대 기준 경로와 소셜 경로는 선택한 출처 URL을 기준으로 해석됩니다.
5. 집중 테스트는 실제 페이지의 공개 항목을 불러와 기준 URL·제목·설명을 같은 기준 콘텐츠와 비교합니다.
<!-- learner:end thread-final-state -->

## 10. 학습 완료 확인

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 정확한 변경 내용·파일 트리를 GitHub 연결 도구로 정적 확인했습니다.
- [x] 보장과 비보장을 커밋별로 구분했습니다.
- [x] 테스트 방법과 증명 범위와 증명하지 않는 범위를 구분했습니다.
- [x] 최종 처리 순서를 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 03-route-aware-metadata-and-canonical-identity.md =====

===== BEGIN FILE: 04-indexing-robots-and-sitemap-policy.md =====
# 개발 과정: 검색 노출·robots·사이트맵 규칙

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `06-seo-security-and-machine-readable-output`
>
> 1단계 검토에서 확정한 구조입니다. 2단계는 이 문서의 고정 항목과 커밋 순서를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- 저장소·브랜치 범위는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md`의 `web/portfolio`는 이 브랜치를 `cce7dd020563`부터 `aff0acdd4cf9`까지 이어지는 독립적인 선형 476개 커밋 이력으로 설명합니다. 아래 SHA는 모두 브랜치 내부 분류와 해당 커밋 객체·변경 내용에 대조했습니다.
- Subject, 중요도, 태그는 브랜치 내부 원본 분류와 일치시켰습니다.
- 아래 역할, 확인 대상, 불변 조건은 1단계 분류 검토에서 저장소 근거에 맞춰 동결했습니다.
- 다른 브랜치 또는 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표

템플릿 미리보기를 검색 수집기에 노출되는 표시 영역에서 실패 시 차단하는 방식으로 유지하고, 배포 모드에서만 페이지 robots, `robots.txt`, 기준 호스트, sitemap 탐색 가능 여부와 활성화된 라우트·프로젝트 URL을 일관되게 공개하는 규칙을 복원합니다.

### 1단계 범위 결정

기존 초안은 핵심 커밋을 포함했지만 최상위 메타데이터 소비자, 단위 회귀, 실행 중인 애플리케이션 회귀가 빠져 있었습니다. 1단계에서는 `67aabeab1553`, `fb3d18fd660b`, `166f05f7be06`을 추가하고, `adc392157f70`을 메타데이터 개발 과정에서 이동해 검색 수집기 출력의 구현→통합→테스트 순서를 완성했습니다.

### 고정된 핵심 불변 조건

- 템플릿 모드는 페이지 메타데이터에서 `noindex,nofollow`, `robots.txt`에서 `Disallow: /`, 사이트맵에서 빈 목록입니다.
- 배포 환경 모드만 index/follow와 `Allow: /`를 내며, 호스트와 sitemap URL은 검증된 공개 출처 URL에서 계산됩니다.
- 사이트맵은 최상위와 활성화된 페이지, 프로젝트 페이지가 활성화되어 있을 때의 활성화된 프로젝트 `<details>` 요소만 포함합니다.
- 검색 수집기 규칙은 콘텐츠 모드를 추측하지 않고 정해진 환경 해석 함수를 사용합니다.
- 단위 테스트와 running-app E2E가 도우미 함수 객체와 직렬화된 HTTP 출력을 서로 다른 수준에서 검증합니다.

### 주요 기술적 어려움

- 페이지별 robots 지시문, 독립 실행형 robots 라우트, sitemap 라우트가 같은 공개 결정을 공유하도록 만드는 문제
- 템플릿 미리보기가 실수로 검색에 노출될 수 있는 상태해지는 반대 방향의 회귀를 막는 문제
- 콘텐츠 페이지 설정값과 필터링된 프로젝트 모델을 기계 판독용 라우트 목록으로 변환하는 문제

## 2. 핵심 질문

- 메타데이터 생성 함수, robots 생성 함수, sitemap 생성 함수는 템플릿·배포 환경에서 각각 무엇을 반환합니까?
- 배포 환경 robots 출력이 사이트 URL이 없을 때 왜 예외 발생하며 호스트·사이트맵을 어떻게 계산합니까?
- 최상위 레이아웃 통합과 running-app E2E는 입력에만 의존하는 단위 테스트보다 무엇을 추가로 증명합니까?
- 사이트맵 라우트 순서 결정과 페이지·프로젝트 활성화 여부 조건은 무엇입니까?
- 현재 테스트가 robots 호스트, sitemap XML 직렬화, 기준 최종 커밋를 어디까지 검증하지 못합니까?

## 3. 완료 기준

- 메타데이터, robots, 사이트맵의 모드 조합표를 하나의 표로 정리했습니다.
- 도우미 함수 규칙과 App Router 라우트 통합을 별도 소유 주체로 설명했습니다.
- `166f05f7be06`의 브라우저·요청 방법과 `fb3d18fd660b`의 입력에만 의존하는 단위 방법을 구분했습니다.
- `70b69f04e8c7 → adc392157f70`의 활성 라우트 sitemap 구현과 결정적인 입력 검증 지점 테스트를 연결했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `55b6061e0052` | feat(seo): 콘텐츠 mode별 metadata 정책 추가 | A | CONTENT, SEO | 콘텐츠 모드에서 페이지별 검색 노출 지시문 생성 |
| 2 | `cb61450ad922` | feat(seo): 콘텐츠 mode별 robots 정책 추가 | A | CONTENT, SEO | 모드별 robots 생성 함수와 App Router robots 라우트 추가 |
| 3 | `67aabeab1553` | feat(seo): layout metadata를 콘텐츠 mode에 연결 | A | CONTENT, SEO | 페이지 검색 노출 지시문을 실제 최상위 메타데이터 출력에 연결 |
| 4 | `fb3d18fd660b` | test(content): readiness와 indexing 계약 검증 | A | CONTENT, VALIDATION, SEO | 메타데이터와 robots의 모드별 동작을 단위 테스트 |
| 5 | `166f05f7be06` | test(e2e): 콘텐츠 mode별 metadata와 robots 검증 | A | CONTENT, VALIDATION, SEO | 실행 중인 애플리케이션에서 검색 노출 규칙 검증 |
| 6 | `70b69f04e8c7` | feat(seo): 공개 route sitemap 생성 | A | ARCH, ROUTING, SEO | 검증된 공개 화면에서 사이트맵 탐색 URL 생성 |
| 7 | `adc392157f70` | test(seo): route metadata와 sitemap 계약 검증 | B | VALIDATION, ROUTING, SEO | 사이트맵 공개 범위와 라우트 URL 순서를 회귀 테스트로 고정 |

## 5. 커밋별 학습 기록

### `55b6061e0052` — feat(seo): 콘텐츠 mode별 metadata 정책 추가

- **중요도:** A
- **태그:** CONTENT, SEO
- **확정된 역할:** 콘텐츠 모드에서 페이지별 검색 노출 지시문 생성

#### 해당 SHA에서 확인할 실제 코드

- `createPortfolioMetadata`의 `shouldIndex`와 `robots` 결과를 확인합니다.
- Template·배포 환경 두 상태 외 값을 생성 함수가 받지 않도록 타입 구분 지점을 확인합니다.
- 이 커밋은 도우미 함수 only이며 최상위 레이아웃 출력에 아직 연결되지 않았다는 점을 기록합니다.

확인 원칙:

- `55b6061e0052^`와 `55b6061e0052`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-55b6061e0052 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 콘텐츠 기반 메타데이터는 있었지만 초기 예시·템플릿 식별 정보가 검색 엔진에 노출되는 것을 막는 명시적인 페이지 지시문이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `createPortfolioMetadata`가 `mode === "production"`을 `shouldIndex`로 계산해 `{follow, index}`를 반환합니다. |
| 데이터·상태·소유 주체 | 부작용 없는 메타데이터 생성 함수가 페이지별 검색 수집기 지시문을 소유하고 모드 선택은 앞선 해석 함수가 소유합니다. |
| 실패·누락·대체 처리 | Template은 false/false, 배포 환경은 true/true입니다. 지원하지 않는 모드는 해석 함수 단계에서 실패해야 하며 생성 함수에는 대체 처리 브랜치가 없습니다. |
| 보장·비보장 | 규칙 객체만 정의하며 애플리케이션 최종 커밋나 `robots.txt`는 아직 바뀌지 않습니다. |
| 후속 연결 | `cb61450ad922`가 독립 실행형 robots 규칙을 만들고 `67aabeab1553`이 페이지 지시문을 최상위 출력에 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 55b6061e0052 — src/lib/site-metadata.ts
const shouldIndex = mode === "production";
robots: { follow: shouldIndex, index: shouldIndex },
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`cb61450ad922`가 독립 실행형 robots 규칙을 만들고 `67aabeab1553`이 페이지 지시문을 최상위 출력에 연결합니다.
<!-- learner:end commit-55b6061e0052 -->


### `cb61450ad922` — feat(seo): 콘텐츠 mode별 robots 정책 추가

- **중요도:** A
- **태그:** CONTENT, SEO
- **확정된 역할:** 모드별 robots 생성 함수와 App Router robots 라우트 추가

#### 해당 SHA에서 확인할 실제 코드

- `createRobots({mode, siteUrl})`의 템플릿 즉시 반환과 배포 모드에서 URL 누락 시 예외 발생을 확인합니다.
- 배포 환경 결과의 `host: siteUrl.origin`과 허용 규칙을 확인합니다.
- `src/app/robots.ts`가 환경 모드와 URL을 어떻게 해석하는지 추적합니다.
- 이 시점 robots 결과에는 sitemap 필드가 아직 없음을 기록합니다.

확인 원칙:

- `cb61450ad922^`와 `cb61450ad922`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-cb61450ad922 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 페이지 meta 지시문만으로 검색 수집기가 직접 요청하는 `/robots.txt`의 사이트 전체 규칙을 통제하지 못했습니다. |
| 실제 변경 파일·심볼·호출 경로 | `src/lib/site-metadata.ts`에 `createRobots`가, `src/app/robots.ts`에 Next `MetadataRoute.Robots` 공개가 추가됩니다. |
| 데이터·상태·소유 주체 | 생성 함수가 모드별 robots 객체 매핑을, App Router 경로가 환경 변수 해석과 프레임워크 진입점을 소유합니다. |
| 실패·누락·대체 처리 | Template은 사이트 URL 없이 전체 차단합니다. 배포 환경인데 URL이 없으면 오류를 예외 발생하며 silent 상대 호스트로 대체 처리하지 않습니다. |
| 보장·비보장 | Template·배포 환경 robots 규칙과 기준 호스트는 생기지만 sitemap 탐색 가능 여부는 아직 없습니다. Full 배포 준비 상태를 호출하지 않고 검증된 URL 도우미 함수를 직접 사용합니다. |
| 후속 연결 | `67aabeab1553`가 페이지 메타데이터 경로를 실제 최상위 레이아웃에 연결하고 테스트가 두 출력을 함께 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// cb61450ad922 — src/lib/site-metadata.ts — createRobots
if (mode === "template") return { rules: { disallow: "/", userAgent: "*" } };
if (!siteUrl) throw new Error("A production site URL is required to create robots.txt.");
return { host: siteUrl.origin, rules: { allow: "/", userAgent: "*" } };
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`67aabeab1553`가 페이지 메타데이터 경로를 실제 최상위 레이아웃에 연결하고 테스트가 두 출력을 함께 검증합니다.
<!-- learner:end commit-cb61450ad922 -->


### `67aabeab1553` — feat(seo): layout metadata를 콘텐츠 mode에 연결

- **중요도:** A
- **태그:** CONTENT, SEO
- **확정된 역할:** 페이지 검색 노출 지시문을 실제 최상위 메타데이터 출력에 연결

#### 해당 SHA에서 확인할 실제 코드

- 최상위 레이아웃이 모드에 따라 동작하는 생성 함수를 호출한 뒤 반환된 robots 지시문이 실제 메타데이터 출력에 포함되는지 확인합니다.
- 배포 환경·템플릿 metadataBase 브랜치와 검색 노출 브랜치가 같은 해석된 모드를 소비하는지 확인합니다.
- Robots 라우트와 최상위 레이아웃이 해석 함수를 각각 호출해도 동일 env 규칙을 공유한다는 점을 기록합니다.

확인 원칙:

- `67aabeab1553^`와 `67aabeab1553`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-67aabeab1553 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 페이지 검색 노출 규칙은 입력에만 의존하는 도우미 함수에만 있었기 때문에 실제 최상위 메타데이터에는 적용되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | `src/app/layout.tsx`가 모드와 metadataBase를 결정한 뒤 `createPortfolioMetadata`의 결과를 프레임워크 메타데이터 공개로 반환합니다. |
| 데이터·상태·소유 주체 | 최상위 레이아웃이 프레임워크 연결을 담당하고 생성 함수가 메타데이터 규칙을 담당합니다. 최상위 라우트와 robots 라우트는 같은 환경 모드 규칙을 각각 읽습니다. |
| 실패·누락·대체 처리 | 배포 환경 URL 검증 실패는 메타데이터 생성을 실패시킵니다. Template은 요청 파생된 출처 URL이어도 noindex/nofollow로 남습니다. |
| 보장·비보장 | 실제 페이지 메타데이터 경로에 규칙이 연결됩니다. Robots 진입점와의 실행 시점 일관성은 아직 테스트로 보호되지 않습니다. |
| 후속 연결 | `fb3d18fd660b`가 입력에만 의존하는 규칙을, `166f05f7be06`이 running 애플리케이션의 meta/robots 진입점을 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`fb3d18fd660b`가 입력에만 의존하는 규칙을, `166f05f7be06`이 running 애플리케이션의 meta/robots 진입점을 검증합니다.
<!-- learner:end commit-67aabeab1553 -->


### `fb3d18fd660b` — test(content): readiness와 indexing 계약 검증

- **중요도:** A
- **태그:** CONTENT, VALIDATION, SEO
- **확정된 역할:** 메타데이터와 robots의 모드별 동작을 단위 테스트

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/site-metadata.test.ts`의 two 경우를 확인합니다.
- Template 경우가 메타데이터 robots와 `createRobots` 차단를 함께 확인하는지 추적합니다.
- 배포 환경 경우가 metadataBase, 기준, index/follow, 절대 소셜 이미지, 호스트·허용을 확인하는지 기록합니다.
- Next 직렬화가 아닌 객체 수준 테스트라는 한계를 구분합니다.

확인 원칙:

- `fb3d18fd660b^`와 `fb3d18fd660b`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-fb3d18fd660b -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 메타데이터와 robots 도우미 함수가 같은 모드 규칙을 구현했지만 한쪽만 바뀌는 불일치를 잡는 회귀가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `site-metadata.test.ts`가 동일 사이트 테스트 입력과 모드를 두 생성 함수에 전달해 템플릿·배포 산출물 쌍을 비교합니다. |
| 데이터·상태·소유 주체 | 테스트 owns 명시적인 URL 테스트 입력 및 observes 입력에만 의존하는 반환 objects; 네트워크·프레임워크 레이어는 개입하지 않습니다. |
| 실패·누락·대체 처리 | Template noindex/nofollow + 차단, 배포 환경 index/follow + 허용·호스트 및 절대 소셜 이미지를 고정합니다. |
| 보장·비보장 | 규칙 alignment를 결정적으로 증명하지만 출력된 `<meta>` 문자열, `/robots.txt` 문구, App Router 통합, 사이트맵은 증명하지 않습니다. |
| 후속 연결 | `166f05f7be06`이 running 서버를 통해 직렬화된 출력을 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`166f05f7be06`이 running 서버를 통해 직렬화된 출력을 검증합니다.
<!-- learner:end commit-fb3d18fd660b -->


### `166f05f7be06` — test(e2e): 콘텐츠 mode별 metadata와 robots 검증

- **중요도:** A
- **태그:** CONTENT, VALIDATION, SEO
- **확정된 역할:** 실행 중인 애플리케이션에서 검색 노출 규칙 검증

#### 해당 SHA에서 확인할 실제 코드

- `tests/e2e/portfolio.spec.ts`의 added Playwright 테스트를 확인합니다.
- `page.goto('/')`, robots meta locator, API 요청 `/robots.txt`의 서로 다른 경로를 추적합니다.
- 예상한 모드가 프로세스 환경에서 결정되는 방식과 정규식 단언문을 확인합니다.
- 호스트·sitemap·기준까지 검증하지 않는 범위를 기록합니다.

확인 원칙:

- `166f05f7be06^`와 `166f05f7be06`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-166f05f7be06 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 부작용 없는 단위 테스트는 Next가 메타데이터 객체를 실제 HTML과 robots 문구로 serialize하고 라우트가 연결되었는지 확인하지 못했습니다. |
| 실제 변경 파일·심볼·호출 경로 | Playwright 테스트가 `/`을 방문해 `meta[name=robots]` 콘텐츠를 보고, 요청 맥락으로 `/robots.txt`를 받아 직렬화된 허용·차단 문구를 확인합니다. |
| 데이터·상태·소유 주체 | 테스트 프로세스 환경이 예상한 모드를 소유하고 실제 애플리케이션·서버 응답이 관찰 결과 대상입니다. |
| 실패·누락·대체 처리 | 홈 응답과 robots 응답이 모두 성공해야 합니다. 모드에 따라 `index/follow` 또는 `noindex/nofollow`, `allow` 또는 `disallow` 정규식이 일치해야 합니다. |
| 보장·비보장 | Framework 통합과 HTTP 직렬화를 증명합니다. 배포 환경 호스트 줄, sitemap 줄, 기준 태그, 모든 라우트, 외부 검색 수집기 동작은 증명하지 않습니다. |
| 후속 연결 | `70b69f04e8c7`가 robots 출력에 sitemap 탐색 가능 여부를 추가하고 라우트 목록을 생성합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 166f05f7be06 — tests/e2e/portfolio.spec.ts
await expect(page.locator('meta[name="robots"]')).toHaveAttribute(
  "content", isProductionContent ? /index.*follow/i : /noindex.*nofollow/i,
);
expect(await robotsResponse.text()).toMatch(
  isProductionContent ? /Allow:\s*\//i : /Disallow:\s*\//i,
);
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`70b69f04e8c7`가 robots 출력에 sitemap 탐색 가능 여부를 추가하고 라우트 목록을 생성합니다.
<!-- learner:end commit-166f05f7be06 -->


### `70b69f04e8c7` — feat(seo): 공개 route sitemap 생성

- **중요도:** A
- **태그:** ARCH, ROUTING, SEO
- **확정된 역할:** 검증된 공개 화면에서 사이트맵 탐색 URL 생성

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/app/sitemap.ts`, `absoluteSiteUrl`, `createSitemap`을 확인합니다.
- 템플릿 모드의 빈 반환, 배포 모드에서 URL 누락 시 예외 발생, 라우트 배열 구축 순서를 추적합니다.
- `content.site.pages` 설정값과 `content.projects`가 어떤 라우트를 추가·제외하는지 확인합니다.
- `createRobots` 결과에 sitemap URL이 추가되는 통합을 확인합니다.

확인 원칙:

- `70b69f04e8c7^`와 `70b69f04e8c7`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-70b69f04e8c7 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 배포 환경 robots는 허용·호스트만 제공했고 검색 엔진이 실제 활성화된 공개 대상 범위를 발견할 사이트맵이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `create사이트맵`이 모드·콘텐츠·siteUrl을 받아 최상위, 활성화된 정적 페이지, 프로젝트 목록과 콘텐츠 프로젝트 `<details>` 요소를 절대 URLs로 변환합니다. App Router sitemap 라우트가 env·콘텐츠를 연결하고 robots가 `/sitemap.xml`을 광고합니다. |
| 데이터·상태·소유 주체 | 콘텐츠 페이지 설정값과 검증된 포트폴리오 프로젝트 목록이 라우트 포함 관계를 소유합니다. Factory가 순서·URL 변환을 소유하고 라우트 모듈이 프레임워크 진입점을 소유합니다. |
| 실패·누락·대체 처리 | Template은 빈 배열, 배포 환경 누락된 URL은 예외 발생입니다. `pages.projects === false`면 목록과 모든 상세를 함께 제외합니다. 각 선택적 설정값은 명시적인 false일 때만 제외합니다. |
| 보장·비보장 | 활성 공개 URLs와 robots 탐색 가능 여부를 보장합니다. `lastModified`, 변경 frequency, 우선순위, 대체 languages는 생성하지 않습니다. XML HTTP 직렬화는 이 커밋에 테스트되지 않습니다. |
| 후속 연결 | `adc392157f70`이 템플릿 빈, 비활성화된 페이지 exclusion, 정확한 절대 URL 순서를 단위 규칙으로 고정합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`adc392157f70`이 템플릿 빈, 비활성화된 페이지 exclusion, 정확한 절대 URL 순서를 단위 규칙으로 고정합니다.
<!-- learner:end commit-70b69f04e8c7 -->


### `adc392157f70` — test(seo): route metadata와 sitemap 계약 검증

- **중요도:** B
- **태그:** VALIDATION, ROUTING, SEO
- **확정된 역할:** 사이트맵 공개 범위와 라우트 URL 순서를 회귀 테스트로 고정

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/site-metadata.test.ts`의 `describe("sitemap")` 두 경우를 확인합니다.
- 템플릿 모드 예상한 `[]`와 배포 환경 테스트 입력의 `interviewMap: false` 변경을 추적합니다.
- 예상한 URL 목록이 최상위→프로젝트→프로젝트 상세→남은 페이지 순서인지 확인합니다.
- 같은 커밋의 라우트 메타데이터 도우미 함수 테스트는 메타데이터 개발 과정의 도우미 함수 보강이며 이 개발 과정에서는 sitemap 단언문만 해석합니다.

확인 원칙:

- `adc392157f70^`와 `adc392157f70`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-adc392157f70 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 사이트맵 생성 로직과 robots 탐색 가능 여부는 구현됐지만 템플릿 leakage, 비활성화된 페이지 inclusion, 출처 URL·순서 회귀를 방지하는 집중된 테스트가 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 테스트는 실제 콘텐츠를 객체 펼치기로 복제해 `interviewMap`만 false로 바꾸고 `createSitemap`의 URL 데이터 구성을 정확한 배열로 비교합니다. |
| 데이터·상태·소유 주체 | 테스트 입력이 페이지 사용 가능 여부 변형을 소유하고 생성 함수 결과가 결정적인 공개 목록입니다. |
| 실패·누락·대체 처리 | 템플릿 모드 결과는 정확히 빈 배열이어야 하고 비활성화된 인터뷰 라우트는 배포 모드 예상 목록에 없어야 합니다. |
| 보장·비보장 | 모드, 비활성 페이지 제외, 프로젝트 상세 포함, 공개 출처 URL과 결정적인 순서를 검증합니다. 테스트 입력보다 많은 프로젝트, 렌더링된 XML, robots의 sitemap 줄이 실제 HTTP로 출력되는지는 검증하지 않습니다. |
| 후속 연결 | 이 커밋이 분류 내 sitemap 변경 과정의 마지막 집중된 회귀 검증 근거입니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// adc392157f70 — src/lib/site-metadata.test.ts
expect(sitemap.map(({ url }) => url)).toEqual([
  "https://portfolio.example.dev/",
  "https://portfolio.example.dev/projects",
  `https://portfolio.example.dev/projects/${content.projects[0]?.id}`,
  "https://portfolio.example.dev/about",
  "https://portfolio.example.dev/resume",
  "https://portfolio.example.dev/contact",
  "https://portfolio.example.dev/journey",
]);
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

이 커밋이 분류 내 sitemap 변경 과정의 마지막 집중된 회귀 검증 근거입니다.
<!-- learner:end commit-adc392157f70 -->


## 6. 불변 조건 변화

<!-- learner:start thread-invariant-evolution -->
| Commit·구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| 55b6061e0052 | Introduced | 페이지 메타데이터가 템플릿의 검색 제외와 배포 환경의 검색 허용을 구분합니다. |
| cb61450ad922 | 확장됨 | The 같은 모드 controls 사이트 전체 robots 허용·차단 및 호스트. |
| 67aabeab1553 | Integrated | 최상위 레이아웃이 실제 애플리케이션에 페이지 검색 지시문을 출력합니다. |
| fb3d18fd660b | 단위 검증된 | 메타데이터 및 robots 객체 규칙을 함께 테스트합니다. |
| 166f05f7be06 | 저장소 이력에서 실행 시점 검증됨 | Playwright가 실행 중인 애플리케이션의 메타 태그와 robots 문구를 검사했습니다. 이 문서 작성 과정에서는 다시 실행하지 않았습니다. |
| 70b69f04e8c7 | 확장됨 | robots 출력이 활성화된 배포 라우트만 담은 사이트맵의 위치를 알립니다. |
| adc392157f70 | 결정적으로 검증됨 | 템플릿의 빈 목록과 배포 환경의 활성 라우트 목록을 고정합니다. |
<!-- learner:end thread-invariant-evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:start thread-failure-fix-test -->
| 실패·위험 | 수정·결정 | 테스트·증거 |
| --- | --- | --- |
| 초기 템플릿이 검색에 노출될 수 있음 | `noindex/nofollow`, robots 차단과 빈 sitemap 적용 | 단위·E2E 테스트가 메타데이터와 robots를 검사하고 sitemap 단위 테스트가 빈 출력을 검사함 |
| 검색 수집기용 출력이 모드를 다르게 해석할 수 있음 | 모든 생성 함수가 같은 정확한 모드 값 집합을 사용함 | 한 단위 테스트 커밋이 메타데이터와 robots를 비교하고 E2E가 직렬화된 두 출력을 확인함 |
| 사이트맵이 비활성 화면을 노출할 수 있음 | 페이지 활성화 설정과 검증된 활성 프로젝트로 라우트 목록을 구성함 | 비활성 인터뷰 라우트가 정확한 예상 URL 목록에서 제외됨 |
<!-- learner:end thread-failure-fix-test -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:start thread-ownership -->
| 시점 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 이전 | 프레임워크 기본값 | 검색 수집기 공개 규칙을 명시하지 않았습니다. |
| 55b/cb | 순수 생성 함수 | 페이지·robots 규칙은 생성 함수가 담당하고 환경 연결은 라우트 모듈이 담당합니다. |
| 67a | 최상위 레이아웃 | 페이지 규칙을 실제 메타데이터 출력에 연결합니다. |
| 70b | 사이트맵 생성 함수 + 라우트 | 기계가 읽을 수 있는 공개 목록은 사이트맵이, 사이트맵 위치 안내는 robots가 담당합니다. |
| 테스트 | 단위 테스트·Playwright | 입력만으로 결정되는 객체 규칙과 프레임워크·HTTP 통합 결과를 각각 검증합니다. |
<!-- learner:end thread-ownership -->

## 9. 최종 개발 과정 상태와 실행 순서

<!-- learner:start thread-final-state -->
**최종 상태**

템플릿 미리보기는 페이지 메타데이터, `robots.txt`, 빈 사이트맵의 세 지점에서 검색 노출을 차단합니다. 배포 환경은 검증된 공개 출처 URL을 사용하고 검색 수집을 허용하며 사이트맵을 알립니다. 사이트맵에는 현재 콘텐츠에서 활성화된 라우트와 검증된 프로젝트 상세 주소만 포함합니다.

**코드 없는 실행 순서**

1. 각 기계 판독용 라우트가 환경에서 정확한 콘텐츠 모드를 해석합니다.
2. 최상위 메타데이터 생성 함수는 배포 모드에서만 `index,follow`를 출력하고 템플릿 모드에서는 `noindex,nofollow`를 출력합니다.
3. robots 라우트는 템플릿 모드에서 모든 수집을 차단하고, 배포 모드에서는 공개 출처 URL을 기준으로 허용 경로·호스트·사이트맵 URL을 출력합니다.
4. sitemap 라우트는 템플릿 모드에서 URL을 반환하지 않고, 배포 환경에서는 활성 페이지와 프로젝트를 절대 URL로 변환합니다.
5. 단위 테스트는 입력만으로 결정되는 출력 객체를 보호하고, 저장소의 Playwright 테스트는 실행 중인 애플리케이션에서 페이지 메타데이터와 robots 문구를 검사합니다.
<!-- learner:end thread-final-state -->

## 10. 학습 완료 확인

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 정확한 변경 내용·파일 트리를 GitHub 연결 도구로 정적 확인했습니다.
- [x] 보장과 비보장을 커밋별로 구분했습니다.
- [x] 테스트 방법과 증명 범위와 증명하지 않는 범위를 구분했습니다.
- [x] 최종 처리 순서를 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 04-indexing-robots-and-sitemap-policy.md =====

===== BEGIN FILE: 05-safe-jsonld-serialization-and-structured-claims.md =====
# 개발 과정: 안전한 JSON-LD 직렬화와 구조화된 정보

> 프로젝트: 42 Archive Portfolio (`web/portfolio`)
>
> 분류: `06-seo-security-and-machine-readable-output`
>
> 1단계 검토에서 확정한 구조입니다. 2단계는 이 문서의 고정 항목과 커밋 순서를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- 저장소·브랜치 범위는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md`의 `web/portfolio`는 이 브랜치를 `cce7dd020563`부터 `aff0acdd4cf9`까지 이어지는 독립적인 선형 476개 커밋 이력으로 설명합니다. 아래 SHA는 모두 브랜치 내부 분류와 해당 커밋 객체·변경 내용에 대조했습니다.
- Subject, 중요도, 태그는 브랜치 내부 원본 분류와 일치시켰습니다.
- 아래 역할, 확인 대상, 불변 조건은 1단계 분류 검토에서 저장소 근거에 맞춰 동결했습니다.
- 다른 브랜치 또는 최종 HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. 개발 과정 목표

검증된 배포 콘텐츠에서만 Person·WebSite·프로젝트 CreativeWork 정보를 만들고, JSON 문자열이 HTML 스크립트 문맥을 종료하거나 바꾸지 못하도록 전용 직렬화 함수와 컴포넌트를 거쳐 최상위 및 프로젝트 상세 라우트에 연결하는 과정을 복원합니다.

### 1단계 범위 결정

기존 네 번째 초안의 커밋 집합과 순서는 적절했습니다. 1단계에서는 일반적인 확인 문구를 직렬화 위험, 안정적인 ID, 선택 주장, 배포 환경 차단 조건, 두 렌더링 분기와 부정 주장 회귀를 확인하는 구체적인 정적 검토 항목으로 교체했습니다.

### 고정된 핵심 불변 조건

- JSON-LD는 `StructuredData` 컴포넌트를 통해서만 원본 스크립트 HTML에 삽입되고 `<`, `>`, `&`가 Unicode 이스케이프 처리로 변환됩니다.
- 사이트 참조 관계와 프로젝트 레코드는 기준이 되는 검증된 콘텐츠와 검증된 공개 출처 URL만 사용합니다.
- 템플릿 모드에서는 사이트 수준과 프로젝트별 구조화 데이터를 모두 emit하지 않습니다.
- Person·WebSite·CreativeWork ID와 작성자 참조는 같은 공개 출처 URL의 안정적인 프래그먼트와 경로로 연결됩니다.
- Repository 콘텐츠가 근거를 제공하지 않는 수상·평점 같은 주장는 추가하지 않습니다.

### 주요 기술적 어려움

- `application/ld+json` 스크립트 안의 JSON 유효성과 HTML 파서 맥락 안전성를 동시에 유지하는 문제
- Site 소유 주체, website, 프로젝트 레코드를 안정적인 `@id` 참조로 연결하면서 콘텐츠 근거를 과장하지 않는 문제
- 프로젝트 상세의 전용 렌더러와 대체 처리 렌더러 두 반환 경로에 기계 판독용 출력을 빠짐없이 적용하는 문제

## 2. 핵심 질문

- 일반 `JSON.stringify` 결과가 `</script>`를 포함할 때 어떤 삽입 위험가 있으며 직렬화 함수는 무엇을 이스케이프 처리합니까?
- Person/Web사이트 참조 관계의 안정적인 ID와 선택적 alternateName·이미지 분기는 어떻게 구성됩니까?
- CreativeWork 레코드는 어떤 프로젝트·사이트 필드만 주장하며 작성자 참조를 어떻게 연결합니까?
- 배포 환경 차단 조건은 최상위 레이아웃과 프로젝트 상세 페이지에서 각각 어디에 위치합니까?
- 최종 테스트는 필수 필드뿐 아니라 지원하지 않는 주장과 스크립트 종료 문자열을 어떻게 검증합니까?

## 3. 완료 기준

- 직렬화 함수 introduction을 콘텐츠 모델보다 먼저 둔 실제 커밋 순서와 보안 rationale를 설명했습니다.
- 사이트 참조 관계와 프로젝트 레코드의 필드, ID, 선택적 분기, 보장하지 않는 범위를 해당 SHA별로 기록했습니다.
- 두 프로젝트 렌더링 경로 모두 구조화 데이터 sibling을 받는 통합을 확인했습니다.
- `c5938ea4b4f8`의 의미상, 부정 주장, 삽입 안전성 테스트를 구분했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `228e40a48d64` | feat(seo): JSON-LD 안전 직렬화 경계 추가 | A | SEO | 원시 스크립트를 삽입하는 유일한 지점과 안전한 직렬화 함수 마련 |
| 2 | `ee98415be696` | feat(seo): 사이트 소유자 JSON-LD 모델 추가 | B | SEO | 사이트 소유자와 웹사이트를 하나의 연결된 그래프로 구성 |
| 3 | `ae4ec172e45a` | feat(seo): production layout에 사이트 JSON-LD 연결 | B | SEO | 배포 모드에서만 최상위 레이아웃에 사이트 그래프 출력 |
| 4 | `7e09745d409e` | feat(seo): 프로젝트 CreativeWork JSON-LD 모델 추가 | B | SEO | 기준 프로젝트 레코드를 연결된 CreativeWork 정보로 변환 |
| 5 | `f7bd33a8b403` | feat(seo): 프로젝트 상세에 JSON-LD 연결 | B | SEO | 두 프로젝트 상세 렌더링 경로에 구조화 데이터 출력 |
| 6 | `c5938ea4b4f8` | test(seo): JSON-LD 계약과 직렬화 검증 | A | VALIDATION, SEO, TEST | 구조화 데이터 의미·부정 주장·스크립트 문맥 안전성 보호 |

## 5. 커밋별 학습 기록

### `228e40a48d64` — feat(seo): JSON-LD 안전 직렬화 경계 추가

- **중요도:** A
- **태그:** SEO
- **확정된 역할:** 원시 스크립트를 삽입하는 유일한 지점과 안전한 직렬화 함수 마련

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/structured-data.tsx`와 `serializeStructuredData`를 확인합니다.
- `JSON.stringify` 뒤 `<`, `>`, `&` 치환 순서와 변경 후 JSON 문자열 의미 구조를 확인합니다.
- `dangerouslySetInnerHTML`이 직렬화 함수 결과만 받는 호출 참조 관계를 추적합니다.
- U+2028/U+2029, 직렬화할 수 없는 값, CSP nonce가 이 커밋에서 다뤄지는지 구분합니다.

확인 원칙:

- `228e40a48d64^`와 `228e40a48d64`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-228e40a48d64 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 구조화 데이터 모델이나 삽입 컴포넌트가 없었습니다. 향후 JSON-LD를 일반 `JSON.stringify` 결과 그대로 스크립트에 넣으면 콘텐츠 문자열의 `</script>`가 HTML 파서 문맥을 닫을 위험이 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `serializeStructuredData(data)`가 JSON 문자열의 HTML 문맥에 영향을 주는 문자를 유니코드 이스케이프로 바꾸고, `StructuredData`는 그 문자열만 `application/ld+json` 스크립트의 `dangerouslySetInnerHTML`에 전달합니다. |
| 데이터·상태·소유 주체 | 직렬화 함수가 문구 안전성 규칙을 소유하고 컴포넌트가 원본 DOM insertion point를 단일화합니다. 입력 레코드는 수정하지 않습니다. |
| 실패·누락·대체 처리 | `JSON.stringify`가 지원하지 않는·cyclic 입력에서 실패하면 그대로 예외를 던집니다. 컴포넌트에는 대체 처리 렌더링이 없습니다. `<`, `>`, `&`는 escaped되어 스크립트 종료 문자열을 구성할 수 없습니다. |
| 보장·비보장 | HTML 스크립트 맥락 termination과 HTML 해석을 방지합니다. 스키마.org 의미상 유효성, CSP nonce, cyclic 객체 처리, 모든 possible Unicode separator 규칙은 보장하지 않습니다. |
| 후속 연결 | `ee98415be696`과 `7e09745d409e`가 안전한 구분 지점에 넣을 사이트·프로젝트 모델을 만듭니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 228e40a48d64 — src/lib/site-metadata.ts — serializeStructuredData
return JSON.stringify(data)
  .replaceAll("<", "\\u003c")
  .replaceAll(">", "\\u003e")
  .replaceAll("&", "\\u0026");
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`ee98415be696`과 `7e09745d409e`가 안전한 구분 지점에 넣을 사이트·프로젝트 모델을 만듭니다.
<!-- learner:end commit-228e40a48d64 -->


### `ee98415be696` — feat(seo): 사이트 소유자 JSON-LD 모델 추가

- **중요도:** B
- **태그:** SEO
- **확정된 역할:** 사이트 소유자와 웹사이트를 하나의 연결된 그래프로 구성

#### 해당 SHA에서 확인할 실제 코드

- `createSiteStructuredData`의 Person/WebSite 레코드와 `@graph` 순서를 확인합니다.
- `/#person`, `/#website` 안정적인 ID와 `author: {"@id": personId}` 참조 관계를 추적합니다.
- Korean 이름과 프로필 사진 선택적 분기를 확인합니다.
- 콘텐츠 필드가 검증된 원본에서 오지만 함수 자체가 배포 준비 상태를 호출하지 않는 경계를 기록합니다.

확인 원칙:

- `ee98415be696^`와 `ee98415be696`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-ee98415be696 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 안전한 직렬화 함수는 있었지만 기계 판독용 사이트 식별 정보를 표현할 데이터 모델이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | `createSiteStructuredData({content, siteUrl})`가 Person 레코드와 같은 작성자를 참조하는 WebSite 레코드를 `@graph`로 반환합니다. |
| 데이터·상태·소유 주체 | 검증된 포트폴리오 집계 객체가 주장의 원본 소유 주체이고 생성 함수가 스키마 대응과 안정적인 ID를 소유합니다. 공개 출처 URL은 호출자가 전달합니다. |
| 실패·누락·대체 처리 | `koreanName`과 `photo`가 없으면 대응하는 속성을 생략합니다. 필수 프로필·사이트 필드는 앞선 스키마 검증을 통과했다고 전제합니다. |
| 보장·비보장 | Person/WebSite 식별 정보와 참조 관계는 만들지만 실행 시점 출력, 배포 모드 전용 차단 조건, 외부 소셜 프로필, 자격 정보와 수상 내역은 아직 없습니다. |
| 후속 연결 | `ae4ec172e45a`가 배포 환경 최상위 레이아웃에 참조 관계를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`ae4ec172e45a`가 배포 환경 최상위 레이아웃에 참조 관계를 연결합니다.
<!-- learner:end commit-ee98415be696 -->


### `ae4ec172e45a` — feat(seo): production layout에 사이트 JSON-LD 연결

- **중요도:** B
- **태그:** SEO
- **확정된 역할:** 배포 모드에서만 최상위 레이아웃에 사이트 그래프 출력

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`의 모드 해석과 `siteStructuredData` 조건부 구축을 확인합니다.
- 배포 환경 브랜치가 `resolveProductionSiteUrl`과 `getPortfolioContent`를 생성 함수에 전달하는지 추적합니다.
- `<body>` 첫 하위로 선택적 `StructuredData`가 삽입되는 위치를 확인합니다.

확인 원칙:

- `ae4ec172e45a^`와 `ae4ec172e45a`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-ae4ec172e45a -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 사이트 참조 관계 생성 함수는 있었지만 어떤 라우트에서도 렌더링 결과에 포함되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | `RootLayout`이 모드를 해석하고 배포 모드에서만 사이트 구조화 데이터를 만들어 `<StructuredData>`를 본문에 삽입합니다. 템플릿 모드에서는 `undefined`를 전달해 아무것도 렌더링하지 않습니다. |
| 데이터·상태·소유 주체 | 최상위 레이아웃이 사이트 전체 출력 실행 주기를 소유합니다. Factory는 모델, 배포 준비 상태 도우미 함수는 공개 출처 URL, 컴포넌트는 직렬화·insertion을 각각 소유합니다. |
| 실패·누락·대체 처리 | Template은 명시적으로 아무 JSON-LD도 출력하지 않습니다. 배포 환경 SITE_URL 검증 실패는 레이아웃 렌더링을 실패시킵니다. |
| 보장·비보장 | 모든 배포 환경 페이지에 사이트 참조 관계가 한 번 포함됩니다. 전체 집계 객체 배포 준비 상태는 일반 빌드 전 검사의 별도 규칙이며 레이아웃은 직접 호출하지 않습니다. |
| 후속 연결 | `7e09745d409e`와 `f7bd33a8b403`이 프로젝트별 레코드와 상세 라우트 출력을 추가합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`7e09745d409e`와 `f7bd33a8b403`이 프로젝트별 레코드와 상세 라우트 출력을 추가합니다.
<!-- learner:end commit-ae4ec172e45a -->


### `7e09745d409e` — feat(seo): 프로젝트 CreativeWork JSON-LD 모델 추가

- **중요도:** B
- **태그:** SEO
- **확정된 역할:** 기준 프로젝트 레코드를 연결된 CreativeWork 정보로 변환

#### 해당 SHA에서 확인할 실제 코드

- `createProjectStructuredData`의 프로젝트 경로, `@id`, 작성자 참조를 확인합니다.
- 요약, 화면 캡처, 언어, 태그, 제목, URL 필드가 어떤 원본 properties에서 오는지 매핑합니다.
- 사이트 수준 `/#person` ID와 동일한 작성자 참조를 사용하는지 확인합니다.
- Awards/ratings·배포 상태 주장가 존재하지 않는 것을 변경 후 객체에서 확인합니다.

확인 원칙:

- `7e09745d409e^`와 `7e09745d409e`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-7e09745d409e -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | 최상위 사이트 참조 관계만 있어 개별 프로젝트 사례가 기계 판독용 creative work로 식별되지 않았습니다. |
| 실제 변경 파일·심볼·호출 경로 | `createProjectStructuredData({content, project, siteUrl})`가 `/projects/<id>` URL과 `#creative-work` ID를 만들고 프로젝트 요약·이미지·태그·제목 및 사이트 언어를 매핑합니다. |
| 데이터·상태·소유 주체 | 프로젝트 집계 객체가 주장 데이터를 소유하고 사이트 콘텐츠는 언어·공개 식별 정보 맥락을 제공합니다. Factory는 대응 only입니다. |
| 실패·누락·대체 처리 | 화면 캡처 경로와 URL 변환은 호출자가 검증된 콘텐츠·공개 URL을 전달한다는 전제입니다. 선택적 제외 브랜치 없이 선택한 필드를 모두 사용합니다. |
| 보장·비보장 | CreativeWork와 사이트 Person 사이의 안정적인 작성자 링크를 보장합니다. 수상, 평점, 고용주, 배포 상태처럼 지원하지 않는 의미 정보는 추가하지 않습니다. |
| 후속 연결 | `f7bd33a8b403`이 실제 프로젝트 상세 라우트의 두 렌더링 분기에 레코드를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`f7bd33a8b403`이 실제 프로젝트 상세 라우트의 두 렌더링 분기에 레코드를 연결합니다.
<!-- learner:end commit-7e09745d409e -->


### `f7bd33a8b403` — feat(seo): 프로젝트 상세에 JSON-LD 연결

- **중요도:** B
- **태그:** SEO
- **확정된 역할:** 두 프로젝트 상세 렌더링 경로에 구조화 데이터 출력

#### 해당 SHA에서 확인할 실제 코드

- 프로젝트 상세 페이지의 기존 `notFound` 검사 뒤 모드·structuredData 구축 순서를 확인합니다.
- 전용 렌더러 반환과 대체 처리 `PageShell` 반환 모두 프래그먼트 sibling으로 `StructuredData`를 받는지 비교합니다.
- 템플릿 모드 null 브랜치와 배포 환경 URL 해석 함수 실패를 확인합니다.
- 라우트 메타데이터 생성과 페이지 본문 JSON-LD 생성이 분리된 프레임워크 경로임을 기록합니다.

확인 원칙:

- `f7bd33a8b403^`와 `f7bd33a8b403`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-f7bd33a8b403 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | CreativeWork 모델은 있었지만 프로젝트 상세 페이지에서 사용하지 않았고, 여러 디자인의 전용 렌더링 경로와 공용 대체 경로 중 일부에만 연결될 위험이 있었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 페이지가 유효한 콘텐츠·프로젝트를 확인한 뒤 모드를 해석합니다. 배포 환경에서 레코드를 만들고 전용 렌더러 프래그먼트와 대체 처리 프래그먼트 양쪽 첫 하위로 `StructuredData`를 넣습니다. |
| 데이터·상태·소유 주체 | 페이지 구분 지점이 프로젝트 선택, 모드 차단 조건, 두 렌더링 분기 통합을 소유합니다. 렌더러는 구조화된 주장을 재해석하지 않습니다. |
| 실패·누락·대체 처리 | 프로젝트 페이지가 비활성화됐거나 프로젝트를 찾지 못하면 JSON-LD 생성 전에 `notFound()`를 호출합니다. 템플릿 모드에서는 구조화 데이터를 만들지 않으며, 배포 모드의 `SITE_URL`이 유효하지 않으면 렌더링이 실패합니다. |
| 보장·비보장 | 지원하는 모든 시각 렌더링 경로에서 같은 프로젝트 정보를 제공합니다. 브라우저 DOM에서의 고유성, 스키마.org 외부 검증과 검색 엔진의 리치 결과 노출 가능 여부는 보장하지 않습니다. |
| 후속 연결 | `c5938ea4b4f8`이 모델과 직렬화 함수의 의미상·안전성 규칙을 함께 잠급니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 커밋은 본문에 정확한 파일·심볼·브랜치를 기록했습니다. 확인된 변경 내용을 임의 축약한 가상 코드를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

`c5938ea4b4f8`이 모델과 직렬화 함수의 의미상·안전성 규칙을 함께 잠급니다.
<!-- learner:end commit-f7bd33a8b403 -->


### `c5938ea4b4f8` — test(seo): JSON-LD 계약과 직렬화 검증

- **중요도:** A
- **태그:** VALIDATION, SEO, TEST
- **확정된 역할:** 구조화 데이터 의미·부정 주장·스크립트 문맥 안전성 보호

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/site-metadata.test.ts`의 `describe("structured data")` 세 테스트를 확인합니다.
- 사이트 참조 관계 예상한 ID·타입·콘텐츠 필드, 프로젝트 예상한 필드를 실제 함수에 연결합니다.
- `not.toHaveProperty("award")`, `not.toHaveProperty("aggregateRating")` 부정 단언문을 확인합니다.
- `serializeStructuredData({value: "</script>"})`의 예상한 이스케이프 처리와 테스트가 실제 DOM을 렌더링하지 않는 한계를 기록합니다.

확인 원칙:

- `c5938ea4b4f8^`와 `c5938ea4b4f8`의 부모와의 변경 차이 및 변경 후 파일 트리를 기준으로 합니다.
- 후속 커밋이나 최종 HEAD의 도우미 함수·테스트를 이 SHA의 구현으로 소급하지 않습니다.
- 실행하지 않은 결과와 정적 검토를 구분합니다.

#### 학습 기록

<!-- learner:start commit-c5938ea4b4f8 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | JSON-LD 모델과 연결은 있었지만 필드 불일치, 지원하지 않는 정보 추가, 스크립트 종료 문자열 회귀를 잡는 집중 검증이 없었습니다. |
| 실제 변경 파일·심볼·호출 경로 | 테스트가 사이트·프로젝트 생성 함수와 직렬화 함수를 직접 호출합니다. 사이트 데이터에서는 Person·WebSite의 ID와 내용을, 프로젝트 데이터에서는 CreativeWork의 식별 정보·내용·URL을 확인합니다. |
| 데이터·상태·소유 주체 | 실제 검증된 콘텐츠 테스트 입력과 명시적인 공개 URL이 예상한 주장의 원본입니다. 테스트는 입력에만 의존하는 데이터·직렬화된 문자열을 관찰합니다. |
| 실패·누락·대체 처리 | 프로젝트 테스트 입력이 없으면 명시적인 오류를 발생시킵니다. 부정 properties가 없어야 하며 `</script>`는 `\u003c/script\u003e`를 포함해야 합니다. |
| 보장·비보장 | 의미 필드 대응, 의도적으로 제한한 주장 범위와 중요한 HTML 삽입 이스케이프 처리를 결정적으로 검증합니다. React 컴포넌트 렌더링, 브라우저 파서, CSP, 외부 스키마 검증과 리치 결과 노출 가능 여부는 검증하지 않습니다. |
| 후속 연결 | 분류 내 구조화 데이터 변경 과정의 마지막 회귀 구분 지점입니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// c5938ea4b4f8 — src/lib/site-metadata.test.ts
expect(structuredData).not.toHaveProperty("award");
expect(structuredData).not.toHaveProperty("aggregateRating");
expect(serializeStructuredData({ value: "</script>" }))
  .toContain("\\u003c/script\\u003e");
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 해당 SHA 체크아웃과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub 연결 도구로 조회한 해당 SHA의 변경 내용과 변경 후 파일에 대한 정적 검토입니다.

**다음 커밋 연결**

분류 내 구조화 데이터 변경 과정의 마지막 회귀 구분 지점입니다.
<!-- learner:end commit-c5938ea4b4f8 -->


## 6. 불변 조건 변화

<!-- learner:start thread-invariant-evolution -->
| Commit·구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| 228e40a48d64 | 보안 구분 지점 도입 | 원본 스크립트 삽입을 한곳으로 모으고 마크업에서 의미가 있는 문자를 이스케이프합니다. |
| ee98415be696 | 의미 모델 도입 | Person과 WebSite가 안정적인 ID와 작성자 참조 관계를 공유합니다. |
| ae4ec172e45a | 배포 모드 조건부 통합 | 사이트 참조 관계는 배포 모드에서만 전체 페이지에 출력됩니다. |
| 7e09745d409e | 확장 | 프로젝트를 지원하지 않는 주장 없이 연결된 CreativeWork 레코드로 변환합니다. |
| f7bd33a8b403 | 라우트 통합 | 두 프로젝트 상세 렌더링 분기 모두 라우트 검증 뒤에 같은 레코드를 출력합니다. |
| c5938ea4b4f8 | 결정적으로 검증 | 올바른 의미 구조, 금지된 주장과 스크립트 종료 문자열 이스케이프 처리를 보호합니다. |
<!-- learner:end thread-invariant-evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- learner:start thread-failure-fix-test -->
| 실패·위험 | 수정·결정 | 테스트·증거 |
| --- | --- | --- |
| 일반 JSON 문자열이 스크립트 문맥을 종료할 위험 | 전용 직렬화 함수가 원문을 삽입하기 전에 `<`, `>`, `&`를 이스케이프 | `</script>`를 사용한 회귀 테스트와 Unicode 이스케이프 확인 |
| 구조화된 주장이 근거를 과장할 위험 | 생성 함수가 기준 콘텐츠에서 지원하는 필드만 대응 | 부정 테스트가 수상 내역과 `aggregateRating`을 금지 |
| 여러 디자인의 상세 경로가 서로 다른 레코드를 출력할 위험 | 페이지 구분 지점이 전용 렌더링 분기와 대체 렌더링 분기를 모두 감쌈 | 정적 검토로 두 분기를 확인했으며 이 커밋에는 브라우저 분기 개수 테스트가 없음 |
<!-- learner:end thread-failure-fix-test -->

## 8. 소유 주체·상태·담당 작업 변화

<!-- learner:start thread-ownership -->
| 시점 | 소유 주체 | 책임 변화 |
| --- | --- | --- |
| 이전 | 소유 주체 없음 | 구조화 데이터 모델이나 안전한 삽입 지점이 없었습니다. |
| 228 | 직렬화 함수 + `StructuredData` | 문자열 안전성과 유일한 원본 스크립트 삽입 지점을 소유합니다. |
| ee/7e | 메타데이터 생성 함수 | 근거를 스키마.org 형식에 대응하고 안정적인 ID를 만드는 작업을 소유합니다. |
| ae/f7 | 최상위 및 프로젝트 페이지 구분 지점 | 배포 모드 차단, 공개 출처 URL과 렌더링 시점을 소유합니다. |
| c593 | 집중 단위 테스트 | 의미 구조·과도한 주장 방지·이스케이프 처리를 보호하지만 브라우저나 스키마.org 적합성 검증까지 증명하지는 않습니다. |
<!-- learner:end thread-ownership -->

## 9. 최종 개발 과정 상태와 실행 순서

<!-- learner:start thread-final-state -->
**최종 상태**

배포 모드에서는 최상위 페이지가 서로 연결된 Person·Web사이트 참조 관계를 하나 노출합니다. 유효한 프로젝트 상세 페이지는 같은 Person을 참조하는 CreativeWork를 노출할 수 있습니다. 템플릿 미리보기에서는 구조화 데이터를 출력하지 않습니다. 모든 레코드는 전용 직렬화 함수를 거치며, 콘텐츠가 JSON-LD 스크립트 문맥을 닫거나 바꾸지 못하도록 처리합니다. 테스트는 지원하지 않는 수상·평점 주장을 명시적으로 거부합니다.

**코드 없는 실행 순서**

1. 배포 모드 페이지가 검증된 공개 출처 URL과 기준 포트폴리오 콘텐츠를 읽습니다.
2. 최상위 또는 프로젝트 생성 함수가 지원하는 필드만 안정적인 스키마.org ID와 참조로 변환합니다.
3. 페이지가 결과 레코드를 공용 `StructuredData` 컴포넌트에 전달합니다. 템플릿 모드에서는 아무것도 렌더링하지 않습니다.
4. 컴포넌트가 JSON을 직렬화하고 마크업에서 의미가 있는 문자를 이스케이프한 뒤 `application/ld+json` 스크립트에 삽입합니다.
5. 집중 테스트는 참조 관계의 식별 정보와 콘텐츠를 비교하고, 지원하지 않는 주장을 금지하며, `</script>` 삽입 구분 지점을 재현합니다.
<!-- learner:end thread-final-state -->

## 10. 학습 완료 확인

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 정확한 변경 내용·파일 트리를 GitHub 연결 도구로 정적 확인했습니다.
- [x] 보장과 비보장을 커밋별로 구분했습니다.
- [x] 테스트 방법과 증명 범위와 증명하지 않는 범위를 구분했습니다.
- [x] 최종 처리 순서를 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 05-safe-jsonld-serialization-and-structured-claims.md =====

===== BEGIN FILE: README.md =====
# 06 — SEO·보안·기계 판독용 출력

> 저장소: `seungwoo7050/42-archive`
>
> 브랜치: `web/portfolio`
>
> 산출물 범위: 이 분류에 한정

## 분류 목적

이 분류는 기준 메타데이터, 검색 수집기 노출 규칙, 콘텐츠에 선언된 URL 무결성, 실제 공개 가능성 검증과 안전하게 삽입한 기계 판독용 주장을 변경 이력 순서로 학습합니다.

## 1단계 분류 검토 및 고정된 구분 지점

- **분류 범위:** SEO·검색 수집기·공개 URL·구조화 데이터의 안전성으로 한정했습니다.
- **이 분류에서 제거:** `902eddcef875`, `ba8da56d3fcf`, `f63c978c71c9`는 쿼리 탐색과 UI 링크 이동 방식을 다루는 분류 02·03과 중복되어 제거했습니다.
- **분리:** 기존 혼합 개발 과정 01을 콘텐츠에 선언된 내부 라우트 무결성과 배포 출처 URL·공개 URL 안전성으로 나눴습니다.
- **분류 내부 이동:** `adc392157f70`은 라우트 메타데이터 개발 과정에서 sitemap·검색 노출 개발 과정으로 옮겼습니다.
- **추가한 주요 커밋:** 라우트 검증 회귀 `3353032ba23b`, 공개 준비 상태 연결 순서, 메타데이터 소유 주체 이전 `55b6061e0052 → 67aabeab1553`, 검색 노출 단위 테스트·E2E 근거를 추가했습니다.
- **의도적 재사용:** `55b6061e0052`는 메타데이터 형식과 검색 노출 지시문, `67aabeab1553`는 메타데이터·출처 URL 통합과 페이지별 검색 노출 통합, `fb3d18fd660b`는 배포 준비 상태 회귀와 검색 노출 회귀라는 서로 다른 파일·결정 관점에서 두 개발 과정에 등장합니다. 같은 코드에 대한 설명은 중복하지 않습니다.
- **최종 검증 범위:** 개발 과정 5개, 고유 SHA 31개입니다. URL 무결성, 공개 가능성 검증, 기준 메타데이터, 검색 수집기 규칙과 안전한 JSON-LD를 독립적인 개발 과정으로 다룹니다.

## 고정된 개발 과정 목록

| 순서 | 파일 | 개발 과정 | 커밋 수 |
| --- | --- | --- | --- |
| 1 | `01-content-declared-internal-route-integrity.md` | 콘텐츠에 선언된 내부 라우트 무결성 | 4 |
| 2 | `02-production-origin-and-publication-url-safety.md` | 배포 출처 URL과 공개 URL 안전성 | 9 |
| 3 | `03-route-aware-metadata-and-canonical-identity.md` | 현재 라우트를 반영하는 메타데이터와 기준 URL 식별 정보 | 8 |
| 4 | `04-indexing-robots-and-sitemap-policy.md` | 검색 노출·robots·sitemap 규칙 | 7 |
| 5 | `05-safe-jsonld-serialization-and-structured-claims.md` | 안전한 JSON-LD 직렬화와 구조화된 주장 | 6 |

## 변경 이력 검증 기준

- `commit/commit-importance.md`의 `web/portfolio`는 이 브랜치를 `cce7dd020563`부터 `aff0acdd4cf9`까지 이어지는 독립적인 선형 476개 커밋 이력으로 설명합니다. 아래 SHA는 모두 브랜치 내부 분류와 해당 커밋 객체·변경 내용에 대조했습니다.
- GitHub 저장소 연결 도구를 통해 정확한 커밋 객체·변경 내용과 해당 SHA의 관련 파일을 확인했습니다.
- 선택한 가장 이른 기준 SHA를 `web/portfolio`와 추가 비교했으며, 해당 SHA가 병합 기준점이고 브랜치가 그 이후 커밋을 포함함을 확인했습니다.
- 실행 컨테이너에서 `github.com`의 DNS를 해석하지 못해 로컬 복제에 실패했습니다. 따라서 이 문서에는 프로젝트 명령 실행 결과를 주장하지 않습니다.

## 2단계 상태

<!-- learner:start readme-phase2-status -->
### 2단계 완료 결과

- 고정한 문서 틀 파일: `6`
- 완료본 대응 파일: `6`
- 개발 과정: `5`
- 고유 SHA: `31`
- 프로젝트 테스트 실행: DNS 해석 오류로 저장소 체크아웃에 실패하여 수행하지 않았습니다.
- 변경 이력 근거: GitHub 연결 도구로 정확한 커밋 변경 내용과 파일을 확인했으며 후속 HEAD 코드를 과거 설명에 소급하지 않았습니다.
- 로컬 결과물 검증: 파일 목록 일치, 고정 필드 정규화, SHA-256 해시, 완료 표식, 커밋 목록 메타데이터, Markdown 코드 블록 구분자 짝, ZIP 내부 경로 제한을 생성 스크립트로 확인했습니다.
<!-- learner:end readme-phase2-status -->

## 검증 조합표

<!-- learner:start readme-validation -->
| 검증 | 결과 |
| --- | --- |
| 문서 틀·완료 파일 집합 | PASS — README와 개발 과정 5개의 상대 경로가 정확히 일치합니다. |
| 고정한 문서 틀 해시 | PASS — 완료본 생성 전후의 SHA-256 명세 파일이 동일합니다. |
| 고정 항목 보존 | PASS — 학습자용 표식 내부를 제외한 공통 문구가 각 파일 쌍에서 동일합니다. |
| 모든 SHA가 대상 브랜치에 포함됨 | PASS — 고유 SHA 31개를 브랜치 전체 분류와 정확한 커밋 조회 결과로 교차 확인했습니다. |
| 미완료 학습자 영역 없음 | PASS — 완료본에 빈 학습자 표나 점검 목록이 남아 있지 않습니다. |
| ZIP에 해당 분류만 포함됨 | PASS — 최상위 항목은 분류 하나이며 문서 틀과 완료본만 포함합니다. |
<!-- learner:end readme-validation -->

## 읽는 순서

1. 먼저 콘텐츠에 선언된 내부 라우트 무결성을 읽어 공개 전에 적용되는 라우트 허용 값 집합을 이해합니다.
2. 이어서 메타데이터와 검색 수집기 출력이 신뢰하는 배포 출처 URL·준비 상태 검증 지점을 살펴봅니다.
3. 기준 메타데이터의 소유 주체와 라우트별 공개 항목을 학습합니다.
4. 페이지별 robots 설정, `robots.txt`와 sitemap 공개 규칙을 학습합니다.
5. 마지막으로 JSON-LD 주장 범위와 스크립트 문맥의 안전성을 학습합니다.
===== END FILE: README.md =====

