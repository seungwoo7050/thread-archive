# Thread: 저장소 내부 테스트를 지원 가능한 release claim으로 확장하기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

기능 테스트가 모두 통과해도 library를 외부에 제공할 수 있다는 결론까지 자동으로 따라오지는 않습니다. In-tree unit test는 구현과 test가 같은 include path와 build 환경을 공유하므로, public header가 private 경로에 몰래 의존하거나 금지해야 할 API가 열려 있어도 놓칠 수 있습니다. 한 compiler에서의 정상 실행은 다른 compiler·OS·data model에서도 같은 결과를 낸다는 뜻이 아니며, 정확한 출력 비교만으로 undefined behavior나 누수를 배제할 수도 없습니다.

이 Thread는 검증을 하나의 거대한 `test` 명령으로 뭉치지 않고, 서로 다른 질문을 담당하는 층으로 나눕니다.

- Unit test: 한 process 안에서 값과 exception behavior가 맞는가?
- Compile contract: public API가 허용·금지한 형태를 compiler가 강제하는가?
- CLI/integration: 실제 binary와 subsystem 조합의 byte-level protocol이 맞는가?
- External consumer: repository 밖에서도 public include와 archive만으로 사용할 수 있는가?
- Property/stress: 손으로 고른 몇 개 사례보다 넓은 상태 공간과 큰 입력을 견디는가?
- Sanitizer/leak/platform checks: 정상 결과 뒤에 숨어 있는 UB·memory·artifact 문제를 관찰하는가?
- Data-model/CI matrix: 지원 범위의 전제를 실행 가능하게 선언하고 여러 compiler/OS에서 반복하는가?

최종 불변 조건은 “모든 환경을 지원한다”가 아닙니다. **검증된 환경과 전제만 지원 범위로 주장하고, 각 검증 층이 증명하지 못하는 부분을 다른 층과 혼동하지 않는 것**입니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `6e78ced59357` | `test(contact): 공개 계약과 명령행 세션 검증` | A | `API, TEST, INTEGRATION` | unit·compile contract·CLI transcript의 기본 삼층 구조 도입 |
| `4bbbfd191669` | `test(contracts): 공개 include와 소유권 규칙 검증` | A | `API, TEST, OWNERSHIP` | public-only include와 repository-wide positive/negative API contract 확립 |
| `01271d795d58` | `test(consumer): 저장소 밖 공개 library 연결 검증` | A | `API, INTEGRATION, PORTABILITY` | 임시 외부 디렉터리에서 installed-style compile/link/run 검증 |
| `9e07d3bc86d3` | `test(boundary): 변환·배치 속성과 대용량 경계 검증` | A | `TEST, DETERMINISM, EDGE` | fixed-seed property와 4,096-job stress를 timeout 아래 실행 |
| `45e9bbfd6b75` | `build(check): sanitizer와 portable 검사 계층 구성` | A | `ARCH, PORTABILITY, TEST` | build·portable·platform 검사와 ASan/UBSan 역할 분리 |
| `ab441fa8737c` | `test(portability): 지원 LP64 데이터 모델 검증` | B | `PORTABILITY, API` | implementation이 전제하는 LP64 model을 executable gate로 명시 |
| `50565bd67e03` | `ci: 지원 compiler와 platform matrix 검증` | B | `PORTABILITY, TEST` | Linux GCC/Clang과 macOS Clang에서 기존 검증 층 반복 실행 |

## `6e78ced59357` — `test(contact): 공개 계약과 명령행 세션 검증`

이 commit 이전 contact 검증은 주로 unit 수준에서 `ContactBook`의 size와 logical order를 확인했습니다. 변경 뒤 Makefile은 같은 subsystem을 서로 다른 관찰 지점으로 나눕니다.

```make
 test-unit: $(TEST_BIN)
 	./$(TEST_BIN)

 test-contract:
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1

 test-integration: bin/ex00_contact_book
 	sh tests/check_cli.sh
```

### Unit: 내부 상태를 public behavior로 관찰

ContactBook test는 10개 값을 넣어 capacity 8 ring의 logical order가 C~J가 되는지, A와 B가 제거되었는지, `write()`가 logical index 순서의 exact text를 내는지 확인합니다.

```text
0|C|note
1|D|note
...
7|J|note
```

이 층은 domain behavior를 자세히 검사하지만 header isolation이나 실제 command parser는 통과하지 않습니다.

### Positive/negative compile contract

Positive translation unit은 public headers를 반복 include하고 exported constructor·method를 사용합니다. Header guard, declaration completeness, public symbol shape가 깨지면 compile되지 않습니다.

Negative translation unit은 의도적으로 private member인 `Contact::name_`에 접근합니다.

```cpp
cppf::Contact contact;
return contact.name_.empty() ? 0 : 1;
```

Makefile 앞의 `!` 때문에 **compile rejection이 기대한 성공**입니다. Member가 실수로 public이 되어 이 파일이 compile되면 target이 실패합니다. Runtime assertion으로는 잡기 어려운 encapsulation widening을 compiler 자체로 고정한 것입니다.

다만 이 시점의 command는 일반 `CPPFLAGS`를 사용합니다. Test include path가 public header의 누락을 가릴 가능성까지 제거하는 작업은 `4bbbfd191669`에서 이루어집니다.

### CLI transcript

`tests/check_cli.sh`는 실제 `bin/ex00_contact_book`에 다음 session을 입력합니다.

```text
ADD Ada|math
ADD Grace|compiler
BAD
LIST
QUIT
```

Expected stdout 전체를 fixture와 `diff -u`로 비교합니다.

```text
ok
ok
error
0|Ada|math
1|Grace|compiler
```

이는 method 단위 behavior가 아니라 process가 command를 읽고 state를 유지하며 protocol bytes를 내보내는 경로를 확인합니다. 그러나 한 contact application에서 이 패턴을 확립한 단계일 뿐, library 전체 public surface나 archive packaging까지는 아직 증명하지 않습니다.

## `4bbbfd191669` — `test(contracts): 공개 include와 소유권 규칙 검증`

핵심 변화는 test source가 repository 내부 보조 include에 접근하지 못하게 하는 것입니다.

```make
PUBLIC_CPPFLAGS := -Iinclude
```

Positive/negative translation unit과 public integration binary는 `-Iinclude`만 사용합니다. 따라서 public header가 `tests/` 또는 private source header를 우연히 include해야만 compile되는 상태는 숨길 수 없습니다.

### Positive contract가 확인하는 것

`tests/compile/public_headers.cpp`는 모든 installed public header를 두 번 include합니다. Subsystem별 positive unit은 단순 declaration parsing을 넘어 실제 API를 instantiate하거나 호출합니다.

- `ContactBook` 생성·add·const lookup·write
- `TextBuffer` 명시적 생성, mutable/const view
- Formatter hierarchy와 factory/pipeline 사용
- Scalar, runtime type, serializer API
- `RandomAccessBatch<int>`와 deque-backed specialization의 sort/equality
- RPN과 BatchEngine public entry points

Public integration binary는 오직 public headers와 archive를 연결해 여러 subsystem을 한 process에서 조합합니다. 예를 들어 ContactBook의 note를 RPN으로 계산하고, TextBuffer를 deep-copy한 뒤 formatter pipeline을 적용하며, BatchEngine·ScalarConverter·RuntimeType·Serializer까지 연결합니다.

### Negative contract가 확인하는 것

Negative file은 “이 코드는 compile되면 안 된다”는 API rule 하나씩을 격리합니다.

| 금지하려는 변화 | 대표 compile-fail 형태 |
| --- | --- |
| Private representation 노출 | `Contact::name_`, `TextBuffer` storage 직접 접근 |
| Const view를 통한 mutation | const `ContactBook`, `TextBuffer`, batch result, const iterator 수정 |
| 원치 않는 implicit conversion | string literal을 `TextBuffer` 인자로 암시 변환 |
| Abstract/utility type 직접 생성 | `FormatterCreator`, `PipelineBuilder`, `ScalarConverter`, `RpnEvaluator`, `RuntimeBase` 생성 |
| Template 요구사항 완화 또는 오표현 | list-backed batch에 `std::sort` 호출, const iterator mutation |
| Serializer API shape drift | private construction, 잘못된 const/pointer 사용 |

이 방식은 runtime behavior가 우연히 같아도 public signature가 넓어지거나 ownership rule이 약해지는 회귀를 잡습니다. 반대로 compiler가 API shape를 받아들였다는 사실만으로 exception cleanup·output correctness·leak freedom을 증명하지는 않습니다.

### Public contract의 두 단계

```text
syntax-only positive/negative units
    -> 허용/금지 declaration shape 확인
public_contract binary + archive link/run
    -> 여러 public component가 실제로 함께 compile·link·execute됨을 확인
```

여전히 build 위치와 repository tree는 공유합니다. Current working directory나 archive path, private generated artifact에 대한 숨은 의존성을 더 강하게 제거하는 단계가 다음 external consumer입니다.

## `01271d795d58` — `test(consumer): 저장소 밖 공개 library 연결 검증`

`tests/check_external_consumer.sh`는 temporary directory를 만들고 consumer source만 복사합니다. Compile command에 제공하는 project 자원은 public include directory와 absolute archive path뿐입니다.

```sh
"$compiler" -I"$project_root/include" \
    -Wall -Wextra -Werror -Wpedantic -pedantic-errors -std=c++98 \
    -Wold-style-cast -Wcast-qual -Woverloaded-virtual \
    -Wnon-virtual-dtor \
    "$temporary_directory/main.cpp" "$archive" \
    -o "$temporary_directory/consumer"

(
    cd "$temporary_directory"
    ./consumer
)
```

Consumer는 ContactBook, FormatPipeline, Formatter, RPN, TextBuffer를 사용하고 exact result를 검사합니다. 실행 directory도 temporary directory이므로 다음과 같은 in-tree 우연을 줄입니다.

- Source tree의 상대 path가 runtime에 존재한다는 가정
- Test/private include path가 compiler command에 섞이는 문제
- App binary가 대신 link해 주는 object에 의존하는 문제
- Archive가 빠뜨린 public implementation symbol

이 commit이 확인하는 것은 **한 대표 external program이 strict C++98 flags로 public headers와 archive만 사용해 compile·link·run할 수 있음**입니다. 설치 prefix, shared library, versioned package metadata, 모든 public API 조합까지 일반화하지는 않습니다.

## `9e07d3bc86d3` — `test(boundary): 변환·배치 속성과 대용량 경계 검증`

Hand-written edge test는 어떤 경계를 의도했는지 선명하지만, 가능한 입력 조합을 넓게 훑지는 못합니다. Property binary는 고정 seed의 간단한 generator를 사용해 실패를 재현 가능하게 유지하면서 더 많은 상태를 검사합니다.

```cpp
const unsigned long fixed_seed = 0x13579BDFUL;

unsigned long nextRandom()
{
    random_state =
        (random_state * 1103515245UL + 12345UL) & 0x7FFFFFFFUL;
    return random_state;
}
```

### 반복하는 세 property

| 대상 | 규모 | 매 입력에서 확인하는 invariant |
| --- | ---: | --- |
| Scalar integer | 2,048개 | `int:` line 일치, 항상 4 newline, 반복 출력 동일, trailing `x` 거부와 no partial output |
| RPN binary expression | 4,096개 | 안전한 범위의 `+ - * /` 결과가 native expected와 일치 |
| Batch | 4,096 jobs | 200KB 초과 입력 parse, `(value,name)` 정렬 결과 일치, 두 번 write byte-identical, 150KB 초과 output |

Property test는 fixed seed와 첫 counterexample context를 출력하므로 실패를 같은 sequence에서 재현할 수 있습니다. Large batch는 tiny fixture가 드러내지 못하는 누적 allocation, iterator, ordering cost와 output determinism을 실제 규모에서 통과시킵니다.

### Timeout도 검증의 일부다

```make
test-property: $(PROPERTY_BIN)
	sh tests/run_with_timeout.sh 30 ./$(PROPERTY_BIN)
```

Timeout wrapper는 child를 실행하고 제한 시간이 지나면 TERM, 이어서 KILL을 시도하며 124를 반환합니다. Test가 무한 loop나 비정상적으로 큰 성능 회귀에 빠져 pipeline 전체를 붙잡는 일을 방지합니다.

Property test는 고정된 한 seed와 정의된 범위만 탐색합니다. Scalar floating grammar의 모든 문자열, RPN overflow 전 범위, allocator failure는 기존 focused/failure tests가 담당합니다. 따라서 property는 example tests를 대체하지 않고 breadth를 추가합니다.

## `45e9bbfd6b75` — `build(check): sanitizer와 portable 검사 계층 구성`

이 commit은 하나였던 sanitizer target을 AddressSanitizer와 UndefinedBehaviorSanitizer로 나누고, build verification을 portable baseline과 host-specific 검사로 분리합니다.

```make
ASAN_FLAGS := -O1 -fsanitize=address -fno-omit-frame-pointer -g
UBSAN_FLAGS := -O1 -fsanitize=undefined -fno-sanitize-recover=all \
    -fno-omit-frame-pointer -g

 test-asan:
	sh tests/run_with_timeout.sh 120 env \
		ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 ./$(ASAN_BIN)

 test-ubsan:
	sh tests/run_with_timeout.sh 120 env \
		UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
		./$(UBSAN_BIN)
```

두 sanitizer는 같은 unit suite를 instrument하지만 관찰 대상이 다릅니다.

- ASan: out-of-bounds, use-after-free와 지원되는 환경의 leak detection
- UBSan: signed overflow, invalid cast 등 instrumentable undefined behavior

정상 test result와 sanitizer result도 서로 대체되지 않습니다. Sanitizer가 조용하다고 domain output이 맞는 것은 아니고, unit test가 맞다고 invalid memory access가 없는 것도 아닙니다.

### Check graph

이 SHA에서 Makefile은 다음 책임으로 target을 나눕니다.

| Target | 포함 내용 | 의도 |
| --- | --- | --- |
| `test` | unit, failure injection, no-elide, compile contract, integration/consumer, property | 기능·API·exception·통합 baseline |
| `check-build` | clean build, `test`, deterministic CLI 재실행, no-op rebuild 확인 | 어느 host에서도 우선 수행할 build baseline |
| `check-portable` | `check-build` + UBSan | 비교적 넓게 실행할 portable 검증 |
| `check-platform` | archive content, dependency audit, leak tool | 특정 host/tool capability가 필요한 검증 |
| `check` | portable + platform | 모든 local capability가 있는 환경의 전체 검사 |

`EXTRA_CXXFLAGS`도 추가되어 외부 caller나 CI가 base warning/C++98 flags를 지우지 않고 추가 조건을 주입할 수 있습니다.

여기서 “portable”은 모든 C++98 implementation에서 성공한다는 뜻이 아닙니다. Project가 여러 host에서 공통으로 실행하려는 baseline과, tool availability에 민감한 검사를 분리한 target 이름입니다. Data model 전제는 아직 암묵적이며 다음 commit이 별도의 gate로 만듭니다.

## `ab441fa8737c` — `test(portability): 지원 LP64 데이터 모델 검증`

RPN과 serializer 등은 `long`, pointer, `size_t` 크기에 영향을 받습니다. Platform마다 이 크기가 다를 수 있는데 test가 현재 machine에서 통과했다는 사실만으로 동일한 numeric boundary를 주장할 수 없습니다.

이 commit은 지원 data model을 LP64로 명시합니다.

```cpp
const bool lp64 =
    CHAR_BIT == 8 &&
    sizeof(short) == 2 &&
    sizeof(int) == 4 &&
    sizeof(long) == 8 &&
    sizeof(void *) == 8 &&
    sizeof(std::size_t) == 8;
```

조건이 맞지 않으면 실제 측정값을 stderr에 출력하고 실패합니다. `check-data-model`은 `check-build`에 들어가므로 이후 “build regression이 통과했다”는 말에는 LP64 precondition을 만족했다는 의미도 포함됩니다.

이 test는 LP64 host에서 implementation이 올바르다는 증명이 아닙니다. 다른 test들이 해석하는 numeric/ABI boundary의 전제를 먼저 확인할 뿐입니다. 반대로 ILP32나 LLP64에서 실패하는 것은 portability bug를 발견했다기보다 **현재 지원 범위 밖임을 명시적으로 거부한 것**입니다.

## `50565bd67e03` — `ci: 지원 compiler와 platform matrix 검증`

CI workflow는 새 production behavior를 만들지 않습니다. 앞서 구성한 executable claims를 서로 다른 environment에서 반복합니다.

| Runner | Compiler | `check-build` | UBSan | ASan |
| --- | --- | :---: | :---: | :---: |
| Ubuntu 22.04 | GCC (`g++`) | 예 | 예 | 예 |
| Ubuntu 22.04 | Clang (`clang++`) | 예 | 예 | 예 |
| macOS latest | Clang (`clang++`) | 예 | 예 | 아니오 |

각 job은 compiler 존재와 version을 출력한 뒤 다음을 수행합니다.

```yaml
- run: make check-build CXX="$CXX_COMMAND"
- run: make test-ubsan CXX="$CXX_COMMAND"
- run: make test-asan CXX="$CXX_COMMAND"   # matrix.asan일 때만
```

`fail-fast: false`이므로 한 조합의 실패가 다른 matrix result를 가리지 않습니다. 각 job에는 30분 timeout이 있습니다. Exact commit의 workflow trigger는 `main` push와 pull request로 선언되어 있습니다.

macOS에서 ASan을 제외한 것은 ASan이 의미 없다는 뜻이 아니라 이 matrix가 그 조합을 support evidence로 실행하지 않는다는 뜻입니다. CI에 없는 compiler version, operating system, non-LP64 model에 대한 보장은 생기지 않습니다.

## 검증 층별 blind spot

| 검증 층 | 주로 잡는 회귀 | 혼자서는 증명하지 못하는 것 |
| --- | --- | --- |
| Unit/failure test | 값, exception type, strong guarantee, exact edge behavior | Public header 독립성, external packaging, 다른 compiler |
| Compile contract | API existence, const/private/abstract/explicit rule | Runtime semantics, cleanup, output bytes |
| CLI fixture | 실제 process protocol과 exact stdout/stderr | Library-only external consumption, broad input space |
| Public integration | Public components 간 link/runtime 조합 | Repository 밖 path independence |
| External consumer | Public include + archive packaging의 대표 사용 | 모든 API 조합, 설치/버전 관리 |
| Property/stress | 넓은 deterministic state space와 large workload | Exhaustive proof, arbitrary seed, injected allocation failure |
| ASan/UBSan/leak | Instrumentable memory·UB·resource symptom | Domain correctness, 모든 tool/optimization 환경 |
| LP64 gate | Numeric/ABI 전제 충족 여부 | 다른 data model 지원, LP64에서의 전체 correctness |
| CI matrix | 지정 compiler/OS에서 위 검증의 반복 가능성 | Matrix 밖 환경과 미래 compiler behavior |

이 표의 핵심은 가장 “강한” 단일 test를 찾는 것이 아닙니다. 서로 다른 false confidence를 줄이는 evidence를 합성하는 것입니다.

## 최종 실행 구조

```text
[developer change]
  -> strict C++98 clean build
  -> unit + injected failure + no-elide
  -> positive/negative public compile contracts
  -> CLI + public integration + external consumer
  -> fixed-seed properties + large batch under timeout
  -> deterministic rerun + no-op rebuild
  -> LP64 data-model gate
  -> UBSan portable layer
  -> archive/dependency/leak platform layer
  -> CI: Linux GCC, Linux Clang, macOS Clang
```

Release claim은 이 graph를 통과한 범위로 제한됩니다.

- Public headers는 private include 없이 compile됩니다.
- 대표 external C++98 consumer는 archive와 public headers만으로 link/run됩니다.
- 기능·failure·API·determinism·large-input assertions가 정의된 suite에서 확인됩니다.
- LP64 data model만 지원 전제로 선언됩니다.
- 지정된 compiler/OS matrix가 같은 baseline을 실행합니다.

반대로 shared-library ABI compatibility, Windows/LLP64, 32-bit platform, 모든 compiler version, exhaustive input correctness는 이 Thread가 주장하지 않습니다.

## Thread 경계와 조사 범위

- 이 Thread는 production 기능 자체를 다시 설명하지 않고, 그 기능에 대해 어떤 evidence를 수집하는지 다룹니다.
- Sanitizer와 CI target이 source에 존재한다는 점은 확인했지만, 이 환경에서 compiler·test binary·workflow를 직접 실행하지 않았습니다.
- 각 설명은 표시된 exact SHA의 Makefile, test source, script, workflow diff를 기준으로 작성했습니다. 후대 target이나 matrix를 이전 commit에 소급하지 않았습니다.
