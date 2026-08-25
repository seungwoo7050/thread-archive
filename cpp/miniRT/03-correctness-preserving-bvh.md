# Thread 3. 결과를 바꾸지 않는 BVH — 측정, 보수적 경계, 결정적 순회, 상태 수명

BVH의 목적은 “더 빨리 그린다”가 아니라 **선형 탐색과 같은 hit와 같은 image를 더 적은 primitive
test로 얻는다**는 것입니다. 이 Thread는 최적화 코드를 바로 넣지 않습니다. 먼저 작업량을
측정할 기준을 만들고, false negative가 없어야 하는 bounds를 정의하고, build/traversal 순서를
결정적으로 고정한 뒤, 마지막으로 가속 구조가 현재 geometry의 파생 상태라는 lifecycle 문제를
수정합니다.

> **조사 범위**
>
> 이 문서는 `cpp/miniRT` 브랜치의 표시된 exact SHA와 그 parent diff만을 근거로 작성했습니다.
> 다른 브랜치나 최종 HEAD의 구현을 과거 커밋에 소급하지 않았습니다. GitHub에서 diff와
> 해당 SHA의 source/test를 확인했으며, 로컬 checkout을 확보하지 못해 build·test·benchmark는
> 실행하지 않았습니다. 따라서 아래의 실행 결과는 repository에 들어 있는 assertion과
> expected value의 의미를 설명한 것이며, 이 작업 환경에서 새로 측정한 결과가 아닙니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f4dcb50939e2` | `perf(render): 광선과 교차 작업량 계측 추가` | A | `PERF, RAY_PIPELINE` | 가속 전 semantic work counters와 wall-clock timing을 렌더 경로에 연결 |
| 2 | `4fb2345c7d35` | `perf(benchmark): 조밀 장면 기준 workload 추가` | B | `PERF, ACCEL` | 400개 구·평면·두 광원의 고정 workload 정의 |
| 3 | `f5a2c4ade16d` | `perf(benchmark): 반복 측정과 결정성 보고 구성` | A | `PERF, DETERMINISM` | warmup·5회 측정·중앙값과 반복 결과 일치 검증 |
| 4 | `7b19f2ad78e3` | `feat(accel): ray-box slab 교차 구현` | A | `ACCEL, HARD, EDGE` | 세 축 interval을 교집합하는 AABB hit와 entry distance 제공 |
| 5 | `a40452885176` | `feat(accel): 도형 경계 계약과 구·평면 bounds 추가` | A | `ARCH, ACCEL, GEOMETRY` | finite bounds를 optional contract로 표현하고 구/평면을 분리 |
| 6 | `b782e22450d8` | `feat(accel): 원기둥의 보수적 bounds 계산 추가` | A | `ACCEL, GEOMETRY, HARD` | 임의 축 원기둥을 누락하지 않는 보수적 AABB 계산 |
| 7 | `419d52d687fc` | `test(accel): AABB와 도형 경계 계산 검증` | A | `TEST, ACCEL, RISK` | parallel/boundary slab과 no-false-negative shape bounds 검증 |
| 8 | `bb65e8092632` | `feat(accel): 결정적 중앙 분할 BVH 구축 구현` | A | `ACCEL, HARD, DETERMINISM` | stable median split과 original shape index tie-break로 tree 고정 |
| 9 | `9a7f29b5d78a` | `feat(accel): 선형·BVH 탐색 모드 계약 연결` | A | `ARCH, ACCEL, DETERMINISM` | 두 mode를 API에 연결하고 equal-`t` winner를 원래 index로 명시 |
| 10 | `f7e969537c10` | `feat(scene): 가속 구조 소유권과 rebuild 경계 구성` | S | `ARCH, ACCEL, SCENE` | Scene이 bounded BVH·unbounded list·ready state를 소유 |
| 11 | `d4f6ee5b6042` | `feat(accel): 결정적 BVH 최근접 순회 구현` | S | `CORE, ACCEL, HARD` | near-first stack traversal, closest pruning, unbounded 합류 구현 |
| 12 | `41c9a59f27a6` | `test(accel): 선형 탐색과 BVH 결과 동치 검증` | A | `TEST, ACCEL, DETERMINISM` | hit/pixel/checksum/equal-t 동치와 primitive test 감소 검증 |
| 13 | `da3e8b43d09e` | `perf(benchmark): 선형 탐색과 BVH 작업량 비교` | A | `PERF, ACCEL, DETERMINISM` | 같은 Scene에서 두 mode를 동일 조건으로 측정 |
| 14 | `9b77225cf6b7` | `perf(benchmark): 측정 schema와 가속 기준 검증 고정` | A | `PERF, ACCEL, DETERMINISM` | versioned schema와 primitive-test 25% 미만 기준 강제 |
| 15 | `ef5320a83c27` | `fix(accel): 가속 구조의 도형 불변식 보호` | S | `ARCH, ACCEL, SCENE` | ready BVH 뒤에서 geometry/storage를 mutate할 public path 제거 |
| 16 | `13f153e23920` | `test(accel): 장면 변경과 가속 상태 불변식 검증` | A | `TEST, ACCEL, SCENE` | immutability·invalidation·linear fallback·rebuild를 하나의 전이로 검증 |

## 1. 가속 전에 측정 계약부터 만든다

### `f4dcb50939e2` — “빠르다”를 wall-clock 하나로 표현하지 않는다

**중요도** `A` · **태그** `PERF, RAY_PIPELINE`

`RenderStats`는 primary, secondary, shadow ray와 primitive/AABB test 수, render milliseconds를
보유합니다. stats pointer는 renderer → trace → shade → scene intersection으로 전달되고, 실제
작업이 발생하는 지점에서 counter가 증가합니다.

```cpp
struct RenderStats {
    std::uint64_t primaryRays;
    std::uint64_t secondaryRays;
    std::uint64_t shadowRays;
    std::uint64_t primitiveTests;
    std::uint64_t aabbTests;
    double renderMilliseconds;
};
```

- primary ray는 pixel을 trace할 때 증가합니다.
- shadow ray는 visibility query를 시작할 때 증가합니다.
- primitive test는 `Shape::intersect`를 호출하기 직전에 증가합니다.
- 아직 BVH가 없으므로 `aabbTests`는 0입니다.

이 위치 선정이 중요합니다. renderer 외부에서 예상치를 계산하는 대신 production path가 수행한
실제 semantic work를 셉니다. wall-clock은 환경에 흔들리지만 primitive test count는 동일 scene과
동일 algorithm에서 훨씬 안정적인 비교 기준입니다.

### `4fb2345c7d35` — 고정된 조밀 장면

**중요도** `B` · **태그** `PERF, ACCEL`

benchmark fixture는 640×360, 20×20 배열의 구 400개, 평면 1개, 광원 2개를 만듭니다. 조밀한
bounded primitive가 많아 선형 탐색 비용이 드러나고, 평면 하나는 후대 unbounded path도 유지하게
합니다. 이 commit은 성능 개선을 주장하지 않고 비교 workload만 고정합니다.

### `f5a2c4ade16d` — 반복 결과가 다르면 시간도 믿지 않는다

**중요도** `A` · **태그** `PERF, DETERMINISM`

scene을 한 번 warmup한 뒤 5회 측정하고, 시간을 기준으로 정렬해 중앙값을 선택합니다.
모든 sample의 checksum과 primitive test 수가 중앙 sample과 다르면 benchmark 자체가 실패합니다.

```cpp
for (const Sample& sample : samples) {
    if (sample.checksum != median.checksum ||
        sample.stats.primitiveTests != median.stats.primitiveTests) {
        throw std::runtime_error(
            "benchmark runs produced different results");
    }
}
```

즉 더 빠른데 다른 그림을 만들거나, 실행마다 다른 작업량을 수행하는 결과는 performance evidence로
인정하지 않습니다. 이 시점에는 linear 한 mode만 측정합니다.

## 2. AABB는 실제 도형을 누락하지 않아야 한다

### `7b19f2ad78e3` — slab interval의 교집합

**중요도** `A` · **태그** `ACCEL, HARD, EDGE`

각 축에서 box의 두 면과 만나는 `t`를 계산해 `[near, far]` interval을 좁힙니다.

```cpp
double near_value = t_min;
double far_value = t_max;
for (int axis = 0; axis < 3; ++axis) {
    const double direction = component(ray.direction, axis);

    if (direction == 0.0) {
        if (origin < slab_min || origin > slab_max)
            return false;
        continue;
    }

    double first = (slab_min - origin) / direction;
    double second = (slab_max - origin) / direction;
    if (first > second)
        std::swap(first, second);

    near_value = std::max(near_value, first);
    far_value = std::min(far_value, second);
    if (far_value < near_value)
        return false;
}
```

direction이 정확히 0인 축은 division하지 않습니다. origin이 slab 안이면 그 축은 interval을
줄이지 않고, 밖이면 즉시 miss입니다. boundary contact는 `far == near`일 때 hit로 남습니다.
optional `entry`에는 최종 near 값을 돌려주어 traversal ordering에 사용할 수 있게 합니다.

이 함수는 AABB가 valid인지 먼저 검사합니다. NaN direction이나 극단적인 floating-point
환경 전체를 별도 정책으로 정하지는 않습니다.

### `a40452885176` — finite와 unbounded를 같은 척하지 않는다

**중요도** `A` · **태그** `ARCH, ACCEL, GEOMETRY`

`Shape::bounds()`는 `std::optional<Aabb>`를 반환합니다. sphere는 center±radius의 exact box를
제공하고 plane은 `std::nullopt`를 반환합니다.

```cpp
virtual std::optional<Aabb> bounds() const {
    return std::nullopt;
}

std::optional<Aabb> Sphere::bounds() const {
    const Vec3 extent(radius, radius, radius);
    return Aabb(center - extent, center + extent);
}

std::optional<Aabb> Plane::bounds() const {
    return std::nullopt;
}
```

무한 평면에 임의의 큰 box를 부여하면 그 box 밖의 실제 hit를 BVH가 제거할 수 있습니다.
optional contract는 “가속할 수 없음”을 오류나 가짜 좌표가 아니라 정상 상태로 표현합니다.

### `b782e22450d8` — 임의 축 원기둥의 보수적 box

**중요도** `A` · **태그** `ACCEL, GEOMETRY, HARD`

원기둥의 axis component마다 축 방향 half-height와 원형 단면의 투영을 합성합니다. side와 cap에서
필요한 extent를 각각 계산해 더 큰 값을 사용합니다.

```cpp
const auto extent_for = [this, half_height](double axis_component) {
    const double absolute_axis = std::fabs(axis_component);
    const double radial =
        std::sqrt(std::max(0.0, 1.0 - axis_component * axis_component));
    const double side_extent =
        absolute_axis * (half_height + kEpsilon) + radius * radial;
    const double cap_extent =
        absolute_axis * half_height +
        std::sqrt(radius * radius + kEpsilon) * radial;
    return std::max(side_extent, cap_extent);
};
```

계산된 min/max는 각 축에서 `std::nextafter`로 바깥쪽 한 representable step 확장합니다.
목표는 가장 작은 box가 아니라 **실제 surface hit를 box가 먼저 false로 버리지 않는 것**입니다.
약간 큰 box는 성능을 조금 잃지만 작은 box는 correctness를 잃습니다.

이 commit부터 `bounds()`는 pure virtual이 되어 모든 built-in shape가 bounded/unbounded 결정을
명시해야 합니다.

### `419d52d687fc` — false-negative 위험을 직접 겨냥한 테스트

**중요도** `A` · **태그** `TEST, ACCEL, RISK`

테스트는 다음 경계를 분리합니다.

- 정면 AABB hit와 entry distance 2
- 음의 direction
- box boundary를 스치는 hit
- slab과 평행하지만 바깥에 있는 miss
- sphere exact bounds
- plane이 계속 unbounded인지
- `(1,1,0)` 방향 원기둥의 bounds가 수학적 extent 이상이면서 지나치게 크지 않은지

원기둥 테스트는 box의 숫자가 특정 구현과 완전히 같다고 요구하지 않습니다. 최소한 실제 extent를
포함하고, `1e-3`보다 작은 여유 안에 있다는 containment+tightness 조건을 검사합니다.

## 3. Tree 모양과 hit winner를 결정적으로 고정한다

### `bb65e8092632` — median split build

**중요도** `A` · **태그** `ACCEL, HARD, DETERMINISM`

build는 primitive vector를 값으로 받아 내부에서 정렬하므로 caller의 원래 storage를 바꾸지
않습니다. leaf는 최대 4개입니다. 그보다 크면 centroid 범위가 가장 긴 축을 고르고 stable sort한
뒤 가운데에서 나눕니다.

```cpp
std::stable_sort(
    primitives.begin() + first,
    primitives.begin() + last,
    [axis](const BvhPrimitive& left, const BvhPrimitive& right) {
        const double l = component(left.bounds.centroid(), axis);
        const double r = component(right.bounds.centroid(), axis);
        if (l != r)
            return l < r;
        return left.shapeIndex < right.shapeIndex;
    });
```

동일 centroid에서는 original `shapeIndex`가 tie-break입니다. extent가 같은 축의 선택도 비교문
순서로 고정됩니다. 따라서 같은 primitive set은 같은 node/leaf order를 만듭니다. 최종 정렬 순서의
shape indices를 별도 contiguous vector에 기록하고 leaf가 `[first,count]` 범위를 가리킵니다.

### `9a7f29b5d78a` — mode API와 equal-`t` rule을 먼저 연결

**중요도** `A` · **태그** `ARCH, ACCEL, DETERMINISM`

`AccelMode::Linear/Bvh`가 RenderSettings와 trace/shadow/intersect call chain에 추가됩니다.
하지만 이 SHA에서 mode는 아직 `(void)mode`이고 실제 탐색은 linear입니다. 이 commit을
“BVH 순회 구현”으로 설명하면 안 됩니다.

대신 candidate update를 index 기반 lambda로 바꾸고 equal distance rule을 명시합니다.

```cpp
if (!found ||
    candidate.t < closest ||
    (candidate.t == closest && index > best_index)) {
    found = true;
    closest = candidate.t;
    best_index = index;
    hit = candidate;
}
```

동일 `t`에서는 **원래 Scene에서 뒤에 추가된 shape index**가 이깁니다. 이후 BVH가 primitive test
순서를 바꾸더라도 이 원래 index로 같은 winner를 선택할 수 있습니다.

## 4. Scene이 BVH의 수명과 unbounded path를 소유한다

### `f7e969537c10` — derived state의 소유권

**중요도** `S` · **태그** `ARCH, ACCEL, SCENE`

Scene은 다음 세 상태를 private로 갖습니다.

```cpp
Bvh bvh_;
std::vector<std::uint32_t> unboundedIndices_;
bool accelerationReady_;
```

`buildAcceleration()`은 각 shape의 bounds를 검사해 valid finite box는 `BvhPrimitive`로,
나머지는 unbounded index로 분리합니다. parser는 모든 directive를 처리하고 필수 항목을 확인한 뒤
한 번 build합니다.

`addShape`는 기존 BVH와 unbounded list를 clear하고 ready를 false로 만듭니다.

```cpp
void Scene::addShape(std::unique_ptr<Shape> shape) {
    if (shape) {
        shapes.push_back(std::move(shape));
        bvh_.clear();
        unboundedIndices_.clear();
        accelerationReady_ = false;
    }
}
```

이것이 첫 rebuild boundary입니다. 하지만 당시 shape vector와 shape geometry field가 public이어서
caller가 `addShape`를 우회하거나 center/radius를 직접 바꾸면 ready flag는 true인 채 stale BVH가
남는 결함이 있습니다. 이 결함은 `ef5320a83c27`에서 수정됩니다.

### `d4f6ee5b6042` — near-first explicit-stack traversal

**중요도** `S` · **태그** `CORE, ACCEL, HARD`

mode가 Linear이거나 acceleration이 ready가 아니면 현재 geometry를 선형 탐색합니다. ready BVH
mode에서는 root box를 검사하고 explicit stack으로 node를 방문합니다.

```cpp
if (mode == AccelMode::Linear || !accelerationReady_) {
    for (std::size_t index = 0; index < shapes.size(); ++index)
        test_shape(static_cast<std::uint32_t>(index));
    return found;
}
```

내부 node의 두 child box entry를 계산하고 가까운 child가 먼저 pop되도록 far를 먼저 push합니다.
entry가 같으면 node index로 순서를 고정합니다.

```cpp
const bool left_first =
    left_entry < right_entry ||
    (left_entry == right_entry && node.left < node.right);

stack.push_back(far_entry);
stack.push_back(near_entry);
```

stack entry의 `entry > closest`이면 이미 발견한 더 가까운 hit 때문에 subtree를 prune합니다.
leaf에서는 original shape index를 `test_shape`에 넘겨 equal-`t` rule을 그대로 적용합니다.
BVH가 끝난 뒤 unbounded indices를 같은 lambda로 검사합니다. 따라서 bounded/unbounded 두 경로도
하나의 closest/winner state에 합류합니다.

stats는 root 1회, internal child 2회씩 AABB test를 세고 실제 shape 호출 때 primitive test를 셉니다.

## 5. 동치는 pixel·primitive identity·작업량으로 검증한다

### `41c9a59f27a6` — reference implementation과 optimized implementation 비교

**중요도** `A` · **태그** `TEST, ACCEL, DETERMINISM`

`requireEquivalentHit`는 found state만 비교하지 않습니다.

- `t`
- hit point
- oriented normal
- material albedo
- `hit.shape` pointer

를 Linear와 Bvh에서 비교합니다. empty, single sphere, plane-only, arbitrary-axis cylinder가 포함됩니다.
동일 위치의 sphere 두 개를 넣은 test는 두 mode 모두 뒤에 추가된 shape를 선택해야 합니다.

dense render는 pixels와 checksum이 exact equal인지 확인하고 다음 작업량 조건을 요구합니다.

```cpp
require(bvh_stats.primitiveTests * 4 <
            linear_stats.primitiveTests,
        "BVH primitive test reduction");
```

즉 fixture에서 BVH primitive tests는 linear의 25% **미만**이어야 합니다. 이는 보편적인 복잡도
증명이 아니라 고정 workload의 regression gate입니다.

### `da3e8b43d09e` — 같은 benchmark 안에서 두 mode를 측정

**중요도** `A` · **태그** `PERF, ACCEL, DETERMINISM`

각 mode를 warmup 1회+측정 5회로 실행하고 mode 내부 checksum/primitive/AABB work가 반복마다
같은지 검사합니다. Linear와 BVH의 checksum이 다르면 report 전에 실패합니다. output은 두 mode의
median time과 work counters를 함께 기록합니다.

### `9b77225cf6b7` — 보고 형식과 최소 작업량 기준을 고정

**중요도** `A` · **태그** `PERF, ACCEL, DETERMINISM`

반복 일치 검사에 primary/secondary/shadow rays까지 추가하고 schema version과 고정 configuration을
명시합니다. benchmark 자체도 primitive ratio 25% 미만을 강제합니다.

```cpp
if (bvh.stats.primitiveTests * 4 >= linear.stats.primitiveTests)
    throw std::runtime_error(
        "BVH did not reduce primitive tests below 25 percent");
```

`medianSpeedup = linear.milliseconds / bvh.milliseconds`는 보고하지만 특정 wall-clock speedup
threshold는 강제하지 않습니다. 실행 환경의 noise 때문에 correctness/work gate와 timing report를
구분한 것입니다.

## 6. ready BVH 뒤에서 geometry를 바꿀 수 있었던 실패를 닫는다

### `ef5320a83c27` — stale acceleration의 root cause 수정

**중요도** `S` · **태그** `ARCH, ACCEL, SCENE`

기존 가정은 “shape 추가는 `addShape`를 통과하므로 BVH가 무효화된다”였습니다. 실제 public API는
다음을 허용했습니다.

- public `scene.shapes`에 직접 insert/reorder
- public `sphere.center`, `radius`, `plane.normal`, `cylinder.axis` 등을 직접 변경

이 경우 `accelerationReady_`는 true이고 BVH node bounds/original indices는 이전 geometry를
가리킵니다. Linear path는 새 geometry를 보고 BVH path는 stale snapshot을 보므로 false negative나
다른 winner가 생길 수 있습니다.

수정은 stale state를 runtime에서 추측해 감지하는 대신 mutation path를 봉인합니다.

- shape fields를 private storage로 바꾸고 const reference/value getter만 제공
- Scene shape vector를 `shapes_` private로 이동
- `shapeCount()`와 `const Shape& shapeAt()`만 공개
- shape 추가는 BVH를 무효화하는 `addShape`만 통과

이제 built-in geometry는 생성 후 immutable이고, shape set 변경은 Scene boundary가 ready state를
반드시 false로 바꿉니다.

### `13f153e23920` — type system과 runtime state 전이를 함께 검증

**중요도** `A` · **태그** `TEST, ACCEL, SCENE`

compile-time assertion은 public `shapes`가 없고, `shapeAt`이 const reference이며, geometry getter
결과에 대입할 수 없음을 확인합니다.

runtime test는 다음 전이를 고정합니다.

```text
shape 1 추가
  -> buildAcceleration()
  -> ready = true
shape 2를 addShape로 추가
  -> ready = false
Bvh mode로 query
  -> stale tree 사용 금지, 현재 shapes를 linear fallback
rebuild
  -> ready = true
  -> 현재 두 shape를 indexing한 BVH query
```

fallback과 rebuilt query 모두 새로 추가된 두 번째 sphere를 hit해야 합니다. 이 test는
“무효화되면 무조건 오류”가 아니라 **correctness를 유지하는 linear fallback**을 계약으로
정합니다.

## 최종 불변 조건

| 불변 조건 | 구현 지점 | 검증 지점 |
| --- | --- | --- |
| BVH bounds는 실제 shape를 누락하지 않는다 | optional bounds, conservative cylinder extent, outward `nextafter` | AABB/shape bounds tests |
| Linear/Bvh는 같은 original shape winner를 고른다 | index 기반 candidate update, equal-`t` later-index rule | hit identity/equal-distance tests |
| traversal order는 결과에 영향을 주지 않는다 | deterministic build tie, near-first/node-index tie, original index winner | pixel/checksum exact equality |
| unbounded shape도 항상 검사한다 | `unboundedIndices_`가 BVH traversal 뒤 같은 `test_shape`에 합류 | plane-only equivalence |
| ready acceleration은 현재 immutable geometry의 snapshot이다 | private shape storage/fields, `addShape` invalidation | compile-time mutation tests + runtime state transition |
| 성능 주장은 behaviorally equal한 결과에만 붙는다 | checksum/work repeat gate, cross-mode checksum gate | accel regression + benchmark |
| 고정 dense scene에서 primitive test가 25% 미만이다 | BVH pruning | test와 benchmark의 explicit threshold |

## Thread 경계

이 Thread는 BVH의 correctness와 work reduction을 다룹니다. 다음은 범위 밖입니다.

- 큰 vector normalization 자체: Thread 2
- metal reflection의 추가 의미: Thread 4
- 여러 worker가 동시에 같은 BVH를 읽는 tile scheduling: Thread 5
- image storage와 파일 publication: Thread 6
- benchmark를 실제 특정 machine에서 실행해 얻은 시간 수치: repository에는 harness가 있지만,
  이 작업 환경에서는 실행하지 않았습니다.

최종 상태에서 BVH는 Scene이 소유하는 선택적 파생 구조입니다. ready하지 않으면 현재 geometry를
linear로 읽고, ready하면 bounded tree와 unbounded list를 함께 탐색합니다. 어느 경우든 최종 hit
winner는 같은 original shape-index rule을 따릅니다.
