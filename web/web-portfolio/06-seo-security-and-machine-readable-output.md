===== BEGIN FILE: 01-content-declared-internal-route-integrity.md =====
# Thread: Content-declared internal route integrity

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `06-seo-security-and-machine-readable-output`
>
> Phase 1 audit에서 확정한 구조입니다. Phase 2는 이 문서의 fixed fields와 commit sequence를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- Repository/branch scope는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md` on `web/portfolio` describes the branch as one independent, linear 476-commit history from `cce7dd020563` through `aff0acdd4cf9`. Every SHA below was matched to that branch-local classification and its exact commit object/diff.
- Subject, importance, tags는 branch-local source classification과 일치시켰습니다.
- 아래 role, investigation target, invariant는 Phase 1 category audit에서 repository evidence에 맞춰 동결했습니다.
- 다른 branch 또는 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표

JSON content가 선언한 root-relative URL을 실제 App Router 공개 surface와 대조하고, disabled page·unknown project·unsupported path가 renderer에 도달하기 전에 source-aware 오류로 누적되는 과정을 복원합니다.

### Phase 1 boundary decision

기존 draft는 query-state URL 작성, external anchor transport, content validation을 하나로 묶었습니다. Phase 1에서는 category 02/03이 소유하는 UI transport 커밋을 제거하고, crawler와 publication surface의 정확성에 직접 영향을 주는 content-source route integrity만 남겼습니다.

### Frozen critical invariants

- 검증 대상은 `/`로 시작하지만 `//`로 시작하지 않는 internal route reference입니다.
- 지원되지 않는 pathname, disabled page, unknown/disabled project는 성공으로 통과하지 않습니다.
- 오류는 해당 JSON source file과 정확한 JSON path를 보존한 채 aggregate `PortfolioContentError`에 합쳐집니다.
- site navigation, global links, project links가 동일한 validator를 사용하고 renderer는 이를 재해석하지 않습니다.

### Major engineering difficulties

- URL 문자열의 transport 분류와 실제 공개 route 존재 여부 검증을 분리하는 문제
- 여러 JSON 파일에서 발견되는 오류를 첫 실패에서 중단하지 않고 source-aware 목록으로 누적하는 문제
- page availability와 enabled project identity를 validator가 일관되게 참조하도록 만드는 문제

## 2. 핵심 질문

- `addInternalRouteIssue`는 어떤 입력을 의도적으로 무시하고 어떤 pathname만 검증합니까?
- 지원 page와 project detail route를 판정하는 실제 table/regular expression은 무엇입니까?
- helper 도입 뒤 site, global link, project link consumer가 어떤 순서로 연결됩니까?
- 회귀 테스트는 어떤 content clone을 변형하고 어떤 오류의 file/message를 확인합니까?

## 3. 완료 기준

- 각 SHA에서 `src/lib/content-loader.ts`의 helper와 caller loop를 parent diff로 확인했습니다.
- external/protocol-relative URL이 이 Thread의 검증 범위 밖이라는 점을 보장과 비보장으로 구분했습니다.
- `PortfolioContentError`의 aggregate issue가 source file과 JSON path를 유지하는 흐름을 설명했습니다.
- `3353032ba23b`의 deterministic content mutation test가 무엇을 증명하고 무엇을 증명하지 않는지 기록했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Frozen role |
| --- | --- | --- | --- | --- | --- |
| 1 | `b380f56f5d90` | feat(content): 내부 route 참조 검증 추가 | A | ARCH, CONTENT, VALIDATION | Reusable internal-route validation primitive |
| 2 | `6b9e10289b64` | feat(content): 사이트와 링크 route 참조 검증 추가 | A | ARCH, CONTENT, VALIDATION | Integrate the route validator with site navigation and global links |
| 3 | `08b4ac81739f` | feat(content): 프로젝트 내부 참조 검증 추가 | A | CONTENT, VALIDATION | Extend integrity checks to project relationships and project-local links |
| 4 | `3353032ba23b` | test(content): Vitest 기반 콘텐츠 계약 검증 추가 | A | CONTENT, VALIDATION, TEST | Deterministic regression coverage for source-aware route failures |

## 5. Commit별 학습 기록

### `b380f56f5d90` — feat(content): 내부 route 참조 검증 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, VALIDATION
- **Frozen role:** Reusable internal-route validation primitive

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/content-loader.ts`의 `addInternalRouteIssue`와 commit parent를 비교합니다.
- `href.startsWith("/")`, `href.startsWith("//")`, `new URL(..., "https://portfolio.invalid")` branch를 추적합니다.
- supported page map, `/projects/<id>` regular expression, `enabledPageIds`, `enabledProjectIds`의 ownership을 확인합니다.
- helper가 아직 어떤 source loop에도 호출되지 않는 integration gap을 확인합니다.

확인 원칙:

- `b380f56f5d90^`와 `b380f56f5d90`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-b380f56f5d90 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Schema와 cross-file reference 검증은 존재했지만 content의 root-relative URL이 실제 공개 route를 가리키는지는 중앙에서 확인하지 않았습니다. 잘못된 `/not-a-route`나 disabled page 링크가 구조상 문자열로 통과할 수 있었습니다. |
| 실제 변경 file/symbol/call path | `src/lib/content-loader.ts`에 `addInternalRouteIssue(issues, file, path, href, enabledPageIds, enabledProjectIds, messagePrefix)`가 추가됩니다. URL을 dummy origin으로 parse한 뒤 `/`, 정적 page map, `/projects/<id>` 순서로 판정합니다. |
| Data/state/owner | 검증 결과의 owner는 loader의 `issues` 배열입니다. helper는 content를 수정하지 않고 issue만 append하며, availability set은 caller가 주입합니다. |
| Failure·absence·fallback | 외부 URL과 `//host/path`는 검사하지 않고 return합니다. unsupported path, disabled page, unknown/disabled project는 각각 source-aware issue를 추가합니다. 이 commit 자체에는 caller가 없어 실제 source loading에는 아직 영향이 없습니다. |
| 보장/비보장 | internal pathname 판정 vocabulary는 생겼지만 site/global/project source 전체 적용은 보장하지 않습니다. malformed percent-encoding을 별도 복구하는 branch도 diff에서 확인되지 않습니다. |
| 후속 연결 | `6b9e10289b64`가 site navigation과 global links에, `08b4ac81739f`가 project links에 이 helper를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// b380f56f5d90 — src/lib/content-loader.ts — addInternalRouteIssue
if (!href.startsWith("/") || href.startsWith("//")) return;
const pathname = new URL(href, "https://portfolio.invalid").pathname;
if (pathname === "/") return;
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

Helper introduction only; no production caller exists until the next commit.
<!-- learner:end commit-b380f56f5d90 -->


### `6b9e10289b64` — feat(content): 사이트와 링크 route 참조 검증 추가

- **Importance:** A
- **Tags:** ARCH, CONTENT, VALIDATION
- **Frozen role:** Integrate the route validator with site navigation and global links

#### 해당 SHA에서 확인할 실제 코드

- `loadPortfolioSource`에서 enabled page/project set이 만들어지는 위치를 확인합니다.
- `source.site.navigation.forEach`와 `source.links.forEach`가 넘기는 file/path/messagePrefix를 비교합니다.
- disabled link도 schema/load path에 남아 있는지, route validation이 enabled flag를 조건으로 건너뛰는지 확인합니다.
- 여러 issue가 최종 `PortfolioContentError`로 합쳐지는 기존 throw boundary를 추적합니다.

확인 원칙:

- `6b9e10289b64^`와 `6b9e10289b64`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-6b9e10289b64 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Validator helper는 존재했지만 호출되지 않아 실제 content load가 잘못된 navigation/link를 거부하지 않았습니다. |
| 실제 변경 file/symbol/call path | `loadPortfolioSource`가 `site.navigation`과 global `links`를 순회하며 `addInternalRouteIssue`를 호출합니다. 각 호출은 `src/content/site.json` 또는 `src/content/links.json`과 배열 index 기반 JSON path를 전달합니다. |
| Data/state/owner | route availability 판단은 loader가 만든 `enabledPageIds`와 `enabledProjectIds`가 소유합니다. 개별 renderer나 selector는 검증 정책을 갖지 않습니다. |
| Failure·absence·fallback | 지원되지 않는 navigation은 `Unsupported internal navigation route`, global link는 `Unsupported internal link route` 계열 issue가 됩니다. issue는 즉시 throw하지 않고 기존 aggregate 배열에 누적됩니다. |
| 보장/비보장 | site와 global link source는 보호되지만 project item 내부 links는 아직 검사하지 않습니다. external URL의 protocol/host 안전성도 이 loader helper의 책임이 아닙니다. |
| 후속 연결 | `08b4ac81739f`가 project-local references와 links까지 같은 boundary로 확장합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`08b4ac81739f`가 project-local references와 links까지 같은 boundary로 확장합니다.
<!-- learner:end commit-6b9e10289b64 -->


### `08b4ac81739f` — feat(content): 프로젝트 내부 참조 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Frozen role:** Extend integrity checks to project relationships and project-local links

#### 해당 SHA에서 확인할 실제 코드

- `source.projects.items.forEach` 안에서 group, tags, stack, links 검증 순서를 확인합니다.
- project link의 file/path가 `src/content/projects.json`과 `$.items[i].links[j].href`로 보존되는지 확인합니다.
- `/projects/<id>`가 enabled project set과 대조되는 branch를 다시 확인합니다.
- 이 commit이 route integrity 외에 추가한 duplicate/reference issue를 route 검사와 구분합니다.

확인 원칙:

- `08b4ac81739f^`와 `08b4ac81739f`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-08b4ac81739f -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Site/global links는 검증됐지만 project case study 안의 internal links는 별도 경로라 unknown project detail을 가리킬 수 있었습니다. |
| 실제 변경 file/symbol/call path | `loadPortfolioSource`의 project loop가 각 `project.links` entry에 `addInternalRouteIssue`를 호출합니다. 같은 loop에서 group ID, duplicate tags/stack, technology reference도 검사합니다. |
| Data/state/owner | Project source index가 JSON path ownership을 제공하고, enabled project IDs가 detail-route validity를 결정합니다. 변형 없이 issue만 누적됩니다. |
| Failure·absence·fallback | Unknown 또는 disabled project detail은 project JSON의 정확한 link path에 issue를 남깁니다. external/protocol-relative href는 계속 검사 범위 밖입니다. |
| 보장/비보장 | 세 종류의 content URL source가 동일 route vocabulary를 사용하게 됩니다. 실제 Next route rendering, HTTP 404, external link security attribute는 이 commit이 보장하지 않습니다. |
| 후속 연결 | `3353032ba23b`가 unsupported global link와 missing project route를 deterministic clone mutation으로 검증합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`3353032ba23b`가 unsupported global link와 missing project route를 deterministic clone mutation으로 검증합니다.
<!-- learner:end commit-08b4ac81739f -->


### `3353032ba23b` — test(content): Vitest 기반 콘텐츠 계약 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, TEST
- **Frozen role:** Deterministic regression coverage for source-aware route failures

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/portfolio.test.ts`의 `captureContentError`가 exception type을 어떻게 고정하는지 확인합니다.
- `rejects duplicate IDs, missing designs, and unsupported navigation` test의 clone mutation을 추적합니다.
- `rejects unsupported internal links and missing project routes`가 global/project link를 어떻게 바꾸는지 확인합니다.
- Assertions가 exact full issue list가 아닌 `arrayContaining/objectContaining`임을 기록합니다.

확인 원칙:

- `3353032ba23b^`와 `3353032ba23b`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-3353032ba23b -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Production validator는 구현됐지만 잘못된 route source가 다시 허용되는 회귀를 자동으로 잡는 executable boundary가 없었습니다. |
| 실제 변경 file/symbol/call path | Vitest/jsdom/Testing Library 기반을 추가하고 `src/lib/portfolio.test.ts`가 cloned JSON을 `loadPortfolioSource`에 주입합니다. `captureContentError`는 thrown value가 `PortfolioContentError`인지 확인합니다. |
| Data/state/owner | Test fixture는 imported JSON을 `structuredClone`해 원본 module state를 변경하지 않습니다. 실패 state는 returned value가 아니라 exception의 `issues` 배열로 관찰합니다. |
| Failure·absence·fallback | `/not-a-route` navigation/global link와 `/projects/not-a-project`를 주입해 file/message를 확인합니다. Disabled page/project reference도 별도 test에서 검사합니다. |
| 보장/비보장 | Source-aware route rejection의 deterministic regression evidence입니다. HTTP router, browser navigation, 모든 exact JSON path/order, malformed URL parser behavior까지 증명하지는 않습니다. |
| 후속 연결 | 이 Thread의 production path는 loader에서 종료됩니다. UI transport와 actual 404 behavior는 다른 category/thread가 소유합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

이 Thread의 production path는 loader에서 종료됩니다. UI transport와 actual 404 behavior는 다른 category/thread가 소유합니다.
<!-- learner:end commit-3353032ba23b -->


## 6. Invariant evolution

<!-- learner:start thread-invariant-evolution -->
| Commit/구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| b380f56f5d90 | Introduced | Internal route vocabulary and issue-producing helper exist, but are not yet integrated. |
| 6b9e10289b64 | Extended | Site navigation and global links consume the helper with source-aware paths. |
| 08b4ac81739f | Completed | Project-local links and project identities enter the same validation boundary. |
| 3353032ba23b | Deterministically verified | Cloned invalid sources reproduce unsupported and disabled-reference failures. |
<!-- learner:end thread-invariant-evolution -->

## 7. Failure → Fix → Test 관계

<!-- learner:start thread-failure-fix-test -->
| Failure/위험 | Fix/결정 | Test/증거 |
| --- | --- | --- |
| Route strings were schema-valid but not route-valid | Central helper classifies supported/disabled/unknown internal paths | Vitest mutates navigation/global/project URLs and inspects `PortfolioContentError` |
| Helper initially had no caller | Two integration commits connect all content URL collections | The final content test invokes the aggregate loader, not the helper in isolation |
<!-- learner:end thread-failure-fix-test -->

## 8. Ownership/state/responsibility 변화

<!-- learner:start thread-ownership -->
| 시점 | Owner | 책임 변화 |
| --- | --- | --- |
| Before | Individual content strings/renderers | No shared knowledge that an internal path corresponds to an enabled route. |
| b380 | `addInternalRouteIssue` | Owns path classification but no source traversal. |
| 6b9 → 08b | `loadPortfolioSource` | Owns traversal, enabled sets, source file/path, and aggregate failure. |
| 335 | `src/lib/portfolio.test.ts` | Owns deterministic regression fixtures without mutating checked-in JSON. |
<!-- learner:end thread-ownership -->

## 9. 최종 Thread 상태와 실행 흐름

<!-- learner:start thread-final-state -->
**최종 상태**

At the end of this Thread, every internal route declared in site navigation, global links, or project links is checked against one enabled-route vocabulary during content loading. Unsupported and disabled destinations prevent a valid portfolio aggregate from being produced. External-link transport attributes and browser navigation remain outside this Thread.

**코드 없는 실행 흐름**

1. JSON modules are schema-parsed into `PortfolioSource`.
2. `loadPortfolioSource` derives enabled page and project identity sets.
3. Each relevant URL field is passed to `addInternalRouteIssue` with its source file and JSON path.
4. External/protocol-relative values leave this validator; internal pathnames are classified as root, supported page, project detail, or invalid.
5. All discovered issues are accumulated and the loader throws one `PortfolioContentError`; only an issue-free source reaches selectors/renderers.
<!-- learner:end thread-final-state -->

## 10. Learning completion check

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 exact diff/tree를 GitHub connector로 정적 확인했습니다.
- [x] 보장과 비보장을 commit별로 구분했습니다.
- [x] test technique과 proves/does-not-prove를 구분했습니다.
- [x] 최종 흐름을 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 01-content-declared-internal-route-integrity.md =====

===== BEGIN FILE: 02-production-origin-and-publication-url-safety.md =====
# Thread: Production origin and publication URL safety

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `06-seo-security-and-machine-readable-output`
>
> Phase 1 audit에서 확정한 구조입니다. Phase 2는 이 문서의 fixed fields와 commit sequence를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- Repository/branch scope는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md` on `web/portfolio` describes the branch as one independent, linear 476-commit history from `cce7dd020563` through `aff0acdd4cf9`. Every SHA below was matched to that branch-local classification and its exact commit object/diff.
- Subject, importance, tags는 branch-local source classification과 일치시켰습니다.
- 아래 role, investigation target, invariant는 Phase 1 category audit에서 repository evidence에 맞춰 동결했습니다.
- 다른 branch 또는 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표

Permissive template content와 실제 공개 가능한 production content를 분리하고, public origin, placeholder removal, assets, project exits, contact method가 모두 충족된 경우에만 verified production result를 반환하는 fail-closed publication boundary를 복원합니다.

### Phase 1 boundary decision

기존 draft는 `428055be3e64`의 predicates만 link-security Thread 끝에 두어 실제 owner와 lifecycle을 잃었습니다. Phase 1에서는 mode/error model부터 prebuild integration과 regression test까지 독립 Thread로 분리하고, 중간 B-level placeholder scanner와 S-level aggregate validator를 복원했습니다.

### Frozen critical invariants

- Missing/empty/`template` mode는 template이며 exact `production`만 strict publication을 요청합니다.
- Production `SITE_URL`은 absolute public HTTP(S)이고 local/reserved/credential-bearing origin이 아닙니다.
- Production result는 all-source placeholder scan, required `/content/` assets, enabled project public exit, usable contact가 모두 성공한 뒤에만 verified `URL`을 포함합니다.
- Readiness failure는 first-error가 아니라 file/path/message issue list로 누적됩니다.
- Normal `npm run build`는 schema check 뒤 readiness check를 반드시 실행합니다.

### Major engineering difficulties

- Starter template를 local preview에서는 허용하되 production publication에서는 fail closed로 바꾸는 문제
- URL syntax, public host policy, asset namespace, project/contact domain rules를 하나의 discriminated result로 모으는 문제
- Validation library의 failure를 CLI exit status와 build lifecycle에 정확히 전달하는 문제

## 2. 핵심 질문

- Mode resolver가 허용하는 정확한 input set과 invalid value behavior는 무엇입니까?
- Placeholder scanner는 어떤 source/file map과 JSON path formatter를 사용합니까?
- `parsePublicSiteUrl`과 `isUsablePublicUrl`의 accept/reject policy는 어디가 다릅니까?
- S-level aggregate validator가 성공하기 전후 ownership과 return type은 어떻게 달라집니까?
- Prebuild gate를 우회할 수 있는 invocation과 test가 증명하지 않는 범위는 무엇입니까?

## 3. 완료 기준

- Mode → scanner → origin/link predicates → aggregate validator → domain completeness → build gate 순서를 설명했습니다.
- S-level `002b642d52a3`의 previous risk, fail-closed decision, discriminated result, remaining gaps를 깊게 기록했습니다.
- `isUsablePublicUrl`이 production `SITE_URL` validator와 동일하지 않은 구체적 non-guarantee를 확인했습니다.
- `fb3d18fd660b`의 fixture transformation과 boundary tests를 production paths에 연결했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Frozen role |
| --- | --- | --- | --- | --- | --- |
| 1 | `b3bd671a3243` | feat(content): 콘텐츠 mode와 readiness 오류 모델 추가 | A | CONTENT, VALIDATION | Define conservative mode and structured readiness protocol |
| 2 | `741bbb4caab7` | feat(content): template placeholder 탐색 경계 추가 | B | CONTENT, ROUTING | Add recursive placeholder discovery with source-aware paths |
| 3 | `47b99d6256ef` | feat(content): public origin과 자산 경계 검증 추가 | A | CONTENT, VALIDATION | Validate public production origin and `/content/` asset namespace |
| 4 | `428055be3e64` | feat(content): 공개 URL과 연락 링크 검증 추가 | A | CONTENT, VALIDATION | Define deployable project/contact URL predicates |
| 5 | `002b642d52a3` | feat(content): production readiness 기본 검사 추가 | S | ARCH, CONTENT, VALIDATION | Establish the aggregate fail-closed production trust boundary |
| 6 | `bcd87ed856bf` | feat(content): 필수 자산과 프로젝트 readiness 추가 | A | CONTENT, VALIDATION | Extend production result with portfolio-specific evidence completeness |
| 7 | `71e7ece7208f` | feat(content): 연락 수단과 build readiness 연결 | A | CONTENT, VALIDATION, DEPLOY | Complete domain readiness and expose one mode-aware entry point |
| 8 | `37c0dbc079ff` | build(content): readiness 검사를 prebuild에 연결 | A | CONTENT, VALIDATION, DEPLOY | Make readiness mandatory for the normal npm build lifecycle |
| 9 | `fb3d18fd660b` | test(content): readiness와 indexing 계약 검증 | A | CONTENT, VALIDATION, SEO | Regression-test the complete readiness result and public-origin boundary |

## 5. Commit별 학습 기록

### `b3bd671a3243` — feat(content): 콘텐츠 mode와 readiness 오류 모델 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Frozen role:** Define conservative mode and structured readiness protocol

#### 해당 SHA에서 확인할 실제 코드

- `PortfolioContentMode`, `PortfolioReadinessEnvironment`, issue/result unions를 확인합니다.
- `PortfolioReadinessError`의 message formatting과 retained `issues` ownership을 확인합니다.
- `resolvePortfolioContentMode`가 undefined/empty/template/production/other를 처리하는 branch를 표로 만듭니다.
- 실제 production checks가 아직 없다는 protocol/implementation boundary를 기록합니다.

확인 원칙:

- `b3bd671a3243^`와 `b3bd671a3243`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-b3bd671a3243 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Schema-valid content와 publishable content를 구분하는 mode, success type, aggregate readiness error가 없었습니다. Template placeholders가 production intent와 분리되지 않았습니다. |
| 실제 변경 file/symbol/call path | 새 `src/lib/content-readiness.ts`가 mode/environment/issue/result types, `PortfolioReadinessError`, `resolvePortfolioContentMode`를 정의합니다. |
| Data/state/owner | Mode resolver는 string input을 immutable union으로 바꾸고, error object가 issue array를 보존합니다. Production success branch의 result type만 `URL`을 소유하도록 discriminated union을 설계합니다. |
| Failure·absence·fallback | undefined, empty, `template`은 conservative template입니다. exact `production`만 production이고, 다른 값은 fallback하지 않고 plain `Error`를 throw합니다. |
| 보장/비보장 | Protocol과 failure representation만 보장합니다. 아직 `SITE_URL`, placeholder, asset, project, contact를 검사하거나 production result를 생성하지 않습니다. |
| 후속 연결 | `741bbb4caab7`부터 concrete issue producer가 추가되고 `002b642d52a3`에서 aggregate success/failure boundary가 생깁니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`741bbb4caab7`부터 concrete issue producer가 추가되고 `002b642d52a3`에서 aggregate success/failure boundary가 생깁니다.
<!-- learner:end commit-b3bd671a3243 -->


### `741bbb4caab7` — feat(content): template placeholder 탐색 경계 추가

- **Importance:** B
- **Tags:** CONTENT, ROUTING
- **Frozen role:** Add recursive placeholder discovery with source-aware paths

#### 해당 SHA에서 확인할 실제 코드

- `contentFiles`가 every `PortfolioSource` key를 exact source filename에 매핑하는지 확인합니다.
- `placeholderMarkers`의 regex와 false-positive/false-negative 가능성을 기록합니다.
- `appendPath`가 identifier key, quoted key, array index를 어떻게 표현하는지 확인합니다.
- `collectPlaceholderIssues`의 string/array/object recursion과 non-object terminal behavior를 추적합니다.

확인 원칙:

- `741bbb4caab7^`와 `741bbb4caab7`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-741bbb4caab7 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Mode와 error types는 있었지만 어떤 source를 검사하고 placeholder 위치를 어떻게 보고할지 구현되지 않았습니다. |
| 실제 변경 file/symbol/call path | `contentFiles`, `placeholderMarkers`, `appendPath`, `findPlaceholderMarker`, `collectPlaceholderIssues`가 `src/lib/content-readiness.ts`에 추가됩니다. |
| Data/state/owner | Scanner는 source object를 변경하지 않고 caller-owned issue array에 file/path/message를 append합니다. Path는 `$`에서 시작해 array index와 property를 재귀적으로 확장합니다. |
| Failure·absence·fallback | String에서 첫 matching marker를 issue로 기록하고 return합니다. array/object는 재귀 순회하며 null/number/boolean은 무시합니다. |
| 보장/비보장 | Declared marker vocabulary 탐지는 보장하지만 arbitrary placeholder 의미, cyclic object 방어, 실제 production gate 호출은 보장하지 않습니다. B-level supporting mechanism입니다. |
| 후속 연결 | `002b642d52a3`가 모든 `contentFiles`에 scanner를 호출해 aggregate production failure에 포함합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`002b642d52a3`가 모든 `contentFiles`에 scanner를 호출해 aggregate production failure에 포함합니다.
<!-- learner:end commit-741bbb4caab7 -->


### `47b99d6256ef` — feat(content): public origin과 자산 경계 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Frozen role:** Validate public production origin and `/content/` asset namespace

#### 해당 SHA에서 확인할 실제 코드

- `isReservedHostname`의 exact domain/suffix set을 확인합니다.
- `parsePublicSiteUrl`의 missing, parse error, protocol, local, reserved, credentials branches를 확인합니다.
- `resolveProductionSiteUrl`이 issue array를 `PortfolioReadinessError`로 바꾸는 path를 확인합니다.
- URL path/query/hash를 명시적으로 거부하거나 normalize하는지 확인해 non-guarantee에 기록합니다.

확인 원칙:

- `47b99d6256ef^`와 `47b99d6256ef`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-47b99d6256ef -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Placeholder scanning만으로는 canonical/robots/sitemap이 신뢰할 public origin을 얻을 수 없고 template asset namespace가 그대로 publish될 수 있었습니다. |
| 실제 변경 file/symbol/call path | `addProductionAssetIssue`, `isReservedHostname`, `parsePublicSiteUrl`, `resolveProductionSiteUrl`가 추가됩니다. URL parser는 missing/malformed/unsafe를 structured issue로 바꿉니다. |
| Data/state/owner | Parsed `URL`은 success value이고 issue array는 caller가 소유합니다. Asset check는 `/content/` prefix policy만 적용합니다. |
| Failure·absence·fallback | Missing, non-URL, non-http(s), localhost/loopback/`.localhost`, reserved example/test/invalid host, username/password를 거부합니다. `resolveProductionSiteUrl`은 any issue에서 aggregate error를 throw합니다. |
| 보장/비보장 | Public host/protocol/credential boundary는 보장하지만 URL path/query/hash를 root origin으로 강제하지는 않습니다. Caller가 `.origin`을 사용할 때만 path가 제거됩니다. |
| 후속 연결 | `002b642d52a3`가 parsed site URL을 aggregate result에 포함하고, later metadata/robots/sitemap consumers가 이 resolver를 직접 사용합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`002b642d52a3`가 parsed site URL을 aggregate result에 포함하고, later metadata/robots/sitemap consumers가 이 resolver를 직접 사용합니다.
<!-- learner:end commit-47b99d6256ef -->


### `428055be3e64` — feat(content): 공개 URL과 연락 링크 검증 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Frozen role:** Define deployable project/contact URL predicates

#### 해당 SHA에서 확인할 실제 코드

- `isUsablePublicUrl`의 placeholder, URL parse, protocol, reserved-host conditions를 확인합니다.
- `isUsableContactHref`가 `mailto:`/`tel:`과 public URL을 어떻게 합성하는지 확인합니다.
- `parsePublicSiteUrl`과 달리 local host/credentials를 재검사하는지 비교합니다.
- 이 commit 시점에 predicates의 production caller가 있는지 확인합니다.

확인 원칙:

- `428055be3e64^`와 `428055be3e64`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-428055be3e64 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Production origin은 검증됐지만 project exit와 contact href가 실제 deployable destination인지 재사용 가능한 방식으로 판정할 helper가 없었습니다. |
| 실제 변경 file/symbol/call path | `src/lib/content-readiness.ts`에 `isUsablePublicUrl`과 `isUsableContactHref`가 추가됩니다. Placeholder marker를 먼저 거부하고 URL protocol/hostname 또는 contact scheme을 판정합니다. |
| Data/state/owner | Predicate는 boolean만 반환하고 issue를 직접 만들지 않습니다. Domain-specific validator가 boolean을 소비해 source path가 있는 issue로 변환할 책임을 가집니다. |
| Failure·absence·fallback | Malformed URL, non-http(s), reserved host는 false입니다. Contact는 placeholder가 아니면 `mailto:`/`tel:`을 즉시 true로 처리하거나 public URL predicate에 위임합니다. |
| 보장/비보장 | `isUsablePublicUrl`은 `parsePublicSiteUrl`과 동일하지 않습니다. 이 diff에서는 localhost와 credentials를 명시적으로 거부하지 않으며, `mailto:`/`tel:` payload syntax도 검증하지 않습니다. 이 점은 later tests에서도 직접 보호되지 않습니다. |
| 후속 연결 | `bcd87ed856bf`가 project links에, `71e7ece7208f`가 contact selection에 predicates를 소비합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`bcd87ed856bf`가 project links에, `71e7ece7208f`가 contact selection에 predicates를 소비합니다.
<!-- learner:end commit-428055be3e64 -->


### `002b642d52a3` — feat(content): production readiness 기본 검사 추가

- **Importance:** S
- **Tags:** ARCH, CONTENT, VALIDATION
- **Frozen role:** Establish the aggregate fail-closed production trust boundary

#### 해당 SHA에서 확인할 실제 코드

- `ProductionReadinessResult`가 union에서 production branch만 추출하는 방식을 확인합니다.
- `validateProductionReadiness`의 issue initialization → origin parse → all-source scan → single failure boundary → success return 순서를 추적합니다.
- Previous helpers가 isolated utilities에서 one authoritative publication result로 바뀌는 ownership transition을 기록합니다.
- 이 SHA에서 아직 asset/project/contact completeness가 없는 gap을 명시합니다.

확인 원칙:

- `002b642d52a3^`와 `002b642d52a3`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-002b642d52a3 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Mode, scanner, origin/link helpers는 각각 존재했지만 caller가 임의로 일부만 사용할 수 있었습니다. `production`이라는 상태가 complete verification 없이 선언될 위험이 남아 있었습니다. |
| 실제 변경 file/symbol/call path | `validateProductionReadiness(content, {SITE_URL})`가 one issue array를 만들고 `parsePublicSiteUrl`과 every `contentFiles` entry의 `collectPlaceholderIssues`를 실행한 뒤, any failure에서 `PortfolioReadinessError`를 throw하고 성공 시 `{mode: "production", siteUrl}`을 반환합니다. |
| Data/state/owner | 이 commit부터 verified `URL`의 ownership은 discriminated success result에 있습니다. Caller는 validation을 통과하지 않고 production result를 구성할 수 없으며 source content는 mutation되지 않습니다. |
| Failure·absence·fallback | Invalid/missing origin과 모든 placeholder issue가 한 목록에 공존하므로 앞선 실패가 뒤의 source 문제를 가리지 않습니다. `!siteUrl \|\| issues.length > 0`가 단일 fail-closed branch입니다. |
| 보장/비보장 | S-level invariant는 ‘production result exists only after origin and all-source placeholder verification’입니다. 그러나 required asset presence, `/content/` placement, enabled project public exit, usable contact, mode-aware template bypass, build integration은 아직 없습니다. |
| 후속 연결 | `bcd87ed856bf`와 `71e7ece7208f`가 domain completeness를 확장하고, `37c0dbc079ff`가 normal build lifecycle에 강제하며, `fb3d18fd660b`가 aggregate categories를 검증합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`bcd87ed856bf`와 `71e7ece7208f`가 domain completeness를 확장하고, `37c0dbc079ff`가 normal build lifecycle에 강제하며, `fb3d18fd660b`가 aggregate categories를 검증합니다.
<!-- learner:end commit-002b642d52a3 -->


### `bcd87ed856bf` — feat(content): 필수 자산과 프로젝트 readiness 추가

- **Importance:** A
- **Tags:** CONTENT, VALIDATION
- **Frozen role:** Extend production result with portfolio-specific evidence completeness

#### 해당 SHA에서 확인할 실제 코드

- site social image, profile photo, resume download의 presence와 `/content/` checks를 확인합니다.
- enabled projects filtering과 zero-project failure를 확인합니다.
- 각 enabled project의 screenshot collection과 `isUsablePublicUrl` link requirement를 추적합니다.
- disabled project가 왜 skip되는지 publication surface와 연결합니다.

확인 원칙:

- `bcd87ed856bf^`와 `bcd87ed856bf`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-bcd87ed856bf -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Generic marker-free content와 public origin만으로 production success가 가능해, 실제 이미지·resume·project exit가 없는 빈 publication도 통과할 수 있었습니다. |
| 실제 변경 file/symbol/call path | `validateProductionReadiness`가 site social image, profile photo, resume URL, enabled project count, project screenshots, enabled public project URL을 검사합니다. |
| Data/state/owner | Domain completeness는 aggregate validator가 소유하고, each failure는 original source file/index path를 사용합니다. Enabled=false project는 publication 대상에서 제외합니다. |
| Failure·absence·fallback | Missing asset은 explicit issue, 잘못된 namespace는 `addProductionAssetIssue`, no enabled project/public exit는 project-specific issue가 됩니다. 검사는 계속 진행되어 여러 category를 함께 보고합니다. |
| 보장/비보장 | Published project마다 repository-owned visual evidence와 최소 하나의 public exit를 요구합니다. Contact method는 아직 요구하지 않고, public-link predicate의 localhost/credential non-guarantee는 그대로입니다. |
| 후속 연결 | `71e7ece7208f`가 contact requirement와 mode-aware entry point를 완성합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`71e7ece7208f`가 contact requirement와 mode-aware entry point를 완성합니다.
<!-- learner:end commit-bcd87ed856bf -->


### `71e7ece7208f` — feat(content): 연락 수단과 build readiness 연결

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, DEPLOY
- **Frozen role:** Complete domain readiness and expose one mode-aware entry point

#### 해당 SHA에서 확인할 실제 코드

- `hasContactMethod`의 enabled, placement, type, href predicate conditions를 확인합니다.
- No usable contact issue의 source/path를 확인합니다.
- `validateBuildReadiness`의 template early return과 production delegation을 확인합니다.
- Helper exports가 private로 축소되는 ownership cleanup을 확인합니다.

확인 원칙:

- `71e7ece7208f^`와 `71e7ece7208f`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-71e7ece7208f -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Production content가 project evidence는 갖춰도 방문자가 연락할 실제 경로가 없을 수 있었고, callers는 mode resolution과 strict validator를 직접 조합해야 했습니다. |
| 실제 변경 file/symbol/call path | `validateProductionReadiness`가 contact placement에 포함된 enabled email/github/website link 중 usable href가 하나 이상인지 검사합니다. `validateBuildReadiness`가 mode를 resolve하고 template은 즉시 success, production은 strict validator에 위임합니다. |
| Data/state/owner | Mode branching의 owner가 one public function으로 이동합니다. Internal helpers와 constant exports는 private로 좁혀져 외부 caller가 partial policy를 조합하기 어려워집니다. |
| Failure·absence·fallback | Usable contact가 없으면 `src/content/links.json:$` issue를 추가합니다. Template mode는 placeholder/publication checks를 의도적으로 skip하고 `{mode, siteUrl: undefined}`를 반환합니다. |
| 보장/비보장 | Mode-aware library entry point와 최소 contact requirement를 보장합니다. Build process가 이 function을 호출하는지는 아직 보장하지 않습니다. |
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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`37c0dbc079ff`가 CLI와 `prebuild`에 연결합니다.
<!-- learner:end commit-71e7ece7208f -->


### `37c0dbc079ff` — build(content): readiness 검사를 prebuild에 연결

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, DEPLOY
- **Frozen role:** Make readiness mandatory for the normal npm build lifecycle

#### 해당 SHA에서 확인할 실제 코드

- `package.json`의 `prebuild`, `content:check`, `content:ready` scripts와 shell short-circuit order를 확인합니다.
- `scripts/validate-content-readiness.ts`의 source load, env read, result logging을 확인합니다.
- Known readiness error는 `process.exitCode = 1`, unexpected error는 rethrow되는 branch를 확인합니다.
- `npm run build`와 direct `next build`의 lifecycle 차이를 non-guarantee로 기록합니다.

확인 원칙:

- `37c0dbc079ff^`와 `37c0dbc079ff`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-37c0dbc079ff -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Correct library function은 존재했지만 build가 호출하지 않으면 incomplete production content가 artifact로 만들어질 수 있었습니다. |
| 실제 변경 file/symbol/call path | `prebuild`가 `npm run content:check && npm run content:ready`로 바뀌고 새 script가 loader와 `validateBuildReadiness`를 호출합니다. |
| Data/state/owner | Library error가 process exit status로 전파됩니다. Template/production success는 log만 만들고 source를 변경하지 않습니다. |
| Failure·absence·fallback | Known aggregate readiness error는 formatted message를 stderr에 쓰고 exit code 1을 설정합니다. Unexpected error는 숨기지 않고 throw합니다. 첫 `content:check` 실패 시 shell `&&`로 readiness는 실행되지 않습니다. |
| 보장/비보장 | Normal `npm run build` lifecycle에는 mandatory gate가 생깁니다. Direct `next build` 또는 script bypass까지 기술적으로 차단하지는 않습니다. |
| 후속 연결 | `fb3d18fd660b`가 library-level behavior를 검증하지만 이 CLI/prebuild process 자체를 spawn해 검증하지는 않습니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 37c0dbc079ff — package.json
"prebuild": "npm run content:check && npm run content:ready",
"content:ready": "node --import tsx scripts/validate-content-readiness.ts"
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`fb3d18fd660b`가 library-level behavior를 검증하지만 이 CLI/prebuild process 자체를 spawn해 검증하지는 않습니다.
<!-- learner:end commit-37c0dbc079ff -->


### `fb3d18fd660b` — test(content): readiness와 indexing 계약 검증

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, SEO
- **Frozen role:** Regression-test the complete readiness result and public-origin boundary

#### 해당 SHA에서 확인할 실제 코드

- `replaceTemplateMarkers`와 `createProductionReadyContent`의 deterministic fixture construction을 확인합니다.
- Mode default/invalid, template bypass, aggregate categories, success result, invalid SITE_URL tests를 분류합니다.
- Assertions가 exact issue order보다 category/path presence를 검사하는 이유와 한계를 기록합니다.
- 같은 commit의 `site-metadata.test.ts`는 indexing Thread에서 별도 관점으로 확인합니다.

확인 원칙:

- `fb3d18fd660b^`와 `fb3d18fd660b`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-fb3d18fd660b -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Readiness chain과 build gate는 있었지만 mode default, aggregate reporting, valid production result, origin rejection을 deterministic하게 잠그는 unit regression이 없었습니다. |
| 실제 변경 file/symbol/call path | `src/lib/content-readiness.test.ts`가 checked-in source를 clone하고 marker를 production-like values로 치환한 뒤 required assets/project link를 채웁니다. Tests는 public `validateBuildReadiness`와 `validateProductionReadiness`를 호출합니다. |
| Data/state/owner | Fixture helper가 mutable clone을 소유하고 imported source는 보존합니다. Failure는 `captureReadinessError`가 typed error와 issue list로 관찰합니다. |
| Failure·absence·fallback | Unsupported mode, template permissiveness, all-category aggregate failure, complete success, malformed/ftp/localhost/example origin rejection을 고정합니다. |
| 보장/비보장 | Boundary/unit regression evidence이며 CLI prebuild execution, actual filesystem asset existence, credential-bearing URL, local project/contact href, browser indexing을 증명하지 않습니다. |
| 후속 연결 | Indexing Thread에서는 같은 commit의 `site-metadata.test.ts`가 template noindex/robots와 production origin behavior를 검증합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

Indexing Thread에서는 같은 commit의 `site-metadata.test.ts`가 template noindex/robots와 production origin behavior를 검증합니다.
<!-- learner:end commit-fb3d18fd660b -->


## 6. Invariant evolution

<!-- learner:start thread-invariant-evolution -->
| Commit/구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| b3bd671a3243 | Introduced | Conservative mode and structured result/error protocol. |
| 741bbb4caab7 | Extended | Every source can produce source-aware placeholder issues. |
| 47b99d6256ef → 428055be3e64 | Extended | Public origin, asset namespace, project/contact URL predicates become explicit. |
| 002b642d52a3 | Architecturally enforced | One aggregate validator owns production success and verified URL. |
| bcd87ed856bf → 71e7ece7208f | Completed | Required evidence, project exits, contact, and mode-aware entry point close domain gaps. |
| 37c0dbc079ff | Integrated | Normal npm build consumes the readiness result and propagates failure to process status. |
| fb3d18fd660b | Deterministically verified | Mode, aggregation, success, and origin boundaries receive regression tests. |
<!-- learner:end thread-invariant-evolution -->

## 7. Failure → Fix → Test 관계

<!-- learner:start thread-failure-fix-test -->
| Failure/위험 | Fix/결정 | Test/증거 |
| --- | --- | --- |
| Schema-valid template could be published | Explicit mode + aggregate production validator + mandatory prebuild | Readiness tests verify template bypass and strict production failures |
| Origin/helper policies could be used partially | S-level success result centralizes origin and source checks | Complete fixture must return `{mode: production, siteUrl: URL}` |
| Publication could lack evidence/exits/contact | Domain completeness checks extend the same issue array | Test expects all input categories without masking later issues |
<!-- learner:end thread-failure-fix-test -->

## 8. Ownership/state/responsibility 변화

<!-- learner:start thread-ownership -->
| 시점 | Owner | 책임 변화 |
| --- | --- | --- |
| Before | Callers/environment | No typed distinction between preview and publishable state. |
| b3bd → 428 | `content-readiness.ts` helpers | Own mode vocabulary and issue producers, but no single success owner. |
| 002 | `validateProductionReadiness` | Owns fail-closed production result and verified URL. |
| 71 | `validateBuildReadiness` | Owns template/production branching and narrows internal helper exports. |
| 37 | npm prebuild + readiness CLI | Owns propagation from library failure to build exit status. |
<!-- learner:end thread-ownership -->

## 9. 최종 Thread 상태와 실행 흐름

<!-- learner:start thread-final-state -->
**최종 상태**

Template mode remains convenient and non-publishing, while production mode can be represented only by a validated result containing a public `URL`. The result is withheld until placeholders, origin, required assets, enabled projects, project exits, and a contact method pass. Normal npm builds invoke this boundary before Next compilation.

**코드 없는 실행 흐름**

1. The readiness CLI loads already schema-validated portfolio sources and reads `PORTFOLIO_CONTENT_MODE`/`SITE_URL`.
2. `validateBuildReadiness` resolves mode; template returns without strict publication checks.
3. Production delegates to `validateProductionReadiness`, which accumulates origin, placeholder, asset, project, and contact issues.
4. Any issue produces one `PortfolioReadinessError`; success returns a discriminated production result with the parsed site URL.
5. The CLI maps known failure to exit code 1, so `npm run build` stops before `next build` through the `prebuild` lifecycle.
<!-- learner:end thread-final-state -->

## 10. Learning completion check

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 exact diff/tree를 GitHub connector로 정적 확인했습니다.
- [x] 보장과 비보장을 commit별로 구분했습니다.
- [x] test technique과 proves/does-not-prove를 구분했습니다.
- [x] 최종 흐름을 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 02-production-origin-and-publication-url-safety.md =====

===== BEGIN FILE: 03-route-aware-metadata-and-canonical-identity.md =====
# Thread: Route-aware metadata and canonical identity

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `06-seo-security-and-machine-readable-output`
>
> Phase 1 audit에서 확정한 구조입니다. Phase 2는 이 문서의 fixed fields와 commit sequence를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- Repository/branch scope는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md` on `web/portfolio` describes the branch as one independent, linear 476-commit history from `cce7dd020563` through `aff0acdd4cf9`. Every SHA below was matched to that branch-local classification and its exact commit object/diff.
- Subject, importance, tags는 branch-local source classification과 일치시켰습니다.
- 아래 role, investigation target, invariant는 Phase 1 category audit에서 repository evidence에 맞춰 동결했습니다.
- 다른 branch 또는 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표

Content-derived site identity를 root layout metadata로 옮기고, production에서는 verified `SITE_URL`을 canonical origin으로 사용하며, 각 공개 route가 query-free canonical path와 route-owned title/description/Open Graph/Twitter 값을 export하도록 확장하는 과정을 복원합니다.

### Phase 1 boundary decision

기존 draft의 route sequence는 대체로 맞았지만 최초 request-header metadata가 후속 mode-aware factory로 대체되는 ownership transfer가 빠져 있었고, sitemap test commit까지 한 Thread에 섞여 있었습니다. Phase 1에서는 `55b6061e0052`와 `67aabeab1553`을 추가해 실제 전환을 복원하고, `adc392157f70`은 sitemap Thread로 이동했습니다.

### Frozen critical invariants

- Production metadata origin은 request host 추정이 아니라 검증된 `SITE_URL`에서 옵니다.
- Root canonical은 `/`, non-root canonical은 해당 route path이며 `view`, `debug` 같은 query state를 포함하지 않습니다.
- Route title/description은 authoritative content에서 읽고, non-root title은 site brand와 결합합니다.
- Disabled page와 unknown project는 metadata export에서도 `notFound()`로 차단됩니다.
- Shared factory가 metadata shape를 소유하고 각 App Router page는 route availability와 content selection을 소유합니다.

### Major engineering difficulties

- Reverse proxy request headers에서 계산한 convenient preview origin과 production canonical origin을 분리하는 문제
- Global site metadata와 route-specific metadata를 중복 없이 합성하면서 canonical identity를 고정하는 문제
- Dynamic project metadata가 page render와 같은 availability/project lookup policy를 사용하도록 맞추는 문제

## 2. 핵심 질문

- 최초 `generateMetadata`는 host/protocol을 어떻게 추정하며 어떤 신뢰 가정을 가집니까?
- Pure factory 도입 뒤 production/template metadataBase ownership이 어떻게 이동합니까?
- `createRouteMetadata`는 root와 non-root title, canonical, Open Graph URL을 어떻게 계산합니까?
- 각 route export가 어떤 content field를 title/description으로 사용하고 disabled state를 어디서 거부합니까?
- Route-export test는 shared helper test와 달리 무엇을 실제로 검증합니까?

## 3. 완료 기준

- `1f4f93ad9a0f → 55b6061e0052 → 67aabeab1553`의 transitional origin/ownership 변화를 설명했습니다.
- 각 route SHA에서 exact page file의 `generateMetadata`와 `notFound` path를 확인했습니다.
- Canonical path와 display-query state를 분리한 contract를 기록했습니다.
- `4358bcd34f2e`가 actual route exports를 호출하는 test technique과 한계를 구분했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Frozen role |
| --- | --- | --- | --- | --- | --- |
| 1 | `1f4f93ad9a0f` | feat(metadata): 콘텐츠 기반 site metadata 추가 | A | CONTENT, SEO | Introduce content-derived site metadata at the root layout |
| 2 | `55b6061e0052` | feat(seo): 콘텐츠 mode별 metadata 정책 추가 | A | CONTENT, SEO | Extract a pure site-metadata policy driven by content mode |
| 3 | `67aabeab1553` | feat(seo): layout metadata를 콘텐츠 mode에 연결 | A | CONTENT, SEO | Transfer root-metadata ownership to the mode-aware policy and verified production origin |
| 4 | `1c40645caead` | feat(seo): route별 검색 metadata 정책 추가 | A | ARCH, ROUTING, SEO | Introduce the shared route-metadata identity policy |
| 5 | `844ff4d7abcb` | feat(seo): 홈과 프로젝트 route metadata 연결 | B | ROUTING, SEO | Apply the shared policy to home, project index, and dynamic project detail |
| 6 | `fd5ff532bfe9` | feat(seo): 프로필 route metadata 연결 | B | ROUTING, SEO | Apply route metadata to about, contact, and resume |
| 7 | `5632c5df9b47` | feat(seo): 여정과 근거 route metadata 연결 | B | ROUTING, SEO | Complete route metadata coverage for journey and interview evidence |
| 8 | `4358bcd34f2e` | test(seo): route metadata export 검증 | B | VALIDATION, ROUTING, SEO | Characterize the actual metadata exports of every public route |

## 5. Commit별 학습 기록

### `1f4f93ad9a0f` — feat(metadata): 콘텐츠 기반 site metadata 추가

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Frozen role:** Introduce content-derived site metadata at the root layout

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`의 static metadata 이전 상태와 async `generateMetadata` diff를 비교합니다.
- `headers()`에서 `x-forwarded-host`/`host`, `x-forwarded-proto`, localhost fallback을 선택하는 순서를 확인합니다.
- `site.title`, `site.description`, `site.socialImage`가 title/Open Graph/Twitter로 흐르는 path를 추적합니다.
- Request-controlled origin과 relative canonical `./`가 남기는 production trust gap을 기록합니다.

확인 원칙:

- `1f4f93ad9a0f^`와 `1f4f93ad9a0f`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-1f4f93ad9a0f -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Root layout은 고정 또는 최소 metadata만 제공해 JSON content의 title, description, social image가 crawler/social output에 일관되게 반영되지 않았습니다. |
| 실제 변경 file/symbol/call path | `src/app/layout.tsx`의 `generateMetadata()`가 request headers로 `metadataBase`를 만들고 module-level `site` content를 title, description, Open Graph, Twitter와 canonical `./`에 배치합니다. |
| Data/state/owner | 이 시점의 metadata shape와 origin 추론을 root layout 함수 하나가 모두 소유합니다. Content는 read-only input이고 request headers가 origin input입니다. |
| Failure·absence·fallback | Forwarded host가 없으면 host, 그것도 없으면 `localhost:3100`을 사용합니다. Protocol header가 없으면 local host만 http, 그 외 https로 추정합니다. Invalid host가 `new URL`에서 throw할 수 있으나 별도 domain error는 없습니다. |
| 보장/비보장 | Site copy 기반 output은 생기지만 request headers를 canonical production truth로 신뢰하며 template/production indexing policy가 없습니다. `./` canonical은 route-specific identity도 표현하지 않습니다. |
| 후속 연결 | `55b6061e0052`가 metadata construction을 pure factory로 분리하고, `67aabeab1553`이 production origin을 validated `SITE_URL`로 이전합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`55b6061e0052`가 metadata construction을 pure factory로 분리하고, `67aabeab1553`이 production origin을 validated `SITE_URL`로 이전합니다.
<!-- learner:end commit-1f4f93ad9a0f -->


### `55b6061e0052` — feat(seo): 콘텐츠 mode별 metadata 정책 추가

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Frozen role:** Extract a pure site-metadata policy driven by content mode

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/lib/site-metadata.ts`의 `createPortfolioMetadata` input/output을 확인합니다.
- Relative social image를 `metadataBase`로 absolute URL로 만드는 branch를 확인합니다.
- `mode === "production"`만 index/follow를 활성화하는 decision을 추적합니다.
- Factory가 아직 root layout에 연결되지 않은 integration state를 기록합니다.

확인 원칙:

- `55b6061e0052^`와 `55b6061e0052`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-55b6061e0052 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Root layout 안에 origin resolution, metadata shape, social image normalization이 결합되어 mode policy를 독립적으로 검증하거나 robots policy와 공유하기 어려웠습니다. |
| 실제 변경 file/symbol/call path | 새 `src/lib/site-metadata.ts`가 `createPortfolioMetadata({metadataBase, mode, site})`를 정의합니다. Social image는 `new URL(site.socialImage, metadataBase)`로 정규화하고 template/production robots directives를 반환합니다. |
| Data/state/owner | Pure factory가 site-level metadata shape와 indexing flag를 소유합니다. Caller는 이미 결정된 mode와 URL을 주입해야 합니다. |
| Failure·absence·fallback | Social image가 없으면 images는 `undefined`입니다. Invalid base/image 조합은 URL constructor가 throw합니다. Unsupported mode는 upstream resolver 책임입니다. |
| 보장/비보장 | Pure deterministic policy는 생겼지만 이 commit에서는 layout caller가 아직 이전되지 않아 application output이 자동으로 바뀌지 않습니다. Canonical은 여전히 `./`입니다. |
| 후속 연결 | `67aabeab1553`이 root layout을 factory에 연결하고 production origin을 readiness validator가 소유하게 합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`67aabeab1553`이 root layout을 factory에 연결하고 production origin을 readiness validator가 소유하게 합니다.
<!-- learner:end commit-55b6061e0052 -->


### `67aabeab1553` — feat(seo): layout metadata를 콘텐츠 mode에 연결

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Frozen role:** Transfer root-metadata ownership to the mode-aware policy and verified production origin

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`에서 inline metadata object가 제거되고 `createPortfolioMetadata` 호출로 대체되는 diff를 확인합니다.
- Production branch의 `resolveProductionSiteUrl(process.env.SITE_URL)`와 template branch의 header-derived URL을 비교합니다.
- Mode resolver와 origin resolver가 throw하는 failure가 `generateMetadata`까지 전파되는 path를 확인합니다.
- Full aggregate readiness가 아니라 mode/public-origin helper를 직접 소비한다는 비보장을 기록합니다.

확인 원칙:

- `67aabeab1553^`와 `67aabeab1553`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-67aabeab1553 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Pure metadata policy는 있었지만 root output은 여전히 이전 inline implementation을 사용했고 production canonical origin도 request headers에 의존했습니다. |
| 실제 변경 file/symbol/call path | `generateMetadata`가 mode를 resolve합니다. Production은 `resolveProductionSiteUrl(SITE_URL)`, template은 request headers로 `metadataBase`를 만든 뒤 `createPortfolioMetadata({metadataBase, mode, site})`를 반환합니다. |
| Data/state/owner | Publication mode/public origin validation은 `content-readiness.ts`, metadata shape는 `site-metadata.ts`, framework export는 layout이 소유하는 분리가 완성됩니다. |
| Failure·absence·fallback | Production에서 SITE_URL이 missing/local/reserved이면 `PortfolioReadinessError`가 전파됩니다. Template은 local preview를 위해 header-derived fallback을 유지합니다. |
| 보장/비보장 | Production canonical base는 request host spoofing에서 분리됩니다. 그러나 이 runtime path는 `validateProductionReadiness` 전체를 호출하지 않아 asset/contact completeness는 prebuild가 별도로 보장합니다. |
| 후속 연결 | `1c40645caead`가 이 root policy 위에 route-specific canonical/title factory를 추가합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`1c40645caead`가 이 root policy 위에 route-specific canonical/title factory를 추가합니다.
<!-- learner:end commit-67aabeab1553 -->


### `1c40645caead` — feat(seo): route별 검색 metadata 정책 추가

- **Importance:** A
- **Tags:** ARCH, ROUTING, SEO
- **Frozen role:** Introduce the shared route-metadata identity policy

#### 해당 SHA에서 확인할 실제 코드

- `RouteMetadataInput`, `routeTitle`, `createRouteMetadata`의 exact type/branches를 확인합니다.
- Root `path === "/"`와 non-root title composition을 비교합니다.
- canonical, Open Graph URL, Twitter/Open Graph images가 어떤 relative values를 유지하는지 확인합니다.
- 같은 commit에서 root canonical `./ → /`와 Open Graph root URL이 보정되는 이유를 기록합니다.

확인 원칙:

- `1c40645caead^`와 `1c40645caead`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-1c40645caead -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Root metadata는 mode-aware했지만 모든 route가 같은 site title/description/canonical을 상속해 페이지별 검색 identity가 없었습니다. |
| 실제 변경 file/symbol/call path | `src/lib/site-metadata.ts`에 typed `RouteMetadataInput`, `routeTitle`, `createRouteMetadata`가 추가됩니다. Factory는 canonical path, route description, OG/Twitter title/images/URL을 반환합니다. |
| Data/state/owner | Factory가 route metadata shape와 title convention을 소유합니다. App route는 path와 authoritative content field만 선택합니다. |
| Failure·absence·fallback | Optional `type`은 `website`로 default하고 social image가 없으면 image arrays를 생략합니다. Runtime path existence는 caller와 content validation 책임입니다. |
| 보장/비보장 | Query-free path identity와 consistent social metadata를 보장하지만 absolute resolution은 parent `metadataBase`와 Next metadata merge에 의존합니다. Locale alternate, pagination, per-route robots는 없습니다. |
| 후속 연결 | 이어지는 세 commits가 모든 현재 public route export를 factory에 연결합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

이어지는 세 commits가 모든 현재 public route export를 factory에 연결합니다.
<!-- learner:end commit-1c40645caead -->


### `844ff4d7abcb` — feat(seo): 홈과 프로젝트 route metadata 연결

- **Importance:** B
- **Tags:** ROUTING, SEO
- **Frozen role:** Apply the shared policy to home, project index, and dynamic project detail

#### 해당 SHA에서 확인할 실제 코드

- `src/app/page.tsx`, `src/app/projects/page.tsx`, `src/app/projects/[projectId]/page.tsx`의 `generateMetadata`를 확인합니다.
- Project detail에서 awaited params, `getProjectById`, page-enabled check, `notFound()` 순서를 추적합니다.
- 각 route가 title/description/type에 선택하는 content field를 표로 기록합니다.

확인 원칙:

- `844ff4d7abcb^`와 `844ff4d7abcb`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-844ff4d7abcb -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Shared route factory는 있었지만 실제 App Router exports가 없어 public pages가 route identity를 사용하지 않았습니다. |
| 실제 변경 file/symbol/call path | Home은 site copy, projects는 projects hero, detail은 matched project title/summary와 `article` type을 `createRouteMetadata`에 전달합니다. |
| Data/state/owner | Page module이 route availability와 record lookup을 소유하고 factory가 metadata shape를 소유합니다. Dynamic params는 metadata function에서 await됩니다. |
| Failure·absence·fallback | Projects page가 disabled이거나 detail project가 missing이면 `notFound()`가 metadata generation에서도 실행됩니다. |
| 보장/비보장 | 핵심 3 route의 wiring을 보장하지만 about/resume/contact/journey/interview routes는 아직 root metadata만 상속합니다. |
| 후속 연결 | `fd5ff532bfe9`와 `5632c5df9b47`이 나머지 public routes를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`fd5ff532bfe9`와 `5632c5df9b47`이 나머지 public routes를 연결합니다.
<!-- learner:end commit-844ff4d7abcb -->


### `fd5ff532bfe9` — feat(seo): 프로필 route metadata 연결

- **Importance:** B
- **Tags:** ROUTING, SEO
- **Frozen role:** Apply route metadata to about, contact, and resume

#### 해당 SHA에서 확인할 실제 코드

- 각 page의 enablement key와 metadata source field를 확인합니다.
- About summary, Contact intro/title, Resume hero body/title가 factory input으로 전달되는지 추적합니다.
- Disabled page가 metadata export에서 `notFound()`되는지 확인합니다.

확인 원칙:

- `fd5ff532bfe9^`와 `fd5ff532bfe9`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-fd5ff532bfe9 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Home/project routes만 route-specific identity를 제공해 profile-related pages는 generic site metadata로 남았습니다. |
| 실제 변경 file/symbol/call path | About, Contact, Resume page가 각각 sync `generateMetadata()`를 export하고 `getPortfolioContent`, `isSitePageEnabled`, `createRouteMetadata`를 연결합니다. |
| Data/state/owner | 각 route가 자기 content source와 page availability를 선택합니다. Shared factory는 formatting만 담당합니다. |
| Failure·absence·fallback | 해당 page flag가 false면 metadata 생성 시점에 `notFound()`합니다. Content field가 schema-valid하다는 전제는 loader boundary에서 옵니다. |
| 보장/비보장 | 세 profile route의 identity는 추가되지만 journey/interview는 아직 남습니다. |
| 후속 연결 | `5632c5df9b47`가 마지막 두 evidence routes를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`5632c5df9b47`가 마지막 두 evidence routes를 연결합니다.
<!-- learner:end commit-fd5ff532bfe9 -->


### `5632c5df9b47` — feat(seo): 여정과 근거 route metadata 연결

- **Importance:** B
- **Tags:** ROUTING, SEO
- **Frozen role:** Complete route metadata coverage for journey and interview evidence

#### 해당 SHA에서 확인할 실제 코드

- `src/app/journey/page.tsx`와 `src/app/interview-map/page.tsx`의 metadata exports를 확인합니다.
- Journey narrative intro와 interview map intro가 description owner인지 확인합니다.
- Page availability flags와 exact canonical paths를 기록합니다.

확인 원칙:

- `5632c5df9b47^`와 `5632c5df9b47`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-5632c5df9b47 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Journey와 interview evidence routes만 generic site identity를 사용해 public route coverage가 불완전했습니다. |
| 실제 변경 file/symbol/call path | 두 page module이 content intro와 presentation hero title을 선택해 `/journey`, `/interview-map` metadata를 만듭니다. |
| Data/state/owner | Route page가 enablement와 source selection을 소유하고 factory가 canonical/social shape를 재사용합니다. |
| Failure·absence·fallback | Disabled route는 `notFound()`합니다. Missing content는 앞선 schema/content validation에서 차단됩니다. |
| 보장/비보장 | 현재 enabled public route set의 metadata exports가 완성됩니다. 실제 rendered head와 absolute canonical serialization은 아직 browser test로 검증하지 않았습니다. |
| 후속 연결 | `4358bcd34f2e`가 page modules의 actual exports를 직접 호출해 wiring regression을 고정합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`4358bcd34f2e`가 page modules의 actual exports를 직접 호출해 wiring regression을 고정합니다.
<!-- learner:end commit-5632c5df9b47 -->


### `4358bcd34f2e` — test(seo): route metadata export 검증

- **Importance:** B
- **Tags:** VALIDATION, ROUTING, SEO
- **Frozen role:** Characterize the actual metadata exports of every public route

#### 해당 SHA에서 확인할 실제 코드

- `src/app/route-metadata.test.ts`가 helper가 아니라 각 page의 `generateMetadata`를 import하는지 확인합니다.
- `it.each` route matrix와 dynamic project detail case를 구분합니다.
- canonical/title/description assertions와 setup-time first-project requirement를 확인합니다.
- Rendered HTML, disabled-route behavior, absolute metadata URL을 증명하지 않는 범위를 기록합니다.

확인 원칙:

- `4358bcd34f2e^`와 `4358bcd34f2e`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-4358bcd34f2e -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Factory implementation은 연결됐지만 route module이 잘못된 path/content를 전달하는 wiring regression을 helper unit test만으로 잡을 수 없었습니다. |
| 실제 변경 file/symbol/call path | Test가 8개 route module의 `generateMetadata`를 alias import하고 static route table과 first project detail invocation을 실행합니다. |
| Data/state/owner | Checked-in validated content가 expected title/description owner입니다. Test는 mutation 없이 actual exports의 return object를 관찰합니다. |
| Failure·absence·fallback | Project fixture가 없으면 setup에서 명시적 error를 throw합니다. Static matrix는 canonical path, title 포함, truthy description을 검사합니다. |
| 보장/비보장 | Route-to-factory wiring과 dynamic project content selection을 증명합니다. Next head rendering, metadata merge, absolute URL, disabled paths, Open Graph/Twitter full shape는 증명하지 않습니다. |
| 후속 연결 | Sitemap/route helper의 추가 contract는 `adc392157f70`에서 indexing Thread 관점으로 검증됩니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

Sitemap/route helper의 추가 contract는 `adc392157f70`에서 indexing Thread 관점으로 검증됩니다.
<!-- learner:end commit-4358bcd34f2e -->


## 6. Invariant evolution

<!-- learner:start thread-invariant-evolution -->
| Commit/구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| 1f4f93ad9a0f | Introduced | Content drives site title/description/social output, but origin is request-derived. |
| 55b6061e0052 | Centralized | A pure mode-aware factory owns the root metadata shape. |
| 67aabeab1553 | Corrected/Integrated | Production origin ownership moves to validated `SITE_URL`; template retains preview fallback. |
| 1c40645caead | Extended | Canonical/title/social identity becomes route-aware and query-free. |
| 844ff4d7abcb → 5632c5df9b47 | Integrated | Every current public page exports route-owned metadata. |
| 4358bcd34f2e | Verified | Actual page exports receive deterministic wiring regression coverage. |
<!-- learner:end thread-invariant-evolution -->

## 7. Failure → Fix → Test 관계

<!-- learner:start thread-failure-fix-test -->
| Failure/위험 | Fix/결정 | Test/증거 |
| --- | --- | --- |
| Request-derived origin could become production identity | Production branch resolves validated SITE_URL before calling pure factory | Factory and readiness tests verify public origin policy; route test verifies page wiring |
| All routes inherited one generic canonical/title | Shared route factory plus per-page exports | Route-export matrix checks canonical/title/description for all routes |
| Dynamic project metadata could diverge from page availability | Metadata export reuses page enablement and project lookup | Project detail test invokes actual async export with a real project ID |
<!-- learner:end thread-failure-fix-test -->

## 8. Ownership/state/responsibility 변화

<!-- learner:start thread-ownership -->
| 시점 | Owner | 책임 변화 |
| --- | --- | --- |
| Before | Root layout inline function | Owns headers, origin, metadata shape, and site content mapping together. |
| 55b | `site-metadata.ts` | Owns pure root metadata policy; caller still must choose trustworthy inputs. |
| 67a | Readiness resolver + layout + metadata factory | Origin, framework integration, and shape become separate responsibilities. |
| 1c onward | Page modules + shared factory | Page owns route/content/availability; factory owns canonical/social formatting. |
<!-- learner:end thread-ownership -->

## 9. 최종 Thread 상태와 실행 흐름

<!-- learner:start thread-final-state -->
**최종 상태**

Production site identity is rooted in validated `SITE_URL`, while template previews may use the current request origin. Every public route selects its authoritative title and description, emits a query-free canonical path and consistent social metadata, and rejects disabled or missing content at its page boundary.

**코드 없는 실행 흐름**

1. The root layout resolves content mode and chooses a validated production origin or a request-derived template preview origin.
2. `createPortfolioMetadata` builds site-level title, description, images, and index directives from that input.
3. Each App Router page validates availability, selects route-owned content, and calls `createRouteMetadata` with one canonical pathname.
4. Next combines route metadata with the root `metadataBase`, so relative canonical/social paths resolve against the selected origin.
5. Focused tests import the actual page exports and compare their canonical/title/description to the same authoritative content.
<!-- learner:end thread-final-state -->

## 10. Learning completion check

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 exact diff/tree를 GitHub connector로 정적 확인했습니다.
- [x] 보장과 비보장을 commit별로 구분했습니다.
- [x] test technique과 proves/does-not-prove를 구분했습니다.
- [x] 최종 흐름을 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 03-route-aware-metadata-and-canonical-identity.md =====

===== BEGIN FILE: 04-indexing-robots-and-sitemap-policy.md =====
# Thread: Indexing, robots, and sitemap policy

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `06-seo-security-and-machine-readable-output`
>
> Phase 1 audit에서 확정한 구조입니다. Phase 2는 이 문서의 fixed fields와 commit sequence를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- Repository/branch scope는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md` on `web/portfolio` describes the branch as one independent, linear 476-commit history from `cce7dd020563` through `aff0acdd4cf9`. Every SHA below was matched to that branch-local classification and its exact commit object/diff.
- Subject, importance, tags는 branch-local source classification과 일치시켰습니다.
- 아래 role, investigation target, invariant는 Phase 1 category audit에서 repository evidence에 맞춰 동결했습니다.
- 다른 branch 또는 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표

Template preview를 crawler-visible surface에서 fail closed로 유지하고, production mode에서만 page robots, `robots.txt`, canonical host, sitemap discovery와 enabled route/project URL을 일관되게 공개하는 정책을 복원합니다.

### Phase 1 boundary decision

기존 draft는 핵심 commits를 포함했지만 root metadata consumer, unit regression, running-application regression이 빠져 있었습니다. Phase 1에서는 `67aabeab1553`, `fb3d18fd660b`, `166f05f7be06`을 추가하고, `adc392157f70`을 metadata Thread에서 이동해 crawler output의 implementation→integration→test sequence를 완성했습니다.

### Frozen critical invariants

- Template mode는 page metadata에서 `noindex,nofollow`, `robots.txt`에서 `Disallow: /`, sitemap에서 빈 목록입니다.
- Production mode만 index/follow와 `Allow: /`를 내며, host와 sitemap URL은 validated public origin에서 계산됩니다.
- Sitemap은 root와 enabled page, projects page가 enabled일 때의 enabled project details만 포함합니다.
- Crawler policy는 content mode를 추측하지 않고 exact environment resolver를 사용합니다.
- Unit tests와 running-app E2E가 helper object와 serialized HTTP output을 서로 다른 수준에서 검증합니다.

### Major engineering difficulties

- Page-level robots directive, standalone robots route, sitemap route가 같은 publication decision을 공유하도록 만드는 문제
- Template preview가 accidentally indexable해지는 반대 방향의 regression을 막는 문제
- Content page flags와 filtered project model을 machine-readable route list로 변환하는 문제

## 2. 핵심 질문

- Metadata factory, robots factory, sitemap factory는 template/production에서 각각 무엇을 반환합니까?
- Production robots output이 site URL이 없을 때 왜 throw하며 host/sitemap을 어떻게 계산합니까?
- Root layout integration과 running-app E2E는 pure unit test보다 무엇을 추가로 증명합니까?
- Sitemap route ordering과 page/project enablement 조건은 무엇입니까?
- 현재 tests가 robots host, sitemap XML serialization, canonical head를 어디까지 검증하지 못합니까?

## 3. 완료 기준

- Metadata, robots, sitemap의 mode matrix를 하나의 표로 정리했습니다.
- Helper policy와 App Router route integration을 별도 ownership으로 설명했습니다.
- `166f05f7be06`의 browser/request technique과 `fb3d18fd660b`의 pure unit technique을 구분했습니다.
- `70b69f04e8c7 → adc392157f70`의 enabled-route sitemap implementation과 deterministic boundary test를 연결했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Frozen role |
| --- | --- | --- | --- | --- | --- |
| 1 | `55b6061e0052` | feat(seo): 콘텐츠 mode별 metadata 정책 추가 | A | CONTENT, SEO | Define page-level indexing directives from the content mode |
| 2 | `cb61450ad922` | feat(seo): 콘텐츠 mode별 robots 정책 추가 | A | CONTENT, SEO | Create the mode-aware robots factory and App Router robots route |
| 3 | `67aabeab1553` | feat(seo): layout metadata를 콘텐츠 mode에 연결 | A | CONTENT, SEO | Integrate page indexing directives with the real root metadata export |
| 4 | `fb3d18fd660b` | test(content): readiness와 indexing 계약 검증 | A | CONTENT, VALIDATION, SEO | Unit-test the aligned metadata and robots mode contract |
| 5 | `166f05f7be06` | test(e2e): 콘텐츠 mode별 metadata와 robots 검증 | A | CONTENT, VALIDATION, SEO | Exercise indexing policy through the running application |
| 6 | `70b69f04e8c7` | feat(seo): 공개 route sitemap 생성 | A | ARCH, ROUTING, SEO | Generate sitemap discovery from the validated publication surface |
| 7 | `adc392157f70` | test(seo): route metadata와 sitemap 계약 검증 | B | VALIDATION, ROUTING, SEO | Lock down the sitemap publication boundary and route URL order |

## 5. Commit별 학습 기록

### `55b6061e0052` — feat(seo): 콘텐츠 mode별 metadata 정책 추가

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Frozen role:** Define page-level indexing directives from the content mode

#### 해당 SHA에서 확인할 실제 코드

- `createPortfolioMetadata`의 `shouldIndex`와 `robots` result를 확인합니다.
- Template/production 두 상태 외 값을 factory가 받지 않도록 type boundary를 확인합니다.
- 이 commit은 helper only이며 root layout output에 아직 연결되지 않았다는 점을 기록합니다.

확인 원칙:

- `55b6061e0052^`와 `55b6061e0052`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-55b6061e0052 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Content-driven metadata는 있었지만 starter/template identity가 검색 엔진에 노출되는 것을 막는 explicit page directive가 없었습니다. |
| 실제 변경 file/symbol/call path | `createPortfolioMetadata`가 `mode === "production"`을 `shouldIndex`로 계산해 `{follow, index}`를 반환합니다. |
| Data/state/owner | Pure metadata factory가 page-level crawler directive를 소유하고 mode selection은 upstream resolver가 소유합니다. |
| Failure·absence·fallback | Template은 false/false, production은 true/true입니다. Unsupported mode는 resolver 단계에서 실패해야 하며 factory에는 fallback branch가 없습니다. |
| 보장/비보장 | Policy object만 정의하며 application head나 `robots.txt`는 아직 바뀌지 않습니다. |
| 후속 연결 | `cb61450ad922`가 standalone robots policy를 만들고 `67aabeab1553`이 page directive를 root output에 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// 55b6061e0052 — src/lib/site-metadata.ts
const shouldIndex = mode === "production";
robots: { follow: shouldIndex, index: shouldIndex },
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`cb61450ad922`가 standalone robots policy를 만들고 `67aabeab1553`이 page directive를 root output에 연결합니다.
<!-- learner:end commit-55b6061e0052 -->


### `cb61450ad922` — feat(seo): 콘텐츠 mode별 robots 정책 추가

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Frozen role:** Create the mode-aware robots factory and App Router robots route

#### 해당 SHA에서 확인할 실제 코드

- `createRobots({mode, siteUrl})`의 template early return과 production missing-URL throw를 확인합니다.
- Production result의 `host: siteUrl.origin`과 allow rule을 확인합니다.
- `src/app/robots.ts`가 environment mode와 URL을 어떻게 resolve하는지 추적합니다.
- 이 시점 robots result에는 sitemap field가 아직 없음을 기록합니다.

확인 원칙:

- `cb61450ad922^`와 `cb61450ad922`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-cb61450ad922 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Page meta directive만으로 crawler가 직접 요청하는 `/robots.txt`의 site-wide policy를 통제하지 못했습니다. |
| 실제 변경 file/symbol/call path | `src/lib/site-metadata.ts`에 `createRobots`가, `src/app/robots.ts`에 Next `MetadataRoute.Robots` export가 추가됩니다. |
| Data/state/owner | Factory가 mode-to-robots object mapping을, App route가 env resolution과 framework endpoint를 소유합니다. |
| Failure·absence·fallback | Template은 site URL 없이 전체 disallow합니다. Production인데 URL이 없으면 error를 throw하며 silent relative host로 fallback하지 않습니다. |
| 보장/비보장 | Template/production robots policy와 canonical host는 생기지만 sitemap discovery는 아직 없습니다. Full production readiness를 호출하지 않고 validated URL helper를 직접 사용합니다. |
| 후속 연결 | `67aabeab1553`가 page metadata path를 실제 root layout에 연결하고 tests가 두 output을 함께 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

```ts
// cb61450ad922 — src/lib/site-metadata.ts — createRobots
if (mode === "template") return { rules: { disallow: "/", userAgent: "*" } };
if (!siteUrl) throw new Error("A production site URL is required to create robots.txt.");
return { host: siteUrl.origin, rules: { allow: "/", userAgent: "*" } };
```

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`67aabeab1553`가 page metadata path를 실제 root layout에 연결하고 tests가 두 output을 함께 검증합니다.
<!-- learner:end commit-cb61450ad922 -->


### `67aabeab1553` — feat(seo): layout metadata를 콘텐츠 mode에 연결

- **Importance:** A
- **Tags:** CONTENT, SEO
- **Frozen role:** Integrate page indexing directives with the real root metadata export

#### 해당 SHA에서 확인할 실제 코드

- Root layout가 mode-aware factory를 호출한 뒤 returned robots directives가 actual metadata output에 포함되는지 확인합니다.
- Production/template metadataBase branch와 indexing branch가 같은 resolved mode를 소비하는지 확인합니다.
- Robots route와 root layout이 resolver를 각각 호출해도 동일 env contract를 공유한다는 점을 기록합니다.

확인 원칙:

- `67aabeab1553^`와 `67aabeab1553`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-67aabeab1553 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Page indexing policy는 pure helper에만 있었기 때문에 real root metadata에는 적용되지 않았습니다. |
| 실제 변경 file/symbol/call path | `src/app/layout.tsx`가 mode와 metadataBase를 결정한 뒤 `createPortfolioMetadata`의 result를 framework metadata export로 반환합니다. |
| Data/state/owner | Root layout은 integration owner이고 policy는 factory owner입니다. Both root and robots route read the same environment mode contract independently. |
| Failure·absence·fallback | Production URL validation failure는 metadata generation을 실패시킵니다. Template은 request-derived origin이어도 noindex/nofollow로 남습니다. |
| 보장/비보장 | Actual page metadata path에 policy가 연결됩니다. Robots endpoint와의 runtime consistency는 아직 test로 보호되지 않습니다. |
| 후속 연결 | `fb3d18fd660b`가 pure policy를, `166f05f7be06`이 running application의 meta/robots endpoint를 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`fb3d18fd660b`가 pure policy를, `166f05f7be06`이 running application의 meta/robots endpoint를 검증합니다.
<!-- learner:end commit-67aabeab1553 -->


### `fb3d18fd660b` — test(content): readiness와 indexing 계약 검증

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, SEO
- **Frozen role:** Unit-test the aligned metadata and robots mode contract

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/site-metadata.test.ts`의 two cases를 확인합니다.
- Template case가 metadata robots와 `createRobots` disallow를 함께 확인하는지 추적합니다.
- Production case가 metadataBase, canonical, index/follow, absolute social image, host/allow를 확인하는지 기록합니다.
- Next serialization이 아닌 object-level test라는 한계를 구분합니다.

확인 원칙:

- `fb3d18fd660b^`와 `fb3d18fd660b`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-fb3d18fd660b -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Metadata와 robots helpers가 같은 mode policy를 구현했지만 한쪽만 바뀌는 drift를 잡는 regression이 없었습니다. |
| 실제 변경 file/symbol/call path | `site-metadata.test.ts`가 동일 site fixture와 mode를 두 factories에 전달해 template/production output pairs를 비교합니다. |
| Data/state/owner | Test owns explicit URL fixtures and observes pure return objects; network/framework layer는 개입하지 않습니다. |
| Failure·absence·fallback | Template noindex/nofollow + disallow, production index/follow + allow/host 및 absolute social image를 고정합니다. |
| 보장/비보장 | Policy alignment를 deterministic하게 증명하지만 emitted `<meta>` 문자열, `/robots.txt` text, App Router integration, sitemap은 증명하지 않습니다. |
| 후속 연결 | `166f05f7be06`이 running server를 통해 serialized outputs를 검증합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`166f05f7be06`이 running server를 통해 serialized outputs를 검증합니다.
<!-- learner:end commit-fb3d18fd660b -->


### `166f05f7be06` — test(e2e): 콘텐츠 mode별 metadata와 robots 검증

- **Importance:** A
- **Tags:** CONTENT, VALIDATION, SEO
- **Frozen role:** Exercise indexing policy through the running application

#### 해당 SHA에서 확인할 실제 코드

- `tests/e2e/portfolio.spec.ts`의 added Playwright test를 확인합니다.
- `page.goto('/')`, robots meta locator, API request `/robots.txt`의 서로 다른 paths를 추적합니다.
- Expected mode가 process environment에서 결정되는 방식과 regex assertions를 확인합니다.
- Host/sitemap/canonical까지 검증하지 않는 범위를 기록합니다.

확인 원칙:

- `166f05f7be06^`와 `166f05f7be06`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-166f05f7be06 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Pure unit tests는 Next가 metadata object를 실제 HTML과 robots text로 serialize하고 routes가 연결되었는지 확인하지 못했습니다. |
| 실제 변경 file/symbol/call path | Playwright test가 `/`을 방문해 `meta[name=robots]` content를 보고, request context로 `/robots.txt`를 받아 serialized allow/disallow text를 확인합니다. |
| Data/state/owner | Test process environment가 expected mode를 소유하고 real application/server response가 observation target입니다. |
| Failure·absence·fallback | Home response와 robots response가 성공해야 하며 mode에 따라 index/follow 또는 noindex/nofollow, allow 또는 disallow regex가 일치해야 합니다. |
| 보장/비보장 | Framework integration과 HTTP serialization을 증명합니다. Production host line, sitemap line, canonical tags, all routes, external crawler behavior는 증명하지 않습니다. |
| 후속 연결 | `70b69f04e8c7`가 robots output에 sitemap discovery를 추가하고 route inventory를 생성합니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`70b69f04e8c7`가 robots output에 sitemap discovery를 추가하고 route inventory를 생성합니다.
<!-- learner:end commit-166f05f7be06 -->


### `70b69f04e8c7` — feat(seo): 공개 route sitemap 생성

- **Importance:** A
- **Tags:** ARCH, ROUTING, SEO
- **Frozen role:** Generate sitemap discovery from the validated publication surface

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/app/sitemap.ts`, `absoluteSiteUrl`, `createSitemap`을 확인합니다.
- Template empty return, production missing-URL throw, route array construction 순서를 추적합니다.
- `content.site.pages` flags와 `content.projects`가 어떤 routes를 추가/제외하는지 확인합니다.
- `createRobots` result에 sitemap URL이 추가되는 integration을 확인합니다.

확인 원칙:

- `70b69f04e8c7^`와 `70b69f04e8c7`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-70b69f04e8c7 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Production robots는 allow/host만 제공했고 검색 엔진이 실제 enabled publication surface를 발견할 sitemap이 없었습니다. |
| 실제 변경 file/symbol/call path | `createSitemap`이 mode/content/siteUrl을 받아 root, enabled static pages, projects index와 content projects details를 absolute URLs로 변환합니다. App Router sitemap route가 env/content를 연결하고 robots가 `/sitemap.xml`을 광고합니다. |
| Data/state/owner | Content page flags와 validated portfolio project list가 route membership을 소유합니다. Factory가 order/URL conversion을 소유하고 route module이 framework endpoint를 소유합니다. |
| Failure·absence·fallback | Template은 빈 array, production missing URL은 throw입니다. `pages.projects === false`면 index와 모든 detail을 함께 제외합니다. 각 optional flag는 explicit false일 때만 제외합니다. |
| 보장/비보장 | Enabled publication URLs와 robots discovery를 보장합니다. `lastModified`, change frequency, priority, alternate languages는 생성하지 않습니다. XML HTTP serialization은 이 commit에 test되지 않습니다. |
| 후속 연결 | `adc392157f70`이 template empty, disabled page exclusion, exact absolute URL order를 unit contract로 고정합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`adc392157f70`이 template empty, disabled page exclusion, exact absolute URL order를 unit contract로 고정합니다.
<!-- learner:end commit-70b69f04e8c7 -->


### `adc392157f70` — test(seo): route metadata와 sitemap 계약 검증

- **Importance:** B
- **Tags:** VALIDATION, ROUTING, SEO
- **Frozen role:** Lock down the sitemap publication boundary and route URL order

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/site-metadata.test.ts`의 `describe("sitemap")` 두 cases를 확인합니다.
- Template mode expected `[]`와 production fixture의 `interviewMap: false` mutation을 추적합니다.
- Expected URL list가 root→projects→project detail→remaining pages 순서인지 확인합니다.
- 같은 commit의 route-metadata helper test는 metadata Thread의 helper 보강이며 이 Thread에서는 sitemap assertions만 해석합니다.

확인 원칙:

- `adc392157f70^`와 `adc392157f70`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-adc392157f70 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Sitemap algorithm과 robots discovery는 구현됐지만 template leakage, disabled page inclusion, origin/order regression을 방지하는 focused test가 없었습니다. |
| 실제 변경 file/symbol/call path | Test는 actual content를 clone-like object spread로 감싸 `interviewMap`만 false로 바꾸고 `createSitemap`의 URL projection을 exact array로 비교합니다. |
| Data/state/owner | Fixture가 page availability variation을 소유하고 factory result가 deterministic publication inventory입니다. |
| Failure·absence·fallback | Template result는 정확히 빈 배열이어야 하고 disabled interview route는 production expected list에 없어야 합니다. |
| 보장/비보장 | Mode, enabled page exclusion, project detail inclusion, public origin, deterministic order를 증명합니다. Multiple projects beyond fixture shape, rendered XML, robots sitemap line의 HTTP output은 증명하지 않습니다. |
| 후속 연결 | 이 commit이 category 내 sitemap story의 마지막 focused regression evidence입니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

이 commit이 category 내 sitemap story의 마지막 focused regression evidence입니다.
<!-- learner:end commit-adc392157f70 -->


## 6. Invariant evolution

<!-- learner:start thread-invariant-evolution -->
| Commit/구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| 55b6061e0052 | Introduced | Page metadata distinguishes template noindex from production index. |
| cb61450ad922 | Extended | The same mode controls site-wide robots allow/disallow and host. |
| 67aabeab1553 | Integrated | The root layout emits the page directive in the real application. |
| fb3d18fd660b | Unit-verified | Metadata and robots object policies are tested together. |
| 166f05f7be06 | Runtime-verified in repository history | The running app's meta tag and robots text are exercised by Playwright; not rerun here. |
| 70b69f04e8c7 | Extended | Robots advertises a sitemap containing only enabled production routes. |
| adc392157f70 | Deterministically verified | Template empty and production enabled-route inventory are locked down. |
<!-- learner:end thread-invariant-evolution -->

## 7. Failure → Fix → Test 관계

<!-- learner:start thread-failure-fix-test -->
| Failure/위험 | Fix/결정 | Test/증거 |
| --- | --- | --- |
| Template starter could be indexed | Noindex/nofollow + robots disallow + empty sitemap | Unit and E2E tests cover metadata and robots; sitemap unit test covers empty output |
| Crawler outputs could disagree on mode | All factories consume the same exact mode vocabulary | One unit commit compares metadata/robots; E2E observes both serialized outputs |
| Sitemap could advertise disabled surfaces | Route inventory is built from page flags and validated enabled projects | Disabled interview route is omitted in exact expected URL list |
<!-- learner:end thread-failure-fix-test -->

## 8. Ownership/state/responsibility 변화

<!-- learner:start thread-ownership -->
| 시점 | Owner | 책임 변화 |
| --- | --- | --- |
| Before | Implicit framework defaults | No explicit crawler publication policy. |
| 55b/cb | Pure factories | Own page and robots policy while route modules own environment integration. |
| 67a | Root layout | Consumes page policy in actual metadata export. |
| 70b | Sitemap factory + route | Owns machine-readable publication inventory; robots owns discovery link. |
| Tests | Unit and Playwright layers | Separate object contract from framework/HTTP integration evidence. |
<!-- learner:end thread-ownership -->

## 9. 최종 Thread 상태와 실행 흐름

<!-- learner:start thread-final-state -->
**최종 상태**

Template previews are closed to indexing through three aligned surfaces: page metadata, robots.txt, and an empty sitemap. Production uses a validated public origin, permits crawling, advertises its sitemap, and lists only routes enabled by current content plus validated project details.

**코드 없는 실행 흐름**

1. Each machine-readable route resolves the exact content mode from the environment.
2. The root metadata factory emits index/follow only for production; template receives noindex/nofollow.
3. The robots route disallows all template crawling or emits production allow/host/sitemap values from the public origin.
4. The sitemap route returns no template URLs; production maps enabled pages and projects to absolute URLs.
5. Unit tests protect pure output objects, while the repository's Playwright test exercises page metadata and robots text through the running application.
<!-- learner:end thread-final-state -->

## 10. Learning completion check

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 exact diff/tree를 GitHub connector로 정적 확인했습니다.
- [x] 보장과 비보장을 commit별로 구분했습니다.
- [x] test technique과 proves/does-not-prove를 구분했습니다.
- [x] 최종 흐름을 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 04-indexing-robots-and-sitemap-policy.md =====

===== BEGIN FILE: 05-safe-jsonld-serialization-and-structured-claims.md =====
# Thread: Safe JSON-LD serialization and structured claims

> Project: 42 Archive Portfolio (`web/portfolio`)
>
> Category: `06-seo-security-and-machine-readable-output`
>
> Phase 1 audit에서 확정한 구조입니다. Phase 2는 이 문서의 fixed fields와 commit sequence를 변경하지 않습니다.

## 0. 분류 출처와 역사 범위

- Repository/branch scope는 `seungwoo7050/42-archive`의 `web/portfolio`로 한정합니다.
- `commit/commit-importance.md` on `web/portfolio` describes the branch as one independent, linear 476-commit history from `cce7dd020563` through `aff0acdd4cf9`. Every SHA below was matched to that branch-local classification and its exact commit object/diff.
- Subject, importance, tags는 branch-local source classification과 일치시켰습니다.
- 아래 role, investigation target, invariant는 Phase 1 category audit에서 repository evidence에 맞춰 동결했습니다.
- 다른 branch 또는 final HEAD를 과거 SHA 설명에 사용하지 않습니다.

## 1. Thread 목표

Validated production content에서만 Person/WebSite와 project CreativeWork claims를 만들고, JSON text가 HTML script context를 종료하거나 변경할 수 없도록 dedicated serializer/component boundary를 거쳐 root와 project detail routes에 연결하는 과정을 복원합니다.

### Phase 1 boundary decision

기존 4번째 draft의 commit set과 순서는 적절했습니다. Phase 1에서는 generic prompts를 serializer threat, stable IDs, optional claims, production gating, dual render branch, negative-claim regression에 맞춘 exact inspection tasks로 교체했습니다.

### Frozen critical invariants

- JSON-LD는 `StructuredData` component를 통해서만 raw script HTML에 삽입되고 `<`, `>`, `&`가 Unicode escape로 변환됩니다.
- Site graph와 project record는 authoritative validated content와 validated public origin만 사용합니다.
- Template mode에서는 site-level과 project-level structured data를 모두 emit하지 않습니다.
- Person/WebSite/CreativeWork IDs와 author references는 같은 public origin의 stable fragments/paths로 연결됩니다.
- Repository content가 근거를 제공하지 않는 award/rating 같은 claims는 추가하지 않습니다.

### Major engineering difficulties

- `application/ld+json` script 안의 JSON validity와 HTML parser context safety를 동시에 유지하는 문제
- Site owner, website, project records를 stable `@id` references로 연결하면서 content evidence를 과장하지 않는 문제
- Project detail의 dedicated renderer와 fallback renderer 두 return paths에 machine-readable output을 빠짐없이 적용하는 문제

## 2. 핵심 질문

- Plain `JSON.stringify` 결과가 `</script>`를 포함할 때 어떤 embedding risk가 있으며 serializer는 무엇을 escape합니까?
- Person/WebSite graph의 stable IDs와 optional alternateName/image branches는 어떻게 구성됩니까?
- CreativeWork record는 어떤 project/site fields만 claim하며 author reference를 어떻게 연결합니까?
- Production gating은 root layout와 project detail page에서 각각 어디에 위치합니까?
- Final tests는 positive fields뿐 아니라 unsupported claims와 script terminator를 어떻게 검증합니까?

## 3. 완료 기준

- Serializer introduction을 content model보다 먼저 둔 실제 commit order와 security rationale를 설명했습니다.
- Site graph와 project record의 fields, IDs, optional branches, non-guarantees를 exact SHA별로 기록했습니다.
- 두 project rendering paths 모두 structured data sibling을 받는 integration을 확인했습니다.
- `c5938ea4b4f8`의 semantic, negative-claim, embedding-safety tests를 구분했습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Frozen role |
| --- | --- | --- | --- | --- | --- |
| 1 | `228e40a48d64` | feat(seo): JSON-LD 안전 직렬화 경계 추가 | A | SEO | Establish the only raw-script embedding boundary and safe serializer |
| 2 | `ee98415be696` | feat(seo): 사이트 소유자 JSON-LD 모델 추가 | B | SEO | Model the site owner and website as one linked graph |
| 3 | `ae4ec172e45a` | feat(seo): production layout에 사이트 JSON-LD 연결 | B | SEO | Emit the site graph at the root layout only in production mode |
| 4 | `7e09745d409e` | feat(seo): 프로젝트 CreativeWork JSON-LD 모델 추가 | B | SEO | Map an authoritative project record to a linked CreativeWork claim |
| 5 | `f7bd33a8b403` | feat(seo): 프로젝트 상세에 JSON-LD 연결 | B | SEO | Emit project structured data across both detail rendering paths |
| 6 | `c5938ea4b4f8` | test(seo): JSON-LD 계약과 직렬화 검증 | A | VALIDATION, SEO, TEST | Protect structured-data semantics, negative claims, and script-context safety |

## 5. Commit별 학습 기록

### `228e40a48d64` — feat(seo): JSON-LD 안전 직렬화 경계 추가

- **Importance:** A
- **Tags:** SEO
- **Frozen role:** Establish the only raw-script embedding boundary and safe serializer

#### 해당 SHA에서 확인할 실제 코드

- 새 `src/components/portfolio/structured-data.tsx`와 `serializeStructuredData`를 확인합니다.
- `JSON.stringify` 뒤 `<`, `>`, `&` replacement 순서와 resulting JSON string semantics를 확인합니다.
- `dangerouslySetInnerHTML`이 serializer result만 받는 call relation을 추적합니다.
- U+2028/U+2029, non-serializable values, CSP nonce가 이 commit에서 다뤄지는지 구분합니다.

확인 원칙:

- `228e40a48d64^`와 `228e40a48d64`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-228e40a48d64 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Structured data model이나 embedding component가 없었고, future JSON-LD를 plain `JSON.stringify`로 raw script에 넣으면 content string의 `</script>`가 HTML parser context를 닫을 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | `serializeStructuredData(data)`가 JSON text의 markup-significant characters를 unicode escapes로 바꾸고 `StructuredData`가 그 string만 `application/ld+json` script의 `dangerouslySetInnerHTML`로 전달합니다. |
| Data/state/owner | Serializer가 text-safety policy를 소유하고 component가 raw DOM insertion point를 단일화합니다. Input record는 수정하지 않습니다. |
| Failure·absence·fallback | `JSON.stringify`가 unsupported/cyclic input에서 실패하면 그대로 throw합니다. Component에는 fallback rendering이 없습니다. `<`, `>`, `&`는 escaped되어 script terminator를 구성할 수 없습니다. |
| 보장/비보장 | HTML script-context termination과 markup interpretation을 방지합니다. Schema.org semantic validity, CSP nonce, cyclic object handling, every possible Unicode separator policy는 보장하지 않습니다. |
| 후속 연결 | `ee98415be696`과 `7e09745d409e`가 safe boundary에 넣을 site/project models를 만듭니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`ee98415be696`과 `7e09745d409e`가 safe boundary에 넣을 site/project models를 만듭니다.
<!-- learner:end commit-228e40a48d64 -->


### `ee98415be696` — feat(seo): 사이트 소유자 JSON-LD 모델 추가

- **Importance:** B
- **Tags:** SEO
- **Frozen role:** Model the site owner and website as one linked graph

#### 해당 SHA에서 확인할 실제 코드

- `createSiteStructuredData`의 Person/WebSite records와 `@graph` order를 확인합니다.
- `/#person`, `/#website` stable IDs와 `author: {"@id": personId}` relation을 추적합니다.
- Korean name과 profile photo optional branches를 확인합니다.
- Content fields가 validated source에서 오지만 function 자체가 readiness를 호출하지 않는 경계를 기록합니다.

확인 원칙:

- `ee98415be696^`와 `ee98415be696`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-ee98415be696 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Safe serializer는 있었지만 machine-readable site identity를 표현할 data model이 없었습니다. |
| 실제 변경 file/symbol/call path | `createSiteStructuredData({content, siteUrl})`가 Person record와 author-linked WebSite record를 `@graph`로 반환합니다. |
| Data/state/owner | Validated portfolio aggregate가 claims의 source owner이고 factory가 schema mapping과 stable IDs를 소유합니다. Public origin은 caller가 전달합니다. |
| Failure·absence·fallback | `koreanName`과 `photo`가 없으면 corresponding properties를 생략합니다. Required profile/site fields는 upstream schema를 전제로 합니다. |
| 보장/비보장 | Person/WebSite identity와 relation은 만들지만 runtime emission, production-only gating, external social profiles, credentials/awards는 아직 없습니다. |
| 후속 연결 | `ae4ec172e45a`가 production root layout에 graph를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`ae4ec172e45a`가 production root layout에 graph를 연결합니다.
<!-- learner:end commit-ee98415be696 -->


### `ae4ec172e45a` — feat(seo): production layout에 사이트 JSON-LD 연결

- **Importance:** B
- **Tags:** SEO
- **Frozen role:** Emit the site graph at the root layout only in production mode

#### 해당 SHA에서 확인할 실제 코드

- `src/app/layout.tsx`의 mode resolution과 `siteStructuredData` conditional construction을 확인합니다.
- Production branch가 `resolveProductionSiteUrl`과 `getPortfolioContent`를 factory에 전달하는지 추적합니다.
- `<body>` 첫 child로 optional `StructuredData`가 삽입되는 위치를 확인합니다.

확인 원칙:

- `ae4ec172e45a^`와 `ae4ec172e45a`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-ae4ec172e45a -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Site graph factory는 있었지만 어떤 route에서도 rendered output에 포함되지 않았습니다. |
| 실제 변경 file/symbol/call path | RootLayout가 mode를 resolve하고 production에서만 site graph를 생성해 `<StructuredData>`를 body에 삽입합니다. Template은 `undefined`와 null render를 사용합니다. |
| Data/state/owner | Root layout가 site-wide emission lifecycle을 소유합니다. Factory는 model, readiness helper는 public origin, component는 serialization/insertion을 각각 소유합니다. |
| Failure·absence·fallback | Template은 명시적으로 아무 JSON-LD도 출력하지 않습니다. Production SITE_URL validation failure는 layout rendering을 실패시킵니다. |
| 보장/비보장 | 모든 production pages에 site graph가 한 번 포함됩니다. Full aggregate readiness는 normal prebuild의 별도 contract이며 layout은 직접 호출하지 않습니다. |
| 후속 연결 | `7e09745d409e`와 `f7bd33a8b403`이 project-specific record와 detail-route emission을 추가합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`7e09745d409e`와 `f7bd33a8b403`이 project-specific record와 detail-route emission을 추가합니다.
<!-- learner:end commit-ae4ec172e45a -->


### `7e09745d409e` — feat(seo): 프로젝트 CreativeWork JSON-LD 모델 추가

- **Importance:** B
- **Tags:** SEO
- **Frozen role:** Map an authoritative project record to a linked CreativeWork claim

#### 해당 SHA에서 확인할 실제 코드

- `createProjectStructuredData`의 project path, `@id`, author reference를 확인합니다.
- Summary, screenshot, language, tags, title, URL fields가 어떤 source properties에서 오는지 매핑합니다.
- Site-level `/#person` ID와 동일한 author reference를 사용하는지 확인합니다.
- Awards/ratings/deployment claims가 존재하지 않는 것을 resulting object에서 확인합니다.

확인 원칙:

- `7e09745d409e^`와 `7e09745d409e`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-7e09745d409e -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | Root site graph만 있어 개별 case study가 machine-readable creative work로 식별되지 않았습니다. |
| 실제 변경 file/symbol/call path | `createProjectStructuredData({content, project, siteUrl})`가 `/projects/<id>` URL과 `#creative-work` ID를 만들고 project summary/image/tags/title 및 site language를 매핑합니다. |
| Data/state/owner | Project aggregate가 claim data를 소유하고 site content는 language/public identity context를 제공합니다. Factory는 mapping only입니다. |
| Failure·absence·fallback | Screenshot path와 URL conversion은 caller가 validated content/public URL을 전달한다는 전제입니다. Optional omission branch 없이 selected fields를 모두 사용합니다. |
| 보장/비보장 | CreativeWork와 site Person 간 stable author link를 보장합니다. Award, rating, employer, deployment status 같은 unsupported semantic claims는 추가하지 않습니다. |
| 후속 연결 | `f7bd33a8b403`이 actual project detail route의 두 rendering branches에 record를 연결합니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`f7bd33a8b403`이 actual project detail route의 두 rendering branches에 record를 연결합니다.
<!-- learner:end commit-7e09745d409e -->


### `f7bd33a8b403` — feat(seo): 프로젝트 상세에 JSON-LD 연결

- **Importance:** B
- **Tags:** SEO
- **Frozen role:** Emit project structured data across both detail rendering paths

#### 해당 SHA에서 확인할 실제 코드

- Project detail page의 existing `notFound` checks 뒤 mode/structuredData construction 순서를 확인합니다.
- Dedicated renderer return과 fallback `PageShell` return 모두 fragment sibling으로 `StructuredData`를 받는지 비교합니다.
- Template mode null branch와 production URL resolver failure를 확인합니다.
- Route metadata generation과 page body JSON-LD generation이 separate framework paths임을 기록합니다.

확인 원칙:

- `f7bd33a8b403^`와 `f7bd33a8b403`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-f7bd33a8b403 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | CreativeWork model은 있었지만 project detail page가 render하지 않았고, multi-design dedicated path와 fallback path 중 하나만 수정할 위험이 있었습니다. |
| 실제 변경 file/symbol/call path | Page가 valid content/project를 확인한 뒤 mode를 resolve합니다. Production에서 record를 만들고 dedicated renderer fragment와 fallback fragment 양쪽 첫 child로 `StructuredData`를 넣습니다. |
| Data/state/owner | Page boundary가 project selection, mode gating, both render branches integration을 소유합니다. Renderer는 structured claims를 재해석하지 않습니다. |
| Failure·absence·fallback | Disabled projects page 또는 missing project는 JSON-LD 생성 전에 `notFound()`합니다. Template은 no structured data. Production SITE_URL invalid이면 render가 실패합니다. |
| 보장/비보장 | 모든 supported visual rendering path에서 같은 project claim을 제공합니다. Browser DOM uniqueness, Schema.org external validation, search-engine rich result eligibility는 보장하지 않습니다. |
| 후속 연결 | `c5938ea4b4f8`이 models와 serializer의 semantic/safety contracts를 함께 잠급니다. |

#### 코드·실행 증거

**최소 코드 발췌**

이 commit은 본문에 exact file/symbol/branch를 기록했습니다. 확인된 diff를 임의 축약한 pseudo-code를 만들지 않기 위해 코드 블록은 생략했습니다.

**실행 증거**

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

`c5938ea4b4f8`이 models와 serializer의 semantic/safety contracts를 함께 잠급니다.
<!-- learner:end commit-f7bd33a8b403 -->


### `c5938ea4b4f8` — test(seo): JSON-LD 계약과 직렬화 검증

- **Importance:** A
- **Tags:** VALIDATION, SEO, TEST
- **Frozen role:** Protect structured-data semantics, negative claims, and script-context safety

#### 해당 SHA에서 확인할 실제 코드

- `src/lib/site-metadata.test.ts`의 `describe("structured data")` 세 tests를 확인합니다.
- Site graph expected IDs/types/content fields, project expected fields를 production functions에 연결합니다.
- `not.toHaveProperty("award")`, `not.toHaveProperty("aggregateRating")` negative assertions를 확인합니다.
- `serializeStructuredData({value: "</script>"})`의 expected escape와 test가 실제 DOM을 render하지 않는 한계를 기록합니다.

확인 원칙:

- `c5938ea4b4f8^`와 `c5938ea4b4f8`의 parent diff와 resulting tree를 기준으로 합니다.
- Later commit이나 final HEAD의 helper/test를 이 SHA에 소급하지 않습니다.
- 실행하지 않은 runtime result와 정적 inspection을 구분합니다.

#### 학습 기록

<!-- learner:start commit-c5938ea4b4f8 -->
| 확인·기록 항목 | 학습자 기록 |
| --- | --- |
| 직전 상태와 부족함 | JSON-LD models와 integration이 있었지만 field drift, unsupported claim addition, script terminator regression을 잡는 focused contract가 없었습니다. |
| 실제 변경 file/symbol/call path | Tests가 site/project factories와 serializer를 직접 호출합니다. Site graph는 Person/WebSite IDs와 content, project는 CreativeWork identity/content/URL을 확인합니다. |
| Data/state/owner | Actual validated content fixture와 explicit public URL이 expected claims의 source입니다. Test는 pure data/serialized string을 관찰합니다. |
| Failure·absence·fallback | Project fixture가 없으면 explicit error를 throw합니다. Negative properties가 없어야 하며 `</script>`는 `\u003c/script\u003e`를 포함해야 합니다. |
| 보장/비보장 | Semantic field mapping, deliberate claim restraint, critical HTML embedding escape를 deterministic하게 증명합니다. React component render, browser parser, CSP, external schema validator, rich result eligibility는 증명하지 않습니다. |
| 후속 연결 | Category 내 structured-data story의 마지막 regression boundary입니다. |

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

프로젝트 명령은 실행하지 않았습니다. `git clone --branch web/portfolio --single-branch --filter=blob:none https://github.com/seungwoo7050/42-archive.git /mnt/data/portfolio-branch-checkout`가 `Could not resolve host: github.com`으로 종료되어 exact-SHA checkout과 의존성 설치가 불가능했습니다. 따라서 아래 결론은 GitHub connector로 조회한 해당 SHA의 diff와 resulting file에 대한 정적 검토입니다.

**다음 commit 연결**

Category 내 structured-data story의 마지막 regression boundary입니다.
<!-- learner:end commit-c5938ea4b4f8 -->


## 6. Invariant evolution

<!-- learner:start thread-invariant-evolution -->
| Commit/구간 | 상태 | 근거 기반 설명 |
| --- | --- | --- |
| 228e40a48d64 | Security boundary introduced | Raw script insertion is centralized and markup-significant characters are escaped. |
| ee98415be696 | Semantic model introduced | Person and WebSite share stable IDs and an author relation. |
| ae4ec172e45a | Production-gated integration | The site graph is emitted globally only for production. |
| 7e09745d409e | Extended | Projects become linked CreativeWork records without unsupported claims. |
| f7bd33a8b403 | Route-integrated | Both project detail rendering branches emit the same record only after route validation. |
| c5938ea4b4f8 | Deterministically verified | Positive semantics, negative claims, and script-terminator escaping are protected. |
<!-- learner:end thread-invariant-evolution -->

## 7. Failure → Fix → Test 관계

<!-- learner:start thread-failure-fix-test -->
| Failure/위험 | Fix/결정 | Test/증거 |
| --- | --- | --- |
| Plain JSON text could terminate the script context | Dedicated serializer escapes `<`, `>`, `&` before raw insertion | Regression uses `</script>` and checks Unicode escapes |
| Structured claims could overstate evidence | Factories map only authoritative content fields | Negative tests forbid award and aggregateRating |
| Multi-design detail paths could diverge | Page boundary wraps both dedicated and fallback render branches | Static inspection confirms both branches; no browser branch-count test in this commit |
<!-- learner:end thread-failure-fix-test -->

## 8. Ownership/state/responsibility 변화

<!-- learner:start thread-ownership -->
| 시점 | Owner | 책임 변화 |
| --- | --- | --- |
| Before | No owner | No structured-data model or safe insertion boundary. |
| 228 | Serializer + `StructuredData` | Own text safety and the only raw-script insertion point. |
| ee/7e | Metadata factories | Own evidence-to-Schema.org mapping and stable IDs. |
| ae/f7 | Root and project page boundaries | Own production gating, public origin, and rendering lifecycle. |
| c593 | Focused unit contract | Protects semantics, restraint, and escaping without claiming browser/schema validation. |
<!-- learner:end thread-ownership -->

## 9. 최종 Thread 상태와 실행 흐름

<!-- learner:start thread-final-state -->
**최종 상태**

Production pages expose one linked Person/WebSite graph and each valid project detail can expose a CreativeWork linked to the same Person. Template previews emit no structured claims. Every record passes through a dedicated serializer that prevents content from closing or altering the JSON-LD script context, and tests explicitly reject unsupported award/rating claims.

**코드 없는 실행 흐름**

1. A production page resolves the validated public origin and authoritative portfolio content.
2. The root or project factory maps only supported fields to stable Schema.org IDs and references.
3. The page passes the resulting record to the shared `StructuredData` component; template mode renders nothing.
4. The component serializes JSON and escapes markup-significant characters before raw insertion into an `application/ld+json` script.
5. Focused tests compare graph identity/content, forbid unsupported claims, and reproduce the `</script>` embedding boundary.
<!-- learner:end thread-final-state -->

## 10. Learning completion check

<!-- learner:start thread-completion-check -->
- [x] 각 SHA의 exact diff/tree를 GitHub connector로 정적 확인했습니다.
- [x] 보장과 비보장을 commit별로 구분했습니다.
- [x] test technique과 proves/does-not-prove를 구분했습니다.
- [x] 최종 흐름을 코드 없이 재구성했습니다.
- [x] 프로젝트 명령은 DNS 제한으로 실행하지 못했으며 그 사실을 모든 실행 증거에 명시했습니다.
<!-- learner:end thread-completion-check -->
===== END FILE: 05-safe-jsonld-serialization-and-structured-claims.md =====

===== BEGIN FILE: README.md =====
# 06-seo-security-and-machine-readable-output

> Repository: `seungwoo7050/42-archive`
>
> Branch: `web/portfolio`
>
> Artifact scope: this category only

## Category purpose

Canonical metadata, crawler indexing policy, content-declared URL integrity, production publication trust, and safely embedded machine-readable claims를 historical implementation sequence로 학습합니다.

## Phase 1 category audit and frozen boundary

- **Category boundary:** SEO/crawler/publication URL/structured-data safety로 한정했습니다.
- **Removed from this category:** `902eddcef875`, `ba8da56d3fcf`, `f63c978c71c9`는 query navigation과 UI link transport의 owner인 category 02/03과 중복되어 제거했습니다.
- **Split:** 기존 혼합 Thread 01을 content-declared internal route integrity와 production origin/publication URL safety로 분리했습니다.
- **Moved within category:** `adc392157f70`은 route metadata Thread에서 sitemap/indexing Thread로 이동했습니다.
- **Added material commits:** route validation regression `3353032ba23b`, publication readiness chain, metadata ownership transfer `55b6061e0052 → 67aabeab1553`, indexing unit/E2E evidence를 추가했습니다.
- **Intentional reuse:** `55b6061e0052`는 metadata shape와 indexing directive, `67aabeab1553`는 metadata-origin integration과 page-indexing integration, `fb3d18fd660b`는 readiness regression과 indexing regression이라는 서로 다른 file/decision 관점에서 두 Thread에 등장합니다. 같은 code claim은 중복 서술하지 않습니다.
- **Final coverage:** 5 Threads, 31 unique SHAs. URL integrity, publication trust, canonical metadata, crawler policy, safe JSON-LD를 독립 engineering stories로 덮습니다.

## Frozen Thread index

| 순서 | 파일 | Thread | Commits |
| --- | --- | --- | --- |
| 1 | `01-content-declared-internal-route-integrity.md` | Content-declared internal route integrity | 4 |
| 2 | `02-production-origin-and-publication-url-safety.md` | Production origin and publication URL safety | 9 |
| 3 | `03-route-aware-metadata-and-canonical-identity.md` | Route-aware metadata and canonical identity | 8 |
| 4 | `04-indexing-robots-and-sitemap-policy.md` | Indexing, robots, and sitemap policy | 7 |
| 5 | `05-safe-jsonld-serialization-and-structured-claims.md` | Safe JSON-LD serialization and structured claims | 6 |

## Historical validation basis

- `commit/commit-importance.md` on `web/portfolio` describes the branch as one independent, linear 476-commit history from `cce7dd020563` through `aff0acdd4cf9`. Every SHA below was matched to that branch-local classification and its exact commit object/diff.
- Exact commit objects/diffs and relevant exact-SHA files were retrieved through the connected GitHub repository interface.
- Earliest selected ancestry anchors were additionally compared against `web/portfolio`; the branch was ahead with the selected SHA as merge base.
- Direct local clone failed because the execution container could not resolve `github.com`; therefore no project command result is claimed in this workbook.

## Phase 2 status

<!-- learner:start readme-phase2-status -->
### Phase 2 completion result

- Frozen scaffold files: `6`
- Completed counterparts: `6`
- Development Threads: `5`
- Unique referenced SHAs: `31`
- Project test execution: not performed because repository checkout failed with DNS resolution error.
- Historical evidence: exact commit diffs/files retrieved through the GitHub connector; no later HEAD code was projected backward.
- Local deliverable validation: file-set equality, fixed-field normalization, frozen SHA-256 hashes, marker completion, commit-map metadata, Markdown fence balance, and ZIP path constraints were executed by this generator.
<!-- learner:end readme-phase2-status -->

## Validation matrix

<!-- learner:start readme-validation -->
| 검증 | 결과 |
| --- | --- |
| scaffold/completed file set | PASS — README와 5 Thread의 relative paths가 정확히 일치합니다. |
| frozen scaffold hash unchanged | PASS — completed 생성 전후 SHA-256 manifest가 동일합니다. |
| fixed fields preserved | PASS — learner marker 내부를 제거한 normalized text가 pair마다 동일합니다. |
| all SHAs branch-scoped | PASS — 31 unique SHAs를 branch-local complete classification 및 exact commit retrieval로 교차 확인했습니다. |
| no unfinished learner region | PASS — completed에 blank learner table/checklist가 남지 않습니다. |
| ZIP contains only category | PASS — top-level은 category 하나이며 scaffold/completed만 포함합니다. |
<!-- learner:end readme-validation -->

## Reading order

1. Start with content-declared internal route integrity to understand route vocabulary before publication.
2. Continue to production origin/readiness because metadata and crawler output trust this boundary.
3. Study canonical metadata ownership and per-route exports.
4. Study aligned page robots, robots.txt, and sitemap publication.
5. Finish with JSON-LD semantic restraint and script-context safety.
===== END FILE: README.md =====

