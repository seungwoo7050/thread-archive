# Thread 04 — 구조 변경을 견디는 Map iterator

> 대상: `seungwoo7050/42-archive`의 `cpp/ft_container` 브랜치
> 검토 방식: 아래 커밋의 정확한 SHA에서 diff와 당시 source를 확인했습니다. 이 환경에서는 build와 test를 실행하지 않았습니다.

## 개요

초기 map iterator는 현재 node와 **iterator를 만들 당시의 root pointer**를 함께 저장합니다. 정적인 tree에서는 이 정보만으로 successor와 predecessor를 찾고, null로 표현된 `end()`에서 `--`하여 최대 node로 돌아갈 수 있습니다.

문제는 root가 container의 고정 주소가 아니라 tree mutation의 결과라는 점입니다. red-black 회전은 root를 바꿀 수 있고, root 삭제는 다른 node를 root로 올리며, `swap`은 tree 전체를 다른 container로 넘깁니다. 기존 element node는 살아 있어도 iterator 안의 copied root는 과거 상태가 됩니다. 따라서 “element iterator가 가리키는 node는 유효하다”와 “그 iterator가 현재 container의 끝까지 올바르게 순회한다”는 서로 다른 보장입니다.

`15a8460ccdfe`는 이 문제를 value가 없는 header sentinel로 바꿉니다.

```text
빈 map
header.parent = NULL
header.left   = &header
header.right  = &header

비어 있지 않은 map
header.parent = root
header.left   = minimum
header.right  = maximum
root.parent   = &header
```

`end()`는 null이나 copied root가 아니라 container 내부 header의 주소입니다. element iterator는 node 주소만 저장하고 parent link를 따라 현재 owner의 header에 도달합니다. insert, erase, clear, swap 뒤에는 header가 root와 양 끝 node를 다시 가리키도록 갱신됩니다. 이로써 root 교체와 element identity를 분리하고, sentinel을 만들기 위해 `Key()`를 생성할 필요도 없앱니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4bbf81cecef4` | `feat(map): 가변 반복자와 tree 순회 구현` | B | `RB_TREE, ITERATOR` | Implements traversal with a null end state plus a copied root pointer. |
| 2 | `50e62b0e0298` | `feat(map): 상수와 역방향 반복자 구현` | B | `RB_TREE, ITERATOR` | Extends that model to const and reverse iteration. |
| 3 | `048ada8fe1c5` | `feat(map): 가변·상수 반복자 상호 비교 지원` | B | `ITERATOR, PUBLIC_API` | Adds mutable/const comparison interoperability. |
| 4 | `15a8460ccdfe` | `fix(map): 값 없는 header로 끝 반복자 상태 안정화` | S | `ARCH, ITERATOR, RB_TREE` | Replaces the root snapshot with a value-free, container-owned header sentinel. |
| 5 | `81d8c4489c16` | `test(map): 회전·삭제·교환 뒤 반복자 상태 검증` | A | `TEST, ITERATOR, RB_TREE` | Verifies saved end positions, element iterators, swap migration, empty reset, and non-default keys. |

## `4bbf81cecef4` — null end와 root snapshot으로 시작한 순회

**중요도** `B` · **태그** `RB_TREE, ITERATOR`

첫 mutable iterator는 두 pointer를 보유합니다.

```cpp
node* _node;
node* _root;

iterator(node* n, node* r)
    : _node(n), _root(r)
{
}
```

`begin()`은 현재 root의 minimum을 가리키고 `end()`는 `_node == NULL`인 iterator입니다.

```cpp
iterator begin() { return iterator(_minimum(_root), _root); }
iterator end()   { return iterator(NULL, _root); }
```

일반 node에서 `++`는 다음 규칙을 사용합니다.

```text
right child가 있음 → right subtree의 minimum
right child가 없음 → 자신이 left child가 되는 첫 ancestor까지 상승
그 ancestor도 없음 → NULL(end)
```

`--`는 대칭이며, null end에서만 copied root가 필요합니다.

```cpp
iterator& operator--()
{
    if (_node == NULL)
        _node = _maximum(_root);
    else
        _node = _previous(_node);
    return *this;
}
```

이 구현은 tree가 변하지 않는 동안에는 충분합니다. node 자체가 살아 있으면 `++`와 `--`는 parent/child link를 읽어 현재 topology를 따라갑니다. 그러나 saved `end()`는 node가 아니라 null과 **과거 root**의 조합입니다.

예를 들어 다음 순서가 가능합니다.

```text
1 삽입 → root = node(1), end가 root(1)를 snapshot
2, 3 삽입 → balancing으로 root = node(2)
saved_end-- → snapshot root(1)의 subtree maximum을 계산
```

새 maximum은 3이지만 copied root 1의 현재 subtree만 보면 1이거나 다른 국소 범위가 될 수 있습니다. iterator가 root의 소유자처럼 topology snapshot을 들고 있는 표현이 limitation의 원인입니다.

## `50e62b0e0298` — const와 reverse가 같은 end 모델을 상속

**중요도** `B` · **태그** `RB_TREE, ITERATOR`

`const_iterator`가 추가되고 mutable iterator에서 const iterator로 변환할 수 있게 됩니다. dereference 형식만 const이고 내부 표현은 똑같이 `_node`와 `_root`를 저장합니다.

```cpp
const_iterator(const iterator& other)
    : _node(other._node), _root(other._root)
{
}
```

map은 공용 reverse adaptor를 사용합니다.

```cpp
typedef ft::reverse_iterator<iterator> reverse_iterator;
typedef ft::reverse_iterator<const_iterator> const_reverse_iterator;

reverse_iterator rbegin() { return reverse_iterator(end()); }
reverse_iterator rend()   { return reverse_iterator(begin()); }
```

reverse iterator의 dereference는 base iterator를 한 칸 감소시키므로 `rbegin()`은 결국 `--end()`에 의존합니다. 따라서 root snapshot 문제는 mutable forward iterator만의 문제가 아니라 const/reverse surface 전체로 퍼집니다. 이 커밋은 필요한 API를 확장하지만 end-state 표현은 바꾸지 않습니다.

## `048ada8fe1c5` — mutable/const 상호 비교

**중요도** `B` · **태그** `ITERATOR, PUBLIC_API`

mutable iterator와 const iterator의 `==`, `!=`를 양방향으로 허용합니다. 비교 기준은 node identity입니다.

```cpp
bool operator==(const iterator& other) const
{
    return _node == other._node;
}

friend bool operator==(const iterator& lhs,
    const const_iterator& rhs)
{
    return const_iterator(lhs) == rhs;
}
```

같은 element를 가리키는 두 iterator는 cv 차이와 무관하게 같습니다. 다만 root snapshot은 equality에 포함되지 않습니다. 두 `end()` 모두 `_node == NULL`이면 copied root가 달라도 같다고 판단하므로, 비교 성공이 `--end()`의 correctness를 증명하지는 않습니다.

후대 sentinel refactor에서는 이 중복 overload가 iterator가 가진 단일 `node_base*`를 비교하는 template 연산으로 단순화됩니다.

## `15a8460ccdfe` — value-free header sentinel로 end-state를 재정의

**중요도** `S` · **태그** `ARCH, ITERATOR, RB_TREE`

### 기존 표현의 실패 원인

기존 iterator가 저장한 `_root`는 element identity가 아니라 container의 가변 topology입니다.

| 구조 변경 | element node 주소 | root snapshot | 결과 |
| --- | --- | --- | --- |
| red-black rotation | 유지 | stale 가능 | saved `end()`의 `--`가 잘못된 maximum을 찾을 수 있음 |
| root erase | 삭제 대상 외 node는 유지 | 삭제된 root를 가리킬 수 있음 | stale/dangling root |
| swap | node는 다른 container로 이동 | 원래 container의 root | element iterator가 새 owner의 end에 도달하지 못할 수 있음 |
| clear | node 전부 제거 | 과거 root | 새 empty state와 무관 |

해결은 iterator가 root를 복사하지 않게 하고, container가 변하지 않는 주소에 end-state를 두는 것입니다.

### value를 가진 node와 sentinel을 분리

`node_base`는 링크와 color/header 표식만 갖고, 실제 node만 `value_type`을 가집니다.

```cpp
struct node_base
{
    node_base* parent;
    node_base* left;
    node_base* right;
    bool red;
    bool is_header;

    explicit node_base(bool header = false)
        : parent(NULL),
          left(header ? this : NULL),
          right(header ? this : NULL),
          red(false),
          is_header(header)
    {
    }
};

struct node : node_base
{
    value_type value;

    explicit node(const value_type& v)
        : node_base(false), value(v)
    {
        this->red = true;
    }
};
```

sentinel에 `value_type`이 없으므로 빈 `map<Key, T>`를 만들 때 `Key()`나 `T()`를 요구하지 않습니다. 이는 default constructor가 없는 key도 map의 key로 사용할 수 있게 하는 표현상의 필요 조건입니다.

dereference할 때만 실제 node로 downcast합니다.

```cpp
reference operator*() const
{
    return static_cast<node*>(_node)->value;
}
```

`end()`를 dereference하는 것은 여전히 유효하지 않은 사용이지만, sentinel 자체를 만들기 위해 fake value를 생성하지 않습니다.

### iterator는 current node 하나만 보유

```cpp
node_base* _node;

explicit iterator(node_base* n)
    : _node(n)
{
}
```

public endpoints는 header links로 정의됩니다.

```cpp
iterator begin() { return iterator(_header.left); }
iterator end()   { return iterator(&_header); }
```

빈 상태에서는 `header.left == header.right == &header`이므로 `begin() == end()`가 자연스럽습니다.

### successor와 predecessor가 header를 이해

```cpp
static node_base* _next(node_base* n)
{
    if (n->is_header)
        return n;
    if (n->right)
        return _minimum(n->right);

    node_base* parent = n->parent;
    while (!parent->is_header && n == parent->right)
    {
        n = parent;
        parent = parent->parent;
    }
    return parent;
}
```

maximum에서 `++`하면 ancestor를 따라 올라가 header에 도달합니다. 반대로 header에서 `--`하면 현재 maximum을 반환합니다.

```cpp
static node_base* _previous(node_base* n)
{
    if (n->is_header)
        return n->right;
    if (n->left)
        return _maximum(n->left);

    node_base* parent = n->parent;
    while (!parent->is_header && n == parent->left)
    {
        n = parent;
        parent = parent->parent;
    }
    return parent;
}
```

saved `end()`가 보유한 주소는 root가 아니라 map 객체 내부 `_header`의 주소입니다. 회전으로 root가 바뀌어도 header 주소는 바뀌지 않고, `--saved_end`는 실행 시점의 `header.right`를 읽습니다.

### mutation 뒤 header를 다시 연결

```cpp
void _reset_header()
{
    _header.parent = NULL;
    _header.left = &_header;
    _header.right = &_header;
    _header.red = false;
    _header.is_header = true;
}

void _refresh_header()
{
    node_base* root = _root();
    if (root == NULL)
    {
        _reset_header();
        return;
    }
    root->parent = &_header;
    _header.left = _minimum(root);
    _header.right = _maximum(root);
}
```

insert와 erase 뒤에는 extrema와 root-parent 관계를 refresh하고, clear는 empty self-links를 복원합니다. swap은 두 root pointer를 교환한 뒤 각 container에서 `_refresh_header()`를 호출합니다.

```text
before swap
left.header  → tree A
right.header → tree B

root pointer 교환

refresh
left.header  → tree B, B.root.parent = left.header
right.header → tree A, A.root.parent = right.header
```

element iterator는 node 주소를 계속 보유합니다. swap 뒤 해당 node의 ancestor chain은 새 owner의 header에 연결되므로 `++`를 반복하면 새 owner의 `end()`에 도달합니다.

여기서 구분해야 할 범위가 있습니다.

- **element iterator**: erase된 element가 아니라면 회전과 swap 뒤에도 같은 node를 가리키며 새 parent links로 순회합니다.
- **saved end iterator**: 같은 container 객체의 header 주소를 보유하므로 그 container의 회전·root erase 뒤 최신 maximum을 봅니다.
- 다른 container로 tree가 이동한 뒤, swap 전에 저장한 end iterator 자체가 element iterator처럼 새 owner로 “이동”하는 것은 이 테스트의 계약이 아닙니다. end는 container-owned sentinel입니다.

### 수정된 불변 조건

1. `end()`는 container 내부 value-free header를 가리킵니다.
2. 빈 map에서 header는 root가 없고 양 extrema link가 자기 자신입니다.
3. 비어 있지 않으면 header는 root/minimum/maximum을 가리키고 root의 parent는 header입니다.
4. iterator는 topology snapshot을 저장하지 않고 node identity만 저장합니다.
5. 회전은 node를 재할당하지 않으며 parent/child link만 변경합니다.
6. mutation이 끝나기 전에 header links가 현재 tree와 다시 일치해야 합니다.

## `81d8c4489c16` — stale-root 시나리오를 직접 고정한 test

**중요도** `A` · **태그** `TEST, ITERATOR, RB_TREE`

이 커밋은 일반 순회 결과만 비교하지 않고, 이전 표현이 실패하던 구조 변경을 각각 직접 만듭니다.

### 저장한 end와 root 변경

```cpp
values.insert(ft::make_pair(1, 10));
ft::map<int, int>::iterator saved_end = values.end();
values.insert(ft::make_pair(2, 20));
values.insert(ft::make_pair(3, 30));
--saved_end;
require(saved_end->first == 3, ...);
```

1, 2, 3의 순차 삽입은 rotation을 유도합니다. test는 saved end가 copied root가 아니라 current header maximum을 사용하는지 확인합니다.

root key 2를 지운 뒤에도 기존 saved end에서 `--`하여 3을 얻고, 반복 삭제 뒤 새 end에서 1을 얻습니다. `clear` 뒤에는 `begin() == end()`여야 합니다.

### element iterator와 rotation

key 1의 iterator를 저장한 뒤 2, 3을 삽입합니다. 저장 iterator는 여전히 key 1을 dereference하고, `++`로 2 → 3 → header end까지 갑니다. 이는 rotation이 node identity를 바꾸지 않고 parent links만 고쳤음을 확인합니다.

### swap 뒤 새 owner의 end

left tree의 key 2 iterator와 right tree의 key 8 iterator를 저장한 뒤 swap합니다. 두 iterator는 원래 value를 계속 가리키고, 증가시키면 각각 tree를 새로 소유한 container의 `end()`와 같아야 합니다.

이 case는 다음 두 사실을 함께 요구합니다.

```text
node 주소 유지
+
swapped root의 parent를 새 header로 재연결
```

### default constructor가 없는 key

private default constructor를 가진 key type으로 빈 map을 만들고 element를 삽입합니다. 이 test가 compile되는 것 자체가 header가 `value_type`을 포함하지 않는다는 증거입니다.

### 증명 범위

이 test가 증명하는 것:

- rotation/root erase 뒤 saved end의 `--`
- rotation 뒤 element iterator identity와 순회
- swap 뒤 element iterator가 새 owner의 end에 도달
- clear 뒤 empty header reset
- value-free sentinel과 non-default-constructible key
- mutable/const 비교의 기본 동작

증명하지 않는 것:

- erase된 element를 가리키던 iterator의 유효성
- container가 파괴된 뒤 iterator의 유효성
- 서로 무관한 map의 iterator 비교에 의미가 있다는 것
- 모든 가능한 mutation sequence
- 실제 test 실행 결과; 이 문서는 source와 assertions를 검토했을 뿐 실행하지 않았습니다.

## 최종 iterator 모델

```text
map object
  └─ _header (주소 고정, value 없음)
      ├─ parent → current root
      ├─ left   → current minimum
      └─ right  → current maximum

element iterator
  └─ _node → concrete node

++:
  right subtree minimum
  또는 parent를 따라 header까지 상승

--:
  header이면 header.right
  아니면 left subtree maximum
  또는 parent를 따라 상승
```

| 연산 | node identity | header가 해야 할 일 | iterator 결과 |
| --- | --- | --- | --- |
| insert + rotation | 기존 node 유지 | root/extrema refresh | 기존 element iterator 유지 |
| erase(non-target nodes) | 나머지 node 유지 | root/extrema refresh | 삭제 대상 외 iterator 유지 |
| clear | node 전부 파괴 | self-linked empty reset | 기존 element iterator 전부 무효 |
| swap | node allocation 그대로 이동 | 각 root를 새 header에 재결합 | element iterator는 새 owner까지 순회 |
| `--end()` | 해당 없음 | current maximum 보유 | mutation 뒤 최신 maximum |

## 이 Thread의 경계

이 Thread는 map의 **iterator end-state와 구조 변경 뒤 순회 안정성**만 다룹니다.

- red-black color와 black-height 복구는 Thread 03의 문제입니다.
- allocator/comparator exception transaction은 Thread 05의 문제입니다.
- header가 독립 public header로 compile되고 multi-TU에서 링크되는지는 Thread 06에서 검증합니다.
- element를 erase한 iterator를 살리는 기능은 제공하지 않으며, 그런 iterator는 무효입니다.
