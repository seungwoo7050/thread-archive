# Thread 02 — Vector의 저장소 소유권, aliasing, 예외 안전한 변경

> 대상: `seungwoo7050/42-archive`의 `cpp/ft_container` 브랜치
> 검토 방식: 아래 커밋의 정확한 SHA에서 diff와 당시 source를 확인했습니다. 이 환경에서는 build와 test를 실행하지 않았습니다.

## 개요

`vector`의 핵심은 index 연산이 아니라 **하나의 allocation 안에서 raw slot과 생성 완료 객체를 구분하는 일**입니다.

```text
_data
  │
  ├── [0, _size)       : constructor가 끝난 live object
  └── [_size, _capacity): 메모리만 확보된 raw storage
```

이 구분이 무너지면 다음과 같은 서로 다른 문제가 같은 형태로 나타납니다.

- 아직 생성하지 않은 slot을 대입하거나 파괴합니다.
- 생성은 끝났는데 `_size`가 반영되지 않아 destructor가 객체를 놓칩니다.
- `_size`를 먼저 늘린 뒤 constructor가 던져 존재하지 않는 객체를 live하다고 기록합니다.
- 삽입 인자가 현재 vector 내부를 가리키는데, reserve나 이동이 먼저 실행되어 입력 자체가 사라집니다.
- 여유 용량이 있는 삽입에서 live object에는 assignment를, raw tail에는 construction을 해야 하는데 둘을 구분하지 않습니다.
- allocator 상한 근처의 용량 배수가 정수 overflow를 일으킵니다.
- 빈 vector의 null pointer에 `+ 0`이나 pointer subtraction을 적용합니다.

이 Thread는 초기 저장소 표현에서 시작해, 재할당·alias 보존·실패 transaction을 거쳐 fill/range 삽입을 두 개의 수명 경로로 다시 만드는 과정입니다. 마지막 테스트 수정은 이 프로젝트가 의도한 self-range 확장의 기대값을 다른 컨테이너의 겹치는 범위 동작에 맡기지 않고 독립 snapshot으로 정의합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `2eb9cb6c4273` | `feat(vector): allocator 기반 저장소 수명 관리` | S | `CORE, VECTOR, ALLOCATOR` | Establishes allocator-backed contiguous storage and the constructed-object ownership boundary. |
| 2 | `9db46550b13d` | `feat(vector): 용량 확장과 원소 재배치 구현` | A | `VECTOR, ALLOCATOR, CORE` | Adds replacement-block reallocation and geometric capacity growth. |
| 3 | `6f3cbf4794c9` | `fix(vector): 용량 계산을 allocator 상한에서 포화` | A | `VECTOR, ALLOCATOR, EDGE` | Corrects growth arithmetic at the allocator limit. |
| 4 | `bdc4c3123bc9` | `fix(vector): 자기 범위 assign과 insert 입력 보존` | A | `VECTOR, EDGE, DEBUG` | Snapshots self-range input before mutation invalidates it. |
| 5 | `bc3a74b9342e` | `fix(vector): allocator 형식과 빈 반복자 연산 보정` | A | `VECTOR, ALLOCATOR, EDGE` | Removes empty-storage pointer arithmetic and adopts allocator-provided size types. |
| 6 | `b3124b3808d5` | `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리` | A | `VECTOR, EXCEPTION, ALLOCATOR` | Makes construction, assignment, resize growth, and aliased push-back transactional. |
| 7 | `9051be26db5e` | `test(vector): 생성·대입·크기 변경 실패 주입` | A | `TEST, VECTOR, EXCEPTION` | Injects element and allocation failures to validate rollback and live-object counts. |
| 8 | `797c33904db3` | `fix(vector): fill·range 삽입의 객체 수명 보존` | S | `CORE, VECTOR, HARD` | Rebuilds fill/range insertion around explicit live-object and raw-storage paths. |
| 9 | `8df3d8e067c0` | `test(vector): 삽입 복사·대입·할당 실패 sweep` | A | `TEST, VECTOR, EXCEPTION` | Sweeps insertion copy, assignment, and allocation failures across both capacity branches. |
| 10 | `5bdb6eb81a89` | `test(vector): 자기 범위 기대값을 명시적 snapshot으로 구성` | A | `TEST, VECTOR, DEBUG` | Replaces a questionable reference modifier with an explicit snapshot-derived test oracle. |

## 저장소 불변 조건

모든 커밋은 결국 다음 네 상태를 일치시키는 문제로 모입니다.

| 상태 | 의미 |
| --- | --- |
| `_alloc` | 현재 block과 그 안의 객체를 생성·파괴·해제할 allocator |
| `_data` | allocation의 시작 주소, 또는 allocation이 없을 때 `NULL` |
| `_size` | 생성이 완료되어 반드시 destroy해야 하는 객체 수 |
| `_capacity` | allocator가 확보한 전체 slot 수 |

따라서 `_size`는 단순한 논리 길이가 아닙니다. destructor와 rollback이 신뢰하는 **수명 회계 값**입니다. constructor가 성공하기 전에 늘려서는 안 되고, 성공한 객체를 파괴하기 전에 줄여서도 안 됩니다.

## `2eb9cb6c4273` — raw allocation과 live object를 처음 분리

**중요도** `S` · **태그** `CORE, VECTOR, ALLOCATOR`

초기 `include/ft_vector.hpp`는 네 상태 필드와 allocator 기반 fill construction, cleanup을 도입합니다.

```cpp
allocator_type _alloc;
pointer        _data;
size_type      _size;
size_type      _capacity;
```

fill constructor의 중요한 순서는 “allocate → 하나씩 construct → 성공한 수만 size 증가”입니다.

```cpp
_data = _alloc.allocate(count);
_capacity = count;
try
{
    while (_size < count)
    {
        _alloc.construct(_data + _size, value);
        ++_size;
    }
}
catch (...)
{
    _destroy_storage();
    throw;
}
```

`construct`가 세 번째 원소에서 던졌다면 `_size == 2`이므로 cleanup은 정확히 앞의 두 객체만 destroy합니다. 아직 constructor가 끝나지 않은 나머지 slot은 raw storage이므로 destroy 대상이 아닙니다.

```text
초기                 allocate 후             두 객체 생성 후
NULL / 0 / 0   →     block / 0 / N     →     block / 2 / N
                                                │
                                                └─ 다음 construct가 throw
                                                   → [0,2) destroy
                                                   → block deallocate
```

이 커밋의 가장 중요한 결정은 `_size`를 “성공한 constructor 수”로 삼은 것입니다. 이후 reserve, resize, assign, insert가 복잡해져도 cleanup이 올바르려면 이 정의가 유지되어야 합니다.

### 보장하는 것

- allocation과 object construction을 별개 단계로 취급합니다.
- fill construction 중 예외가 나면 생성 완료 prefix만 파괴하고 block을 해제합니다.
- destructor는 live prefix를 파괴한 뒤 storage를 해제합니다.
- 빈 vector는 `_data == NULL`, `_size == 0`, `_capacity == 0`으로 표현됩니다.

### 아직 없는 것

용량 증가, self-aliasing, null-backed iterator arithmetic, modifier rollback, in-place insertion은 아직 없습니다. 이 커밋은 후속 기능이 의존할 소유권 표현을 정한 단계입니다.

## `9db46550b13d` — 교체 block을 완성한 뒤 publish하는 재할당

**중요도** `A` · **태그** `VECTOR, ALLOCATOR, CORE`

`reserve`와 geometric growth, `_reallocate`가 추가됩니다. 재할당은 기존 storage를 먼저 파괴하지 않습니다.

```cpp
pointer new_data = _alloc.allocate(new_cap);
size_type i = 0;
try
{
    for (; i < _size; ++i)
        _alloc.construct(new_data + i, _data[i]);
}
catch (...)
{
    while (i)
        _alloc.destroy(new_data + --i);
    _alloc.deallocate(new_data, new_cap);
    throw;
}
```

새 block의 모든 원소 복사가 성공한 뒤에만 기존 원소를 파괴하고 기존 block을 해제한 다음, `_data`, `_size`, `_capacity`를 교체합니다.

```text
old block: [live old objects]   ← 복사 중에도 그대로 유지
new block: [constructed prefix][raw tail]

복사 실패 → new prefix만 destroy/deallocate, old block 유지
복사 완료 → old block destroy/deallocate, new block publish
```

이 순서는 element copy constructor나 allocator가 예외를 던져도 기존 vector를 잃지 않게 합니다. 재할당 경로에서 observable state를 바꾸는 시점은 새 block이 완성된 뒤입니다.

초기 용량 증가는 0에서 1로 시작하고 이후 두 배를 목표로 합니다. 하지만 단순 `_capacity * 2`는 allocator 상한 근처에서 overflow할 수 있으며, 다음 커밋이 그 산술을 교정합니다.

## `6f3cbf4794c9` — `max_size()` 앞에서 overflow 없이 포화

**중요도** `A` · **태그** `VECTOR, ALLOCATOR, EDGE`

용량 계산과 `resize`, `assign`, `insert`에 allocator 상한 검사가 추가됩니다. 핵심은 곱셈을 한 뒤 overflow를 검사하는 것이 아니라, 뺄셈으로 배수 가능 여부를 먼저 판단하는 것입니다.

```cpp
const size_type limit = max_size();

if (minimum > limit)
    throw std::length_error("ft::vector capacity");

if (_capacity == 0)
    next = 1;
else if (_capacity > limit - _capacity)
    next = limit;
else
    next = _capacity * 2;

if (next < minimum)
    next = minimum;
```

`_capacity > limit - _capacity`이면 `2 * _capacity`가 limit을 넘으므로 곱셈하지 않고 곧바로 `limit`에 포화합니다. 이 경계가 없으면 unsigned overflow로 작은 capacity가 계산되어, 실제 필요한 수보다 작은 block을 확보하거나 잘못된 재할당 경로로 들어갈 수 있습니다.

이 커밋은 allocation failure와 별개인 **요청 크기 표현의 오류**를 `length_error`로 조기에 분리합니다.

## `bdc4c3123bc9` — mutation 전에 self-range를 snapshot

**중요도** `A` · **태그** `VECTOR, EDGE, DEBUG`

range `assign`과 `insert`의 입력 iterator가 자기 vector를 가리킬 수 있다는 문제가 드러납니다. 예를 들어 다음 호출에서 `first`와 `last`는 mutation 대상과 같은 storage를 참조합니다.

```cpp
values.assign(values.begin() + 1, values.end());
values.insert(values.begin() + 2, values.begin(), values.end());
```

기존 storage를 먼저 clear하거나 reserve하면 입력 iterator가 곧바로 무효화됩니다. 이 커밋은 입력을 임시 vector에 먼저 복사합니다.

```cpp
template <class InputIt>
void assign(InputIt first, InputIt last, ...)
{
    vector tmp(first, last, _alloc);
    swap(tmp);
}
```

range insert도 먼저 입력 범위를 `tmp`에 담고, 기존 tail을 별도 vector에 보존한 뒤 대상의 tail을 지우고 입력과 tail을 다시 붙이는 방식으로 변경됩니다.

이 결정은 “입력을 다 읽기 전에 그 입력을 파괴하지 않는다”는 aliasing 규칙을 확립합니다. 다만 초기 insert 해결책은 tail을 지우고 재구성하므로 다음과 같은 비용과 부작용이 있습니다.

- 여유 capacity가 있어도 불필요한 임시 저장소와 복사가 생깁니다.
- 삽입 뒤에도 보존될 수 있었던 prefix 주소 외의 상태를 더 크게 흔듭니다.
- raw tail과 live slot을 직접 구분하는 근본적인 in-place 알고리즘은 아직 없습니다.

`797c33904db3`은 snapshot 원칙은 유지하면서 삽입 자체를 재할당/제자리 경로로 다시 구현합니다.

## `bc3a74b9342e` — allocator 형식과 빈 storage의 pointer 연산

**중요도** `A` · **태그** `VECTOR, ALLOCATOR, EDGE`

두 종류의 표현 문제가 함께 교정됩니다.

첫째, `size_type`과 `difference_type`을 고정된 표준 정수 형식 대신 allocator가 제공하는 형식에서 가져옵니다.

```cpp
typedef typename allocator_type::difference_type difference_type;
typedef typename allocator_type::size_type size_type;
```

둘째, 빈 vector의 `_data == NULL` 상태에서 `NULL + 0`이나 `NULL - NULL` 같은 pointer arithmetic을 피합니다.

```cpp
iterator _iterator_at(size_type index)
{
    return _data ? _data + index : _data;
}

size_type _index_of(const_iterator pos) const
{
    return _data ? static_cast<size_type>(pos - _data) : 0;
}
```

빈 범위 erase도 `first == last`이면 subtraction을 하지 않고 count를 0으로 정합니다. 결과 값이 직관적으로 0이더라도 C++ pointer 연산의 operand가 유효한 array를 가리키지 않으면 그 표현 자체를 안전하다고 볼 수 없습니다. 이 커밋은 빈 상태를 별도 branch로 다룹니다.

## `b3124b3808d5` — 생성 완료 전에는 기존 상태를 교체하지 않기

**중요도** `A` · **태그** `VECTOR, EXCEPTION, ALLOCATOR`

여러 modifier에 흩어진 예외 창을 같은 transaction 원칙으로 정리합니다.

### fill construction

local block에서 모든 객체를 만든 뒤에만 멤버 상태를 publish합니다.

```cpp
pointer new_data = _alloc.allocate(count);
size_type constructed = 0;
try
{
    for (; constructed < count; ++constructed)
        _alloc.construct(new_data + constructed, value);
}
catch (...)
{
    while (constructed)
        _alloc.destroy(new_data + --constructed);
    _alloc.deallocate(new_data, count);
    throw;
}

_data = new_data;
_size = count;
_capacity = count;
```

객체 생성 도중에는 `_data`가 새 block을 가리키지 않으므로 partially constructed state가 vector의 정상 상태처럼 노출되지 않습니다.

### assign

fill/range assign은 같은 allocator로 임시 vector를 완성한 뒤 storage 세 필드만 교체합니다.

```cpp
vector tmp(count, value, _alloc);
_swap_storage(tmp);
```

`_swap_storage`가 `_alloc`을 바꾸지 않는 점이 중요합니다. temporary가 target과 같은 allocator로 replacement block을 만들고, 교체 후 temporary가 target의 old block을 같은 allocator로 파괴합니다.

### resize growth

resize가 기존 capacity 안에서 새 객체를 만들다가 실패하면, 이 호출에서 추가한 suffix만 파괴합니다.

```cpp
const size_type old_size = _size;
try
{
    while (_size < count)
    {
        _alloc.construct(_data + _size, value);
        ++_size;
    }
}
catch (...)
{
    while (_size > old_size)
        _alloc.destroy(_data + --_size);
    throw;
}
```

### aliased `push_back`

capacity가 꽉 찼을 때 `value`가 vector 내부 원소를 참조한다면 `reserve`가 그 참조를 무효화합니다. 따라서 reserve 전에 복사합니다.

```cpp
if (_size == _capacity)
{
    value_type value_copy(value);
    reserve(_next_capacity(_size + 1));
    _alloc.construct(_data + _size, value_copy);
}
else
    _alloc.construct(_data + _size, value);
++_size;
```

`_size`는 마지막 construction이 성공한 뒤에만 증가합니다. 이 한 줄의 위치가 raw slot을 live object로 publish하는 commit point입니다.

## `9051be26db5e` — 수명 오류를 직접 계측하는 실패 주입

**중요도** `A` · **태그** `TEST, VECTOR, EXCEPTION`

`tests/test_vector_exceptions.cpp`는 일반 정수 대신 copy/assignment에서 선택적으로 예외를 던지는 `tracked_value`와, allocation call 위치를 선택해 실패시키는 `tracking_allocator`를 사용합니다.

`tracked_value`는 live object 주소를 기록해 다음 오류를 구분합니다.

- 생성되지 않은 주소를 copy source로 사용했는가
- 이미 파괴했거나 생성하지 않은 주소를 destroy했는가
- 예외 뒤 live object 수가 vector의 size/소유 block과 일치하는가

테스트가 겨냥하는 production 경로는 다음과 같습니다.

| 주입 지점 | 기대하는 상태 |
| --- | --- |
| fill constructor copy 실패 | 생성된 prefix와 allocation이 모두 회수됨 |
| fill/range assign 실패 | 원래 target 내용과 storage가 유지됨 |
| copy assignment 실패 | target이 교체되지 않음 |
| resize suffix construction 실패 | 호출 전 size로 돌아감 |
| aliased `push_back`의 복사/재할당 | source value를 mutation 전에 보존함 |
| allocator 상한/실패 | 잘못된 size publish나 block leak가 없음 |
| 빈 iterator 연산 | null-backed empty state에서 invalid arithmetic을 피함 |

이 테스트는 특정 throw count를 바꿔가며 실패 branch를 재현한다는 점에서 단순 결과 비교보다 강합니다. 다만 이 시점에는 fill/range insert의 복잡한 spare-capacity 수명 문제를 충분히 다루지 않으며, 그 부분은 다음 fix와 test가 맡습니다.

## `797c33904db3` — 삽입을 재할당과 제자리 수명 알고리즘으로 분리

**중요도** `S` · **태그** `CORE, VECTOR, HARD`

이 커밋은 Thread의 핵심입니다. fill/range 삽입에서 먼저 입력을 보존하고, 이후 **capacity가 부족한 경로**와 **이미 확보된 raw tail을 사용하는 경로**를 완전히 분리합니다.

### 1. 입력 capture가 mutation보다 먼저 온다

fill insert는 `value`가 자기 원소일 수 있으므로 먼저 복사합니다.

```cpp
value_type value_copy(value);

if (_size + count > _capacity)
    _insert_fill_reallocate(index, count, value_copy,
        _next_capacity(_size + count));
else
    _insert_fill_in_place(index, count, value_copy);
```

range insert는 입력을 같은 allocator의 temporary vector로 모두 읽습니다.

```cpp
size_type index = _index_of(pos);
vector tmp(first, last, _alloc);

if (tmp.empty())
    return;
```

따라서 reserve, assignment, tail 이동이 시작될 때 원래 iterator가 무효화되어도 삽입 데이터는 `tmp`에 남습니다. 이는 self-range뿐 아니라 single-pass input iterator를 mutation 전에 소비하는 역할도 합니다.

### 2. 재할당 경로: 새 배열 전체를 완성한 뒤 교체

새 block에는 prefix, 삽입 값, 기존 suffix 순으로 construction합니다.

```cpp
pointer new_data = _alloc.allocate(new_capacity);
size_type constructed = 0;

try
{
    for (size_type i = 0; i < index; ++i, ++constructed)
        _alloc.construct(new_data + constructed, _data[i]);

    for (size_type i = 0; i < count; ++i, ++constructed)
        _alloc.construct(new_data + constructed, value);

    for (size_type i = index; i < _size; ++i, ++constructed)
        _alloc.construct(new_data + constructed, _data[i]);
}
catch (...)
{
    while (constructed)
        _alloc.destroy(new_data + --constructed);
    _alloc.deallocate(new_data, new_capacity);
    throw;
}

_replace_storage(new_data, constructed, new_capacity);
```

range 버전은 가운데 loop의 source만 snapshot vector로 바뀝니다. 예외가 나면 old storage는 아직 손대지 않았으므로 새 prefix만 회수하면 됩니다. 모든 construction이 성공해야 `_replace_storage`가 old storage를 파괴하고 새 block을 publish합니다.

이 경로는 copy construction/allocation failure에 대해 원래 내용과 storage를 유지하는 강한 rollback 구조를 가집니다.

### 3. 제자리 경로: live slot에는 assignment, raw tail에는 construction

여유 capacity가 있다고 해서 모든 목적지에 assignment할 수 있는 것은 아닙니다. 기존 `_size` 이후는 객체가 없는 raw memory입니다. 반대로 기존 범위 안에는 이미 객체가 있으므로 placement construction을 겹쳐 실행할 수 없습니다.

```text
삽입 전
[ prefix | tail live objects | raw capacity ........ ]
          ^ index              ^ old_size

삽입 후
[ prefix | inserted | shifted tail | raw capacity .. ]
```

구현은 삽입 수 `count`와 기존 tail 길이 `tail_size = old_size - index`의 관계에 따라 다시 나뉩니다.

#### `count <= tail_size`

1. 끝의 `count`개 live object를 raw tail에 copy-construct합니다.
2. 나머지 live tail을 뒤에서 앞으로 assignment해 겹침을 피합니다.
3. 삽입 구간의 기존 live slot에 새 값을 assignment합니다.

```cpp
for (; constructed < count; ++constructed)
    _alloc.construct(_data + old_size + constructed,
        _data[old_size - count + constructed]);

for (size_type i = old_size - count; i > index; --i)
    _data[i + count - 1] = _data[i - 1];

for (size_type i = 0; i < count; ++i)
    _data[index + i] = value;
```

#### `count > tail_size`

삽입 값 일부와 기존 tail 전체가 raw 영역으로 넘어갑니다.

1. 기존 tail 뒤에 생기는 `extra = count - tail_size`개의 삽입 값을 raw slot에 construct합니다.
2. 기존 tail을 그 뒤 raw slot에 copy-construct합니다.
3. 기존 tail이 있던 live slot에는 삽입 값을 assignment합니다.

```cpp
const size_type extra = count - tail_size;

for (; constructed < extra; ++constructed)
    _alloc.construct(_data + old_size + constructed, value);

for (size_type i = 0; i < tail_size; ++i, ++constructed)
    _alloc.construct(_data + old_size + constructed,
        _data[index + i]);

for (size_type i = 0; i < tail_size; ++i)
    _data[index + i] = value;
```

range 버전도 같은 수명 배치를 사용하고, 삽입 값 source만 snapshot vector의 원소로 바뀝니다.

### 4. 예외 뒤 무엇을 되돌릴 수 있는가

새 raw tail에 construction한 객체 수를 `constructed`로 따로 세고, 예외가 나면 그 suffix만 destroy합니다.

```cpp
catch (...)
{
    _destroy_constructed_tail(old_size, constructed);
    throw;
}
_size = old_size + count;
```

`_size`는 모든 construction과 assignment가 끝난 뒤에만 증가합니다. 따라서 실패해도 destructor가 새 raw tail을 기존 live 영역으로 오인하지 않습니다.

다만 제자리 경로의 예외 보장은 재할당 경로와 같지 않습니다.

- raw tail copy construction 도중 실패하면 기존 live 구간을 아직 대입하지 않았으므로 원래 내용이 유지됩니다.
- tail 이동이나 삽입 값 assignment가 던지면 일부 기존 live object는 이미 대입된 상태일 수 있습니다.
- catch는 새로 construct한 tail을 회수하고 `_size`를 old size로 유지하지만, 이미 성공한 assignment의 값을 되돌리지는 않습니다.

즉 제자리 assignment 실패에서는 **객체 수명과 컨테이너 유효성은 유지하지만 내용의 완전한 rollback은 보장하지 않는 기본 보장**입니다. 이 차이를 숨기지 않는 것이 이 알고리즘을 이해하는 핵심입니다.

## `8df3d8e067c0` — 두 capacity branch의 실패 지점을 sweep

**중요도** `A` · **태그** `TEST, VECTOR, EXCEPTION`

후속 테스트는 삽입의 서로 다른 단계에 실패를 주입합니다.

- fill argument가 자기 원소를 참조하는 alias case
- 재할당 경로의 prefix/inserted/suffix copy construction 실패
- 제자리 경로의 raw-tail construction 실패
- 제자리 tail 이동/삽입 assignment 실패
- range insertion의 각 copy 위치를 바꾸는 failure sweep
- allocation 자체가 실패하는 경로
- 여유 capacity가 있을 때 prefix 주소가 유지되는지

검증 기준도 실패 종류에 따라 다릅니다.

| 실패 종류 | 테스트가 요구하는 것 |
| --- | --- |
| 새 block allocation/copy 실패 | 원래 값·size·block ownership 유지 |
| raw tail construction 실패 | 새 tail 객체 전부 파괴, old size 유지 |
| live-slot assignment 실패 | old size와 유효한 수명 회계 유지, 이후 사용 가능 |
| 성공한 spare-capacity insert | allocation 없이 기존 prefix 주소 유지 |
| aliased fill/range | mutation 전 snapshot의 값이 삽입됨 |

assignment failure에서 원래 내용까지 항상 같다고 요구하지 않는 것은 production 코드의 실제 보장과 맞습니다. 테스트는 강한 보장이 있는 경로와 기본 보장만 가능한 경로를 구분합니다.

## `5bdb6eb81a89` — self-range 확장의 기대값을 독립적으로 계산

**중요도** `A` · **태그** `TEST, VECTOR, DEBUG`

self-range assign/insert의 expected result를 `std::vector`에 같은 overlapping modifier를 호출해 만들던 방식이 바뀝니다. 대신 source 범위를 먼저 별도 snapshot으로 만들고, 그 snapshot을 이용해 기대 sequence를 명시적으로 조립합니다.

```text
잘못된 oracle 의존:
expected.insert(expected.begin() + pos,
                expected.begin() + first,
                expected.begin() + last);

수정된 oracle:
snapshot = 원래 source 범위 복사
expected = 원래 prefix + snapshot + 원래 suffix
```

이 프로젝트는 self-range 입력을 mutation 전에 보존하는 동작을 의도적으로 제공합니다. 그렇다면 기대값도 그 계약에서 직접 도출해야 합니다. 다른 컨테이너에 겹치는 source/destination을 그대로 전달하면 reference implementation의 별도 전제나 미정의 영역을 기대값으로 오인할 수 있습니다.

이 커밋은 production 코드를 바꾸지 않지만, regression이 실제로 보호하려는 계약을 독립적으로 정의한다는 점에서 중요도가 A입니다. 잘못된 oracle은 올바른 구현을 실패시키거나 잘못된 구현과 함께 통과할 수 있습니다.

## 최종 보장과 경계

| 경로 | 성공 commit point | 예외 시 상태 |
| --- | --- | --- |
| fill/range construction | 모든 원소 construct 뒤 멤버 publish | 생성 prefix destroy, block deallocate |
| `reserve`/재할당 | 새 block 전체 복사 뒤 storage 교체 | old block과 내용 유지 |
| fill/range `assign` | temporary 완성 뒤 storage swap | target 유지 |
| `resize` growth | 각 construct 뒤 size 증가, 전체 실패 시 old size까지 rollback | 추가 suffix destroy |
| full-capacity `push_back` | aliased value snapshot → reserve → construct → size 증가 | 기존 vector 유지 |
| insert 재할당 | prefix+삽입+suffix 전체 construct 뒤 교체 | 기존 vector 유지 |
| insert 제자리, construction 실패 | raw tail의 생성 수만 회수 | old size와 기존 내용 유지 |
| insert 제자리, assignment 실패 | 새 raw tail 회수, size는 old size | 유효하지만 일부 기존 값은 바뀔 수 있음 |
| self-range input | mutation 전 temporary snapshot | 입력 iterator 무효화와 무관하게 snapshot 보존 |
| capacity 계산 | allocator 상한 안에서만 배수/포화 | 상한 초과는 `length_error` |
| 빈 iterator 계산 | null pointer arithmetic을 branch로 회피 | index/end는 빈 상태로 유지 |

최종 실행 흐름은 다음과 같습니다.

```text
[요청 크기와 max_size 검사]
        ↓
[입력 alias 가능성 capture]
        ↓
capacity 부족? ── yes ──> [새 block에 prefix / inserted / suffix construct]
        │                          ├─ 실패: 새 prefix만 회수
        │                          └─ 성공: old storage 파괴 후 publish
        │
        no
        ↓
[raw tail에는 construct / 기존 live slot에는 assignment]
        ├─ 실패: 새로 construct한 tail만 destroy, old size 유지
        └─ 성공: 마지막에 _size publish
```

이 Thread가 보장하는 것은 모든 예외 뒤 값이 반드시 원래와 같다는 것이 아닙니다. 핵심 보장은 다음 세 가지입니다.

1. 생성되지 않은 slot을 live object로 기록하지 않습니다.
2. 생성한 객체를 누락 없이 파괴하고, 각 allocation을 정확한 allocator로 해제합니다.
3. mutation이 입력을 무효화하기 전에 aliased input을 보존합니다.

map node 소유권, tree balancing, header sentinel, stateful map comparator transaction은 별도 Thread의 문제입니다.
