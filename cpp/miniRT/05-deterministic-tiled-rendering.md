# Thread 5. 결정적인 tile renderer — 병렬 분배와 worker 실패 회수

직렬 renderer의 pixel kernel은 이미 결정적입니다. 문제는 여러 thread가 같은 image와 scene을
공유할 때도 pixel bytes, checksum, semantic work counters가 scheduling에 따라 달라지지 않게
만드는 것입니다. 이 Thread는 thread를 바로 추가하지 않고 먼저 고정 tile 순회를 만든 뒤,
tile ownership을 원자적으로 분배하고 worker-local stats를 merge합니다. 마지막에는 worker body가
throw할 때 `std::terminate`나 partial success로 끝나던 위험을 caller-visible exception으로
바꿉니다.

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
| 1 | `498266fc0abf` | `refactor(render): 직렬 렌더링을 고정 tile 순회로 전환` | B | `REFACTOR, CONCURRENCY, DETERMINISM` | thread 없이 16×16 tile index와 clipped bounds를 먼저 고정 |
| 2 | `849f878ca0b0` | `feat(render): 원자적 tile 분배와 작업자 통계 병합 구현` | S | `ARCH, CONCURRENCY, DETERMINISM` | atomic claim, disjoint pixel writes, worker-local stats, join/merge 도입 |
| 3 | `18459bfda416` | `feat(renderer): 작업자 수 설정과 자동 선택 추가` | B | `CONCURRENCY, PERF` | explicit thread count와 hardware-concurrency 기반 자동 정책 |
| 4 | `3619550fa354` | `test(render): 작업자 수에 따른 함수 결과 동치 검증` | A | `TEST, CONCURRENCY, DETERMINISM` | 1/4 workers와 Linear/Bvh 조합의 pixels/checksum/work 동치 검증 |
| 5 | `ca2d108f2255` | `test(render): 실행 모드별 PPM byte 결정성 검증` | A | `TEST, DETERMINISM, OUTPUT` | CLI 경로에서 4개 mode/thread 조합의 exact PPM bytes 비교 |
| 6 | `0536e4829070` | `fix(renderer): 작업자 예외를 호출자에게 전달` | A | `CONCURRENCY, RISK, DEBUG` | worker exception을 저장하고 새 작업을 중단한 뒤 join 후 caller에서 rethrow |
| 7 | `b5c708ac981a` | `test(renderer): 작업자 실패 전파와 회수 검증` | A | `TEST, CONCURRENCY, RISK` | throwing shape로 sentinel exception이 caller에 도달하는지 고정 |

## `498266fc0abf` — 병렬화 전에 작업 단위를 고정하다

**중요도** `B` · **태그** `REFACTOR, CONCURRENCY, DETERMINISM`

기존 row-major 이중 loop를 16×16 tile 순회로 바꾸지만 thread는 아직 하나입니다. image width와
height를 tile size로 ceiling-divide하고, 마지막 tile은 image 경계에서 clip합니다.

```cpp
const int tile_size = 16;
const int tiles_x = (scene.width + tile_size - 1) / tile_size;
const int tiles_y = (scene.height + tile_size - 1) / tile_size;

for (int tile = 0; tile < tiles_x * tiles_y; ++tile) {
    const int tile_x = tile % tiles_x;
    const int tile_y = tile / tiles_x;
    const int x_begin = tile_x * tile_size;
    const int y_begin = tile_y * tile_size;
    const int x_end = std::min(x_begin + tile_size, scene.width);
    const int y_end = std::min(y_begin + tile_size, scene.height);
    // 기존 pixel kernel
}
```

pixel offset은 전역 coordinate `(x,y)`로 계산하므로 tile 순서가 바뀌어도 각 pixel의 저장 위치는
같습니다. 이 commit의 목적은 성능 향상이 아니라 **작업 partition 자체를 먼저 검증 가능한
representation으로 만드는 것**입니다.

고정 tile이 중요한 이유는 다음 commit에서 worker가 “다음 row” 같은 가변 상태가 아니라 하나의
정수 tile id를 claim할 수 있기 때문입니다.

## `849f878ca0b0` — tile 하나가 pixel write ownership 단위가 되다

**중요도** `S` · **태그** `ARCH, CONCURRENCY, DETERMINISM`

공유 상태는 `std::atomic<std::size_t> next_tile` 하나입니다. 각 worker가
`fetch_add(1, memory_order_relaxed)`로 서로 다른 tile id를 얻습니다.

```cpp
for (;;) {
    const std::size_t tile =
        next_tile.fetch_add(1, std::memory_order_relaxed);
    if (tile >= tile_count)
        break;
    renderTile(scene, settings, frame, tile, image, local.stats);
}
```

`relaxed` ordering으로 충분한 이유는 atomic 값이 다른 mutable state의 publish 순서를 전달하지
않기 때문입니다. 그것은 **중복 없는 정수 ticket 배분**에만 사용됩니다. Scene과 camera frame은
worker 시작 전에 완성되어 read-only이고, 서로 다른 tile은 겹치지 않는 pixel range를 씁니다.

### Pixel ownership

```text
tile id -> 고정된 [x_begin,x_end) × [y_begin,y_end)
pixel (x,y) -> offset ((y * width) + x) * 3
각 tile id는 한 worker만 claim
따라서 각 pixel byte는 정확히 한 worker만 write
```

float 결과를 thread 간에 합산하지 않습니다. 각 pixel의 tracing은 독립적이고 같은 scene/ray에서
같은 계산을 수행합니다. schedule에 따라 달라질 수 있는 shared reduction은 integer stats뿐입니다.

### Worker-local stats

각 worker는 cache-line 간섭을 줄이기 위해 `alignas(64)` local state를 갖고, 모든 thread를 join한
뒤 caller가 정수 counters를 순서대로 합칩니다. render time은 전체 wall-clock 구간에서 한 번
측정합니다.

### Thread lifecycle

RAII `ThreadJoiner`는 생성된 모든 thread를 scope exit에서 join합니다. 그러나 이 최초 구현에는
중대한 gap이 있습니다. worker lambda 내부에서 exception이 빠져나가면 C++ thread entry를 넘으므로
`std::terminate`가 호출됩니다. Joiner가 정상 cleanup을 담당해도 uncaught worker exception은 그
경로에 도달하지 못합니다. 이 위험은 `0536e4829070`에서 수정됩니다.

## `18459bfda416` — worker-count policy

**중요도** `B` · **태그** `CONCURRENCY, PERF`

`RenderSettings::threadCount`가 추가됩니다.

- `threadCount > 0`: 사용자가 지정한 수
- `threadCount == 0`: `std::thread::hardware_concurrency()`
- hardware 값이 0이면 1
- worker 수는 tile count보다 많지 않게 제한
- tile이 없으면 worker를 만들지 않음

benchmark는 비교 noise를 줄이기 위해 명시적으로 1 worker를 사용합니다. 자동 선택이
benchmark configuration에 몰래 들어오지 않게 한 것입니다.

이 policy는 optimal thread count를 증명하지 않습니다. portable default와 explicit override를
제공합니다.

## `3619550fa354` — function-level determinism

**중요도** `A` · **태그** `TEST, CONCURRENCY, DETERMINISM`

고정 96×54 scene을 다음 네 조합으로 렌더합니다.

```text
Linear, 1 worker
Linear, 4 workers
Bvh,    1 worker
Bvh,    4 workers
```

각 acceleration mode 안에서 1/4 worker 결과의 pixels와 checksum이 exact equal이어야 합니다.
primary/shadow/secondary/primitive/AABB counters도 같아야 합니다. primary rays는
`96 × 54`와 일치해야 합니다.

이 테스트는 두 종류의 bug를 구분해 잡습니다.

- overlapping/missing tile: pixel bytes 또는 primary ray count가 달라짐
- schedule-dependent stats merge: pixels는 같아도 counters가 달라짐

Linear와 Bvh 사이 primitive/AABB counts는 당연히 다를 수 있으므로, work equality는 같은 mode의
thread-count 비교에 적용합니다.

## `ca2d108f2255` — CLI artifact까지 exact bytes로 비교

**중요도** `A` · **태그** `TEST, DETERMINISM, OUTPUT`

function result가 같아도 CLI option parsing이나 file writer가 mode마다 다르게 동작할 수 있습니다.
integration test는 네 조합을 CLI로 실행해 checksum뿐 아니라 생성된 PPM 파일을 byte-for-byte
비교합니다.

```text
--linear --threads 1
--linear --threads 4
--bvh    --threads 1
--bvh    --threads 4
```

이 test는 tile completion order가 PPM row order를 바꾸지 않는다는 점까지 검증합니다. worker는
공유 stream에 직접 쓰지 않고 image의 고정 offset을 채우며, serialization은 모든 join이 끝난 뒤
단일 순서로 수행되기 때문입니다.

## `0536e4829070` — worker exception을 process failure가 아니라 renderer failure로 바꾸다

**중요도** `A` · **태그** `CONCURRENCY, RISK, DEBUG`

### 기존 실패

`Shape::intersect`, allocation, 또는 기타 worker body가 throw하면 exception이 thread entry를
탈출해 `std::terminate`를 호출할 수 있었습니다. caller는 원래 exception을 catch할 수 없고,
부분 image나 cleanup contract도 설명할 수 없습니다.

### 결정

worker마다 `std::exception_ptr` slot을 두고 body 전체를 `try/catch (...)`로 감쌉니다.

```cpp
try {
    for (;;) {
        const std::size_t tile =
            next_tile.fetch_add(1, std::memory_order_relaxed);
        if (tile >= tile_count)
            break;
        renderTile(...);
    }
} catch (...) {
    worker_errors[index] = std::current_exception();
    next_tile.store(tile_count, std::memory_order_relaxed);
}
```

첫 오류를 본 worker는 `next_tile`을 end로 보내 **아직 claim되지 않은 새 작업**을 중단합니다.
이미 claim된 다른 tile은 즉시 취소되지 않을 수 있습니다. 모든 worker를 join한 뒤 caller thread가
index 순서에서 첫 exception을 `std::rethrow_exception`합니다.

중요한 순서는 다음과 같습니다.

```text
worker catches original failure
  -> stores exception_ptr in its own slot
  -> stops future tile claims
  -> all created threads join
  -> caller checks errors
  -> caller rethrows
  -> no stats merge / successful Image return
```

worker마다 독립 slot을 쓰므로 exception storage 자체에 mutex가 필요하지 않습니다. join은 worker
writes가 caller에게 보이게 하는 synchronization boundary입니다.

이 commit은 이미 write된 pixel을 rollback하지 않습니다. 하지만 renderer가 Image를 성공으로
return하지 않으므로 partial buffer가 외부 artifact로 publish되는 정상 경로는 없습니다.

## `b5c708ac981a` — 실제 production call path에서 throw시키다

**중요도** `A` · **태그** `TEST, CONCURRENCY, RISK`

test-only `ThrowingShape`가 `intersect`에서 sentinel `std::runtime_error`를 던집니다. Scene에 넣고
4 workers로 render한 뒤 caller가 같은 message의 runtime_error를 받아야 합니다.

이 방식은 runtime wrapper나 가짜 worker 함수를 테스트하지 않습니다. 실제 경로는 다음과 같습니다.

```text
renderScene
  -> worker claims tile
  -> traceRay
  -> Scene::intersect
  -> ThrowingShape::intersect throws
  -> worker catch / exception_ptr
  -> join all
  -> caller rethrow
```

테스트가 직접 thread join 시간을 측정하거나 OS-level thread leak를 관찰하지는 않습니다.
그러나 production code에서 rethrow가 join scope 뒤에 위치하고, sentinel이 caller까지 도달하는
경로를 고정합니다.

## 최종 불변 조건

| 불변 조건 | 코드상 근거 | 검증 |
| --- | --- | --- |
| tile id는 한 worker만 소유한다 | atomic `fetch_add` | primary count와 exact pixels |
| 서로 다른 worker는 같은 pixel을 쓰지 않는다 | 고정 tile bounds와 global offset | 1/4 worker byte equality |
| scheduling은 semantic work를 바꾸지 않는다 | 독립 pixel tracing, worker-local integer stats | counters equality |
| serialization은 tile completion order와 무관하다 | join 후 row-major PPM writer | CLI exact PPM comparison |
| 모든 시작된 thread는 종료 전에 join된다 | RAII join scope | exception path 구조 + throwing regression |
| worker failure는 caller에서 관찰된다 | `exception_ptr`, join 후 rethrow | sentinel runtime_error |
| failure image는 성공으로 반환되지 않는다 | rethrow가 stats merge/return보다 앞 | throwing regression |

## Thread 경계

이 Thread는 **이미 유효한 Scene을 여러 worker가 읽어 Image를 만드는 과정**을 다룹니다.
BVH가 immutable read model을 제공하는 이유는 Thread 3에 있고, Image storage arithmetic와
PPM publication은 Thread 6에 있습니다. thread cancellation의 즉시성, work stealing, affinity,
NUMA 최적화, progress reporting은 범위 밖입니다.
