# Thread 05 — Stateful allocator와 Map failure transaction

> 대상: `seungwoo7050/42-archive`의 `cpp/ft_container` 브랜치
> 검토 방식: 아래 커밋의 정확한 SHA에서 diff와 당시 source를 확인했습니다. 이 환경에서는 build와 test를 실행하지 않았습니다.

## 개요

map의 correctness는 key/value가 정렬되어 보이는 것만으로 끝나지 않습니다.

- comparator는 어떤 key가 equivalent하며 node links가 어떤 ordering을 뜻하는지 결정합니다.
- value allocator와 그로부터 rebound된 node allocator는 각 node를 어느 resource state가 allocate하고 deallocate해야 하는지 결정합니다.
- copy assignment와 swap은 comparator, allocator, tree ownership을 함께 다루므로 이들을 어떤 순서로 바꾸느냐가 failure semantics를 결정합니다.

이 Thread의 공통 규칙은 다음과 같습니다.

> **예외를 던질 수 있는 policy 작업은 물리적 tree ownership을 옮기기 전에 끝내고, 할당된 node는 어느 시점에도 owner 없는 상태로 남기지 않습니다.**

초기 구현은 이 규칙을 네 곳에서 어깁니다.

1. rebound node allocator를 default-construct해 public value allocator의 state를 잃습니다.
2. node를 할당한 뒤 comparator를 다시 호출해, 비교 예외가 나면 unlinked allocation이 남을 수 있습니다.
3. constructor와 copy assignment가 target tree를 직접 구축·파괴해 중간 실패를 격리하지 못합니다.
4. swap이 tree와 allocator를 먼저 교환한 뒤 comparator를 교환해, comparator assignment가 던지면 policy와 physical owner가 서로 어긋납니다.

후속 fix와 test는 allocator identity, comparator call 순서, temporary tree, outstanding allocation accounting을 이용해 이 네 경계를 닫습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ae180871b160` | `fix(map): 값 allocator 상태로 노드 allocator 구성` | A | `RB_TREE, ALLOCATOR, RISK` | Preserves value-allocator state when rebinding to node allocation. |
| 2 | `cb08194d17b0` | `fix(map): 삽입 위치를 노드 할당 전에 확정` | A | `RB_TREE, EXCEPTION, DEBUG` | Finishes comparator search before allocating an unlinked node. |
| 3 | `55d3b3e7c104` | `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리` | A | `RB_TREE, EXCEPTION, ALLOCATOR` | Cleans partial constructors and builds copy-assignment replacement trees off to the side. |
| 4 | `d72b04c5ddc6` | `test(map): 비교·할당 실패 시 노드 소유권 검증` | A | `TEST, RB_TREE, EXCEPTION` | Injects comparison and allocation failures and accounts for every node. |
| 5 | `0f4dd84e44ed` | `fix(map): 비교자 교환 실패 전에 tree 소유권 유지` | A | `RB_TREE, EXCEPTION, DEBUG` | Exchanges comparator state before allocator and tree ownership can move. |
| 6 | `55d4ba1fb493` | `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증` | A | `TEST, RB_TREE, EXCEPTION` | Verifies copy assignment and public swap under comparator-assignment failure. |

## `ae180871b160` — rebind는 type만 바꾸고 allocator state는 이어야 한다

**중요도** `A` · **태그** `RB_TREE, ALLOCATOR, RISK`

초기 map에는 두 allocator object가 있습니다.

```cpp
allocator_type _alloc;
node_allocator _node_alloc;
```

`allocator_type`은 public `value_type`용 allocator이고, `node_allocator`는 이를 internal node type으로 rebind한 allocator입니다. 문제는 초기 constructor가 `_node_alloc(node_allocator())`처럼 독립 default construction했다는 점입니다.

stateless `std::allocator`에서는 차이가 보이지 않습니다. 그러나 allocator가 다음 상태를 보유하면 결과가 달라집니다.

```text
arena pointer
pool id
allocation counter
failure-injection state
resource handle
```

public map에 전달된 allocator는 arena A를 가리키는데 node allocator는 default arena B를 가리킬 수 있습니다. 그러면 `get_allocator()`가 말하는 owner와 실제 node allocation owner가 분리됩니다.

fix는 모든 constructor에서 node allocator를 value allocator로부터 구성합니다.

```cpp
explicit map(const key_compare& comp = key_compare(),
    const allocator_type& alloc = allocator_type())
    : _alloc(alloc),
      _node_alloc(node_allocator(alloc)),
      _root(NULL),
      _size(0),
      _comp(comp)
{
}
```

range constructor도 전달받은 `alloc`, copy constructor도 `other._alloc`을 rebound constructor에 넘깁니다.

```cpp
_node_alloc(node_allocator(other._alloc))
```

rebind의 의미는 다음과 같이 정리됩니다.

```text
allocator<value_type, state S>
        │ rebind
        ▼
allocator<node, state S>
```

allocated type은 바뀌지만 resource identity는 유지됩니다. 그 결과 node를 allocate한 allocator family와 destroy/deallocate하는 allocator family가 같은 state를 공유합니다.

이 커밋은 failure rollback을 직접 바꾸지는 않습니다. 다만 후속 tracking allocator test가 node allocation까지 같은 counter와 failure point를 관찰할 수 있는 기반입니다.

## `cb08194d17b0` — comparator를 모두 끝낸 뒤 node를 할당

**중요도** `A` · **태그** `RB_TREE, EXCEPTION, DEBUG`

### 이전 위험

기존 insert는 search loop로 parent를 찾은 뒤 node를 allocate/construct하고, 어느 child link에 붙일지 결정하기 위해 comparator를 한 번 더 호출했습니다.

```cpp
node* created = _create_node(value);
created->parent = parent;

if (_comp(value.first, parent->value.first))
    parent->left = created;
else
    parent->right = created;
```

이 마지막 `_comp`가 던지면 상태는 다음과 같습니다.

```text
created node: allocation + construction 완료
tree link: 아직 없음
local cleanup guard: 없음
```

node는 tree에서 도달할 수 없고 `_clear(_root)`도 찾지 못합니다. allocate한 resource가 어느 container의 reachable ownership에도 들어가지 않은 window입니다.

### 결정: search 결과에 attachment side까지 포함

search loop에서 방향을 결정할 때 `insert_left`를 함께 저장합니다.

```cpp
node* parent = NULL;
node* cur = _root;
bool insert_left = false;

while (cur)
{
    parent = cur;
    if (_comp(value.first, cur->value.first))
    {
        insert_left = true;
        cur = cur->left;
    }
    else if (_comp(cur->value.first, value.first))
    {
        insert_left = false;
        cur = cur->right;
    }
    else
        return ft::make_pair(iterator(cur, _root), false);
}
```

모든 comparator call과 duplicate detection이 끝난 다음 node를 만듭니다.

```cpp
node* created = _create_node(value);
created->parent = parent;

if (insert_left)
    parent->left = created;
else
    parent->right = created;
```

allocation 이후에는 pointer link, color fixup, size 증가만 남습니다. comparator 예외는 allocation 전에 발생하므로 새 node ownership 자체가 생기지 않습니다.

```text
비교 중 예외
→ tree unchanged
→ 새 allocation 없음

비교 완료
→ allocation/construct
→ 즉시 parent link에 publish
→ balancing
```

이 fix는 “catch에서 orphan node를 찾아 정리”하는 사후 처리보다 transaction boundary를 앞당긴 root-cause 수정입니다.

## `55d3b3e7c104` — constructor rollback과 copy assignment의 replacement tree

**중요도** `A` · **태그** `RB_TREE, EXCEPTION, ALLOCATOR`

이 커밋은 construction과 assignment를 같은 방식으로 처리하지 않습니다. constructor는 아직 완성되지 않은 `this` 안에서 partial tree를 직접 만들고, assignment는 완성된 target을 보존해야 하므로 temporary map에 replacement를 먼저 만듭니다.

### range/copy constructor: partial tree를 명시적으로 clear

C++ constructor body가 예외를 던지면 object destructor는 호출되지 않습니다. member destructor는 호출되지만 raw tree nodes를 소유하는 map destructor는 실행되지 않으므로, 이미 insert된 node를 직접 정리해야 합니다.

```cpp
try
{
    insert(first, last);
}
catch (...)
{
    clear();
    throw;
}
```

copy constructor도 같습니다.

```cpp
try
{
    insert(other.begin(), other.end());
}
catch (...)
{
    clear();
    throw;
}
```

각 successful insert로 tree에 연결된 node는 `clear()`에서 도달 가능하므로 모두 destroy/deallocate됩니다.

### copy assignment: 원본 target을 먼저 지우지 않는다

이전 assignment는 다음 순서였습니다.

```text
target.clear()
target comparator 변경
source를 하나씩 insert
```

중간 allocation/comparison 예외가 나면 target의 원래 값은 이미 사라지고 partial copy만 남습니다.

fix는 target allocator를 사용해 replacement map을 별도로 완성합니다.

```cpp
map tmp(other.begin(), other.end(), other._comp, _alloc);
_swap_tree_and_compare(tmp);
```

여기서 allocator 선택이 중요합니다.

- 새 node는 **target의 `_alloc`**로 생성됩니다.
- ordering은 **source의 `_comp`**를 사용합니다.
- temporary construction이 실패하면 `tmp`만 rollback되고 target은 untouched입니다.
- 성공 후 tree와 comparator를 교환하면 old target tree가 `tmp`로 이동합니다.
- scope 종료 시 `tmp`가 old target tree를 target allocator로 해제합니다.

```text
target before: [allocator T][comparator Ct][tree Old]
source:        [allocator S][comparator Cs][tree Source]

tmp build:     [allocator T][comparator Cs][tree New]

commit:
target         [allocator T][comparator Cs][tree New]
tmp            [allocator T][comparator Ct][tree Old]

tmp destructor → Old를 allocator T로 해제
```

allocator를 교환하지 않는 internal helper는 copy assignment가 target allocator identity를 유지하도록 합니다.

### 아직 남은 ordering defect

이 SHA의 `_swap_tree_and_compare`는 tree와 size를 먼저 바꾼 뒤 comparator를 교환합니다.

```cpp
std::swap(_root, other._root);
std::swap(_size, other._size);
std::swap(_comp, other._comp);
```

comparator assignment가 던질 수 있다면 tree ownership은 이미 이동했는데 comparator는 교환되지 않은 상태가 될 수 있습니다. 이 subtle failure는 `0f4dd84e44ed`이 교환 순서를 바꾸어 닫습니다.

즉 이 커밋은 allocation/comparator-call failure transaction을 크게 개선하지만, comparator **assignment** failure까지 완결하지는 않습니다.

## `d72b04c5ddc6` — comparison과 allocation failure를 node count로 관찰

**중요도** `A` · **태그** `TEST, RB_TREE, EXCEPTION`

production code는 바뀌지 않고 `tests/test_map_exceptions.cpp`가 추가됩니다. test는 두 개의 stateful policy를 사용합니다.

### throwing comparator

```cpp
struct comparison_state
{
    int calls;
    int throw_on_call;
};
```

`throwing_less::operator()`는 지정한 call index에서 예외를 던집니다. 이를 통해 search 중 첫 비교, 두 번째 비교, range insertion 도중 특정 비교 등을 결정적으로 재현합니다.

### tracking allocator

allocator state에는 다음 값이 있습니다.

```cpp
int outstanding_blocks;
int allocation_calls;
int throw_on_call;
```

allocate 성공 시 `outstanding_blocks`를 증가시키고 deallocate 시 감소시킵니다. rebind constructor는 같은 state pointer를 전달하므로 value allocator와 node allocator가 하나의 ownership ledger를 공유합니다.

### 검증 matrix

| 대상 | 주입 지점 | 핵심 assertion |
| --- | --- | --- |
| single insert | comparator call 여러 위치 | 실패하면 기존 key만 남고 `outstanding == size` |
| range constructor | comparator / allocation | constructor 실패 뒤 outstanding 0 |
| copy constructor | comparator | source node 수만 남고 partial copy node는 0 |
| copy assignment | comparator / allocation | target의 기존 key 유지, temporary node 전부 rollback |
| scope 종료 | 정상·실패 공통 | outstanding 0 |

특히 insert test는 comparator가 throw한 경우 새 allocation이 tree 밖에 남지 않는지 확인합니다.

```text
실패 직후
outstanding_blocks == values.size()

scope 종료
outstanding_blocks == 0
```

이 test는 public output만 비교하는 대신 **allocated node 수와 reachable logical size의 관계**를 직접 관찰합니다.

### 증명 범위

증명하는 것:

- tested comparator call positions에서 insert가 orphan node를 만들지 않음
- range/copy constructor의 partial tree rollback
- copy assignment의 comparator/allocation failure에서 target 보존
- tracking allocator state가 rebind node allocation까지 이어짐
- scope 종료 뒤 allocation counter가 0

증명하지 않는 것:

- comparator assignment가 던지는 swap/commit failure
- allocator assignment/swap 자체가 던지는 경우
- 모든 user type constructor/destructor behavior
- test의 실제 실행 결과; source와 assertions만 검토했습니다.

comparator assignment failure는 별도의 policy test가 필요하며 다음 두 커밋이 담당합니다.

## `0f4dd84e44ed` — policy 교환을 physical ownership보다 먼저

**중요도** `A` · **태그** `RB_TREE, EXCEPTION, DEBUG`

### 실패 가능한 commit 순서

copy assignment helper의 이전 순서는 다음과 같습니다.

```text
1. tree/root 교환       ← physical ownership 이동
2. size 교환
3. comparator 교환      ← 여기서 throw 가능
```

3에서 예외가 나면 target은 source ordering으로 만든 new tree를 이미 받았지만 comparator는 old ordering일 수 있습니다. temporary에는 old target tree가 있지만 comparator 상태가 어느 쪽까지 바뀌었는지 불명확합니다.

public `swap`도 comparator를 가장 마지막에 바꾸므로 더 위험합니다. allocator와 node allocator, root, size가 이미 이동한 뒤 comparator가 던질 수 있습니다.

### 수정: comparator부터 교환

internal assignment commit helper:

```cpp
void _swap_tree_and_compare(map& other)
{
    std::swap(_comp, other._comp);
    std::swap(_header.parent, other._header.parent);
    std::swap(_size, other._size);
    _refresh_header();
    other._refresh_header();
}
```

public swap:

```cpp
void swap(map& other)
{
    std::swap(_comp, other._comp);
    std::swap(_alloc, other._alloc);
    std::swap(_node_alloc, other._node_alloc);
    std::swap(_header.parent, other._header.parent);
    std::swap(_size, other._size);
    _refresh_header();
    other._refresh_header();
}
```

이 SHA에는 Thread 04의 header representation이 이미 존재하므로 physical tree ownership은 `_header.parent`가 나타냅니다.

새 순서는 다음과 같습니다.

```text
potentially throwing policy exchange
        ↓ 성공한 경우에만
allocator/tree ownership exchange
        ↓
header-root/extrema refresh
```

test comparator처럼 assignment가 **자기 상태를 바꾸기 전에** throw하면 comparator exchange에서 중단되고 allocator/root/size에는 손대지 않습니다. 두 map은 각각 원래 tree와 allocator를 그대로 보유합니다.

### copy assignment와 public swap의 차이

| 연산 | 교환하는 것 | allocator 결과 |
| --- | --- | --- |
| copy assignment helper | comparator, root, size | target allocator 유지 |
| public `swap` | comparator, value allocator, node allocator, root, size | allocator와 tree를 함께 교환 |

copy assignment의 temporary tree는 이미 target allocator로 만들어졌기 때문에 allocator를 바꾸지 않습니다. public swap은 각 tree가 자신을 할당한 allocator와 함께 이동해야 하므로 allocator objects도 tree와 같은 transaction에 포함됩니다.

### 보장 범위의 정확한 한계

이 fix가 확실히 제거하는 것은 **tree ownership을 먼저 이동시킨 뒤 comparator assignment가 실패하는 순서**입니다.

다만 C++98의 일반 user-defined comparator assignment가 자체적으로 strong guarantee를 제공한다고 이 코드가 강제할 수는 없습니다. comparator의 `operator=`가 일부 member를 바꾼 뒤 throw하면 첫 `std::swap(_comp, other._comp)` 자체가 partial policy state를 남길 수 있습니다. 후속 test는 throw-before-mutation comparator를 사용합니다.

또한 allocator swap/assignment가 throw하는 type까지 이 commit이 transactionally rollback한다고 증명하지 않습니다. 문서의 보장은 실제 코드와 test가 다루는 범위로 제한합니다.

## `55d4ba1fb493` — comparator assignment failure 뒤 두 owner를 확인

**중요도** `A` · **태그** `TEST, RB_TREE, EXCEPTION`

`tests/test_map_policy_exceptions.cpp`는 comparator 호출이 아니라 comparator **대입**에서 예외를 던집니다.

```cpp
throwing_compare& operator=(const throwing_compare& other)
{
    if (_control)
    {
        ++_control->assignments;
        if (_control->throw_on_assignment)
            throw policy_assignment_failure();
    }
    _control = other._control;
    _reverse = other._reverse;
    return *this;
}
```

throw가 실제 state 변경 전에 발생하므로 `0f4dd84e44ed`의 ordering contract를 직접 관찰할 수 있습니다.

### failed copy assignment

source는 ascending comparator와 source allocator를, target은 reverse comparator와 target allocator를 사용합니다. target의 comparator assignment를 실패시킨 뒤 다음을 확인합니다.

- 예외가 실제 comparator exchange까지 도달했습니다.
- target key order와 내용은 기존 `{9, 7}` 그대로입니다.
- target allocator의 outstanding block 수는 baseline과 같습니다.
- temporary replacement tree는 모두 해제됐습니다.
- 실패 뒤 target에 key 8을 삽입하고 찾을 수 있어 container가 계속 usable합니다.

여기서 source allocator와 target allocator counter가 분리되어 있으므로 잘못된 owner가 deallocate하는 문제도 드러납니다.

### failed public swap

left와 right는 서로 다른 comparator 방향과 allocator state를 가집니다. left comparator assignment를 실패시킨 뒤 다음을 확인합니다.

```text
left keys unchanged
right keys unchanged
left allocator identity unchanged
right allocator identity unchanged
각 allocator outstanding count unchanged
scope 종료 뒤 양쪽 outstanding 0
```

comparator exchange가 첫 단계이기 때문에 throw 시 allocator와 tree가 아직 이동하지 않았다는 것을 검증합니다.

### 증명하는 것 / 증명하지 않는 것

증명하는 것:

- tested throw-before-write comparator에서 failed copy assignment가 target tree와 target allocator ownership을 보존
- failed public swap이 양쪽 tree와 allocator identity를 보존
- temporary tree가 누수 없이 제거
- 실패 뒤 target이 계속 usable
- 모든 scope 종료 뒤 node allocation counter가 0

증명하지 않는 것:

- comparator assignment가 일부 state를 변경한 뒤 throw하는 경우
- allocator swap/assignment가 던지는 경우
- comparator copy constructor 자체의 모든 failure 시점
- multi-threaded use
- 실제 실행 성공; test code와 assertion만 확인했습니다.

## 최종 failure transaction

### Insert

```text
[comparator search + duplicate detection + insert_left 결정]
        ├─ throw → tree unchanged, allocation 없음
        └─ success
             ↓
        [node allocate + construct]
             ├─ construct throw → allocation 즉시 deallocate
             └─ success → parent link에 publish → balancing → size 증가
```

### Copy assignment

```text
[target allocator + source comparator로 tmp tree 생성]
        ├─ comparison/allocation throw
        │    → tmp rollback
        │    → target unchanged
        └─ success
             ↓
        [comparator exchange]
             ├─ tested assignment throw
             │    → physical tree unchanged
             │    → tmp destructor가 replacement tree 해제
             └─ success
                  ↓
             [root/size exchange + header refresh]
                  ↓
             tmp가 old target tree를 해제
```

### Public swap

```text
[comparator exchange]
    ├─ tested assignment throw → 둘 다 original tree/allocator 유지
    └─ success
         ↓
[value allocator + node allocator exchange]
         ↓
[root + size exchange]
         ↓
[각 header refresh]
```

## Ownership ledger

| 자원/상태 | owner | failure 뒤 요구되는 상태 |
| --- | --- | --- |
| value allocator state | map object | public allocator identity와 일치 |
| rebound node allocator | 같은 allocator family state | node allocate/deallocate ledger 공유 |
| unlinked allocation | `_create_node` local path | construct 실패 시 즉시 deallocate |
| linked node | tree root에서 도달 가능한 map | clear/destructor/erase가 회수 |
| replacement tree | temporary map | commit 전 실패 시 temporary가 전부 회수 |
| old target tree | target, commit 뒤 temporary | temporary destructor가 target allocator로 해제 |
| comparator state | map ordering policy | physical tree 이동 전에 교환 시도 |
| header root/extrema links | 각 map object | tree 이동 성공 뒤 `_refresh_header()`로 재결합 |

## 이 Thread의 경계

이 Thread는 **stateful allocator/comparator와 예외 아래 node ownership transaction**을 다룹니다.

- red-black balancing 자체는 Thread 03의 관심사입니다.
- header sentinel과 iterator topology는 Thread 04의 관심사입니다.
- vector의 contiguous lifetime transaction은 Thread 02에서 별도로 다룹니다.
- arbitrary policy type이 partial assignment 후 throw하는 경우까지 strong guarantee를 일반화하지 않습니다.
- allocator swap 자체의 예외와 allocator propagation 규칙 전반을 증명하지 않습니다.
