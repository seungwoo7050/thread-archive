# Thread 03 — Map: 비균형 BST에서 검증 가능한 red-black core까지

> 대상: `seungwoo7050/42-archive`의 `cpp/ft_container` 브랜치
> 검토 방식: 아래 커밋의 정확한 SHA에서 diff와 당시 source를 확인했습니다. 이 환경에서는 build와 test를 실행하지 않았습니다.

## 개요

초기 `map`은 comparator가 정한 순서로 node를 연결하는 ordinary binary search tree입니다. 이 상태만으로도 정렬 순회, `find`, `lower_bound`, `upper_bound` 같은 공개 결과는 맞을 수 있습니다. 그러나 정렬된 key를 차례로 넣으면 tree가 한쪽 linked list처럼 기울어져 search와 insert가 선형 시간이 됩니다.

이 Thread는 세 종류의 correctness를 분리합니다.

| 층 | 묻는 질문 | 대표 증거 |
| --- | --- | --- |
| 기능적 correctness | 순회 결과와 query가 `std::map`과 같은가? | 정렬·혼합 입력 differential test |
| 구조적 correctness | root/parent/color/black-height/reachability가 모두 유효한가? | white-box inspector + deterministic random operations |
| 점근적 correctness | 입력 순서가 나빠도 높이와 비교 횟수가 logarithmic 범위인가? | 1,024-node height/comparator upper bound |

삽입 balancing은 red parent를 해결하는 비교적 국소적인 문제입니다. 삭제 balancing은 제거된 black node가 남긴 black-height 부족을, replacement가 `NULL`인 경우까지 포함해 parent context와 함께 위로 전달해야 하므로 더 어렵습니다.

또 하나의 시점 구분이 필요합니다. 삽입·삭제 balancing 커밋의 당시 표현은 `_root == NULL`을 빈 tree로 사용합니다. 후대의 white-box test SHA에서는 병렬 Thread 04의 value-free header sentinel이 이미 들어와 `_header.parent`가 root를 가리킵니다. 이 문서는 각 SHA의 표현을 그대로 사용하며 header를 과거 balancing 코드에 소급하지 않습니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `2c6dd8acdd20` | `feat(map): 노드 소유권과 빈 tree 상태 구현` | A | `RB_TREE, ALLOCATOR, ARCH` | Establishes individually allocated nodes and single-root ownership. |
| 2 | `c0fdb8e3f84c` | `feat(map): 삽입과 첨자 및 복사 동작 구현` | B | `RB_TREE` | Adds comparator-defined BST insertion, indexing, and copying. |
| 3 | `0f70c1fcc520` | `feat(map): 삭제와 clear 및 swap 구현` | B | `RB_TREE, ALLOCATOR` | Adds ordinary BST erasure and tree exchange. |
| 4 | `f29fd8a91523` | `feat(map): 레드-블랙 삽입 회전과 색 보정 구현` | S | `CORE, RB_TREE, HARD` | Introduces rotations and insertion recoloring. |
| 5 | `8f8b67961819` | `test(map): 정렬 입력 삽입과 검색 경계 stress 검증` | B | `TEST, RB_TREE` | Exercises adversarial sorted insertion and query boundaries. |
| 6 | `a055cb19500b` | `feat(map): 레드-블랙 삭제 보정 구현` | S | `CORE, RB_TREE, HARD` | Adds deletion transplant state and double-black correction. |
| 7 | `86922f1ddfa0` | `test(map): 반복 삭제·복사·대입·교환 stress 검증` | B | `TEST, RB_TREE` | Stresses repeated erasure, copying, assignment, and swap against `std::map`. |
| 8 | `cd67e6a31bb7` | `test(map): 무작위 연산마다 레드-블랙 불변식 검증` | A | `TEST, RB_TREE, RISK` | Validates all structural red-black invariants after deterministic random operations. |
| 9 | `cd8ebbb2c01e` | `perf(map): 높이와 비교 횟수 회귀 상한 추가` | A | `PERF, RB_TREE, TEST` | Enforces red-black height and comparator-count upper bounds. |

## `2c6dd8acdd20` — node 단위 소유권과 단일 root

**중요도** `A` · **태그** `RB_TREE, ALLOCATOR, ARCH`

`include/ft_map.hpp`의 첫 표현은 각 key/value를 독립 node에 저장합니다.

```cpp
struct node
{
    value_type value;
    node* parent;
    node* left;
    node* right;

    explicit node(const value_type& v)
        : value(v), parent(NULL), left(NULL), right(NULL)
    {
    }
};
```

map은 value allocator와 node allocator, `_root`, `_size`, comparator를 보유합니다. node 생성은 allocate와 construct를 분리하고, construct가 던지면 방금 받은 block을 즉시 해제합니다.

```cpp
node* n = _node_alloc.allocate(1);
try
{
    _node_alloc.construct(n, node(value));
}
catch (...)
{
    _node_alloc.deallocate(n, 1);
    throw;
}
```

`_root` 하나에서 도달 가능한 node가 map의 소유 tree이며, recursive `_clear`는 left/right subtree를 먼저 해제한 뒤 현재 node를 destroy/deallocate합니다.

```text
map
 └─ _root
     ├─ left subtree
     └─ right subtree
```

이 표현은 vector와 달리 node별 allocation을 사용하므로 다른 node의 삽입이나 회전이 기존 `value` 객체를 옮기지 않습니다. 이는 후대 iterator 안정성의 기반이지만, 이 커밋에는 iterator와 balancing이 아직 없습니다.

이 SHA의 internal node allocator는 value allocator의 state에서 구성되지 않고 독립 default construction됩니다. stateful allocator에서 ownership identity가 끊기는 문제는 Thread 05의 `ae180871b160`이 고칩니다. 여기서는 초기 node ownership substrate만 확립됩니다.

## `c0fdb8e3f84c` — comparator가 정의하는 ordinary BST

**중요도** `B` · **태그** `RB_TREE`

삽입은 root에서 시작해 comparator를 양방향으로 호출합니다.

```cpp
if (_comp(value.first, cur->value.first))
    cur = cur->left;
else if (_comp(cur->value.first, value.first))
    cur = cur->right;
else
    return ft::make_pair(iterator(cur, _root), false);
```

두 비교가 모두 거짓이면 key가 equivalent하므로 기존 node와 `false`를 반환합니다. 새 key는 leaf 위치를 찾은 뒤 node를 만들고 parent의 left/right에 연결합니다. `operator[]`는 `mapped_type()`을 넣은 `pair`를 삽입하고 반환 iterator의 `second`를 돌려줍니다.

range/copy constructor는 source를 순회하며 insert하고, copy assignment는 기존 tree를 먼저 clear한 뒤 comparator를 대입하고 source 원소를 다시 삽입합니다. 이 시점의 copy assignment는 replacement를 미리 완성하지 않으므로 중간 예외에서 원래 target을 유지하지 못합니다. 그 문제는 balancing과 다른 failure-transaction 관심사로서 Thread 05에서 수정됩니다.

또한 당시 insert는 node allocation 뒤 attachment 방향을 정하려고 comparator를 다시 호출합니다. 그 마지막 비교가 던지면 아직 tree에 연결되지 않은 node가 owner 없이 남을 수 있으며, `cb08194d17b0`이 비교를 allocation 앞으로 옮깁니다.

이 Thread에서 이 커밋의 역할은 **공개 ordering을 만드는 비균형 BST baseline**입니다.

## `0f70c1fcc520` — balancing 없는 구조적 삭제

**중요도** `B` · **태그** `RB_TREE, ALLOCATOR`

`erase`, `clear`, `swap`과 `_transplant`, `_erase_node`가 추가됩니다. 삭제는 세 경우로 나뉩니다.

```text
1. left 없음  → target을 right child로 대체
2. right 없음 → target을 left child로 대체
3. 두 child   → right subtree의 minimum(successor)을 target 위치로 이동
```

핵심 helper는 parent가 가리키던 link 또는 root를 replacement로 바꾸고, replacement의 parent를 갱신합니다.

```cpp
void _transplant(node* old_node, node* new_node)
{
    if (old_node->parent == NULL)
        _root = new_node;
    else if (old_node == old_node->parent->left)
        old_node->parent->left = new_node;
    else
        old_node->parent->right = new_node;

    if (new_node)
        new_node->parent = old_node->parent;
}
```

두 child가 있는 경우 successor의 `value`를 target에 대입하지 않고 **successor node 자체를 이식**합니다.

```cpp
node* successor = _minimum(target->right);

if (successor->parent != target)
{
    _transplant(successor, successor->right);
    successor->right = target->right;
    successor->right->parent = successor;
}

_transplant(target, successor);
successor->left = target->left;
successor->left->parent = successor;
```

map의 key는 `const Key`이므로 target의 `pair<const Key, T>`에 successor 값을 대입하는 방식은 적합하지 않습니다. node를 물리적으로 옮기면 삭제 대상 외의 value 객체를 재구성하지 않아도 됩니다.

아직 color가 없으므로 이 삭제는 BST ordering만 보존합니다. red-black tree에 그대로 적용하면 black-height가 깨질 수 있으며, `a055cb19500b`이 삭제 당시 node color와 replacement context를 추가합니다.

## `f29fd8a91523` — red-black 삽입: 새 node를 red로 시작하는 이유

**중요도** `S` · **태그** `CORE, RB_TREE, HARD`

node에 `bool red`가 추가되고 새 node는 red로 시작합니다.

```cpp
explicit node(const value_type& v)
    : value(v), parent(NULL), left(NULL), right(NULL), red(true)
{
}
```

새 leaf를 black으로 넣으면 그 경로의 black node 수가 즉시 하나 늘어 모든 root-to-null 경로의 black height가 달라집니다. red로 넣으면 black height는 유지되고, 위 parent도 red인 경우에만 “red node의 child는 black” 규칙을 복구하면 됩니다.

빈 tree의 첫 root는 즉시 black으로 바뀝니다.

```cpp
_root = _create_node(value);
_root->red = false;
```

### 회전이 보존해야 하는 것

left rotation은 `x->right`인 `y`를 subtree root로 올립니다.

```cpp
node* y = x->right;
x->right = y->left;

if (y->left)
    y->left->parent = x;

y->parent = x->parent;

if (x->parent == NULL)
    _root = y;
else if (x == x->parent->left)
    x->parent->left = y;
else
    x->parent->right = y;

y->left = x;
x->parent = y;
```

right rotation은 대칭입니다. 회전은 key의 in-order 순서를 바꾸지 않으면서 subtree 높이와 parent relation을 바꿉니다. 따라서 다음 네 연결을 모두 갱신해야 합니다.

1. 회전 축 안쪽 child
2. 올라가는 node의 이전 parent
3. container root 또는 parent의 child link
4. 내려가는 node의 parent

한 곳이라도 빠지면 sorted traversal이 우연히 맞더라도 parent climb iterator나 후속 balancing이 잘못된 구조를 읽습니다.

### 삽입 보정의 두 계열

보정은 `z`의 parent가 red인 동안 반복합니다. parent가 grandparent의 left child인 경우를 기준으로 보면 다음과 같습니다.

#### red uncle: 색을 위로 전달

```text
        G(black)                  G(red)
       /        \                /      \
   P(red)      U(red)   →    P(black) U(black)
     /
   z(red)
```

parent와 uncle을 black, grandparent를 red로 바꾸고 `z = grandparent`로 올라갑니다. local red-red는 해소되지만 grandparent와 그 parent가 새 red-red가 될 수 있으므로 반복합니다.

#### black uncle: inner를 outer로 만든 뒤 회전

`z`가 parent의 right child인 left-right 형태라면 먼저 parent를 left rotation합니다. 그 뒤 parent를 black, grandparent를 red로 바꾸고 grandparent를 right rotation합니다.

```text
inner:                outer 변환:           최종:
      G                    G                    z
     /                    /                    / \
    P      --L(P)-->      z      --R(G)-->    P   G
     \                  /
      z                P
```

실제 구현은 left/right를 대칭으로 처리하며 마지막에 root를 black으로 강제합니다.

```cpp
if (_root)
    _root->red = false;
```

### 이 커밋이 세운 불변 조건

- root는 black입니다.
- `NULL` leaf는 black으로 취급됩니다.
- red node의 parent와 child는 black입니다.
- 모든 node에서 descendant null까지의 black node 수가 같습니다.
- rotation 뒤에도 BST ordering과 parent/root link가 유지됩니다.

이 SHA는 삽입만 보정합니다. ordinary deletion은 black node를 제거할 수 있으므로 별도의 double-black recovery가 필요합니다.

## `8f8b67961819` — 정렬 입력에서 공개 결과를 비교

**중요도** `B` · **태그** `TEST, RB_TREE`

테스트는 key `1..96`을 오름차순과 내림차순으로 각각 삽입하고 `std::map`과 다음 결과를 비교합니다.

- insertion 성공 여부와 반환 iterator key
- 전체 sorted iteration
- `find`
- `lower_bound`
- `upper_bound`
- `equal_range`

query key는 `-3`부터 `135`까지 7씩 증가시켜 존재/부재와 양쪽 경계를 섞습니다.

정렬 입력은 ordinary BST를 가장 쉽게 한쪽으로 기울게 만드는 적대적 입력입니다. 그러나 이 테스트가 확인하는 것은 결과 parity입니다. balancing을 제거해 선형 tree가 되어도 96개 범위에서 값이 맞으면 통과할 수 있습니다. 따라서 이 커밋은 공개 기능의 stress evidence이며, red-black 색 규칙이나 logarithmic height를 직접 증명하지 않습니다.

## `a055cb19500b` — red-black 삭제: `NULL` replacement의 parent를 따로 운반

**중요도** `S` · **태그** `CORE, RB_TREE, HARD`

red node를 제거하면 black height는 바뀌지 않습니다. black node를 제거하면 해당 경로에서 black 하나가 사라지므로 replacement가 “추가 black”을 가진 것처럼 보정해야 합니다.

삭제 구현은 세 상태를 따로 기록합니다.

```cpp
node* moved = target;
node* fix = NULL;
node* fix_parent = NULL;
bool moved_was_red = moved->red;
```

- `moved`: 실제로 target 위치를 대체하거나 tree에서 빠져나가는 물리 node
- `moved_was_red`: 제거된 물리 위치의 원래 색
- `fix`: black-height 부족을 떠안는 replacement child
- `fix_parent`: `fix == NULL`이어도 sibling 방향을 찾기 위한 parent

### 두 child가 있는 삭제

successor를 target 위치로 옮길 때 제거된 색은 target 색이 아니라 successor가 원래 있던 위치의 색입니다.

```cpp
moved = _minimum(target->right);
moved_was_red = moved->red;
fix = moved->right;
```

successor가 target의 바로 오른쪽 child인지 더 아래에 있는지에 따라 `fix_parent`가 다릅니다.

```cpp
if (moved->parent == target)
{
    fix_parent = moved;
    if (fix)
        fix->parent = moved;
}
else
{
    fix_parent = moved->parent;
    _transplant(moved, moved->right);
    moved->right = target->right;
    moved->right->parent = moved;
}
```

그 뒤 successor node를 target 위치로 이식하고 target의 left subtree와 color를 이어받게 합니다.

```cpp
_transplant(target, moved);
moved->left = target->left;
moved->left->parent = moved;
moved->red = target->red;
```

target node를 파괴한 뒤, 원래 tree에서 빠져나간 `moved`가 black이었을 때만 fixup을 실행합니다.

```cpp
_destroy_node(target);
--_size;

if (!moved_was_red)
    _erase_fixup(fix, fix_parent);

if (_root)
    _root->red = false;
```

### 왜 `fix_parent`가 별도인가

replacement child가 실제 node라면 `fix->parent`를 읽을 수 있습니다. 하지만 삭제에서 흔한 replacement는 `NULL`입니다. null pointer에는 parent field가 없으므로 다음을 판단할 수 없습니다.

- double-black 위치가 parent의 left인지 right인지
- sibling이 어느 쪽인지
- double-black을 위로 올릴 때 새 parent가 누구인지

그래서 `(x, parent)` 쌍으로 state를 전달합니다. 이후 header sentinel이 들어오는 시점과 무관하게, null replacement의 parent context가 필요하다는 삭제 알고리즘의 핵심은 같습니다.

### fixup의 네 경우

아래는 `x`가 parent의 left child인 경우이며 오른쪽은 대칭입니다.

#### 1. sibling이 red

sibling을 black, parent를 red로 바꾸고 parent를 left rotation합니다. 이 변환은 새로운 sibling을 black으로 만들어 나머지 경우로 환원합니다.

```cpp
if (_is_red(sibling))
{
    sibling->red = false;
    parent->red = true;
    _rotate_left(parent);
    sibling = parent->right;
}
```

#### 2. sibling과 두 child가 모두 black

sibling을 red로 바꾸고 추가 black을 parent로 올립니다.

```cpp
if (_is_black(sibling ? sibling->left : NULL)
    && _is_black(sibling ? sibling->right : NULL))
{
    if (sibling)
        sibling->red = true;
    x = parent;
    parent = x ? x->parent : NULL;
}
```

#### 3. far child가 black이고 near child가 red

near child를 black, sibling을 red로 바꾼 뒤 sibling을 right rotation해 far-red 형태로 바꿉니다.

#### 4. far child가 red

sibling이 parent color를 받고, parent와 far child를 black으로 바꾼 뒤 parent를 left rotation합니다. black-height 부족이 local에서 해소되므로 `x = _root`로 반복을 끝냅니다.

```cpp
if (sibling)
    sibling->red = parent ? parent->red : false;
if (parent)
    parent->red = false;
if (sibling && sibling->right)
    sibling->right->red = false;
if (parent)
    _rotate_left(parent);

x = _root;
parent = NULL;
```

loop 뒤 실제 replacement node가 있으면 black으로 만듭니다.

```cpp
if (x)
    x->red = false;
```

### 보장과 한계

이 커밋은 ordinary transplant를 유지하면서 제거된 black의 영향을 복구합니다. insert와 erase 뒤 root를 black으로 만들고, null을 black leaf로 취급합니다.

그러나 공개 결과만 비교해서는 색이나 black-height가 맞는지 알 수 없습니다. 다음 두 test 층이 각각 functional stress와 structural invariant를 검증합니다.

## `86922f1ddfa0` — 반복 mutation의 differential stress

**중요도** `B` · **태그** `TEST, RB_TREE`

테스트는 다음 작업을 `std::map`과 나란히 수행합니다.

- ascending/descending tree의 copy construction
- assignment
- member `swap`
- 혼합된 32개 key 삽입
- 매 세 번째 key 삭제
- 남은 tree가 빌 때까지 `begin()`과 마지막 원소를 번갈아 iterator erase
- 각 단계의 전체 순회와 query 결과 비교

반복적으로 root, leaf, one-child, two-child 삭제가 섞이게 만들고, 복사·교환 뒤에도 public 상태가 일치하는지 확인합니다. 하지만 red-red나 black-height 불일치가 즉시 결과 오류로 드러나지 않으면 이 테스트만으로는 탐지할 수 없습니다.

## `cd67e6a31bb7` — 출력 뒤에 숨은 tree 구조를 직접 검사

**중요도** `A` · **태그** `TEST, RB_TREE, RISK`

이 SHA에서는 병렬 Thread 04의 value-free header sentinel이 이미 존재합니다. 따라서 inspector는 초기 balancing SHA의 `_root` 표현이 아니라 당시의 `_header` 구조를 검사합니다. 이를 과거 커밋에 소급하지 않습니다.

`map`은 test 전용 `ft::detail::map_inspector`에 friend access를 허용하고, inspector는 다음을 검증합니다.

### header와 root

- header의 `is_header` flag가 참인가
- header가 black인가
- 빈 map이면 `parent == NULL`, `left == header`, `right == header`인가
- non-empty면 root가 존재하고 `root->parent == header`인가
- root가 black인가
- `header.left/right`가 실제 minimum/maximum과 같은가

### 각 subtree

- child의 parent가 예상 parent와 같은가
- lower/upper bound에 따라 strict BST ordering을 지키는가
- red node가 red child를 갖지 않는가
- left/right black height가 같은가
- reachable node 수가 `_size`와 같은가
- header가 일반 child로 도달되지 않는가

재귀 결과는 node count, height, black height까지 계산합니다.

테스트 workload는 세 층입니다.

1. 고정 삽입/삭제 sequence마다 invariant 검사
2. key `0..95`를 넣고 **현재 root key를 반복 삭제**하면서 매 단계 검사
3. 고정 seed `0x5EED1234`로 3,000회 insert, `operator[]`, key/iterator erase, query, copy, assignment, swap, occasional clear를 수행하고 매 단계 두 map의 public 결과와 내부 invariant 검사

실패하면 seed, step, operation prefix를 출력하도록 구성되어 무작위 test를 재현 가능한 deterministic regression으로 만듭니다.

이 test는 functional parity와 구조적 correctness를 동시에 확인합니다. 다만 valid red-black tree라도 구현의 비교 횟수가 비정상적으로 많거나 불필요한 전체 순회를 할 수 있으므로, complexity upper bound는 다음 커밋이 별도로 맡습니다.

## `cd8ebbb2c01e` — timing 대신 구조 높이와 comparator 횟수에 상한 설정

**중요도** `A` · **태그** `PERF, RB_TREE, TEST`

`tests/test_complexity.cpp`는 comparator 호출마다 counter를 증가시키는 `counting_less`와 inspector의 `height`를 사용합니다. wall-clock time은 machine load와 compiler 최적화에 민감하지만, 높이와 비교 횟수는 구현의 구조를 직접 반영합니다.

각 scenario는 1,024개 key를 다음 순서로 삽입합니다.

- ascending
- descending
- 고정 seed shuffle

검사 상한은 source에 명시되어 있습니다.

```cpp
const size_type logarithm = ceil_log2(values.size() + 1);
const size_type height_limit = 2 * logarithm;

require(validation.height <= height_limit,
    label + " red-black height limit");
```

red-black tree의 높이 상한을 `2 * ceil(log2(n + 1))`로 두고, 전체 삽입 비교 횟수와 각 `find` 비교 횟수도 느슨하지만 logarithmic한 식으로 제한합니다.

```cpp
const size_type insertion_limit =
    values.size() * (4 * logarithm + 4);

require(insertion_comparisons <= insertion_limit,
    label + " insertion comparison limit");

require(comparisons <= 2 * validation.height + 2,
    label + " find comparison limit");
```

존재하는 모든 key와 범위 밖 missing key 두 개를 조회합니다. 따라서 balancing을 제거해 ascending/descending tree가 높이 1,024에 가까워지거나 search가 선형화되면 공개 결과가 맞더라도 이 test는 실패하도록 설계됩니다.

이 상한은 이론적 최솟값이나 정확한 연산 수를 요구하지 않습니다. 의도는 micro-optimization이 아니라 **점근적 회귀를 안정적으로 차단하는 것**입니다.

## 불변 조건이 형성된 순서

| 불변 조건 | 처음 표현된 시점 | 보강·검증 |
| --- | --- | --- |
| root에서 모든 node가 도달 가능하며 각 node는 한 번만 파괴됨 | `2c6dd8acdd20` | randomized inspector의 reachable count와 size |
| comparator가 strict ordering과 key equivalence를 결정함 | `c0fdb8e3f84c` | differential query, inspector lower/upper bound |
| 삭제는 const key 값을 대입하지 않고 node를 이식함 | `0f70c1fcc520` | repeated key/iterator/root erase |
| root black, red-red 금지, 동일 black height | `f29fd8a91523`, `a055cb19500b` | `cd67e6a31bb7` |
| null replacement도 parent context와 함께 보정됨 | `a055cb19500b` | 고정 deletion sequence와 repeated current-root erase |
| tree height와 비교 횟수가 logarithmic 범위에 남음 | balancing commits | `cd8ebbb2c01e` |

## 세 test 층이 각각 증명하는 것

| Test | 증명하는 것 | 증명하지 않는 것 |
| --- | --- | --- |
| `8f8b...`, `86922...` | 대표적 적대 입력과 mutation 뒤 public 결과가 `std::map`과 일치 | 내부 color, parent link, black height, 실제 높이 |
| `cd67...` | 매 operation 뒤 구조와 public 결과가 모두 유효 | 모든 가능한 operation sequence, exact complexity |
| `cd8e...` | 세 입력 순서에서 height/comparator count가 명시 상한 이하 | 실제 시간, cache 성능, 모든 key distribution |

## 최종 상태

red-black map의 mutation 흐름은 다음처럼 정리됩니다.

```text
INSERT
[comparator search]
    → [new red node link]
    → parent black? 종료
    → parent red?
        ├─ uncle red: recolor, grandparent로 상승
        └─ uncle black: inner rotation → recolor → outer rotation
    → root black

ERASE
[target 또는 successor를 물리적으로 transplant]
    → [실제로 제거된 위치의 color 기록]
    → red 제거: 종료
    → black 제거:
        (replacement node 또는 NULL, explicit parent)로 fixup
        ├─ red sibling을 회전해 black sibling case로 변환
        ├─ black sibling + black children: 부족분을 parent로 전달
        └─ near/far child 회전·재색으로 local 부족분 해소
    → root black
```

이 Thread의 가장 중요한 결론은 “정렬된 출력”과 “red-black tree”가 같은 주장이 아니라는 점입니다. public sequence가 맞아도 tree는 비균형이거나 색·parent link가 손상될 수 있습니다. 그래서 differential test, white-box invariant 검사, complexity upper bound가 모두 필요합니다.

header sentinel을 통한 `end()` 안정성은 Thread 04, stateful allocator와 comparator 예외 transaction은 Thread 05에서 별도로 다룹니다.
