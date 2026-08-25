# Thread 7. 재현 가능한 검증 — library 경계, CTest, sanitizer, platform CI

기능과 회귀 테스트가 repository에 있어도 개발자의 임의 command에만 의존하면 같은 검증이
반복된다고 보기 어렵습니다. 이 Thread는 renderer core를 재사용 가능한 CMake target으로 분리하고,
component/integration tests를 CTest에 등록한 뒤, sanitizer configuration과 Ubuntu/macOS CI로
검증 환경을 명시합니다.

이 Thread의 커밋은 모두 중요도 B입니다. 새로운 rendering semantics를 만들기보다 이미 정의된
계약을 반복 실행 가능한 build graph에 연결하는 지원 작업이기 때문입니다.

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
| 1 | `2cf2f17980bb` | `build(cmake): 코어 라이브러리와 검증 타깃 구성` | B | `BUILD, TEST` | production core를 library로 분리하고 CTest에 실제 target path를 등록 |
| 2 | `0e8c3b51e3b7` | `test(core): 수학·기하·파서·출력 회귀 기준 추가` | B | `TEST, DETERMINISM` | 핵심 component와 고정 fixture를 한 native regression executable에 연결 |
| 3 | `58d53cce0ee5` | `build(sanitizers): 메모리와 정의되지 않은 동작 검사 구성` | B | `BUILD, TEST, RISK` | GCC/Clang ASan+UBSan compile/link instrumentation option 추가 |
| 4 | `4491bea4d93c` | `ci: 플랫폼별 빌드와 회귀 검사 자동화` | B | `BUILD, TEST, INTEGRATION` | Ubuntu/macOS release regression과 Linux sanitizer job 자동화 |

## `2cf2f17980bb` — production source를 테스트가 직접 링크할 수 있게 하다

**중요도** `B` · **태그** `BUILD, TEST`

기존 direct Make build를 CMake 구조로 옮기고 C++17과 warning policy를 target 단위로 고정합니다.
핵심 source는 `raycore` library가 되고 CLI executable은 이를 link합니다.

```cmake
add_library(raycore
    src/math.cpp
    src/geometry.cpp
    src/scene.cpp
    src/parser.cpp
    src/camera.cpp
    src/shading.cpp
    src/renderer.cpp
    src/output.cpp
)

target_include_directories(raycore PUBLIC include)
target_compile_features(raycore PUBLIC cxx_std_17)

add_executable(ray-scene-tracer src/main.cpp)
target_link_libraries(ray-scene-tracer PRIVATE raycore)
```

테스트가 production source를 별도 복사하거나 `main.cpp`까지 포함할 필요가 없습니다. 같은
`raycore`를 link하므로 tests와 CLI가 다른 구현을 검증하는 drift를 줄입니다.

CTest의 smoke command는 hard-coded binary path가 아니라 generator expression
`$<TARGET_FILE:ray-scene-tracer>`를 사용합니다. single-config와 multi-config generator에서 실제
build output location을 CMake가 해결합니다.

Makefile은 독립 compiler graph를 계속 관리하지 않고 CMake configure/build/test를 호출하는 얇은
entry point가 됩니다. 따라서 source 목록과 flags의 권위가 CMake로 모입니다.

## `0e8c3b51e3b7` — broad core regression executable

**중요도** `B` · **태그** `TEST, DETERMINISM`

`ray-core-tests`가 `raycore`를 link하고 다음 component를 한 process에서 검사합니다.

- vector/math operations
- sphere/plane/cylinder geometry의 대표 hit
- parser valid/invalid path와 source line
- P3 serialization 형태
- renderer image dimensions와 pixel storage

parser invalid fixture는 오류 line이 3인지 확인해 diagnostic location도 contract로 봅니다.
PPM test는 header와 pixel text를 고정합니다. renderer test는 full visual correctness보다
component 간 연결과 output representation의 기본 상태를 봅니다.

CMake는 source fixture 위치를 compile definition이나 source-dir parameter로 전달해 test가
실행 working directory에 우연히 의존하지 않게 합니다.

이 executable은 이후 별도의 acceleration, material, determinism, output failure tests가
추가되기 전의 broad baseline입니다. 하나의 broad test가 후속 specialized regression을 대체하지
않습니다.

## `58d53cce0ee5` — sanitizer를 개발자 개인 flag가 아닌 configuration으로 만들다

**중요도** `B` · **태그** `BUILD, TEST, RISK`

`RAY_ENABLE_SANITIZERS` option이 켜지면 GCC/Clang 계열에서 AddressSanitizer와
UndefinedBehaviorSanitizer compile/link flags를 `raycore`의 public usage requirements에
추가합니다.

```cmake
option(RAY_ENABLE_SANITIZERS
       "Enable address and undefined-behavior sanitizers"
       OFF)

if(RAY_ENABLE_SANITIZERS)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
        target_compile_options(raycore PUBLIC
            -fsanitize=address,undefined
            -fno-omit-frame-pointer)
        target_link_options(raycore PUBLIC
            -fsanitize=address,undefined)
    else()
        message(FATAL_ERROR
            "Sanitizers are supported only with GCC or Clang")
    endif()
endif()
```

`PUBLIC`인 이유는 instrumented library를 link하는 executables/tests의 link step에도 sanitizer
runtime가 필요하기 때문입니다. unsupported compiler에서 조용히 option을 무시하지 않고 configure
실패로 만듭니다.

`.gitignore`는 별도 sanitizer build directory를 포함해 source tree와 build artifact를 분리합니다.

이 configuration은 sanitizer가 찾을 수 있는 종류의 문제를 관찰 가능하게 만들 뿐, 모든 memory/
UB bug가 없음을 증명하지 않습니다. test suite가 실제 path를 실행해야 instrumentation도 의미가
있습니다.

## `4491bea4d93c` — 검증 matrix를 push/PR에 연결

**중요도** `B` · **태그** `BUILD, TEST, INTEGRATION`

GitHub Actions는 push와 pull request에서 다음 job을 실행합니다.

### Release regression

- Ubuntu
- macOS
- release configuration
- configure → build → CTest

두 platform에서 같은 source/test graph를 실행해 compiler, standard library, filesystem/CLI 차이의
대표 범위를 확인합니다.

### Sanitizer regression

- Ubuntu
- debug configuration
- `RAY_ENABLE_SANITIZERS=ON`
- ASan/UBSan environment options
- build → CTest

sanitizer job을 release matrix와 분리해 diagnostic symbols와 instrumentation을 유지합니다.
macOS sanitizer job은 이 commit의 matrix에 없습니다. Windows CI도 없습니다.

CI는 benchmark time을 pass/fail threshold로 사용하지 않습니다. 성능 harness가 repository에
있더라도 shared runner wall-clock을 correctness gate로 삼으면 noise가 커지기 때문입니다.
BVH의 primitive-work threshold는 native test/benchmark code 자체가 결정적으로 검사합니다.

## 검증 계층

```text
raycore
  ├─ CLI executable
  ├─ core component tests
  ├─ acceleration/material/render/output specialized tests
  └─ benchmark executable

CTest
  ├─ native test executables
  └─ CLI smoke scripts

CI
  ├─ Ubuntu release + CTest
  ├─ macOS release + CTest
  └─ Ubuntu debug ASan/UBSan + CTest
```

## 이 Thread가 보장하는 것

- CLI와 native tests가 같은 `raycore` implementation을 link합니다.
- CMake가 source list, language standard, include path, test registration의 기준점입니다.
- smoke tests는 실제 target path를 사용합니다.
- sanitizer instrumentation은 지원 compiler에서 library와 consumers에 함께 적용됩니다.
- 대표 Linux/macOS release와 Linux sanitizer 검증이 push/PR마다 자동 실행되도록 정의됩니다.

## 이 Thread가 보장하지 않는 것

- 이 작업 환경에서 해당 CI나 CTest가 실제로 통과했다는 주장: repository 정의만 조사했고 실행하지
  않았습니다.
- Windows/MSVC build, macOS sanitizer, 다른 architecture
- fuzzing, coverage threshold, mutation testing, race sanitizer
- benchmark wall-clock의 machine-independent 재현성
- CI provider outage나 external action dependency의 availability

이 Thread는 각 기능의 정답을 새로 정의하지 않습니다. 앞선 Thread가 만든 invariants를 같은
production library와 반복 가능한 command graph에서 실행하도록 만드는 운영상의 검증 경계입니다.
