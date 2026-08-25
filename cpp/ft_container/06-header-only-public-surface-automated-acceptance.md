# Thread 06 — Header-only 공개 표면과 자동화된 acceptance

> 대상: `seungwoo7050/42-archive`의 `cpp/ft_container` 브랜치
> 검토 방식: 아래 커밋의 정확한 SHA에서 diff와 당시 source를 확인했습니다. 이 환경에서는 build, test, sanitizer, CI를 실행하지 않았습니다.

## 개요

header-only library는 repository 내부의 하나의 큰 test translation unit에서 잘 동작해도 실제 consumer에게는 깨질 수 있습니다.

- 특정 header를 먼저 include해야만 compile될 수 있습니다.
- component header가 필요한 standard/project dependency를 우연히 다른 header에서 받아올 수 있습니다.
- template 또는 header-defined function이 여러 translation unit에서 사용될 때 link 문제가 생길 수 있습니다.
- 한 compiler에서만 허용되는 C++98 확장이나 warning이 숨어 있을 수 있습니다.
- functional test는 통과하지만 memory lifetime 또는 undefined behavior가 남을 수 있습니다.

이 Thread는 acceptance boundary를 단계적으로 넓힙니다.

```text
aggregate header
    ↓
aggregate를 통한 기존 test
    ↓
map까지 포함한 최종 public surface
    ↓
각 public header 독립 compile
    ↓
여러 translation unit의 실제 consumer link/run
    ↓
전체 acceptance를 ASan/UBSan으로 재실행
    ↓
compiler/platform matrix와 sanitizer CI
```

각 단계는 앞 단계를 반복하는 것이 아니라 다른 failure class를 겨냥합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-established role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `80e169e83212` | `feat(headers): 공용 도구와 순차 컨테이너 통합 헤더 추가` | B | `PUBLIC_API, ARCH` | Introduces the aggregate header. |
| 2 | `3c64a69dd252` | `test(headers): 통합 헤더의 순차 컨테이너 표면 검증` | C | `TEST, PUBLIC_API` | Confirms the current sequential surface compiles through that bundle. |
| 3 | `112af1753538` | `feat(map): 관계 연산과 통합 공개 헤더 완성` | B | `PUBLIC_API, RB_TREE` | Adds map to the aggregate public entry point. |
| 4 | `d938c0079994` | `test(headers): 공개 헤더를 각각 독립 compile` | B | `TEST, PUBLIC_API, CXX98` | Compiles every public header independently as the first include. |
| 5 | `072c49832ddc` | `test(consumer): 다중 번역 단위 공개 헤더 사용 검증` | B | `TEST, PUBLIC_API, INTEGRATION` | Links independent vector and map consumers in multiple translation units. |
| 6 | `1be03ae8daef` | `build(makefile): 격리된 sanitizer 검사 대상 추가` | B | `PUBLIC_API, PRACTICAL, RISK` | Runs the complete acceptance surface under isolated ASan/UBSan instrumentation. |
| 7 | `228f457988be` | `ci: compiler 행렬과 sanitizer 검사 구성` | B | `CXX98, PUBLIC_API, PRACTICAL` | Automates compiler, platform, and sanitizer checks in CI. |

## `80e169e83212` — aggregate header를 공개 진입점으로 추가

**중요도** `B` · **태그** `PUBLIC_API, ARCH`

`include/ft_containers.hpp`가 추가됩니다.

```cpp
#ifndef FT_CONTAINERS_HPP
# define FT_CONTAINERS_HPP

# include "ft_algorithm.hpp"
# include "ft_iterator.hpp"
# include "ft_pair.hpp"
# include "ft_stack.hpp"
# include "ft_type_traits.hpp"
# include "ft_vector.hpp"

#endif
```

이 SHA의 bundle에는 utility, vector, stack만 포함되고 map은 아직 없습니다. component header를 없애거나 숨기지 않고, 지원되는 조합을 한 번에 가져오는 convenience entry point를 추가합니다.

이 변화는 runtime behavior를 만들지 않습니다. public include graph에 root를 하나 추가하고, consumer가 다음과 같이 사용할 수 있게 합니다.

```cpp
#include "ft_containers.hpp"
```

다만 aggregate header가 존재한다는 것만으로 포함 목록이 완전하거나 각 component header가 독립적인지는 알 수 없습니다. 다음 commit은 기존 test의 include surface를 이 bundle 하나로 바꿉니다.

## `3c64a69dd252` — 기존 순차 test를 aggregate include로 통과

**중요도** `C` · **태그** `TEST, PUBLIC_API`

`tests/test_containers.cpp`에서 개별 project headers를 모두 제거하고 aggregate header 하나로 교체합니다.

```diff
-#include "ft_algorithm.hpp"
-#include "ft_iterator.hpp"
-#include "ft_pair.hpp"
-#include "ft_stack.hpp"
-#include "ft_type_traits.hpp"
-#include "ft_vector.hpp"
+#include "ft_containers.hpp"
```

behavior assertion은 바뀌지 않습니다. 따라서 compile failure가 발생하면 aggregate header의 누락 또는 include composition 문제로 귀속됩니다.

이 smoke가 증명하는 범위는 작습니다.

- 한 translation unit에서 bundle 하나로 utility/vector/stack 기존 test source를 compile할 수 있습니다.
- 기존 broad checks가 같은 symbols를 사용할 수 있습니다.

다음은 증명하지 않습니다.

- component header 각각의 self-containment
- map 포함
- multi-TU linkage
- compiler portability
- sanitizer cleanliness

중요도 C인 이유도 여기에 있습니다. 새로운 semantics보다 한 include composition을 확인하는 작은 integration step입니다.

## `112af1753538` — map을 bundle과 public relation surface에 편입

**중요도** `B` · **태그** `PUBLIC_API, RB_TREE`

이 commit에는 map 내부/public API 변경이 함께 섞여 있습니다. 이 Thread에서는 공개 표면과 직접 관련된 부분만 다룹니다.

aggregate header에 map이 추가됩니다.

```diff
 # include "ft_algorithm.hpp"
 # include "ft_iterator.hpp"
+# include "ft_map.hpp"
 # include "ft_pair.hpp"
```

map에는 `value_compare`와 `value_comp()`가 추가됩니다.

```cpp
class value_compare
{
    friend class map;

public:
    bool operator()(const value_type& lhs,
        const value_type& rhs) const
    {
        return comp(lhs.first, rhs.first);
    }

protected:
    key_compare comp;

    explicit value_compare(key_compare c)
        : comp(c)
    {
    }
};
```

non-member relation operators는 Thread 01의 공용 algorithms를 사용합니다.

```cpp
return lhs.size() == rhs.size()
    && ft::equal(lhs.begin(), lhs.end(), rhs.begin());
```

```cpp
return ft::lexicographical_compare(
    lhs.begin(), lhs.end(),
    rhs.begin(), rhs.end());
```

free `swap`은 member `swap`으로 위임합니다.

```cpp
template <class Key, class T, class Compare, class Alloc>
void swap(map<Key, T, Compare, Alloc>& lhs,
    map<Key, T, Compare, Alloc>& rhs)
{
    lhs.swap(rhs);
}
```

이 SHA 이후 `ft_containers.hpp`는 utility, vector, stack, map을 함께 노출하는 최종 aggregate entry point가 됩니다. 그러나 bundle 사용만으로 개별 header의 누락 dependency를 찾지는 못하므로 독립 compile 단계가 이어집니다.

## `d938c0079994` — 각 public header를 첫 include로 독립 compile

**중요도** `B` · **태그** `TEST, PUBLIC_API, CXX98`

`tests/headers/*.cpp`가 추가됩니다. 각 source는 검증 대상 header를 첫 project include로 가져오고, 그 header의 최소 public symbol을 사용합니다.

예를 들어 map test는 다음과 같습니다.

```cpp
#include "ft_map.hpp"

int main()
{
    ft::map<int, int> values;
    values.insert(ft::make_pair(2, 6));
    return values.find(2)->second == 6 ? 0 : 1;
}
```

vector, stack, pair, iterator, algorithm, type traits, aggregate header도 각각 별도 source입니다.

Makefile은 이를 executable로 link하지 않고 object로 compile하는 `headers` target을 만듭니다.

```make
HEADER_TEST_SOURCES := $(wildcard tests/headers/*.cpp)
HEADER_TEST_OBJECTS := $(patsubst tests/headers/%.cpp,\
    $(BUILD_DIR)/headers/%.o,$(HEADER_TEST_SOURCES))

$(BUILD_DIR)/headers/%.o: tests/headers/%.cpp $(HEADERS) \
        | $(BUILD_DIR)/headers
	$(CXX) $(CXXFLAGS) $(CPPFLAGS) -c $< -o $@

headers: $(HEADER_TEST_OBJECTS)
```

이 단계가 잡는 대표 failure는 다음과 같습니다.

```text
ft_map.hpp가 ft_pair.hpp의 선언을 쓰지만 직접 include하지 않음
↓
monolithic test에서는 앞선 include가 우연히 제공
↓
ft_map.hpp 단독 compile에서 실패
```

독립 compile은 include order dependency와 header self-containment를 확인합니다. 하지만 여러 translation unit을 함께 link했을 때의 duplicate definition/ODR 문제는 아직 확인하지 않습니다.

## `072c49832ddc` — 실제 consumer처럼 여러 translation unit을 link

**중요도** `B` · **태그** `TEST, PUBLIC_API, INTEGRATION`

`tests/consumer` 아래에 세 translation unit이 추가됩니다.

```text
vector_consumer.cpp ─┐
                     ├─ link → consumer_test
map_consumer.cpp    ─┤
main.cpp            ─┘
```

vector consumer는 `ft_vector.hpp`만 직접 include해 push/insert/const iteration을 사용하고 결과 29를 반환합니다.

```cpp
int vector_consumer_result()
{
    ft::vector<int> values;
    for (int value = 1; value <= 5; ++value)
        values.push_back(value);
    values.insert(values.begin() + 2, 2, 7);

    int result = 0;
    for (ft::vector<int>::const_iterator it = values.begin();
        it != values.end(); ++it)
        result += *it;
    return result;
}
```

map consumer는 insert, erase, const iteration을 사용해 결과 55를 만듭니다. `main.cpp`는 두 함수를 호출하고 exact result를 확인합니다.

Makefile은 source별 object를 만든 뒤 하나의 binary로 link하고 실행합니다.

```make
$(CONSUMER_BIN): $(CONSUMER_OBJECTS)
	$(CXX) $(CXXFLAGS) $(CONSUMER_OBJECTS) -o $@

consumer: $(CONSUMER_BIN)
	./$(CONSUMER_BIN)

check: test headers consumer
```

이 test는 단일 test file보다 consumer 현실에 가깝습니다.

- 각 translation unit이 필요한 header를 독립 include합니다.
- 동일 template/header code가 여러 object에 나타납니다.
- 최종 link가 duplicate symbol 또는 missing definition을 드러낼 수 있습니다.
- linked binary가 vector와 map의 대표 public operations를 실제 실행합니다.

다만 두 consumer path만 다루며 모든 template specialization 조합이나 모든 public API를 link하지는 않습니다.

## `1be03ae8daef` — normal build와 격리된 ASan/UBSan acceptance

**중요도** `B` · **태그** `PUBLIC_API, PRACTICAL, RISK`

Makefile에 sanitizer flags가 추가됩니다.

```make
SANITIZER_FLAGS := -O1 -g -fno-omit-frame-pointer \
	-fsanitize=address,undefined
```

`sanitize` target은 기존 object를 재사용하지 않고 별도 build directory에서 전체 `check`를 재귀 실행합니다.

```make
sanitize:
	$(MAKE) BUILD_DIR=$(BUILD_DIR)/sanitize \
		CXXFLAGS="$(CXXFLAGS) $(SANITIZER_FLAGS)" check
```

격리가 필요한 이유는 build artifact의 instrumentation이 compile flags에 의해 결정되기 때문입니다. normal object와 sanitizer object를 같은 경로에서 공유하면 다음 문제가 생길 수 있습니다.

- flags를 바꿔도 timestamp 때문에 stale normal object가 재사용됩니다.
- 일부 object만 instrumented인 혼합 binary가 만들어집니다.
- normal build 결과와 sanitizer result가 서로 덮어씁니다.

별도 `build/sanitize` 아래에서 `check`를 다시 실행하므로 다음 acceptance surface가 같은 instrumentation을 거칩니다.

```text
behavior/failure tests
+ independent header compile
+ multi-TU consumer link/run
```

ASan은 out-of-bounds, use-after-free, double-free, leak 같은 memory issue를 관찰하고 UBSan은 instrumented undefined behavior를 관찰할 수 있습니다. 이 commit은 target을 **정의**하지만, 특정 환경에서 실제로 성공했다는 결과를 diff만으로 주장할 수는 없습니다.

## `228f457988be` — compiler/platform와 sanitizer를 CI contract로 고정

**중요도** `B` · **태그** `CXX98, PUBLIC_API, PRACTICAL`

GitHub Actions workflow가 push와 pull request에서 실행됩니다.

compiler matrix:

| OS | Compiler | Command |
| --- | --- | --- |
| Ubuntu | `g++` | `make CXX=g++ check` |
| Ubuntu | `clang++` | `make CXX=clang++ check` |
| macOS | `clang++` | `make CXX=clang++ check` |

`fail-fast: false`이므로 한 조합이 실패해도 나머지 조합의 결과를 함께 볼 수 있습니다.

별도 Linux sanitizer job은 다음 환경을 사용합니다.

```yaml
ASAN_OPTIONS: detect_leaks=1:halt_on_error=1
UBSAN_OPTIONS: halt_on_error=1:print_stacktrace=1
```

실행 command는 다음과 같습니다.

```yaml
run: make CXX=clang++ sanitize
```

이로써 strict C++98 source가 특정 local compiler에서만 우연히 통과하는 것을 줄이고, memory/UB instrumentation도 반복 가능한 acceptance에 포함합니다.

CI가 보장하는 것은 workflow가 실행된 revision과 제공된 runner/toolchain 조합의 결과입니다. 모든 compiler version, 모든 operating system, 모든 architecture를 증명하지는 않습니다.

## Acceptance layer별 역할

| 단계 | Technique | 잡는 대표 failure | 단독으로 증명하지 않는 것 |
| --- | --- | --- | --- |
| aggregate header | bundle composition | 지원 header 누락 | component self-containment |
| aggregate smoke | 기존 test include 교체 | bundle을 통한 symbol 노출 | map/multi-TU/portability |
| map public completion | bundle + relations | 최종 공개 진입점 누락 | 독립 compile |
| independent headers | header-first object compile | include-order/transitive dependency | link/ODR |
| multi-TU consumer | separate compile + link + run | duplicate/missing definition, consumer integration | sanitizer cleanliness |
| isolated sanitizer | full check under ASan/UBSan | instrumented memory/UB faults | 비계측 behavior 전체 |
| CI matrix | OS/compiler jobs | toolchain/portability regression | 모든 환경 |

## 최종 public acceptance flow

```text
make check
  ├─ test
  │    └─ container, exception, iterator, randomized, complexity binaries
  ├─ headers
  │    └─ 각 public header를 독립 translation unit로 compile
  └─ consumer
       └─ vector/map separate objects + main을 link하고 실행

make sanitize
  └─ 별도 build/sanitize에서 위 check 전체를 ASan/UBSan으로 재실행

CI
  ├─ Linux g++ check
  ├─ Linux clang++ check
  ├─ macOS clang++ check
  └─ Linux clang++ sanitize
```

## 이 Thread의 경계

이 Thread는 core container algorithm을 새로 설계하지 않습니다. 완성된 header-only public surface가 repository 내부 배열에 의존하지 않고 consumer에게 전달될 수 있는지 검증 범위를 넓힙니다.

- vector lifetime과 insertion transaction은 Thread 02에 속합니다.
- red-black invariant와 complexity는 Thread 03에 속합니다.
- iterator sentinel과 swap traversal은 Thread 04에 속합니다.
- allocator/comparator failure transaction은 Thread 05에 속합니다.
- 실제 CI run 결과나 sanitizer 통과 여부는 이 작업 환경에서 재실행하지 않았으므로 주장하지 않습니다.
