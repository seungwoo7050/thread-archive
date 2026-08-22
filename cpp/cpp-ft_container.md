===== BEGIN FILE: 01-cxx98-generic-interface-foundation.md =====
# C++98 Generic Interface Foundation

## 1. Thread 목표

### Source-established significance

The branch begins by constructing the generic vocabulary that C++98 does not provide in the later standard-library form used by modern code. The sequence is a real dependency chain: SFINAE separates count and range overloads, pair supplies map's value and return contracts, shared algorithms support relational operators, and iterator metadata enables one reverse adaptor to serve both pointer-backed and tree-backed iterators. The strict build then turns those assumptions into an enforceable compatibility boundary. Most individual implementations are conventional, but together they prevent each container from inventing incompatible local substitutes.

### 이 Thread에서 복원할 것

- 위 significance가 설명하는 변화 과정을 각 commit의 실제 SHA 코드로 재구성합니다.
- source가 확정한 commit 역할과 importance를 바꾸지 않고, 실제 implementation/failure/test 근거만 직접 채웁니다.

### Source에서 직접 연결되는 architecture

- The utility layer is split into independent headers for traits, pairs, algorithms, and iterators. Container headers consume those abstractions, while `ft_containers.hpp` provides an aggregate public entry point.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- C++98에서 count overload와 iterator-range overload를 runtime branching 없이 어떻게 분리하는가?
- pair, range algorithms, iterator traits가 container 내부 중복 구현을 어떻게 줄이는가?
- reverse iterator의 base convention이 pointer-backed vector와 tree-backed map에 공통으로 적용될 수 있는 이유는 무엇인가?
- strict C++98 build가 utility contract의 일부가 되는 지점은 어디인가?

## 3. 완료 기준

- A: 주요 subsystem/boundary/failure path/integration point를 실제 코드와 설계 판단으로 연결하고, 관련 regression 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- B: Thread 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- 모든 commit은 해당 SHA의 코드 또는 test/build diff를 근거로 기록합니다.
- Thread 최종 설명은 source 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 commit 간 변화로 재구성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| --- | --- | --- | --- | --- | --- |
| 1 | `ecc0668d6d9c` | `feat(type-traits): CXX98 타입 선택 도구 구현` | A | CXX98, ARCH, LEARNING | Establishes substitution-based overload selection and compile-time type classification. |
| 2 | `1c8692d14118` | `feat(pair): 값 쌍과 관계 연산 구현` | B | CXX98 | Supplies the pair value and return type required by associative-container APIs. |
| 3 | `07cf5893a53c` | `feat(algorithm): 공용 범위 비교 알고리즘 구현` | B | CXX98, PRACTICAL | Centralizes equality and lexicographical comparison for container relations. |
| 4 | `e3462cec55a1` | `feat(iterator): iterator 기본 형식과 traits 정의` | B | CXX98, ITERATOR | Defines iterator type metadata and pointer traits. |
| 5 | `7a8e3d32bb4d` | `feat(iterator): 역방향 반복자의 양방향 동작 구현` | B | ITERATOR | Adds the bidirectional reverse-iterator base convention. |
| 6 | `ae50e9038643` | `feat(iterator): 역방향 반복자의 임의 접근 연산 완성` | B | ITERATOR | Extends reverse iteration to the random-access operations needed by vector. |
| 7 | `455098520e83` | `test(utils): 공용 타입·값·범위·반복자 도구 검증` | B | TEST, CXX98 | Verifies the utility layer before containers depend on it. |
| 8 | `f36ec7e7e047` | `build(makefile): CXX98 검사 빌드 구성` | B | CXX98, PRACTICAL | Makes strict warning-enabled C++98 compilation the repeatable baseline. |

## 5. Commit별 학습 기록

### 1. feat(type-traits): CXX98 타입 선택 도구 구현

- SHA: `ecc0668d6d9c`
- Importance: A
- Tags: CXX98, ARCH, LEARNING
- Source-established role: Establishes substitution-based overload selection and compile-time type classification.
- Source summary: Introduces `enable_if`, integral constants, and integral-type detection for C++98 template dispatch.
- Source rationale: This is the project-wide mechanism that makes fill and range overloads coexist without runtime dispatch. It is a significant foundational interface decision, though it implements a standard utility rather than a defining container mechanism.

#### 해당 SHA에서 확인할 실제 코드

- 해당 SHA의 traits 전용 header에서 `enable_if`가 조건이 참일 때만 nested `type`을 노출하는 선언을 찾고, false case가 substitution에서 어떻게 사라지는지 타입 선언 수준에서 표시합니다.
- `integral_constant`, `true_type`, `false_type`의 value/type 관계와 `is_integral` primary template 및 C++98 integral specialization 목록을 확인합니다.
- 이 SHA에는 아직 실제 container overload consumer가 없다면 그 사실을 기록하고, later HEAD의 사용처를 소급해서 설명하지 않습니다.
- 확인한 파일/심볼: `include/ft_type_traits.hpp`의 `enable_if`, `integral_constant`, `true_type`, `false_type`, `is_integral`.
- 필요한 경우 비교할 직전 관련 SHA/parent: 해당 commit의 parent에는 이 header가 없으며, 이 SHA에서는 독립 utility 선언만 추가됩니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: 프로젝트 내부에 compile-time 조건으로 overload 후보를 제거하거나 integral type을 분류하는 공용 형식이 없었습니다.
- 해결하려던 문제: C++98에서 count 인자와 iterator 인자를 받는 template overload를 runtime 분기 없이 구분할 기반이 필요했습니다.
- 선택한 결정: `enable_if<false, T>`에는 `type`을 두지 않고, `enable_if<true, T>` specialization만 `type`을 노출했습니다. `is_integral`은 기본적으로 false이며 지원하는 정수 형식만 true specialization으로 열거했습니다.
- 새로 생긴 책임 경계 또는 상태 변화: overload 선택과 type 분류는 container 구현이 아니라 `ft_type_traits.hpp`가 담당하게 됐습니다. runtime 상태와 ownership 변화는 없습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: false 조건은 compile error를 직접 발생시키는 경로가 아니라 substitution 대상에서 해당 후보를 제거하는 경계입니다. 이 SHA에는 실제 consumer가 없으므로 동작 연결은 아직 선언 수준에서만 확인됩니다.
- state/ownership/lifecycle 영향: 없습니다. 전부 compile-time 형식과 상수입니다.
- 이 commit이 보장하는 것과 남은 한계: 기본 SFINAE와 C++98 integral 분류를 제공합니다. cv-qualified 정수 형식 처리나 실제 fill/range overload 적용은 이 SHA에서 증명하지 않습니다.
- 다음 관련 commit과의 연결: `pair`, algorithm, iterator utility가 같은 독립 header 계층을 확장하며, 이후 vector/map 공개 API가 이 공용 vocabulary를 소비합니다.

### 2. feat(pair): 값 쌍과 관계 연산 구현

- SHA: `1c8692d14118`
- Importance: B
- Tags: CXX98
- Source-established role: Supplies the pair value and return type required by associative-container APIs.
- Source summary: Implements `ft::pair`, converting construction, relational operators, and `make_pair`.
- Source rationale: The type is necessary support for map values and return contracts, but the implementation follows established value-type semantics and contains limited project-specific judgment.

#### 해당 SHA에서 확인할 실제 코드

- `ft::pair`의 `first`/`second` 저장 형식과 default/value/converting-copy/assignment 경로를 확인합니다.
- 관계 연산에서 `first`를 우선 비교하고, `first`가 서로 less가 아닐 때만 `second`를 비교하는 lexicographic 조건을 직접 추적합니다.
- `make_pair`가 호출자에게 template 인자를 명시하지 않게 하는 반환 형식 구성을 확인합니다.
- 확인한 파일/심볼: `include/ft_pair.hpp`의 `pair`, 관계 연산자, `make_pair`.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 pair header가 없고, 이 commit이 독립 value type을 추가합니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: 두 값을 묶어 map value와 `insert` 결과를 표현할 프로젝트 내부 형식이 없었습니다.
- 해결하려던 문제: 서로 다른 형식을 보존하는 두 필드, 변환 가능한 pair 간 복사, 일관된 관계 연산이 필요했습니다.
- 선택한 결정: `first_type`/`second_type`과 public `first`/`second`를 갖는 값 형식을 만들고, converting constructor와 assignment를 멤버 단위로 구현했습니다. `<`는 first 우선의 사전식 비교입니다.
- 새로 생긴 책임 경계 또는 상태 변화: map은 나중에 key/value 묶음과 `(iterator, bool)` 반환을 같은 공용 pair 계약으로 표현할 수 있게 됐습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: associative container가 사용할 값/결과 형식을 먼저 고정합니다.
- 핵심 코드와 상태 변화: pair는 두 멤버를 직접 소유하며 별도 자원 관리가 없습니다. 관계 연산은 `==`와 `<`를 기준으로 나머지를 파생합니다.
- 다음 commit에 넘기는 전제: container 관계 연산은 pair 원소 비교와 공용 range algorithm을 조합할 수 있습니다.

### 3. feat(algorithm): 공용 범위 비교 알고리즘 구현

- SHA: `07cf5893a53c`
- Importance: B
- Tags: CXX98, PRACTICAL
- Source-established role: Centralizes equality and lexicographical comparison for container relations.
- Source summary: Adds shared equality and lexicographical range-comparison algorithms.
- Source rationale: Centralizing these algorithms avoids duplicated comparison logic in containers, but the change is a straightforward implementation of expected generic algorithms within an already understood design.

#### 해당 SHA에서 확인할 실제 코드

- `equal` 기본/함수 객체 overload가 첫 번째 범위를 한 번씩 전진하며 mismatch에서 조기 종료하는지 확인합니다.
- `lexicographical_compare`의 조기 결정 조건과 prefix 처리 시점을 구분해서 기록합니다.
- 구현이 random-access 연산을 요구하지 않는지 실제 iterator 사용 연산만 추립니다.
- 확인한 파일/심볼: `include/ft_algorithm.hpp`의 `equal` 두 overload와 `lexicographical_compare` 두 overload.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 공용 range comparison header가 없습니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: container별 관계 연산이 생기면 범위 비교 loop를 중복 구현해야 하는 상태였습니다.
- 해결하려던 문제: 서로 다른 iterator 종류에도 적용되는 equality와 lexicographic ordering이 필요했습니다.
- 선택한 결정: `!=`, dereference, prefix increment만으로 진행하는 generic loop를 두고, mismatch 또는 ordering이 결정되는 즉시 반환합니다. 한 범위가 다른 범위의 prefix이면 종료 iterator 상태로 짧은 범위를 판정합니다.
- 새로 생긴 책임 경계 또는 상태 변화: 범위 비교 규칙은 container 밖 공용 algorithm header가 담당합니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: vector/map 관계 연산이 동일한 비교 규칙을 재사용하게 합니다.
- 핵심 코드와 상태 변화: 입력 범위를 읽기만 하며 iterator와 원소를 수정하지 않습니다. random-access 연산을 요구하지 않습니다.
- 다음 commit에 넘기는 전제: iterator가 최소한 dereference, increment, equality를 제공하면 공용 algorithm에 참여할 수 있습니다.

### 4. feat(iterator): iterator 기본 형식과 traits 정의

- SHA: `e3462cec55a1`
- Importance: B
- Tags: CXX98, ITERATOR
- Source-established role: Defines iterator type metadata and pointer traits.
- Source summary: Defines the iterator base form and `iterator_traits`, including pointer specializations.
- Source rationale: The traits layer is required by reverse iterators and containers, yet it is conventional support infrastructure rather than a project-defining architecture decision.

#### 해당 SHA에서 확인할 실제 코드

- `iterator` base template의 category/value/difference/pointer/reference alias와 `iterator_traits` 추출 방식을 확인합니다.
- mutable pointer와 const pointer specialization이 random-access category 및 `std::ptrdiff_t`를 노출하는지 확인합니다.
- reverse iterator가 concrete container가 아니라 traits protocol에 의존할 수 있게 된 경계를 기록합니다.
- 확인한 파일/심볼: `include/ft_iterator.hpp`의 `iterator`, `iterator_traits<Iterator>`, `iterator_traits<T*>`, `iterator_traits<const T*>`.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 iterator metadata protocol이 없습니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: generic adaptor가 iterator의 category와 value/reference/difference type을 공통 방식으로 얻을 수 없었습니다.
- 해결하려던 문제: class iterator와 raw pointer를 동일한 traits protocol로 다뤄야 했습니다.
- 선택한 결정: class iterator는 nested typedef를 추출하고, `T*`와 `const T*`는 random-access category와 `std::ptrdiff_t`를 명시하는 specialization을 제공합니다.
- 새로 생긴 책임 경계 또는 상태 변화: concrete container가 아니라 iterator type 자체 또는 pointer specialization이 traversal metadata를 제공합니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: reverse iterator가 vector pointer와 map iterator 모두에서 형식을 추출할 기반입니다.
- 핵심 코드와 상태 변화: compile-time aliases만 추가되며 runtime 객체 상태는 없습니다.
- 다음 commit에 넘기는 전제: reverse adaptor는 `iterator_traits<Iterator>`만 보면 reference, pointer, difference type을 결정할 수 있습니다.

### 5. feat(iterator): 역방향 반복자의 양방향 동작 구현

- SHA: `7a8e3d32bb4d`
- Importance: B
- Tags: ITERATOR
- Source-established role: Adds the bidirectional reverse-iterator base convention.
- Source summary: Implements bidirectional reverse-iterator construction, dereference, increment, decrement, and equality.
- Source rationale: This establishes expected reverse traversal semantics using the base-iterator convention, but it is normal implementation work inside the utility design.

#### 해당 SHA에서 확인할 실제 코드

- `reverse_iterator`가 저장하는 base iterator의 의미를 확인하고, dereference에서 base 복사본을 decrement한 뒤 접근하는 순서를 추적합니다.
- `operator++`/`operator--`가 underlying iterator 방향을 어떻게 반대로 적용하는지 확인합니다.
- converting constructor와 heterogeneous equality가 mutable/const 호환 타입 사이에서 어떤 변환만 허용하는지 확인합니다.
- 확인한 파일/심볼: `include/ft_iterator.hpp`의 `reverse_iterator`, `base`, `operator*`, `operator++`, `operator--`, heterogeneous `==`/`!=`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `e3462cec55a1`이 traits protocol을 먼저 제공합니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: iterator metadata는 있었지만 traversal 방향을 뒤집는 공용 adaptor는 없었습니다.
- 해결하려던 문제: vector와 map이 별도 reverse iterator를 구현하지 않고 같은 base convention을 사용해야 했습니다.
- 선택한 결정: 저장된 `_current`는 reverse 원소 자체가 아니라 그 원소 바로 다음의 forward position입니다. dereference는 복사한 base를 한 번 감소시켜 접근하며, reverse `++`는 base `--`, reverse `--`는 base `++`입니다.
- 새로 생긴 책임 경계 또는 상태 변화: adaptor는 base iterator 하나만 소유하고 실제 container/element ownership은 갖지 않습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: bidirectional traversal이 가능한 모든 iterator에 공통 reverse view를 제공합니다.
- 핵심 코드와 상태 변화: `_current`만 이동하며 underlying sequence는 변하지 않습니다. conversion 가능 여부는 base iterator 변환 가능성에 의해 제한됩니다.
- 다음 commit에 넘기는 전제: pointer-backed vector를 완전히 지원하려면 arithmetic, ordering, indexing, distance가 추가돼야 합니다.

### 6. feat(iterator): 역방향 반복자의 임의 접근 연산 완성

- SHA: `ae50e9038643`
- Importance: B
- Tags: ITERATOR
- Source-established role: Extends reverse iteration to the random-access operations needed by vector.
- Source summary: Completes reverse-iterator random-access arithmetic, ordering, indexing, and distance.
- Source rationale: The change makes the adaptor usable by pointer-backed vector iterators. It is technically correct supporting work, but it does not alter the project's architecture or critical ownership model.

#### 해당 SHA에서 확인할 실제 코드

- addition/subtraction/compound movement/indexing에서 positive reverse movement가 base subtraction으로 바뀌는 식을 확인합니다.
- 두 reverse iterator의 distance에서 피연산자 순서가 왜 뒤집혀 있는지 코드 식 그대로 기록합니다.
- 관계 연산이 base ordering을 어떻게 반전시키는지 확인하고 vector의 pointer iterator에서 요구되는 연산만 구분합니다.
- 확인한 파일/심볼: `include/ft_iterator.hpp`의 `operator+`, `operator-`, `operator+=`, `operator-=`, `operator[]`, 관계 연산자와 non-member distance.
- 필요한 경우 비교할 직전 관련 SHA/parent: `7a8e3d32bb4d`의 bidirectional base convention을 확장합니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: reverse traversal은 가능했지만 vector iterator처럼 offset 이동과 ordering을 요구하는 API는 완성되지 않았습니다.
- 해결하려던 문제: reverse 방향에서 forward base의 산술과 순서가 반전된다는 규칙을 모든 random-access 연산에 일관되게 적용해야 했습니다.
- 선택한 결정: `r + n`은 `base - n`, `r - n`은 `base + n`, `lhs - rhs`는 `rhs.base() - lhs.base()`로 구현했습니다. `<`도 base의 `>`에 대응합니다.
- 새로 생긴 책임 경계 또는 상태 변화: random-access 기능은 base iterator가 같은 연산을 제공한다는 compile-time 전제를 가집니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: vector의 `rbegin`/`rend` 및 reverse random-access 표면을 완성합니다.
- 핵심 코드와 상태 변화: base 위치만 산술적으로 이동하며 element 수명이나 storage는 바꾸지 않습니다.
- 다음 commit에 넘기는 전제: utility test가 raw array/pointer를 통해 base convention과 traits를 조합 검증할 수 있습니다.

### 7. test(utils): 공용 타입·값·범위·반복자 도구 검증

- SHA: `455098520e83`
- Importance: B
- Tags: TEST, CXX98
- Source-established role: Verifies the utility layer before containers depend on it.
- Source summary: Adds initial checks for pair, type traits, range algorithms, iterator traits, and reverse iteration.
- Source rationale: The tests establish basic confidence in the utility substrate before containers depend on it, but they exercise ordinary behavior rather than a difficult invariant or regression.

#### 해당 SHA에서 확인할 실제 코드

- utility test executable에서 pair, integral classification, prefix equality, lexicographic comparison, raw-array reverse traversal, pointer `iterator_traits` 각각의 assertion을 찾습니다.
- 각 assertion이 단일 utility만 보는지, 여러 utility의 조합을 보는지 구분합니다.
- 이 테스트가 container 구현 전에 진단 범위를 줄이는 baseline이라는 역할과, container ownership/failure를 아직 증명하지 않는다는 한계를 기록합니다.
- 확인한 파일/심볼: `tests/test_containers.cpp`의 utility 관련 test 함수와 `main` 호출 경로.
- 필요한 경우 비교할 직전 관련 SHA/parent: utility headers가 모두 존재하는 `ae50e9038643` 이후 상태입니다.

#### Test/verification 학습 기록

- 대상 production invariant: pair/traits/algorithm/reverse adaptor가 서로 호환되는 C++98 utility surface를 제공해야 합니다.
- 재현하는 failure 또는 boundary: integral과 non-integral 분류, equal mismatch, lexicographic prefix/차이, raw-array reverse traversal, pointer traits 형식입니다.
- test technique: 고정 입력에 대한 deterministic unit/integration assertion입니다. 표준 컨테이너와의 differential test나 failure injection은 아닙니다.
- 통과하는 production 코드 경로: `ft::pair` 생성·비교, `is_integral`, `equal`, `lexicographical_compare`, `iterator_traits<T*>`, `reverse_iterator<int*>`입니다.
- 이 테스트가 증명하는 것: 대표 입력에서 utility 선언이 함께 compile되고 예상 값을 반환합니다.
- 이 테스트가 증명하지 않는 것: container ownership, allocator state, exception rollback, 모든 변환/형식 조합은 증명하지 않습니다.
- 성격: 여러 utility를 한 executable에서 확인하는 초기 broad baseline입니다. 특정 과거 버그를 재현하는 regression은 아닙니다.
- 후속 변경에서 막아야 하는 회귀: public utility 이름/형식/기본 의미가 바뀌어 이후 container compile 또는 기본 비교가 깨지는 회귀입니다.
- 실행 증거: 이 작업 환경에서는 repository checkout이 불가능해 executable을 실행하지 않았습니다. 위 내용은 SHA diff의 test code 검사 결과입니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: container 구현 전 utility 문제를 독립적으로 좁혀 잡을 기준을 만듭니다.
- 핵심 코드와 상태 변화: production 상태 변화는 없고 test target이 utility API를 소비합니다.
- 다음 commit에 넘기는 전제: strict C++98 flags와 반복 가능한 Make target이 이 test를 동일 조건으로 compile/run해야 합니다.

### 8. build(makefile): CXX98 검사 빌드 구성

- SHA: `f36ec7e7e047`
- Importance: B
- Tags: CXX98, PRACTICAL
- Source-established role: Makes strict warning-enabled C++98 compilation the repeatable baseline.
- Source summary: Creates the strict C++98 Makefile test build and ignores generated build artifacts.
- Source rationale: This turns language-version and warning compatibility into a repeatable project constraint. It is important practical infrastructure, but not a core container mechanism.

#### 해당 SHA에서 확인할 실제 코드

- Makefile에서 C++98 language mode와 strict warning flags가 실제 compile command에 들어가는 지점을 확인합니다.
- public header dependency, build directory 분리, `test` target의 executable 실행 및 nonzero 중단 동작을 확인합니다.
- header-only library에서 consumer compilation 자체가 verification이 되는 경계를 기록합니다.
- 확인한 파일/심볼: `Makefile`의 `CXXFLAGS`, `CPPFLAGS`, `BUILD_DIR`, test binary rule, `test`, `clean`, `fclean`, `re`; `.gitignore`의 build artifact 제외.
- 필요한 경우 비교할 직전 관련 SHA/parent: `455098520e83`의 test source는 있었지만 반복 가능한 build contract가 없었습니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: test source가 존재해도 호출자마다 compiler mode와 warning 조건이 달라질 수 있었습니다.
- 해결하려던 문제: C++98 전용 public headers를 현대 기본 mode나 느슨한 warning 조건에서 우연히 통과시키지 않아야 했습니다.
- 선택한 결정: compile command에 `-Wall -Wextra -Werror -std=c++98`과 `-Iinclude`를 고정하고, 모든 public header를 dependency로 둔 test binary를 `build/`에 생성합니다. `test` loop는 각 binary를 실행하고 nonzero 즉시 종료합니다.
- 새로 생긴 책임 경계 또는 상태 변화: Makefile이 language/warning mode와 test 실행 순서를 반복 가능한 acceptance boundary로 소유합니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: utility API의 C++98 compile 가능성을 프로젝트 build 규칙으로 강제합니다.
- 핵심 코드와 상태 변화: source/header가 바뀌면 binary가 재빌드되며 generated artifacts는 source tree와 분리됩니다.
- 다음 commit에 넘기는 전제: 이후 vector/map/test가 같은 flags와 target convention에 편입될 수 있습니다.
- 실행 증거: Makefile 코드는 검사했으나 이 환경에서는 checkout 실패 때문에 `make test`를 실제 실행하지 않았습니다.

## 6. Invariant ledger

### Source에서 확정된 관련 invariant

- Every supported public header is self-contained under strict C++98 compilation, and the header-only implementation is safe to include from multiple translation units without linkage or ODR failures.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 commit | 부족함이 드러난 commit/상태 | 강화·복구한 fix | 고정한 test/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Every supported public header is self-contained under strict C++98 compilation, and the... | `ecc0668d6d9c`에서 독립 utility header 구성이 시작됨 | 이 Thread 시점에는 전체 public header 단독 compile 및 multi-TU link 검사가 아직 없음 | 이 Thread 내부 fix는 없음. 후속 Thread 06의 `d938c0079994`, `072c49832ddc`가 검증 범위를 확장함 | `455098520e83`, `f36ec7e7e047` | `include/ft_*.hpp`, `tests/test_containers.cpp`, Makefile의 `-std=c++98` test rule |

## 7. Failure → Fix → Test 연결

| 기존 상태/production change | fix 또는 verification | Source에서 확정된 연결 관점 | 실제 failure/root cause | 실제 test production path |
| --- | --- | --- | --- | --- |
| `Utility layer` | `455098520e83` | 공용 utility baseline verification | utility별 선언이 있어도 조합 compile과 대표 의미가 검증되지 않은 상태 | test executable이 pair/traits/algorithm/reverse iterator를 직접 호출 |
| `C++98 build contract` | `f36ec7e7e047` | strict C++98 compilation baseline | 수동 compiler invocation에서는 language mode와 warning 조건이 재현되지 않음 | Makefile compile rule → test binary → nonzero 즉시 중단 loop |

## 8. Ownership / state / responsibility 변화

| 시점 | Owner / state / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `ecc0668d6d9c` | Establishes substitution-based overload selection and compile-time type classification. | 공용 type dispatch 부재 | traits header가 SFINAE와 integral 분류 담당 | `include/ft_type_traits.hpp` |
| `1c8692d14118` | Supplies the pair value and return type required by associative-container APIs. | 두 값/결과 형식 부재 | `ft::pair`가 두 멤버와 관계 연산 소유 | `include/ft_pair.hpp` |
| `07cf5893a53c` | Centralizes equality and lexicographical comparison for container relations. | container별 비교 중복 가능 | 공용 range algorithm이 읽기 전용 비교 담당 | `include/ft_algorithm.hpp` |
| `e3462cec55a1` | Defines iterator type metadata and pointer traits. | iterator별 metadata 접근 규칙 부재 | traits protocol과 pointer specialization 확립 | `include/ft_iterator.hpp` |
| `7a8e3d32bb4d` | Adds the bidirectional reverse-iterator base convention. | reverse traversal 부재 | adaptor가 base iterator 하나로 반대 방향 표현 | `reverse_iterator::_current`, `operator*`, `++`, `--` |
| `ae50e9038643` | Extends reverse iteration to the random-access operations needed by vector. | bidirectional 기능만 존재 | 산술·순서·distance가 base 반전 규칙으로 확장 | reverse random-access operators |
| `455098520e83` | Verifies the utility layer before containers depend on it. | 선언만 존재 | 대표 utility composition을 test가 소비 | `tests/test_containers.cpp` |
| `f36ec7e7e047` | Makes strict warning-enabled C++98 compilation the repeatable baseline. | 수동 build 조건 | Makefile이 flags, artifacts, execution을 소유 | `Makefile` |

## 9. Thread 최종 상태

- 최종적으로 성립한 representation/state: traits, pair, range algorithms, iterator metadata, reverse adaptor가 서로 독립 public header로 존재하며 Makefile이 C++98 compile/test 조건을 묶습니다.
- 최종적으로 보장하는 invariant: 해당 SHA들의 코드상 utility는 서로 조합 가능한 공용 vocabulary를 제공하고 strict C++98 test target에 편입됩니다. 전체 public header self-containment와 multi-TU ODR 안정성은 후속 Thread 06에서 별도 검증됩니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: adaptor의 random-access 연산은 base iterator가 해당 연산을 제공해야 합니다. 이 Thread의 test는 allocator/exception/container lifetime을 다루지 않습니다.
- 최종 verification evidence: `455098520e83`의 utility assertions와 `f36ec7e7e047`의 strict build/test rule을 코드로 확인했습니다. 실제 실행은 수행하지 못했습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point commit: compile-time dispatch 경계를 만든 `ecc0668d6d9c`와 반복 가능한 acceptance 조건을 만든 `f36ec7e7e047`입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 단계명은 source가 정의한 Thread progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 commit | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | failure/cleanup | 다음 단계에 남기는 invariant |
| --- | --- | --- | --- | --- | --- | --- |
| Overload selection / type classification | `ecc0668d6d9c` | `ft_type_traits.hpp` | template 조건과 후보 type | true specialization만 nested `type` 공개 | runtime cleanup 없음; false는 substitution에서 탈락 | 공용 compile-time dispatch 가능 |
| Pair value contract | `1c8692d14118` | `ft_pair.hpp` | 두 값 또는 변환 가능한 pair | 두 멤버 저장·변환·사전식 비교 | 별도 자원 없음 | map value/result 형식 준비 |
| Shared range comparison | `07cf5893a53c` | `ft_algorithm.hpp` | 두 iterator range | mismatch/order 결정까지 순차 전진 | mutation/cleanup 없음 | container 관계 연산 재사용 가능 |
| Iterator metadata | `e3462cec55a1` | `ft_iterator.hpp` | class iterator 또는 pointer | traits가 category/value/difference 추출 | runtime 없음 | adaptor가 concrete container에서 분리됨 |
| Reverse traversal convention | `7a8e3d32bb4d` | `reverse_iterator` | forward base position | dereference 시 base 복사 후 감소, 이동 방향 반전 | element ownership 없음 | bidirectional reverse traversal |
| Random-access reverse operations | `ae50e9038643` | reverse operators | base와 offset/다른 reverse iterator | 산술·순서·distance 피연산 방향 반전 | base precondition 위반은 별도 처리 없음 | vector reverse API 지원 |
| Utility composition test | `455098520e83` | `tests/test_containers.cpp` | 고정 utility 입력 | assertions로 대표 결과 검사 | 실패 시 test process 종료 | container 도입 전 baseline |
| Strict C++98 build baseline | `f36ec7e7e047` | `Makefile` | source/header/test | strict flags로 build 후 binary loop 실행 | nonzero 즉시 중단, clean은 build 제거 | 반복 가능한 C++98 acceptance boundary |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 source 순서대로 확인했습니다.
- [x] 각 commit 기록에 final HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A commit은 decision, failure boundary, ownership/state transition을 설명할 수 있습니다.
- [x] Test/perf commit은 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → failure/risk → root cause → 수정 → regression 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 commit history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] Thread 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 01-cxx98-generic-interface-foundation.md =====

===== BEGIN FILE: 02-vector-ownership-aliasing-exception-safe-mutation.md =====
# Vector Ownership, Aliasing, and Exception-safe Mutation

## 1. Thread 목표

### Source-established significance

The original representation is durable, but the first modifier implementations reveal why vector is primarily a lifetime-management problem rather than only an indexed sequence. Capacity overflow, null-storage arithmetic, self-aliasing, and exceptions from user-defined types each expose a different way that logical size can diverge from actual constructed objects. The thread progressively moves input capture and replacement construction before mutation, separates construction in raw slots from assignment in live slots, and adds failure-injection evidence. The final test-oracle correction also demonstrates that tests for a deliberately extended self-range contract must derive expected values independently rather than assuming another container defines the same overlap behavior.

### 이 Thread에서 복원할 것

- 위 significance가 설명하는 변화 과정을 각 commit의 실제 SHA 코드로 재구성합니다.
- source가 확정한 commit 역할과 importance를 바꾸지 않고, 실제 implementation/failure/test 근거만 직접 채웁니다.

### Source에서 직접 연결되는 architecture

- `ft::vector` owns an allocator, a contiguous allocation pointer, a constructed-element count, and an allocation capacity. The range `[data, data + size)` contains live objects; `[data + size, data + capacity)` is raw storage and must never be treated as constructed.

### Source에서 직접 연결되는 Major Engineering Difficulties

- Implementing vector insertion across both spare-capacity and reallocation paths while distinguishing assignment into live objects from construction into raw storage, preserving aliased input, and cleaning partially constructed tails after exceptions.
- Making vector's general construction, assignment, resize, and growth paths transactional enough to preserve lifetime and ownership invariants when user-defined copies, assignments, or allocations throw.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `size`가 단순 길이가 아니라 live-object ownership count가 되는 이유는 무엇인가?
- reallocation과 in-place insertion에서 exception guarantee가 달라지는 이유를 실제 lifetime operation으로 설명할 수 있는가?
- self-range/aliased value를 mutation 전에 snapshot해야 하는 정확한 invalidation 지점은 어디인가?
- `allocator::max_size()`와 null-backed empty storage가 각각 어떤 비정상 경계를 만든다?
- 최종 insertion path에서 raw storage와 live object를 어떻게 구분해 construct/assign/destroy하는가?

## 3. 완료 기준

- S: 핵심 architecture/invariant를 직전 상태 → failure 가능성 → 결정 → 실제 핵심 코드 → ownership/lifecycle/state transition → 후속 fix/test까지 코드 근거로 설명할 수 있어야 합니다.
- A: 주요 subsystem/boundary/failure path/integration point를 실제 코드와 설계 판단으로 연결하고, 관련 regression 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- 모든 commit은 해당 SHA의 코드 또는 test/build diff를 근거로 기록합니다.
- Thread 최종 설명은 source 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 commit 간 변화로 재구성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| --- | --- | --- | --- | --- | --- |
| 1 | `2eb9cb6c4273` | `feat(vector): allocator 기반 저장소 수명 관리` | S | CORE, VECTOR, ALLOCATOR | Establishes allocator-backed contiguous storage and the constructed-object ownership boundary. |
| 2 | `9db46550b13d` | `feat(vector): 용량 확장과 원소 재배치 구현` | A | VECTOR, ALLOCATOR, CORE | Adds replacement-block reallocation and geometric capacity growth. |
| 3 | `6f3cbf4794c9` | `fix(vector): 용량 계산을 allocator 상한에서 포화` | A | VECTOR, ALLOCATOR, EDGE | Corrects growth arithmetic at the allocator limit. |
| 4 | `bdc4c3123bc9` | `fix(vector): 자기 범위 assign과 insert 입력 보존` | A | VECTOR, EDGE, DEBUG | Snapshots self-range input before mutation invalidates it. |
| 5 | `bc3a74b9342e` | `fix(vector): allocator 형식과 빈 반복자 연산 보정` | A | VECTOR, ALLOCATOR, EDGE | Removes empty-storage pointer arithmetic and adopts allocator-provided size types. |
| 6 | `b3124b3808d5` | `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리` | A | VECTOR, EXCEPTION, ALLOCATOR | Makes construction, assignment, resize growth, and aliased push-back transactional. |
| 7 | `9051be26db5e` | `test(vector): 생성·대입·크기 변경 실패 주입` | A | TEST, VECTOR, EXCEPTION | Injects element and allocation failures to validate rollback and live-object counts. |
| 8 | `797c33904db3` | `fix(vector): fill·range 삽입의 객체 수명 보존` | S | CORE, VECTOR, HARD | Rebuilds fill/range insertion around explicit live-object and raw-storage paths. |
| 9 | `8df3d8e067c0` | `test(vector): 삽입 복사·대입·할당 실패 sweep` | A | TEST, VECTOR, EXCEPTION | Sweeps insertion copy, assignment, and allocation failures across both capacity branches. |
| 10 | `5bdb6eb81a89` | `test(vector): 자기 범위 기대값을 명시적 snapshot으로 구성` | A | TEST, VECTOR, DEBUG | Replaces a questionable reference modifier with an explicit snapshot-derived test oracle. |

## 5. Commit별 학습 기록

### 1. feat(vector): allocator 기반 저장소 수명 관리

- SHA: `2eb9cb6c4273`
- Importance: S
- Tags: CORE, VECTOR, ALLOCATOR
- Source-established role: Establishes allocator-backed contiguous storage and the constructed-object ownership boundary.
- Source summary: Establishes vector's allocator-backed contiguous storage, size/capacity state, construction rollback, and destruction path.
- Source rationale: This commit defines the ownership representation on which every later vector operation depends: allocated storage is distinct from constructed elements, and cleanup is centralized. Omitting it would leave a major gap in explaining the container's architecture and lifetime model.

#### Source에서 확정된 핵심 판단

- Problem: A C++98 vector implementation must own raw contiguous storage while separately tracking which positions contain live `T` objects. Allocation alone does not create elements, and destruction must never be applied to unconstructed slots or omitted for constructed ones.
- Decision: The commit represents vector as allocator state, a data pointer, a constructed size, and an allocation capacity. Fill construction allocates one block and constructs elements sequentially, while the failure path destroys the completed prefix and deallocates the block. Destruction centralizes reverse element teardown and storage release.
- Why it mattered: Every later vector operation—reserve, resize, assignment, insertion, erasure, swap, and exception rollback—depends on this separation between storage ownership and object lifetime. It is the core invariant that later fixes refine rather than replace.

#### 해당 SHA에서 확인할 실제 코드

- 해당 SHA의 첫 vector header에서 allocator, storage pointer, live-element count, capacity를 나타내는 핵심 상태 필드를 찾습니다.
- raw allocation과 element construction이 분리되는 helper/constructor 경로를 따라가며, `_size`가 언제 증가하는지 확인합니다.
- fill construction 중 copy/construct가 실패할 때 이미 생성된 prefix를 역순 destroy하고 block을 deallocate하는 failure branch를 추적합니다.
- destructor/cleanup helper가 정확히 live prefix만 destroy한 뒤 storage를 해제하는지 확인합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 `_alloc`, `_data`, `_size`, `_capacity`, fill constructor/`_assign_fill`, `_destroy_elements`, destructor.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 vector header가 없으며 이 commit이 representation을 처음 추가합니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: vector 객체, 연속 storage 소유권, live-object count가 전혀 정의되지 않았습니다.
- 해결하려던 문제: allocator가 반환한 raw memory와 실제로 constructor가 완료된 `T` 객체를 분리해 추적해야 했습니다.
- 선택한 결정: 한 allocation block을 `_data`가 소유하고, `_capacity`는 slot 수, `_size`는 `[0, _size)`에 존재하는 live object 수로 정의했습니다. construction 성공 직후에만 `_size`를 증가시킵니다.
- 새로 생긴 책임 경계 또는 상태 변화: allocator는 block과 element construction/destruction을 수행하고, vector는 그 호출 순서와 완료된 prefix 길이를 소유합니다.

#### S-level 심화 추적

- 기존 설계가 충분하지 않았던 이유: 이 commit 이전에는 설계 자체가 없었습니다. `capacity`를 곧 live count로 취급하면 uninitialized slot을 destroy하거나 construction에 실패한 slot까지 size가 가리키게 됩니다.
- 핵심 state/ownership/lifetime transition: `NULL/0/0` → block allocate 후 아직 live object 0개 → element별 construct 성공마다 `_size` 증가 → 완성된 `_data/_size/_capacity`; 실패하면 완성 prefix를 역순 destroy하고 block deallocate 후 예외 재전파입니다.
- failure scenario별 cleanup/rollback: allocate 실패 전에는 소유권을 얻지 않습니다. construct 실패 후에는 `[0, _size)`만 destroy하며 allocation 전체를 같은 `_alloc`로 해제합니다.
- 이 commit이 보장하는 것: 정상 생성/소멸과 fill construction 실패에서 live prefix와 raw tail을 구분합니다.
- 이 commit이 아직 보장하지 않는 것: capacity growth, self-aliasing, empty null arithmetic, 일반 modifier의 exception transaction, insertion lifetime은 아직 없습니다.
- 후속 fix/test와 연결되는 구조: 모든 후속 helper가 `_size`를 commit marker로 사용합니다. `9051be26db5e`가 live count/invalid destroy를 계측하고 `797c33904db3`가 insertion에 같은 경계를 적용합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: 네 상태 필드와 construction loop의 성공 후 `_size` 증가, catch의 prefix destroy/deallocate 순서입니다.

### 2. feat(vector): 용량 확장과 원소 재배치 구현

- SHA: `9db46550b13d`
- Importance: A
- Tags: VECTOR, ALLOCATOR, CORE
- Source-established role: Adds replacement-block reallocation and geometric capacity growth.
- Source summary: Adds capacity queries, reserve, geometric growth, and exception-aware reallocation.
- Source rationale: Reallocation is the central dynamic mechanism of vector: values must be copied into a new block before the old block is released. The decision is significant, though later commits strengthen arithmetic and transactional edge cases.

#### 해당 SHA에서 확인할 실제 코드

- `reserve`, `max_size`, growth helper, relocation helper의 caller/callee 관계를 그립니다.
- 새 block allocate → 기존 element copy-construct → 성공 후 old element destroy/deallocate → state publish 순서를 확인합니다.
- copy construction 실패 시 new block의 constructed prefix만 정리되고 original storage가 그대로 owner인지 확인합니다.
- 이 SHA의 doubling arithmetic가 allocator limit에서 아직 hardened되지 않았다는 source 명시를 기록하고, 후속 `6f3cbf4794c9`에서 같은 계산을 비교합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 `capacity`, `max_size`, `reserve`, `_next_capacity`, `_reallocate`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `2eb9cb6c4273`의 fixed block representation과 비교합니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: 초기 allocation 이후 capacity를 늘리거나 원소를 다른 block으로 옮길 경로가 없었습니다.
- 해결하려던 문제: 기존 값과 ownership을 잃지 않고 더 큰 연속 block으로 교체해야 했습니다.
- 선택한 결정: 새 block을 allocate하고 기존 live prefix를 copy-construct합니다. 전부 성공한 뒤 기존 live objects와 block을 정리하고 새 pointer/capacity를 publish합니다.
- 새로 생긴 책임 경계 또는 상태 변화: reallocation helper가 old/new block이 동시에 존재하는 transaction 구간과 최종 ownership transfer를 담당합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: copy constructor가 중간에 던지는 경우입니다. new block의 완료 prefix만 destroy/deallocate하며 old state는 건드리지 않습니다.
- state/ownership/lifecycle 영향: 성공 전에는 old vector가 owner이고 new block은 helper의 임시 소유입니다. 성공 시 old를 release하고 `_data/_capacity`를 교체합니다. `_size` 값은 동일합니다.
- 이 commit이 보장하는 것과 남은 한계: reallocation construction failure의 strong guarantee를 제공합니다. 하지만 doubling 계산이 `max_size()` 인접 구간에서 overflow할 위험은 남습니다.
- 다음 관련 commit과의 연결: `6f3cbf4794c9`가 growth arithmetic을 allocator 상한에서 포화시킵니다.

### 3. fix(vector): 용량 계산을 allocator 상한에서 포화

- SHA: `6f3cbf4794c9`
- Importance: A
- Tags: VECTOR, ALLOCATOR, EDGE
- Source-established role: Corrects growth arithmetic at the allocator limit.
- Source summary: Makes vector length checks and growth arithmetic saturate safely at the allocator's `max_size()`.
- Source rationale: The fix closes a non-obvious unsigned-overflow boundary that could select an invalid capacity or report the wrong failure. It restores an important allocator-limit contract without changing the core representation.

#### 해당 SHA에서 확인할 실제 코드

- `resize`, fill assignment, insertion, growth helper에서 `max_size()`를 넘는 요청을 mutation 전에 거부하는 branch를 확인합니다.
- doubling을 먼저 수행하지 않고 allocator limit에서 saturate하는 조건식과, validated minimum을 선택하는 순서를 추적합니다.
- 직전 관련 구현 `9db46550b13d`와 비교해 unsigned capacity arithmetic의 failure window가 정확히 어디서 사라졌는지 기록합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 요청 길이 preflight와 `_next_capacity`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `9db46550b13d`의 먼저 doubling하는 계산.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: 현재 capacity를 두 배로 만든 뒤 allocator limit과 비교해도 안전하다고 가정했습니다.
- 실제 failure 또는 위험: unsigned multiplication/addition이 먼저 wrap되면 작은 capacity가 선택되거나 잘못된 failure 경로로 들어갈 수 있습니다.
- root cause: arithmetic 자체가 유효한지 확인하기 전에 overflow 가능한 표현식을 평가했습니다.
- 수정된 invariant/decision: requested minimum을 먼저 `max_size()`와 비교하고, doubling 가능 여부를 `limit - _capacity` 형태로 판단해 계산 자체가 overflow하지 않게 합니다. 두 배가 불가능하면 limit에서 포화합니다.
- mutation/commit 순서의 변경: invalid length는 allocation이나 state mutation 전에 `length_error`로 거부됩니다.
- 실제 수정 코드 근거: `_next_capacity`의 `if (_capacity > limit - _capacity)` 계열 분기와 각 public modifier의 size preflight입니다.
- 연결되는 regression test SHA/근거: source가 연결한 `0ce21f9cf12d`의 bounded allocator가 max 5에서 포화와 `reserve(6)` 거부, 최종 outstanding block 0을 검사합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: allocator가 작은 `max_size()`를 반환하거나 capacity가 limit 절반을 넘는 경계입니다.
- state/ownership/lifecycle 영향: 잘못된 요청을 mutation 전에 거부하므로 기존 block과 live objects는 변하지 않습니다.
- 이 commit이 보장하는 것과 남은 한계: capacity 선택 arithmetic은 allocator limit 안에서 안전합니다. 실제 allocation 실패는 여전히 allocator 예외로 처리됩니다.
- 다음 관련 commit과의 연결: 이후 insertion/resize transaction은 이 validated capacity를 전제로 새 block을 준비합니다.

### 4. fix(vector): 자기 범위 assign과 insert 입력 보존

- SHA: `bdc4c3123bc9`
- Importance: A
- Tags: VECTOR, EDGE, DEBUG
- Source-established role: Snapshots self-range input before mutation invalidates it.
- Source summary: Snapshots range input before vector assign or insert mutates the source container.
- Source rationale: The prior implementation could invalidate its own iterators or read overwritten values during self-range modification. Separating input capture from mutation is a significant aliasing correction that establishes the library's chosen snapshot contract.

#### 해당 SHA에서 확인할 실제 코드

- range assign이 destination clear/mutation 전에 source range 전체를 독립 temporary vector로 materialize하는지 확인합니다.
- range insert가 source range와 insertion point 이후 suffix를 mutation 전에 어떤 독립 storage에 보존하는지 확인합니다.
- self-reference일 때 invalidation 이전에 input을 모두 소비한다는 순서를 표시하고, reconstruction이 여전히 incremental mutation이라는 한계도 기록합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 range `assign`, range `insert`, 입력 snapshot temporary와 tail snapshot.
- 필요한 경우 비교할 직전 관련 SHA/parent: self-range를 직접 소비하던 직전 구현 및 regression `61e8b46e668f`.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: `first`/`last`가 destination과 무관하다고 보고 clear/shift/reallocation 중에도 계속 읽었습니다.
- 실제 failure 또는 위험: source가 같은 vector이면 clear, overwrite 또는 reallocation 순간 iterator가 무효화되거나 아직 읽지 않은 값이 바뀝니다.
- root cause: input consumption과 destination mutation이 같은 시간 구간에 섞여 있었습니다.
- 수정된 invariant/decision: mutation 전에 source range를 독립 temporary에 모두 복사합니다. insert는 source뿐 아니라 position 이후 suffix도 보존한 뒤 reconstruction합니다.
- mutation/commit 순서의 변경: capture 완료 → destination mutation/rebuild 순으로 분리됩니다.
- 실제 수정 코드 근거: range overload 첫 부분의 temporary vector 생성과 insert tail snapshot입니다.
- 연결되는 regression test SHA/근거: `61e8b46e668f`가 self-range assign/insert 결과를 검사하고, `5bdb6eb81a89`가 그 expected oracle을 독립 snapshot 방식으로 교정합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: destination range가 source range와 동일하거나 부분 중첩될 때입니다.
- state/ownership/lifecycle 영향: snapshot이 완성되기 전 destination은 변하지 않습니다. snapshot 자체의 construction failure는 temporary destructor가 정리합니다.
- 이 commit이 보장하는 것과 남은 한계: chosen snapshot semantics의 입력 보존을 보장합니다. 이 시점 insert reconstruction은 여전히 public operations를 통한 incremental mutation이라 일반 exception guarantee는 완성되지 않았습니다.
- 다음 관련 commit과의 연결: `797c33904db3`가 snapshot을 유지하면서 insertion lifetime algorithm 자체를 재설계합니다.

### 5. fix(vector): allocator 형식과 빈 반복자 연산 보정

- SHA: `bc3a74b9342e`
- Importance: A
- Tags: VECTOR, ALLOCATOR, EDGE
- Source-established role: Removes empty-storage pointer arithmetic and adopts allocator-provided size types.
- Source summary: Uses allocator-provided size types and avoids pointer arithmetic/subtraction on an unallocated empty vector.
- Source rationale: The fix addresses both custom-allocator interface correctness and undefined empty-storage arithmetic. Although compact, it restores a fundamental boundary invariant used by end, insert, and erase.

#### 해당 SHA에서 확인할 실제 코드

- public `size_type`/`difference_type`가 allocator-provided type으로 바뀐 선언을 찾습니다.
- `_iterator_at`, `_index_of`, empty-range erase에서 null-backed empty state를 별도 처리하는 branch를 확인합니다.
- 해당 SHA의 empty `begin()==end()`, zero-count insert, empty erase 경로에서 null pointer addition/subtraction이 실제로 사라졌는지 추적합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 allocator typedefs, `_iterator_at`, `_index_of`, empty erase/no-op modifier branches.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent의 `_data + index`, `position - _data` 직접 연산과 regression `ccb98587e777`.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: empty vector의 `_data == NULL`이어도 0을 더하거나 두 null pointer를 빼는 식이 harmless하다고 가정했습니다.
- 실제 failure 또는 위험: C++ pointer arithmetic/subtraction은 유효한 array object 범위라는 전제가 있어 null-backed empty state에는 적용할 수 없습니다. custom allocator의 size/difference type도 무시됐습니다.
- root cause: 논리적으로 0이라는 결과와 language-level pointer operation의 유효성을 동일시했습니다.
- 수정된 invariant/decision: empty/null state는 explicit branch로 iterator/index를 반환하고, public size types는 allocator가 제공하는 형식을 사용합니다.
- mutation/commit 순서의 변경: zero-count/empty-range modifier는 pointer 계산 전에 no-op로 종료됩니다.
- 실제 수정 코드 근거: `_iterator_at`과 `_index_of`의 `_data == NULL` 검사, empty erase branch입니다.
- 연결되는 regression test SHA/근거: `ccb98587e777`가 empty begin/end, zero-count insert, empty range erase와 allocator 상태를 검사합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: default-constructed vector와 모든 원소를 지운 뒤 null storage인 상태입니다.
- state/ownership/lifecycle 영향: no-op 경로가 storage나 live count를 건드리지 않습니다.
- 이 commit이 보장하는 것과 남은 한계: empty API 경로에서 invalid pointer arithmetic을 제거합니다. dereferenceable iterator를 제공하는 것은 아니며 empty begin/end는 비교만 가능합니다.
- 다음 관련 commit과의 연결: 이후 transaction/insertion helper가 위치를 index로 바꿀 때 이 safe conversion을 사용합니다.

### 6. fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리

- SHA: `b3124b3808d5`
- Importance: A
- Tags: VECTOR, EXCEPTION, ALLOCATOR
- Source-established role: Makes construction, assignment, resize growth, and aliased push-back transactional.
- Source summary: Makes fill construction, assignment, resize growth, and aliased push-back use rollback or temporary-storage transactions.
- Source rationale: This substantially raises vector's failure guarantees by committing state only after construction succeeds and by snapshotting aliased values before reallocation. It is significant core hardening, but insertion requires a separate, still more defining lifetime redesign.

#### Source에서 확정된 핵심 판단

- Problem: The initial vector surface could clear or partially extend the active sequence before all replacement construction succeeded. A throwing element copy could therefore lose the original assignment target, leave a partially grown suffix, or read an aliased push-back argument after reallocation invalidated it.
- Decision: Fill construction now builds a complete block before publishing state. Fill and range assignment build a temporary vector and exchange storage only after success. Resize growth records the old size and destroys only the newly completed suffix on failure. Reallocating push-back copies an aliased argument before reserve can invalidate it.
- Why it mattered: This commit applies a consistent transaction pattern across several high-risk paths: prepare replacement state, track completed construction, and commit only when the new state is valid. It is not the final insertion solution, but it establishes the failure-handling method that the later insertion redesign follows.

#### 해당 SHA에서 확인할 실제 코드

- `_initialize_fill`가 complete block을 만들기 전 `_data/_size/_capacity`를 publish하지 않는지 확인합니다.
- fill/range assignment가 destination allocator를 유지한 temporary vector를 만들고 storage-only exchange로 commit하는 순서를 추적합니다.
- `resize` growth에서 old size를 기준으로 새 suffix만 rollback하는 branch를 확인합니다.
- reallocation이 필요한 `push_back(values[i])` 같은 aliased argument를 `reserve` 전에 복사하는 코드 위치를 확인합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 `_initialize_fill`, assignment overloads, `_swap_storage`, `resize`, `push_back`.
- 필요한 경우 비교할 직전 관련 SHA/parent: 기존 clear-then-build/partial growth 경로와 후속 `9051be26db5e`.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: destination state를 먼저 비우거나 `_size`를 진행시키면서 replacement를 만들었습니다. push argument가 자기 원소여도 reserve 후 읽을 수 있다고 가정했습니다.
- 실제 failure 또는 위험: copy/construct 예외로 원래 assignment 대상이 사라지거나 partially grown suffix가 남고, aliased reference는 reallocation 후 dangling이 됩니다.
- root cause: prepare와 commit이 분리되지 않았고, 외부처럼 보이는 reference가 내부 storage를 가리킬 가능성을 고려하지 않았습니다.
- 수정된 invariant/decision: replacement temporary/block을 완성한 뒤 storage만 교환합니다. resize는 old size를 commit 기준으로 삼고, push는 reallocation 전에 value snapshot을 만듭니다.
- mutation/commit 순서의 변경: prepare/snapshot → construct with progress count → success 후 publish; failure 시 newly constructed suffix/block만 정리합니다.
- 실제 수정 코드 근거: `_initialize_fill`의 local pointer/progress, assignment의 temporary + `_swap_storage`, resize catch, push snapshot입니다.
- 연결되는 regression test SHA/근거: `9051be26db5e`가 fill/copy assignment, resize, aliased push-back에 copy/allocation failure를 주입합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: replacement construction, resize suffix construction, reserve 중 allocation/copy, self-aliased push argument입니다.
- state/ownership/lifecycle 영향: `_size`는 성공한 live object만 포함하며, failed temporary가 자신의 allocator로 block을 정리합니다. destination allocator object는 assignment commit에서 바뀌지 않습니다.
- 이 commit이 보장하는 것과 남은 한계: 여러 일반 modifier에서 original preservation 또는 정확한 suffix rollback을 제공합니다. 복잡한 fill/range insertion의 in-place lifetime은 아직 해결되지 않았습니다.
- 다음 관련 commit과의 연결: `797c33904db3`가 같은 transaction 원리를 insertion reallocation에 적용하고 in-place 경로는 별도 basic guarantee로 설계합니다.

### 7. test(vector): 생성·대입·크기 변경 실패 주입

- SHA: `9051be26db5e`
- Importance: A
- Tags: TEST, VECTOR, EXCEPTION
- Source-established role: Injects element and allocation failures to validate rollback and live-object counts.
- Source summary: Adds tracked-object and allocator failure injection for construction, assignment, resize, and aliased push-back.
- Source rationale: The suite validates live-object counts, invalid copies/destructions, block release, and preservation of original values. It materially changes confidence in vector's lifetime guarantees rather than merely adding normal-path coverage.

#### 해당 SHA에서 확인할 실제 코드

- `tracked_value`가 live object 수, dead-source copy, duplicate destruction, 선택적 copy/assignment throw를 어떻게 기록하는지 확인합니다.
- tracking allocator가 outstanding block, allocation failure, small `max_size`를 독립적으로 계측하는지 확인합니다.
- fill construction/assignment/copy assignment/resize growth/aliased push_back failure가 어떤 production path를 통과하는지 각각 연결합니다.
- 각 실패 뒤 original value 보존 여부와 live/block count zero 조건을 분리해서 기록합니다.
- 확인한 파일/심볼: `tests/test_vector_exceptions.cpp`의 `tracked_value`, tracking/bounded allocator state, 각 `test_*` 함수.
- 필요한 경우 비교할 직전 관련 SHA/parent: production fix `b3124b3808d5`; allocator-limit regression에는 `6f3cbf4794c9`가 연결됩니다.

#### Test/verification 학습 기록

- 대상 production invariant: `_size`는 live object 수와 일치하고 실패한 construction/assignment/growth가 block 또는 object를 누수하거나 invalid destroy하지 않아야 합니다.
- 재현하는 failure 또는 boundary: 지정한 copy/assignment attempt와 allocation attempt에서 예외, 작은 `max_size`, aliased push-back입니다.
- test technique: shared counters를 이용한 deterministic failure injection과 lifetime instrumentation입니다.
- 통과하는 production 코드 경로: fill/copy construction, fill/range/copy assignment, resize growth, reserve/push-back, allocator limit checks입니다.
- 이 테스트가 증명하는 것: 주입 지점에서 예외가 발생해도 expected original value가 보존되는 경로, live address 집합, invalid copy/destroy count, outstanding block count가 일관됩니다.
- 이 테스트가 증명하지 않는 것: 모든 element type/allocator 구현, 모든 possible failure interleaving, insertion의 복잡한 in-place branch는 아직 증명하지 않습니다.
- 성격: 특정 transaction fix를 고정하는 deterministic regression suite입니다.
- 후속 변경에서 막아야 하는 회귀: state를 성공 전에 publish하거나 failed prefix/suffix/block cleanup을 누락하고, reallocation 뒤 aliased value를 읽는 회귀입니다.
- 실행 증거: test source와 production diff만 검사했습니다. repository checkout 제약으로 binary는 실행하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: allocator가 아예 block을 주지 않는 경우와 element construction 후 일부 progress가 있는 경우를 분리합니다.
- state/ownership/lifecycle 영향: `tracked_value`의 live-address registry와 allocator block counter가 서로 다른 차원의 leak/invalid lifetime을 검출합니다.
- 이 commit이 보장하는 것과 남은 한계: 검사한 deterministic 지점의 rollback을 증명합니다. sanitizer나 실제 표준 allocator 전 조합을 대체하지 않습니다.
- 다음 관련 commit과의 연결: insertion redesign 뒤에는 `8df3d8e067c0`이 같은 기법을 fill/range insert 양쪽 capacity branch로 확장합니다.

### 8. fix(vector): fill·range 삽입의 객체 수명 보존

- SHA: `797c33904db3`
- Importance: S
- Tags: CORE, VECTOR, HARD
- Source-established role: Rebuilds fill/range insertion around explicit live-object and raw-storage paths.
- Source summary: Replaces vector fill/range insertion with explicit reallocation and in-place algorithms that distinguish live objects from uninitialized storage.
- Source rationale: This solves a project-defining object-lifetime problem. The new paths track constructed tails, snapshot aliased input, preserve spare capacity when possible, and clean partial construction on failure; without it the completed vector's correctness under non-trivial element types cannot be explained.

#### Source에서 확정된 핵심 판단

- Problem: The first insertion algorithm shifted values by constructing and destroying positions without consistently distinguishing already-live elements from raw capacity. Range insertion also rebuilt a tail through multiple public operations. With non-trivial or throwing element types, these approaches could violate lifetime bookkeeping, leak constructed objects, or leave invalid state.
- Decision: Insertion is split by both input form and capacity condition. Reallocation paths construct prefix, inserted values, and suffix into a fresh block and publish it only after completion. In-place paths construct only into the uninitialized tail, assign within the live range, track how many tail objects were completed, and destroy that tail if construction fails. Fill values and range inputs are snapshotted before mutation.
- Why it mattered: This is the decisive vector modifier design. It reconciles contiguous layout, capacity preservation, aliasing, construction versus assignment, and exception cleanup in one mechanism. The final vector cannot be considered correct for general element types without it.

#### 해당 SHA에서 확인할 실제 코드

- fill/range 각각에서 reallocation path와 spare-capacity in-place path가 분리된 helper 구조를 찾습니다.
- reallocation path의 prefix → inserted values → suffix construction 순서와 complete 후에만 old storage를 교체하는 commit boundary를 추적합니다.
- in-place path에서 raw tail slot에는 `construct`, 이미 live인 위치에는 assignment를 사용하는 구간 경계를 인덱스로 표시합니다.
- 부분 tail construction 또는 assignment failure 시 실제로 destroy되는 객체 범위와 `_size` bookkeeping을 확인하고 strong/basic guarantee를 구분합니다.
- fill value와 range input이 mutation 전에 snapshot되는 위치를 확인합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 fill/range `insert`, reallocation helpers, in-place helpers, `_replace_storage`/tail cleanup 경로.
- 필요한 경우 비교할 직전 관련 SHA/parent: `bdc4c3123bc9`의 incremental reconstruction, transaction pattern `b3124b3808d5`, regression `8df3d8e067c0`.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: move 대상 slot이 live인지 raw인지와 무관하게 construct/destroy를 섞고, range tail을 여러 public modifier로 다시 만들었습니다.
- 실제 failure 또는 위험: live object 위에 placement construction, raw slot에 assignment, partially constructed tail leak, `_size`와 실제 live objects 불일치가 가능합니다.
- root cause: insertion을 단일 shift 문제로 취급하고 object lifetime 영역 `[0,size)`와 raw 영역 `[size,capacity)`의 다른 연산 규칙을 분리하지 않았습니다.
- 수정된 invariant/decision: capacity 조건으로 reallocate/in-place를 나누고, in-place에서도 `count <= tail_size`와 `count > tail_size`를 구분합니다. raw tail에만 construct하고 live range에는 assignment합니다.
- mutation/commit 순서의 변경: input snapshot → 필요한 raw tail construction(progress 기록) → live assignment/shift → 최종 `_size` publish입니다. reallocation은 new block 전체 완성 후 old 교체입니다.
- 실제 수정 코드 근거: prefix/inserted/suffix construction loop, `constructed` progress counter, catch의 tail destroy, 성공 마지막의 `_size += count`입니다.
- 연결되는 regression test SHA/근거: `8df3d8e067c0`이 copy/assignment/allocation failure positions를 sweep하고 capacity/address/lifetime counters를 검사합니다.

#### S-level 심화 추적

- 기존 설계가 충분하지 않았던 이유: trivial `int`에서는 byte-like shift가 통과해도 non-trivial `T`의 constructor/destructor contract에는 맞지 않았습니다. public operations를 조합하면 중간 state가 외부 exception에 노출됩니다.
- 핵심 state/ownership/lifetime transition: reallocation은 old owner 유지 + new temporary block → prefix/insert/suffix live count 증가 → 완성 후 old destroy/deallocate → new publish입니다. in-place는 기존 live prefix 유지 → raw tail 일부 construct → live slots assignment → size commit입니다.
- failure scenario별 cleanup/rollback: reallocation copy/construct failure는 new prefix만 destroy/deallocate하고 old unchanged입니다. in-place tail construction failure는 새 tail만 destroy하며 original size를 유지합니다. live assignment failure는 이미 바뀐 값은 복구하지 못하지만 constructed tail을 정리하고 `_size`를 old size로 유지해 container를 destructible/usable한 valid state로 둡니다.
- 이 commit이 보장하는 것: raw/live 영역에 맞는 lifetime operation, aliased input capture, reallocation strong guarantee, in-place lifetime bookkeeping의 basic guarantee, spare capacity 유지입니다.
- 이 commit이 아직 보장하지 않는 것: throwing assignment를 포함한 in-place insertion에서 original 값의 strong guarantee는 제공하지 않습니다. arbitrary invalid iterators는 precondition 밖입니다.
- 후속 fix/test와 연결되는 구조: `8df3d8e067c0`이 두 branch와 failure 종류를 직접 sweep하고, `5bdb6eb81a89`가 self-range expected oracle 신뢰성을 고칩니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: `_size` 전후의 live/raw 경계, tail construction progress, reallocation publish boundary, in-place assignment failure 시 보장 수준입니다.

### 9. test(vector): 삽입 복사·대입·할당 실패 sweep

- SHA: `8df3d8e067c0`
- Importance: A
- Tags: TEST, VECTOR, EXCEPTION
- Source-established role: Sweeps insertion copy, assignment, and allocation failures across both capacity branches.
- Source summary: Sweeps copy, assignment, and allocation failures across fill and range insertion, including aliasing and spare-capacity behavior.
- Source rationale: The test matrix probes both reallocation and in-place branches, checks post-failure usability, and verifies no leaked or double-destroyed objects. It provides high-value regression evidence for the most complex vector modifier.

#### 해당 SHA에서 확인할 실제 코드

- copy/assignment/allocation failure position을 sweep하는 반복 구조와 각 injection counter reset 지점을 확인합니다.
- spare-capacity range insertion에서 capacity와 unaffected prefix 주소가 유지되는 assertion을 찾습니다.
- reallocating insertion sweep에서 허용되는 post-state가 unchanged original 또는 full inserted result뿐인지 확인합니다.
- 각 scenario 종료 후 live object, invalid lifetime operation, outstanding allocation이 zero인지 확인합니다.
- 확인한 파일/심볼: `tests/test_vector_exceptions.cpp`의 insertion test 함수, failure counter loops, capacity/address assertions, final tracker assertions.
- 필요한 경우 비교할 직전 관련 SHA/parent: production redesign `797c33904db3`.

#### Test/verification 학습 기록

- 대상 production invariant: fill/range insertion이 reallocation과 spare-capacity 양쪽에서 live/raw lifetime을 지키고 실패 후 소유 자원을 정확히 정리해야 합니다.
- 재현하는 failure 또는 boundary: fill value alias, self/range snapshot, new-block copy construction, in-place tail construction, live assignment, allocator allocation failure입니다.
- test technique: fail-at index를 증가시키는 deterministic failure sweep과 lifetime/allocation instrumentation입니다.
- 통과하는 production 코드 경로: fill/range reallocation helpers와 in-place helpers의 두 tail-size branch입니다.
- 이 테스트가 증명하는 것: reallocation 실패는 unchanged/full-result만 남고, in-place assignment failure에서도 size/lifetime bookkeeping과 이후 사용 가능성이 유지되며, leak/double-destroy가 없습니다. spare capacity와 unaffected prefix address도 지정 scenario에서 유지됩니다.
- 이 테스트가 증명하지 않는 것: 모든 possible `T` semantics나 무한 failure positions, throwing destructor, invalid input iterator는 다루지 않습니다.
- 성격: 가장 복잡한 modifier의 deterministic regression/failure-injection matrix입니다.
- 후속 변경에서 막아야 하는 회귀: raw slot assignment, live slot construct, size 조기 publish, partial tail cleanup 누락, 불필요한 reallocation, aliased input 재독취입니다.
- 실행 증거: code inspection만 수행했습니다. 실제 sweep binary 결과를 생성하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: failure가 new block publish 전인지, raw tail construction 중인지, live assignment 중인지에 따라 보장 수준이 다릅니다.
- state/ownership/lifecycle 영향: trackers가 object address와 allocation block을 따로 세어 logical size만으로 놓칠 수 있는 문제를 검출합니다.
- 이 commit이 보장하는 것과 남은 한계: 구현이 의도한 strong/basic guarantee 분리를 테스트합니다. 성능 복잡도나 모든 iterator invalidation 규칙은 별도입니다.
- 다음 관련 commit과의 연결: self-range scenario의 expected result가 독립적인지 `5bdb6eb81a89`에서 교정됩니다.

### 10. test(vector): 자기 범위 기대값을 명시적 snapshot으로 구성

- SHA: `5bdb6eb81a89`
- Importance: A
- Tags: TEST, VECTOR, DEBUG
- Source-established role: Replaces a questionable reference modifier with an explicit snapshot-derived test oracle.
- Source summary: Builds self-range expected values explicitly instead of asking `std::vector` to perform overlapping modifiers.
- Source rationale: The previous oracle coupled the regression to implementation- or version-sensitive overlapping-range behavior. This correction makes the chosen snapshot contract independently testable and restores trust in a significant edge-case test.

#### 해당 SHA에서 확인할 실제 코드

- self-range insertion expected sequence를 unchanged prefix + snapshotted source + unchanged suffix로 직접 조립하는 코드를 확인합니다.
- self-range assignment expected sequence를 selected interior range에서 독립적으로 만드는 코드를 확인합니다.
- `std::vector`에 overlapping modifier를 호출하던 이전 oracle을 제거한 diff를 직전 parent와 비교합니다.
- 이 테스트가 project-defined snapshot semantics를 검증하지만 일반적인 모든 overlapping operation의 표준 의미를 증명하는 것은 아니라는 범위를 기록합니다.
- 확인한 파일/심볼: `tests/test_containers.cpp` 또는 해당 vector regression source의 self-range expected construction.
- 필요한 경우 비교할 직전 관련 SHA/parent: `61e8b46e668f`의 reference-container modifier oracle.

#### Test/verification 학습 기록

- 대상 production invariant: project가 선택한 self-range snapshot contract에서 destination mutation 전 source values가 고정돼야 합니다.
- 재현하는 failure 또는 boundary: insertion source/destination이 같은 vector이고 source가 position 전후와 중첩되는 경우, interior self-range assignment입니다.
- test technique: expected sequence를 prefix/source snapshot/suffix로 직접 만드는 independent oracle입니다.
- 통과하는 production 코드 경로: `bdc4c3123bc9` 이후 range input snapshot과 `797c33904db3`의 range insertion implementation입니다.
- 이 테스트가 증명하는 것: 구현 결과가 명시한 snapshot semantics와 일치합니다.
- 이 테스트가 증명하지 않는 것: `std::vector`의 overlapping modifier 표준 의미나 모든 임의 중첩 조합을 증명하지 않습니다.
- 성격: 기존 regression test 자체의 oracle 결함을 고친 deterministic test correction입니다.
- 후속 변경에서 막아야 하는 회귀: expected 값을 구현과 같은 잘못된 modifier 호출로 계산해 양쪽이 함께 틀리거나 platform별 behavior에 의존하는 회귀입니다.
- 실행 증거: 수정된 expected-building code를 diff로 확인했으며 실행하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: production failure가 아니라 verification 신뢰성의 경계입니다. oracle이 정의되지 않은/지원 범위 밖 동작에 기대면 test pass가 근거가 되지 않습니다.
- state/ownership/lifecycle 영향: production state 변화는 없지만 self-aliasing contract의 기대 sequence가 독립적으로 고정됩니다.
- 이 commit이 보장하는 것과 남은 한계: 이 테스트의 expected 값이 project contract에서 직접 유도됩니다. 전체 standard conformance oracle은 아닙니다.
- 다음 관련 commit과의 연결: vector Thread의 최종 evidence에서 self-range 결과와 failure/lifetime evidence를 분리해 신뢰할 수 있게 합니다.

## 6. Invariant ledger

### Source에서 확정된 관련 invariant

- Every vector element in `[0, size)` is constructed exactly once and destroyed exactly once; storage beyond `size` remains uninitialized until explicitly constructed.
- Vector storage is deallocated by allocator state compatible with the state that allocated it. A failed allocation or construction must not leak a block or leave `size` claiming ownership of an unconstructed object.
- Reallocation commits only after the replacement block is complete. Where the supported contract requires preservation, failed construction leaves the original vector unchanged; in-place operations at minimum keep lifetime bookkeeping valid and the container destructible.
- Range assignment and insertion must snapshot self-referential input before mutating the source vector, so source iterators and aliased values are not invalidated before they are consumed.
- Requested vector sizes and growth calculations must respect `allocator::max_size()` without unsigned overflow or invalid capacity selection.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 commit | 부족함이 드러난 commit/상태 | 강화·복구한 fix | 고정한 test/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Every vector element in `[0, size)` is constructed exactly once and destroyed exactly once | `2eb9cb6c4273` | 초기 insert/general mutation이 live/raw slot을 일관되게 구분하지 않음 | `b3124b3808d5`, 결정적으로 `797c33904db3` | `9051be26db5e`, `8df3d8e067c0` | state fields, construction progress, tail cleanup, `_size` commit |
| Vector storage is deallocated by allocator state compatible with the state that allocated it | `2eb9cb6c4273` | replacement/temporary 실패에서 block cleanup을 검증할 evidence 부족 | `9db46550b13d`, `b3124b3808d5`, `797c33904db3` | `0ce21f9cf12d`, `9051be26db5e`, `8df3d8e067c0` | `_alloc.allocate/deallocate`, tracking allocator block counter |
| Reallocation commits only after the replacement block is complete | `9db46550b13d` | assignment/resize/push/insert에 같은 transaction이 일관되게 적용되지 않음 | `b3124b3808d5`, `797c33904db3` | `9051be26db5e`, `8df3d8e067c0` | new block prefix/insert/suffix construction 후 `_replace_storage` |
| Range assignment and insertion must snapshot self-referential input before mutating the source vector | `bdc4c3123bc9` | 초기 regression oracle가 reference container의 overlapping modifier에 의존 | production은 `797c33904db3`에서 유지, oracle은 `5bdb6eb81a89`에서 교정 | `61e8b46e668f`, `5bdb6eb81a89` | temporary source capture, explicit prefix/source/suffix expected values |
| Requested vector sizes and growth calculations must respect `allocator::max_size()` without unsigned overflow | `6f3cbf4794c9` | `9db46550b13d`의 doubling-first arithmetic | `6f3cbf4794c9` | `0ce21f9cf12d` 및 `9051be26db5e` bounded allocator | `limit - _capacity` 포화 분기와 length preflight |

## 7. Failure → Fix → Test 연결

| 기존 상태/production change | fix 또는 verification | Source에서 확정된 연결 관점 | 실제 failure/root cause | 실제 test production path |
| --- | --- | --- | --- | --- |
| `6f3cbf4794c9` | `0ce21f9cf12d` | bounded allocator로 capacity saturation/length rejection/block cleanup 검증 | doubling overflow와 limit 초과 요청이 mutation 전 분리되지 않을 위험 | bounded `max_size` → growth/reserve preflight → final block counter |
| `bdc4c3123bc9` | `61e8b46e668f` | self-range insert/assign 결과 회귀 검증 | mutation 중 source iterator/value invalidation | range snapshot → assign/insert result comparison |
| `bc3a74b9342e` | `ccb98587e777` | null-storage no-op modifier와 allocator 상태 검증 | null pointer addition/subtraction | empty begin/end, zero insert, empty erase no-op branch |
| `b3124b3808d5` | `9051be26db5e` | construction/assignment/resize/aliased push-back failure injection | state 조기 publish와 incomplete cleanup | tracked copy/assign/alloc throw → rollback → value/live/block assertions |
| `797c33904db3` | `8df3d8e067c0` | insert copy/assignment/allocation failure sweep | live/raw slot 혼동과 partial tail leak | reallocation/in-place fill/range helpers → fail-at sweep |
| `bdc4c3123bc9 → 61e8b46e668f` | `5bdb6eb81a89` | self-range oracle를 explicit snapshot expected value로 교정 | oracle가 overlapping reference modifier에 의존 | explicit expected assembly → production self-range result comparison |

- `0ce21f9cf12d`, `61e8b46e668f`, `ccb98587e777`는 이 Development Thread의 commit map에는 포함되지 않지만, source가 확정한 관련 regression commit입니다. Thread membership을 변경하지 않고 fix의 검증 연결만 기록합니다.

## 8. Ownership / state / responsibility 변화

| 시점 | Owner / state / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `2eb9cb6c4273` | Establishes allocator-backed contiguous storage and the constructed-object ownership boundary. | vector state 부재 | `_data` block과 `_size` live prefix 분리 | vector fields, constructor/destructor |
| `9db46550b13d` | Adds replacement-block reallocation and geometric capacity growth. | fixed capacity | new block 임시 소유 후 성공 시 transfer | `_reallocate`, `reserve` |
| `6f3cbf4794c9` | Corrects growth arithmetic at the allocator limit. | overflow 가능한 doubling | limit-safe saturated capacity | `_next_capacity`, preflight |
| `bdc4c3123bc9` | Snapshots self-range input before mutation invalidates it. | source와 destination mutation 결합 | temporary가 input lifetime 소유 | range assign/insert snapshot |
| `bc3a74b9342e` | Removes empty-storage pointer arithmetic and adopts allocator-provided size types. | null pointer 산술 가능 | empty branch와 allocator typedefs | `_iterator_at`, `_index_of` |
| `b3124b3808d5` | Makes construction, assignment, resize growth, and aliased push-back transactional. | destination 조기 mutation | temporary/new suffix가 commit 전 책임 | `_initialize_fill`, `_swap_storage`, resize/push catch |
| `9051be26db5e` | Injects element and allocation failures to validate rollback and live-object counts. | failure behavior 추론 | test state가 objects/blocks를 계측 | `tracked_value`, tracking allocator |
| `797c33904db3` | Rebuilds fill/range insertion around explicit live-object and raw-storage paths. | insertion shift가 lifetime 영역 혼동 | reallocate/in-place helper가 각 영역 책임 | insertion helpers, `constructed` counter |
| `8df3d8e067c0` | Sweeps insertion copy, assignment, and allocation failures across both capacity branches. | 일부 normal-path evidence | branch/failure별 regression matrix | fail-at loops, capacity/address/lifetime assertions |
| `5bdb6eb81a89` | Replaces a questionable reference modifier with an explicit snapshot-derived test oracle. | oracle가 다른 modifier behavior에 의존 | test가 expected sequence를 직접 소유 | explicit expected assembly |

## 9. Thread 최종 상태

- 최종적으로 성립한 representation/state: vector는 allocator, contiguous block pointer, live prefix count, raw slot capacity를 소유합니다. 모든 modifier는 `[0,size)`와 `[size,capacity)`의 연산 규칙을 구분합니다.
- 최종적으로 보장하는 invariant: construct 성공한 object만 size에 포함되고 같은 allocator family가 block을 해제합니다. reallocation은 replacement 완성 후 commit하며, self-referential input은 mutation 전에 snapshot됩니다. growth는 `max_size()`를 overflow 없이 존중합니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: in-place insertion 중 element assignment가 던지면 값의 strong guarantee는 없습니다. 대신 newly constructed tail을 정리하고 old size를 유지하는 basic lifetime guarantee입니다. invalid iterator와 throwing destructor는 지원 범위 밖입니다.
- 최종 verification evidence: tracked object/allocator failure injection, insertion sweep, bounded allocator, empty-state regression, independent self-range oracle를 코드로 확인했습니다. 이 작업에서는 test executable을 실제 실행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point commit: representation을 만든 `2eb9cb6c4273`, 일반 transaction을 만든 `b3124b3808d5`, insertion lifetime을 결정적으로 고친 `797c33904db3`입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 단계명은 source가 정의한 Thread progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 commit | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | failure/cleanup | 다음 단계에 남기는 invariant |
| --- | --- | --- | --- | --- | --- | --- |
| Allocation ownership model | `2eb9cb6c4273` | vector fields/construct/destruct | allocator와 element count | raw block allocate 후 성공 element만 size 증가 | completed prefix destroy + block deallocate | size == live count |
| Replacement-block reallocation | `9db46550b13d` | `reserve`, `_reallocate` | old live block, larger capacity | new block copy 완료 후 owner 교체 | new prefix rollback, old unchanged | reallocation commit boundary |
| Allocator-limit growth | `6f3cbf4794c9` | `_next_capacity`, preflights | requested minimum/current capacity/limit | overflow 없는 saturation 및 reject | mutation 전 `length_error` | capacity <= max_size |
| Aliased range capture | `bdc4c3123bc9` | range assign/insert | source가 destination일 수 있음 | temporary에 source와 필요 tail capture | temporary construction failure 시 destination unchanged | mutation 전 input 고정 |
| Empty-storage boundary | `bc3a74b9342e` | `_iterator_at`, `_index_of` | `_data == NULL`, size 0 | pointer 산술 없이 empty 결과 반환 | no-op | empty begin/end/modifier 유효 |
| Transactional general mutation | `b3124b3808d5` | init/assign/resize/push | replacement 또는 suffix 필요 | prepare → construct → publish | temporary/suffix rollback | 일반 mutation의 commit-after-success |
| Failure injection | `9051be26db5e` | vector exception tests | 선택된 fail attempt | production 예외 경로 실행 및 counters 관찰 | scope 종료 후 live/block zero | rollback evidence |
| Insertion lifetime redesign | `797c33904db3` | insert helpers | position/count/range/capacity | reallocate 또는 raw-tail construct + live assignment | new block/tail cleanup, size 마지막 publish | insertion lifetime 유효 |
| Insertion failure sweep | `8df3d8e067c0` | insertion regressions | copy/assign/alloc fail index | 모든 주요 branch 반복 실행 | trackers와 post-state 검사 | 복잡한 modifier regression 고정 |
| Independent self-range oracle | `5bdb6eb81a89` | self-range expected builder | original values와 selected range | prefix + snapshot + suffix 직접 조립 | production mutation 사용 안 함 | test oracle 독립성 |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 source 순서대로 확인했습니다.
- [x] 각 commit 기록에 final HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A commit은 decision, failure boundary, ownership/state transition을 설명할 수 있습니다.
- [x] Test/perf commit은 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → failure/risk → root cause → 수정 → regression 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 commit history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] Thread 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 02-vector-ownership-aliasing-exception-safe-mutation.md =====

===== BEGIN FILE: 03-map-unbalanced-bst-to-verified-red-black-core.md =====
# Map from Unbalanced Search Tree to Verified Red-black Core

## 1. Thread 목표

### Source-established significance

The public map surface first exists as an ordinary BST, which is sufficient to establish comparator semantics and observable ordering but not the required worst-case complexity. Insertion balancing supplies the first red-black mechanism; deletion then adds the more difficult state restoration for removed black nodes and null replacements. The later tests intentionally move beyond output comparison: a white-box inspector proves parent links, header state, red constraints, black height, and node reachability, while complexity tests prove that the structure remains logarithmic under ascending, descending, and shuffled input. The sequence distinguishes functional ordering from structural correctness and asymptotic correctness.

### 이 Thread에서 복원할 것

- 위 significance가 설명하는 변화 과정을 각 commit의 실제 SHA 코드로 재구성합니다.
- source가 확정한 commit 역할과 importance를 바꾸지 않고, 실제 implementation/failure/test 근거만 직접 채웁니다.

### Source에서 직접 연결되는 architecture

- `ft::map` stores values in allocator-created nodes linked as a red-black tree. A value-free header sentinel owns the root, minimum, and maximum links and is also the stable representation of `end()`.

### Source에서 직접 연결되는 Major Engineering Difficulties

- Implementing red-black insertion and deletion, especially deletion where a removed black node may leave a null replacement and the algorithm must carry explicit parent context through symmetric sibling, recoloring, and rotation cases.
- Verifying properties that public sorted output cannot establish: red-black color rules, black height, parent links, header extrema, node reachability, iterator stability, and logarithmic structural bounds.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- ordinary BST가 public ordering은 만족하면서도 map의 asymptotic requirement를 위반할 수 있는 이유는 무엇인가?
- red-black insertion과 deletion이 각각 어떤 state를 보존/복구해야 하는가?
- null replacement가 있는 deletion에서 explicit parent context가 필요한 이유는 무엇인가?
- sorted output parity만으로 증명할 수 없는 structural property는 무엇이며 inspector가 무엇을 추가로 증명하는가?
- height/comparator upper bound가 timing benchmark보다 어떤 회귀를 안정적으로 잡는가?

## 3. 완료 기준

- A: 주요 subsystem/boundary/failure path/integration point를 실제 코드와 설계 판단으로 연결하고, 관련 regression 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- B: Thread 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- S: 핵심 architecture/invariant를 직전 상태 → failure 가능성 → 결정 → 실제 핵심 코드 → ownership/lifecycle/state transition → 후속 fix/test까지 코드 근거로 설명할 수 있어야 합니다.
- 모든 commit은 해당 SHA의 코드 또는 test/build diff를 근거로 기록합니다.
- Thread 최종 설명은 source 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 commit 간 변화로 재구성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| --- | --- | --- | --- | --- | --- |
| 1 | `2c6dd8acdd20` | `feat(map): 노드 소유권과 빈 tree 상태 구현` | A | RB_TREE, ALLOCATOR, ARCH | Establishes individually allocated nodes and single-root ownership. |
| 2 | `c0fdb8e3f84c` | `feat(map): 삽입과 첨자 및 복사 동작 구현` | B | RB_TREE | Adds comparator-defined BST insertion, indexing, and copying. |
| 3 | `0f70c1fcc520` | `feat(map): 삭제와 clear 및 swap 구현` | B | RB_TREE, ALLOCATOR | Adds ordinary BST erasure and tree exchange. |
| 4 | `f29fd8a91523` | `feat(map): 레드-블랙 삽입 회전과 색 보정 구현` | S | CORE, RB_TREE, HARD | Introduces rotations and insertion recoloring. |
| 5 | `8f8b67961819` | `test(map): 정렬 입력 삽입과 검색 경계 stress 검증` | B | TEST, RB_TREE | Exercises adversarial sorted insertion and query boundaries. |
| 6 | `a055cb19500b` | `feat(map): 레드-블랙 삭제 보정 구현` | S | CORE, RB_TREE, HARD | Adds deletion transplant state and double-black correction. |
| 7 | `86922f1ddfa0` | `test(map): 반복 삭제·복사·대입·교환 stress 검증` | B | TEST, RB_TREE | Stresses repeated erasure, copying, assignment, and swap against `std::map`. |
| 8 | `cd67e6a31bb7` | `test(map): 무작위 연산마다 레드-블랙 불변식 검증` | A | TEST, RB_TREE, RISK | Validates all structural red-black invariants after deterministic random operations. |
| 9 | `cd8ebbb2c01e` | `perf(map): 높이와 비교 횟수 회귀 상한 추가` | A | PERF, RB_TREE, TEST | Enforces red-black height and comparator-count upper bounds. |

## 5. Commit별 학습 기록

### 1. feat(map): 노드 소유권과 빈 tree 상태 구현

- SHA: `2c6dd8acdd20`
- Importance: A
- Tags: RB_TREE, ALLOCATOR, ARCH
- Source-established role: Establishes individually allocated nodes and single-root ownership.
- Source summary: Introduces map's individually allocated nodes, parent/child links, null-root empty state, and recursive ownership cleanup.
- Source rationale: This is the associative container's foundational ownership substrate and enables stable value addresses. It is significant, but the final defining architecture arrives later with red-black balancing and a value-free header sentinel.

#### 해당 SHA에서 확인할 실제 코드

- map node의 parent/left/right link와 stored key/value의 위치, root/size empty state를 확인합니다.
- value allocator에서 node type으로 rebound된 allocator가 node allocate/construct/destroy/deallocate에 사용되는 경로를 추적합니다.
- node construction throw 시 allocation을 즉시 해제하는 failure branch와 recursive clear의 single-root ownership 경로를 확인합니다.
- 이 SHA의 tree가 아직 unbalanced이며 node allocator state construction은 후속 fix 대상이라는 source-defined 한계를 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 nested `node`, `_root`, `_size`, `_alloc`, `_node_alloc`, `_create_node`, `_destroy_node`, recursive `_clear`.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 map header가 없고 이 commit이 초기 ownership representation을 추가합니다. allocator state 문제는 `ae180871b160`에서 고칩니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: map object와 node ownership이 정의되지 않았습니다.
- 해결하려던 문제: key/value의 안정된 주소를 유지하면서 각 tree node를 독립적으로 생성·파괴하고, root 하나에서 전체 ownership을 도달 가능하게 해야 했습니다.
- 선택한 결정: node가 value와 parent/left/right link를 함께 보유하고 map이 `_root`와 `_size`를 소유합니다. value allocator를 node type으로 rebind해 node 단위로 allocate/construct합니다.
- 새로 생긴 책임 경계 또는 상태 변화: map의 root가 모든 reachable node의 단일 ownership entry가 되고, `_clear`가 post-order로 subtree를 해제합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: allocate 성공 뒤 node construction이 던지는 경우입니다. `_create_node`가 방금 받은 block을 즉시 deallocate하고 예외를 재전파합니다.
- state/ownership/lifecycle 영향: successful node만 tree에 연결되며 clear는 child subtree를 먼저 해제한 뒤 해당 node를 destroy/deallocate합니다.
- 이 commit이 보장하는 것과 남은 한계: 개별 node lifetime과 빈 `_root == NULL`, `_size == 0` 상태를 정의합니다. tree는 unbalanced이고 `_node_alloc`을 기본 생성해 stateful value allocator와 상태가 분리되는 결함이 있습니다.
- 다음 관련 commit과의 연결: `c0fdb8e3f84c`가 comparator search와 link를 추가하고, `f29fd8a91523`이 color/rotation을 추가합니다. allocator state는 Thread 05에서 교정됩니다.

### 2. feat(map): 삽입과 첨자 및 복사 동작 구현

- SHA: `c0fdb8e3f84c`
- Importance: B
- Tags: RB_TREE
- Source-established role: Adds comparator-defined BST insertion, indexing, and copying.
- Source summary: Adds unbalanced BST insertion, `operator[]`, range/copy construction, and copy assignment.
- Source rationale: These operations establish map's observable uniqueness and insertion semantics, but they operate within an unbalanced and weakly transactional design that later commits replace or harden.

#### 해당 SHA에서 확인할 실제 코드

- insertion search가 comparator로 왼쪽/오른쪽을 결정하고, 양방향 less가 모두 false일 때 duplicate로 판단하는 조건을 확인합니다.
- `operator[]`가 default mapped value를 만들어 insertion path를 재사용하는 호출 관계를 확인합니다.
- range/copy construction과 assignment가 visible element insertion을 어떻게 반복하는지 추적합니다.
- node allocation 뒤 parent side를 결정하기 위해 comparator를 다시 호출하는 위치를 표시하고, 후속 `cb08194d17b0`의 fix 대상임을 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `insert`, `operator[]`, range/copy constructor, `operator=`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `2c6dd8acdd20`의 node-only state; comparator-after-allocation fix `cb08194d17b0`.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: nodes를 만들고 지울 수는 있지만 key로 위치를 찾거나 public insertion/copy를 수행할 수 없었습니다.
- 해결하려던 문제: comparator만으로 strict ordering과 key equivalence를 정의하고 map API가 tree search를 재사용해야 했습니다.
- 선택한 결정: `_comp(value.first, current.first)`이면 left, 반대 비교면 right, 둘 다 false면 duplicate입니다. `operator[]`는 `(key, mapped_type())`를 insert하고 returned iterator의 second를 반환합니다.
- 새로 생긴 책임 경계 또는 상태 변화: comparator가 tree ordering/equivalence를 결정하고 insertion이 root 또는 parent child link와 `_size`를 갱신합니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: balanced 여부와 무관하게 map의 observable uniqueness, sorted traversal에 필요한 BST ordering을 확립합니다.
- 핵심 코드와 상태 변화: duplicate에서는 allocation 없이 기존 node를 반환하고, 새 key는 search parent 아래 link 후 size 증가입니다. range/copy는 같은 public insertion을 반복합니다.
- 다음 commit에 넘기는 전제: erase는 node identity와 parent/child links를 기반으로 transplant할 수 있습니다. 다만 allocation 뒤 comparator 재호출과 destructive assignment는 후속 exception fix 대상입니다.

### 3. feat(map): 삭제와 clear 및 swap 구현

- SHA: `0f70c1fcc520`
- Importance: B
- Tags: RB_TREE, ALLOCATOR
- Source-established role: Adds ordinary BST erasure and tree exchange.
- Source summary: Adds node erasure, range erasure, clear, and map swap for the initial BST.
- Source rationale: The commit completes normal mutation and ownership operations, but deletion is not yet red-black aware and swap is later revised for iterator and policy-exception correctness.

#### 해당 SHA에서 확인할 실제 코드

- 0/1-child erase의 transplant와 2-child erase의 in-order successor physical transplant 경로를 구분합니다.
- `pair<const Key, T>`의 immutable key 때문에 key assignment 대신 node transplant를 택한 코드 위치를 확인합니다.
- range erase에서 current node destroy 전에 successor iterator/node를 저장하는 순서를 확인합니다.
- `clear`와 `swap`이 root/size/allocator/node allocator/comparator 상태를 어떻게 이동시키는지 확인하고, deletion balancing은 아직 없음을 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 erase overloads, minimum/successor lookup, transplant helper, `clear`, member `swap`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `c0fdb8e3f84c`의 unbalanced insertion; RB deletion fix `a055cb19500b`.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: insert된 node를 개별/범위로 제거하거나 두 map의 tree ownership을 교환할 public mutation이 없었습니다.
- 해결하려던 문제: const key를 수정하지 않고 BST ordering을 유지한 채 0/1/2-child node를 제거해야 했습니다.
- 선택한 결정: target value를 successor value로 대입하지 않고 successor node 자체를 target 위치로 transplant합니다. range erase는 다음 위치를 먼저 확보한 뒤 현재 node를 지웁니다.
- 새로 생긴 책임 경계 또는 상태 변화: erase가 link repair, size 감소, node destroy/deallocate를 담당하고 swap이 전체 tree/policy/allocator state를 교환합니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: ordinary BST의 ownership mutation 표면을 완성합니다.
- 핵심 코드와 상태 변화: transplant는 parent의 child 또는 root를 교체하고 replacement parent를 고칩니다. 실제 removed node만 한 번 destroy/deallocate합니다.
- 다음 commit에 넘기는 전제: node physical movement 방식은 RB deletion에도 유지되지만 color와 black-height repair가 추가돼야 합니다.

### 4. feat(map): 레드-블랙 삽입 회전과 색 보정 구현

- SHA: `f29fd8a91523`
- Importance: S
- Tags: CORE, RB_TREE, HARD
- Source-established role: Introduces rotations and insertion recoloring.
- Source summary: Adds red/black node state, rotations, and insertion fix-up to keep the map balanced.
- Source rationale: This commit turns the map from a potentially linear BST into the project's defining ordered-container mechanism. The rotation and recoloring cases establish the core logarithmic structure and red-black invariants; omitting it would make the final architecture incomprehensible.

#### Source에서 확정된 핵심 판단

- Problem: The baseline map is an ordered BST, so ascending or descending insertion can make its height linear. Correct sorted iteration alone is insufficient because map's defining performance depends on maintaining a balanced search structure.
- Decision: Nodes gain red/black state. New non-root nodes begin red, the root remains black, and insertion repair handles red-uncles through recoloring and black-uncles through the appropriate single or double rotations. Left and right cases are implemented symmetrically.
- Why it mattered: This commit introduces the mechanism that makes map a red-black tree rather than merely a sorted linked structure. It establishes the logarithmic architecture that later deletion, invariant validation, and complexity tests must preserve.

#### 해당 SHA에서 확인할 실제 코드

- node color state가 추가된 지점과 new node/root의 초기 color 규칙을 확인합니다.
- left/right rotation이 parent-child links와 root를 갱신하는 모든 assignment를 순서대로 추적합니다.
- red parent에서 red uncle recolor case와 black uncle의 inner/outer rotation case를 좌우 대칭으로 표로 정리합니다.
- repair 종료 후 root black 강제가 어디서 수행되는지 확인하고, stored value를 이동하지 않고 links만 바꾸는지 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 node `red` field, left/right rotation helpers, insertion fix-up, `insert`의 fix-up 호출.
- 필요한 경우 비교할 직전 관련 SHA/parent: `c0fdb8e3f84c`의 ordinary BST insertion과 test `8f8b67961819`.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: comparator ordering은 맞지만 sorted input에서 높이가 node 수까지 증가하는 ordinary BST였습니다.
- 해결하려던 문제: insertion 후 red-red 위반을 제거하면서 in-order ordering, parent links, root ownership을 유지해야 했습니다.
- 선택한 결정: 새 node는 red로 link하고 parent가 red인 동안 uncle color에 따라 recolor 또는 rotation합니다. root는 항상 black으로 normalize합니다.
- 새로 생긴 책임 경계 또는 상태 변화: insertion은 단순 link 이후 color/shape repair 단계까지 책임집니다. rotations는 value/object ownership을 이동하지 않고 links만 재배열합니다.

#### S-level 심화 추적

- 기존 설계가 충분하지 않았던 이유: ascending/descending input에서는 모든 node가 한쪽 child가 되어 search/insert가 O(n)입니다. sorted output만으로 이 문제는 보이지 않습니다.
- 핵심 state/ownership/lifetime transition: red leaf link → parent red 여부 검사 → red uncle이면 parent/uncle black, grandparent red 후 위로 진행 → black uncle이면 inner case를 먼저 rotate해 outer case로 만든 뒤 parent/grandparent recolor와 rotation → root black입니다.
- failure scenario별 cleanup/rollback: balancing 단계 자체는 allocation이나 user copy를 하지 않고 pointer/color만 바꿉니다. comparator와 node construction은 repair 전에 끝납니다. 따라서 fix-up 중 일반 exception cleanup 경로는 없습니다.
- 이 commit이 보장하는 것: insertion 뒤 root black, no red-red path, ordering/parent links를 유지하는 red-black insertion mechanism입니다.
- 이 commit이 아직 보장하지 않는 것: erase 후 black-height 복구, internal invariant white-box proof, explicit height bound는 아직 없습니다.
- 후속 fix/test와 연결되는 구조: `8f8b67961819`가 adversarial output/query를 검사하고, `a055cb19500b`가 deletion repair, `cd67e6a31bb7`/`cd8ebbb2c01e`가 structure/complexity를 직접 검증합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: 양방향 rotation의 root/parent/child assignment와 red-uncle vs inner/outer black-uncle cases입니다.

### 5. test(map): 정렬 입력 삽입과 검색 경계 stress 검증

- SHA: `8f8b67961819`
- Importance: B
- Tags: TEST, RB_TREE
- Source-established role: Exercises adversarial sorted insertion and query boundaries.
- Source summary: Stress-tests ascending and descending insertion plus query boundaries against `std::map`.
- Source rationale: The scenarios are well chosen to expose an ordinary BST's worst case and broaden functional confidence, but they do not directly validate height or every red-black invariant.

#### 해당 SHA에서 확인할 실제 코드

- ascending 96 keys와 descending 96 keys scenario가 각각 어떻게 구성되는지 확인합니다.
- 각 insertion 결과와 in-order sequence를 `std::map`과 비교하는 assertion을 찾습니다.
- present/absent query spread에서 `find`, bounds, `equal_range`, end parity를 확인합니다.
- 이 stress가 rotation/recoloring을 많이 유발하지만 직접 height/black-height proof는 아니라는 증명 범위를 기록합니다.
- 확인한 파일/심볼: map tests의 ascending/descending insertion scenario와 comparison/query helper.
- 필요한 경우 비교할 직전 관련 SHA/parent: production balancing `f29fd8a91523`.

#### Test/verification 학습 기록

- 대상 production invariant: adversarial insertion order에서도 map의 observable ordering, uniqueness, query boundary가 `std::map`과 일치해야 합니다.
- 재현하는 failure 또는 boundary: 1부터 96까지 정렬/역정렬 삽입, 존재/부재 key의 `find`, `lower_bound`, `upper_bound`, `equal_range`, `end`입니다.
- test technique: deterministic adversarial input을 사용한 differential integration test입니다.
- 통과하는 production 코드 경로: insert search/link/fix-up, in-order iterator, lookup/bounds입니다.
- 이 테스트가 증명하는 것: 많은 rotation/recolor가 필요한 입력에서도 public contents와 query 결과가 reference와 일치합니다.
- 이 테스트가 증명하지 않는 것: 실제 height, root color, red adjacency, black height, parent-link consistency는 직접 보지 않습니다.
- 성격: broad functional stress입니다. structure-specific proof는 아닙니다.
- 후속 변경에서 막아야 하는 회귀: sorted input에서 값 누락/중복, ordering 깨짐, lookup boundary 오류입니다.
- 실행 증거: test code 검사만 수행했으며 binary 실행 결과는 없습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: insertion balancing 직후 public behavior가 유지되는지 확인합니다.
- 핵심 코드와 상태 변화: test가 두 map에 같은 insertion/query를 적용하고 결과만 비교합니다.
- 다음 commit에 넘기는 전제: erase balancing은 별도 mechanism이며 이 insertion stress로 검증되지 않습니다.

### 6. feat(map): 레드-블랙 삭제 보정 구현

- SHA: `a055cb19500b`
- Importance: S
- Tags: CORE, RB_TREE, HARD
- Source-established role: Adds deletion transplant state and double-black correction.
- Source summary: Implements red-black deletion transplant bookkeeping and symmetric double-black fix-up.
- Source rationale: Deletion is the most intricate core state transition in the tree: removed color, replacement child, parent context, sibling cases, rotations, and recoloring must remain coherent even when the child is null. This is indispensable to the map's correctness story.

#### Source에서 확정된 핵심 판단

- Problem: Deleting a node from a red-black tree can reduce the black count on one root-to-leaf path. The replacement may be null, so the repair cannot rely on a normal node object to carry color and parent context. Incorrect handling can silently preserve sorted output while corrupting black height or future rotations.
- Decision: Erasure records the node physically moved, its original color, the replacement child, and an explicit replacement parent. If a black node was removed, a symmetric double-black repair examines sibling color and near/far child colors, performs recoloring or rotations, and finally forces the replacement and root black.
- Why it mattered: Deletion is a separate defining mechanism from insertion balancing and is substantially more failure-prone. This commit preserves the red-black invariants across all erase forms and enables repeated arbitrary deletion without degrading or corrupting the tree.

#### 해당 SHA에서 확인할 실제 코드

- erase에서 physically moved node, original color, replacement child, replacement parent를 어떤 local state로 보존하는지 확인합니다.
- replacement child가 null일 때도 parent context를 유지하여 fix-up을 계속하는 caller/callee 흐름을 추적합니다.
- red sibling, black sibling with two black children, near/far red child cases를 좌우 대칭으로 정리하고 각 case의 rotate/recolor 순서를 기록합니다.
- two-child deletion에서 successor node를 transplant하고 target color를 복사하되 immutable value object는 유지하는지 확인합니다.
- repair 후 replacement/root black normalization이 수행되는 지점을 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 erase node path, `_transplant`, `_is_black`, deletion fix-up, rotations.
- 필요한 경우 비교할 직전 관련 SHA/parent: ordinary erase `0f70c1fcc520`, repeated deletion test `86922f1ddfa0`.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: insertion은 red-black이지만 erase는 ordinary BST transplant만 수행해 black-height와 color rules를 깨뜨릴 수 있었습니다.
- 해결하려던 문제: 실제 removed black contribution을 replacement path에 복구하고, replacement가 null이어도 sibling case를 계산해야 했습니다.
- 선택한 결정: `moved`, `moved_was_red`, `fix`, `fix_parent`를 별도로 추적합니다. removed effective node가 black일 때만 double-black repair를 호출합니다.
- 새로 생긴 책임 경계 또는 상태 변화: erase는 structural transplant와 color deficit repair를 분리하고, null child의 parent context는 local state가 소유합니다.

#### S-level 심화 추적

- 기존 설계가 충분하지 않았던 이유: black node 제거는 한쪽 root-to-null path의 black count를 줄입니다. sorted order와 node count는 맞아도 RB invariant가 무너질 수 있습니다.
- 핵심 state/ownership/lifetime transition: target 선택 → 0/1-child면 replacement와 parent 기록, 2-child면 successor를 physically transplant하고 successor의 original color/replacement 기록 → black 제거이면 fix loop → target node destroy/deallocate입니다.
- failure scenario별 cleanup/rollback: erase repair는 allocation/comparator/value assignment를 하지 않으므로 일반 exception branch는 없습니다. 핵심 위험은 wrong link/color transition이지 예외 rollback이 아닙니다. node는 link에서 완전히 분리된 뒤 정확히 한 번 해제됩니다.
- 이 commit이 보장하는 것: null replacement를 포함한 erase 후 red-black color/black-height 복구와 immutable key 보존입니다.
- 이 commit이 아직 보장하지 않는 것: public output tests만으로 모든 parent/header/black-height property를 입증하지는 않습니다. 그 evidence는 `cd67e6a31bb7`에서 추가됩니다.
- 후속 fix/test와 연결되는 구조: `86922f1ddfa0`이 반복 erase/copy/swap behavior를, `cd67e6a31bb7`이 매 operation structure를 검사합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: explicit `fix_parent`, symmetric sibling cases, successor color transfer, final black normalization입니다.

### 7. test(map): 반복 삭제·복사·대입·교환 stress 검증

- SHA: `86922f1ddfa0`
- Importance: B
- Tags: TEST, RB_TREE
- Source-established role: Stresses repeated erasure, copying, assignment, and swap against `std::map`.
- Source summary: Adds repeated key/iterator erasure and copy, assignment, and swap stress comparisons.
- Source rationale: The tests exercise broad map behavior after deletion balancing, but later invariant-aware randomized verification provides the stronger architectural evidence. This remains solid normal regression work.

#### 해당 SHA에서 확인할 실제 코드

- ascending/descending map의 copy, assignment, swap 시나리오와 standard counterpart 비교 지점을 확인합니다.
- mixed insertion sequence를 repeated key erase로 줄이는 흐름과 min/max iterator erase를 번갈아 수행하는 흐름을 확인합니다.
- 각 단계에서 contents/boundary queries가 검증되는지 확인하고, 이 test가 internal color/black-height를 직접 보는지 여부를 구분합니다.
- 확인한 파일/심볼: map test source의 copy/assignment/swap, repeated key erase, alternating begin/`--end()` erase scenarios.
- 필요한 경우 비교할 직전 관련 SHA/parent: production deletion repair `a055cb19500b`.

#### Test/verification 학습 기록

- 대상 production invariant: repeated erase와 ownership-moving operations 뒤에도 contents, ordering, query results가 reference와 일치해야 합니다.
- 재현하는 failure 또는 boundary: mixed 32-key tree에서 every-third key erase, 이후 min/max 교대 erase로 empty; ascending/descending copy, assignment, swap입니다.
- test technique: deterministic differential stress against `std::map`입니다.
- 통과하는 production 코드 경로: key/iterator erase, deletion fix-up, copy constructor, assignment, swap, iterator traversal, bounds입니다.
- 이 테스트가 증명하는 것: 반복 mutation과 map-to-map operation 후 observable state가 일치하고 끝까지 empty로 정리됩니다.
- 이 테스트가 증명하지 않는 것: internal red/black color와 black height, unreachable node, stale header를 직접 검사하지 않습니다.
- 성격: broad normal-path regression입니다.
- 후속 변경에서 막아야 하는 회귀: deletion sequence 특정 case에서 key 누락/중복, iterator traversal 깨짐, copy/swap contents 오류입니다.
- 실행 증거: test implementation을 검사했으며 실행하지 않았습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: deletion mechanism을 다양한 tree shape와 public ownership operation에서 압박합니다.
- 핵심 코드와 상태 변화: reference/ft map을 같은 sequence로 변경하고 단계별 parity를 확인합니다.
- 다음 commit에 넘기는 전제: public parity가 통과해도 latent structural corruption이 남을 수 있으므로 inspector가 필요합니다.

### 8. test(map): 무작위 연산마다 레드-블랙 불변식 검증

- SHA: `cd67e6a31bb7`
- Importance: A
- Tags: TEST, RB_TREE, RISK
- Source-established role: Validates all structural red-black invariants after deterministic random operations.
- Source summary: Adds a constrained white-box inspector and deterministic differential/randomized validation of every red-black invariant after structural operations.
- Source rationale: This materially changes confidence in the core tree: public sorted output alone cannot prove parent links, header extrema, red constraints, black height, or reachable-node counts. Reproducible operation logs make failures diagnosable.

#### Source에서 확정된 핵심 판단

- Problem: Sorted output and parity with `std::map` cannot prove that the internal tree is valid. Parent links may be inconsistent, header extrema stale, red nodes adjacent, black heights unequal, or unreachable nodes excluded from iteration until a later operation exposes the corruption.
- Decision: A narrowly scoped friend inspector validates header state, root linkage and color, extrema, strict subtree bounds, parent/child consistency, red-child rules, equal black height, and reachable-node count. Fixed deletion sequences, repeated current-root deletion, and 3,000 deterministic mixed operations validate both primary and secondary maps after every step.
- Why it mattered: This test changes the evidence standard for the project from output equivalence to representation correctness. The fixed seed, step number, and operation prefix also make deep balancing regressions reproducible rather than intermittent.

#### 해당 SHA에서 확인할 실제 코드

- test-only inspector가 header marker/color, empty extrema, root parent/color, cached extrema, parent-child consistency, strict subtree bounds, red-child rule, black height, reachable count를 각각 검사하는 코드를 찾습니다.
- fixed structural sequences, current-root repeated deletion, 3,000 deterministic pseudo-random operations의 실행 경로를 구분합니다.
- random stream의 operation 종류와 primary/secondary map 모두를 매 step 검증하는 위치를 확인합니다.
- fixed seed, current step, operation prefix가 failure reproduction에 어떻게 출력되는지 확인합니다.
- 확인한 파일/심볼: `tests/support/map_inspector.hpp`의 inspector와 `tests/test_map_randomized.cpp`의 fixed/random scenarios, seed `0x5EED1234`.
- 필요한 경우 비교할 직전 관련 SHA/parent: RB insertion/deletion commits와 sentinel commit `15a8460ccdfe`의 header representation.

#### Test/verification 학습 기록

- 대상 production invariant: strict key ordering, black root/header, no red-red, equal black height, bidirectional parent links, correct header root/min/max, reachable node count == size입니다.
- 재현하는 failure 또는 boundary: fixed difficult deletion shapes, repeatedly deleting current root, insert/erase/find/copy/assignment/swap 계열의 3,000-step deterministic mixed operations입니다.
- test technique: constrained white-box structural validation + deterministic randomized differential test입니다.
- 통과하는 production 코드 경로: insertion/deletion rotations and fix-ups, header refresh, clear/copy/assignment/swap, lookup and iterator traversal입니다.
- 이 테스트가 증명하는 것: 검사한 every step에서 tree representation 전체가 명시한 invariants와 public reference state를 동시에 만족합니다.
- 이 테스트가 증명하지 않는 것: 모든 possible operation sequence를 exhaustive하게 탐색하거나 wall-clock complexity, exception safety를 증명하지 않습니다.
- 성격: 고정 seed와 operation prefix를 갖는 reproducible structural regression입니다.
- 후속 변경에서 막아야 하는 회귀: public output에는 즉시 안 보이는 stale parent/extrema, red-red, unequal black height, unreachable/extra node입니다.
- 실행 증거: inspector와 deterministic driver 코드를 검사했으나 3,000 operations를 실제 실행하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: null leaves의 black-height base case, empty header self-reference, root parent가 header인 non-empty case, swap 후 두 header 갱신입니다.
- state/ownership/lifecycle 영향: reachable node count가 size와 같아야 하므로 detached leak 또는 duplicate reachability를 구조 수준에서 탐지합니다. allocator block 자체는 별도 test 대상입니다.
- 이 commit이 보장하는 것과 남은 한계: deterministic sequences에 대한 강한 structure evidence입니다. exception injection과 asymptotic upper bound는 각각 다른 test가 필요합니다.
- 다음 관련 commit과의 연결: `cd8ebbb2c01e`가 same inspector의 height 접근과 counting comparator로 complexity claim을 executable bound로 만듭니다.

### 9. perf(map): 높이와 비교 횟수 회귀 상한 추가

- SHA: `cd8ebbb2c01e`
- Importance: A
- Tags: PERF, RB_TREE, TEST
- Source-established role: Enforces red-black height and comparator-count upper bounds.
- Source summary: Adds executable red-black height and comparator-count upper bounds for adversarial and shuffled insertion orders.
- Source rationale: The test converts the map's logarithmic complexity claim into deterministic structural and operation-count limits. It can detect disabled balancing or hidden linear searches that functional tests would miss.

#### 해당 SHA에서 확인할 실제 코드

- counting comparator가 insertion/search comparison count를 어떤 state에 누적하는지 확인합니다.
- inspector height 측정과 1,024 ascending/descending/shuffled scenario 구성을 확인합니다.
- height upper bound `2 * ceil(log2(n + 1))`와 conservative `n log n` insertion comparison bound를 실제 test 식에서 확인합니다.
- successful/unsuccessful `find`가 measured height의 constant multiple 안에 있어야 하는 assertion을 확인하고, wall-clock timing을 쓰지 않는 이유를 기록합니다.
- 확인한 파일/심볼: `tests/test_complexity.cpp`의 `counting_less`, `ceil_log2`, scenario builder, height/comparison assertions; inspector height helper.
- 필요한 경우 비교할 직전 관련 SHA/parent: `cd67e6a31bb7`의 white-box inspector와 RB production core.

#### Test/verification 학습 기록

- 대상 production invariant: insertion order와 무관하게 red-black height가 logarithmic bound 안이고 search comparator count가 height의 constant multiple 안이어야 합니다.
- 재현하는 failure 또는 boundary: 1,024 ascending, descending, fixed shuffle insertion과 present/absent find입니다.
- test technique: structural bound + operation-count bound입니다. wall-clock benchmark가 아닙니다.
- 통과하는 production 코드 경로: comparator-driven insert search, insertion fix-up/rotations, find search, inspector height traversal입니다.
- 이 테스트가 증명하는 것: test scenarios에서 height가 `2 * ceil(log2(n + 1))` 이하이고 insertion/search comparison count가 설정된 conservative upper bound 안입니다.
- 이 테스트가 증명하지 않는 것: 실제 latency, cache behavior, allocator cost, 모든 n/ordering을 측정하지 않습니다.
- 성격: deterministic asymptotic regression입니다.
- 후속 변경에서 막아야 하는 회귀: balancing 호출 제거, root/link bug로 height 폭증, hidden linear scan 또는 redundant comparator explosion입니다.
- 실행 증거: bound 식과 scenario code를 검사했으며 perf test를 실행하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: ordinary BST여도 functional tests가 통과할 수 있는 sorted input이 핵심 adversarial boundary입니다.
- state/ownership/lifecycle 영향: production ownership을 바꾸지 않으며 height와 comparator state만 관찰합니다.
- 이 commit이 보장하는 것과 남은 한계: deterministic structural/operation complexity evidence를 제공합니다. timing performance를 보장하지 않습니다.
- 다음 관련 commit과의 연결: 이 Thread의 최종 red-black architecture가 correctness와 asymptotic evidence를 모두 갖추게 합니다.

## 6. Invariant ledger

### Source에서 확정된 관련 invariant

- Map keys remain strictly ordered according to the comparator; the root is black; red nodes have no red children; every null-leaf path has the same black height; parent/child links agree; and the number of reachable value nodes equals `size()`.
- The map header sentinel is black and value-free. For an empty map it self-references as both extrema; for a non-empty map it points to the current root, minimum, and maximum, and the root points back to the header.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 commit | 부족함이 드러난 commit/상태 | 강화·복구한 fix | 고정한 test/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Map keys remain strictly ordered; root black; no red-red; equal black height; consistent links; reachable count == size | ordering은 `c0fdb8e3f84c`, insertion RB rules는 `f29fd8a91523` | ordinary erase `0f70c1fcc520`은 black-height를 복구하지 않음; public parity만으로 internal corruption을 못 봄 | deletion rules `a055cb19500b` | `8f8b67961819`, `86922f1ddfa0`, 결정적으로 `cd67e6a31bb7`, `cd8ebbb2c01e` | comparator search, rotations/fix-ups, inspector recursive validation, height/count bounds |
| The map header sentinel is black and value-free; root/min/max links remain current | 이 Thread 밖의 관련 commit `15a8460ccdfe` | 이전 root-snapshot iterator/NULL-root representation은 stable end와 cached extrema를 통합하지 못함 | `15a8460ccdfe` | `81d8c4489c16`, 그리고 `cd67e6a31bb7` inspector | `node_base` header, `_refresh_header`, header/root/extrema checks |

## 7. Failure → Fix → Test 연결

| 기존 상태/production change | fix 또는 verification | Source에서 확정된 연결 관점 | 실제 failure/root cause | 실제 test production path |
| --- | --- | --- | --- | --- |
| `f29fd8a91523` | `8f8b67961819` | adversarial sorted insertion과 query boundary stress | ordinary BST worst case와 rotation/recolor functional regression 위험 | sorted insert → fix-up → traversal/find/bounds differential |
| `a055cb19500b` | `86922f1ddfa0` | repeated erase/copy/assignment/swap stress | null replacement와 varying sibling case에서 observable state corruption 위험 | repeated key/iterator erase → deletion fix → parity |
| `red-black structure` | `cd67e6a31bb7` | white-box invariant validation after every operation | sorted output가 stale links/color/black height/unreachable nodes를 숨김 | every operation → inspector full structure + std parity |
| `red-black complexity` | `cd8ebbb2c01e` | height/comparator-count deterministic upper bounds | balancing 비활성화나 hidden linear search가 normal tests를 통과할 수 있음 | adversarial/shuffle insert → height/count assertions; find count |

## 8. Ownership / state / responsibility 변화

| 시점 | Owner / state / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `2c6dd8acdd20` | Establishes individually allocated nodes and single-root ownership. | map/node ownership 부재 | root에서 모든 individually allocated node 도달 | node links, `_root`, `_clear` |
| `f29fd8a91523` | Introduces rotations and insertion recoloring. | unbalanced shape | color와 rotations가 logarithmic shape 책임 | node `red`, rotations, insertion fix-up |
| `a055cb19500b` | Adds deletion transplant state and double-black correction. | ordinary transplant만 수행 | erase local state가 removed color/null parent context 소유 | moved/fix/fix_parent, deletion fix-up |
| `cd67e6a31bb7` | Validates all structural red-black invariants after deterministic random operations. | public results만 관찰 | inspector가 representation correctness 관찰 | `map_inspector.hpp`, randomized driver |
| `cd8ebbb2c01e` | Enforces red-black height and comparator-count upper bounds. | complexity claim이 암묵적 | test가 height/comparison upper bound 소유 | `test_complexity.cpp` |

## 9. Thread 최종 상태

- 최종적으로 성립한 representation/state: allocator-created value nodes가 parent/left/right/color link로 red-black tree를 이루며, 관련 sentinel commit 이후 embedded value-free header가 root/min/max와 `end()`를 나타냅니다.
- 최종적으로 보장하는 invariant: comparator strict ordering, black root/header, red node의 black children, 모든 null path의 같은 black height, 양방향 parent/child 일치, reachable node count와 size 일치, logarithmic structural height입니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: comparator는 tree lifetime 동안 ordering semantics를 일관되게 유지해야 합니다. 이 Thread의 structural/perf tests는 exception safety와 allocator state compatibility를 대신하지 않습니다.
- 최종 verification evidence: sorted/differential stress, repeated deletion stress, every-step white-box deterministic random validation, height/comparator-count upper bounds를 코드로 확인했습니다. 실제 test command는 실행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point commit: insertion balancing `f29fd8a91523`, deletion balancing `a055cb19500b`, evidence 기준을 바꾼 `cd67e6a31bb7`입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 단계명은 source가 정의한 Thread progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 commit | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | failure/cleanup | 다음 단계에 남기는 invariant |
| --- | --- | --- | --- | --- | --- | --- |
| Node ownership | `2c6dd8acdd20` | node/create/destroy/clear | key/value와 empty root | node allocate/construct, root 도달성 | construct failure deallocate; post-order clear | individually owned nodes |
| Unbalanced BST insertion | `c0fdb8e3f84c` | `insert`, `operator[]` | key/comparator, root | compare search → duplicate 또는 parent child link | node construction rollback; later comparator risk 남음 | strict ordering/uniqueness |
| Unbalanced erase/swap | `0f70c1fcc520` | erase/transplant/clear/swap | target node | 0/1-child replace 또는 successor transplant | removed node one-time destroy | immutable key 유지, balancing 미완성 |
| RB insertion repair | `f29fd8a91523` | rotations/fix-up | red inserted node | recolor 또는 inner/outer rotations | allocation 없음 | insertion 후 RB rules |
| Adversarial insertion stress | `8f8b67961819` | sorted scenarios | 96 ordered keys | ft/std insert와 query 비교 | assertion failure | public insertion/query parity |
| RB deletion repair | `a055cb19500b` | erase + double-black fix | target, moved color, replacement/parent | sibling cases로 black deficit 이동/해소 | link repair 후 node release | erase 후 RB rules |
| Deletion/lifecycle stress | `86922f1ddfa0` | repeated erase/copy/swap tests | mixed trees | key/iterator erase와 ownership operations 반복 | test assertion | observable mutation parity |
| White-box invariant validation | `cd67e6a31bb7` | inspector/random driver | each operation result | header/root/subtree/color/black height/count recursive 검사 | seed/step/prefix로 failure 진단 | representation correctness evidence |
| Structural complexity limits | `cd8ebbb2c01e` | complexity test | 1,024 ordered/shuffled keys | height와 comparator calls 측정 | deterministic bound assertion | logarithmic regression boundary |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 source 순서대로 확인했습니다.
- [x] 각 commit 기록에 final HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A commit은 decision, failure boundary, ownership/state transition을 설명할 수 있습니다.
- [x] Test/perf commit은 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → failure/risk → root cause → 수정 → regression 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 commit history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] Thread 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 03-map-unbalanced-bst-to-verified-red-black-core.md =====

===== BEGIN FILE: 04-stable-map-iterators-through-structural-mutation.md =====
# Stable Map Iterators through Structural Mutation

## 1. Thread 목표

### Source-established significance

The initial iterator can traverse a static tree, but `--end()` depends on a root pointer copied into the iterator. Rotations, root erasure, and swap make that snapshot stale even though element nodes remain valid. The header-sentinel refactor moves end-state and extrema ownership into the container, lets nodes navigate to the current end through parent links, and removes any need to construct a key merely to represent the sentinel. The follow-up tests target exactly the structural operations that invalidate the earlier model, making this a clear limitation-to-architecture-correction progression.

### 이 Thread에서 복원할 것

- 위 significance가 설명하는 변화 과정을 각 commit의 실제 SHA 코드로 재구성합니다.
- source가 확정한 commit 역할과 importance를 바꾸지 않고, 실제 implementation/failure/test 근거만 직접 채웁니다.

### Source에서 직접 연결되는 architecture

- `ft::map` stores values in allocator-created nodes linked as a red-black tree. A value-free header sentinel owns the root, minimum, and maximum links and is also the stable representation of `end()`.

### Source에서 직접 연결되는 Major Engineering Difficulties

- Replacing a root-carrying/null-end iterator design with a value-free header sentinel that remains correct through rotations, root replacement, erasure, clear, and swap and does not require a default-constructible key.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- static tree에서는 동작하던 root-snapshot iterator가 rotation/root erase/swap에서 왜 실패하는가?
- value-free header sentinel이 `end()`, extrema, empty state, non-default key 문제를 한 번에 어떻게 해결하는가?
- element iterator identity와 container-owned end sentinel 책임은 어떻게 분리되는가?
- swap 뒤 기존 element iterator가 새 owner의 end에 도달하려면 어떤 parent/header 연결이 필요한가?

## 3. 완료 기준

- B: Thread 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- S: 핵심 architecture/invariant를 직전 상태 → failure 가능성 → 결정 → 실제 핵심 코드 → ownership/lifecycle/state transition → 후속 fix/test까지 코드 근거로 설명할 수 있어야 합니다.
- A: 주요 subsystem/boundary/failure path/integration point를 실제 코드와 설계 판단으로 연결하고, 관련 regression 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- 모든 commit은 해당 SHA의 코드 또는 test/build diff를 근거로 기록합니다.
- Thread 최종 설명은 source 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 commit 간 변화로 재구성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| --- | --- | --- | --- | --- | --- |
| 1 | `4bbf81cecef4` | `feat(map): 가변 반복자와 tree 순회 구현` | B | RB_TREE, ITERATOR | Implements traversal with a null end state plus a copied root pointer. |
| 2 | `50e62b0e0298` | `feat(map): 상수와 역방향 반복자 구현` | B | RB_TREE, ITERATOR | Extends that model to const and reverse iteration. |
| 3 | `048ada8fe1c5` | `feat(map): 가변·상수 반복자 상호 비교 지원` | B | ITERATOR, PUBLIC_API | Adds mutable/const comparison interoperability. |
| 4 | `15a8460ccdfe` | `fix(map): 값 없는 header로 끝 반복자 상태 안정화` | S | ARCH, ITERATOR, RB_TREE | Replaces the root snapshot with a value-free, container-owned header sentinel. |
| 5 | `81d8c4489c16` | `test(map): 회전·삭제·교환 뒤 반복자 상태 검증` | A | TEST, ITERATOR, RB_TREE | Verifies saved end positions, element iterators, swap migration, empty reset, and non-default keys. |

## 5. Commit별 학습 기록

### 1. feat(map): 가변 반복자와 tree 순회 구현

- SHA: `4bbf81cecef4`
- Importance: B
- Tags: RB_TREE, ITERATOR
- Source-established role: Implements traversal with a null end state plus a copied root pointer.
- Source summary: Adds mutable bidirectional tree traversal using successor/predecessor navigation and a root-bearing end iterator.
- Source rationale: The commit provides necessary ordered traversal, but the root snapshot coupled into each iterator is later shown to be structurally insufficient. It is an intermediate implementation rather than the final iterator model.

#### 해당 SHA에서 확인할 실제 코드

- mutable iterator가 current node 외에 root snapshot을 어떤 필드로 보관하는지 확인합니다.
- successor는 right-subtree minimum 또는 parent climb, predecessor는 대칭 경로를 사용하는지 추적합니다.
- `begin()`의 minimum 선택과 null `end()` 표현, `--end()`가 copied root에서 maximum을 찾는 경로를 확인합니다.
- root가 rotation/replacement/swap으로 바뀌었을 때 saved iterator의 root snapshot이 왜 stale해질 수 있는지 이 SHA의 state만 근거로 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 mutable `iterator`, iterator fields `_node`/`_root`, successor/predecessor helpers, `begin`, `end`.
- 필요한 경우 비교할 직전 관련 SHA/parent: iterator가 없던 직전 map 상태; limitation fix `15a8460ccdfe`.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: tree ordering과 mutation은 있어도 public iterator로 in-order traversal할 수 없었습니다.
- 해결하려던 문제: 현재 node에서 다음/이전 key로 이동하고 null end에서 현재 최대 node로 되돌아갈 정보가 필요했습니다.
- 선택한 결정: iterator가 current node와 생성 시점의 root pointer를 함께 저장합니다. current가 null인 `end()`에서 `--`하면 copied root의 maximum을 찾습니다.
- 새로 생긴 책임 경계 또는 상태 변화: element traversal topology 일부를 iterator가 snapshot으로 소유합니다. element나 tree lifetime ownership은 갖지 않습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: static tree에서 bidirectional ordered traversal의 첫 public model입니다.
- 핵심 코드와 상태 변화: `++`는 right subtree minimum 또는 ancestor climb, `--`는 left subtree maximum 또는 ancestor climb입니다. null end만 root snapshot을 사용합니다.
- 다음 commit에 넘기는 전제: const/reverse iterator도 같은 topology와 root snapshot limitation을 상속합니다.

### 2. feat(map): 상수와 역방향 반복자 구현

- SHA: `50e62b0e0298`
- Importance: B
- Tags: RB_TREE, ITERATOR
- Source-established role: Extends that model to const and reverse iteration.
- Source summary: Adds const and reverse map iterators and const-qualified traversal accessors.
- Source rationale: The change extends the initial traversal design with expected const correctness and reverse iteration. It remains ordinary API work and inherits the earlier end-state coupling.

#### 해당 SHA에서 확인할 실제 코드

- const iterator의 reference/pointer type과 mutable→const converting constructor를 확인합니다.
- const successor/predecessor가 mutable traversal과 같은 topology를 사용하되 write access를 노출하지 않는지 확인합니다.
- `rbegin()`이 `end()`, `rend()`가 `begin()`을 base로 shared `reverse_iterator`에 연결하는 코드 위치를 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `const_iterator`, const `begin/end`, reverse iterator typedefs와 `rbegin/rend`.
- 필요한 경우 비교할 직전 관련 SHA/parent: mutable iterator `4bbf81cecef4`와 공용 reverse adaptor `7a8e3d32bb4d`/`ae50e9038643`.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: non-const map만 traversal할 수 있고 reverse public surface가 없었습니다.
- 해결하려던 문제: const map에서 key/value를 수정하지 않는 traversal과 기존 reverse adaptor 연결이 필요했습니다.
- 선택한 결정: const pointer/reference를 노출하는 별도 iterator를 두고 mutable iterator에서 converting construction을 허용합니다. reverse traversal은 `ft::reverse_iterator`에 end/begin을 base로 전달합니다.
- 새로 생긴 책임 경계 또는 상태 변화: constness는 exposed type에서 보장되지만 topology state는 mutable iterator와 같은 node/root snapshot입니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: const-correct ordered/reverse traversal을 공개 API에 추가합니다.
- 핵심 코드와 상태 변화: runtime tree state는 바뀌지 않고 iterator wrapper와 accessors만 확장됩니다.
- 다음 commit에 넘기는 전제: mutable/const iterator가 같은 node identity를 표현하므로 서로 비교 가능해야 합니다.

### 3. feat(map): 가변·상수 반복자 상호 비교 지원

- SHA: `048ada8fe1c5`
- Importance: B
- Tags: ITERATOR, PUBLIC_API
- Source-established role: Adds mutable/const comparison interoperability.
- Source summary: Allows mutable and const map iterators to compare symmetrically.
- Source rationale: The change restores an expected interoperability property of the public iterator interface, but it is localized and later subsumed by the sentinel-based iterator refactor.

#### 해당 SHA에서 확인할 실제 코드

- const iterator가 mutable iterator에 대해 equality/inequality를 제공하는 overload를 찾습니다.
- 반대 operand ordering을 friend/non-member overload가 어떤 conversion으로 처리하는지 확인합니다.
- 비교 identity가 exposed reference type이 아니라 current node에 기반하는지 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 iterator/const_iterator mixed `operator==`와 `operator!=`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `50e62b0e0298`의 별도 mutable/const iterator types.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: 의미상 같은 위치라도 mutable iterator와 const iterator를 양쪽 operand 순서로 직접 비교할 수 없었습니다.
- 해결하려던 문제: standard iterator usage에서 `iterator == const_iterator`와 역순 비교가 모두 필요했습니다.
- 선택한 결정: exposed value type이 아니라 내부 current node pointer를 identity로 비교하고 필요한 friend/overload로 양방향 호출을 허용합니다.
- 새로 생긴 책임 경계 또는 상태 변화: iterator interoperability가 public API에 추가되며 tree state는 변하지 않습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: mutable→const conversion과 위치 비교를 실제 generic code에서 사용할 수 있게 합니다.
- 핵심 코드와 상태 변화: equality는 node identity만 확인합니다. root snapshot이 같거나 최신인지까지 확인하지 않으므로 기존 end limitation은 남습니다.
- 다음 commit에 넘기는 전제: sentinel refactor에서도 위치 identity는 하나의 structural pointer로 유지할 수 있습니다.

### 4. fix(map): 값 없는 header로 끝 반복자 상태 안정화

- SHA: `15a8460ccdfe`
- Importance: S
- Tags: ARCH, ITERATOR, RB_TREE
- Source-established role: Replaces the root snapshot with a value-free, container-owned header sentinel.
- Source summary: Refactors map to a value-free header sentinel that owns root/minimum/maximum links and represents `end()`.
- Source rationale: This is a major architectural correction: end iterators no longer carry stale root snapshots, rotations and swap can preserve element iterators, empty maps need no default key, and extrema become container-owned state. It defines the final map representation.

#### Source에서 확정된 핵심 판단

- Problem: The initial `end()` was represented by null plus a root pointer copied into each iterator. Rotations, root deletion, and swap could change the owning root while saved iterators retained stale topology. A sentinel containing a full value would also impose an invalid default-construction requirement on the key.
- Decision: The map is refactored around `node_base` and a value-free header sentinel. `header.parent` owns the root, `header.left` and `header.right` cache extrema, and `end()` points to the header. Root parent links terminate at the header, allowing successor and predecessor traversal to discover the current container boundary through topology.
- Why it mattered: This commit resolves iterator stability, current-maximum lookup from a saved end iterator, empty-state representation, swap migration, and non-default key support with one structural decision. It is the final map architecture, not a local iterator patch.

#### 해당 SHA에서 확인할 실제 코드

- `node_base`의 structural links/color/header marker와 value-bearing derived node의 분리를 확인합니다.
- header의 `parent=root`, `left=min`, `right=max` 관계와 empty state에서 extrema self-reference를 만드는 초기화/refresh 코드를 찾습니다.
- iterator state가 single `node_base*`로 줄어든 뒤 maximum increment → header, header decrement → current maximum이 되는 traversal을 추적합니다.
- insert/rotation/transplant/erase/clear/swap이 header/root/extrema invariant를 갱신하는 모든 주요 지점을 연결합니다.
- swap 뒤 moved root의 parent가 새 container header로 재부착되는 순서와 saved element iterator가 destination end에 도달하는 구조를 확인합니다.
- empty map이 default-constructible key를 요구하지 않게 된 이유를 header에 value가 없다는 실제 type layout으로 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `node_base`, derived `node`, embedded `_header`, header reset/refresh helpers, iterator next/previous, `begin/end`, insert/erase/clear/swap header repair.
- 필요한 경우 비교할 직전 관련 SHA/parent: root-snapshot iterator commits `4bbf81cecef4`, `50e62b0e0298`, `048ada8fe1c5`; regression `81d8c4489c16`.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: iterator마다 root pointer를 복사하면 `--end()`에 충분하다고 가정했습니다. end는 null current node였습니다.
- 실제 failure 또는 위험: rotation/root erase는 root를 바꾸고 swap은 tree owner를 바꾸므로 saved iterator의 root snapshot이 stale합니다. null end는 container identity와 current extrema를 스스로 표현하지 못합니다. value-bearing sentinel은 key default construction을 강제합니다.
- root cause: end/extrema 책임이 container가 아니라 개별 iterator snapshot에 분산됐습니다.
- 수정된 invariant/decision: container가 value-free embedded header를 영구 소유하고 모든 structural path가 header↔root/min/max links를 유지합니다. iterator는 current `node_base*` 하나만 저장합니다.
- mutation/commit 순서의 변경: mutation/rotation/transplant 후 root parent를 header로 붙이고 extrema cache를 갱신합니다. swap 후에는 교환된 각 root의 parent를 새 owner header로 재부착합니다.
- 실제 수정 코드 근거: `_header.is_header`, empty self-links, `_header.parent/left/right`, root parent assignment, iterator `_previous(header)`와 `_next(max)`입니다.
- 연결되는 regression test SHA/근거: `81d8c4489c16`이 saved end, rotations, former-root erase, swap migration, clear reset, non-default key를 직접 검사합니다.

#### S-level 심화 추적

- 기존 설계가 충분하지 않았던 이유: element node identity는 rotation에서 유지되지만 iterator가 별도 root snapshot을 들고 있으면 topology의 현재 container boundary와 분리됩니다. local patch로 각 iterator snapshot을 갱신할 방법도 없습니다.
- 핵심 state/ownership/lifetime transition: 기존 `_root`/null end + `(node, root snapshot)` → embedded `_header` 생성 → root parent를 header로 변경하고 extrema cache 설정 → iterator를 single node_base pointer로 축소 → end는 header address입니다.
- failure scenario별 cleanup/rollback: 이 fix의 핵심 failure는 exception이 아니라 structural stale state입니다. allocation/value lifetime은 derived `node`만 담당하며 header는 allocator로 생성/파괴하지 않습니다. clear는 value nodes를 해제하고 header를 empty self-reference로 reset합니다.
- 이 commit이 보장하는 것: saved container end가 current maximum을 찾고, element iterators가 updated parent links를 따라 end에 도달하며, swap 후 node iterators가 destination header에 연결됩니다. empty map 생성은 key constructor를 호출하지 않습니다.
- 이 commit이 아직 보장하지 않는 것: erased element를 가리키는 iterator는 무효입니다. 다른 container의 iterator 비교나 invalid dereference는 precondition 밖입니다. policy/allocator swap exception은 Thread 05에서 다룹니다.
- 후속 fix/test와 연결되는 구조: `81d8c4489c16`이 architecture promise를 deterministic scenarios로 고정하고, `cd67e6a31bb7` inspector가 header/root/extrema를 every operation 검사합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: value-free base/derived split, embedded header links, root parent termination, max→header/header→max traversal, swap reattachment입니다.

### 5. test(map): 회전·삭제·교환 뒤 반복자 상태 검증

- SHA: `81d8c4489c16`
- Importance: A
- Tags: TEST, ITERATOR, RB_TREE
- Source-established role: Verifies saved end positions, element iterators, swap migration, empty reset, and non-default keys.
- Source summary: Tests saved end iterators, element iterators through rotations, root erasure, swap, empty reset, and non-default-constructible keys.
- Source rationale: These cases validate the exact structural promises that motivated the header sentinel. The suite protects an architecture boundary that ordinary result comparison cannot observe.

#### 해당 SHA에서 확인할 실제 코드

- saved end iterator를 rotation을 일으키는 insertion 전후 및 former-root erase 전후에 유지한 test를 찾고 `--end()` expected maximum을 확인합니다.
- clear 뒤 `begin()==end()` reset assertion을 확인합니다.
- element iterator를 rotations 동안 유지하고 parent links를 통해 header까지 advance하는 test를 확인합니다.
- 두 map swap 후 held element iterators가 original node를 dereference하고 새 owner의 end sentinel까지 도달하는지 확인합니다.
- non-default-constructible key fixture가 empty map creation과 insertion을 검증하는 이유를 기록합니다.
- 확인한 파일/심볼: `tests/test_map_iterators.cpp`의 saved-end, rotation/root-erase, element-iterator, swap, clear, non-default-key scenarios.
- 필요한 경우 비교할 직전 관련 SHA/parent: production refactor `15a8460ccdfe`.

#### Test/verification 학습 기록

- 대상 production invariant: header가 current root/min/max와 end identity를 소유하고 non-erasing structural mutation이 element node identity와 traversal boundary를 보존해야 합니다.
- 재현하는 failure 또는 boundary: saved end 후 rotation, root replacement/erase, clear; held element iterators 후 rotation/swap; non-default-constructible key의 empty map입니다.
- test technique: architecture-specific deterministic regression입니다. 일부는 iterator state를 통해 internal topology를 간접 관찰합니다.
- 통과하는 production 코드 경로: insert rotations/header refresh, erase/transplant/header refresh, iterator next/previous, swap root reattachment, clear reset, map constructor입니다.
- 이 테스트가 증명하는 것: 해당 scenarios에서 saved end와 element iterator가 current/new owner header를 따라가며 extrema와 empty reset이 맞고 header가 key value를 요구하지 않습니다.
- 이 테스트가 증명하지 않는 것: erased node iterator validity, 모든 possible rotation/deletion sequence, allocator/policy exception safety는 증명하지 않습니다.
- 성격: root-snapshot limitation을 직접 재현하는 deterministic architectural regression입니다.
- 후속 변경에서 막아야 하는 회귀: iterator에 root snapshot 재도입, header extrema 갱신 누락, swap 후 root parent stale, clear 후 header self-link 누락, value-bearing sentinel입니다.
- 실행 증거: test source와 production path를 검사했으며 binary를 실제 실행하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: 저장한 iterator를 만든 뒤 topology/owner가 바뀌는 시간적 경계입니다. 단순히 mutation 후 새 iterator를 얻는 test로는 검출되지 않습니다.
- state/ownership/lifecycle 영향: header는 container lifetime에 고정되고 value nodes만 owner 간 이동합니다. swap 뒤 held element iterator는 node와 함께 새 owner topology에 속합니다.
- 이 commit이 보장하는 것과 남은 한계: exact motivating cases에 대한 evidence를 제공합니다. exhaustive iterator conformance suite는 아닙니다.
- 다음 관련 commit과의 연결: 이후 randomized inspector와 consumer tests가 same final representation을 더 넓게 사용합니다.

## 6. Invariant ledger

### Source에서 확정된 관련 invariant

- The map header sentinel is black and value-free. For an empty map it self-references as both extrema; for a non-empty map it points to the current root, minimum, and maximum, and the root points back to the header.
- Rotations and non-erasing structural changes preserve element-node identity. Saved element iterators follow updated parent links, and `end()` remains attached to container-owned sentinel state rather than a stale root snapshot.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 commit | 부족함이 드러난 commit/상태 | 강화·복구한 fix | 고정한 test/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| The map header sentinel is black and value-free; empty extrema self-reference; non-empty header caches root/min/max | `15a8460ccdfe` | `4bbf81cecef4`의 null end/root snapshot은 container-owned current state가 아님 | `15a8460ccdfe` | `81d8c4489c16`, `cd67e6a31bb7` | `node_base`, embedded `_header`, reset/refresh, inspector header checks |
| Rotations and non-erasing structural changes preserve element-node identity; iterators follow updated links | node identity는 초기 node model부터 유지, iterator traversal은 `4bbf81cecef4` | root snapshot이 rotation/root erase/swap 후 stale | `15a8460ccdfe` | `81d8c4489c16` | single-pointer iterator, root parent header, swap reattachment, saved iterator tests |

## 7. Failure → Fix → Test 연결

| 기존 상태/production change | fix 또는 verification | Source에서 확정된 연결 관점 | 실제 failure/root cause | 실제 test production path |
| --- | --- | --- | --- | --- |
| `4bbf81cecef4 / 50e62b0e0298 / 048ada8fe1c5` | `15a8460ccdfe` | root-snapshot/null-end limitation을 header sentinel architecture로 교정 | end/extrema를 iterator snapshot이 소유해 topology change 후 stale; value sentinel은 key default construction 강제 | iterator state 제거 → header links → structural paths의 refresh/reattach |
| `15a8460ccdfe` | `81d8c4489c16` | rotation/root erase/swap/clear/non-default key iterator-state regression | refactor의 핵심 promise가 ordinary post-mutation result test로는 보이지 않음 | mutation 전 saved iterator/key fixture → mutation → decrement/advance/dereference/end assertions |

## 8. Ownership / state / responsibility 변화

| 시점 | Owner / state / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `15a8460ccdfe` | Replaces the root snapshot with a value-free, container-owned header sentinel. | iterator가 current node와 root snapshot 소유; null end | container header가 root/min/max/end 소유; iterator는 node_base pointer만 소유 | `node_base`, `_header`, iterator fields, refresh/swap |
| `81d8c4489c16` | Verifies saved end positions, element iterators, swap migration, empty reset, and non-default keys. | architecture promise가 code inspection에만 의존 | regression scenarios가 시간적 iterator state를 고정 | `tests/test_map_iterators.cpp` |

## 9. Thread 최종 상태

- 최종적으로 성립한 representation/state: map object 안에 value가 없는 black header가 영구 존재합니다. empty이면 `left/right`가 자신을 가리키고 root는 없으며, non-empty이면 `parent=root`, `left=min`, `right=max`, `root.parent=&header`입니다. iterator는 단일 `node_base*`를 저장합니다.
- 최종적으로 보장하는 invariant: rotations와 non-erasing link changes는 element node address를 바꾸지 않습니다. saved element iterator는 current parent chain을 따라 container header에 도달하고, saved end는 container header이므로 current maximum을 찾습니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: erased element iterator는 무효입니다. iterator owner가 사라진 뒤 사용하거나 unrelated container iterators를 비교하는 동작은 보장하지 않습니다.
- 최종 verification evidence: saved end before rotations/root erase, held element iterator through rotations/swap, clear reset, non-default key scenarios를 코드로 확인했습니다. 실행 결과는 생성하지 못했습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point commit: `15a8460ccdfe`입니다. 앞선 세 commit은 limitation이 생기는 root-snapshot model을 구성합니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 단계명은 source가 정의한 Thread progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 commit | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | failure/cleanup | 다음 단계에 남기는 invariant |
| --- | --- | --- | --- | --- | --- | --- |
| Root-snapshot mutable iterator | `4bbf81cecef4` | iterator fields, next/previous | node와 creation-time root | subtree minimum/maximum 또는 parent climb | exception 없음; stale snapshot risk | static tree bidirectional traversal |
| Const/reverse extension | `50e62b0e0298` | const iterator, rbegin/rend | mutable topology model | const view와 shared reverse adaptor 연결 | same stale-root limitation | const/reverse public surface |
| Mixed iterator comparison | `048ada8fe1c5` | mixed equality operators | mutable/const positions | current node identity 비교 | owner/root validity는 검사 안 함 | interoperability |
| Header-sentinel refactor | `15a8460ccdfe` | node_base/header/iterators/refresh/swap | root snapshot/null end | container-owned header와 topology termination으로 전환 | clear reset; structural path마다 header repair | stable end/extrema/value-free sentinel |
| Mutation/swap iterator regressions | `81d8c4489c16` | iterator tests | mutation 전에 저장한 end/element iterator | rotation/erase/swap/clear 후 동일 object로 traversal | assertion failure로 regression 검출 | final iterator architecture evidence |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 source 순서대로 확인했습니다.
- [x] 각 commit 기록에 final HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A commit은 decision, failure boundary, ownership/state transition을 설명할 수 있습니다.
- [x] Test/perf commit은 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → failure/risk → root cause → 수정 → regression 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 commit history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] Thread 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 04-stable-map-iterators-through-structural-mutation.md =====

===== BEGIN FILE: 05-stateful-allocators-map-failure-transactions.md =====
# Stateful Allocators and Map Failure Transactions

## 1. Thread 목표

### Source-established significance

Map correctness depends on more than key/value ordering: allocator identity determines which state owns each node, and comparator state determines whether the same links describe a valid ordering. The thread first repairs lost allocator state, then ensures a comparator cannot throw after an allocation has no owner. It next makes construction and assignment transactional, and finally addresses a subtler policy failure where comparator assignment can interrupt swap. The resulting rule is consistent across the sequence: policy state must be settled before physical tree ownership is committed, and every failure path must leave each node associated with exactly one valid owner.

### 이 Thread에서 복원할 것

- 위 significance가 설명하는 변화 과정을 각 commit의 실제 SHA 코드로 재구성합니다.
- source가 확정한 commit 역할과 importance를 바꾸지 않고, 실제 implementation/failure/test 근거만 직접 채웁니다.

### Source에서 직접 연결되는 architecture

- The map's comparator defines key equivalence and tree ordering; the rebound node allocator defines physical node ownership. Copy, assignment, and swap must keep policy state and owned tree state coherent.

### Source에서 직접 연결되는 Major Engineering Difficulties

- Preserving stateful allocator and comparator semantics through node rebinding, insertion, construction, copy assignment, and public swap, including exceptions raised at comparator-call and comparator-assignment boundaries.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- allocator rebind 후에도 ownership identity를 유지해야 하는 이유는 무엇인가?
- comparator call과 node allocation의 순서를 어떻게 배치해야 unowned-node failure window가 사라지는가?
- constructor/copy assignment에서 temporary tree가 ownership transaction을 어떻게 격리하는가?
- comparator assignment 자체가 throw할 수 있을 때 policy state와 physical tree ownership 중 무엇을 먼저 commit해야 하는가?

## 3. 완료 기준

- A: 주요 subsystem/boundary/failure path/integration point를 실제 코드와 설계 판단으로 연결하고, 관련 regression 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- 모든 commit은 해당 SHA의 코드 또는 test/build diff를 근거로 기록합니다.
- Thread 최종 설명은 source 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 commit 간 변화로 재구성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| --- | --- | --- | --- | --- | --- |
| 1 | `ae180871b160` | `fix(map): 값 allocator 상태로 노드 allocator 구성` | A | RB_TREE, ALLOCATOR, RISK | Preserves value-allocator state when rebinding to node allocation. |
| 2 | `cb08194d17b0` | `fix(map): 삽입 위치를 노드 할당 전에 확정` | A | RB_TREE, EXCEPTION, DEBUG | Finishes comparator search before allocating an unlinked node. |
| 3 | `55d3b3e7c104` | `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리` | A | RB_TREE, EXCEPTION, ALLOCATOR | Cleans partial constructors and builds copy-assignment replacement trees off to the side. |
| 4 | `d72b04c5ddc6` | `test(map): 비교·할당 실패 시 노드 소유권 검증` | A | TEST, RB_TREE, EXCEPTION | Injects comparison and allocation failures and accounts for every node. |
| 5 | `0f4dd84e44ed` | `fix(map): 비교자 교환 실패 전에 tree 소유권 유지` | A | RB_TREE, EXCEPTION, DEBUG | Exchanges comparator state before allocator and tree ownership can move. |
| 6 | `55d4ba1fb493` | `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증` | A | TEST, RB_TREE, EXCEPTION | Verifies copy assignment and public swap under comparator-assignment failure. |

## 5. Commit별 학습 기록

### 1. fix(map): 값 allocator 상태로 노드 allocator 구성

- SHA: `ae180871b160`
- Importance: A
- Tags: RB_TREE, ALLOCATOR, RISK
- Source-established role: Preserves value-allocator state when rebinding to node allocation.
- Source summary: Constructs the rebound node allocator from the map's value allocator state.
- Source rationale: Default-constructing the node allocator silently lost state for custom allocators, separating allocation ownership from the public allocator contract. This small fix restores a significant resource-ownership invariant.

#### 해당 SHA에서 확인할 실제 코드

- default/range/copy constructor에서 node allocator를 value allocator state로부터 구성하는 initializer를 확인합니다.
- rebind가 allocated type만 바꾸고 ownership identity는 유지한다는 점을 allocator constructor 호출과 `get_allocator()` 관계로 확인합니다.
- 이 SHA 이전 구현이 node allocator를 default-construct하던 지점을 parent diff에서 찾아 stateful allocator에서 어떤 불일치가 생길 수 있었는지 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 value allocator `_alloc`, rebound `_node_alloc`, default/range/copy constructor initializer lists, `get_allocator`.
- 필요한 경우 비교할 직전 관련 SHA/parent: 초기 node model `2c6dd8acdd20`은 `_node_alloc()`을 독립 default construction했습니다.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: allocator rebind type을 default-construct해도 public value allocator와 같은 resource owner라고 가정했습니다.
- 실제 failure 또는 위험: allocator가 pool id, arena pointer, accounting state를 보유하면 value allocator와 node allocator가 서로 다른 state를 가집니다. node를 잘못된 owner로 allocate/deallocate하거나 public allocator contract와 실제 ownership이 분리될 수 있습니다.
- root cause: rebind를 단순 type 변환으로만 보고 state 전파를 누락했습니다.
- 수정된 invariant/decision: 모든 map constructor에서 `_node_alloc(_alloc)` 또는 제공된 value allocator에서 rebound allocator를 구성해 type만 node로 바꾸고 identity/state는 이어갑니다.
- mutation/commit 순서의 변경: node allocation이 시작되기 전에 두 allocator object의 compatible state가 constructor initializer 단계에서 확정됩니다.
- 실제 수정 코드 근거: constructor initializer list의 `_alloc(alloc), _node_alloc(_alloc)` 계열 변경입니다.
- 연결되는 regression test SHA/근거: `d72b04c5ddc6`의 tracking allocator state가 rebind 후에도 같은 shared block counter와 failure point를 관찰합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: public allocator type과 internal node allocator type 사이의 rebind boundary입니다. 컴파일은 통과하지만 stateful allocator에서만 드러나는 silent ownership 오류입니다.
- state/ownership/lifecycle 영향: node allocation과 destruction/deallocation이 같은 allocator family state에 귀속됩니다.
- 이 commit이 보장하는 것과 남은 한계: constructor에서 rebind identity를 유지합니다. insertion/constructor/assignment 실패 시 ownership transaction 자체는 다음 commits에서 강화됩니다.
- 다음 관련 commit과의 연결: `cb08194d17b0`은 node가 allocator에서 나온 직후 tree owner에 연결되기 전의 orphan window를 제거합니다.

### 2. fix(map): 삽입 위치를 노드 할당 전에 확정

- SHA: `cb08194d17b0`
- Importance: A
- Tags: RB_TREE, EXCEPTION, DEBUG
- Source-established role: Finishes comparator search before allocating an unlinked node.
- Source summary: Records the insertion side during tree search so no comparator call occurs after node allocation.
- Source rationale: A comparator exception after allocation could orphan an unlinked node. Moving all comparisons before allocation fixes the root cause at the transaction boundary and restores one-owner-at-all-times behavior.

#### 해당 SHA에서 확인할 실제 코드

- insertion traversal 중 최종 attachment side를 비교 결과와 함께 저장하는 local state를 확인합니다.
- duplicate detection과 모든 comparator call이 `_create_node` 또는 equivalent allocation 이전에 끝나는지 호출 순서를 추적합니다.
- allocation 이후 linking/red-black repair에는 pointer/color operation만 남는지 확인하여 comparator throw 뒤 unowned node window가 사라졌는지 검증합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `insert` search loop, `parent`, `insert_left` local state, `_create_node`, child link와 insertion fix-up.
- 필요한 경우 비교할 직전 관련 SHA/parent: `c0fdb8e3f84c`는 node allocation 뒤 parent side를 결정하려 comparator를 다시 호출했습니다.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: search로 parent를 찾은 뒤 node를 만들고, 어느 child에 붙일지 comparator를 한 번 더 호출했습니다.
- 실제 failure 또는 위험: node allocation/construction이 끝난 뒤 그 comparator call이 던지면 node는 아직 tree에 연결되지 않았고 catch owner도 없어 orphan/leak가 됩니다.
- root cause: potentially throwing policy operation을 resource acquisition 뒤, ownership commit 전 구간에 남겼습니다.
- 수정된 invariant/decision: search 중 매 비교 결과로 `insert_left`를 함께 기록하고 duplicate 판정까지 끝냅니다. 이후 node를 만든 뒤에는 저장된 side로 즉시 link하고 pointer/color repair만 수행합니다.
- mutation/commit 순서의 변경: 모든 comparator calls → allocation/construction → parent child link → size/fix-up입니다. allocate 후 compare가 사라집니다.
- 실제 수정 코드 근거: search loop의 `bool insert_left` 갱신과 `_create_node` 뒤 comparator가 없는 link branch입니다.
- 연결되는 regression test SHA/근거: `d72b04c5ddc6`이 comparator fail point를 sweep하고 allocation 이후 comparator invocation이 발생하면 outstanding node count로 드러나게 합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: acquired node가 아직 root/parent에서 reachable하지 않은 짧은 구간입니다.
- state/ownership/lifecycle 영향: comparator failure는 acquisition 전에 발생하므로 map state/allocator blocks가 변하지 않습니다. acquisition 성공 후 node는 즉시 map reachability에 들어갑니다.
- 이 commit이 보장하는 것과 남은 한계: normal insertion에서 post-allocation comparator throw orphan을 제거합니다. bulk construction과 assignment가 여러 successful inserts 뒤 실패하는 rollback은 별도입니다.
- 다음 관련 commit과의 연결: `55d3b3e7c104`가 여러 insertion으로 만든 partial tree 전체의 transaction을 처리합니다.

### 3. fix(map): 생성과 복사 대입 실패를 임시 tree로 격리

- SHA: `55d3b3e7c104`
- Importance: A
- Tags: RB_TREE, EXCEPTION, ALLOCATOR
- Source-established role: Cleans partial constructors and builds copy-assignment replacement trees off to the side.
- Source summary: Adds constructor rollback and copy-and-swap-style temporary-tree assignment for map.
- Source rationale: Range/copy construction now clears partial trees on failure, while assignment preserves the destination until a complete replacement exists. This is significant resource and state transaction engineering, though it does not redefine the tree structure itself.

#### Source에서 확정된 핵심 판단

- Problem: Range and copy construction inserted nodes incrementally without cleaning the partially built tree when comparison or allocation failed. Copy assignment cleared the destination before rebuilding, so failure destroyed the original value and could leave a partial replacement.
- Decision: Constructors catch failures, clear every reachable new node, and rethrow. Assignment builds a complete temporary map using the destination allocator and source comparator, then exchanges comparator and tree state only after construction succeeds.
- Why it mattered: The change establishes map's transaction boundary for bulk ownership changes. It preserves the destination under failure and ensures that a partially built tree never escapes without an owner.

#### 해당 SHA에서 확인할 실제 코드

- range/copy constructor에서 insertion failure를 catch하고 `clear()` 후 rethrow하는 rollback 경로를 확인합니다.
- copy assignment가 destination allocator와 source comparator로 temporary map을 먼저 완성하는지 확인합니다.
- complete temporary tree 이후 private exchange가 새 tree를 install하고 temporary가 old destination을 destroy하는 ownership 흐름을 그립니다.
- source가 comparator policy swap 자체의 throw ordering은 후속 commit에서 hardened된다고 명시하므로, 이 SHA에서 그 residual risk를 표시합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 range/copy constructors, `operator=`, private tree/comparator exchange helper, `clear`.
- 필요한 경우 비교할 직전 관련 SHA/parent: clear-then-insert assignment `c0fdb8e3f84c`; failure tests `d72b04c5ddc6`; ordering fix `0f4dd84e44ed`.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: constructor body에서 successful inserts가 자동으로 정리될 것이라 가정했고, assignment는 destination을 먼저 clear했습니다.
- 실제 failure 또는 위험: constructor가 완성되지 않으면 map destructor가 호출되지 않아 partial nodes가 남을 수 있습니다. assignment 실패는 original target을 잃고 partial replacement만 남깁니다.
- root cause: multi-node build의 temporary ownership과 commit point가 명시되지 않았습니다.
- 수정된 invariant/decision: constructor catch가 현재 reachable tree를 clear합니다. assignment는 target allocator와 source comparator로 complete temporary를 만들고 성공 후에만 policy/tree state를 exchange합니다.
- mutation/commit 순서의 변경: temporary build → complete 확인 → exchange commit → temporary destructor가 old target tree cleanup입니다. build failure면 target untouched, temporary/constructor partial tree만 clear합니다.
- 실제 수정 코드 근거: constructor `try { insert(...) } catch (...) { clear(); throw; }`, `operator=`의 temporary map과 private exchange 호출입니다.
- 연결되는 regression test SHA/근거: `d72b04c5ddc6`이 range/copy construction comparison/allocation failures와 assignment target preservation/outstanding nodes를 검사합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: object construction이 완료되기 전 destructor 자동 호출이 없는 구간과 assignment replacement build 구간입니다.
- state/ownership/lifecycle 영향: partial constructor nodes는 under-construction object가 clear하고, replacement nodes는 temporary가 소유합니다. commit 후 temporary가 old target owner가 됩니다.
- 이 commit이 보장하는 것과 남은 한계: comparator/allocation failure에서 partial constructor cleanup과 target preservation을 제공합니다. comparator exchange 자체가 throw할 때 tree/allocator보다 뒤에서 수행되면 partial commit 위험이 남아 `0f4dd84e44ed`가 순서를 고칩니다.
- 다음 관련 commit과의 연결: `d72b04c5ddc6`이 transaction을 검증하고, `0f4dd84e44ed`/`55d4ba1fb493`가 policy-assignment failure를 별도로 다룹니다.

### 4. test(map): 비교·할당 실패 시 노드 소유권 검증

- SHA: `d72b04c5ddc6`
- Importance: A
- Tags: TEST, RB_TREE, EXCEPTION
- Source-established role: Injects comparison and allocation failures and accounts for every node.
- Source summary: Adds comparator and allocator failure injection for insertion, constructors, copying, and assignment.
- Source rationale: The suite verifies that every allocated node remains owned or is released and that failed assignment preserves the target. It directly validates the map transaction boundaries established by the preceding fixes.

#### 해당 SHA에서 확인할 실제 코드

- stateful comparator와 rebound-surviving allocator fixture의 identity/failure counter/outstanding node tracking을 확인합니다.
- insertion comparator failure sweep가 allocation 이후 comparator call이 없다는 production invariant를 어떻게 검증하는지 확인합니다.
- range/copy construction의 comparison/allocation failure 후 residual node count가 zero인지 확인합니다.
- copy assignment failure에서 pre-existing target keys와 baseline allocation count가 보존되는지 확인합니다.
- generated input iterator가 다른 container implementation에 의존하지 않고 test values를 공급하는 방식을 확인합니다.
- 확인한 파일/심볼: `tests/test_map_exceptions.cpp`의 `throwing_less`, shared comparator state, tracking allocator/shared block state, generated input iterator, insertion/constructor/copy/assignment failure tests.
- 필요한 경우 비교할 직전 관련 SHA/parent: production fixes `ae180871b160`, `cb08194d17b0`, `55d3b3e7c104`.

#### Test/verification 학습 기록

- 대상 production invariant: rebind allocator identity가 유지되고, every allocation은 tree/temporary owner에 연결되거나 released되며, failed bulk operation이 target을 보존해야 합니다.
- 재현하는 failure 또는 boundary: comparator call index, allocator allocation index, range/copy constructor progress, copy assignment replacement progress입니다.
- test technique: deterministic comparator/allocation failure injection과 outstanding-node accounting입니다.
- 통과하는 production 코드 경로: insertion search/acquisition/link, range/copy constructor catch/clear, temporary-tree assignment build/exchange입니다.
- 이 테스트가 증명하는 것: 검사한 fail points에서 insertion은 leak 없이 실패하고, constructors는 residual block 0이며, assignment target keys와 baseline target allocation count가 유지됩니다. rebind된 allocator copies가 같은 shared owner state를 사용합니다.
- 이 테스트가 증명하지 않는 것: comparator assignment 중 failure와 public swap commit ordering은 이 SHA의 suite 범위가 아닙니다. 모든 third-party allocator/comparator behavior도 exhaustive하지 않습니다.
- 성격: transaction boundary를 고정하는 deterministic failure-injection regression입니다.
- 후속 변경에서 막아야 하는 회귀: allocate-after/compare ordering 역전, constructor catch 제거, assignment target 조기 clear, rebind state 단절입니다.
- 실행 증거: fixture와 fail-point loops를 코드로 확인했습니다. test binary를 실행하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: `throwing_less`는 지정 call에서 던지고 tracking allocator는 지정 acquisition에서 던집니다. generated iterator는 external container behavior 없이 range construction을 진행시킵니다.
- state/ownership/lifecycle 영향: shared allocator state가 rebound type마다 같은 outstanding count를 보므로 node block leak를 직접 계측합니다.
- 이 commit이 보장하는 것과 남은 한계: preceding fixes의 compare/allocation transaction을 강하게 뒷받침합니다. policy object assignment failure는 다음 fix/test가 필요합니다.
- 다음 관련 commit과의 연결: `0f4dd84e44ed`가 comparator assignment를 ownership commit보다 앞으로 옮기고 `55d4ba1fb493`가 그 failure를 주입합니다.

### 5. fix(map): 비교자 교환 실패 전에 tree 소유권 유지

- SHA: `0f4dd84e44ed`
- Importance: A
- Tags: RB_TREE, EXCEPTION, DEBUG
- Source-established role: Exchanges comparator state before allocator and tree ownership can move.
- Source summary: Moves comparator exchange before allocator and tree ownership exchange in map swap and assignment commit.
- Source rationale: If comparator assignment throws after tree pointers move, ordering policy and physical ownership can diverge. This small ordering fix prevents partial ownership transfer and restores a high-risk exception invariant.

#### 해당 SHA에서 확인할 실제 코드

- public swap과 private assignment exchange에서 comparator exchange가 allocator/root/size ownership exchange보다 먼저 수행되는 순서를 확인합니다.
- comparator assignment throw 시점에 두 map의 original root/size/allocator가 아직 움직이지 않았는지 state mutation 순서를 표시합니다.
- comparator step 성공 후 ownership swap과 header refresh가 수행되는 commit phase를 구분합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 public `swap`과 private tree/comparator exchange helper; comparator swap line과 allocator/header/size swap/refresh lines.
- 필요한 경우 비교할 직전 관련 SHA/parent: transaction commit `55d3b3e7c104`; regression `55d4ba1fb493`.

#### Failure → Fix 추적

- 기존 가정 또는 직전 구현 상태: tree/header/allocator ownership을 먼저 교환한 뒤 comparator를 교환해도 된다고 봤습니다.
- 실제 failure 또는 위험: comparator assignment가 그 뒤 던지면 physical tree는 상대 map으로 이동했는데 ordering policy 또는 allocator state는 원래 위치에 남아 search ordering과 deallocation ownership이 분리됩니다.
- root cause: potentially throwing policy commit을 irreversible physical ownership movement 뒤에 배치했습니다.
- 수정된 invariant/decision: comparator exchange를 public swap과 private assignment commit의 첫 단계로 수행합니다. 이 단계가 완료되기 전 root/header/size/allocator는 움직이지 않습니다.
- mutation/commit 순서의 변경: comparator policy exchange → allocator/tree/header/size ownership exchange → moved root를 new header에 reattach/refresh입니다.
- 실제 수정 코드 근거: 두 exchange function에서 comparator swap line이 physical state swaps보다 위로 이동한 diff입니다.
- 연결되는 regression test SHA/근거: `55d4ba1fb493`이 comparator assignment throw를 copy assignment와 public swap에 주입하고 contents/allocator/node counts/usability를 검사합니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: comparator copy/assignment가 throw 가능한 policy operation이라는 점입니다.
- state/ownership/lifecycle 영향: failure 시 physical node tree와 allocator owners는 원래 map에 남고 complete replacement temporary는 자신의 allocator로 해제됩니다.
- 이 commit이 보장하는 것과 남은 한계: tree ownership이 comparator exchange failure 뒤 부분 이동하지 않게 합니다. comparator type 자체가 assignment 전에 state를 변형한 뒤 던지는 경우까지 일반적인 strong guarantee를 구현이 별도로 복원하지는 않습니다. regression fixture는 throw 전에 자기 state를 유지하는 동작을 검증합니다.
- 다음 관련 commit과의 연결: `55d4ba1fb493`가 distinct ordering directions/allocator owners로 policy-tree mismatch를 observable하게 만듭니다.

### 6. test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증

- SHA: `55d4ba1fb493`
- Importance: A
- Tags: TEST, RB_TREE, EXCEPTION
- Source-established role: Verifies copy assignment and public swap under comparator-assignment failure.
- Source summary: Injects comparator-assignment failure into copy assignment and public swap with distinct allocator owners and order directions.
- Source rationale: The tests make partial policy exchange observable and verify both maps retain contents, allocator identity, node counts, and usability. They strongly validate the subtle commit-ordering fix.

#### 해당 SHA에서 확인할 실제 코드

- assignment에서 throw할 수 있는 comparator와 distinct allocator identity/outstanding block을 추적하는 fixture를 확인합니다.
- copy assignment failure에서 target의 original descending sequence, allocator node count, 추가 insertion 가능성이 보존되는지 확인합니다.
- public swap failure에서 두 map의 contents, allocator identity, node ownership count가 모두 original 상태인지 확인합니다.
- completed temporary tree가 failure 뒤 release되는 assertion을 확인합니다.
- 확인한 파일/심볼: `tests/test_map_policy_exceptions.cpp`의 stateful/throwing comparator, tracking allocator owner state, copy-assignment failure와 public-swap failure tests.
- 필요한 경우 비교할 직전 관련 SHA/parent: production ordering fix `0f4dd84e44ed`.

#### Test/verification 학습 기록

- 대상 production invariant: comparator assignment failure 전에 policy/tree/allocator ownership이 partial exchange되지 않고 각 map이 original ordering과 node owner를 유지해야 합니다.
- 재현하는 failure 또는 boundary: opposite ordering directions를 가진 comparators, distinct allocator ids, completed replacement temporary, comparator assignment throw입니다.
- test technique: deterministic policy-assignment failure injection + allocator ownership accounting + post-failure usability check입니다.
- 통과하는 production 코드 경로: copy assignment temporary construction/private exchange, public swap, comparator swap, temporary destructor, subsequent insert/traversal입니다.
- 이 테스트가 증명하는 것: chosen throwing comparator에서 failed copy assignment는 original target sequence/allocator/count를 유지하고 temporary nodes를 release하며, failed public swap은 양쪽 contents/allocator/count를 유지합니다. 두 maps는 이후 insert/search/traversal에 사용 가능합니다.
- 이 테스트가 증명하지 않는 것: comparator가 자기 assignment 중 state를 부분 변경한 뒤 던지는 arbitrary behavior, allocator swap 자체의 throw, concurrent access는 다루지 않습니다.
- 성격: subtle commit-ordering fix를 직접 고정하는 deterministic regression입니다.
- 후속 변경에서 막아야 하는 회귀: comparator exchange를 tree/allocator swap 뒤로 이동하거나, failed temporary ownership을 target에 일부 연결하고 cleanup을 누락하는 회귀입니다.
- 실행 증거: test fixture와 assertions를 코드로 검사했으며 실행하지 않았습니다.

#### A-level 핵심 확인

- 중요 boundary/failure path: temporary tree가 완성된 뒤 최종 commit 첫 policy assignment에서 실패하는 late failure입니다.
- state/ownership/lifecycle 영향: target/source physical trees는 그대로이고 temporary가 replacement nodes를 소유한 채 scope unwinding으로 해제됩니다. allocator counters가 이 ownership 분리를 확인합니다.
- 이 commit이 보장하는 것과 남은 한계: 해당 comparator/allocator 모델에서 state preservation과 leak absence를 증명합니다. 언어 수준의 모든 policy implementation을 일반화하지는 않습니다.
- 다음 관련 commit과의 연결: 이 Thread의 최종 transaction rule을 검증하며 이후 acceptance/sanitizer suite에 포함됩니다.

## 6. Invariant ledger

### Source에서 확정된 관련 invariant

- The map's comparator defines key equivalence and tree ordering; the rebound node allocator defines physical node ownership. Copy, assignment, and swap must keep policy state and owned tree state coherent.
- At every failure boundary, each map node has exactly one owner or has been released. Comparator, allocator, and tree ownership must not become partially exchanged when policy assignment throws.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 commit | 부족함이 드러난 commit/상태 | 강화·복구한 fix | 고정한 test/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Comparator defines ordering; rebound allocator defines physical ownership; bulk operations keep them coherent | comparator/tree는 `c0fdb8e3f84c`, node allocator는 `2c6dd8acdd20` | node allocator default construction으로 public allocator state 상실; assignment/swap commit 순서 위험 | `ae180871b160`, `55d3b3e7c104`, `0f4dd84e44ed` | `d72b04c5ddc6`, `55d4ba1fb493` | constructor allocator initializers, temporary assignment, policy-first exchange, owner counters |
| Every failure leaves each node with exactly one owner or released; no partial policy/tree exchange | node create rollback은 `2c6dd8acdd20` | post-allocation comparator throw orphan, partial constructors, clear-first assignment, tree-first comparator swap | `cb08194d17b0`, `55d3b3e7c104`, `0f4dd84e44ed` | `d72b04c5ddc6`, `55d4ba1fb493` | compare-before-allocate, catch-clear, temporary owner, policy-first commit, block counts |

## 7. Failure → Fix → Test 연결

| 기존 상태/production change | fix 또는 verification | Source에서 확정된 연결 관점 | 실제 failure/root cause | 실제 test production path |
| --- | --- | --- | --- | --- |
| `ae180871b160` | `d72b04c5ddc6` | stateful allocator ownership이 failure suite에서 간접 검증 | rebind type default construction이 allocator identity를 끊음 | tracking allocator copy/rebind → node operations → shared outstanding counts |
| `cb08194d17b0` | `d72b04c5ddc6` | comparator failure after allocation이 없음을 sweep | attachment-side comparator가 acquired unlinked node 뒤 throw | compare fail sweep → allocation count/owned node assertions |
| `55d3b3e7c104` | `d72b04c5ddc6` | constructor rollback과 assignment target preservation 검증 | partial constructor cleanup 부재, target clear-first | range/copy/assignment failure → clear/temp destruction → keys/counts |
| `0f4dd84e44ed` | `55d4ba1fb493` | comparator-assignment failure에서 policy/tree ownership 보존 검증 | physical ownership을 policy보다 먼저 옮긴 partial commit | comparator assignment throw → original contents/allocator/count/usability |

## 8. Ownership / state / responsibility 변화

| 시점 | Owner / state / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `ae180871b160` | Preserves value-allocator state when rebinding to node allocation. | node allocator 독립 default state | value allocator family state로 node allocator 구성 | constructor initializers |
| `cb08194d17b0` | Finishes comparator search before allocating an unlinked node. | acquired node와 link 사이 policy throw 가능 | all compare before acquisition, then immediate link | `insert_left`, `_create_node` ordering |
| `55d3b3e7c104` | Cleans partial constructors and builds copy-assignment replacement trees off to the side. | under-construction/target가 partial state 소유 | constructor clear 또는 temporary가 replacement 소유 | catch-clear, temporary assignment |
| `d72b04c5ddc6` | Injects comparison and allocation failures and accounts for every node. | ownership 근거가 코드 추론 | shared test state가 every block 계측 | map exception fixtures |
| `0f4dd84e44ed` | Exchanges comparator state before allocator and tree ownership can move. | physical ownership 먼저 교환 | policy 성공 뒤에만 physical commit | swap/exchange order |
| `55d4ba1fb493` | Verifies copy assignment and public swap under comparator-assignment failure. | late policy failure evidence 부족 | distinct owner/order tests가 preservation 고정 | policy exception tests |

## 9. Thread 최종 상태

- 최종적으로 성립한 representation/state: map은 comparator policy, public value allocator, compatible rebound node allocator, header-owned tree를 함께 보유합니다. replacement tree는 target 밖 temporary에서 완성되고 commit 전까지 독립 owner입니다.
- 최종적으로 보장하는 invariant: comparator calls는 insertion acquisition 전에 끝납니다. partial constructors는 reachable nodes를 clear합니다. failed assignment/swap은 physical tree/allocator ownership을 부분 교환하지 않으며 각 node는 original map 또는 temporary에 속하거나 해제됩니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: comparator/allocator의 자체 copy/assignment semantics가 내부 state를 바꾸고 throw하는 임의 구현에 대해 별도 universal rollback은 없습니다. tests는 repository의 deterministic fixtures가 정의한 failure behavior를 검증합니다.
- 최종 verification evidence: comparator/allocation fail-point sweeps, constructor residual count 0, assignment target preservation, distinct allocator/comparator swap failure tests를 코드로 확인했습니다. 실제 실행은 수행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point commit: unowned acquisition window를 제거한 `cb08194d17b0`, temporary-tree transaction `55d3b3e7c104`, policy-first commit `0f4dd84e44ed`입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 단계명은 source가 정의한 Thread progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 commit | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | failure/cleanup | 다음 단계에 남기는 invariant |
| --- | --- | --- | --- | --- | --- | --- |
| Allocator rebind identity | `ae180871b160` | map constructor initializers | value allocator state | rebound node allocator를 value allocator에서 구성 | construction 전 policy 단계 | allocation/deallocation owner compatibility |
| Comparison-before-allocation boundary | `cb08194d17b0` | `insert` search/link | key, comparator, current tree | parent와 side 확정 후 node acquire/link | compare throw는 acquisition 전 | no unowned node after policy throw |
| Temporary-tree construction/assignment | `55d3b3e7c104` | constructors, `operator=`, exchange | input/source와 original target | temporary complete 후 commit | partial clear/temp destructor | original target preserved until commit |
| Failure-injection ownership proof | `d72b04c5ddc6` | map exception tests | compare/alloc fail index | production path 반복 주입 | outstanding nodes/keys 관찰 | transaction evidence |
| Comparator-before-ownership exchange | `0f4dd84e44ed` | public/private swap | two policies and trees | comparator exchange 먼저, then physical ownership | policy throw 시 trees untouched | no partial physical transfer |
| Comparator-assignment regression | `55d4ba1fb493` | policy exception tests | opposite ordering/distinct allocators | late failure 후 original state 검사 | temporary nodes released | policy/tree/owner coherence evidence |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 source 순서대로 확인했습니다.
- [x] 각 commit 기록에 final HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A commit은 decision, failure boundary, ownership/state transition을 설명할 수 있습니다.
- [x] Test/perf commit은 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → failure/risk → root cause → 수정 → regression 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 commit history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] Thread 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 05-stateful-allocators-map-failure-transactions.md =====

===== BEGIN FILE: 06-header-only-public-surface-automated-acceptance.md =====
# Header-only Public Surface and Automated Acceptance

## 1. Thread 목표

### Source-established significance

A header-only library can appear correct inside one monolithic test while still depending on include order, emitting duplicate definitions, or compiling only under one toolchain. This progression expands the acceptance boundary from a convenience include, to independently self-contained headers, to a real linked consumer, and finally to sanitizer and cross-platform automation. These commits do not define the core containers, but they make the finished public surface reproducible and consumable outside the repository's internal test arrangement.

### 이 Thread에서 복원할 것

- 위 significance가 설명하는 변화 과정을 각 commit의 실제 SHA 코드로 재구성합니다.
- source가 확정한 commit 역할과 importance를 바꾸지 않고, 실제 implementation/failure/test 근거만 직접 채웁니다.

### Source에서 직접 연결되는 architecture

- Verification is layered: public differential tests, targeted edge and failure tests, iterator-state tests, a constrained internal tree inspector, deterministic randomized operations, complexity bounds, independent-header compilation, linked consumer tests, sanitizers, and compiler/platform CI.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- header-only library가 monolithic test 하나를 통과해도 consumer에게 깨질 수 있는 이유는 무엇인가?
- aggregate header smoke, independent-header compile, multi-TU link가 서로 다른 어떤 failure class를 잡는가?
- sanitizer build를 normal build와 격리해야 하는 이유는 무엇인가?
- CI matrix가 C++98 portability와 memory-safety acceptance를 어떻게 repeatable contract로 만드는가?

## 3. 완료 기준

- B: Thread 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- C: Thread 이해에 필요한 맥락과 최소 검증 포인트만 확인합니다. S/A와 같은 깊이의 분석을 강제하지 않습니다.
- 모든 commit은 해당 SHA의 코드 또는 test/build diff를 근거로 기록합니다.
- Thread 최종 설명은 source 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 commit 간 변화로 재구성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| --- | --- | --- | --- | --- | --- |
| 1 | `80e169e83212` | `feat(headers): 공용 도구와 순차 컨테이너 통합 헤더 추가` | B | PUBLIC_API, ARCH | Introduces the aggregate header. |
| 2 | `3c64a69dd252` | `test(headers): 통합 헤더의 순차 컨테이너 표면 검증` | C | TEST, PUBLIC_API | Confirms the current sequential surface compiles through that bundle. |
| 3 | `112af1753538` | `feat(map): 관계 연산과 통합 공개 헤더 완성` | B | PUBLIC_API, RB_TREE | Adds map to the aggregate public entry point. |
| 4 | `d938c0079994` | `test(headers): 공개 헤더를 각각 독립 compile` | B | TEST, PUBLIC_API, CXX98 | Compiles every public header independently as the first include. |
| 5 | `072c49832ddc` | `test(consumer): 다중 번역 단위 공개 헤더 사용 검증` | B | TEST, PUBLIC_API, INTEGRATION | Links independent vector and map consumers in multiple translation units. |
| 6 | `1be03ae8daef` | `build(makefile): 격리된 sanitizer 검사 대상 추가` | B | PUBLIC_API, PRACTICAL, RISK | Runs the complete acceptance surface under isolated ASan/UBSan instrumentation. |
| 7 | `228f457988be` | `ci: compiler 행렬과 sanitizer 검사 구성` | B | CXX98, PUBLIC_API, PRACTICAL | Automates compiler, platform, and sanitizer checks in CI. |

## 5. Commit별 학습 기록

### 1. feat(headers): 공용 도구와 순차 컨테이너 통합 헤더 추가

- SHA: `80e169e83212`
- Importance: B
- Tags: PUBLIC_API, ARCH
- Source-established role: Introduces the aggregate header.
- Source summary: Adds an aggregate public header for the utility layer, vector, and stack.
- Source rationale: This creates a deliberate packaging boundary for a header-only library. It matters to consumers, but the implementation is simple composition rather than a major runtime or ownership decision.

#### 해당 SHA에서 확인할 실제 코드

- `ft_containers.hpp`가 traits/iterators/algorithms/pair/vector/stack의 aggregate entry point로 어떤 component headers를 include하는지 확인합니다.
- component header를 직접 include하는 경로가 제거되지 않았는지 공개 surface 관점에서 확인합니다.
- 이 commit은 runtime behavior가 아니라 packaging boundary를 만든다는 source 역할과 실제 diff가 일치하는지 기록합니다.
- 확인한 파일/심볼: `include/ft_containers.hpp`의 include guard와 `ft_algorithm.hpp`, `ft_iterator.hpp`, `ft_pair.hpp`, `ft_stack.hpp`, `ft_type_traits.hpp`, `ft_vector.hpp` include 목록입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: 직전에는 각 component header만 존재했고 하나의 공개 bundle은 없었습니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: consumer가 필요한 utility/container headers를 개별 선택해 include해야 했습니다.
- 해결하려던 문제: 정상 조합을 매 consumer가 반복해서 구성하면 누락과 include-order 차이가 공개 사용법에 섞입니다.
- 선택한 결정: 기존 component headers를 변경하거나 숨기지 않고, 그 위에 `ft_containers.hpp`라는 convenience entry point를 추가했습니다.
- 새로 생긴 책임 경계 또는 상태 변화: component header는 개별 API를 계속 담당하고 aggregate header는 지원되는 조합을 노출하는 packaging 책임만 가집니다. 이 SHA에는 map이 아직 포함되지 않습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: header-only library의 첫 명시적 public bundle을 만듭니다.
- 핵심 코드와 상태 변화: runtime state는 없습니다. preprocessor include graph에 aggregate root가 추가됩니다.
- 다음 commit에 넘기는 전제: 이 bundle 하나로 기존 순차 컨테이너 tests가 그대로 compile/run되어야 합니다.

### 2. test(headers): 통합 헤더의 순차 컨테이너 표면 검증

- SHA: `3c64a69dd252`
- Importance: C
- Tags: TEST, PUBLIC_API
- Source-established role: Confirms the current sequential surface compiles through that bundle.
- Source summary: Switches the existing test to include the aggregate header instead of component headers.
- Source rationale: The change is a small integration smoke test for one include list. It adds little new technical evidence beyond confirming that the bundle contains the already tested headers.

#### 해당 SHA에서 확인할 실제 코드

- 기존 consumer-style test가 component headers 대신 aggregate header 하나만 include하도록 바뀐 diff를 확인합니다.
- runtime assertion 자체는 바뀌지 않았는지 확인하여 failure가 public header composition 문제로 귀속되는 구조를 기록합니다.
- 확인한 파일/심볼: `tests/test_containers.cpp`의 여러 `ft_*.hpp` includes가 `#include "ft_containers.hpp"` 하나로 교체된 지점입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: aggregate header 도입 `80e169e83212`입니다.

#### Test/verification 학습 기록

- 대상 production invariant: aggregate header가 utility, vector, stack의 기존 public symbols와 필요한 내부 dependencies를 빠짐없이 노출해야 합니다.
- 재현하는 failure 또는 boundary: bundle include 누락 또는 include ordering 때문에 기존 test source가 compile되지 않는 경우입니다.
- test technique: 기존 broad behavioral test의 include surface만 aggregate header로 바꾸는 integration smoke입니다.
- 통과하는 production 코드 경로: `ft_containers.hpp`의 transitive include graph를 거쳐 기존 utility/vector/stack test code가 compile되고 실행됩니다.
- 이 테스트가 증명하는 것: 한 translation unit에서 현재 순차 surface가 aggregate include만으로 기존 checks를 사용할 수 있음을 증명합니다.
- 이 테스트가 증명하지 않는 것: 각 component header의 독립 self-containment, 여러 translation units의 linkage/ODR, map 포함 여부, compiler/platform portability는 증명하지 않습니다.
- 성격: 특정 regression보다 public bundle의 broad smoke입니다.
- 후속 변경에서 막아야 하는 회귀: aggregate header에서 필요한 component include를 제거해 기존 test가 직접 include에 우연히 의존하도록 만드는 회귀입니다.
- 실행 증거: include diff와 기존 test 연결을 코드로 검사했으며 executable은 실행하지 않았습니다.

#### C-level 최소 확인

- Thread 이해에 필요한 맥락: 구현 동작을 새로 시험한 것이 아니라 test의 진입 include를 바꿔 packaging boundary를 통과시켰습니다.
- 최소 코드/검증 근거: `tests/test_containers.cpp`의 include replacement 한 곳이며 test body 변경은 없습니다.

### 3. feat(map): 관계 연산과 통합 공개 헤더 완성

- SHA: `112af1753538`
- Importance: B
- Tags: PUBLIC_API, RB_TREE
- Source-established role: Adds map to the aggregate public entry point.
- Source summary: Adds map value comparison, relational operators, non-member swap, and inclusion in the aggregate header.
- Source rationale: This completes the baseline public surface through established shared algorithms. It is useful integration work without a new core data-structure decision.

#### 해당 SHA에서 확인할 실제 코드

- `ft_containers.hpp`에 map이 추가된 include diff를 확인합니다.
- 같은 SHA에서 map relations/value comparator/non-member swap이 공개 surface에 완성되는 위치를 확인하되, 이 Thread에서는 aggregate header로 노출되는 경계에 초점을 둡니다.
- aggregate consumer가 internal include graph를 알 필요 없이 map까지 사용할 수 있는지 해당 SHA의 test/include 관계로 확인합니다.
- 확인한 파일/심볼: `include/ft_containers.hpp`의 `#include "ft_map.hpp"`; `include/ft_map.hpp`의 `value_compare`, `value_comp`, six relational operators, non-member `swap`입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: `80e169e83212`의 aggregate list에는 map이 없었습니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: aggregate entry point는 utility/vector/stack까지만 포함해 associative container surface가 불완전했습니다.
- 해결하려던 문제: map을 직접 include해야 했고, map의 public comparison/swap surface도 동일 commit에서 완성될 필요가 있었습니다.
- 선택한 결정: map header를 aggregate list에 추가하고, key comparator를 value pairs에 적용하는 `value_compare`, shared `ft::equal`/`ft::lexicographical_compare` 기반 relations, member swap forwarding을 공개했습니다.
- 새로 생긴 책임 경계 또는 상태 변화: `ft_containers.hpp`가 이 repository의 utility, sequential, adaptor, associative surface를 한 entry point에서 노출합니다. runtime tree representation은 이 commit에서 새로 정의되지 않습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: aggregate public surface를 map까지 확장해 baseline bundle을 완성합니다.
- 핵심 코드와 상태 변화: include graph에 `ft_map.hpp`가 추가되고 map public non-member/value-comparison symbols가 생깁니다.
- 다음 commit에 넘기는 전제: bundle만 아니라 각 component header도 독립적으로 자신의 dependencies를 include해야 합니다.

### 4. test(headers): 공개 헤더를 각각 독립 compile

- SHA: `d938c0079994`
- Importance: B
- Tags: TEST, PUBLIC_API, CXX98
- Source-established role: Compiles every public header independently as the first include.
- Source summary: Compiles each public header independently as the first include in a minimal translation unit.
- Source rationale: This enforces header self-containment and catches accidental transitive dependencies, an important header-only API practice. It is significant integration hygiene rather than core container logic.

#### 해당 SHA에서 확인할 실제 코드

- 각 public header마다 정확히 하나의 library header를 first include하는 minimal translation unit 목록을 확인합니다.
- 각 translation unit이 representative operation을 instantiate하여 선언뿐 아니라 필요한 template dependency까지 컴파일하는지 확인합니다.
- Makefile의 dedicated `headers` target과 object output 경로를 확인합니다.
- 확인한 파일/심볼: `tests/headers/{ft_algorithm,ft_containers,ft_iterator,ft_map,ft_pair,ft_stack,ft_type_traits,ft_vector}.cpp`; Makefile의 `HEADER_TEST_SOURCES`, `HEADER_TEST_OBJECTS`, `build/headers/%.o`, `headers` target입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: aggregate expansion `112af1753538` 뒤의 public header set입니다.

#### Test/verification 학습 기록

- 대상 production invariant: 지원하는 각 public header는 strict C++98 translation unit의 첫 library include로 단독 compile되어야 합니다.
- 재현하는 failure 또는 boundary: header가 자신이 사용하는 standard/project declaration을 직접 include하지 않고 다른 header의 선행 include에 기대는 경우입니다.
- test technique: independent translation-unit compile probes입니다. 각 source를 `-c`하여 별도 object로 만듭니다.
- 통과하는 production 코드 경로: 각 header의 include guard/include graph와 representative template instantiation입니다. algorithm은 `equal`/lexicographic comparison, iterator는 reverse dereference, vector/map/stack은 기본 operation을 instantiate합니다.
- 이 테스트가 증명하는 것: 각 probe가 요구하는 declarations/templates가 include order 도움 없이 compile됨을 증명합니다.
- 이 테스트가 증명하지 않는 것: object들을 하나로 link하지 않으므로 duplicate external definitions/ODR failures, runtime behavior, 모든 template argument 조합은 증명하지 않습니다.
- 성격: public-header self-containment를 직접 고정하는 deterministic compile regression입니다.
- 후속 변경에서 막아야 하는 회귀: component header의 required include를 제거하고 aggregate 또는 test include order에 기대는 회귀입니다.
- 실행 증거: probe sources와 Make rules를 코드로 검사했으며 `make headers`는 실행하지 않았습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: aggregate smoke에서 component별 compile isolation으로 acceptance boundary를 확대합니다.
- 핵심 코드와 상태 변화: public header마다 최소 TU가 생기고 outputs는 `build/headers`에 분리됩니다.
- 다음 commit에 넘기는 전제: compile self-containment를 넘어 독립 TUs를 실제 executable로 link하고 실행해야 합니다.

### 5. test(consumer): 다중 번역 단위 공개 헤더 사용 검증

- SHA: `072c49832ddc`
- Importance: B
- Tags: TEST, PUBLIC_API, INTEGRATION
- Source-established role: Links independent vector and map consumers in multiple translation units.
- Source summary: Adds a linked multi-translation-unit consumer for vector and map and composes a complete `check` target.
- Source rationale: The change verifies real header-only consumption and catches ODR or linkage failures that single-file tests cannot. It is strong practical integration work but does not alter container semantics.

#### 해당 SHA에서 확인할 실제 코드

- vector consumer TU, map consumer TU, shared declaration header, caller/main TU의 include와 symbol 관계를 그립니다.
- 각 TU가 독립 compile된 뒤 하나의 executable로 link되는 target을 확인합니다.
- single-file test가 잡지 못하는 header-only ODR/linkage failure를 이 구조가 어떻게 노출하는지 기록합니다.
- new `check` target이 behavioral tests, header compilation, linked consumer를 어떤 순서/의존성으로 묶는지 확인합니다.
- 확인한 파일/심볼: `tests/consumer/consumer_api.hpp`, `vector_consumer.cpp`, `map_consumer.cpp`, `main.cpp`; Makefile의 `CONSUMER_OBJECTS`, `CONSUMER_BIN`, `consumer`, `check` targets입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: independent compile probes `d938c0079994`입니다.

#### Test/verification 학습 기록

- 대상 production invariant: header-only definitions와 shared transitive utilities가 여러 independently compiled TUs에서 사용돼도 하나의 executable로 link되고 예상 동작을 해야 합니다.
- 재현하는 failure 또는 boundary: non-inline external definition duplication, missing linkage, include-order/TU-local dependency, consumer-facing runtime integration 오류입니다.
- test technique: multi-translation-unit compile + link + executable integration test입니다.
- 통과하는 production 코드 경로: vector TU는 1..5 push 후 index 2에 두 개의 7을 insert해 합계 29를 반환합니다. map TU는 3/1/2를 insert하고 key 1을 erase한 뒤 key+value 합계 55를 반환합니다. main TU가 두 external functions를 호출해 값을 검사합니다.
- 이 테스트가 증명하는 것: 이 TU 구성에서 overlapping header-only dependencies가 link되며 vector insertion과 map insertion/erase/iteration의 representative consumer flow가 실행 가능한 구조임을 증명합니다.
- 이 테스트가 증명하지 않는 것: 모든 public header가 동시에 여러 TUs에서 모든 instantiation으로 사용되는 경우, shared-library ABI, dynamic loading, 모든 ODR-sensitive 조합은 다루지 않습니다.
- 성격: 실제 consumer shape를 재현한 deterministic link/runtime integration입니다.
- 후속 변경에서 막아야 하는 회귀: header에 non-inline external symbol을 추가해 duplicate definition을 만들거나, component header가 다른 TU의 include side effect에 기대거나, public templates의 link visibility를 잃는 회귀입니다.
- 실행 증거: source, link rule, expected 29/55 계산을 검사했으며 consumer executable은 실행하지 않았습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: compile-only acceptance를 linked consumer acceptance로 전환합니다.
- 핵심 코드와 상태 변화: 세 `.cpp` objects가 별도 compile되어 `build/consumer_test`로 link되고 `consumer` target이 실행합니다. `check`는 `test`, `headers`, `consumer`를 모두 prerequisite로 둡니다.
- 다음 commit에 넘기는 전제: 동일한 complete check surface를 instrumented build에서도 실행할 수 있어야 합니다.

### 6. build(makefile): 격리된 sanitizer 검사 대상 추가

- SHA: `1be03ae8daef`
- Importance: B
- Tags: PUBLIC_API, PRACTICAL, RISK
- Source-established role: Runs the complete acceptance surface under isolated ASan/UBSan instrumentation.
- Source summary: Adds an isolated ASan/UBSan build of the complete check suite.
- Source rationale: Separate instrumented output prevents flag mixing and broadens detection of lifetime and pointer errors. This strengthens verification infrastructure but represents standard tooling rather than a project-specific core decision.

#### 해당 SHA에서 확인할 실제 코드

- ASan/UBSan flags, debug info, frame-pointer options가 sanitizer build에만 적용되는 Makefile 경로를 확인합니다.
- recursive make가 instrumented objects를 별도 build directory에 두어 normal objects와 섞이지 않게 하는지 확인합니다.
- sanitizer target이 complete `check` surface를 다시 실행하는지 확인하고, value parity test만으로 보이지 않는 pointer/lifetime 오류 종류를 source 범위 안에서 기록합니다.
- 확인한 파일/심볼: Makefile의 `SANITIZER_FLAGS`, phony `sanitize`, recursive `$(MAKE) BUILD_DIR=$(BUILD_DIR)/sanitize CXXFLAGS="..." check`입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: complete `check` target 도입 `072c49832ddc`입니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: normal build/test/header/consumer acceptance만 있었고 instrumented output을 분리하는 경로가 없었습니다.
- 해결하려던 문제: normal flags로 결과가 맞아도 exercised path의 out-of-bounds, use-after-free, invalid lifetime, undefined arithmetic가 관찰되지 않을 수 있습니다. 같은 object directory를 flag만 바꿔 재사용하면 stale/mixed objects가 생길 수 있습니다.
- 선택한 결정: `-O1 -g -fno-omit-frame-pointer -fsanitize=address,undefined`를 recursive make의 `CXXFLAGS`에 추가하고 `BUILD_DIR=build/sanitize`로 complete `check`를 다시 빌드·실행합니다.
- 새로 생긴 책임 경계 또는 상태 변화: normal artifacts와 instrumented artifacts가 경로로 분리되며 sanitizer target은 별도 acceptance mode를 담당합니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: 같은 behavioral/header/multi-TU surface에 dynamic memory/UB diagnostics를 추가합니다.
- 핵심 코드와 상태 변화: build directory와 flags만 override하고 target graph는 `check`를 재사용합니다.
- 다음 commit에 넘기는 전제: local normal/sanitizer commands를 여러 compiler/platform에서 자동 반복해야 합니다.

### 7. ci: compiler 행렬과 sanitizer 검사 구성

- SHA: `228f457988be`
- Importance: B
- Tags: CXX98, PUBLIC_API, PRACTICAL
- Source-established role: Automates compiler, platform, and sanitizer checks in CI.
- Source summary: Runs the complete checks on GCC and Clang across Linux/macOS plus a Linux sanitizer job.
- Source rationale: The workflow makes portability and memory-safety verification repeatable at branch level. It is valuable release engineering, but it does not itself establish a container invariant.

#### 해당 SHA에서 확인할 실제 코드

- CI main matrix의 GCC/Clang Linux 및 Clang macOS 조합과 각 job이 실행하는 complete check target을 확인합니다.
- fail-fast disabled 설정과 platform-specific result가 동시에 보존되는지 확인합니다.
- separate Linux Clang sanitizer job의 leak detection/immediate failure/UB stack trace 설정과 sanitizer target 호출을 확인합니다.
- workflow 권한이 read-only로 충분한 acceptance 작업만 수행하는지 확인합니다.
- 확인한 파일/심볼: `.github/workflows/ci.yml`의 `compiler-matrix`, three include entries, `make CXX=${{ matrix.compiler }} check`, `sanitizers` job, ASAN/UBSAN environment, `make CXX=clang++ sanitize`, `permissions: contents: read`입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: local sanitizer target `1be03ae8daef`입니다.

#### 설계·상태 변화 기록

- 이 commit 직전 상태: acceptance commands는 Makefile에 있었지만 실행 여부가 개발자의 local environment에 의존했습니다.
- 해결하려던 문제: 하나의 compiler/OS에서만 통과하는 C++98 extension, warning 차이, platform-sensitive header/build issue와 sanitizer omission을 반복적으로 잡기 어렵습니다.
- 선택한 결정: push와 pull request에서 Ubuntu/g++, Ubuntu/clang++, macOS/clang++의 complete `check`를 fail-fast 없이 수행하고, 별도 Ubuntu/clang++ sanitizer job을 둡니다.
- 새로 생긴 책임 경계 또는 상태 변화: repository workflow가 cross-toolchain normal acceptance와 Linux instrumented acceptance 실행 책임을 가집니다. content write permission은 부여하지 않습니다.

#### B-level 확인

- Thread 흐름에서 맡는 구현 역할: local acceptance graph를 compiler/platform matrix와 sanitizer automation으로 승격합니다.
- 핵심 코드와 상태 변화: workflow jobs가 기존 Make targets를 그대로 호출하므로 local/CI acceptance 명령이 분기되지 않습니다.
- 다음 commit에 넘기는 전제: 이 Thread의 마지막 commit입니다. 이후 public-surface 변경은 같은 `check`/`sanitize`/CI graph에 편입되어야 합니다.

## 6. Invariant ledger

### Source에서 확정된 관련 invariant

- Every supported public header is self-contained under strict C++98 compilation, and the header-only implementation is safe to include from multiple translation units without linkage or ODR failures.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 commit | 부족함이 드러난 commit/상태 | 강화·복구한 fix | 고정한 test/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Every supported public header is self-contained under strict C++98 compilation, and the header-only implementation is safe to include from multiple translation units without linkage or ODR failures. | aggregate entry는 `80e169e83212`; map 포함은 `112af1753538` | one-TU aggregate smoke `3c64a69dd252`는 component self-containment와 link/ODR를 보지 못함 | production fix가 아니라 acceptance를 `d938c0079994`와 `072c49832ddc`로 확대 | `d938c0079994`, `072c49832ddc`, 이후 `1be03ae8daef`, `228f457988be` | first-include compile probes, separate consumer objects/link/run, isolated sanitizer target, CI matrix |

## 7. Failure → Fix → Test 연결

| 기존 상태/production change | fix 또는 verification | Source에서 확정된 연결 관점 | 실제 failure/root cause | 실제 test production path |
| --- | --- | --- | --- | --- |
| `80e169e83212` | `3c64a69dd252` | aggregate sequential surface smoke | bundle include가 component symbol/dependency를 누락할 수 있음 | aggregate include → unchanged utility/vector/stack tests |
| `112af1753538` | `d938c0079994` | aggregate expansion 이후 component self-containment 검사 | monolithic include order가 missing direct include를 가릴 수 있음 | each header first include → representative instantiation → object compile |
| `d938c0079994` | `072c49832ddc` | compile isolation에서 multi-TU link acceptance로 확대 | compile-only objects는 duplicate definitions/link visibility를 보지 못함 | vector/map objects + main → link → expected 29/55 run |
| `072c49832ddc` | `1be03ae8daef` | complete check surface를 sanitizer instrumentation으로 확대 | correct output가 invalid memory/UB를 숨길 수 있고 flag-mixed artifacts가 진단을 왜곡할 수 있음 | isolated build/sanitize → recursive complete `check` |
| `1be03ae8daef` | `228f457988be` | local acceptance를 compiler/platform/sanitizer CI로 자동화 | local 단일 toolchain 실행의 비반복성/portability blind spot | Linux GCC/Clang, macOS Clang `check`; Linux Clang `sanitize` |

## 8. Ownership / state / responsibility 변화

| 시점 | Owner / state / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `80e169e83212` | Introduces the aggregate header. | consumer가 component 조합 소유 | repository가 supported bundle include list 제공 | `ft_containers.hpp` |
| `3c64a69dd252` | Confirms the current sequential surface compiles through that bundle. | bundle은 선언만 존재 | existing test의 sole library entry로 사용 | test include replacement |
| `112af1753538` | Adds map to the aggregate public entry point. | aggregate에 associative surface 없음 | map와 relations/swap도 bundle에서 노출 | aggregate/map header diff |
| `d938c0079994` | Compiles every public header independently as the first include. | include correctness가 monolithic TU에 묶임 | 각 header가 자기 dependency 책임을 독립 probe로 부담 | `tests/headers`, `headers` target |
| `072c49832ddc` | Links independent vector and map consumers in multiple translation units. | compile-only verification | consumer objects/link/runtime가 ODR/link 책임 검증 | `tests/consumer`, `consumer`, `check` |
| `1be03ae8daef` | Runs the complete acceptance surface under isolated ASan/UBSan instrumentation. | normal artifacts만 존재 | separate instrumented artifact owner/path | recursive sanitize Make |
| `228f457988be` | Automates compiler, platform, and sanitizer checks in CI. | developer가 local 실행 책임 | workflow가 push/PR matrix 실행 책임 | CI jobs/permissions |

## 9. Thread 최종 상태

- 최종적으로 성립한 representation/state: component public headers와 aggregate `ft_containers.hpp`가 공존합니다. Makefile acceptance는 behavioral tests, independent-header objects, multi-TU linked consumer를 `check`로 묶고, 동일 graph를 separate sanitizer build에서 재실행합니다. CI는 normal compiler/platform matrix와 sanitizer job을 호출합니다.
- 최종적으로 보장하는 invariant: repository가 정의한 probes 범위에서 각 public header는 strict C++98로 first-include compile되고, header-only implementation은 representative vector/map consumer TUs를 함께 link할 수 있도록 구성됩니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: probes가 instantiate하지 않은 모든 template/type 조합, ABI/shared-library compatibility, unexercised runtime path의 memory safety를 전부 증명하지 않습니다. CI workflow 정의는 확인했지만 이 작업에서 run 결과를 검증하지 않았습니다.
- 최종 verification evidence: aggregate include diff, eight independent header probes, separate consumer compile/link/run rules와 expected 29/55, isolated ASan/UBSan recursive target, three normal matrix entries와 one sanitizer job을 코드로 확인했습니다. local checkout 제한 때문에 명령은 실행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point commit: self-containment를 분리한 `d938c0079994`, real linked consumer를 만든 `072c49832ddc`, complete acceptance를 자동화한 `228f457988be`입니다.

## 10. 최종 architecture 또는 execution flow 정리

아래 단계명은 source가 정의한 Thread progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 commit | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | failure/cleanup | 다음 단계에 남기는 invariant |
| --- | --- | --- | --- | --- | --- | --- |
| Aggregate header | `80e169e83212` | `include/ft_containers.hpp` | separate utility/vector/stack headers | supported include list를 bundle로 구성 | preprocessor/compile failure로 누락 노출 | one public entry exists |
| Aggregate smoke test | `3c64a69dd252` | `tests/test_containers.cpp` | existing test + many direct includes | sole aggregate include로 교체 | compile/test nonzero | current sequential bundle usable in one TU |
| Map public integration | `112af1753538` | aggregate/map headers | aggregate without map | map include + public comparisons/swap 노출 | compile surface | bundle includes associative surface |
| Independent component compilation | `d938c0079994` | `tests/headers`, Make `headers` | possible transitive include dependence | each header first include, instantiate, `-c` | failing object stops target | component self-containment evidence |
| Multi-TU linked consumer | `072c49832ddc` | `tests/consumer`, Make `consumer/check` | isolated objects only | separate vector/map/main objects link and run | compile/link/nonzero runtime | representative ODR/link consumer evidence |
| Isolated sanitizers | `1be03ae8daef` | Make `sanitize` | normal complete check | recursive check with sanitizer flags/new build dir | sanitizer abort/nonzero propagates | instrumented acceptance separated |
| Cross-compiler/platform CI | `228f457988be` | `.github/workflows/ci.yml` | local targets | matrix `check` + separate `sanitize` automation | jobs report independently, fail-fast false | repeatable branch-level acceptance configuration |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 source 순서대로 확인했습니다.
- [x] 각 commit 기록에 final HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A commit은 decision, failure boundary, ownership/state transition을 설명할 수 있습니다. 이 Thread에는 S/A commit이 없어 해당 항목은 비적용임을 확인했습니다.
- [x] Test/perf commit은 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → failure/risk → root cause → 수정 → regression 연결을 설명할 수 있습니다. 이 Thread는 production fix보다 acceptance 확대 순서가 중심입니다.
- [x] Invariant ledger가 commit history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] Thread 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 06-header-only-public-surface-automated-acceptance.md =====

===== BEGIN FILE: README.md =====
# ft_container Development Thread 학습 골격

## 목적

이 디렉터리는 확정된 `commit-importance.md`와 `commit-bodies.md`만을 기준으로, 실제 commit history와 해당 SHA의 코드를 직접 읽으며 `ft_container`의 개발 과정을 복원하기 위한 학습 골격입니다.

완성형 해설서가 아닙니다. 문서에 미리 적힌 내용은 source에서 이미 확정된 Thread 구조, commit metadata, 역할, 중요도, 태그, invariant와 engineering difficulty입니다. 실제 구현 해석, 함수별 추적, ownership/lifetime 확인, failure path, test 결과, 최종 설명은 학습자가 채웁니다.

## 권장 학습 순서

Development Thread의 source 순서를 그대로 따릅니다.

1. `01-cxx98-generic-interface-foundation.md`
2. `02-vector-ownership-aliasing-exception-safe-mutation.md`
3. `03-map-unbalanced-bst-to-verified-red-black-core.md`
4. `04-stable-map-iterators-through-structural-mutation.md`
5. `05-stateful-allocators-map-failure-transactions.md`
6. `06-header-only-public-surface-automated-acceptance.md`

동일한 commit이 여러 Thread에 나타나는 경우 제거하지 않고 각 Thread의 관점으로 다시 확인합니다. 이 세트에서는 source가 정의한 Thread membership과 순서를 그대로 유지합니다.

## Thread 문서 사용법

- 먼저 `Thread 목표`, `핵심 질문`, `Commit map`을 읽어 학습 범위를 고정합니다.
- 각 commit에서는 반드시 표시된 SHA 시점의 diff와 코드를 확인합니다.
- `Source-established role`, `Source summary`, `Source rationale`는 재평가하지 않습니다.
- 코드 확인 지시는 답이 아니라 탐색 범위입니다. 실제 파일, 함수, 상태 필드, branch, cleanup 순서는 직접 기록합니다.
- fix는 기존 가정 → failure/risk → root cause → 수정된 decision/invariant → 실제 수정 코드 → regression test 순으로 복원합니다.
- test/perf commit은 대상 production invariant, failure/boundary, technique, production path, 증명 범위와 비증명 범위를 분리합니다.
- Thread 마지막에는 commit별 기록을 다시 묶어 invariant ledger와 최종 execution/architecture flow를 직접 완성합니다.

## 해당 SHA 코드 확인 원칙

- final HEAD의 코드를 과거 commit 설명에 소급해서 사용하지 않습니다.
- `git show <sha> --stat`으로 변경 범위를 먼저 확인하고, `git show <sha> -- <path>`로 commit diff를 봅니다. 해당 SHA의 파일 전체 상태는 `git show <sha>:<path>`로 확인합니다.
- 수정 전후 차이가 핵심이면 해당 commit의 parent 또는 문서가 지정한 직전 관련 SHA와 비교합니다.
- later commit에서 바뀐 helper 이름이나 최종 representation을 earlier commit에 끌어오지 않습니다.
- 실제 코드 근거를 적을 때 SHA, 파일 경로, 함수/형식/테스트 이름을 함께 기록합니다.

## final HEAD 소급 사용 금지

이 골격의 목적은 완성된 코드를 설명하는 것이 아니라 설계 → 구현 → 실패 → 수정 → 검증의 발전 과정을 복원하는 것입니다. earlier commit의 부족한 상태도 그대로 읽어야 하며, later fix의 결론으로 earlier code를 정당화하거나 재해석하지 않습니다.

## S/A/B/C별 학습 깊이

- S: 핵심 architecture/invariant입니다. Problem, 기존 상태, failure 가능성, 결정, 핵심 코드, ownership/lifecycle/state transition, 후속 fix/test까지 추적합니다.
- A: 주요 subsystem, boundary, failure path, integration point입니다. 핵심 코드와 설계 판단, 관련 regression을 확인합니다.
- B: Thread 흐름에서 맡는 구현 역할과 필요한 코드/state 변화를 확인합니다.
- C: Thread 이해에 필요한 맥락만 확인합니다. S/A와 동일한 깊이의 분석을 만들지 않습니다.

## 실제 코드 삽입 기준

- 전체 파일을 복사하지 않고 결정을 설명하는 최소 코드만 삽입합니다.
- 상태 필드, 핵심 분기, ownership transfer, construct/destroy, event가 아닌 container state mutation, error/failure branch, cleanup, test injection처럼 판단에 필요한 부분을 우선합니다.
- 코드 조각마다 `SHA / file / symbol`을 식별할 수 있게 기록합니다.
- 변경 전/후가 핵심이면 두 시점의 최소 대응 코드만 나란히 기록하고 차이를 직접 설명합니다.
- source에 없는 구현 사실을 추정해서 채우지 않습니다.

## Test commit 학습 방법

- 먼저 이 test가 보호하는 production invariant를 한 문장으로 적습니다.
- 어떤 failure/boundary를 주입하거나 재현하는지 확인합니다.
- differential, failure injection, white-box, deterministic randomized, structural bound, integration compile/link 중 실제 technique을 코드로 확인합니다.
- test fixture에서 끝내지 말고 실제 production path까지 연결합니다.
- 성공 assertion이 증명하는 것과 증명하지 않는 것을 분리합니다.
- broad integration test인지 특정 regression을 고정하는 deterministic test인지 근거를 적습니다.
- 후속 변경이 어떤 회귀를 만들면 이 test가 실패해야 하는지 적습니다.

## 문서 완료 기준

- 모든 Thread commit을 source 순서대로 해당 SHA에서 확인했습니다.
- S/A commit의 핵심 decision과 failure/ownership/state transition을 실제 코드 근거로 설명할 수 있습니다.
- fix와 관련 regression test의 연결을 복원했습니다.
- invariant ledger에서 도입 → 부족함 노출 → 보강/fix → regression 고정의 변화를 설명할 수 있습니다.
- map과 vector의 핵심 lifetime/ownership/iterator/tree invariant를 source의 표현과 충돌 없이 설명할 수 있습니다.
- test가 증명하는 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- final HEAD를 earlier commit 설명에 소급 사용한 부분이 없습니다.
- 최종 architecture 또는 execution flow를 commit history 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: README.md =====

