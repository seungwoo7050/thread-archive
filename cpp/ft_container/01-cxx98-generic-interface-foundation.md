# Thread 01 — C++98 제네릭 인터페이스 기반

> 대상: `seungwoo7050/42-archive`의 `cpp/ft_container` 브랜치
> 검토 방식: 아래 커밋의 정확한 SHA에서 diff와 당시 source를 확인했습니다. 이 환경에서는 build와 test를 실행하지 않았습니다.

## 개요

이 Thread는 컨테이너 자체보다 먼저 필요한 공용 언어를 만든 과정입니다. C++98에는 이후 표준에서 익숙해진 type trait와 iterator utility가 없으므로, 각 컨테이너가 제각각 구현하지 않도록 네 종류의 공용 요소를 분리합니다.

- `enable_if`와 `is_integral`: 정수 개수 인자와 iterator 범위 인자를 compile time에 구분합니다.
- `pair`: map의 `value_type`과 `insert` 결과처럼 두 값을 함께 표현합니다.
- `equal`, `lexicographical_compare`: 컨테이너 관계 연산의 범위 비교를 공용화합니다.
- `iterator_traits`, `reverse_iterator`: pointer 기반 vector와 node 기반 map이 같은 iterator 어댑터를 사용하게 합니다.

개별 구현은 작지만 의존 관계는 분명합니다. 컨테이너가 public overload와 관계 연산, 역방향 순회를 제공하려면 먼저 이 vocabulary가 일관된 형식으로 존재해야 합니다. 마지막 Makefile은 이를 `-std=c++98 -Wall -Wextra -Werror`로 반복 검사할 수 있는 최소 실행 경계로 고정합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ecc0668d6d9c` | `feat(type-traits): CXX98 타입 선택 도구 구현` | A | `CXX98, ARCH, LEARNING` | Establishes substitution-based overload selection and compile-time type classification. |
| 2 | `1c8692d14118` | `feat(pair): 값 쌍과 관계 연산 구현` | B | `CXX98` | Supplies the pair value and return type required by associative-container APIs. |
| 3 | `07cf5893a53c` | `feat(algorithm): 공용 범위 비교 알고리즘 구현` | B | `CXX98, PRACTICAL` | Centralizes equality and lexicographical comparison for container relations. |
| 4 | `e3462cec55a1` | `feat(iterator): iterator 기본 형식과 traits 정의` | B | `CXX98, ITERATOR` | Defines iterator type metadata and pointer traits. |
| 5 | `7a8e3d32bb4d` | `feat(iterator): 역방향 반복자의 양방향 동작 구현` | B | `ITERATOR` | Adds the bidirectional reverse-iterator base convention. |
| 6 | `ae50e9038643` | `feat(iterator): 역방향 반복자의 임의 접근 연산 완성` | B | `ITERATOR` | Extends reverse iteration to the random-access operations needed by vector. |
| 7 | `455098520e83` | `test(utils): 공용 타입·값·범위·반복자 도구 검증` | B | `TEST, CXX98` | Verifies the utility layer before containers depend on it. |
| 8 | `f36ec7e7e047` | `build(makefile): CXX98 검사 빌드 구성` | B | `CXX98, PRACTICAL` | Makes strict warning-enabled C++98 compilation the repeatable baseline. |

## `ecc0668d6d9c` — 대입 가능한 overload만 남기는 compile-time 도구

**중요도** `A` · **태그** `CXX98, ARCH, LEARNING`

이 커밋은 `include/ft_type_traits.hpp`를 추가하고 `enable_if`, `integral_constant`, `true_type`, `false_type`, `is_integral`을 정의합니다.

```cpp
template <bool Cond, class T = void>
struct enable_if
{
};

template <class T>
struct enable_if<true, T>
{
    typedef T type;
};

template <class T, T V>
struct integral_constant
{
    static const T value = V;
    typedef T value_type;
    typedef integral_constant<T, V> type;

    operator value_type() const
    {
        return value;
    }
};
```

핵심은 `enable_if<false, T>`에 `type`이 없다는 점입니다. 어떤 함수 template의 반환형이나 dummy parameter가 `enable_if<조건, T>::type`을 요구할 때 조건이 거짓이면 해당 선언은 유효한 후보가 되지 않습니다. runtime에서 “정수인가 iterator인가”를 검사하는 방식이 아니라, template substitution 단계에서 사용할 수 없는 overload가 사라집니다.

`is_integral`은 primary template을 `false_type`으로 두고 C++98 정수 형식만 명시적으로 `true_type`으로 specialization합니다.

```cpp
template <class T>
struct is_integral : false_type
{
};

template <>
struct is_integral<int> : true_type
{
};

/* bool, char 계열, wchar_t, short, unsigned short,
   unsigned int, long, unsigned long도 같은 방식으로 specialization */
```

이 SHA에서는 아직 vector의 range/fill overload가 이 trait를 소비하지 않습니다. 따라서 확정할 수 있는 것은 **공용 compile-time 분류 도구가 생겼다**는 데까지입니다. 이후 컨테이너 사용처를 이 커밋에 소급해 설명할 수는 없습니다.

또한 구현은 나열된 원형 정수 형식만 다룹니다. `const int`처럼 cv-qualified 형식을 제거하는 별도 trait는 이 diff에 없습니다. 이 Thread의 기반은 최소 C++98 dispatch vocabulary이며, 완전한 현대식 type-traits 재현이 아닙니다.

## `1c8692d14118` — map 값과 반환 형식을 위한 `pair`

**중요도** `B` · **태그** `CXX98`

`include/ft_pair.hpp`에 두 값을 직접 보관하는 `ft::pair`와 관계 연산, `make_pair`가 추가됩니다.

```cpp
template <class T1, class T2>
struct pair
{
    typedef T1 first_type;
    typedef T2 second_type;

    first_type first;
    second_type second;

    pair() : first(), second() {}

    pair(const first_type& a, const second_type& b)
        : first(a), second(b) {}

    template <class U1, class U2>
    pair(const pair<U1, U2>& other)
        : first(other.first), second(other.second) {}
};
```

사전식 순서는 첫 번째 값이 다를 때 `first`로 결정하고, 서로 어느 쪽도 작지 않을 때만 `second`를 비교합니다.

```cpp
return lhs.first < rhs.first
    || (!(rhs.first < lhs.first) && lhs.second < rhs.second);
```

이 결정은 `map<Key, T>`의 원소인 `pair<const Key, T>`뿐 아니라 `insert`가 반환하는 `pair<iterator, bool>`에도 같은 표현을 사용할 수 있게 합니다. 이 커밋은 자원 수명이나 컨테이너 상태를 다루지 않습니다. 공용 값 형식과 그 비교 규칙을 고정하는 지원 작업입니다.

## `07cf5893a53c` — 관계 연산이 공유하는 범위 비교

**중요도** `B` · **태그** `CXX98, PRACTICAL`

`include/ft_algorithm.hpp`는 `equal`과 `lexicographical_compare`를 각각 기본 비교와 predicate/comparator overload로 제공합니다.

```cpp
template <class InputIt1, class InputIt2>
bool equal(InputIt1 first1, InputIt1 last1, InputIt2 first2)
{
    for (; first1 != last1; ++first1, ++first2)
    {
        if (!(*first1 == *first2))
            return false;
    }
    return true;
}
```

```cpp
template <class InputIt1, class InputIt2>
bool lexicographical_compare(InputIt1 first1, InputIt1 last1,
    InputIt2 first2, InputIt2 last2)
{
    for (; first1 != last1 && first2 != last2; ++first1, ++first2)
    {
        if (*first1 < *first2)
            return true;
        if (*first2 < *first1)
            return false;
    }
    return first1 == last1 && first2 != last2;
}
```

필요한 iterator 능력은 역참조, 증가, 동등 비교뿐입니다. 즉 random-access 연산을 요구하지 않으므로 pointer와 tree iterator 모두 같은 구현을 사용할 수 있습니다. `lexicographical_compare`의 마지막 반환은 공통 prefix가 같을 때 더 짧은 범위를 작은 것으로 판정합니다.

이 단계에서 알고리즘은 독립 utility일 뿐이며, 실제 vector/map 관계 연산은 후속 커밋에서 이를 소비합니다.

## `e3462cec55a1` — iterator 형식을 읽는 공용 metadata

**중요도** `B` · **태그** `CXX98, ITERATOR`

`include/ft_iterator.hpp`에 iterator 기본 형식과 `iterator_traits`가 추가됩니다.

```cpp
template <class Category, class T, class Distance = std::ptrdiff_t,
    class Pointer = T*, class Reference = T&>
struct iterator
{
    typedef Category iterator_category;
    typedef T value_type;
    typedef Distance difference_type;
    typedef Pointer pointer;
    typedef Reference reference;
};

template <class Iterator>
struct iterator_traits
{
    typedef typename Iterator::difference_type difference_type;
    typedef typename Iterator::value_type value_type;
    typedef typename Iterator::pointer pointer;
    typedef typename Iterator::reference reference;
    typedef typename Iterator::iterator_category iterator_category;
};
```

일반 iterator는 nested typedef를 통해 형식을 노출하고, raw pointer는 partial specialization으로 random-access iterator처럼 취급됩니다.

```cpp
template <class T>
struct iterator_traits<T*>
{
    typedef std::ptrdiff_t difference_type;
    typedef T value_type;
    typedef T* pointer;
    typedef T& reference;
    typedef std::random_access_iterator_tag iterator_category;
};
```

`const T*` specialization도 별도로 존재합니다. 이 공용 metadata 덕분에 이후 `reverse_iterator<Iterator>`가 구체 iterator가 pointer인지 class인지 알 필요 없이 `difference_type`, `reference`, `pointer`를 얻습니다.

## `7a8e3d32bb4d` — 역방향 반복자의 base convention

**중요도** `B` · **태그** `ITERATOR`

첫 `reverse_iterator`는 양방향 순회에 필요한 동작을 만듭니다. 내부에는 정방향 iterator `_current`를 저장하고, 실제 역참조 대상은 `base()` 바로 앞 원소입니다.

```cpp
reference operator*() const
{
    iterator_type tmp = _current;
    return *--tmp;
}

reverse_iterator& operator++()
{
    --_current;
    return *this;
}

reverse_iterator& operator--()
{
    ++_current;
    return *this;
}
```

이 convention은 `rbegin()`을 `reverse_iterator(end())`로, `rend()`를 `reverse_iterator(begin())`로 만들 수 있게 합니다.

```text
정방향:  [a] [b] [c]  end
역방향:  rend          [c] [b] [a]  rbegin의 base는 end
```

역방향에서 한 칸 앞으로 가는 `++`가 정방향 base를 한 칸 뒤로 보내는 `--`라는 점이 핵심입니다. 이 커밋은 `+`, `-`, `[]`, 순서 비교 같은 임의 접근 연산은 아직 제공하지 않습니다.

## `ae50e9038643` — vector에 필요한 임의 접근 역방향 연산

**중요도** `B` · **태그** `ITERATOR`

앞선 base convention을 유지하면서 random-access 연산을 추가합니다.

```cpp
reverse_iterator operator+(difference_type n) const
{
    return reverse_iterator(_current - n);
}

reverse_iterator operator-(difference_type n) const
{
    return reverse_iterator(_current + n);
}

difference_type operator-(const reverse_iterator& other) const
{
    return other.base() - base();
}
```

순서 비교도 base 순서의 반대입니다.

```cpp
return rhs.base() < lhs.base();  // reverse_iterator의 operator<
```

따라서 vector의 정방향 iterator가 pointer라면 동일 adapter가 산술과 `operator[]`까지 제공하고, map의 bidirectional iterator는 지원하지 않는 연산을 인스턴스화하지 않는 범위에서 같은 adapter를 사용할 수 있습니다. 이 커밋은 iterator category에 따라 member 자체를 제거하지는 않습니다. 실제 사용자가 underlying iterator가 지원하지 않는 산술을 호출하면 그 표현이 유효하지 않습니다.

## `455098520e83` — 컨테이너 도입 전 utility smoke test

**중요도** `B` · **태그** `TEST, CXX98`

`tests/test_containers.cpp`는 이 시점의 utility를 한 translation unit에서 확인합니다.

- `ft::pair` 생성, 비교, `make_pair`
- `is_integral<int>`와 비정수 형식의 분류
- 배열 범위에 대한 `equal`, `lexicographical_compare`
- raw pointer를 감싼 `reverse_iterator`
- `iterator_traits`의 pointer metadata

이 테스트는 공용 utility의 대표 동작을 묶어 확인하지만, 다음 항목은 증명하지 않습니다.

- `enable_if`를 실제 fill/range overload에 연결한 결과
- 각 header를 다른 include 없이 단독 compile할 수 있는지
- 여러 translation unit에서 header-only 정의가 중복 symbol을 만들지 않는지
- vector나 map의 수명·복잡도·예외 보장

그 범위는 이후 컨테이너 Thread와 공개 header acceptance Thread가 맡습니다.

## `f36ec7e7e047` — C++98를 반복 가능한 build 조건으로 고정

**중요도** `B` · **태그** `CXX98, PRACTICAL`

Makefile은 utility test를 다음 조건으로 compile합니다.

```make
CXX := c++
CXXFLAGS := -Wall -Wextra -Werror -std=c++98
CPPFLAGS := -Iinclude
```

`all`, `test`, `clean`, `fclean`, `re`를 제공하고 build 산출물을 별도 디렉터리에 둡니다. 의미 있는 변화는 “C++98를 의도한다”는 문서 수준의 주장 대신, C++98 문법과 warning-free compile을 매번 같은 명령으로 검사할 수 있게 했다는 점입니다.

다만 이 시점의 build는 단일 compiler와 단일 test source에 한정됩니다. header별 독립 compile, 다중 translation unit link, sanitizer, compiler/platform matrix는 Thread 06에서 별도로 확장됩니다.

## 최종 상태

이 Thread가 끝났을 때 공용 계층의 책임은 다음처럼 나뉩니다.

| 파일 | 맡은 책임 | 맡지 않는 책임 |
| --- | --- | --- |
| `ft_type_traits.hpp` | compile-time 조건부 overload와 정수 형식 분류 | runtime type 판정, 완전한 현대식 trait 집합 |
| `ft_pair.hpp` | 두 값의 저장·변환·관계 연산 | 컨테이너 node/storage 소유권 |
| `ft_algorithm.hpp` | 범위 동등·사전식 비교 | 정렬, mutation, iterator 생성 |
| `ft_iterator.hpp` | iterator metadata와 reverse adapter | underlying iterator의 유효성·수명 |
| `Makefile` | 엄격한 C++98 utility build/test 진입점 | cross-compiler·multi-TU·sanitizer acceptance |

의존 방향도 한쪽입니다.

```text
[type traits] ──> container overload 선택
[pair]        ──> map value / insert 결과
[algorithms]  ──> vector·map 관계 연산
[traits + reverse_iterator]
              ──> pointer-backed vector / node-backed map iterator
```

이 Thread는 vector의 raw storage 수명, map의 red-black tree, iterator 안정성, stateful allocator failure transaction을 설명하지 않습니다. 공용 vocabulary가 먼저 존재하도록 만든 개발 단위이며, 각 컨테이너가 이를 실제로 소비하면서 생긴 문제는 뒤의 독립 Thread에서 다룹니다.
