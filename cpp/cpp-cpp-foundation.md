===== BEGIN FILE: 01-direct-ownership-failure-safe-value.md =====
# Direct Ownership Becomes a Failure-safe Value

## Thread 목표

직접 할당한 C 문자열 저장소가 단순한 소유 객체에서 독립 복사, 자기 참조 안전성, 강한 예외 보장을 갖는 정규 값으로 발전하는 과정을 복원합니다.

**Source significance:** 단순한 memory ownership에서 시작해 success, aliasing, failure 모두에서 regular value로 동작하는 단계로 발전합니다. 여기서 확립된 copy-and-swap과 detached-allocation pattern은 이후 polymorphic pipeline copying과 transactional replacement에 재사용됩니다.

## 이 Thread를 이해하기 위한 핵심 질문

- `TextBuffer`의 최초 표현 불변식은 무엇이며 왜 내부 null 상태를 허용하지 않는가?
- deep copy가 없을 때 어떤 lifetime 결합과 이중 해제 위험이 생기는가?
- copy-and-swap에서 실제 상태 변경이 일어나는 commit point는 어디인가?
- `operator+=`가 기존 저장소를 해제하기 전에 새 값을 완성해야 하는 이유는 무엇인가?
- failure injection과 no-elide 빌드가 성공 경로 테스트에서 보이지 않는 어떤 위험을 드러내는가?

## 완료 기준

- [x] 각 단계에서 포인터, 크기, NUL 종료 문자의 관계를 실제 코드로 설명할 수 있다.
- [x] 복사 생성과 대입에서 source/target allocation이 독립적임을 해당 SHA 코드와 테스트로 증명할 수 있다.
- [x] 할당 실패 시 target 상태와 live-allocation baseline이 유지되는 경로를 추적할 수 있다.
- [x] copy elision이 없어도 반환값과 copy-and-swap이 안전한 이유를 실제 호출 흐름으로 설명할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- 직접 소유한 resource는 정확히 한 번 해제되고, 복사는 pointer alias가 아닌 독립 ownership을 만든다.
- strong guarantee가 문서화된 연산은 allocation 실패 시 owning object의 observable state를 바꾸지 않는다.
- 완성되지 않은 candidate는 publish하지 않는다.
- `c_str()`로 얻은 borrowed pointer는 원본의 lifetime을 넘거나 ownership을 획득하지 않는다.

### Major engineering difficulty

- 수동 할당 C 문자열의 deep copy와 exception-safe assignment 구현.
- allocation failure sweep과 live-block accounting으로 strong guarantee를 검증.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `aa3b5ba6c3c4` | feat(buffer): 종료 문자를 포함한 문자열 저장소 소유 | A | OWNERSHIP, CORE | non-null owned `char[]` 표현과 non-throwing `swap()`을 확립합니다. |
| 2 | `0bc528c7d58e` | feat(buffer): 깊은 복사와 정규 대입 구현 | S | OWNERSHIP, EXCEPTION, CORE | 독립 deep copy와 copy-and-swap assignment를 추가합니다. |
| 3 | `93faed0d67a2` | feat(buffer): 결합·비교·출력 연산 제공 | A | OWNERSHIP, EXCEPTION, CORE | self-safe, allocate-before-commit composition으로 direct owner를 확장합니다. |
| 4 | `47134f9e3b29` | test(buffer): 할당 실패와 복사 생략 비활성화 검증 | A | TEST, EXCEPTION, OWNERSHIP | 관찰된 모든 allocation failure를 주입하고 copy elision을 비활성화해 검증합니다. |

## Commit별 학습 기록

### `aa3b5ba6c3c4` — feat(buffer): 종료 문자를 포함한 문자열 저장소 소유

- Importance: **A**
- Tags: **OWNERSHIP, CORE**
- Source 역할: non-null owned `char[]` 표현과 non-throwing `swap()`을 확립합니다.
- Source classification summary: Introduces an owning NUL-terminated `char` buffer with checked access and non-throwing swap.

#### 핵심 설계 / failure boundary 확인
- [x] 해당 SHA에서 `TextBuffer`의 data pointer와 size를 저장하는 상태 필드를 찾고, default/null-input construction이 동일한 empty representation을 만드는 초기화 순서를 기록하세요.
- [x] `size() + 1` allocation과 마지막 NUL byte를 만드는 constructor/destructor 경로를 함께 추적하세요.
- [x] const/mutable `at()`가 terminator 위치를 logical range에서 제외하는 branch와 `c_str()` 반환의 borrowed-lifetime 조건을 확인하세요.
- [x] `swap()`이 pointer와 size를 함께 교환하며 throw하지 않도록 구성된 실제 코드를 찾으세요.
- [x] 이 SHA에서 copy가 어떻게 금지되는지 public interface에서 확인하고, 왜 아직 regular value가 아닌지 기록하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `0bc528c7d58e`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/TextBuffer.hpp`의 `TextBuffer`, `data_`, `size_`; `src/TextBuffer.cpp`의 default/`const char *` constructor, destructor, `at()`, `c_str()`, `swap()`.
- 핵심 코드 발췌 위치: `aa3b5ba6c3c4:src/TextBuffer.cpp`에서 empty 값도 `new char[1]`로 만들고 `data_[0] = '\0'`을 기록합니다. 문자열 constructor는 `size_ + 1`을 할당해 terminator까지 복사하며, `swap()`은 `data_`와 `size_`를 함께 교환합니다.
- 변경 전/후 차이: 이 commit에서 직접 소유하는 문자열 값이 처음 생겼습니다. null 입력과 default construction은 모두 non-null empty representation으로 정규화되지만, copy constructor와 assignment는 private이므로 독립 값 복사는 아직 제공되지 않습니다.
- 직접 확인한 ownership/lifetime/state 관계: `TextBuffer` 한 객체가 `data_` 배열의 유일한 owner이고 destructor가 `delete[]` 합니다. `size_`는 terminator를 제외한 logical length이며 `data_[size_]`는 항상 NUL입니다. `c_str()`는 소유권을 넘기지 않는 borrowed pointer라 객체 파괴나 mutation 이후 사용할 수 없습니다.
- 직접 확인한 failure path: constructor allocation이 실패하면 객체 construction 자체가 끝나지 않아 owner가 생기지 않습니다. `at(index)`는 `index >= size_`에서 예외를 던져 terminator 접근을 거부하며 state를 변경하지 않습니다. 이 시점에는 copy failure 경로가 API 밖입니다.
- 실행한 테스트와 결과: 미실행. 저장소 checkout 네트워크가 차단되어 command는 수행하지 않았고, 지정 SHA의 구현·Make target·test source만 검사했습니다.
- 이 commit을 한 문장으로 설명: null 내부 상태 없이 NUL-terminated `char[]`를 단일 소유하는 최소 `TextBuffer` 표현을 확립했습니다.

### `0bc528c7d58e` — feat(buffer): 깊은 복사와 정규 대입 구현

- Importance: **S**
- Tags: **OWNERSHIP, EXCEPTION, CORE**
- Source 역할: 독립 deep copy와 copy-and-swap assignment를 추가합니다.
- Source classification summary: Adds deep copying and copy-and-swap assignment to `TextBuffer`.

#### 이 commit 직전 상태와 문제
- 직전 관련 Thread SHA `aa3b5ba6c3c4`를 먼저 checkout하여 이 commit이 추가되기 전 representation/ownership/state-publication 방식을 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 직전 관련 SHA `aa3b5ba6c3c4`와 비교해 copy constructor/assignment 선언이 public contract에 어떻게 추가되는지 확인하세요.
- [x] copy constructor가 source와 별도 `size + 1` allocation을 만들고 terminator까지 복사하는 코드를 추적하세요.
- [x] assignment에서 temporary construction → non-throwing `swap()` → temporary destruction 순서를 실제 코드 라인으로 기록하세요.
- [x] allocation이 temporary construction 중 실패할 때 target state가 아직 변경되지 않았음을 제어 흐름으로 증명하세요.
- [x] alias를 통한 self-assignment가 별도 `this == &other` branch 없이도 안전한 이유를 객체 lifetime과 allocation 기준으로 설명하세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `93faed0d67a2`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/TextBuffer.hpp`의 public copy constructor/assignment; `src/TextBuffer.cpp`의 `TextBuffer(const TextBuffer&)`, `operator=`, `swap()`.
- 핵심 코드 발췌 위치: `0bc528c7d58e:src/TextBuffer.cpp`의 copy constructor는 `new char[other.size_ + 1]` 후 `std::memcpy(..., size_ + 1)`를 수행합니다. assignment의 핵심은 `TextBuffer copy(other); swap(copy); return *this;`입니다.
- 변경 전/후 차이: 직전 SHA에서는 copy가 private이었습니다. 이후 source와 별도 저장소를 가진 deep copy가 public contract가 되었고 assignment가 target을 직접 덮지 않고 완성된 temporary와 교환합니다.
- 직접 확인한 ownership/lifetime/state 관계: copy construction 성공 후 source와 copy는 서로 다른 `char[]` owner입니다. assignment 전에는 target이 기존 배열을, temporary가 새 배열을 소유합니다. `swap()` 뒤 target이 새 배열을, temporary가 이전 target 배열을 소유하고 scope 종료 시 temporary destructor가 이전 배열을 해제합니다.
- 직접 확인한 failure path: temporary copy construction 중 `new[]`가 실패하면 `swap()`에 도달하지 않아 target의 pointer, size, bytes가 그대로입니다. self-assignment도 source가 곧 target이어도 먼저 독립 copy를 만들기 때문에 별도 alias branch 없이 안전합니다.
- 실행한 테스트와 결과: 미실행. 저장소 checkout 네트워크가 차단되어 command는 수행하지 않았고, 지정 SHA의 구현·Make target·test source만 검사했습니다.
- 이 commit을 한 문장으로 설명: deep copy와 copy-and-swap으로 `TextBuffer`를 독립 ownership과 강한 대입 보장을 가진 값으로 바꿨습니다.

### `93faed0d67a2` — feat(buffer): 결합·비교·출력 연산 제공

- Importance: **A**
- Tags: **OWNERSHIP, EXCEPTION, CORE**
- Source 역할: self-safe, allocate-before-commit composition으로 direct owner를 확장합니다.
- Source classification summary: Adds concatenation, comparison, and stream operations with overflow and allocate-before-commit handling.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `0bc528c7d58e`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] `operator+=`에서 `size + other.size + 1` overflow를 실제 allocation 전에 검사하는 식과 branch를 확인하세요.
- [x] joined storage를 완성한 뒤 old storage를 release하는 정확한 순서를 추적하고, 실패 시 old value가 남는 지점을 표시하세요.
- [x] self-concatenation에서 `other`가 `*this`와 alias여도 source bytes가 release 전에 모두 사용되는지 코드 순서로 확인하세요.
- [x] non-member `operator+`가 copy + compound addition을 재사용하는 caller/callee 관계를 기록하세요.
- [x] 비교 및 stream insertion이 allocation representation을 노출하지 않고 value semantics만 제공하는지 public API를 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `47134f9e3b29`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/TextBuffer.hpp`의 `operator+=`, 비교·stream 연산 선언; `src/TextBuffer.cpp`의 `operator+=`, non-member `operator+`, `operator==`, `operator<`, `operator<<`.
- 핵심 코드 발췌 위치: `93faed0d67a2:src/TextBuffer.cpp`의 `operator+=`는 `other.size_ > max - size_ - 1`을 먼저 검사하고, `joined` 배열에 기존 bytes와 `other`의 terminator까지 복사한 다음에만 old `data_`를 해제하고 새 pointer/size를 게시합니다.
- 변경 전/후 차이: 값 복사만 가능하던 상태에서 결합·비교·출력으로 확장되었습니다. 결합은 기존 저장소를 먼저 변경하는 대신 detached allocation을 완성한 뒤 commit합니다.
- 직접 확인한 ownership/lifetime/state 관계: 새 배열은 함수 내부 candidate owner이고 모든 복사가 끝날 때까지 기존 `data_`가 source로 살아 있습니다. commit 뒤 `TextBuffer`가 candidate를 소유합니다. 비교와 stream insertion은 내부 pointer ownership을 노출하지 않고 bytes의 값만 관찰합니다.
- 직접 확인한 failure path: 길이 합 overflow는 allocation 전에 거부됩니다. allocation 또는 복사 준비 단계의 예외에서는 old array가 해제되지 않습니다. `buffer += buffer`에서도 source bytes를 읽는 동안 old storage가 살아 있어 alias가 안전합니다. final destination stream failure의 rollback은 이 commit의 보장이 아닙니다.
- 실행한 테스트와 결과: 미실행. 저장소 checkout 네트워크가 차단되어 command는 수행하지 않았고, 지정 SHA의 구현·Make target·test source만 검사했습니다.
- 이 commit을 한 문장으로 설명: allocate-before-commit을 결합 연산까지 확장해 overflow, allocation failure, self-alias를 처리했습니다.

### `47134f9e3b29` — test(buffer): 할당 실패와 복사 생략 비활성화 검증

- Importance: **A**
- Tags: **TEST, EXCEPTION, OWNERSHIP**
- Source 역할: 관찰된 모든 allocation failure를 주입하고 copy elision을 비활성화해 검증합니다.
- Source classification summary: Adds deterministic allocation-failure injection and no-elide builds for buffer operations.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `93faed0d67a2`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] test executable에서 global allocation 함수가 counted `malloc`-backed implementation으로 교체되는 지점을 찾으세요.
- [x] construction, copy construction, assignment, aliased self-assignment, `+`, `+=` 각각에서 관찰된 allocation site를 어떻게 순회해 failure를 주입하는지 기록하세요.
- [x] 각 실패 후 object state와 live-allocation baseline을 어떤 assertion으로 확인하는지 구분하세요.
- [x] copy elision을 비활성화한 별도 build target/flags와 실행 test가 무엇인지 확인하세요.
- [x] 이 테스트가 production code의 어떤 allocation/copy path를 통과하며, 일반 unit test가 놓치던 temporary lifetime을 무엇으로 드러내는지 적으세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **TextBuffer의 strong exception guarantee와 leak freedom**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **allocation 실패 및 copy-elision 부재**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **deterministic allocation-failure sweep + no-elide build**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **construction/copy/assignment/addition/compound-addition allocation paths**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **strong guarantee와 leak baseline이 관찰된 allocation sites에서 유지됨**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **관찰되지 않은 실행 경로나 다른 allocator 환경까지 자동으로 증명하지는 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic regression / failure-injection**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/support/FailingNew.hpp`, `tests/support/FailingNew.cpp`; `tests/failure/test_buffer_failure.cpp`; `Makefile`의 `failure-test`, `test-no-elide` 계열 target.
- 핵심 코드 발췌 위치: `47134f9e3b29:tests/failure/test_buffer_failure.cpp`의 construction/copy/assignment/aliased assignment/`+`/`+=` failure sweep과, `FailingNew`의 allocation-attempt 및 live-block counter. `Makefile`은 `-fno-elide-constructors`를 적용한 별도 binary를 구성합니다.
- 변경 전/후 차이: production 동작은 바꾸지 않고, 정상 경로만 보던 검증에 관찰된 각 allocation attempt의 deterministic failure와 copy-elision 비활성화 실행 구성을 추가했습니다.
- 직접 확인한 ownership/lifetime/state 관계: failure controller가 정확한 allocation attempt를 실패시키고 test는 예외 후 source/target text와 live block baseline을 비교합니다. no-elide build는 반환 temporary와 복사/destruction이 실제로 발생해도 owner가 중복 해제되지 않는지 노출합니다.
- 직접 확인한 failure path: 각 연산을 한 번 성공시켜 allocation 횟수를 관찰한 뒤 1부터 그 횟수까지 실패 지점을 이동합니다. assignment와 `+=`는 기존 target 값 유지, constructor/`+`는 partial object leak 없음, 모든 case는 live block baseline 복구를 검사하도록 작성되어 있습니다. 관찰하지 않은 allocator 동작이나 실행 경로까지 증명하지는 않습니다.
- 실행한 테스트와 결과: 미실행. 저장소 checkout 네트워크가 차단되어 command는 수행하지 않았고, 지정 SHA의 구현·Make target·test source만 검사했습니다. 실행 대상으로 확인한 command는 `make failure-test`와 no-elide test target입니다.
- 이 commit을 한 문장으로 설명: deterministic allocation failure와 no-elide 구성을 통해 `TextBuffer`의 강한 보장과 단일 해제를 회귀 계약으로 만들었습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `aa3b5ba6c3c4` | non-null `char[]` ownership과 NUL 종료 표현, non-throwing `swap()` 도입 | `TextBuffer`가 `char *data_`와 `size_`를 갖고 모든 constructor가 non-null NUL-terminated 배열을 만들며 destructor가 `delete[]` 합니다. | 복사가 private이므로 독립 value copy와 copy failure 보장은 아직 없습니다. |
| `0bc528c7d58e` | 독립 deep copy와 copy-and-swap 대입으로 regular value/strong assignment guarantee 확립 | copy constructor가 별도 `size + 1` 배열을 만들고 assignment가 temporary construction 뒤 `swap()`으로만 게시합니다. | 결합 같은 후속 mutation 연산의 overflow와 failure 보장은 아직 다루지 않습니다. |
| `93faed0d67a2` | allocate-before-commit 결합과 self-concatenation 안전성으로 패턴 확장 | `operator+=`가 길이 overflow를 먼저 검사하고 detached joined 배열을 완성한 후 old storage를 교체합니다. | destination stream의 최종 write 실패 rollback과 관찰하지 않은 allocator 특성은 범위 밖입니다. |
| `47134f9e3b29` | 모든 관찰 allocation failure와 no-elide 조건에서 보장 검증 | `FailingNew`와 failure sweep이 각 관찰 allocation site 뒤 state/live-block baseline을 검사하고 no-elide binary를 별도로 만듭니다. | 실행 환경 전체나 관찰되지 않은 path에 대한 형식 증명은 아닙니다. |

## Failure → Fix → Test 연결

- `0bc528c7d58e`: deep copy + copy-and-swap으로 assignment failure 시 target 보존을 설계합니다.
- `93faed0d67a2`: allocate-before-commit으로 composition까지 동일한 failure discipline을 확장합니다.
- `47134f9e3b29`: allocation failure sweep과 no-elide build로 이 보장을 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: 직접 소유 pointer를 shallow copy하거나, target 저장소를 먼저 해제한 뒤 새 값을 만들면 alias·double free·failure 중 state loss가 발생합니다.
- 이를 드러낸 실제 failure 또는 test gap: 최초 구현은 copy를 금지해 위험을 피했지만 regular value가 아니었고, 정상 unit test만으로는 allocation 중간 실패와 반환 temporary lifetime을 확인할 수 없었습니다.
- 수정/강화된 decision: copy constructor는 별도 allocation을 만들고, assignment와 composition은 throw 가능한 준비를 detached object/storage에서 끝낸 뒤 non-throwing publication을 수행합니다.
- 해당 코드 위치: `0bc528c7d58e:src/TextBuffer.cpp`의 copy constructor와 `operator=`, `93faed0d67a2:src/TextBuffer.cpp`의 `operator+=`.
- 이를 고정하는 regression/evidence: `47134f9e3b29:tests/failure/test_buffer_failure.cpp`, `tests/support/FailingNew.cpp`, `Makefile`의 no-elide 구성.

## Ownership / state / responsibility 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: non-null `char[]` ownership과 NUL 종료 표현, non-throwing `swap()` 도입
- Thread 종료 상태: 모든 관찰 allocation failure와 no-elide 조건에서 보장 검증
- [x] 중간 commit마다 owner/state publisher/cleanup 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned state가 함께 등장하면 각각의 lifetime 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `aa3b5ba6c3c4`: 객체가 non-null `char[]`의 owner가 되고 `c_str()`만 borrowed view를 제공합니다.
2. `0bc528c7d58e`: 복사 owner를 source와 분리하고, assignment의 state publisher를 non-throwing `swap()` 하나로 제한합니다.
3. `93faed0d67a2`: 동일한 detached preparation을 결합에 적용합니다. old storage는 candidate 완성 전까지 source이자 rollback state로 남습니다.
4. `47134f9e3b29`: failure controller가 candidate 생성의 각 allocation을 끊어도 기존 owner와 live-block 수가 보존되는지 검사합니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `construction → owned representation → deep copy / assignment → composition → injected-failure verification`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준 호출 흐름은 constructor/copy가 독립 배열을 만든 뒤, assignment는 `TextBuffer copy(other)`를 완성하고 `swap(copy)`로 게시하며, `operator+`는 value copy 후 `operator+=`를 재사용하는 형태입니다. `operator+=`는 overflow 검사와 detached allocation을 모두 통과한 뒤에만 pointer와 size를 바꿉니다.

시작 시점과 비교하면 copy 금지 owner가 deep-copy 가능한 regular value가 되었고, assignment와 composition 모두 allocation failure에서 기존 observable value를 보존합니다. 남는 경계는 `c_str()` borrowed pointer의 invalidation, destination stream 자체의 write failure, 테스트가 관찰하지 못한 allocator/환경입니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: constructor 또는 caller 연산 → `TextBuffer` copy constructor / `operator+=` → allocation·byte copy → `swap()` 또는 pointer publication → 비교·`c_str()`·stream을 통한 관찰.
- 핵심 상태 필드: `char *data_`, `std::size_t size_`, 그리고 invariant `data_ != 0`, `data_[size_] == '\0'`.
- resource owner / borrowed view: 각 `TextBuffer`가 자신의 `data_`를 단독 소유하고 `c_str()` 결과만 객체 lifetime에 종속된 borrowed view입니다.
- commit point: assignment는 `swap(copy)`, `operator+=`는 joined bytes 완성 후 old array 해제와 새 `data_`/`size_` 대입입니다.
- cleanup path: 실패 전 candidate constructor가 소유권을 얻지 못하거나 local temporary가 자신이 소유한 배열을 파괴합니다. target의 old array는 commit 전까지 유지됩니다.
- 최종 invariant 설명: 성공·self-alias·관찰된 allocation failure·copy-elision 부재에서 각 배열은 정확히 한 owner에게 속하고, strong-guarantee 연산은 완성된 값만 publish합니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 01-direct-ownership-failure-safe-value.md =====

===== BEGIN FILE: 02-polymorphic-cloning-owning-aggregate.md =====
# Polymorphic Cloning Becomes a Regular Owning Aggregate

## Thread 목표

가상 인터페이스 자체보다 더 어려운 문제인 동적 객체의 생성·복제·소유·파괴를 추적하고, 여러 clone을 소유하는 aggregate가 실패 중에도 누수 없이 정규 값으로 동작하는 과정을 복원합니다.

**Source significance:** 이 branch의 중심 C++ object-model progression입니다. runtime polymorphism만으로는 부족하며, 각 dynamic object의 create/copy/own/destroy 책임을 정하고 incomplete cloning이 leak이나 기존 aggregate corruption을 만들지 않음을 검증합니다.

## 이 Thread를 이해하기 위한 핵심 질문

- base object 복사 대신 `clone()`이 필요한 이유는 무엇인가?
- virtual destructor가 ownership protocol의 일부인 이유는 무엇인가?
- `FormatPipeline::append()`는 borrowed prototype을 어느 시점에 owned clone으로 바꾸는가?
- copy constructor 도중 clone이 실패하면 왜 pipeline destructor만으로 정리가 불가능한가?
- copy construction과 assignment 실패가 서로 다른 cleanup mechanism을 요구하는 지점은 어디인가?

## 완료 기준

- [x] prototype, clone, pipeline slot 각각의 owner와 lifetime을 commit별로 그릴 수 있다.
- [x] pipeline copy constructor의 partial-construction cleanup과 assignment의 copy-and-swap을 구분해 설명할 수 있다.
- [x] abstractness, virtual destruction, clone ownership이 각각 어떤 test/compile contract로 고정되는지 찾을 수 있다.
- [x] clone failure sweep에서 source와 destination이 각각 어떤 상태로 남는지 실제 테스트를 근거로 설명할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- polymorphically owned resource는 정확히 한 번 해제되고, copying은 dynamic object의 독립 ownership을 만든다.
- 완성되지 않은 partial pipeline은 temporary/partial owner 내부에만 존재하며 publish되지 않는다.
- strong guarantee가 적용되는 assignment는 clone 실패 시 destination observable state를 보존한다.

### Major engineering difficulty

- `clone()`이 raw pointer를 반환하는 heterogeneous polymorphic object의 ownership과 copying.
- constructor가 완료되지 않아 destructor가 호출되지 않는 상황에서 partial clones 정리.
- clone failure sweep과 live-object accounting.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `835d87865762` | feat(format): 다형적 formatter 인터페이스 정의 | S | ARCH, POLYMORPHISM, OWNERSHIP | abstract formatter behavior, virtual destruction, virtual copying을 정의합니다. |
| 2 | `62ed45f8adf9` | feat(format): formatter 소유 pipeline 구현 | S | ARCH, POLYMORPHISM, OWNERSHIP | pipeline이 formatter lifetime을 빌리지 않고 clone을 직접 소유하게 합니다. |
| 3 | `bf4d9bed705c` | feat(format): pipeline 깊은 복사 구현 | S | OWNERSHIP, EXCEPTION, POLYMORPHISM | heterogeneous owner를 deep-copy하고 partial construction을 정리합니다. |
| 4 | `0427713637b8` | test(format): 가상 소멸·추상 계약·CLI 검증 | A | API, TEST, POLYMORPHISM | abstractness, virtual destruction, clone ownership, headers, CLI를 검증합니다. |
| 5 | `2c99290b9268` | test(format): 복제 실패 뒤 부분 객체 정리 검증 | A | TEST, EXCEPTION, POLYMORPHISM | copy construction/assignment의 clone failure를 전 위치에서 sweep합니다. |

## Commit별 학습 기록

### `835d87865762` — feat(format): 다형적 formatter 인터페이스 정의

- Importance: **S**
- Tags: **ARCH, POLYMORPHISM, OWNERSHIP**
- Source 역할: abstract formatter behavior, virtual destruction, virtual copying을 정의합니다.
- Source classification summary: Defines the abstract formatter interface, virtual destruction, cloning, and concrete transformations.

#### 이 commit 직전 상태와 문제
- 이 Thread의 첫 commit이므로, `git show <sha>^`가 가능한 경우 parent에서 관련 type/기능이 없거나 다른 형태였는지 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] `Formatter` 선언에서 abstractness를 만드는 pure virtual functions, virtual destructor, `clone()`, `apply()`, name contract를 확인하세요.
- [x] 각 concrete formatter가 `clone()`에서 dynamic type을 보존한 독립 heap object를 만드는 실제 코드를 찾으세요.
- [x] prefix/suffix formatter가 자신의 `TextBuffer` configuration을 소유하는 상태와 copy semantics를 확인하세요.
- [x] uppercase 구현에서 `std::toupper` 호출 전에 plain `char`를 `unsigned char`로 변환하는 경로를 확인하세요.
- [x] caller가 derived type을 몰라도 `Formatter&`/pointer를 통해 동작·복제·삭제할 수 있는 call graph를 그리세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `62ed45f8adf9`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/Formatter.hpp`의 `Formatter`, `UppercaseFormatter`, `PrefixFormatter`, `SuffixFormatter`; `src/Formatter.cpp`의 destructor, `clone()`, `apply()`, `name()` 구현.
- 핵심 코드 발췌 위치: `835d87865762:include/cppf/Formatter.hpp`에서 `Formatter`는 virtual destructor와 pure virtual `clone()`, `apply()`, `name()`을 선언합니다. `src/Formatter.cpp`의 각 `clone()`은 `new <구체 타입>(*this)`를 반환하며 uppercase 변환은 `std::toupper(static_cast<unsigned char>(...))`를 사용합니다.
- 변경 전/후 차이: parent에는 formatter 계층이 없었습니다. 이 commit에서 호출자가 구체 타입을 몰라도 base reference/pointer로 변환·복제·삭제할 수 있는 object-model contract가 생겼습니다.
- 직접 확인한 ownership/lifetime/state 관계: `PrefixFormatter::prefix_`와 `SuffixFormatter::suffix_`는 각 formatter가 소유하는 `TextBuffer`입니다. `clone()` 성공 시 반환 pointer의 ownership은 호출자에게 이전되고, virtual destructor는 base pointer 삭제가 실제 dynamic destructor까지 도달하게 합니다.
- 직접 확인한 failure path: `new` 또는 `TextBuffer` 복사가 실패하면 `clone()`은 pointer를 반환하지 않습니다. 이 SHA에는 여러 clone을 모아 관리하는 aggregate나 local guard가 아직 없으므로, 성공한 raw pointer를 누가 즉시 소유하는지는 caller protocol에 남아 있습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 header와 implementation을 검사했으며 build/test command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: virtual destruction과 virtual copying을 포함한 formatter 소유권 protocol을 정의했습니다.

### `62ed45f8adf9` — feat(format): formatter 소유 pipeline 구현

- Importance: **S**
- Tags: **ARCH, POLYMORPHISM, OWNERSHIP**
- Source 역할: pipeline이 formatter lifetime을 빌리지 않고 clone을 직접 소유하게 합니다.
- Source classification summary: Introduces a bounded pipeline that owns formatter clones and applies them in order.

#### 이 commit 직전 상태와 문제
- 직전 관련 Thread SHA `835d87865762`를 먼저 checkout하여 이 commit이 추가되기 전 representation/ownership/state-publication 방식을 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] `FormatPipeline`의 fixed pointer array, size, capacity 표현을 찾아 null/valid prefix 상태를 기록하세요.
- [x] `append()`에서 capacity check → `clone()` → slot store → size increment의 순서를 실제 코드로 확인하세요.
- [x] 입력 formatter reference는 borrowed이고 clone은 pipeline-owned가 되는 ownership 전환 시점을 표시하세요.
- [x] destructor가 성공적으로 저장된 clone만 정확히 한 번 `delete`하는 범위를 확인하세요.
- [x] `apply()`가 input copy를 만들고 insertion order대로 step을 fold하는 경로와 empty-pipeline identity를 확인하세요.
- [x] 이 SHA에서 pipeline copy가 여전히 금지되어 있는 public/private declaration을 확인하세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `bf4d9bed705c`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/FormatPipeline.hpp`의 `max_steps`, `steps_`, `size_`; `src/FormatPipeline.cpp`의 constructor, destructor, `append()`, `apply()`, `swap()`.
- 핵심 코드 발췌 위치: `62ed45f8adf9:src/FormatPipeline.cpp`의 `append()`는 capacity를 먼저 검사하고 `formatter.clone()`을 호출한 뒤 `steps_[size_]`에 저장하고 마지막에 `++size_` 합니다. destructor는 `[0, size_)`의 pointer만 `delete`합니다.
- 변경 전/후 차이: formatter 객체 단위의 clone contract에서, 최대 8개 clone을 insertion order로 직접 소유하고 실행하는 aggregate가 추가되었습니다. copy constructor와 assignment는 여전히 private입니다.
- 직접 확인한 ownership/lifetime/state 관계: `append()` 인자의 `Formatter&`는 호출자 소유의 borrowed prototype입니다. `clone()`이 반환된 순간 local `copy`가 미게시 owner가 되고, slot 저장과 size 증가가 끝나면 pipeline이 clone의 유일한 owner가 됩니다. 유효 상태는 `steps_[0..size_)`가 소유 pointer이고 나머지는 null인 prefix입니다.
- 직접 확인한 failure path: capacity 초과는 clone 전에 거부됩니다. `clone()` 실패 시 slot과 `size_`가 변경되지 않습니다. 성공 뒤에는 destructor가 active prefix를 삭제합니다. 다만 pipeline 자체의 독립 deep copy와 복사 도중 부분 clone 정리는 아직 제공되지 않습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 header와 implementation을 검사했으며 build/test command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: borrowed formatter를 owned clone으로 바꾸어 순서대로 실행하는 bounded pipeline을 만들었습니다.

### `bf4d9bed705c` — feat(format): pipeline 깊은 복사 구현

- Importance: **S**
- Tags: **OWNERSHIP, EXCEPTION, POLYMORPHISM**
- Source 역할: heterogeneous owner를 deep-copy하고 partial construction을 정리합니다.
- Source classification summary: Implements deep pipeline copying, partial-construction cleanup, and copy-and-swap assignment.

#### 이 commit 직전 상태와 문제
- 직전 관련 Thread SHA `62ed45f8adf9`를 먼저 checkout하여 이 commit이 추가되기 전 representation/ownership/state-publication 방식을 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 직전 관련 SHA `62ed45f8adf9`와 비교해 copy constructor와 assignment가 어떻게 열리는지 확인하세요.
- [x] copy constructor 시작 시 pointer array 전체를 null-initialize하는 순서를 찾고, clone 성공 prefix의 representation을 기록하세요.
- [x] 중간 `clone()` 실패 시 catch block이 이미 생성된 prefix를 직접 delete하고 rethrow하는 코드를 추적하세요.
- [x] constructor가 완료되지 않으면 destructor가 호출되지 않는다는 사실이 왜 이 explicit cleanup을 필요로 하는지 실제 경로에 연결하세요.
- [x] assignment의 complete-copy → swap → old-state destruction 흐름과 destination preservation을 확인하세요.
- [x] dynamic formatter type과 insertion order가 copy 후 유지되는지 clone/apply 경로로 확인하세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `0427713637b8`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/FormatPipeline.hpp`의 public copy constructor/assignment; `src/FormatPipeline.cpp`의 copy constructor, catch cleanup, `operator=`, `swap()`.
- 핵심 코드 발췌 위치: `bf4d9bed705c:src/FormatPipeline.cpp`에서 copy constructor는 모든 `steps_`를 null로 초기화한 뒤 `append(*other.steps_[index])`를 반복합니다. catch block은 현재 `size_`만큼 직접 `delete`하고 rethrow하며, assignment는 `FormatPipeline copy(other); swap(copy);`입니다.
- 변경 전/후 차이: 직전 SHA에서 복사가 금지됐지만, 이후 dynamic type과 insertion order를 유지하는 heterogeneous deep copy가 public contract가 되었습니다. 실패한 constructor와 실패한 assignment가 서로 다른 cleanup 경로를 사용합니다.
- 직접 확인한 ownership/lifetime/state 관계: copy constructor의 성공 prefix는 아직 완성되지 않은 `this`가 임시로 소유합니다. construction 성공 뒤 새 pipeline이 모든 clone을 소유합니다. assignment에서는 완성된 local `copy`가 candidate이고 `swap()` 뒤 old destination clone들은 local object로 이동해 scope 종료 때 삭제됩니다.
- 직접 확인한 failure path: copy constructor 중 clone 실패 시 객체 destructor가 호출되지 않으므로 catch가 이미 만든 prefix를 직접 삭제해야 합니다. assignment 중 같은 실패는 local `copy` construction에서 끝나 `swap()`에 도달하지 않으므로 destination의 size, step sequence, behavior가 보존됩니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 implementation과 후속 테스트가 겨냥하는 경로를 코드로 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: explicit partial-construction cleanup과 copy-and-swap으로 polymorphic aggregate를 failure-safe regular value로 만들었습니다.

### `0427713637b8` — test(format): 가상 소멸·추상 계약·CLI 검증

- Importance: **A**
- Tags: **API, TEST, POLYMORPHISM**
- Source 역할: abstractness, virtual destruction, clone ownership, headers, CLI를 검증합니다.
- Source classification summary: Adds abstractness, virtual-destruction, public-header, ownership-counter, and CLI checks.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `bf4d9bed705c`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] counted test formatter의 live/destroyed counters가 clone ownership과 base-pointer deletion을 어떻게 관찰하는지 확인하세요.
- [x] abstract `Formatter`를 직접 instantiate하려는 compile-fail case와 기대 실패 조건을 찾으세요.
- [x] public header repeated inclusion/consumer-visible use를 검사하는 positive compile case를 확인하세요.
- [x] pipeline CLI integration fixture가 실제 binary에서 virtual dispatch, archive linkage, step order를 어떤 transcript로 검증하는지 기록하세요.
- [x] 이 test bundle이 output correctness와 별개로 non-virtual destruction/accidental concreteness/ownership leak를 각각 어떻게 겨냥하는지 분리하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **polymorphic abstractness, virtual destruction, clone ownership, public consumption**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **accidentally concrete base, non-virtual delete, hidden ownership leak, process integration drift**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **compile-fail + live-object counter + public-header compile + CLI fixture**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **Formatter clone/delete and pipeline execution paths**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **object-model contract가 runtime output 외에도 유지됨**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **모든 clone failure position까지는 이 commit 하나로 증명하지 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **broad contract + integration**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `2c99290b9268`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_formatter.cpp`, `tests/test_format_pipeline.cpp`, `tests/compile/formatter_abstract_fail.cpp`, `tests/compile/format_headers.cpp`, `tests/check_cli.sh`, `Makefile`의 contract/integration targets.
- 핵심 코드 발췌 위치: `0427713637b8`의 counted formatter tests는 clone·live·destroyed counters를 관찰하고 base pointer delete를 실행합니다. compile-fail translation unit은 `Formatter` 직접 생성을 시도하며, positive translation unit은 public headers를 반복 include합니다. CLI script는 실제 formatter pipeline binary의 transcript를 fixture와 비교합니다.
- 변경 전/후 차이: production object model은 유지하고, runtime output만으로 드러나지 않던 abstractness, virtual destruction, clone ownership, header isolation, process integration을 별도 검증 층으로 추가했습니다.
- 직접 확인한 ownership/lifetime/state 관계: counted formatter의 clone 증가와 pipeline scope 종료 후 live count 복구가 clone ownership을 관찰합니다. base pointer 삭제 뒤 derived destruction counter가 증가해야 virtual destructor contract가 성립합니다.
- 직접 확인한 failure path: abstract base가 concrete가 되거나 destructor가 non-virtual이면 compile/runtime counter 검사가 실패하도록 작성되어 있습니다. CLI fixture는 step 순서나 archive linkage drift를 잡지만, clone 실패 위치를 하나씩 주입하지는 않습니다.
- 실행한 테스트와 결과: 미실행. 실행 대상으로 `make test-contract`, unit test, CLI integration target을 확인했으나 현재 환경에서는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: formatter object-model contract를 compile, ownership counter, 실제 CLI의 세 층으로 고정했습니다.

### `2c99290b9268` — test(format): 복제 실패 뒤 부분 객체 정리 검증

- Importance: **A**
- Tags: **TEST, EXCEPTION, POLYMORPHISM**
- Source 역할: copy construction/assignment의 clone failure를 전 위치에서 sweep합니다.
- Source classification summary: Sweeps clone failures during pipeline copy construction and assignment.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `0427713637b8`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] copy construction과 assignment 각각에서 clone 실패를 formatter position별로 주입하는 test control을 찾으세요.
- [x] failed constructor 뒤 이미 생성된 clones가 모두 사라지는지 live-object counter assertions을 확인하세요.
- [x] failed assignment 뒤 destination의 기존 step sequence와 behavior가 유지되는 assertion을 확인하세요.
- [x] source pipeline이 failure sweep 전후 동일하게 살아 있는지 확인하는 증거를 기록하세요.
- [x] production code에서 constructor catch cleanup과 assignment copy-and-swap 두 경로 중 어느 것을 각 test case가 통과하는지 매핑하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **partial polymorphic copy cleanup과 strong assignment guarantee**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **clone failure at each formatter position**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **deterministic clone-failure sweep + live-object counters**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **FormatPipeline copy constructor catch cleanup / assignment copy-and-swap**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **failed construction leak freedom과 failed assignment target preservation**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **factory creation failure나 다른 subsystem allocation failure는 범위 밖**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic regression / failure-injection**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/support/TestFormatter.hpp`, `tests/support/TestFormatter.cpp`의 clone failure control과 live counters; `tests/failure/test_pipeline_failure.cpp`; `Makefile`의 failure test target.
- 핵심 코드 발췌 위치: `2c99290b9268:tests/failure/test_pipeline_failure.cpp`는 source pipeline의 formatter 수를 기준으로 failure position을 이동하며 copy construction과 assignment를 각각 시도합니다. `TestFormatter`의 `failCloneOn()`/clone-attempt counter가 지정 clone에서 예외를 발생시킵니다.
- 변경 전/후 차이: 앞선 broad contract 검증에, 복사 중 각 formatter 위치를 결정적으로 실패시키는 회귀가 추가되었습니다. production 코드는 바뀌지 않습니다.
- 직접 확인한 ownership/lifetime/state 관계: failed copy construction 뒤 live count가 source만 남는 baseline으로 돌아오는지 확인합니다. failed assignment 뒤 destination의 기존 step sequence와 적용 결과, source pipeline의 결과가 모두 그대로인지 확인합니다.
- 직접 확인한 failure path: constructor case는 `FormatPipeline` copy constructor의 catch cleanup을, assignment case는 local candidate construction 실패로 `swap()`을 건너뛰는 경로를 통과합니다. 이 테스트는 factory creation이나 일반 allocation 전체가 아니라 formatter clone failure 위치만 다룹니다.
- 실행한 테스트와 결과: 미실행. failure injection 구현과 Make target은 검사했으나 test binary는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: 모든 clone 위치에서 부분 객체 정리와 failed assignment의 destination 보존을 deterministic regression으로 만들었습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `835d87865762` | virtual destructor와 `clone()`으로 polymorphic copy/deletion protocol 정의 | `Formatter`의 pure virtual `clone/apply/name`, virtual destructor, 각 derived `new Derived(*this)` 구현으로 dynamic copy/delete protocol을 확인했습니다. | 성공한 raw clone을 즉시 인수할 aggregate/guard와 multi-object failure cleanup은 아직 없습니다. |
| `62ed45f8adf9` | pipeline이 borrowed formatter가 아니라 clone을 소유하도록 ownership boundary 확립 | `steps_[max_steps]`, `size_`, capacity-before-clone, slot store-before-size increment, active-prefix destruction으로 pipeline ownership을 확인했습니다. | pipeline copy가 private이어서 aggregate value semantics와 partial-copy cleanup은 아직 없습니다. |
| `bf4d9bed705c` | heterogeneous deep copy와 failed-constructor cleanup, copy-and-swap assignment 확립 | copy constructor의 null initialization·append loop·catch delete와 assignment의 complete copy 후 `swap()`을 확인했습니다. | 실제 clone failure 위치별 회귀와 public abstractness/virtual-delete 증거는 후속 검증이 필요합니다. |
| `0427713637b8` | abstractness/virtual destruction/public contract/process integration 증거 추가 | compile-fail abstractness, repeated public-header compile, counted clone/destruction, CLI fixture가 object-model과 process contract를 확인합니다. | 모든 clone 위치에서 constructor/assignment 실패를 주입하지는 않습니다. |
| `2c99290b9268` | 모든 formatter 위치의 clone failure에서 partial cleanup과 target preservation 검증 | clone failure position sweep가 failed constructor의 live baseline과 failed assignment의 destination/source behavior를 비교합니다. | factory creation, allocator 전 지점, 다른 subsystem failure는 이 테스트 범위 밖입니다. |

## Failure → Fix → Test 연결

- `bf4d9bed705c`: clone 실패 중 incomplete copy constructor가 destructor에 의존할 수 없는 문제를 explicit cleanup으로 해결합니다.
- `0427713637b8`: virtual destruction/abstractness/clone ownership의 broader contract evidence를 추가합니다.
- `2c99290b9268`: clone failure를 formatter position별로 sweep하며 constructor cleanup과 assignment transaction을 직접 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: virtual dispatch만으로는 heap에 생긴 derived object의 owner, 복사 방법, base pointer destruction, 부분 복사 실패 cleanup이 정해지지 않습니다.
- 이를 드러낸 실제 failure 또는 test gap: pipeline copy constructor가 여러 `clone()` 중 하나에서 실패하면 완성되지 않은 객체의 destructor가 호출되지 않으며, 정상 output test만으로는 leaked prefix나 non-virtual delete를 볼 수 없습니다.
- 수정/강화된 decision: `Formatter`가 virtual destructor와 owning `clone()` protocol을 제공하고, pipeline은 borrowed prototype을 즉시 clone해 소유합니다. copy constructor는 catch에서 성공 prefix를 직접 정리하고 assignment는 완성된 copy만 swap합니다.
- 해당 코드 위치: `835d87865762:include/cppf/Formatter.hpp`, `62ed45f8adf9:src/FormatPipeline.cpp`, `bf4d9bed705c:src/FormatPipeline.cpp`.
- 이를 고정하는 regression/evidence: `0427713637b8`의 compile/counter/CLI tests와 `2c99290b9268:tests/failure/test_pipeline_failure.cpp`의 clone-position sweep.

## Ownership / state / responsibility 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: virtual destructor와 `clone()`으로 polymorphic copy/deletion protocol 정의
- Thread 종료 상태: 모든 formatter 위치의 clone failure에서 partial cleanup과 target preservation 검증
- [x] 중간 commit마다 owner/state publisher/cleanup 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned state가 함께 등장하면 각각의 lifetime 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `835d87865762`: concrete formatter가 자신의 configuration을 소유하고, `clone()` 성공 pointer를 caller에게 넘기는 protocol이 정의됩니다.
2. `62ed45f8adf9`: pipeline이 borrowed prototype을 받아 독립 clone을 만들고 `steps_[0..size_)`의 cleanup owner가 됩니다.
3. `bf4d9bed705c`: pipeline 복사 중 성공한 clone prefix는 incomplete constructor가 직접 정리하고, assignment publication은 `swap()` 한 번으로 제한됩니다.
4. `0427713637b8`: abstractness, virtual deletion, clone ownership, public headers, CLI execution 경로가 서로 다른 검사로 고정됩니다.
5. `2c99290b9268`: 각 clone failure에서 constructor prefix는 사라지고 assignment destination은 보존되는지 counters와 behavior로 확인합니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `borrowed formatter prototype → virtual clone → owned pipeline slot → deep-copied aggregate → failure cleanup verification`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준으로 caller는 concrete formatter를 stack이나 다른 owner에 둔 채 `FormatPipeline::append(const Formatter&)`에 borrowed reference를 전달합니다. `append()`가 virtual `clone()`으로 independent dynamic object를 만들고 pipeline slot이 이를 소유합니다. pipeline copy는 source의 각 dynamic object를 다시 clone하며, `apply()`는 insertion order로 virtual dispatch를 수행합니다.

시작 시점과 비교하면 단일 virtual interface가 독립 복사 가능한 owning aggregate로 확장되었습니다. construction 실패와 assignment 실패의 cleanup 책임도 구분됩니다. 다만 raw-pointer clone protocol은 caller가 pipeline 밖에서 직접 사용할 때 즉시 ownership을 인수해야 하며, 후속 tests가 관찰하지 않은 allocator·factory failure는 이 Thread만으로 증명되지 않습니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: caller의 concrete `Formatter` → `FormatPipeline::append()` → virtual `clone()` → owned `steps_[size_]` → `apply()`의 ordered virtual dispatch → `TextBuffer` 결과 반환.
- 핵심 상태 필드: `Formatter *steps_[max_steps]`, `std::size_t size_`; 각 derived formatter의 `TextBuffer prefix_` 또는 `suffix_`.
- resource owner / borrowed view: append 인자와 source pipeline step 참조는 borrowed이고, 반환된 clone은 성공한 slot 또는 copy-construction prefix가 소유합니다.
- commit point: 단일 append는 clone 성공 후 slot 저장과 `++size_`; assignment는 complete local copy를 만든 뒤 `swap(copy)`입니다.
- cleanup path: append 전 clone 실패는 반환 owner가 없고 pipeline은 불변입니다. copy constructor 실패는 catch가 `[0, size_)` clone을 직접 삭제하며, assignment 실패는 local candidate가 target과 swap되기 전에 정리됩니다.
- 최종 invariant 설명: 각 polymorphic clone은 정확히 한 pipeline에 소유되고 base pointer로 안전하게 파괴되며, deep copy는 dynamic type과 순서를 유지하고 incomplete aggregate는 publish되지 않습니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 02-polymorphic-cloning-owning-aggregate.md =====

===== BEGIN FILE: 03-factory-transaction-boundary.md =====
# Factory Assembly Acquires a Transaction Boundary

## Thread 목표

factory가 반환한 raw owning pointer를 안전하게 넘기는 것과 기존 pipeline 상태를 원자적으로 교체하는 것이 별개의 문제임을 확인하고, clear-then-build에서 candidate-then-swap으로 수정되는 과정을 복원합니다.

**Source significance:** 초기 builder는 resource cleanup은 수행하지만 partially rebuilt target을 노출할 수 있습니다. 수정은 leak freedom과 object-state atomicity를 분리해 인식하고, 전체 candidate 성공 후 one non-throwing swap을 commit point로 둡니다.

## 이 Thread를 이해하기 위한 핵심 질문

- factory specification grammar와 formatter ownership transfer의 경계는 어디인가?
- 로컬 RAII guard가 누수를 막아도 target의 기존 상태는 왜 보존되지 않을 수 있는가?
- 초기 `replace()`의 commit point는 사실상 어디에 분산되어 있었는가?
- fix 이후 candidate가 실패할 때 target과 partial clones는 각각 누가 정리하는가?
- regression test와 failure sweep은 각각 어떤 수준의 보장을 증명하는가?

## 완료 기준

- [x] creator → local guard → pipeline clone의 ownership handoff를 실제 코드로 추적할 수 있다.
- [x] fix 전 clear-and-append 경로와 fix 후 candidate-and-swap 경로를 관련 SHA끼리 비교할 수 있다.
- [x] leak freedom과 strong state preservation이 서로 다른 보장임을 failure path로 설명할 수 있다.
- [x] unknown formatter, null/count 오류, clone/allocation failure가 target에 미치는 영향을 테스트별로 구분할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- candidate는 완성되기 전 target에 publish되지 않는다.
- strong guarantee가 적용되는 replacement는 creation/clone/allocation 실패 시 target observable state를 보존한다.
- polymorphically owned resource는 정확히 한 번 해제된다.

### Major engineering difficulty

- factory creation이 raw pointer를 반환할 때 heterogeneous ownership handoff 관리.
- multi-step factory creation 중 partial target mutation 방지.
- allocation/clone failure sweep으로 ownership transition 전체 검증.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `4c34654a4602` | feat(factory): 문자열 명세로 formatter 생성 | A | ARCH, POLYMORPHISM, API | creator abstraction과 formatter specification grammar를 도입합니다. |
| 2 | `fc0b8b7a40a0` | feat(factory): formatter 임시 소유와 pipeline 교체 구현 | A | OWNERSHIP, INTEGRATION, EXCEPTION | creator raw pointer를 즉시 guard하고 target pipeline을 assembly합니다. |
| 3 | `907bfbd5c37c` | fix(factory): 교체 실패에도 기존 파이프라인 보존 | S | DEBUG, EXCEPTION, CORE | clear-then-build를 candidate-then-swap publication으로 수정합니다. |
| 4 | `466d7abdb60f` | test(factory): 교체 실패 상태 보존과 CLI 검증 | B | TEST, EXCEPTION | rejected replacement 뒤 state preservation과 CLI behavior를 회귀 검증합니다. |
| 5 | `af4e35ca7d92` | test(factory): 생성·복제·할당 실패 정리 검증 | A | TEST, EXCEPTION, OWNERSHIP | creation, clone, allocation failure 전 구간의 ownership handoff를 sweep합니다. |

## Commit별 학습 기록

### `4c34654a4602` — feat(factory): 문자열 명세로 formatter 생성

- Importance: **A**
- Tags: **ARCH, POLYMORPHISM, API**
- Source 역할: creator abstraction과 formatter specification grammar를 도입합니다.
- Source classification summary: Adds a polymorphic creator and grammar for constructing formatters from specifications.

#### 핵심 설계 / failure boundary 확인
- [x] exact `upper`, `prefix=<payload>`, `suffix=<payload>` grammar를 분기하는 parser/factory 코드를 찾고 malformed와 unknown classification branch를 분리해 기록하세요.
- [x] `FormatterCreator`가 abstract가 되는 선언과 virtual destructor를 확인하세요.
- [x] `create()`의 반환 타입에서 raw owning pointer transfer가 어떻게 드러나는지 public signature를 확인하세요.
- [x] factory가 생성하는 concrete dynamic types와 caller가 base pointer만 받는 관계를 추적하세요.
- [x] `PipelineBuilder`가 이 시점에는 어떤 operation boundary만 선언하고 있는지 실제 public declaration을 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `fc0b8b7a40a0`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/Factory.hpp`의 `InvalidSpecification`, `UnknownFormatter`, `FormatterCreator`, `DefaultFormatterCreator`, `PipelineBuilder`; `src/Factory.cpp`의 `DefaultFormatterCreator::create()`.
- 핵심 코드 발췌 위치: `4c34654a4602:src/Factory.cpp`에서 empty specification은 `InvalidSpecification`, exact `upper`는 `UppercaseFormatter`, non-empty `prefix=`/`suffix=` payload는 해당 concrete formatter, 나머지는 `UnknownFormatter`로 분기됩니다.
- 변경 전/후 차이: formatter를 코드에서 직접 생성하던 경계에 문자열 specification grammar와 polymorphic creator가 추가되었습니다. `FormatterCreator`는 virtual destructor와 pure virtual `create()`를 가지며 `PipelineBuilder`는 static replacement operation을 선언합니다.
- 직접 확인한 ownership/lifetime/state 관계: `create()`의 반환형은 `Formatter *`이고 성공 시 heap object ownership이 caller에게 이전됩니다. caller는 구체 dynamic type을 알 필요가 없지만, 반환 직후 delete 책임을 인수해야 합니다.
- 직접 확인한 failure path: empty key, payload 없는 `prefix=`/`suffix=`, unknown key는 pointer를 반환하기 전에 예외로 끝납니다. 이 SHA에는 여러 specification을 조립하는 구현이나 raw pointer를 보호하는 local owner가 아직 없습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 public declaration과 parser/factory implementation을 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: formatter 문자열 grammar와 raw-owning polymorphic creation boundary를 도입했습니다.

### `fc0b8b7a40a0` — feat(factory): formatter 임시 소유와 pipeline 교체 구현

- Importance: **A**
- Tags: **OWNERSHIP, INTEGRATION, EXCEPTION**
- Source 역할: creator raw pointer를 즉시 guard하고 target pipeline을 assembly합니다.
- Source classification summary: Adds RAII ownership of factory results and pipeline replacement from a specification list.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `4c34654a4602`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] creator가 반환한 raw pointer를 즉시 소유하는 local RAII owner의 constructor/destructor와 scope를 찾으세요.
- [x] local formatter를 `FormatPipeline::append()`에 넘긴 뒤 pipeline이 clone을 소유하고 local guard가 원본을 delete하는 두 단계 ownership을 추적하세요.
- [x] null specification array/count consistency와 capacity를 work 시작 전에 검사하는 branch를 확인하세요.
- [x] empty specification list가 target을 empty pipeline으로 바꾸는 경로를 확인하세요.
- [x] 가장 중요하게, target을 먼저 clear/empty로 만들고 이후 직접 append하는 mutation 순서를 찾고 중간 failure 시 observable target이 무엇이 되는지 기록하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `907bfbd5c37c`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/Factory.cpp`의 unnamed-namespace `FormatterOwner`, `PipelineBuilder::replace()`; `FormatPipeline::append()`/`swap()`.
- 핵심 코드 발췌 위치: `fc0b8b7a40a0:src/Factory.cpp`의 `FormatterOwner`는 creator raw pointer를 constructor에서 받아 destructor에서 `delete`합니다. `replace()`는 validation 후 `target.swap(empty)`로 기존 값을 먼저 비우고, 각 owner의 formatter를 `target.append()`에 전달합니다.
- 변경 전/후 차이: creator 결과의 leak 방지와 specification list 조립이 구현되었습니다. 그러나 assembly destination이 target 자체라 replacement의 상태 변경은 시작 시 empty swap과 각 append에 분산됩니다.
- 직접 확인한 ownership/lifetime/state 관계: creator 성공 직후 local `FormatterOwner`가 원본 dynamic object를 소유합니다. `target.append(formatter.get())`는 borrowed reference를 받아 별도 clone을 만들고 target이 clone을 소유합니다. loop iteration 종료 시 owner는 creator 원본을 삭제합니다.
- 직접 확인한 failure path: null/count 불일치와 capacity 초과는 mutation 전에 거부됩니다. 이후 create 또는 append/clone이 실패하면 local owner와 현재 scope의 objects는 누수 없이 정리되지만, target은 이미 비워졌거나 성공한 앞 단계 clone만 가진 partial pipeline으로 남습니다. empty specification list는 target을 empty로 교체합니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 ownership handoff와 mutation 순서를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: factory 결과 누수는 막았지만 target을 증분 변경해 강한 교체 보장은 만들지 못했습니다.

### `907bfbd5c37c` — fix(factory): 교체 실패에도 기존 파이프라인 보존

- Importance: **S**
- Tags: **DEBUG, EXCEPTION, CORE**
- Source 역할: clear-then-build를 candidate-then-swap publication으로 수정합니다.
- Source classification summary: Builds a complete candidate pipeline before swapping it into the target.

#### Failure → Fix → Test chain
- **기존 가정:** factory-created temporaries를 leak 없이 정리하면 replacement도 충분히 안전하다고 볼 수 있었다.
- **실제 failure / 위험:** later specification의 create/clone 실패가 target을 이미 비웠거나 partial pipeline으로 남길 수 있었다.
- **root cause:** multi-step operation의 commit point가 target 내부 여러 mutation으로 분산되어 있었다.
- **수정된 invariant / decision:** 완전한 candidate만 one non-throwing swap으로 target에 publish한다.
- **실제 코드 확인:** `fc0b8b7a40a0`과 `907bfbd5c37c`의 `PipelineBuilder::replace()`를 비교해 mutation destination과 final swap을 확인한다.
- **regression test:** `466d7abdb60f`의 seeded target preservation, 이어서 `af4e35ca7d92`의 full failure sweep을 확인한다.

#### 이 commit 직전 상태와 문제
- 직전 관련 Thread SHA `fc0b8b7a40a0`를 먼저 checkout하여 이 commit이 추가되기 전 representation/ownership/state-publication 방식을 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 직전 관련 SHA `fc0b8b7a40a0`의 clear-then-build 코드와 이 SHA의 candidate-then-swap 코드를 직접 diff하세요.
- [x] 새 `FormatPipeline candidate`가 생성되고 모든 create/append가 candidate에만 적용되는 caller/callee 흐름을 추적하세요.
- [x] create 또는 clone exception이 발생하면 candidate destructor와 local guard가 무엇을 정리하고 target에는 어떤 write도 하지 않는지 확인하세요.
- [x] 모든 specification 성공 뒤 실행되는 단 하나의 non-throwing `target.swap(candidate)`를 commit point로 표시하세요.
- [x] swap 이후 candidate가 old target state를 소유하고 scope 종료 시 해제하는 lifetime 전환을 그리세요.
- [x] fix가 resource cleanup이 아니라 object-state atomicity를 복구한 것임을 실제 before/after state mutation 순서로 설명하세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `466d7abdb60f`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/Factory.cpp`의 `PipelineBuilder::replace()`, `FormatterOwner`; `FormatPipeline::append()`, destructor, `swap()`.
- 핵심 코드 발췌 위치: `907bfbd5c37c:src/Factory.cpp`는 `FormatPipeline candidate`를 만들고 모든 `create()`/`candidate.append()`를 끝낸 뒤 마지막에 `target.swap(candidate)`를 한 번 호출합니다.
- 변경 전/후 차이: 직전의 `target.swap(empty)`와 direct append가 제거되고, mutation destination이 local candidate로 이동했습니다. resource cleanup 방식은 유지되지만 object-state publication 지점은 final swap 하나로 축소되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: 각 iteration에서 creator object는 `FormatterOwner`, append가 만든 clone은 candidate가 소유합니다. final swap 전 target은 계속 old pipeline을 소유합니다. swap 후 target이 complete candidate를, local candidate가 old target을 소유하며 scope 종료 시 old target clones를 삭제합니다.
- 직접 확인한 failure path: create, formatter construction, clone, candidate capacity/allocation 중 어느 단계에서 throw해도 local owner와 candidate destructor가 새 resources를 정리하고 `target.swap()`에는 도달하지 않습니다. 따라서 target size, step order, output behavior가 유지됩니다.
- 실행한 테스트와 결과: 미실행. fix 전후 `Factory.cpp`를 직접 비교하고 후속 regression source를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: 완성된 candidate만 non-throwing swap으로 게시해 leak freedom과 target atomicity를 함께 만족시켰습니다.

### `466d7abdb60f` — test(factory): 교체 실패 상태 보존과 CLI 검증

- Importance: **B**
- Tags: **TEST, EXCEPTION**
- Source 역할: rejected replacement 뒤 state preservation과 CLI behavior를 회귀 검증합니다.
- Source classification summary: Adds regression and CLI checks for failed replacement preserving the prior pipeline.

#### Thread 흐름에서 확인할 구현 역할
- [x] 직전 관련 SHA `907bfbd5c37c`와의 차이 중 이 Thread의 흐름에 필요한 부분만 확인하세요.
- [x] seeded target pipeline을 준비한 뒤 중간 unknown formatter를 넣는 regression case를 찾으세요.
- [x] invalid null/count combination이 validation 단계에서 target을 보존하는 case를 확인하세요.
- [x] 각 failure 뒤 transformed output 또는 step behavior가 seed와 동일함을 어떤 assertion으로 확인하는지 기록하세요.
- [x] CLI invalid configuration에서 nonzero status, stderr diagnostic, stdout empty를 동시에 검증하는 fixture를 찾으세요.
- [x] 이 테스트가 candidate-then-swap의 대표 failure를 고정하지만 모든 allocation/clone site를 sweep하지는 않는다는 범위를 실제 fixture 수로 확인하세요.
- [x] 이 commit이 다음 관련 commit의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **PipelineBuilder strong replacement guarantee**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **중간 unknown formatter와 invalid null/count**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **seeded-state regression + CLI fixture**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **builder validation / candidate assembly / final swap**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **대표 rejection paths에서 prior target과 stdout atomicity가 유지됨**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **모든 create/clone/allocation failure point는 후속 sweep가 담당**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic regression + integration**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `af4e35ca7d92`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_factory.cpp`, `tests/check_cli.sh`의 factory cases, 관련 fixtures와 `Makefile`의 unit/integration targets.
- 핵심 코드 발췌 위치: `466d7abdb60f:tests/test_factory.cpp`는 기존 pipeline을 seed한 뒤 list 중간에 unknown formatter를 두거나 null/count 조합을 잘못 전달하고, exception 후 seed 적용 결과가 동일한지 확인합니다. CLI script는 invalid configuration의 nonzero status, expected stderr, empty stdout를 비교합니다.
- 변경 전/후 차이: candidate-then-swap production fix 위에 대표적인 grammar/validation rejection과 process-level output atomicity 회귀가 추가되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: regression은 failure 전 target behavior를 baseline으로 저장하고 failure 뒤 같은 transformation을 재실행합니다. empty list success는 old target이 local candidate로 이동해 파괴되고 target이 zero-step pipeline이 되는 정상 commit도 확인합니다.
- 직접 확인한 failure path: unknown middle item은 candidate에 앞 step clone이 이미 존재하는 상태에서 예외를 발생시켜 candidate destructor 경로를 통과합니다. null/count failure는 work 시작 전 validation 경로입니다. 고정 fixture 수만 확인하므로 모든 allocation/clone site는 sweep하지 않습니다.
- 실행한 테스트와 결과: 미실행. unit/CLI fixture와 expected assertions를 검사했지만 binary는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: 대표 rejection에서 seeded target과 CLI stdout이 보존되는지 고정한 회귀입니다.

### `af4e35ca7d92` — test(factory): 생성·복제·할당 실패 정리 검증

- Importance: **A**
- Tags: **TEST, EXCEPTION, OWNERSHIP**
- Source 역할: creation, clone, allocation failure 전 구간의 ownership handoff를 sweep합니다.
- Source classification summary: Sweeps creation, clone, and allocation failures while checking cleanup and target preservation.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `466d7abdb60f`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] custom creator와 counted formatter가 create failure와 clone failure를 어떻게 구분해 주입하는지 test doubles를 확인하세요.
- [x] observed allocation/clone failure point를 순회하는 sweep loop와 failure index 제어를 기록하세요.
- [x] creator → local guard → pipeline clone → partial candidate destructor → final target까지 ownership transition별 live count assertion을 매핑하세요.
- [x] 모든 injected exception 뒤 original target behavior가 유지되는 assertion을 확인하세요.
- [x] 누수뿐 아니라 premature mutation도 검출하도록 어떤 baseline/state 비교를 함께 수행하는지 기록하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **factory-builder ownership handoff와 target preservation**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **creation, clone, allocation failure at every observed point**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **custom creator + counted formatter + deterministic sweep**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **creator → guard → pipeline clone → candidate cleanup → target swap**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **전체 ownership transition에서 leak/premature mutation이 없음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **stream/CLI transport failures는 주 대상이 아님**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic failure-injection regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/failure/test_factory_failure.cpp`; `tests/support/TestFormatter.hpp/.cpp`; `tests/support/FailingNew.hpp/.cpp`; custom creator test double; `Makefile`의 factory failure binary.
- 핵심 코드 발췌 위치: `af4e35ca7d92:tests/failure/test_factory_failure.cpp`는 create failure, clone failure, observed allocation attempt를 각각 지정해 `PipelineBuilder::replace()`를 반복 호출하고 live counters와 target output을 비교합니다.
- 변경 전/후 차이: 대표 unknown/null rejection에 더해 creator → local guard → clone → partial candidate → final swap 전 구간의 결정적 failure injection이 추가되었습니다. production 코드는 변경되지 않습니다.
- 직접 확인한 ownership/lifetime/state 관계: custom creator가 만든 formatter는 local `FormatterOwner`가 먼저 소유하고, append 성공 시 candidate가 별도 clone을 소유합니다. tests는 각 실패 뒤 formatter live count와 allocation live-block baseline이 복구되고 original target behavior가 동일한지 함께 확인합니다.
- 직접 확인한 failure path: create 자체의 throw, returned object를 clone하는 throw, 문자열·container 등 관찰된 allocation 실패가 final swap 전에 발생하도록 failure index를 이동합니다. 누수만 검사하지 않고 target transformation 결과도 비교해 premature mutation을 검출합니다. stream transport failure는 범위 밖입니다.
- 실행한 테스트와 결과: 미실행. failure double과 sweep loop, Make target을 검사했으며 test executable은 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: factory assembly의 모든 관찰 ownership handoff 실패에서 cleanup과 target 보존을 동시에 검증했습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `4c34654a4602` | creator abstraction, grammar, raw-pointer ownership transfer 경계 도입 | `create()`의 exact grammar 분기, abstract creator, virtual destructor, raw `Formatter *` 반환을 확인했습니다. | raw pointer 성공 결과를 인수할 guard와 multi-step publication은 아직 없습니다. |
| `fc0b8b7a40a0` | factory result를 즉시 RAII guard가 소유하지만 target은 incremental mutation 상태 | `FormatterOwner`가 creator 결과를 즉시 삭제 책임으로 감싸고 `append()`가 별도 clone을 target에 저장합니다. | target을 먼저 비우고 직접 append하므로 later failure에 old state가 사라지거나 partial state가 노출됩니다. |
| `907bfbd5c37c` | 완성된 candidate만 non-throwing `swap()`으로 publish하도록 transaction boundary 수정 | 모든 create/append를 local `candidate`에 적용하고 마지막 `target.swap(candidate)`만 target을 변경합니다. | 대표/전체 failure regression과 process output 증거는 후속 commits가 필요합니다. |
| `466d7abdb60f` | 중간 unknown formatter와 invalid null/count에서 기존 target 보존 regression | seeded target + unknown/null-count rejection과 CLI nonzero/stderr/empty-stdout fixture로 대표 경로를 고정합니다. | 모든 create/clone/allocation failure position을 순회하지는 않습니다. |
| `af4e35ca7d92` | create/clone/allocation failure 전 지점에서 cleanup과 target preservation 검증 | custom creator, counted clone, `FailingNew` sweep가 live baselines와 target behavior를 모든 관찰 실패 지점에서 비교합니다. | stream/CLI transport failure와 관찰되지 않은 allocator path는 범위 밖입니다. |

## Failure → Fix → Test 연결

- `fc0b8b7a40a0` 초기 builder: resource cleanup은 수행하지만 target을 incremental하게 변경합니다.
- `907bfbd5c37c` fix: complete candidate를 만든 뒤 one non-throwing swap으로 publish합니다.
- `466d7abdb60f` regression: 대표 rejection에서 seeded target과 CLI success output이 보존되는지 확인합니다.
- `af4e35ca7d92` failure sweep: create/clone/allocation 전 지점의 cleanup과 target preservation을 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: raw factory result를 RAII로 정리하는 것만으로는 여러 step을 교체하는 target의 이전 상태까지 보호되지 않습니다.
- 이를 드러낸 실제 failure 또는 test gap: `fc0b8b7a40a0`은 target을 먼저 비운 뒤 append하므로 두 번째 이후 specification의 create/clone 실패에서 누수는 없어도 empty 또는 partial target을 남깁니다.
- 수정/강화된 decision: validation 후 모든 throw 가능한 creation과 cloning을 local `FormatPipeline candidate`에서 끝내고, target에는 non-throwing `swap()` 한 번만 수행합니다.
- 해당 코드 위치: `fc0b8b7a40a0:src/Factory.cpp`의 `target.swap(empty)`/direct append와 `907bfbd5c37c:src/Factory.cpp`의 candidate assembly/final swap.
- 이를 고정하는 regression/evidence: `466d7abdb60f:tests/test_factory.cpp`와 CLI fixture, `af4e35ca7d92:tests/failure/test_factory_failure.cpp`.

## Ownership / state / responsibility 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: creator abstraction, grammar, raw-pointer ownership transfer 경계 도입
- Thread 종료 상태: create/clone/allocation failure 전 지점에서 cleanup과 target preservation 검증
- [x] 중간 commit마다 owner/state publisher/cleanup 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned state가 함께 등장하면 각각의 lifetime 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `4c34654a4602`: creator가 specification을 concrete formatter로 변환하고 raw ownership을 caller에게 넘깁니다.
2. `fc0b8b7a40a0`: local guard가 creator object를 정리하고 pipeline이 clone을 소유하지만, state publisher가 target의 clear와 각 append로 분산됩니다.
3. `907bfbd5c37c`: publication 책임이 final `target.swap(candidate)` 하나로 이동하고, candidate가 모든 partial resource의 cleanup owner가 됩니다.
4. `466d7abdb60f`: representative rejection과 실제 CLI에서 prior state/empty stdout을 확인합니다.
5. `af4e35ca7d92`: create·clone·allocation failure 위치별로 guard, candidate, target의 ownership과 state baseline을 검사합니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `specification → creator raw pointer → local RAII owner → pipeline clone into candidate → final swap into target`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준으로 `PipelineBuilder::replace()`는 null/count와 capacity를 먼저 검사하고, 각 specification을 `FormatterCreator::create()`로 변환합니다. raw pointer는 즉시 `FormatterOwner`에 들어가며 `candidate.append()`가 별도 polymorphic clone을 만듭니다. 모든 step이 성공한 경우에만 `target.swap(candidate)`가 실행됩니다.

초기 factory grammar와 비교하면 resource ownership handoff뿐 아니라 multi-step state replacement에 transaction boundary가 생겼습니다. 실패 시 새 formatter와 partial candidate는 local owners가 정리하고 old target은 계속 관찰됩니다. 보장 범위에는 source input이나 외부 stream position rollback이 포함되지 않으며, 테스트가 관찰하지 않은 환경 전체에 대한 형식 증명도 아닙니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: specification array → `PipelineBuilder::replace()` validation → `FormatterCreator::create()` → local `FormatterOwner` → `FormatPipeline::append()`/virtual `clone()` into candidate → `target.swap(candidate)`.
- 핵심 상태 필드: target/candidate의 `Formatter *steps_[max_steps]`, `size_`; local owner의 `Formatter *formatter_`.
- resource owner / borrowed view: creator 반환 직후 local guard가 원본을 소유하고 append 인자는 borrowed reference이며 candidate가 clone을 소유합니다. target은 commit 전까지 기존 clones를 소유합니다.
- commit point: 모든 specification 처리 성공 뒤의 단일 non-throwing `target.swap(candidate)`입니다.
- cleanup path: create 실패 전에는 pointer가 없고, create 후 실패는 `FormatterOwner`가 원본을 삭제하며 candidate destructor가 성공한 clone prefix를 삭제합니다. swap 후에는 candidate가 old target을 파괴합니다.
- 최종 invariant 설명: factory assembly 중 완성되지 않은 pipeline은 local candidate 밖으로 노출되지 않고, 모든 관찰 create/clone/allocation failure에서 target behavior와 ownership baseline이 유지됩니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 03-factory-transaction-boundary.md =====

===== BEGIN FILE: 04-scalar-text-target-projection.md =====
# Scalar Text Is Separated from Target Projection

## Thread 목표

입력 text의 문법/의미 판정과 각 target type으로의 representability/rendering을 분리하고, locale·negative zero·overflow·underflow 경계를 보존한 뒤 완성된 4-line report만 publish하는 흐름을 복원합니다.

**Source significance:** parsing과 rendering을 별도 책임으로 발전시킵니다. permissive extraction이나 host locale가 source meaning을 바꾸지 못하게 하고, 4개 projection이 partial prefix가 아니라 하나의 deterministic report로 publish되게 합니다.

## 이 Thread를 이해하기 위한 핵심 질문

- 하나의 parser가 source literal의 의미를 먼저 정규화해야 하는 이유는 무엇인가?
- classic locale 고정과 complete-input 검증이 permissive stream parsing의 어떤 문제를 막는가?
- textual zero와 nonzero-underflow-to-zero를 어떻게 구분해야 하는가?
- negative zero의 sign을 일반 비교와 별개로 보존해야 하는 이유는 무엇인가?
- temporary stream staging이 보장하는 atomicity와 보장하지 않는 destination-stream 영역은 무엇인가?

## 완료 기준

- [x] character/integer/floating/special 분류가 projection 이전에 끝나는 실제 경로를 추적할 수 있다.
- [x] float suffix, whitespace, trailing bytes, overflow, nonzero underflow 경계를 테스트와 parser 코드에서 대응시킬 수 있다.
- [x] float/double projection에서 representability 판단과 canonical rendering을 구분할 수 있다.
- [x] caller locale/format flags와 staged output의 관계를 테스트로 확인할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- accepted text는 complete ASCII grammar와 일치해야 하며 `LONG_MIN`, negative zero, finite overflow, nonzero underflow 같은 의미 경계를 보존한다.
- 완성되지 않은 report는 publish하지 않는다.
- deterministic rendering은 locale과 caller stream formatting state의 영향을 받지 않는다.

### Major engineering difficulty

- locale drift 없이 floating literal을 parsing하면서 negative zero를 보존하고 silent nonzero underflow를 거부.
- classic-locale rendering과 caller stream-state noninterference.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `6a3d0461faab` | feat(scalar): scalar 리터럴 문법과 종류 분류 | A | PARSING, ARCH, NUMERIC | 명시적 scalar grammar와 intermediate semantic representation을 만듭니다. |
| 2 | `a863f4899a93` | feat(scalar): locale 고정 수치 추출과 경계 보존 | A | NUMERIC, PARSING, HARD | locale independence, negative zero, overflow, nonzero-underflow 경계를 보존합니다. |
| 3 | `fc7faa10dc66` | test(scalar): literal 문법과 수치 범위 검증 | A | TEST, NUMERIC, EDGE | valid/invalid grammar와 numerical edge conditions를 고정합니다. |
| 4 | `7cdcec341fb1` | feat(scalar): 부동소수점 표현과 원자 출력 구현 | A | NUMERIC, DETERMINISM, EXCEPTION | canonical float/double projection과 staged whole-report rendering을 추가합니다. |
| 5 | `afea789fd753` | test(scalar): 변환 가능성·출력·CLI 오류 검증 | A | TEST, NUMERIC, DETERMINISM | exact output, stream noninterference, public headers, CLI failure를 검증합니다. |

## Commit별 학습 기록

### `6a3d0461faab` — feat(scalar): scalar 리터럴 문법과 종류 분류

- Importance: **A**
- Tags: **PARSING, ARCH, NUMERIC**
- Source 역할: 명시적 scalar grammar와 intermediate semantic representation을 만듭니다.
- Source classification summary: Introduces an intermediate scalar-literal model and explicit ASCII grammar.

#### 핵심 설계 / failure boundary 확인
- [x] scalar literal parser가 character/integer/floating/special 종류를 결정하는 grammar branches를 찾으세요.
- [x] ASCII byte 기준 검사와 surrounding whitespace/trailing material rejection을 어떤 helper가 담당하는지 확인하세요.
- [x] lone non-digit character가 character literal로 우선되는 precedence를 실제 branch order에서 확인하세요.
- [x] parser가 즉시 출력하지 않고 normalized intermediate representation을 만드는 상태 필드/enum을 기록하세요.
- [x] special value와 negative-zero recognition이 여러 projection에 중복되지 않고 parser 단계에 모이는지 call graph로 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `a863f4899a93`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ScalarLiteral.hpp`의 `LiteralKind`, `ScalarLiteral`, `ScalarParseError`, `parseScalarLiteral()`; `src/ScalarLiteral.cpp`의 byte/grammar helpers.
- 핵심 코드 발췌 위치: `6a3d0461faab:src/ScalarLiteral.cpp`에서 입력 byte를 먼저 검사하고 special, lone printable non-digit character, finite grammar 순서로 분류합니다. 결과는 `kind`, `value`, `float_suffix`, `negative_zero`를 가진 intermediate object입니다.
- 변경 전/후 차이: input text를 target별 출력 함수에서 즉석 해석하는 대신, source literal의 종류와 의미를 먼저 하나의 parser가 결정하는 내부 representation이 생겼습니다.
- 직접 확인한 ownership/lifetime/state 관계: parser는 입력 `std::string`을 borrowed read-only source로 사용하고 값만 `ScalarLiteral`에 복사합니다. parser state는 호출 범위의 local value이며 아직 destination stream에 쓰지 않습니다.
- 직접 확인한 failure path: empty text, NUL, non-ASCII, surrounding/embedded whitespace, incomplete number, trailing bytes는 `ScalarParseError`로 끝납니다. lone non-digit character branch가 finite parsing보다 먼저라 printable 단일 문자는 character로 고정됩니다. 이 시점의 numeric extraction 경계는 후속 hardening 전 상태입니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 parser declaration과 implementation을 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: complete ASCII scalar grammar를 target projection 앞의 normalized semantic value로 분리했습니다.

### `a863f4899a93` — feat(scalar): locale 고정 수치 추출과 경계 보존

- Importance: **A**
- Tags: **NUMERIC, PARSING, HARD**
- Source 역할: locale independence, negative zero, overflow, nonzero-underflow 경계를 보존합니다.
- Source classification summary: Hardens locale-independent numeric extraction, suffix grammar, negative zero, overflow, and underflow handling.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `6a3d0461faab`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] numeric extraction stream/locale가 classic locale로 고정되는 지점을 찾고 host locale 사용을 차단하는 방식을 기록하세요.
- [x] `f` suffix를 허용하기 전에 decimal point 또는 exponent 존재를 요구하는 grammar branch를 확인하세요.
- [x] textual zero와 nonzero mantissa가 machine zero로 underflow한 경우를 구분하는 검사 순서를 추적하세요.
- [x] overflow와 silent nonzero underflow rejection이 stream extraction success 여부와 별도로 검사되는지 확인하세요.
- [x] negative zero lexeme의 sign이 일반 `value == 0` 비교와 별도 상태로 보존되는 코드를 찾으세요.
- [x] non-ASCII/malformed byte rejection과 printable single-character precedence가 충돌하지 않는 분기 순서를 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `fc7faa10dc66`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ScalarLiteral.cpp`의 `validateFiniteGrammar()`, `allMantissaDigitsAreZero()`, `extractFiniteValue()`, `parseScalarLiteral()`; `src/ScalarLiteral.hpp`의 `negative_zero`.
- 핵심 코드 발췌 위치: `a863f4899a93:src/ScalarLiteral.cpp`는 numeric stream에 `std::locale::classic()`을 적용하고 `input.fail() || !input.eof()`를 검사합니다. `f` suffix는 point나 exponent가 있을 때만 허용하며, nonzero mantissa가 extraction 후 `0.0`이면 underflow로 거부합니다.
- 변경 전/후 차이: permissive numeric extraction에 의존하던 경계를 classic-locale complete parse, finite overflow, nonzero-underflow, suffix grammar, negative-zero 보존으로 강화했습니다.
- 직접 확인한 ownership/lifetime/state 관계: lexeme의 sign과 mantissa-zero 여부는 machine `double`과 별도로 `negative_zero`에 보존됩니다. textual all-zero는 parser가 직접 `+0.0`/`-0.0`을 만들고, nonzero lexeme만 stream extraction을 거칩니다.
- 직접 확인한 failure path: `42f`는 point/exponent가 없어 거부되고 `1e309` 같은 finite overflow는 fail/non-finite 검사로 거부됩니다. `1e-9999`처럼 nonzero digits가 machine zero가 되면 all-zero가 아니므로 거부됩니다. `-0`, `-0.0`, `-0e10`은 zero이면서 sign metadata를 유지합니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 grammar 및 numeric boundary code를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: locale와 machine conversion이 source text의 overflow, underflow, negative-zero 의미를 바꾸지 못하게 했습니다.

### `fc7faa10dc66` — test(scalar): literal 문법과 수치 범위 검증

- Importance: **A**
- Tags: **TEST, NUMERIC, EDGE**
- Source 역할: valid/invalid grammar와 numerical edge conditions를 고정합니다.
- Source classification summary: Adds exhaustive scalar grammar and numerical-boundary tests.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `a863f4899a93`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] character precedence, signed integer, decimal/exponent, float suffix, negative zero, inf/NaN의 accepted cases를 test table/fixtures에서 분류하세요.
- [x] whitespace, embedded NUL/non-ASCII, trailing garbage, malformed exponent, overflow, nonzero-underflow rejection cases를 각각 production parser branch에 연결하세요.
- [x] complete token 소비를 증명하는 test가 stream prefix-parse만 성공하는 잘못된 구현을 어떻게 잡는지 확인하세요.
- [x] literal grammar failure와 numerical representability failure가 test expectation에서 구분되는지 기록하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **scalar complete grammar와 numeric boundary preservation**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **malformed tokens, overflow, nonzero underflow, negative-zero/special boundaries**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **boundary-oriented unit suite**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **literal parser and numeric extraction paths**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **accepted/rejected source language 경계가 고정됨**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **최종 4-line rendering/CLI atomicity는 후속 commits가 담당**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic boundary regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `7cdcec341fb1`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_scalar_literal.cpp`; test suite registration; literal parser 관련 expected exception helpers.
- 핵심 코드 발췌 위치: `fc7faa10dc66:tests/test_scalar_literal.cpp`의 accepted tables/cases는 character, signed integer, point/exponent, `f` suffix, specials, negative zero를 다루고 rejected cases는 whitespace, NUL/non-ASCII, trailing garbage, malformed exponent, `42f`, overflow와 nonzero underflow를 포함합니다.
- 변경 전/후 차이: parser implementation의 분기별 source-language boundary가 deterministic unit regression으로 고정되었습니다. 최종 4-line rendering과 CLI output은 아직 이 commit의 주 대상이 아닙니다.
- 직접 확인한 ownership/lifetime/state 관계: tests는 반환 `ScalarLiteral`의 kind/value/suffix/sign metadata를 직접 비교하며 destination stream이나 외부 state를 만들지 않습니다.
- 직접 확인한 failure path: prefix만 읽는 parser라면 통과할 `1.0x`, malformed exponent, embedded NUL을 expected rejection으로 둬 complete consumption을 검사합니다. overflow와 nonzero-underflow도 grammar success와 별개로 exception을 기대해 numerical boundary를 분리합니다.
- 실행한 테스트와 결과: 미실행. test cases와 production branch mapping을 검사했으나 unit binary는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: scalar source language의 승인·거부 경계를 수치 한계까지 회귀 테스트로 고정했습니다.

### `7cdcec341fb1` — feat(scalar): 부동소수점 표현과 원자 출력 구현

- Importance: **A**
- Tags: **NUMERIC, DETERMINISM, EXCEPTION**
- Source 역할: canonical float/double projection과 staged whole-report rendering을 추가합니다.
- Source classification summary: Adds float and double projections plus classic-locale staged rendering.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `fc7faa10dc66`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] float와 double projection의 finite-range 및 nonzero-underflow checks를 casting 전에 수행하는 위치를 찾으세요.
- [x] negative zero, NaN, infinity, finite precision, `.0` suffix를 canonical spelling으로 만드는 rendering helpers를 확인하세요.
- [x] caller stream이 아니라 classic-locale temporary stream에 4개 projection line을 먼저 쓰는 순서를 추적하세요.
- [x] temporary rendering이 성공한 뒤 caller destination에 한 번에 bytes를 전달하는 publication point를 찾으세요.
- [x] destination 자체가 final write 중 fail하는 경우까지 rollback하지 않는 boundary를 실제 write structure에서 확인하세요.
- [x] caller locale/precision/flags를 읽거나 수정하지 않고 result가 고정되는지 implementation을 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `afea789fd753`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ScalarConverter.cpp`의 `canProjectChar()`, `canProjectInt()`, `canProjectFloat()`, `finiteNumber()`, projection writers, `ScalarConverter::write()`.
- 핵심 코드 발췌 위치: `7cdcec341fb1:src/ScalarConverter.cpp`에서 float projection은 range를 검사하고 cast 결과가 zero가 되는 nonzero 값을 거부합니다. 네 projection은 classic-locale `std::ostringstream rendered`에 모두 기록된 뒤 `result` bytes가 destination에 한 번 `output.write()` 됩니다.
- 변경 전/후 차이: normalized literal에 char/int/float/double representability와 canonical spelling을 적용하는 출력 계층이 추가되었습니다. caller stream에 line을 하나씩 직접 쓰지 않고 report 전체를 먼저 staging합니다.
- 직접 확인한 ownership/lifetime/state 관계: parser result와 `rendered`/`result`는 local candidate state입니다. caller stream은 final write 전까지 변경되지 않습니다. caller의 locale, precision, flags는 읽거나 바꾸지 않고 temporary stream만 classic locale과 자체 precision을 사용합니다.
- 직접 확인한 failure path: parse/projection/rendering 중 예외가 나면 destination write에 도달하지 않아 partial line이 없습니다. float overflow나 nonzero-underflow는 `impossible` projection으로 표현됩니다. 그러나 final `output.write()` 자체가 중간에 실패한 경우 destination bytes나 stream position을 rollback하는 코드는 없습니다.
- 실행한 테스트와 결과: 미실행. projection과 staged publication implementation을 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: source 의미를 target별로 투영해 classic-locale 4-line report를 완성한 뒤 한 번에 게시합니다.

### `afea789fd753` — test(scalar): 변환 가능성·출력·CLI 오류 검증

- Importance: **A**
- Tags: **TEST, NUMERIC, DETERMINISM**
- Source 역할: exact output, stream noninterference, public headers, CLI failure를 검증합니다.
- Source classification summary: Verifies exact projections, locale independence, stream-state preservation, public headers, and CLI errors.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `7cdcec341fb1`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] printable/control character, escapes, integer bounds, finite float/double, NaN/inf, negative zero, one-target-only representability cases의 exact 4-line expectations를 확인하세요.
- [x] test가 locale와 stream formatting state를 변경한 뒤에도 canonical output과 caller state 보존을 어떻게 assert하는지 기록하세요.
- [x] public-header compile test가 converter를 private parser 없이 사용할 수 있음을 어떻게 확인하는지 보세요.
- [x] CLI invalid literal이 nonzero status와 empty stdout를 보장하는 fixture를 찾으세요.
- [x] unit/compile/CLI 각각이 parser, projection, integration 중 어느 production path를 증명하는지 구분하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **scalar representability, canonical output, caller stream noninterference**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **type-specific impossible cases, locale/flags drift, invalid CLI input**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **exact-output unit + stream-state manipulation + public compile + CLI fixture**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **ScalarConverter projection/render + process adapter**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **완성된 scalar subsystem의 public/process contract**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **destination stream final-write rollback까지 보장하지는 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **broad integration + deterministic regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_scalar_converter.cpp`, `tests/compile/scalar_headers.cpp`, scalar private-construction compile-fail case, `tests/check_cli.sh`의 scalar fixtures, `Makefile` targets.
- 핵심 코드 발췌 위치: `afea789fd753:tests/test_scalar_converter.cpp`는 printable/control char, escapes, int bounds, finite/special float/double, negative zero, target별 impossible output의 정확한 네 줄을 비교합니다. caller stream locale/flags/precision을 변경한 뒤 output과 기존 state도 비교합니다.
- 변경 전/후 차이: parser unit 경계 위에 projection correctness, byte-level canonical output, stream noninterference, public header visibility, invalid CLI behavior가 추가되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: tests는 동일 input의 report가 caller formatting state와 무관하고, `ScalarConverter` 내부 private parser를 외부 consumer가 알 필요 없음을 compile contract로 확인합니다.
- 직접 확인한 failure path: invalid literal은 `InvalidScalar`를 기대하고 destination buffer가 empty인지 검사합니다. CLI fixture는 nonzero status와 empty stdout, diagnostic stderr를 비교합니다. 이 evidence도 destination의 final write failure rollback까지는 다루지 않습니다.
- 실행한 테스트와 결과: 미실행. exact expectations, compile units, CLI fixtures를 검사했으나 binary/command는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: scalar subsystem의 target별 표현 가능성, canonical bytes, public/process contract를 검증했습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `6a3d0461faab` | ASCII scalar grammar와 intermediate semantic representation 도입 | `LiteralKind`/`ScalarLiteral`과 explicit ASCII branch order로 character/finite/special 의미를 projection 전에 정규화합니다. | numeric extraction의 locale·overflow·underflow·suffix 세부 경계는 아직 강화 전입니다. |
| `a863f4899a93` | classic locale, suffix grammar, negative zero, overflow/nonzero-underflow 경계 강화 | classic-locale complete extraction, point/exponent 없는 `f` rejection, all-zero 판정, finite overflow 및 nonzero-underflow rejection을 확인했습니다. | 최종 target representability와 four-line publication은 아직 없습니다. |
| `fc7faa10dc66` | 문법과 수치 경계를 boundary-oriented tests로 고정 | accepted/rejected tables가 whitespace/NUL/trailing/malformed/overflow/underflow/negative-zero/special 분기를 production parser에 연결합니다. | rendering bytes와 CLI/output atomicity는 검증하지 않습니다. |
| `7cdcec341fb1` | float/double projection과 temporary-stream staged whole-report rendering 도입 | char/int/float/double projection, canonical finite/special spelling, classic temporary stream, final single `write()`를 확인했습니다. | destination stream의 실제 final-write partial failure는 rollback하지 않습니다. |
| `afea789fd753` | exact output, locale independence, stream-state noninterference, CLI failure 검증 | exact four-line output, locale/flags preservation, public compile, invalid CLI empty stdout를 검사합니다. | 모든 possible destination streambuf failure와 platform floating implementation을 형식적으로 증명하지는 않습니다. |

## Failure → Fix → Test 연결

- 명시적 fix commit은 이 Thread에 없습니다. parsing boundary를 `6a3d0461faab`/`a863f4899a93`에서 강화하고, `fc7faa10dc66`에서 grammar/numeric regression을 고정합니다.
- `7cdcec341fb1`은 whole-report staging을 도입하고, `afea789fd753`은 exact output/stream-state/CLI failure를 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: stream extraction이 prefix만 받아들이거나 caller/host locale에 의존하면 동일 text가 다른 의미로 승인될 수 있고, projection을 즉시 출력하면 뒤 단계 failure 전에 partial report가 노출됩니다.
- 이를 드러낸 실제 failure 또는 test gap: suffix·trailing byte·overflow·nonzero-underflow·negative zero는 단순 `double` 값만으로 구분되지 않으며, 정상 output case만으로 caller formatting state 오염이나 invalid input의 partial stdout을 잡을 수 없습니다.
- 수정/강화된 decision: parser가 complete ASCII grammar와 semantic metadata를 먼저 만들고, projection은 target representability만 판단합니다. renderer는 classic-locale temporary stream에서 네 줄을 완성한 뒤 destination에 한 번 씁니다.
- 해당 코드 위치: `6a3d0461faab`/`a863f4899a93:src/ScalarLiteral.cpp`, `7cdcec341fb1:src/ScalarConverter.cpp`.
- 이를 고정하는 regression/evidence: `fc7faa10dc66:tests/test_scalar_literal.cpp`, `afea789fd753:tests/test_scalar_converter.cpp`, compile/CLI fixtures.

## Responsibility 변화

- Source 기준 흐름: parser가 source literal 의미를 정규화하고, projection이 target representability를 판단하며, renderer가 classic-locale canonical report를 staging합니다.
- [x] 각 책임이 실제 어떤 class/helper/function에 위치하는지 SHA별로 기록하세요.
- [x] parsing failure, projection impossibility, formatting failure의 경계를 구분하세요.

### 코드 검사로 복원한 변화

1. `6a3d0461faab`: parser가 source token의 kind와 normalized value를 만들고 destination output 책임을 갖지 않습니다.
2. `a863f4899a93`: parser 책임에 classic locale, complete consumption, negative-zero metadata, finite overflow와 nonzero-underflow 판정이 추가됩니다.
3. `fc7faa10dc66`: source-language acceptance boundary가 parser-level tests로 고정됩니다.
4. `7cdcec341fb1`: converter가 target별 representability와 canonical rendering을 담당하고 complete report만 destination에 게시합니다.
5. `afea789fd753`: exact bytes, caller stream noninterference, public header, process failure behavior가 별도 evidence로 추가됩니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `ASCII token → literal classification/normalized meaning → per-target representability → canonical render → staged four-line publication`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준으로 `ScalarConverter::write()`는 먼저 `scalar_detail::parseScalarLiteral()`을 호출해 source text를 `ScalarLiteral`로 정규화합니다. 이후 char/int/float/double writer가 동일 semantic value를 각 target 범위에 맞춰 `impossible`, `Non displayable`, canonical finite/special text로 변환합니다. 네 줄은 local classic-locale stream에서 완성된 후 destination에 전달됩니다.

시작 시점과 비교하면 text parsing과 target projection이 분리되어 locale, trailing input, negative zero, overflow와 underflow 경계를 잃지 않습니다. invalid input이나 local rendering failure는 destination에 아무 prefix도 쓰지 않습니다. 남는 경계는 destination stream의 final write failure rollback과 테스트 matrix 밖 floating/platform 차이입니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: input `std::string` → `ScalarConverter::write()` → `parseScalarLiteral()` → `ScalarLiteral` → char/int/float/double projection helpers → local `ostringstream rendered` → destination `output.write()`.
- 핵심 상태 필드: `ScalarLiteral::{kind, value, float_suffix, negative_zero}`와 local rendered byte string.
- resource owner / borrowed view: input text와 destination stream은 caller-owned borrowed objects이고 parser result, temporary stream, result string은 call-local owners입니다.
- commit point: 네 projection line을 모두 성공적으로 만든 뒤 실행하는 final `output.write(result.data(), result.size())`입니다.
- cleanup path: grammar/numeric failure는 `ScalarParseError`에서 public `InvalidScalar`로 변환되고 local candidates가 자동 파괴됩니다. projection/rendering failure도 final write 전이면 destination은 untouched입니다.
- 최종 invariant 설명: accepted text는 complete classic ASCII grammar와 semantic sign/range를 보존하고, target별 판단과 canonical rendering은 caller stream state와 무관하며 incomplete report는 publish되지 않습니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 04-scalar-text-target-projection.md =====

===== BEGIN FILE: 05-checked-rpn-undefined-arithmetic.md =====
# Checked RPN Evaluation Avoids Undefined Arithmetic

## Thread 목표

signed `long` 연산을 실행한 뒤 overflow를 검사하는 잘못된 접근을 피하고, token parser와 stack grammar 위에서 모든 산술 precondition을 먼저 검증하는 구현을 복원합니다.

**Source significance:** signed overflow를 실행한 뒤 감지하면 이미 늦습니다. evaluator는 `LONG_MIN`의 비대칭 magnitude까지 고려해 limit/magnitude reasoning을 실제 arithmetic 실행 전에 수행합니다.

## 이 Thread를 이해하기 위한 핵심 질문

- `LONG_MIN`이 양수 최대값보다 magnitude가 1 큰 사실이 parsing과 multiplication에 어떤 영향을 주는가?
- operand pop 순서가 subtraction/division 의미에 어떻게 반영되는가?
- 각 연산에서 overflow 여부를 실제 연산 전에 어떻게 판정하는가?
- division의 두 특수 실패 조건은 무엇이며 왜 별도 검사가 필요한가?
- malformed token과 stack-shape 오류가 arithmetic helper까지 도달하지 않도록 어떤 계층이 막는가?

## 완료 기준

- [x] signed decimal token accumulation이 `LONG_MIN`까지 안전하게 도달하는 코드를 설명할 수 있다.
- [x] +, -, *, / 각각의 precondition check와 실제 signed operation의 순서를 실제 코드로 증명할 수 있다.
- [x] right-then-left pop과 non-commutative result를 테스트 케이스로 연결할 수 있다.
- [x] overflow/underflow/division-by-zero/malformed stack의 regression coverage를 구분할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- signed arithmetic은 실행 전에 검사되어 error detection 자체가 undefined overflow에 의존하지 않는다.
- accepted integer token은 complete ASCII grammar와 `LONG_MIN`/`LONG_MAX` 경계를 보존한다.

### Major engineering difficulty

- overflowing expression을 먼저 평가하지 않고 모든 signed `long` arithmetic 검사.
- `LONG_MIN`의 비대칭 magnitude를 고려한 multiplication/division 처리.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `57a25e8475ab` | feat(rpn): signed token과 stack 문법 처리 | A | PARSING, NUMERIC, CORE | complete signed decimal operand parsing과 stack-shape rules를 확립합니다. |
| 2 | `e1641a714172` | feat(rpn): overflow 검사 산술 연산 구현 | S | NUMERIC, HARD, CORE | 모든 signed operator에 실행 전 precondition checks를 추가합니다. |
| 3 | `aa0cc5e3e063` | test(rpn): 산술 경계와 잘못된 token 검증 | A | TEST, NUMERIC, EDGE | literal limits, overflow directions, operand order, malformed expression을 검증합니다. |

## Commit별 학습 기록

### `57a25e8475ab` — feat(rpn): signed token과 stack 문법 처리

- Importance: **A**
- Tags: **PARSING, NUMERIC, CORE**
- Source 역할: complete signed decimal operand parsing과 stack-shape rules를 확립합니다.
- Source classification summary: Introduces signed decimal token parsing and the structural RPN stack language.

#### 핵심 설계 / failure boundary 확인
- [x] ASCII space 규칙에 따른 token separation과 complete signed-decimal recognition을 담당하는 코드를 찾으세요.
- [x] magnitude accumulation이 overflow 없이 `LONG_MAX`와 `LONG_MIN`의 asymmetric magnitude를 모두 허용하는 계산을 추적하세요.
- [x] malformed number/unknown token을 operator 단계 전에 거부하는 branch를 확인하세요.
- [x] evaluation stack push/pop과 expression 종료 시 exactly one result를 요구하는 구조적 validation을 기록하세요.
- [x] locale-sensitive stream prefix parsing을 피하기 위해 manual parser가 사용되는 지점을 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `e1641a714172`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/RpnEvaluator.hpp`의 `RpnEvaluator::evaluate()`; `src/RpnEvaluator.cpp`의 `parseLong()`, token loop, `isOperator()`, local stack.
- 핵심 코드 발췌 위치: `57a25e8475ab:src/RpnEvaluator.cpp`의 `parseLong()`은 sign을 분리하고 `unsigned long magnitude`를 `(limit - digit) / 10`과 비교한 뒤 누적합니다. 음수 limit은 `LONG_MAX + 1`로 두어 `LONG_MIN`을 별도 branch에서 만듭니다.
- 변경 전/후 차이: locale-sensitive stream extraction 대신 ASCII space tokenization과 complete signed-decimal parser가 도입되었고, evaluator가 local `std::vector<long>` stack의 구조를 직접 검증하게 되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: expression은 caller-owned borrowed string이고 token substring과 evaluation stack은 call-local state입니다. operand token은 완전히 파싱된 후에만 stack에 push되며 결과는 종료 시 stack에 정확히 하나 남을 때만 반환됩니다.
- 직접 확인한 failure path: sign만 있는 token, unknown byte가 섞인 number, unknown operator/token, operand 부족, 종료 시 0개 또는 2개 이상 결과는 `invalid_argument` 또는 range exception으로 끝납니다. local stack은 외부 객체에 publish되지 않습니다. 이 SHA의 산술 operator에는 아직 모든 overflow precondition이 추가되기 전입니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 tokenizer/parser/stack code를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: `LONG_MIN`까지 안전하게 만드는 signed token parser와 RPN stack language를 확립했습니다.

### `e1641a714172` — feat(rpn): overflow 검사 산술 연산 구현

- Importance: **S**
- Tags: **NUMERIC, HARD, CORE**
- Source 역할: 모든 signed operator에 실행 전 precondition checks를 추가합니다.
- Source classification summary: Implements checked addition, subtraction, multiplication, and division before signed operations execute.

#### 이 commit 직전 상태와 문제
- 직전 관련 Thread SHA `57a25e8475ab`를 먼저 checkout하여 이 commit이 추가되기 전 representation/ownership/state-publication 방식을 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] addition/subtraction이 operand sign에 따라 `LONG_MIN`/`LONG_MAX` margin을 비교하는 helper/branch를 찾으세요.
- [x] multiplication이 signed multiplication 자체를 실행하기 전에 sign과 unsigned magnitude로 범위를 판단하는 과정을 단계별로 기록하세요.
- [x] division의 zero divisor와 `LONG_MIN / -1`을 실제 division 전에 차단하는 branch를 확인하세요.
- [x] operator 적용 시 stack에서 right operand를 먼저, left operand를 나중에 pop하는 코드를 확인하세요.
- [x] 각 helper에서 범위 검사가 통과한 뒤에만 signed operation expression이 평가됨을 실제 control flow로 증명하세요.
- [x] overflow/invalid operation exception이 local evaluation stack 밖에 partial result를 publish하지 않는 이유를 call scope로 설명하세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `aa0cc5e3e063`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/RpnEvaluator.cpp`의 `magnitudeOf()`, `checkedAdd()`, `checkedSubtract()`, `checkedMultiply()`, `checkedDivide()`, `applyOperator()`, `evaluate()`.
- 핵심 코드 발췌 위치: `e1641a714172:src/RpnEvaluator.cpp`에서 add/subtract는 sign별 limit 식을 먼저 검사합니다. multiply는 `-(value + 1) + 1` 형태의 unsigned magnitude로 `LONG_MIN`을 처리하고 `left_magnitude > limit / right_magnitude`를 실제 곱셈 전에 검사합니다.
- 변경 전/후 차이: 직전 parser/stack 구현 위에 모든 signed operator의 precondition-first arithmetic이 추가되었습니다. 결과를 계산한 뒤 overflow를 판정하는 방식은 사용하지 않습니다.
- 직접 확인한 ownership/lifetime/state 관계: operator token 처리 시 stack에서 `right`를 먼저, `left`를 나중에 꺼내 `applyOperator(left, right, op)`에 전달합니다. checked helper가 성공한 값만 다시 local stack에 push하므로 실패 결과는 외부나 stack에 게시되지 않습니다.
- 직접 확인한 failure path: addition/subtraction은 limit subtraction/addition으로 margin을 검사하고, multiplication은 sign에 따라 `LONG_MAX` 또는 `LONG_MAX + 1` magnitude limit을 사용합니다. division은 `right == 0`과 `LONG_MIN / -1`을 실제 `/` 전에 거부합니다. 모든 signed `+ - * /` expression은 해당 검사 뒤에만 평가됩니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 checked helper와 call order를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: signed arithmetic을 실행하기 전에 모든 overflow와 invalid division 조건을 판정하도록 만들었습니다.

### `aa0cc5e3e063` — test(rpn): 산술 경계와 잘못된 token 검증

- Importance: **A**
- Tags: **TEST, NUMERIC, EDGE**
- Source 역할: literal limits, overflow directions, operand order, malformed expression을 검증합니다.
- Source classification summary: Covers RPN syntax, operand order, all arithmetic boundaries, division by zero, and malformed stacks.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `e1641a714172`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] 정상 +,-,*,/와 subtraction/division operand order를 구분하는 test cases를 찾으세요.
- [x] `LONG_MIN`/`LONG_MAX` literal parsing과 모든 overflow/underflow direction을 각각 어떤 expression으로 재현하는지 기록하세요.
- [x] division by zero와 `LONG_MIN / -1` case가 별도 regression으로 존재하는지 확인하세요.
- [x] malformed number, unknown token, insufficient/extra operands, spacing boundaries가 parser/stack grammar의 어느 branch를 통과하는지 매핑하세요.
- [x] 테스트가 UB 발생 뒤 결과를 검사하는 것이 아니라 UB expression 자체가 실행되지 않도록 error path를 관찰하는 방식을 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **RPN grammar와 precondition-first checked arithmetic**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **all overflow/underflow directions, division by zero, malformed stack/tokens**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **boundary unit suite**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **token parser, stack evaluator, checked operator helpers**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **exercised signed operations이 UB 경계를 넘기 전에 거부됨**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **모든 가능한 긴 expression/state-space를 exhaustive하게 증명하지는 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic boundary regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_rpn_evaluator.cpp`, `tests/compile/rpn_headers.cpp`, RPN private-construction compile-fail unit, test registration/Make target.
- 핵심 코드 발췌 위치: `aa0cc5e3e063:tests/test_rpn_evaluator.cpp`는 normal operators와 `8 3 -`, `8 3 /` 같은 operand order, `LONG_MIN`/`LONG_MAX` literals, add/subtract/multiply 양방향 overflow, division by zero, `LONG_MIN -1 /`, malformed stack/token을 구분합니다.
- 변경 전/후 차이: checked-arithmetic implementation의 각 branch와 parser/stack shape가 deterministic boundary unit suite로 고정되었습니다. production code는 변경되지 않습니다.
- 직접 확인한 ownership/lifetime/state 관계: success case는 반환 `long`만 비교하고 failure case는 exception을 기대합니다. evaluator state는 매 호출마다 local stack이므로 실패 뒤 persistent target이나 partial result를 검사할 외부 객체는 없습니다.
- 직접 확인한 failure path: overflow expression은 결과 값을 관찰하지 않고 exception path를 기대하므로 checked helper가 실제 undefined expression을 실행하지 않아야 test가 sanitizer/정상 실행에서도 끝납니다. tab/newline, malformed sign, extra/insufficient operands도 parser 또는 final-size branch에 연결됩니다. 모든 가능한 긴 expression을 exhaustive하게 다루지는 않습니다.
- 실행한 테스트와 결과: 미실행. boundary cases와 public compile units를 검사했으나 unit binary는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: token limit, operand order, 모든 산술 방향과 malformed stack을 UB 전 거부 계약으로 고정했습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `57a25e8475ab` | signed decimal token grammar, exact limits, stack-shape language 도입 | manual sign/magnitude parser가 `LONG_MAX + 1` negative limit과 pre-multiply digit guard로 exact signed bounds를 만들고 local stack shape를 검사합니다. | operator overflow precondition은 아직 완전하지 않아 arithmetic safety는 후속 commit에 남습니다. |
| `e1641a714172` | 모든 operator에 precondition-first overflow/invalid-operation checks 도입 | add/sub sign margins, unsigned magnitude multiplication limit, zero 및 `LONG_MIN / -1` division guards가 실제 operation보다 먼저 실행됩니다. | 모든 긴 expression/state-space에 대한 exhaustive proof와 외부 caller side effect는 다루지 않습니다. |
| `aa0cc5e3e063` | long extremes, overflow directions, operand order, malformed expressions 검증 | literal extremes, non-commutative order, overflow/underflow directions, invalid division, malformed tokens/stacks를 deterministic cases로 검사합니다. | finite case set이므로 모든 가능한 token 길이와 expression 조합을 형식적으로 증명하지는 않습니다. |

## Failure → Fix → Test 연결

- 명시적 fix commit은 이 Thread에 없습니다. `e1641a714172`가 undefined signed overflow를 피하는 precondition-first decision을 구현합니다.
- `aa0cc5e3e063`이 long limits, 모든 overflow direction, division boundary, malformed expression을 회귀 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: signed overflow를 먼저 계산한 뒤 결과 범위를 검사하면 검사 자체가 이미 undefined behavior 뒤에 실행됩니다. 또한 `LONG_MIN`은 양의 `long`으로 직접 magnitude를 표현할 수 없습니다.
- 이를 드러낸 실제 failure 또는 test gap: 일반적인 `-value`나 `left * right` 기반 검사는 `LONG_MIN`과 overflow product에서 안전하지 않고, commutative operator만 시험하면 right/left pop 순서 오류도 놓칩니다.
- 수정/강화된 decision: token은 unsigned magnitude와 asymmetric limit으로 파싱하고, 각 operator는 sign·limit·magnitude precondition을 통과한 뒤에만 signed expression을 실행합니다.
- 해당 코드 위치: `57a25e8475ab:src/RpnEvaluator.cpp`의 `parseLong()`, `e1641a714172:src/RpnEvaluator.cpp`의 checked helpers와 `evaluate()` pop order.
- 이를 고정하는 regression/evidence: `aa0cc5e3e063:tests/test_rpn_evaluator.cpp`의 long extremes, 모든 overflow 방향, division special cases, malformed expression tests.

## State / responsibility 변화

- Source 기준 흐름: token parser가 complete signed operands를 만들고, evaluator stack이 구조를 소유하며, arithmetic helper가 signed operation 이전의 range precondition을 책임집니다.
- [x] parser stack과 arithmetic helper 사이에서 어떤 값이 전달되고 어디서 exception이 발생하는지 기록하세요.

### 코드 검사로 복원한 변화

1. `57a25e8475ab`: ASCII-space tokenizer와 unsigned magnitude parser가 complete signed operands를 만들고 local stack이 expression structure를 소유합니다.
2. `e1641a714172`: arithmetic responsibility가 checked helper로 분리되고, stack에는 precondition을 통과한 결과만 다시 들어갑니다.
3. `aa0cc5e3e063`: parser limits, pop order, 각 operator의 success/failure boundary가 deterministic unit cases로 연결됩니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `ASCII-space tokenization → signed operand parse → stack-shape validation → checked operator precondition → signed operation → single-result validation`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준으로 `RpnEvaluator::evaluate()`는 ASCII space만 separator로 사용해 token을 완전히 분리합니다. signed number는 `parseLong()`이 unsigned magnitude로 범위를 확인하고, operator는 stack에서 right와 left를 꺼낸 뒤 checked helper에 전달합니다. helper가 성공한 경우에만 result를 push하며 종료 시 exactly one value를 요구합니다.

시작 시점과 비교하면 token grammar와 stack grammar 위에 UB를 발생시키지 않는 arithmetic boundary가 생겼습니다. 실패는 call-local state에서 exception으로 끝나며 partial result를 외부에 게시하지 않습니다. 남는 경계는 유한한 regression suite 밖 expression 조합과 caller가 exception 이후 수행하는 외부 side effect입니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: expression string → `RpnEvaluator::evaluate()` token loop → `parseLong()` 또는 operator branch → right/left pop → `checkedAdd/Subtract/Multiply/Divide()` → validated result push → single final result 반환.
- 핵심 상태 필드: call-local `std::vector<long> stack`, token index, unsigned `magnitude`/`limit`, checked helper의 left/right values.
- resource owner / borrowed view: expression은 borrowed input이고 token strings와 stack은 evaluator call이 소유합니다. persistent heap ownership이나 external target은 없습니다.
- commit point: 각 operator는 checked helper 성공 뒤 `stack.push_back(result)`하고, 전체 evaluation은 final stack size가 1일 때 반환합니다.
- cleanup path: malformed token/stack, range failure, zero division, `LONG_MIN / -1`은 exception으로 local stack을 파괴합니다. overflowed signed expression은 실행되지 않습니다.
- 최종 invariant 설명: accepted operands는 exact `long` grammar와 limits를 만족하고, 모든 signed operation은 정의된 결과 범위가 확인된 뒤에만 실행되며 non-commutative operand order가 보존됩니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 05-checked-rpn-undefined-arithmetic.md =====

===== BEGIN FILE: 06-generic-containers-transactional-batch.md =====
# Generic Containers Converge in a Transactional Batch Engine

## Thread 목표

random-access template abstraction을 vector/deque 두 표현에 적용하고, record parsing·중복 검사·RPN 계산·정렬·stream 완료 판정까지 하나의 delayed-publication transaction으로 통합하는 과정을 복원합니다.

**Source significance:** generic containers, checked arithmetic, deterministic order, local candidates, delayed publication을 final subsystem에서 결합합니다. 후속 commit은 successful stream completion의 의미를 정교화하고, 어떤 협력 failure path도 partial batch를 publish하지 못함을 검증합니다.

## 이 Thread를 이해하기 위한 핵심 질문

- `std::sort` 사용이 template의 random-access requirement를 어떻게 실제 compile requirement로 만드는가?
- throwing element type에서도 batch copy/assignment가 어떤 보장을 유지해야 하는가?
- `BatchEngine::replace()`의 rollback value는 무엇이며 왜 보상 코드가 필요 없는가?
- canonical `(value, name)` total order가 입력 순열 독립성과 어떤 관계가 있는가?
- vector/deque 결과 불일치를 publish 전에 검사하는 이유는 무엇인가?
- 마지막 newline이 없는 정상 EOF와 실제 stream failure를 어떻게 구분하는가?
- syntax/arithmetic/stream/allocation failure sweep이 하나의 transaction invariant를 어떻게 검증하는가?

## 완료 기준

- [x] template requirement와 vector/deque substitution을 public header와 tests에서 확인할 수 있다.
- [x] batch replace의 candidate state, duplicate tracking, RPN result accumulation, final swap 위치를 그릴 수 있다.
- [x] sorting/serialization이 deterministic external behavior를 만드는 근거를 실제 comparator와 staging 코드에서 확인할 수 있다.
- [x] stream reader fix 전후에서 clean EOF/final unterminated line/failure의 분기를 비교할 수 있다.
- [x] seeded prior state가 모든 rejection path 뒤 유지되는 regression/failure-sweep 구조를 설명할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- 완성되지 않은 batch candidate는 publish되지 않는다.
- strong guarantee replacement는 parse/arithmetic/stream-read preparation/allocation 실패 시 prior observable state를 보존한다.
- batch output은 total `(value, name)` order를 가지며 input permutation과 repeated rendering에 불변이다.
- signed arithmetic은 실행 전에 검사된다.

### Major engineering difficulty

- vector/deque generic behavior와 deterministic total order, transactional publication을 동시에 유지.
- whole-stream batch replacement 중 partial target mutation 방지.
- clean final unterminated line과 actual stream failure 구분.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `708c025ef2a0` | feat(template): 임의 접근 container batch 추상화 추가 | S | ARCH, GENERIC, CORE | configurable random-access batch abstraction과 cross-container range comparison을 정의합니다. |
| 2 | `aaeff163baf8` | test(template): iterator·정렬·복사 실패 계약 검증 | A | TEST, GENERIC, EXCEPTION | iterator/algorithm/container substitution과 throwing-value behavior를 검증합니다. |
| 3 | `d0295f82614b` | feat(batch): 입력 문법과 원자 교체 구현 | S | ARCH, EXCEPTION, INTEGRATION | record parsing, duplicate detection, RPN evaluation, swap-on-success publication을 통합합니다. |
| 4 | `42d411e42268` | feat(batch): 결과 정렬과 직렬화 제공 | A | DETERMINISM, EXCEPTION, CORE | total result order와 classic-locale staged serialization을 추가합니다. |
| 5 | `af57a8f9c5fe` | feat(batch): 두 container의 정렬 결과 대조 | A | GENERIC, INTEGRATION, DETERMINISM | vector/deque candidates를 독립적으로 sort하고 disagreement를 commit 전에 거부합니다. |
| 6 | `9ba0e7c897ed` | test(batch): 입력 순열과 출력 결정성 검증 | A | TEST, DETERMINISM, EDGE | input permutation invariance와 byte-identical repeated output을 검증합니다. |
| 7 | `ea23237ad506` | fix(batch): 입력 stream 종료 상태를 명확히 구분 | A | DEBUG, PARSING, EDGE | clean EOF, final unterminated record, input failure를 명확히 구분합니다. |
| 8 | `b4ddd78fb9aa` | test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증 | A | TEST, EXCEPTION, RISK | syntax/arithmetic/stream/allocation failure 전 범위에서 seeded state preservation을 검증합니다. |

## Commit별 학습 기록

### `708c025ef2a0` — feat(template): 임의 접근 container batch 추상화 추가

- Importance: **S**
- Tags: **ARCH, GENERIC, CORE**
- Source 역할: configurable random-access batch abstraction과 cross-container range comparison을 정의합니다.
- Source classification summary: Introduces `RandomAccessBatch` over configurable random-access containers and cross-container range equality.

#### 이 commit 직전 상태와 문제
- 이 Thread의 첫 commit이므로, `git show <sha>^`가 가능한 경우 parent에서 관련 type/기능이 없거나 다른 형태였는지 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] `RandomAccessBatch<T, Container>` public header에서 value/container template parameters와 default `std::vector`를 확인하세요.
- [x] container의 iterator/const_iterator category를 그대로 expose하는 typedef와 begin/end API를 찾으세요.
- [x] `at()`, insertion, range iteration, `std::sort` 기반 sort, range equality가 어떤 container capability를 요구하는지 기록하세요.
- [x] `std::list` 같은 non-random-access container가 왜 sort instantiation 단계에서 맞지 않는지 실제 algorithm requirement와 연결하세요.
- [x] assignment의 copy-and-swap 구현과 underlying container copy failure 시 target preservation 경로를 확인하세요.
- [x] vector와 deque instantiation이 같은 abstraction을 통해 동작하도록 구현이 concrete representation에 의존하지 않는 지점을 찾으세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `aaeff163baf8`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/RandomAccessBatch.hpp`의 `RandomAccessBatch<T, Container>`, iterator typedefs, `values_`, `push_back()`, `at()`, `sort()`, `swap()`, `equal_ranges()`.
- 핵심 코드 발췌 위치: `708c025ef2a0:include/cppf/RandomAccessBatch.hpp`는 `Container = std::vector<T>`를 기본값으로 두고 container의 iterator/const_iterator를 그대로 노출합니다. `sort()`는 `std::sort(values_.begin(), values_.end(), compare)`를 호출하고 assignment는 complete copy 뒤 `swap()`합니다.
- 변경 전/후 차이: concrete vector 사용 대신 random-access operations를 만족하는 container parameter를 교체할 수 있는 header-only batch abstraction과 서로 다른 range를 비교하는 helper가 생겼습니다.
- 직접 확인한 ownership/lifetime/state 관계: `RandomAccessBatch`가 `Container values_`를 값으로 소유하며 iterator는 그 container lifetime과 mutation 규칙에 종속된 borrowed view입니다. copy는 underlying container의 독립 value copy이고 assignment commit은 container swap입니다.
- 직접 확인한 failure path: element/container copy가 실패하면 copy constructor는 underlying container가 partial elements를 정리하고, assignment는 local copy construction 단계에서 target `values_`를 건드리지 않습니다. `at()`은 범위 밖을 거부합니다. `std::list`는 template 선언 자체가 아니라 `std::sort` instantiation의 random-access requirement에서 제외됩니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 public template implementation을 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: vector/deque로 치환 가능한 random-access batch와 copy-and-swap value semantics를 정의했습니다.

### `aaeff163baf8` — test(template): iterator·정렬·복사 실패 계약 검증

- Importance: **A**
- Tags: **TEST, GENERIC, EXCEPTION**
- Source 역할: iterator/algorithm/container substitution과 throwing-value behavior를 검증합니다.
- Source classification summary: Tests iterators, algorithms, vector/deque substitution, copying, and throwing element types.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `708c025ef2a0`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] vector/deque 각각에서 mutable/const iterator와 standard algorithm을 사용하는 tests를 구분하세요.
- [x] checked access, sorting, range equality, copy, assignment, self-assignment의 expected behavior를 확인하세요.
- [x] throwing value type의 copy failure injection과 live-object counter가 어떻게 구성되는지 찾으세요.
- [x] failed construction에서 partial copied values leak가 없는지, failed assignment에서 destination 보존이 되는지 각각의 assertion을 기록하세요.
- [x] generic interface shape test와 exception guarantee test가 서로 어떤 production template instantiation을 사용하는지 구분하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **RandomAccessBatch generic interface와 value/exception semantics**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **vector/deque substitution, throwing element copy**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **multi-container unit + throwing test type/live counters**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **template instantiation, container copy/sort/assignment paths**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **generic capability와 failed-copy cleanup/target preservation**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **non-random-access container support를 증명하지 않으며 오히려 요구사항 밖으로 고정**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **broad generic contract + failure regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `d0295f82614b`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_random_access_batch.cpp`; throwing value test type와 live/copy counters; `tests/compile/template_headers.cpp`, `template_const_iterator_fail.cpp`, `template_list_sort_fail.cpp`; Make contract target.
- 핵심 코드 발췌 위치: `aaeff163baf8:tests/test_random_access_batch.cpp`는 vector와 deque instantiation에서 iterator/const_iterator, standard algorithms, access, sort, equality, copy/assignment/self-assignment를 사용합니다. throwing element는 지정 copy에서 예외를 발생시키고 live count를 노출합니다.
- 변경 전/후 차이: generic interface의 정상 치환만이 아니라 element copy failure에서 construction cleanup과 assignment target preservation을 확인하는 test layer가 추가되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: failed copy construction 뒤 성공했던 temporary elements가 사라져 live baseline으로 돌아와야 하고, failed assignment 뒤 destination values와 source values가 각각 유지되어야 합니다. iterator compile cases는 const batch의 mutation을 금지하는 public shape를 고정합니다.
- 직접 확인한 failure path: vector/deque의 element copying 중 throw를 위치별로 제어하고, `std::list` sort 사용은 expected compile failure로 다룹니다. 따라서 non-random-access support를 증명하는 것이 아니라 요구사항 밖임을 명시합니다.
- 실행한 테스트와 결과: 미실행. unit/compile-contract source와 기대 조건을 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: container 치환, algorithm 사용, throwing element에서의 cleanup과 strong assignment를 검증했습니다.

### `d0295f82614b` — feat(batch): 입력 문법과 원자 교체 구현

- Importance: **S**
- Tags: **ARCH, EXCEPTION, INTEGRATION**
- Source 역할: record parsing, duplicate detection, RPN evaluation, swap-on-success publication을 통합합니다.
- Source classification summary: Implements whole-stream batch parsing, uniqueness, RPN evaluation, and swap-on-success replacement.

#### 이 commit 직전 상태와 문제
- 직전 관련 Thread SHA `aaeff163baf8`를 먼저 checkout하여 이 commit이 추가되기 전 representation/ownership/state-publication 방식을 확인하세요.
- Source가 확정한 Problem/Decision을 실제 diff와 대응시키되, source에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 입력 record를 name과 RPN expression으로 split/trim하고 identifier grammar를 검사하는 helper chain을 찾으세요.
- [x] duplicate-name tracking 구조와 duplicate rejection이 candidate publication보다 앞에서 일어나는지 확인하세요.
- [x] 각 expression이 `RpnEvaluator`를 호출해 `JobResult` candidate에 추가되는 call graph를 그리세요.
- [x] existing `results_`와 분리된 local candidate result set/map이 whole-input 처리 동안 유지되는지 확인하세요.
- [x] empty input, malformed record, duplicate, evaluator error, stream failure 각각이 final swap에 도달하지 않는 branch를 기록하세요.
- [x] complete input 성공 뒤 candidate vector가 `results_`와 swap되는 단 하나의 publication point를 표시하세요.

#### Ownership / lifecycle / state transition
- [x] 상태 필드별 owner, lifetime, valid state를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 non-throwing commit operation의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 temporary/candidate state와 성공 후 published state를 구분해 그리세요.

#### Failure scenario와 보장 경계
- [x] source가 지목한 failure를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 observable state를 기록하세요.
- [x] 이 commit이 보장하는 것과 아직 보장하지 않는 것을 source와 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `42d411e42268`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/BatchEngine.hpp`의 `JobResult`, `BatchEngine::results_`, `replace()`; `src/BatchEngine.cpp`의 trim/name/line parser, duplicate map, `RpnEvaluator::evaluate()`, local candidate, final swap.
- 핵심 코드 발췌 위치: `d0295f82614b:src/BatchEngine.cpp`는 각 line을 한 개의 `|`로 분리하고 field trim/name grammar를 검사합니다. local candidate와 `std::map<std::string, long> seen`에만 결과를 누적하고 complete non-empty input 뒤 `results_.swap(candidate)`를 호출합니다.
- 변경 전/후 차이: generic container groundwork 위에 whole-stream parsing, duplicate rejection, checked RPN, persistent result replacement가 통합되었습니다. 기존 `results_`를 line마다 변경하지 않고 local state가 전체 입력을 소유합니다.
- 직접 확인한 ownership/lifetime/state 관계: input stream은 borrowed source이고 parsed strings, seen map, candidate results는 call-local owners입니다. `JobResult`가 name/value를 값으로 소유하며 final swap 전 `results_`는 prior batch를 계속 소유합니다.
- 직접 확인한 failure path: malformed/blank record, missing/extra separator, invalid name, duplicate name, RPN exception, stream failure, empty input은 final swap에 도달하지 않습니다. local containers가 partial records를 정리하므로 보상 mutation 없이 prior results가 유지됩니다. 이 시점의 line-loop EOF 판정은 후속 fix 전입니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 parser, evaluator call, candidate publication을 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: 전체 입력을 local candidate에서 검증·계산한 뒤 한 번만 교체하는 batch transaction을 만들었습니다.

### `42d411e42268` — feat(batch): 결과 정렬과 직렬화 제공

- Importance: **A**
- Tags: **DETERMINISM, EXCEPTION, CORE**
- Source 역할: total result order와 classic-locale staged serialization을 추가합니다.
- Source classification summary: Adds total result ordering and classic-locale staged batch serialization.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `d0295f82614b`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] result comparator가 value를 먼저, name을 tie-breaker로 비교해 total order를 만드는 실제 코드를 확인하세요.
- [x] sorting이 published `results_`가 아니라 candidate에서 일어나도록 state mutation 순서를 추적하세요.
- [x] serialization이 classic-locale temporary stream에서 완성된 bytes를 만든 뒤 destination으로 쓰는 staging을 확인하세요.
- [x] caller stream locale/flags를 덮어쓰지 않는지 implementation과 tests를 함께 확인하세요.
- [x] formatting stage의 failure는 partial record sequence를 publish하지 않지만 final destination write failure는 rollback 대상이 아님을 실제 code boundary로 기록하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `af57a8f9c5fe`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/BatchEngine.cpp`의 `resultLess()`, candidate sort, `BatchEngine::write()`; `JobResult::name()/value()`.
- 핵심 코드 발췌 위치: `42d411e42268:src/BatchEngine.cpp`의 comparator는 value를 먼저 비교하고 같으면 name을 비교합니다. candidate는 publication 전에 sort되고 `write()`는 classic-locale temporary stream에 모든 `value | name` row를 만든 뒤 destination에 한 번 씁니다.
- 변경 전/후 차이: 입력 순서대로 저장하던 result set에 `(value, name)` total order와 deterministic serialization이 추가되었습니다. sorting과 formatting 모두 published state나 caller stream을 준비 중간에 직접 변경하지 않습니다.
- 직접 확인한 ownership/lifetime/state 관계: sort 대상은 local candidate이며 성공 후 vector가 `results_`로 이동합니다. write candidate인 `ostringstream`와 string은 local owner이고 caller stream의 flags/locale는 수정하지 않습니다.
- 직접 확인한 failure path: sort comparison/element operation이나 local formatting이 실패하면 result publication 또는 destination write 전입니다. final destination `write()` 실패는 이미 보낸 bytes를 rollback하지 않습니다. comparator는 equal value에 name tie-breaker를 두어 insertion permutation에 의존하지 않습니다.
- 실행한 테스트와 결과: 미실행. comparator와 staged serializer를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: total order와 staged classic-locale serialization으로 batch 외부 결과를 결정적으로 만들었습니다.

### `af57a8f9c5fe` — feat(batch): 두 container의 정렬 결과 대조

- Importance: **A**
- Tags: **GENERIC, INTEGRATION, DETERMINISM**
- Source 역할: vector/deque candidates를 독립적으로 sort하고 disagreement를 commit 전에 거부합니다.
- Source classification summary: Runs and sorts vector- and deque-backed batches, then rejects disagreement before commit.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `42d411e42268`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] accepted job이 vector-backed와 deque-backed `RandomAccessBatch` candidate 양쪽에 어떻게 추가되는지 추적하세요.
- [x] 두 candidate가 동일 comparator로 독립 sort되는 호출을 확인하세요.
- [x] `equal_ranges()` 또는 대응 비교가 disagreement를 detect하는 위치와 `logic_error` 발생 전 publication 상태를 확인하세요.
- [x] 불일치 시 prior engine state가 유지되는 이유를 candidate lifetime과 final commit 순서로 설명하세요.
- [x] 추가 memory/sort work가 deliberate verification이라는 사실이 코드 구조에서 어떻게 드러나는지 기록하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `9ba0e7c897ed`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/BatchEngine.cpp`의 vector/deque `RandomAccessBatch` candidates, dual `push_back()`, `sort()`, `equal_ranges()`, `logic_error`, vector-to-`std::vector` candidate construction, final `results_.swap()`.
- 핵심 코드 발췌 위치: `af57a8f9c5fe:src/BatchEngine.cpp`는 각 accepted `JobResult`를 vector-backed와 deque-backed batch 양쪽에 추가하고 같은 `resultLess`로 독립 정렬합니다. range가 다르면 `batch container disagreement`를 throw하고, 같을 때만 vector range로 final candidate를 만들어 게시합니다.
- 변경 전/후 차이: 단일 representation 계산에서 두 container가 같은 semantic result를 만드는지 production path에서 대조하는 구조로 확장되었습니다. 추가 memory와 sort work는 commit 전 검증에 사용됩니다.
- 직접 확인한 ownership/lifetime/state 관계: vector/deque candidates와 final `std::vector` candidate는 모두 local owners입니다. `results_`는 두 sort와 equality, final vector construction이 끝날 때까지 prior state를 보유합니다.
- 직접 확인한 failure path: 어느 container의 insertion/sort/allocation이나 equality 전 단계가 실패하거나 두 range가 불일치하면 final swap이 실행되지 않습니다. local destructors가 양쪽 candidate를 정리합니다. 같은 comparator 구현을 공유하므로 독립 oracle 자체는 아니지만 representation disagreement는 탐지합니다.
- 실행한 테스트와 결과: 미실행. dual-container implementation과 publication 순서를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: vector와 deque 결과를 commit 전에 독립 정렬·대조해 representation disagreement를 거부했습니다.

### `9ba0e7c897ed` — test(batch): 입력 순열과 출력 결정성 검증

- Importance: **A**
- Tags: **TEST, DETERMINISM, EDGE**
- Source 역할: input permutation invariance와 byte-identical repeated output을 검증합니다.
- Source classification summary: Verifies input-permutation invariance, tie ordering, single-element behavior, and repeatable output.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `af57a8f9c5fe`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] 동일 job set의 여러 insertion permutation을 만드는 fixtures와 expected canonical order를 확인하세요.
- [x] equal-valued jobs에서 name tie-breaker가 빠졌을 때 잡힐 수 있는 test case를 찾으세요.
- [x] vector/deque batch ranges가 동일함을 확인하는 assertion을 기록하세요.
- [x] 동일 engine state를 여러 번 serialize해 byte-identical output을 비교하는 deterministic regression을 확인하세요.
- [x] 이 테스트가 단순 sortedness가 아니라 permutation invariance까지 증명하도록 입력 구성이 어떻게 설계되었는지 적으세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **canonical total order와 output determinism**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **input permutation, equal-value tie, repeated rendering**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **permutation regression + repeated-byte comparison**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **comparator, vector/deque sorting, serialization**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **입력 순서/representation에 독립적인 결과**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **stream failure rollback이나 allocation failure는 이 commit 단독 범위가 아님**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `ea23237ad506`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_batch_engine.cpp`의 permutation/tie/single-element/repeated-output cases; `tests/test_random_access_batch.cpp`의 cross-container equality; CLI fixtures.
- 핵심 코드 발췌 위치: `9ba0e7c897ed:tests/test_batch_engine.cpp`는 동일 job set의 서로 다른 입력 순서를 하나의 expected `(value, name)` row sequence와 비교하고, equal value에서 `Alpha`, `alpha`, `beta`, `zeta` name order를 확인합니다. 같은 engine을 반복 serialize한 bytes도 비교합니다.
- 변경 전/후 차이: sortedness 한 사례에서 입력 permutation 독립성, tie total order, vector/deque equality, repeated rendering의 byte determinism으로 검증 범위가 넓어졌습니다.
- 직접 확인한 ownership/lifetime/state 관계: 각 permutation은 별도 engine/candidate에서 계산되며 결과 bytes만 공통 oracle과 비교됩니다. repeated write는 persistent `results_`를 변경하지 않아야 합니다.
- 직접 확인한 failure path: name tie-breaker가 없거나 unstable/input-dependent order이면 equal-valued permutations이 다른 bytes를 만들어 실패합니다. 이 commit은 stream/allocation failure rollback을 직접 주입하지 않습니다.
- 실행한 테스트와 결과: 미실행. deterministic cases와 expected bytes를 검사했으며 test command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: canonical order가 입력 순서와 container 표현, 반복 출력에 독립적임을 고정했습니다.

### `ea23237ad506` — fix(batch): 입력 stream 종료 상태를 명확히 구분

- Importance: **A**
- Tags: **DEBUG, PARSING, EDGE**
- Source 역할: clean EOF, final unterminated record, input failure를 명확히 구분합니다.
- Source classification summary: Distinguishes clean EOF, an unterminated final record, and actual stream failure.

#### Failure → Fix → Test chain
- **기존 가정:** 일반적인 `getline` loop 종료를 clean input completion과 동일하게 취급할 수 있었다.
- **실제 failure / 위험:** valid final unterminated line을 버리거나 실제 I/O fault를 EOF처럼 받아들일 수 있었다.
- **root cause:** line extraction 종료 상태가 complete line / clean EOF / actual failure로 분류되지 않았다.
- **수정된 invariant / decision:** record reader가 세 outcome을 구분하고 clean completion일 때만 batch transaction을 commit한다.
- **실제 코드 확인:** 기존 reader/loop와 이 SHA의 reader helper를 비교해 stream flags와 return classification을 확인한다.
- **regression test:** `b4ddd78fb9aa`에서 stream failure 뒤 seeded state가 보존되는 경로를 확인한다.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `9ba0e7c897ed`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] 직전 batch reader의 일반 `getline` loop와 이 SHA의 record-reader helper를 관련 code로 비교하세요.
- [x] reader가 complete line, final unterminated line + clean EOF, actual failure의 세 outcome을 어떤 stream flags로 구분하는지 확인하세요.
- [x] trailing newline이 없는 마지막 record가 candidate에 포함되는 path를 추적하세요.
- [x] `badbit` 또는 non-EOF failure가 transaction rejection으로 이어지고 final swap을 막는 branch를 확인하세요.
- [x] fix가 syntax/arithmetic transaction에 transport-state success condition을 추가한 것임을 state publication 위치와 연결하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `b4ddd78fb9aa`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/BatchEngine.cpp`의 `readLine(std::istream&, std::string&)`, `BatchEngine::replace()` loop; 직전 `std::getline` 기반 loop.
- 핵심 코드 발췌 위치: `ea23237ad506:src/BatchEngine.cpp`의 `readLine()`은 `input.get(value)`로 newline까지 누적합니다. extraction 종료 후 `!input.eof()`면 input failure를 throw하고, clean EOF에서는 `!line.empty()`를 반환해 newline 없는 final record를 한 번 더 처리합니다.
- 변경 전/후 차이: 일반 `getline` loop 종료와 후속 flag 판정에 의존하던 reader를 complete line, clean EOF의 final unterminated line, non-EOF failure로 명시적으로 분리했습니다.
- 직접 확인한 ownership/lifetime/state 관계: line은 local candidate record이고 read helper가 true를 반환한 경우에만 parse/RPN/vector/deque candidates로 전달됩니다. stream은 caller-owned이며 position을 consume하지만 engine state는 final swap 전까지 유지됩니다.
- 직접 확인한 failure path: final line에 newline이 없어도 bytes가 있으면 정상 record로 처리합니다. initial/between-record read가 `badbit` 등 non-EOF failure로 끝나면 `invalid batch input`을 throw해 commit을 막습니다. input position이나 flags를 원상복구하지는 않습니다.
- 실행한 테스트와 결과: 미실행. fix 전후 reader와 후속 bad-stream tests를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: 성공한 input completion의 정의를 clean EOF, final unterminated record, 실제 read failure로 분리했습니다.

### `b4ddd78fb9aa` — test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증

- Importance: **A**
- Tags: **TEST, EXCEPTION, RISK**
- Source 역할: syntax/arithmetic/stream/allocation failure 전 범위에서 seeded state preservation을 검증합니다.
- Source classification summary: Sweeps malformed input, arithmetic, stream, and allocation failures after seeding engine state.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `ea23237ad506`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] known result로 engine을 seed하는 setup과 prior serialized bytes baseline을 확인하세요.
- [x] malformed input, RPN arithmetic failure, stream failure를 각각 주입하는 test cases와 production failure path를 매핑하세요.
- [x] observed allocation failure point sweep이 parsing, duplicate tracking, evaluator stack, two candidates, sort/compare 중 어디를 통과하는지 기록하세요.
- [x] 모든 rejection 후 result objects와 serialized bytes가 seed와 동일하고 live allocation baseline이 복구되는 assertions을 확인하세요.
- [x] CLI failure cases에서 stdout이 비어 있음을 검사해 object-state atomicity가 process-output atomicity로 연결되는 지점을 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **BatchEngine whole-input strong transaction**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **syntax, arithmetic, stream, allocation failures**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **seeded-state failure sweep + live-block accounting + CLI failure fixtures**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **parsing, duplicate set, RPN, two candidates, sort/compare, final publication**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **협력 layer failure 뒤 state/bytes/baseline rollback**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **외부 stream position 자체를 되돌리는 보장은 아님**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic failure-injection + integration**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_batch_engine.cpp`의 `checkInvalidPreserves()`, `checkOverflowPreserves()`, bad-stream case; `tests/failure/test_batch_failure.cpp`; `tests/support/FailingNew.cpp`; batch CLI failure fixtures.
- 핵심 코드 발췌 위치: `b4ddd78fb9aa:tests/test_batch_engine.cpp`는 failure 전 serialized bytes와 첫 `JobResult` 주소를 저장하고 syntax/RPN overflow/division/bad stream 뒤 동일한지 검사합니다. `tests/failure/test_batch_failure.cpp`는 observed allocation count를 얻어 1..N failure sweep을 수행합니다.
- 변경 전/후 차이: representative parsing tests에서 syntax, arithmetic, stream, allocation 협력 layer 전체를 대상으로 seeded prior state와 ownership baseline을 확인하는 회귀로 확장되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: engine을 `seed | 7`로 채운 뒤 replacement failure마다 size/value/output와 live-block baseline을 비교합니다. 주소 동일성 검사는 실패 중 `results_` vector 자체가 swap/reallocation되지 않았음을 관찰합니다.
- 직접 확인한 failure path: malformed records/duplicates, RPN invalid/overflow, pre-set bad stream, parsing·map·RPN stack·dual candidates·sort/final candidate에서 관찰된 allocation failures가 모두 final publication 전에 예외로 끝납니다. CLI invalid cases는 empty stdout를 요구합니다. consumed input stream position은 rollback 대상이 아닙니다.
- 실행한 테스트와 결과: 미실행. unit failure helpers, allocation sweep, CLI fixtures, Make targets를 검사했으나 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: batch transaction의 모든 관찰 협력 failure에서 prior objects, bytes, 주소, live allocations가 유지되는지 검증했습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `708c025ef2a0` | configurable random-access container abstraction과 cross-container equality 도입 | `Container` parameter, inherited iterator types, `std::sort`, underlying value ownership, copy-and-swap assignment, `equal_ranges`를 확인했습니다. | non-random-access container는 sort requirement 밖이며 실제 throwing-value evidence는 아직 없습니다. |
| `aaeff163baf8` | iterator/algorithm/substitution과 throwing-value failure 계약 검증 | vector/deque substitution, iterator algorithms, compile-fail list sort, throwing element copy/live counters가 generic/exception contract를 확인합니다. | whole-stream parsing과 persistent transaction은 아직 도입 전입니다. |
| `d0295f82614b` | whole-input candidate와 swap-on-success publication으로 batch transaction 확립 | line grammar, duplicate map, RPN call, local candidate, non-empty/stream completion 후 단일 `results_.swap(candidate)`를 확인했습니다. | canonical sort/serialization과 stream completion의 세부 구분은 후속 변경이 필요합니다. |
| `42d411e42268` | `(value, name)` total order와 classic-locale staged serialization 추가 | value-first/name-tie comparator와 candidate sort, classic temporary serialization, final single write를 확인했습니다. | destination write rollback과 cross-container agreement는 아직 보장하지 않습니다. |
| `af57a8f9c5fe` | vector/deque를 독립 sort하고 disagreement를 commit 전 거부 | 각 result를 vector/deque batches에 넣고 독립 sort 후 `equal_ranges`가 true일 때만 final vector를 게시합니다. | 두 경로가 같은 comparator를 공유하므로 완전히 독립된 semantic oracle는 아닙니다. |
| `9ba0e7c897ed` | input permutation independence와 repeated byte determinism 검증 | 서로 다른 permutations, equal-value name ties, one element, repeated bytes가 canonical result를 비교합니다. | stream/allocation failure rollback은 이 commit 단독 범위가 아닙니다. |
| `ea23237ad506` | record reader가 clean EOF/final unterminated line/input failure를 분리하도록 수정 | char-level `readLine()`이 newline, non-empty final EOF record, non-EOF failure를 분류하고 failure에서 swap을 막습니다. | caller stream position/flags 자체는 되돌리지 않습니다. |
| `b4ddd78fb9aa` | syntax/arithmetic/stream/allocation failure 전 범위에서 seeded state rollback 검증 | seeded bytes와 first-result address, syntax/RPN/bad-stream cases, observed allocation sweep와 live-block baseline을 확인합니다. | 관찰되지 않은 환경 failure와 external stream rollback은 증명하지 않습니다. |

## Failure → Fix → Test 연결

- `d0295f82614b`: whole-input candidate와 swap-on-success transaction을 도입합니다.
- `ea23237ad506` fix: successful input completion의 정의를 clean EOF/final unterminated line/actual failure로 정교화합니다.
- `b4ddd78fb9aa`: syntax/arithmetic/stream/allocation failure 뒤 seeded state rollback을 폭넓게 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: generic container의 value semantics가 안전해도 whole-stream operation이 persistent results를 line마다 변경하면 parse·RPN·sort·stream failure에서 partial batch가 노출됩니다. 일반 getline 종료를 성공으로 간주하는 것도 transport fault를 숨길 수 있습니다.
- 이를 드러낸 실제 failure 또는 test gap: duplicate/RPN failure는 여러 valid records 뒤 발생할 수 있고, vector/deque 또는 sort allocation도 late failure입니다. newline 없는 final record와 bad stream은 같은 loop 종료처럼 보일 수 있습니다.
- 수정/강화된 decision: 모든 records와 duplicate state, checked RPN results, vector/deque sort/compare, final vector construction을 local candidates에서 끝내고 clean input completion 뒤 swap합니다. reader는 EOF와 failure를 명시적으로 분류합니다.
- 해당 코드 위치: `d0295f82614b`부터 `ea23237ad506`까지의 `src/BatchEngine.cpp`, 특히 `replace()`, `resultLess()`, `readLine()`, `write()`.
- 이를 고정하는 regression/evidence: `9ba0e7c897ed`의 permutation/output tests와 `b4ddd78fb9aa`의 seeded syntax/arithmetic/stream/allocation failure tests.

## Ownership / state / responsibility 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: configurable random-access container abstraction과 cross-container equality 도입
- Thread 종료 상태: syntax/arithmetic/stream/allocation failure 전 범위에서 seeded state rollback 검증
- [x] 중간 commit마다 owner/state publisher/cleanup 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned state가 함께 등장하면 각각의 lifetime 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `708c025ef2a0`/`aaeff163baf8`: generic owner가 vector/deque substitution과 throwing element copy에서 value/assignment 보장을 제공합니다.
2. `d0295f82614b`: persistent `results_`의 publisher가 whole-input final swap 하나로 제한됩니다.
3. `42d411e42268`/`af57a8f9c5fe`: publication 전에 total-order sorting, dual representation comparison, final vector construction이 추가됩니다.
4. `9ba0e7c897ed`: canonical result가 input permutation과 repeated rendering에 독립적인지 bytes로 고정합니다.
5. `ea23237ad506`: transaction success 조건에 clean transport completion을 추가하고 final unterminated record를 보존합니다.
6. `b4ddd78fb9aa`: syntax, arithmetic, stream, allocation failure 뒤 prior vector object/address/bytes와 allocation baseline을 검사합니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `input stream → complete record read → grammar/uniqueness → checked RPN → vector/deque candidates → total-order sort/compare → final vector publication → staged serialization`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준으로 `BatchEngine::replace()`는 `readLine()`이 승인한 complete record만 parse합니다. name grammar와 duplicate를 검사하고 checked RPN 값을 vector/deque-backed local batches에 동시에 넣습니다. clean completion과 non-empty condition 뒤 두 batch를 같은 total comparator로 sort하고 range agreement를 검사한 후, vector range로 final candidate를 완성해 `results_.swap(candidate)`합니다. `write()`는 published results를 classic-locale temporary stream에서 직렬화합니다.

초기 generic container와 비교하면 generic value semantics, whole-input transaction, deterministic total order, dual-representation check, transport completion이 하나의 subsystem으로 결합되었습니다. 모든 관찰 preparation failure는 prior result object와 bytes를 보존하지만 input stream position과 destination final write는 rollback하지 않습니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: input stream → `readLine()` → `parseLine()`/name validation → duplicate map → `RpnEvaluator::evaluate()` → vector/deque `RandomAccessBatch::push_back()` → dual `sort(resultLess)` → `equal_ranges()` → final `std::vector` candidate → `results_.swap(candidate)` → staged `write()`.
- 핵심 상태 필드: persistent `std::vector<JobResult> results_`; local vector/deque batches, `seen` map, line/name/expression, final vector candidate.
- resource owner / borrowed view: caller owns input/output streams; engine owns published `results_`; all parsing, duplicate, RPN, dual-container and serialization candidates are call-local owners. `results()` returns a const borrowed view.
- commit point: clean non-empty input, both sorts, agreement, final vector construction을 모두 통과한 뒤의 `results_.swap(candidate)`입니다.
- cleanup path: syntax/duplicate/RPN/stream/allocation/sort/disagreement failure는 local map·batches·strings가 정리되고 prior `results_`는 그대로입니다. output staging failure는 destination write 전 정리됩니다.
- 최종 invariant 설명: published batch는 complete successful input 전체에서 나온 unique jobs이며 `(value, name)` total order로 vector/deque가 동의한 결과입니다. incomplete candidate는 publish되지 않고 repeated serialization은 input order와 caller formatting state에 독립적입니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 06-generic-containers-transactional-batch.md =====

===== BEGIN FILE: 07-contactbook-replacement-guarantee.md =====
# ContactBook Replacement Gains the Guarantee Used Elsewhere

## Thread 목표

고정 크기 ring buffer도 내부 `Contact` 대입이 allocation을 일으킬 수 있으므로 slot content와 ring metadata를 하나의 transaction으로 다뤄야 함을 확인하고, 뒤늦은 strong-guarantee 수정과 회귀 검증을 복원합니다.

**Source significance:** 작은 fixed array도 value assignment가 allocation할 수 있으면 strong guarantee가 필요함을 보여줍니다. slot content, `next_`, `size_`를 하나의 logical transaction으로 다루도록 초기 subsystem을 candidate-and-swap discipline에 맞춥니다.

## 이 Thread를 이해하기 위한 핵심 질문

- logical oldest-to-newest order와 physical slot index는 어떻게 분리되는가?
- full-capacity replacement에서 direct `Contact` assignment가 왜 부분 상태 변경 위험을 만드는가?
- detached replacement를 먼저 완성한 뒤 slot에 swap하는 순서가 어떤 invariant를 보존하는가?
- `next_`와 `size_`는 왜 slot commit 이후에만 변경되어야 하는가?
- failure sweep이 logical order와 allocation baseline까지 확인하는 이유는 무엇인가?

## 완료 기준

- [x] 초기 ring representation과 logical index 변환을 해당 SHA에서 설명할 수 있다.
- [x] 초기 direct assignment path와 fix의 detached-candidate path를 관련 SHA끼리 비교할 수 있다.
- [x] slot content, `next_`, `size_`의 coupled invariant를 실패 시나리오로 설명할 수 있다.
- [x] full-capacity failure regression이 실제 이전 취약 경로를 어떻게 재현하는지 확인할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- 완성되지 않은 contact replacement candidate는 stored slot에 publish되지 않는다.
- strong guarantee replacement는 allocation 실패 시 slot content와 ring metadata의 observable state를 보존한다.
- 직접 소유 값의 실패가 logical order/cursor publication과 분리되어야 한다.

### Major engineering difficulty

- allocation 가능한 value assignment가 있는 fixed array에서 partial mutation 방지.
- allocation failure sweep과 logical-order/live-block accounting으로 ring transaction 검증.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `2f9b934b0825` | feat(contact): 고정 크기 연락처 저장 순서 보존 | B | CORE | selected ring slot을 ordinary `Contact` assignment로 교체하는 초기 구현입니다. |
| 2 | `0ad14a57cab6` | fix(contact): 할당 실패에도 저장 상태 보존 | A | DEBUG, EXCEPTION, OWNERSHIP | detached replacement를 준비해 swap한 뒤 ring metadata를 advance합니다. |
| 3 | `8930c4d17bc1` | test(contact): 연락처 교체 실패 회귀 검증 | A | TEST, EXCEPTION, EDGE | full-capacity allocation failure를 sweep하며 order/value/leak baseline을 검증합니다. |

## Commit별 학습 기록

### `2f9b934b0825` — feat(contact): 고정 크기 연락처 저장 순서 보존

- Importance: **B**
- Tags: **CORE**
- Source 역할: selected ring slot을 ordinary `Contact` assignment로 교체하는 초기 구현입니다.
- Source classification summary: Adds an eight-slot circular `ContactBook` with logical oldest-to-newest indexing.

#### Thread 흐름에서 확인할 구현 역할
- [x] `ContactBook`의 8-slot backing array, logical size, next insertion cursor를 찾으세요.
- [x] valid/empty contact insertion policy와 capacity 도달 후 replacement slot 선택을 확인하세요.
- [x] `at()`가 logical oldest-to-newest index를 physical array index로 변환하는 식과 out-of-range branch를 기록하세요.
- [x] full-capacity replacement에서 selected slot에 ordinary `Contact` assignment를 수행하는 코드를 찾으세요.
- [x] 이 assignment가 allocation failure 시 slot/metadata transaction을 완전히 격리하지 못한다는 source 설명을 코드상 mutation 순서와 함께 기록하세요.
- [x] 이 commit이 다음 관련 commit의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `0ad14a57cab6`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/ContactBook.hpp`의 `capacity`, `contacts_`, `size_`, `next_`, `add()`, `at()`; `src/ContactBook.cpp`의 insertion/replacement와 logical-index 계산.
- 핵심 코드 발췌 위치: `2f9b934b0825:src/ContactBook.cpp`에서 non-empty contact는 `contacts_[next_] = contact`로 selected slot에 직접 대입되고, 이후 `next_ = (next_ + 1) % capacity`, `size_` 증가가 실행됩니다. `at()`는 full이면 `first = next_`, 아니면 0을 사용합니다.
- 변경 전/후 차이: 최대 8개 contact를 저장하고 capacity 이후 oldest physical slot을 덮는 circular representation이 추가되었습니다. logical observation은 physical 배열 순서와 분리되어 oldest-to-newest로 제공됩니다.
- 직접 확인한 ownership/lifetime/state 관계: `ContactBook`이 fixed array의 각 `Contact` 값을 소유합니다. `next_`는 다음 insertion physical slot이고 `size_`는 유효 logical count입니다. `at()`가 반환하는 `const Contact&`는 book lifetime과 후속 replacement에 종속된 borrowed reference입니다.
- 직접 확인한 failure path: empty contact는 아무 변경 없이 반환하고 out-of-range logical index는 예외입니다. full book에서 ordinary `Contact::operator=`가 여러 owned string을 복사하다 실패하면 committed slot 내부가 부분 변경될 수 있으므로, 뒤의 metadata가 아직 안 바뀌어도 slot/content/order transaction은 보장되지 않습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 ring representation과 mutation 순서를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: logical order를 보존하는 8-slot ring을 만들었지만 replacement는 throwing direct assignment에 의존했습니다.

### `0ad14a57cab6` — fix(contact): 할당 실패에도 저장 상태 보존

- Importance: **A**
- Tags: **DEBUG, EXCEPTION, OWNERSHIP**
- Source 역할: detached replacement를 준비해 swap한 뒤 ring metadata를 advance합니다.
- Source classification summary: Copies a contact into a detached candidate before swapping and advancing the ring.

#### Failure → Fix → Test chain
- **기존 가정:** fixed array slot에 `Contact`를 대입하고 ring metadata를 갱신하는 정상 경로면 충분했다.
- **실제 failure / 위험:** `Contact` string copy allocation이 throw하면 stored slot과 logical cursor/size의 conceptual transaction이 깨질 수 있었다.
- **root cause:** throwing value assignment를 committed slot에 직접 적용했다.
- **수정된 invariant / decision:** detached replacement를 완성한 뒤 slot에 non-throwing swap하고, metadata는 그 뒤에 advance한다.
- **실제 코드 확인:** `2f9b934b0825`의 direct assignment와 이 SHA의 detached-copy/swap/update 순서를 비교한다.
- **regression test:** `8930c4d17bc1`의 full-capacity allocation failure sweep을 확인한다.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `2f9b934b0825`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] 초기 관련 SHA `2f9b934b0825`와 `ContactBook::add()`를 직접 비교해 direct assignment 제거 지점을 찾으세요.
- [x] incoming contact를 detached local replacement로 복사하는 시점과 그 복사에서 allocation exception이 날 수 있는 경로를 확인하세요.
- [x] replacement 완성 후 selected slot과 non-throwing `swap()`하는 commit point를 표시하세요.
- [x] `next_`와 `size_` mutation이 slot swap 성공 뒤에만 실행되는 정확한 순서를 기록하세요.
- [x] copy failure 시 slot content, logical oldest position, size가 모두 untouched인 이유를 before/after state로 설명하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `8930c4d17bc1`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ContactBook.cpp`의 `ContactBook::add()`; `Contact` default construction, assignment, `swap()`; `size_`, `next_` update order.
- 핵심 코드 발췌 위치: `0ad14a57cab6:src/ContactBook.cpp`는 `Contact replacement; replacement = contact; contacts_[next_].swap(replacement);`를 실행한 뒤에만 cursor와 size를 갱신합니다.
- 변경 전/후 차이: stored slot에 incoming value를 직접 대입하던 코드를 detached local replacement 완성 후 non-throwing slot swap으로 바꿨습니다. metadata update는 slot commit 뒤로 유지됩니다.
- 직접 확인한 ownership/lifetime/state 관계: copy 준비 중에는 existing slot이 old contact resources를 계속 소유하고 local `replacement`가 새 copies를 소유합니다. swap 후 slot이 새 contact를, replacement가 old slot value를 소유하며 scope 종료 시 old resources를 파괴합니다.
- 직접 확인한 failure path: `replacement = contact` 중 allocation이 실패하면 slot swap과 `next_`/`size_` update에 도달하지 않습니다. 따라서 physical slot bytes, logical oldest mapping, size/cursor가 모두 prior state로 남습니다. 성공 후 swap은 publication point이며 old value cleanup은 local replacement가 담당합니다.
- 실행한 테스트와 결과: 미실행. fix 전후 `ContactBook::add()`를 비교하고 후속 failure test를 검사했으며 command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: detached Contact를 완성한 뒤 slot swap과 metadata advance를 수행해 ring replacement를 transaction으로 만들었습니다.

### `8930c4d17bc1` — test(contact): 연락처 교체 실패 회귀 검증

- Importance: **A**
- Tags: **TEST, EXCEPTION, EDGE**
- Source 역할: full-capacity allocation failure를 sweep하며 order/value/leak baseline을 검증합니다.
- Source classification summary: Sweeps allocation failures during full-book replacement and verifies logical order and leak baselines.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `0ad14a57cab6`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] book을 capacity까지 채워 oldest replacement path를 강제로 만드는 test setup을 확인하세요.
- [x] replacement copy의 모든 관찰 allocation point에 failure를 주입하는 sweep loop를 찾으세요.
- [x] 각 실패 후 size, logical order, field values, live-allocation count를 비교하는 assertions을 기록하세요.
- [x] failure sweep 뒤 한 번의 successful insertion이 정상 ring advance를 확인하는 이유를 actual expected order와 연결하세요.
- [x] 이 deterministic regression이 직접 throwing assignment를 stored slot에 다시 도입하는 변경을 어떻게 잡는지 production path를 매핑하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **ContactBook full-capacity replacement strong guarantee**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **allocation failure while replacing oldest slot**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **full-capacity deterministic allocation-failure sweep + live-block accounting**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **Contact copy → detached replacement → slot swap → ring metadata**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **size/order/fields/live blocks가 실패 뒤 불변**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **다른 contact operations의 모든 실패 형태를 포괄하지는 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/failure/test_contact_failure.cpp`; `tests/support/FailingNew.hpp/.cpp`; contact/book value comparison helpers; `Makefile`의 contact failure target.
- 핵심 코드 발췌 위치: `8930c4d17bc1:tests/failure/test_contact_failure.cpp`는 book을 capacity 8까지 채워 oldest-slot replacement 경로를 만들고, 성공 run에서 관찰한 allocation attempts를 1..N으로 순회해 각 지점에서 `std::bad_alloc`을 주입합니다.
- 변경 전/후 차이: production fix 위에 full-capacity 취약 경로를 직접 반복하는 deterministic failure regression이 추가되었습니다. failure 뒤 상태 확인과 이후 정상 insertion 확인을 함께 수행합니다.
- 직접 확인한 ownership/lifetime/state 관계: 각 failure 전 live-block baseline과 logical size/order/각 contact field를 저장합니다. exception 뒤 모두 동일해야 하며 outer scope 종료 후 전체 live count도 원래 baseline으로 돌아와야 합니다.
- 직접 확인한 failure path: detached replacement copy의 각 관찰 allocation이 실패해도 slot/content/cursor가 바뀌지 않는지 검사합니다. sweep 뒤 한 번 성공 insertion을 수행해 oldest 하나만 제거되고 new contact가 newest로 추가되는 정상 advance를 확인합니다. 다른 Contact operation의 모든 failure는 범위 밖입니다.
- 실행한 테스트와 결과: 미실행. failure controller, sweep loop, expected order와 Make target을 검사했으나 binary는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: full ring replacement의 모든 관찰 allocation 실패에서 값·순서·크기·live blocks를 고정했습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `2f9b934b0825` | 8-slot circular buffer와 logical order 도입, replacement는 direct assignment 상태 | 8-element `contacts_`, `size_`, `next_`, full일 때 `first = next_`인 logical indexing과 direct slot assignment를 확인했습니다. | throwing `Contact` assignment가 committed slot을 부분 변경할 수 있어 content와 metadata의 강한 transaction이 없습니다. |
| `0ad14a57cab6` | detached `Contact` 완성 → non-throwing swap → metadata advance 순서로 수정 | default local `replacement`를 완성한 뒤 slot `swap()`을 하고 그 후 cursor/size를 갱신합니다. | deterministic full-capacity allocation failure evidence는 후속 test가 필요합니다. |
| `8930c4d17bc1` | full-capacity allocation failure sweep으로 size/order/values/live blocks 보존 검증 | capacity-seeded book에서 allocation positions를 sweep하고 실패 뒤 size/order/fields/live-block baseline과 성공 retry order를 비교합니다. | 다른 Contact APIs나 관찰되지 않은 allocator path 전체를 포괄하지는 않습니다. |

## Failure → Fix → Test 연결

- `2f9b934b0825` 초기 상태: full-capacity slot replacement가 ordinary `Contact` assignment를 사용합니다.
- `0ad14a57cab6` fix: detached replacement → slot swap → metadata advance 순서로 변경합니다.
- `8930c4d17bc1` regression: allocation failure 전 지점에서 size/order/values/live blocks를 고정합니다.

### 학습자 연결 기록
- 최초 위험/맹점: fixed array 자체는 allocation하지 않아도 slot의 `Contact` value assignment가 owned strings를 복사하므로 committed slot에 직접 대입하면 중간 throw가 partial value를 남길 수 있습니다.
- 이를 드러낸 실제 failure 또는 test gap: 초기 `add()`는 slot assignment 뒤 metadata를 바꿨지만, metadata가 unchanged여도 slot content가 이미 변할 수 있어 logical order의 old value 보존이 성립하지 않았습니다.
- 수정/강화된 decision: incoming contact를 detached local value에 완전히 복사하고, 성공 후 slot과 non-throwing swap한 뒤에만 `next_`와 `size_`를 갱신합니다.
- 해당 코드 위치: `2f9b934b0825:src/ContactBook.cpp`의 direct assignment와 `0ad14a57cab6:src/ContactBook.cpp`의 replacement/swap/update sequence.
- 이를 고정하는 regression/evidence: `8930c4d17bc1:tests/failure/test_contact_failure.cpp`의 full-capacity allocation-failure sweep과 successful retry.

## Ownership / state / responsibility 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: 8-slot circular buffer와 logical order 도입, replacement는 direct assignment 상태
- Thread 종료 상태: full-capacity allocation failure sweep으로 size/order/values/live blocks 보존 검증
- [x] 중간 commit마다 owner/state publisher/cleanup 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned state가 함께 등장하면 각각의 lifetime 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `2f9b934b0825`: book이 physical slots와 logical `size_`/`next_`를 소유하지만 replacement publisher는 throwing slot assignment입니다.
2. `0ad14a57cab6`: throw 가능한 copy 책임이 detached `replacement`로 이동하고 slot publication은 `swap()`으로, logical metadata publication은 그 뒤로 분리됩니다.
3. `8930c4d17bc1`: full-capacity failure마다 old logical sequence와 allocation ownership이 유지되고 이후 success가 정확히 한 번 advance하는지 검사합니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `logical insertion request → choose physical slot → detached Contact copy → slot swap commit → cursor/size advance → logical oldest-to-newest observation`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준으로 `ContactBook::add()`는 empty input을 무시하고, incoming contact를 local `replacement`에 먼저 복사합니다. 복사가 끝난 뒤 `contacts_[next_].swap(replacement)`로 selected physical slot을 교체하고, 그 다음 `next_`를 회전시키며 capacity 미만일 때만 `size_`를 증가시킵니다. `at()`는 full 여부에 따라 logical oldest physical index를 계산합니다.

초기 ring과 비교하면 slot value와 cursor/size가 하나의 성공 transaction으로 취급됩니다. allocation 실패는 stored values와 logical order를 보존하고 old slot resources는 성공 뒤 local replacement가 정리합니다. 보장 범위는 contact replacement 경로이며 external borrowed reference의 후속 successful replacement invalidation까지 없애지는 않습니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: incoming `Contact&` → `ContactBook::add()` empty check → local `Contact replacement` copy → `contacts_[next_].swap(replacement)` → `next_`/`size_` advance → `at()`의 logical-to-physical mapping.
- 핵심 상태 필드: `Contact contacts_[capacity]`, `std::size_t size_`, `std::size_t next_`; 각 Contact 내부 owned string values.
- resource owner / borrowed view: book slots와 local replacement가 Contact resources를 값으로 소유하고, `add()` input과 `at()` 반환 reference는 borrowed입니다.
- commit point: detached copy 성공 뒤 selected slot과 실행하는 non-throwing `swap()`이며, metadata는 그 commit 뒤에만 바뀝니다.
- cleanup path: copy allocation failure는 local replacement가 자신의 partial state를 정리하고 slot/metadata를 건드리지 않습니다. 성공 후 local replacement destructor가 old slot resources를 정리합니다.
- 최종 invariant 설명: 실패 시 slot content·logical order·cursor·size가 함께 보존되고, 성공 시 정확히 한 physical slot 교체와 한 번의 ring advance만 관찰됩니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 07-contactbook-replacement-guarantee.md =====

===== BEGIN FILE: 08-verification-supported-release-claims.md =====
# Verification Expands from In-tree Tests to Supported Release Claims

## Thread 목표

in-tree 기능 테스트만으로는 확인할 수 없는 public API shape, 외부 consumer packaging, deterministic breadth, sanitizer/host prerequisite, ABI, compiler/platform 범위를 단계적으로 실행 가능한 release claim으로 확장하는 과정을 복원합니다.

**Source significance:** source-level intent와 실제 supportable release claim의 간극을 단계적으로 줄입니다. interface shape, external packaging, state-space breadth, UB, host prerequisites, ABI assumptions, compiler/platform variation을 서로 다른 verification layer로 다룹니다.

## 이 Thread를 이해하기 위한 핵심 질문

- unit test, compile-contract, CLI fixture가 각각 어떤 blind spot을 담당하는가?
- positive/negative translation unit이 runtime test와 다른 종류의 API 회귀를 어떻게 막는가?
- repository 밖 consumer가 in-tree build에서 가려질 수 있는 어떤 dependency를 노출하는가?
- fixed-seed property와 large batch가 hand-written fixture를 대체하지 않고 보완하는 이유는 무엇인가?
- portable baseline과 host-specific checks를 분리하지 않으면 어떤 지원 주장 왜곡이 생기는가?
- LP64 check와 compiler/platform matrix가 각각 어떤 portability claim을 executable하게 만드는가?

## 완료 기준

- [x] verification layer별 입력, 실패 조건, 증명하는 계약, 증명하지 않는 범위를 구분할 수 있다.
- [x] public header isolation과 external archive consumption을 실제 build commands/tests에서 확인할 수 있다.
- [x] fixed seed, sanitizer, ABI assertion, CI matrix가 서로 중복되지 않는 증거를 제공하는 이유를 설명할 수 있다.
- [x] portable target과 platform target의 포함 관계 및 host prerequisite를 실제 Makefile/CI에서 복원할 수 있다.

## Source에 연결된 invariant / engineering difficulty

### Critical invariant

- public headers는 private include path 없이 compile되고 private representation은 inaccessible해야 한다.
- 지원하는 C++98 LP64 platform assumptions는 명시적으로 executable하게 검증된다.
- deterministic output과 owned-resource release는 process/release scope에서도 증거가 필요하다.

### Major engineering difficulty

- portable verification과 host-specific archive/dependency/leak/sanitizer capability 분리.
- compile-time API enforcement, external-consumer verification, reproducible property testing, compiler/platform matrix 구성.

위 항목은 source가 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `6e78ced59357` | test(contact): 공개 계약과 명령행 세션 검증 | A | API, TEST, INTEGRATION | unit, compile-contract, CLI integration layer를 도입합니다. |
| 2 | `4bbbfd191669` | test(contracts): 공개 include와 소유권 규칙 검증 | A | API, TEST, OWNERSHIP | library 전체 positive/negative public contract를 확장합니다. |
| 3 | `01271d795d58` | test(consumer): 저장소 밖 공개 library 연결 검증 | A | API, INTEGRATION, PORTABILITY | repository 밖에서 public library consumer를 compile/run합니다. |
| 4 | `9e07d3bc86d3` | test(boundary): 변환·배치 속성과 대용량 경계 검증 | A | TEST, DETERMINISM, EDGE | fixed-seed properties와 large-batch stress를 추가합니다. |
| 5 | `45e9bbfd6b75` | build(check): sanitizer와 portable 검사 계층 구성 | A | ARCH, PORTABILITY, TEST | build/portable/platform checks를 분리하고 ASan/UBSan 역할을 구분합니다. |
| 6 | `ab441fa8737c` | test(portability): 지원 LP64 데이터 모델 검증 | B | PORTABILITY, API | 지원 LP64 ABI assumptions를 executable check로 만듭니다. |
| 7 | `50565bd67e03` | ci: 지원 compiler와 platform matrix 검증 | B | PORTABILITY, TEST | established claims를 GCC/Clang, Linux/macOS matrix에서 실행합니다. |

## Commit별 학습 기록

### `6e78ced59357` — test(contact): 공개 계약과 명령행 세션 검증

- Importance: **A**
- Tags: **API, TEST, INTEGRATION**
- Source 역할: unit, compile-contract, CLI integration layer를 도입합니다.
- Source classification summary: Introduces unit, compile-contract, and CLI fixture layers for the contact subsystem.

#### 핵심 설계 / failure boundary 확인
- [x] contact subsystem tests가 unit, compile-contract, command-line fixture로 분리되는 build/test targets를 찾으세요.
- [x] positive compile test가 public header를 두 번 include하고 exported names만 사용하는 translation unit을 확인하세요.
- [x] private representation 접근을 의도적으로 시도해 compile-fail을 요구하는 negative contract를 확인하세요.
- [x] real binary session fixture가 ADD/LIST/invalid/QUIT transcript 전체 bytes를 비교하는 방식을 기록하세요.
- [x] 세 layer가 각각 domain behavior, API shape, process protocol 중 무엇을 검증하는지 구분하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **Contact public API, domain order, process protocol**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **private representation exposure, header isolation, session drift**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **unit + positive/negative compile + CLI transcript**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **Contact/ContactBook public headers and real app**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **multi-layer verification pattern이 작동함**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **repository 전체 API/release packaging까지는 아직 아님**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **broad integration/contract**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `4bbbfd191669`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `Makefile`의 contact unit/contract/integration targets; `tests/compile/contact_headers.cpp`, contact private-access negative translation unit; `tests/check_cli.sh`; contact unit tests와 실제 app binary.
- 핵심 코드 발췌 위치: `6e78ced59357`의 positive compile unit은 public contact header를 반복 include하고 exported API만 사용합니다. negative unit은 private representation 접근을 시도하며 harness는 compiler rejection을 success로 취급합니다. CLI script는 scripted ADD/LIST/invalid/QUIT session의 status와 exact bytes를 비교합니다.
- 변경 전/후 차이: in-process domain unit test 하나에서 public header shape와 실제 process protocol까지 서로 다른 failure 조건으로 검사하는 세 층 구조가 생겼습니다.
- 직접 확인한 ownership/lifetime/state 관계: unit layer는 `Contact`/`ContactBook` 값과 logical order를, compile layer는 const/private ownership boundary를, CLI layer는 app이 입력을 읽고 state를 갱신해 stdout/stderr로 내보내는 lifetime 전체를 관찰합니다.
- 직접 확인한 failure path: private member가 public이 되거나 header가 self-contained하지 않으면 compile contract가, command parsing/output가 바뀌면 transcript가 실패하도록 작성되어 있습니다. 이 commit의 범위는 contact subsystem이며 repository 전체 archive packaging이나 sanitizer evidence는 아직 아닙니다.
- 실행한 테스트와 결과: 미실행. target dependency, translation units, CLI fixture를 검사했으나 compiler/app command는 수행하지 않았습니다.
- 이 commit을 한 문장으로 설명: contact behavior, public API shape, real CLI session을 분리해 검증하는 기본 패턴을 만들었습니다.

### `4bbbfd191669` — test(contracts): 공개 include와 소유권 규칙 검증

- Importance: **A**
- Tags: **API, TEST, OWNERSHIP**
- Source 역할: library 전체 positive/negative public contract를 확장합니다.
- Source classification summary: Expands positive and negative compile contracts across every public header and ownership boundary.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `6e78ced59357`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] library 전체 public headers를 대상으로 하는 positive consumer translation units와 link step을 확인하세요.
- [x] abstract interface, protected/private construction, explicit conversion, const access, utility non-construction, list-backed sorting, pointer signature를 각각 어떤 negative file이 거부하는지 분류하세요.
- [x] negative compile test가 "실행 결과"가 아니라 compiler rejection 자체를 expected success로 다루는 harness를 확인하세요.
- [x] private include path 없이 archive와 public include만으로 contract가 성립하는지 build command를 기록하세요.
- [x] runtime tests가 통과해도 API widening/encapsulation regression을 이 layer가 어떻게 잡는지 예시 하나를 실제 test와 연결하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **repository-wide public API shape/ownership boundary**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **implicit conversion, accidental mutability/construction, private include leakage**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **positive/negative compile-contract suite**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **all installed public headers + archive link**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **external consumer가 허용/금지된 API shape를 compiler 수준에서 고정**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **runtime behavioral correctness나 leak freedom 자체는 별도 evidence가 필요**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **broad compile contract**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `01271d795d58`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `Makefile`의 `PUBLIC_CPPFLAGS := -Iinclude`, `test-contract`, public-contract link rule; `tests/compile/public_headers.cpp`와 subsystem positive units; abstract/private/const/explicit/list-sort 등 negative units.
- 핵심 코드 발췌 위치: `4bbbfd191669:Makefile`의 positive commands는 `-Iinclude ... -fsyntax-only`만 사용하고, negative commands는 `@! $(CXX) ... -fsyntax-only <fail.cpp>`로 rejection을 기대합니다. public integration binary도 public include path와 archive만 링크합니다.
- 변경 전/후 차이: contact 한 영역의 compile contract가 모든 installed public headers와 주요 ownership/API restrictions로 확대되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: compile suite는 `TextBuffer` 내부 storage/implicit conversion, formatter abstractness, creator/builder construction, scalar/RPN utility construction, runtime type access, serializer pointer/const shape, batch result mutability, template const iterator와 container requirement를 compiler-visible boundary로 고정합니다.
- 직접 확인한 failure path: 허용 API가 빠지거나 private header가 필요하면 positive unit이 실패하고, 금지 API가 우연히 열리면 expected-fail unit이 성공해 Make target이 실패합니다. 이 layer는 compiler rejection을 증명할 뿐 runtime output, exception cleanup, leak freedom은 별도 evidence가 필요합니다.
- 실행한 테스트와 결과: 미실행. compile command와 positive/negative file set을 검사했으나 compiler는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: public include만으로 허용·금지 API shape를 repository 전체에서 compiler 계약으로 만들었습니다.

### `01271d795d58` — test(consumer): 저장소 밖 공개 library 연결 검증

- Importance: **A**
- Tags: **API, INTEGRATION, PORTABILITY**
- Source 역할: repository 밖에서 public library consumer를 compile/run합니다.
- Source classification summary: Builds and runs a consumer outside the repository using only public headers and the archive.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `4bbbfd191669`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] temporary consumer directory가 repository tree 밖에 생성되는 setup과 cleanup trap을 확인하세요.
- [x] compiler include path가 exported include directory만, linker input이 `libcpp_foundation.a`만 사용하도록 제한되는 command를 기록하세요.
- [x] consumer가 test-only helper 없이 실제 public objects를 사용하는 source를 확인하세요.
- [x] working-directory assumption, private include, unarchived object가 있으면 어느 compile/link/run 단계에서 실패하는지 추적하세요.
- [x] external location에서 executable을 실제 run하는 단계까지 포함되는지 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **public archive/header external consumability**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **in-tree-only include/path/object assumptions**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **out-of-tree compile/link/run consumer**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **exported include + static archive + public objects**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **packaging boundary가 실제 external location에서 성립함**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **모든 downstream build system/platform을 증명하지는 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **broad integration/release regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `9e07d3bc86d3`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/check_external_consumer.sh`, `tests/consumer/external_main.cpp`, `Makefile`의 `test-consumer`와 `test-integration`, `libcpp_foundation.a`.
- 핵심 코드 발췌 위치: `01271d795d58:tests/check_external_consumer.sh`는 `${TMPDIR:-/tmp}` 아래 `mktemp -d`로 외부 디렉터리를 만들고 consumer source를 복사합니다. compiler에는 `-I"$project_root/include"`, copied `main.cpp`, 전달받은 absolute archive만 주고 그 디렉터리에서 executable을 실행합니다.
- 변경 전/후 차이: repository 내부 translation unit/link에서 실제 out-of-tree consumer compile/link/run으로 packaging 검증이 확장되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: cleanup trap이 copied source와 executable을 삭제하고 temporary directory를 제거합니다. consumer는 test support나 source object를 직접 소유·링크하지 않고 exported headers와 static archive만 사용합니다.
- 직접 확인한 failure path: private include, working-directory-relative asset, archive에 빠진 symbol/object가 있으면 compile/link/run 중 실패합니다. compiler 존재와 argument/archive file은 script가 먼저 검사합니다. 한 compiler command/host의 consumer일 뿐 모든 downstream build system을 증명하지는 않습니다.
- 실행한 테스트와 결과: 미실행. script의 exact command와 cleanup/run scope를 검사했으나 외부 consumer를 실제 컴파일하지 않았습니다.
- 이 commit을 한 문장으로 설명: public headers와 정적 archive만으로 repository 밖 consumer가 실제 실행되는 packaging boundary를 검사합니다.

### `9e07d3bc86d3` — test(boundary): 변환·배치 속성과 대용량 경계 검증

- Importance: **A**
- Tags: **TEST, DETERMINISM, EDGE**
- Source 역할: fixed-seed properties와 large-batch stress를 추가합니다.
- Source classification summary: Adds fixed-seed scalar/RPN properties and a 4,096-job batch stress check.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `01271d795d58`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] fixed linear-congruential seed와 first-counterexample reporting을 구현한 property test driver를 찾으세요.
- [x] generated integer literals가 4-line scalar output과 trailing invalid byte rejection을 어떻게 반복 검증하는지 확인하세요.
- [x] bounded binary RPN expressions를 direct computation과 비교하는 oracle 구성과 overflow 회피 범위를 확인하세요.
- [x] 4,096-job / 200KB 이상 batch의 생성, 독립 sort oracle, record-by-record comparison, repeated serialization 검증을 추적하세요.
- [x] deterministic seed가 breadth를 늘리면서 flaky randomness를 피하는 구조를 실제 test inputs와 counterexample output에서 확인하세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **scalar/RPN/batch boundary breadth와 determinism**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **generated edge cases and large allocation/sort/output growth**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **fixed-seed property-style generation + large stress case**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **scalar parser/render, RPN checked ops, batch parse/sort/serialize**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **넓은 deterministic state-space에서 established invariants 유지**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **formal exhaustive proof나 unfixed randomness를 제공하지는 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **deterministic property/stress**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `45e9bbfd6b75`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/property/test_boundary_properties.cpp`의 fixed LCG, scalar/RPN properties, `ExpectedJob` oracle, large batch; `tests/run_with_timeout.sh`; `Makefile`의 `test-property`.
- 핵심 코드 발췌 위치: `9e07d3bc86d3:tests/property/test_boundary_properties.cpp`는 seed `0x13579BDF`, LCG `state * 1103515245 + 12345`, first-counterexample text를 사용합니다. scalar 2,048개, bounded binary RPN 4,096개, 4,096-job·200KB 초과 batch를 생성합니다.
- 변경 전/후 차이: hand-written boundaries를 대체하지 않고 reproducible generated breadth와 large allocation/sort/output growth를 추가했습니다.
- 직접 확인한 ownership/lifetime/state 관계: scalar는 exact int line·4 newlines·repeatability·trailing `x` rejection/empty output을 확인합니다. RPN은 overflow를 피한 bounded direct arithmetic oracle와 비교하고, batch는 독립 `ExpectedJob` vector를 `std::sort`한 뒤 record-by-record 및 repeated bytes를 비교합니다.
- 직접 확인한 failure path: 실패 시 fixed seed와 첫 counterexample을 출력해 재현 가능하게 합니다. timeout wrapper는 hang/과도한 runtime을 별도 failure로 만듭니다. fixed finite sample이므로 formal exhaustive proof나 임의 seed 다양성은 제공하지 않습니다.
- 실행한 테스트와 결과: 미실행. generator, oracle, counts, timeout target을 검사했으나 property binary는 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: 고정 seed 속성 검사와 4,096-job stress로 deterministic 검증 폭을 넓혔습니다.

### `45e9bbfd6b75` — build(check): sanitizer와 portable 검사 계층 구성

- Importance: **A**
- Tags: **ARCH, PORTABILITY, TEST**
- Source 역할: build/portable/platform checks를 분리하고 ASan/UBSan 역할을 구분합니다.
- Source classification summary: Separates build, portable, and platform verification while adding ASan and UBSan layers.

#### 핵심 설계 / failure boundary 확인
- [x] 필요하면 직전 관련 SHA `9e07d3bc86d3`와 비교하여 책임, state mutation 순서, test boundary가 어떻게 달라졌는지 확인하세요.
- [x] Makefile/check targets에서 build, portable, platform layer가 어떤 dependency graph로 재구성되는지 그리세요.
- [x] clean rebuild, tests, compiler contracts, archive, external consumer, properties, UBSan 중 portable baseline에 포함되는 항목을 실제 target prerequisite로 확인하세요.
- [x] ASan과 host-specific release inspection이 별도 platform checks로 분리되는 이유를 prerequisites/commands에서 확인하세요.
- [x] aggregate target이 unavailable host tool 때문에 portable checks까지 생략하지 않도록 구성된 branch/gating을 기록하세요.
- [x] reconstruction, incremental no-op, deterministic artifact 검증이 어느 layer에서 실행되는지 찾으세요.
- [x] 이 commit의 변경이 어떤 invariant/failure path/API boundary를 강화하는지 실제 코드와 test를 연결해 적으세요.
- [x] 이 commit의 보장 범위를 넘는 항목은 무엇인지 source에 근거해 별도로 적으세요.



#### 다음 관련 commit과 연결
- 다음 Thread SHA `ab441fa8737c`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `Makefile`의 `test-asan`, `test-ubsan`, `test-sanitize`, `check-build`, `check-portable`, `check-platform`, `check`; archive/dependency/leak/determinism scripts.
- 핵심 코드 발췌 위치: `45e9bbfd6b75:Makefile`에서 `check-build`는 diff check, clean rebuild, `test`, deterministic CLI 두 번, incremental `make -q all`을 수행합니다. `check-portable`은 여기에 UBSan을 추가하고 `check-platform`은 archive/dependency/leak scripts만 실행합니다.
- 변경 전/후 차이: 하나의 검사 묶음을 portable baseline과 host/tool-dependent release inspection으로 분리하고 ASan/UBSan binary/targets를 별도로 만들었습니다.
- 직접 확인한 ownership/lifetime/state 관계: `test` 아래 unit, deterministic failure injection, no-elide, compile contract, CLI/public integration, external consumer, property test가 모입니다. UBSan은 `check-portable` dependency이고, ASan은 standalone `test-asan`/`test-sanitize`로 남아 host capability에 따라 선택됩니다.
- 직접 확인한 failure path: portable checks는 host-specific `ar`/dependency/leak inspection 실패 때문에 생략되지 않고, timeout wrapper와 sanitizer halt options가 UB/memory error를 target failure로 만듭니다. **관찰된 차이:** scaffold 문구와 달리 이 SHA의 `check-platform`에는 ASan이 포함되지 않으며 aggregate `check`도 ASan을 호출하지 않습니다. ASan matrix 실행은 후속 CI commit에서 추가됩니다.
- 실행한 테스트와 결과: 미실행. Make dependency graph와 commands를 검사했으나 sanitizer/host utility를 실행하지 않았습니다.
- 이 commit을 한 문장으로 설명: portable regression·UBSan과 host-specific release inspection을 분리하고 ASan을 별도 capability target으로 제공했습니다.

### `ab441fa8737c` — test(portability): 지원 LP64 데이터 모델 검증

- Importance: **B**
- Tags: **PORTABILITY, API**
- Source 역할: 지원 LP64 ABI assumptions를 executable check로 만듭니다.
- Source classification summary: Adds compile-time LP64 data-model assertions.

#### Thread 흐름에서 확인할 구현 역할
- [x] 직전 관련 SHA `45e9bbfd6b75`와의 차이 중 이 Thread의 흐름에 필요한 부분만 확인하세요.
- [x] CHAR_BIT/short/int/long/pointer/size_t 크기를 compile-time에 assert하는 translation unit을 확인하세요.
- [x] expected data model이 8-bit byte, 2-byte short, 4-byte int, 8-byte long/pointer/size_t임을 test expression에서 확인하세요.
- [x] 이 check가 scalar limits, RPN `long`, allocation size, pointer token 중 어떤 subsystem assumptions와 연결되는지 source references를 찾아 적으세요.
- [x] 지원하지 않는 ABI에서 runtime 오동작 대신 compile-time failure로 끝나는 harness 동작을 확인하세요.
- [x] 이 commit이 다음 관련 commit의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **supported LP64 data model**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **unsupported ABI silently compiling**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **compile-time portability assertions**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **public/platform build boundary before runtime**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **declared ABI assumptions을 만족하지 않으면 조기 실패**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **LP64 내부의 모든 platform 차이를 증명하지는 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **compile-time portability contract**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 다음 관련 commit과 연결
- 다음 Thread SHA `50565bd67e03`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 failure boundary를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/portability/test_data_model.cpp`; `Makefile`의 `DATA_MODEL_BIN`, `check-data-model`, `check-build` dependency.
- 핵심 코드 발췌 위치: `ab441fa8737c:tests/portability/test_data_model.cpp`는 `CHAR_BIT == 8`, `sizeof(short)==2`, `sizeof(int)==4`, `sizeof(long)==8`, pointer/`size_t` 8을 `const bool lp64`로 계산하고 false면 diagnostic 후 `return 1` 합니다.
- 변경 전/후 차이: 지원 data model 가정이 문서상의 전제에서 executable gate로 추가되어 `check-build`가 실제 binary를 실행하게 되었습니다.
- 직접 확인한 ownership/lifetime/state 관계: 이 check는 scalar/RPN의 `long` 범위, `size_t`/pointer 크기 등 build가 전제한 host representation을 production 실행 전 verification 단계에서 판정합니다.
- 직접 확인한 failure path: unsupported model은 test executable이 성공적으로 compile된 뒤 runtime에 exit 1로 끝납니다. **관찰된 차이:** scaffold에는 compile-time assertion/compile-time failure로 고정되어 있지만 해당 SHA의 실제 코드는 static/typedef assertion이 아니라 runtime `bool`과 exit status를 사용합니다. 고정 scaffold text는 유지하고 이 불일치를 여기에 기록합니다.
- 실행한 테스트와 결과: 미실행. source와 Make target을 검사했으나 data-model binary를 compile/run하지 않았습니다.
- 이 commit을 한 문장으로 설명: LP64 전제를 executable runtime gate로 만들었으며, scaffold의 compile-time 설명과 실제 구현에는 차이가 있습니다.

### `50565bd67e03` — ci: 지원 compiler와 platform matrix 검증

- Importance: **B**
- Tags: **PORTABILITY, TEST**
- Source 역할: established claims를 GCC/Clang, Linux/macOS matrix에서 실행합니다.
- Source classification summary: Adds GCC/Clang Linux and Clang macOS CI with sanitizer coverage.

#### Thread 흐름에서 확인할 구현 역할
- [x] 직전 관련 SHA `ab441fa8737c`와의 차이 중 이 Thread의 흐름에 필요한 부분만 확인하세요.
- [x] CI matrix에서 GCC/Clang Linux와 Clang macOS 조합을 실제 configuration으로 확인하세요.
- [x] 각 job이 어떤 established build/check target을 실행하는지 기록하세요.
- [x] matrix fail-fast disabled 설정이 한 job 실패 시 다른 evidence를 계속 수집하도록 하는지 확인하세요.
- [x] UBSan은 어디서 공통 실행되고 ASan은 어느 host로 제한되는지 configuration을 확인하세요.
- [x] minimal permissions와 explicit timeout가 실제 workflow에 선언되어 있는지 확인하세요.
- [x] 이 commit이 다음 관련 commit의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### Test commit 학습 구분
- 대상 production invariant: source가 확정한 방향은 **supported compiler/platform matrix**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 재현 failure / boundary: source가 확정한 방향은 **compiler/OS-specific regression**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- test technique: source가 확정한 방향은 **CI matrix with sanitizer/check targets**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 통과하는 production path: source가 확정한 방향은 **established build and verification stack**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하는 것: source가 확정한 방향은 **GCC/Clang Linux와 Clang macOS에서 지속적 evidence 확보**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 이 테스트가 증명하지 않는 것: source가 확정한 방향은 **matrix 밖 compiler/OS support를 의미하지 않음**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.
- 성격: source가 확정한 방향은 **broad CI verification**입니다. 실제 test code/fixture를 읽고 구체적인 파일·case·assertion을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `.github/workflows/ci.yml`; `Makefile`의 `check-build`, `test-ubsan`, `test-asan`, `check-data-model`.
- 핵심 코드 발췌 위치: `50565bd67e03:.github/workflows/ci.yml`은 ubuntu-22.04 GCC, ubuntu-22.04 Clang, macOS latest Clang matrix를 정의하고 `fail-fast: false`, `timeout-minutes: 30`, `contents: read`를 설정합니다. 모든 job은 `make check-build`, UBSan을 실행하고 ASan은 Linux 두 job만 실행합니다.
- 변경 전/후 차이: local executable checks를 compiler/OS matrix에서 반복 실행하는 workflow가 추가되었습니다. `check-build`에는 runtime LP64 gate가 포함됩니다.
- 직접 확인한 ownership/lifetime/state 관계: 각 clean GitHub checkout이 독립 workspace를 소유하고 selected compiler를 `CXX`로 Make targets에 전달합니다. matrix job별 artifact/state는 공유하지 않으므로 compiler/OS-specific failure가 다른 evidence를 덮지 않습니다.
- 직접 확인한 failure path: compiler 선택, build/regression/data-model, UBSan, Linux ASan 중 하나가 nonzero면 해당 job이 실패합니다. `fail-fast: false`라 다른 matrix job은 계속됩니다. **보장 경계:** workflow push trigger는 `main`만이며 PR에도 실행됩니다. 또한 CI는 `make check-platform`/`make check`를 호출하지 않아 archive/dependency/leak scripts는 이 matrix evidence에 포함되지 않습니다. 실제 workflow run 결과는 검사하지 않았습니다.
- 실행한 테스트와 결과: 미실행. workflow configuration과 호출 target만 검사했으며 GitHub Actions run 성공을 주장하지 않습니다.
- 이 commit을 한 문장으로 설명: GCC/Clang Linux와 Clang macOS에서 build·regression·LP64·UBSan 및 Linux ASan을 반복하도록 구성했습니다.

## Invariant ledger

| SHA | Source에서 확정된 invariant 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `6e78ced59357` | unit + compile-contract + CLI의 multi-layer verification pattern 시작 | contact unit, repeated public-header compile, private-access expected failure, exact CLI session fixture가 behavior/API/process 층을 분리합니다. | repository 전체 public contract, external packaging, sanitizer/ABI matrix는 아직 없습니다. |
| `4bbbfd191669` | library 전체 public API/ownership negative contracts로 확대 | `-Iinclude`만 사용하는 positive syntax units와 expected-fail API/ownership units, public archive link rule을 확인했습니다. | runtime behavior와 leak/exception cleanup은 compiler contract만으로 증명되지 않습니다. |
| `01271d795d58` | repository 밖 consumer로 packaging/public-header isolation 검증 | 외부 temporary directory에서 copied consumer를 public include + absolute static archive만으로 compile/link/run하고 trap으로 정리합니다. | 한 host/compiler consumer이며 모든 downstream build system/platform을 포괄하지 않습니다. |
| `9e07d3bc86d3` | fixed-seed properties와 large-batch deterministic stress 추가 | fixed LCG seed, scalar 2,048, RPN 4,096, 4,096-job large batch와 first-counterexample reporting을 확인했습니다. | 유한 fixed sample이므로 exhaustive/formal proof는 아닙니다. |
| `45e9bbfd6b75` | portable baseline과 platform checks 분리, ASan/UBSan prerequisite 구분 | `check-build`/`check-portable`/`check-platform` dependency와 UBSan/ASan standalone targets를 확인했습니다. | 실제 `check-platform`/aggregate `check`에는 ASan이 포함되지 않아 scaffold 표현과 차이가 있습니다. |
| `ab441fa8737c` | LP64 data model을 compile-time assertions로 executable claim화 | LP64 sizes를 `const bool`로 검사하고 false에서 exit 1인 executable과 `check-data-model` target을 확인했습니다. | compile-time assertion이 아니라 runtime gate라는 scaffold/implementation 불일치가 있으며 LP64 내부 차이는 남습니다. |
| `50565bd67e03` | GCC/Clang, Linux/macOS matrix에서 established claims 연속 실행 | ubuntu GCC/Clang, macOS Clang matrix가 check-build·UBSan·Linux ASan을 fail-fast false로 실행하도록 구성됩니다. | push는 main만이며 check-platform/archive/dependency/leak와 matrix 밖 host/compiler는 지원 증거가 아닙니다. |

## Failure → Fix → Test 연결

- 이 Thread는 한 개의 bug fix chain보다 verification blind spot을 단계적으로 줄이는 흐름입니다. 각 layer가 이전 layer가 증명하지 못한 무엇을 추가하는지 학습자가 기록하세요.

### 학습자 연결 기록
- 최초 위험/맹점: in-tree unit tests가 통과해도 public header가 private include에 의존하거나, archive가 빠진 object를 숨기거나, API가 우연히 넓어지거나, 특정 compiler/ABI에서만 실패할 수 있습니다.
- 이를 드러낸 실제 failure 또는 test gap: runtime output은 abstractness/const/private shape를 보지 못하고, in-tree link는 relative path와 loose objects를 숨기며, hand-written fixtures는 state-space breadth와 large growth를 제한합니다. 단일 host 실행은 sanitizer/ABI/compiler 차이를 보여 주지 못합니다.
- 수정/강화된 decision: behavior, positive/negative compile, CLI, out-of-tree consumer, fixed-seed property/stress, portable/host target, LP64 gate, CI matrix를 서로 다른 failure 조건으로 누적합니다.
- 해당 코드 위치: `6e78ced59357`~`50565bd67e03`의 `Makefile`, `tests/compile/`, `tests/check_external_consumer.sh`, `tests/property/test_boundary_properties.cpp`, `tests/portability/test_data_model.cpp`, `.github/workflows/ci.yml`.
- 이를 고정하는 regression/evidence: 각 layer 자체가 이전 layer가 볼 수 없는 failure를 nonzero build/test/compile/run status로 드러냅니다. 다만 data-model check는 scaffold 설명과 달리 runtime이며 CI는 check-platform을 실행하지 않는다는 한계를 함께 기록합니다.

## Verification responsibility 변화

- Source 기준 흐름: behavior → API shape → external packaging → breadth/stress → portable/platform separation → ABI → compiler/platform matrix로 검증 책임이 확장됩니다.
- [x] 각 layer가 실행되는 build/CI target과 prerequisite를 실제 코드에서 연결하세요.

### 코드 검사로 복원한 변화

1. `6e78ced59357`: verification 책임이 contact behavior에서 compile-time API shape와 real process transcript로 확장됩니다.
2. `4bbbfd191669`: public-only positive/negative compiler 계약이 library 전체 API와 ownership restrictions를 다룹니다.
3. `01271d795d58`: packaging 책임이 repository 밖 compile/link/run으로 이동해 in-tree assumptions를 제거합니다.
4. `9e07d3bc86d3`: fixed-seed generated inputs와 large batch가 deterministic breadth와 growth를 검증합니다.
5. `45e9bbfd6b75`: portable baseline/UBSan과 host-specific archive/dependency/leak checks, standalone ASan capability가 분리됩니다.
6. `ab441fa8737c`: LP64 가정이 runtime executable gate가 됩니다. scaffold의 compile-time 표현과 실제 code 차이를 명시했습니다.
7. `50565bd67e03`: clean Linux/macOS compiler jobs가 build/regression/data-model/UBSan 및 Linux ASan을 반복합니다. check-platform은 CI 범위 밖입니다.

## Thread 최종 상태

- Source가 확정한 최종 흐름: `unit behavior → compile-time public contract → process fixture → external consumer → deterministic property/stress → portable/platform check layers → ABI gate → CI matrix`
- [x] 마지막 Thread SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] Thread 시작 시점과 비교해 새로 보장되는 invariant를 정리하세요.
- [x] source가 보장하지 않는 영역이나 외부 side effect/stream position 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 Thread 해석

마지막 Thread SHA 기준으로 verification stack은 unit/failure/no-elide tests, public-only positive/negative compiler contracts, real CLI/public integration, out-of-tree consumer, fixed-seed property/stress, deterministic rebuild/output, runtime LP64 gate, UBSan/ASan targets, compiler/OS CI matrix로 구성됩니다. 각 층은 동일한 "테스트 통과"를 반복하는 것이 아니라 source behavior, API shape, packaging, state-space breadth, UB/memory fault, ABI, toolchain variation을 서로 다른 방식으로 관찰합니다.

시작 시점과 비교하면 support claim이 in-tree contact behavior에서 exported archive와 특정 GCC/Clang LP64 hosts까지 넓어졌습니다. 하지만 실제 증거는 구성 코드를 검사한 것이며 실행 결과는 없습니다. 또 data-model check는 compile-time이 아니라 runtime이고, CI는 host-specific `check-platform`을 실행하지 않으며 push trigger는 main에 제한됩니다. 따라서 matrix 밖 compiler/OS와 archive/dependency/leak의 CI 지속 실행은 주장할 수 없습니다.

## 최종 architecture 또는 execution flow 정리

다음 항목은 학습자가 실제 commit code를 읽은 뒤 완성합니다. 완성형 정답을 source 밖에서 추정해 채우지 않습니다.

```text
[입력/호출자]
    ↓
[검증/생성/후보 상태]
    ↓
[핵심 ownership/state transition]
    ↓
[commit/publication point]
    ↓
[output / observable state]

실패 분기:
[throw/failure source] → [cleanup owner] → [보존되는 prior state]
```

- 실제 caller → callee 흐름: source/build change → unit/failure/no-elide → public positive/negative compile → CLI/public integration → external consumer → fixed-seed property/stress → `check-build`/LP64 gate → UBSan/ASan → GitHub Actions compiler/OS jobs.
- 핵심 상태 필드: Make target dependency graph, compile translation units, fixture expected bytes, property `random_state`/`first_failure`, runtime `lp64` bool, CI matrix entries and conditions.
- resource owner / borrowed view: each test process owns its temporary objects; external-consumer script owns and traps a temporary directory; CI jobs own isolated checkouts. Public consumers borrow only installed headers/API and link the archive.
- commit point: 각 verification layer는 compiler/linker/process exit status와 exact assertion/byte comparison으로 claim을 승인합니다. CI job은 모든 configured step success일 때만 green입니다.
- cleanup path: compile/run failure는 nonzero로 상위 Make/CI를 중단하고 scripts/traps가 temporary files를 제거합니다. sanitizer는 configured halt option으로 첫 detected fault를 failure로 만듭니다.
- 최종 invariant 설명: supportable claim은 단일 unit result가 아니라 public API isolation, external packaging, deterministic behavior, LP64 runtime gate, selected sanitizer와 compiler/OS configuration에서 실행 가능하도록 코드화되어 있습니다. 구성 밖 platform과 실행되지 않은 target은 보장으로 확대하지 않습니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub checkout에 필요한 네트워크 연결이 차단되어 build/test command를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] Commit map의 SHA/순서를 그대로 따라 모든 관련 code tree를 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B importance에 맞는 깊이로 code/test evidence를 채웠습니다.
- [x] source가 확정한 invariant와 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] failure path에서 state mutation 전후와 cleanup owner를 설명할 수 있습니다.
- [x] test commit마다 production invariant, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 마지막 상태를 commit history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 08-verification-supported-release-claims.md =====

===== BEGIN FILE: README.md =====
# cpp-foundation Development Threads

## 목적

이 디렉터리는 `commit-importance.md`에 정의된 Development Threads를 따라 실제 commit history와 각 SHA의 코드를 직접 읽으며 프로젝트의 설계 → 구현 → 실패 → 수정 → 검증 과정을 복원하기 위한 학습 골격입니다.

완성형 프로젝트 해설서가 아닙니다. 미리 작성된 내용은 source에서 확정된 thread 구조, commit metadata, 역할, invariant/failure 방향뿐이며, 실제 구현 해석과 코드 증거는 학습자가 채웁니다.

## 권장 학습 순서

1. [`01-direct-ownership-failure-safe-value.md`](01-direct-ownership-failure-safe-value.md)
2. [`02-polymorphic-cloning-owning-aggregate.md`](02-polymorphic-cloning-owning-aggregate.md)
3. [`03-factory-transaction-boundary.md`](03-factory-transaction-boundary.md)
4. [`04-scalar-text-target-projection.md`](04-scalar-text-target-projection.md)
5. [`05-checked-rpn-undefined-arithmetic.md`](05-checked-rpn-undefined-arithmetic.md)
6. [`06-generic-containers-transactional-batch.md`](06-generic-containers-transactional-batch.md)
7. [`07-contactbook-replacement-guarantee.md`](07-contactbook-replacement-guarantee.md)
8. [`08-verification-supported-release-claims.md`](08-verification-supported-release-claims.md)

순서는 source의 Development Threads 순서를 그대로 따릅니다. 동일 commit이 여러 thread에 등장하는 경우 제거하지 않고 각 thread 관점에서 다시 확인합니다.

## Thread 문서 사용법

각 문서에서 먼저 Thread 목표, 핵심 질문, 완료 기준, Commit map을 읽습니다. 이후 commit을 map 순서대로 진행합니다.

각 commit에서는 다음 원칙을 지킵니다.

- 먼저 해당 SHA를 checkout하거나 `git show <sha>`로 정확한 시점의 diff와 파일을 확인합니다.
- 문서가 지목한 type/function/state/test를 해당 SHA에서 직접 찾습니다.
- 필요하면 문서가 지정한 직전 관련 SHA와 비교합니다.
- source에 없는 파일명, 함수명, ownership 관계를 추정해 채우지 않습니다.
- 학습 기록에는 실제 확인한 경로, 심볼, 코드 라인, test case를 근거로 남깁니다.

## 해당 SHA 코드 확인 원칙

**final HEAD의 코드를 과거 commit 설명에 소급해서 사용하지 않습니다.**

후속 refactor나 fix가 이미 적용된 HEAD를 기준으로 과거 설계를 설명하면 failure 원인과 수정 경계가 사라집니다. 반드시 학습 대상 commit의 tree를 기준으로 확인하고, 전후 비교가 필요할 때만 관련 SHA끼리 비교합니다.

권장 최소 기록 형식은 다음과 같습니다.

```text
SHA:
Path:
Symbol / test case:
직전 관련 SHA:
확인한 state / ownership / failure path:
이 코드가 증명하는 invariant:
```

## Importance별 학습 깊이

### S

프로젝트의 핵심 architecture/invariant입니다. 직전 상태, problem, failure 가능성, decision, 핵심 코드, ownership/lifecycle/state transition, failure path, 보장/비보장 범위, 후속 fix/test까지 추적합니다.

### A

주요 subsystem, boundary, failure path, integration point입니다. 핵심 코드와 설계 판단, 전후 state 변화, test evidence까지 확인합니다.

### B

Thread 흐름에서 맡는 구현 역할과 필요한 state/API 변화를 확인합니다. S/A와 동일한 깊이의 분석란을 억지로 만들지 않습니다.

### C

Thread 이해에 필요한 맥락으로만 사용합니다. source에 C commit이 Thread에 포함되지 않았다면 별도 학습 항목을 추가하지 않습니다.

## 실제 코드 삽입 기준

문서에 코드를 붙일 때는 설명용으로 재작성하지 말고 **해당 SHA의 실제 코드**만 사용합니다.

- 핵심 invariant를 직접 만드는 상태 필드나 함수
- ownership transfer, clone, delete, swap, candidate publication 지점
- failure/error branch와 cleanup path
- parser boundary, overflow precondition, stream-state 판정
- production path를 실제 통과하는 regression test
- fix 전/후 차이를 보여 주는 최소 코드

코드 발췌에는 SHA, path, symbol을 함께 적습니다. 긴 파일 전체를 복사하지 않습니다.

## Test commit 학습 방법

Test commit에서는 단순히 "테스트가 통과했다"고 기록하지 않습니다. 반드시 다음을 구분합니다.

- 어떤 production invariant를 대상으로 하는가
- 어떤 failure 또는 boundary를 재현하는가
- 어떤 test technique을 사용하는가
- 실제 어떤 production code path를 통과하는가
- 이 테스트가 증명하는 것
- 이 테스트가 증명하지 않는 것
- broad integration인지 deterministic regression/failure injection인지
- 후속 변경에서 어떤 회귀를 막는가

실행 결과는 사용한 compiler/build target과 함께 학습자가 직접 기록합니다.

## 문서 완료 기준

Thread 하나는 다음 조건을 모두 만족해야 완료입니다.

- Commit map의 모든 SHA를 source 순서대로 확인했습니다.
- S/A/B/C 중요도에 맞는 깊이로 실제 코드 근거를 채웠습니다.
- Invariant ledger에 introduction/strengthening/failure/fix/test 흐름을 실제 코드 증거와 연결했습니다.
- fix commit은 기존 가정 → failure/risk → root cause → 수정 decision → 실제 수정 코드 → regression test가 연결되어 있습니다.
- test commit은 production invariant, failure boundary, technique, production path, 증명/비증명 범위가 구분되어 있습니다.
- ownership/state/responsibility 변화가 의미 있는 thread에서는 transition을 직접 설명할 수 있습니다.
- Thread 최종 상태와 execution/architecture flow를 final HEAD가 아닌 해당 commit sequence를 근거로 설명할 수 있습니다.
===== END FILE: README.md =====

