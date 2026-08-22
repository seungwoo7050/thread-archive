===== BEGIN FILE: 01-geometric-contracts-to-first-image.md =====
# Thread 1. From geometric contracts to the first rendered image

## 1. Thread 목표

공통 `Shape`/`HitRecord` 계약에서 시작해 유효한 `.rt` 입력이 하나의 `Scene`으로 구성되고, 카메라 광선·조명·그림자·직렬 픽셀 루프를 거쳐 P3 PPM과 체크섬으로 노출되는 최초의 완전한 실행 경로를 복원합니다. 이 Thread의 직렬 렌더러는 이후 BVH와 멀티스레드 구현이 보존해야 하는 의미적 기준선입니다.

### Source significance

> This progression turns isolated mathematical and geometric types into an externally usable renderer.
> The important sequence is not merely feature accumulation: the shape contract gives every primitive
> one result model, the scene supplies one authoritative selection boundary, the parser constructs
> that state, camera and shading transform it into pixel colors, and the renderer/output/CLI layers
> preserve a deterministic artifact. The serial implementation also becomes the semantic baseline
> against which later BVH and threaded versions are judged.

### 이 Thread에 연결된 source invariant

- Parsed scene directives must be syntactically valid, finite, in range, and geometrically non-degenerate; required singleton directives must be present and not duplicated.
- Golden checksums and exact PPM bytes remain stable across repeat runs, acceleration modes, and tested worker counts unless an intentional rendering contract changes.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `Shape::intersect`와 `HitRecord`가 geometry, scene traversal, shading 사이의 경계를 어떻게 고정하는가?
- `Scene::intersect`가 최근접 hit와 exact equal-`t` 선택을 어떤 순서 규칙으로 결정하는가?
- parser는 line-local 오류와 whole-file 누락을 어떻게 구분하고, 불완전한 장면 반환을 어떻게 막는가?
- 화면 좌표가 정규화된 카메라 광선으로 변환되는 실제 caller/callee 흐름은 무엇인가?
- 직접광과 shadow ray의 시작·끝 구간이 self-intersection과 light 뒤쪽 geometry를 어떻게 배제하는가?
- 한 픽셀의 색이 byte RGB, 체크섬, P3 파일, CLI exit status로 이어지는 순서를 어디서 확인할 수 있는가?

## 3. 완료 기준

- [x] 각 커밋의 해당 SHA에서 실제 선언·정의·호출 지점을 기록했습니다.
- [x] parser 입력부터 PPM 파일까지의 전체 caller → callee 흐름을 실제 심볼과 파일 경로로 설명할 수 있습니다.
- [x] geometry hit 계약, scene candidate selection, shading 정책, image storage, output/CLI 책임을 구분할 수 있습니다.
- [x] 잘못된 장면이 출력 파일을 만들지 않는 이유를 실행 순서와 smoke test로 연결했습니다.
- [x] 후속 BVH·tile renderer가 무엇을 최적화하되 어떤 결과를 보존해야 하는지 직렬 기준선에서 설명할 수 있습니다.
- [x] 모든 참조 SHA가 `cpp/miniRT` branch HEAD의 ancestry에 속하는지 확인했습니다.
- [ ] 해당 SHA checkout에서 build/test/benchmark 명령을 직접 실행했습니다. 로컬 외부 네트워크와 checkout이 제공되지 않아 실행 evidence는 만들지 않았습니다.

### 검증 범위

- 지정 branch HEAD: `7d08c7c13fa68c3e60eea3c7014658b0a133e6f0`
- 각 참조 SHA는 Thread 내부의 연속 compare chain에서 `behind_by = 0`, merge base가 선행 SHA였고, Thread 종료 SHA도 branch HEAD의 조상으로 확인했습니다.
- 구현 설명은 해당 commit의 diff/file content를 기준으로 작성했으며, final HEAD의 후속 API를 과거 SHA에 소급하지 않았습니다.
- 테스트와 benchmark는 source mechanism과 production path만 검사했습니다. 실행 결과, sanitizer 결과, wall-clock 수치는 기록하지 않았습니다.

## 4. Commit map

1. `f3f1d04cc836` — `feat(geometry): hit와 도형 교차 계약 정의`
 - Importance: S
 - Tags: ARCH, GEOMETRY, SCENE
 - Source-defined role: Establishes the common shape/hit contract that lets scene traversal and shading consume heterogeneous geometry.

2. `2a01cb406d9d` — `feat(scene): 카메라·조명과 장면 aggregate 구성`
 - Importance: A
 - Tags: ARCH, SCENE
 - Source-defined role: Creates the scene aggregate that owns the state needed to trace an image.

3. `41a1d6bbe5ef` — `feat(scene): 선형 최근접 교차 탐색 구현`
 - Importance: A
 - Tags: CORE, SCENE
 - Source-defined role: Defines the linear closest-hit reference semantics later preserved by acceleration.

4. `3545eb1e82df` — `feat(parser): 소스 위치 오류와 line tokenization 구성`
 - Importance: B
 - Tags: PARSER, PRACTICAL
 - Source-defined role: Establishes source-located parser diagnostics and line tokenization.

5. `6bff18bf0bac` — `feat(parser): 줄 단위 지시어 dispatch 기반 구성`
 - Importance: A
 - Tags: ARCH, PARSER
 - Source-defined role: Defines the directive-dispatch grammar boundary.

6. `1e1fda47d913` — `feat(parser): 필수 지시어 검증과 입력 loader 완성`
 - Importance: A
 - Tags: PARSER, INTEGRATION
 - Source-defined role: Completes required-directive validation and file/text scene loading.

7. `e6da5f987b97` — `feat(camera): 화면 좌표를 카메라 광선으로 변환`
 - Importance: A
 - Tags: CORE, RAY_PIPELINE
 - Source-defined role: Converts pixel coordinates into camera rays.

8. `e8b7dc42a52c` — `feat(render): 직접광과 그림자 추적 구현`
 - Importance: S
 - Tags: CORE, RAY_PIPELINE, RISK
 - Source-defined role: Defines ambient/direct lighting and shadow visibility.

9. `c742b2401e52` — `feat(renderer): 직렬 이미지 렌더링 구현`
 - Importance: S
 - Tags: ARCH, CORE, RAY_PIPELINE
 - Source-defined role: Executes the complete serial image-rendering loop.

10. `1bc7cacd30aa` — `feat(output): PPM 직렬화와 이미지 체크섬 구현`
 - Importance: A
 - Tags: OUTPUT, DETERMINISM
 - Source-defined role: Publishes a P3 representation and deterministic checksum.

11. `b983f0ea2744` — `feat(cli): 장면 렌더링 명령 연결`
 - Importance: B
 - Tags: CLI, INTEGRATION
 - Source-defined role: Connects the pipeline to the CLI.

12. `d05a6ab48bb1` — `test(render): 장면 렌더링 smoke 검사 추가`
 - Importance: B
 - Tags: TEST, INTEGRATION
 - Source-defined role: Verifies the first complete valid and invalid command paths.

## 5. Commit별 학습 기록

### 5.1 `f3f1d04cc836` — `feat(geometry): hit와 도형 교차 계약 정의`

- Importance: S
- Tags: ARCH, GEOMETRY, SCENE
- Thread order: 1/12

#### Source에서 확정된 역할

- Development Thread role: Establishes the common shape/hit contract that lets scene traversal and shading consume heterogeneous geometry.

#### S-level architecture와 invariant 복원

- **직전 관련 상태:** 공통 교차 결과가 없으면 각 도형이 서로 다른 반환 형식이나 출력 인자를 사용하게 되고, 장면 순회와 조명 코드는 구·평면·원기둥의 구체 타입을 알아야 합니다. 이 SHA는 이후 subsystem이 공유할 최초의 geometry 결과 계약을 만드는 지점입니다.
- **핵심 구현 결정:** `include/ray/geometry.hpp`에 다형적 `Shape`를 두고 virtual destructor, `[t_min, t_max]`를 받는 `intersect`, material 접근을 공통 interface로 정의합니다. `HitRecord`는 교차 거리 `t`, 점, 정규화된 법선, material 값, 원본 `Shape` 포인터, 앞면 여부를 한 결과로 묶습니다. `HitRecord::setFaceNormal`은 incoming ray와 outward normal의 내적 부호로 `frontFace`를 정한 뒤 저장 법선을 ray 반대쪽으로 맞춥니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/geometry.hpp — `Shape`, `HitRecord`, `HitRecord::setFaceNormal`
 - src/geometry.cpp — 해당 시점의 공통 geometry 구현
- **caller → callee / data flow:** concrete shape의 `intersect(ray, tMin, tMax, record)` → candidate parameter/point/outward normal 계산 → `setFaceNormal` → material 값과 source shape identity 기록 → scene/shading이 동일 record 소비
- **ownership·state transition:** `HitRecord`가 point·normal·material을 값으로 보유합니다. `HitRecord::shape`는 `const Shape*` 비소유 포인터이므로 record 수명보다 shape owner의 수명이 길어야 합니다. normal은 outward normal 그대로가 아니라 incoming ray에 대해 방향이 정해진 상태로 저장됩니다.
- **failure/edge branch:** 이 SHA는 호출자가 주는 유효 구간 밖의 root를 거부할 수 있는 interface만 제공합니다. shape lifetime을 record가 연장하지 않으며, 구체 도형의 완전한 교차 수식과 scene winner 규칙은 아직이 commit의 보장이 아닙니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 모든 도형이 동일한 hit 표현을 생산하고, shading은 concrete type 없이 point·oriented normal·material·identity를 읽을 수 있습니다.
- **이 SHA가 보장하지 않는 것:** 구·평면·원기둥별 계산, scene-level closest selection, exact equal-`t` 규칙은 후속 commit에서 확인해야 합니다.
- **직접 확인/후속 evidence:** 후속 `41a1d6bbe5ef`, `e8b7dc42a52c`, BVH 관련 SHA가 같은 record를 실제로 소비하는 것을 역사 순서대로 대조했습니다. 실행 명령은 수행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: 이 Thread의 시작점
- 다음 Thread commit: `2a01cb406d9d`
- 이 commit이 다음 단계에 제공하는 것: `2a01cb406d9d`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.2 `2a01cb406d9d` — `feat(scene): 카메라·조명과 장면 aggregate 구성`

- Importance: A
- Tags: ARCH, SCENE
- Thread order: 2/12

#### Source에서 확정된 역할

- Development Thread role: Creates the scene aggregate that owns the state needed to trace an image.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** 공통 shape/hit 계약만으로는 한 장의 이미지를 추적하는 데 필요한 해상도, 카메라, ambient/background, 조명과 도형 집합을 한곳에서 관리할 수 없습니다.
- **핵심 구현 결정:** `Scene` aggregate에 width/height, 필수 directive 존재 여부, ambient/background, `Camera`, lights, shapes를 모읍니다. 이 SHA의 shape 저장은 `std::vector<std::shared_ptr<Shape>>`이므로 final HEAD의 `unique_ptr`·private storage를 소급하지 않습니다. `addLight`와 aggregate field가 parser와 renderer의 공통 상태를 만듭니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/scene.hpp — `Scene`, `Camera`, light/shape storage
 - src/scene.cpp — scene aggregate 동작
- **caller → callee / data flow:** parser 또는 caller가 directive 값을 검증 → `Scene` field 설정/shape·light 추가 → camera/shading/renderer가 같은 aggregate를 읽음
- **ownership·state transition:** 이 시점에는 Scene과 외부 `shared_ptr` 보유자가 shape ownership을 공유할 수 있습니다. required-directive flags는 장면 구성 완료 여부를 나타내지만 whole-file validation은 아직 parser 후속 commit에 있습니다.
- **failure/edge branch:** aggregate 자체는 누락·중복 directive를 거부하지 않고, 가속 구조나 invalidation 상태도 갖지 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 한 장면의 렌더링 입력을 하나의 값 경계에서 전달할 수 있습니다.
- **이 SHA가 보장하지 않는 것:** 최근접 교차, parser 완료 조건, acceleration ownership은 각각 후속 commit의 책임입니다.
- **직접 확인/후속 evidence:** `2a01cb406d9d`의 field와 저장 타입을 해당 SHA에서 확인하고, `ef5320a83c27`의 private/immutable 상태를 이 시점에 소급하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `f3f1d04cc836`
- 다음 Thread commit: `41a1d6bbe5ef`
- 이 commit이 다음 단계에 제공하는 것: `41a1d6bbe5ef`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.3 `41a1d6bbe5ef` — `feat(scene): 선형 최근접 교차 탐색 구현`

- Importance: A
- Tags: CORE, SCENE
- Thread order: 3/12

#### Source에서 확정된 역할

- Development Thread role: Defines the linear closest-hit reference semantics later preserved by acceleration.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** Scene이 도형을 보유해도 광선에 대해 어떤 hit를 authoritative result로 선택할지 정해지지 않으면 shading과 향후 BVH가 비교할 기준이 없습니다.
- **핵심 구현 결정:** `Scene::intersect`가 shape 저장 순서로 선형 순회합니다. 현재 `closest`를 각 `Shape::intersect`의 upper bound로 넘기고 candidate가 성공할 때 authoritative record와 `closest`를 교체합니다. interval upper bound가 포함되고 candidate가 성공하면 그대로 갱신되므로 exact equal-`t`에서는 뒤에 순회한 shape가 winner가 됩니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/scene.cpp — `Scene::intersect`
 - include/ray/scene.hpp — scene query 선언
- **caller → callee / data flow:** ray + caller interval → shapes 순서대로 primitive dispatch → 현재 closest 이하 candidate → record 교체 → 마지막 winner 반환
- **ownership·state transition:** `closest`와 output `HitRecord`가 loop의 authoritative state입니다. record의 non-owning shape pointer는 Scene이 보유한 shape를 가리킵니다.
- **failure/edge branch:** 가속 없이 모든 shape를 검사하므로 correctness 기준은 생기지만 work는 O(N)입니다. equal-`t` winner는 단순히 “먼저 발견한 것”이 아니라 뒤 index임을 보존해야 합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** caller 구간 안의 최근접 hit와 저장 순서에 따른 exact tie semantics를 정의합니다.
- **이 SHA가 보장하지 않는 것:** primitive work 계측과 BVH는 아직 없습니다.
- **직접 확인/후속 evidence:** 후속 `9a7f29b5d78a`의 `(t, original index)` candidate rule이 이 선형 기준을 명시적으로 재현하는 것을 대조했습니다.

#### Thread 내 연결

- 이전 Thread commit: `2a01cb406d9d`
- 다음 Thread commit: `3545eb1e82df`
- 이 commit이 다음 단계에 제공하는 것: `3545eb1e82df`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.4 `3545eb1e82df` — `feat(parser): 소스 위치 오류와 line tokenization 구성`

- Importance: B
- Tags: PARSER, PRACTICAL
- Thread order: 4/12

#### Source에서 확정된 역할

- Development Thread role: Establishes source-located parser diagnostics and line tokenization.

#### B-level 구현 역할 복원

- **직전 관련 상태:** 장면 문자열을 해석할 때 단순 예외 메시지만 있으면 어느 파일의 몇 번째 줄이 잘못됐는지 caller가 알 수 없고, 주석·공백·token 처리를 directive마다 반복하게 됩니다.
- **핵심 구현 결정:** `ParseError`에 source name과 line을 저장하고 `source[:line]: message` 형태를 만듭니다. 공통 trim과 whitespace tokenization helper를 추가해 line-local parsing의 기반을 둡니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/parser.hpp — `ParseError`와 parser 선언
 - src/parser.cpp — trim/tokenization 및 오류 문자열 구성
- **caller → callee / data flow:** source text line → trim/token split → validation failure → source/line을 포함한 `ParseError`
- **ownership·state transition:** line number가 0보다 큰 오류는 실제 줄에 귀속되고, whole-file/file-open 오류는 후속 loader에서 line 0으로 구분할 수 있는 표현이 생깁니다.
- **failure/edge branch:** 이 commit은 완전한 directive grammar나 required singleton 검증을 아직 제공하지 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 파서 오류를 입력 위치와 연결하고 모든 directive가 같은 tokenization 기반을 사용할 수 있습니다.
- **이 SHA가 보장하지 않는 것:** dispatch와 loader 완료 조건은 다음 commit들에 남아 있습니다.
- **직접 확인/후속 evidence:** 후속 invalid fixture test가 source line을 assertion하는 경로와 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `41a1d6bbe5ef`
- 다음 Thread commit: `6bff18bf0bac`
- 이 commit이 다음 단계에 제공하는 것: `6bff18bf0bac`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.5 `6bff18bf0bac` — `feat(parser): 줄 단위 지시어 dispatch 기반 구성`

- Importance: A
- Tags: ARCH, PARSER
- Thread order: 5/12

#### Source에서 확정된 역할

- Development Thread role: Defines the directive-dispatch grammar boundary.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** tokenization만으로는 identifier별 arity·범위·중복 규칙을 한곳에서 적용하거나 unknown directive를 fail closed로 처리할 수 없습니다.
- **핵심 구현 결정:** 물리적 줄을 순회하며 comment 제거, trim, 빈 줄 skip, token split, identifier dispatch 순서로 처리합니다. directive handler는 exact arity와 값 검증을 담당하고, unknown identifier는 source-located `ParseError`로 거부합니다. singleton duplicate 검사와 non-degenerate 값 검증도 handler 경계에 둡니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/parser.cpp — line loop, directive dispatch, directive별 handler/validator
- **caller → callee / data flow:** 각 physical line → comment 제거 → tokens → first token dispatch → validated Scene mutation 또는 즉시 ParseError
- **ownership·state transition:** 한 줄은 검증이 끝난 뒤에만 Scene을 변경합니다. unknown/surplus token은 무시되지 않습니다.
- **failure/edge branch:** line-local grammar는 생기지만 파일 전체가 끝났을 때 R/A/C 누락을 막는 final validation은 아직 없습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** directive 문법과 Scene mutation 사이에 fail-closed 경계를 만듭니다.
- **이 SHA가 보장하지 않는 것:** 불완전하지만 line-local하게 유효한 장면이 반환되지 않도록 하는 whole-file 검사는 `1e1fda47d913`에서 완성됩니다.
- **직접 확인/후속 evidence:** 해당 SHA의 parser diff에서 dispatch와 handler 추가를 확인했습니다. 후속 material token grammar를이 시점에 소급하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `3545eb1e82df`
- 다음 Thread commit: `1e1fda47d913`
- 이 commit이 다음 단계에 제공하는 것: `1e1fda47d913`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.6 `1e1fda47d913` — `feat(parser): 필수 지시어 검증과 입력 loader 완성`

- Importance: A
- Tags: PARSER, INTEGRATION
- Thread order: 6/12

#### Source에서 확정된 역할

- Development Thread role: Completes required-directive validation and file/text scene loading.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** 각 줄이 유효해도 필수 singleton인 resolution, ambient, camera 중 하나가 빠진 파일은 렌더링 가능한 Scene이 아닙니다.
- **핵심 구현 결정:** 전체 line 처리 후 R/A/C 존재 flags를 검사하고 누락 시 line 0의 `ParseError`를 던집니다. `parseSceneText`, 파일 stream을 여는 경로, public loader를 연결하고 파일 open failure도 source-level error로 만듭니다. 유효/무효 `.rt` fixture가 추가됩니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/parser.hpp — text/file loader API
 - src/parser.cpp — `parseSceneText`, file load, required-directive final validation
 - scenes/basic.rt
 - scenes/invalid.rt
- **caller → callee / data flow:** file open → text read → line-local parse/mutation → EOF required singleton validation → 완성된 Scene 반환
- **ownership·state transition:** 성공 반환이 Scene completeness의 commit point입니다. line-local 오류는 실제 line, 파일 open/whole-file 누락은 line 0으로 표현됩니다.
- **failure/edge branch:** 어느 단계에서든 `ParseError`가 나오면 Scene은 caller에 성공값으로 반환되지 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 유효한 directive 집합만 renderer로 전달됩니다.
- **이 SHA가 보장하지 않는 것:** CLI가 loader를 호출해 output 전에 실패하는 경로는 `b983f0ea2744`에서 외부 계약이 됩니다.
- **직접 확인/후속 evidence:** invalid fixture와 후속 smoke script의 no-output assertion을 production 순서에 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `6bff18bf0bac`
- 다음 Thread commit: `e6da5f987b97`
- 이 commit이 다음 단계에 제공하는 것: `e6da5f987b97`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.7 `e6da5f987b97` — `feat(camera): 화면 좌표를 카메라 광선으로 변환`

- Importance: A
- Tags: CORE, RAY_PIPELINE
- Thread order: 7/12

#### Source에서 확정된 역할

- Development Thread role: Converts pixel coordinates into camera rays.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** Scene의 camera directive 값은 존재하지만 image coordinate를 world-space ray로 바꾸는 정규직교 frame과 FOV/aspect mapping이 없습니다.
- **핵심 구현 결정:** `buildCameraFrame`이 camera direction을 정규화하고, forward와 평행하지 않은 world-up 후보를 고른 뒤 cross product로 right/up을 구성합니다. `makeCameraRay`는 pixel sample을 normalized screen coordinate로 바꾸고 aspect/FOV scale과 Y 반전을 적용한 후 방향을 다시 정규화합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/camera.hpp — camera frame/ray API
 - src/camera.cpp — `buildCameraFrame`, `makeCameraRay`
- **caller → callee / data flow:** camera position/direction/FOV + image dimensions + sample `(x,y)` → precomputed frame → NDC/screen coordinate → normalized world ray
- **ownership·state transition:** frame은 렌더 전체에서 재사용 가능한 derived read-only 값입니다. 방향이 +Z에 가까운 경우 up 후보를 바꾸어 cross product 붕괴를 피합니다.
- **failure/edge branch:** safe dimension 처리로 분모 0을 피하지만 parser가 보장해야 할 실제 positive resolution과 finite/nonzero direction 정책을 대신하지 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 각 화면 sample이 일관된 camera ray로 변환됩니다.
- **이 SHA가 보장하지 않는 것:** ray가 scene color로 변환되는 shading과 pixel loop는 다음 commit들에 있습니다.
- **직접 확인/후속 evidence:** 후속 serial renderer가 `(x+0.5, y+0.5)`를 전달하는 호출 경로와 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `1e1fda47d913`
- 다음 Thread commit: `e8b7dc42a52c`
- 이 commit이 다음 단계에 제공하는 것: `e8b7dc42a52c`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.8 `e8b7dc42a52c` — `feat(render): 직접광과 그림자 추적 구현`

- Importance: S
- Tags: CORE, RAY_PIPELINE, RISK
- Thread order: 8/12

#### Source에서 확정된 역할

- Development Thread role: Defines ambient/direct lighting and shadow visibility.

#### S-level architecture와 invariant 복원

- **직전 관련 상태:** camera ray와 hit 계약이 있어도 miss 색, ambient term, Lambertian direct light, visibility를 계산하는 terminal tracing path가 없습니다.
- **핵심 구현 결정:** `traceRay`가 miss에서 background를 반환하고 hit에서는 ambient contribution을 시작값으로 둡니다. 각 light에 대해 거리와 normalized light direction을 구하고 `dot(normal, lightDirection)`이 양수일 때만 shadow query를 수행합니다. shadow ray는 normal 방향으로 `kRayTMin`만큼 이동한 점에서 시작하며 upper bound를 `distance - kRayTMin`로 제한해 light 뒤 도형을 occluder로 보지 않습니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/renderer.hpp — tracing/shading API
 - src/shading.cpp — `traceRay`, direct-light/shadow path
 - src/scene.cpp — occlusion query가 통과하는 scene traversal
- **caller → callee / data flow:** primary ray → `Scene::intersect` → miss background 또는 hit ambient → each light: positive diffuse → bounded shadow ray → visible direct contribution → clamped color의 입력
- **ownership·state transition:** shadow ray는 원래 primary ray와 별개이며 origin offset과 `[kRayTMin, lightDistance-kRayTMin]` 구간을 갖습니다. material은 `HitRecord`에서 값으로 읽습니다.
- **failure/edge branch:** 거리 epsilon 이하 또는 diffuse term 이하이면 shadow query 자체를 생략합니다. 이 SHA의 `maxDepth` 인자는 아직 recursive material에 소비되지 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** self-intersection을 줄이는 시작 offset과 light segment에 한정된 visibility를 포함한 deterministic direct lighting을 정의합니다.
- **이 SHA가 보장하지 않는 것:** 반사 recursion은 `85583e1e9beb`에서 추가됩니다. 이 commit은 random sampling이나 global illumination을 제공하지 않습니다.
- **직접 확인/후속 evidence:** 직렬 renderer와 material thread에서이 함수를 소비하는 상태를 각 SHA별로 분리해 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `e6da5f987b97`
- 다음 Thread commit: `c742b2401e52`
- 이 commit이 다음 단계에 제공하는 것: `c742b2401e52`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.9 `c742b2401e52` — `feat(renderer): 직렬 이미지 렌더링 구현`

- Importance: S
- Tags: ARCH, CORE, RAY_PIPELINE
- Thread order: 9/12

#### Source에서 확정된 역할

- Development Thread role: Executes the complete serial image-rendering loop.

#### S-level architecture와 invariant 복원

- **직전 관련 상태:** camera와 shading 함수가 있어도 모든 pixel을 정확한 순서와 quantization 규칙으로 채우는 image-level 실행 경로가 없습니다.
- **핵심 구현 결정:** `renderScene`이 `Image`를 만들고 row-major로 모든 `(x,y)`를 순회합니다. sample은 pixel center `(x+0.5, y+0.5)` 하나이며, camera ray → `traceRay` → `[0,1]` clamp → `std::lround(component*255)` 순서로 byte RGB를 저장합니다. 이 직렬 순서와 산술이 향후 BVH/threads의 의미적 기준선입니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/renderer.hpp — `RenderSettings`, `Image`, `renderScene`
 - src/renderer.cpp — serial pixel loop
- **caller → callee / data flow:** Scene + settings → camera frame → row-major pixel center → camera ray → trace → clamp/quantize → contiguous RGB bytes → Image
- **ownership·state transition:** 한 cursor 또는 pixel offset이 image storage를 순차적으로 채웁니다. 렌더 반환 전 모든 pixel이 완성됩니다.
- **failure/edge branch:** 이 시점의 `Image` allocation은 `width*height*3` 산술 overflow를 아직 검사하지 않습니다. `samplesPerPixel`, `tMin/tMax` 설정도 실제 loop에서 모두 소비되는 상태가 아닙니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 한 scene에 대해 deterministic single-sample image byte baseline을 만듭니다.
- **이 SHA가 보장하지 않는 것:** 안전한 image representation은 Thread 6, tile/parallel execution은 Thread 5에서 보완됩니다.
- **직접 확인/후속 evidence:** 후속 exact PPM/checksum 및 worker-count equivalence가이 pixel kernel 결과를 비교하는 것을 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `e8b7dc42a52c`
- 다음 Thread commit: `1bc7cacd30aa`
- 이 commit이 다음 단계에 제공하는 것: `1bc7cacd30aa`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.10 `1bc7cacd30aa` — `feat(output): PPM 직렬화와 이미지 체크섬 구현`

- Importance: A
- Tags: OUTPUT, DETERMINISM
- Thread order: 10/12

#### Source에서 확정된 역할

- Development Thread role: Publishes a P3 representation and deterministic checksum.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** 완성된 RGB byte buffer가 있어도 외부 artifact와 repeat-run 비교값이 없습니다.
- **핵심 구현 결정:** `writePpm`이 P3 header와 RGB 값을 text로 직렬화하고, checksum 함수가 dimensions의 low/high byte와 pixel bytes를 순서대로 FNV-1a 방식으로 반영해 16진수 문자열을 만듭니다. 이 SHA의 offset basis는 `1469598103934665603ULL`이며 표준 상수 수정 전 상태입니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/output.hpp — PPM/checksum API
 - src/output.cpp — `writePpm`, image checksum
- **caller → callee / data flow:** Image dimensions/pixels → P3 text 또는 dimension bytes + pixel bytes → checksum hex
- **ownership·state transition:** PPM과 checksum 모두 quantized bytes를 source로 사용합니다. path writer는이 시점에 대상 파일을 직접 열어 씁니다.
- **failure/edge branch:** stream/write/atomic replacement safety와 public storage validation은 아직 없습니다. 초기 checksum constant는 `89c3c7269877`에서 수정됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 외부에서 읽을 수 있는 P3 representation과 repeat-run comparison surface를 제공합니다.
- **이 SHA가 보장하지 않는 것:** 이 시점 checksum 값은 standard FNV-1a와 일치하지 않으며, 파일 실패 시 기존 대상 보존도 보장하지 않습니다.
- **직접 확인/후속 evidence:** Thread 6의 checksum fix와 golden test를 별도 역사 단계로 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `c742b2401e52`
- 다음 Thread commit: `b983f0ea2744`
- 이 commit이 다음 단계에 제공하는 것: `b983f0ea2744`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.11 `b983f0ea2744` — `feat(cli): 장면 렌더링 명령 연결`

- Importance: B
- Tags: CLI, INTEGRATION
- Thread order: 11/12

#### Source에서 확정된 역할

- Development Thread role: Connects the pipeline to the CLI.

#### B-level 구현 역할 복원

- **직전 관련 상태:** library-level parse/render/output API가 있어도 사용자가 입력·출력 경로와 checksum 요청을 전달하고 실패를 exit status로 관찰할 수 없습니다.
- **핵심 구현 결정:** `src/main.cpp`가 3개 또는 optional `--checksum`을 포함한 4개 인자를 검증합니다. 성공 경로는 scene load → render → PPM write → optional checksum 출력 순서이며, usage 오류는 2, runtime/parse 오류는 1, 성공은 0으로 종료합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/main.cpp — argument parsing과 top-level try/catch
 - Makefile — CLI source 연결
- **caller → callee / data flow:** argv → usage validation → `loadScene` → `renderScene` → `writePpm` → optional checksum stdout → exit 0; exception → stderr → exit 1
- **ownership·state transition:** output 생성은 parsing 이후에만 시작하므로 invalid scene은 writer에 도달하지 않습니다.
- **failure/edge branch:** CLI는 오류를 message/exit status로 변환하지만 당시 writer가 이미 연 대상의 transactional preservation까지 제공하지는 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 최초의 end-to-end command contract와 parse-before-output ordering을 제공합니다.
- **이 SHA가 보장하지 않는 것:** worker/mode/depth options는 후속 commit에서 추가됩니다.
- **직접 확인/후속 evidence:** `d05a6ab48bb1` smoke script가 같은 executable path를 통해 valid/invalid 동작을 검증하는 것을 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `1bc7cacd30aa`
- 다음 Thread commit: `d05a6ab48bb1`
- 이 commit이 다음 단계에 제공하는 것: `d05a6ab48bb1`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.12 `d05a6ab48bb1` — `test(render): 장면 렌더링 smoke 검사 추가`

- Importance: B
- Tags: TEST, INTEGRATION
- Thread order: 12/12

#### Source에서 확정된 역할

- Development Thread role: Verifies the first complete valid and invalid command paths.

#### B-level 구현 역할 복원

- **직전 관련 상태:** end-to-end CLI 경로가 만들어졌지만 반복 출력과 잘못된 scene의 no-output behavior를 자동으로 고정하는 증거가 없습니다.
- **핵심 구현 결정:** `tests/render_smoke.sh`가 임시 디렉터리와 cleanup trap을 사용합니다. unknown directive를 포함한 invalid input은 nonzero여야 하고 유효한 output을 만들면 실패합니다. valid 64×32 scene은 두 번 실행해 P3 header, 16진 checksum 형식·동일성, `cmp -s` byte equality를 확인합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/render_smoke.sh — CLI integration regression
 - Makefile — `test` target
- **caller → callee / data flow:** shell fixture 작성 → CLI invalid invocation/no-output assertion → valid invocation 두 번 → header/checksum/byte comparisons
- **ownership·state transition:** test artifact는 temp directory에 한정되고 trap이 제거합니다. production path는 `main`부터 parser, renderer, output까지 전체를 통과합니다.
- **failure/edge branch:** broad smoke이므로 개별 geometry 수식, image overflow, writer replacement failure를 격리해 증명하지 않습니다.

#### Test commit 분석 기준

- **대상 production invariant:** 유효 scene은 결정적인 P3/checksum을 만들고, 잘못된 scene은 성공 artifact를 만들지 않습니다.
- **test technique:** 임시 파일 기반 shell integration, exit-status 검사, header 검사, checksum equality, byte-for-byte comparison
- **통과하는 production path:** `main` → loader/parser → render → output
- **이 test가 증명하는 것:** 두 정상 실행의 외부 bytes/checksum 일치와 invalid command의 실패를 증명합니다.
- **이 test가 증명하지 않는 것:** transactional output failure, sanitizer clean, 모든 parser edge를 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 최초 완전한 valid command와 invalid command의 외부 observable contract를 고정합니다.
- **이 SHA가 보장하지 않는 것:** 테스트 자체의 소스 메커니즘은 검사했지만이 환경에서는 executable을 빌드·실행하지 않았습니다.
- **직접 확인/후속 evidence:** 테스트 성격: broad CLI integration + repeat-run deterministic regression. 실제 명령 결과는 기록하지 않습니다.

#### Thread 내 연결

- 이전 Thread commit: `b983f0ea2744`
- 다음 Thread commit: 이 Thread의 종료점
- 이 commit이 Thread 종료에 제공하는 것: Thread-level invariant ledger와 최종 실행 순서에서이 SHA의 결과를 최종 상태에 반영했습니다.

## 6. Invariant ledger

| Invariant | 최초 도입/기준 | 강화 또는 수정 | 부족함/위험 노출 | 고정한 test/evidence | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| 공통 hit record와 oriented normal | f3f1d04cc836 | f3f1d04cc836 | shape lifetime은 비소유 pointer에 의존 | 후속 scene/shading 사용 경로 | `Shape::intersect`, `HitRecord::setFaceNormal` |
| 선형 최근접 및 equal-`t` winner | 41a1d6bbe5ef | 9a7f29b5d78a에서 명시적 `(t,index)` rule | BVH traversal order가 winner를 바꿀 위험 | 41c9a59f27a6 | `Scene::intersect`의 inclusive upper bound와 candidate 교체 |
| 완성된 Scene만 반환 | 3545eb1e82df/6bff18bf0bac | 1e1fda47d913 | line-local validity만으로 R/A/C 누락 가능 | d05a6ab48bb1 invalid command | EOF required-directive check와 ParseError |
| deterministic serial image bytes | c742b2401e52 | 1bc7cacd30aa에서 artifact/checksum 노출 | image arithmetic와 output failure safety는 미완성 | d05a6ab48bb1 | center sample, clamp, `lround`, row-major RGB |

### Ledger 보완 기록

- 각 invariant는 위 표의 SHA에서 observable behavior 또는 state로 처음 나타났습니다.
- 후속 commit이 같은 용어를 사용하더라도 그 보장을 과거 SHA에 소급하지 않았습니다.
- test/evidence 열은 production path와 assertion 또는 deterministic work gate를 함께 가리킵니다.
- 실행하지 않은 test는 source-level evidence로만 기록했습니다.

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Decision/Fix | Test 또는 evidence | 실제 실패 처리와 assertion |
| --- | --- | --- | --- |
| unknown/malformed/missing scene directive | parser dispatch + EOF required validation | d05a6ab48bb1 smoke invalid path | writer 전에 ParseError → nonzero → output 부재 |
| shadow self-hit 또는 light 뒤 occluder | normal offset + bounded shadow interval | 후속 rendering regressions/직접 코드 검사 | `point+n·epsilon`, upper bound `distance-epsilon` |
| future traversal이 exact tie winner 변경 | 선형 저장 순서 semantics를 기준선으로 유지 | 41c9a59f27a6 | record/shape identity와 full pixels 비교 |

### 연결 검토

- feature commit도 어떤 잘못된 state 또는 semantic drift를 막는지 production path에 연결했습니다.
- fix commit은 기존 가정 → 실제 위험 → root cause → corrected decision → regression 순서로 기록했습니다.
- test가 broad integration인지 deterministic boundary/differential/failure-injection regression인지 commit 기록에서 구분했습니다.
- assertion이 증명하지 않는 범위와 실행하지 못한 항목을 별도로 남겼습니다.

## 8. Ownership / state / responsibility 변화

`Scene`은 카메라·조명·도형 집합의 aggregate입니다. 초기 SHA에서 shapes는
`shared_ptr` storage이므로 외부와 소유를 공유할 수 있고, `HitRecord::shape`는 수명을
연장하지 않는 `const Shape*`입니다. parser는 성공 반환 전까지 Scene construction을
담당하고, renderer는 Scene을 읽어 새 `Image`를 소유해 반환합니다. output은 Image를
소비하지만이 Thread의 초기 path writer는 아직 final path publication을 transactional하게
관리하지 않습니다. 후속 Thread의 `unique_ptr`/private storage와 atomic output을이 시점에
소급하지 않았습니다.

### 학습자 최종 기록

- **source state와 derived state:** `Scene`은 카메라·조명·도형 집합의 aggregate입니다. 초기 SHA에서 shapes는 `shared_ptr` storage이므로 외부와 소유를 공유할 수 있고, `HitRecord::shape`는 수명을 연장하지 않는 `const Shape*`입니다. parser는 성공 반환 전까지 Scene construction을 담당하고, renderer는 Scene을 읽어 새 `Image`를 소유해 반환합니다. output은 Image를 소비하지만이 Thread의 초기 path writer는 아직 final path publication을 transactional하게 관리하지 않습니다. 후속 Thread의 `unique_ptr`/private storage와 atomic output을이 시점에 소급하지 않았습니다.
- **mutation/transition boundary:** commit별 `ownership·state transition`과 위 invariant ledger에 표시했습니다.
- **failure 시 복구 상태:** Failure → Fix → Test 표와 각 fix/test section에 정상·오류 상태를 구분했습니다.

## 9. Thread 최종 상태

Thread 종료 시 유효 `.rt` 파일은 source-located validation을 통과한 `Scene`으로 바뀌고,
각 pixel center는 camera ray가 되어 선형 closest-hit, direct lighting, bounded shadow visibility를
통과합니다. 결과는 고정 clamp/rounding으로 RGB bytes가 되고 P3 및 checksum으로 노출됩니다.
CLI는 parsing 전에 output을 시작하지 않으며 smoke regression은 invalid no-output과 두 정상
실행의 exact bytes/checksum을 고정합니다. 다만 큰 vector 안정성, acceleration, parallelism,
image mutation, transactional publication은 뒤 Thread의 책임입니다.

### 직접 작성한 결론

- **Thread 시작과 종료의 behavior 차이:** Thread 종료 시 유효 `.rt` 파일은 source-located validation을 통과한 `Scene`으로 바뀌고, 각 pixel center는 camera ray가 되어 선형 closest-hit, direct lighting, bounded shadow visibility를 통과합니다. 결과는 고정 clamp/rounding으로 RGB bytes가 되고 P3 및 checksum으로 노출됩니다. CLI는 parsing 전에 output을 시작하지 않으며 smoke regression은 invalid no-output과 두 정상 실행의 exact bytes/checksum을 고정합니다. 다만 큰 vector 안정성, acceleration, parallelism, image mutation, transactional publication은 뒤 Thread의 책임입니다.
- **아직 다른 Thread 또는 외부 검증이 보완해야 하는 항목:** normalization overflow, BVH correctness, reflection depth, thread failure, representation validation, atomic output publication은 각각 Thread 2~6에서 보완됩니다.

## 10. 최종 architecture 또는 execution flow 정리

### Source가 확정한 흐름 anchor

```text
`main` → `loadScene`/parser → `Scene` → `renderScene` → `makeCameraRay` → `traceRay` → `Scene::intersect`/shadow query → RGB quantization → `Image` → `writePpm`/`imageChecksum`
```

### 실제 코드로 완성한 흐름

1. CLI가 입력 경로와 optional checksum 요청을 검증합니다.
2. loader가 파일을 열고 line tokenizer/dispatcher가 검증된 directive만 Scene에 반영합니다.
3. EOF에서 required singleton R/A/C를 검사한 뒤에만 Scene을 반환합니다.
4. renderer가 camera frame을 만들고 row-major pixel center마다 camera ray를 생성합니다.
5. `traceRay`가 선형 `Scene::intersect`로 authoritative hit를 고르고 miss/background 또는 ambient/direct/shadow color를 계산합니다.
6. color를 `[0,1]`로 clamp하고 `lround(component*255)`로 quantize해 RGB storage에 씁니다.
7. writer/checksum이 같은 Image bytes를 외부 P3와 비교값으로 노출합니다.
8. smoke test는 전체 production path의 valid/invalid 외부 결과를 검사합니다.

### 학습자의 최종 설명

Thread 종료 시 유효 `.rt` 파일은 source-located validation을 통과한 `Scene`으로 바뀌고,
각 pixel center는 camera ray가 되어 선형 closest-hit, direct lighting, bounded shadow visibility를
통과합니다. 결과는 고정 clamp/rounding으로 RGB bytes가 되고 P3 및 checksum으로 노출됩니다.
CLI는 parsing 전에 output을 시작하지 않으며 smoke regression은 invalid no-output과 두 정상
실행의 exact bytes/checksum을 고정합니다. 다만 큰 vector 안정성, acceleration, parallelism,
image mutation, transactional publication은 뒤 Thread의 책임입니다.

남은 경계는 다음과 같습니다. normalization overflow, BVH correctness, reflection depth, thread failure, representation validation, atomic output publication은 각각 Thread 2~6에서 보완됩니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 source 순서대로 확인했습니다.
- [x] 각 commit의 SHA, subject, importance, tags를 그대로 유지했습니다.
- [x] 모든 핵심 설명에 해당 SHA의 file path와 symbol 근거를 기록했습니다.
- [x] final HEAD의 구조를 과거 SHA에 소급하지 않았습니다.
- [x] S/A/B importance에 맞춰 architecture, subsystem, localized role의 깊이를 구분했습니다.
- [x] source에서 확정하지 않은 실행 결과나 runtime 수치를 사실로 채우지 않았습니다.
- [x] failure와 fix/test를 실제 production path로 연결했습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] invariant ledger의 각 변화를 commit evidence와 연결했습니다.
- [ ] 해당 SHA checkout에서 테스트·benchmark·sanitizer를 직접 실행했습니다. 환경 제한 때문에 미실행 상태입니다.
- [x] 별도의 프로젝트 재학습 없이이 Thread의 설계 → 구현 → 위험 → 수정 → 검증 발전을 설명할 수 있는 기록을 남겼습니다.
===== END FILE: 01-geometric-contracts-to-first-image.md =====

===== BEGIN FILE: 02-large-vector-normalization.md =====
# Thread 2. Large finite vectors and normalization stability

## 1. Thread 목표

일반적인 단위 벡터 테스트로는 드러나지 않는 큰 유한 벡터의 중간 overflow를 추적하고, 공유 수학 계층의 작은 구현 변경이 카메라·normal·cylinder axis·ray direction 전반의 유효성을 어떻게 복구하는지 확인합니다.

### Source significance

> This is a compact root-cause correction thread. The original mathematical interface was already
> shared by camera directions, normals, axes, and rays; the later test demonstrates that ordinary
> unit-vector cases were insufficient to protect it. The thread matters because a small implementation
> detail in a foundational value type could silently turn a meaningful finite direction into a
> zero-like result throughout the renderer.

### 이 Thread에 연결된 source invariant

- A non-negligible finite vector can be converted to its unit direction.

### 이 Thread에 연결된 engineering difficulty

- Maintaining numerical validity across normalization, quadratic intersection, near-zero directions, large finite values, and conservative cylinder bounds.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 기존 magnitude 계산에서 입력은 finite인데 왜 제곱 합이 infinity가 되는가?
- infinity로 계산된 길이가 normalization 결과를 zero-like 값으로 붕괴시키는 실제 연산 순서는 무엇인가?
- `std::hypot` 기반 계산이 public interface를 바꾸지 않고 어떤 numerical invariant를 복구하는가?
- 회귀 테스트가 일반 정규화 정확도가 아니라 정확히 어떤 overflow mechanism을 고정하는가?

## 3. 완료 기준

- [x] fix 직전과 fix SHA의 magnitude/normalization 코드를 직접 비교했습니다.
- [x] `(1e308, 0, 0)`이 기존 구현에서 실패하는 연산 과정을 수치적으로 설명할 수 있습니다.
- [x] 변경된 production path와 regression test의 호출 경로를 연결했습니다.
- [x] 이 테스트가 보장하는 범위와 보장하지 않는 다른 numerical edge를 구분했습니다.
- [x] 모든 참조 SHA가 `cpp/miniRT` branch HEAD의 ancestry에 속하는지 확인했습니다.
- [ ] 해당 SHA checkout에서 build/test/benchmark 명령을 직접 실행했습니다. 로컬 외부 네트워크와 checkout이 제공되지 않아 실행 evidence는 만들지 않았습니다.

### 검증 범위

- 지정 branch HEAD: `7d08c7c13fa68c3e60eea3c7014658b0a133e6f0`
- 각 참조 SHA는 Thread 내부의 연속 compare chain에서 `behind_by = 0`, merge base가 선행 SHA였고, Thread 종료 SHA도 branch HEAD의 조상으로 확인했습니다.
- 구현 설명은 해당 commit의 diff/file content를 기준으로 작성했으며, final HEAD의 후속 API를 과거 SHA에 소급하지 않았습니다.
- 테스트와 benchmark는 source mechanism과 production path만 검사했습니다. 실행 결과, sanitizer 결과, wall-clock 수치는 기록하지 않았습니다.

## 4. Commit map

1. `aa92a87c98a3` — `fix(math): 큰 유한 벡터를 안정적으로 정규화`
 - Importance: A
 - Tags: DEBUG, EDGE, RAY_PIPELINE
 - Source-defined role: Replaces overflow-prone sum-of-squares magnitude with `std::hypot`.

2. `ff18d1cc3afc` — `test(math): 큰 유한 벡터 정규화 검증`
 - Importance: B
 - Tags: TEST, EDGE
 - Source-defined role: Fixes the exact large-finite-vector regression as a permanent test case.

## 5. Commit별 학습 기록

### 5.1 `aa92a87c98a3` — `fix(math): 큰 유한 벡터를 안정적으로 정규화`

- Importance: A
- Tags: DEBUG, EDGE, RAY_PIPELINE
- Thread order: 1/2

#### Source에서 확정된 역할

- Development Thread role: Replaces overflow-prone sum-of-squares magnitude with `std::hypot`.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** `Vec3::length`가 `sqrt(x*x + y*y + z*z)` 계열의 `lengthSquared()` 결과에 의존했습니다. 입력 component가 finite여도 `1e308 * 1e308`은 `double` 범위를 넘어 infinity가 되므로 실제 vector는 의미 있는 방향을 갖는데도 중간 계산만 overflow합니다.
- **핵심 구현 결정:** `src/math.cpp`에서 public API와 normalization branch는 유지하고 magnitude 계산만 3-argument `std::hypot(x, y, z)`로 교체합니다. `hypot`은 component scale을 조정해 제곱합의 불필요한 중간 overflow/underflow를 줄입니다.

#### Failure → Fix 연결

- **기존 가정:** finite components의 제곱합도 meaningful finite magnitude를 만든다.
- **실제 failure 또는 위험:** 큰 component를 제곱하는 중간 연산이 infinity가 되어 normalize 결과를 0 방향처럼 만듭니다.
- **root cause:** 결과 범위가 아니라 naive sum-of-squares evaluation의 중간 overflow입니다.
- **수정된 decision/invariant:** scaled algorithm을 사용하는 `std::hypot`으로 magnitude를 계산합니다.
- **regression 연결:** `ff18d1cc3afc`의 `(1e308,0,0) == (1,0,0)` assertion입니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/math.cpp — `Vec3::length`
 - include/ray/math.hpp — `Vec3`, normalization API
- **caller → callee / data flow:** `Vec3(1e308,0,0)` → old: square `inf` → length `inf` → component/`inf` = zero-like; fixed: `hypot` = `1e308` → division = `(1,0,0)`
- **ownership·state transition:** ownership 변화는 없습니다. 동일한 immutable component 입력에서 derived magnitude와 normalized output만 바뀝니다. near-zero epsilon 정책과 normalization interface는 그대로입니다.
- **failure/edge branch:** 수정 전 위험은 input non-finiteness가 아니라 finite input의 intermediate overflow입니다. 이 fix는 parser range, NaN/Inf 수용 정책, 모든 subnormal 정확도를 새로 정의하지 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** non-negligible large finite vector가 avoidable overflow 때문에 zero-like direction으로 붕괴하지 않습니다.
- **이 SHA가 보장하지 않는 것:** 모든 가능한 방향과 비정상 부동소수 값에 대한 완전한 정책은 보장하지 않습니다.
- **직접 확인/후속 evidence:** immediate parent의 sum-of-squares와이 SHA의 `std::hypot` 한 줄 차이를 확인하고, 후속 exact regression과 연결했습니다. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: 이 Thread의 시작점
- 다음 Thread commit: `ff18d1cc3afc`
- 이 commit이 다음 단계에 제공하는 것: `ff18d1cc3afc`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.2 `ff18d1cc3afc` — `test(math): 큰 유한 벡터 정규화 검증`

- Importance: B
- Tags: TEST, EDGE
- Thread order: 2/2

#### Source에서 확정된 역할

- Development Thread role: Fixes the exact large-finite-vector regression as a permanent test case.

#### B-level 구현 역할 복원

- **직전 관련 상태:** production fix만 존재하면 향후 `lengthSquared` 기반 구현으로 되돌아가도 일반적인 작은 unit-vector test는 회귀를 잡지 못할 수 있습니다.
- **핵심 구현 결정:** `tests/core_tests.cpp`에 정확히 `(1e308, 0, 0)`을 normalize하고 `Vec3(1, 0, 0)`과 exact equality로 비교하는 case를 추가합니다. 다른 component가 0이므로 fixed path의 기대값은 표현 가능한 정확한 1과 0입니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/core_tests.cpp — large finite vector normalization assertion
 - src/math.cpp — `Vec3::length`/normalization production path
- **caller → callee / data flow:** test vector construction → public normalization → `Vec3::length` → `std::hypot` → component division → exact vector comparison
- **ownership·state transition:** fixture나 mutable 외부 상태 없이 하나의 deterministic value regression입니다.
- **failure/edge branch:** fix 이전에는 magnitude가 infinity가 되고 x component가 0처럼 되어 assertion이 실패합니다. 이 테스트는 NaN, infinity, subnormal, arbitrary 3-axis approximate accuracy를 다루지 않습니다.

#### Test commit 분석 기준

- **대상 production invariant:** non-negligible large finite vector는 unit direction으로 변환됩니다.
- **test technique:** 단일 deterministic boundary input과 exact expected vector 비교
- **통과하는 production path:** `Vec3` construction → normalization → `Vec3::length` → division
- **이 test가 증명하는 것:** `(1e308,0,0)`의 과거 intermediate-overflow 회귀가 차단됩니다.
- **이 test가 증명하지 않는 것:** 모든 vector direction, NaN/Inf 정책, subnormal precision을 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 과거에 실패한 large-finite overflow mechanism을 정확한 입력으로 고정합니다.
- **이 SHA가 보장하지 않는 것:** 일반 numerical conformance suite를 대신하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 소스와 호출 경로를 검사했습니다. 이 환경에서는 해당 test executable을 빌드하거나 fix 전/후로 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `aa92a87c98a3`
- 다음 Thread commit: 이 Thread의 종료점
- 이 commit이 Thread 종료에 제공하는 것: Thread-level invariant ledger와 최종 실행 순서에서이 SHA의 결과를 최종 상태에 반영했습니다.

## 6. Invariant ledger

| Invariant | 최초 도입/기준 | 강화 또는 수정 | 부족함/위험 노출 | 고정한 test/evidence | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| 큰 finite vector의 unit-direction 변환 | 기존 normalization API | aa92a87c98a3 | naive component square가 infinity가 됨 | ff18d1cc3afc | `Vec3::length`의 `std::hypot`과 exact regression |

### Ledger 보완 기록

- 각 invariant는 위 표의 SHA에서 observable behavior 또는 state로 처음 나타났습니다.
- 후속 commit이 같은 용어를 사용하더라도 그 보장을 과거 SHA에 소급하지 않았습니다.
- test/evidence 열은 production path와 assertion 또는 deterministic work gate를 함께 가리킵니다.
- 실행하지 않은 test는 source-level evidence로만 기록했습니다.

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Decision/Fix | Test 또는 evidence | 실제 실패 처리와 assertion |
| --- | --- | --- | --- |
| finite component 제곱합의 intermediate infinity | `std::hypot`의 scaled magnitude | ff18d1cc3afc | `normalize(Vec3(1e308,0,0)) == Vec3(1,0,0)` |

### 연결 검토

- feature commit도 어떤 잘못된 state 또는 semantic drift를 막는지 production path에 연결했습니다.
- fix commit은 기존 가정 → 실제 위험 → root cause → corrected decision → regression 순서로 기록했습니다.
- test가 broad integration인지 deterministic boundary/differential/failure-injection regression인지 commit 기록에서 구분했습니다.
- assertion이 증명하지 않는 범위와 실행하지 못한 항목을 별도로 남겼습니다.

## 8. Ownership / state / responsibility 변화

이 Thread는 object ownership을 바꾸지 않습니다. source state는 세 `double` component이고, derived state는 magnitude와 normalized components입니다. fix 전에는 finite source가 `inf` intermediate와 zero-like output으로 전이됐고, fix 후에는 finite magnitude와 unit direction으로 전이됩니다. public interface, caller ownership, epsilon branch는 유지됩니다.

### 학습자 최종 기록

- **source state와 derived state:** 이 Thread는 object ownership을 바꾸지 않습니다. source state는 세 `double` component이고, derived state는 magnitude와 normalized components입니다. fix 전에는 finite source가 `inf` intermediate와 zero-like output으로 전이됐고, fix 후에는 finite magnitude와 unit direction으로 전이됩니다. public interface, caller ownership, epsilon branch는 유지됩니다.
- **mutation/transition boundary:** commit별 `ownership·state transition`과 위 invariant ledger에 표시했습니다.
- **failure 시 복구 상태:** Failure → Fix → Test 표와 각 fix/test section에 정상·오류 상태를 구분했습니다.

## 9. Thread 최종 상태

large finite component를 가진 non-negligible vector도 avoidable intermediate overflow 없이 길이와 unit direction을 계산합니다. 정확한 과거 실패 입력이 regression으로 남아 naive sum-of-squares 회귀를 막습니다. NaN/Inf/subnormal 전체 정책은이 Thread가 정의하지 않습니다.

### 직접 작성한 결론

- **Thread 시작과 종료의 behavior 차이:** large finite component를 가진 non-negligible vector도 avoidable intermediate overflow 없이 길이와 unit direction을 계산합니다. 정확한 과거 실패 입력이 regression으로 남아 naive sum-of-squares 회귀를 막습니다. NaN/Inf/subnormal 전체 정책은이 Thread가 정의하지 않습니다.
- **아직 다른 Thread 또는 외부 검증이 보완해야 하는 항목:** non-finite 입력 정책과 geometry-specific numerical edges는 별도 parser/geometry contracts에 남습니다.

## 10. 최종 architecture 또는 execution flow 정리

### Source가 확정한 흐름 anchor

```text
`Vec3` components → `Vec3::length` → normalization divisor → component division → camera/normal/axis/ray consumers
```

### 실제 코드로 완성한 흐름

1. caller가 finite `Vec3`를 normalization API에 전달합니다.
2. `Vec3::length`가 세 component를 `std::hypot`으로 결합합니다.
3. 기존 near-zero branch가 magnitude를 검사합니다.
4. 유효 magnitude이면 각 component를 그 값으로 나눕니다.
5. camera direction, normal, cylinder axis, ray direction consumer가 같은 unit vector를 읽습니다.
6. core regression은 `(1e308,0,0)`이 정확한 +X unit vector인지 검사합니다.

### 학습자의 최종 설명

large finite component를 가진 non-negligible vector도 avoidable intermediate overflow 없이 길이와 unit direction을 계산합니다. 정확한 과거 실패 입력이 regression으로 남아 naive sum-of-squares 회귀를 막습니다. NaN/Inf/subnormal 전체 정책은이 Thread가 정의하지 않습니다.

남은 경계는 다음과 같습니다. non-finite 입력 정책과 geometry-specific numerical edges는 별도 parser/geometry contracts에 남습니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 source 순서대로 확인했습니다.
- [x] 각 commit의 SHA, subject, importance, tags를 그대로 유지했습니다.
- [x] 모든 핵심 설명에 해당 SHA의 file path와 symbol 근거를 기록했습니다.
- [x] final HEAD의 구조를 과거 SHA에 소급하지 않았습니다.
- [x] S/A/B importance에 맞춰 architecture, subsystem, localized role의 깊이를 구분했습니다.
- [x] source에서 확정하지 않은 실행 결과나 runtime 수치를 사실로 채우지 않았습니다.
- [x] failure와 fix/test를 실제 production path로 연결했습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] invariant ledger의 각 변화를 commit evidence와 연결했습니다.
- [ ] 해당 SHA checkout에서 테스트·benchmark·sanitizer를 직접 실행했습니다. 환경 제한 때문에 미실행 상태입니다.
- [x] 별도의 프로젝트 재학습 없이이 Thread의 설계 → 구현 → 위험 → 수정 → 검증 발전을 설명할 수 있는 기록을 남겼습니다.
===== END FILE: 02-large-vector-normalization.md =====

===== BEGIN FILE: 03-correctness-preserving-bvh.md =====
# Thread 3. Correctness-preserving BVH acceleration

## 1. Thread 목표

선형 탐색을 의미적 기준으로 유지한 채, 측정 가능한 work baseline에서 출발해 conservative bounds, deterministic build, owned derived state, explicit-stack traversal, equivalence tests, benchmark gate, stale-cache 방지까지 BVH의 전체 correctness story를 복원합니다.

### Source significance

> The thread shows that acceleration was treated as a semantic transformation, not merely a faster
> container. It first establishes comparable work metrics, then introduces conservative bounds,
> deterministic construction, and a traversal that reuses the linear candidate rule. The later
> ownership correction is crucial historical evidence: a BVH can be algorithmically correct yet still
> become wrong if its source geometry remains externally mutable. The final design therefore combines
> algorithm, result-ordering, bounded/unbounded partitioning, ownership, invalidation, fallback,
> rebuild, equivalence testing, and measured work reduction.

### 이 Thread에 연결된 source invariant

- A hit result must have the same semantic winner in linear and BVH modes, including exact equal-distance cases, regardless of BVH build or traversal order.
- Every acceleration bound must conservatively contain its shape. Unbounded geometry must never be forced into an arbitrary finite box.
- A ready BVH must describe the current shape set and current built-in geometry. Shape mutation must invalidate derived acceleration, and BVH requests against invalidated state must fall back to current linear geometry.
- Scene owns each shape exactly once, and callers cannot mutate built-in geometry or reorder shape storage behind the acceleration lifecycle.

### 이 Thread에 연결된 engineering difficulty

- Reordering primitive tests through a BVH without changing equal-distance selection, materials, normals, hit pointers, or final image bytes.
- Managing the BVH as derived state whose correctness depends on shape lifetime, geometry immutability, invalidation, rebuilding, and 대체 동작.
- Designing performance evidence that rejects behaviorally different results, separates primitive from AABB work, fixes workload configuration, and reports repeatable median measurements.
- Maintaining numerical validity across conservative cylinder bounds.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 가속 전 work counter는 어떤 경계에서 실제 primitive dispatch와 shadow query를 세는가?
- AABB slab test와 각 shape bounds가 false negative를 만들지 않기 위해 어떤 interval·padding 규칙을 사용하는가?
- deterministic median split은 centroid tie를 어떻게 원래 shape index로 해소하는가?
- BVH가 scene의 독립 상태가 아니라 derived cache인 이유와 build/invalidate/fallback/rebuild 상태 전이는 무엇인가?
- traversal 순서가 바뀌어도 exact equal-`t` winner, material, normal, shape identity가 유지되는 이유는 무엇인가?
- benchmark가 빠른데 다른 이미지를 만드는 구현이나 culling이 부족한 구현을 어떻게 거부하는가?
- 초기 lifecycle 설계가 public mutable geometry 때문에 불충분했던 root cause와 최종 API 차단은 무엇인가?

## 3. 완료 기준

- [x] 선형과 BVH가 공유하는 candidate-update 규칙을 실제 함수 단위로 추적했습니다.
- [x] sphere, plane, arbitrary-axis cylinder가 bounded/unbounded로 분류되는 코드와 bounds 계산을 확인했습니다.
- [x] BVH build 결과의 node/primitive 저장 구조와 stable ordering 근거를 기록했습니다.
- [x] Scene acceleration state의 build → ready → mutation invalidation → linear fallback → rebuild 전이를 코드와 테스트로 재현했습니다.
- [x] hit record 전체, pixel buffer, checksum, primitive/AABB counts를 비교하는 검증 계층을 구분했습니다.
- [x] schemaVersion 1 benchmark와 primitive-test ratio gate가 무엇을 증명하고 elapsed speedup이 왜 환경 의존적인지 설명할 수 있습니다.
- [x] 모든 참조 SHA가 `cpp/miniRT` branch HEAD의 ancestry에 속하는지 확인했습니다.
- [ ] 해당 SHA checkout에서 build/test/benchmark 명령을 직접 실행했습니다. 로컬 외부 네트워크와 checkout이 제공되지 않아 실행 evidence는 만들지 않았습니다.

### 검증 범위

- 지정 branch HEAD: `7d08c7c13fa68c3e60eea3c7014658b0a133e6f0`
- 각 참조 SHA는 Thread 내부의 연속 compare chain에서 `behind_by = 0`, merge base가 선행 SHA였고, Thread 종료 SHA도 branch HEAD의 조상으로 확인했습니다.
- 구현 설명은 해당 commit의 diff/file content를 기준으로 작성했으며, final HEAD의 후속 API를 과거 SHA에 소급하지 않았습니다.
- 테스트와 benchmark는 source mechanism과 production path만 검사했습니다. 실행 결과, sanitizer 결과, wall-clock 수치는 기록하지 않았습니다.

## 4. Commit map

1. `f4dcb50939e2` — `perf(render): 광선과 교차 작업량 계측 추가`
 - Importance: A
 - Tags: PERF, RAY_PIPELINE
 - Source-defined role: Adds semantic work counters and timing before acceleration changes behavior.

2. `4fb2345c7d35` — `perf(benchmark): 조밀 장면 기준 workload 추가`
 - Importance: B
 - Tags: PERF, ACCEL
 - Source-defined role: Establishes a fixed dense-scene workload.

3. `f5a2c4ade16d` — `perf(benchmark): 반복 측정과 결정성 보고 구성`
 - Importance: A
 - Tags: PERF, DETERMINISM
 - Source-defined role: Defines repeated median measurement and rejects inconsistent results.

4. `7b19f2ad78e3` — `feat(accel): ray-box slab 교차 구현`
 - Importance: A
 - Tags: ACCEL, HARD, EDGE
 - Source-defined role: Implements the ray/AABB interval test.

5. `a40452885176` — `feat(accel): 도형 경계 계약과 구·평면 bounds 추가`
 - Importance: A
 - Tags: ARCH, ACCEL, GEOMETRY
 - Source-defined role: Defines which shapes provide finite bounds and which remain unbounded.

6. `b782e22450d8` — `feat(accel): 원기둥의 보수적 bounds 계산 추가`
 - Importance: A
 - Tags: ACCEL, GEOMETRY, HARD
 - Source-defined role: Adds conservative arbitrary-axis cylinder bounds.

7. `419d52d687fc` — `test(accel): AABB와 도형 경계 계산 검증`
 - Importance: A
 - Tags: TEST, ACCEL, RISK
 - Source-defined role: Verifies AABB edge behavior and the no-false-negative bounds contract.

8. `bb65e8092632` — `feat(accel): 결정적 중앙 분할 BVH 구축 구현`
 - Importance: A
 - Tags: ACCEL, HARD, DETERMINISM
 - Source-defined role: Builds a deterministic median-split BVH.

9. `9a7f29b5d78a` — `feat(accel): 선형·BVH 탐색 모드 계약 연결`
 - Importance: A
 - Tags: ARCH, ACCEL, DETERMINISM
 - Source-defined role: Preserves the linear equal-`t` winner through original shape indices and exposes both modes.

10. `f7e969537c10` — `feat(scene): 가속 구조 소유권과 rebuild 경계 구성`
 - Importance: S
 - Tags: ARCH, ACCEL, SCENE
 - Source-defined role: Makes acceleration owned, rebuildable derived scene state with an unbounded-shape path.

11. `d4f6ee5b6042` — `feat(accel): 결정적 BVH 최근접 순회 구현`
 - Importance: S
 - Tags: CORE, ACCEL, HARD
 - Source-defined role: Implements near-first explicit-stack traversal and pruning.

12. `41c9a59f27a6` — `test(accel): 선형 탐색과 BVH 결과 동치 검증`
 - Importance: A
 - Tags: TEST, ACCEL, DETERMINISM
 - Source-defined role: Proves linear/BVH hit and pixel equivalence while checking work reduction.

13. `da3e8b43d09e` — `perf(benchmark): 선형 탐색과 BVH 작업량 비교`
 - Importance: A
 - Tags: PERF, ACCEL, DETERMINISM
 - Source-defined role: Measures both modes under the same correctness constraints.

14. `9b77225cf6b7` — `perf(benchmark): 측정 schema와 가속 기준 검증 고정`
 - Importance: A
 - Tags: PERF, ACCEL, DETERMINISM
 - Source-defined role: Fixes a versioned benchmark schema and enforces a primitive-test reduction threshold.

15. `ef5320a83c27` — `fix(accel): 가속 구조의 도형 불변식 보호`
 - Importance: S
 - Tags: ARCH, ACCEL, SCENE
 - Source-defined role: Prevents callers from mutating geometry behind a ready BVH.

16. `13f153e23920` — `test(accel): 장면 변경과 가속 상태 불변식 검증`
 - Importance: A
 - Tags: TEST, ACCEL, SCENE
 - Source-defined role: Verifies immutability, invalidation, linear fallback, and rebuild as one state transition.

## 5. Commit별 학습 기록

### 5.1 `f4dcb50939e2` — `perf(render): 광선과 교차 작업량 계측 추가`

- Importance: A
- Tags: PERF, RAY_PIPELINE
- Thread order: 1/16

#### Source에서 확정된 역할

- Development Thread role: Adds semantic work counters and timing before acceleration changes behavior.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** BVH를 추가하기 전에 “빠르다”를 말하려면 동일한 renderer가 실제로 수행한 semantic work를 세는 기준이 필요합니다. scene size를 근사하거나 elapsed time만 재면 culling 정도와 환경 noise를 분리할 수 없습니다.
- **핵심 구현 결정:** `RenderStats`에 primary/secondary/shadow ray, primitive test, 향후 AABB test, elapsed milliseconds를 둡니다. optional stats sink를 render → trace/shade → scene intersection/occlusion 경로에 전달합니다. primitive counter는 각 `Shape::intersect` dispatch 직전에, shadow counter는 positive diffuse term 뒤 실제 occlusion query 직전에 증가합니다. elapsed interval은 complete image rendering을 steady clock으로 감쌉니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/renderer.hpp — `RenderStats`, optional stats parameters
 - src/renderer.cpp — primary rays와 elapsed interval
 - src/shading.cpp — secondary/shadow counters
 - src/scene.cpp — primitive dispatch counter
- **caller → callee / data flow:** render entry → optional sink 전달 → 실제 ray/query/primitive dispatch에서 integer counter 증가 → image 완료 후 elapsed 기록
- **ownership·state transition:** sink가 null이면 기존 호출과 결과를 유지합니다. counters는 관찰용 derived data이며 image selection에 참여하지 않습니다. `aabbTests` field는 준비되지만이 SHA에는 실제 box traversal이 없습니다.
- **failure/edge branch:** counter를 dispatch 뒤에 올리면 예외/early return work를 누락하고, scene size로 추정하면 실제 pruning을 측정하지 못합니다. 이 commit은 그런 위치 오류를 피하도록 semantic boundary에 계측을 둡니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 후속 acceleration이 같은 image를 만들면서 primitive work를 줄였는지 비교할 수 있는 기준을 제공합니다.
- **이 SHA가 보장하지 않는 것:** elapsed time은 환경 의존적이고, AABB work는 아직 0입니다.
- **직접 확인/후속 evidence:** 각 signature와 increment 위치를 해당 SHA diff에서 확인했습니다. runtime counter 값은 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: 이 Thread의 시작점
- 다음 Thread commit: `4fb2345c7d35`
- 이 commit이 다음 단계에 제공하는 것: `4fb2345c7d35`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.2 `4fb2345c7d35` — `perf(benchmark): 조밀 장면 기준 workload 추가`

- Importance: B
- Tags: PERF, ACCEL
- Thread order: 2/16

#### Source에서 확정된 역할

- Development Thread role: Establishes a fixed dense-scene workload.

#### B-level 구현 역할 복원

- **직전 관련 상태:** 계측만 있어도 장면이 작거나 매번 다르면 acceleration 전후를 반복 비교할 수 없습니다.
- **핵심 구현 결정:** `benchmarks/render_benchmark.cpp`에 640×360 camera, 두 light, plane, 20×20 sphere grid를 코드로 만드는 고정 dense scene을 추가하고 render stats와 checksum을 출력합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - benchmarks/render_benchmark.cpp — fixed dense scene construction and single measured render
 - CMakeLists.txt — benchmark target
- **caller → callee / data flow:** hard-coded configuration → dense Scene construction → render with stats → checksum/work/time report
- **ownership·state transition:** workload geometry와 resolution이 source에 고정되어 비교 입력이 됩니다. 이 시점에는 반복 median이나 linear/BVH 두 모드 비교가 아직 없습니다.
- **failure/edge branch:** 한 번의 elapsed measurement는 warm-up과 OS scheduling noise에 취약합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** acceleration 전 semantic-work baseline을 재사용 가능한 executable로 만듭니다.
- **이 SHA가 보장하지 않는 것:** 반복 결정성, median, cross-mode equality와 gate는 후속 commit입니다.
- **직접 확인/후속 evidence:** scene 구성의 plane + 400 spheres, 해상도와 light 수를 source에서 확인했습니다. benchmark는 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `f4dcb50939e2`
- 다음 Thread commit: `f5a2c4ade16d`
- 이 commit이 다음 단계에 제공하는 것: `f5a2c4ade16d`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.3 `f5a2c4ade16d` — `perf(benchmark): 반복 측정과 결정성 보고 구성`

- Importance: A
- Tags: PERF, DETERMINISM
- Thread order: 3/16

#### Source에서 확정된 역할

- Development Thread role: Defines repeated median measurement and rejects inconsistent results.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** 단일 benchmark run은 cache/warm-up과 scheduling 변동을 대표값으로 오인할 수 있고, 반복마다 다른 image/work를 만들어도 단순 시간 출력만으로는 알아차리지 못합니다.
- **핵심 구현 결정:** 한 번의 warm-up 뒤 5회 측정하고 elapsed 값을 정렬해 median을 보고합니다. 각 run의 checksum과 primitive count가 첫 측정과 같지 않으면 benchmark를 실패시켜 시간 비교 전에 결과와 deterministic work를 검증합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - benchmarks/render_benchmark.cpp — warm-up, five samples, consistency checks, median/report
- **caller → callee / data flow:** warm-up(discard) → measured render ×5 → per-run checksum/work equality → elapsed sort → median output
- **ownership·state transition:** checksum과 integer work는 deterministic contract이고 elapsed는 관찰값입니다. benchmark가 mismatch를 발견하면 representative time을 성공값으로 보고하지 않습니다.
- **failure/edge branch:** 결과가 다른 빠른 implementation이나 nondeterministic work를 시간 개선으로 받아들이는 것을 거부합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 같은 모드·workload 안에서 반복 결과와 work 일치 후 median time을 제공합니다.
- **이 SHA가 보장하지 않는 것:** 아직 BVH mode가 없으므로 acceleration cross-mode equivalence는 증명하지 않습니다.
- **직접 확인/후속 evidence:** warm-up 횟수, 5회 sample, checksum/primitive consistency와 median 로직을 확인했습니다. 실제 median은 측정하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `4fb2345c7d35`
- 다음 Thread commit: `7b19f2ad78e3`
- 이 commit이 다음 단계에 제공하는 것: `7b19f2ad78e3`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.4 `7b19f2ad78e3` — `feat(accel): ray-box slab 교차 구현`

- Importance: A
- Tags: ACCEL, HARD, EDGE
- Thread order: 4/16

#### Source에서 확정된 역할

- Development Thread role: Implements the ray/AABB interval test.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** BVH node를 pruning하려면 ray와 axis-aligned box의 겹치는 parameter interval을 false negative 없이 계산해야 합니다.
- **핵심 구현 결정:** `Aabb`와 slab intersection을 구현합니다. caller의 `[tMin,tMax]`에서 시작해 x/y/z 각 축의 near/far를 교집합합니다. direction component가 0이면 origin이 slab 안인지 직접 검사하고, 음수 방향은 near/far를 swap합니다. `far < near`일 때만 miss이므로 `far == near`인 접촉도 hit로 보존합니다. 필요하면 entry distance를 반환합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/accel.hpp — `Aabb`와 intersection API
 - src/accel.cpp — slab test
- **caller → callee / data flow:** ray + valid box + caller interval → axis별 slab interval → running near/far 교집합 → optional entry + hit/miss
- **ownership·state transition:** running near/far가 축을 지날 때마다 좁아지는 local state입니다. box invalidity와 parallel-outside는 즉시 miss입니다.
- **failure/edge branch:** `far <= near`로 거부하면 tangent/contact traversal을 false negative로 만들 수 있습니다. 0으로 나누기 전에 parallel branch를 분리해야 합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 닫힌 interval 기준의 ray-box candidate test를 제공합니다.
- **이 SHA가 보장하지 않는 것:** shape bounds의 보수성과 BVH build/traversal correctness는 아직 별도입니다.
- **직접 확인/후속 evidence:** 후속 boundary tests가 parallel, negative direction, exact contact를 고정하는 것을 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `f5a2c4ade16d`
- 다음 Thread commit: `a40452885176`
- 이 commit이 다음 단계에 제공하는 것: `a40452885176`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.5 `a40452885176` — `feat(accel): 도형 경계 계약과 구·평면 bounds 추가`

- Importance: A
- Tags: ARCH, ACCEL, GEOMETRY
- Thread order: 5/16

#### Source에서 확정된 역할

- Development Thread role: Defines which shapes provide finite bounds and which remain unbounded.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** AABB test가 있어도 모든 shape가 유한 bounds를 갖는다는 잘못된 가정은 무한 평면을 임의 box로 잘라 hit를 누락시킵니다.
- **핵심 구현 결정:** `Shape::bounds` 계약을 추가해 finite bounds가 있으면 value를, 없으면 `std::nullopt`를 반환하도록 합니다. Sphere는 center ± radius의 정확한 box를 제공하고 Plane은 무한 geometry이므로 `nullopt`를 반환합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/geometry.hpp — virtual `bounds` contract
 - src/geometry.cpp — `Sphere::bounds`, `Plane::bounds`
- **caller → callee / data flow:** shape → optional bounds → bounded shape는 BVH 후보, unbounded shape는 별도 linear path
- **ownership·state transition:** bounds는 source geometry에서 계산되는 derived value입니다. `nullopt`는 오류가 아니라 acceleration partition 정보입니다.
- **failure/edge branch:** Plane에 arbitrary finite box를 부여하면 box 밖의 실제 hit가 pruning되어 false negative가 됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 가속 가능한 유한 geometry와 별도 검사해야 하는 무한 geometry를 type-polymorphic하게 구분합니다.
- **이 SHA가 보장하지 않는 것:** 원기둥 bounds와 Scene partition/lifecycle은 다음 commit들에 있습니다.
- **직접 확인/후속 evidence:** Sphere exact bounds와 Plane `nullopt`를 해당 SHA에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `7b19f2ad78e3`
- 다음 Thread commit: `b782e22450d8`
- 이 commit이 다음 단계에 제공하는 것: `b782e22450d8`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.6 `b782e22450d8` — `feat(accel): 원기둥의 보수적 bounds 계산 추가`

- Importance: A
- Tags: ACCEL, GEOMETRY, HARD
- Thread order: 6/16

#### Source에서 확정된 역할

- Development Thread role: Adds conservative arbitrary-axis cylinder bounds.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** arbitrary-axis finite cylinder는 center±radius 같은 단순 box로 감쌀 수 없습니다. side radius와 half-height가 각 world axis에 다르게 투영됩니다.
- **핵심 구현 결정:** 정규화된 cylinder axis의 각 component에 대해 cap 방향 extent와 축에 수직인 원의 최대 projection을 결합합니다. side extent는 radius와 `sqrt(max(0, 1-axis_i²))` 계열로, cap extent는 half-height·`abs(axis_i)`로 계산합니다. epsilon을 반영하고 `std::nextafter`로 min/max를 바깥쪽으로 한 단계 확장해 rounding으로 shape를 잘라내지 않게 합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/geometry.cpp — `Cylinder::bounds`
 - include/ray/geometry.hpp — cylinder bounds override
- **caller → callee / data flow:** center/normalized axis/radius/half-height → 축별 cap+side 최대 extent → conservative min/max → outward `nextafter`
- **ownership·state transition:** bounds는 cylinder geometry의 derived snapshot입니다. 이후 geometry mutation이 허용되면이 snapshot과 BVH가 stale해질 수 있다는 문제가 Thread 후반에 드러납니다.
- **failure/edge branch:** projection 일부를 빼거나 rounding inward가 되면 실제 surface point가 box 밖에 놓여 traversal false negative가 발생합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 지원하는 arbitrary-axis cylinder를 포함하는 보수적 finite box를 제공합니다.
- **이 SHA가 보장하지 않는 것:** tightest possible box나 모든 floating-point non-finite geometry 정책을 목표로 하지 않습니다.
- **직접 확인/후속 evidence:** 후속 test가 sampled/exact cylinder points가 bounds 안인지 검사하는 경로와 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `a40452885176`
- 다음 Thread commit: `419d52d687fc`
- 이 commit이 다음 단계에 제공하는 것: `419d52d687fc`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.7 `419d52d687fc` — `test(accel): AABB와 도형 경계 계산 검증`

- Importance: A
- Tags: TEST, ACCEL, RISK
- Thread order: 7/16

#### Source에서 확정된 역할

- Development Thread role: Verifies AABB edge behavior and the no-false-negative bounds contract.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** slab와 bounds 수식이 추가됐지만 접촉·parallel·negative direction과 cylinder containment를 자동으로 고정하지 않으면 작은 비교 연산 변경이 false negative를 만들 수 있습니다.
- **핵심 구현 결정:** `tests/core_tests.cpp`에 AABB forward entry 2, negative direction, boundary contact hit, parallel outside miss를 추가합니다. Sphere exact bounds, Plane no-bounds, arbitrary-axis Cylinder의 계산된 surface points가 box 안에 있고 허용 오차 내에 있는지도 검증합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/core_tests.cpp — AABB and shape-bounds regression cases
 - src/accel.cpp — slab production path
 - src/geometry.cpp — shape bounds
- **caller → callee / data flow:** deterministic ray/box cases → slab result/entry assertions; geometry construction → bounds → containment assertions
- **ownership·state transition:** fixture는 local value objects이며 external state가 없습니다.
- **failure/edge branch:** contact를 miss로 바꾸거나 cylinder extent를 작게 만들면 assertion이 실패합니다.

#### Test commit 분석 기준

- **대상 production invariant:** AABB interval과 shape bounds가 유효한 hit 후보를 거짓으로 제거하지 않습니다.
- **test technique:** deterministic boundary testing + geometry containment assertions
- **통과하는 production path:** `Aabb` slab test, `Sphere::bounds`, `Plane::bounds`, `Cylinder::bounds`
- **이 test가 증명하는 것:** parallel/negative/contact 사례와 대표 shape bounds behavior를 고정합니다.
- **이 test가 증명하지 않는 것:** BVH build/traversal 전체 또는 연속 공간의 모든 point를 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 선택된 edge와 shape samples에서 no-false-negative bounds 계약을 고정합니다.
- **이 SHA가 보장하지 않는 것:** 모든 방향·모든 surface point의 수학적 증명이나 실제 BVH traversal equivalence를 대신하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 코드와 production path를 검사했습니다. executable은 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `b782e22450d8`
- 다음 Thread commit: `bb65e8092632`
- 이 commit이 다음 단계에 제공하는 것: `bb65e8092632`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.8 `bb65e8092632` — `feat(accel): 결정적 중앙 분할 BVH 구축 구현`

- Importance: A
- Tags: ACCEL, HARD, DETERMINISM
- Thread order: 8/16

#### Source에서 확정된 역할

- Development Thread role: Builds a deterministic median-split BVH.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** bounded primitive 목록과 AABB가 있어도 build order가 비결정적이거나 leaf에 너무 많은 primitive를 두면 reproducibility와 work reduction을 확보할 수 없습니다.
- **핵심 구현 결정:** `Bvh::build`가 입력 primitive reference와 original shape index를 복사하고 기존 nodes/indices를 지운 뒤 recursive median split을 수행합니다. node bounds와 centroid bounds를 합치고, centroid extent가 가장 큰 축을 선택합니다. centroid가 같으면 original shape index로 tie-break하며 stable ordering을 유지합니다. count ≤ 4는 leaf로 만들고 그 외에는 median `count/2`에서 나눕니다. leaf는 flattened primitive index range를 가리킵니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/accel.hpp — BVH node/primitive storage
 - src/accel.cpp — `Bvh::build` and recursive builder
- **caller → callee / data flow:** bounded primitive refs → node bounds/centroid bounds → deterministic split axis → stable sort `(centroid, shapeIndex)` → median recursion → flat nodes + primitive indices
- **ownership·state transition:** BVH owns node/index arrays but shapes 자체는 소유하지 않고 scene-owned objects를 참조합니다. empty build는 empty state로 종료합니다.
- **failure/edge branch:** centroid-only unstable sort는 동률에서 build shape를 바꿀 수 있고, original index를 잃으면 exact equal-`t` semantics를 복원하기 어렵습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 같은 ordered shape input에서 결정적인 median-split tree와 leaf ordering을 만듭니다.
- **이 SHA가 보장하지 않는 것:** 이 commit은 아직 Scene mode/lifecycle이나 traversal winner를 완성하지 않습니다.
- **직접 확인/후속 evidence:** node/leaf limit, axis selection, index tie-break와 storage layout을 해당 SHA diff에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `419d52d687fc`
- 다음 Thread commit: `9a7f29b5d78a`
- 이 commit이 다음 단계에 제공하는 것: `9a7f29b5d78a`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.9 `9a7f29b5d78a` — `feat(accel): 선형·BVH 탐색 모드 계약 연결`

- Importance: A
- Tags: ARCH, ACCEL, DETERMINISM
- Thread order: 9/16

#### Source에서 확정된 역할

- Development Thread role: Preserves the linear equal-`t` winner through original shape indices and exposes both modes.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** BVH가 primitive 순서를 바꾸면 단순 “먼저 발견한 최소 t” 규칙은 선형 traversal의 exact equal-distance winner를 바꿀 수 있습니다.
- **핵심 구현 결정:** `AccelMode`를 호출 경로에 노출하고 candidate 비교를 `(smaller t) OR (equal t AND larger original shape index)`로 명시합니다. 이는 `41a1d6bbe5ef`에서 inclusive upper bound 때문에 생긴 “뒤 저장 index가 exact tie winner” 규칙을 traversal order와 분리합니다. 이 SHA에서는 mode parameter wiring이 생기지만 실제 BVH traversal은 아직 다음 commit들에 있습니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/renderer.hpp — `AccelMode`/settings
 - include/ray/accel.hpp — indexed primitive representation
 - src/scene.cpp — candidate update helper/linear path
 - src/renderer.cpp, src/shading.cpp — mode 전달
- **caller → callee / data flow:** candidate `(t,index)` → current winner와 비교 → smaller t 또는 exact tie/larger index만 authoritative record 교체
- **ownership·state transition:** winner state가 거리뿐 아니라 original index를 포함합니다. traversal order는 더 이상 semantic tie-break source가 아닙니다.
- **failure/edge branch:** `<=` 또는 discovery order만 쓰면 tree shape/near-first order에 따라 material·normal·shape identity가 바뀔 수 있습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 선형 semantics를 가속 traversal에서도 재사용할 수 있는 explicit candidate rule을 정의합니다.
- **이 SHA가 보장하지 않는 것:** BVH ready state와 traversal은 아직 완성되지 않았으며 mode가 실제 다른 path를 선택하지 않는 단계입니다.
- **직접 확인/후속 evidence:** 선형 baseline과 candidate helper의 tie 방향을 대조했습니다.

#### Thread 내 연결

- 이전 Thread commit: `bb65e8092632`
- 다음 Thread commit: `f7e969537c10`
- 이 commit이 다음 단계에 제공하는 것: `f7e969537c10`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.10 `f7e969537c10` — `feat(scene): 가속 구조 소유권과 rebuild 경계 구성`

- Importance: S
- Tags: ARCH, ACCEL, SCENE
- Thread order: 10/16

#### Source에서 확정된 역할

- Development Thread role: Makes acceleration owned, rebuildable derived scene state with an unbounded-shape path.

#### S-level architecture와 invariant 복원

- **직전 관련 상태:** BVH object만 존재하면 누가 build하고 shape mutation 뒤 언제 invalidated되는지, Plane 같은 unbounded geometry를 어디서 검사하는지 불명확합니다.
- **핵심 구현 결정:** `Scene`이 BVH와 unbounded shape index 목록, ready flag를 소유합니다. `addShape`가 shape를 Scene에 넣고 acceleration을 invalidated state로 바꿉니다. `buildAcceleration`은 현재 shapes에서 finite bounds와 unbounded를 partition하고 bounded refs로 BVH를 rebuild한 뒤 ready를 true로 만듭니다. parser는 scene 구성이 끝난 뒤 build를 호출합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/scene.hpp — BVH/unbounded indices/ready state
 - src/scene.cpp — `addShape`, invalidation, `buildAcceleration`
 - src/parser.cpp — complete scene 뒤 acceleration build
- **caller → callee / data flow:** Scene shape mutation → ready=false → build: bounded/unbounded partition + BVH rebuild → ready=true → query가 derived state 사용 가능
- **ownership·state transition:** shape objects는 Scene이 소유하고 BVH는 그 수명 안에서 non-owning references/indices를 가진 derived cache입니다. unbounded list도 같은 source set에서 파생됩니다.
- **failure/edge branch:** ready cache가 source geometry와 다르면 false negative가 가능합니다. 이 초기 설계는 public mutable built-in geometry를 완전히 차단하지 못해 `ef5320a83c27`에서 수정됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** shape set 변경을 통한 invalidation, explicit rebuild, bounded/unbounded 두 query path의 lifecycle을 정의합니다.
- **이 SHA가 보장하지 않는 것:** 외부가 이미 보유한 shape 또는 public geometry field를 변경하는 stale-cache 경로는 남아 있습니다. 실제 BVH traversal은 다음 SHA입니다.
- **직접 확인/후속 evidence:** build/invalidate/partition 순서를 확인하고 후속 immutability fix의 root cause와 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `9a7f29b5d78a`
- 다음 Thread commit: `d4f6ee5b6042`
- 이 commit이 다음 단계에 제공하는 것: `d4f6ee5b6042`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.11 `d4f6ee5b6042` — `feat(accel): 결정적 BVH 최근접 순회 구현`

- Importance: S
- Tags: CORE, ACCEL, HARD
- Thread order: 11/16

#### Source에서 확정된 역할

- Development Thread role: Implements near-first explicit-stack traversal and pruning.

#### S-level architecture와 invariant 복원

- **직전 관련 상태:** Scene-owned ready BVH가 있어도 node traversal, pruning, leaf dispatch, unbounded merge가 없으면 mode가 의미 있는 acceleration을 수행하지 않습니다.
- **핵심 구현 결정:** `Scene::intersect`가 Linear mode이거나 ready가 아니면 기존 전체 선형 path로 fallback합니다. ready BVH에서는 explicit stack으로 root부터 순회하고 node AABB entry가 현재 closest보다 크면 prune합니다. leaf는 indexed primitive를 실제 intersect하고 공통 `(t,index)` rule로 winner를 갱신합니다. 두 child가 hit하면 entry가 먼 child를 먼저 push하고 가까운 child를 나중에 push해 stack에서 near-first로 pop되게 하며, entry tie는 node index로 결정합니다. 마지막에 unbounded indices도 같은 candidate rule로 검사합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/scene.cpp — mode-aware `Scene::intersect`, explicit-stack BVH traversal, unbounded pass
 - include/ray/accel.hpp — nodes/primitive indices
- **caller → callee / data flow:** query → linear/not-ready fallback OR root box → explicit stack → node box/prune → leaves primitive dispatch → bounded winner → unbounded dispatch → shared candidate result
- **ownership·state transition:** stack은 query-local이며 Scene/BVH를 변경하지 않습니다. `closest`와 original winner index가 pruning과 semantic selection의 authoritative state입니다. stats sink가 있으면 AABB/primitive work를 실제 dispatch 시점에 셉니다.
- **failure/edge branch:** far child를 잘못된 순서로 push하거나 equal-entry tie를 비결정적으로 두어도 image winner는 candidate rule로 보호되지만 work sequence가 흔들릴 수 있습니다. unbounded pass를 생략하면 Plane hit가 사라집니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** ready BVH에서 near-first pruning을 수행하면서 선형 equal-`t` winner와 unbounded geometry를 보존합니다. stale/invalid state에서는 current geometry의 선형 결과를 반환합니다.
- **이 SHA가 보장하지 않는 것:** algorithm이 source geometry와 동기화돼 있다는 가정은 public mutation 때문에 아직 완전하지 않으며 `ef5320a83c27`에서 닫힙니다.
- **직접 확인/후속 evidence:** 후속 differential tests가 hit record 전체와 pixels를 비교하고 primitive work reduction을 확인하는 경로와 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `f7e969537c10`
- 다음 Thread commit: `41c9a59f27a6`
- 이 commit이 다음 단계에 제공하는 것: `41c9a59f27a6`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.12 `41c9a59f27a6` — `test(accel): 선형 탐색과 BVH 결과 동치 검증`

- Importance: A
- Tags: TEST, ACCEL, DETERMINISM
- Thread order: 12/16

#### Source에서 확정된 역할

- Development Thread role: Proves linear/BVH hit and pixel equivalence while checking work reduction.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** BVH traversal이 compile되고 빨라 보여도 hit boolean만 같다고 material, oriented normal, source shape identity, final bytes가 같다는 뜻은 아닙니다.
- **핵심 구현 결정:** `tests/accel_tests.cpp`가 empty/single/unbounded Plane/arbitrary-axis Cylinder/exact overlapping sphere cases에서 Linear과 Bvh의 hit boolean, `t`, point, normal, material, shape pointer를 비교합니다. dense scene은 전체 pixel vector와 checksum을 exact 비교하고, BVH primitive tests가 linear의 1/4 미만인지 검사합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/accel_tests.cpp — differential hit and render tests
 - src/scene.cpp — both intersection modes
 - src/renderer.cpp — full image path
- **caller → callee / data flow:** 동일 Scene/ray 또는 settings → Linear result/stats → Bvh result/stats → full-record/pixel/checksum/work assertions
- **ownership·state transition:** 동일 source Scene에서 두 derived execution mode만 바뀝니다. exact overlapping sphere case는 original index tie rule을 직접 노립니다.
- **failure/edge branch:** 다른 hit pointer/material/normal이나 다른 pixel을 만드는 “빠른” BVH는 work gate 전에 실패합니다.

#### Test commit 분석 기준

- **대상 production invariant:** Linear/BVH는 traversal order와 무관하게 같은 semantic winner와 image bytes를 만듭니다.
- **test technique:** full-record differential assertions, exact buffer/checksum comparison, primitive-count ratio assertion
- **통과하는 production path:** `Scene::intersect` 두 mode와 full `renderScene`
- **이 test가 증명하는 것:** 대표 edge와 dense workload에서 결과 동치 및 primitive pruning을 증명합니다.
- **이 test가 증명하지 않는 것:** 전체 입력 공간, wall-clock portability, source geometry immutability를 아직 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 선택된 geometry edge와 dense render에서 semantic equivalence와 유의미한 primitive work reduction을 함께 고정합니다.
- **이 SHA가 보장하지 않는 것:** 모든 가능한 scene과 platform time speedup을 증명하지 않습니다. stale geometry mutation은 별도 test가 필요합니다.
- **직접 확인/후속 evidence:** 테스트 코드 분류: deterministic differential regression + integration pixel equivalence + semantic-work gate. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `d4f6ee5b6042`
- 다음 Thread commit: `da3e8b43d09e`
- 이 commit이 다음 단계에 제공하는 것: `da3e8b43d09e`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.13 `da3e8b43d09e` — `perf(benchmark): 선형 탐색과 BVH 작업량 비교`

- Importance: A
- Tags: PERF, ACCEL, DETERMINISM
- Thread order: 13/16

#### Source에서 확정된 역할

- Development Thread role: Measures both modes under the same correctness constraints.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** 기존 benchmark는 단일 execution path만 반복하므로 BVH가 들어온 뒤 같은 workload에서 결과와 work를 직접 나란히 비교하지 못합니다.
- **핵심 구현 결정:** dense Scene의 acceleration을 build하고 Linear/Bvh 각각 warm-up과 5회 측정을 수행합니다. 각 mode 내부 checksum/primitive/AABB counts 일치와 두 mode 간 checksum equality를 확인한 뒤 median elapsed와 work를 한 report에 냅니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - benchmarks/render_benchmark.cpp — two-mode measurement/report
- **caller → callee / data flow:** same built Scene → Linear warmup/samples → Bvh warmup/samples → internal consistency → cross-mode checksum equality → paired report
- **ownership·state transition:** workload와 output semantics는 공유되고 mode만 독립 변수입니다. elapsed는 mode별 median, counters는 deterministic work입니다.
- **failure/edge branch:** 다른 image를 만드는 acceleration은 speed comparison 전에 거부됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 동일 correctness constraints 아래 두 mode의 work/time을 비교하는 benchmark 구조를 제공합니다.
- **이 SHA가 보장하지 않는 것:** 아직 versioned schema나 hard primitive ratio gate가 없습니다.
- **직접 확인/후속 evidence:** 두 mode loop와 cross-mode checksum check를 source에서 확인했습니다. benchmark를 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `41c9a59f27a6`
- 다음 Thread commit: `9b77225cf6b7`
- 이 commit이 다음 단계에 제공하는 것: `9b77225cf6b7`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.14 `9b77225cf6b7` — `perf(benchmark): 측정 schema와 가속 기준 검증 고정`

- Importance: A
- Tags: PERF, ACCEL, DETERMINISM
- Thread order: 14/16

#### Source에서 확정된 역할

- Development Thread role: Fixes a versioned benchmark schema and enforces a primitive-test reduction threshold.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** 사람이 읽는 benchmark 출력만 있으면 field 의미와 workload가 drift할 수 있고, BVH가 거의 pruning하지 않아도 checksum만 같으면 성공할 수 있습니다.
- **핵심 구현 결정:** report에 `schemaVersion: 1`과 고정 configuration을 넣고 reference JSON을 추가합니다. per-run consistency를 유지한 채 `bvhPrimitiveTests * 4 < linearPrimitiveTests`를 강제하고 primitive ratio와 elapsed speedup을 별도 field로 냅니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - benchmarks/render_benchmark.cpp — versioned schema and gate
 - benchmarks/reference.json — fixed reference shape/configuration
- **caller → callee / data flow:** correctness-consistent samples → schema/config report → primitive ratio gate → ratios/speedup output
- **ownership·state transition:** schema와 workload metadata가 machine-comparable contract가 됩니다. gate는 deterministic integer work에 걸리고 elapsed speedup은 보고만 합니다.
- **failure/edge branch:** image가 같지만 culling이 부족한 regression은 primitive ratio gate에서 실패합니다. 환경이 느려 wall-clock speedup이 작아도 deterministic gate와 구분됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** benchmark contract drift와 심각한 pruning regression을 자동으로 감지합니다.
- **이 SHA가 보장하지 않는 것:** wall-clock 성능을 모든 머신에서 보장하지 않으며 reference 수치는 환경 결과로 재생성될 수 있습니다.
- **직접 확인/후속 evidence:** schema version, configuration fields와 4× inequality를 확인했습니다. 실제 reference run은 수행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `da3e8b43d09e`
- 다음 Thread commit: `ef5320a83c27`
- 이 commit이 다음 단계에 제공하는 것: `ef5320a83c27`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.15 `ef5320a83c27` — `fix(accel): 가속 구조의 도형 불변식 보호`

- Importance: S
- Tags: ARCH, ACCEL, SCENE
- Thread order: 15/16

#### Source에서 확정된 역할

- Development Thread role: Prevents callers from mutating geometry behind a ready BVH.

#### S-level architecture와 invariant 복원

- **직전 관련 상태:** 초기 lifecycle은 `addShape`를 통한 shape-set 변경은 invalidated했지만, public shape storage나 public Sphere/Plane/Cylinder geometry field를 caller가 직접 바꾸면 ready BVH bounds가 stale해질 수 있었습니다. algorithm 자체가 맞아도 source와 cache가 달라지는 correctness hole입니다.
- **핵심 구현 결정:** built-in geometry fields와 Scene shape storage를 private `*_` 상태로 바꾸고 const getter만 노출합니다. Scene은 shape를 정확히 한 번 소유하며 `shapeCount`와 `const Shape& shapeAt` 같은 read-only 접근만 제공합니다. 외부 code가 geometry를 재배치하거나 bounds-affecting field를 대입할 수 없게 compile-time API를 닫습니다.

#### Failure → Fix 연결

- **기존 가정:** `addShape` invalidation만 있으면 ready BVH가 항상 current geometry를 표현합니다.
- **실제 failure 또는 위험:** public geometry/storage mutation은 `addShape`를 거치지 않아 ready cache가 stale해집니다.
- **root cause:** derived cache source의 mutation 권한이 acceleration owner 밖에 노출돼 있었습니다.
- **수정된 decision/invariant:** Scene 단일 소유와 built-in geometry const-only API로 mutation channel을 제거합니다.
- **regression 연결:** `13f153e23920`이 compile-time 비공개성과 build→invalidate→fallback→rebuild를 함께 검증합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/geometry.hpp — private geometry fields/const getters
 - src/geometry.cpp — getter-backed implementations
 - include/ray/scene.hpp — private shape storage and read-only access
 - src/scene.cpp — ownership/access adaptation
- **caller → callee / data flow:** validated geometry construction → Scene ownership transfer → immutable built-in geometry read → acceleration build; 이후 변경은 Scene mutation API를 통해서만 일어나 invalidation 가능
- **ownership·state transition:** authoritative source geometry는 Scene-owned shape object이고 BVH는 그 immutable snapshot의 derived cache입니다. caller는 non-owning const view만 받습니다.
- **failure/edge branch:** 수정 전에는 ready flag가 true인 채 geometry center/radius/axis가 바뀌어 old box가 new shape를 cull할 수 있었습니다. 단순 rebuild 빈도 증가가 아니라 mutation channel 차단이 root cause 수정입니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 지원 built-in geometry와 shape ordering이 acceleration lifecycle 뒤에서 변하지 않습니다. Scene mutation API가 cache invalidation의 단일 경계가 됩니다.
- **이 SHA가 보장하지 않는 것:** 새로운 mutable custom Shape type이 내부적으로 geometry를 바꾸는 경우까지 일반적으로 막는 type system은 아닙니다.
- **직접 확인/후속 evidence:** `13f153e23920`의 compile-time accessibility checks와 runtime state transition test로 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `9b77225cf6b7`
- 다음 Thread commit: `13f153e23920`
- 이 commit이 다음 단계에 제공하는 것: `13f153e23920`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.16 `13f153e23920` — `test(accel): 장면 변경과 가속 상태 불변식 검증`

- Importance: A
- Tags: TEST, ACCEL, SCENE
- Thread order: 16/16

#### Source에서 확정된 역할

- Development Thread role: Verifies immutability, invalidation, linear fallback, and rebuild as one state transition.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** immutability fix가 선언 변경만으로 끝나면 실제 invalidation/fallback/rebuild 전이가 깨졌는지 알기 어렵고, public field가 실수로 다시 노출돼도 runtime test만으로는 잡지 못할 수 있습니다.
- **핵심 구현 결정:** `tests/accel_tests.cpp`에 public `shapes` 접근 불가, `shapeAt` 결과의 const성, geometry getter 비대입 가능성을 compile-time trait/assertion으로 고정합니다. runtime에서는 acceleration build 후 shape 추가가 ready를 끄는지, Bvh 요청이 stale tree를 쓰지 않고 current linear geometry로 fallback하는지, rebuild 뒤 같은 hit를 내는지 연속으로 확인합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/accel_tests.cpp — immutability compile-time checks and acceleration lifecycle regression
 - src/scene.cpp — `addShape`, ready flag, fallback, rebuild
- **caller → callee / data flow:** Scene build → acceleration ready → shape addition → ready false → Bvh-mode query/linear fallback sees new shape → rebuild → ready true → same semantic hit
- **ownership·state transition:** source set과 derived state의 다섯 단계가 하나의 test 안에서 observable합니다. compile-time 부분은 mutation API가 아예 형성되지 않음을 검사합니다.
- **failure/edge branch:** stale BVH를 그대로 사용하면 new/changed shape hit가 누락되고, fallback이 old tree를 참조하면 pre/post rebuild 결과가 달라집니다.

#### Test commit 분석 기준

- **대상 production invariant:** ready BVH는 current immutable source를 설명하고, source-set mutation 뒤에는 사용되지 않습니다.
- **test technique:** type traits/static assertions + build/invalidate/fallback/rebuild runtime sequence
- **통과하는 production path:** Scene ownership/accessors, `addShape`, `buildAcceleration`, mode-aware `intersect`
- **이 test가 증명하는 것:** 공개 mutation channel 차단과 stale-cache fallback/recovery를 증명합니다.
- **이 test가 증명하지 않는 것:** concurrent mutation이나 user-defined mutable Shape 내부를 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 지원 API에서 geometry immutability, mutation invalidation, safe fallback, rebuild recovery를 하나의 lifecycle invariant로 고정합니다.
- **이 SHA가 보장하지 않는 것:** arbitrary custom shape의 내부 mutable state와 concurrent Scene mutation은 범위 밖입니다.
- **직접 확인/후속 evidence:** 테스트 성격: compile-time API regression + deterministic state-transition regression. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `ef5320a83c27`
- 다음 Thread commit: 이 Thread의 종료점
- 이 commit이 Thread 종료에 제공하는 것: Thread-level invariant ledger와 최종 실행 순서에서이 SHA의 결과를 최종 상태에 반영했습니다.

## 6. Invariant ledger

| Invariant | 최초 도입/기준 | 강화 또는 수정 | 부족함/위험 노출 | 고정한 test/evidence | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| 비교 가능한 semantic work | f4dcb50939e2 | f5a2c4ade16d | 시간만으로 correctness/work를 판단할 위험 | 4fb2345c7d35/f5a2c4ade16d | actual dispatch counters, warm-up, five-run median |
| 보수적 ray-box/shape bounds | 7b19f2ad78e3 | b782e22450d8 | parallel/contact 비교와 cylinder rounding false negative | 419d52d687fc | closed slab interval, nullopt plane, outward nextafter |
| 결정적 BVH build | bb65e8092632 | bb65e8092632 | centroid tie에서 unstable order | 41c9a59f27a6 | largest extent axis, centroid+shapeIndex stable ordering, leaf≤4 |
| 선형과 동일한 semantic winner | 41a1d6bbe5ef | 9a7f29b5d78a/d4f6ee5b6042 | tree traversal order가 equal-`t` winner 변경 | 41c9a59f27a6 | smaller t 또는 equal t/larger original index |
| BVH는 Scene source의 derived state | f7e969537c10 | ef5320a83c27 | public mutation 뒤 ready cache stale | 13f153e23920 | Scene single ownership, const geometry, invalidate/fallback/rebuild |
| 동일 결과와 충분한 work reduction | 41c9a59f27a6/da3e8b43d09e | 9b77225cf6b7 | 다른 image 또는 미미한 pruning을 성능 성공으로 오인 | versioned benchmark gate | checksum equality + `bvh*4 < linear` |

### Ledger 보완 기록

- 각 invariant는 위 표의 SHA에서 observable behavior 또는 state로 처음 나타났습니다.
- 후속 commit이 같은 용어를 사용하더라도 그 보장을 과거 SHA에 소급하지 않았습니다.
- test/evidence 열은 production path와 assertion 또는 deterministic work gate를 함께 가리킵니다.
- 실행하지 않은 test는 source-level evidence로만 기록했습니다.

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Decision/Fix | Test 또는 evidence | 실제 실패 처리와 assertion |
| --- | --- | --- | --- |
| AABB contact/parallel 처리 오류 | closed slab interval과 zero-direction branch | 419d52d687fc | boundary hit/parallel miss/entry assertions |
| cylinder bounds가 실제 geometry를 자름 | axis projection + outward expansion | 419d52d687fc | 대표 surface point containment |
| BVH reorder가 exact tie winner 변경 | original shape index 포함 candidate rule | 41c9a59f27a6 | overlapping shape full-record differential |
| unbounded Plane가 BVH에서 누락 | bounded/unbounded partition과 final linear pass | 41c9a59f27a6 | Plane case와 full pixel equivalence |
| shape mutation 뒤 stale ready BVH | private immutable source + mutation invalidation/fallback | 13f153e23920 | compile-time API checks + build→add→fallback→rebuild |
| checksum은 같지만 culling 회귀 | versioned primitive-count ratio gate | 9b77225cf6b7 benchmark | `bvhPrimitiveTests*4 < linearPrimitiveTests` |

### 연결 검토

- feature commit도 어떤 잘못된 state 또는 semantic drift를 막는지 production path에 연결했습니다.
- fix commit은 기존 가정 → 실제 위험 → root cause → corrected decision → regression 순서로 기록했습니다.
- test가 broad integration인지 deterministic boundary/differential/failure-injection regression인지 commit 기록에서 구분했습니다.
- assertion이 증명하지 않는 범위와 실행하지 못한 항목을 별도로 남겼습니다.

## 8. Ownership / state / responsibility 변화

최종 상태에서 `Scene`이 shape를 정확히 한 번 소유하고 built-in geometry를 const-only로
노출합니다. `Bvh` node/index 배열과 unbounded index list는 source shape set에서 파생된
Scene-owned cache입니다. BVH leaf references는 shape를 소유하지 않으므로 Scene lifetime에
종속됩니다. `addShape`는 source mutation과 cache invalidation의 단일 경계이고,
`buildAcceleration`은 current source를 partition/build한 뒤 ready를 commit합니다.
ready가 false인 Bvh-mode query는 stale cache를 읽지 않고 current source의 linear path로
fallback합니다. query-local stack, closest distance, winner index는 traversal 중에만 변합니다.

### 학습자 최종 기록

- **source state와 derived state:** 최종 상태에서 `Scene`이 shape를 정확히 한 번 소유하고 built-in geometry를 const-only로 노출합니다. `Bvh` node/index 배열과 unbounded index list는 source shape set에서 파생된 Scene-owned cache입니다. BVH leaf references는 shape를 소유하지 않으므로 Scene lifetime에 종속됩니다. `addShape`는 source mutation과 cache invalidation의 단일 경계이고, `buildAcceleration`은 current source를 partition/build한 뒤 ready를 commit합니다. ready가 false인 Bvh-mode query는 stale cache를 읽지 않고 current source의 linear path로 fallback합니다. query-local stack, closest distance, winner index는 traversal 중에만 변합니다.
- **mutation/transition boundary:** commit별 `ownership·state transition`과 위 invariant ledger에 표시했습니다.
- **failure 시 복구 상태:** Failure → Fix → Test 표와 각 fix/test section에 정상·오류 상태를 구분했습니다.

## 9. Thread 최종 상태

BVH는 선형 renderer를 대체하는 별도 의미가 아니라 같은 candidate rule을 더 적은 primitive
dispatch로 실행하는 derived structure입니다. conservative bounds와 unbounded partition이
hit 누락을 막고, deterministic build/near-first stack이 reproducible work order를 제공하며,
original index tie-break가 traversal order로부터 semantic winner를 분리합니다. ownership fix는
source geometry를 cache 뒤에서 바꾸는 API를 제거했고, lifecycle test는 invalidation, fallback,
rebuild를 고정합니다. correctness equality가 먼저 통과한 뒤에만 versioned benchmark가 work
reduction과 환경 의존적 elapsed speedup을 보고합니다.

### 직접 작성한 결론

- **Thread 시작과 종료의 behavior 차이:** BVH는 선형 renderer를 대체하는 별도 의미가 아니라 같은 candidate rule을 더 적은 primitive dispatch로 실행하는 derived structure입니다. conservative bounds와 unbounded partition이 hit 누락을 막고, deterministic build/near-first stack이 reproducible work order를 제공하며, original index tie-break가 traversal order로부터 semantic winner를 분리합니다. ownership fix는 source geometry를 cache 뒤에서 바꾸는 API를 제거했고, lifecycle test는 invalidation, fallback, rebuild를 고정합니다. correctness equality가 먼저 통과한 뒤에만 versioned benchmark가 work reduction과 환경 의존적 elapsed speedup을 보고합니다.
- **아직 다른 Thread 또는 외부 검증이 보완해야 하는 항목:** concurrent Scene mutation과 arbitrary user-defined mutable Shape 내부 상태는 지원 contract 밖입니다. elapsed speedup은 실행 환경에서 별도 측정해야 합니다.

## 10. 최종 architecture 또는 execution flow 정리

### Source가 확정한 흐름 anchor

```text
work baseline → `Aabb`/shape bounds → deterministic `Bvh` build → Scene-owned derived state → mode-aware `Scene::intersect` → explicit-stack traversal → differential tests/benchmark → geometry immutability fix
```

### 실제 코드로 완성한 흐름

1. 계측된 직렬 renderer가 ray 종류, primitive dispatch와 elapsed baseline을 수집합니다.
2. 각 finite shape는 conservative optional AABB를 제공하고 Plane은 unbounded로 남습니다.
3. Scene build가 bounded refs와 unbounded indices를 분리합니다.
4. BVH builder가 longest centroid extent를 고르고 original index tie-break로 median split합니다.
5. query는 Linear/not-ready이면 full linear path, ready Bvh이면 explicit stack path를 고릅니다.
6. node AABB entry와 current closest로 prune하고 near child를 먼저 처리합니다.
7. leaf와 unbounded candidate 모두 `(t, original index)` 규칙으로 authoritative record를 갱신합니다.
8. differential tests가 full record, pixels, checksum을 비교하고 work reduction을 검사합니다.
9. source mutation은 ready를 끄며 fallback 뒤 explicit rebuild로 cache를 다시 commit합니다.
10. benchmark는 correctness-consistent samples에서 schema와 primitive ratio/median time을 냅니다.

### 학습자의 최종 설명

BVH는 선형 renderer를 대체하는 별도 의미가 아니라 같은 candidate rule을 더 적은 primitive
dispatch로 실행하는 derived structure입니다. conservative bounds와 unbounded partition이
hit 누락을 막고, deterministic build/near-first stack이 reproducible work order를 제공하며,
original index tie-break가 traversal order로부터 semantic winner를 분리합니다. ownership fix는
source geometry를 cache 뒤에서 바꾸는 API를 제거했고, lifecycle test는 invalidation, fallback,
rebuild를 고정합니다. correctness equality가 먼저 통과한 뒤에만 versioned benchmark가 work
reduction과 환경 의존적 elapsed speedup을 보고합니다.

남은 경계는 다음과 같습니다. concurrent Scene mutation과 arbitrary user-defined mutable Shape 내부 상태는 지원 contract 밖입니다. elapsed speedup은 실행 환경에서 별도 측정해야 합니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 source 순서대로 확인했습니다.
- [x] 각 commit의 SHA, subject, importance, tags를 그대로 유지했습니다.
- [x] 모든 핵심 설명에 해당 SHA의 file path와 symbol 근거를 기록했습니다.
- [x] final HEAD의 구조를 과거 SHA에 소급하지 않았습니다.
- [x] S/A/B importance에 맞춰 architecture, subsystem, localized role의 깊이를 구분했습니다.
- [x] source에서 확정하지 않은 실행 결과나 runtime 수치를 사실로 채우지 않았습니다.
- [x] failure와 fix/test를 실제 production path로 연결했습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] invariant ledger의 각 변화를 commit evidence와 연결했습니다.
- [ ] 해당 SHA checkout에서 테스트·benchmark·sanitizer를 직접 실행했습니다. 환경 제한 때문에 미실행 상태입니다.
- [x] 별도의 프로젝트 재학습 없이이 Thread의 설계 → 구현 → 위험 → 수정 → 검증 발전을 설명할 수 있는 기록을 남겼습니다.
===== END FILE: 03-correctness-preserving-bvh.md =====

===== BEGIN FILE: 04-material-syntax-and-reflection.md =====
# Thread 4. Material syntax to bounded recursive reflection

## 1. Thread 목표

기존 diffuse-only `traceRay`에 deterministic perfect-metal recursion을 추가하고, scene syntax와 CLI depth setting이 그 bounded transport contract를 어떻게 노출하면서 기존 diffuse scene을 보존하는지 확인합니다.

### Source significance

> The material thread changes `traceRay` from a terminal direct-light computation into bounded
> recursion without introducing randomness or schedule-dependent sampling. Keeping omitted material
> tokens diffuse preserves existing scenes, while the depth contract makes recursive work finite and
> externally configurable. The tests show both the new metal path and the unchanged diffuse golden,
> which is the relevant compatibility boundary.

### 이 Thread에 연결된 source invariant

- Recursive metal transport is bounded by `maxDepth`.
- Omitting the material token preserves diffuse behavior and the existing diffuse golden.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- material value가 `Diffuse`와 `Metal`을 구분한 뒤 `traceRay` control flow는 어떻게 갈라지는가?
- metal depth zero와 depth positive의 반환·secondary-ray 생성 규칙은 무엇인가?
- reflected ray origin offset, acceleration mode, statistics sink, decremented depth가 재귀 호출에 어떻게 전달되는가?
- optional material token이 기존 `.rt` arity와 diffuse default를 어떻게 보존하는가?
- test가 grammar, unknown-material failure, recursion, secondary count, diffuse golden을 각각 어떤 경로로 검증하는가?

## 3. 완료 기준

- [x] metal reflection의 실제 수식과 ray 생성 순서를 해당 SHA에서 기록했습니다.
- [x] depth가 감소하는 지점과 depth zero 종료 결과를 코드로 설명할 수 있습니다.
- [x] `sp`, `pl`, `cy`의 base arity와 optional-token arity를 실제 parser 분기에서 확인했습니다.
- [x] 기존 material 생략 scene이 diffuse로 남는 backward-compatibility 증거를 테스트와 연결했습니다.
- [x] CLI `--max-depth`의 default, 범위, 중복·오류 처리를 renderer setting까지 추적했습니다.
- [x] 모든 참조 SHA가 `cpp/miniRT` branch HEAD의 ancestry에 속하는지 확인했습니다.
- [ ] 해당 SHA checkout에서 build/test/benchmark 명령을 직접 실행했습니다. 로컬 외부 네트워크와 checkout이 제공되지 않아 실행 evidence는 만들지 않았습니다.

### 검증 범위

- 지정 branch HEAD: `7d08c7c13fa68c3e60eea3c7014658b0a133e6f0`
- 각 참조 SHA는 Thread 내부의 연속 compare chain에서 `behind_by = 0`, merge base가 선행 SHA였고, Thread 종료 SHA도 branch HEAD의 조상으로 확인했습니다.
- 구현 설명은 해당 commit의 diff/file content를 기준으로 작성했으며, final HEAD의 후속 API를 과거 SHA에 소급하지 않았습니다.
- 테스트와 benchmark는 source mechanism과 production path만 검사했습니다. 실행 결과, sanitizer 결과, wall-clock 수치는 기록하지 않았습니다.

## 4. Commit map

1. `85583e1e9beb` — `feat(material): metal 모델과 깊이 제한 반사 구현`
 - Importance: A
 - Tags: CORE, MATERIAL, RAY_PIPELINE
 - Source-defined role: Extends tracing with a deterministic perfect-metal branch and depth consumption.

2. `a90130a5b030` — `feat(parser): 선택적 도형 재질 문법 추가`
 - Importance: B
 - Tags: PARSER, MATERIAL
 - Source-defined role: Adds optional material tokens while retaining diffuse defaults.

3. `9a352ffe8233` — `test(material): 재질 파싱과 반사 깊이 검증`
 - Importance: B
 - Tags: TEST, MATERIAL, DETERMINISM
 - Source-defined role: Verifies parsing, unknown-material failure, recursion depth, secondary-ray counts, and diffuse compatibility.

4. `3aa806753cc4` — `feat(cli): 반사 깊이 option과 기본값 추가`
 - Importance: B
 - Tags: CLI, MATERIAL
 - Source-defined role: Exposes reflection depth through the CLI and adopts a default of four.

## 5. Commit별 학습 기록

### 5.1 `85583e1e9beb` — `feat(material): metal 모델과 깊이 제한 반사 구현`

- Importance: A
- Tags: CORE, MATERIAL, RAY_PIPELINE
- Thread order: 1/4

#### Source에서 확정된 역할

- Development Thread role: Extends tracing with a deterministic perfect-metal branch and depth consumption.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** `traceRay`는 모든 material을 ambient/direct diffuse로 끝내며 `maxDepth`를 실제로 소비하지 않았습니다. reflective object를 추가하면 종료 조건, secondary-ray accounting, self-intersection offset과 recursive state 전달을 명시해야 합니다.
- **핵심 구현 결정:** `MaterialType`에 `Diffuse`와 `Metal`을 두고 default를 Diffuse로 유지합니다. hit material이 Metal이면 depth가 0일 때 black을 반환하고, 그 외에는 `r = d - 2·dot(d,n)·n`으로 perfect reflection direction을 계산합니다. origin은 oriented normal 방향으로 epsilon offset하고 secondary counter를 증가시킨 뒤 같은 Scene, acceleration mode, stats sink와 `depth-1`을 재귀 호출합니다. 반환은 albedo component-wise 곱으로 tint합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/material.hpp — `MaterialType`, default material
 - src/shading.cpp — metal branch in `traceRay`
 - include/ray/renderer.hpp — depth-aware tracing signature
- **caller → callee / data flow:** primary/secondary ray → hit material → Diffuse: 기존 direct lighting; Metal: depth check → reflected ray → recursive `traceRay(depth-1)` → albedo modulation
- **ownership·state transition:** depth는 recursive budget이며 각 metal bounce에서만 감소합니다. Scene과 camera/geometry는 read-only로 공유되고 stats sink에 secondary count가 누적됩니다.
- **failure/edge branch:** depth 감소가 없으면 mirror cycle이 무한 재귀로 이어질 수 있고, reflected origin offset이 없으면 방금 hit한 surface를 즉시 다시 맞을 수 있습니다. depth 0은 partial diffuse fallback이 아니라 명시적 black입니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** randomness 없는 bounded perfect-metal transport와 기존 diffuse terminal path의 공존을 정의합니다.
- **이 SHA가 보장하지 않는 것:** roughness, Fresnel, refraction, stochastic sampling은 구현하지 않습니다. CLI default depth는 후속 SHA에서 바뀝니다.
- **직접 확인/후속 evidence:** material enum, branch, reflection formula, depth decrement, stats 전달을 해당 SHA에서 확인하고 후속 tests에 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: 이 Thread의 시작점
- 다음 Thread commit: `a90130a5b030`
- 이 commit이 다음 단계에 제공하는 것: `a90130a5b030`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.2 `a90130a5b030` — `feat(parser): 선택적 도형 재질 문법 추가`

- Importance: B
- Tags: PARSER, MATERIAL
- Thread order: 2/4

#### Source에서 확정된 역할

- Development Thread role: Adds optional material tokens while retaining diffuse defaults.

#### B-level 구현 역할 복원

- **직전 관련 상태:** renderer가 material type을 이해해도 `.rt` grammar가 모든 shape를 diffuse로만 만들면 feature를 입력에서 선택할 수 없습니다. 반대로 token을 필수로 만들면 기존 scene이 깨집니다.
- **핵심 구현 결정:** `sp`, `pl`, `cy` directive가 기존 base arity 또는 base+1 arity를 허용합니다. token이 없으면 Diffuse, 정확히 `diffuse`/`metal`이면 해당 type, 다른 문자열이나 surplus token은 source-located `ParseError`입니다. material 검증은 shape를 Scene에 추가하기 전에 완료됩니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/parser.cpp — optional material-token parser and shape directive arity branches
- **caller → callee / data flow:** shape tokens → base/base+1 arity check → optional material decode/default → geometry validation/construction → `Scene::addShape`
- **ownership·state transition:** omitted token은 기존 Material default를 명시적으로 보존합니다. invalid token에서는 Scene mutation 전 예외가 발생합니다.
- **failure/edge branch:** unknown token을 diffuse로 조용히 처리하면 오타가 렌더 결과만 바꾸고 오류가 드러나지 않습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 기존 scene syntax의 backward compatibility와 explicit metal 선택을 동시에 제공합니다.
- **이 SHA가 보장하지 않는 것:** material parameter(roughness 등) 확장 grammar는 없습니다.
- **직접 확인/후속 evidence:** 세 shape directive의 arity/default/unknown branches를 해당 SHA에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `85583e1e9beb`
- 다음 Thread commit: `9a352ffe8233`
- 이 commit이 다음 단계에 제공하는 것: `9a352ffe8233`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.3 `9a352ffe8233` — `test(material): 재질 파싱과 반사 깊이 검증`

- Importance: B
- Tags: TEST, MATERIAL, DETERMINISM
- Thread order: 3/4

#### Source에서 확정된 역할

- Development Thread role: Verifies parsing, unknown-material failure, recursion depth, secondary-ray counts, and diffuse compatibility.

#### B-level 구현 역할 복원

- **직전 관련 상태:** material implementation과 grammar가 있어도 parser compatibility, exact recursive result, depth stop, secondary work, diffuse baseline을 한꺼번에 고정하는 regression이 없습니다.
- **핵심 구현 결정:** `tests/material_tests.cpp`가 omitted/explicit diffuse/metal parsing과 unknown token failure를 검사합니다. mirror Plane과 constant background를 사용해 depth 1 결과가 background `(0.25,0.5,0.75)`와 albedo `(0.8,0.5,0.25)`의 곱 `(0.2,0.25,0.1875)`인지 확인하고, depth 0 black, repeat determinism, `secondaryRays == 1`을 고정합니다. 기존 diffuse scene checksum `456dc8d87ebf194f`도 유지합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/material_tests.cpp — parser/reflection/depth/stats/golden regressions
 - src/parser.cpp — optional grammar
 - src/shading.cpp — `traceRay` material branch
- **caller → callee / data flow:** parser cases → material assertions/errors; controlled mirror scene → `traceRay` depth 1/0 → exact color and counter; diffuse render → golden checksum
- **ownership·state transition:** mirror fixture는 light contribution 없이 recursive background만 관찰하게 구성되어 reflection arithmetic을 격리합니다.
- **failure/edge branch:** depth가 감소하지 않거나 albedo 곱 위치가 바뀌거나 omitted token default가 바뀌면 서로 다른 assertion이 실패합니다.

#### Test commit 분석 기준

- **대상 production invariant:** material omission은 diffuse를 유지하고 metal recursion은 depth로 제한되며 deterministic합니다.
- **test technique:** direct parser assertions, controlled mirror/background fixture, exact counter/color, full-render golden
- **통과하는 production path:** parser → Material → HitRecord → `traceRay` recursion → image checksum
- **이 test가 증명하는 것:** 대표 metal path와 기존 diffuse behavior가 동시에 보호됩니다.
- **이 test가 증명하지 않는 것:** rough materials, refraction, arbitrary deep scenes나 wall-clock behavior를 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** grammar, bounded recursion, secondary accounting, deterministic metal color, diffuse compatibility를 고정합니다.
- **이 SHA가 보장하지 않는 것:** 복잡한 multi-bounce scene, all material combinations, physical realism은 증명하지 않습니다. 이 SHA의 test code가 public Scene storage를 쓰더라도 후속 immutability API를 과거에 소급하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 성격: parser boundary + deterministic unit/integration regression + compatibility golden. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `a90130a5b030`
- 다음 Thread commit: `3aa806753cc4`
- 이 commit이 다음 단계에 제공하는 것: `3aa806753cc4`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.4 `3aa806753cc4` — `feat(cli): 반사 깊이 option과 기본값 추가`

- Importance: B
- Tags: CLI, MATERIAL
- Thread order: 4/4

#### Source에서 확정된 역할

- Development Thread role: Exposes reflection depth through the CLI and adopts a default of four.

#### B-level 구현 역할 복원

- **직전 관련 상태:** reflection depth가 library setting에만 있으면 user가 CLI에서 조절할 수 없고 초기 default 1은 한 bounce만 허용합니다.
- **핵심 구현 결정:** `src/main.cpp`의 option parser에 `--max-depth`를 추가하고 unsigned/integer input을 0..32로 제한합니다. option 중복, 값 누락, malformed/out-of-range는 usage failure로 처리합니다. `RenderSettings` 기본 depth를 1에서 4로 바꾸고 parsed value를 renderer에 전달합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/main.cpp — `--max-depth` parsing and validation
 - include/ray/renderer.hpp — default `maxDepth`
 - benchmarks/tests callers — explicit setting adaptation where needed
- **caller → callee / data flow:** argv scan → option/value validation → settings.maxDepth → render → `traceRay` recursive budget
- **ownership·state transition:** 0은 metal recursion disabled/black-at-metal-hit 의미를 유지하고, 1..32는 최대 bounce budget입니다. default 4는 CLI와 settings construction에 적용됩니다.
- **failure/edge branch:** 무제한 값은 stack/work 폭증을 허용하고, duplicate option은 ambiguous authority를 만듭니다. 둘 다 renderer 전에 거부됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** bounded reflection budget을 external command contract로 노출하고 합리적 default를 정합니다.
- **이 SHA가 보장하지 않는 것:** depth는 quality/performance knob이지 physically convergent path tracer 설정이 아닙니다.
- **직접 확인/후속 evidence:** default change, range 0..32, duplicate/missing-value branches와 settings 전달을 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `9a352ffe8233`
- 다음 Thread commit: 이 Thread의 종료점
- 이 commit이 Thread 종료에 제공하는 것: Thread-level invariant ledger와 최종 실행 순서에서이 SHA의 결과를 최종 상태에 반영했습니다.

## 6. Invariant ledger

| Invariant | 최초 도입/기준 | 강화 또는 수정 | 부족함/위험 노출 | 고정한 test/evidence | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| omitted material은 diffuse | 기존 diffuse-only behavior | a90130a5b030 | optional syntax가 old arity를 깨뜨릴 위험 | 9a352ffe8233 | base/base+1 arity와 diffuse golden |
| metal recursion은 bounded | 85583e1e9beb | 3aa806753cc4에서 CLI 0..32/default 4 | depth 감소 누락 시 unbounded recursion | 9a352ffe8233 | depth 0 black, depth 1 exact color, secondary=1 |
| reflection은 deterministic | 85583e1e9beb | 9a352ffe8233 | random/schedule source 없음 | repeat color/checksum tests | perfect reflection formula와 same state propagation |

### Ledger 보완 기록

- 각 invariant는 위 표의 SHA에서 observable behavior 또는 state로 처음 나타났습니다.
- 후속 commit이 같은 용어를 사용하더라도 그 보장을 과거 SHA에 소급하지 않았습니다.
- test/evidence 열은 production path와 assertion 또는 deterministic work gate를 함께 가리킵니다.
- 실행하지 않은 test는 source-level evidence로만 기록했습니다.

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Decision/Fix | Test 또는 evidence | 실제 실패 처리와 assertion |
| --- | --- | --- | --- |
| unknown material token이 silently diffuse 처리 | exact token decode와 ParseError | 9a352ffe8233 parser tests | unknown material failure assertion |
| metal ray 무한 재귀 | depth zero stop와 `depth-1` | 9a352ffe8233 | depth 0 black/depth 1 one secondary |
| feature 추가로 기존 diffuse scene 변화 | omitted token default + unchanged diffuse path | 9a352ffe8233 | golden `456dc8d87ebf194f` |
| 과도하거나 모호한 CLI depth | 0..32 범위와 duplicate/missing reject | CLI contract source inspection | renderer 전에 option parse failure |

### 연결 검토

- feature commit도 어떤 잘못된 state 또는 semantic drift를 막는지 production path에 연결했습니다.
- fix commit은 기존 가정 → 실제 위험 → root cause → corrected decision → regression 순서로 기록했습니다.
- test가 broad integration인지 deterministic boundary/differential/failure-injection regression인지 commit 기록에서 구분했습니다.
- assertion이 증명하지 않는 범위와 실행하지 못한 항목을 별도로 남겼습니다.

## 8. Ownership / state / responsibility 변화

Material은 `HitRecord`에 값으로 복사되며 reflected ray도 stack value입니다. Scene/geometry는 recursion 동안 read-only로 공유됩니다. recursion depth가 remaining-work state이고 stats sink가 secondary count를 누적합니다. parser는 material token 검증 후 shape construction/Scene ownership transfer를 수행하므로 unknown token에서 partial shape를 추가하지 않습니다.

### 학습자 최종 기록

- **source state와 derived state:** Material은 `HitRecord`에 값으로 복사되며 reflected ray도 stack value입니다. Scene/geometry는 recursion 동안 read-only로 공유됩니다. recursion depth가 remaining-work state이고 stats sink가 secondary count를 누적합니다. parser는 material token 검증 후 shape construction/Scene ownership transfer를 수행하므로 unknown token에서 partial shape를 추가하지 않습니다.
- **mutation/transition boundary:** commit별 `ownership·state transition`과 위 invariant ledger에 표시했습니다.
- **failure 시 복구 상태:** Failure → Fix → Test 표와 각 fix/test section에 정상·오류 상태를 구분했습니다.

## 9. Thread 최종 상태

Diffuse는 기존 direct-light path와 golden을 유지하고 Metal은 perfect reflection ray를 depth budget 안에서 재귀 추적합니다. scene syntax는 token 생략을 diffuse로 해석해 backward compatibility를 유지하며, CLI는 default 4와 0..32 범위를 renderer setting에 전달합니다. roughness/refraction/stochastic transport는 범위 밖입니다.

### 직접 작성한 결론

- **Thread 시작과 종료의 behavior 차이:** Diffuse는 기존 direct-light path와 golden을 유지하고 Metal은 perfect reflection ray를 depth budget 안에서 재귀 추적합니다. scene syntax는 token 생략을 diffuse로 해석해 backward compatibility를 유지하며, CLI는 default 4와 0..32 범위를 renderer setting에 전달합니다. roughness/refraction/stochastic transport는 범위 밖입니다.
- **아직 다른 Thread 또는 외부 검증이 보완해야 하는 항목:** 물리 기반 Fresnel, roughness, transparency/refraction, stochastic anti-aliasing은이 material contract에 포함되지 않습니다.

## 10. 최종 architecture 또는 execution flow 정리

### Source가 확정한 흐름 anchor

```text
shape material token/default → `MaterialType` → `HitRecord::material` → `traceRay` diffuse/metal branch → reflected ray with decremented depth → CLI `--max-depth`
```

### 실제 코드로 완성한 흐름

1. parser가 shape base arity 또는 optional material token을 검증합니다.
2. omitted token은 Diffuse, exact `metal`은 Metal 값으로 shape material에 저장됩니다.
3. primary ray hit가 material 값을 `HitRecord`로 전달합니다.
4. `traceRay`가 Diffuse direct-light 또는 Metal branch를 선택합니다.
5. Metal depth 0은 black으로 종료하고, 양수이면 reflected direction과 offset origin을 만듭니다.
6. secondary counter를 올리고 같은 Scene/mode/stats에 `depth-1`을 전달합니다.
7. recursive color에 metal albedo를 곱해 caller로 반환합니다.
8. CLI `--max-depth`가 bounded budget을 외부에서 설정합니다.

### 학습자의 최종 설명

Diffuse는 기존 direct-light path와 golden을 유지하고 Metal은 perfect reflection ray를 depth budget 안에서 재귀 추적합니다. scene syntax는 token 생략을 diffuse로 해석해 backward compatibility를 유지하며, CLI는 default 4와 0..32 범위를 renderer setting에 전달합니다. roughness/refraction/stochastic transport는 범위 밖입니다.

남은 경계는 다음과 같습니다. 물리 기반 Fresnel, roughness, transparency/refraction, stochastic anti-aliasing은이 material contract에 포함되지 않습니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 source 순서대로 확인했습니다.
- [x] 각 commit의 SHA, subject, importance, tags를 그대로 유지했습니다.
- [x] 모든 핵심 설명에 해당 SHA의 file path와 symbol 근거를 기록했습니다.
- [x] final HEAD의 구조를 과거 SHA에 소급하지 않았습니다.
- [x] S/A/B importance에 맞춰 architecture, subsystem, localized role의 깊이를 구분했습니다.
- [x] source에서 확정하지 않은 실행 결과나 runtime 수치를 사실로 채우지 않았습니다.
- [x] failure와 fix/test를 실제 production path로 연결했습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] invariant ledger의 각 변화를 commit evidence와 연결했습니다.
- [ ] 해당 SHA checkout에서 테스트·benchmark·sanitizer를 직접 실행했습니다. 환경 제한 때문에 미실행 상태입니다.
- [x] 별도의 프로젝트 재학습 없이이 Thread의 설계 → 구현 → 위험 → 수정 → 검증 발전을 설명할 수 있는 기록을 남겼습니다.
===== END FILE: 04-material-syntax-and-reflection.md =====

===== BEGIN FILE: 05-deterministic-tiled-rendering.md =====
# Thread 5. Deterministic tiled rendering and worker failure recovery

## 1. Thread 목표

row-major serial renderer를 고정 tile work unit으로 바꾼 뒤, atomic claim과 worker-owned pixels로 병렬화하고, worker count와 schedule이 pixels·checksums·semantic counters를 바꾸지 않으며 worker failure도 caller로 복구되는 과정을 복원합니다.

### Source significance

> The preparatory tile refactor separates pixel addressing from execution order, after which the
> scheduler can claim tiles without overlapping writes. Determinism follows from per-pixel
> independence, fixed intra-pixel operation order, explicit hit tie-breaking, and integer statistic
> merging—not from deterministic thread scheduling. The later exception fix closes the lifecycle gap
> left by the initial parallel implementation: a failure inside `traceRay` must not escape a worker,
> terminate the process, or return a partial successful image.

### 이 Thread에 연결된 source invariant

- A pixel byte is written by exactly one worker; thread count and tile completion order must not change pixels, checksums, or semantic work counts for a fixed mode and scene.
- All started worker threads are joined. Worker failures are surfaced on the caller thread rather than escaping a worker or yielding a partial successful image.

### 이 Thread에 연결된 engineering difficulty

- Parallelizing image generation without overlapping writes or schedule-dependent floating-point accumulation while still producing deterministic statistics.
- Recovering safely from thread creation or worker-body failure and transferring the original failure to the caller after complete thread cleanup.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 16×16 tile decomposition이 image edge에서 모든 픽셀을 정확히 한 번 방문하는 근거는 무엇인가?
- relaxed atomic counter만으로 tile assignment가 충분한 이유와 pixel memory race가 없는 이유는 무엇인가?
- scene/camera shared-read와 worker-local stats가 schedule-dependent 결과를 어떻게 피하는가?
- worker count auto/explicit/cap policy가 scheduler semantics와 benchmark reproducibility에 어떤 영향을 주는가?
- in-process pixel/work equivalence와 CLI PPM byte equivalence가 서로 다른 무엇을 검증하는가?
- 초기 worker body에서 exception이 발생하면 왜 `std::terminate`가 문제였고, 최종 fix는 어떤 cleanup ordering을 보장하는가?

## 3. 완료 기준

- [x] serial tile loop와 parallel worker loop의 동일한 pixel kernel을 비교했습니다.
- [x] tile index → tile bounds → `(x, y)` → RGB offset의 ownership 경로를 기록했습니다.
- [x] atomic ordering, per-worker stats, join, merge 순서를 실제 코드에서 확인했습니다.
- [x] one/four workers와 linear/BVH 조합에서 image와 counters가 같아야 하는 이유를 테스트별로 설명할 수 있습니다.
- [x] worker creation failure와 worker-body failure의 cleanup 경로를 구분했습니다.
- [x] throwing shape regression이 원래 예외 type/message, stop-assignment, join, rethrow를 어떻게 증명하는지 연결했습니다.
- [x] 모든 참조 SHA가 `cpp/miniRT` branch HEAD의 ancestry에 속하는지 확인했습니다.
- [ ] 해당 SHA checkout에서 build/test/benchmark 명령을 직접 실행했습니다. 로컬 외부 네트워크와 checkout이 제공되지 않아 실행 evidence는 만들지 않았습니다.

### 검증 범위

- 지정 branch HEAD: `7d08c7c13fa68c3e60eea3c7014658b0a133e6f0`
- 각 참조 SHA는 Thread 내부의 연속 compare chain에서 `behind_by = 0`, merge base가 선행 SHA였고, Thread 종료 SHA도 branch HEAD의 조상으로 확인했습니다.
- 구현 설명은 해당 commit의 diff/file content를 기준으로 작성했으며, final HEAD의 후속 API를 과거 SHA에 소급하지 않았습니다.
- 테스트와 benchmark는 source mechanism과 production path만 검사했습니다. 실행 결과, sanitizer 결과, wall-clock 수치는 기록하지 않았습니다.

## 4. Commit map

1. `498266fc0abf` — `refactor(render): 직렬 렌더링을 고정 tile 순회로 전환`
 - Importance: B
 - Tags: REFACTOR, CONCURRENCY, DETERMINISM
 - Source-defined role: Converts serial row loop into fixed tile traversal without threads.

2. `849f878ca0b0` — `feat(render): 원자적 tile 분배와 작업자 통계 병합 구현`
 - Importance: S
 - Tags: ARCH, CONCURRENCY, DETERMINISM
 - Source-defined role: Distributes tiles atomically, assigns disjoint pixel writes, merges worker-local stats.

3. `18459bfda416` — `feat(renderer): 작업자 수 설정과 자동 선택 추가`
 - Importance: B
 - Tags: CONCURRENCY, PERF
 - Source-defined role: Adds explicit/automatic worker-count policy.

4. `3619550fa354` — `test(render): 작업자 수에 따른 함수 결과 동치 검증`
 - Importance: A
 - Tags: TEST, CONCURRENCY, DETERMINISM
 - Source-defined role: Verifies equal pixels/work 1 vs 4 workers in both accel modes.

5. `ca2d108f2255` — `test(render): 실행 모드별 PPM byte 결정성 검증`
 - Importance: A
 - Tags: TEST, DETERMINISM, OUTPUT
 - Source-defined role: Verifies exact PPM-byte equality through CLI.

6. `0536e4829070` — `fix(renderer): 작업자 예외를 호출자에게 전달`
 - Importance: A
 - Tags: CONCURRENCY, RISK, DEBUG
 - Source-defined role: Captures worker failures, stops new work, joins all workers, rethrows caller.

7. `b5c708ac981a` — `test(renderer): 작업자 실패 전파와 회수 검증`
 - Importance: A
 - Tags: TEST, CONCURRENCY, RISK
 - Source-defined role: Injects throwing shape to lock propagation/recovery.

## 5. Commit별 학습 기록

### 5.1 `498266fc0abf` — `refactor(render): 직렬 렌더링을 고정 tile 순회로 전환`

- Importance: B
- Tags: REFACTOR, CONCURRENCY, DETERMINISM
- Thread order: 1/7

#### Source에서 확정된 역할

- Development Thread role: Converts serial row loop into fixed tile traversal without threads.

#### B-level 구현 역할 복원

- **직전 관련 상태:** row-major serial loop는 정확하지만 병렬 scheduler가 나눌 명시적인 work unit이 없습니다. 바로 thread를 추가하면 pixel addressing 변화와 scheduling 변화가 한 commit에 섞여 결과 drift 원인을 구분하기 어렵습니다.
- **핵심 구현 결정:** thread를 추가하지 않고 image를 고정 16×16 tile grid로 분해합니다. tile index를 `(tileX,tileY)`로 바꾸고 마지막 tile의 end를 image width/height로 clamp한 뒤 tile 내부를 안정된 y/x 순서로 순회합니다. pixel center, camera ray, trace, clamp, quantization, global RGB offset은 기존과 동일합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/renderer.cpp — serial 16×16 tile decomposition and unchanged pixel kernel
- **caller → callee / data flow:** image dimensions → ceil-div tile counts → row-major tile index → clipped tile bounds → each pixel → existing ray/color/byte path
- **ownership·state transition:** execution은 여전히 한 thread이며 Image가 유일한 mutable output입니다. 각 pixel은 정확히 한 tile에 속합니다.
- **failure/edge branch:** edge tile을 fixed size로 처리하면 image 밖을 쓰거나 마지막 row/column을 누락할 수 있습니다. global offset 대신 tile-local offset을 쓰면 storage 위치가 달라집니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 병렬화 전 work partition을 고정하면서 serial image semantics를 유지합니다.
- **이 SHA가 보장하지 않는 것:** worker, atomic claim, stats merge, failure recovery는 아직 없습니다.
- **직접 확인/후속 evidence:** tile count ceiling, clipped bounds와 기존 pixel kernel 재사용을 해당 SHA에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: 이 Thread의 시작점
- 다음 Thread commit: `849f878ca0b0`
- 이 commit이 다음 단계에 제공하는 것: `849f878ca0b0`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.2 `849f878ca0b0` — `feat(render): 원자적 tile 분배와 작업자 통계 병합 구현`

- Importance: S
- Tags: ARCH, CONCURRENCY, DETERMINISM
- Thread order: 2/7

#### Source에서 확정된 역할

- Development Thread role: Distributes tiles atomically, assigns disjoint pixel writes, merges worker-local stats.

#### S-level architecture와 invariant 복원

- **직전 관련 상태:** fixed tiles가 있어도 여러 worker가 같은 tile/pixel을 처리하지 않게 분배하고, semantic counters를 race 없이 합치며, thread creation 중간 실패 시 시작한 thread를 회수해야 합니다.
- **핵심 구현 결정:** atomic `nextTile.fetch_add(1, memory_order_relaxed)`로 각 tile index를 한 worker에게만 배정합니다. tile partition이 서로 겹치지 않으므로 worker는 lock 없이 global Image의 disjoint byte range를 씁니다. Scene/camera/frame은 shared read-only이고 worker마다 cache-line 정렬된 `RenderStats`를 보유합니다. caller는 모든 worker를 join한 뒤 integer counters를 정해진 순서로 합칩니다. `ThreadJoiner` RAII가 thread construction 또는 caller-side failure에서도 이미 시작한 thread를 join합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/renderer.cpp — worker function, atomic tile claim, thread creation/join, stats merge
 - CMakeLists.txt — Threads dependency
- **caller → callee / data flow:** tile_count + worker_count → threads start → relaxed atomic unique claims → disjoint pixel kernel + local stats → join all → deterministic integer merge → Image/stats return
- **ownership·state transition:** shared mutable scheduler state는 atomic counter 하나이고, shared Image writes는 주소가 disjoint합니다. worker-local stats만 각 thread가 변경합니다. merge 전까지 global stats를 동시 수정하지 않습니다.
- **failure/edge branch:** 이 초기 implementation은 thread 생성/호출자 예외의 join은 처리하지만 worker body 안에서 `traceRay`가 던진 예외는 thread entry 밖으로 빠져 `std::terminate`를 유발합니다. 이 gap은 `0536e4829070`에서 수정됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 정상 경로에서 tile 중복 없이 병렬 pixel 생성, race-free local stats, complete join 후 deterministic integer aggregation을 제공합니다.
- **이 SHA가 보장하지 않는 것:** worker exception을 caller에게 전달하지 못합니다. schedule 자체는 nondeterministic이지만 pixel 결과는 independent kernel과 tie rule에 의존합니다.
- **직접 확인/후속 evidence:** atomic memory order, disjoint tile writes, local stats array, join/merge와 RAII joiner를 해당 SHA에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `498266fc0abf`
- 다음 Thread commit: `18459bfda416`
- 이 commit이 다음 단계에 제공하는 것: `18459bfda416`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.3 `18459bfda416` — `feat(renderer): 작업자 수 설정과 자동 선택 추가`

- Importance: B
- Tags: CONCURRENCY, PERF
- Thread order: 3/7

#### Source에서 확정된 역할

- Development Thread role: Adds explicit/automatic worker-count policy.

#### B-level 구현 역할 복원

- **직전 관련 상태:** 병렬 renderer가 내부에서 worker 수를 고정하면 작은 image에서 과도한 thread를 만들고 benchmark가 머신마다 다른 worker 수를 사용해 비교하기 어렵습니다.
- **핵심 구현 결정:** `RenderSettings::threadCount`를 추가합니다. 0은 `hardware_concurrency()` 기반 자동 선택이며 API가 0을 반환하면 1로 fallback합니다. explicit/auto 모두 최소 1이고 tile count보다 크게 만들지 않습니다. benchmark는 reproducible comparison을 위해 explicit 1 worker를 지정합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/renderer.hpp — `threadCount` setting
 - src/renderer.cpp — auto/fallback/cap policy
 - benchmarks/render_benchmark.cpp — explicit one-worker setting
- **caller → callee / data flow:** settings.threadCount → explicit or hardware count → zero fallback → cap to tile count → worker launch
- **ownership·state transition:** thread count는 scheduling policy이고 pixel semantics에는 참여하지 않습니다. no-work/empty tile count에서 불필요한 worker를 만들지 않는 경계가 생깁니다.
- **failure/edge branch:** `hardware_concurrency()==0`을 그대로 사용하면 worker가 없어 image가 채워지지 않습니다. tile 수보다 많은 worker는 의미 없이 resource를 소모합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** portable auto policy와 deterministic benchmark용 explicit override를 제공합니다.
- **이 SHA가 보장하지 않는 것:** thread affinity, work stealing, adaptive tile size는 제공하지 않습니다.
- **직접 확인/후속 evidence:** 0 semantics, fallback 1, tile cap, benchmark explicit setting을 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `849f878ca0b0`
- 다음 Thread commit: `3619550fa354`
- 이 commit이 다음 단계에 제공하는 것: `3619550fa354`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.4 `3619550fa354` — `test(render): 작업자 수에 따른 함수 결과 동치 검증`

- Importance: A
- Tags: TEST, CONCURRENCY, DETERMINISM
- Thread order: 4/7

#### Source에서 확정된 역할

- Development Thread role: Verifies equal pixels/work 1 vs 4 workers in both accel modes.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** parallel design reasoning만으로 one/four workers가 exact same pixels와 work를 만든다고 보장할 수 없습니다. Linear/Bvh traversal mode도 함께 바뀌면 race나 counter 누락이 mode-specific하게 드러날 수 있습니다.
- **핵심 구현 결정:** `tests/render_tests.cpp`가 96×54 scene에 diffuse/metal/plane/cylinder를 넣고 Linear/Bvh 각각 threadCount 1과 4를 렌더합니다. pixel vector와 checksum을 exact 비교하고 primary/secondary/shadow/primitive/AABB counters도 같은 mode의 worker 수 사이에서 비교합니다. primary count가 `96*54`인지도 확인합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/render_tests.cpp — worker-count differential render tests
 - src/renderer.cpp — scheduler/pixel kernel/stats merge
 - src/scene.cpp — both acceleration modes
- **caller → callee / data flow:** same Scene/settings except threadCount → mode별 1-worker baseline → 4-worker result → bytes/checksum/work equality
- **ownership·state transition:** worker count만 독립 변수이고 Scene, mode, depth, resolution은 고정됩니다.
- **failure/edge branch:** tile overlap/missing tile, shared stats race, schedule-dependent floating accumulation이 있으면 pixels 또는 counters가 달라집니다.

#### Test commit 분석 기준

- **대상 production invariant:** 고정 scene/mode에서 thread count는 pixels/checksum/work counters를 바꾸지 않습니다.
- **test technique:** 1-vs-4 worker exact buffer/checksum/counter comparison across Linear/Bvh
- **통과하는 production path:** `renderScene` scheduler → full ray/scene/shading pipeline
- **이 test가 증명하는 것:** 대표 scene에서 disjoint-write와 local-stats design의 결과 동치를 증명합니다.
- **이 test가 증명하지 않는 것:** CLI bytes, worker failure, data-race sanitizer 결과를 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 선택된 mixed-material scene에서 worker schedule이 exact output과 semantic work를 바꾸지 않음을 고정합니다.
- **이 SHA가 보장하지 않는 것:** worker exception, CLI option parsing, every thread count/platform을 증명하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 성격: in-process deterministic differential integration. 소스를 검사했으며 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `18459bfda416`
- 다음 Thread commit: `ca2d108f2255`
- 이 commit이 다음 단계에 제공하는 것: `ca2d108f2255`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.5 `ca2d108f2255` — `test(render): 실행 모드별 PPM byte 결정성 검증`

- Importance: A
- Tags: TEST, DETERMINISM, OUTPUT
- Thread order: 5/7

#### Source에서 확정된 역할

- Development Thread role: Verifies exact PPM-byte equality through CLI.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** in-process Image equality가 같아도 CLI option routing이나 PPM serialization이 mode/worker setting에 따라 다른 artifact를 만들 수 있습니다.
- **핵심 구현 결정:** `tests/render_determinism.sh`가 같은 scene을 Linear/Bvh × 1/4 workers 네 조합으로 CLI 실행합니다. 각 checksum을 비교하고 `cmp -s`로 PPM bytes를 exact 비교합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/render_determinism.sh — four-mode CLI regression
 - src/main.cpp — accel/thread options
 - src/output.cpp — PPM serialization
- **caller → callee / data flow:** temporary scene/output paths → four CLI invocations → checksum capture/equality → PPM byte comparisons → cleanup
- **ownership·state transition:** process boundary, argument parsing, rendering, serialization 전체가 포함됩니다. output files는 temp area에 격리됩니다.
- **failure/edge branch:** CLI가 setting을 잘못 전달하거나 writer ordering이 달라지면 in-process test는 통과해도 byte comparison이 실패합니다.

#### Test commit 분석 기준

- **대상 production invariant:** acceleration mode와 worker count는 external PPM bytes/checksum을 바꾸지 않습니다.
- **test technique:** 네 CLI process 실행, checksum equality, `cmp -s` exact bytes
- **통과하는 production path:** `main` option parsing → render → output
- **이 test가 증명하는 것:** process boundary를 포함한 deterministic artifact equivalence를 증명합니다.
- **이 test가 증명하지 않는 것:** 실패 cleanup, sanitizer, 성능 향상을 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 테스트된 mode/worker 조합의 externally published PPM과 checksum이 동일합니다.
- **이 SHA가 보장하지 않는 것:** atomic writer failure나 worker exception을 주입하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 성격: end-to-end deterministic CLI integration. 스크립트를 검사했으며 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `3619550fa354`
- 다음 Thread commit: `0536e4829070`
- 이 commit이 다음 단계에 제공하는 것: `0536e4829070`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.6 `0536e4829070` — `fix(renderer): 작업자 예외를 호출자에게 전달`

- Importance: A
- Tags: CONCURRENCY, RISK, DEBUG
- Thread order: 6/7

#### Source에서 확정된 역할

- Development Thread role: Captures worker failures, stops new work, joins all workers, rethrows caller.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** 초기 worker lambda는 `traceRay`/shape code가 던진 예외를 catch하지 않습니다. C++ thread entry를 빠져나간 예외는 caller가 catch할 수 없고 `std::terminate`를 호출하므로 RAII joiner만으로는 정상 recovery가 불가능합니다.
- **핵심 구현 결정:** worker별 `std::exception_ptr` slot을 만들고 worker body 전체를 try/catch로 감쌉니다. catch는 `std::current_exception()`을 저장하고 `nextTile.store(tileCount)`로 새 work claim을 중단시킵니다. caller는 시작한 모든 thread를 먼저 join한 다음 error slots를 deterministic order로 검사해 첫 예외를 `std::rethrow_exception`합니다. stats merge와 successful Image 반환은 error scan 뒤에만 수행됩니다.

#### Failure → Fix 연결

- **기존 가정:** RAII joiner가 있으면 worker 내부 production exception도 caller가 처리할 수 있습니다.
- **실제 failure 또는 위험:** thread entry 밖으로 빠지는 예외는 caller로 전파되지 않고 `std::terminate`를 호출합니다.
- **root cause:** exception transport channel과 post-join rethrow 순서가 없었습니다.
- **수정된 decision/invariant:** worker-local `exception_ptr` capture, global new-work stop, complete join, caller rethrow를 추가합니다.
- **regression 연결:** `b5c708ac981a`의 throwing shape가 exact type/message recovery를 고정합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/renderer.cpp — worker error slots, stop signal, join, post-join rethrow
- **caller → callee / data flow:** worker production exception → catch/store → scheduler stop → other workers finish current tile/exit → caller joins all → error scan → original exception rethrow
- **ownership·state transition:** exception ownership은 worker stack에서 `exception_ptr` value로 이전됩니다. Image 일부가 이미 써졌을 수 있지만 caller에게 성공값으로 반환되지 않습니다. 모든 thread resource cleanup이 rethrow보다 앞섭니다.
- **failure/edge branch:** join 전에 rethrow하면 아직 실행 중인 thread와 joinable thread destructor 문제가 남습니다. catch 없이 thread boundary를 넘으면 process가 terminate됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** worker-body failure를 original dynamic type/message로 caller thread에 전달하고, 모든 started worker를 회수한 뒤 실패합니다.
- **이 SHA가 보장하지 않는 것:** 이미 계산된 partial bytes를 rollback하지는 않지만 실패 결과로 노출하지 않습니다. worker가 cooperative stop 전에 진행 중인 tile은 끝낼 수 있습니다.
- **직접 확인/후속 evidence:** catch/store/stop, join-all, post-join error scan/rethrow ordering을 해당 SHA에서 확인하고 throwing-shape test와 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `ca2d108f2255`
- 다음 Thread commit: `b5c708ac981a`
- 이 commit이 다음 단계에 제공하는 것: `b5c708ac981a`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.7 `b5c708ac981a` — `test(renderer): 작업자 실패 전파와 회수 검증`

- Importance: A
- Tags: TEST, CONCURRENCY, RISK
- Thread order: 7/7

#### Source에서 확정된 역할

- Development Thread role: Injects throwing shape to lock propagation/recovery.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** exception transport fix가 있어도 실제 `Shape::intersect` 호출 중 worker thread에서 던져지는 실패를 deterministic하게 재현하지 않으면 catch boundary가 빠져도 검출하기 어렵습니다.
- **핵심 구현 결정:** `tests/render_tests.cpp`에 unbounded `ThrowingShape`를 두고 `intersect`가 `std::runtime_error("worker exception sentinel")`를 던지게 합니다. 32×16 image와 4 workers로 render를 호출해 동일 exception type/message가 caller에서 관찰되는지 확인합니다. 호출이 test assertion까지 돌아온다는 사실은 process termination이 없고 renderer의 join/rethrow path가 완료됐음을 전제로 합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/render_tests.cpp — `ThrowingShape` and worker-failure regression
 - src/renderer.cpp — worker catch/stop/join/rethrow
 - src/scene.cpp — unbounded shape dispatch
- **caller → callee / data flow:** parallel render → worker tile → ray trace → Scene dispatch → `ThrowingShape::intersect` throws → renderer capture/stop/join → caller catches sentinel
- **ownership·state transition:** failure injection은 매번 같은 production acquisition/dispatch 지점에서 발생합니다. 외부 mutable flag나 timing에 의존하지 않습니다.
- **failure/edge branch:** fix 전에는 expected catch로 돌아오지 않고 process termination이 발생합니다. wrong message/type 또는 swallowed failure도 assertion이 잡습니다.

#### Test commit 분석 기준

- **대상 production invariant:** worker production failure는 process를 종료하거나 성공 image를 반환하지 않고 caller에 재전파됩니다.
- **test technique:** custom throwing Shape를 실제 scene dispatch에 삽입하고 exact exception type/message assertion
- **통과하는 production path:** parallel `renderScene` → `traceRay` → `Scene::intersect` → Shape virtual dispatch → worker recovery
- **이 test가 증명하는 것:** worker-body 예외가 caller에서 관찰되고 renderer가 control을 되돌립니다.
- **이 test가 증명하지 않는 것:** 모든 thread resource를 external leak detector로 측정하거나 partial pixels rollback을 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 실제 production worker path에서 deterministic exception propagation과 caller recovery를 고정합니다.
- **이 SHA가 보장하지 않는 것:** 모든 worker가 정확히 어느 tile에서 멈췄는지, partial pixel count, thread leak을 별도 counter로 직접 측정하지는 않습니다. join은 성공적인 caller return/rethrow 경로와 production code 검사로 확인합니다.
- **직접 확인/후속 evidence:** 테스트 성격: deterministic failure injection + concurrency lifecycle regression. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `0536e4829070`
- 다음 Thread commit: 이 Thread의 종료점
- 이 commit이 Thread 종료에 제공하는 것: Thread-level invariant ledger와 최종 실행 순서에서이 SHA의 결과를 최종 상태에 반영했습니다.

## 6. Invariant ledger

| Invariant | 최초 도입/기준 | 강화 또는 수정 | 부족함/위험 노출 | 고정한 test/evidence | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| 각 pixel은 정확히 한 tile에 속함 | 498266fc0abf | 849f878ca0b0 | edge tile 누락/overlap | 3619550fa354/ca2d108f2255 | ceil grid, clipped bounds, unique atomic claim |
| worker count/schedule은 bytes와 work를 바꾸지 않음 | 849f878ca0b0 | 18459bfda416 | shared writes/stats race | 3619550fa354/ca2d108f2255 | disjoint pixels, local stats, exact comparisons |
| started threads는 모두 join됨 | 849f878ca0b0 creation-path RAII | 0536e4829070 worker-body path | worker exception이 `std::terminate` | b5c708ac981a | capture→stop→join→rethrow |
| worker failure는 성공 image로 반환되지 않음 | 0536e4829070 | 0536e4829070 | partial bytes가 존재할 수 있음 | b5c708ac981a | post-join error scan precedes result return |

### Ledger 보완 기록

- 각 invariant는 위 표의 SHA에서 observable behavior 또는 state로 처음 나타났습니다.
- 후속 commit이 같은 용어를 사용하더라도 그 보장을 과거 SHA에 소급하지 않았습니다.
- test/evidence 열은 production path와 assertion 또는 deterministic work gate를 함께 가리킵니다.
- 실행하지 않은 test는 source-level evidence로만 기록했습니다.

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Decision/Fix | Test 또는 evidence | 실제 실패 처리와 assertion |
| --- | --- | --- | --- |
| tile edge 계산 오류/중복 claim | clipped fixed tiles + atomic fetch_add | 3619550fa354 | 1/4 worker exact pixels and primary count |
| schedule-dependent stats | worker-local counters + post-join integer merge | 3619550fa354 | mode별 counter equality |
| CLI setting 또는 serialization drift | same pixel kernel and output path | ca2d108f2255 | four-mode checksum/PPM `cmp -s` |
| worker production exception이 process terminate | `exception_ptr` capture, stop, join, rethrow | b5c708ac981a | throwing Shape sentinel recovered at caller |

### 연결 검토

- feature commit도 어떤 잘못된 state 또는 semantic drift를 막는지 production path에 연결했습니다.
- fix commit은 기존 가정 → 실제 위험 → root cause → corrected decision → regression 순서로 기록했습니다.
- test가 broad integration인지 deterministic boundary/differential/failure-injection regression인지 commit 기록에서 구분했습니다.
- assertion이 증명하지 않는 범위와 실행하지 못한 항목을 별도로 남겼습니다.

## 8. Ownership / state / responsibility 변화

`Image`는 caller-side renderer invocation이 소유하지만 각 tile의 byte range는 claim한 worker에게
일시적으로 독점됩니다. Scene, camera frame과 acceleration은 shared read-only입니다. atomic
counter는 tile ownership만 배정하고 pixel ordering을 동기화하지 않습니다. 각 worker가 자체
`RenderStats`와 `exception_ptr` slot을 소유합니다. caller는 thread objects의 owner이며 모든
started thread를 join한 뒤 stats/error ownership을 회수합니다. 실패 시 exception value만
caller로 전송되고 partial Image는 성공 결과로 반환되지 않습니다.

### 학습자 최종 기록

- **source state와 derived state:** `Image`는 caller-side renderer invocation이 소유하지만 각 tile의 byte range는 claim한 worker에게 일시적으로 독점됩니다. Scene, camera frame과 acceleration은 shared read-only입니다. atomic counter는 tile ownership만 배정하고 pixel ordering을 동기화하지 않습니다. 각 worker가 자체 `RenderStats`와 `exception_ptr` slot을 소유합니다. caller는 thread objects의 owner이며 모든 started thread를 join한 뒤 stats/error ownership을 회수합니다. 실패 시 exception value만 caller로 전송되고 partial Image는 성공 결과로 반환되지 않습니다.
- **mutation/transition boundary:** commit별 `ownership·state transition`과 위 invariant ledger에 표시했습니다.
- **failure 시 복구 상태:** Failure → Fix → Test 표와 각 fix/test section에 정상·오류 상태를 구분했습니다.

## 9. Thread 최종 상태

renderer는 fixed 16×16 tiles를 nondeterministic schedule로 처리하지만 tile ownership과 pixel
writes는 disjoint하고 각 pixel의 floating operation order는 serial baseline과 같습니다.
counters는 worker-local integer state라 join 후 merge order가 결과를 바꾸지 않습니다. one/four
workers와 Linear/Bvh의 in-process 및 CLI exact comparison이이 성질을 고정합니다. worker가
production path에서 실패하면 새 work를 중단하고 모든 worker를 join한 뒤 original exception을
caller에서 rethrow하므로 process termination이나 partial-success 반환을 막습니다.

### 직접 작성한 결론

- **Thread 시작과 종료의 behavior 차이:** renderer는 fixed 16×16 tiles를 nondeterministic schedule로 처리하지만 tile ownership과 pixel writes는 disjoint하고 각 pixel의 floating operation order는 serial baseline과 같습니다. counters는 worker-local integer state라 join 후 merge order가 결과를 바꾸지 않습니다. one/four workers와 Linear/Bvh의 in-process 및 CLI exact comparison이이 성질을 고정합니다. worker가 production path에서 실패하면 새 work를 중단하고 모든 worker를 join한 뒤 original exception을 caller에서 rethrow하므로 process termination이나 partial-success 반환을 막습니다.
- **아직 다른 Thread 또는 외부 검증이 보완해야 하는 항목:** ThreadSanitizer evidence, cooperative cancellation 중 진행 중 tile의 조기 중단, partial buffer rollback은 제공하지 않습니다.

## 10. 최종 architecture 또는 execution flow 정리

### Source가 확정한 흐름 anchor

```text
16×16 tile decomposition → atomic tile claim → worker-exclusive pixel region → worker-local `RenderStats` → join → integer merge → caller result or post-join rethrow
```

### 실제 코드로 완성한 흐름

1. renderer가 image를 16×16 tiles와 clipped edge tiles로 분해합니다.
2. settings에서 explicit/automatic worker 수를 계산하고 tile count로 cap합니다.
3. 각 worker가 relaxed atomic counter로 unique tile index를 claim합니다.
4. claim한 worker만 tile의 global RGB byte range를 씁니다.
5. Scene/camera는 읽기만 하고 worker-local stats에 semantic work를 기록합니다.
6. worker failure는 local `exception_ptr`에 저장하고 counter를 tileCount로 보내 새 claim을 막습니다.
7. caller는 모든 threads를 join합니다.
8. error가 있으면 merge/success return 전에 original exception을 rethrow합니다.
9. 정상 시 local integer stats를 합치고 complete Image를 반환합니다.
10. in-process와 CLI tests가 thread/mode 조합의 exact 결과를 비교합니다.

### 학습자의 최종 설명

renderer는 fixed 16×16 tiles를 nondeterministic schedule로 처리하지만 tile ownership과 pixel
writes는 disjoint하고 각 pixel의 floating operation order는 serial baseline과 같습니다.
counters는 worker-local integer state라 join 후 merge order가 결과를 바꾸지 않습니다. one/four
workers와 Linear/Bvh의 in-process 및 CLI exact comparison이이 성질을 고정합니다. worker가
production path에서 실패하면 새 work를 중단하고 모든 worker를 join한 뒤 original exception을
caller에서 rethrow하므로 process termination이나 partial-success 반환을 막습니다.

남은 경계는 다음과 같습니다. ThreadSanitizer evidence, cooperative cancellation 중 진행 중 tile의 조기 중단, partial buffer rollback은 제공하지 않습니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 source 순서대로 확인했습니다.
- [x] 각 commit의 SHA, subject, importance, tags를 그대로 유지했습니다.
- [x] 모든 핵심 설명에 해당 SHA의 file path와 symbol 근거를 기록했습니다.
- [x] final HEAD의 구조를 과거 SHA에 소급하지 않았습니다.
- [x] S/A/B importance에 맞춰 architecture, subsystem, localized role의 깊이를 구분했습니다.
- [x] source에서 확정하지 않은 실행 결과나 runtime 수치를 사실로 채우지 않았습니다.
- [x] failure와 fix/test를 실제 production path로 연결했습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] invariant ledger의 각 변화를 commit evidence와 연결했습니다.
- [ ] 해당 SHA checkout에서 테스트·benchmark·sanitizer를 직접 실행했습니다. 환경 제한 때문에 미실행 상태입니다.
- [x] 별도의 프로젝트 재학습 없이이 Thread의 설계 → 구현 → 위험 → 수정 → 검증 발전을 설명할 수 있는 기록을 남겼습니다.
===== END FILE: 05-deterministic-tiled-rendering.md =====

===== BEGIN FILE: 06-image-representation-and-atomic-output.md =====
# Thread 6. Image representation and atomic PPM publication

## 1. Thread 목표

image dimension/storage/index arithmetic에서 시작해 checksum definition, mutable public storage validation, checked stream serialization, same-directory temporary file와 final replacement까지 안전한 PPM publication contract를 복원합니다.

### Source significance

> This thread expands the output contract from “write bytes” to “publish only a complete, internally
> consistent image.” Allocation and indexing safety prevent buffer mismatch at construction; later
> validation protects callers that mutate public `Image` fields directly. Standardized checksums make
> deterministic regressions comparable, while temporary-file publication ensures validation, stream,
> flush, close, or replacement failures do not destroy a previously valid output.

### 이 Thread에 연결된 source invariant

- Valid image storage is positive and exactly sized without multiplication or index overflow.
- A final PPM path changes only after complete successful serialization, flush, close, and replacement. Any earlier failure preserves the existing destination and attempts to remove the temporary file.
- Golden checksums and exact PPM bytes remain stable unless an intentional rendering contract changes.

### 이 Thread에 연결된 engineering difficulty

- Publishing output atomically enough to avoid destroying a previous file when validation, serialization, flush, close, or replacement fails.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `width × height × 3` 계산은 어느 type domain에서 어떤 순서로 overflow를 검사하는가?
- allocation size와 PPM pixel offset이 동일한 representation invariant를 공유하는가?
- FNV-1a checksum은 dimensions와 quantized bytes를 어떤 순서로 반영하며 왜 두 수준의 golden이 필요한가?
- public image fields가 construction 이후 storage mismatch를 만들 수 있었던 gap은 무엇인가?
- validation은 checksum과 writer에서 언제 실행되며 기존 파일 truncation보다 왜 앞서야 하는가?
- transactional writer의 commit point는 어디이며 validation/open/write/flush/close/replace failure별 destination과 temp 상태는 무엇인가?

## 3. 완료 기준

- [x] checked storage-size 함수와 모든 index operand conversion을 실제 코드에서 확인했습니다.
- [x] valid/invalid `Image` 상태를 dimensions와 pixel vector length로 판정할 수 있습니다.
- [x] 초기 checksum constant와 standard FNV-1a fix를 해당 SHA별로 구분했습니다.
- [x] small-image golden과 full-render golden이 각각 checksum encoding과 pipeline semantics를 어떻게 고정하는지 설명할 수 있습니다.
- [x] path writer의 temp 생성부터 replacement commit까지 정상·실패 cleanup 흐름을 기록했습니다.
- [x] 기존 destination 보존이 invalid representation, stream failure, replacement failure에서 각각 어떻게 검증되는지 연결했습니다.
- [x] 모든 참조 SHA가 `cpp/miniRT` branch HEAD의 ancestry에 속하는지 확인했습니다.
- [ ] 해당 SHA checkout에서 build/test/benchmark 명령을 직접 실행했습니다. 로컬 외부 네트워크와 checkout이 제공되지 않아 실행 evidence는 만들지 않았습니다.

### 검증 범위

- 지정 branch HEAD: `7d08c7c13fa68c3e60eea3c7014658b0a133e6f0`
- 각 참조 SHA는 Thread 내부의 연속 compare chain에서 `behind_by = 0`, merge base가 선행 SHA였고, Thread 종료 SHA도 branch HEAD의 조상으로 확인했습니다.
- 구현 설명은 해당 commit의 diff/file content를 기준으로 작성했으며, final HEAD의 후속 API를 과거 SHA에 소급하지 않았습니다.
- 테스트와 benchmark는 source mechanism과 production path만 검사했습니다. 실행 결과, sanitizer 결과, wall-clock 수치는 기록하지 않았습니다.

## 4. Commit map

1. `71096cd311d5` — `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지`
 - Importance: A
 - Tags: OUTPUT, RISK, EDGE
 - Source-defined role: Makes allocation sizing/pixel offsets overflow-aware.

2. `3d2e6a5becb7` — `test(image): 잘못된 차원과 저장 크기 계산 검증`
 - Importance: B
 - Tags: TEST, OUTPUT
 - Source-defined role: Verifies positive dimensions/exact storage.

3. `89c3c7269877` — `fix(output): 표준 FNV-1a 기준값 적용`
 - Importance: B
 - Tags: DEBUG, OUTPUT
 - Source-defined role: Corrects the FNV-1a definition.

4. `eac2ecd13c33` — `test(output): PPM과 렌더링 체크섬 기준 고정`
 - Importance: A
 - Tags: TEST, DETERMINISM, OUTPUT
 - Source-defined role: Pins checksum/full-render goldens.

5. `4eb50073bc3e` — `fix(output): 불일치한 이미지 저장소 거부`
 - Importance: A
 - Tags: OUTPUT, RISK, EDGE
 - Source-defined role: Validates image dimensions/pixel storage agree.

6. `918dd1efeaf3` — `test(output): 잘못된 이미지 저장소 처리 검증`
 - Importance: B
 - Tags: TEST, OUTPUT, RISK
 - Source-defined role: Exercises short/oversized storage and existing destination preservation.

7. `053235a7a5e1` — `fix(output): PPM 출력 실패 시 기존 파일 보존`
 - Importance: A
 - Tags: OUTPUT, RISK, PRACTICAL
 - Source-defined role: Writes checked stream and publishes temp+final replacement.

8. `c6a6a7562a4d` — `test(output): 출력 실패의 대상 보존과 정리 검증`
 - Importance: A
 - Tags: TEST, OUTPUT, RISK
 - Source-defined role: Injects serialization/replacement failures, verifies cleanup/preservation.

## 5. Commit별 학습 기록

### 5.1 `71096cd311d5` — `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지`

- Importance: A
- Tags: OUTPUT, RISK, EDGE
- Thread order: 1/8

#### Source에서 확정된 역할

- Development Thread role: Makes allocation sizing/pixel offsets overflow-aware.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** `Image` construction의 `width * height * 3`과 PPM offset의 `(y * width + x) * 3`이 signed `int` domain에서 먼저 계산되면, 이후 `size_t`로 변환해도 이미 overflow한 값입니다. 잘못된 allocation 또는 out-of-bounds indexing으로 이어질 수 있습니다.
- **핵심 구현 결정:** `pixelStorageSize`가 width/height 양수를 먼저 검사하고 operands를 `size_t`로 변환한 뒤 단계별 division check를 합니다. `w > max / h`, `w*h > max / 3`이면 `overflow_error`, non-positive dimensions면 `invalid_argument`입니다. Image constructor는이 helper 결과로 vector를 만듭니다. PPM/checksum indexing도 각 operand를 먼저 `size_t`로 올려 multiplication이 unsigned storage domain에서 일어나게 합니다.

#### Failure → Fix 연결

- **기존 가정:** int dimensions를 곱한 뒤 size_t로 바꿔도 allocation/index가 안전합니다.
- **실제 failure 또는 위험:** signed intermediate가 overflow하거나 wrap된 작은 size가 storage와 logical image를 분리합니다.
- **root cause:** range check 이전에 multiplication이 더 좁은 domain에서 실행됐습니다.
- **수정된 decision/invariant:** operands를 먼저 size_t로 변환하고 division-based precondition checks를 수행합니다.
- **regression 연결:** `3d2e6a5becb7`이 positive dimensions와 exact storage size를 고정합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/renderer.hpp — Image declaration/storage helper exposure if present
 - src/renderer.cpp — `pixelStorageSize`, `Image` constructor
 - src/output.cpp — size-safe pixel offset calculation
- **caller → callee / data flow:** signed dimensions → positivity validation → size_t conversion → checked pixel count → checked RGB byte count → vector allocation; pixel `(x,y)` → pre-converted size_t offset
- **ownership·state transition:** dimension values와 pixel vector size가 construction 시점에 일치합니다. 예외가 발생하면 vector allocation/partial Image 성공값이 없습니다.
- **failure/edge branch:** overflow 후 비교하는 검사는 너무 늦습니다. signed multiplication은 undefined behavior가 될 수 있고 작은 vector와 큰 logical dimensions의 불일치를 만듭니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 지원 platform의 `size_t` 범위 안에서 positive dimensions의 exact RGB byte count와 index arithmetic을 계산합니다.
- **이 SHA가 보장하지 않는 것:** public `Image` fields를 caller가 construction 후 변경하면 representation을 다시 깨뜨릴 수 있으며 `4eb50073bc3e`에서 사용 전 validation이 추가됩니다. `int` dimension 범위 때문에 64-bit `size_t`에서는 실제 overflow branch에 도달하기 어려울 수 있습니다.
- **직접 확인/후속 evidence:** multiplication 전 checks와 output operand conversion을 해당 SHA에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: 이 Thread의 시작점
- 다음 Thread commit: `3d2e6a5becb7`
- 이 commit이 다음 단계에 제공하는 것: `3d2e6a5becb7`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.2 `3d2e6a5becb7` — `test(image): 잘못된 차원과 저장 크기 계산 검증`

- Importance: B
- Tags: TEST, OUTPUT
- Thread order: 2/8

#### Source에서 확정된 역할

- Development Thread role: Verifies positive dimensions/exact storage.

#### B-level 구현 역할 복원

- **직전 관련 상태:** checked helper가 있어도 zero/negative dimensions가 거부되고 정상 dimensions가 정확한 RGB byte vector를 만드는지 자동 검증이 없습니다.
- **핵심 구현 결정:** `tests/core_tests.cpp`가 `Image(2,3)`의 storage length가 18인지 확인하고 zero/negative width 또는 height가 `invalid_argument`를 던지는지 검사합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/core_tests.cpp — Image dimension/storage tests
 - src/renderer.cpp — `pixelStorageSize`, Image constructor
- **caller → callee / data flow:** valid/invalid dimension construction → helper validation/arithmetic → vector length 또는 expected exception
- **ownership·state transition:** fixture는 constructor outcome만 관찰합니다. valid Image는 dimensions와 pixel size가 일치합니다.
- **failure/edge branch:** positivity check 제거 또는 channel multiplier 오류가 assertion에서 드러납니다.

#### Test commit 분석 기준

- **대상 production invariant:** Image dimensions는 양수이며 storage는 정확히 width×height×3 bytes입니다.
- **test technique:** small exact-size assertion + invalid-dimension exception assertions
- **통과하는 production path:** Image constructor → `pixelStorageSize` → vector allocation
- **이 test가 증명하는 것:** 대표 정상/비정상 dimension contract를 보호합니다.
- **이 test가 증명하지 않는 것:** 실제 size_t overflow branch와 post-construction mutation을 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 대표 valid size와 non-positive boundary를 고정합니다.
- **이 SHA가 보장하지 않는 것:** `int` dimensions와 64-bit size_t 조합에서 실제 multiplication-overflow exception을 강제로 재현하지는 않습니다. construction 이후 public mutation도이 테스트 범위 밖입니다.
- **직접 확인/후속 evidence:** 테스트 성격: constructor boundary regression. 소스를 검사했으며 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `71096cd311d5`
- 다음 Thread commit: `89c3c7269877`
- 이 commit이 다음 단계에 제공하는 것: `89c3c7269877`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.3 `89c3c7269877` — `fix(output): 표준 FNV-1a 기준값 적용`

- Importance: B
- Tags: DEBUG, OUTPUT
- Thread order: 3/8

#### Source에서 확정된 역할

- Development Thread role: Corrects the FNV-1a definition.

#### B-level 구현 역할 복원

- **직전 관련 상태:** 초기 checksum은 FNV-1a prime을 사용했지만 64-bit offset basis를 `1469598103934665603`으로 두어 표준값의 마지막 digit 7이 빠져 있었습니다. deterministic하더라도 표준 FNV-1a로 식별할 수 없는 값입니다.
- **핵심 구현 결정:** `src/output.cpp`의 initial hash를 표준 64-bit FNV-1a offset basis `14695981039346656037ULL`로 수정합니다. byte xor 뒤 prime multiplication 순서와 dimensions/pixels 입력 순서는 유지합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - src/output.cpp — checksum initial constant and update loop
- **caller → callee / data flow:** standard offset basis → dimension bytes → pixel bytes 각각 xor/multiply → fixed-width hex
- **ownership·state transition:** Image representation은 바뀌지 않고 derived checksum 값만 의도적으로 변경됩니다.
- **failure/edge branch:** 기존 golden은 모두 새 정의와 불일치하므로 후속 commit이 새 기준을 명시적으로 고정해야 합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** checksum 구현이 표준 FNV-1a 정의와 일치합니다.
- **이 SHA가 보장하지 않는 것:** FNV-1a는 암호학적 collision resistance를 제공하지 않으며 artifact identity의 lightweight regression surface입니다.
- **직접 확인/후속 evidence:** 한 자리 차이와 unchanged prime/update order를 해당 SHA에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `3d2e6a5becb7`
- 다음 Thread commit: `eac2ecd13c33`
- 이 commit이 다음 단계에 제공하는 것: `eac2ecd13c33`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.4 `eac2ecd13c33` — `test(output): PPM과 렌더링 체크섬 기준 고정`

- Importance: A
- Tags: TEST, DETERMINISM, OUTPUT
- Thread order: 4/8

#### Source에서 확정된 역할

- Development Thread role: Pins checksum/full-render goldens.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** checksum 정의가 수정됐지만 exact expected 값을 source에 고정하지 않으면 dimensions/byte ordering이나 upstream rendering drift가 조용히 지나갈 수 있습니다.
- **핵심 구현 결정:** `tests/core_tests.cpp`에 작은 hand-built image checksum `0fde7b4d509f1daf`를 넣어 local encoding을 고정하고, basic scene full render checksum `456dc8d87ebf194f`를 고정해 parser/camera/intersection/shading/quantization까지 포함한 pipeline baseline을 만듭니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/core_tests.cpp — small checksum golden and full-render golden
 - src/output.cpp — checksum production path
 - src/renderer.cpp and upstream pipeline — full-render input
- **caller → callee / data flow:** small explicit bytes → checksum exact value; parsed/basic Scene → full render bytes → same checksum function → full golden
- **ownership·state transition:** 두 golden은 같은 checksum function을 쓰지만 upstream 범위가 다릅니다. small case는 encoding order, full case는 renderer semantics까지 포함합니다.
- **failure/edge branch:** checksum constant/update 순서 변경은 둘 다 깨지고, rendering-only drift는 full golden만 깨질 수 있어 fault localization이 가능합니다.

#### Test commit 분석 기준

- **대상 production invariant:** dimension/pixel byte order와 basic full-render result가 의도 없이 바뀌지 않습니다.
- **test technique:** two-level exact checksum goldens
- **통과하는 production path:** Image checksum directly; full parser/render/image/checksum pipeline
- **이 test가 증명하는 것:** local encoding과 broad pipeline drift를 서로 다른 수준에서 탐지합니다.
- **이 test가 증명하지 않는 것:** collision-free equality, PPM text bytes, output failure cleanup을 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 표준ized checksum byte contract와 당시 complete rendering baseline을 함께 고정합니다.
- **이 SHA가 보장하지 않는 것:** hash collision 가능성 때문에 exact pixel comparison을 완전히 대신하지 않으며 intentional rendering change 시 golden update가 필요합니다.
- **직접 확인/후속 evidence:** 테스트 성격: deterministic local golden + broad integration golden. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `89c3c7269877`
- 다음 Thread commit: `4eb50073bc3e`
- 이 commit이 다음 단계에 제공하는 것: `4eb50073bc3e`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.5 `4eb50073bc3e` — `fix(output): 불일치한 이미지 저장소 거부`

- Importance: A
- Tags: OUTPUT, RISK, EDGE
- Thread order: 5/8

#### Source에서 확정된 역할

- Development Thread role: Validates image dimensions/pixel storage agree.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** constructor는 valid storage를 만들지만 `Image`의 public width/height/pixels를 caller가 수정할 수 있어 checksum/writer가 out-of-bounds read를 하거나 malformed file을 만들 수 있습니다.
- **핵심 구현 결정:** `Image::validate`가 positive dimensions와 checked expected storage size를 다시 계산하고 `pixels.size()`가 정확히 같지 않으면 예외를 던집니다. checksum과 PPM writer는 indexing, stream open/truncation보다 먼저 validation을 호출합니다.

#### Failure → Fix 연결

- **기존 가정:** Image constructor가 storage invariant를 영구히 보장합니다.
- **실제 failure 또는 위험:** public fields가 construction 후 short/oversized representation을 만들 수 있습니다.
- **root cause:** mutable aggregate를 소비할 때 invariant를 재검증하지 않았습니다.
- **수정된 decision/invariant:** checksum/output entry에서 exact size validation을 side effect 전에 수행합니다.
- **regression 연결:** `918dd1efeaf3`이 short/oversized 상태와 destination preservation을 검증합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/renderer.hpp — `Image::validate`
 - src/renderer.cpp — representation validation
 - src/output.cpp — checksum/writer precondition call
- **caller → callee / data flow:** possibly externally mutated Image → validate dimensions/expected bytes/exact vector length → only then checksum indexing or output operation
- **ownership·state transition:** public mutable representation은 유지되지만 모든 public consumer 앞에서 invariant를 재확립합니다. validation은 object를 고치지 않고 fail closed합니다.
- **failure/edge branch:** storage가 짧으면 out-of-bounds read, 길면 ignored trailing data/ambiguous representation이 됩니다. writer가 파일을 먼저 열면 invalid input이 기존 destination을 파괴할 수 있습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** checksum과 output이 정확히 sized Image만 소비하며 invalid input에서 destination side effect 전에 실패할 수 있습니다.
- **이 SHA가 보장하지 않는 것:** 객체 자체를 immutable하게 바꾸지는 않고 consumer discipline에 의존합니다. stream/replace failure transactional behavior는 다음 fix입니다.
- **직접 확인/후속 evidence:** validation call order가 path open보다 앞서는 것을 해당 SHA에서 확인했습니다.

#### Thread 내 연결

- 이전 Thread commit: `eac2ecd13c33`
- 다음 Thread commit: `918dd1efeaf3`
- 이 commit이 다음 단계에 제공하는 것: `918dd1efeaf3`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.6 `918dd1efeaf3` — `test(output): 잘못된 이미지 저장소 처리 검증`

- Importance: B
- Tags: TEST, OUTPUT, RISK
- Thread order: 6/8

#### Source에서 확정된 역할

- Development Thread role: Exercises short/oversized storage and existing destination preservation.

#### B-level 구현 역할 복원

- **직전 관련 상태:** representation validation이 추가됐지만 short/oversized 양쪽과 writer의 pre-truncation ordering을 고정하는 test가 없습니다.
- **핵심 구현 결정:** `tests/core_tests.cpp`가 valid Image에서 byte 하나를 `pop_back`해 checksum과 writer가 거부하는지 확인합니다. writer destination에는 먼저 `preserve me`를 써 두고 invalid write 뒤 그대로인지 확인합니다. 이후 byte 둘을 push해 expected보다 1 큰 storage도 validation error인지 검사합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/core_tests.cpp — malformed Image regression and existing destination assertion
 - src/renderer.cpp — `Image::validate`
 - src/output.cpp — checksum/path writer
- **caller → callee / data flow:** valid Image → short mutation → checksum reject → existing file seed → path write reject before open/truncate → content preserved → oversized mutation → reject
- **ownership·state transition:** failure injection은 public vector mutation으로 deterministic합니다. destination 내용은 observable external state입니다.
- **failure/edge branch:** validation이 after-open으로 이동하면 exception은 나더라도 `preserve me`가 사라져 preservation assertion이 실패합니다.

#### Test commit 분석 기준

- **대상 production invariant:** invalid Image는 checksum/output에 사용되지 않고 기존 destination을 변경하지 않습니다.
- **test technique:** pixel vector short/oversized mutation, exception assertion, seeded destination content comparison
- **통과하는 production path:** `Image::validate` → checksum/path writer precondition
- **이 test가 증명하는 것:** 두 mismatch 방향과 validation-before-truncation을 고정합니다.
- **이 test가 증명하지 않는 것:** valid serialization 중 I/O failure와 temp cleanup을 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** short와 oversized representation을 모두 거부하고 invalid input이 기존 destination을 건드리지 않음을 고정합니다.
- **이 SHA가 보장하지 않는 것:** valid image의 serialization 도중 stream/flush/replace failure는이 테스트가 주입하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 성격: deterministic representation failure injection + pre-side-effect preservation regression. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `4eb50073bc3e`
- 다음 Thread commit: `053235a7a5e1`
- 이 commit이 다음 단계에 제공하는 것: `053235a7a5e1`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.7 `053235a7a5e1` — `fix(output): PPM 출력 실패 시 기존 파일 보존`

- Importance: A
- Tags: OUTPUT, RISK, PRACTICAL
- Thread order: 7/8

#### Source에서 확정된 역할

- Development Thread role: Writes checked stream and publishes temp+final replacement.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** valid Image를 직접 final path에 쓰면 serialization, flush, close 중간 실패가 기존 파일을 부분 PPM으로 바꿉니다. validation-only preservation으로는 실제 I/O failure를 다루지 못합니다.
- **핵심 구현 결정:** stream overload가 Image를 validate하고 P3 header/pixels를 쓴 뒤 stream state를 검사합니다. path overload는 target과 같은 directory에 `target + ".tmp." + steady-clock stamp + atomic sequence` 형태의 임시 파일을 만들고 RAII `TemporaryOutput`이 commit 전까지 삭제 책임을 가집니다. serialize → flush → close가 모두 성공한 뒤 POSIX `rename` 또는 Windows replacement API로 final path를 교체하고 마지막에 temp guard를 committed로 표시합니다.

#### Failure → Fix 연결

- **기존 가정:** valid Image를 final path에 직접 쓰면 output failure도 예외로 충분히 처리됩니다.
- **실제 failure 또는 위험:** 예외가 나기 전에 기존 destination이 truncate되거나 partial PPM으로 바뀝니다.
- **root cause:** serialization work와 externally visible publication이 같은 파일에서 동시에 진행됐습니다.
- **수정된 decision/invariant:** same-directory temporary candidate와 final replacement를 분리하고 RAII cleanup/commit을 둡니다.
- **regression 연결:** `c6a6a7562a4d`가 stream 및 replacement failure를 deterministic하게 주입합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - include/ray/output.hpp — checked stream/path overloads
 - src/output.cpp — stream validation, `TemporaryOutput`, temp naming, flush/close, platform replacement
- **caller → callee / data flow:** Image validate → same-directory temp open → complete P3 write → stream check → flush check → close check → atomic-style replacement → temp guard commit; any throw before commit → guard removes temp
- **ownership·state transition:** 기존 destination은 final replacement까지 authoritative합니다. temp file이 new candidate를 소유하고, replacement 성공이 publication commit point입니다. guard destructor가 uncommitted temp cleanup을 맡습니다.
- **failure/edge branch:** validation/open/serialization/flush/close/replacement 어느 단계든 예외가 나면 final path를 직접 수정하지 않습니다. replacement가 실패하면 temp를 제거하고 기존 target을 유지합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** API-level 정상/오류 반환 관점에서 incomplete PPM을 final path로 publish하지 않고 기존 file을 보존합니다.
- **이 SHA가 보장하지 않는 것:** directory fsync나 power-loss crash consistency까지 보장하지 않습니다. POSIX/Windows replacement semantics 차이가 있으며 동일 filesystem을 위해 same-directory temp를 사용합니다.
- **직접 확인/후속 evidence:** temp lifetime, stream checks, replacement-before-commit과 destructor cleanup 순서를 해당 SHA에서 확인하고 failure-injection tests와 연결했습니다.

#### Thread 내 연결

- 이전 Thread commit: `918dd1efeaf3`
- 다음 Thread commit: `c6a6a7562a4d`
- 이 commit이 다음 단계에 제공하는 것: `c6a6a7562a4d`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.8 `c6a6a7562a4d` — `test(output): 출력 실패의 대상 보존과 정리 검증`

- Importance: A
- Tags: TEST, OUTPUT, RISK
- Thread order: 8/8

#### Source에서 확정된 역할

- Development Thread role: Injects serialization/replacement failures, verifies cleanup/preservation.

#### A-level subsystem와 decision 복원

- **직전 관련 상태:** transactional writer가 있어도 실제 stream failure와 final replacement failure에서 destination/temp 상태를 관찰하지 않으면 cleanup ordering regression을 잡기 어렵습니다.
- **핵심 구현 결정:** `tests/output_tests.cpp`가 custom failing stream buffer로 serialization 중 failure를 주입합니다. 임시 directory RAII 아래에서 정상 atomic replacement가 exact expected output을 만들고 temp leftovers가 없는지 확인합니다. replacement failure는 destination path를 directory로 만들고 sentinel을 두어 replace가 실패하도록 한 뒤 directory/sentinel이 보존되고 `.tmp.` 파일이 남지 않는지 검사합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - tests/output_tests.cpp — failing buffer, temp directory fixture, replacement failure/preservation assertions
 - src/output.cpp — stream writer and transactional path writer
- **caller → callee / data flow:** failing stream buffer → checked stream write throws; seeded destination/directory → temp candidate complete → replacement fails → guard cleanup → destination/sentinel/temp-directory scan assertions
- **ownership·state transition:** destination, sentinel, temp directory listing이 failure 전후 external state입니다. failure injection은 timing이나 disk-full에 의존하지 않습니다.
- **failure/edge branch:** stream state를 검사하지 않으면 short output을 성공으로 볼 수 있고, replacement failure 뒤 guard commit/cleanup 순서가 틀리면 temp leak 또는 destination 손상이 나타납니다.

#### Test commit 분석 기준

- **대상 production invariant:** final destination은 complete success에서만 변경되고 실패 시 기존 state와 temp cleanliness가 유지됩니다.
- **test technique:** custom failing streambuf, directory-as-destination replacement failure, sentinel and directory enumeration
- **통과하는 production path:** checked stream serializer and path-level temp/replace/RAII cleanup
- **이 test가 증명하는 것:** 두 주요 failure phase와 정상 publication의 state outcome을 고정합니다.
- **이 test가 증명하지 않는 것:** OS crash durability, every errno, cross-filesystem replacement을 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** serialization failure 감지, successful replacement, replacement failure 시 기존 target 보존, uncommitted temp cleanup을 deterministic tests로 고정합니다.
- **이 SHA가 보장하지 않는 것:** 실제 ENOSPC, permission change race, process crash/power loss와 directory fsync durability를 증명하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 성격: deterministic I/O failure injection + transactional state regression. 실행은 하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `053235a7a5e1`
- 다음 Thread commit: 이 Thread의 종료점
- 이 commit이 Thread 종료에 제공하는 것: Thread-level invariant ledger와 최종 실행 순서에서이 SHA의 결과를 최종 상태에 반영했습니다.

## 6. Invariant ledger

| Invariant | 최초 도입/기준 | 강화 또는 수정 | 부족함/위험 노출 | 고정한 test/evidence | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| positive exact-size RGB storage | 71096cd311d5 | 4eb50073bc3e에서 consumer 재검증 | public mutation으로 size mismatch | 3d2e6a5becb7/918dd1efeaf3 | checked multiplication + `Image::validate` |
| 표준 FNV-1a checksum | 1bc7cacd30aa 초기 비표준 상수 | 89c3c7269877 | offset basis digit 누락 | eac2ecd13c33 | standard basis + local/full goldens |
| invalid Image는 destination side effect 전 거부 | 4eb50073bc3e | 4eb50073bc3e | writer가 open/truncate 먼저 할 위험 | 918dd1efeaf3 | validate before checksum/open |
| final path는 complete write 후에만 변경 | 053235a7a5e1 | 053235a7a5e1 | direct write가 기존 파일 partial/truncate | c6a6a7562a4d | temp serialize/flush/close → replace → commit |

### Ledger 보완 기록

- 각 invariant는 위 표의 SHA에서 observable behavior 또는 state로 처음 나타났습니다.
- 후속 commit이 같은 용어를 사용하더라도 그 보장을 과거 SHA에 소급하지 않았습니다.
- test/evidence 열은 production path와 assertion 또는 deterministic work gate를 함께 가리킵니다.
- 실행하지 않은 test는 source-level evidence로만 기록했습니다.

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Decision/Fix | Test 또는 evidence | 실제 실패 처리와 assertion |
| --- | --- | --- | --- |
| dimension/storage multiplication 또는 index overflow | size_t pre-conversion + division checks | 3d2e6a5becb7 | valid size/non-positive boundary assertions |
| public pixels short/oversized | `Image::validate` exact equality | 918dd1efeaf3 | checksum/write reject and existing content preservation |
| 비표준 checksum definition | correct FNV-1a offset basis | eac2ecd13c33 | small/full exact goldens |
| serialization/flush/close 중 기존 파일 손상 | same-directory temp and checked stream | c6a6a7562a4d | failing stream + no publish |
| final replacement failure/temp leak | replacement as commit point + RAII temp cleanup | c6a6a7562a4d | directory target/sentinel preservation and leftover scan |

### 연결 검토

- feature commit도 어떤 잘못된 state 또는 semantic drift를 막는지 production path에 연결했습니다.
- fix commit은 기존 가정 → 실제 위험 → root cause → corrected decision → regression 순서로 기록했습니다.
- test가 broad integration인지 deterministic boundary/differential/failure-injection regression인지 commit 기록에서 구분했습니다.
- assertion이 증명하지 않는 범위와 실행하지 못한 항목을 별도로 남겼습니다.

## 8. Ownership / state / responsibility 변화

construction 시 `Image`가 dimensions와 exact RGB vector를 소유하지만 public mutation 가능성 때문에
각 consumer가 representation을 재검증합니다. checksum은 Image를 읽기만 합니다. stream writer는
caller stream을 소유하지 않고 상태만 검사합니다. path writer는 `TemporaryOutput` guard를 통해
same-directory candidate file의 cleanup 책임을 소유하고, 기존 destination은 replacement 성공 전까지
authoritative합니다. replacement가 성공한 뒤 guard를 commit하면 temp cleanup 책임이 해제됩니다.
예외 경로에서는 stack unwinding이 uncommitted temp를 제거합니다.

### 학습자 최종 기록

- **source state와 derived state:** construction 시 `Image`가 dimensions와 exact RGB vector를 소유하지만 public mutation 가능성 때문에 각 consumer가 representation을 재검증합니다. checksum은 Image를 읽기만 합니다. stream writer는 caller stream을 소유하지 않고 상태만 검사합니다. path writer는 `TemporaryOutput` guard를 통해 same-directory candidate file의 cleanup 책임을 소유하고, 기존 destination은 replacement 성공 전까지 authoritative합니다. replacement가 성공한 뒤 guard를 commit하면 temp cleanup 책임이 해제됩니다. 예외 경로에서는 stack unwinding이 uncommitted temp를 제거합니다.
- **mutation/transition boundary:** commit별 `ownership·state transition`과 위 invariant ledger에 표시했습니다.
- **failure 시 복구 상태:** Failure → Fix → Test 표와 각 fix/test section에 정상·오류 상태를 구분했습니다.

## 9. Thread 최종 상태

Image storage와 offset은 multiplication 전에 checked `size_t` domain에서 계산되고, public mutation으로
생긴 short/oversized state는 checksum/output side effect 전에 거부됩니다. checksum은 표준 64-bit
FNV-1a로 정의되고 local/full goldens이 byte order와 pipeline을 고정합니다. valid PPM은 final path가
아닌 same-directory temp에 완전히 serialize·flush·close된 뒤 replacement됩니다. validation, stream,
close, replace failure에서는 기존 destination이 유지되고 temp guard가 candidate를 제거합니다.
power-loss durability와 directory fsync는이 API-level contract 밖입니다.

### 직접 작성한 결론

- **Thread 시작과 종료의 behavior 차이:** Image storage와 offset은 multiplication 전에 checked `size_t` domain에서 계산되고, public mutation으로 생긴 short/oversized state는 checksum/output side effect 전에 거부됩니다. checksum은 표준 64-bit FNV-1a로 정의되고 local/full goldens이 byte order와 pipeline을 고정합니다. valid PPM은 final path가 아닌 same-directory temp에 완전히 serialize·flush·close된 뒤 replacement됩니다. validation, stream, close, replace failure에서는 기존 destination이 유지되고 temp guard가 candidate를 제거합니다. power-loss durability와 directory fsync는이 API-level contract 밖입니다.
- **아직 다른 Thread 또는 외부 검증이 보완해야 하는 항목:** process crash·power loss에 대한 durable atomicity, directory fsync, 모든 filesystem/Windows edge는 별도 시스템 수준 검증이 필요합니다.

## 10. 최종 architecture 또는 execution flow 정리

### Source가 확정한 흐름 anchor

```text
dimensions → checked `pixelStorageSize` → `Image::pixels`/safe offset → `Image::validate` → checksum or checked P3 serialization → same-directory temporary file → flush/close → final replacement → temporary-file commit
```

### 실제 코드로 완성한 흐름

1. Image constructor가 positive dimensions를 검사하고 RGB storage size를 checked 계산합니다.
2. pixel access/serialization은 operands를 size_t로 올린 뒤 offset을 계산합니다.
3. checksum 또는 writer entry가 `Image::validate`로 exact storage equality를 확인합니다.
4. checksum은 standard FNV-1a basis에서 dimension bytes와 pixel bytes를 순서대로 반영합니다.
5. path writer가 target과 같은 directory에 unique temp file과 RAII guard를 만듭니다.
6. checked stream serializer가 P3 전체를 쓰고 stream state를 검사합니다.
7. flush와 close가 성공해야 replacement 단계로 이동합니다.
8. OS replacement 성공이 externally visible commit point입니다.
9. commit 전 예외에서는 guard가 temp를 제거하고 기존 destination을 보존합니다.
10. failure-injection tests가 invalid representation, stream failure, replacement failure state를 검사합니다.

### 학습자의 최종 설명

Image storage와 offset은 multiplication 전에 checked `size_t` domain에서 계산되고, public mutation으로
생긴 short/oversized state는 checksum/output side effect 전에 거부됩니다. checksum은 표준 64-bit
FNV-1a로 정의되고 local/full goldens이 byte order와 pipeline을 고정합니다. valid PPM은 final path가
아닌 same-directory temp에 완전히 serialize·flush·close된 뒤 replacement됩니다. validation, stream,
close, replace failure에서는 기존 destination이 유지되고 temp guard가 candidate를 제거합니다.
power-loss durability와 directory fsync는이 API-level contract 밖입니다.

남은 경계는 다음과 같습니다. process crash·power loss에 대한 durable atomicity, directory fsync, 모든 filesystem/Windows edge는 별도 시스템 수준 검증이 필요합니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 source 순서대로 확인했습니다.
- [x] 각 commit의 SHA, subject, importance, tags를 그대로 유지했습니다.
- [x] 모든 핵심 설명에 해당 SHA의 file path와 symbol 근거를 기록했습니다.
- [x] final HEAD의 구조를 과거 SHA에 소급하지 않았습니다.
- [x] S/A/B importance에 맞춰 architecture, subsystem, localized role의 깊이를 구분했습니다.
- [x] source에서 확정하지 않은 실행 결과나 runtime 수치를 사실로 채우지 않았습니다.
- [x] failure와 fix/test를 실제 production path로 연결했습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] invariant ledger의 각 변화를 commit evidence와 연결했습니다.
- [ ] 해당 SHA checkout에서 테스트·benchmark·sanitizer를 직접 실행했습니다. 환경 제한 때문에 미실행 상태입니다.
- [x] 별도의 프로젝트 재학습 없이이 Thread의 설계 → 구현 → 위험 → 수정 → 검증 발전을 설명할 수 있는 기록을 남겼습니다.
===== END FILE: 06-image-representation-and-atomic-output.md =====

===== BEGIN FILE: 07-reproducible-verification.md =====
# Thread 7. Reproducible verification infrastructure

## 1. Thread 목표

production core와 검증 executable을 같은 build graph에 묶고, component regressions, sanitizer instrumentation, Ubuntu/macOS CI까지 이어지는 재현 가능한 검증 경로를 복원합니다.

### Source significance

> These commits do not define the renderer's algorithms, but they turn local checks into a repeatable
> project-wide verification path. The progression matters because later geometry, BVH, concurrency,
> and output failure tests all depend on a stable library/test build and can be exercised under
> multiple platforms and runtime instrumentation rather than only through one command-line smoke path.

### 이 Thread에 연결된 source invariant

- Verification targets reuse the same production `raycore` objects rather than a separate implementation.
- Release portability checks and sanitizer diagnostics run as explicit automated jobs.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `raycore` library와 CLI/test/benchmark targets의 dependency 경계는 어떻게 구성되는가?
- CTest가 active CMake configuration에서 실제로 빌드된 executable을 smoke script에 어떻게 전달하는가?
- native core regression은 shell integration test와 어떤 책임을 나누는가?
- fixture path가 process working directory에 의존하지 않도록 어떤 compile/configuration 정보가 전달되는가?
- sanitizer option은 compile과 link 양쪽에 어떻게 전파되고 unsupported compiler를 어떻게 처리하는가?
- release portability와 sanitizer diagnostics를 CI job으로 분리한 이유와 실행 test set은 무엇인가?

## 3. 완료 기준

- [x] CMake target graph와 public include/link dependency를 실제 설정에서 그렸습니다.
- [x] CLI smoke와 native component test가 각각 통과하는 production path를 구분했습니다.
- [x] source fixture lookup이 build-tree 실행에서도 재현되는 근거를 기록했습니다.
- [x] ASan/UBSan enable path, compiler gate, frame pointer, runtime environment를 확인했습니다.
- [x] push/PR에서 Ubuntu·macOS release와 Linux sanitizer CTest가 실제로 실행되는 workflow를 추적했습니다.
- [x] 이 infrastructure가 algorithm correctness 자체를 대신 증명하지 않는다는 한계를 적었습니다.
- [x] 모든 참조 SHA가 `cpp/miniRT` branch HEAD의 ancestry에 속하는지 확인했습니다.
- [ ] 해당 SHA checkout에서 build/test/benchmark 명령을 직접 실행했습니다. 로컬 외부 네트워크와 checkout이 제공되지 않아 실행 evidence는 만들지 않았습니다.

### 검증 범위

- 지정 branch HEAD: `7d08c7c13fa68c3e60eea3c7014658b0a133e6f0`
- 각 참조 SHA는 Thread 내부의 연속 compare chain에서 `behind_by = 0`, merge base가 선행 SHA였고, Thread 종료 SHA도 branch HEAD의 조상으로 확인했습니다.
- 구현 설명은 해당 commit의 diff/file content를 기준으로 작성했으며, final HEAD의 후속 API를 과거 SHA에 소급하지 않았습니다.
- 테스트와 benchmark는 source mechanism과 production path만 검사했습니다. 실행 결과, sanitizer 결과, wall-clock 수치는 기록하지 않았습니다.

## 4. Commit map

1. `2cf2f17980bb` — `build(cmake): 코어 라이브러리와 검증 타깃 구성`
 - Importance: B
 - Tags: BUILD, TEST
 - Source-defined role: Separates `raycore`, executable, CTest targets under CMake.

2. `0e8c3b51e3b7` — `test(core): 수학·기하·파서·출력 회귀 기준 추가`
 - Importance: B
 - Tags: TEST, DETERMINISM
 - Source-defined role: Adds broad component regression coverage.

3. `58d53cce0ee5` — `build(sanitizers): 메모리와 정의되지 않은 동작 검사 구성`
 - Importance: B
 - Tags: BUILD, TEST, RISK
 - Source-defined role: Adds ASan/UBSan config.

4. `4491bea4d93c` — `ci: 플랫폼별 빌드와 회귀 검사 자동화`
 - Importance: B
 - Tags: BUILD, TEST, INTEGRATION
 - Source-defined role: Runs release regressions Ubuntu/macOS and sanitizer checks Linux.

## 5. Commit별 학습 기록

### 5.1 `2cf2f17980bb` — `build(cmake): 코어 라이브러리와 검증 타깃 구성`

- Importance: B
- Tags: BUILD, TEST
- Thread order: 1/4

#### Source에서 확정된 역할

- Development Thread role: Separates `raycore`, executable, CTest targets under CMake.

#### B-level 구현 역할 복원

- **직전 관련 상태:** Make 기반 단일 executable과 shell smoke만으로는 production sources를 여러 test/benchmark target이 안정적으로 재사용하거나 IDE/CTest/CI가 같은 dependency graph를 구성하기 어렵습니다.
- **핵심 구현 결정:** CMake 3.16, C++17, extensions off를 기준으로 production sources를 `raycore` library로 모읍니다. public include directory와 compiler별 warning flags를 library에 연결하고, CLI는 `raycore`를 link합니다. `BUILD_TESTING`/CTest 아래 smoke test는 configure 시 실제 target path를 전달하도록 generator expression을 사용합니다. Make targets는 CMake configure/build/ctest에 위임하고 nested rebuild를 만들지 않게 인자를 전달합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - CMakeLists.txt — `raycore`, CLI, testing, warnings and standard
 - Makefile — CMake delegation
 - tests/render_smoke.sh — built executable argument
- **caller → callee / data flow:** CMake configure → raycore compile → CLI link → CTest test registration with built target path → `ctest` execution
- **ownership·state transition:** production implementation의 authoritative object graph는 `raycore` 하나입니다. executable/test가 별도 source copy를 컴파일하지 않고 link dependency를 공유합니다.
- **failure/edge branch:** test가 hard-coded `./ray` 경로를 사용하면 multi-config/build-tree 환경에서 다른 binary를 실행하거나 찾지 못할 수 있습니다. source 목록 중복은 production/test drift를 만듭니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** CLI와 verification targets가 동일 production core를 사용하고 CTest가 active build artifact를 가리키는 재현 가능한 build 경계를 제공합니다.
- **이 SHA가 보장하지 않는 것:** 이 commit 자체가 algorithm correctness나 sanitizer clean을 증명하지는 않습니다.
- **직접 확인/후속 evidence:** target graph, public include/link, warning branch, smoke target path와 Make delegation을 source에서 확인했습니다. CMake/CTest 명령은 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: 이 Thread의 시작점
- 다음 Thread commit: `0e8c3b51e3b7`
- 이 commit이 다음 단계에 제공하는 것: `0e8c3b51e3b7`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.2 `0e8c3b51e3b7` — `test(core): 수학·기하·파서·출력 회귀 기준 추가`

- Importance: B
- Tags: TEST, DETERMINISM
- Thread order: 2/4

#### Source에서 확정된 역할

- Development Thread role: Adds broad component regression coverage.

#### B-level 구현 역할 복원

- **직전 관련 상태:** shell smoke은 process-level valid/invalid rendering을 검증하지만 수학, primitive distance, parser line attribution, P3 local representation을 작은 실패 단위로 구분하기 어렵습니다.
- **핵심 구현 결정:** native `ray-core-tests` executable을 추가해 `raycore`를 link합니다. source fixture를 build working directory와 무관하게 찾도록 `RAY_SOURCE_DIR` compile definition을 제공합니다. tests는 vector/math, primitive intersection distance, invalid fixture의 line 3 ParseError, exact P3 text, parsed basic scene dimensions를 포함합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - CMakeLists.txt — `ray-core-tests`, `RAY_SOURCE_DIR`, CTest registration
 - tests/core_tests.cpp — component regressions
- **caller → callee / data flow:** CTest → native test executable → direct raycore API calls/fixture parse → assertions; shell smoke는 별도 CLI process path 유지
- **ownership·state transition:** test executable은 production library를 link하되 fixture location만 compile-time source root를 전달받습니다. process current directory가 fixture authority가 아닙니다.
- **failure/edge branch:** source root가 없으면 out-of-tree build에서 fixture lookup이 깨질 수 있습니다. broad smoke만 있으면 local failure가 checksum mismatch 하나로만 보일 수 있습니다.

#### Test commit 분석 기준

- **대상 production invariant:** 핵심 수학·기하·parser location·P3 encoding·basic fixture behavior가 production library에서 재현됩니다.
- **test technique:** single native assertion executable linked to raycore with source-root fixture definition
- **통과하는 production path:** direct public/core APIs, parser, output serialization
- **이 test가 증명하는 것:** selected component regressions를 shell process 없이 빠르게 고정합니다.
- **이 test가 증명하지 않는 것:** threading, BVH equivalence, sanitizer diagnostics, all CLI behavior를 증명하지 않습니다.
- **실행 상태:** 테스트 구현과 production 호출 경로는 해당 SHA에서 확인했지만, 이 환경에서는 checkout/build가 불가능해 명령을 실행하지 않았습니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** core component와 parser/output representation의 빠른 deterministic regression layer를 추가합니다.
- **이 SHA가 보장하지 않는 것:** 이 시점 test set은 이후 BVH, materials, concurrency, transactional output 전체를 아직 포함하지 않습니다.
- **직접 확인/후속 evidence:** 테스트 성격: broad native component regression. target/source definition과 production calls를 검사했으며 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `2cf2f17980bb`
- 다음 Thread commit: `58d53cce0ee5`
- 이 commit이 다음 단계에 제공하는 것: `58d53cce0ee5`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.3 `58d53cce0ee5` — `build(sanitizers): 메모리와 정의되지 않은 동작 검사 구성`

- Importance: B
- Tags: BUILD, TEST, RISK
- Thread order: 3/4

#### Source에서 확정된 역할

- Development Thread role: Adds ASan/UBSan config.

#### B-level 구현 역할 복원

- **직전 관련 상태:** 기능 tests가 통과해도 out-of-bounds, use-after-free, signed overflow 같은 memory/undefined behavior가 입력에서 관찰되지 않거나 platform별로 잠복할 수 있습니다.
- **핵심 구현 결정:** `RAY_ENABLE_SANITIZERS` CMake option을 default OFF로 추가합니다. GCC/Clang 계열에서 `-fsanitize=address,undefined`와 `-fno-omit-frame-pointer`를 `raycore`의 compile/link interface에 적용해 이를 link하는 executable/tests에도 runtime instrumentation이 이어지게 합니다. unsupported compiler에서는 silent ignore가 아니라 configure fatal error를 냅니다. sanitizer build directory를 ignore합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 - CMakeLists.txt — sanitizer option, compiler gate, compile/link options
 -.gitignore — sanitizer build tree
- **caller → callee / data flow:** configure option ON → compiler-family validation → instrumented raycore compile/link interface → linked tests/CLI instrumentation → CTest runtime diagnostics
- **ownership·state transition:** option OFF인 normal build와 ON인 diagnostic build가 별도 build tree에서 공존합니다. frame pointer가 diagnostic stack trace를 지원합니다.
- **failure/edge branch:** compile flags만 적용하고 link flags를 빼면 sanitizer runtime link가 실패할 수 있습니다. unsupported toolchain에서 option을 무시하면 user가 검사됐다고 오인합니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 지원 compiler에서 ASan/UBSan-enabled verification configuration을 명시적으로 구성합니다.
- **이 SHA가 보장하지 않는 것:** ThreadSanitizer, leak sanitizer portability, 모든 UB 검출, test coverage가 닿지 않는 path를 보장하지 않습니다. option 존재 자체가 clean run evidence는 아닙니다.
- **직접 확인/후속 evidence:** compile/link propagation과 fatal compiler gate를 확인했습니다. sanitizer binary를 빌드·실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `0e8c3b51e3b7`
- 다음 Thread commit: `4491bea4d93c`
- 이 commit이 다음 단계에 제공하는 것: `4491bea4d93c`가 소비하거나 검증할 수 있는 현재 contract/state를 만듭니다. 구체적인 연결은 다음 기록에서 현재 SHA와 비교해 설명했습니다.

### 5.4 `4491bea4d93c` — `ci: 플랫폼별 빌드와 회귀 검사 자동화`

- Importance: B
- Tags: BUILD, TEST, INTEGRATION
- Thread order: 4/4

#### Source에서 확정된 역할

- Development Thread role: Runs release regressions Ubuntu/macOS and sanitizer checks Linux.

#### B-level 구현 역할 복원

- **직전 관련 상태:** 로컬 CMake/CTest와 sanitizer option이 있어도 어떤 platform/configuration에서 지속적으로 실행되는지 자동화되지 않으면 portability와 diagnostic contract가 사람의 명령에 의존합니다.
- **핵심 구현 결정:** `.github/workflows/ci.yml`이 push와 pull request에서 실행됩니다. Release job은 Ubuntu와 macOS matrix로 CMake configure(`BUILD_TESTING=ON`), build, `ctest --output-on-failure`를 수행합니다. 별도 Linux Debug sanitizer job은 `RAY_ENABLE_SANITIZERS=ON`으로 configure하고 ASAN/UBSAN 환경 변수를 설정해 leak detection, halt-on-error, stack trace와 함께 같은 CTest graph를 실행합니다.

#### 실제 코드와 실행 경로

- **확인한 file path와 symbol:**
 -.github/workflows/ci.yml — release matrix and sanitizer job
 - CMakeLists.txt — consumed configuration options/tests
- **caller → callee / data flow:** push/PR → platform checkout/configure → build → CTest; Linux sanitizer branch → instrumented Debug build → ASAN/UBSAN runtime env → CTest
- **ownership·state transition:** release portability와 runtime instrumentation은 독립 jobs라 한쪽 실패가 다른 증거로 대체되지 않습니다. 동일 registered tests를 다른 configuration에서 실행합니다.
- **failure/edge branch:** macOS/Ubuntu compile 차이, release-only issues, sanitizer diagnostics는 각 job에서 별도 failure status가 됩니다.

#### 보장 범위와 남은 공백

- **이 SHA가 보장하는 것:** 지정 workflow가 유지되는 동안 두 OS의 release regression과 Linux ASan/UBSan regression을 자동으로 요청합니다.
- **이 SHA가 보장하지 않는 것:** Windows, 다른 compiler/version, race detector, 실제 workflow run 성공을이 source 검사만으로 증명하지 않습니다. 이 작업에서는 과거 CI run이나 명령 결과를 성공으로 기록하지 않았습니다.
- **직접 확인/후속 evidence:** workflow triggers, matrix, configure/build/ctest commands와 sanitizer environment를 source에서 확인했습니다. CI는 새로 실행하지 않았습니다.

#### Thread 내 연결

- 이전 Thread commit: `58d53cce0ee5`
- 다음 Thread commit: 이 Thread의 종료점
- 이 commit이 Thread 종료에 제공하는 것: Thread-level invariant ledger와 최종 실행 순서에서이 SHA의 결과를 최종 상태에 반영했습니다.

## 6. Invariant ledger

| Invariant | 최초 도입/기준 | 강화 또는 수정 | 부족함/위험 노출 | 고정한 test/evidence | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| tests가 동일 production core를 사용 | 2cf2f17980bb | 0e8c3b51e3b7에서 native tests 확장 | 별도 source list/drift | CTest target graph | `raycore` public link dependency |
| fixture lookup은 working directory와 무관 | 0e8c3b51e3b7 | 0e8c3b51e3b7 | out-of-tree CTest에서 relative path 실패 | native parser fixture test | `RAY_SOURCE_DIR` definition |
| memory/UB instrumentation은 explicit config | 58d53cce0ee5 | 4491bea4d93c CI job | option 존재만으로 clean run 오인 | Linux sanitizer CTest job | compile+link flags and runtime env |
| release portability는 Ubuntu/macOS에서 자동 검사 | 4491bea4d93c | 4491bea4d93c | 한 platform local build만 확인 | release matrix | configure/build/ctest per OS |

### Ledger 보완 기록

- 각 invariant는 위 표의 SHA에서 observable behavior 또는 state로 처음 나타났습니다.
- 후속 commit이 같은 용어를 사용하더라도 그 보장을 과거 SHA에 소급하지 않았습니다.
- test/evidence 열은 production path와 assertion 또는 deterministic work gate를 함께 가리킵니다.
- 실행하지 않은 test는 source-level evidence로만 기록했습니다.

## 7. Failure → Fix → Test 연결

| Failure 또는 위험 | Decision/Fix | Test 또는 evidence | 실제 실패 처리와 assertion |
| --- | --- | --- | --- |
| test가 다른 source copy 또는 binary 실행 | `raycore` link + generated target path | CTest smoke/native tests | active build artifact path |
| out-of-tree fixture lookup 실패 | compile-time source root | 0e8c3b51e3b7 core test | invalid fixture line assertion |
| sanitizer compile만 되고 link/runtime 누락 | PUBLIC compile+link options and explicit job | 4491bea4d93c sanitizer job | instrumented CTest with ASAN/UBSAN env |
| single-platform portability blind spot | Ubuntu/macOS release matrix | CI workflow | 각 OS configure/build/ctest status |

### 연결 검토

- feature commit도 어떤 잘못된 state 또는 semantic drift를 막는지 production path에 연결했습니다.
- fix commit은 기존 가정 → 실제 위험 → root cause → corrected decision → regression 순서로 기록했습니다.
- test가 broad integration인지 deterministic boundary/differential/failure-injection regression인지 commit 기록에서 구분했습니다.
- assertion이 증명하지 않는 범위와 실행하지 못한 항목을 별도로 남겼습니다.

## 8. Ownership / state / responsibility 변화

CMake target graph가 production source 목록의 authority를 `raycore`에 둡니다. CLI, native tests, benchmark는 library를 소유하지 않고 link dependency로 소비합니다. CTest가 executable invocation metadata를 소유하고 workflow가 build directories/configurations를 job별로 격리합니다. sanitizer option은 `raycore` public usage requirement로 linked targets에 전파됩니다.

### 학습자 최종 기록

- **source state와 derived state:** CMake target graph가 production source 목록의 authority를 `raycore`에 둡니다. CLI, native tests, benchmark는 library를 소유하지 않고 link dependency로 소비합니다. CTest가 executable invocation metadata를 소유하고 workflow가 build directories/configurations를 job별로 격리합니다. sanitizer option은 `raycore` public usage requirement로 linked targets에 전파됩니다.
- **mutation/transition boundary:** commit별 `ownership·state transition`과 위 invariant ledger에 표시했습니다.
- **failure 시 복구 상태:** Failure → Fix → Test 표와 각 fix/test section에 정상·오류 상태를 구분했습니다.

## 9. Thread 최종 상태

동일 production `raycore`를 사용하는 CLI·native tests·benchmark가 CMake graph에 있고 CTest가 shell integration과 component regressions를 실행합니다. 별도 sanitizer configuration은 GCC/Clang compile/link instrumentation을 적용하고, CI는 Ubuntu/macOS Release와 Linux ASan/UBSan jobs를 분리합니다. source inspection은이 자동화의 구성만 확인하며 실제 workflow/test 성공을 대신하지 않습니다.

### 직접 작성한 결론

- **Thread 시작과 종료의 behavior 차이:** 동일 production `raycore`를 사용하는 CLI·native tests·benchmark가 CMake graph에 있고 CTest가 shell integration과 component regressions를 실행합니다. 별도 sanitizer configuration은 GCC/Clang compile/link instrumentation을 적용하고, CI는 Ubuntu/macOS Release와 Linux ASan/UBSan jobs를 분리합니다. source inspection은이 자동화의 구성만 확인하며 실제 workflow/test 성공을 대신하지 않습니다.
- **아직 다른 Thread 또는 외부 검증이 보완해야 하는 항목:** 이 작업에서는 로컬 build/test/sanitizer/CI를 실행하지 않았습니다. Windows, TSan, 추가 compiler와 actual historical run status는 별도 evidence가 필요합니다.

## 10. 최종 architecture 또는 execution flow 정리

### Source가 확정한 흐름 anchor

```text
CMake configure → `raycore` → CLI/tests/benchmark link → CTest registration → optional sanitizer compile/link instrumentation → GitHub Actions release matrix and sanitizer job
```

### 실제 코드로 완성한 흐름

1. CMake가 C++17/no-extensions와 compiler warnings를 설정합니다.
2. production sources를 `raycore` library로 컴파일합니다.
3. CLI, tests, benchmark가 같은 library와 public include dependency를 link합니다.
4. CTest가 native core tests와 built executable을 받는 shell smoke를 등록합니다.
5. optional sanitizer configuration이 supported compiler인지 확인하고 compile/link flags를 전파합니다.
6. GitHub Actions release matrix가 Ubuntu/macOS에서 configure, build, CTest를 실행합니다.
7. Linux sanitizer job이 Debug/instrumented graph를 만들고 ASAN/UBSAN runtime options로 CTest를 실행합니다.

### 학습자의 최종 설명

동일 production `raycore`를 사용하는 CLI·native tests·benchmark가 CMake graph에 있고 CTest가 shell integration과 component regressions를 실행합니다. 별도 sanitizer configuration은 GCC/Clang compile/link instrumentation을 적용하고, CI는 Ubuntu/macOS Release와 Linux ASan/UBSan jobs를 분리합니다. source inspection은이 자동화의 구성만 확인하며 실제 workflow/test 성공을 대신하지 않습니다.

남은 경계는 다음과 같습니다. 이 작업에서는 로컬 build/test/sanitizer/CI를 실행하지 않았습니다. Windows, TSan, 추가 compiler와 actual historical run status는 별도 evidence가 필요합니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 source 순서대로 확인했습니다.
- [x] 각 commit의 SHA, subject, importance, tags를 그대로 유지했습니다.
- [x] 모든 핵심 설명에 해당 SHA의 file path와 symbol 근거를 기록했습니다.
- [x] final HEAD의 구조를 과거 SHA에 소급하지 않았습니다.
- [x] S/A/B importance에 맞춰 architecture, subsystem, localized role의 깊이를 구분했습니다.
- [x] source에서 확정하지 않은 실행 결과나 runtime 수치를 사실로 채우지 않았습니다.
- [x] failure와 fix/test를 실제 production path로 연결했습니다.
- [x] test가 증명하는 것과 증명하지 않는 것을 구분했습니다.
- [x] invariant ledger의 각 변화를 commit evidence와 연결했습니다.
- [ ] 해당 SHA checkout에서 테스트·benchmark·sanitizer를 직접 실행했습니다. 환경 제한 때문에 미실행 상태입니다.
- [x] 별도의 프로젝트 재학습 없이이 Thread의 설계 → 구현 → 위험 → 수정 → 검증 발전을 설명할 수 있는 기록을 남겼습니다.
===== END FILE: 07-reproducible-verification.md =====

===== BEGIN FILE: README.md =====
# miniRT Development Thread 학습 골격

## 목적

이 문서 세트는 `commit-importance.md`에 정의된 Development Threads와
`commit-bodies.md`의 commit 의도를 기준으로, 실제 commit history와 해당 SHA의 코드를
직접 읽으며 설계 → 구현 → 실패 → 수정 → 검증 과정을 복원하기 위한 학습 골격입니다.

완성형 프로젝트 해설서가 아닙니다. 각 문서의 빈 기록란은 학습자가 해당 SHA의 코드,
직전 관련 SHA의 코드, production test path를 직접 확인한 뒤 채워야 합니다.

## 권장 학습 순서

1. [`01-geometric-contracts-to-first-image.md`](01-geometric-contracts-to-first-image.md)
2. [`02-large-vector-normalization.md`](02-large-vector-normalization.md)
3. [`03-correctness-preserving-bvh.md`](03-correctness-preserving-bvh.md)
4. [`04-material-syntax-and-reflection.md`](04-material-syntax-and-reflection.md)
5. [`05-deterministic-tiled-rendering.md`](05-deterministic-tiled-rendering.md)
6. [`06-image-representation-and-atomic-output.md`](06-image-representation-and-atomic-output.md)
7. [`07-reproducible-verification.md`](07-reproducible-verification.md)

Thread 순서와 각 Thread 내부 commit 순서는 source의 Development Threads를 그대로 따릅니다.
동일 commit이 여러 Thread에 나타나는 경우에는 각 문서에서 별도로 확인해야 하며,
임의로 중복을 제거하지 않습니다.

## Thread 문서 사용법

각 commit은 다음 순서로 학습합니다.

1. commit map에서 SHA, subject, importance, tags, source-defined role을 확인합니다.
2. 반드시 해당 SHA를 checkout하거나 `git show`로 해당 시점의 파일을 읽습니다.
3. 문서에 지정된 심볼, caller/callee, state mutation, ownership, failure branch, test path를 확인합니다.
4. 필요한 최소 코드만 증거로 삽입하고, 코드가 증명하는 contract를 직접 설명합니다.
5. 직전 관련 commit과 비교해 무엇이 새로 보장되고 무엇이 아직 보장되지 않는지 적습니다.
6. Thread의 invariant ledger와 Failure → Fix → Test 표를 실제 코드 근거로 완성합니다.
7. 마지막에 execution flow와 architecture 설명을 자신의 문장으로 작성합니다.

## 해당 SHA 코드 확인 원칙

다음과 같은 방식으로 특정 시점의 코드를 확인합니다.

```sh
git show <sha> --stat
git diff <sha>^ <sha> -- <path>
git show <sha>:<path>
git grep -n "<symbol>" <sha> -- .
```

- `git show <sha>:<path>`로 해당 SHA의 파일 내용을 읽습니다.
- 변경 전 상태는 우선 `<sha>^` 또는 문서에 지정된 직전 관련 SHA와 비교합니다.
- test commit은 test code만 읽지 말고 실제로 통과하는 production function path까지 추적합니다.
- source가 특정 follow-up fix/test를 연결한 경우 그 SHA를 별도로 열어 비교합니다.
- path와 symbol name은 실제 repository에서 확인한 값을 기록합니다.

## final HEAD 소급 사용 금지

최종 HEAD의 구조, 이름, private field, helper, option, error handling을 과거 commit에 소급해
설명하면 안 됩니다. 각 기록은 반드시 해당 SHA에서 관찰한 코드만 근거로 작성합니다.

과거 SHA에 아직 존재하지 않는 후속 보장은 다음과 같이 구분합니다.

- 해당 SHA가 이미 보장하는 것
- 해당 SHA에서는 아직 보장하지 않는 것
- 후속 어느 commit이 그 공백을 보완하는지

## Importance별 학습 깊이

### S

프로젝트의 핵심 architecture 또는 invariant로 취급합니다.

- 직전 상태와 문제
- 기존 설계의 부족한 점
- 핵심 decision
- 실제 핵심 코드와 caller/callee
- ownership, lifecycle, state transition
- 주요 실패 처리
- 이 SHA가 보장하는 것과 아직 보장하지 않는 것
- 후속 fix와 regression test

위 항목을 빠짐없이 실제 코드 근거로 작성합니다.

### A

주요 subsystem, integration boundary, 실패 처리, numerical/geometric decision을 이해하는 깊이입니다.

- 핵심 구현과 state change
- 선택한 algorithm 또는 boundary
- 위험한 edge/failure branch
- 관련 test 또는 benchmark evidence
- 다음 관련 commit과의 연결

### B

Thread 흐름에서 맡는 구현 역할을 이해하는 깊이입니다.

- 변경된 핵심 심볼
- 필요한 caller/callee
- state 또는 data representation의 변화
- 해당 commit이 Thread의 다음 단계에 제공하는 것
- 관련 regression이 있으면 그 production path

### C

Thread 이해에 필요한 맥락만 확인합니다. 문서 유지보수나 evidence snapshot을
S/A와 같은 깊이로 분석하지 않습니다. 현재 source의 Development Threads에는 C commit이 없지만,
다른 문맥에서 C commit을 참조할 때이 기준을 적용합니다.

## 실제 코드 삽입 기준

코드는 설명을 대신하기 위한 대량 복사가 아니라 contract를 증명하기 위한 최소 증거로 삽입합니다.

- 핵심 field 또는 type declaration
- decision이 드러나는 condition/branch
- ownership transfer 또는 invalidation 지점
- caller에서 callee로 state가 전달되는 부분
- failure cleanup 또는 commit point
- regression이 주입하는 failure와 assertion
- 변경 전후 차이를 설명하는 데 필요한 최소 범위

코드마다 다음을 함께 기록합니다.

- SHA
- file path
- symbol
- 선택한 line 범위
- 이 코드가 증명하는 invariant 또는 state transition
- 이 코드만으로는 증명되지 않는 항목

## Test commit 학습 방법

각 test commit에서 반드시 다음을 구분합니다.

- 대상으로 삼는 production invariant
- 재현하는 failure 또는 boundary
- 사용한 test technique
- 실제로 통과하는 production code path
- test가 증명하는 것
- test가 증명하지 않는 것
- broad integration test인지 deterministic regression인지
- 후속 변경에서 막는 regression

golden checksum이나 byte comparison이 존재하는 경우, 값만 기록하지 말고
어떤 upstream behavior까지 포함하는지와 local encoding test와의 차이를 설명합니다.

## 문서 완료 기준

모든 Thread 문서에서 다음 조건을 만족해야 완료입니다.

- 모든 commit을 source 순서대로 학습했습니다.
- SHA, subject, importance, tags를 변경하지 않았습니다.
- 각 기록에 해당 SHA의 실제 file path와 symbol이 있습니다.
- final HEAD를 과거 commit 설명에 소급하지 않았습니다.
- S/A/B 깊이가 구분되어 있습니다.
- source가 확정하지 않은 구현 해석을 사실처럼 채우지 않았습니다.
- fix commit은 assumption → failure/risk → root cause → decision → code → regression으로 연결했습니다.
- test commit은 production path와 증명 범위를 구분했습니다.
- invariant ledger가 commit history에 따라 완성되었습니다.
- Thread의 최종 architecture 또는 execution flow를 실제 코드 근거로 설명할 수 있습니다.
- 별도의 프로젝트 재학습 없이 설계 → 구현 → 실패 → 수정 → 검증의 발전 과정을 설명할 수 있습니다.
===== END FILE: README.md =====

