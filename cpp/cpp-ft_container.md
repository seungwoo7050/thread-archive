===== BEGIN FILE: 01-cxx98-generic-interface-foundation.md =====
# C++98 제네릭 인터페이스 기반

## 1. 개발 흐름의 목표

### 원문에서 정한 의의

C++98에는 이후 표준 라이브러리에서 제공하는 여러 제네릭 도구가 없습니다. 이 개발 흐름은 SFINAE, `pair`, 범위 비교 알고리즘, 반복자 형식 정보와 역방향 반복자를 의존 순서대로 만들고, 엄격한 C++98 빌드로 공통 사용 규칙을 고정합니다.

<details>
<summary>영문 원문</summary>

The branch begins by constructing the generic vocabulary that C++98 does not provide in the later standard-library form used by modern code. The sequence is a real dependency chain: SFINAE separates count and range overloads, pair supplies map's value and return contracts, shared algorithms support relational operators, and iterator metadata enables one reverse adaptor to serve both pointer-backed and tree-backed iterators. The strict build then turns those assumptions into an enforceable compatibility boundary. Most individual implementations are conventional, but together they prevent each container from inventing incompatible local substitutes.

</details>


### 이 개발 흐름에서 확인할 내용

- 위 의의에 제시된 변화 과정을 각 커밋의 실제 SHA 코드로 재구성합니다.
- 원문이 확정한 커밋 역할과 중요도를 바꾸지 않고, 실제 구현/실패/테스트 근거만 직접 채웁니다.

### 원문에서 확인되는 설계

- 공용 도구는 traits, pair, 알고리즘, 반복자별 독립 헤더로 나뉩니다. 각 컨테이너 헤더는 이 도구를 사용하고, `ft_containers.hpp`는 전체 공개 API를 한 번에 포함하는 진입점을 제공합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- C++98에서 count overload와 반복자-range overload를 런타임 branching 없이 어떻게 분리하는가?
- pair, range 알고리즘, 반복자 traits가 컨테이너 내부 중복 구현을 어떻게 줄이는가?
- reverse 반복자의 base convention이 포인터-backed vector와 tree-backed map에 공통으로 적용될 수 있는 이유는 무엇인가?
- strict C++98 빌드가 utility 규약의 일부가 되는 지점은 어디인가?

## 3. 완료 기준

- A: 주요 구성 요소/경계/실패 처리/통합 point를 실제 코드와 설계 판단으로 연결하고, 관련 회귀 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- B: 개발 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- 모든 커밋은 해당 SHA의 코드 또는 테스트/빌드 diff를 근거로 기록합니다.
- 개발 흐름 최종 설명은 원문 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 커밋 간 변화로 재구성합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `ecc0668d6d9c` | `feat(type-traits): CXX98 타입 선택 도구 구현` | A | CXX98, ARCH, LEARNING | 대입 가능한 경우에만 오버로드 후보를 남기고 형식을 컴파일 시점에 분류하는 기반을 만듭니다. |
| 2 | `1c8692d14118` | `feat(pair): 값 쌍과 관계 연산 구현` | B | CXX98 | 연관 컨테이너 API에 필요한 값 쌍과 반환 형식을 제공합니다. |
| 3 | `07cf5893a53c` | `feat(algorithm): 공용 범위 비교 알고리즘 구현` | B | CXX98, PRACTICAL | 컨테이너 관계 연산에 쓰는 동등성·사전식 비교를 공용 알고리즘으로 모읍니다. |
| 4 | `e3462cec55a1` | `feat(iterator): iterator 기본 형식과 traits 정의` | B | CXX98, ITERATOR | 반복자 형식 정보와 포인터용 traits를 정의합니다. |
| 5 | `7a8e3d32bb4d` | `feat(iterator): 역방향 반복자의 양방향 동작 구현` | B | ITERATOR | 양방향 역방향 반복자의 기준 규칙을 추가합니다. |
| 6 | `ae50e9038643` | `feat(iterator): 역방향 반복자의 임의 접근 연산 완성` | B | ITERATOR | vector에 필요한 임의 접근 역방향 연산을 완성합니다. |
| 7 | `455098520e83` | `test(utils): 공용 타입·값·범위·반복자 도구 검증` | B | TEST, CXX98 | 컨테이너가 사용하기 전에 공용 도구 계층을 검증합니다. |
| 8 | `f36ec7e7e047` | `build(makefile): CXX98 검사 빌드 구성` | B | CXX98, PRACTICAL | 엄격한 경고 옵션을 적용한 C++98 빌드를 반복 가능한 기준으로 만듭니다. |

## 5. 커밋별 학습 기록

### 1. feat(type-traits): CXX98 타입 선택 도구 구현

- SHA: `ecc0668d6d9c`
- 중요도: A
- 태그: CXX98, ARCH, LEARNING
- 원문에서 정한 역할: 대입 가능한 경우에만 오버로드 후보를 남기고 형식을 컴파일 시점에 분류하는 기반을 만듭니다.
<!-- 원문 요약: Introduces `enable_if`, integral constants, and integral-type detection for C++98 template dispatch. -->
<!-- 원문 판단 근거: This is the project-wide mechanism that makes fill and range overloads coexist without runtime dispatch. It is a significant foundational interface decision, though it implements a standard utility rather than a defining container mechanism. -->

#### 해당 SHA에서 확인할 실제 코드

- 해당 SHA의 traits 전용 header에서 `enable_if`가 조건이 참일 때만 nested `type`을 노출하는 선언을 찾고, false case가 substitution에서 어떻게 사라지는지 타입 선언 수준에서 표시합니다.
- `integral_constant`, `true_type`, `false_type`의 value/type 관계와 `is_integral` primary template 및 C++98 integral specialization 목록을 확인합니다.
- 이 SHA에는 아직 실제 컨테이너 overload consumer가 없다면 그 사실을 기록하고, later HEAD의 사용처를 소급해서 설명하지 않습니다.
- 확인한 파일/심볼: `include/ft_type_traits.hpp`의 `enable_if`, `integral_constant`, `true_type`, `false_type`, `is_integral`.
- 필요한 경우 비교할 직전 관련 SHA/parent: 해당 커밋의 parent에는 이 header가 없으며, 이 SHA에서는 독립 utility 선언만 추가됩니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 프로젝트 내부에 컴파일 시점 조건으로 overload 후보를 제거하거나 integral type을 분류하는 공용 형식이 없었습니다.
- 해결하려던 문제: C++98에서 count 인자와 반복자 인자를 받는 template overload를 런타임 분기 없이 구분할 기반이 필요했습니다.
- 선택한 결정: `enable_if<false, T>`에는 `type`을 두지 않고, `enable_if<true, T>` specialization만 `type`을 노출했습니다. `is_integral`은 기본적으로 false이며 지원하는 정수 형식만 true specialization으로 열거했습니다.
- 새로 생긴 책임 경계 또는 상태 변화: overload 선택과 type 분류는 컨테이너 구현이 아니라 `ft_type_traits.hpp`가 담당하게 됐습니다. 런타임 상태와 소유권 변화는 없습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: false 조건은 컴파일 error를 직접 발생시키는 경로가 아니라 substitution 대상에서 해당 후보를 제거하는 경계입니다. 이 SHA에는 실제 consumer가 없으므로 동작 연결은 아직 선언 수준에서만 확인됩니다.
- 상태·소유권·수명 영향: 없습니다. 전부 컴파일 시점 형식과 상수입니다.
- 이 커밋이 보장하는 것과 남은 한계: 기본 SFINAE와 C++98 integral 분류를 제공합니다. cv-qualified 정수 형식 처리나 실제 fill/range overload 적용은 이 SHA에서 증명하지 않습니다.
- 다음 관련 커밋과의 연결: `pair`, 알고리즘, 반복자 utility가 같은 독립 header 계층을 확장하며, 이후 vector/map 공개 API가 이 공용 vocabulary를 소비합니다.

### 2. feat(pair): 값 쌍과 관계 연산 구현

- SHA: `1c8692d14118`
- 중요도: B
- 태그: CXX98
- 원문에서 정한 역할: 연관 컨테이너 API에 필요한 값 쌍과 반환 형식을 제공합니다.
<!-- 원문 요약: Implements `ft::pair`, converting construction, relational operators, and `make_pair`. -->
<!-- 원문 판단 근거: The type is necessary support for map values and return contracts, but the implementation follows established value-type semantics and contains limited project-specific judgment. -->

#### 해당 SHA에서 확인할 실제 코드

- `ft::pair`의 `first`/`second` 저장 형식과 default/value/converting-copy/대입 경로를 확인합니다.
- 관계 연산에서 `first`를 우선 비교하고, `first`가 서로 less가 아닐 때만 `second`를 비교하는 lexicographic 조건을 직접 추적합니다.
- `make_pair`가 호출자에게 template 인자를 명시하지 않게 하는 반환 형식 구성을 확인합니다.
- 확인한 파일/심볼: `include/ft_pair.hpp`의 `pair`, 관계 연산자, `make_pair`.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 pair header가 없고, 이 커밋이 독립 value type을 추가합니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 두 값을 묶어 map value와 `insert` 결과를 표현할 프로젝트 내부 형식이 없었습니다.
- 해결하려던 문제: 서로 다른 형식을 보존하는 두 필드, 변환 가능한 pair 간 복사, 일관된 관계 연산이 필요했습니다.
- 선택한 결정: `first_type`/`second_type`과 공개 `first`/`second`를 갖는 값 형식을 만들고, converting 생성자와 대입을 멤버 단위로 구현했습니다. `<`는 first 우선의 사전식 비교입니다.
- 새로 생긴 책임 경계 또는 상태 변화: map은 나중에 key/value 묶음과 `(iterator, bool)` 반환을 같은 공용 pair 계약으로 표현할 수 있게 됐습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: associative 컨테이너가 사용할 값/결과 형식을 먼저 고정합니다.
- 핵심 코드와 상태 변화: pair는 두 멤버를 직접 소유하며 별도 자원 관리가 없습니다. 관계 연산은 `==`와 `<`를 기준으로 나머지를 파생합니다.
- 다음 커밋에 넘기는 전제: 컨테이너 관계 연산은 pair 원소 비교와 공용 range 알고리즘을 조합할 수 있습니다.

### 3. feat(algorithm): 공용 범위 비교 알고리즘 구현

- SHA: `07cf5893a53c`
- 중요도: B
- 태그: CXX98, PRACTICAL
- 원문에서 정한 역할: 컨테이너 관계 연산에 쓰는 동등성·사전식 비교를 공용 알고리즘으로 모읍니다.
<!-- 원문 요약: Adds shared equality and lexicographical range-comparison algorithms. -->
<!-- 원문 판단 근거: Centralizing these algorithms avoids duplicated comparison logic in containers, but the change is a straightforward implementation of expected generic algorithms within an already understood design. -->

#### 해당 SHA에서 확인할 실제 코드

- `equal` 기본/함수 객체 overload가 첫 번째 범위를 한 번씩 전진하며 mismatch에서 조기 종료하는지 확인합니다.
- `lexicographical_compare`의 조기 결정 조건과 앞부분 처리 시점을 구분해서 기록합니다.
- 구현이 random-access 연산을 요구하지 않는지 실제 반복자 사용 연산만 추립니다.
- 확인한 파일/심볼: `include/ft_algorithm.hpp`의 `equal` 두 overload와 `lexicographical_compare` 두 overload.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 공용 range comparison header가 없습니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 컨테이너별 관계 연산이 생기면 범위 비교 loop를 중복 구현해야 하는 상태였습니다.
- 해결하려던 문제: 서로 다른 반복자 종류에도 적용되는 equality와 lexicographic ordering이 필요했습니다.
- 선택한 결정: `!=`, dereference, 앞부분 increment만으로 진행하는 generic loop를 두고, mismatch 또는 ordering이 결정되는 즉시 반환합니다. 한 범위가 다른 범위의 앞부분이면 종료 반복자 상태로 짧은 범위를 판정합니다.
- 새로 생긴 책임 경계 또는 상태 변화: 범위 비교 규칙은 컨테이너 밖 공용 알고리즘 header가 담당합니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: vector/map 관계 연산이 동일한 비교 규칙을 재사용하게 합니다.
- 핵심 코드와 상태 변화: 입력 범위를 읽기만 하며 반복자와 원소를 수정하지 않습니다. random-access 연산을 요구하지 않습니다.
- 다음 커밋에 넘기는 전제: 반복자가 최소한 dereference, increment, equality를 제공하면 공용 알고리즘에 참여할 수 있습니다.

### 4. feat(iterator): iterator 기본 형식과 traits 정의

- SHA: `e3462cec55a1`
- 중요도: B
- 태그: CXX98, ITERATOR
- 원문에서 정한 역할: 반복자 형식 정보와 포인터용 traits를 정의합니다.
<!-- 원문 요약: Defines the iterator base form and `iterator_traits`, including pointer specializations. -->
<!-- 원문 판단 근거: The traits layer is required by reverse iterators and containers, yet it is conventional support infrastructure rather than a project-defining architecture decision. -->

#### 해당 SHA에서 확인할 실제 코드

- `iterator` base template의 category/value/difference/포인터/reference alias와 `iterator_traits` 추출 방식을 확인합니다.
- mutable 포인터와 const 포인터 specialization이 random-access category 및 `std::ptrdiff_t`를 노출하는지 확인합니다.
- reverse 반복자가 concrete 컨테이너가 아니라 traits protocol에 의존할 수 있게 된 경계를 기록합니다.
- 확인한 파일/심볼: `include/ft_iterator.hpp`의 `iterator`, `iterator_traits<Iterator>`, `iterator_traits<T*>`, `iterator_traits<const T*>`.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 반복자 메타데이터 protocol이 없습니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: generic adaptor가 반복자의 category와 value/reference/difference type을 공통 방식으로 얻을 수 없었습니다.
- 해결하려던 문제: class 반복자와 raw 포인터를 동일한 traits protocol로 다뤄야 했습니다.
- 선택한 결정: class 반복자는 nested typedef를 추출하고, `T*`와 `const T*`는 random-access category와 `std::ptrdiff_t`를 명시하는 specialization을 제공합니다.
- 새로 생긴 책임 경계 또는 상태 변화: concrete 컨테이너가 아니라 반복자 type 자체 또는 포인터 specialization이 traversal 메타데이터를 제공합니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: reverse 반복자가 vector 포인터와 map 반복자 모두에서 형식을 추출할 기반입니다.
- 핵심 코드와 상태 변화: 컴파일 시점 aliases만 추가되며 런타임 객체 상태는 없습니다.
- 다음 커밋에 넘기는 전제: reverse adaptor는 `iterator_traits<Iterator>`만 보면 reference, 포인터, difference type을 결정할 수 있습니다.

### 5. feat(iterator): 역방향 반복자의 양방향 동작 구현

- SHA: `7a8e3d32bb4d`
- 중요도: B
- 태그: ITERATOR
- 원문에서 정한 역할: 양방향 역방향 반복자의 기준 규칙을 추가합니다.
<!-- 원문 요약: Implements bidirectional reverse-iterator construction, dereference, increment, decrement, and equality. -->
<!-- 원문 판단 근거: This establishes expected reverse traversal semantics using the base-iterator convention, but it is normal implementation work inside the utility design. -->

#### 해당 SHA에서 확인할 실제 코드

- `reverse_iterator`가 저장하는 base 반복자의 의미를 확인하고, dereference에서 base 복사본을 decrement한 뒤 접근하는 순서를 추적합니다.
- `operator++`/`operator--`가 underlying 반복자 방향을 어떻게 반대로 적용하는지 확인합니다.
- converting 생성자와 heterogeneous equality가 mutable/const 호환 타입 사이에서 어떤 변환만 허용하는지 확인합니다.
- 확인한 파일/심볼: `include/ft_iterator.hpp`의 `reverse_iterator`, `base`, `operator*`, `operator++`, `operator--`, heterogeneous `==`/`!=`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `e3462cec55a1`이 traits protocol을 먼저 제공합니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 반복자 메타데이터는 있었지만 traversal 방향을 뒤집는 공용 adaptor는 없었습니다.
- 해결하려던 문제: vector와 map이 별도 reverse 반복자를 구현하지 않고 같은 base convention을 사용해야 했습니다.
- 선택한 결정: 저장된 `_current`는 reverse 원소 자체가 아니라 그 원소 바로 다음의 forward 위치입니다. dereference는 복사한 base를 한 번 감소시켜 접근하며, reverse `++`는 base `--`, reverse `--`는 base `++`입니다.
- 새로 생긴 책임 경계 또는 상태 변화: adaptor는 base 반복자 하나만 소유하고 실제 컨테이너/element 소유권은 갖지 않습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: bidirectional traversal이 가능한 모든 반복자에 공통 reverse view를 제공합니다.
- 핵심 코드와 상태 변화: `_current`만 이동하며 underlying 순서는 변하지 않습니다. conversion 가능 여부는 base 반복자 변환 가능성에 의해 제한됩니다.
- 다음 커밋에 넘기는 전제: 포인터-backed vector를 완전히 지원하려면 arithmetic, ordering, indexing, distance가 추가돼야 합니다.

### 6. feat(iterator): 역방향 반복자의 임의 접근 연산 완성

- SHA: `ae50e9038643`
- 중요도: B
- 태그: ITERATOR
- 원문에서 정한 역할: vector에 필요한 임의 접근 역방향 연산을 완성합니다.
<!-- 원문 요약: Completes reverse-iterator random-access arithmetic, ordering, indexing, and distance. -->
<!-- 원문 판단 근거: The change makes the adaptor usable by pointer-backed vector iterators. It is technically correct supporting work, but it does not alter the project's architecture or critical ownership model. -->

#### 해당 SHA에서 확인할 실제 코드

- addition/subtraction/compound movement/indexing에서 positive reverse movement가 base subtraction으로 바뀌는 식을 확인합니다.
- 두 reverse 반복자의 distance에서 피연산자 순서가 왜 뒤집혀 있는지 코드 식 그대로 기록합니다.
- 관계 연산이 base ordering을 어떻게 반전시키는지 확인하고 vector의 포인터 반복자에서 요구되는 연산만 구분합니다.
- 확인한 파일/심볼: `include/ft_iterator.hpp`의 `operator+`, `operator-`, `operator+=`, `operator-=`, `operator[]`, 관계 연산자와 non-member distance.
- 필요한 경우 비교할 직전 관련 SHA/parent: `7a8e3d32bb4d`의 bidirectional base convention을 확장합니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: reverse traversal은 가능했지만 vector 반복자처럼 offset 이동과 ordering을 요구하는 API는 완성되지 않았습니다.
- 해결하려던 문제: reverse 방향에서 forward base의 산술과 순서가 반전된다는 규칙을 모든 random-access 연산에 일관되게 적용해야 했습니다.
- 선택한 결정: `r + n`은 `base - n`, `r - n`은 `base + n`, `lhs - rhs`는 `rhs.base() - lhs.base()`로 구현했습니다. `<`도 base의 `>`에 대응합니다.
- 새로 생긴 책임 경계 또는 상태 변화: random-access 기능은 base 반복자가 같은 연산을 제공한다는 컴파일 시점 전제를 가집니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: vector의 `rbegin`/`rend` 및 reverse random-access 표면을 완성합니다.
- 핵심 코드와 상태 변화: base 위치만 산술적으로 이동하며 element 수명이나 storage는 바꾸지 않습니다.
- 다음 커밋에 넘기는 전제: utility 테스트가 raw array/포인터를 통해 base convention과 traits를 조합 검증할 수 있습니다.

### 7. test(utils): 공용 타입·값·범위·반복자 도구 검증

- SHA: `455098520e83`
- 중요도: B
- 태그: TEST, CXX98
- 원문에서 정한 역할: 컨테이너가 사용하기 전에 공용 도구 계층을 검증합니다.
<!-- 원문 요약: Adds initial checks for pair, type traits, range algorithms, iterator traits, and reverse iteration. -->
<!-- 원문 판단 근거: The tests establish basic confidence in the utility substrate before containers depend on it, but they exercise ordinary behavior rather than a difficult invariant or regression. -->

#### 해당 SHA에서 확인할 실제 코드

- utility 테스트 executable에서 pair, integral classification, 앞부분 equality, lexicographic comparison, raw-array reverse traversal, 포인터 `iterator_traits` 각각의 검사문을 찾습니다.
- 각 검사문이 단일 utility만 보는지, 여러 utility의 조합을 보는지 구분합니다.
- 이 테스트가 컨테이너 구현 전에 진단 범위를 줄이는 baseline이라는 역할과, 컨테이너 소유권/실패를 아직 증명하지 않는다는 한계를 기록합니다.
- 확인한 파일/심볼: `tests/test_containers.cpp`의 utility 관련 테스트 함수와 `main` 호출 경로.
- 필요한 경우 비교할 직전 관련 SHA/parent: utility headers가 모두 존재하는 `ae50e9038643` 이후 상태입니다.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: pair/traits/알고리즘/reverse adaptor가 서로 호환되는 C++98 utility surface를 제공해야 합니다.
- 재현하는 실패 또는 경계: integral과 non-integral 분류, equal mismatch, lexicographic 앞부분/차이, raw-array reverse traversal, 포인터 traits 형식입니다.
- 테스트 방식: 고정 입력에 대한 결정적 unit/통합 검사문입니다. 표준 컨테이너와의 differential 테스트나 실패 주입은 아닙니다.
- 통과하는 production 코드 경로: `ft::pair` 생성·비교, `is_integral`, `equal`, `lexicographical_compare`, `iterator_traits<T*>`, `reverse_iterator<int*>`입니다.
- 이 테스트가 증명하는 것: 대표 입력에서 utility 선언이 함께 컴파일되고 예상 값을 반환합니다.
- 이 테스트가 증명하지 않는 것: 컨테이너 소유권, allocator 상태, exception 되돌리기, 모든 변환/형식 조합은 증명하지 않습니다.
- 성격: 여러 utility를 한 executable에서 확인하는 초기 broad baseline입니다. 특정 과거 버그를 재현하는 회귀는 아닙니다.
- 후속 변경에서 막아야 하는 회귀: 공개 utility 이름/형식/기본 의미가 바뀌어 이후 컨테이너 컴파일 또는 기본 비교가 깨지는 회귀입니다.
- 실행 증거: 이 작업 환경에서는 repository 체크아웃이 불가능해 executable을 실행하지 않았습니다. 위 내용은 SHA diff의 테스트 코드 검사 결과입니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: 컨테이너 구현 전 utility 문제를 독립적으로 좁혀 잡을 기준을 만듭니다.
- 핵심 코드와 상태 변화: production 상태 변화는 없고 테스트 대상이 utility API를 소비합니다.
- 다음 커밋에 넘기는 전제: strict C++98 flags와 반복 가능한 Make 대상이 이 테스트를 동일 조건으로 컴파일/run해야 합니다.

### 8. build(makefile): CXX98 검사 빌드 구성

- SHA: `f36ec7e7e047`
- 중요도: B
- 태그: CXX98, PRACTICAL
- 원문에서 정한 역할: 엄격한 경고 옵션을 적용한 C++98 빌드를 반복 가능한 기준으로 만듭니다.
<!-- 원문 요약: Creates the strict C++98 Makefile test build and ignores generated build artifacts. -->
<!-- 원문 판단 근거: This turns language-version and warning compatibility into a repeatable project constraint. It is important practical infrastructure, but not a core container mechanism. -->

#### 해당 SHA에서 확인할 실제 코드

- Makefile에서 C++98 language mode와 strict warning flags가 실제 컴파일 명령에 들어가는 지점을 확인합니다.
- 공개 헤더 dependency, 빌드 directory 분리, `test` 대상의 executable 실행 및 nonzero 중단 동작을 확인합니다.
- header-only library에서 consumer compilation 자체가 verification이 되는 경계를 기록합니다.
- 확인한 파일/심볼: `Makefile`의 `CXXFLAGS`, `CPPFLAGS`, `BUILD_DIR`, 테스트 실행 파일 rule, `test`, `clean`, `fclean`, `re`; `.gitignore`의 빌드 artifact 제외.
- 필요한 경우 비교할 직전 관련 SHA/parent: `455098520e83`의 테스트 소스는 있었지만 반복 가능한 빌드 규약이 없었습니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 테스트 소스가 존재해도 호출자마다 compiler mode와 warning 조건이 달라질 수 있었습니다.
- 해결하려던 문제: C++98 전용 공개 headers를 현대 기본 mode나 느슨한 warning 조건에서 우연히 통과시키지 않아야 했습니다.
- 선택한 결정: 컴파일 명령에 `-Wall -Wextra -Werror -std=c++98`과 `-Iinclude`를 고정하고, 모든 공개 헤더를 dependency로 둔 테스트 실행 파일을 `build/`에 생성합니다. `test` loop는 각 실행 파일을 실행하고 nonzero 즉시 종료합니다.
- 새로 생긴 책임 경계 또는 상태 변화: Makefile이 language/warning mode와 테스트 실행 순서를 반복 가능한 acceptance 경계로 소유합니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: utility API의 C++98 컴파일 가능성을 프로젝트 빌드 규칙으로 강제합니다.
- 핵심 코드와 상태 변화: 원본/header가 바뀌면 실행 파일이 재빌드되며 generated artifacts는 소스 트리와 분리됩니다.
- 다음 커밋에 넘기는 전제: 이후 vector/map/테스트가 같은 flags와 대상 convention에 편입될 수 있습니다.
- 실행 증거: Makefile 코드는 검사했으나이 환경에서는 체크아웃 실패 때문에 `make test`를 실제 실행하지 않았습니다.

## 6. 불변식 변화 기록

### 원문에서 정한 관련 불변식

- 지원 대상인 모든 공개 헤더는 엄격한 C++98 모드에서 단독으로 컴파일할 수 있어야 하며, 헤더 전용 구현은 여러 번역 단위에서 포함해도 링크 오류나 ODR 위반이 없어야 합니다.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 커밋 | 부족함이 드러난 커밋/상태 | 강화·복구한 fix | 고정한 테스트/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Every supported 공개 헤더 is self-contained under strict C++98 compilation, and the... | `ecc0668d6d9c`에서 독립 utility header 구성이 시작됨 | 이 개발 흐름 시점에는 전체 공개 헤더 단독 컴파일 및 multi-TU link 검사가 아직 없음 | 이 개발 흐름 내부 fix는 없음. 후속 개발 흐름 06의 `d938c0079994`, `072c49832ddc`가 검증 범위를 확장함 | `455098520e83`, `f36ec7e7e047` | `include/ft_*.hpp`, `tests/test_containers.cpp`, Makefile의 `-std=c++98` 테스트 rule |

## 7. 문제 → 수정 → 검증 연결

| 기존 상태/production change | fix 또는 verification | 원문에서 확정된 연결 관점 | 실제 실패/root cause | 테스트가 통과하는 실제 실행 경로 |
| --- | --- | --- | --- | --- |
| `Utility layer` | `455098520e83` | 공용 utility baseline verification | utility별 선언이 있어도 조합 컴파일과 대표 의미가 검증되지 않은 상태 | 테스트 executable이 pair/traits/알고리즘/reverse 반복자를 직접 호출 |
| `C++98 build contract` | `f36ec7e7e047` | strict C++98 compilation baseline | 수동 compiler invocation에서는 language mode와 warning 조건이 재현되지 않음 | Makefile 컴파일 rule → 테스트 실행 파일 → nonzero 즉시 중단 loop |

## 8. 소유권·상태·담당 변화

| 시점 | Owner / 상태 / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `ecc0668d6d9c` | 대입 가능한 경우에만 오버로드 후보를 남기고 형식을 컴파일 시점에 분류하는 기반을 만듭니다. | 공용 type dispatch 부재 | traits header가 SFINAE와 integral 분류 담당 | `include/ft_type_traits.hpp` |
| `1c8692d14118` | 연관 컨테이너 API에 필요한 값 쌍과 반환 형식을 제공합니다. | 두 값/결과 형식 부재 | `ft::pair`가 두 멤버와 관계 연산 소유 | `include/ft_pair.hpp` |
| `07cf5893a53c` | 컨테이너 관계 연산에 쓰는 동등성·사전식 비교를 공용 알고리즘으로 모읍니다. | 컨테이너별 비교 중복 가능 | 공용 range 알고리즘이 읽기 전용 비교 담당 | `include/ft_algorithm.hpp` |
| `e3462cec55a1` | 반복자 형식 정보와 포인터용 traits를 정의합니다. | 반복자별 메타데이터 접근 규칙 부재 | traits protocol과 포인터 specialization 확립 | `include/ft_iterator.hpp` |
| `7a8e3d32bb4d` | 양방향 역방향 반복자의 기준 규칙을 추가합니다. | reverse traversal 부재 | adaptor가 base 반복자 하나로 반대 방향 표현 | `reverse_iterator::_current`, `operator*`, `++`, `--` |
| `ae50e9038643` | vector에 필요한 임의 접근 역방향 연산을 완성합니다. | bidirectional 기능만 존재 | 산술·순서·distance가 base 반전 규칙으로 확장 | reverse random-access operators |
| `455098520e83` | 컨테이너가 사용하기 전에 공용 도구 계층을 검증합니다. | 선언만 존재 | 대표 utility composition을 테스트가 소비 | `tests/test_containers.cpp` |
| `f36ec7e7e047` | 엄격한 경고 옵션을 적용한 C++98 빌드를 반복 가능한 기준으로 만듭니다. | 수동 빌드 조건 | Makefile이 flags, artifacts, execution을 소유 | `Makefile` |

## 9. 개발 흐름의 최종 상태

- 최종적으로 성립한 표현/상태: traits, pair, range 알고리즘, 반복자 메타데이터, reverse adaptor가 서로 독립 공개 헤더로 존재하며 Makefile이 C++98 컴파일/테스트 조건을 묶습니다.
- 최종적으로 보장하는 불변식: 해당 SHA들의 코드상 utility는 서로 조합 가능한 공용 vocabulary를 제공하고 strict C++98 테스트 대상에 편입됩니다. 전체 공개 헤더 self-containment와 multi-TU ODR 안정성은 후속 개발 흐름 06에서 별도 검증됩니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: adaptor의 random-access 연산은 base 반복자가 해당 연산을 제공해야 합니다. 이 개발 흐름의 테스트는 allocator/exception/컨테이너 수명을 다루지 않습니다.
- 최종 verification 근거: `455098520e83`의 utility 검사문과 `f36ec7e7e047`의 strict 빌드·테스트 rule을 코드로 확인했습니다. 실제 실행은 수행하지 못했습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point 커밋: 컴파일 시점 dispatch 경계를 만든 `ecc0668d6d9c`와 반복 가능한 acceptance 조건을 만든 `f36ec7e7e047`입니다.

## 10. 최종 설계와 실행 흐름

아래 단계명은 원문이 정의한 개발 흐름 progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 커밋 | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | 실패/정리 | 다음 단계에 남기는 불변식 |
| --- | --- | --- | --- | --- | --- | --- |
| Overload selection / type classification | `ecc0668d6d9c` | `ft_type_traits.hpp` | template 조건과 후보 type | true specialization만 nested `type` 공개 | 런타임 정리 없음; false는 substitution에서 탈락 | 공용 컴파일 시점 dispatch 가능 |
| Pair value 규약 | `1c8692d14118` | `ft_pair.hpp` | 두 값 또는 변환 가능한 pair | 두 멤버 저장·변환·사전식 비교 | 별도 자원 없음 | map value/result 형식 준비 |
| Shared range comparison | `07cf5893a53c` | `ft_algorithm.hpp` | 두 반복자 range | mismatch/order 결정까지 순차 전진 | 변경/정리 없음 | 컨테이너 관계 연산 재사용 가능 |
| Iterator 메타데이터 | `e3462cec55a1` | `ft_iterator.hpp` | class 반복자 또는 포인터 | traits가 category/value/difference 추출 | 런타임 없음 | adaptor가 concrete 컨테이너에서 분리됨 |
| Reverse traversal convention | `7a8e3d32bb4d` | `reverse_iterator` | forward base 위치 | dereference 시 base 복사 후 감소, 이동 방향 반전 | element 소유권 없음 | bidirectional reverse traversal |
| Random-access reverse operations | `ae50e9038643` | reverse operators | base와 offset/다른 reverse 반복자 | 산술·순서·distance 피연산 방향 반전 | base precondition 위반은 별도 처리 없음 | vector reverse API 지원 |
| Utility composition 테스트 | `455098520e83` | `tests/test_containers.cpp` | 고정 utility 입력 | 검사문으로 대표 결과 검사 | 실패 시 테스트 프로세스 종료 | 컨테이너 도입 전 baseline |
| Strict C++98 빌드 baseline | `f36ec7e7e047` | `Makefile` | 원본/header/테스트 | strict flags로 빌드 후 실행 파일 loop 실행 | nonzero 즉시 중단, clean은 빌드 제거 | 반복 가능한 C++98 acceptance 경계 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 커밋 기록에 최종 HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A 커밋은 결정, 실패 경계, 소유권/상태 변화를 설명할 수 있습니다.
- [x] 테스트·성능 관련 커밋은 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → 실패/risk → root cause → 수정 → 회귀 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 커밋 history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] 개발 흐름 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 01-cxx98-generic-interface-foundation.md =====

===== BEGIN FILE: 02-vector-ownership-aliasing-exception-safe-mutation.md =====
# vector의 소유권·별칭 참조·예외 안전 변경

## 1. 개발 흐름의 목표

### 원문에서 정한 의의

vector의 핵심 문제는 인덱스 접근이 아니라 할당된 저장 공간과 실제로 생성된 객체의 수명을 정확히 구분하는 데 있습니다. 용량 계산, 빈 저장소, 자기 참조 입력, 예외를 던지는 원소, 삽입의 재할당·제자리 처리 경로를 차례로 수정하고 실패 주입으로 검증합니다.

<details>
<summary>영문 원문</summary>

The original representation is durable, but the first modifier implementations reveal why vector is primarily a lifetime-management problem rather than only an indexed sequence. Capacity overflow, null-storage arithmetic, self-aliasing, and exceptions from user-defined types each expose a different way that logical size can diverge from actual constructed objects. The thread progressively moves input capture and replacement construction before mutation, separates construction in raw slots from assignment in live slots, and adds failure-injection evidence. The final test-oracle correction also demonstrates that tests for a deliberately extended self-range contract must derive expected values independently rather than assuming another container defines the same overlap behavior.

</details>


### 이 개발 흐름에서 확인할 내용

- 위 의의에 제시된 변화 과정을 각 커밋의 실제 SHA 코드로 재구성합니다.
- 원문이 확정한 커밋 역할과 중요도를 바꾸지 않고, 실제 구현/실패/테스트 근거만 직접 채웁니다.

### 원문에서 확인되는 설계

- `ft::vector`는 할당자, 연속 저장소 포인터, 생성 완료 원소 수, 할당 용량을 소유합니다. `[data, data + size)`에는 살아 있는 객체가 있고, `[data + size, data + capacity)`는 미초기화 저장 공간이므로 생성된 객체처럼 다루면 안 됩니다.

### 원문에서 확인되는 주요 구현 난점

- vector 삽입에서 여유 용량 사용과 재할당 경로를 모두 구현하면서, 살아 있는 객체에는 대입하고 미초기화 저장 공간에는 생성해야 합니다. 별칭 입력을 보존하고 예외 뒤 부분 생성된 꼬리 객체도 정리해야 합니다.
- 사용자 정의 복사·대입·할당이 예외를 던져도 객체 수명과 소유권 불변식을 지키도록 vector의 생성·대입·크기 변경·확장 경로를 실패에 안전하게 구성해야 합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- `size`가 단순 길이가 아니라 살아 있는 객체 소유권 count가 되는 이유는 무엇인가?
- 재할당과 제자리 삽입에서 exception guarantee가 달라지는 이유를 실제 수명 작업으로 설명할 수 있는가?
- self-range/aliased value를 변경 전에 스냅샷해야 하는 정확한 invalidation 지점은 어디인가?
- `allocator::max_size()`와 null-backed empty storage가 각각 어떤 비정상 경계를 만든다?
- 최종 삽입 path에서 미초기화 저장 공간과 살아 있는 객체를 어떻게 구분해 construct/assign/destroy하는가?

## 3. 완료 기준

- S: 핵심 architecture/불변식을 직전 상태 → 실패 가능성 → 결정 → 실제 핵심 코드 → 소유권/lifecycle/상태 변화 → 후속 fix/테스트까지 코드 근거로 설명할 수 있어야 합니다.
- A: 주요 구성 요소/경계/실패 처리/통합 point를 실제 코드와 설계 판단으로 연결하고, 관련 회귀 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- 모든 커밋은 해당 SHA의 코드 또는 테스트/빌드 diff를 근거로 기록합니다.
- 개발 흐름 최종 설명은 원문 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 커밋 간 변화로 재구성합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `2eb9cb6c4273` | `feat(vector): allocator 기반 저장소 수명 관리` | S | CORE, VECTOR, ALLOCATOR | 할당자 기반 연속 저장소와 생성 완료 객체의 소유 범위를 정합니다. |
| 2 | `9db46550b13d` | `feat(vector): 용량 확장과 원소 재배치 구현` | A | VECTOR, ALLOCATOR, CORE | 새 저장소를 완성한 뒤 교체하는 재할당과 기하급수적 용량 증가를 추가합니다. |
| 3 | `6f3cbf4794c9` | `fix(vector): 용량 계산을 allocator 상한에서 포화` | A | VECTOR, ALLOCATOR, EDGE | 할당자 상한에서 용량 증가 계산이 넘치지 않도록 수정합니다. |
| 4 | `bdc4c3123bc9` | `fix(vector): 자기 범위 assign과 insert 입력 보존` | A | VECTOR, EDGE, DEBUG | 변경 전에 자기 범위 입력을 별도 저장해 무효화를 막습니다. |
| 5 | `bc3a74b9342e` | `fix(vector): allocator 형식과 빈 반복자 연산 보정` | A | VECTOR, ALLOCATOR, EDGE | 빈 저장소에서 포인터 산술을 제거하고 할당자가 제공하는 크기 형식을 사용합니다. |
| 6 | `b3124b3808d5` | `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리` | A | VECTOR, EXCEPTION, ALLOCATOR | 생성·대입·크기 확장·별칭 참조 `push_back`을 실패에 안전한 교체 방식으로 바꿉니다. |
| 7 | `9051be26db5e` | `test(vector): 생성·대입·크기 변경 실패 주입` | A | TEST, VECTOR, EXCEPTION | 원소 연산과 할당 실패를 주입해 되돌리기와 살아 있는 객체 수를 검증합니다. |
| 8 | `797c33904db3` | `fix(vector): fill·range 삽입의 객체 수명 보존` | S | CORE, VECTOR, HARD | fill/range 삽입을 살아 있는 객체 영역과 미초기화 저장 공간으로 구분해 다시 구현합니다. |
| 9 | `8df3d8e067c0` | `test(vector): 삽입 복사·대입·할당 실패 sweep` | A | TEST, VECTOR, EXCEPTION | 용량 여유 여부 두 분기에서 삽입 복사·대입·할당 실패를 모두 순회 검증합니다. |
| 10 | `5bdb6eb81a89` | `test(vector): 자기 범위 기대값을 명시적 snapshot으로 구성` | A | TEST, VECTOR, DEBUG | 신뢰하기 어려운 참조 컨테이너 호출 대신 명시적 스냅샷으로 기대값을 만듭니다. |

## 5. 커밋별 학습 기록

### 1. feat(vector): allocator 기반 저장소 수명 관리

- SHA: `2eb9cb6c4273`
- 중요도: S
- 태그: CORE, VECTOR, ALLOCATOR
- 원문에서 정한 역할: 할당자 기반 연속 저장소와 생성 완료 객체의 소유 범위를 정합니다.
<!-- 원문 요약: Establishes vector's allocator-backed contiguous storage, size/capacity state, construction rollback, and destruction path. -->
<!-- 원문 판단 근거: This commit defines the ownership representation on which every later vector operation depends: allocated storage is distinct from constructed elements, and cleanup is centralized. Omitting it would leave a major gap in explaining the container's architecture and lifetime model. -->

#### 원문에서 정한 핵심 판단

- 문제: C++98 vector는 연속된 미초기화 저장 공간을 소유하면서, 어느 위치에 실제 `T` 객체가 생성되었는지 별도로 추적해야 합니다. 메모리를 할당했다고 객체가 생기는 것은 아니며, 생성되지 않은 슬롯을 소멸하거나 생성된 객체의 소멸을 빠뜨려서는 안 됩니다.
- 결정: vector 상태를 할당자, 데이터 포인터, 생성 완료 원소 수, 할당 용량으로 나눕니다. fill 생성은 블록 하나를 할당한 뒤 원소를 순서대로 생성하고, 중간에 실패하면 완성된 앞부분만 역순으로 소멸한 뒤 블록을 해제합니다. 소멸 경로는 살아 있는 원소 정리와 저장 공간 해제를 한곳에서 수행합니다.
- 중요한 이유: 이후의 reserve, resize, 대입, 삽입, 삭제, swap, 예외 복구는 모두 저장 공간 소유권과 객체 수명을 분리한 이 표현에 의존합니다. 후속 수정은 이 불변식을 교체하지 않고 구체화합니다.

#### 해당 SHA에서 확인할 실제 코드

- 해당 SHA의 첫 vector header에서 allocator, storage 포인터, live-element count, capacity를 나타내는 핵심 상태 필드를 찾습니다.
- raw 할당과 element 생성이 분리되는 helper/생성자 경로를 따라가며, `_size`가 언제 증가하는지 확인합니다.
- fill 생성 중 copy/construct가 실패할 때 이미 생성된 앞부분을 역순 destroy하고 block을 deallocate하는 실패 브랜치를 추적합니다.
- 소멸자/정리 helper가 정확히 live 앞부분만 destroy한 뒤 storage를 해제하는지 확인합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 `_alloc`, `_data`, `_size`, `_capacity`, fill 생성자/`_assign_fill`, `_destroy_elements`, 소멸자.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 vector header가 없으며 이 커밋이 표현을 처음 추가합니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: vector 객체, 연속 storage 소유권, 살아 있는 객체 count가 전혀 정의되지 않았습니다.
- 해결하려던 문제: allocator가 반환한 raw memory와 실제로 생성자가 완료된 `T` 객체를 분리해 추적해야 했습니다.
- 선택한 결정: 한 할당 block을 `_data`가 소유하고, `_capacity`는 slot 수, `_size`는 `[0, _size)`에 존재하는 살아 있는 객체 수로 정의했습니다. 생성 성공 직후에만 `_size`를 증가시킵니다.
- 새로 생긴 책임 경계 또는 상태 변화: allocator는 block과 element 생성/소멸을 수행하고, vector는 그 호출 순서와 완료된 앞부분 길이를 소유합니다.

#### 중요도 S 심화 추적

- 기존 설계가 충분하지 않았던 이유: 이 커밋 이전에는 설계 자체가 없었습니다. `capacity`를 곧 live count로 취급하면 uninitialized slot을 destroy하거나 생성에 실패한 slot까지 size가 가리키게 됩니다.
- 핵심 상태/소유권/수명 transition: `NULL/0/0` → block allocate 후 아직 살아 있는 객체 0개 → element별 construct 성공마다 `_size` 증가 → 완성된 `_data/_size/_capacity`; 실패하면 완성 앞부분을 역순 destroy하고 block deallocate 후 예외 재전파입니다.
- 실패 상황별 정리/되돌리기: allocate 실패 전에는 소유권을 얻지 않습니다. construct 실패 후에는 `[0, _size)`만 destroy하며 할당 전체를 같은 `_alloc`로 해제합니다.
- 이 커밋이 보장하는 것: 정상 생성/소멸과 fill 생성 실패에서 live 앞부분과 raw tail을 구분합니다.
- 이 커밋이 아직 보장하지 않는 것: capacity growth, self-aliasing, empty null arithmetic, 일반 modifier의 exception transaction, 삽입 수명은 아직 없습니다.
- 후속 fix/테스트와 연결되는 구조: 모든 후속 helper가 `_size`를 커밋 marker로 사용합니다. `9051be26db5e`가 live count/invalid destroy를 계측하고 `797c33904db3`가 삽입에 같은 경계를 적용합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: 네 상태 필드와 생성 loop의 성공 후 `_size` 증가, catch의 앞부분 destroy/deallocate 순서입니다.

### 2. feat(vector): 용량 확장과 원소 재배치 구현

- SHA: `9db46550b13d`
- 중요도: A
- 태그: VECTOR, ALLOCATOR, CORE
- 원문에서 정한 역할: 새 저장소를 완성한 뒤 교체하는 재할당과 기하급수적 용량 증가를 추가합니다.
<!-- 원문 요약: Adds capacity queries, reserve, geometric growth, and exception-aware reallocation. -->
<!-- 원문 판단 근거: Reallocation is the central dynamic mechanism of vector: values must be copied into a new block before the old block is released. The decision is significant, though later commits strengthen arithmetic and transactional 경계 조건s. -->

#### 해당 SHA에서 확인할 실제 코드

- `reserve`, `max_size`, growth helper, relocation helper의 호출자/피호출자 관계를 그립니다.
- 새 block allocate → 기존 element copy-construct → 성공 후 old element destroy/deallocate → 상태 publish 순서를 확인합니다.
- 복사 생성 실패 시 new block의 constructed 앞부분만 정리되고 original storage가 그대로 소유자인지 확인합니다.
- 이 SHA의 doubling arithmetic가 allocator limit에서 아직 hardened되지 않았다는 원문 명시를 기록하고, 후속 `6f3cbf4794c9`에서 같은 계산을 비교합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 `capacity`, `max_size`, `reserve`, `_next_capacity`, `_reallocate`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `2eb9cb6c4273`의 fixed block 표현과 비교합니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 초기 할당 이후 capacity를 늘리거나 원소를 다른 block으로 옮길 경로가 없었습니다.
- 해결하려던 문제: 기존 값과 소유권을 잃지 않고 더 큰 연속 block으로 교체해야 했습니다.
- 선택한 결정: 새 block을 allocate하고 기존 live 앞부분을 copy-construct합니다. 전부 성공한 뒤 기존 살아 있는 객체와 block을 정리하고 새 포인터/capacity를 publish합니다.
- 새로 생긴 책임 경계 또는 상태 변화: 재할당 helper가 old/new block이 동시에 존재하는 transaction 구간과 최종 소유권 transfer를 담당합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: 복사 생성자가 중간에 던지는 경우입니다. new block의 완료 앞부분만 destroy/deallocate하며 old 상태는 건드리지 않습니다.
- 상태·소유권·수명 영향: 성공 전에는 old vector가 소유자이고 new block은 helper의 임시 소유입니다. 성공 시 old를 release하고 `_data/_capacity`를 교체합니다. `_size` 값은 동일합니다.
- 이 커밋이 보장하는 것과 남은 한계: 재할당 생성 실패의 강한 예외 보장을 제공합니다. 하지만 doubling 계산이 `max_size()` 인접 구간에서 오버플로할 위험은 남습니다.
- 다음 관련 커밋과의 연결: `6f3cbf4794c9`가 growth arithmetic을 allocator 상한에서 포화시킵니다.

### 3. fix(vector): 용량 계산을 allocator 상한에서 포화

- SHA: `6f3cbf4794c9`
- 중요도: A
- 태그: VECTOR, ALLOCATOR, EDGE
- 원문에서 정한 역할: 할당자 상한에서 용량 증가 계산이 넘치지 않도록 수정합니다.
<!-- 원문 요약: Makes vector length checks and growth arithmetic saturate safely at the allocator's `max_size()`. -->
<!-- 원문 판단 근거: The fix closes a non-obvious unsigned-overflow boundary that could select an invalid capacity or report the wrong failure. It restores an important allocator-limit contract without changing the core representation. -->

#### 해당 SHA에서 확인할 실제 코드

- `resize`, fill 대입, 삽입, growth helper에서 `max_size()`를 넘는 요청을 변경 전에 거부하는 브랜치를 확인합니다.
- doubling을 먼저 수행하지 않고 allocator limit에서 saturate하는 조건식과, validated minimum을 선택하는 순서를 추적합니다.
- 직전 관련 구현 `9db46550b13d`와 비교해 unsigned capacity arithmetic의 실패 window가 정확히 어디서 사라졌는지 기록합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 요청 길이 preflight와 `_next_capacity`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `9db46550b13d`의 먼저 doubling하는 계산.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: 현재 capacity를 두 배로 만든 뒤 allocator limit과 비교해도 안전하다고 가정했습니다.
- 실제 실패 또는 위험: unsigned multiplication/addition이 먼저 wrap되면 작은 capacity가 선택되거나 잘못된 실패 경로로 들어갈 수 있습니다.
- root cause: arithmetic 자체가 유효한지 확인하기 전에 오버플로 가능한 표현식을 평가했습니다.
- 수정된 불변식/결정: requested minimum을 먼저 `max_size()`와 비교하고, doubling 가능 여부를 `limit - _capacity` 형태로 판단해 계산 자체가 오버플로하지 않게 합니다. 두 배가 불가능하면 limit에서 포화합니다.
- 변경/커밋 순서의 변경: invalid length는 할당이나 상태 변경 전에 `length_error`로 거부됩니다.
- 실제 수정 코드 근거: `_next_capacity`의 `if (_capacity > limit - _capacity)` 계열 분기와 각 공개 modifier의 size preflight입니다.
- 연결되는 회귀 테스트 SHA/근거: 원본이 연결한 `0ce21f9cf12d`의 bounded allocator가 max 5에서 포화와 `reserve(6)` 거부, 최종 outstanding block 0을 검사합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: allocator가 작은 `max_size()`를 반환하거나 capacity가 limit 절반을 넘는 경계입니다.
- 상태·소유권·수명 영향: 잘못된 요청을 변경 전에 거부하므로 기존 block과 살아 있는 객체는 변하지 않습니다.
- 이 커밋이 보장하는 것과 남은 한계: capacity 선택 arithmetic은 allocator limit 안에서 안전합니다. 실제 할당 실패는 여전히 allocator 예외로 처리됩니다.
- 다음 관련 커밋과의 연결: 이후 삽입/resize transaction은 이 validated capacity를 전제로 새 block을 준비합니다.

### 4. fix(vector): 자기 범위 assign과 insert 입력 보존

- SHA: `bdc4c3123bc9`
- 중요도: A
- 태그: VECTOR, EDGE, DEBUG
- 원문에서 정한 역할: 변경 전에 자기 범위 입력을 별도 저장해 무효화를 막습니다.
<!-- 원문 요약: Snapshots range input before vector assign or insert mutates the source container. -->
<!-- 원문 판단 근거: The prior implementation could invalidate its own iterators or read overwritten values during self-range modification. Separating input capture from mutation is a significant aliasing correction that establishes the library's chosen snapshot contract. -->

#### 해당 SHA에서 확인할 실제 코드

- range assign이 destination clear/변경 전에 원본 range 전체를 독립 임시 객체 vector로 materialize하는지 확인합니다.
- range insert가 원본 range와 삽입 point 이후 뒷부분을 변경 전에 어떤 독립 storage에 보존하는지 확인합니다.
- self-reference일 때 invalidation 이전에 입력을 모두 소비한다는 순서를 표시하고, reconstruction이 여전히 incremental 변경이라는 한계도 기록합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 range `assign`, range `insert`, 입력 스냅샷 임시 객체와 tail 스냅샷.
- 필요한 경우 비교할 직전 관련 SHA/parent: self-range를 직접 소비하던 직전 구현 및 회귀 `61e8b46e668f`.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: `first`/`last`가 destination과 무관하다고 보고 clear/shift/재할당 중에도 계속 읽었습니다.
- 실제 실패 또는 위험: 원본이 같은 vector이면 clear, overwrite 또는 재할당 순간 반복자가 무효화되거나 아직 읽지 않은 값이 바뀝니다.
- root cause: 입력 consumption과 destination 변경이 같은 시간 구간에 섞여 있었습니다.
- 수정된 불변식/결정: 변경 전에 원본 range를 독립 임시 객체에 모두 복사합니다. insert는 원본뿐 아니라 위치 이후 뒷부분도 보존한 뒤 reconstruction합니다.
- 변경/커밋 순서의 변경: capture 완료 → destination 변경/rebuild 순으로 분리됩니다.
- 실제 수정 코드 근거: range overload 첫 부분의 임시 객체 vector 생성과 insert tail 스냅샷입니다.
- 연결되는 회귀 테스트 SHA/근거: `61e8b46e668f`가 self-range assign/insert 결과를 검사하고, `5bdb6eb81a89`가 그 expected oracle을 독립 스냅샷 방식으로 교정합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: destination range가 원본 range와 동일하거나 부분 중첩될 때입니다.
- 상태·소유권·수명 영향: 스냅샷이 완성되기 전 destination은 변하지 않습니다. 스냅샷 자체의 생성 실패는 임시 객체 소멸자가 정리합니다.
- 이 커밋이 보장하는 것과 남은 한계: chosen 스냅샷 semantics의 입력 보존을 보장합니다. 이 시점 insert reconstruction은 여전히 공개 operations를 통한 incremental 변경이라 일반 exception guarantee는 완성되지 않았습니다.
- 다음 관련 커밋과의 연결: `797c33904db3`가 스냅샷을 유지하면서 삽입 수명 알고리즘 자체를 재설계합니다.

### 5. fix(vector): allocator 형식과 빈 반복자 연산 보정

- SHA: `bc3a74b9342e`
- 중요도: A
- 태그: VECTOR, ALLOCATOR, EDGE
- 원문에서 정한 역할: 빈 저장소에서 포인터 산술을 제거하고 할당자가 제공하는 크기 형식을 사용합니다.
<!-- 원문 요약: Uses allocator-provided size types and avoids pointer arithmetic/subtraction on an unallocated empty vector. -->
<!-- 원문 판단 근거: The fix addresses both custom-allocator interface correctness and undefined empty-storage arithmetic. Although compact, it restores a fundamental boundary invariant used by end, insert, and erase. -->

#### 해당 SHA에서 확인할 실제 코드

- 공개 `size_type`/`difference_type`가 allocator-provided type으로 바뀐 선언을 찾습니다.
- `_iterator_at`, `_index_of`, empty-range erase에서 null-backed empty 상태를 별도 처리하는 브랜치를 확인합니다.
- 해당 SHA의 empty `begin()==end()`, zero-count insert, empty erase 경로에서 null 포인터 addition/subtraction이 실제로 사라졌는지 추적합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 allocator typedefs, `_iterator_at`, `_index_of`, empty erase/no-op modifier branches.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent의 `_data + index`, `position - _data` 직접 연산과 회귀 `ccb98587e777`.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: empty vector의 `_data == NULL`이어도 0을 더하거나 두 null 포인터를 빼는 식이 harmless하다고 가정했습니다.
- 실제 실패 또는 위험: C++ 포인터 arithmetic/subtraction은 유효한 array object 범위라는 전제가 있어 null-backed empty 상태에는 적용할 수 없습니다. custom allocator의 size/difference type도 무시됐습니다.
- root cause: 논리적으로 0이라는 결과와 language-level 포인터 작업의 유효성을 동일시했습니다.
- 수정된 불변식/결정: empty/null 상태는 explicit 브랜치로 반복자/index를 반환하고, 공개 size types는 allocator가 제공하는 형식을 사용합니다.
- 변경/커밋 순서의 변경: zero-count/empty-range modifier는 포인터 계산 전에 no-op로 종료됩니다.
- 실제 수정 코드 근거: `_iterator_at`과 `_index_of`의 `_data == NULL` 검사, empty erase 브랜치입니다.
- 연결되는 회귀 테스트 SHA/근거: `ccb98587e777`가 empty begin/end, zero-count insert, empty range erase와 allocator 상태를 검사합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: default-constructed vector와 모든 원소를 지운 뒤 null storage인 상태입니다.
- 상태·소유권·수명 영향: no-op 경로가 storage나 live count를 건드리지 않습니다.
- 이 커밋이 보장하는 것과 남은 한계: empty API 경로에서 invalid 포인터 arithmetic을 제거합니다. dereferenceable 반복자를 제공하는 것은 아니며 empty begin/end는 비교만 가능합니다.
- 다음 관련 커밋과의 연결: 이후 transaction/삽입 helper가 위치를 index로 바꿀 때 이 safe conversion을 사용합니다.

### 6. fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리

- SHA: `b3124b3808d5`
- 중요도: A
- 태그: VECTOR, EXCEPTION, ALLOCATOR
- 원문에서 정한 역할: 생성·대입·크기 확장·별칭 참조 `push_back`을 실패에 안전한 교체 방식으로 바꿉니다.
<!-- 원문 요약: Makes fill construction, assignment, resize growth, and aliased push-back use rollback or temporary-storage transactions. -->
<!-- 원문 판단 근거: This substantially raises vector's failure guarantees by committing state only after construction succeeds and by snapshotting aliased values before reallocation. It is significant core hardening, but insertion requires a separate, still more defining lifetime redesign. -->

#### 원문에서 정한 핵심 판단

- 문제: 초기 vector 구현은 대체 원소를 모두 생성하기 전에 기존 원소를 지우거나 크기를 일부 늘릴 수 있었습니다. 원소 복사 중 예외가 발생하면 기존 대입 대상이 사라지거나, 일부만 생성된 뒷부분이 남거나, 재할당으로 무효화된 자기 참조 `push_back` 인자를 다시 읽을 수 있었습니다.
- 결정: fill 생성은 완전한 블록을 만든 뒤에만 객체 상태에 반영합니다. fill·range 대입은 임시 vector를 완성한 뒤 저장 공간을 교환합니다. `resize` 확장은 기존 크기를 기록하고 실패 시 새로 생성한 뒷부분만 소멸합니다. 재할당이 필요한 `push_back`은 `reserve`가 별칭 참조를 무효화하기 전에 인자 값을 복사합니다.
- 중요한 이유: 대체 상태를 먼저 준비하고, 생성 완료 범위를 추적하며, 새 상태가 유효할 때만 반영하는 방식을 여러 위험 경로에 일관되게 적용했습니다. 삽입 문제를 모두 해결한 것은 아니지만, 후속 삽입 재설계가 따르는 실패 처리 원칙을 확립했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `_initialize_fill`가 complete block을 만들기 전 `_data/_size/_capacity`를 publish하지 않는지 확인합니다.
- fill/range 대입이 destination allocator를 유지한 임시 객체 vector를 만들고 storage-only exchange로 커밋하는 순서를 추적합니다.
- `resize` growth에서 old size를 기준으로 새 뒷부분만 되돌리기하는 브랜치를 확인합니다.
- 재할당이 필요한 `push_back(values[i])` 같은 aliased argument를 `reserve` 전에 복사하는 코드 위치를 확인합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 `_initialize_fill`, 대입 overloads, `_swap_storage`, `resize`, `push_back`.
- 필요한 경우 비교할 직전 관련 SHA/parent: 기존 clear-then-빌드/partial growth 경로와 후속 `9051be26db5e`.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: destination 상태를 먼저 비우거나 `_size`를 진행시키면서 replacement를 만들었습니다. push argument가 자기 원소여도 reserve 후 읽을 수 있다고 가정했습니다.
- 실제 실패 또는 위험: copy/construct 예외로 원래 대입 대상이 사라지거나 partially grown 뒷부분이 남고, aliased reference는 재할당 후 dangling이 됩니다.
- root cause: prepare와 커밋이 분리되지 않았고, 외부처럼 보이는 reference가 내부 storage를 가리킬 가능성을 고려하지 않았습니다.
- 수정된 불변식/결정: replacement 임시 객체/block을 완성한 뒤 storage만 교환합니다. resize는 old size를 커밋 기준으로 삼고, push는 재할당 전에 value 스냅샷을 만듭니다.
- 변경/커밋 순서의 변경: prepare/스냅샷 → construct with progress count → success 후 publish; 실패 시 newly constructed 뒷부분/block만 정리합니다.
- 실제 수정 코드 근거: `_initialize_fill`의 local 포인터/progress, 대입의 임시 객체 + `_swap_storage`, resize catch, push 스냅샷입니다.
- 연결되는 회귀 테스트 SHA/근거: `9051be26db5e`가 fill/copy 대입, resize, aliased push-back에 copy/할당 실패를 주입합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: replacement 생성, resize 뒷부분 생성, reserve 중 할당/copy, self-aliased push argument입니다.
- 상태·소유권·수명 영향: `_size`는 성공한 살아 있는 객체만 포함하며, failed 임시 객체가 자신의 allocator로 block을 정리합니다. destination allocator object는 대입 커밋에서 바뀌지 않습니다.
- 이 커밋이 보장하는 것과 남은 한계: 여러 일반 modifier에서 original 보존 또는 정확한 뒷부분 되돌리기를 제공합니다. 복잡한 fill/range 삽입의 제자리 수명은 아직 해결되지 않았습니다.
- 다음 관련 커밋과의 연결: `797c33904db3`가 같은 transaction 원리를 삽입 재할당에 적용하고 제자리 경로는 별도 basic guarantee로 설계합니다.

### 7. test(vector): 생성·대입·크기 변경 실패 주입

- SHA: `9051be26db5e`
- 중요도: A
- 태그: TEST, VECTOR, EXCEPTION
- 원문에서 정한 역할: 원소 연산과 할당 실패를 주입해 되돌리기와 살아 있는 객체 수를 검증합니다.
<!-- 원문 요약: Adds tracked-object and allocator failure injection for construction, assignment, resize, and aliased push-back. -->
<!-- 원문 판단 근거: The suite validates live-object counts, invalid copies/destructions, block release, and preservation of original values. It materially changes confidence in vector's lifetime guarantees rather than merely adding normal-path coverage. -->

#### 해당 SHA에서 확인할 실제 코드

- `tracked_value`가 살아 있는 객체 수, dead-원본 copy, duplicate 소멸, 선택적 copy/대입 throw를 어떻게 기록하는지 확인합니다.
- tracking allocator가 outstanding block, 할당 실패, small `max_size`를 독립적으로 계측하는지 확인합니다.
- fill 생성/대입/copy 대입/resize growth/aliased push_back 실패가 어떤 실제 실행 경로를 통과하는지 각각 연결합니다.
- 각 실패 뒤 original value 보존 여부와 live/block count zero 조건을 분리해서 기록합니다.
- 확인한 파일/심볼: `tests/test_vector_exceptions.cpp`의 `tracked_value`, tracking/bounded allocator 상태, 각 `test_*` 함수.
- 필요한 경우 비교할 직전 관련 SHA/parent: production fix `b3124b3808d5`; allocator-limit 회귀에는 `6f3cbf4794c9`가 연결됩니다.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: `_size`는 살아 있는 객체 수와 일치하고 실패한 생성/대입/growth가 block 또는 object를 누수하거나 invalid destroy하지 않아야 합니다.
- 재현하는 실패 또는 경계: 지정한 copy/대입 attempt와 할당 attempt에서 예외, 작은 `max_size`, aliased push-back입니다.
- 테스트 방식: shared counters를 이용한 결정적 실패 주입과 수명 instrumentation입니다.
- 통과하는 production 코드 경로: fill/복사 생성, fill/range/copy 대입, resize growth, reserve/push-back, allocator limit checks입니다.
- 이 테스트가 증명하는 것: 주입 지점에서 예외가 발생해도 expected original value가 보존되는 경로, live address 집합, invalid copy/destroy count, outstanding block count가 일관됩니다.
- 이 테스트가 증명하지 않는 것: 모든 element type/allocator 구현, 모든 possible 실패 interleaving, 삽입의 복잡한 제자리 브랜치는 아직 증명하지 않습니다.
- 성격: 특정 transaction fix를 고정하는 결정적 회귀 테스트 모음입니다.
- 후속 변경에서 막아야 하는 회귀: 상태를 성공 전에 publish하거나 failed 앞부분/뒷부분/block 정리를 누락하고, 재할당 뒤 aliased value를 읽는 회귀입니다.
- 실행 증거: 테스트 소스와 production diff만 검사했습니다. repository 체크아웃 제약으로 실행 파일은 실행하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: allocator가 아예 block을 주지 않는 경우와 element 생성 후 일부 progress가 있는 경우를 분리합니다.
- 상태·소유권·수명 영향: `tracked_value`의 live-address registry와 allocator block counter가 서로 다른 차원의 leak/invalid 수명을 검출합니다.
- 이 커밋이 보장하는 것과 남은 한계: 검사한 결정적 지점의 되돌리기를 증명합니다. sanitizer나 실제 표준 allocator 전 조합을 대체하지 않습니다.
- 다음 관련 커밋과의 연결: 삽입 redesign 뒤에는 `8df3d8e067c0`이 같은 기법을 fill/range insert 양쪽 capacity 브랜치로 확장합니다.

### 8. fix(vector): fill·range 삽입의 객체 수명 보존

- SHA: `797c33904db3`
- 중요도: S
- 태그: CORE, VECTOR, HARD
- 원문에서 정한 역할: fill/range 삽입을 살아 있는 객체 영역과 미초기화 저장 공간으로 구분해 다시 구현합니다.
<!-- 원문 요약: Replaces vector fill/range insertion with explicit reallocation and in-place algorithms that distinguish live objects from uninitialized storage. -->
<!-- 원문 판단 근거: This solves a project-defining object-lifetime problem. The new paths track constructed tails, snapshot aliased input, preserve spare capacity when possible, and clean partial construction on failure; without it the completed vector's correctness under non-trivial element types cannot be explained. -->

#### 원문에서 정한 핵심 판단

- 문제: 최초 삽입 알고리즘은 이미 객체가 존재하는 위치와 아직 객체가 없는 여유 저장 공간을 일관되게 구분하지 않은 채 생성과 소멸을 섞어 원소를 이동했습니다. range 삽입도 여러 공개 연산을 조합해 뒷부분을 다시 만들었습니다. 복사나 대입이 예외를 던지는 비단순 원소 형식에서는 객체 수명 추적이 깨지고, 생성된 객체가 누수되거나 vector가 잘못된 상태로 남을 수 있었습니다.
- 결정: 입력 형태와 여유 용량 여부에 따라 삽입 경로를 나눕니다. 재할당 경로는 새 블록에 기존 앞부분, 삽입값, 기존 뒷부분을 모두 생성한 뒤에만 교체합니다. 제자리 경로는 미초기화된 꼬리 슬롯에만 객체를 생성하고, 이미 객체가 있는 구간에는 대입합니다. 새로 생성한 꼬리 객체 수를 기록해 실패 시 그 범위만 소멸하며, fill 값과 range 입력은 변경 전에 별도 저장합니다.
- 중요한 이유: 연속 저장 형태, 여유 용량 보존, 별칭 참조, 생성과 대입의 구분, 예외 뒤 정리를 하나의 삽입 구현 안에서 함께 해결한 핵심 설계입니다. 이 구분 없이는 일반적인 원소 형식에 대해 vector가 올바르다고 보기 어렵습니다.

#### 해당 SHA에서 확인할 실제 코드

- fill/range 각각에서 재할당 path와 spare-capacity 제자리 path가 분리된 helper 구조를 찾습니다.
- 재할당 path의 앞부분 → inserted values → 뒷부분 생성 순서와 complete 후에만 기존 저장소를 교체하는 커밋 경계를 추적합니다.
- 제자리 path에서 raw tail slot에는 `construct`, 이미 live인 위치에는 대입을 사용하는 구간 경계를 인덱스로 표시합니다.
- 부분 tail 생성 또는 대입 실패 시 실제로 destroy되는 객체 범위와 `_size` bookkeeping을 확인하고 strong/basic guarantee를 구분합니다.
- fill value와 range 입력이 변경 전에 스냅샷되는 위치를 확인합니다.
- 확인한 파일/심볼: `include/ft_vector.hpp`의 fill/range `insert`, 재할당 helpers, 제자리 helpers, `_replace_storage`/tail 정리 경로.
- 필요한 경우 비교할 직전 관련 SHA/parent: `bdc4c3123bc9`의 incremental reconstruction, transaction pattern `b3124b3808d5`, 회귀 `8df3d8e067c0`.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: move 대상 slot이 live인지 raw인지와 무관하게 construct/destroy를 섞고, range tail을 여러 공개 modifier로 다시 만들었습니다.
- 실제 실패 또는 위험: 살아 있는 객체 위에 placement 생성, raw slot에 대입, partially constructed tail leak, `_size`와 실제 살아 있는 객체 불일치가 가능합니다.
- root cause: 삽입을 단일 shift 문제로 취급하고 object 수명 영역 `[0,size)`와 raw 영역 `[size,capacity)`의 다른 연산 규칙을 분리하지 않았습니다.
- 수정된 불변식/결정: capacity 조건으로 reallocate/제자리를 나누고, 제자리에서도 `count <= tail_size`와 `count > tail_size`를 구분합니다. raw tail에만 construct하고 live range에는 대입합니다.
- 변경/커밋 순서의 변경: 입력 스냅샷 → 필요한 raw tail 생성(progress 기록) → live 대입/shift → 최종 `_size` publish입니다. 재할당은 new block 전체 완성 후 old 교체입니다.
- 실제 수정 코드 근거: 앞부분/inserted/뒷부분 생성 loop, `constructed` progress counter, catch의 tail destroy, 성공 마지막의 `_size += count`입니다.
- 연결되는 회귀 테스트 SHA/근거: `8df3d8e067c0`이 copy/대입/할당 실패 positions를 sweep하고 capacity/address/수명 counters를 검사합니다.

#### 중요도 S 심화 추적

- 기존 설계가 충분하지 않았던 이유: trivial `int`에서는 byte-like shift가 통과해도 non-trivial `T`의 생성자/소멸자 규약에는 맞지 않았습니다. 공개 operations를 조합하면 중간 상태가 외부 exception에 노출됩니다.
- 핵심 상태/소유권/수명 transition: 재할당은 old 소유자 유지 + new 임시 객체 block → 앞부분/insert/뒷부분 live count 증가 → 완성 후 old destroy/deallocate → new publish입니다. 제자리는 기존 live 앞부분 유지 → raw tail 일부 construct → live slots 대입 → size 커밋입니다.
- 실패 상황별 정리/되돌리기: 재할당 copy/construct 실패는 new 앞부분만 destroy/deallocate하고 old unchanged입니다. 제자리 tail 생성 실패는 새 tail만 destroy하며 original size를 유지합니다. live 대입 실패는 이미 바뀐 값은 복구하지 못하지만 constructed tail을 정리하고 `_size`를 old size로 유지해 컨테이너를 destructible/usable한 valid 상태로 둡니다.
- 이 커밋이 보장하는 것: raw/live 영역에 맞는 수명 작업, aliased 입력 capture, 재할당 강한 예외 보장, 제자리 수명 bookkeeping의 basic guarantee, spare capacity 유지입니다.
- 이 커밋이 아직 보장하지 않는 것: throwing 대입을 포함한 제자리 삽입에서 original 값의 강한 예외 보장은 제공하지 않습니다. arbitrary invalid 반복자는 precondition 밖입니다.
- 후속 fix/테스트와 연결되는 구조: `8df3d8e067c0`이 두 브랜치와 실패 종류를 직접 sweep하고, `5bdb6eb81a89`가 self-range expected oracle 신뢰성을 고칩니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: `_size` 전후의 live/raw 경계, tail 생성 progress, 재할당 publish 경계, 제자리 대입 실패 시 보장 수준입니다.

### 9. test(vector): 삽입 복사·대입·할당 실패 sweep

- SHA: `8df3d8e067c0`
- 중요도: A
- 태그: TEST, VECTOR, EXCEPTION
- 원문에서 정한 역할: 용량 여유 여부 두 분기에서 삽입 복사·대입·할당 실패를 모두 순회 검증합니다.
<!-- 원문 요약: Sweeps copy, assignment, and allocation failures across fill and range insertion, including aliasing and spare-capacity behavior. -->
<!-- 원문 판단 근거: The test matrix probes both reallocation and in-place branches, checks post-failure usability, and verifies no leaked or double-destroyed objects. It provides high-value regression evidence for the most complex vector modifier. -->

#### 해당 SHA에서 확인할 실제 코드

- copy/대입/할당 실패 위치를 sweep하는 반복 구조와 각 injection counter reset 지점을 확인합니다.
- spare-capacity range 삽입에서 capacity와 unaffected 앞부분 주소가 유지되는 검사문을 찾습니다.
- reallocating 삽입 sweep에서 허용되는 post-상태가 unchanged original 또는 full inserted result뿐인지 확인합니다.
- 각 scenario 종료 후 살아 있는 객체, invalid 수명 작업, outstanding 할당이 zero인지 확인합니다.
- 확인한 파일/심볼: `tests/test_vector_exceptions.cpp`의 삽입 테스트 함수, 실패 counter loops, capacity/address 검사문, final tracker 검사문.
- 필요한 경우 비교할 직전 관련 SHA/parent: production redesign `797c33904db3`.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: fill/range 삽입이 재할당과 spare-capacity 양쪽에서 live/raw 수명을 지키고 실패 후 소유 자원을 정확히 정리해야 합니다.
- 재현하는 실패 또는 경계: fill value alias, self/range 스냅샷, new-block 복사 생성, 제자리 tail 생성, live 대입, allocator 할당 실패입니다.
- 테스트 방식: fail-at index를 증가시키는 결정적 실패 sweep과 수명/할당 instrumentation입니다.
- 통과하는 production 코드 경로: fill/range 재할당 helpers와 제자리 helpers의 두 tail-size 브랜치입니다.
- 이 테스트가 증명하는 것: 재할당 실패는 unchanged/full-result만 남고, 제자리 대입 실패에서도 size/수명 bookkeeping과 이후 사용 가능성이 유지되며, leak/double-destroy가 없습니다. spare capacity와 unaffected 앞부분 address도 지정 scenario에서 유지됩니다.
- 이 테스트가 증명하지 않는 것: 모든 possible `T` semantics나 무한 실패 positions, throwing 소멸자, invalid 입력 반복자는 다루지 않습니다.
- 성격: 가장 복잡한 modifier의 결정적 회귀 테스트/실패-injection matrix입니다.
- 후속 변경에서 막아야 하는 회귀: raw slot 대입, live slot construct, size 조기 publish, partial tail 정리 누락, 불필요한 재할당, aliased 입력 재독취입니다.
- 실행 증거: 코드 검토만 수행했습니다. 실제 sweep 실행 파일 결과를 생성하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: 실패가 new block publish 전인지, raw tail 생성 중인지, live 대입 중인지에 따라 보장 수준이 다릅니다.
- 상태·소유권·수명 영향: trackers가 object address와 할당 block을 따로 세어 logical size만으로 놓칠 수 있는 문제를 검출합니다.
- 이 커밋이 보장하는 것과 남은 한계: 구현이 의도한 strong/basic guarantee 분리를 테스트합니다. 성능 복잡도나 모든 반복자 invalidation 규칙은 별도입니다.
- 다음 관련 커밋과의 연결: self-range scenario의 expected result가 독립적인지 `5bdb6eb81a89`에서 교정됩니다.

### 10. test(vector): 자기 범위 기대값을 명시적 snapshot으로 구성

- SHA: `5bdb6eb81a89`
- 중요도: A
- 태그: TEST, VECTOR, DEBUG
- 원문에서 정한 역할: 신뢰하기 어려운 참조 컨테이너 호출 대신 명시적 스냅샷으로 기대값을 만듭니다.
<!-- 원문 요약: Builds self-range expected values explicitly instead of asking `std::vector` to perform overlapping modifiers. -->
<!-- 원문 판단 근거: The previous oracle coupled the regression to implementation- or version-sensitive overlapping-range behavior. This correction makes the chosen snapshot contract independently testable and restores trust in a significant edge-case test. -->

#### 해당 SHA에서 확인할 실제 코드

- self-range 삽입 expected 순서를 unchanged 앞부분 + snapshotted 원본 + unchanged 뒷부분으로 직접 조립하는 코드를 확인합니다.
- self-range 대입 expected 순서를 selected interior range에서 독립적으로 만드는 코드를 확인합니다.
- `std::vector`에 overlapping modifier를 호출하던 이전 oracle을 제거한 diff를 직전 parent와 비교합니다.
- 이 테스트가 프로젝트-defined 스냅샷 semantics를 검증하지만 일반적인 모든 overlapping 작업의 표준 의미를 증명하는 것은 아니라는 범위를 기록합니다.
- 확인한 파일/심볼: `tests/test_containers.cpp` 또는 해당 vector 회귀 원본의 self-range expected 생성.
- 필요한 경우 비교할 직전 관련 SHA/parent: `61e8b46e668f`의 reference-컨테이너 modifier oracle.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: 프로젝트가 선택한 self-range 스냅샷 규약에서 destination 변경 전 원본 values가 고정돼야 합니다.
- 재현하는 실패 또는 경계: 삽입 원본/destination이 같은 vector이고 원본이 위치 전후와 중첩되는 경우, interior self-range 대입입니다.
- 테스트 방식: expected 순서를 앞부분/원본 스냅샷/뒷부분으로 직접 만드는 independent oracle입니다.
- 통과하는 production 코드 경로: `bdc4c3123bc9` 이후 range 입력 스냅샷과 `797c33904db3`의 range 삽입 구현입니다.
- 이 테스트가 증명하는 것: 구현 결과가 명시한 스냅샷 semantics와 일치합니다.
- 이 테스트가 증명하지 않는 것: `std::vector`의 overlapping modifier 표준 의미나 모든 임의 중첩 조합을 증명하지 않습니다.
- 성격: 기존 회귀 테스트 자체의 oracle 결함을 고친 결정적 테스트 correction입니다.
- 후속 변경에서 막아야 하는 회귀: expected 값을 구현과 같은 잘못된 modifier 호출로 계산해 양쪽이 함께 틀리거나 platform별 동작에 의존하는 회귀입니다.
- 실행 증거: 수정된 expected-building 코드를 diff로 확인했으며 실행하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: production 실패가 아니라 verification 신뢰성의 경계입니다. oracle이 정의되지 않은/지원 범위 밖 동작에 기대면 테스트 pass가 근거가 되지 않습니다.
- 상태·소유권·수명 영향: production 상태 변화는 없지만 self-aliasing 규약의 기대 순서가 독립적으로 고정됩니다.
- 이 커밋이 보장하는 것과 남은 한계: 이 테스트의 expected 값이 프로젝트 규약에서 직접 유도됩니다. 전체 standard conformance oracle은 아닙니다.
- 다음 관련 커밋과의 연결: vector 개발 흐름의 최종 근거에서 self-range 결과와 실패/수명 근거를 분리해 신뢰할 수 있게 합니다.

## 6. 불변식 변화 기록

### 원문에서 정한 관련 불변식

- `[0, size)`의 모든 vector 원소는 정확히 한 번 생성되고 한 번 소멸해야 합니다. `size` 이후의 저장 공간은 명시적으로 생성하기 전까지 미초기화 상태여야 합니다.
- vector 저장소는 해당 블록을 할당한 상태와 호환되는 할당자로 해제해야 합니다. 할당이나 생성이 실패해도 블록이 누수되거나 `size`가 생성되지 않은 객체를 소유한다고 표시해서는 안 됩니다.
- 재할당은 대체 블록이 완성된 뒤에만 상태를 바꿉니다. 기존 값 보존이 요구되는 연산은 생성 실패 시 원래 vector를 유지해야 하며, 제자리 연산도 최소한 객체 수명 추적을 일관되게 유지해 컨테이너를 안전하게 소멸할 수 있어야 합니다.
- 범위 대입과 삽입은 원본 vector를 변경하기 전에 자기 참조 입력을 별도 저장해야 합니다. 그래야 원본 반복자와 별칭 값이 모두 사용되기 전에 무효화되지 않습니다.
- 요청 크기와 용량 증가 계산은 부호 없는 정수 오버플로와 잘못된 용량 선택 없이 `allocator::max_size()`를 지켜야 합니다.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 커밋 | 부족함이 드러난 커밋/상태 | 강화·복구한 fix | 고정한 테스트/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Every vector element in `[0, size)` is constructed exactly once and destroyed exactly once | `2eb9cb6c4273` | 초기 insert/general 변경이 live/raw slot을 일관되게 구분하지 않음 | `b3124b3808d5`, 결정적으로 `797c33904db3` | `9051be26db5e`, `8df3d8e067c0` | 상태 fields, 생성 progress, tail 정리, `_size` 커밋 |
| Vector storage is deallocated by allocator 상태 compatible with the 상태 that allocated it | `2eb9cb6c4273` | replacement/임시 객체 실패에서 block 정리를 검증할 근거 부족 | `9db46550b13d`, `b3124b3808d5`, `797c33904db3` | `0ce21f9cf12d`, `9051be26db5e`, `8df3d8e067c0` | `_alloc.allocate/deallocate`, tracking allocator block counter |
| Reallocation 커밋 only after the replacement block is complete | `9db46550b13d` | 대입/resize/push/insert에 같은 transaction이 일관되게 적용되지 않음 | `b3124b3808d5`, `797c33904db3` | `9051be26db5e`, `8df3d8e067c0` | new block 앞부분/insert/뒷부분 생성 후 `_replace_storage` |
| Range 대입 and 삽입 must 스냅샷 self-referential 입력 before mutating the 원본 vector | `bdc4c3123bc9` | 초기 회귀 oracle가 reference 컨테이너의 overlapping modifier에 의존 | production은 `797c33904db3`에서 유지, oracle은 `5bdb6eb81a89`에서 교정 | `61e8b46e668f`, `5bdb6eb81a89` | 임시 객체 원본 capture, explicit 앞부분/원본/뒷부분 expected values |
| Requested vector sizes and growth calculations must respect `allocator::max_size()` without unsigned 오버플로 | `6f3cbf4794c9` | `9db46550b13d`의 doubling-first arithmetic | `6f3cbf4794c9` | `0ce21f9cf12d` 및 `9051be26db5e` bounded allocator | `limit - _capacity` 포화 분기와 length preflight |

## 7. 문제 → 수정 → 검증 연결

| 기존 상태/production change | fix 또는 verification | 원문에서 확정된 연결 관점 | 실제 실패/root cause | 테스트가 통과하는 실제 실행 경로 |
| --- | --- | --- | --- | --- |
| `6f3cbf4794c9` | `0ce21f9cf12d` | bounded allocator로 capacity saturation/length rejection/block 정리 검증 | doubling 오버플로와 limit 초과 요청이 변경 전 분리되지 않을 위험 | bounded `max_size` → growth/reserve preflight → final block counter |
| `bdc4c3123bc9` | `61e8b46e668f` | self-range insert/assign 결과 회귀 검증 | 변경 중 원본 반복자/value invalidation | range 스냅샷 → assign/insert result comparison |
| `bc3a74b9342e` | `ccb98587e777` | null-storage no-op modifier와 allocator 상태 검증 | null 포인터 addition/subtraction | empty begin/end, zero insert, empty erase no-op 브랜치 |
| `b3124b3808d5` | `9051be26db5e` | 생성/대입/resize/aliased push-back 실패 주입 | 상태 조기 publish와 incomplete 정리 | tracked copy/assign/alloc throw → 되돌리기 → value/live/block 검사문 |
| `797c33904db3` | `8df3d8e067c0` | insert copy/대입/할당 실패 sweep | live/raw slot 혼동과 partial tail leak | 재할당/제자리 fill/range helpers → fail-at sweep |
| `bdc4c3123bc9 → 61e8b46e668f` | `5bdb6eb81a89` | self-range oracle를 explicit 스냅샷 expected value로 교정 | oracle가 overlapping reference modifier에 의존 | explicit expected assembly → production self-range result comparison |

- `0ce21f9cf12d`, `61e8b46e668f`, `ccb98587e777`는 이 개발 흐름의 커밋 map에는 포함되지 않지만, 원문이 확정한 관련 회귀 커밋입니다. 개발 흐름 membership을 변경하지 않고 fix의 검증 연결만 기록합니다.

## 8. 소유권·상태·담당 변화

| 시점 | Owner / 상태 / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `2eb9cb6c4273` | 할당자 기반 연속 저장소와 생성 완료 객체의 소유 범위를 정합니다. | vector 상태 부재 | `_data` block과 `_size` live 앞부분 분리 | vector fields, 생성자/소멸자 |
| `9db46550b13d` | 새 저장소를 완성한 뒤 교체하는 재할당과 기하급수적 용량 증가를 추가합니다. | fixed capacity | new block 임시 소유 후 성공 시 transfer | `_reallocate`, `reserve` |
| `6f3cbf4794c9` | 할당자 상한에서 용량 증가 계산이 넘치지 않도록 수정합니다. | 오버플로 가능한 doubling | limit-safe saturated capacity | `_next_capacity`, preflight |
| `bdc4c3123bc9` | 변경 전에 자기 범위 입력을 별도 저장해 무효화를 막습니다. | 원본과 destination 변경 결합 | 임시 객체가 입력 수명 소유 | range assign/insert 스냅샷 |
| `bc3a74b9342e` | 빈 저장소에서 포인터 산술을 제거하고 할당자가 제공하는 크기 형식을 사용합니다. | null 포인터 산술 가능 | empty 브랜치와 allocator typedefs | `_iterator_at`, `_index_of` |
| `b3124b3808d5` | 생성·대입·크기 확장·별칭 참조 `push_back`을 실패에 안전한 교체 방식으로 바꿉니다. | destination 조기 변경 | 임시 객체/new 뒷부분이 커밋 전 책임 | `_initialize_fill`, `_swap_storage`, resize/push catch |
| `9051be26db5e` | 원소 연산과 할당 실패를 주입해 되돌리기와 살아 있는 객체 수를 검증합니다. | 실패 동작 추론 | 테스트 상태가 objects/blocks를 계측 | `tracked_value`, tracking allocator |
| `797c33904db3` | fill/range 삽입을 살아 있는 객체 영역과 미초기화 저장 공간으로 구분해 다시 구현합니다. | 삽입 shift가 수명 영역 혼동 | reallocate/제자리 helper가 각 영역 책임 | 삽입 helpers, `constructed` counter |
| `8df3d8e067c0` | 용량 여유 여부 두 분기에서 삽입 복사·대입·할당 실패를 모두 순회 검증합니다. | 일부 normal-path 근거 | 브랜치/실패별 회귀 matrix | fail-at loops, capacity/address/수명 검사문 |
| `5bdb6eb81a89` | 신뢰하기 어려운 참조 컨테이너 호출 대신 명시적 스냅샷으로 기대값을 만듭니다. | oracle가 다른 modifier 동작에 의존 | 테스트가 expected 순서를 직접 소유 | explicit expected assembly |

## 9. 개발 흐름의 최종 상태

- 최종적으로 성립한 표현/상태: vector는 allocator, contiguous block 포인터, live 앞부분 count, raw slot capacity를 소유합니다. 모든 modifier는 `[0,size)`와 `[size,capacity)`의 연산 규칙을 구분합니다.
- 최종적으로 보장하는 불변식: construct 성공한 object만 size에 포함되고 같은 allocator family가 block을 해제합니다. 재할당은 replacement 완성 후 커밋하며, self-referential 입력은 변경 전에 스냅샷됩니다. growth는 `max_size()`를 오버플로 없이 존중합니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: 제자리 삽입 중 element 대입이 던지면 값의 강한 예외 보장은 없습니다. 대신 newly constructed tail을 정리하고 old size를 유지하는 basic 수명 guarantee입니다. invalid 반복자와 throwing 소멸자는 지원 범위 밖입니다.
- 최종 verification 근거: tracked object/allocator 실패 주입, 삽입 sweep, bounded allocator, empty-상태 회귀, independent self-range oracle를 코드로 확인했습니다. 이 작업에서는 테스트 executable을 실제 실행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point 커밋: 표현을 만든 `2eb9cb6c4273`, 일반 transaction을 만든 `b3124b3808d5`, 삽입 수명을 결정적으로 고친 `797c33904db3`입니다.

## 10. 최종 설계와 실행 흐름

아래 단계명은 원문이 정의한 개발 흐름 progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 커밋 | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | 실패/정리 | 다음 단계에 남기는 불변식 |
| --- | --- | --- | --- | --- | --- | --- |
| Allocation 소유권 model | `2eb9cb6c4273` | vector fields/construct/destruct | allocator와 element count | raw block allocate 후 성공 element만 size 증가 | completed 앞부분 destroy + block deallocate | size == live count |
| Replacement-block 재할당 | `9db46550b13d` | `reserve`, `_reallocate` | old live block, larger capacity | new block copy 완료 후 소유자 교체 | new 앞부분 되돌리기, old unchanged | 재할당 커밋 경계 |
| Allocator-limit growth | `6f3cbf4794c9` | `_next_capacity`, preflights | requested minimum/current capacity/limit | 오버플로 없는 saturation 및 reject | 변경 전 `length_error` | capacity <= max_size |
| Aliased range capture | `bdc4c3123bc9` | range assign/insert | 원본이 destination일 수 있음 | 임시 객체에 원본과 필요 tail capture | 임시 객체 생성 실패 시 destination unchanged | 변경 전 입력 고정 |
| Empty-storage 경계 | `bc3a74b9342e` | `_iterator_at`, `_index_of` | `_data == NULL`, size 0 | 포인터 산술 없이 empty 결과 반환 | no-op | empty begin/end/modifier 유효 |
| Transactional general 변경 | `b3124b3808d5` | init/assign/resize/push | replacement 또는 뒷부분 필요 | prepare → construct → publish | 임시 객체/뒷부분 되돌리기 | 일반 변경의 커밋-after-success |
| Failure injection | `9051be26db5e` | vector exception 테스트 | 선택된 fail attempt | production 예외 경로 실행 및 counters 관찰 | scope 종료 후 live/block zero | 되돌리기 근거 |
| Insertion 수명 redesign | `797c33904db3` | insert helpers | 위치/count/range/capacity | reallocate 또는 raw-tail construct + live 대입 | new block/tail 정리, size 마지막 publish | 삽입 수명 유효 |
| Insertion 실패 sweep | `8df3d8e067c0` | 삽입 regressions | copy/assign/alloc fail index | 모든 주요 브랜치 반복 실행 | trackers와 post-상태 검사 | 복잡한 modifier 회귀 고정 |
| Independent self-range oracle | `5bdb6eb81a89` | self-range expected builder | original values와 selected range | 앞부분 + 스냅샷 + 뒷부분 직접 조립 | production 변경 사용 안 함 | 테스트 oracle 독립성 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 커밋 기록에 최종 HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A 커밋은 결정, 실패 경계, 소유권/상태 변화를 설명할 수 있습니다.
- [x] 테스트·성능 관련 커밋은 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → 실패/risk → root cause → 수정 → 회귀 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 커밋 history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] 개발 흐름 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 02-vector-ownership-aliasing-exception-safe-mutation.md =====

===== BEGIN FILE: 03-map-unbalanced-bst-to-verified-red-black-core.md =====
# 비균형 검색 트리에서 검증된 레드-블랙 트리 map으로

## 1. 개발 흐름의 목표

### 원문에서 정한 의의

일반 이진 검색 트리만으로도 정렬된 공개 동작은 만들 수 있지만 최악의 경우 로그 시간 복잡도를 보장할 수 없습니다. 삽입·삭제 보정으로 레드-블랙 트리를 완성한 뒤, 공개 출력 비교뿐 아니라 부모 링크·색상 규칙·검정 높이·도달 가능성·트리 높이와 비교 횟수까지 검사합니다.

<details>
<summary>영문 원문</summary>

The public map surface first exists as an ordinary BST, which is sufficient to establish comparator semantics and observable ordering but not the required worst-case complexity. Insertion balancing supplies the first red-black mechanism; deletion then adds the more difficult state restoration for removed black nodes and null replacements. The later tests intentionally move beyond output comparison: a white-box inspector proves parent links, header state, red constraints, black height, and node reachability, while complexity tests prove that the structure remains logarithmic under ascending, descending, and shuffled input. The sequence distinguishes functional ordering from structural correctness and asymptotic correctness.

</details>


### 이 개발 흐름에서 확인할 내용

- 위 의의에 제시된 변화 과정을 각 커밋의 실제 SHA 코드로 재구성합니다.
- 원문이 확정한 커밋 역할과 중요도를 바꾸지 않고, 실제 구현/실패/테스트 근거만 직접 채웁니다.

### 원문에서 확인되는 설계

- `ft::map`은 할당자로 만든 노드에 값을 저장하고 이를 레드-블랙 트리로 연결합니다. 값을 저장하지 않는 헤더 센티널이 루트·최솟값·최댓값 링크를 보유하며 안정적인 `end()` 표현도 담당합니다.

### 원문에서 확인되는 주요 구현 난점

- 레드-블랙 삽입과 삭제를 구현해야 합니다. 특히 검정 노드를 지운 뒤 대체 노드가 null일 수 있으므로, 형제 노드·재색칠·회전의 대칭 사례를 처리하는 동안 부모 정보를 명시적으로 유지해야 합니다.
- 정렬된 공개 출력만으로는 확인할 수 없는 색상 규칙, 검정 높이, 부모 링크, 헤더의 최솟값·최댓값, 노드 도달 가능성, 반복자 안정성, 로그 높이 상한을 별도로 검증해야 합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- ordinary BST가 공개 ordering은 만족하면서도 map의 asymptotic requirement를 위반할 수 있는 이유는 무엇인가?
- red-black 삽입과 deletion이 각각 어떤 상태를 보존/복구해야 하는가?
- null replacement가 있는 deletion에서 explicit parent context가 필요한 이유는 무엇인가?
- sorted 출력 parity만으로 증명할 수 없는 structural property는 무엇이며 inspector가 무엇을 추가로 증명하는가?
- height/comparator upper bound가 timing 벤치마크보다 어떤 회귀를 안정적으로 잡는가?

## 3. 완료 기준

- A: 주요 구성 요소/경계/실패 처리/통합 point를 실제 코드와 설계 판단으로 연결하고, 관련 회귀 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- B: 개발 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- S: 핵심 architecture/불변식을 직전 상태 → 실패 가능성 → 결정 → 실제 핵심 코드 → 소유권/lifecycle/상태 변화 → 후속 fix/테스트까지 코드 근거로 설명할 수 있어야 합니다.
- 모든 커밋은 해당 SHA의 코드 또는 테스트/빌드 diff를 근거로 기록합니다.
- 개발 흐름 최종 설명은 원문 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 커밋 간 변화로 재구성합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `2c6dd8acdd20` | `feat(map): 노드 소유권과 빈 tree 상태 구현` | A | RB_TREE, ALLOCATOR, ARCH | 노드를 개별 할당하고 루트 하나가 전체 트리를 소유하는 기반을 만듭니다. |
| 2 | `c0fdb8e3f84c` | `feat(map): 삽입과 첨자 및 복사 동작 구현` | B | RB_TREE | 비교자로 정렬되는 BST 삽입·첨자 접근·복사를 추가합니다. |
| 3 | `0f70c1fcc520` | `feat(map): 삭제와 clear 및 swap 구현` | B | RB_TREE, ALLOCATOR | 일반 BST 삭제와 트리 교환을 추가합니다. |
| 4 | `f29fd8a91523` | `feat(map): 레드-블랙 삽입 회전과 색 보정 구현` | S | CORE, RB_TREE, HARD | 회전과 삽입 후 색상 보정을 도입합니다. |
| 5 | `8f8b67961819` | `test(map): 정렬 입력 삽입과 검색 경계 stress 검증` | B | TEST, RB_TREE | 정렬 입력 삽입과 조회 경계를 불리한 입력으로 검증합니다. |
| 6 | `a055cb19500b` | `feat(map): 레드-블랙 삭제 보정 구현` | S | CORE, RB_TREE, HARD | 삭제 시 이식 상태와 이중 검정 보정을 추가합니다. |
| 7 | `86922f1ddfa0` | `test(map): 반복 삭제·복사·대입·교환 stress 검증` | B | TEST, RB_TREE | `std::map`과 비교하며 반복 삭제·복사·대입·교환을 스트레스 검증합니다. |
| 8 | `cd67e6a31bb7` | `test(map): 무작위 연산마다 레드-블랙 불변식 검증` | A | TEST, RB_TREE, RISK | 결정적 무작위 연산 뒤 모든 레드-블랙 트리 불변식을 검사합니다. |
| 9 | `cd8ebbb2c01e` | `perf(map): 높이와 비교 횟수 회귀 상한 추가` | A | PERF, RB_TREE, TEST | 트리 높이와 비교 횟수의 상한을 강제합니다. |

## 5. 커밋별 학습 기록

### 1. feat(map): 노드 소유권과 빈 tree 상태 구현

- SHA: `2c6dd8acdd20`
- 중요도: A
- 태그: RB_TREE, ALLOCATOR, ARCH
- 원문에서 정한 역할: 노드를 개별 할당하고 루트 하나가 전체 트리를 소유하는 기반을 만듭니다.
<!-- 원문 요약: Introduces map's individually allocated nodes, parent/child links, null-root empty state, and recursive ownership cleanup. -->
<!-- 원문 판단 근거: This is the associative container's foundational ownership substrate and enables stable value addresses. It is significant, but the final defining architecture arrives later with red-black balancing and a value-free header sentinel. -->

#### 해당 SHA에서 확인할 실제 코드

- map node의 parent/left/right link와 stored key/value의 위치, root/size empty 상태를 확인합니다.
- value allocator에서 node type으로 rebound된 allocator가 node allocate/construct/destroy/deallocate에 사용되는 경로를 추적합니다.
- node 생성 throw 시 할당을 즉시 해제하는 실패 브랜치와 recursive clear의 single-root 소유권 경로를 확인합니다.
- 이 SHA의 tree가 아직 unbalanced이며 node allocator 상태 생성은 후속 fix 대상이라는 원본-defined 한계를 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 nested `node`, `_root`, `_size`, `_alloc`, `_node_alloc`, `_create_node`, `_destroy_node`, recursive `_clear`.
- 필요한 경우 비교할 직전 관련 SHA/parent: parent에는 map header가 없고 이 커밋이 초기 소유권 표현을 추가합니다. allocator 상태 문제는 `ae180871b160`에서 고칩니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: map object와 node 소유권이 정의되지 않았습니다.
- 해결하려던 문제: key/value의 안정된 주소를 유지하면서 각 tree node를 독립적으로 생성·파괴하고, root 하나에서 전체 소유권을 도달 가능하게 해야 했습니다.
- 선택한 결정: node가 value와 parent/left/right link를 함께 보유하고 map이 `_root`와 `_size`를 소유합니다. value allocator를 node type으로 rebind해 node 단위로 allocate/construct합니다.
- 새로 생긴 책임 경계 또는 상태 변화: map의 root가 모든 reachable node의 단일 소유권 entry가 되고, `_clear`가 post-order로 subtree를 해제합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: allocate 성공 뒤 node 생성이 던지는 경우입니다. `_create_node`가 방금 받은 block을 즉시 deallocate하고 예외를 재전파합니다.
- 상태·소유권·수명 영향: successful node만 tree에 연결되며 clear는 child subtree를 먼저 해제한 뒤 해당 node를 destroy/deallocate합니다.
- 이 커밋이 보장하는 것과 남은 한계: 개별 node 수명과 빈 `_root == NULL`, `_size == 0` 상태를 정의합니다. tree는 unbalanced이고 `_node_alloc`을 기본 생성해 stateful value allocator와 상태가 분리되는 결함이 있습니다.
- 다음 관련 커밋과의 연결: `c0fdb8e3f84c`가 comparator search와 link를 추가하고, `f29fd8a91523`이 color/rotation을 추가합니다. allocator 상태는 개발 흐름 05에서 교정됩니다.

### 2. feat(map): 삽입과 첨자 및 복사 동작 구현

- SHA: `c0fdb8e3f84c`
- 중요도: B
- 태그: RB_TREE
- 원문에서 정한 역할: 비교자로 정렬되는 BST 삽입·첨자 접근·복사를 추가합니다.
<!-- 원문 요약: Adds unbalanced BST insertion, `operator[]`, range/copy construction, and copy assignment. -->
<!-- 원문 판단 근거: These operations establish map's observable uniqueness and insertion semantics, but they operate within an unbalanced and weakly transactional design that later commits replace or harden. -->

#### 해당 SHA에서 확인할 실제 코드

- 삽입 search가 comparator로 왼쪽/오른쪽을 결정하고, 양방향 less가 모두 false일 때 duplicate로 판단하는 조건을 확인합니다.
- `operator[]`가 default mapped value를 만들어 삽입 path를 재사용하는 호출 관계를 확인합니다.
- range/복사 생성과 대입이 visible element 삽입을 어떻게 반복하는지 추적합니다.
- node 할당 뒤 parent side를 결정하기 위해 comparator를 다시 호출하는 위치를 표시하고, 후속 `cb08194d17b0`의 fix 대상임을 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `insert`, `operator[]`, range/복사 생성자, `operator=`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `2c6dd8acdd20`의 node-only 상태; comparator-after-할당 fix `cb08194d17b0`.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: nodes를 만들고 지울 수는 있지만 key로 위치를 찾거나 공개 삽입/copy를 수행할 수 없었습니다.
- 해결하려던 문제: comparator만으로 strict ordering과 key equivalence를 정의하고 map API가 tree search를 재사용해야 했습니다.
- 선택한 결정: `_comp(value.first, current.first)`이면 left, 반대 비교면 right, 둘 다 false면 duplicate입니다. `operator[]`는 `(key, mapped_type())`를 insert하고 returned 반복자의 second를 반환합니다.
- 새로 생긴 책임 경계 또는 상태 변화: comparator가 tree ordering/equivalence를 결정하고 삽입이 root 또는 parent child link와 `_size`를 갱신합니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: balanced 여부와 무관하게 map의 observable uniqueness, sorted traversal에 필요한 BST ordering을 확립합니다.
- 핵심 코드와 상태 변화: duplicate에서는 할당 없이 기존 node를 반환하고, 새 key는 search parent 아래 link 후 size 증가입니다. range/copy는 같은 공개 삽입을 반복합니다.
- 다음 커밋에 넘기는 전제: erase는 node identity와 parent/child links를 기반으로 transplant할 수 있습니다. 다만 할당 뒤 comparator 재호출과 destructive 대입은 후속 exception fix 대상입니다.

### 3. feat(map): 삭제와 clear 및 swap 구현

- SHA: `0f70c1fcc520`
- 중요도: B
- 태그: RB_TREE, ALLOCATOR
- 원문에서 정한 역할: 일반 BST 삭제와 트리 교환을 추가합니다.
<!-- 원문 요약: Adds node erasure, range erasure, clear, and map swap for the initial BST. -->
<!-- 원문 판단 근거: The commit completes normal mutation and ownership operations, but deletion is not yet red-black aware and swap is later revised for iterator and policy-exception correctness. -->

#### 해당 SHA에서 확인할 실제 코드

- 0/1-child erase의 transplant와 2-child erase의 in-order successor physical transplant 경로를 구분합니다.
- `pair<const Key, T>`의 immutable key 때문에 key 대입 대신 node transplant를 택한 코드 위치를 확인합니다.
- range erase에서 current node destroy 전에 successor 반복자/node를 저장하는 순서를 확인합니다.
- `clear`와 `swap`이 root/size/allocator/node allocator/comparator 상태를 어떻게 이동시키는지 확인하고, deletion balancing은 아직 없음을 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 erase overloads, minimum/successor lookup, transplant helper, `clear`, member `swap`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `c0fdb8e3f84c`의 unbalanced 삽입; RB deletion fix `a055cb19500b`.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: insert된 node를 개별/범위로 제거하거나 두 map의 tree 소유권을 교환할 공개 변경이 없었습니다.
- 해결하려던 문제: const key를 수정하지 않고 BST ordering을 유지한 채 0/1/2-child node를 제거해야 했습니다.
- 선택한 결정: 대상 value를 successor value로 대입하지 않고 successor node 자체를 대상 위치로 transplant합니다. range erase는 다음 위치를 먼저 확보한 뒤 현재 node를 지웁니다.
- 새로 생긴 책임 경계 또는 상태 변화: erase가 link repair, size 감소, node destroy/deallocate를 담당하고 swap이 전체 tree/policy/allocator 상태를 교환합니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: ordinary BST의 소유권 변경 표면을 완성합니다.
- 핵심 코드와 상태 변화: transplant는 parent의 child 또는 root를 교체하고 replacement parent를 고칩니다. 실제 removed node만 한 번 destroy/deallocate합니다.
- 다음 커밋에 넘기는 전제: node physical movement 방식은 RB deletion에도 유지되지만 color와 black-height repair가 추가돼야 합니다.

### 4. feat(map): 레드-블랙 삽입 회전과 색 보정 구현

- SHA: `f29fd8a91523`
- 중요도: S
- 태그: CORE, RB_TREE, HARD
- 원문에서 정한 역할: 회전과 삽입 후 색상 보정을 도입합니다.
<!-- 원문 요약: Adds red/black node state, rotations, and insertion fix-up to keep the map balanced. -->
<!-- 원문 판단 근거: This commit turns the map from a potentially linear BST into the project's defining ordered-container mechanism. The rotation and recoloring cases establish the core logarithmic structure and red-black invariants; omitting it would make the final architecture incomprehensible. -->

#### 원문에서 정한 핵심 판단

- 문제: 초기 map은 정렬된 이진 검색 트리이므로 오름차순이나 내림차순으로 삽입하면 높이가 원소 수에 비례해 늘어날 수 있습니다. 정렬된 순회 결과만 맞는 것으로는 충분하지 않으며, map의 핵심 성능은 검색 트리의 균형을 유지하는 데 달려 있습니다.
- 결정: 노드에 적색·검정 상태를 추가합니다. 루트가 아닌 새 노드는 적색으로 시작하고 루트는 항상 검정으로 유지합니다. 삽입 보정은 삼촌 노드가 적색이면 색을 바꾸고, 검정이면 상황에 맞는 단일 또는 이중 회전을 수행합니다. 왼쪽과 오른쪽 경우는 대칭으로 구현합니다.
- 중요한 이유: map을 단순히 정렬된 노드 연결이 아니라 레드-블랙 트리로 만드는 핵심 동작을 도입했습니다. 이후 삭제 보정, 불변식 검사, 복잡도 테스트가 보존해야 할 로그 시간 기반을 확립합니다.

#### 해당 SHA에서 확인할 실제 코드

- node color 상태가 추가된 지점과 new node/root의 초기 color 규칙을 확인합니다.
- left/right rotation이 parent-child links와 root를 갱신하는 모든 대입을 순서대로 추적합니다.
- red parent에서 red uncle recolor case와 black uncle의 inner/outer rotation case를 좌우 대칭으로 표로 정리합니다.
- repair 종료 후 root black 강제가 어디서 수행되는지 확인하고, stored value를 이동하지 않고 links만 바꾸는지 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 node `red` field, left/right rotation helpers, 삽입 fix-up, `insert`의 fix-up 호출.
- 필요한 경우 비교할 직전 관련 SHA/parent: `c0fdb8e3f84c`의 ordinary BST 삽입과 테스트 `8f8b67961819`.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: comparator ordering은 맞지만 sorted 입력에서 높이가 node 수까지 증가하는 ordinary BST였습니다.
- 해결하려던 문제: 삽입 후 red-red 위반을 제거하면서 in-order ordering, parent links, root 소유권을 유지해야 했습니다.
- 선택한 결정: 새 node는 red로 link하고 parent가 red인 동안 uncle color에 따라 recolor 또는 rotation합니다. root는 항상 black으로 normalize합니다.
- 새로 생긴 책임 경계 또는 상태 변화: 삽입은 단순 link 이후 color/도형 repair 단계까지 책임집니다. rotations는 value/object 소유권을 이동하지 않고 links만 재배열합니다.

#### 중요도 S 심화 추적

- 기존 설계가 충분하지 않았던 이유: ascending/descending 입력에서는 모든 node가 한쪽 child가 되어 search/insert가 O(n)입니다. sorted 출력만으로 이 문제는 보이지 않습니다.
- 핵심 상태/소유권/수명 transition: red leaf link → parent red 여부 검사 → red uncle이면 parent/uncle black, grandparent red 후 위로 진행 → black uncle이면 inner case를 먼저 rotate해 outer case로 만든 뒤 parent/grandparent recolor와 rotation → root black입니다.
- 실패 상황별 정리/되돌리기: balancing 단계 자체는 할당이나 user copy를 하지 않고 포인터/color만 바꿉니다. comparator와 node 생성은 repair 전에 끝납니다. 따라서 fix-up 중 일반 exception 정리 경로는 없습니다.
- 이 커밋이 보장하는 것: 삽입 뒤 root black, no red-red path, ordering/parent links를 유지하는 red-black 삽입 mechanism입니다.
- 이 커밋이 아직 보장하지 않는 것: erase 후 black-height 복구, internal 불변식 white-box proof, explicit height bound는 아직 없습니다.
- 후속 fix/테스트와 연결되는 구조: `8f8b67961819`가 adversarial 출력/query를 검사하고, `a055cb19500b`가 deletion repair, `cd67e6a31bb7`/`cd8ebbb2c01e`가 structure/complexity를 직접 검증합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: 양방향 rotation의 root/parent/child 대입과 red-uncle vs inner/outer black-uncle cases입니다.

### 5. test(map): 정렬 입력 삽입과 검색 경계 stress 검증

- SHA: `8f8b67961819`
- 중요도: B
- 태그: TEST, RB_TREE
- 원문에서 정한 역할: 정렬 입력 삽입과 조회 경계를 불리한 입력으로 검증합니다.
<!-- 원문 요약: Stress-tests ascending and descending insertion plus query boundaries against `std::map`. -->
<!-- 원문 판단 근거: The scenarios are well chosen to expose an ordinary BST's worst case and broaden functional confidence, but they do not directly validate height or every red-black invariant. -->

#### 해당 SHA에서 확인할 실제 코드

- ascending 96 keys와 descending 96 keys scenario가 각각 어떻게 구성되는지 확인합니다.
- 각 삽입 결과와 in-order 순서를 `std::map`과 비교하는 검사문을 찾습니다.
- present/absent query spread에서 `find`, bounds, `equal_range`, end parity를 확인합니다.
- 이 stress가 rotation/recoloring을 많이 유발하지만 직접 height/black-height proof는 아니라는 증명 범위를 기록합니다.
- 확인한 파일/심볼: map 테스트의 ascending/descending 삽입 scenario와 comparison/query helper.
- 필요한 경우 비교할 직전 관련 SHA/parent: production balancing `f29fd8a91523`.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: adversarial 삽입 order에서도 map의 observable ordering, uniqueness, query 경계가 `std::map`과 일치해야 합니다.
- 재현하는 실패 또는 경계: 1부터 96까지 정렬/역정렬 삽입, 존재/부재 key의 `find`, `lower_bound`, `upper_bound`, `equal_range`, `end`입니다.
- 테스트 방식: 결정적 adversarial 입력을 사용한 differential 통합 테스트입니다.
- 통과하는 production 코드 경로: insert search/link/fix-up, in-order 반복자, lookup/bounds입니다.
- 이 테스트가 증명하는 것: 많은 rotation/recolor가 필요한 입력에서도 공개 contents와 query 결과가 reference와 일치합니다.
- 이 테스트가 증명하지 않는 것: 실제 height, root color, red adjacency, black height, parent-link consistency는 직접 보지 않습니다.
- 성격: broad functional stress입니다. structure-specific proof는 아닙니다.
- 후속 변경에서 막아야 하는 회귀: sorted 입력에서 값 누락/중복, ordering 깨짐, lookup 경계 오류입니다.
- 실행 증거: 테스트 코드 검사만 수행했으며 실행 파일 실행 결과는 없습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: 삽입 balancing 직후 공개 동작이 유지되는지 확인합니다.
- 핵심 코드와 상태 변화: 테스트가 두 map에 같은 삽입/query를 적용하고 결과만 비교합니다.
- 다음 커밋에 넘기는 전제: erase balancing은 별도 mechanism이며 이 삽입 stress로 검증되지 않습니다.

### 6. feat(map): 레드-블랙 삭제 보정 구현

- SHA: `a055cb19500b`
- 중요도: S
- 태그: CORE, RB_TREE, HARD
- 원문에서 정한 역할: 삭제 시 이식 상태와 이중 검정 보정을 추가합니다.
<!-- 원문 요약: Implements red-black deletion transplant bookkeeping and symmetric double-black fix-up. -->
<!-- 원문 판단 근거: Deletion is the most intricate core state transition in the tree: removed color, replacement child, parent context, sibling cases, rotations, and recoloring must remain coherent even when the child is null. This is indispensable to the map's correctness story. -->

#### 원문에서 정한 핵심 판단

- 문제: 레드-블랙 트리에서 노드를 삭제하면 특정 루트-리프 경로의 검정 노드 수가 하나 줄 수 있습니다. 대체 자식이 null일 수도 있으므로 일반 노드 객체만으로 색상과 부모 정보를 전달할 수 없습니다. 이 경우를 잘못 처리하면 정렬된 출력은 당장 유지되더라도 검정 높이나 이후 회전이 손상될 수 있습니다.
- 결정: 삭제 과정에서 실제로 이동한 노드, 원래 색상, 대체 자식, 대체 자식의 부모를 명시적으로 기록합니다. 검정 노드를 제거했다면 이중 검정 보정이 형제 노드와 가까운·먼 자식의 색을 확인해 색상 변경이나 회전을 수행하고, 마지막에 대체 노드와 루트를 검정으로 맞춥니다.
- 중요한 이유: 삭제 보정은 삽입 균형 조정과 별개의 핵심 동작이며 경우의 수가 더 많아 오류가 나기 쉽습니다. 모든 삭제 형태에서 레드-블랙 불변식을 유지해 임의 순서의 반복 삭제 뒤에도 트리가 무너지거나 손상되지 않게 합니다.

#### 해당 SHA에서 확인할 실제 코드

- erase에서 physically moved node, original color, replacement child, replacement parent를 어떤 local 상태로 보존하는지 확인합니다.
- replacement child가 null일 때도 parent context를 유지하여 fix-up을 계속하는 호출자/피호출자 흐름을 추적합니다.
- red sibling, black sibling with two black children, near/far red child cases를 좌우 대칭으로 정리하고 각 case의 rotate/recolor 순서를 기록합니다.
- two-child deletion에서 successor node를 transplant하고 대상 color를 복사하되 immutable value object는 유지하는지 확인합니다.
- repair 후 replacement/root black normalization이 수행되는 지점을 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 erase node path, `_transplant`, `_is_black`, deletion fix-up, rotations.
- 필요한 경우 비교할 직전 관련 SHA/parent: ordinary erase `0f70c1fcc520`, repeated deletion 테스트 `86922f1ddfa0`.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 삽입은 red-black이지만 erase는 ordinary BST transplant만 수행해 black-height와 color rules를 깨뜨릴 수 있었습니다.
- 해결하려던 문제: 실제 removed black contribution을 replacement path에 복구하고, replacement가 null이어도 sibling case를 계산해야 했습니다.
- 선택한 결정: `moved`, `moved_was_red`, `fix`, `fix_parent`를 별도로 추적합니다. removed effective node가 black일 때만 double-black repair를 호출합니다.
- 새로 생긴 책임 경계 또는 상태 변화: erase는 structural transplant와 color deficit repair를 분리하고, null child의 parent context는 local 상태가 소유합니다.

#### 중요도 S 심화 추적

- 기존 설계가 충분하지 않았던 이유: black node 제거는 한쪽 root-to-null path의 black count를 줄입니다. sorted order와 node count는 맞아도 RB 불변식이 무너질 수 있습니다.
- 핵심 상태/소유권/수명 transition: 대상 선택 → 0/1-child면 replacement와 parent 기록, 2-child면 successor를 physically transplant하고 successor의 original color/replacement 기록 → black 제거이면 fix loop → 대상 node destroy/deallocate입니다.
- 실패 상황별 정리/되돌리기: erase repair는 할당/comparator/value 대입을 하지 않으므로 일반 exception 브랜치는 없습니다. 핵심 위험은 wrong link/color transition이지 예외 되돌리기가 아닙니다. node는 link에서 완전히 분리된 뒤 정확히 한 번 해제됩니다.
- 이 커밋이 보장하는 것: null replacement를 포함한 erase 후 red-black color/black-height 복구와 immutable key 보존입니다.
- 이 커밋이 아직 보장하지 않는 것: 공개 출력 테스트만으로 모든 parent/header/black-height property를 입증하지는 않습니다. 그 근거는 `cd67e6a31bb7`에서 추가됩니다.
- 후속 fix/테스트와 연결되는 구조: `86922f1ddfa0`이 반복 erase/copy/swap 동작을, `cd67e6a31bb7`이 매 작업 structure를 검사합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: explicit `fix_parent`, symmetric sibling cases, successor color transfer, final black normalization입니다.

### 7. test(map): 반복 삭제·복사·대입·교환 stress 검증

- SHA: `86922f1ddfa0`
- 중요도: B
- 태그: TEST, RB_TREE
- 원문에서 정한 역할: `std::map`과 비교하며 반복 삭제·복사·대입·교환을 스트레스 검증합니다.
<!-- 원문 요약: Adds repeated key/iterator erasure and copy, assignment, and swap stress comparisons. -->
<!-- 원문 판단 근거: The tests exercise broad map behavior after deletion balancing, but later invariant-aware randomized verification provides the stronger architectural evidence. This remains solid normal regression work. -->

#### 해당 SHA에서 확인할 실제 코드

- ascending/descending map의 copy, 대입, swap 시나리오와 standard counterpart 비교 지점을 확인합니다.
- mixed 삽입 순서를 repeated key erase로 줄이는 흐름과 min/max 반복자 erase를 번갈아 수행하는 흐름을 확인합니다.
- 각 단계에서 contents/경계 queries가 검증되는지 확인하고, 이 테스트가 internal color/black-height를 직접 보는지 여부를 구분합니다.
- 확인한 파일/심볼: map 테스트 소스의 copy/대입/swap, repeated key erase, alternating begin/`--end()` erase scenarios.
- 필요한 경우 비교할 직전 관련 SHA/parent: production deletion repair `a055cb19500b`.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: repeated erase와 소유권-moving operations 뒤에도 contents, ordering, query results가 reference와 일치해야 합니다.
- 재현하는 실패 또는 경계: mixed 32-key tree에서 every-third key erase, 이후 min/max 교대 erase로 empty; ascending/descending copy, 대입, swap입니다.
- 테스트 방식: 결정적 differential stress against `std::map`입니다.
- 통과하는 production 코드 경로: key/반복자 erase, deletion fix-up, 복사 생성자, 대입, swap, 반복자 traversal, bounds입니다.
- 이 테스트가 증명하는 것: 반복 변경과 map-to-map 작업 후 관찰 가능한 상태가 일치하고 끝까지 empty로 정리됩니다.
- 이 테스트가 증명하지 않는 것: internal red/black color와 black height, unreachable node, stale header를 직접 검사하지 않습니다.
- 성격: broad normal-path 회귀입니다.
- 후속 변경에서 막아야 하는 회귀: deletion 순서 특정 case에서 key 누락/중복, 반복자 traversal 깨짐, copy/swap contents 오류입니다.
- 실행 증거: 테스트 구현을 검사했으며 실행하지 않았습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: deletion mechanism을 다양한 tree 도형과 공개 소유권 작업에서 압박합니다.
- 핵심 코드와 상태 변화: reference/ft map을 같은 순서로 변경하고 단계별 parity를 확인합니다.
- 다음 커밋에 넘기는 전제: 공개 parity가 통과해도 latent structural corruption이 남을 수 있으므로 inspector가 필요합니다.

### 8. test(map): 무작위 연산마다 레드-블랙 불변식 검증

- SHA: `cd67e6a31bb7`
- 중요도: A
- 태그: TEST, RB_TREE, RISK
- 원문에서 정한 역할: 결정적 무작위 연산 뒤 모든 레드-블랙 트리 불변식을 검사합니다.
<!-- 원문 요약: Adds a constrained white-box inspector and deterministic differential/randomized validation of every red-black invariant after structural operations. -->
<!-- 원문 판단 근거: This materially changes confidence in the core tree: public sorted output alone cannot prove parent links, header extrema, red constraints, black height, or reachable-node counts. Reproducible operation logs make failures diagnosable. -->

#### 원문에서 정한 핵심 판단

- 문제: 정렬된 출력과 `std::map`과의 결과 일치만으로 내부 트리가 유효하다고 증명할 수 없습니다. 부모 링크가 맞지 않거나, 헤더의 최솟값·최댓값이 오래된 값을 가리키거나, 적색 노드가 연속되거나, 검정 높이가 다르거나, 도달할 수 없는 노드가 생겨도 이후 연산 전까지 드러나지 않을 수 있습니다.
- 결정: 테스트 전용 friend 검사기가 헤더 상태, 루트 연결과 색상, 최솟값·최댓값, 하위 트리의 엄격한 키 범위, 부모·자식 링크 일치, 적색 자식 규칙, 검정 높이, 도달 가능한 노드 수를 검사합니다. 고정 삭제 순서, 현재 루트 반복 삭제, 3,000회의 결정적 혼합 연산에서 주 map과 보조 map을 매 단계 검증합니다.
- 중요한 이유: 검증 기준을 공개 출력의 일치에서 내부 표현의 정확성까지 넓혔습니다. 고정 난수 시드, 단계 번호, 연산 앞부분을 함께 기록하므로 깊은 균형 조정 오류도 간헐적인 실패가 아니라 재현 가능한 회귀로 다룰 수 있습니다.

#### 해당 SHA에서 확인할 실제 코드

- 테스트-only inspector가 header marker/color, empty extrema, root parent/color, cached extrema, parent-child consistency, strict subtree bounds, red-child rule, black height, reachable count를 각각 검사하는 코드를 찾습니다.
- fixed structural sequences, current-root repeated deletion, 3,000 결정적 pseudo-random operations의 실행 경로를 구분합니다.
- random 스트림의 작업 종류와 primary/secondary map 모두를 매 step 검증하는 위치를 확인합니다.
- fixed seed, current step, 작업 앞부분이 실패 reproduction에 어떻게 출력되는지 확인합니다.
- 확인한 파일/심볼: `tests/support/map_inspector.hpp`의 inspector와 `tests/test_map_randomized.cpp`의 fixed/random scenarios, seed `0x5EED1234`.
- 필요한 경우 비교할 직전 관련 SHA/parent: RB 삽입/deletion 커밋과 sentinel 커밋 `15a8460ccdfe`의 header 표현.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: strict key ordering, black root/header, no red-red, equal black height, bidirectional parent links, correct header root/min/max, reachable node count == size입니다.
- 재현하는 실패 또는 경계: fixed difficult deletion shapes, repeatedly deleting current root, insert/erase/find/copy/대입/swap 계열의 3,000-step 결정적 mixed operations입니다.
- 테스트 방식: constrained white-box structural validation + 결정적 randomized differential 테스트입니다.
- 통과하는 production 코드 경로: 삽입/deletion rotations and fix-ups, header refresh, clear/copy/대입/swap, lookup and 반복자 traversal입니다.
- 이 테스트가 증명하는 것: 검사한 every step에서 tree 표현 전체가 명시한 invariants와 공개 reference 상태를 동시에 만족합니다.
- 이 테스트가 증명하지 않는 것: 모든 possible 작업 순서를 exhaustive하게 탐색하거나 wall-clock complexity, exception safety를 증명하지 않습니다.
- 성격: 고정 seed와 작업 앞부분을 갖는 reproducible structural 회귀입니다.
- 후속 변경에서 막아야 하는 회귀: 공개 출력에는 즉시 안 보이는 stale parent/extrema, red-red, unequal black height, unreachable/extra node입니다.
- 실행 증거: inspector와 결정적 driver 코드를 검사했으나 3,000 operations를 실제 실행하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: null leaves의 black-height base case, empty header self-reference, root parent가 header인 non-empty case, swap 후 두 header 갱신입니다.
- 상태·소유권·수명 영향: reachable node count가 size와 같아야 하므로 detached leak 또는 duplicate reachability를 구조 수준에서 탐지합니다. allocator block 자체는 별도 테스트 대상입니다.
- 이 커밋이 보장하는 것과 남은 한계: 결정적 sequences에 대한 강한 structure 근거입니다. exception injection과 asymptotic upper bound는 각각 다른 테스트가 필요합니다.
- 다음 관련 커밋과의 연결: `cd8ebbb2c01e`가 same inspector의 height 접근과 counting comparator로 complexity claim을 executable bound로 만듭니다.

### 9. perf(map): 높이와 비교 횟수 회귀 상한 추가

- SHA: `cd8ebbb2c01e`
- 중요도: A
- 태그: PERF, RB_TREE, TEST
- 원문에서 정한 역할: 트리 높이와 비교 횟수의 상한을 강제합니다.
<!-- 원문 요약: Adds executable red-black height and comparator-count upper bounds for adversarial and shuffled insertion orders. -->
<!-- 원문 판단 근거: The test converts the map's logarithmic complexity claim into deterministic structural and operation-count limits. It can detect disabled balancing or hidden linear searches that functional tests would miss. -->

#### 해당 SHA에서 확인할 실제 코드

- counting comparator가 삽입/search comparison count를 어떤 상태에 누적하는지 확인합니다.
- inspector height 측정과 1,024 ascending/descending/shuffled scenario 구성을 확인합니다.
- height upper bound `2 * ceil(log2(n + 1))`와 conservative `n log n` 삽입 comparison bound를 실제 테스트 식에서 확인합니다.
- successful/unsuccessful `find`가 measured height의 constant multiple 안에 있어야 하는 검사문을 확인하고, wall-clock timing을 쓰지 않는 이유를 기록합니다.
- 확인한 파일/심볼: `tests/test_complexity.cpp`의 `counting_less`, `ceil_log2`, scenario builder, height/comparison 검사문; inspector height helper.
- 필요한 경우 비교할 직전 관련 SHA/parent: `cd67e6a31bb7`의 white-box inspector와 RB production core.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: 삽입 order와 무관하게 red-black height가 logarithmic bound 안이고 search comparator count가 height의 constant multiple 안이어야 합니다.
- 재현하는 실패 또는 경계: 1,024 ascending, descending, fixed shuffle 삽입과 present/absent find입니다.
- 테스트 방식: structural bound + 작업-count bound입니다. wall-clock 벤치마크가 아닙니다.
- 통과하는 production 코드 경로: comparator-driven insert search, 삽입 fix-up/rotations, find search, inspector height traversal입니다.
- 이 테스트가 증명하는 것: 테스트 scenarios에서 height가 `2 * ceil(log2(n + 1))` 이하이고 삽입/search comparison count가 설정된 conservative upper bound 안입니다.
- 이 테스트가 증명하지 않는 것: 실제 latency, cache 동작, allocator cost, 모든 n/ordering을 측정하지 않습니다.
- 성격: 결정적 asymptotic 회귀입니다.
- 후속 변경에서 막아야 하는 회귀: balancing 호출 제거, root/link bug로 height 폭증, hidden linear scan 또는 redundant comparator explosion입니다.
- 실행 증거: bound 식과 scenario 코드를 검사했으며 perf 테스트를 실행하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: ordinary BST여도 functional 테스트가 통과할 수 있는 sorted 입력이 핵심 adversarial 경계입니다.
- 상태·소유권·수명 영향: production 소유권을 바꾸지 않으며 height와 comparator 상태만 관찰합니다.
- 이 커밋이 보장하는 것과 남은 한계: 결정적 structural/작업 complexity 근거를 제공합니다. timing performance를 보장하지 않습니다.
- 다음 관련 커밋과의 연결: 이 개발 흐름의 최종 red-black architecture가 correctness와 asymptotic 근거를 모두 갖추게 합니다.

## 6. 불변식 변화 기록

### 원문에서 정한 관련 불변식

- map의 키는 비교자 기준으로 엄격하게 정렬되어야 합니다. 루트는 검정이고, 적색 노드는 적색 자식을 가질 수 없으며, 모든 null 리프 경로의 검정 높이는 같아야 합니다. 부모·자식 링크는 서로 일치하고 도달 가능한 값 노드 수는 `size()`와 같아야 합니다.
- map의 헤더 센티널은 검정이며 값을 저장하지 않습니다. 빈 map에서는 최솟값과 최댓값 링크가 모두 자기 자신을 가리키고, 비어 있지 않으면 현재 루트·최솟값·최댓값을 가리키며 루트의 부모는 다시 헤더를 가리켜야 합니다.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 커밋 | 부족함이 드러난 커밋/상태 | 강화·복구한 fix | 고정한 테스트/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Map keys remain strictly ordered; root black; no red-red; equal black height; consistent links; reachable count == size | ordering은 `c0fdb8e3f84c`, 삽입 RB rules는 `f29fd8a91523` | ordinary erase `0f70c1fcc520`은 black-height를 복구하지 않음; 공개 parity만으로 internal corruption을 못 봄 | deletion rules `a055cb19500b` | `8f8b67961819`, `86922f1ddfa0`, 결정적으로 `cd67e6a31bb7`, `cd8ebbb2c01e` | comparator search, rotations/fix-ups, inspector recursive validation, height/count bounds |
| The map header sentinel is black and value-free; root/min/max links remain current | 이 개발 흐름 밖의 관련 커밋 `15a8460ccdfe` | 이전 root-스냅샷 반복자/NULL-root 표현은 stable end와 cached extrema를 통합하지 못함 | `15a8460ccdfe` | `81d8c4489c16`, 그리고 `cd67e6a31bb7` inspector | `node_base` header, `_refresh_header`, header/root/extrema checks |

## 7. 문제 → 수정 → 검증 연결

| 기존 상태/production change | fix 또는 verification | 원문에서 확정된 연결 관점 | 실제 실패/root cause | 테스트가 통과하는 실제 실행 경로 |
| --- | --- | --- | --- | --- |
| `f29fd8a91523` | `8f8b67961819` | adversarial sorted 삽입과 query 경계 stress | ordinary BST worst case와 rotation/recolor functional 회귀 위험 | sorted insert → fix-up → traversal/find/bounds differential |
| `a055cb19500b` | `86922f1ddfa0` | repeated erase/copy/대입/swap stress | null replacement와 varying sibling case에서 관찰 가능한 상태 corruption 위험 | repeated key/반복자 erase → deletion fix → parity |
| `red-black structure` | `cd67e6a31bb7` | white-box 불변식 validation after every 작업 | sorted 출력이 stale links/color/black height/unreachable nodes를 숨김 | every 작업 → inspector full structure + std parity |
| `red-black complexity` | `cd8ebbb2c01e` | height/comparator-count 결정적 upper bounds | balancing 비활성화나 hidden linear search가 normal 테스트를 통과할 수 있음 | adversarial/shuffle insert → height/count 검사문; find count |

## 8. 소유권·상태·담당 변화

| 시점 | Owner / 상태 / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `2c6dd8acdd20` | 노드를 개별 할당하고 루트 하나가 전체 트리를 소유하는 기반을 만듭니다. | map/node 소유권 부재 | root에서 모든 individually allocated node 도달 | node links, `_root`, `_clear` |
| `f29fd8a91523` | 회전과 삽입 후 색상 보정을 도입합니다. | unbalanced 도형 | color와 rotations가 logarithmic 도형 책임 | node `red`, rotations, 삽입 fix-up |
| `a055cb19500b` | 삭제 시 이식 상태와 이중 검정 보정을 추가합니다. | ordinary transplant만 수행 | erase local 상태가 removed color/null parent context 소유 | moved/fix/fix_parent, deletion fix-up |
| `cd67e6a31bb7` | 결정적 무작위 연산 뒤 모든 레드-블랙 트리 불변식을 검사합니다. | 공개 results만 관찰 | inspector가 표현 correctness 관찰 | `map_inspector.hpp`, randomized driver |
| `cd8ebbb2c01e` | 트리 높이와 비교 횟수의 상한을 강제합니다. | complexity claim이 암묵적 | 테스트가 height/comparison upper bound 소유 | `test_complexity.cpp` |

## 9. 개발 흐름의 최종 상태

- 최종적으로 성립한 표현/상태: allocator-created value nodes가 parent/left/right/color link로 red-black tree를 이루며, 관련 sentinel 커밋 이후 embedded value-free header가 root/min/max와 `end()`를 나타냅니다.
- 최종적으로 보장하는 불변식: comparator strict ordering, black root/header, red node의 black children, 모든 null path의 같은 black height, 양방향 parent/child 일치, reachable node count와 size 일치, logarithmic structural height입니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: comparator는 tree 수명 동안 ordering semantics를 일관되게 유지해야 합니다. 이 개발 흐름의 structural/perf 테스트는 exception safety와 allocator 상태 compatibility를 대신하지 않습니다.
- 최종 verification 근거: sorted/differential stress, repeated deletion stress, every-step white-box 결정적 random validation, height/comparator-count upper bounds를 코드로 확인했습니다. 실제 테스트 명령은 실행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point 커밋: 삽입 balancing `f29fd8a91523`, deletion balancing `a055cb19500b`, 근거 기준을 바꾼 `cd67e6a31bb7`입니다.

## 10. 최종 설계와 실행 흐름

아래 단계명은 원문이 정의한 개발 흐름 progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 커밋 | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | 실패/정리 | 다음 단계에 남기는 불변식 |
| --- | --- | --- | --- | --- | --- | --- |
| Node 소유권 | `2c6dd8acdd20` | node/create/destroy/clear | key/value와 empty root | node allocate/construct, root 도달성 | construct 실패 deallocate; post-order clear | individually owned nodes |
| Unbalanced BST 삽입 | `c0fdb8e3f84c` | `insert`, `operator[]` | key/comparator, root | compare search → duplicate 또는 parent child link | node 생성 되돌리기; later comparator risk 남음 | strict ordering/uniqueness |
| Unbalanced erase/swap | `0f70c1fcc520` | erase/transplant/clear/swap | 대상 node | 0/1-child replace 또는 successor transplant | removed node one-time destroy | immutable key 유지, balancing 미완성 |
| RB 삽입 repair | `f29fd8a91523` | rotations/fix-up | red inserted node | recolor 또는 inner/outer rotations | 할당 없음 | 삽입 후 RB rules |
| Adversarial 삽입 stress | `8f8b67961819` | sorted scenarios | 96 ordered keys | ft/std insert와 query 비교 | 검사문 실패 | 공개 삽입/query parity |
| RB deletion repair | `a055cb19500b` | erase + double-black fix | 대상, moved color, replacement/parent | sibling cases로 black deficit 이동/해소 | link repair 후 node release | erase 후 RB rules |
| Deletion/lifecycle stress | `86922f1ddfa0` | repeated erase/copy/swap 테스트 | mixed trees | key/반복자 erase와 소유권 operations 반복 | 테스트 검사문 | observable 변경 parity |
| White-box 불변식 validation | `cd67e6a31bb7` | inspector/random driver | each 작업 result | header/root/subtree/color/black height/count recursive 검사 | seed/step/앞부분으로 실패 진단 | 표현 correctness 근거 |
| Structural complexity limits | `cd8ebbb2c01e` | complexity 테스트 | 1,024 ordered/shuffled keys | height와 comparator calls 측정 | 결정적 bound 검사문 | logarithmic 회귀 경계 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 커밋 기록에 최종 HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A 커밋은 결정, 실패 경계, 소유권/상태 변화를 설명할 수 있습니다.
- [x] 테스트·성능 관련 커밋은 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → 실패/risk → root cause → 수정 → 회귀 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 커밋 history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] 개발 흐름 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 03-map-unbalanced-bst-to-verified-red-black-core.md =====

===== BEGIN FILE: 04-stable-map-iterators-through-structural-mutation.md =====
# 트리 변경 중에도 안정적인 map 반복자

## 1. 개발 흐름의 목표

### 원문에서 정한 의의

초기 반복자는 루트 포인터를 복사해 `end()`를 표현하므로 회전, 루트 삭제, `swap()` 뒤에 오래된 루트를 가리킬 수 있습니다. 값이 없는 헤더 센티널을 컨테이너가 소유하도록 바꿔 현재 루트·최솟값·최댓값과 종료 위치를 한곳에서 관리하고, 구조 변경 뒤 반복자 안정성을 검증합니다.

<details>
<summary>영문 원문</summary>

The initial iterator can traverse a static tree, but `--end()` depends on a root pointer copied into the iterator. Rotations, root erasure, and swap make that snapshot stale even though element nodes remain valid. The header-sentinel refactor moves end-state and extrema ownership into the container, lets nodes navigate to the current end through parent links, and removes any need to construct a key merely to represent the sentinel. The follow-up tests target exactly the structural operations that invalidate the earlier model, making this a clear limitation-to-architecture-correction progression.

</details>


### 이 개발 흐름에서 확인할 내용

- 위 의의에 제시된 변화 과정을 각 커밋의 실제 SHA 코드로 재구성합니다.
- 원문이 확정한 커밋 역할과 중요도를 바꾸지 않고, 실제 구현/실패/테스트 근거만 직접 채웁니다.

### 원문에서 확인되는 설계

- `ft::map`은 할당자로 만든 노드에 값을 저장하고 이를 레드-블랙 트리로 연결합니다. 값을 저장하지 않는 헤더 센티널이 루트·최솟값·최댓값 링크를 보유하며 안정적인 `end()` 표현도 담당합니다.

### 원문에서 확인되는 주요 구현 난점

- 루트 포인터와 null 종료 상태를 들고 다니는 반복자를, 회전·루트 교체·삭제·`clear`·`swap` 뒤에도 유효하고 기본 생성 가능한 키를 요구하지 않는 값 없는 헤더 센티널 방식으로 바꿔야 합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- 정적 tree에서는 동작하던 root-스냅샷 반복자가 rotation/root erase/swap에서 왜 실패하는가?
- value-free header sentinel이 `end()`, extrema, empty 상태, non-default key 문제를 한 번에 어떻게 해결하는가?
- element 반복자 identity와 컨테이너-owned end sentinel 책임은 어떻게 분리되는가?
- swap 뒤 기존 element 반복자가 새 소유자의 end에 도달하려면 어떤 parent/header 연결이 필요한가?

## 3. 완료 기준

- B: 개발 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- S: 핵심 architecture/불변식을 직전 상태 → 실패 가능성 → 결정 → 실제 핵심 코드 → 소유권/lifecycle/상태 변화 → 후속 fix/테스트까지 코드 근거로 설명할 수 있어야 합니다.
- A: 주요 구성 요소/경계/실패 처리/통합 point를 실제 코드와 설계 판단으로 연결하고, 관련 회귀 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- 모든 커밋은 해당 SHA의 코드 또는 테스트/빌드 diff를 근거로 기록합니다.
- 개발 흐름 최종 설명은 원문 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 커밋 간 변화로 재구성합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `4bbf81cecef4` | `feat(map): 가변 반복자와 tree 순회 구현` | B | RB_TREE, ITERATOR | null 종료 상태와 복사한 루트 포인터를 사용하는 반복자 순회를 구현합니다. |
| 2 | `50e62b0e0298` | `feat(map): 상수와 역방향 반복자 구현` | B | RB_TREE, ITERATOR | 같은 모델을 상수 반복자와 역방향 반복자로 확장합니다. |
| 3 | `048ada8fe1c5` | `feat(map): 가변·상수 반복자 상호 비교 지원` | B | ITERATOR, PUBLIC_API | 변경 가능 반복자와 상수 반복자 사이의 비교를 지원합니다. |
| 4 | `15a8460ccdfe` | `fix(map): 값 없는 header로 끝 반복자 상태 안정화` | S | ARCH, ITERATOR, RB_TREE | 복사한 루트 대신 값을 저장하지 않는 컨테이너 소유 헤더 센티널을 사용합니다. |
| 5 | `81d8c4489c16` | `test(map): 회전·삭제·교환 뒤 반복자 상태 검증` | A | TEST, ITERATOR, RB_TREE | 저장한 `end`, 원소 반복자, `swap` 이동, 빈 상태 복원, 기본 생성 불가 키를 검증합니다. |

## 5. 커밋별 학습 기록

### 1. feat(map): 가변 반복자와 tree 순회 구현

- SHA: `4bbf81cecef4`
- 중요도: B
- 태그: RB_TREE, ITERATOR
- 원문에서 정한 역할: null 종료 상태와 복사한 루트 포인터를 사용하는 반복자 순회를 구현합니다.
<!-- 원문 요약: Adds mutable bidirectional tree traversal using successor/predecessor navigation and a root-bearing end iterator. -->
<!-- 원문 판단 근거: The commit provides necessary ordered traversal, but the root snapshot coupled into each iterator is later shown to be structurally insufficient. It is an intermediate implementation rather than the final iterator model. -->

#### 해당 SHA에서 확인할 실제 코드

- mutable 반복자가 current node 외에 root 스냅샷을 어떤 필드로 보관하는지 확인합니다.
- successor는 right-subtree minimum 또는 parent climb, predecessor는 대칭 경로를 사용하는지 추적합니다.
- `begin()`의 minimum 선택과 null `end()` 표현, `--end()`가 copied root에서 maximum을 찾는 경로를 확인합니다.
- root가 rotation/replacement/swap으로 바뀌었을 때 saved 반복자의 root 스냅샷이 왜 stale해질 수 있는지이 SHA의 상태만 근거로 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 mutable `iterator`, 반복자 fields `_node`/`_root`, successor/predecessor helpers, `begin`, `end`.
- 필요한 경우 비교할 직전 관련 SHA/parent: 반복자가 없던 직전 map 상태; limitation fix `15a8460ccdfe`.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: tree ordering과 변경은 있어도 공개 반복자로 in-order traversal할 수 없었습니다.
- 해결하려던 문제: 현재 node에서 다음/이전 key로 이동하고 null end에서 현재 최대 node로 되돌아갈 정보가 필요했습니다.
- 선택한 결정: 반복자가 current node와 생성 시점의 root 포인터를 함께 저장합니다. current가 null인 `end()`에서 `--`하면 copied root의 maximum을 찾습니다.
- 새로 생긴 책임 경계 또는 상태 변화: element traversal topology 일부를 반복자가 스냅샷으로 소유합니다. element나 tree 수명 소유권은 갖지 않습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: 정적 tree에서 bidirectional ordered traversal의 첫 공개 model입니다.
- 핵심 코드와 상태 변화: `++`는 right subtree minimum 또는 ancestor climb, `--`는 left subtree maximum 또는 ancestor climb입니다. null end만 root 스냅샷을 사용합니다.
- 다음 커밋에 넘기는 전제: const/reverse 반복자도 같은 topology와 root 스냅샷 limitation을 상속합니다.

### 2. feat(map): 상수와 역방향 반복자 구현

- SHA: `50e62b0e0298`
- 중요도: B
- 태그: RB_TREE, ITERATOR
- 원문에서 정한 역할: 같은 모델을 상수 반복자와 역방향 반복자로 확장합니다.
<!-- 원문 요약: Adds const and reverse map iterators and const-qualified traversal accessors. -->
<!-- 원문 판단 근거: The change extends the initial traversal design with expected const correctness and reverse iteration. It remains ordinary API work and inherits the earlier end-state coupling. -->

#### 해당 SHA에서 확인할 실제 코드

- const 반복자의 reference/포인터 type과 mutable→const converting 생성자를 확인합니다.
- const successor/predecessor가 mutable traversal과 같은 topology를 사용하되 write access를 노출하지 않는지 확인합니다.
- `rbegin()`이 `end()`, `rend()`가 `begin()`을 base로 shared `reverse_iterator`에 연결하는 코드 위치를 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `const_iterator`, const `begin/end`, reverse 반복자 typedefs와 `rbegin/rend`.
- 필요한 경우 비교할 직전 관련 SHA/parent: mutable 반복자 `4bbf81cecef4`와 공용 reverse adaptor `7a8e3d32bb4d`/`ae50e9038643`.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: non-const map만 traversal할 수 있고 reverse 공개 API가 없었습니다.
- 해결하려던 문제: const map에서 key/value를 수정하지 않는 traversal과 기존 reverse adaptor 연결이 필요했습니다.
- 선택한 결정: const 포인터/reference를 노출하는 별도 반복자를 두고 mutable 반복자에서 converting 생성을 허용합니다. reverse traversal은 `ft::reverse_iterator`에 end/begin을 base로 전달합니다.
- 새로 생긴 책임 경계 또는 상태 변화: constness는 exposed type에서 보장되지만 topology 상태는 mutable 반복자와 같은 node/root 스냅샷입니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: const-correct ordered/reverse traversal을 공개 API에 추가합니다.
- 핵심 코드와 상태 변화: 런타임 tree 상태는 바뀌지 않고 반복자 wrapper와 accessors만 확장됩니다.
- 다음 커밋에 넘기는 전제: mutable/const 반복자가 같은 node identity를 표현하므로 서로 비교 가능해야 합니다.

### 3. feat(map): 가변·상수 반복자 상호 비교 지원

- SHA: `048ada8fe1c5`
- 중요도: B
- 태그: ITERATOR, PUBLIC_API
- 원문에서 정한 역할: 변경 가능 반복자와 상수 반복자 사이의 비교를 지원합니다.
<!-- 원문 요약: Allows mutable and const map iterators to compare symmetrically. -->
<!-- 원문 판단 근거: The change restores an expected interoperability property of the public iterator interface, but it is localized and later subsumed by the sentinel-based iterator refactor. -->

#### 해당 SHA에서 확인할 실제 코드

- const 반복자가 mutable 반복자에 대해 equality/inequality를 제공하는 overload를 찾습니다.
- 반대 operand ordering을 friend/non-member overload가 어떤 conversion으로 처리하는지 확인합니다.
- 비교 identity가 exposed reference type이 아니라 current node에 기반하는지 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 반복자/const_iterator mixed `operator==`와 `operator!=`.
- 필요한 경우 비교할 직전 관련 SHA/parent: `50e62b0e0298`의 별도 mutable/const 반복자 types.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: 의미상 같은 위치라도 mutable 반복자와 const 반복자를 양쪽 operand 순서로 직접 비교할 수 없었습니다.
- 해결하려던 문제: standard 반복자 usage에서 `iterator == const_iterator`와 역순 비교가 모두 필요했습니다.
- 선택한 결정: exposed value type이 아니라 내부 current node 포인터를 identity로 비교하고 필요한 friend/overload로 양방향 호출을 허용합니다.
- 새로 생긴 책임 경계 또는 상태 변화: 반복자 interoperability가 공개 API에 추가되며 tree 상태는 변하지 않습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: mutable→const conversion과 위치 비교를 실제 generic 코드에서 사용할 수 있게 합니다.
- 핵심 코드와 상태 변화: equality는 node identity만 확인합니다. root 스냅샷이 같거나 최신인지까지 확인하지 않으므로 기존 end limitation은 남습니다.
- 다음 커밋에 넘기는 전제: sentinel refactor에서도 위치 identity는 하나의 structural 포인터로 유지할 수 있습니다.

### 4. fix(map): 값 없는 header로 끝 반복자 상태 안정화

- SHA: `15a8460ccdfe`
- 중요도: S
- 태그: ARCH, ITERATOR, RB_TREE
- 원문에서 정한 역할: 복사한 루트 대신 값을 저장하지 않는 컨테이너 소유 헤더 센티널을 사용합니다.
<!-- 원문 요약: Refactors map to a value-free header sentinel that owns root/minimum/maximum links and represents `end()`. -->
<!-- 원문 판단 근거: This is a major architectural correction: end iterators no longer carry stale root snapshots, rotations and swap can preserve element iterators, empty maps need no default key, and extrema become container-owned state. It defines the final map representation. -->

#### 원문에서 정한 핵심 판단

- 문제: 초기 `end()` 반복자는 null과 생성 당시 복사한 루트 포인터로 종료 위치를 표현했습니다. 회전, 루트 삭제, `swap()`으로 현재 루트가 바뀌어도 저장해 둔 반복자는 이전 트리 연결을 계속 참조할 수 있었습니다. 센티널에 실제 값을 저장하면 키 형식에 불필요한 기본 생성 가능 조건도 생깁니다.
- 결정: map을 `node_base`와 값을 저장하지 않는 헤더 센티널 중심으로 바꿉니다. `header.parent`는 루트를, `header.left`와 `header.right`는 최솟값과 최댓값을 가리키며 `end()`는 헤더를 가리킵니다. 루트의 부모 링크도 헤더에서 끝나므로 다음·이전 노드 탐색이 현재 컨테이너의 종료 위치를 트리 연결을 통해 찾을 수 있습니다.
- 중요한 이유: 반복자 안정성, 저장된 종료 반복자에서 현재 최댓값 찾기, 빈 상태 표현, `swap()` 뒤 소유자 이동, 기본 생성할 수 없는 키 지원을 하나의 표현 변경으로 해결했습니다. 반복자만 국소적으로 고친 것이 아니라 map의 최종 내부 표현을 정한 변경입니다.

#### 해당 SHA에서 확인할 실제 코드

- `node_base`의 structural links/color/header marker와 value-bearing derived node의 분리를 확인합니다.
- header의 `parent=root`, `left=min`, `right=max` 관계와 empty 상태에서 extrema self-reference를 만드는 초기화/refresh 코드를 찾습니다.
- 반복자 상태가 single `node_base*`로 줄어든 뒤 maximum increment → header, header decrement → current maximum이 되는 traversal을 추적합니다.
- insert/rotation/transplant/erase/clear/swap이 header/root/extrema 불변식을 갱신하는 모든 주요 지점을 연결합니다.
- swap 뒤 moved root의 parent가 새 컨테이너 header로 재부착되는 순서와 saved element 반복자가 destination end에 도달하는 구조를 확인합니다.
- empty map이 default-constructible key를 요구하지 않게 된 이유를 header에 value가 없다는 실제 type layout으로 확인합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `node_base`, derived `node`, embedded `_header`, header reset/refresh helpers, 반복자 next/previous, `begin/end`, insert/erase/clear/swap header repair.
- 필요한 경우 비교할 직전 관련 SHA/parent: root-스냅샷 반복자 커밋 `4bbf81cecef4`, `50e62b0e0298`, `048ada8fe1c5`; 회귀 `81d8c4489c16`.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: 반복자마다 root 포인터를 복사하면 `--end()`에 충분하다고 가정했습니다. end는 null current node였습니다.
- 실제 실패 또는 위험: rotation/root erase는 root를 바꾸고 swap은 tree 소유자를 바꾸므로 saved 반복자의 root 스냅샷이 stale합니다. null end는 컨테이너 identity와 current extrema를 스스로 표현하지 못합니다. value-bearing sentinel은 key default 생성을 강제합니다.
- root cause: end/extrema 책임이 컨테이너가 아니라 개별 반복자 스냅샷에 분산됐습니다.
- 수정된 불변식/결정: 컨테이너가 value-free embedded header를 영구 소유하고 모든 structural path가 header↔root/min/max links를 유지합니다. 반복자는 current `node_base*` 하나만 저장합니다.
- 변경/커밋 순서의 변경: 변경/rotation/transplant 후 root parent를 header로 붙이고 extrema cache를 갱신합니다. swap 후에는 교환된 각 root의 parent를 새 소유자 header로 재부착합니다.
- 실제 수정 코드 근거: `_header.is_header`, empty self-links, `_header.parent/left/right`, root parent 대입, 반복자 `_previous(header)`와 `_next(max)`입니다.
- 연결되는 회귀 테스트 SHA/근거: `81d8c4489c16`이 saved end, rotations, former-root erase, swap migration, clear reset, non-default key를 직접 검사합니다.

#### 중요도 S 심화 추적

- 기존 설계가 충분하지 않았던 이유: element node identity는 rotation에서 유지되지만 반복자가 별도 root 스냅샷을 들고 있으면 topology의 현재 컨테이너 경계와 분리됩니다. local patch로 각 반복자 스냅샷을 갱신할 방법도 없습니다.
- 핵심 상태/소유권/수명 transition: 기존 `_root`/null end + `(node, root snapshot)` → embedded `_header` 생성 → root parent를 header로 변경하고 extrema cache 설정 → 반복자를 single node_base 포인터로 축소 → end는 header address입니다.
- 실패 상황별 정리/되돌리기: 이 fix의 핵심 실패는 exception이 아니라 structural stale 상태입니다. 할당/value 수명은 derived `node`만 담당하며 header는 allocator로 생성/파괴하지 않습니다. clear는 value nodes를 해제하고 header를 empty self-reference로 reset합니다.
- 이 커밋이 보장하는 것: saved 컨테이너 end가 current maximum을 찾고, element 반복자가 updated parent links를 따라 end에 도달하며, swap 후 node 반복자가 destination header에 연결됩니다. empty map 생성은 key 생성자를 호출하지 않습니다.
- 이 커밋이 아직 보장하지 않는 것: erased element를 가리키는 반복자는 무효입니다. 다른 컨테이너의 반복자 비교나 invalid dereference는 precondition 밖입니다. policy/allocator swap exception은 개발 흐름 05에서 다룹니다.
- 후속 fix/테스트와 연결되는 구조: `81d8c4489c16`이 architecture promise를 결정적 scenarios로 고정하고, `cd67e6a31bb7` inspector가 header/root/extrema를 every 작업 검사합니다.
- 프로젝트 architecture 설명에 반드시 포함할 코드 근거: value-free base/derived split, embedded header links, root parent termination, max→header/header→max traversal, swap reattachment입니다.

### 5. test(map): 회전·삭제·교환 뒤 반복자 상태 검증

- SHA: `81d8c4489c16`
- 중요도: A
- 태그: TEST, ITERATOR, RB_TREE
- 원문에서 정한 역할: 저장한 `end`, 원소 반복자, `swap` 이동, 빈 상태 복원, 기본 생성 불가 키를 검증합니다.
<!-- 원문 요약: Tests saved end iterators, element iterators through rotations, root erasure, swap, empty reset, and non-default-constructible keys. -->
<!-- 원문 판단 근거: These cases validate the exact structural promises that motivated the header sentinel. The suite protects an architecture boundary that ordinary result comparison cannot observe. -->

#### 해당 SHA에서 확인할 실제 코드

- saved end 반복자를 rotation을 일으키는 삽입 전후 및 former-root erase 전후에 유지한 테스트를 찾고 `--end()` expected maximum을 확인합니다.
- clear 뒤 `begin()==end()` reset 검사문을 확인합니다.
- element 반복자를 rotations 동안 유지하고 parent links를 통해 header까지 advance하는 테스트를 확인합니다.
- 두 map swap 후 held element 반복자가 original node를 dereference하고 새 소유자의 end sentinel까지 도달하는지 확인합니다.
- non-default-constructible key 테스트 준비 코드가 empty map creation과 삽입을 검증하는 이유를 기록합니다.
- 확인한 파일/심볼: `tests/test_map_iterators.cpp`의 saved-end, rotation/root-erase, element-반복자, swap, clear, non-default-key scenarios.
- 필요한 경우 비교할 직전 관련 SHA/parent: production refactor `15a8460ccdfe`.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: header가 current root/min/max와 end identity를 소유하고 non-erasing structural 변경이 element node identity와 traversal 경계를 보존해야 합니다.
- 재현하는 실패 또는 경계: saved end 후 rotation, root replacement/erase, clear; held element 반복자 후 rotation/swap; non-default-constructible key의 empty map입니다.
- 테스트 방식: architecture-specific 결정적 회귀 테스트입니다. 일부는 반복자 상태를 통해 internal topology를 간접 관찰합니다.
- 통과하는 production 코드 경로: insert rotations/header refresh, erase/transplant/header refresh, 반복자 next/previous, swap root reattachment, clear reset, map 생성자입니다.
- 이 테스트가 증명하는 것: 해당 scenarios에서 saved end와 element 반복자가 current/new 소유자 header를 따라가며 extrema와 empty reset이 맞고 header가 key value를 요구하지 않습니다.
- 이 테스트가 증명하지 않는 것: erased node 반복자 validity, 모든 possible rotation/deletion 순서, allocator/policy exception safety는 증명하지 않습니다.
- 성격: root-스냅샷 limitation을 직접 재현하는 결정적 architectural 회귀입니다.
- 후속 변경에서 막아야 하는 회귀: 반복자에 root 스냅샷 재도입, header extrema 갱신 누락, swap 후 root parent stale, clear 후 header self-link 누락, value-bearing sentinel입니다.
- 실행 증거: 테스트 소스와 실제 실행 경로를 검사했으며 실행 파일을 실제 실행하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: 저장한 반복자를 만든 뒤 topology/소유자가 바뀌는 시간적 경계입니다. 단순히 변경 후 새 반복자를 얻는 테스트로는 검출되지 않습니다.
- 상태·소유권·수명 영향: header는 컨테이너 수명에 고정되고 value nodes만 소유자 간 이동합니다. swap 뒤 held element 반복자는 node와 함께 새 소유자 topology에 속합니다.
- 이 커밋이 보장하는 것과 남은 한계: exact motivating cases에 대한 근거를 제공합니다. exhaustive 반복자 conformance suite는 아닙니다.
- 다음 관련 커밋과의 연결: 이후 randomized inspector와 consumer 테스트가 same final 표현을 더 넓게 사용합니다.

## 6. 불변식 변화 기록

### 원문에서 정한 관련 불변식

- map의 헤더 센티널은 검정이며 값을 저장하지 않습니다. 빈 map에서는 최솟값과 최댓값 링크가 모두 자기 자신을 가리키고, 비어 있지 않으면 현재 루트·최솟값·최댓값을 가리키며 루트의 부모는 다시 헤더를 가리켜야 합니다.
- 회전과 원소를 삭제하지 않는 트리 변경은 원소 노드의 정체성을 유지합니다. 저장해 둔 원소 반복자는 갱신된 부모 링크를 따라가며, `end()`는 오래된 루트 복사본이 아니라 컨테이너가 소유한 센티널 상태에 연결되어야 합니다.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 커밋 | 부족함이 드러난 커밋/상태 | 강화·복구한 fix | 고정한 테스트/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| The map header sentinel is black and value-free; empty extrema self-reference; non-empty header caches root/min/max | `15a8460ccdfe` | `4bbf81cecef4`의 null end/root 스냅샷은 컨테이너-owned current 상태가 아님 | `15a8460ccdfe` | `81d8c4489c16`, `cd67e6a31bb7` | `node_base`, embedded `_header`, reset/refresh, inspector header checks |
| Rotations and non-erasing structural changes preserve element-node identity; 반복자 follow updated links | node identity는 초기 node model부터 유지, 반복자 traversal은 `4bbf81cecef4` | root 스냅샷이 rotation/root erase/swap 후 stale | `15a8460ccdfe` | `81d8c4489c16` | single-포인터 반복자, root parent header, swap reattachment, saved 반복자 테스트 |

## 7. 문제 → 수정 → 검증 연결

| 기존 상태/production change | fix 또는 verification | 원문에서 확정된 연결 관점 | 실제 실패/root cause | 테스트가 통과하는 실제 실행 경로 |
| --- | --- | --- | --- | --- |
| `4bbf81cecef4 / 50e62b0e0298 / 048ada8fe1c5` | `15a8460ccdfe` | root-스냅샷/null-end limitation을 header sentinel architecture로 교정 | end/extrema를 반복자 스냅샷이 소유해 topology change 후 stale; value sentinel은 key default 생성 강제 | 반복자 상태 제거 → header links → structural paths의 refresh/reattach |
| `15a8460ccdfe` | `81d8c4489c16` | rotation/root erase/swap/clear/non-default key 반복자-상태 회귀 | refactor의 핵심 promise가 ordinary post-변경 result 테스트로는 보이지 않음 | 변경 전 saved 반복자/key 테스트 준비 코드 → 변경 → decrement/advance/dereference/end 검사문 |

## 8. 소유권·상태·담당 변화

| 시점 | Owner / 상태 / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `15a8460ccdfe` | 복사한 루트 대신 값을 저장하지 않는 컨테이너 소유 헤더 센티널을 사용합니다. | 반복자가 current node와 root 스냅샷 소유; null end | 컨테이너 header가 root/min/max/end 소유; 반복자는 node_base 포인터만 소유 | `node_base`, `_header`, 반복자 fields, refresh/swap |
| `81d8c4489c16` | 저장한 `end`, 원소 반복자, `swap` 이동, 빈 상태 복원, 기본 생성 불가 키를 검증합니다. | architecture promise가 코드 검토에만 의존 | 회귀 scenarios가 시간적 반복자 상태를 고정 | `tests/test_map_iterators.cpp` |

## 9. 개발 흐름의 최종 상태

- 최종적으로 성립한 표현/상태: map object 안에 value가 없는 black header가 영구 존재합니다. empty이면 `left/right`가 자신을 가리키고 root는 없으며, non-empty이면 `parent=root`, `left=min`, `right=max`, `root.parent=&header`입니다. 반복자는 단일 `node_base*`를 저장합니다.
- 최종적으로 보장하는 불변식: rotations와 non-erasing link changes는 element node address를 바꾸지 않습니다. saved element 반복자는 current parent chain을 따라 컨테이너 header에 도달하고, saved end는 컨테이너 header이므로 current maximum을 찾습니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: erased element 반복자는 무효입니다. 반복자 소유자가 사라진 뒤 사용하거나 unrelated 컨테이너 반복자를 비교하는 동작은 보장하지 않습니다.
- 최종 verification 근거: saved end before rotations/root erase, held element 반복자 through rotations/swap, clear reset, non-default key scenarios를 코드로 확인했습니다. 실행 결과는 생성하지 못했습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point 커밋: `15a8460ccdfe`입니다. 앞선 세 커밋은 limitation이 생기는 root-스냅샷 model을 구성합니다.

## 10. 최종 설계와 실행 흐름

아래 단계명은 원문이 정의한 개발 흐름 progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 커밋 | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | 실패/정리 | 다음 단계에 남기는 불변식 |
| --- | --- | --- | --- | --- | --- | --- |
| Root-스냅샷 mutable 반복자 | `4bbf81cecef4` | 반복자 fields, next/previous | node와 creation-time root | subtree minimum/maximum 또는 parent climb | exception 없음; stale 스냅샷 risk | 정적 tree bidirectional traversal |
| Const/reverse extension | `50e62b0e0298` | const 반복자, rbegin/rend | mutable topology model | const view와 shared reverse adaptor 연결 | same stale-root limitation | const/reverse 공개 API |
| Mixed 반복자 comparison | `048ada8fe1c5` | mixed equality operators | mutable/const positions | current node identity 비교 | 소유자/root validity는 검사 안 함 | interoperability |
| Header-sentinel refactor | `15a8460ccdfe` | node_base/header/반복자/refresh/swap | root 스냅샷/null end | 컨테이너-owned header와 topology termination으로 전환 | clear reset; structural path마다 header repair | stable end/extrema/value-free sentinel |
| Mutation/swap 반복자 regressions | `81d8c4489c16` | 반복자 테스트 | 변경 전에 저장한 end/element 반복자 | rotation/erase/swap/clear 후 동일 object로 traversal | 검사문 실패로 회귀 검출 | final 반복자 architecture 근거 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 커밋 기록에 최종 HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A 커밋은 결정, 실패 경계, 소유권/상태 변화를 설명할 수 있습니다.
- [x] 테스트·성능 관련 커밋은 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → 실패/risk → root cause → 수정 → 회귀 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 커밋 history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] 개발 흐름 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 04-stable-map-iterators-through-structural-mutation.md =====

===== BEGIN FILE: 05-stateful-allocators-map-failure-transactions.md =====
# 상태를 가진 할당자와 map 실패 트랜잭션

## 1. 개발 흐름의 목표

### 원문에서 정한 의의

map의 정확성은 키 정렬뿐 아니라 어떤 할당자 상태가 노드를 만들고 해제하는지, 어떤 비교자 상태가 트리 순서를 정의하는지에도 달려 있습니다. 재바인딩 과정의 상태 손실, 할당 뒤 비교 예외, 부분 생성, 복사 대입과 `swap()` 실패를 수정해 정책 상태를 먼저 확정한 뒤 트리 소유권을 옮기도록 합니다.

<details>
<summary>영문 원문</summary>

Map correctness depends on more than key/value ordering: allocator identity determines which state owns each node, and comparator state determines whether the same links describe a valid ordering. The thread first repairs lost allocator state, then ensures a comparator cannot throw after an allocation has no owner. It next makes construction and assignment transactional, and finally addresses a subtler policy failure where comparator assignment can interrupt swap. The resulting rule is consistent across the sequence: policy state must be settled before physical tree ownership is committed, and every 실패 처리 must leave each node associated with exactly one valid owner.

</details>


### 이 개발 흐름에서 확인할 내용

- 위 의의에 제시된 변화 과정을 각 커밋의 실제 SHA 코드로 재구성합니다.
- 원문이 확정한 커밋 역할과 중요도를 바꾸지 않고, 실제 구현/실패/테스트 근거만 직접 채웁니다.

### 원문에서 확인되는 설계

- map의 비교자는 키 동등성과 트리 순서를 정하고, 노드 형식으로 재바인딩한 할당자는 실제 노드 소유를 담당합니다. 복사·대입·`swap`은 비교자·할당자 상태와 소유 트리 상태를 함께 일관되게 유지해야 합니다.

### 원문에서 확인되는 주요 구현 난점

- 노드 재바인딩, 삽입, 생성, 복사 대입, 공개 `swap` 전 과정에서 상태를 가진 할당자와 비교자의 의미를 보존해야 하며, 비교 호출과 비교자 대입에서 발생하는 예외도 처리해야 합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- allocator rebind 후에도 소유권 identity를 유지해야 하는 이유는 무엇인가?
- comparator call과 node 할당의 순서를 어떻게 배치해야 unowned-node 실패 window가 사라지는가?
- 생성자/copy 대입에서 임시 객체 tree가 소유권 transaction을 어떻게 격리하는가?
- comparator 대입 자체가 throw할 수 있을 때 policy 상태와 physical tree 소유권 중 무엇을 먼저 커밋해야 하는가?

## 3. 완료 기준

- A: 주요 구성 요소/경계/실패 처리/통합 point를 실제 코드와 설계 판단으로 연결하고, 관련 회귀 또는 다음 fix와의 관계를 설명할 수 있어야 합니다.
- 모든 커밋은 해당 SHA의 코드 또는 테스트/빌드 diff를 근거로 기록합니다.
- 개발 흐름 최종 설명은 원문 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 커밋 간 변화로 재구성합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `ae180871b160` | `fix(map): 값 allocator 상태로 노드 allocator 구성` | A | RB_TREE, ALLOCATOR, RISK | 노드 할당자로 재바인딩해도 값 할당자의 상태를 보존합니다. |
| 2 | `cb08194d17b0` | `fix(map): 삽입 위치를 노드 할당 전에 확정` | A | RB_TREE, EXCEPTION, DEBUG | 연결되지 않은 노드를 할당하기 전에 비교 검색을 모두 끝냅니다. |
| 3 | `55d3b3e7c104` | `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리` | A | RB_TREE, EXCEPTION, ALLOCATOR | 부분 생성 객체를 정리하고 복사 대입용 대체 트리를 별도로 완성합니다. |
| 4 | `d72b04c5ddc6` | `test(map): 비교·할당 실패 시 노드 소유권 검증` | A | TEST, RB_TREE, EXCEPTION | 비교와 할당 실패를 주입하고 모든 노드의 생성·해제를 추적합니다. |
| 5 | `0f4dd84e44ed` | `fix(map): 비교자 교환 실패 전에 tree 소유권 유지` | A | RB_TREE, EXCEPTION, DEBUG | 할당자와 트리 소유권을 옮기기 전에 비교자 상태를 먼저 교환합니다. |
| 6 | `55d4ba1fb493` | `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증` | A | TEST, RB_TREE, EXCEPTION | 비교자 대입 실패 상황에서 복사 대입과 공개 `swap`을 검증합니다. |

## 5. 커밋별 학습 기록

### 1. fix(map): 값 allocator 상태로 노드 allocator 구성

- SHA: `ae180871b160`
- 중요도: A
- 태그: RB_TREE, ALLOCATOR, RISK
- 원문에서 정한 역할: 노드 할당자로 재바인딩해도 값 할당자의 상태를 보존합니다.
<!-- 원문 요약: Constructs the rebound node allocator from the map's value allocator state. -->
<!-- 원문 판단 근거: Default-constructing the node allocator silently lost state for custom allocators, separating allocation ownership from the public allocator contract. This small fix restores a significant resource-ownership invariant. -->

#### 해당 SHA에서 확인할 실제 코드

- default/range/복사 생성자에서 node allocator를 value allocator 상태로부터 구성하는 initializer를 확인합니다.
- rebind가 allocated type만 바꾸고 소유권 identity는 유지한다는 점을 allocator 생성자 호출과 `get_allocator()` 관계로 확인합니다.
- 이 SHA 이전 구현이 node allocator를 default-construct하던 지점을 parent diff에서 찾아 stateful allocator에서 어떤 불일치가 생길 수 있었는지 기록합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 value allocator `_alloc`, rebound `_node_alloc`, default/range/복사 생성자 initializer lists, `get_allocator`.
- 필요한 경우 비교할 직전 관련 SHA/parent: 초기 node model `2c6dd8acdd20`은 `_node_alloc()`을 독립 default 생성했습니다.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: allocator rebind type을 default-construct해도 공개 value allocator와 같은 자원 소유자라고 가정했습니다.
- 실제 실패 또는 위험: allocator가 pool id, arena 포인터, accounting 상태를 보유하면 value allocator와 node allocator가 서로 다른 상태를 가집니다. node를 잘못된 소유자로 allocate/deallocate하거나 공개 allocator 규약과 실제 소유권이 분리될 수 있습니다.
- root cause: rebind를 단순 type 변환으로만 보고 상태 전파를 누락했습니다.
- 수정된 불변식/결정: 모든 map 생성자에서 `_node_alloc(_alloc)` 또는 제공된 value allocator에서 rebound allocator를 구성해 type만 node로 바꾸고 identity/상태는 이어갑니다.
- 변경/커밋 순서의 변경: node 할당이 시작되기 전에 두 allocator object의 compatible 상태가 생성자 initializer 단계에서 확정됩니다.
- 실제 수정 코드 근거: 생성자 initializer list의 `_alloc(alloc), _node_alloc(_alloc)` 계열 변경입니다.
- 연결되는 회귀 테스트 SHA/근거: `d72b04c5ddc6`의 tracking allocator 상태가 rebind 후에도 같은 shared block counter와 실패 point를 관찰합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: 공개 allocator type과 internal node allocator type 사이의 rebind 경계입니다. 컴파일은 통과하지만 stateful allocator에서만 드러나는 silent 소유권 오류입니다.
- 상태·소유권·수명 영향: node 할당과 소멸/deallocation이 같은 allocator family 상태에 귀속됩니다.
- 이 커밋이 보장하는 것과 남은 한계: 생성자에서 rebind identity를 유지합니다. 삽입/생성자/대입 실패 시 소유권 transaction 자체는 다음 커밋에서 강화됩니다.
- 다음 관련 커밋과의 연결: `cb08194d17b0`은 node가 allocator에서 나온 직후 tree 소유자에 연결되기 전의 orphan window를 제거합니다.

### 2. fix(map): 삽입 위치를 노드 할당 전에 확정

- SHA: `cb08194d17b0`
- 중요도: A
- 태그: RB_TREE, EXCEPTION, DEBUG
- 원문에서 정한 역할: 연결되지 않은 노드를 할당하기 전에 비교 검색을 모두 끝냅니다.
<!-- 원문 요약: Records the insertion side during tree search so no comparator call occurs after node allocation. -->
<!-- 원문 판단 근거: A comparator exception after allocation could orphan an unlinked node. Moving all comparisons before allocation fixes the root cause at the transaction boundary and restores one-owner-at-all-times behavior. -->

#### 해당 SHA에서 확인할 실제 코드

- 삽입 traversal 중 최종 attachment side를 비교 결과와 함께 저장하는 local 상태를 확인합니다.
- duplicate detection과 모든 comparator call이 `_create_node` 또는 equivalent 할당 이전에 끝나는지 호출 순서를 추적합니다.
- 할당 이후 linking/red-black repair에는 포인터/color 작업만 남는지 확인하여 comparator throw 뒤 unowned node window가 사라졌는지 검증합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 `insert` search loop, `parent`, `insert_left` local 상태, `_create_node`, child link와 삽입 fix-up.
- 필요한 경우 비교할 직전 관련 SHA/parent: `c0fdb8e3f84c`는 node 할당 뒤 parent side를 결정하려 comparator를 다시 호출했습니다.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: search로 parent를 찾은 뒤 node를 만들고, 어느 child에 붙일지 comparator를 한 번 더 호출했습니다.
- 실제 실패 또는 위험: node 할당/생성이 끝난 뒤 그 comparator call이 던지면 node는 아직 tree에 연결되지 않았고 catch 소유자도 없어 orphan/leak가 됩니다.
- root cause: potentially throwing policy 작업을 자원 acquisition 뒤, 소유권 커밋 전 구간에 남겼습니다.
- 수정된 불변식/결정: search 중 매 비교 결과로 `insert_left`를 함께 기록하고 duplicate 판정까지 끝냅니다. 이후 node를 만든 뒤에는 저장된 side로 즉시 link하고 포인터/color repair만 수행합니다.
- 변경/커밋 순서의 변경: 모든 comparator calls → 할당/생성 → parent child link → size/fix-up입니다. allocate 후 compare가 사라집니다.
- 실제 수정 코드 근거: search loop의 `bool insert_left` 갱신과 `_create_node` 뒤 comparator가 없는 link 브랜치입니다.
- 연결되는 회귀 테스트 SHA/근거: `d72b04c5ddc6`이 comparator fail point를 sweep하고 할당 이후 comparator invocation이 발생하면 outstanding node count로 드러나게 합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: acquired node가 아직 root/parent에서 reachable하지 않은 짧은 구간입니다.
- 상태·소유권·수명 영향: comparator 실패는 acquisition 전에 발생하므로 map 상태/allocator blocks가 변하지 않습니다. acquisition 성공 후 node는 즉시 map reachability에 들어갑니다.
- 이 커밋이 보장하는 것과 남은 한계: normal 삽입에서 post-할당 comparator throw orphan을 제거합니다. bulk 생성과 대입이 여러 successful inserts 뒤 실패하는 되돌리기는 별도입니다.
- 다음 관련 커밋과의 연결: `55d3b3e7c104`가 여러 삽입으로 만든 partial tree 전체의 transaction을 처리합니다.

### 3. fix(map): 생성과 복사 대입 실패를 임시 tree로 격리

- SHA: `55d3b3e7c104`
- 중요도: A
- 태그: RB_TREE, EXCEPTION, ALLOCATOR
- 원문에서 정한 역할: 부분 생성 객체를 정리하고 복사 대입용 대체 트리를 별도로 완성합니다.
<!-- 원문 요약: Adds constructor rollback and copy-and-swap-style temporary-tree assignment for map. -->
<!-- 원문 판단 근거: Range/copy construction now clears partial trees on failure, while assignment preserves the destination until a complete replacement exists. This is significant resource and state transaction engineering, though it does not redefine the tree structure itself. -->

#### 원문에서 정한 핵심 판단

- 문제: range 생성과 복사 생성은 노드를 하나씩 추가하다가 비교나 할당이 실패해도 부분 생성된 트리를 정리하지 못했습니다. 복사 대입은 대체 트리를 만들기 전에 대상을 비웠으므로 실패하면 기존 값이 사라지고 일부만 만들어진 대체 상태가 남을 수 있었습니다.
- 결정: 생성자는 실패를 잡아 새로 만들어져 도달 가능한 모든 노드를 정리한 뒤 예외를 다시 던집니다. 대입은 대상의 할당자와 원본의 비교자를 사용해 임시 map을 완성하고, 생성이 성공한 뒤에만 비교자와 트리 상태를 교환합니다.
- 중요한 이유: 여러 노드의 소유권을 한꺼번에 바꾸는 연산의 상태 확정 지점을 정했습니다. 실패해도 기존 대상을 보존하고, 부분 생성된 트리가 소유자 없이 남거나 외부에 반영되지 않게 합니다.

#### 해당 SHA에서 확인할 실제 코드

- range/복사 생성자에서 삽입 실패를 catch하고 `clear()` 후 rethrow하는 되돌리기 경로를 확인합니다.
- copy 대입이 destination allocator와 원본 comparator로 임시 객체 map을 먼저 완성하는지 확인합니다.
- complete 임시 객체 tree 이후 비공개 exchange가 새 tree를 install하고 임시 객체가 old destination을 destroy하는 소유권 흐름을 그립니다.
- 원본이 comparator policy swap 자체의 throw ordering은 후속 커밋에서 hardened된다고 명시하므로, 이 SHA에서 그 residual risk를 표시합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 range/copy constructors, `operator=`, 비공개 tree/comparator exchange helper, `clear`.
- 필요한 경우 비교할 직전 관련 SHA/parent: clear-then-insert 대입 `c0fdb8e3f84c`; 실패 테스트 `d72b04c5ddc6`; ordering fix `0f4dd84e44ed`.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: 생성자 body에서 successful inserts가 자동으로 정리될 것이라 가정했고, 대입은 destination을 먼저 clear했습니다.
- 실제 실패 또는 위험: 생성자가 완성되지 않으면 map 소멸자가 호출되지 않아 partial nodes가 남을 수 있습니다. 대입 실패는 original 대상을 잃고 partial replacement만 남깁니다.
- root cause: multi-node 빌드의 임시 객체 소유권과 상태 확정 지점이 명시되지 않았습니다.
- 수정된 불변식/결정: 생성자 catch가 현재 reachable tree를 clear합니다. 대입은 대상 allocator와 원본 comparator로 complete 임시 객체를 만들고 성공 후에만 policy/tree 상태를 exchange합니다.
- 변경/커밋 순서의 변경: 임시 객체 빌드 → complete 확인 → exchange 커밋 → 임시 객체 소멸자가 기존 대상 tree 정리입니다. 빌드 실패면 대상 untouched, 임시 객체/생성자 partial tree만 clear합니다.
- 실제 수정 코드 근거: 생성자 `try { insert(...) } catch (...) { clear(); throw; }`, `operator=`의 임시 객체 map과 비공개 exchange 호출입니다.
- 연결되는 회귀 테스트 SHA/근거: `d72b04c5ddc6`이 range/복사 생성 comparison/할당 failures와 대입 대상 보존/outstanding nodes를 검사합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: object 생성이 완료되기 전 소멸자 자동 호출이 없는 구간과 대입 replacement 빌드 구간입니다.
- 상태·소유권·수명 영향: partial 생성자 nodes는 under-생성 object가 clear하고, replacement nodes는 임시 객체가 소유합니다. 커밋 후 임시 객체가 기존 대상 소유자가 됩니다.
- 이 커밋이 보장하는 것과 남은 한계: comparator/할당 실패에서 partial 생성자 정리와 대상 보존을 제공합니다. comparator exchange 자체가 throw할 때 tree/allocator보다 뒤에서 수행되면 partial 커밋 위험이 남아 `0f4dd84e44ed`가 순서를 고칩니다.
- 다음 관련 커밋과의 연결: `d72b04c5ddc6`이 transaction을 검증하고, `0f4dd84e44ed`/`55d4ba1fb493`가 policy-대입 실패를 별도로 다룹니다.

### 4. test(map): 비교·할당 실패 시 노드 소유권 검증

- SHA: `d72b04c5ddc6`
- 중요도: A
- 태그: TEST, RB_TREE, EXCEPTION
- 원문에서 정한 역할: 비교와 할당 실패를 주입하고 모든 노드의 생성·해제를 추적합니다.
<!-- 원문 요약: Adds comparator and allocator failure injection for insertion, constructors, copying, and assignment. -->
<!-- 원문 판단 근거: The suite verifies that every allocated node remains owned or is released and that failed assignment preserves the target. It directly validates the map transaction boundaries established by the preceding fixes. -->

#### 해당 SHA에서 확인할 실제 코드

- stateful comparator와 rebound-surviving allocator 테스트 준비 코드의 identity/실패 counter/outstanding node tracking을 확인합니다.
- 삽입 comparator 실패 sweep가 할당 이후 comparator call이 없다는 실제 코드의 불변식를 어떻게 검증하는지 확인합니다.
- range/복사 생성의 comparison/할당 실패 후 residual node count가 zero인지 확인합니다.
- copy 대입 실패에서 pre-existing 대상 keys와 baseline 할당 count가 보존되는지 확인합니다.
- generated 입력 반복자가 다른 컨테이너 구현에 의존하지 않고 테스트 values를 공급하는 방식을 확인합니다.
- 확인한 파일/심볼: `tests/test_map_exceptions.cpp`의 `throwing_less`, shared comparator 상태, tracking allocator/shared block 상태, generated 입력 반복자, 삽입/생성자/copy/대입 실패 테스트.
- 필요한 경우 비교할 직전 관련 SHA/parent: production fixes `ae180871b160`, `cb08194d17b0`, `55d3b3e7c104`.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: rebind allocator identity가 유지되고, every 할당은 tree/임시 객체 소유자에 연결되거나 released되며, failed bulk 작업이 대상을 보존해야 합니다.
- 재현하는 실패 또는 경계: comparator call index, allocator 할당 index, range/복사 생성자 progress, copy 대입 replacement progress입니다.
- 테스트 방식: 결정적 comparator/할당 실패 주입과 outstanding-node accounting입니다.
- 통과하는 production 코드 경로: 삽입 search/acquisition/link, range/복사 생성자 catch/clear, 임시 객체-tree 대입 빌드/exchange입니다.
- 이 테스트가 증명하는 것: 검사한 fail points에서 삽입은 leak 없이 실패하고, constructors는 residual block 0이며, 대입 대상 keys와 baseline 대상 할당 count가 유지됩니다. rebind된 allocator copies가 같은 shared 소유자 상태를 사용합니다.
- 이 테스트가 증명하지 않는 것: comparator 대입 중 실패와 공개 swap 커밋 ordering은 이 SHA의 suite 범위가 아닙니다. 모든 third-party allocator/comparator 동작도 exhaustive하지 않습니다.
- 성격: transaction 경계를 고정하는 결정적 실패-injection 회귀입니다.
- 후속 변경에서 막아야 하는 회귀: allocate-after/compare ordering 역전, 생성자 catch 제거, 대입 대상 조기 clear, rebind 상태 단절입니다.
- 실행 증거: 테스트 준비 코드와 fail-point loops를 코드로 확인했습니다. 테스트 실행 파일을 실행하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: `throwing_less`는 지정 call에서 던지고 tracking allocator는 지정 acquisition에서 던집니다. generated 반복자는 external 컨테이너 동작 없이 range 생성을 진행시킵니다.
- 상태·소유권·수명 영향: shared allocator 상태가 rebound type마다 같은 outstanding count를 보므로 node block leak를 직접 계측합니다.
- 이 커밋이 보장하는 것과 남은 한계: preceding fixes의 compare/할당 transaction을 강하게 뒷받침합니다. policy object 대입 실패는 다음 fix/테스트가 필요합니다.
- 다음 관련 커밋과의 연결: `0f4dd84e44ed`가 comparator 대입을 소유권 커밋보다 앞으로 옮기고 `55d4ba1fb493`가 그 실패를 주입합니다.

### 5. fix(map): 비교자 교환 실패 전에 tree 소유권 유지

- SHA: `0f4dd84e44ed`
- 중요도: A
- 태그: RB_TREE, EXCEPTION, DEBUG
- 원문에서 정한 역할: 할당자와 트리 소유권을 옮기기 전에 비교자 상태를 먼저 교환합니다.
<!-- 원문 요약: Moves comparator exchange before allocator and tree ownership exchange in map swap and assignment commit. -->
<!-- 원문 판단 근거: If comparator assignment throws after tree pointers move, ordering policy and physical ownership can diverge. This small ordering fix prevents partial ownership transfer and restores a high-risk exception invariant. -->

#### 해당 SHA에서 확인할 실제 코드

- 공개 swap과 비공개 대입 exchange에서 comparator exchange가 allocator/root/size 소유권 exchange보다 먼저 수행되는 순서를 확인합니다.
- comparator 대입 throw 시점에 두 map의 original root/size/allocator가 아직 움직이지 않았는지 상태 변경 순서를 표시합니다.
- comparator step 성공 후 소유권 swap과 header refresh가 수행되는 커밋 phase를 구분합니다.
- 확인한 파일/심볼: `include/ft_map.hpp`의 공개 `swap`과 비공개 tree/comparator exchange helper; comparator swap line과 allocator/header/size swap/refresh lines.
- 필요한 경우 비교할 직전 관련 SHA/parent: transaction 커밋 `55d3b3e7c104`; 회귀 `55d4ba1fb493`.

#### 실패 원인과 수정 내용

- 기존 가정 또는 직전 구현 상태: tree/header/allocator 소유권을 먼저 교환한 뒤 comparator를 교환해도 된다고 봤습니다.
- 실제 실패 또는 위험: comparator 대입이 그 뒤 던지면 physical tree는 상대 map으로 이동했는데 ordering policy 또는 allocator 상태는 원래 위치에 남아 search ordering과 deallocation 소유권이 분리됩니다.
- root cause: potentially throwing policy 커밋을 irreversible physical 소유권 movement 뒤에 배치했습니다.
- 수정된 불변식/결정: comparator exchange를 공개 swap과 비공개 대입 커밋의 첫 단계로 수행합니다. 이 단계가 완료되기 전 root/header/size/allocator는 움직이지 않습니다.
- 변경/커밋 순서의 변경: comparator policy exchange → allocator/tree/header/size 소유권 exchange → moved root를 new header에 reattach/refresh입니다.
- 실제 수정 코드 근거: 두 exchange function에서 comparator swap line이 physical 상태 swaps보다 위로 이동한 diff입니다.
- 연결되는 회귀 테스트 SHA/근거: `55d4ba1fb493`이 comparator 대입 throw를 copy 대입과 공개 swap에 주입하고 contents/allocator/node counts/usability를 검사합니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: comparator copy/대입이 throw 가능한 policy 작업이라는 점입니다.
- 상태·소유권·수명 영향: 실패 시 physical node tree와 allocator owners는 원래 map에 남고 complete replacement 임시 객체는 자신의 allocator로 해제됩니다.
- 이 커밋이 보장하는 것과 남은 한계: tree 소유권이 comparator exchange 실패 뒤 부분 이동하지 않게 합니다. comparator type 자체가 대입 전에 상태를 변형한 뒤 던지는 경우까지 일반적인 강한 예외 보장을 구현이 별도로 복원하지는 않습니다. 회귀 테스트 준비 코드는 throw 전에 자기 상태를 유지하는 동작을 검증합니다.
- 다음 관련 커밋과의 연결: `55d4ba1fb493`가 distinct ordering directions/allocator owners로 policy-tree mismatch를 observable하게 만듭니다.

### 6. test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증

- SHA: `55d4ba1fb493`
- 중요도: A
- 태그: TEST, RB_TREE, EXCEPTION
- 원문에서 정한 역할: 비교자 대입 실패 상황에서 복사 대입과 공개 `swap`을 검증합니다.
<!-- 원문 요약: Injects comparator-assignment failure into copy assignment and public swap with distinct allocator owners and order directions. -->
<!-- 원문 판단 근거: The tests make partial policy exchange observable and verify both maps retain contents, allocator identity, node counts, and usability. They strongly validate the subtle commit-ordering fix. -->

#### 해당 SHA에서 확인할 실제 코드

- 대입에서 throw할 수 있는 comparator와 distinct allocator identity/outstanding block을 추적하는 테스트 준비 코드를 확인합니다.
- copy 대입 실패에서 대상의 original descending 순서, allocator node count, 추가 삽입 가능성이 보존되는지 확인합니다.
- 공개 swap 실패에서 두 map의 contents, allocator identity, node 소유권 count가 모두 original 상태인지 확인합니다.
- completed 임시 객체 tree가 실패 뒤 release되는 검사문을 확인합니다.
- 확인한 파일/심볼: `tests/test_map_policy_exceptions.cpp`의 stateful/throwing comparator, tracking allocator 소유자 상태, copy-대입 실패와 공개-swap 실패 테스트.
- 필요한 경우 비교할 직전 관련 SHA/parent: production ordering fix `0f4dd84e44ed`.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: comparator 대입 실패 전에 policy/tree/allocator 소유권이 partial exchange되지 않고 각 map이 original ordering과 node 소유자를 유지해야 합니다.
- 재현하는 실패 또는 경계: opposite ordering directions를 가진 comparators, distinct allocator ids, completed replacement 임시 객체, comparator 대입 throw입니다.
- 테스트 방식: 결정적 policy-대입 실패 주입 + allocator 소유권 accounting + post-실패 usability check입니다.
- 통과하는 production 코드 경로: copy 대입 임시 객체 생성/비공개 exchange, 공개 swap, comparator swap, 임시 객체 소멸자, subsequent insert/traversal입니다.
- 이 테스트가 증명하는 것: chosen throwing comparator에서 failed copy 대입은 original 대상 순서/allocator/count를 유지하고 임시 객체 nodes를 release하며, failed 공개 swap은 양쪽 contents/allocator/count를 유지합니다. 두 maps는 이후 insert/search/traversal에 사용 가능합니다.
- 이 테스트가 증명하지 않는 것: comparator가 자기 대입 중 상태를 부분 변경한 뒤 던지는 arbitrary 동작, allocator swap 자체의 throw, concurrent access는 다루지 않습니다.
- 성격: subtle 커밋-ordering fix를 직접 고정하는 결정적 회귀 테스트입니다.
- 후속 변경에서 막아야 하는 회귀: comparator exchange를 tree/allocator swap 뒤로 이동하거나, failed 임시 객체 소유권을 대상에 일부 연결하고 정리를 누락하는 회귀입니다.
- 실행 증거: 테스트 테스트 준비 코드와 검사문을 코드로 검사했으며 실행하지 않았습니다.

#### 중요도 A 핵심 확인

- 중요 경계/실패 처리: 임시 객체 tree가 완성된 뒤 최종 커밋 첫 policy 대입에서 실패하는 late 실패입니다.
- 상태·소유권·수명 영향: 대상/원본 physical trees는 그대로이고 임시 객체가 replacement nodes를 소유한 채 scope unwinding으로 해제됩니다. allocator counters가 이 소유권 분리를 확인합니다.
- 이 커밋이 보장하는 것과 남은 한계: 해당 comparator/allocator 모델에서 상태 보존과 leak absence를 증명합니다. 언어 수준의 모든 policy 구현을 일반화하지는 않습니다.
- 다음 관련 커밋과의 연결: 이 개발 흐름의 최종 transaction rule을 검증하며 이후 acceptance/sanitizer suite에 포함됩니다.

## 6. 불변식 변화 기록

### 원문에서 정한 관련 불변식

- map의 비교자는 키 동등성과 트리 순서를 정하고, 노드 형식으로 재바인딩한 할당자는 실제 노드 소유를 담당합니다. 복사·대입·`swap`은 비교자·할당자 상태와 소유 트리 상태를 함께 일관되게 유지해야 합니다.
- 모든 실패 지점에서 각 map 노드는 정확히 하나의 소유자에게 속하거나 이미 해제되어야 합니다. 비교자 대입이 예외를 던져도 비교자·할당자·트리 소유권이 일부만 교환된 상태가 되어서는 안 됩니다.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 커밋 | 부족함이 드러난 커밋/상태 | 강화·복구한 fix | 고정한 테스트/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| Comparator defines ordering; rebound allocator defines physical 소유권; bulk operations keep them coherent | comparator/tree는 `c0fdb8e3f84c`, node allocator는 `2c6dd8acdd20` | node allocator default 생성으로 공개 allocator 상태 상실; 대입/swap 커밋 순서 위험 | `ae180871b160`, `55d3b3e7c104`, `0f4dd84e44ed` | `d72b04c5ddc6`, `55d4ba1fb493` | 생성자 allocator initializers, 임시 객체 대입, policy-first exchange, 소유자 counters |
| Every 실패 leaves each node with exactly one 소유자 or released; no partial policy/tree exchange | node create 되돌리기는 `2c6dd8acdd20` | post-할당 comparator throw orphan, partial constructors, clear-first 대입, tree-first comparator swap | `cb08194d17b0`, `55d3b3e7c104`, `0f4dd84e44ed` | `d72b04c5ddc6`, `55d4ba1fb493` | compare-before-allocate, catch-clear, 임시 객체 소유자, policy-first 커밋, block counts |

## 7. 문제 → 수정 → 검증 연결

| 기존 상태/production change | fix 또는 verification | 원문에서 확정된 연결 관점 | 실제 실패/root cause | 테스트가 통과하는 실제 실행 경로 |
| --- | --- | --- | --- | --- |
| `ae180871b160` | `d72b04c5ddc6` | stateful allocator 소유권이 실패 suite에서 간접 검증 | rebind type default 생성이 allocator identity를 끊음 | tracking allocator copy/rebind → node operations → shared outstanding counts |
| `cb08194d17b0` | `d72b04c5ddc6` | comparator 실패 after 할당이 없음을 sweep | attachment-side comparator가 acquired unlinked node 뒤 throw | compare fail sweep → 할당 count/owned node 검사문 |
| `55d3b3e7c104` | `d72b04c5ddc6` | 생성자 되돌리기와 대입 대상 보존 검증 | partial 생성자 정리 부재, 대상 clear-first | range/copy/대입 실패 → clear/temp 소멸 → keys/counts |
| `0f4dd84e44ed` | `55d4ba1fb493` | comparator-대입 실패에서 policy/tree 소유권 보존 검증 | physical 소유권을 policy보다 먼저 옮긴 partial 커밋 | comparator 대입 throw → original contents/allocator/count/usability |

## 8. 소유권·상태·담당 변화

| 시점 | Owner / 상태 / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `ae180871b160` | 노드 할당자로 재바인딩해도 값 할당자의 상태를 보존합니다. | node allocator 독립 default 상태 | value allocator family 상태로 node allocator 구성 | 생성자 initializers |
| `cb08194d17b0` | 연결되지 않은 노드를 할당하기 전에 비교 검색을 모두 끝냅니다. | acquired node와 link 사이 policy throw 가능 | all compare before acquisition, then immediate link | `insert_left`, `_create_node` ordering |
| `55d3b3e7c104` | 부분 생성 객체를 정리하고 복사 대입용 대체 트리를 별도로 완성합니다. | under-생성/대상이 부분 상태 소유 | 생성자 clear 또는 임시 객체가 replacement 소유 | catch-clear, 임시 객체 대입 |
| `d72b04c5ddc6` | 비교와 할당 실패를 주입하고 모든 노드의 생성·해제를 추적합니다. | 소유권 근거가 코드 추론 | shared 테스트 상태가 every block 계측 | map exception 테스트 준비 코드 |
| `0f4dd84e44ed` | 할당자와 트리 소유권을 옮기기 전에 비교자 상태를 먼저 교환합니다. | physical 소유권 먼저 교환 | policy 성공 뒤에만 physical 커밋 | swap/exchange order |
| `55d4ba1fb493` | 비교자 대입 실패 상황에서 복사 대입과 공개 `swap`을 검증합니다. | late policy 실패 근거 부족 | distinct 소유자/order 테스트가 보존 고정 | policy exception 테스트 |

## 9. 개발 흐름의 최종 상태

- 최종적으로 성립한 표현/상태: map은 comparator policy, 공개 value allocator, compatible rebound node allocator, header-owned tree를 함께 보유합니다. replacement tree는 대상 밖 임시 객체에서 완성되고 커밋 전까지 독립 소유자입니다.
- 최종적으로 보장하는 불변식: comparator calls는 삽입 acquisition 전에 끝납니다. partial constructors는 reachable nodes를 clear합니다. failed 대입/swap은 physical tree/allocator 소유권을 부분 교환하지 않으며 각 node는 original map 또는 임시 객체에 속하거나 해제됩니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: comparator/allocator의 자체 copy/대입 semantics가 내부 상태를 바꾸고 throw하는 임의 구현에 대해 별도 universal 되돌리기는 없습니다. 테스트는 repository의 결정적 테스트 준비 코드가 정의한 실패 동작을 검증합니다.
- 최종 verification 근거: comparator/할당 fail-point sweeps, 생성자 residual count 0, 대입 대상 보존, distinct allocator/comparator swap 실패 테스트를 코드로 확인했습니다. 실제 실행은 수행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point 커밋: unowned acquisition window를 제거한 `cb08194d17b0`, 임시 객체-tree transaction `55d3b3e7c104`, policy-first 커밋 `0f4dd84e44ed`입니다.

## 10. 최종 설계와 실행 흐름

아래 단계명은 원문이 정의한 개발 흐름 progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 커밋 | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | 실패/정리 | 다음 단계에 남기는 불변식 |
| --- | --- | --- | --- | --- | --- | --- |
| Allocator rebind identity | `ae180871b160` | map 생성자 initializers | value allocator 상태 | rebound node allocator를 value allocator에서 구성 | 생성 전 policy 단계 | 할당/deallocation 소유자 compatibility |
| Comparison-before-할당 경계 | `cb08194d17b0` | `insert` search/link | key, comparator, current tree | parent와 side 확정 후 node acquire/link | compare throw는 acquisition 전 | no unowned node after policy throw |
| Temporary-tree 생성/대입 | `55d3b3e7c104` | constructors, `operator=`, exchange | 입력/원본과 original 대상 | 임시 객체 complete 후 커밋 | partial clear/temp 소멸자 | original 대상 preserved until 커밋 |
| Failure-injection 소유권 proof | `d72b04c5ddc6` | map exception 테스트 | compare/alloc fail index | 실제 실행 경로 반복 주입 | outstanding nodes/keys 관찰 | transaction 근거 |
| Comparator-before-소유권 exchange | `0f4dd84e44ed` | 공개/비공개 swap | two policies and trees | comparator exchange 먼저, then physical 소유권 | policy throw 시 trees untouched | no partial physical transfer |
| Comparator-대입 회귀 | `55d4ba1fb493` | policy exception 테스트 | opposite ordering/distinct allocators | late 실패 후 original 상태 검사 | 임시 객체 nodes released | policy/tree/소유자 coherence 근거 |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 커밋 기록에 최종 HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A 커밋은 결정, 실패 경계, 소유권/상태 변화를 설명할 수 있습니다.
- [x] 테스트·성능 관련 커밋은 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → 실패/risk → root cause → 수정 → 회귀 연결을 설명할 수 있습니다.
- [x] Invariant ledger가 커밋 history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] 개발 흐름 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 05-stateful-allocators-map-failure-transactions.md =====

===== BEGIN FILE: 06-header-only-public-surface-automated-acceptance.md =====
# 헤더 전용 공개 API와 자동 검증

## 1. 개발 흐름의 목표

### 원문에서 정한 의의

헤더 전용 라이브러리는 하나의 통합 테스트에서만 성공해도 include 순서, 중복 정의, 외부 소비자 링크 문제를 숨길 수 있습니다. 공개 헤더 단독 컴파일, 여러 번역 단위 링크, 저장소 밖 소비자, sanitizer와 플랫폼별 CI로 검증 범위를 넓힙니다.

<details>
<summary>영문 원문</summary>

A header-only library can appear correct inside one monolithic test while still depending on include order, emitting duplicate definitions, or compiling only under one toolchain. This progression expands the acceptance boundary from a convenience include, to independently self-contained headers, to a real linked consumer, and finally to sanitizer and cross-platform automation. These commits do not define the core containers, but they make the finished public surface reproducible and consumable outside the repository's internal test arrangement.

</details>


### 이 개발 흐름에서 확인할 내용

- 위 의의에 제시된 변화 과정을 각 커밋의 실제 SHA 코드로 재구성합니다.
- 원문이 확정한 커밋 역할과 중요도를 바꾸지 않고, 실제 구현/실패/테스트 근거만 직접 채웁니다.

### 원문에서 확인되는 설계

- 검증은 공개 API 비교 테스트, 경계·실패 테스트, 반복자 상태 테스트, 제한된 내부 트리 검사기, 결정적 무작위 연산, 복잡도 상한, 개별 헤더 컴파일, 외부 소비자 링크 테스트, sanitizer, 컴파일러·플랫폼 CI로 나뉩니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- header-only library가 monolithic 테스트 하나를 통과해도 consumer에게 깨질 수 있는 이유는 무엇인가?
- aggregate header smoke, independent-header 컴파일, multi-TU link가 서로 다른 어떤 실패 class를 잡는가?
- sanitizer 빌드를 normal 빌드와 격리해야 하는 이유는 무엇인가?
- CI matrix가 C++98 portability와 memory-safety acceptance를 어떻게 repeatable 규약으로 만드는가?

## 3. 완료 기준

- B: 개발 흐름에서 맡는 구현 역할, 필요한 상태 변화와 핵심 코드 위치를 해당 SHA 기준으로 확인할 수 있어야 합니다.
- C: 개발 흐름 이해에 필요한 맥락과 최소 검증 포인트만 확인합니다. S/A와 같은 깊이의 분석을 강제하지 않습니다.
- 모든 커밋은 해당 SHA의 코드 또는 테스트/빌드 diff를 근거로 기록합니다.
- 개발 흐름 최종 설명은 원문 요약을 복사하는 것으로 끝내지 않고, 직접 확인한 코드 근거와 커밋 간 변화로 재구성합니다.

## 4. 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 정한 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `80e169e83212` | `feat(headers): 공용 도구와 순차 컨테이너 통합 헤더 추가` | B | PUBLIC_API, ARCH | 통합 헤더를 도입합니다. |
| 2 | `3c64a69dd252` | `test(headers): 통합 헤더의 순차 컨테이너 표면 검증` | C | TEST, PUBLIC_API | 현재 순차 컨테이너 API가 통합 헤더를 통해 컴파일되는지 확인합니다. |
| 3 | `112af1753538` | `feat(map): 관계 연산과 통합 공개 헤더 완성` | B | PUBLIC_API, RB_TREE | 통합 공개 진입점에 map을 추가합니다. |
| 4 | `d938c0079994` | `test(headers): 공개 헤더를 각각 독립 compile` | B | TEST, PUBLIC_API, CXX98 | 각 공개 헤더를 첫 include로 독립 컴파일합니다. |
| 5 | `072c49832ddc` | `test(consumer): 다중 번역 단위 공개 헤더 사용 검증` | B | TEST, PUBLIC_API, INTEGRATION | 독립 vector/map 사용 코드를 여러 번역 단위에서 링크합니다. |
| 6 | `1be03ae8daef` | `build(makefile): 격리된 sanitizer 검사 대상 추가` | B | PUBLIC_API, PRACTICAL, RISK | 전체 인수 검증을 격리된 ASan/UBSan 계측으로 실행합니다. |
| 7 | `228f457988be` | `ci: compiler 행렬과 sanitizer 검사 구성` | B | CXX98, PUBLIC_API, PRACTICAL | 컴파일러·플랫폼·sanitizer 검사를 CI로 자동화합니다. |

## 5. 커밋별 학습 기록

### 1. feat(headers): 공용 도구와 순차 컨테이너 통합 헤더 추가

- SHA: `80e169e83212`
- 중요도: B
- 태그: PUBLIC_API, ARCH
- 원문에서 정한 역할: 통합 헤더를 도입합니다.
<!-- 원문 요약: Adds an aggregate public header for the utility layer, vector, and stack. -->
<!-- 원문 판단 근거: This creates a deliberate packaging boundary for a header-only library. It matters to consumers, but the implementation is simple composition rather than a major runtime or ownership decision. -->

#### 해당 SHA에서 확인할 실제 코드

- `ft_containers.hpp`가 traits/반복자/알고리즘/pair/vector/stack의 aggregate entry point로 어떤 component headers를 include하는지 확인합니다.
- component header를 직접 include하는 경로가 제거되지 않았는지 공개 surface 관점에서 확인합니다.
- 이 커밋은 런타임 동작이 아니라 packaging 경계를 만든다는 원문 역할과 실제 diff가 일치하는지 기록합니다.
- 확인한 파일/심볼: `include/ft_containers.hpp`의 include guard와 `ft_algorithm.hpp`, `ft_iterator.hpp`, `ft_pair.hpp`, `ft_stack.hpp`, `ft_type_traits.hpp`, `ft_vector.hpp` include 목록입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: 직전에는 각 component header만 존재했고 하나의 공개 bundle은 없었습니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: consumer가 필요한 utility/컨테이너 headers를 개별 선택해 include해야 했습니다.
- 해결하려던 문제: 정상 조합을 매 consumer가 반복해서 구성하면 누락과 include-order 차이가 공개 사용법에 섞입니다.
- 선택한 결정: 기존 component headers를 변경하거나 숨기지 않고, 그 위에 `ft_containers.hpp`라는 convenience entry point를 추가했습니다.
- 새로 생긴 책임 경계 또는 상태 변화: component header는 개별 API를 계속 담당하고 aggregate header는 지원되는 조합을 노출하는 packaging 책임만 가집니다. 이 SHA에는 map이 아직 포함되지 않습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: header-only library의 첫 명시적 공개 bundle을 만듭니다.
- 핵심 코드와 상태 변화: 런타임 상태는 없습니다. preprocessor include graph에 aggregate root가 추가됩니다.
- 다음 커밋에 넘기는 전제: 이 bundle 하나로 기존 순차 컨테이너 테스트가 그대로 컴파일/run되어야 합니다.

### 2. test(headers): 통합 헤더의 순차 컨테이너 표면 검증

- SHA: `3c64a69dd252`
- 중요도: C
- 태그: TEST, PUBLIC_API
- 원문에서 정한 역할: 현재 순차 컨테이너 API가 통합 헤더를 통해 컴파일되는지 확인합니다.
<!-- 원문 요약: Switches the existing test to include the aggregate header instead of component headers. -->
<!-- 원문 판단 근거: The change is a small integration smoke test for one include list. It adds little new technical evidence beyond confirming that the bundle contains the already tested headers. -->

#### 해당 SHA에서 확인할 실제 코드

- 기존 consumer-style 테스트가 component headers 대신 aggregate header 하나만 include하도록 바뀐 diff를 확인합니다.
- 런타임 검사문 자체는 바뀌지 않았는지 확인하여 실패가 공개 헤더 composition 문제로 귀속되는 구조를 기록합니다.
- 확인한 파일/심볼: `tests/test_containers.cpp`의 여러 `ft_*.hpp` includes가 `#include "ft_containers.hpp"` 하나로 교체된 지점입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: aggregate header 도입 `80e169e83212`입니다.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: aggregate header가 utility, vector, stack의 기존 공개 symbols와 필요한 내부 dependencies를 빠짐없이 노출해야 합니다.
- 재현하는 실패 또는 경계: bundle include 누락 또는 include ordering 때문에 기존 테스트 소스가 컴파일되지 않는 경우입니다.
- 테스트 방식: 기존 broad behavioral 테스트의 include surface만 aggregate header로 바꾸는 통합 smoke입니다.
- 통과하는 production 코드 경로: `ft_containers.hpp`의 transitive include graph를 거쳐 기존 utility/vector/stack 테스트 코드가 컴파일되고 실행됩니다.
- 이 테스트가 증명하는 것: 한 번역 단위에서 현재 순차 surface가 aggregate include만으로 기존 checks를 사용할 수 있음을 증명합니다.
- 이 테스트가 증명하지 않는 것: 각 component header의 독립 self-containment, 여러 translation units의 링크/ODR, map 포함 여부, compiler/platform portability는 증명하지 않습니다.
- 성격: 특정 회귀보다 공개 bundle의 broad smoke입니다.
- 후속 변경에서 막아야 하는 회귀: aggregate header에서 필요한 component include를 제거해 기존 테스트가 직접 include에 우연히 의존하도록 만드는 회귀입니다.
- 실행 증거: include diff와 기존 테스트 연결을 코드로 검사했으며 executable은 실행하지 않았습니다.

#### C-level 최소 확인

- 개발 흐름 이해에 필요한 맥락: 구현 동작을 새로 시험한 것이 아니라 테스트의 진입 include를 바꿔 packaging 경계를 통과시켰습니다.
- 최소 코드/검증 근거: `tests/test_containers.cpp`의 include replacement 한 곳이며 테스트 body 변경은 없습니다.

### 3. feat(map): 관계 연산과 통합 공개 헤더 완성

- SHA: `112af1753538`
- 중요도: B
- 태그: PUBLIC_API, RB_TREE
- 원문에서 정한 역할: 통합 공개 진입점에 map을 추가합니다.
<!-- 원문 요약: Adds map value comparison, relational operators, non-member swap, and inclusion in the aggregate header. -->
<!-- 원문 판단 근거: This completes the baseline public surface through established shared algorithms. It is useful integration work without a new core data-structure decision. -->

#### 해당 SHA에서 확인할 실제 코드

- `ft_containers.hpp`에 map이 추가된 include diff를 확인합니다.
- 같은 SHA에서 map relations/value comparator/non-member swap이 공개 surface에 완성되는 위치를 확인하되, 이 개발 흐름에서는 aggregate header로 노출되는 경계에 초점을 둡니다.
- aggregate consumer가 internal include graph를 알 필요 없이 map까지 사용할 수 있는지 해당 SHA의 테스트/include 관계로 확인합니다.
- 확인한 파일/심볼: `include/ft_containers.hpp`의 `#include "ft_map.hpp"`; `include/ft_map.hpp`의 `value_compare`, `value_comp`, six relational operators, non-member `swap`입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: `80e169e83212`의 aggregate list에는 map이 없었습니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: aggregate entry point는 utility/vector/stack까지만 포함해 associative 컨테이너 surface가 불완전했습니다.
- 해결하려던 문제: map을 직접 include해야 했고, map의 공개 comparison/swap surface도 동일 커밋에서 완성될 필요가 있었습니다.
- 선택한 결정: map header를 aggregate list에 추가하고, key comparator를 value pairs에 적용하는 `value_compare`, shared `ft::equal`/`ft::lexicographical_compare` 기반 relations, member swap forwarding을 공개했습니다.
- 새로 생긴 책임 경계 또는 상태 변화: `ft_containers.hpp`가 이 repository의 utility, sequential, adaptor, associative surface를 한 entry point에서 노출합니다. 런타임 tree 표현은 이 커밋에서 새로 정의되지 않습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: aggregate 공개 API를 map까지 확장해 baseline bundle을 완성합니다.
- 핵심 코드와 상태 변화: include graph에 `ft_map.hpp`가 추가되고 map 공개 non-member/value-comparison symbols가 생깁니다.
- 다음 커밋에 넘기는 전제: bundle만 아니라 각 component header도 독립적으로 자신의 dependencies를 include해야 합니다.

### 4. test(headers): 공개 헤더를 각각 독립 compile

- SHA: `d938c0079994`
- 중요도: B
- 태그: TEST, PUBLIC_API, CXX98
- 원문에서 정한 역할: 각 공개 헤더를 첫 include로 독립 컴파일합니다.
<!-- 원문 요약: Compiles each public header independently as the first include in a minimal translation unit. -->
<!-- 원문 판단 근거: This enforces header self-containment and catches accidental transitive dependencies, an important header-only API practice. It is significant integration hygiene rather than core container logic. -->

#### 해당 SHA에서 확인할 실제 코드

- 각 공개 헤더마다 정확히 하나의 library header를 first include하는 minimal 번역 단위 목록을 확인합니다.
- 각 번역 단위가 representative 작업을 instantiate하여 선언뿐 아니라 필요한 template dependency까지 컴파일하는지 확인합니다.
- Makefile의 dedicated `headers` 대상과 object 출력 경로를 확인합니다.
- 확인한 파일/심볼: `tests/headers/{ft_algorithm,ft_containers,ft_iterator,ft_map,ft_pair,ft_stack,ft_type_traits,ft_vector}.cpp`; Makefile의 `HEADER_TEST_SOURCES`, `HEADER_TEST_OBJECTS`, `build/headers/%.o`, `headers` 대상입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: aggregate expansion `112af1753538` 뒤의 공개 헤더 set입니다.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: 지원하는 각 공개 헤더는 strict C++98 번역 단위의 첫 library include로 단독 컴파일되어야 합니다.
- 재현하는 실패 또는 경계: header가 자신이 사용하는 standard/프로젝트 declaration을 직접 include하지 않고 다른 header의 선행 include에 기대는 경우입니다.
- 테스트 방식: independent translation-unit 컴파일 probes입니다. 각 원본을 `-c`하여 별도 object로 만듭니다.
- 통과하는 production 코드 경로: 각 header의 include guard/include graph와 representative template instantiation입니다. 알고리즘은 `equal`/lexicographic comparison, 반복자는 reverse dereference, vector/map/stack은 기본 작업을 instantiate합니다.
- 이 테스트가 증명하는 것: 각 probe가 요구하는 declarations/templates가 include order 도움 없이 컴파일됨을 증명합니다.
- 이 테스트가 증명하지 않는 것: object들을 하나로 link하지 않으므로 duplicate external definitions/ODR failures, 런타임 동작, 모든 template argument 조합은 증명하지 않습니다.
- 성격: 공개 헤더 self-containment를 직접 고정하는 결정적 컴파일 회귀입니다.
- 후속 변경에서 막아야 하는 회귀: component header의 required include를 제거하고 aggregate 또는 테스트 include order에 기대는 회귀입니다.
- 실행 증거: probe sources와 Make rules를 코드로 검사했으며 `make headers`는 실행하지 않았습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: aggregate smoke에서 component별 컴파일 isolation으로 acceptance 경계를 확대합니다.
- 핵심 코드와 상태 변화: 공개 헤더마다 최소 TU가 생기고 outputs는 `build/headers`에 분리됩니다.
- 다음 커밋에 넘기는 전제: 컴파일 self-containment를 넘어 독립 TUs를 실제 executable로 link하고 실행해야 합니다.

### 5. test(consumer): 다중 번역 단위 공개 헤더 사용 검증

- SHA: `072c49832ddc`
- 중요도: B
- 태그: TEST, PUBLIC_API, INTEGRATION
- 원문에서 정한 역할: 독립 vector/map 사용 코드를 여러 번역 단위에서 링크합니다.
<!-- 원문 요약: Adds a linked multi-translation-unit consumer for vector and map and composes a complete `check` target. -->
<!-- 원문 판단 근거: The change verifies real header-only consumption and catches ODR or linkage failures that single-file tests cannot. It is strong practical integration work but does not alter container semantics. -->

#### 해당 SHA에서 확인할 실제 코드

- vector consumer TU, map consumer TU, shared declaration header, caller/main TU의 include와 symbol 관계를 그립니다.
- 각 TU가 독립 컴파일된 뒤 하나의 executable로 link되는 대상을 확인합니다.
- single-file 테스트가 잡지 못하는 header-only ODR/링크 실패를 이 구조가 어떻게 노출하는지 기록합니다.
- new `check` 대상이 behavioral 테스트, header compilation, linked consumer를 어떤 순서/의존성으로 묶는지 확인합니다.
- 확인한 파일/심볼: `tests/consumer/consumer_api.hpp`, `vector_consumer.cpp`, `map_consumer.cpp`, `main.cpp`; Makefile의 `CONSUMER_OBJECTS`, `CONSUMER_BIN`, `consumer`, `check` targets입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: independent 컴파일 probes `d938c0079994`입니다.

#### 테스트·검증 기록

- 대상 실제 코드의 불변식: header-only definitions와 shared transitive utilities가 여러 independently compiled TUs에서 사용돼도 하나의 executable로 link되고 예상 동작을 해야 합니다.
- 재현하는 실패 또는 경계: non-inline external definition duplication, missing 링크, include-order/TU-local dependency, consumer-facing 런타임 통합 오류입니다.
- 테스트 방식: multi-translation-unit 컴파일 + link + executable 통합 테스트입니다.
- 통과하는 production 코드 경로: vector TU는 1..5 push 후 index 2에 두 개의 7을 insert해 합계 29를 반환합니다. map TU는 3/1/2를 insert하고 key 1을 erase한 뒤 key+value 합계 55를 반환합니다. main TU가 두 external functions를 호출해 값을 검사합니다.
- 이 테스트가 증명하는 것: 이 TU 구성에서 overlapping header-only dependencies가 link되며 vector 삽입과 map 삽입/erase/iteration의 representative consumer flow가 실행 가능한 구조임을 증명합니다.
- 이 테스트가 증명하지 않는 것: 모든 공개 헤더가 동시에 여러 TUs에서 모든 instantiation으로 사용되는 경우, shared-library ABI, 동적 loading, 모든 ODR-sensitive 조합은 다루지 않습니다.
- 성격: 실제 consumer 도형을 재현한 결정적 link/런타임 통합입니다.
- 후속 변경에서 막아야 하는 회귀: header에 non-inline external symbol을 추가해 duplicate definition을 만들거나, component header가 다른 TU의 include side effect에 기대거나, 공개 templates의 link visibility를 잃는 회귀입니다.
- 실행 증거: 원본, link rule, expected 29/55 계산을 검사했으며 consumer executable은 실행하지 않았습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: 컴파일-only acceptance를 linked consumer acceptance로 전환합니다.
- 핵심 코드와 상태 변화: 세 `.cpp` objects가 별도 컴파일되어 `build/consumer_test`로 link되고 `consumer` 대상이 실행합니다. `check`는 `test`, `headers`, `consumer`를 모두 prerequisite로 둡니다.
- 다음 커밋에 넘기는 전제: 동일한 complete check surface를 instrumented 빌드에서도 실행할 수 있어야 합니다.

### 6. build(makefile): 격리된 sanitizer 검사 대상 추가

- SHA: `1be03ae8daef`
- 중요도: B
- 태그: PUBLIC_API, PRACTICAL, RISK
- 원문에서 정한 역할: 전체 인수 검증을 격리된 ASan/UBSan 계측으로 실행합니다.
<!-- 원문 요약: Adds an isolated ASan/UBSan build of the complete check suite. -->
<!-- 원문 판단 근거: Separate instrumented output prevents flag mixing and broadens detection of lifetime and pointer errors. This strengthens verification infrastructure but represents standard tooling rather than a project-specific core decision. -->

#### 해당 SHA에서 확인할 실제 코드

- ASan/UBSan flags, debug info, 프레임-포인터 options가 sanitizer 빌드에만 적용되는 Makefile 경로를 확인합니다.
- recursive make가 instrumented objects를 별도 빌드 directory에 두어 normal objects와 섞이지 않게 하는지 확인합니다.
- sanitizer 대상이 complete `check` surface를 다시 실행하는지 확인하고, value parity 테스트만으로 보이지 않는 포인터/수명 오류 종류를 원본 범위 안에서 기록합니다.
- 확인한 파일/심볼: Makefile의 `SANITIZER_FLAGS`, phony `sanitize`, recursive `$(MAKE) BUILD_DIR=$(BUILD_DIR)/sanitize CXXFLAGS="..." check`입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: complete `check` 대상 도입 `072c49832ddc`입니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: normal 빌드·테스트/header/consumer acceptance만 있었고 instrumented 출력을 분리하는 경로가 없었습니다.
- 해결하려던 문제: normal flags로 결과가 맞아도 exercised path의 out-of-bounds, use-after-free, invalid 수명, undefined arithmetic가 관찰되지 않을 수 있습니다. 같은 object directory를 flag만 바꿔 재사용하면 stale/mixed objects가 생길 수 있습니다.
- 선택한 결정: `-O1 -g -fno-omit-frame-pointer -fsanitize=address,undefined`를 recursive make의 `CXXFLAGS`에 추가하고 `BUILD_DIR=build/sanitize`로 complete `check`를 다시 빌드·실행합니다.
- 새로 생긴 책임 경계 또는 상태 변화: normal artifacts와 instrumented artifacts가 경로로 분리되며 sanitizer 대상은 별도 acceptance mode를 담당합니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: 같은 behavioral/header/multi-TU surface에 동적 memory/UB diagnostics를 추가합니다.
- 핵심 코드와 상태 변화: 빌드 directory와 flags만 override하고 대상 graph는 `check`를 재사용합니다.
- 다음 커밋에 넘기는 전제: local normal/sanitizer commands를 여러 compiler/platform에서 자동 반복해야 합니다.

### 7. ci: compiler 행렬과 sanitizer 검사 구성

- SHA: `228f457988be`
- 중요도: B
- 태그: CXX98, PUBLIC_API, PRACTICAL
- 원문에서 정한 역할: 컴파일러·플랫폼·sanitizer 검사를 CI로 자동화합니다.
<!-- 원문 요약: Runs the complete checks on GCC and Clang across Linux/macOS plus a Linux sanitizer job. -->
<!-- 원문 판단 근거: The workflow makes portability and memory-safety verification repeatable at branch level. It is valuable release engineering, but it does not itself establish a container invariant. -->

#### 해당 SHA에서 확인할 실제 코드

- CI main matrix의 GCC/Clang Linux 및 Clang macOS 조합과 각 job이 실행하는 complete check 대상을 확인합니다.
- fail-fast disabled 설정과 platform-specific result가 동시에 보존되는지 확인합니다.
- separate Linux Clang sanitizer job의 leak detection/immediate 실패/UB stack trace 설정과 sanitizer 대상 호출을 확인합니다.
- workflow 권한이 read-only로 충분한 acceptance 작업만 수행하는지 확인합니다.
- 확인한 파일/심볼: `.github/workflows/ci.yml`의 `compiler-matrix`, three include entries, `make CXX=${{ matrix.compiler }} check`, `sanitizers` job, ASAN/UBSAN 환경, `make CXX=clang++ sanitize`, `permissions: contents: read`입니다.
- 필요한 경우 비교할 직전 관련 SHA/parent: local sanitizer 대상 `1be03ae8daef`입니다.

#### 설계·상태 변화 기록

- 이 커밋 직전 상태: acceptance commands는 Makefile에 있었지만 실행 여부가 개발자의 local 환경에 의존했습니다.
- 해결하려던 문제: 하나의 compiler/OS에서만 통과하는 C++98 extension, warning 차이, platform-sensitive header/빌드 issue와 sanitizer omission을 반복적으로 잡기 어렵습니다.
- 선택한 결정: push와 pull 요청에서 Ubuntu/g++, Ubuntu/clang++, macOS/clang++의 complete `check`를 fail-fast 없이 수행하고, 별도 Ubuntu/clang++ sanitizer job을 둡니다.
- 새로 생긴 책임 경계 또는 상태 변화: repository workflow가 cross-toolchain normal acceptance와 Linux instrumented acceptance 실행 책임을 가집니다. content write permission은 부여하지 않습니다.

#### 중요도 B 확인

- 개발 흐름에서 맡는 구현 역할: local acceptance graph를 compiler/platform matrix와 sanitizer automation으로 승격합니다.
- 핵심 코드와 상태 변화: workflow jobs가 기존 Make targets를 그대로 호출하므로 local/CI acceptance 명령이 분기되지 않습니다.
- 다음 커밋에 넘기는 전제: 이 개발 흐름의 마지막 커밋입니다. 이후 공개-surface 변경은 같은 `check`/`sanitize`/CI graph에 편입되어야 합니다.

## 6. 불변식 변화 기록

### 원문에서 정한 관련 불변식

- 지원 대상인 모든 공개 헤더는 엄격한 C++98 모드에서 단독으로 컴파일할 수 있어야 하며, 헤더 전용 구현은 여러 번역 단위에서 포함해도 링크 오류나 ODR 위반이 없어야 합니다.

### 시간에 따른 변화 기록

| Invariant | 처음 도입된 커밋 | 부족함이 드러난 커밋/상태 | 강화·복구한 fix | 고정한 테스트/perf | 직접 확인한 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| 지원 대상인 모든 공개 헤더는 엄격한 C++98 모드에서 단독으로 컴파일할 수 있어야 하며, 헤더 전용 구현은 여러 번역 단위에서 포함해도 링크 오류나 ODR 위반이 없어야 합니다. | aggregate entry는 `80e169e83212`; map 포함은 `112af1753538` | one-TU aggregate smoke `3c64a69dd252`는 component self-containment와 link/ODR를 보지 못함 | production fix가 아니라 acceptance를 `d938c0079994`와 `072c49832ddc`로 확대 | `d938c0079994`, `072c49832ddc`, 이후 `1be03ae8daef`, `228f457988be` | first-include 컴파일 probes, separate consumer objects/link/run, isolated sanitizer 대상, CI matrix |

## 7. 문제 → 수정 → 검증 연결

| 기존 상태/production change | fix 또는 verification | 원문에서 확정된 연결 관점 | 실제 실패/root cause | 테스트가 통과하는 실제 실행 경로 |
| --- | --- | --- | --- | --- |
| `80e169e83212` | `3c64a69dd252` | aggregate sequential surface smoke | bundle include가 component symbol/dependency를 누락할 수 있음 | aggregate include → unchanged utility/vector/stack 테스트 |
| `112af1753538` | `d938c0079994` | aggregate expansion 이후 component self-containment 검사 | monolithic include order가 missing direct include를 가릴 수 있음 | each header first include → representative instantiation → object 컴파일 |
| `d938c0079994` | `072c49832ddc` | 컴파일 isolation에서 multi-TU link acceptance로 확대 | 컴파일-only objects는 duplicate definitions/link visibility를 보지 못함 | vector/map objects + main → link → expected 29/55 run |
| `072c49832ddc` | `1be03ae8daef` | complete check surface를 sanitizer instrumentation으로 확대 | correct 출력이 invalid memory/UB를 숨길 수 있고 flag-mixed artifacts가 진단을 왜곡할 수 있음 | isolated 빌드/sanitize → recursive complete `check` |
| `1be03ae8daef` | `228f457988be` | local acceptance를 compiler/platform/sanitizer CI로 자동화 | local 단일 toolchain 실행의 비반복성/portability blind spot | Linux GCC/Clang, macOS Clang `check`; Linux Clang `sanitize` |

## 8. 소유권·상태·담당 변화

| 시점 | Owner / 상태 / responsibility | 변경 전 | 변경 후 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| `80e169e83212` | 통합 헤더를 도입합니다. | consumer가 component 조합 소유 | repository가 supported bundle include list 제공 | `ft_containers.hpp` |
| `3c64a69dd252` | 현재 순차 컨테이너 API가 통합 헤더를 통해 컴파일되는지 확인합니다. | bundle은 선언만 존재 | existing 테스트의 sole library entry로 사용 | 테스트 include replacement |
| `112af1753538` | 통합 공개 진입점에 map을 추가합니다. | aggregate에 associative surface 없음 | map와 relations/swap도 bundle에서 노출 | aggregate/map header diff |
| `d938c0079994` | 각 공개 헤더를 첫 include로 독립 컴파일합니다. | include correctness가 monolithic TU에 묶임 | 각 header가 자기 dependency 책임을 독립 probe로 부담 | `tests/headers`, `headers` 대상 |
| `072c49832ddc` | 독립 vector/map 사용 코드를 여러 번역 단위에서 링크합니다. | 컴파일-only verification | consumer objects/link/런타임이 ODR/link 책임 검증 | `tests/consumer`, `consumer`, `check` |
| `1be03ae8daef` | 전체 인수 검증을 격리된 ASan/UBSan 계측으로 실행합니다. | normal artifacts만 존재 | separate instrumented artifact 소유자/path | recursive sanitize Make |
| `228f457988be` | 컴파일러·플랫폼·sanitizer 검사를 CI로 자동화합니다. | developer가 local 실행 책임 | workflow가 push/PR matrix 실행 책임 | CI jobs/permissions |

## 9. 개발 흐름의 최종 상태

- 최종적으로 성립한 표현/상태: component 공개 headers와 aggregate `ft_containers.hpp`가 공존합니다. Makefile acceptance는 behavioral 테스트, independent-header objects, multi-TU linked consumer를 `check`로 묶고, 동일 graph를 separate sanitizer 빌드에서 재실행합니다. CI는 normal compiler/platform matrix와 sanitizer job을 호출합니다.
- 최종적으로 보장하는 불변식: repository가 정의한 probes 범위에서 각 공개 헤더는 strict C++98로 first-include 컴파일되고, header-only 구현은 representative vector/map consumer TUs를 함께 link할 수 있도록 구성됩니다.
- 남아 있는 precondition 또는 보장하지 않는 범위: probes가 instantiate하지 않은 모든 template/type 조합, ABI/shared-library compatibility, unexercised 런타임 path의 memory safety를 전부 증명하지 않습니다. CI workflow 정의는 확인했지만이 작업에서 run 결과를 검증하지 않았습니다.
- 최종 verification 근거: aggregate include diff, eight independent header probes, separate consumer 컴파일/link/run rules와 expected 29/55, isolated ASan/UBSan recursive 대상, three normal matrix entries와 one sanitizer job을 코드로 확인했습니다. local 체크아웃 제한 때문에 명령은 실행하지 않았습니다.
- 이 상태에 도달하기 위해 필요했던 핵심 turning point 커밋: self-containment를 분리한 `d938c0079994`, real linked consumer를 만든 `072c49832ddc`, complete acceptance를 자동화한 `228f457988be`입니다.

## 10. 최종 설계와 실행 흐름

아래 단계명은 원문이 정의한 개발 흐름 progression을 따라가는 탐색 순서입니다. 실제 함수·상태·분기·코드 조각은 해당 SHA에서 직접 채웁니다.

| 단계 | 관련 커밋 | 실제 코드 위치 | 입력/기존 상태 | 핵심 transition | 실패/정리 | 다음 단계에 남기는 불변식 |
| --- | --- | --- | --- | --- | --- | --- |
| Aggregate header | `80e169e83212` | `include/ft_containers.hpp` | separate utility/vector/stack headers | supported include list를 bundle로 구성 | preprocessor/컴파일 실패로 누락 노출 | one 공개 entry exists |
| Aggregate smoke 테스트 | `3c64a69dd252` | `tests/test_containers.cpp` | existing 테스트 + many direct includes | sole aggregate include로 교체 | 컴파일/테스트 nonzero | current sequential bundle usable in one TU |
| Map 공개 통합 | `112af1753538` | aggregate/map headers | aggregate without map | map include + 공개 comparisons/swap 노출 | 컴파일 surface | bundle includes associative surface |
| Independent component compilation | `d938c0079994` | `tests/headers`, Make `headers` | possible transitive include dependence | each header first include, instantiate, `-c` | failing object stops 대상 | component self-containment 근거 |
| Multi-TU linked consumer | `072c49832ddc` | `tests/consumer`, Make `consumer/check` | isolated objects only | separate vector/map/main objects link and run | 컴파일/link/nonzero 런타임 | representative ODR/link consumer 근거 |
| Isolated sanitizers | `1be03ae8daef` | Make `sanitize` | normal complete check | recursive check with sanitizer flags/new 빌드 dir | sanitizer abort/nonzero propagates | instrumented acceptance separated |
| Cross-compiler/platform CI | `228f457988be` | `.github/workflows/ci.yml` | local targets | matrix `check` + separate `sanitize` automation | jobs report independently, fail-fast false | repeatable 브랜치-level acceptance configuration |

## 11. 학습 완료 자가 점검

- [x] 커밋 목록의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 커밋 기록에 최종 HEAD가 아니라 해당 SHA의 실제 코드 근거가 있습니다.
- [x] S/A 커밋은 결정, 실패 경계, 소유권/상태 변화를 설명할 수 있습니다. 이 개발 흐름에는 S/A 커밋이 없어 해당 항목은 비적용임을 확인했습니다.
- [x] 테스트·성능 관련 커밋은 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] Fix가 있는 경우 기존 가정 → 실패/risk → root cause → 수정 → 회귀 연결을 설명할 수 있습니다. 이 개발 흐름은 production fix보다 acceptance 확대 순서가 중심입니다.
- [x] Invariant ledger가 커밋 history에 따라 어떻게 변했는지 설명할 수 있습니다.
- [x] 개발 흐름 최종 상태와 architecture/execution flow를 실제 코드 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: 06-header-only-public-surface-automated-acceptance.md =====

===== BEGIN FILE: README.md =====
# ft_container 개발 흐름 학습 안내

## 목적

이 디렉터리는 확정된 `commit-importance.md`와 `commit-bodies.md`만을 기준으로, 실제 커밋 history와 해당 SHA의 코드를 직접 읽으며 `ft_container`의 개발 과정을 복원하기 위한 학습 골격입니다.

완성형 해설서가 아닙니다. 문서에 미리 적힌 내용은 원본에서 이미 확정된 개발 흐름 구조, 커밋 메타데이터, 역할, 중요도, 태그, 불변식과 engineering difficulty입니다. 실제 구현 해석, 함수별 추적, 소유권/수명 확인, 실패 처리, 테스트 결과, 최종 설명은 학습자가 채웁니다.

## 권장 학습 순서

개발 흐름의 원문 순서를 그대로 따릅니다.

1. `01-cxx98-generic-interface-foundation.md`
2. `02-vector-ownership-aliasing-exception-safe-mutation.md`
3. `03-map-unbalanced-bst-to-verified-red-black-core.md`
4. `04-stable-map-iterators-through-structural-mutation.md`
5. `05-stateful-allocators-map-failure-transactions.md`
6. `06-header-only-public-surface-automated-acceptance.md`

동일한 커밋이 여러 개발 흐름에 나타나는 경우 제거하지 않고 각 개발 흐름의 관점으로 다시 확인합니다. 이 세트에서는 원문이 정의한 개발 흐름 membership과 순서를 그대로 유지합니다.

## 개발 흐름 문서 사용법

- 먼저 `Thread 목표`, `핵심 질문`, `커밋 목록`을 읽어 학습 범위를 고정합니다.
- 각 커밋에서는 반드시 표시된 SHA 시점의 diff와 코드를 확인합니다.
- `원문에서 정한 역할`, `원문 요약`, `원문 판단 근거`는 재평가하지 않습니다.
- 코드 확인 지시는 답이 아니라 탐색 범위입니다. 실제 파일, 함수, 상태 필드, 브랜치, 정리 순서는 직접 기록합니다.
- fix는 기존 가정 → 실패/risk → root cause → 수정된 결정/불변식 → 실제 수정 코드 → 회귀 테스트 순으로 복원합니다.
- 테스트/perf 커밋은 대상 실제 코드의 불변식, 실패/경계, 검증 방식, 실제 실행 경로, 증명 범위와 비증명 범위를 분리합니다.
- 개발 흐름 마지막에는 커밋별 기록을 다시 묶어 불변식 ledger와 최종 execution/architecture flow를 직접 완성합니다.

## 해당 SHA 코드 확인 원칙

- 최종 HEAD의 코드를 과거 커밋 설명에 소급해서 사용하지 않습니다.
- `git show <sha> --stat`으로 변경 범위를 먼저 확인하고, `git show <sha> -- <path>`로 커밋 diff를 봅니다. 해당 SHA의 파일 전체 상태는 `git show <sha>:<path>`로 확인합니다.
- 수정 전후 차이가 핵심이면 해당 커밋의 parent 또는 문서가 지정한 직전 관련 SHA와 비교합니다.
- later 커밋에서 바뀐 helper 이름이나 최종 표현을 earlier 커밋에 끌어오지 않습니다.
- 실제 코드 근거를 적을 때 SHA, 파일 경로, 함수/형식/테스트 이름을 함께 기록합니다.

## 최종 HEAD 소급 사용 금지

이 골격의 목적은 완성된 코드를 설명하는 것이 아니라 설계 → 구현 → 실패 → 수정 → 검증의 발전 과정을 복원하는 것입니다. earlier 커밋의 부족한 상태도 그대로 읽어야 하며, later fix의 결론으로 earlier 코드를 정당화하거나 재해석하지 않습니다.

## S/A/B/C별 학습 깊이

- S: 핵심 architecture/불변식입니다. Problem, 기존 상태, 실패 가능성, 결정, 핵심 코드, 소유권/lifecycle/상태 변화, 후속 fix/테스트까지 추적합니다.
- A: 주요 구성 요소, 경계, 실패 처리, 통합 point입니다. 핵심 코드와 설계 판단, 관련 회귀를 확인합니다.
- B: 개발 흐름에서 맡는 구현 역할과 필요한 코드/상태 변화를 확인합니다.
- C: 개발 흐름 이해에 필요한 맥락만 확인합니다. S/A와 동일한 깊이의 분석을 만들지 않습니다.

## 실제 코드 삽입 기준

- 전체 파일을 복사하지 않고 결정을 설명하는 최소 코드만 삽입합니다.
- 상태 필드, 핵심 분기, 소유권 transfer, construct/destroy, event가 아닌 컨테이너 상태 변경, error/실패 브랜치, 정리, 테스트 injection처럼 판단에 필요한 부분을 우선합니다.
- 코드 조각마다 `SHA / file / symbol`을 식별할 수 있게 기록합니다.
- 변경 전/후가 핵심이면 두 시점의 최소 대응 코드만 나란히 기록하고 차이를 직접 설명합니다.
- 원본에 없는 구현 사실을 추정해서 채우지 않습니다.

## 테스트 커밋 학습 방법

- 먼저이 테스트가 보호하는 실제 코드의 불변식를 한 문장으로 적습니다.
- 어떤 실패/경계를 주입하거나 재현하는지 확인합니다.
- differential, 실패 주입, white-box, 결정적 randomized, structural bound, 통합 컴파일/link 중 실제 검증 방식을 코드로 확인합니다.
- 테스트 테스트 준비 코드에서 끝내지 말고 실제 실제 실행 경로까지 연결합니다.
- 성공 검사문이 증명하는 것과 증명하지 않는 것을 분리합니다.
- broad 통합 테스트인지 특정 회귀를 고정하는 결정적 테스트인지 근거를 적습니다.
- 후속 변경이 어떤 회귀를 만들면이 테스트가 실패해야 하는지 적습니다.

## 문서 완료 기준

- 모든 개발 흐름 커밋을 원문 순서대로 해당 SHA에서 확인했습니다.
- S/A 커밋의 핵심 결정과 실패/소유권/상태 변화를 실제 코드 근거로 설명할 수 있습니다.
- fix와 관련 회귀 테스트의 연결을 복원했습니다.
- 불변식 ledger에서 도입 → 부족함 노출 → 보강/fix → 회귀 고정의 변화를 설명할 수 있습니다.
- map과 vector의 핵심 수명/소유권/반복자/tree 불변식을 원본의 표현과 충돌 없이 설명할 수 있습니다.
- 테스트가 증명하는 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- 최종 HEAD를 earlier 커밋 설명에 소급 사용한 부분이 없습니다.
- 최종 architecture 또는 execution flow를 커밋 history 근거로 자기 말로 설명할 수 있습니다.
===== END FILE: README.md =====

