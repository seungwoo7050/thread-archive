===== BEGIN FILE: 01-direct-ownership-failure-safe-value.md =====
# 직접 소유 객체를 실패에 안전한 값 형식으로 만들기

## 개발 흐름의 목표

직접 할당한 C 문자열 저장소가 단순한 소유 객체에서 독립 복사, 자기 참조 안전성, 강한 예외 보장을 갖는 정규 값으로 발전하는 과정을 복원합니다.

**원문에서 정한 의의:** 단순한 메모리 소유 객체가 성공·별칭 참조·실패 상황에서도 일반 값처럼 복사·대입 가능한 형식으로 발전합니다. 여기서 확립한 복사 후 교환과 새 저장소를 먼저 완성하는 방식은 이후 다형 파이프라인 복사와 트랜잭션 방식의 교체에도 재사용됩니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- `TextBuffer`의 최초 표현 불변식은 무엇이며 왜 내부 null 상태를 허용하지 않는가?
- 깊은 복사가 없을 때 어떤 수명 결합과 이중 해제 위험이 생기는가?
- 복사 후 교환에서 실제 상태 변경이 일어나는 상태 확정 지점은 어디인가?
- `operator+=`가 기존 저장소를 해제하기 전에 새 값을 완성해야 하는 이유는 무엇인가?
- 실패 주입과 no-elide 빌드가 성공 경로 테스트에서 보이지 않는 어떤 위험을 드러내는가?

## 완료 기준

- [x] 각 단계에서 포인터, 크기, NUL 종료 문자의 관계를 실제 코드로 설명할 수 있습니다.
- [x] 복사 생성과 대입에서 원본/대상 할당이 독립적임을 해당 SHA 코드와 테스트로 증명할 수 있습니다.
- [x] 할당 실패 시 대상 상태와 현재 할당 수 기준값이 유지되는 경로를 추적할 수 있습니다.
- [x] 복사 생략이 없어도 반환값과 복사 후 교환이 안전한 이유를 실제 호출 흐름으로 설명할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- 직접 소유한 자원은 정확히 한 번 해제되고, 복사는 포인터 alias가 아닌 독립 소유권을 만든다.
- 강한 예외 보장이 문서화된 연산은 할당 실패 시 owning object의 관찰 가능한 상태를 바꾸지 않는다.
- 완성되지 않은 후보는 publish하지 않는다.
- `c_str()`로 얻은 비소유 포인터는 원본의 수명을 넘거나 소유권을 획득하지 않는다.

### 주요 구현 난점

- 수동 할당 C 문자열의 깊은 복사와 exception-safe 대입 구현.
- 할당 실패 sweep과 할당 블록 accounting으로 강한 예외 보장을 검증.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `aa3b5ba6c3c4` | feat(buffer): 종료 문자를 포함한 문자열 저장소 소유 | A | OWNERSHIP, CORE | null이 아닌 NUL 종료 `char[]` 소유 표현과 예외를 던지지 않는 `swap()`을 확립합니다. |
| 2 | `0bc528c7d58e` | feat(buffer): 깊은 복사와 정규 대입 구현 | S | OWNERSHIP, EXCEPTION, CORE | 독립적인 깊은 복사와 복사 후 교환 대입을 추가합니다. |
| 3 | `93faed0d67a2` | feat(buffer): 결합·비교·출력 연산 제공 | A | OWNERSHIP, EXCEPTION, CORE | 자기 참조에도 안전하게 새 저장소를 완성한 뒤 반영하는 결합 연산으로 직접 소유 객체를 확장합니다. |
| 4 | `47134f9e3b29` | test(buffer): 할당 실패와 복사 생략 비활성화 검증 | A | TEST, EXCEPTION, OWNERSHIP | 관찰된 모든 할당 실패를 주입하고 복사 생략을 비활성화해 검증합니다. |

## 커밋별 학습 기록

### `aa3b5ba6c3c4` — feat(buffer): 종료 문자를 포함한 문자열 저장소 소유

- 중요도: **A**
- 태그: **OWNERSHIP, CORE**
- 원문에서 정한 역할: null이 아닌 NUL 종료 `char[]` 소유 표현과 예외를 던지지 않는 `swap()`을 확립합니다.
<!-- 원문 분류 요약: Introduces an owning NUL-terminated `char` buffer with checked access and non-throwing swap. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 해당 SHA에서 `TextBuffer`의 data 포인터와 size를 저장하는 상태 필드를 찾고, default/null-입력 생성이 동일한 empty 표현을 만드는 초기화 순서를 기록하세요.
- [x] `size() + 1` 할당과 마지막 NUL byte를 만드는 생성자/소멸자 경로를 함께 추적하세요.
- [x] const/mutable `at()`가 terminator 위치를 logical range에서 제외하는 브랜치와 `c_str()` 반환의 borrowed-수명 조건을 확인하세요.
- [x] `swap()`이 포인터와 size를 함께 교환하며 throw하지 않도록 구성된 실제 코드를 찾으세요.
- [x] 이 SHA에서 copy가 어떻게 금지되는지 공개 인터페이스에서 확인하고, 왜 아직 일반 값 형식이 아닌지 기록하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `0bc528c7d58e`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/TextBuffer.hpp`의 `TextBuffer`, `data_`, `size_`; `src/TextBuffer.cpp`의 default/`const char *` 생성자, 소멸자, `at()`, `c_str()`, `swap()`.
- 핵심 코드 발췌 위치: `aa3b5ba6c3c4:src/TextBuffer.cpp`에서 empty 값도 `new char[1]`로 만들고 `data_[0] = '\0'`을 기록합니다. 문자열 생성자는 `size_ + 1`을 할당해 terminator까지 복사하며, `swap()`은 `data_`와 `size_`를 함께 교환합니다.
- 변경 전/후 차이: 이 커밋에서 직접 소유하는 문자열 값이 처음 생겼습니다. null 입력과 default 생성은 모두 non-null empty 표현으로 정규화되지만, 복사 생성자와 대입 연산자는 비공개이므로 독립 값 복사는 아직 제공되지 않습니다.
- 직접 확인한 소유권·수명·상태 관계: `TextBuffer` 한 객체가 `data_` 배열의 유일한 소유자이고 소멸자가 `delete[]` 합니다. `size_`는 terminator를 제외한 logical length이며 `data_[size_]`는 항상 NUL입니다. `c_str()`는 소유권을 넘기지 않는 비소유 포인터라 객체 파괴나 변경 이후 사용할 수 없습니다.
- 직접 확인한 실패 처리: 생성자 할당이 실패하면 객체 생성 자체가 끝나지 않아 소유자가 생기지 않습니다. `at(index)`는 `index >= size_`에서 예외를 던져 terminator 접근을 거부하며 상태를 변경하지 않습니다. 이 시점에는 copy 실패 경로가 API 밖입니다.
- 실행한 테스트와 결과: 미실행. 저장소 체크아웃 네트워크가 차단되어 명령은 수행하지 않았고, 지정 SHA의 구현·Make 대상·테스트 소스만 검사했습니다.
- 이 커밋을 한 문장으로 설명: null 내부 상태 없이 NUL-terminated `char[]`를 단일 소유하는 최소 `TextBuffer` 표현을 확립했습니다.

### `0bc528c7d58e` — feat(buffer): 깊은 복사와 정규 대입 구현

- 중요도: **S**
- 태그: **OWNERSHIP, EXCEPTION, CORE**
- 원문에서 정한 역할: 독립적인 깊은 복사와 복사 후 교환 대입을 추가합니다.
<!-- 원문 분류 요약: Adds deep copying and copy-and-swap assignment to `TextBuffer`. -->

#### 이 커밋 직전 상태와 문제
- 직전 관련 개발 흐름 SHA `aa3b5ba6c3c4`를 먼저 체크아웃하여 이 커밋이 추가되기 전 표현/소유권/상태-반영 방식을 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 직전 관련 SHA `aa3b5ba6c3c4`와 비교해 복사 생성자/대입 선언이 공개 규약에 어떻게 추가되는지 확인하세요.
- [x] 복사 생성자가 원본과 별도 `size + 1` 할당을 만들고 terminator까지 복사하는 코드를 추적하세요.
- [x] 대입에서 임시 객체 생성 → 예외를 던지지 않는 `swap()` → 임시 객체 소멸 순서를 실제 코드 라인으로 기록하세요.
- [x] 할당이 임시 객체 생성 중 실패할 때 대상 상태가 아직 변경되지 않았음을 제어 흐름으로 증명하세요.
- [x] alias를 통한 자기 대입이 별도 `this == &other` 브랜치 없이도 안전한 이유를 객체 수명과 할당 기준으로 설명하세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `93faed0d67a2`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/TextBuffer.hpp`의 공개 복사 생성자/대입; `src/TextBuffer.cpp`의 `TextBuffer(const TextBuffer&)`, `operator=`, `swap()`.
- 핵심 코드 발췌 위치: `0bc528c7d58e:src/TextBuffer.cpp`의 복사 생성자는 `new char[other.size_ + 1]` 후 `std::memcpy(..., size_ + 1)`를 수행합니다. 대입의 핵심은 `TextBuffer copy(other); swap(copy); return *this;`입니다.
- 변경 전/후 차이: 직전 SHA에서는 복사가 비공개였습니다. 이후 원본과 별도 저장소를 가진 깊은 복사가 공개 규약이 되었고 대입이 대상을 직접 덮지 않고 완성된 임시 객체와 교환합니다.
- 직접 확인한 소유권·수명·상태 관계: 복사 생성 성공 후 원본과 copy는 서로 다른 `char[]` 소유자입니다. 대입 전에는 대상이 기존 배열을, 임시 객체가 새 배열을 소유합니다. `swap()` 뒤 대상이 새 배열을, 임시 객체가 이전 대상 배열을 소유하고 scope 종료 시 임시 객체 소멸자가 이전 배열을 해제합니다.
- 직접 확인한 실패 처리: 임시 객체 복사 생성 중 `new[]`가 실패하면 `swap()`에 도달하지 않아 대상의 포인터, size, bytes가 그대로입니다. 자기 대입도 원본이 곧 대상이어도 먼저 독립 copy를 만들기 때문에 별도 alias 브랜치 없이 안전합니다.
- 실행한 테스트와 결과: 미실행. 저장소 체크아웃 네트워크가 차단되어 명령은 수행하지 않았고, 지정 SHA의 구현·Make 대상·테스트 소스만 검사했습니다.
- 이 커밋을 한 문장으로 설명: 깊은 복사와 복사 후 교환으로 `TextBuffer`를 독립 소유권과 강한 대입 보장을 가진 값으로 바꿨습니다.

### `93faed0d67a2` — feat(buffer): 결합·비교·출력 연산 제공

- 중요도: **A**
- 태그: **OWNERSHIP, EXCEPTION, CORE**
- 원문에서 정한 역할: 자기 참조에도 안전하게 새 저장소를 완성한 뒤 반영하는 결합 연산으로 직접 소유 객체를 확장합니다.
<!-- 원문 분류 요약: Adds concatenation, comparison, and stream operations with overflow and allocate-before-commit handling. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `0bc528c7d58e`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] `operator+=`에서 `size + other.size + 1` 오버플로를 실제 할당 전에 검사하는 식과 브랜치를 확인하세요.
- [x] joined storage를 완성한 뒤 기존 저장소를 release하는 정확한 순서를 추적하고, 실패 시 old value가 남는 지점을 표시하세요.
- [x] self-concatenation에서 `other`가 `*this`와 alias여도 원본 bytes가 release 전에 모두 사용되는지 코드 순서로 확인하세요.
- [x] non-member `operator+`가 copy + compound addition을 재사용하는 호출자/피호출자 관계를 기록하세요.
- [x] 비교 및 스트림 삽입이 할당 표현을 노출하지 않고 value semantics만 제공하는지 공개 API를 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `47134f9e3b29`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/TextBuffer.hpp`의 `operator+=`, 비교·스트림 연산 선언; `src/TextBuffer.cpp`의 `operator+=`, non-member `operator+`, `operator==`, `operator<`, `operator<<`.
- 핵심 코드 발췌 위치: `93faed0d67a2:src/TextBuffer.cpp`의 `operator+=`는 `other.size_ > max - size_ - 1`을 먼저 검사하고, `joined` 배열에 기존 bytes와 `other`의 terminator까지 복사한 다음에만 old `data_`를 해제하고 새 포인터/size를 게시합니다.
- 변경 전/후 차이: 값 복사만 가능하던 상태에서 결합·비교·출력으로 확장되었습니다. 결합은 기존 저장소를 먼저 변경하는 대신 detached 할당을 완성한 뒤 커밋합니다.
- 직접 확인한 소유권·수명·상태 관계: 새 배열은 함수 내부 후보 소유자이고 모든 복사가 끝날 때까지 기존 `data_`가 원본으로 살아 있습니다. 커밋 뒤 `TextBuffer`가 후보를 소유합니다. 비교와 스트림 삽입은 내부 포인터 소유권을 노출하지 않고 bytes의 값만 관찰합니다.
- 직접 확인한 실패 처리: 길이 합 오버플로는 할당 전에 거부됩니다. 할당 또는 복사 준비 단계의 예외에서는 기존 배열이 해제되지 않습니다. `buffer += buffer`에서도 원본 bytes를 읽는 동안 기존 저장소가 살아 있어 alias가 안전합니다. final destination 스트림 실패의 되돌리기는 이 커밋의 보장이 아닙니다.
- 실행한 테스트와 결과: 미실행. 저장소 체크아웃 네트워크가 차단되어 명령은 수행하지 않았고, 지정 SHA의 구현·Make 대상·테스트 소스만 검사했습니다.
- 이 커밋을 한 문장으로 설명: allocate-before-커밋을 결합 연산까지 확장해 오버플로, 할당 실패, self-alias를 처리했습니다.

### `47134f9e3b29` — test(buffer): 할당 실패와 복사 생략 비활성화 검증

- 중요도: **A**
- 태그: **TEST, EXCEPTION, OWNERSHIP**
- 원문에서 정한 역할: 관찰된 모든 할당 실패를 주입하고 복사 생략을 비활성화해 검증합니다.
<!-- 원문 분류 요약: Adds deterministic allocation-failure injection and no-elide builds for buffer operations. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `93faed0d67a2`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] 테스트 executable에서 global 할당 함수가 counted `malloc`-backed 구현으로 교체되는 지점을 찾으세요.
- [x] 생성, 복사 생성, 대입, 별칭을 통한 자기 대입, `+`, `+=` 각각에서 관찰된 할당 site를 어떻게 순회해 실패를 주입하는지 기록하세요.
- [x] 각 실패 후 object 상태와 현재 할당 수 기준값을 어떤 검사문으로 확인하는지 구분하세요.
- [x] 복사 생략을 비활성화한 별도 빌드 대상/flags와 실행 테스트가 무엇인지 확인하세요.
- [x] 이 테스트가 production 코드의 어떤 할당/copy path를 통과하며, 일반 unit 테스트가 놓치던 임시 객체 수명을 무엇으로 드러내는지 적으세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **TextBuffer의 강한 예외 보장과 leak freedom**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **할당 실패 및 copy-elision 부재**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **결정적 할당-실패 sweep + no-elide 빌드**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **생성/copy/대입/addition/compound-addition 할당 paths**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **강한 예외 보장과 leak baseline이 관찰된 할당 sites에서 유지됨**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **관찰되지 않은 실행 경로나 다른 allocator 환경까지 자동으로 증명하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 회귀 테스트 / 실패-injection**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/support/FailingNew.hpp`, `tests/support/FailingNew.cpp`; `tests/failure/test_buffer_failure.cpp`; `Makefile`의 `failure-test`, `test-no-elide` 계열 대상.
- 핵심 코드 발췌 위치: `47134f9e3b29:tests/failure/test_buffer_failure.cpp`의 생성/copy/대입/aliased 대입/`+`/`+=` 실패 sweep과, `FailingNew`의 할당-attempt 및 할당 블록 counter. `Makefile`은 `-fno-elide-constructors`를 적용한 별도 실행 파일을 구성합니다.
- 변경 전/후 차이: production 동작은 바꾸지 않고, 정상 경로만 보던 검증에 관찰된 각 할당 attempt의 결정적 실패와 copy-elision 비활성화 실행 구성을 추가했습니다.
- 직접 확인한 소유권·수명·상태 관계: 실패 controller가 정확한 할당 attempt를 실패시키고 테스트는 예외 후 원본/대상 text와 live block baseline을 비교합니다. no-elide 빌드는 반환 임시 객체와 복사/소멸이 실제로 발생해도 소유자가 중복 해제되지 않는지 노출합니다.
- 직접 확인한 실패 처리: 각 연산을 한 번 성공시켜 할당 횟수를 관찰한 뒤 1부터 그 횟수까지 실패 지점을 이동합니다. 대입과 `+=`는 기존 대상 값 유지, 생성자/`+`는 partial object leak 없음, 모든 case는 live block baseline 복구를 검사하도록 작성되어 있습니다. 관찰하지 않은 allocator 동작이나 실행 경로까지 증명하지는 않습니다.
- 실행한 테스트와 결과: 미실행. 저장소 체크아웃 네트워크가 차단되어 명령은 수행하지 않았고, 지정 SHA의 구현·Make 대상·테스트 소스만 검사했습니다. 실행 대상으로 확인한 명령은 `make failure-test`와 no-elide 테스트 대상입니다.
- 이 커밋을 한 문장으로 설명: 결정적 할당 실패와 no-elide 구성을 통해 `TextBuffer`의 강한 보장과 단일 해제를 회귀 계약으로 만들었습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `aa3b5ba6c3c4` | non-null `char[]` 소유권과 NUL 종료 표현, 예외를 던지지 않는 `swap()` 도입 | `TextBuffer`가 `char *data_`와 `size_`를 갖고 모든 생성자가 non-null NUL-terminated 배열을 만들며 소멸자가 `delete[]` 합니다. | 복사가 비공개이므로 독립 value copy와 copy 실패 보장은 아직 없습니다. |
| `0bc528c7d58e` | 독립 깊은 복사와 복사 후 교환 대입으로 일반 값 형식/strong 대입 guarantee 확립 | 복사 생성자가 별도 `size + 1` 배열을 만들고 대입이 임시 객체 생성 뒤 `swap()`으로만 게시합니다. | 결합 같은 후속 변경 연산의 오버플로와 실패 보장은 아직 다루지 않습니다. |
| `93faed0d67a2` | allocate-before-커밋 결합과 self-concatenation 안전성으로 패턴 확장 | `operator+=`가 길이 오버플로를 먼저 검사하고 detached joined 배열을 완성한 후 기존 저장소를 교체합니다. | destination 스트림의 최종 write 실패 되돌리기와 관찰하지 않은 allocator 특성은 범위 밖입니다. |
| `47134f9e3b29` | 모든 관찰 할당 실패와 no-elide 조건에서 보장 검증 | `FailingNew`와 실패 sweep이 각 관찰 할당 site 뒤 상태/현재 할당 블록 수 기준값을 검사하고 no-elide 실행 파일을 별도로 만듭니다. | 실행 환경 전체나 관찰되지 않은 path에 대한 형식 증명은 아닙니다. |

## 문제 → 수정 → 검증 연결

- `0bc528c7d58e`: 깊은 복사 + 복사 후 교환으로 대입 실패 시 대상 보존을 설계합니다.
- `93faed0d67a2`: allocate-before-커밋으로 composition까지 동일한 실패 discipline을 확장합니다.
- `47134f9e3b29`: 할당 실패 sweep과 no-elide 빌드로 이 보장을 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: 직접 소유 포인터를 shallow copy하거나, 대상 저장소를 먼저 해제한 뒤 새 값을 만들면 alias·double free·실패 중 상태 loss가 발생합니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: 최초 구현은 copy를 금지해 위험을 피했지만 일반 값 형식이 아니었고, 정상 unit 테스트만으로는 할당 중간 실패와 반환 임시 객체 수명을 확인할 수 없었습니다.
- 수정/강화된 결정: 복사 생성자는 별도 할당을 만들고, 대입과 composition은 throw 가능한 준비를 detached object/storage에서 끝낸 뒤 예외를 던지지 않는 반영을 수행합니다.
- 해당 코드 위치: `0bc528c7d58e:src/TextBuffer.cpp`의 복사 생성자와 `operator=`, `93faed0d67a2:src/TextBuffer.cpp`의 `operator+=`.
- 이를 고정하는 회귀/근거: `47134f9e3b29:tests/failure/test_buffer_failure.cpp`, `tests/support/FailingNew.cpp`, `Makefile`의 no-elide 구성.

## 소유권·상태·담당 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: non-null `char[]` 소유권과 NUL 종료 표현, 예외를 던지지 않는 `swap()` 도입
- 개발 흐름 종료 상태: 모든 관찰 할당 실패와 no-elide 조건에서 보장 검증
- [x] 중간 커밋마다 소유자/상태 publisher/정리 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned 상태가 함께 등장하면 각각의 수명 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `aa3b5ba6c3c4`: 객체가 non-null `char[]`의 소유자가 되고 `c_str()`만 비소유 참조를 제공합니다.
2. `0bc528c7d58e`: 복사 소유자를 원본과 분리하고, 대입의 상태 publisher를 예외를 던지지 않는 `swap()` 하나로 제한합니다.
3. `93faed0d67a2`: 동일한 detached preparation을 결합에 적용합니다. 기존 저장소는 후보 완성 전까지 원본이자 되돌리기 상태로 남습니다.
4. `47134f9e3b29`: 실패 controller가 후보 생성의 각 할당을 끊어도 기존 소유자와 할당 블록 수가 보존되는지 검사합니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `construction → owned representation → deep copy / assignment → composition → injected-failure verification`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준 호출 흐름은 생성자/copy가 독립 배열을 만든 뒤, 대입은 `TextBuffer copy(other)`를 완성하고 `swap(copy)`로 게시하며, `operator+`는 value copy 후 `operator+=`를 재사용하는 형태입니다. `operator+=`는 오버플로 검사와 detached 할당을 모두 통과한 뒤에만 포인터와 size를 바꿉니다.

시작 시점과 비교하면 copy 금지 소유자가 deep-copy 가능한 일반 값 형식이 되었고, 대입과 composition 모두 할당 실패에서 기존 observable value를 보존합니다. 남는 경계는 `c_str()` 비소유 포인터의 invalidation, destination 스트림 자체의 write 실패, 테스트가 관찰하지 못한 allocator/환경입니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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

- 실제 caller → callee 흐름: 생성자 또는 caller 연산 → `TextBuffer` 복사 생성자 / `operator+=` → 할당·byte copy → `swap()` 또는 포인터 반영 → 비교·`c_str()`·스트림을 통한 관찰.
- 핵심 상태 필드: `char *data_`, `std::size_t size_`, 그리고 불변식 `data_ != 0`, `data_[size_] == '\0'`.
- 자원 소유자 / 비소유 참조: 각 `TextBuffer`가 자신의 `data_`를 단독 소유하고 `c_str()` 결과만 객체 수명에 종속된 비소유 참조입니다.
- 상태 확정 지점: 대입은 `swap(copy)`, `operator+=`는 joined bytes 완성 후 기존 배열 해제와 새 `data_`/`size_` 대입입니다.
- 정리 path: 실패 전 후보 생성자가 소유권을 얻지 못하거나 local 임시 객체가 자신이 소유한 배열을 파괴합니다. 대상의 기존 배열은 커밋 전까지 유지됩니다.
- 최종 불변식 설명: 성공·self-alias·관찰된 할당 실패·copy-elision 부재에서 각 배열은 정확히 한 소유자에게 속하고, strong-guarantee 연산은 완성된 값만 publish합니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 01-direct-ownership-failure-safe-value.md =====

===== BEGIN FILE: 02-polymorphic-cloning-owning-aggregate.md =====
# 다형 복제를 독립 복사 가능한 소유 집합으로 만들기

## 개발 흐름의 목표

가상 인터페이스 자체보다 더 어려운 문제인 동적 객체의 생성·복제·소유·파괴를 추적하고, 여러 복제를 소유하는 aggregate가 실패 중에도 누수 없이 정규 값으로 동작하는 과정을 복원합니다.

**원문에서 정한 의의:** 이 개발 흐름은 C++ 객체 모델의 핵심 발전 과정을 다룹니다. 런타임 다형성만으로는 부족하며, 각 동적 객체의 생성·복사·소유·파괴 주체를 정하고 복제가 중간에 실패해도 누수나 기존 집합 손상이 발생하지 않음을 검증합니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- base object 복사 대신 `clone()`이 필요한 이유는 무엇인가?
- virtual 소멸자가 소유권 protocol의 일부인 이유는 무엇인가?
- `FormatPipeline::append()`는 borrowed prototype을 어느 시점에 owned 복제로 바꾸는가?
- 복사 생성자 도중 복제가 실패하면 왜 파이프라인 소멸자만으로 정리가 불가능한가?
- 복사 생성과 대입 실패가 서로 다른 정리 mechanism을 요구하는 지점은 어디인가?

## 완료 기준

- [x] prototype, 복제, 파이프라인 slot 각각의 소유자와 수명을 커밋별로 그릴 수 있습니다.
- [x] 파이프라인 복사 생성자의 partial-생성 정리와 대입의 복사 후 교환을 구분해 설명할 수 있습니다.
- [x] abstractness, 가상 소멸, 복제 소유권이 각각 어떤 테스트/컴파일 규약으로 고정되는지 찾을 수 있습니다.
- [x] 복제 실패 sweep에서 원본과 destination이 각각 어떤 상태로 남는지 실제 테스트를 근거로 설명할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- polymorphically owned 자원은 정확히 한 번 해제되고, copying은 동적 객체의 독립 소유권을 만든다.
- 완성되지 않은 partial 파이프라인은 임시 객체/partial 소유자 내부에만 존재하며 publish되지 않는다.
- 강한 예외 보장이 적용되는 대입은 복제 실패 시 destination 관찰 가능한 상태를 보존합니다.

### 주요 구현 난점

- `clone()`이 raw 포인터를 반환하는 heterogeneous polymorphic object의 소유권과 copying.
- 생성자가 완료되지 않아 소멸자가 호출되지 않는 상황에서 partial clones 정리.
- 복제 실패 sweep과 살아 있는 객체 accounting.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `835d87865762` | feat(format): 다형적 formatter 인터페이스 정의 | S | ARCH, POLYMORPHISM, OWNERSHIP | 포매터의 추상 동작, 가상 소멸, 가상 복사를 정의합니다. |
| 2 | `62ed45f8adf9` | feat(format): formatter 소유 pipeline 구현 | S | ARCH, POLYMORPHISM, OWNERSHIP | 파이프라인이 포매터의 수명을 빌리지 않고 복제본을 직접 소유하게 합니다. |
| 3 | `bf4d9bed705c` | feat(format): pipeline 깊은 복사 구현 | S | OWNERSHIP, EXCEPTION, POLYMORPHISM | 서로 다른 동적 형식의 소유 객체를 깊게 복사하고 부분 생성 실패를 정리합니다. |
| 4 | `0427713637b8` | test(format): 가상 소멸·추상 계약·CLI 검증 | A | API, TEST, POLYMORPHISM | 추상 클래스 여부, 가상 소멸, 복제본 소유권, 공개 헤더, CLI를 검증합니다. |
| 5 | `2c99290b9268` | test(format): 복제 실패 뒤 부분 객체 정리 검증 | A | TEST, EXCEPTION, POLYMORPHISM | 파이프라인 복사 생성과 대입 중 모든 복제 실패 위치를 순회 검증합니다. |

## 커밋별 학습 기록

### `835d87865762` — feat(format): 다형적 formatter 인터페이스 정의

- 중요도: **S**
- 태그: **ARCH, POLYMORPHISM, OWNERSHIP**
- 원문에서 정한 역할: 포매터의 추상 동작, 가상 소멸, 가상 복사를 정의합니다.
<!-- 원문 분류 요약: Defines the abstract formatter interface, virtual destruction, cloning, and concrete transformations. -->

#### 이 커밋 직전 상태와 문제
- 이 개발 흐름의 첫 커밋이므로, `git show <sha>^`가 가능한 경우 parent에서 관련 type/기능이 없거나 다른 형태였는지 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] `Formatter` 선언에서 abstractness를 만드는 pure virtual functions, virtual 소멸자, `clone()`, `apply()`, name 규약을 확인하세요.
- [x] 각 concrete 포매터가 `clone()`에서 동적 type을 보존한 독립 heap object를 만드는 실제 코드를 찾으세요.
- [x] 앞부분/뒷부분 포매터가 자신의 `TextBuffer` configuration을 소유하는 상태와 copy semantics를 확인하세요.
- [x] uppercase 구현에서 `std::toupper` 호출 전에 plain `char`를 `unsigned char`로 변환하는 경로를 확인하세요.
- [x] caller가 derived type을 몰라도 `Formatter&`/포인터를 통해 동작·복제·삭제할 수 있는 call graph를 그리세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `62ed45f8adf9`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/Formatter.hpp`의 `Formatter`, `UppercaseFormatter`, `PrefixFormatter`, `SuffixFormatter`; `src/Formatter.cpp`의 소멸자, `clone()`, `apply()`, `name()` 구현.
- 핵심 코드 발췌 위치: `835d87865762:include/cppf/Formatter.hpp`에서 `Formatter`는 virtual 소멸자와 pure virtual `clone()`, `apply()`, `name()`을 선언합니다. `src/Formatter.cpp`의 각 `clone()`은 `new <구체 타입>(*this)`를 반환하며 uppercase 변환은 `std::toupper(static_cast<unsigned char>(...))`를 사용합니다.
- 변경 전/후 차이: parent에는 포매터 계층이 없었습니다. 이 커밋에서 호출자가 구체 타입을 몰라도 base reference/포인터로 변환·복제·삭제할 수 있는 object-model 규약이 생겼습니다.
- 직접 확인한 소유권·수명·상태 관계: `PrefixFormatter::prefix_`와 `SuffixFormatter::suffix_`는 각 포매터가 소유하는 `TextBuffer`입니다. `clone()` 성공 시 반환 포인터의 소유권은 호출자에게 이전되고, virtual 소멸자는 base 포인터 삭제가 실제 동적 소멸자까지 도달하게 합니다.
- 직접 확인한 실패 처리: `new` 또는 `TextBuffer` 복사가 실패하면 `clone()`은 포인터를 반환하지 않습니다. 이 SHA에는 여러 복제를 모아 관리하는 aggregate나 local guard가 아직 없으므로, 성공한 raw 포인터를 누가 즉시 소유하는지는 caller protocol에 남아 있습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 header와 구현을 검사했으며 빌드·테스트 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 가상 소멸과 virtual copying을 포함한 포매터 소유권 protocol을 정의했습니다.

### `62ed45f8adf9` — feat(format): formatter 소유 pipeline 구현

- 중요도: **S**
- 태그: **ARCH, POLYMORPHISM, OWNERSHIP**
- 원문에서 정한 역할: 파이프라인이 포매터의 수명을 빌리지 않고 복제본을 직접 소유하게 합니다.
<!-- 원문 분류 요약: Introduces a bounded pipeline that owns formatter clones and applies them in order. -->

#### 이 커밋 직전 상태와 문제
- 직전 관련 개발 흐름 SHA `835d87865762`를 먼저 체크아웃하여 이 커밋이 추가되기 전 표현/소유권/상태-반영 방식을 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] `FormatPipeline`의 fixed 포인터 array, size, capacity 표현을 찾아 null/valid 앞부분 상태를 기록하세요.
- [x] `append()`에서 capacity check → `clone()` → slot store → size increment의 순서를 실제 코드로 확인하세요.
- [x] 입력 포매터 reference는 borrowed이고 복제는 파이프라인-owned가 되는 소유권 전환 시점을 표시하세요.
- [x] 소멸자가 성공적으로 저장된 복제만 정확히 한 번 `delete`하는 범위를 확인하세요.
- [x] `apply()`가 입력 copy를 만들고 삽입 order대로 step을 fold하는 경로와 empty-파이프라인 identity를 확인하세요.
- [x] 이 SHA에서 파이프라인 copy가 여전히 금지되어 있는 공개/비공개 declaration을 확인하세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `bf4d9bed705c`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/FormatPipeline.hpp`의 `max_steps`, `steps_`, `size_`; `src/FormatPipeline.cpp`의 생성자, 소멸자, `append()`, `apply()`, `swap()`.
- 핵심 코드 발췌 위치: `62ed45f8adf9:src/FormatPipeline.cpp`의 `append()`는 capacity를 먼저 검사하고 `formatter.clone()`을 호출한 뒤 `steps_[size_]`에 저장하고 마지막에 `++size_` 합니다. 소멸자는 `[0, size_)`의 포인터만 `delete`합니다.
- 변경 전/후 차이: 포매터 객체 단위의 복제 규약에서, 최대 8개 복제를 삽입 order로 직접 소유하고 실행하는 aggregate가 추가되었습니다. 복사 생성자와 대입은 여전히 비공개입니다.
- 직접 확인한 소유권·수명·상태 관계: `append()` 인자의 `Formatter&`는 호출자 소유의 borrowed prototype입니다. `clone()`이 반환된 순간 local `copy`가 미게시 소유자가 되고, slot 저장과 size 증가가 끝나면 파이프라인이 복제의 유일한 소유자가 됩니다. 유효 상태는 `steps_[0..size_)`가 소유 포인터이고 나머지는 null인 앞부분입니다.
- 직접 확인한 실패 처리: capacity 초과는 복제 전에 거부됩니다. `clone()` 실패 시 slot과 `size_`가 변경되지 않습니다. 성공 뒤에는 소멸자가 active 앞부분을 삭제합니다. 다만 파이프라인 자체의 독립 깊은 복사와 복사 도중 부분 복제 정리는 아직 제공되지 않습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 header와 구현을 검사했으며 빌드·테스트 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: borrowed 포매터를 owned 복제로 바꾸어 순서대로 실행하는 bounded 파이프라인을 만들었습니다.

### `bf4d9bed705c` — feat(format): pipeline 깊은 복사 구현

- 중요도: **S**
- 태그: **OWNERSHIP, EXCEPTION, POLYMORPHISM**
- 원문에서 정한 역할: 서로 다른 동적 형식의 소유 객체를 깊게 복사하고 부분 생성 실패를 정리합니다.
<!-- 원문 분류 요약: Implements deep pipeline copying, partial-construction cleanup, and copy-and-swap assignment. -->

#### 이 커밋 직전 상태와 문제
- 직전 관련 개발 흐름 SHA `62ed45f8adf9`를 먼저 체크아웃하여 이 커밋이 추가되기 전 표현/소유권/상태-반영 방식을 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 직전 관련 SHA `62ed45f8adf9`와 비교해 복사 생성자와 대입이 어떻게 열리는지 확인하세요.
- [x] 복사 생성자 시작 시 포인터 array 전체를 null-initialize하는 순서를 찾고, 복제 성공 앞부분의 표현을 기록하세요.
- [x] 중간 `clone()` 실패 시 catch block이 이미 생성된 앞부분을 직접 delete하고 rethrow하는 코드를 추적하세요.
- [x] 생성자가 완료되지 않으면 소멸자가 호출되지 않는다는 사실이 왜이 explicit 정리를 필요로 하는지 실제 경로에 연결하세요.
- [x] 대입의 complete-copy → swap → old-상태 소멸 흐름과 destination 보존을 확인하세요.
- [x] 동적 포매터 type과 삽입 order가 copy 후 유지되는지 복제/apply 경로로 확인하세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `0427713637b8`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/FormatPipeline.hpp`의 공개 복사 생성자/대입; `src/FormatPipeline.cpp`의 복사 생성자, catch 정리, `operator=`, `swap()`.
- 핵심 코드 발췌 위치: `bf4d9bed705c:src/FormatPipeline.cpp`에서 복사 생성자는 모든 `steps_`를 null로 초기화한 뒤 `append(*other.steps_[index])`를 반복합니다. catch block은 현재 `size_`만큼 직접 `delete`하고 rethrow하며, 대입은 `FormatPipeline copy(other); swap(copy);`입니다.
- 변경 전/후 차이: 직전 SHA에서 복사가 금지됐지만, 이후 동적 type과 삽입 order를 유지하는 heterogeneous 깊은 복사가 공개 규약이 되었습니다. 실패한 생성자와 실패한 대입이 서로 다른 정리 경로를 사용합니다.
- 직접 확인한 소유권·수명·상태 관계: 복사 생성자의 성공 앞부분은 아직 완성되지 않은 `this`가 임시로 소유합니다. 생성 성공 뒤 새 파이프라인이 모든 복제를 소유합니다. 대입에서는 완성된 local `copy`가 후보이고 `swap()` 뒤 old destination 복제들은 local object로 이동해 scope 종료 때 삭제됩니다.
- 직접 확인한 실패 처리: 복사 생성자 중 복제 실패 시 객체 소멸자가 호출되지 않으므로 catch가 이미 만든 앞부분을 직접 삭제해야 합니다. 대입 중 같은 실패는 local `copy` 생성에서 끝나 `swap()`에 도달하지 않으므로 destination의 size, step 순서, 동작이 보존됩니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 구현과 후속 테스트가 겨냥하는 경로를 코드로 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: explicit partial-생성 정리와 복사 후 교환으로 polymorphic aggregate를 실패-safe 일반 값 형식으로 만들었습니다.

### `0427713637b8` — test(format): 가상 소멸·추상 계약·CLI 검증

- 중요도: **A**
- 태그: **API, TEST, POLYMORPHISM**
- 원문에서 정한 역할: 추상 클래스 여부, 가상 소멸, 복제본 소유권, 공개 헤더, CLI를 검증합니다.
<!-- 원문 분류 요약: Adds abstractness, virtual-destruction, public-header, ownership-counter, and CLI checks. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `bf4d9bed705c`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] counted 테스트 포매터의 live/destroyed counters가 복제 소유권과 base-포인터 deletion을 어떻게 관찰하는지 확인하세요.
- [x] abstract `Formatter`를 직접 instantiate하려는 컴파일 실패 case와 기대 실패 조건을 찾으세요.
- [x] 공개 헤더 repeated inclusion/consumer-visible use를 검사하는 positive 컴파일 case를 확인하세요.
- [x] 파이프라인 CLI 통합 테스트 준비 코드가 실제 실행 파일에서 가상 함수 호출, archive 링크, step order를 어떤 transcript로 검증하는지 기록하세요.
- [x] 이 테스트 bundle이 출력 correctness와 별개로 non-가상 소멸/accidental concreteness/소유권 leak를 각각 어떻게 겨냥하는지 분리하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **polymorphic abstractness, 가상 소멸, 복제 소유권, 공개 consumption**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **accidentally concrete base, non-virtual delete, hidden 소유권 leak, 프로세스 통합 drift**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **컴파일 실패 + 살아 있는 객체 counter + 공개 헤더 컴파일 + CLI 테스트 준비 코드**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **Formatter 복제/delete and 파이프라인 execution paths**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **object-model 규약이 런타임 출력 외에도 유지됨**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **이 커밋 하나만으로 모든 복제 실패 위치를 증명하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **broad 규약 + 통합**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `2c99290b9268`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_formatter.cpp`, `tests/test_format_pipeline.cpp`, `tests/compile/formatter_abstract_fail.cpp`, `tests/compile/format_headers.cpp`, `tests/check_cli.sh`, `Makefile`의 규약/통합 targets.
- 핵심 코드 발췌 위치: `0427713637b8`의 counted 포매터 테스트는 복제·live·destroyed counters를 관찰하고 base 포인터 delete를 실행합니다. 컴파일 실패 번역 단위는 `Formatter` 직접 생성을 시도하며, positive 번역 단위는 공개 headers를 반복 include합니다. CLI 스크립트는 실제 포매터 파이프라인 실행 파일의 transcript를 테스트 준비 코드와 비교합니다.
- 변경 전/후 차이: production 객체 모델은 유지하고, 런타임 출력만으로 드러나지 않던 abstractness, 가상 소멸, 복제 소유권, header isolation, 프로세스 통합을 별도 검증 층으로 추가했습니다.
- 직접 확인한 소유권·수명·상태 관계: counted 포매터의 복제 증가와 파이프라인 scope 종료 후 live count 복구가 복제 소유권을 관찰합니다. base 포인터 삭제 뒤 derived 소멸 counter가 증가해야 virtual 소멸자 규약이 성립합니다.
- 직접 확인한 실패 처리: abstract base가 concrete가 되거나 소멸자가 non-virtual이면 컴파일/런타임 counter 검사가 실패하도록 작성되어 있습니다. CLI 테스트 준비 코드는 step 순서나 archive 링크 drift를 잡지만, 복제 실패 위치를 하나씩 주입하지는 않습니다.
- 실행한 테스트와 결과: 미실행. 실행 대상으로 `make test-contract`, unit 테스트, CLI 통합 대상을 확인했으나 현재 환경에서는 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 포매터 object-model 규약을 컴파일, 소유권 counter, 실제 CLI의 세 층으로 고정했습니다.

### `2c99290b9268` — test(format): 복제 실패 뒤 부분 객체 정리 검증

- 중요도: **A**
- 태그: **TEST, EXCEPTION, POLYMORPHISM**
- 원문에서 정한 역할: 파이프라인 복사 생성과 대입 중 모든 복제 실패 위치를 순회 검증합니다.
<!-- 원문 분류 요약: Sweeps clone failures during pipeline copy construction and assignment. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `0427713637b8`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] 복사 생성과 대입 각각에서 복제 실패를 포매터 위치별로 주입하는 테스트 control을 찾으세요.
- [x] failed 생성자 뒤 이미 생성된 clones가 모두 사라지는지 살아 있는 객체 counter 검사문을 확인하세요.
- [x] failed 대입 뒤 destination의 기존 step 순서와 동작이 유지되는 검사문을 확인하세요.
- [x] 원본 파이프라인이 실패 sweep 전후 동일하게 살아 있는지 확인하는 증거를 기록하세요.
- [x] production 코드에서 생성자 catch 정리와 대입 복사 후 교환 두 경로 중 어느 것을 각 테스트 case가 통과하는지 매핑하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **partial polymorphic copy 정리와 strong 대입 guarantee**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **복제 실패 at each 포매터 위치**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **결정적 복제-실패 sweep + 살아 있는 객체 counters**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **FormatPipeline 복사 생성자 catch 정리 / 대입 복사 후 교환**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **failed 생성 leak freedom과 failed 대입 대상 보존**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **팩터리 creation 실패나 다른 구성 요소 할당 실패는 범위 밖**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 회귀 테스트 / 실패-injection**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/support/TestFormatter.hpp`, `tests/support/TestFormatter.cpp`의 복제 실패 control과 live counters; `tests/failure/test_pipeline_failure.cpp`; `Makefile`의 실패 테스트 대상.
- 핵심 코드 발췌 위치: `2c99290b9268:tests/failure/test_pipeline_failure.cpp`는 원본 파이프라인의 포매터 수를 기준으로 실패 위치를 이동하며 복사 생성과 대입을 각각 시도합니다. `TestFormatter`의 `failCloneOn()`/복제-attempt counter가 지정 복제에서 예외를 발생시킵니다.
- 변경 전/후 차이: 앞선 broad 규약 검증에, 복사 중 각 포매터 위치를 결정적으로 실패시키는 회귀가 추가되었습니다. production 코드는 바뀌지 않습니다.
- 직접 확인한 소유권·수명·상태 관계: failed 복사 생성 뒤 live count가 원본만 남는 baseline으로 돌아오는지 확인합니다. failed 대입 뒤 destination의 기존 step 순서와 적용 결과, 원본 파이프라인의 결과가 모두 그대로인지 확인합니다.
- 직접 확인한 실패 처리: 생성자 case는 `FormatPipeline` 복사 생성자의 catch 정리를, 대입 case는 local 후보 생성 실패로 `swap()`을 건너뛰는 경로를 통과합니다. 이 테스트는 팩터리 creation이나 일반 할당 전체가 아니라 포매터 복제 실패 위치만 다룹니다.
- 실행한 테스트와 결과: 미실행. 실패 주입 구현과 Make 대상은 검사했으나 테스트 실행 파일은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 모든 복제 위치에서 부분 객체 정리와 failed 대입의 destination 보존을 결정적 회귀 테스트로 만들었습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `835d87865762` | virtual 소멸자와 `clone()`으로 polymorphic copy/deletion protocol 정의 | `Formatter`의 pure virtual `clone/apply/name`, virtual 소멸자, 각 derived `new Derived(*this)` 구현으로 동적 copy/delete protocol을 확인했습니다. | 성공한 raw 복제를 즉시 인수할 aggregate/guard와 multi-object 실패 정리는 아직 없습니다. |
| `62ed45f8adf9` | 파이프라인이 borrowed 포매터가 아니라 복제를 소유하도록 소유권 경계 확립 | `steps_[max_steps]`, `size_`, capacity-before-복제, slot store-before-size increment, active-앞부분 소멸으로 파이프라인 소유권을 확인했습니다. | 파이프라인 복사가 비공개여서 aggregate value semantics와 partial-copy 정리는 아직 없습니다. |
| `bf4d9bed705c` | heterogeneous 깊은 복사와 failed-생성자 정리, 복사 후 교환 대입 확립 | 복사 생성자의 null initialization·append loop·catch delete와 대입의 complete copy 후 `swap()`을 확인했습니다. | 실제 복제 실패 위치별 회귀와 공개 abstractness/virtual-delete 증거는 후속 검증이 필요합니다. |
| `0427713637b8` | abstractness/가상 소멸/공개 규약/프로세스 통합 증거 추가 | 컴파일 실패 abstractness, repeated 공개 헤더 컴파일, counted 복제/소멸, CLI 테스트 준비 코드가 object-model과 프로세스 규약을 확인합니다. | 모든 복제 위치에서 생성자/대입 실패를 주입하지는 않습니다. |
| `2c99290b9268` | 모든 포매터 위치의 복제 실패에서 partial 정리와 대상 보존 검증 | 복제 실패 위치 sweep가 failed 생성자의 live baseline과 failed 대입의 destination/원본 동작을 비교합니다. | 팩터리 creation, allocator 전 지점, 다른 구성 요소 실패는 이 테스트 범위 밖입니다. |

## 문제 → 수정 → 검증 연결

- `bf4d9bed705c`: 복제 실패 중 incomplete 복사 생성자가 소멸자에 의존할 수 없는 문제를 explicit 정리로 해결합니다.
- `0427713637b8`: 가상 소멸/abstractness/복제 소유권의 broader 규약 근거를 추가합니다.
- `2c99290b9268`: 복제 실패를 포매터 위치별로 sweep하며 생성자 정리와 대입 transaction을 직접 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: 가상 함수 호출만으로는 heap에 생긴 derived object의 소유자, 복사 방법, base 포인터 소멸, 부분 복사 실패 정리가 정해지지 않습니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: 파이프라인 복사 생성자가 여러 `clone()` 중 하나에서 실패하면 완성되지 않은 객체의 소멸자가 호출되지 않으며, 정상 출력 테스트만으로는 leaked 앞부분나 non-virtual delete를 볼 수 없습니다.
- 수정/강화된 결정: `Formatter`가 virtual 소멸자와 owning `clone()` protocol을 제공하고, 파이프라인은 borrowed prototype을 즉시 복제해 소유합니다. 복사 생성자는 catch에서 성공 앞부분을 직접 정리하고 대입은 완성된 copy만 swap합니다.
- 해당 코드 위치: `835d87865762:include/cppf/Formatter.hpp`, `62ed45f8adf9:src/FormatPipeline.cpp`, `bf4d9bed705c:src/FormatPipeline.cpp`.
- 이를 고정하는 회귀/근거: `0427713637b8`의 컴파일/counter/CLI 테스트와 `2c99290b9268:tests/failure/test_pipeline_failure.cpp`의 복제-위치 sweep.

## 소유권·상태·담당 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: virtual 소멸자와 `clone()`으로 polymorphic copy/deletion protocol 정의
- 개발 흐름 종료 상태: 모든 포매터 위치의 복제 실패에서 partial 정리와 대상 보존 검증
- [x] 중간 커밋마다 소유자/상태 publisher/정리 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned 상태가 함께 등장하면 각각의 수명 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `835d87865762`: concrete 포매터가 자신의 configuration을 소유하고, `clone()` 성공 포인터를 caller에게 넘기는 protocol이 정의됩니다.
2. `62ed45f8adf9`: 파이프라인이 borrowed prototype을 받아 독립 복제를 만들고 `steps_[0..size_)`의 정리 소유자가 됩니다.
3. `bf4d9bed705c`: 파이프라인 복사 중 성공한 복제 앞부분은 incomplete 생성자가 직접 정리하고, 대입 반영은 `swap()` 한 번으로 제한됩니다.
4. `0427713637b8`: abstractness, virtual deletion, 복제 소유권, 공개 headers, CLI execution 경로가 서로 다른 검사로 고정됩니다.
5. `2c99290b9268`: 각 복제 실패에서 생성자 앞부분은 사라지고 대입 destination은 보존되는지 counters와 동작으로 확인합니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `borrowed formatter prototype → virtual clone → owned pipeline slot → deep-copied aggregate → failure cleanup verification`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준으로 caller는 concrete 포매터를 stack이나 다른 소유자에 둔 채 `FormatPipeline::append(const Formatter&)`에 borrowed reference를 전달합니다. `append()`가 virtual `clone()`으로 independent 동적 객체를 만들고 파이프라인 slot이 이를 소유합니다. 파이프라인 copy는 원본의 각 동적 객체를 다시 복제하며, `apply()`는 삽입 order로 가상 함수 호출을 수행합니다.

시작 시점과 비교하면 단일 virtual 인터페이스가 독립 복사 가능한 owning aggregate로 확장되었습니다. 생성 실패와 대입 실패의 정리 책임도 구분됩니다. 다만 raw-포인터 복제 protocol은 caller가 파이프라인 밖에서 직접 사용할 때 즉시 소유권을 인수해야 하며, 후속 테스트가 관찰하지 않은 allocator·팩터리 실패는 이 개발 흐름만으로 증명되지 않습니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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

- 실제 caller → callee 흐름: caller의 concrete `Formatter` → `FormatPipeline::append()` → virtual `clone()` → owned `steps_[size_]` → `apply()`의 ordered 가상 함수 호출 → `TextBuffer` 결과 반환.
- 핵심 상태 필드: `Formatter *steps_[max_steps]`, `std::size_t size_`; 각 derived 포매터의 `TextBuffer prefix_` 또는 `suffix_`.
- 자원 소유자 / 비소유 참조: append 인자와 원본 파이프라인 step 참조는 borrowed이고, 반환된 복제는 성공한 slot 또는 copy-생성 앞부분이 소유합니다.
- 상태 확정 지점: 단일 append는 복제 성공 후 slot 저장과 `++size_`; 대입은 complete local copy를 만든 뒤 `swap(copy)`입니다.
- 정리 path: append 전 복제 실패는 반환 소유자가 없고 파이프라인은 불변입니다. 복사 생성자 실패는 catch가 `[0, size_)` 복제를 직접 삭제하며, 대입 실패는 local 후보가 대상과 swap되기 전에 정리됩니다.
- 최종 불변식 설명: 각 polymorphic 복제는 정확히 한 파이프라인에 소유되고 base 포인터로 안전하게 파괴되며, 깊은 복사는 동적 type과 순서를 유지하고 incomplete aggregate는 publish되지 않습니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 02-polymorphic-cloning-owning-aggregate.md =====

===== BEGIN FILE: 03-factory-transaction-boundary.md =====
# 팩터리 조립에 원자적 교체 지점 만들기

## 개발 흐름의 목표

팩터리가 반환한 raw owning 포인터를 안전하게 넘기는 것과 기존 파이프라인 상태를 원자적으로 교체하는 것이 별개의 문제임을 확인하고, clear-then-빌드에서 후보-then-swap으로 수정되는 과정을 복원합니다.

**원문에서 정한 의의:** 초기 빌더는 자원 누수는 막지만 대상에 부분적으로 재구성된 상태를 노출할 수 있습니다. 수정 후에는 누수 방지와 객체 상태의 원자적 교체를 별개로 다루고, 전체 후보가 완성된 뒤 예외를 던지지 않는 한 번의 `swap()`만 상태 확정 지점으로 사용합니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- 팩터리 specification grammar와 포매터 소유권 transfer의 경계는 어디인가?
- 로컬 RAII guard가 누수를 막아도 대상의 기존 상태는 왜 보존되지 않을 수 있는가?
- 초기 `replace()`의 상태 확정 지점은 사실상 어디에 분산되어 있었는가?
- fix 이후 후보가 실패할 때 대상과 partial clones는 각각 누가 정리하는가?
- 회귀 테스트와 실패 sweep은 각각 어떤 수준의 보장을 증명하는가?

## 완료 기준

- [x] creator → local guard → 파이프라인 복제의 소유권 handoff를 실제 코드로 추적할 수 있습니다.
- [x] fix 전 clear-and-append 경로와 fix 후 후보-and-swap 경로를 관련 SHA끼리 비교할 수 있습니다.
- [x] leak freedom과 strong 상태 보존이 서로 다른 보장임을 실패 처리로 설명할 수 있습니다.
- [x] unknown 포매터, null/count 오류, 복제/할당 실패가 대상에 미치는 영향을 테스트별로 구분할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- 후보는 완성되기 전 대상에 publish되지 않는다.
- 강한 예외 보장이 적용되는 replacement는 creation/복제/할당 실패 시 대상 관찰 가능한 상태를 보존합니다.
- polymorphically owned 자원은 정확히 한 번 해제됩니다.

### 주요 구현 난점

- 팩터리 creation이 raw 포인터를 반환할 때 heterogeneous 소유권 handoff 관리.
- multi-step 팩터리 creation 중 partial 대상 변경 방지.
- 할당/복제 실패 sweep으로 소유권 transition 전체 검증.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `4c34654a4602` | feat(factory): 문자열 명세로 formatter 생성 | A | ARCH, POLYMORPHISM, API | 생성자 추상화와 포매터 명세 문법을 도입합니다. |
| 2 | `fc0b8b7a40a0` | feat(factory): formatter 임시 소유와 pipeline 교체 구현 | A | OWNERSHIP, INTEGRATION, EXCEPTION | 생성자가 반환한 원시 포인터를 즉시 보호 객체가 소유하고 대상 파이프라인을 조립합니다. |
| 3 | `907bfbd5c37c` | fix(factory): 교체 실패에도 기존 파이프라인 보존 | S | DEBUG, EXCEPTION, CORE | 대상을 먼저 비운 뒤 만드는 방식을 후보를 완성한 뒤 교환하는 방식으로 수정합니다. |
| 4 | `466d7abdb60f` | test(factory): 교체 실패 상태 보존과 CLI 검증 | B | TEST, EXCEPTION | 교체가 거부된 뒤 기존 상태 보존과 CLI 동작을 회귀 검증합니다. |
| 5 | `af4e35ca7d92` | test(factory): 생성·복제·할당 실패 정리 검증 | A | TEST, EXCEPTION, OWNERSHIP | 생성·복제·할당 실패 전 구간의 소유권 전달을 순회 검증합니다. |

## 커밋별 학습 기록

### `4c34654a4602` — feat(factory): 문자열 명세로 formatter 생성

- 중요도: **A**
- 태그: **ARCH, POLYMORPHISM, API**
- 원문에서 정한 역할: 생성자 추상화와 포매터 명세 문법을 도입합니다.
<!-- 원문 분류 요약: Adds a polymorphic creator and grammar for constructing formatters from specifications. -->

#### 핵심 설계 / 실패 경계 확인
- [x] exact `upper`, `prefix=<payload>`, `suffix=<payload>` grammar를 분기하는 parser/팩터리 코드를 찾고 malformed와 unknown classification 브랜치를 분리해 기록하세요.
- [x] `FormatterCreator`가 abstract가 되는 선언과 virtual 소멸자를 확인하세요.
- [x] `create()`의 반환 타입에서 raw owning 포인터 transfer가 어떻게 드러나는지 공개 signature를 확인하세요.
- [x] 팩터리가 생성하는 concrete 동적 types와 caller가 base 포인터만 받는 관계를 추적하세요.
- [x] `PipelineBuilder`가 이 시점에는 어떤 작업 경계만 선언하고 있는지 실제 공개 declaration을 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `fc0b8b7a40a0`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/Factory.hpp`의 `InvalidSpecification`, `UnknownFormatter`, `FormatterCreator`, `DefaultFormatterCreator`, `PipelineBuilder`; `src/Factory.cpp`의 `DefaultFormatterCreator::create()`.
- 핵심 코드 발췌 위치: `4c34654a4602:src/Factory.cpp`에서 empty specification은 `InvalidSpecification`, exact `upper`는 `UppercaseFormatter`, non-empty `prefix=`/`suffix=` 전달값은 해당 concrete 포매터, 나머지는 `UnknownFormatter`로 분기됩니다.
- 변경 전/후 차이: 포매터를 코드에서 직접 생성하던 경계에 문자열 specification grammar와 polymorphic creator가 추가되었습니다. `FormatterCreator`는 virtual 소멸자와 pure virtual `create()`를 가지며 `PipelineBuilder`는 정적 replacement 작업을 선언합니다.
- 직접 확인한 소유권·수명·상태 관계: `create()`의 반환형은 `Formatter *`이고 성공 시 heap object 소유권이 caller에게 이전됩니다. caller는 구체 동적 type을 알 필요가 없지만, 반환 직후 delete 책임을 인수해야 합니다.
- 직접 확인한 실패 처리: empty key, 전달값 없는 `prefix=`/`suffix=`, unknown key는 포인터를 반환하기 전에 예외로 끝납니다. 이 SHA에는 여러 specification을 조립하는 구현이나 raw 포인터를 보호하는 local 소유자가 아직 없습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 공개 declaration과 parser/팩터리 구현을 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 포매터 문자열 grammar와 raw-owning polymorphic creation 경계를 도입했습니다.

### `fc0b8b7a40a0` — feat(factory): formatter 임시 소유와 pipeline 교체 구현

- 중요도: **A**
- 태그: **OWNERSHIP, INTEGRATION, EXCEPTION**
- 원문에서 정한 역할: 생성자가 반환한 원시 포인터를 즉시 보호 객체가 소유하고 대상 파이프라인을 조립합니다.
<!-- 원문 분류 요약: Adds RAII ownership of factory results and pipeline replacement from a specification list. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `4c34654a4602`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] creator가 반환한 raw 포인터를 즉시 소유하는 local RAII 소유자의 생성자/소멸자와 scope를 찾으세요.
- [x] local 포매터를 `FormatPipeline::append()`에 넘긴 뒤 파이프라인이 복제를 소유하고 local guard가 원본을 delete하는 두 단계 소유권을 추적하세요.
- [x] null specification array/count consistency와 capacity를 work 시작 전에 검사하는 브랜치를 확인하세요.
- [x] empty specification list가 대상을 empty 파이프라인으로 바꾸는 경로를 확인하세요.
- [x] 가장 중요하게, 대상을 먼저 clear/empty로 만들고 이후 직접 append하는 변경 순서를 찾고 중간 실패 시 observable 대상이 무엇이 되는지 기록하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `907bfbd5c37c`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/Factory.cpp`의 unnamed-namespace `FormatterOwner`, `PipelineBuilder::replace()`; `FormatPipeline::append()`/`swap()`.
- 핵심 코드 발췌 위치: `fc0b8b7a40a0:src/Factory.cpp`의 `FormatterOwner`는 creator raw 포인터를 생성자에서 받아 소멸자에서 `delete`합니다. `replace()`는 validation 후 `target.swap(empty)`로 기존 값을 먼저 비우고, 각 소유자의 포매터를 `target.append()`에 전달합니다.
- 변경 전/후 차이: creator 결과의 leak 방지와 specification list 조립이 구현되었습니다. 그러나 assembly destination이 대상 자체라 replacement의 상태 변경은 시작 시 empty swap과 각 append에 분산됩니다.
- 직접 확인한 소유권·수명·상태 관계: creator 성공 직후 local `FormatterOwner`가 원본 동적 객체를 소유합니다. `target.append(formatter.get())`는 borrowed reference를 받아 별도 복제를 만들고 대상이 복제를 소유합니다. loop iteration 종료 시 소유자는 creator 원본을 삭제합니다.
- 직접 확인한 실패 처리: null/count 불일치와 capacity 초과는 변경 전에 거부됩니다. 이후 create 또는 append/복제가 실패하면 local 소유자와 현재 scope의 objects는 누수 없이 정리되지만, 대상은 이미 비워졌거나 성공한 앞 단계 복제만 가진 partial 파이프라인으로 남습니다. empty specification list는 대상을 empty로 교체합니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 소유권 handoff와 변경 순서를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 팩터리 결과 누수는 막았지만 대상을 증분 변경해 강한 교체 보장은 만들지 못했습니다.

### `907bfbd5c37c` — fix(factory): 교체 실패에도 기존 파이프라인 보존

- 중요도: **S**
- 태그: **DEBUG, EXCEPTION, CORE**
- 원문에서 정한 역할: 대상을 먼저 비운 뒤 만드는 방식을 후보를 완성한 뒤 교환하는 방식으로 수정합니다.
<!-- 원문 분류 요약: Builds a complete candidate pipeline before swapping it into the target. -->

#### 실패 → 수정 → 테스트 연결
- **기존 가정:** 팩터리-created temporaries를 leak 없이 정리하면 replacement도 충분히 안전하다고 볼 수 있었다.
- **실제 실패 / 위험:** later specification의 create/복제 실패가 대상을 이미 비웠거나 partial 파이프라인으로 남길 수 있었다.
- **root cause:** multi-step 작업의 상태 확정 지점이 대상 내부 여러 변경으로 분산되어 있었다.
- **수정된 불변식 / 결정:** 완전한 후보만 one 예외를 던지지 않는 swap으로 대상에 publish합니다.
- **실제 코드 확인:** `fc0b8b7a40a0`과 `907bfbd5c37c`의 `PipelineBuilder::replace()`를 비교해 변경 destination과 final swap을 확인한다.
- **회귀 테스트:** `466d7abdb60f`의 seeded 대상 보존, 이어서 `af4e35ca7d92`의 full 실패 sweep을 확인한다.

#### 이 커밋 직전 상태와 문제
- 직전 관련 개발 흐름 SHA `fc0b8b7a40a0`를 먼저 체크아웃하여 이 커밋이 추가되기 전 표현/소유권/상태-반영 방식을 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 직전 관련 SHA `fc0b8b7a40a0`의 clear-then-빌드 코드와 이 SHA의 후보-then-swap 코드를 직접 diff하세요.
- [x] 새 `FormatPipeline candidate`가 생성되고 모든 create/append가 후보에만 적용되는 호출자/피호출자 흐름을 추적하세요.
- [x] create 또는 복제 exception이 발생하면 후보 소멸자와 local guard가 무엇을 정리하고 대상에는 어떤 write도 하지 않는지 확인하세요.
- [x] 모든 specification 성공 뒤 실행되는 단 하나의 예외를 던지지 않는 `target.swap(candidate)`를 상태 확정 지점으로 표시하세요.
- [x] swap 이후 후보가 기존 대상 상태를 소유하고 scope 종료 시 해제하는 수명 전환을 그리세요.
- [x] fix가 자원 정리가 아니라 object-상태 atomicity를 복구한 것임을 실제 before/after 상태 변경 순서로 설명하세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `466d7abdb60f`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/Factory.cpp`의 `PipelineBuilder::replace()`, `FormatterOwner`; `FormatPipeline::append()`, 소멸자, `swap()`.
- 핵심 코드 발췌 위치: `907bfbd5c37c:src/Factory.cpp`는 `FormatPipeline candidate`를 만들고 모든 `create()`/`candidate.append()`를 끝낸 뒤 마지막에 `target.swap(candidate)`를 한 번 호출합니다.
- 변경 전/후 차이: 직전의 `target.swap(empty)`와 direct append가 제거되고, 변경 destination이 local 후보로 이동했습니다. 자원 정리 방식은 유지되지만 object-상태 반영 지점은 final swap 하나로 축소되었습니다.
- 직접 확인한 소유권·수명·상태 관계: 각 iteration에서 creator object는 `FormatterOwner`, append가 만든 복제는 후보가 소유합니다. final swap 전 대상은 계속 old 파이프라인을 소유합니다. swap 후 대상이 complete 후보를, local 후보가 기존 대상을 소유하며 scope 종료 시 기존 대상 clones를 삭제합니다.
- 직접 확인한 실패 처리: create, 포매터 생성, 복제, 후보 capacity/할당 중 어느 단계에서 throw해도 local 소유자와 후보 소멸자가 새 자원을 정리하고 `target.swap()`에는 도달하지 않습니다. 따라서 대상 size, step order, 출력 동작이 유지됩니다.
- 실행한 테스트와 결과: 미실행. fix 전후 `Factory.cpp`를 직접 비교하고 후속 회귀 원본을 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 완성된 후보만 예외를 던지지 않는 swap으로 게시해 leak freedom과 대상 atomicity를 함께 만족시켰습니다.

### `466d7abdb60f` — test(factory): 교체 실패 상태 보존과 CLI 검증

- 중요도: **B**
- 태그: **TEST, EXCEPTION**
- 원문에서 정한 역할: 교체가 거부된 뒤 기존 상태 보존과 CLI 동작을 회귀 검증합니다.
<!-- 원문 분류 요약: Adds regression and CLI checks for failed replacement preserving the prior pipeline. -->

#### 개발 흐름에서 확인할 구현 역할
- [x] 직전 관련 SHA `907bfbd5c37c`와의 차이 중 이 개발 흐름의 흐름에 필요한 부분만 확인하세요.
- [x] seeded 대상 파이프라인을 준비한 뒤 중간 unknown 포매터를 넣는 회귀 case를 찾으세요.
- [x] invalid null/count combination이 validation 단계에서 대상을 보존하는 case를 확인하세요.
- [x] 각 실패 뒤 transformed 출력 또는 step 동작이 seed와 동일함을 어떤 검사문으로 확인하는지 기록하세요.
- [x] CLI invalid configuration에서 nonzero status, stderr diagnostic, stdout empty를 동시에 검증하는 테스트 준비 코드를 찾으세요.
- [x] 이 테스트가 후보-then-swap의 대표 실패를 고정하지만 모든 할당/복제 site를 sweep하지는 않는다는 범위를 실제 테스트 준비 코드 수로 확인하세요.
- [x] 이 커밋이 다음 관련 커밋의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **PipelineBuilder strong replacement guarantee**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **중간 unknown 포매터와 invalid null/count**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **seeded-상태 회귀 + CLI 테스트 준비 코드**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **builder validation / 후보 assembly / final swap**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **대표 rejection paths에서 prior 대상과 stdout atomicity가 유지됨**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **모든 create/복제/할당 실패 point는 후속 sweep가 담당**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 회귀 테스트 + 통합**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `af4e35ca7d92`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_factory.cpp`, `tests/check_cli.sh`의 팩터리 cases, 관련 테스트 준비 코드와 `Makefile`의 unit/통합 targets.
- 핵심 코드 발췌 위치: `466d7abdb60f:tests/test_factory.cpp`는 기존 파이프라인을 seed한 뒤 list 중간에 unknown 포매터를 두거나 null/count 조합을 잘못 전달하고, exception 후 seed 적용 결과가 동일한지 확인합니다. CLI 스크립트는 invalid configuration의 nonzero status, expected stderr, empty stdout를 비교합니다.
- 변경 전/후 차이: 후보-then-swap production fix 위에 대표적인 grammar/validation rejection과 프로세스-level 출력 atomicity 회귀가 추가되었습니다.
- 직접 확인한 소유권·수명·상태 관계: 회귀는 실패 전 대상 동작을 baseline으로 저장하고 실패 뒤 같은 transformation을 재실행합니다. empty list success는 기존 대상이 local 후보로 이동해 파괴되고 대상이 zero-step 파이프라인이 되는 정상 커밋도 확인합니다.
- 직접 확인한 실패 처리: unknown middle item은 후보에 앞 step 복제가 이미 존재하는 상태에서 예외를 발생시켜 후보 소멸자 경로를 통과합니다. null/count 실패는 work 시작 전 validation 경로입니다. 고정 테스트 준비 코드 수만 확인하므로 모든 할당/복제 site는 sweep하지 않습니다.
- 실행한 테스트와 결과: 미실행. unit/CLI 테스트 준비 코드와 expected 검사문을 검사했지만 실행 파일은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 대표 rejection에서 seeded 대상과 CLI stdout이 보존되는지 고정한 회귀입니다.

### `af4e35ca7d92` — test(factory): 생성·복제·할당 실패 정리 검증

- 중요도: **A**
- 태그: **TEST, EXCEPTION, OWNERSHIP**
- 원문에서 정한 역할: 생성·복제·할당 실패 전 구간의 소유권 전달을 순회 검증합니다.
<!-- 원문 분류 요약: Sweeps creation, clone, and allocation failures while checking cleanup and target preservation. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `466d7abdb60f`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] custom creator와 counted 포매터가 create 실패와 복제 실패를 어떻게 구분해 주입하는지 테스트 doubles를 확인하세요.
- [x] observed 할당/복제 실패 point를 순회하는 sweep loop와 실패 index 제어를 기록하세요.
- [x] creator → local guard → 파이프라인 복제 → partial 후보 소멸자 → final 대상까지 소유권 transition별 live count 검사문을 매핑하세요.
- [x] 모든 injected exception 뒤 original 대상 동작이 유지되는 검사문을 확인하세요.
- [x] 누수뿐 아니라 premature 변경도 검출하도록 어떤 baseline/상태 비교를 함께 수행하는지 기록하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **팩터리-builder 소유권 handoff와 대상 보존**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **creation, 복제, 할당 실패 at every observed point**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **custom creator + counted 포매터 + 결정적 sweep**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **creator → guard → 파이프라인 복제 → 후보 정리 → 대상 swap**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **전체 소유권 transition에서 leak/premature 변경이 없음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **스트림/CLI transport failures는 주 대상이 아님**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 실패-injection 회귀**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/failure/test_factory_failure.cpp`; `tests/support/TestFormatter.hpp/.cpp`; `tests/support/FailingNew.hpp/.cpp`; custom creator 테스트 double; `Makefile`의 팩터리 실패 실행 파일.
- 핵심 코드 발췌 위치: `af4e35ca7d92:tests/failure/test_factory_failure.cpp`는 create 실패, 복제 실패, observed 할당 attempt를 각각 지정해 `PipelineBuilder::replace()`를 반복 호출하고 live counters와 대상 출력을 비교합니다.
- 변경 전/후 차이: 대표 unknown/null rejection에 더해 creator → local guard → 복제 → partial 후보 → final swap 전 구간의 결정적 실패 주입이 추가되었습니다. production 코드는 변경되지 않습니다.
- 직접 확인한 소유권·수명·상태 관계: custom creator가 만든 포매터는 local `FormatterOwner`가 먼저 소유하고, append 성공 시 후보가 별도 복제를 소유합니다. 테스트는 각 실패 뒤 포매터 live count와 할당 현재 할당 블록 수 기준값이 복구되고 original 대상 동작이 동일한지 함께 확인합니다.
- 직접 확인한 실패 처리: create 자체의 throw, returned object를 복제하는 throw, 문자열·컨테이너 등 관찰된 할당 실패가 final swap 전에 발생하도록 실패 index를 이동합니다. 누수만 검사하지 않고 대상 transformation 결과도 비교해 premature 변경을 검출합니다. 스트림 transport 실패는 범위 밖입니다.
- 실행한 테스트와 결과: 미실행. 실패 double과 sweep loop, Make 대상을 검사했으며 테스트 executable은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 팩터리 assembly의 모든 관찰 소유권 handoff 실패에서 정리와 대상 보존을 동시에 검증했습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `4c34654a4602` | creator abstraction, grammar, raw-포인터 소유권 transfer 경계 도입 | `create()`의 exact grammar 분기, abstract creator, virtual 소멸자, raw `Formatter *` 반환을 확인했습니다. | raw 포인터 성공 결과를 인수할 guard와 multi-step 반영은 아직 없습니다. |
| `fc0b8b7a40a0` | 팩터리 result를 즉시 RAII guard가 소유하지만 대상은 incremental 변경 상태 | `FormatterOwner`가 creator 결과를 즉시 삭제 책임으로 감싸고 `append()`가 별도 복제를 대상에 저장합니다. | 대상을 먼저 비우고 직접 append하므로 later 실패에 old 상태가 사라지거나 부분 상태가 노출됩니다. |
| `907bfbd5c37c` | 완성된 후보만 예외를 던지지 않는 `swap()`으로 publish하도록 transaction 경계 수정 | 모든 create/append를 local `candidate`에 적용하고 마지막 `target.swap(candidate)`만 대상을 변경합니다. | 대표/전체 실패 회귀와 프로세스 출력 증거는 후속 커밋이 필요합니다. |
| `466d7abdb60f` | 중간 unknown 포매터와 invalid null/count에서 기존 대상 보존 회귀 | seeded 대상 + unknown/null-count rejection과 CLI nonzero/stderr/empty-stdout 테스트 준비 코드로 대표 경로를 고정합니다. | 모든 create/복제/할당 실패 위치를 순회하지는 않습니다. |
| `af4e35ca7d92` | create/복제/할당 실패 전 지점에서 정리와 대상 보존 검증 | custom creator, counted 복제, `FailingNew` sweep가 live baselines와 대상 동작을 모든 관찰 실패 지점에서 비교합니다. | 스트림/CLI transport 실패와 관찰되지 않은 allocator path는 범위 밖입니다. |

## 문제 → 수정 → 검증 연결

- `fc0b8b7a40a0` 초기 builder: 자원 정리는 수행하지만 대상을 incremental하게 변경합니다.
- `907bfbd5c37c` fix: complete candidate를 만든 뒤 one non-throwing swap으로 publish합니다.
- `466d7abdb60f` 회귀: 대표 rejection에서 seeded 대상과 CLI success 출력이 보존되는지 확인합니다.
- `af4e35ca7d92` 실패 sweep: create/복제/할당 전 지점의 정리와 대상 보존을 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: raw 팩터리 result를 RAII로 정리하는 것만으로는 여러 step을 교체하는 대상의 이전 상태까지 보호되지 않습니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: `fc0b8b7a40a0`은 대상을 먼저 비운 뒤 append하므로 두 번째 이후 specification의 create/복제 실패에서 누수는 없어도 empty 또는 partial 대상을 남깁니다.
- 수정/강화된 결정: validation 후 모든 throw 가능한 creation과 cloning을 local `FormatPipeline candidate`에서 끝내고, 대상에는 예외를 던지지 않는 `swap()` 한 번만 수행합니다.
- 해당 코드 위치: `fc0b8b7a40a0:src/Factory.cpp`의 `target.swap(empty)`/direct append와 `907bfbd5c37c:src/Factory.cpp`의 후보 assembly/final swap.
- 이를 고정하는 회귀/근거: `466d7abdb60f:tests/test_factory.cpp`와 CLI 테스트 준비 코드, `af4e35ca7d92:tests/failure/test_factory_failure.cpp`.

## 소유권·상태·담당 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: creator abstraction, grammar, raw-포인터 소유권 transfer 경계 도입
- 개발 흐름 종료 상태: create/복제/할당 실패 전 지점에서 정리와 대상 보존 검증
- [x] 중간 커밋마다 소유자/상태 publisher/정리 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned 상태가 함께 등장하면 각각의 수명 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `4c34654a4602`: creator가 specification을 concrete 포매터로 변환하고 raw 소유권을 caller에게 넘깁니다.
2. `fc0b8b7a40a0`: local guard가 creator object를 정리하고 파이프라인이 복제를 소유하지만, 상태 publisher가 대상의 clear와 각 append로 분산됩니다.
3. `907bfbd5c37c`: 반영 책임이 final `target.swap(candidate)` 하나로 이동하고, 후보가 모든 partial 자원의 정리 소유자가 됩니다.
4. `466d7abdb60f`: representative rejection과 실제 CLI에서 prior 상태/empty stdout을 확인합니다.
5. `af4e35ca7d92`: create·복제·할당 실패 위치별로 guard, 후보, 대상의 소유권과 상태 baseline을 검사합니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `specification → creator raw pointer → local RAII owner → pipeline clone into candidate → final swap into target`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준으로 `PipelineBuilder::replace()`는 null/count와 capacity를 먼저 검사하고, 각 specification을 `FormatterCreator::create()`로 변환합니다. raw 포인터는 즉시 `FormatterOwner`에 들어가며 `candidate.append()`가 별도 polymorphic 복제를 만듭니다. 모든 step이 성공한 경우에만 `target.swap(candidate)`가 실행됩니다.

초기 팩터리 grammar와 비교하면 자원 소유권 handoff뿐 아니라 multi-step 상태 replacement에 transaction 경계가 생겼습니다. 실패 시 새 포매터와 partial 후보는 local owners가 정리하고 기존 대상은 계속 관찰됩니다. 보장 범위에는 원본 입력이나 외부 스트림 위치 되돌리기가 포함되지 않으며, 테스트가 관찰하지 않은 환경 전체에 대한 형식 증명도 아닙니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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

- 실제 caller → callee 흐름: specification array → `PipelineBuilder::replace()` validation → `FormatterCreator::create()` → local `FormatterOwner` → `FormatPipeline::append()`/virtual `clone()` into 후보 → `target.swap(candidate)`.
- 핵심 상태 필드: 대상/후보의 `Formatter *steps_[max_steps]`, `size_`; local 소유자의 `Formatter *formatter_`.
- 자원 소유자 / 비소유 참조: creator 반환 직후 local guard가 원본을 소유하고 append 인자는 borrowed reference이며 후보가 복제를 소유합니다. 대상은 커밋 전까지 기존 clones를 소유합니다.
- 상태 확정 지점: 모든 specification 처리 성공 뒤의 단일 예외를 던지지 않는 `target.swap(candidate)`입니다.
- 정리 path: create 실패 전에는 포인터가 없고, create 후 실패는 `FormatterOwner`가 원본을 삭제하며 후보 소멸자가 성공한 복제 앞부분을 삭제합니다. swap 후에는 후보가 기존 대상을 파괴합니다.
- 최종 불변식 설명: 팩터리 assembly 중 완성되지 않은 파이프라인은 local 후보 밖으로 노출되지 않고, 모든 관찰 create/복제/할당 실패에서 대상 동작과 소유권 baseline이 유지됩니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 03-factory-transaction-boundary.md =====

===== BEGIN FILE: 04-scalar-text-target-projection.md =====
# 스칼라 텍스트와 출력 대상 표현 분리하기

## 개발 흐름의 목표

입력 text의 문법/의미 판정과 각 대상 type으로의 representability/rendering을 분리하고, locale·negative zero·오버플로·언더플로 경계를 보존한 뒤 완성된 4-line report만 publish하는 흐름을 복원합니다.

**원문에서 정한 의의:** 파싱과 출력을 서로 다른 역할로 분리합니다. 느슨한 숫자 추출이나 시스템 로캘이 입력 의미를 바꾸지 못하게 하고, 네 가지 변환 결과를 부분 출력이 아닌 하나의 결정적 보고서로 반영합니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- 하나의 parser가 원본 literal의 의미를 먼저 정규화해야 하는 이유는 무엇인가?
- classic locale 고정과 complete-입력 검증이 permissive 스트림 parsing의 어떤 문제를 막는가?
- textual zero와 nonzero-언더플로-to-zero를 어떻게 구분해야 하는가?
- negative zero의 sign을 일반 비교와 별개로 보존해야 하는 이유는 무엇인가?
- 임시 객체 스트림 준비 영역이 보장하는 atomicity와 보장하지 않는 destination-스트림 영역은 무엇인가?

## 완료 기준

- [x] character/integer/floating/special 분류가 projection 이전에 끝나는 실제 경로를 추적할 수 있습니다.
- [x] float 뒷부분, whitespace, trailing bytes, 오버플로, nonzero 언더플로 경계를 테스트와 parser 코드에서 대응시킬 수 있습니다.
- [x] float/double projection에서 representability 판단과 canonical rendering을 구분할 수 있습니다.
- [x] caller locale/format flags와 staged 출력의 관계를 테스트로 확인할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- accepted text는 complete ASCII grammar와 일치해야 하며 `LONG_MIN`, negative zero, finite 오버플로, nonzero 언더플로 같은 의미 경계를 보존합니다.
- 완성되지 않은 report는 publish하지 않는다.
- 결정적 rendering은 locale과 caller 스트림 formatting 상태의 영향을 받지 않는다.

### 주요 구현 난점

- locale drift 없이 floating literal을 parsing하면서 negative zero를 보존하고 silent nonzero 언더플로를 거부.
- classic-locale rendering과 caller 스트림-상태 noninterference.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `6a3d0461faab` | feat(scalar): scalar 리터럴 문법과 종류 분류 | A | PARSING, ARCH, NUMERIC | 명시적인 스칼라 문법과 중간 의미 표현을 만듭니다. |
| 2 | `a863f4899a93` | feat(scalar): locale 고정 수치 추출과 경계 보존 | A | NUMERIC, PARSING, HARD | 로캘 독립성, 음수 0, 오버플로, 0이 아닌 언더플로 경계를 보존합니다. |
| 3 | `fc7faa10dc66` | test(scalar): literal 문법과 수치 범위 검증 | A | TEST, NUMERIC, EDGE | 유효·잘못된 문법과 수치 경계 조건을 고정합니다. |
| 4 | `7cdcec341fb1` | feat(scalar): 부동소수점 표현과 원자 출력 구현 | A | NUMERIC, DETERMINISM, EXCEPTION | 표준화된 float/double 변환과 전체 보고서를 임시 저장한 뒤 한 번에 출력하는 기능을 추가합니다. |
| 5 | `afea789fd753` | test(scalar): 변환 가능성·출력·CLI 오류 검증 | A | TEST, NUMERIC, DETERMINISM | 정확한 출력, 스트림 상태 비간섭, 공개 헤더, CLI 실패를 검증합니다. |

## 커밋별 학습 기록

### `6a3d0461faab` — feat(scalar): scalar 리터럴 문법과 종류 분류

- 중요도: **A**
- 태그: **PARSING, ARCH, NUMERIC**
- 원문에서 정한 역할: 명시적인 스칼라 문법과 중간 의미 표현을 만듭니다.
<!-- 원문 분류 요약: Introduces an intermediate scalar-literal model and explicit ASCII grammar. -->

#### 핵심 설계 / 실패 경계 확인
- [x] scalar literal parser가 character/integer/floating/special 종류를 결정하는 grammar branches를 찾으세요.
- [x] ASCII byte 기준 검사와 surrounding whitespace/trailing 재질 rejection을 어떤 helper가 담당하는지 확인하세요.
- [x] lone non-digit character가 character literal로 우선되는 precedence를 실제 브랜치 order에서 확인하세요.
- [x] parser가 즉시 출력하지 않고 normalized intermediate 표현을 만드는 상태 필드/enum을 기록하세요.
- [x] special value와 negative-zero recognition이 여러 projection에 중복되지 않고 parser 단계에 모이는지 call graph로 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `a863f4899a93`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ScalarLiteral.hpp`의 `LiteralKind`, `ScalarLiteral`, `ScalarParseError`, `parseScalarLiteral()`; `src/ScalarLiteral.cpp`의 byte/grammar helpers.
- 핵심 코드 발췌 위치: `6a3d0461faab:src/ScalarLiteral.cpp`에서 입력 byte를 먼저 검사하고 special, lone printable non-digit character, finite grammar 순서로 분류합니다. 결과는 `kind`, `value`, `float_suffix`, `negative_zero`를 가진 intermediate object입니다.
- 변경 전/후 차이: 입력 text를 대상별 출력 함수에서 즉석 해석하는 대신, 원본 literal의 종류와 의미를 먼저 하나의 parser가 결정하는 내부 표현이 생겼습니다.
- 직접 확인한 소유권·수명·상태 관계: parser는 입력 `std::string`을 borrowed read-only 원본으로 사용하고 값만 `ScalarLiteral`에 복사합니다. parser 상태는 호출 범위의 local value이며 아직 destination 스트림에 쓰지 않습니다.
- 직접 확인한 실패 처리: empty text, NUL, non-ASCII, surrounding/embedded whitespace, incomplete number, trailing bytes는 `ScalarParseError`로 끝납니다. lone non-digit character 브랜치가 finite parsing보다 먼저라 printable 단일 문자는 character로 고정됩니다. 이 시점의 numeric extraction 경계는 후속 hardening 전 상태입니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 parser declaration과 구현을 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: complete ASCII scalar grammar를 대상 projection 앞의 normalized semantic value로 분리했습니다.

### `a863f4899a93` — feat(scalar): locale 고정 수치 추출과 경계 보존

- 중요도: **A**
- 태그: **NUMERIC, PARSING, HARD**
- 원문에서 정한 역할: 로캘 독립성, 음수 0, 오버플로, 0이 아닌 언더플로 경계를 보존합니다.
<!-- 원문 분류 요약: Hardens locale-independent numeric extraction, suffix grammar, negative zero, overflow, and underflow handling. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `6a3d0461faab`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] numeric extraction 스트림/locale가 classic locale로 고정되는 지점을 찾고 host locale 사용을 차단하는 방식을 기록하세요.
- [x] `f` 뒷부분을 허용하기 전에 decimal point 또는 exponent 존재를 요구하는 grammar 브랜치를 확인하세요.
- [x] textual zero와 nonzero mantissa가 machine zero로 언더플로한 경우를 구분하는 검사 순서를 추적하세요.
- [x] 오버플로와 silent nonzero 언더플로 rejection이 스트림 extraction success 여부와 별도로 검사되는지 확인하세요.
- [x] negative zero lexeme의 sign이 일반 `value == 0` 비교와 별도 상태로 보존되는 코드를 찾으세요.
- [x] non-ASCII/malformed byte rejection과 printable single-character precedence가 충돌하지 않는 분기 순서를 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `fc7faa10dc66`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ScalarLiteral.cpp`의 `validateFiniteGrammar()`, `allMantissaDigitsAreZero()`, `extractFiniteValue()`, `parseScalarLiteral()`; `src/ScalarLiteral.hpp`의 `negative_zero`.
- 핵심 코드 발췌 위치: `a863f4899a93:src/ScalarLiteral.cpp`는 numeric 스트림에 `std::locale::classic()`을 적용하고 `input.fail() || !input.eof()`를 검사합니다. `f` 뒷부분은 point나 exponent가 있을 때만 허용하며, nonzero mantissa가 extraction 후 `0.0`이면 언더플로로 거부합니다.
- 변경 전/후 차이: permissive numeric extraction에 의존하던 경계를 classic-locale complete parse, finite 오버플로, nonzero-언더플로, 뒷부분 grammar, negative-zero 보존으로 강화했습니다.
- 직접 확인한 소유권·수명·상태 관계: lexeme의 sign과 mantissa-zero 여부는 machine `double`과 별도로 `negative_zero`에 보존됩니다. textual all-zero는 parser가 직접 `+0.0`/`-0.0`을 만들고, nonzero lexeme만 스트림 extraction을 거칩니다.
- 직접 확인한 실패 처리: `42f`는 point/exponent가 없어 거부되고 `1e309` 같은 finite 오버플로는 fail/non-finite 검사로 거부됩니다. `1e-9999`처럼 nonzero digits가 machine zero가 되면 all-zero가 아니므로 거부됩니다. `-0`, `-0.0`, `-0e10`은 zero이면서 sign 메타데이터를 유지합니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 grammar 및 numeric 경계 코드를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: locale와 machine conversion이 원본 text의 오버플로, 언더플로, negative-zero 의미를 바꾸지 못하게 했습니다.

### `fc7faa10dc66` — test(scalar): literal 문법과 수치 범위 검증

- 중요도: **A**
- 태그: **TEST, NUMERIC, EDGE**
- 원문에서 정한 역할: 유효·잘못된 문법과 수치 경계 조건을 고정합니다.
<!-- 원문 분류 요약: Adds exhaustive scalar grammar and numerical-boundary tests. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `a863f4899a93`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] character precedence, signed integer, decimal/exponent, float 뒷부분, negative zero, inf/NaN의 accepted cases를 테스트 table/테스트 준비 코드에서 분류하세요.
- [x] whitespace, embedded NUL/non-ASCII, trailing garbage, malformed exponent, 오버플로, nonzero-언더플로 rejection cases를 각각 production parser 브랜치에 연결하세요.
- [x] complete token 소비를 증명하는 테스트가 스트림 앞부분-parse만 성공하는 잘못된 구현을 어떻게 잡는지 확인하세요.
- [x] literal grammar 실패와 numerical representability 실패가 테스트 expectation에서 구분되는지 기록하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **scalar complete grammar와 numeric 경계 보존**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **malformed tokens, 오버플로, nonzero 언더플로, negative-zero/special boundaries**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **경계-oriented unit suite**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **literal parser and numeric extraction paths**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **accepted/rejected 원본 language 경계가 고정됨**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **최종 4-line rendering/CLI atomicity는 후속 커밋이 담당**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 경계 회귀**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `7cdcec341fb1`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_scalar_literal.cpp`; 테스트 suite registration; literal parser 관련 expected exception helpers.
- 핵심 코드 발췌 위치: `fc7faa10dc66:tests/test_scalar_literal.cpp`의 accepted tables/cases는 character, signed integer, point/exponent, `f` 뒷부분, specials, negative zero를 다루고 rejected cases는 whitespace, NUL/non-ASCII, trailing garbage, malformed exponent, `42f`, 오버플로와 nonzero 언더플로를 포함합니다.
- 변경 전/후 차이: parser 구현의 분기별 원본-language 경계가 결정적 unit 회귀로 고정되었습니다. 최종 4-line rendering과 CLI 출력은 아직 이 커밋의 주 대상이 아닙니다.
- 직접 확인한 소유권·수명·상태 관계: 테스트는 반환 `ScalarLiteral`의 kind/value/뒷부분/sign 메타데이터를 직접 비교하며 destination 스트림이나 외부 상태를 만들지 않습니다.
- 직접 확인한 실패 처리: 앞부분만 읽는 parser라면 통과할 `1.0x`, malformed exponent, embedded NUL을 expected rejection으로 둬 complete consumption을 검사합니다. 오버플로와 nonzero-언더플로도 grammar success와 별개로 exception을 기대해 numerical 경계를 분리합니다.
- 실행한 테스트와 결과: 미실행. 테스트 cases와 production 브랜치 mapping을 검사했으나 unit 실행 파일은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: scalar 원본 language의 승인·거부 경계를 수치 한계까지 회귀 테스트로 고정했습니다.

### `7cdcec341fb1` — feat(scalar): 부동소수점 표현과 원자 출력 구현

- 중요도: **A**
- 태그: **NUMERIC, DETERMINISM, EXCEPTION**
- 원문에서 정한 역할: 표준화된 float/double 변환과 전체 보고서를 임시 저장한 뒤 한 번에 출력하는 기능을 추가합니다.
<!-- 원문 분류 요약: Adds float and double projections plus classic-locale staged rendering. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `fc7faa10dc66`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] float와 double projection의 finite-range 및 nonzero-언더플로 checks를 casting 전에 수행하는 위치를 찾으세요.
- [x] negative zero, NaN, infinity, finite precision, `.0` 뒷부분을 canonical spelling으로 만드는 rendering helpers를 확인하세요.
- [x] caller 스트림이 아니라 classic-locale 임시 객체 스트림에 4개 projection line을 먼저 쓰는 순서를 추적하세요.
- [x] 임시 객체 rendering이 성공한 뒤 caller destination에 한 번에 bytes를 전달하는 반영 point를 찾으세요.
- [x] destination 자체가 final write 중 fail하는 경우까지 되돌리기하지 않는 경계를 실제 write structure에서 확인하세요.
- [x] caller locale/precision/flags를 읽거나 수정하지 않고 result가 고정되는지 구현을 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `afea789fd753`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ScalarConverter.cpp`의 `canProjectChar()`, `canProjectInt()`, `canProjectFloat()`, `finiteNumber()`, projection writers, `ScalarConverter::write()`.
- 핵심 코드 발췌 위치: `7cdcec341fb1:src/ScalarConverter.cpp`에서 float projection은 range를 검사하고 cast 결과가 zero가 되는 nonzero 값을 거부합니다. 네 projection은 classic-locale `std::ostringstream rendered`에 모두 기록된 뒤 `result` bytes가 destination에 한 번 `output.write()` 됩니다.
- 변경 전/후 차이: normalized literal에 char/int/float/double representability와 canonical spelling을 적용하는 출력 계층이 추가되었습니다. caller 스트림에 line을 하나씩 직접 쓰지 않고 report 전체를 먼저 준비 영역합니다.
- 직접 확인한 소유권·수명·상태 관계: parser result와 `rendered`/`result`는 local 후보 상태입니다. caller 스트림은 final write 전까지 변경되지 않습니다. caller의 locale, precision, flags는 읽거나 바꾸지 않고 임시 객체 스트림만 classic locale과 자체 precision을 사용합니다.
- 직접 확인한 실패 처리: parse/projection/rendering 중 예외가 나면 destination write에 도달하지 않아 partial line이 없습니다. float 오버플로나 nonzero-언더플로는 `impossible` projection으로 표현됩니다. 그러나 final `output.write()` 자체가 중간에 실패한 경우 destination bytes나 스트림 위치를 되돌리기하는 코드는 없습니다.
- 실행한 테스트와 결과: 미실행. projection과 staged 반영 구현을 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 원본 의미를 대상별로 투영해 classic-locale 4-line report를 완성한 뒤 한 번에 게시합니다.

### `afea789fd753` — test(scalar): 변환 가능성·출력·CLI 오류 검증

- 중요도: **A**
- 태그: **TEST, NUMERIC, DETERMINISM**
- 원문에서 정한 역할: 정확한 출력, 스트림 상태 비간섭, 공개 헤더, CLI 실패를 검증합니다.
<!-- 원문 분류 요약: Verifies exact projections, locale independence, stream-state preservation, public headers, and CLI errors. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `7cdcec341fb1`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] printable/control character, escapes, integer bounds, finite float/double, NaN/inf, negative zero, one-대상-only representability cases의 exact 4-line expectations를 확인하세요.
- [x] 테스트가 locale와 스트림 formatting 상태를 변경한 뒤에도 canonical 출력과 caller 상태 보존을 어떻게 assert하는지 기록하세요.
- [x] 공개 헤더 컴파일 테스트가 converter를 비공개 parser 없이 사용할 수 있음을 어떻게 확인하는지 보세요.
- [x] CLI invalid literal이 nonzero status와 empty stdout를 보장하는 테스트 준비 코드를 찾으세요.
- [x] unit/컴파일/CLI 각각이 parser, projection, 통합 중 어느 실제 실행 경로를 증명하는지 구분하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **scalar representability, canonical 출력, caller 스트림 noninterference**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **type-specific impossible cases, locale/flags drift, invalid CLI 입력**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **exact-출력 unit + 스트림-상태 manipulation + 공개 컴파일 + CLI 테스트 준비 코드**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **ScalarConverter projection/렌더링 + 프로세스 adapter**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **완성된 scalar 구성 요소의 공개/프로세스 규약**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **destination 스트림 final-write 되돌리기까지 보장하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **broad 통합 + 결정적 회귀 테스트**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_scalar_converter.cpp`, `tests/compile/scalar_headers.cpp`, scalar 비공개-생성 컴파일 실패 case, `tests/check_cli.sh`의 scalar 테스트 준비 코드, `Makefile` targets.
- 핵심 코드 발췌 위치: `afea789fd753:tests/test_scalar_converter.cpp`는 printable/control char, escapes, int bounds, finite/special float/double, negative zero, 대상별 impossible 출력의 정확한 네 줄을 비교합니다. caller 스트림 locale/flags/precision을 변경한 뒤 출력과 기존 상태도 비교합니다.
- 변경 전/후 차이: parser unit 경계 위에 projection correctness, byte-level canonical 출력, 스트림 noninterference, 공개 헤더 visibility, invalid CLI 동작이 추가되었습니다.
- 직접 확인한 소유권·수명·상태 관계: 테스트는 동일 입력의 report가 caller formatting 상태와 무관하고, `ScalarConverter` 내부 비공개 parser를 외부 consumer가 알 필요 없음을 컴파일 규약으로 확인합니다.
- 직접 확인한 실패 처리: invalid literal은 `InvalidScalar`를 기대하고 destination buffer가 empty인지 검사합니다. CLI 테스트 준비 코드는 nonzero status와 empty stdout, diagnostic stderr를 비교합니다. 이 근거도 destination의 final write 실패 되돌리기까지는 다루지 않습니다.
- 실행한 테스트와 결과: 미실행. exact expectations, 컴파일 units, CLI 테스트 준비 코드를 검사했으나 실행 파일/명령은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: scalar 구성 요소의 대상별 표현 가능성, canonical bytes, 공개/프로세스 규약을 검증했습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `6a3d0461faab` | ASCII scalar grammar와 intermediate semantic 표현 도입 | `LiteralKind`/`ScalarLiteral`과 explicit ASCII 브랜치 order로 character/finite/special 의미를 projection 전에 정규화합니다. | numeric extraction의 locale·오버플로·언더플로·뒷부분 세부 경계는 아직 강화 전입니다. |
| `a863f4899a93` | classic locale, 뒷부분 grammar, negative zero, 오버플로/nonzero-언더플로 경계 강화 | classic-locale complete extraction, point/exponent 없는 `f` rejection, all-zero 판정, finite 오버플로 및 nonzero-언더플로 rejection을 확인했습니다. | 최종 대상 representability와 four-line 반영은 아직 없습니다. |
| `fc7faa10dc66` | 문법과 수치 경계를 경계-oriented 테스트로 고정 | accepted/rejected tables가 whitespace/NUL/trailing/malformed/오버플로/언더플로/negative-zero/special 분기를 production parser에 연결합니다. | rendering bytes와 CLI/출력 atomicity는 검증하지 않습니다. |
| `7cdcec341fb1` | float/double projection과 임시 객체-스트림 staged whole-report rendering 도입 | char/int/float/double projection, canonical finite/special spelling, classic 임시 객체 스트림, final single `write()`를 확인했습니다. | destination 스트림의 실제 final-write partial 실패는 되돌리기하지 않습니다. |
| `afea789fd753` | exact 출력, locale independence, 스트림-상태 noninterference, CLI 실패 검증 | exact four-line 출력, locale/flags 보존, 공개 컴파일, invalid CLI empty stdout를 검사합니다. | 모든 possible destination streambuf 실패와 platform floating 구현을 형식적으로 증명하지는 않습니다. |

## 문제 → 수정 → 검증 연결

- 명시적 fix 커밋은 이 개발 흐름에 없습니다. parsing 경계를 `6a3d0461faab`/`a863f4899a93`에서 강화하고, `fc7faa10dc66`에서 grammar/numeric 회귀를 고정합니다.
- `7cdcec341fb1`은 whole-report 준비 영역을 도입하고, `afea789fd753`은 exact 출력/스트림-상태/CLI 실패를 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: 스트림 extraction이 앞부분만 받아들이거나 caller/host locale에 의존하면 동일 text가 다른 의미로 승인될 수 있고, projection을 즉시 출력하면 뒤 단계 실패 전에 partial report가 노출됩니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: 뒷부분·trailing byte·오버플로·nonzero-언더플로·negative zero는 단순 `double` 값만으로 구분되지 않으며, 정상 출력 case만으로 caller formatting 상태 오염이나 invalid 입력의 partial stdout을 잡을 수 없습니다.
- 수정/강화된 결정: parser가 complete ASCII grammar와 semantic 메타데이터를 먼저 만들고, projection은 대상 representability만 판단합니다. 렌더러는 classic-locale 임시 객체 스트림에서 네 줄을 완성한 뒤 destination에 한 번 씁니다.
- 해당 코드 위치: `6a3d0461faab`/`a863f4899a93:src/ScalarLiteral.cpp`, `7cdcec341fb1:src/ScalarConverter.cpp`.
- 이를 고정하는 회귀/근거: `fc7faa10dc66:tests/test_scalar_literal.cpp`, `afea789fd753:tests/test_scalar_converter.cpp`, 컴파일/CLI 테스트 준비 코드.

## Responsibility 변화

- Source 기준 흐름: parser가 원본 literal 의미를 정규화하고, projection이 대상 representability를 판단하며, 렌더러가 classic-locale canonical report를 준비 영역합니다.
- [x] 각 책임이 실제 어떤 class/helper/function에 위치하는지 SHA별로 기록하세요.
- [x] parsing 실패, projection impossibility, formatting 실패의 경계를 구분하세요.

### 코드 검사로 복원한 변화

1. `6a3d0461faab`: parser가 원본 token의 kind와 normalized value를 만들고 destination 출력 책임을 갖지 않습니다.
2. `a863f4899a93`: parser 책임에 classic locale, complete consumption, negative-zero 메타데이터, finite 오버플로와 nonzero-언더플로 판정이 추가됩니다.
3. `fc7faa10dc66`: 원본-language acceptance 경계가 parser-level 테스트로 고정됩니다.
4. `7cdcec341fb1`: converter가 대상별 representability와 canonical rendering을 담당하고 complete report만 destination에 게시합니다.
5. `afea789fd753`: exact bytes, caller 스트림 noninterference, 공개 헤더, 프로세스 실패 동작이 별도 근거로 추가됩니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `ASCII token → literal classification/normalized meaning → per-target representability → canonical render → staged four-line publication`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준으로 `ScalarConverter::write()`는 먼저 `scalar_detail::parseScalarLiteral()`을 호출해 원본 text를 `ScalarLiteral`로 정규화합니다. 이후 char/int/float/double writer가 동일 semantic value를 각 대상 범위에 맞춰 `impossible`, `Non displayable`, canonical finite/special text로 변환합니다. 네 줄은 local classic-locale 스트림에서 완성된 후 destination에 전달됩니다.

시작 시점과 비교하면 text parsing과 대상 projection이 분리되어 locale, trailing 입력, negative zero, 오버플로와 언더플로 경계를 잃지 않습니다. invalid 입력이나 local rendering 실패는 destination에 아무 앞부분도 쓰지 않습니다. 남는 경계는 destination 스트림의 final write 실패 되돌리기와 테스트 matrix 밖 floating/platform 차이입니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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

- 실제 caller → callee 흐름: 입력 `std::string` → `ScalarConverter::write()` → `parseScalarLiteral()` → `ScalarLiteral` → char/int/float/double projection helpers → local `ostringstream rendered` → destination `output.write()`.
- 핵심 상태 필드: `ScalarLiteral::{kind, value, float_suffix, negative_zero}`와 local rendered byte string.
- 자원 소유자 / 비소유 참조: 입력 text와 destination 스트림은 caller-owned borrowed objects이고 parser result, 임시 객체 스트림, result string은 call-local owners입니다.
- 상태 확정 지점: 네 projection line을 모두 성공적으로 만든 뒤 실행하는 final `output.write(result.data(), result.size())`입니다.
- 정리 path: grammar/numeric 실패는 `ScalarParseError`에서 공개 `InvalidScalar`로 변환되고 local candidates가 자동 파괴됩니다. projection/rendering 실패도 final write 전이면 destination은 untouched입니다.
- 최종 불변식 설명: accepted text는 complete classic ASCII grammar와 semantic sign/range를 보존하고, 대상별 판단과 canonical rendering은 caller 스트림 상태와 무관하며 incomplete report는 publish되지 않습니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 04-scalar-text-target-projection.md =====

===== BEGIN FILE: 05-checked-rpn-undefined-arithmetic.md =====
# 검사된 RPN 계산으로 정의되지 않은 산술 피하기

## 개발 흐름의 목표

signed `long` 연산을 실행한 뒤 오버플로를 검사하는 잘못된 접근을 피하고, token parser와 stack grammar 위에서 모든 산술 precondition을 먼저 검증하는 구현을 복원합니다.

**원문에서 정한 의의:** 부호 있는 정수 오버플로는 연산 후에 감지하면 이미 정의되지 않은 동작이 발생했을 수 있습니다. 계산기는 `LONG_MIN`의 비대칭 범위까지 고려해 실제 연산 전에 한계와 절댓값을 검사합니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- `LONG_MIN`이 양수 최대값보다 magnitude가 1 큰 사실이 parsing과 multiplication에 어떤 영향을 주는가?
- operand pop 순서가 subtraction/division 의미에 어떻게 반영되는가?
- 각 연산에서 오버플로 여부를 실제 연산 전에 어떻게 판정하는가?
- division의 두 특수 실패 조건은 무엇이며 왜 별도 검사가 필요한가?
- malformed token과 stack-도형 오류가 arithmetic helper까지 도달하지 않도록 어떤 계층이 막는가?

## 완료 기준

- [x] signed decimal token accumulation이 `LONG_MIN`까지 안전하게 도달하는 코드를 설명할 수 있습니다.
- [x] +, -, *, / 각각의 precondition check와 실제 signed 작업의 순서를 실제 코드로 증명할 수 있습니다.
- [x] right-then-left pop과 non-commutative result를 테스트 케이스로 연결할 수 있습니다.
- [x] 오버플로/언더플로/division-by-zero/malformed stack의 회귀 coverage를 구분할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- signed arithmetic은 실행 전에 검사되어 error detection 자체가 undefined 오버플로에 의존하지 않는다.
- accepted integer token은 complete ASCII grammar와 `LONG_MIN`/`LONG_MAX` 경계를 보존합니다.

### 주요 구현 난점

- overflowing expression을 먼저 평가하지 않고 모든 signed `long` arithmetic 검사.
- `LONG_MIN`의 비대칭 magnitude를 고려한 multiplication/division 처리.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `57a25e8475ab` | feat(rpn): signed token과 stack 문법 처리 | A | PARSING, NUMERIC, CORE | 부호 있는 10진수 피연산자 파싱과 스택 형태 규칙을 확립합니다. |
| 2 | `e1641a714172` | feat(rpn): overflow 검사 산술 연산 구현 | S | NUMERIC, HARD, CORE | 모든 부호 있는 연산자에 실행 전 조건 검사를 추가합니다. |
| 3 | `aa0cc5e3e063` | test(rpn): 산술 경계와 잘못된 token 검증 | A | TEST, NUMERIC, EDGE | 리터럴 한계, 오버플로 방향, 피연산자 순서, 잘못된 식을 검증합니다. |

## 커밋별 학습 기록

### `57a25e8475ab` — feat(rpn): signed token과 stack 문법 처리

- 중요도: **A**
- 태그: **PARSING, NUMERIC, CORE**
- 원문에서 정한 역할: 부호 있는 10진수 피연산자 파싱과 스택 형태 규칙을 확립합니다.
<!-- 원문 분류 요약: Introduces signed decimal token parsing and the structural RPN stack language. -->

#### 핵심 설계 / 실패 경계 확인
- [x] ASCII space 규칙에 따른 token separation과 complete signed-decimal recognition을 담당하는 코드를 찾으세요.
- [x] magnitude accumulation이 오버플로 없이 `LONG_MAX`와 `LONG_MIN`의 asymmetric magnitude를 모두 허용하는 계산을 추적하세요.
- [x] malformed number/unknown token을 operator 단계 전에 거부하는 브랜치를 확인하세요.
- [x] evaluation stack push/pop과 expression 종료 시 exactly one result를 요구하는 구조적 validation을 기록하세요.
- [x] locale-sensitive 스트림 앞부분 parsing을 피하기 위해 manual parser가 사용되는 지점을 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `e1641a714172`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/RpnEvaluator.hpp`의 `RpnEvaluator::evaluate()`; `src/RpnEvaluator.cpp`의 `parseLong()`, token loop, `isOperator()`, local stack.
- 핵심 코드 발췌 위치: `57a25e8475ab:src/RpnEvaluator.cpp`의 `parseLong()`은 sign을 분리하고 `unsigned long magnitude`를 `(limit - digit) / 10`과 비교한 뒤 누적합니다. 음수 limit은 `LONG_MAX + 1`로 두어 `LONG_MIN`을 별도 브랜치에서 만듭니다.
- 변경 전/후 차이: locale-sensitive 스트림 extraction 대신 ASCII space tokenization과 complete signed-decimal parser가 도입되었고, evaluator가 local `std::vector<long>` stack의 구조를 직접 검증하게 되었습니다.
- 직접 확인한 소유권·수명·상태 관계: expression은 caller-owned borrowed string이고 token substring과 evaluation stack은 call-local 상태입니다. operand token은 완전히 파싱된 후에만 stack에 push되며 결과는 종료 시 stack에 정확히 하나 남을 때만 반환됩니다.
- 직접 확인한 실패 처리: sign만 있는 token, unknown byte가 섞인 number, unknown operator/token, operand 부족, 종료 시 0개 또는 2개 이상 결과는 `invalid_argument` 또는 range exception으로 끝납니다. local stack은 외부 객체에 publish되지 않습니다. 이 SHA의 산술 operator에는 아직 모든 오버플로 precondition이 추가되기 전입니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 tokenizer/parser/stack 코드를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: `LONG_MIN`까지 안전하게 만드는 signed token parser와 RPN stack language를 확립했습니다.

### `e1641a714172` — feat(rpn): overflow 검사 산술 연산 구현

- 중요도: **S**
- 태그: **NUMERIC, HARD, CORE**
- 원문에서 정한 역할: 모든 부호 있는 연산자에 실행 전 조건 검사를 추가합니다.
<!-- 원문 분류 요약: Implements checked addition, subtraction, multiplication, and division before signed operations execute. -->

#### 이 커밋 직전 상태와 문제
- 직전 관련 개발 흐름 SHA `57a25e8475ab`를 먼저 체크아웃하여 이 커밋이 추가되기 전 표현/소유권/상태-반영 방식을 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] addition/subtraction이 operand sign에 따라 `LONG_MIN`/`LONG_MAX` margin을 비교하는 helper/브랜치를 찾으세요.
- [x] multiplication이 signed multiplication 자체를 실행하기 전에 sign과 unsigned magnitude로 범위를 판단하는 과정을 단계별로 기록하세요.
- [x] division의 zero divisor와 `LONG_MIN / -1`을 실제 division 전에 차단하는 브랜치를 확인하세요.
- [x] operator 적용 시 stack에서 right operand를 먼저, left operand를 나중에 pop하는 코드를 확인하세요.
- [x] 각 helper에서 범위 검사가 통과한 뒤에만 signed 작업 expression이 평가됨을 실제 control flow로 증명하세요.
- [x] 오버플로/invalid 작업 exception이 local evaluation stack 밖에 partial result를 publish하지 않는 이유를 call scope로 설명하세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `aa0cc5e3e063`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/RpnEvaluator.cpp`의 `magnitudeOf()`, `checkedAdd()`, `checkedSubtract()`, `checkedMultiply()`, `checkedDivide()`, `applyOperator()`, `evaluate()`.
- 핵심 코드 발췌 위치: `e1641a714172:src/RpnEvaluator.cpp`에서 add/subtract는 sign별 limit 식을 먼저 검사합니다. multiply는 `-(value + 1) + 1` 형태의 unsigned magnitude로 `LONG_MIN`을 처리하고 `left_magnitude > limit / right_magnitude`를 실제 곱셈 전에 검사합니다.
- 변경 전/후 차이: 직전 parser/stack 구현 위에 모든 signed operator의 precondition-first arithmetic이 추가되었습니다. 결과를 계산한 뒤 오버플로를 판정하는 방식은 사용하지 않습니다.
- 직접 확인한 소유권·수명·상태 관계: operator token 처리 시 stack에서 `right`를 먼저, `left`를 나중에 꺼내 `applyOperator(left, right, op)`에 전달합니다. checked helper가 성공한 값만 다시 local stack에 push하므로 실패 결과는 외부나 stack에 게시되지 않습니다.
- 직접 확인한 실패 처리: addition/subtraction은 limit subtraction/addition으로 margin을 검사하고, multiplication은 sign에 따라 `LONG_MAX` 또는 `LONG_MAX + 1` magnitude limit을 사용합니다. division은 `right == 0`과 `LONG_MIN / -1`을 실제 `/` 전에 거부합니다. 모든 signed `+ - * /` expression은 해당 검사 뒤에만 평가됩니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 checked helper와 call order를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: signed arithmetic을 실행하기 전에 모든 오버플로와 invalid division 조건을 판정하도록 만들었습니다.

### `aa0cc5e3e063` — test(rpn): 산술 경계와 잘못된 token 검증

- 중요도: **A**
- 태그: **TEST, NUMERIC, EDGE**
- 원문에서 정한 역할: 리터럴 한계, 오버플로 방향, 피연산자 순서, 잘못된 식을 검증합니다.
<!-- 원문 분류 요약: Covers RPN syntax, operand order, all arithmetic boundaries, division by zero, and malformed stacks. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `e1641a714172`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] 정상 +,-,*,/와 subtraction/division operand order를 구분하는 테스트 cases를 찾으세요.
- [x] `LONG_MIN`/`LONG_MAX` literal parsing과 모든 오버플로/언더플로 direction을 각각 어떤 expression으로 재현하는지 기록하세요.
- [x] division by zero와 `LONG_MIN / -1` case가 별도 회귀로 존재하는지 확인하세요.
- [x] malformed number, unknown token, insufficient/extra operands, spacing boundaries가 parser/stack grammar의 어느 브랜치를 통과하는지 매핑하세요.
- [x] 테스트가 UB 발생 뒤 결과를 검사하는 것이 아니라 UB expression 자체가 실행되지 않도록 error path를 관찰하는 방식을 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **RPN grammar와 precondition-first checked arithmetic**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **all 오버플로/언더플로 directions, division by zero, malformed stack/tokens**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **경계 unit suite**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **token parser, stack evaluator, checked operator helpers**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **exercised signed operations이 UB 경계를 넘기 전에 거부됨**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **모든 가능한 긴 expression/상태-space를 exhaustive하게 증명하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 경계 회귀**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_rpn_evaluator.cpp`, `tests/compile/rpn_headers.cpp`, RPN 비공개-생성 컴파일 실패 unit, 테스트 registration/Make 대상.
- 핵심 코드 발췌 위치: `aa0cc5e3e063:tests/test_rpn_evaluator.cpp`는 normal operators와 `8 3 -`, `8 3 /` 같은 operand order, `LONG_MIN`/`LONG_MAX` literals, add/subtract/multiply 양방향 오버플로, division by zero, `LONG_MIN -1 /`, malformed stack/token을 구분합니다.
- 변경 전/후 차이: checked-arithmetic 구현의 각 브랜치와 parser/stack 도형이 결정적 경계 unit suite로 고정되었습니다. production 코드는 변경되지 않습니다.
- 직접 확인한 소유권·수명·상태 관계: success case는 반환 `long`만 비교하고 실패 case는 exception을 기대합니다. evaluator 상태는 매 호출마다 local stack이므로 실패 뒤 영속 대상이나 partial result를 검사할 외부 객체는 없습니다.
- 직접 확인한 실패 처리: 오버플로 expression은 결과 값을 관찰하지 않고 exception path를 기대하므로 checked helper가 실제 undefined expression을 실행하지 않아야 테스트가 sanitizer/정상 실행에서도 끝납니다. tab/newline, malformed sign, extra/insufficient operands도 parser 또는 final-size 브랜치에 연결됩니다. 모든 가능한 긴 expression을 exhaustive하게 다루지는 않습니다.
- 실행한 테스트와 결과: 미실행. 경계 cases와 공개 컴파일 units를 검사했으나 unit 실행 파일은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: token limit, operand order, 모든 산술 방향과 malformed stack을 UB 전 거부 계약으로 고정했습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `57a25e8475ab` | signed decimal token grammar, exact limits, stack-도형 language 도입 | manual sign/magnitude parser가 `LONG_MAX + 1` negative limit과 pre-multiply digit guard로 exact signed bounds를 만들고 local stack 도형을 검사합니다. | operator 오버플로 precondition은 아직 완전하지 않아 arithmetic safety는 후속 커밋에 남습니다. |
| `e1641a714172` | 모든 operator에 precondition-first 오버플로/invalid-작업 checks 도입 | add/sub sign margins, unsigned magnitude multiplication limit, zero 및 `LONG_MIN / -1` division guards가 실제 작업보다 먼저 실행됩니다. | 모든 긴 expression/상태-space에 대한 exhaustive proof와 외부 caller side effect는 다루지 않습니다. |
| `aa0cc5e3e063` | long extremes, 오버플로 directions, operand order, malformed expressions 검증 | literal extremes, non-commutative order, 오버플로/언더플로 directions, invalid division, malformed tokens/stacks를 결정적 cases로 검사합니다. | finite case set이므로 모든 가능한 token 길이와 expression 조합을 형식적으로 증명하지는 않습니다. |

## 문제 → 수정 → 검증 연결

- 명시적 fix 커밋은 이 개발 흐름에 없습니다. `e1641a714172`가 undefined signed 오버플로를 피하는 precondition-first 결정을 구현합니다.
- `aa0cc5e3e063`이 long limits, 모든 오버플로 direction, division 경계, malformed expression을 회귀 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: signed 오버플로를 먼저 계산한 뒤 결과 범위를 검사하면 검사 자체가 이미 undefined 동작 뒤에 실행됩니다. 또한 `LONG_MIN`은 양의 `long`으로 직접 magnitude를 표현할 수 없습니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: 일반적인 `-value`나 `left * right` 기반 검사는 `LONG_MIN`과 오버플로 product에서 안전하지 않고, commutative operator만 시험하면 right/left pop 순서 오류도 놓칩니다.
- 수정/강화된 결정: token은 unsigned magnitude와 asymmetric limit으로 파싱하고, 각 operator는 sign·limit·magnitude precondition을 통과한 뒤에만 signed expression을 실행합니다.
- 해당 코드 위치: `57a25e8475ab:src/RpnEvaluator.cpp`의 `parseLong()`, `e1641a714172:src/RpnEvaluator.cpp`의 checked helpers와 `evaluate()` pop order.
- 이를 고정하는 회귀/근거: `aa0cc5e3e063:tests/test_rpn_evaluator.cpp`의 long extremes, 모든 오버플로 방향, division special cases, malformed expression 테스트.

## State / responsibility 변화

- Source 기준 흐름: token parser가 complete signed operands를 만들고, evaluator stack이 구조를 소유하며, arithmetic helper가 signed 작업 이전의 range precondition을 책임집니다.
- [x] parser stack과 arithmetic helper 사이에서 어떤 값이 전달되고 어디서 exception이 발생하는지 기록하세요.

### 코드 검사로 복원한 변화

1. `57a25e8475ab`: ASCII-space tokenizer와 unsigned magnitude parser가 complete signed operands를 만들고 local stack이 expression structure를 소유합니다.
2. `e1641a714172`: arithmetic responsibility가 checked helper로 분리되고, stack에는 precondition을 통과한 결과만 다시 들어갑니다.
3. `aa0cc5e3e063`: parser limits, pop order, 각 operator의 success/실패 경계가 결정적 unit cases로 연결됩니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `ASCII-space tokenization → signed operand parse → stack-shape validation → checked operator precondition → signed operation → single-result validation`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준으로 `RpnEvaluator::evaluate()`는 ASCII space만 separator로 사용해 token을 완전히 분리합니다. signed number는 `parseLong()`이 unsigned magnitude로 범위를 확인하고, operator는 stack에서 right와 left를 꺼낸 뒤 checked helper에 전달합니다. helper가 성공한 경우에만 result를 push하며 종료 시 exactly one value를 요구합니다.

시작 시점과 비교하면 token grammar와 stack grammar 위에 UB를 발생시키지 않는 arithmetic 경계가 생겼습니다. 실패는 call-local 상태에서 exception으로 끝나며 partial result를 외부에 게시하지 않습니다. 남는 경계는 유한한 회귀 테스트 모음 밖 expression 조합과 caller가 exception 이후 수행하는 외부 side effect입니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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

- 실제 caller → callee 흐름: expression string → `RpnEvaluator::evaluate()` token loop → `parseLong()` 또는 operator 브랜치 → right/left pop → `checkedAdd/Subtract/Multiply/Divide()` → validated result push → single final result 반환.
- 핵심 상태 필드: call-local `std::vector<long> stack`, token index, unsigned `magnitude`/`limit`, checked helper의 left/right values.
- 자원 소유자 / 비소유 참조: expression은 borrowed 입력이고 token strings와 stack은 evaluator call이 소유합니다. 영속 heap 소유권이나 external 대상은 없습니다.
- 상태 확정 지점: 각 operator는 checked helper 성공 뒤 `stack.push_back(result)`하고, 전체 evaluation은 final stack size가 1일 때 반환합니다.
- 정리 path: malformed token/stack, range 실패, zero division, `LONG_MIN / -1`은 exception으로 local stack을 파괴합니다. overflowed signed expression은 실행되지 않습니다.
- 최종 불변식 설명: accepted operands는 exact `long` grammar와 limits를 만족하고, 모든 signed 작업은 정의된 결과 범위가 확인된 뒤에만 실행되며 non-commutative operand order가 보존됩니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 05-checked-rpn-undefined-arithmetic.md =====

===== BEGIN FILE: 06-generic-containers-transactional-batch.md =====
# 제네릭 컨테이너를 일괄 처리 트랜잭션으로 결합하기

## 개발 흐름의 목표

random-access template abstraction을 vector/deque 두 표현에 적용하고, record parsing·중복 검사·RPN 계산·정렬·스트림 완료 판정까지 하나의 delayed-반영 transaction으로 통합하는 과정을 복원합니다.

**원문에서 정한 의의:** 제네릭 컨테이너, 검사된 산술, 결정적 정렬, 지역 후보 상태, 지연된 반영을 마지막 하위 시스템에서 결합합니다. 후속 커밋은 스트림 입력이 성공적으로 끝났다는 의미를 구체화하고, 어떤 실패에서도 부분 일괄 결과가 반영되지 않음을 검증합니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- `std::sort` 사용이 template의 random-access requirement를 어떻게 실제 컴파일 requirement로 만드는가?
- throwing element type에서도 batch copy/대입이 어떤 보장을 유지해야 하는가?
- `BatchEngine::replace()`의 되돌리기 value는 무엇이며 왜 보상 코드가 필요 없는가?
- canonical `(value, name)` total order가 입력 순열 독립성과 어떤 관계가 있는가?
- vector/deque 결과 불일치를 publish 전에 검사하는 이유는 무엇인가?
- 마지막 newline이 없는 정상 EOF와 실제 스트림 실패를 어떻게 구분하는가?
- syntax/arithmetic/스트림/할당 실패 sweep이 하나의 transaction 불변식을 어떻게 검증하는가?

## 완료 기준

- [x] template requirement와 vector/deque substitution을 공개 헤더와 테스트에서 확인할 수 있습니다.
- [x] batch replace의 후보 상태, duplicate tracking, RPN result accumulation, final swap 위치를 그릴 수 있습니다.
- [x] sorting/serialization이 결정적 external 동작을 만드는 근거를 실제 comparator와 준비 영역 코드에서 확인할 수 있습니다.
- [x] 스트림 reader fix 전후에서 clean EOF/final unterminated line/실패의 분기를 비교할 수 있습니다.
- [x] seeded prior 상태가 모든 rejection path 뒤 유지되는 회귀/실패-sweep 구조를 설명할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- 완성되지 않은 batch 후보는 publish되지 않는다.
- 강한 예외 보장 replacement는 parse/arithmetic/스트림-read preparation/할당 실패 시 prior 관찰 가능한 상태를 보존합니다.
- batch 출력은 total `(value, name)` order를 가지며 입력 permutation과 repeated rendering에 불변입니다.
- signed arithmetic은 실행 전에 검사됩니다.

### 주요 구현 난점

- vector/deque generic 동작과 결정적 total order, transactional 반영을 동시에 유지.
- whole-스트림 batch replacement 중 partial 대상 변경 방지.
- clean final unterminated line과 actual 스트림 실패 구분.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `708c025ef2a0` | feat(template): 임의 접근 container batch 추상화 추가 | S | ARCH, GENERIC, CORE | 설정 가능한 임의 접근 일괄 처리 형식과 서로 다른 컨테이너 간 범위 비교를 정의합니다. |
| 2 | `aaeff163baf8` | test(template): iterator·정렬·복사 실패 계약 검증 | A | TEST, GENERIC, EXCEPTION | 반복자·알고리즘·컨테이너 교체와 예외를 던지는 값 형식의 동작을 검증합니다. |
| 3 | `d0295f82614b` | feat(batch): 입력 문법과 원자 교체 구현 | S | ARCH, EXCEPTION, INTEGRATION | 레코드 파싱, 중복 검사, RPN 계산, 성공 시 교환을 한 처리 과정으로 결합합니다. |
| 4 | `42d411e42268` | feat(batch): 결과 정렬과 직렬화 제공 | A | DETERMINISM, EXCEPTION, CORE | 전체 결과 정렬 규칙과 고정 로캘을 사용하는 단계별 직렬화를 추가합니다. |
| 5 | `af57a8f9c5fe` | feat(batch): 두 container의 정렬 결과 대조 | A | GENERIC, INTEGRATION, DETERMINISM | vector와 deque 후보를 독립적으로 정렬하고 결과가 다르면 반영 전에 거부합니다. |
| 6 | `9ba0e7c897ed` | test(batch): 입력 순열과 출력 결정성 검증 | A | TEST, DETERMINISM, EDGE | 입력 순열 불변성과 반복 실행의 바이트 단위 동일 출력을 검증합니다. |
| 7 | `ea23237ad506` | fix(batch): 입력 stream 종료 상태를 명확히 구분 | A | DEBUG, PARSING, EDGE | 정상 EOF, 줄바꿈 없는 마지막 레코드, 실제 입력 실패를 명확히 구분합니다. |
| 8 | `b4ddd78fb9aa` | test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증 | A | TEST, EXCEPTION, RISK | 문법·산술·스트림·할당 실패 전 범위에서 기존 상태 보존을 검증합니다. |

## 커밋별 학습 기록

### `708c025ef2a0` — feat(template): 임의 접근 container batch 추상화 추가

- 중요도: **S**
- 태그: **ARCH, GENERIC, CORE**
- 원문에서 정한 역할: 설정 가능한 임의 접근 일괄 처리 형식과 서로 다른 컨테이너 간 범위 비교를 정의합니다.
<!-- 원문 분류 요약: Introduces `RandomAccessBatch` over configurable random-access containers and cross-container range equality. -->

#### 이 커밋 직전 상태와 문제
- 이 개발 흐름의 첫 커밋이므로, `git show <sha>^`가 가능한 경우 parent에서 관련 type/기능이 없거나 다른 형태였는지 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] `RandomAccessBatch<T, Container>` 공개 헤더에서 value/컨테이너 template parameters와 default `std::vector`를 확인하세요.
- [x] 컨테이너의 반복자/const_iterator category를 그대로 expose하는 typedef와 begin/end API를 찾으세요.
- [x] `at()`, 삽입, range iteration, `std::sort` 기반 sort, range equality가 어떤 컨테이너 capability를 요구하는지 기록하세요.
- [x] `std::list` 같은 non-random-access 컨테이너가 왜 sort instantiation 단계에서 맞지 않는지 실제 알고리즘 requirement와 연결하세요.
- [x] 대입의 복사 후 교환 구현과 underlying 컨테이너 copy 실패 시 대상 보존 경로를 확인하세요.
- [x] vector와 deque instantiation이 같은 abstraction을 통해 동작하도록 구현이 concrete 표현에 의존하지 않는 지점을 찾으세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `aaeff163baf8`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/RandomAccessBatch.hpp`의 `RandomAccessBatch<T, Container>`, 반복자 typedefs, `values_`, `push_back()`, `at()`, `sort()`, `swap()`, `equal_ranges()`.
- 핵심 코드 발췌 위치: `708c025ef2a0:include/cppf/RandomAccessBatch.hpp`는 `Container = std::vector<T>`를 기본값으로 두고 컨테이너의 반복자/const_iterator를 그대로 노출합니다. `sort()`는 `std::sort(values_.begin(), values_.end(), compare)`를 호출하고 대입은 complete copy 뒤 `swap()`합니다.
- 변경 전/후 차이: concrete vector 사용 대신 random-access operations를 만족하는 컨테이너 parameter를 교체할 수 있는 header-only batch abstraction과 서로 다른 range를 비교하는 helper가 생겼습니다.
- 직접 확인한 소유권·수명·상태 관계: `RandomAccessBatch`가 `Container values_`를 값으로 소유하며 반복자는 그 컨테이너 수명과 변경 규칙에 종속된 비소유 참조입니다. copy는 underlying 컨테이너의 독립 value copy이고 대입 커밋은 컨테이너 swap입니다.
- 직접 확인한 실패 처리: element/컨테이너 copy가 실패하면 복사 생성자는 underlying 컨테이너가 partial elements를 정리하고, 대입은 local 복사 생성 단계에서 대상 `values_`를 건드리지 않습니다. `at()`은 범위 밖을 거부합니다. `std::list`는 template 선언 자체가 아니라 `std::sort` instantiation의 random-access requirement에서 제외됩니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 공개 template 구현을 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: vector/deque로 치환 가능한 random-access batch와 복사 후 교환 value semantics를 정의했습니다.

### `aaeff163baf8` — test(template): iterator·정렬·복사 실패 계약 검증

- 중요도: **A**
- 태그: **TEST, GENERIC, EXCEPTION**
- 원문에서 정한 역할: 반복자·알고리즘·컨테이너 교체와 예외를 던지는 값 형식의 동작을 검증합니다.
<!-- 원문 분류 요약: Tests iterators, algorithms, vector/deque substitution, copying, and throwing element types. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `708c025ef2a0`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] vector/deque 각각에서 mutable/const 반복자와 standard 알고리즘을 사용하는 테스트를 구분하세요.
- [x] checked access, sorting, range equality, copy, 대입, 자기 대입의 expected 동작을 확인하세요.
- [x] throwing value type의 copy 실패 주입과 살아 있는 객체 counter가 어떻게 구성되는지 찾으세요.
- [x] failed 생성에서 partial copied values leak가 없는지, failed 대입에서 destination 보존이 되는지 각각의 검사문을 기록하세요.
- [x] generic 인터페이스 도형 테스트와 exception guarantee 테스트가 서로 어떤 production template instantiation을 사용하는지 구분하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **RandomAccessBatch generic 인터페이스와 value/exception semantics**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **vector/deque substitution, throwing element copy**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **multi-컨테이너 unit + throwing 테스트 type/live counters**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **template instantiation, 컨테이너 copy/sort/대입 paths**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **generic capability와 failed-copy 정리/대상 보존**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **non-random-access 컨테이너 support를 증명하지 않으며 오히려 요구사항 밖으로 고정**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **broad generic 규약 + 실패 회귀**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `d0295f82614b`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_random_access_batch.cpp`; throwing value 테스트 type와 live/copy counters; `tests/compile/template_headers.cpp`, `template_const_iterator_fail.cpp`, `template_list_sort_fail.cpp`; Make 규약 대상.
- 핵심 코드 발췌 위치: `aaeff163baf8:tests/test_random_access_batch.cpp`는 vector와 deque instantiation에서 반복자/const_iterator, standard 알고리즘, access, sort, equality, copy/대입/자기 대입을 사용합니다. throwing element는 지정 copy에서 예외를 발생시키고 live count를 노출합니다.
- 변경 전/후 차이: generic 인터페이스의 정상 치환만이 아니라 element copy 실패에서 생성 정리와 대입 대상 보존을 확인하는 테스트 layer가 추가되었습니다.
- 직접 확인한 소유권·수명·상태 관계: failed 복사 생성 뒤 성공했던 임시 객체 elements가 사라져 live baseline으로 돌아와야 하고, failed 대입 뒤 destination values와 원본 values가 각각 유지되어야 합니다. 반복자 컴파일 cases는 const batch의 변경을 금지하는 공개 도형을 고정합니다.
- 직접 확인한 실패 처리: vector/deque의 element copying 중 throw를 위치별로 제어하고, `std::list` sort 사용은 expected 컴파일 실패로 다룹니다. 따라서 non-random-access support를 증명하는 것이 아니라 요구사항 밖임을 명시합니다.
- 실행한 테스트와 결과: 미실행. unit/컴파일-규약 원본과 기대 조건을 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 컨테이너 치환, 알고리즘 사용, throwing element에서의 정리와 strong 대입을 검증했습니다.

### `d0295f82614b` — feat(batch): 입력 문법과 원자 교체 구현

- 중요도: **S**
- 태그: **ARCH, EXCEPTION, INTEGRATION**
- 원문에서 정한 역할: 레코드 파싱, 중복 검사, RPN 계산, 성공 시 교환을 한 처리 과정으로 결합합니다.
<!-- 원문 분류 요약: Implements whole-stream batch parsing, uniqueness, RPN evaluation, and swap-on-success replacement. -->

#### 이 커밋 직전 상태와 문제
- 직전 관련 개발 흐름 SHA `aaeff163baf8`를 먼저 체크아웃하여 이 커밋이 추가되기 전 표현/소유권/상태-반영 방식을 확인하세요.
- 원문이 확정한 문제와 결정을 실제 diff와 대응시키되, 원문에 없는 동기를 추가로 추정하지 마세요.

#### 해당 SHA에서 확인할 실제 코드
- [x] 입력 record를 name과 RPN expression으로 split/trim하고 identifier grammar를 검사하는 helper chain을 찾으세요.
- [x] duplicate-name tracking 구조와 duplicate rejection이 후보 반영보다 앞에서 일어나는지 확인하세요.
- [x] 각 expression이 `RpnEvaluator`를 호출해 `JobResult` 후보에 추가되는 call graph를 그리세요.
- [x] existing `results_`와 분리된 local 후보 result set/map이 whole-입력 처리 동안 유지되는지 확인하세요.
- [x] empty 입력, malformed record, duplicate, evaluator error, 스트림 실패 각각이 final swap에 도달하지 않는 브랜치를 기록하세요.
- [x] complete 입력 성공 뒤 후보 vector가 `results_`와 swap되는 단 하나의 반영 point를 표시하세요.

#### 소유권·수명·상태 변화
- [x] 상태 필드별 소유자, 수명, valid 상태를 표로 직접 정리하세요.
- [x] throw 가능한 연산과 예외를 던지지 않는 커밋 작업의 순서를 실제 코드 라인 기준으로 적으세요.
- [x] 성공 전 임시 객체/후보 상태와 성공 후 published 상태를 구분해 그리세요.

#### 실패 상황과 보장 경계
- [x] 원문이 지목한 실패를 하나 이상 실제 제어 흐름으로 따라가고, exception 직전/직후 관찰 가능한 상태를 기록하세요.
- [x] 이 커밋이 보장하는 것과 아직 보장하지 않는 것을 원문과 해당 SHA 코드에 근거해 구분하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `42d411e42268`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/BatchEngine.hpp`의 `JobResult`, `BatchEngine::results_`, `replace()`; `src/BatchEngine.cpp`의 trim/name/line parser, duplicate map, `RpnEvaluator::evaluate()`, local 후보, final swap.
- 핵심 코드 발췌 위치: `d0295f82614b:src/BatchEngine.cpp`는 각 line을 한 개의 `|`로 분리하고 field trim/name grammar를 검사합니다. local 후보와 `std::map<std::string, long> seen`에만 결과를 누적하고 complete non-empty 입력 뒤 `results_.swap(candidate)`를 호출합니다.
- 변경 전/후 차이: generic 컨테이너 groundwork 위에 whole-스트림 parsing, duplicate rejection, checked RPN, 영속 result replacement가 통합되었습니다. 기존 `results_`를 line마다 변경하지 않고 local 상태가 전체 입력을 소유합니다.
- 직접 확인한 소유권·수명·상태 관계: 입력 스트림은 borrowed 원본이고 parsed strings, seen map, 후보 results는 call-local owners입니다. `JobResult`가 name/value를 값으로 소유하며 final swap 전 `results_`는 prior batch를 계속 소유합니다.
- 직접 확인한 실패 처리: malformed/blank record, missing/extra separator, invalid name, duplicate name, RPN exception, 스트림 실패, empty 입력은 final swap에 도달하지 않습니다. local 컨테이너가 partial records를 정리하므로 보상 변경 없이 prior results가 유지됩니다. 이 시점의 line-loop EOF 판정은 후속 fix 전입니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 parser, evaluator call, 후보 반영을 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 전체 입력을 local 후보에서 검증·계산한 뒤 한 번만 교체하는 batch transaction을 만들었습니다.

### `42d411e42268` — feat(batch): 결과 정렬과 직렬화 제공

- 중요도: **A**
- 태그: **DETERMINISM, EXCEPTION, CORE**
- 원문에서 정한 역할: 전체 결과 정렬 규칙과 고정 로캘을 사용하는 단계별 직렬화를 추가합니다.
<!-- 원문 분류 요약: Adds total result ordering and classic-locale staged batch serialization. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `d0295f82614b`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] result comparator가 value를 먼저, name을 tie-breaker로 비교해 total order를 만드는 실제 코드를 확인하세요.
- [x] sorting이 published `results_`가 아니라 후보에서 일어나도록 상태 변경 순서를 추적하세요.
- [x] serialization이 classic-locale 임시 객체 스트림에서 완성된 bytes를 만든 뒤 destination으로 쓰는 준비 영역을 확인하세요.
- [x] caller 스트림 locale/flags를 덮어쓰지 않는지 구현과 테스트를 함께 확인하세요.
- [x] formatting stage의 실패는 partial record 순서를 publish하지 않지만 final destination write 실패는 되돌리기 대상이 아님을 실제 코드 경계로 기록하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `af57a8f9c5fe`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/BatchEngine.cpp`의 `resultLess()`, 후보 sort, `BatchEngine::write()`; `JobResult::name()/value()`.
- 핵심 코드 발췌 위치: `42d411e42268:src/BatchEngine.cpp`의 comparator는 value를 먼저 비교하고 같으면 name을 비교합니다. 후보는 반영 전에 sort되고 `write()`는 classic-locale 임시 객체 스트림에 모든 `value | name` row를 만든 뒤 destination에 한 번 씁니다.
- 변경 전/후 차이: 입력 순서대로 저장하던 result set에 `(value, name)` total order와 결정적 serialization이 추가되었습니다. sorting과 formatting 모두 published 상태나 caller 스트림을 준비 중간에 직접 변경하지 않습니다.
- 직접 확인한 소유권·수명·상태 관계: sort 대상은 local 후보이며 성공 후 vector가 `results_`로 이동합니다. write 후보인 `ostringstream`와 string은 local 소유자이고 caller 스트림의 flags/locale는 수정하지 않습니다.
- 직접 확인한 실패 처리: sort comparison/element 작업이나 local formatting이 실패하면 result 반영 또는 destination write 전입니다. final destination `write()` 실패는 이미 보낸 bytes를 되돌리기하지 않습니다. comparator는 equal value에 name tie-breaker를 두어 삽입 permutation에 의존하지 않습니다.
- 실행한 테스트와 결과: 미실행. comparator와 staged serializer를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: total order와 staged classic-locale serialization으로 batch 외부 결과를 결정적으로 만들었습니다.

### `af57a8f9c5fe` — feat(batch): 두 container의 정렬 결과 대조

- 중요도: **A**
- 태그: **GENERIC, INTEGRATION, DETERMINISM**
- 원문에서 정한 역할: vector와 deque 후보를 독립적으로 정렬하고 결과가 다르면 반영 전에 거부합니다.
<!-- 원문 분류 요약: Runs and sorts vector- and deque-backed batches, then rejects disagreement before commit. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `42d411e42268`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] accepted job이 vector-backed와 deque-backed `RandomAccessBatch` 후보 양쪽에 어떻게 추가되는지 추적하세요.
- [x] 두 후보가 동일 comparator로 독립 sort되는 호출을 확인하세요.
- [x] `equal_ranges()` 또는 대응 비교가 disagreement를 detect하는 위치와 `logic_error` 발생 전 반영 상태를 확인하세요.
- [x] 불일치 시 prior engine 상태가 유지되는 이유를 후보 수명과 final 커밋 순서로 설명하세요.
- [x] 추가 memory/sort work가 deliberate verification이라는 사실이 코드 구조에서 어떻게 드러나는지 기록하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `9ba0e7c897ed`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/BatchEngine.cpp`의 vector/deque `RandomAccessBatch` candidates, dual `push_back()`, `sort()`, `equal_ranges()`, `logic_error`, vector-to-`std::vector` 후보 생성, final `results_.swap()`.
- 핵심 코드 발췌 위치: `af57a8f9c5fe:src/BatchEngine.cpp`는 각 accepted `JobResult`를 vector-backed와 deque-backed batch 양쪽에 추가하고 같은 `resultLess`로 독립 정렬합니다. range가 다르면 `batch container disagreement`를 throw하고, 같을 때만 vector range로 final 후보를 만들어 게시합니다.
- 변경 전/후 차이: 단일 표현 계산에서 두 컨테이너가 같은 semantic result를 만드는지 실제 실행 경로에서 대조하는 구조로 확장되었습니다. 추가 memory와 sort work는 커밋 전 검증에 사용됩니다.
- 직접 확인한 소유권·수명·상태 관계: vector/deque candidates와 final `std::vector` 후보는 모두 local owners입니다. `results_`는 두 sort와 equality, final vector 생성이 끝날 때까지 prior 상태를 보유합니다.
- 직접 확인한 실패 처리: 어느 컨테이너의 삽입/sort/할당이나 equality 전 단계가 실패하거나 두 range가 불일치하면 final swap이 실행되지 않습니다. local destructors가 양쪽 후보를 정리합니다. 같은 comparator 구현을 공유하므로 독립 oracle 자체는 아니지만 표현 disagreement는 탐지합니다.
- 실행한 테스트와 결과: 미실행. dual-컨테이너 구현과 반영 순서를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: vector와 deque 결과를 커밋 전에 독립 정렬·대조해 표현 disagreement를 거부했습니다.

### `9ba0e7c897ed` — test(batch): 입력 순열과 출력 결정성 검증

- 중요도: **A**
- 태그: **TEST, DETERMINISM, EDGE**
- 원문에서 정한 역할: 입력 순열 불변성과 반복 실행의 바이트 단위 동일 출력을 검증합니다.
<!-- 원문 분류 요약: Verifies input-permutation invariance, tie ordering, single-element behavior, and repeatable output. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `af57a8f9c5fe`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] 동일 job set의 여러 삽입 permutation을 만드는 테스트 준비 코드와 expected canonical order를 확인하세요.
- [x] equal-valued jobs에서 name tie-breaker가 빠졌을 때 잡힐 수 있는 테스트 case를 찾으세요.
- [x] vector/deque batch ranges가 동일함을 확인하는 검사문을 기록하세요.
- [x] 동일 engine 상태를 여러 번 serialize해 byte-identical 출력을 비교하는 결정적 회귀 테스트를 확인하세요.
- [x] 이 테스트가 단순 sortedness가 아니라 permutation invariance까지 증명하도록 입력 구성이 어떻게 설계되었는지 적으세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **canonical total order와 출력 determinism**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **입력 permutation, equal-value tie, repeated rendering**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **permutation 회귀 + repeated-byte comparison**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **comparator, vector/deque sorting, serialization**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **입력 순서/표현에 독립적인 결과**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **스트림 실패 되돌리기가나 할당 실패는 이 커밋 단독 범위가 아님**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 회귀 테스트**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `ea23237ad506`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_batch_engine.cpp`의 permutation/tie/single-element/repeated-출력 cases; `tests/test_random_access_batch.cpp`의 cross-컨테이너 equality; CLI 테스트 준비 코드.
- 핵심 코드 발췌 위치: `9ba0e7c897ed:tests/test_batch_engine.cpp`는 동일 job set의 서로 다른 입력 순서를 하나의 expected `(value, name)` row 순서와 비교하고, equal value에서 `Alpha`, `alpha`, `beta`, `zeta` name order를 확인합니다. 같은 engine을 반복 serialize한 bytes도 비교합니다.
- 변경 전/후 차이: sortedness 한 사례에서 입력 permutation 독립성, tie total order, vector/deque equality, repeated rendering의 byte determinism으로 검증 범위가 넓어졌습니다.
- 직접 확인한 소유권·수명·상태 관계: 각 permutation은 별도 engine/후보에서 계산되며 결과 bytes만 공통 oracle과 비교됩니다. repeated write는 영속 `results_`를 변경하지 않아야 합니다.
- 직접 확인한 실패 처리: name tie-breaker가 없거나 unstable/입력-dependent order이면 equal-valued permutations이 다른 bytes를 만들어 실패합니다. 이 커밋은 스트림/할당 실패 되돌리기를 직접 주입하지 않습니다.
- 실행한 테스트와 결과: 미실행. 결정적 cases와 expected bytes를 검사했으며 테스트 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: canonical order가 입력 순서와 컨테이너 표현, 반복 출력에 독립적임을 고정했습니다.

### `ea23237ad506` — fix(batch): 입력 stream 종료 상태를 명확히 구분

- 중요도: **A**
- 태그: **DEBUG, PARSING, EDGE**
- 원문에서 정한 역할: 정상 EOF, 줄바꿈 없는 마지막 레코드, 실제 입력 실패를 명확히 구분합니다.
<!-- 원문 분류 요약: Distinguishes clean EOF, an unterminated final record, and actual stream failure. -->

#### 실패 → 수정 → 테스트 연결
- **기존 가정:** 일반적인 `getline` loop 종료를 clean 입력 completion과 동일하게 취급할 수 있었다.
- **실제 실패 / 위험:** valid final unterminated line을 버리거나 실제 I/O fault를 EOF처럼 받아들일 수 있었다.
- **root cause:** line extraction 종료 상태가 complete line / clean EOF / actual 실패로 분류되지 않았다.
- **수정된 불변식 / 결정:** record reader가 세 outcome을 구분하고 clean completion일 때만 batch transaction을 커밋합니다.
- **실제 코드 확인:** 기존 reader/loop와 이 SHA의 reader helper를 비교해 스트림 flags와 return classification을 확인합니다.
- **회귀 테스트:** `b4ddd78fb9aa`에서 스트림 실패 뒤 seeded 상태가 보존되는 경로를 확인한다.

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `9ba0e7c897ed`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] 직전 batch reader의 일반 `getline` loop와 이 SHA의 record-reader helper를 관련 코드로 비교하세요.
- [x] reader가 complete line, final unterminated line + clean EOF, actual 실패의 세 outcome을 어떤 스트림 flags로 구분하는지 확인하세요.
- [x] trailing newline이 없는 마지막 record가 후보에 포함되는 path를 추적하세요.
- [x] `badbit` 또는 non-EOF 실패가 transaction rejection으로 이어지고 final swap을 막는 브랜치를 확인하세요.
- [x] fix가 syntax/arithmetic transaction에 transport-상태 success condition을 추가한 것임을 상태 반영 위치와 연결하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `b4ddd78fb9aa`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/BatchEngine.cpp`의 `readLine(std::istream&, std::string&)`, `BatchEngine::replace()` loop; 직전 `std::getline` 기반 loop.
- 핵심 코드 발췌 위치: `ea23237ad506:src/BatchEngine.cpp`의 `readLine()`은 `input.get(value)`로 newline까지 누적합니다. extraction 종료 후 `!input.eof()`면 입력 실패를 throw하고, clean EOF에서는 `!line.empty()`를 반환해 newline 없는 final record를 한 번 더 처리합니다.
- 변경 전/후 차이: 일반 `getline` loop 종료와 후속 flag 판정에 의존하던 reader를 complete line, clean EOF의 final unterminated line, non-EOF 실패로 명시적으로 분리했습니다.
- 직접 확인한 소유권·수명·상태 관계: line은 local 후보 record이고 read helper가 true를 반환한 경우에만 parse/RPN/vector/deque candidates로 전달됩니다. 스트림은 caller-owned이며 위치를 consume하지만 engine 상태는 final swap 전까지 유지됩니다.
- 직접 확인한 실패 처리: final line에 newline이 없어도 bytes가 있으면 정상 record로 처리합니다. initial/between-record read가 `badbit` 등 non-EOF 실패로 끝나면 `invalid batch input`을 throw해 커밋을 막습니다. 입력 위치가나 flags를 원상복구하지는 않습니다.
- 실행한 테스트와 결과: 미실행. fix 전후 reader와 후속 bad-스트림 테스트를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 성공한 입력 completion의 정의를 clean EOF, final unterminated record, 실제 read 실패로 분리했습니다.

### `b4ddd78fb9aa` — test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증

- 중요도: **A**
- 태그: **TEST, EXCEPTION, RISK**
- 원문에서 정한 역할: 문법·산술·스트림·할당 실패 전 범위에서 기존 상태 보존을 검증합니다.
<!-- 원문 분류 요약: Sweeps malformed input, arithmetic, stream, and allocation failures after seeding engine state. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `ea23237ad506`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] known result로 engine을 seed하는 setup과 prior serialized bytes baseline을 확인하세요.
- [x] malformed 입력, RPN arithmetic 실패, 스트림 실패를 각각 주입하는 테스트 cases와 production 실패 처리를 매핑하세요.
- [x] observed 할당 실패 point sweep이 parsing, duplicate tracking, evaluator stack, two candidates, sort/compare 중 어디를 통과하는지 기록하세요.
- [x] 모든 rejection 후 result objects와 serialized bytes가 seed와 동일하고 live 할당 baseline이 복구되는 검사문을 확인하세요.
- [x] CLI 실패 cases에서 stdout이 비어 있음을 검사해 object-상태 atomicity가 프로세스-출력 atomicity로 연결되는 지점을 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **BatchEngine whole-입력 strong transaction**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **syntax, arithmetic, 스트림, 할당 failures**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **seeded-상태 실패 sweep + 할당 블록 accounting + CLI 실패 테스트 준비 코드**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **parsing, duplicate set, RPN, two candidates, sort/compare, final 반영**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **협력 layer 실패 뒤 상태/bytes/baseline 되돌리기**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **외부 스트림 위치 자체를 되돌리는 보장은 아님**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 실패-injection + 통합**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/test_batch_engine.cpp`의 `checkInvalidPreserves()`, `checkOverflowPreserves()`, bad-스트림 case; `tests/failure/test_batch_failure.cpp`; `tests/support/FailingNew.cpp`; batch CLI 실패 테스트 준비 코드.
- 핵심 코드 발췌 위치: `b4ddd78fb9aa:tests/test_batch_engine.cpp`는 실패 전 serialized bytes와 첫 `JobResult` 주소를 저장하고 syntax/RPN 오버플로/division/bad 스트림 뒤 동일한지 검사합니다. `tests/failure/test_batch_failure.cpp`는 observed 할당 count를 얻어 1..N 실패 sweep을 수행합니다.
- 변경 전/후 차이: representative parsing 테스트에서 syntax, arithmetic, 스트림, 할당 협력 layer 전체를 대상으로 seeded prior 상태와 소유권 baseline을 확인하는 회귀로 확장되었습니다.
- 직접 확인한 소유권·수명·상태 관계: engine을 `seed | 7`로 채운 뒤 replacement 실패마다 size/value/출력과 현재 할당 블록 수 기준값을 비교합니다. 주소 동일성 검사는 실패 중 `results_` vector 자체가 swap/재할당되지 않았음을 관찰합니다.
- 직접 확인한 실패 처리: malformed records/duplicates, RPN invalid/오버플로, pre-set bad 스트림, parsing·map·RPN stack·dual candidates·sort/final 후보에서 관찰된 할당 failures가 모두 final 반영 전에 예외로 끝납니다. CLI invalid cases는 empty stdout를 요구합니다. consumed 입력 스트림 위치는 되돌리기 대상이 아닙니다.
- 실행한 테스트와 결과: 미실행. unit 실패 helpers, 할당 sweep, CLI 테스트 준비 코드, Make targets를 검사했으나 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: batch transaction의 모든 관찰 협력 실패에서 prior objects, bytes, 주소, live allocations가 유지되는지 검증했습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `708c025ef2a0` | configurable random-access 컨테이너 abstraction과 cross-컨테이너 equality 도입 | `Container` parameter, inherited 반복자 types, `std::sort`, underlying value 소유권, 복사 후 교환 대입, `equal_ranges`를 확인했습니다. | non-random-access 컨테이너는 sort requirement 밖이며 실제 throwing-value 근거는 아직 없습니다. |
| `aaeff163baf8` | 반복자/알고리즘/substitution과 throwing-value 실패 계약 검증 | vector/deque substitution, 반복자 알고리즘, 컴파일 실패 list sort, throwing element copy/live counters가 generic/exception 규약을 확인합니다. | whole-스트림 parsing과 영속 transaction은 아직 도입 전입니다. |
| `d0295f82614b` | whole-입력 후보와 swap-on-success 반영으로 batch transaction 확립 | line grammar, duplicate map, RPN call, local 후보, non-empty/스트림 completion 후 단일 `results_.swap(candidate)`를 확인했습니다. | canonical sort/serialization과 스트림 completion의 세부 구분은 후속 변경이 필요합니다. |
| `42d411e42268` | `(value, name)` total order와 classic-locale staged serialization 추가 | value-first/name-tie comparator와 후보 sort, classic 임시 객체 serialization, final single write를 확인했습니다. | destination write 되돌리기와 cross-컨테이너 agreement는 아직 보장하지 않습니다. |
| `af57a8f9c5fe` | vector/deque를 독립 sort하고 disagreement를 커밋 전 거부 | 각 result를 vector/deque batches에 넣고 독립 sort 후 `equal_ranges`가 true일 때만 final vector를 게시합니다. | 두 경로가 같은 comparator를 공유하므로 완전히 독립된 semantic oracle는 아닙니다. |
| `9ba0e7c897ed` | 입력 permutation independence와 repeated byte determinism 검증 | 서로 다른 permutations, equal-value name ties, one element, repeated bytes가 canonical result를 비교합니다. | 스트림/할당 실패 되돌리기는 이 커밋 단독 범위가 아닙니다. |
| `ea23237ad506` | record reader가 clean EOF/final unterminated line/입력 실패를 분리하도록 수정 | char-level `readLine()`이 newline, non-empty final EOF record, non-EOF 실패를 분류하고 실패에서 swap을 막습니다. | caller 스트림 위치/flags 자체는 되돌리지 않습니다. |
| `b4ddd78fb9aa` | syntax/arithmetic/스트림/할당 실패 전 범위에서 seeded 상태 되돌리기 검증 | seeded bytes와 first-result address, syntax/RPN/bad-스트림 cases, observed 할당 sweep와 현재 할당 블록 수 기준값을 확인합니다. | 관찰되지 않은 환경 실패와 external 스트림 되돌리기는 증명하지 않습니다. |

## 문제 → 수정 → 검증 연결

- `d0295f82614b`: whole-입력 후보와 swap-on-success transaction을 도입합니다.
- `ea23237ad506` fix: successful input completion의 정의를 clean EOF/final unterminated line/actual failure로 정교화합니다.
- `b4ddd78fb9aa`: syntax/arithmetic/스트림/할당 실패 뒤 seeded 상태 되돌리기를 폭넓게 검증합니다.

### 학습자 연결 기록
- 최초 위험/맹점: generic 컨테이너의 value semantics가 안전해도 whole-스트림 작업이 영속 results를 line마다 변경하면 parse·RPN·sort·스트림 실패에서 partial batch가 노출됩니다. 일반 getline 종료를 성공으로 간주하는 것도 transport fault를 숨길 수 있습니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: duplicate/RPN 실패는 여러 valid records 뒤 발생할 수 있고, vector/deque 또는 sort 할당도 late 실패입니다. newline 없는 final record와 bad 스트림은 같은 loop 종료처럼 보일 수 있습니다.
- 수정/강화된 결정: 모든 records와 duplicate 상태, checked RPN results, vector/deque sort/compare, final vector 생성을 local candidates에서 끝내고 clean 입력 completion 뒤 swap합니다. reader는 EOF와 실패를 명시적으로 분류합니다.
- 해당 코드 위치: `d0295f82614b`부터 `ea23237ad506`까지의 `src/BatchEngine.cpp`, 특히 `replace()`, `resultLess()`, `readLine()`, `write()`.
- 이를 고정하는 회귀/근거: `9ba0e7c897ed`의 permutation/출력 테스트와 `b4ddd78fb9aa`의 seeded syntax/arithmetic/스트림/할당 실패 테스트.

## 소유권·상태·담당 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: configurable random-access 컨테이너 abstraction과 cross-컨테이너 equality 도입
- 개발 흐름 종료 상태: syntax/arithmetic/스트림/할당 실패 전 범위에서 seeded 상태 되돌리기 검증
- [x] 중간 커밋마다 소유자/상태 publisher/정리 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned 상태가 함께 등장하면 각각의 수명 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `708c025ef2a0`/`aaeff163baf8`: generic 소유자가 vector/deque substitution과 throwing element copy에서 value/대입 보장을 제공합니다.
2. `d0295f82614b`: 영속 `results_`의 publisher가 whole-입력 final swap 하나로 제한됩니다.
3. `42d411e42268`/`af57a8f9c5fe`: 반영 전에 total-order sorting, dual 표현 comparison, final vector 생성이 추가됩니다.
4. `9ba0e7c897ed`: canonical result가 입력 permutation과 repeated rendering에 독립적인지 bytes로 고정합니다.
5. `ea23237ad506`: transaction success 조건에 clean transport completion을 추가하고 final unterminated record를 보존합니다.
6. `b4ddd78fb9aa`: syntax, arithmetic, 스트림, 할당 실패 뒤 prior vector object/address/bytes와 할당 baseline을 검사합니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `input stream → complete record read → grammar/uniqueness → checked RPN → vector/deque candidates → total-order sort/compare → final vector publication → staged serialization`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준으로 `BatchEngine::replace()`는 `readLine()`이 승인한 complete record만 parse합니다. name grammar와 duplicate를 검사하고 checked RPN 값을 vector/deque-backed local batches에 동시에 넣습니다. clean completion과 non-empty condition 뒤 두 batch를 같은 total comparator로 sort하고 range agreement를 검사한 후, vector range로 final 후보를 완성해 `results_.swap(candidate)`합니다. `write()`는 published results를 classic-locale 임시 객체 스트림에서 직렬화합니다.

초기 generic 컨테이너와 비교하면 generic value semantics, whole-입력 transaction, 결정적 total order, dual-표현 check, transport completion이 하나의 구성 요소로 결합되었습니다. 모든 관찰 preparation 실패는 prior result object와 bytes를 보존하지만 입력 스트림 위치와 destination final write는 되돌리기하지 않습니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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

- 실제 caller → callee 흐름: 입력 스트림 → `readLine()` → `parseLine()`/name validation → duplicate map → `RpnEvaluator::evaluate()` → vector/deque `RandomAccessBatch::push_back()` → dual `sort(resultLess)` → `equal_ranges()` → final `std::vector` 후보 → `results_.swap(candidate)` → staged `write()`.
- 핵심 상태 필드: 영속 `std::vector<JobResult> results_`; local vector/deque batches, `seen` map, line/name/expression, final vector 후보.
- 자원 소유자 / 비소유 참조: caller owns 입력/출력 streams; engine owns published `results_`; all parsing, duplicate, RPN, dual-컨테이너 and serialization candidates are call-local owners. `results()` returns a const 비소유 참조.
- 상태 확정 지점: clean non-empty 입력, both sorts, agreement, final vector 생성을 모두 통과한 뒤의 `results_.swap(candidate)`입니다.
- 정리 path: syntax/duplicate/RPN/스트림/할당/sort/disagreement 실패는 local map·batches·strings가 정리되고 prior `results_`는 그대로입니다. 출력 준비 영역 실패는 destination write 전 정리됩니다.
- 최종 불변식 설명: published batch는 complete successful 입력 전체에서 나온 unique jobs이며 `(value, name)` total order로 vector/deque가 동의한 결과입니다. incomplete 후보는 publish되지 않고 repeated serialization은 입력 order와 caller formatting 상태에 독립적입니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 06-generic-containers-transactional-batch.md =====

===== BEGIN FILE: 07-contactbook-replacement-guarantee.md =====
# ContactBook 교체 연산에도 강한 예외 보장 적용하기

## 개발 흐름의 목표

고정 크기 ring buffer도 내부 `Contact` 대입이 할당을 일으킬 수 있으므로 slot content와 ring 메타데이터를 하나의 transaction으로 다뤄야 함을 확인하고, 뒤늦은 strong-guarantee 수정과 회귀 검증을 복원합니다.

**원문에서 정한 의의:** 작은 고정 배열도 값 대입 과정에서 메모리를 할당한다면 강한 예외 보장이 필요함을 보여줍니다. 슬롯 값, `next_`, `size_`를 하나의 논리적 변경으로 다루고 다른 하위 시스템과 같은 후보 생성 후 교환 방식을 적용합니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- logical oldest-to-newest order와 physical slot index는 어떻게 분리되는가?
- full-capacity replacement에서 direct `Contact` 대입이 왜 부분 상태 변경 위험을 만드는가?
- detached replacement를 먼저 완성한 뒤 slot에 swap하는 순서가 어떤 불변식을 보존하는가?
- `next_`와 `size_`는 왜 slot 커밋 이후에만 변경되어야 하는가?
- 실패 sweep이 logical order와 할당 baseline까지 확인하는 이유는 무엇인가?

## 완료 기준

- [x] 초기 ring 표현과 logical index 변환을 해당 SHA에서 설명할 수 있습니다.
- [x] 초기 direct 대입 path와 fix의 detached-후보 path를 관련 SHA끼리 비교할 수 있습니다.
- [x] slot content, `next_`, `size_`의 coupled 불변식을 실패 시나리오로 설명할 수 있습니다.
- [x] full-capacity 실패 회귀가 실제 이전 취약 경로를 어떻게 재현하는지 확인할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- 완성되지 않은 contact replacement 후보는 stored slot에 publish되지 않는다.
- 강한 예외 보장 replacement는 할당 실패 시 slot content와 ring 메타데이터의 관찰 가능한 상태를 보존합니다.
- 직접 소유 값의 실패가 logical order/cursor 반영과 분리되어야 합니다.

### 주요 구현 난점

- 할당 가능한 value 대입이 있는 fixed array에서 partial 변경 방지.
- 할당 실패 sweep과 logical-order/할당 블록 accounting으로 ring transaction 검증.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `2f9b934b0825` | feat(contact): 고정 크기 연락처 저장 순서 보존 | B | CORE | 선택한 원형 버퍼 슬롯을 일반 `Contact` 대입으로 교체하는 초기 구현입니다. |
| 2 | `0ad14a57cab6` | fix(contact): 할당 실패에도 저장 상태 보존 | A | DEBUG, EXCEPTION, OWNERSHIP | 분리된 대체 값을 준비해 교환한 뒤 원형 버퍼 메타데이터를 진행시킵니다. |
| 3 | `8930c4d17bc1` | test(contact): 연락처 교체 실패 회귀 검증 | A | TEST, EXCEPTION, EDGE | 가득 찬 상태의 교체 중 할당 실패를 순회하며 순서·값·누수 기준값을 검증합니다. |

## 커밋별 학습 기록

### `2f9b934b0825` — feat(contact): 고정 크기 연락처 저장 순서 보존

- 중요도: **B**
- 태그: **CORE**
- 원문에서 정한 역할: 선택한 원형 버퍼 슬롯을 일반 `Contact` 대입으로 교체하는 초기 구현입니다.
<!-- 원문 분류 요약: Adds an eight-slot circular `ContactBook` with logical oldest-to-newest indexing. -->

#### 개발 흐름에서 확인할 구현 역할
- [x] `ContactBook`의 8-slot backing array, logical size, next 삽입 cursor를 찾으세요.
- [x] valid/empty contact 삽입 policy와 capacity 도달 후 replacement slot 선택을 확인하세요.
- [x] `at()`가 logical oldest-to-newest index를 physical array index로 변환하는 식과 out-of-range 브랜치를 기록하세요.
- [x] full-capacity replacement에서 selected slot에 ordinary `Contact` 대입을 수행하는 코드를 찾으세요.
- [x] 이 대입이 할당 실패 시 slot/메타데이터 transaction을 완전히 격리하지 못한다는 원본 설명을 코드상 변경 순서와 함께 기록하세요.
- [x] 이 커밋이 다음 관련 커밋의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `0ad14a57cab6`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `include/cppf/ContactBook.hpp`의 `capacity`, `contacts_`, `size_`, `next_`, `add()`, `at()`; `src/ContactBook.cpp`의 삽입/replacement와 logical-index 계산.
- 핵심 코드 발췌 위치: `2f9b934b0825:src/ContactBook.cpp`에서 non-empty contact는 `contacts_[next_] = contact`로 selected slot에 직접 대입되고, 이후 `next_ = (next_ + 1) % capacity`, `size_` 증가가 실행됩니다. `at()`는 full이면 `first = next_`, 아니면 0을 사용합니다.
- 변경 전/후 차이: 최대 8개 contact를 저장하고 capacity 이후 oldest physical slot을 덮는 circular 표현이 추가되었습니다. logical observation은 physical 배열 순서와 분리되어 oldest-to-newest로 제공됩니다.
- 직접 확인한 소유권·수명·상태 관계: `ContactBook`이 fixed array의 각 `Contact` 값을 소유합니다. `next_`는 다음 삽입 physical slot이고 `size_`는 유효 logical count입니다. `at()`가 반환하는 `const Contact&`는 book 수명과 후속 replacement에 종속된 borrowed reference입니다.
- 직접 확인한 실패 처리: empty contact는 아무 변경 없이 반환하고 out-of-range logical index는 예외입니다. full book에서 ordinary `Contact::operator=`가 여러 owned string을 복사하다 실패하면 committed slot 내부가 부분 변경될 수 있으므로, 뒤의 메타데이터가 아직 안 바뀌어도 slot/content/order transaction은 보장되지 않습니다.
- 실행한 테스트와 결과: 미실행. 지정 SHA의 ring 표현과 변경 순서를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: logical order를 보존하는 8-slot ring을 만들었지만 replacement는 throwing direct 대입에 의존했습니다.

### `0ad14a57cab6` — fix(contact): 할당 실패에도 저장 상태 보존

- 중요도: **A**
- 태그: **DEBUG, EXCEPTION, OWNERSHIP**
- 원문에서 정한 역할: 분리된 대체 값을 준비해 교환한 뒤 원형 버퍼 메타데이터를 진행시킵니다.
<!-- 원문 분류 요약: Copies a contact into a detached candidate before swapping and advancing the ring. -->

#### 실패 → 수정 → 테스트 연결
- **기존 가정:** fixed array slot에 `Contact`를 대입하고 ring 메타데이터를 갱신하는 정상 경로면 충분했습니다.
- **실제 실패 / 위험:** `Contact` string copy 할당이 throw하면 stored slot과 logical cursor/size의 conceptual transaction이 깨질 수 있었다.
- **root cause:** throwing value 대입을 committed slot에 직접 적용했습니다.
- **수정된 불변식 / 결정:** detached replacement를 완성한 뒤 slot에 예외를 던지지 않는 swap하고, 메타데이터는 그 뒤에 advance합니다.
- **실제 코드 확인:** `2f9b934b0825`의 direct 대입과 이 SHA의 detached-copy/swap/update 순서를 비교한다.
- **회귀 테스트:** `8930c4d17bc1`의 full-capacity 할당 실패 sweep을 확인한다.

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `2f9b934b0825`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] 초기 관련 SHA `2f9b934b0825`와 `ContactBook::add()`를 직접 비교해 direct 대입 제거 지점을 찾으세요.
- [x] incoming contact를 detached local replacement로 복사하는 시점과 그 복사에서 할당 exception이 날 수 있는 경로를 확인하세요.
- [x] replacement 완성 후 selected slot과 예외를 던지지 않는 `swap()`하는 상태 확정 지점을 표시하세요.
- [x] `next_`와 `size_` 변경이 slot swap 성공 뒤에만 실행되는 정확한 순서를 기록하세요.
- [x] copy 실패 시 slot content, logical oldest 위치, size가 모두 untouched인 이유를 before/after 상태로 설명하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `8930c4d17bc1`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `src/ContactBook.cpp`의 `ContactBook::add()`; `Contact` default 생성, 대입, `swap()`; `size_`, `next_` update order.
- 핵심 코드 발췌 위치: `0ad14a57cab6:src/ContactBook.cpp`는 `Contact replacement; replacement = contact; contacts_[next_].swap(replacement);`를 실행한 뒤에만 cursor와 size를 갱신합니다.
- 변경 전/후 차이: stored slot에 incoming value를 직접 대입하던 코드를 detached local replacement 완성 후 예외를 던지지 않는 slot swap으로 바꿨습니다. 메타데이터 update는 slot 커밋 뒤로 유지됩니다.
- 직접 확인한 소유권·수명·상태 관계: copy 준비 중에는 existing slot이 old contact 자원을 계속 소유하고 local `replacement`가 새 copies를 소유합니다. swap 후 slot이 새 contact를, replacement가 old slot value를 소유하며 scope 종료 시 old 자원을 파괴합니다.
- 직접 확인한 실패 처리: `replacement = contact` 중 할당이 실패하면 slot swap과 `next_`/`size_` update에 도달하지 않습니다. 따라서 physical slot bytes, logical oldest mapping, size/cursor가 모두 prior 상태로 남습니다. 성공 후 swap은 반영 point이며 old value 정리는 local replacement가 담당합니다.
- 실행한 테스트와 결과: 미실행. fix 전후 `ContactBook::add()`를 비교하고 후속 실패 테스트를 검사했으며 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: detached Contact를 완성한 뒤 slot swap과 메타데이터 advance를 수행해 ring replacement를 transaction으로 만들었습니다.

### `8930c4d17bc1` — test(contact): 연락처 교체 실패 회귀 검증

- 중요도: **A**
- 태그: **TEST, EXCEPTION, EDGE**
- 원문에서 정한 역할: 가득 찬 상태의 교체 중 할당 실패를 순회하며 순서·값·누수 기준값을 검증합니다.
<!-- 원문 분류 요약: Sweeps allocation failures during full-book replacement and verifies logical order and leak baselines. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `0ad14a57cab6`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] book을 capacity까지 채워 oldest replacement path를 강제로 만드는 테스트 setup을 확인하세요.
- [x] replacement copy의 모든 관찰 할당 point에 실패를 주입하는 sweep loop를 찾으세요.
- [x] 각 실패 후 size, logical order, field values, live-할당 count를 비교하는 검사문을 기록하세요.
- [x] 실패 sweep 뒤 한 번의 successful 삽입이 정상 ring advance를 확인하는 이유를 actual expected order와 연결하세요.
- [x] 이 결정적 회귀 테스트가 직접 throwing 대입을 stored slot에 다시 도입하는 변경을 어떻게 잡는지 실제 실행 경로를 매핑하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **ContactBook full-capacity replacement 강한 예외 보장**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **할당 실패 while replacing oldest slot**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **full-capacity 결정적 할당-실패 sweep + 할당 블록 accounting**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **Contact copy → detached replacement → slot swap → ring 메타데이터**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **size/order/fields/live blocks가 실패 뒤 불변**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **다른 contact operations의 모든 실패 형태를 포괄하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 회귀 테스트**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/failure/test_contact_failure.cpp`; `tests/support/FailingNew.hpp/.cpp`; contact/book value comparison helpers; `Makefile`의 contact 실패 대상.
- 핵심 코드 발췌 위치: `8930c4d17bc1:tests/failure/test_contact_failure.cpp`는 book을 capacity 8까지 채워 oldest-slot replacement 경로를 만들고, 성공 run에서 관찰한 할당 attempts를 1..N으로 순회해 각 지점에서 `std::bad_alloc`을 주입합니다.
- 변경 전/후 차이: production fix 위에 full-capacity 취약 경로를 직접 반복하는 결정적 실패 회귀가 추가되었습니다. 실패 뒤 상태 확인과 이후 정상 삽입 확인을 함께 수행합니다.
- 직접 확인한 소유권·수명·상태 관계: 각 실패 전 현재 할당 블록 수 기준값과 logical size/order/각 contact field를 저장합니다. exception 뒤 모두 동일해야 하며 outer scope 종료 후 전체 live count도 원래 baseline으로 돌아와야 합니다.
- 직접 확인한 실패 처리: detached replacement copy의 각 관찰 할당이 실패해도 slot/content/cursor가 바뀌지 않는지 검사합니다. sweep 뒤 한 번 성공 삽입을 수행해 oldest 하나만 제거되고 new contact가 newest로 추가되는 정상 advance를 확인합니다. 다른 Contact 작업의 모든 실패는 범위 밖입니다.
- 실행한 테스트와 결과: 미실행. 실패 controller, sweep loop, expected order와 Make 대상을 검사했으나 실행 파일은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: full ring replacement의 모든 관찰 할당 실패에서 값·순서·크기·live blocks를 고정했습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `2f9b934b0825` | 8-slot circular buffer와 logical order 도입, replacement는 direct 대입 상태 | 8-element `contacts_`, `size_`, `next_`, full일 때 `first = next_`인 logical indexing과 direct slot 대입을 확인했습니다. | throwing `Contact` 대입이 committed slot을 부분 변경할 수 있어 content와 메타데이터의 강한 transaction이 없습니다. |
| `0ad14a57cab6` | detached `Contact` 완성 → 예외를 던지지 않는 swap → 메타데이터 advance 순서로 수정 | default local `replacement`를 완성한 뒤 slot `swap()`을 하고 그 후 cursor/size를 갱신합니다. | 결정적 full-capacity 할당 실패 근거는 후속 테스트가 필요합니다. |
| `8930c4d17bc1` | full-capacity 할당 실패 sweep으로 size/order/values/live blocks 보존 검증 | capacity-seeded book에서 할당 positions를 sweep하고 실패 뒤 size/order/fields/현재 할당 블록 수 기준값과 성공 retry order를 비교합니다. | 다른 Contact APIs나 관찰되지 않은 allocator path 전체를 포괄하지는 않습니다. |

## 문제 → 수정 → 검증 연결

- `2f9b934b0825` 초기 상태: full-capacity slot replacement가 ordinary `Contact` 대입을 사용합니다.
- `0ad14a57cab6` fix: detached replacement → slot swap → metadata advance 순서로 변경합니다.
- `8930c4d17bc1` 회귀: 할당 실패 전 지점에서 size/order/values/live blocks를 고정합니다.

### 학습자 연결 기록
- 최초 위험/맹점: fixed array 자체는 할당하지 않아도 slot의 `Contact` value 대입이 owned strings를 복사하므로 committed slot에 직접 대입하면 중간 throw가 partial value를 남길 수 있습니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: 초기 `add()`는 slot 대입 뒤 메타데이터를 바꿨지만, 메타데이터가 unchanged여도 slot content가 이미 변할 수 있어 logical order의 old value 보존이 성립하지 않았습니다.
- 수정/강화된 결정: incoming contact를 detached local value에 완전히 복사하고, 성공 후 slot과 예외를 던지지 않는 swap한 뒤에만 `next_`와 `size_`를 갱신합니다.
- 해당 코드 위치: `2f9b934b0825:src/ContactBook.cpp`의 direct 대입과 `0ad14a57cab6:src/ContactBook.cpp`의 replacement/swap/update 순서.
- 이를 고정하는 회귀/근거: `8930c4d17bc1:tests/failure/test_contact_failure.cpp`의 full-capacity 할당-실패 sweep과 successful retry.

## 소유권·상태·담당 변화

- Source에서 확인되는 핵심 transition을 아래에 실제 코드 근거로 완성하세요.
- 시작 상태: 8-slot circular buffer와 logical order 도입, replacement는 direct 대입 상태
- 개발 흐름 종료 상태: full-capacity 할당 실패 sweep으로 size/order/values/live blocks 보존 검증
- [x] 중간 커밋마다 소유자/상태 publisher/정리 책임이 어디로 이동하거나 강화되는지 적으세요.
- [x] borrowed와 owned 상태가 함께 등장하면 각각의 수명 종료 지점을 표시하세요.

### 코드 검사로 복원한 변화

1. `2f9b934b0825`: book이 physical slots와 logical `size_`/`next_`를 소유하지만 replacement publisher는 throwing slot 대입입니다.
2. `0ad14a57cab6`: throw 가능한 copy 책임이 detached `replacement`로 이동하고 slot 반영은 `swap()`으로, logical 메타데이터 반영은 그 뒤로 분리됩니다.
3. `8930c4d17bc1`: full-capacity 실패마다 old logical 순서와 할당 소유권이 유지되고 이후 success가 정확히 한 번 advance하는지 검사합니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `logical insertion request → choose physical slot → detached Contact copy → slot swap commit → cursor/size advance → logical oldest-to-newest observation`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준으로 `ContactBook::add()`는 empty 입력을 무시하고, incoming contact를 local `replacement`에 먼저 복사합니다. 복사가 끝난 뒤 `contacts_[next_].swap(replacement)`로 selected physical slot을 교체하고, 그 다음 `next_`를 회전시키며 capacity 미만일 때만 `size_`를 증가시킵니다. `at()`는 full 여부에 따라 logical oldest physical index를 계산합니다.

초기 ring과 비교하면 slot value와 cursor/size가 하나의 성공 transaction으로 취급됩니다. 할당 실패는 stored values와 logical order를 보존하고 old slot 자원은 성공 뒤 local replacement가 정리합니다. 보장 범위는 contact replacement 경로이며 external borrowed reference의 후속 successful replacement invalidation까지 없애지는 않습니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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
- 자원 소유자 / 비소유 참조: book slots와 local replacement가 Contact 자원을 값으로 소유하고, `add()` 입력과 `at()` 반환 reference는 borrowed입니다.
- 상태 확정 지점: detached copy 성공 뒤 selected slot과 실행하는 예외를 던지지 않는 `swap()`이며, 메타데이터는 그 커밋 뒤에만 바뀝니다.
- 정리 path: copy 할당 실패는 local replacement가 자신의 부분 상태를 정리하고 slot/메타데이터를 건드리지 않습니다. 성공 후 local replacement 소멸자가 old slot 자원을 정리합니다.
- 최종 불변식 설명: 실패 시 slot content·logical order·cursor·size가 함께 보존되고, 성공 시 정확히 한 physical slot 교체와 한 번의 ring advance만 관찰됩니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 07-contactbook-replacement-guarantee.md =====

===== BEGIN FILE: 08-verification-supported-release-claims.md =====
# 저장소 내부 테스트를 배포 지원 근거로 확장하기

## 개발 흐름의 목표

in-tree 기능 테스트만으로는 확인할 수 없는 공개 API 도형, 외부 consumer packaging, 결정적 breadth, sanitizer/host prerequisite, ABI, compiler/platform 범위를 단계적으로 실행 가능한 release claim으로 확장하는 과정을 복원합니다.

**원문에서 정한 의의:** 코드에 적힌 의도와 실제로 주장할 수 있는 배포 지원 범위의 차이를 단계적으로 줄입니다. 공개 API 형태, 외부 패키징, 입력 상태 범위, 정의되지 않은 동작, 호스트 전제, ABI, 컴파일러·플랫폼 차이를 서로 다른 검증 계층에서 확인합니다.

## 이 개발 흐름을 이해하기 위한 핵심 질문

- unit 테스트, 컴파일-규약, CLI 테스트 준비 코드가 각각 어떤 blind spot을 담당하는가?
- positive/negative 번역 단위가 런타임 테스트와 다른 종류의 API 회귀를 어떻게 막는가?
- repository 밖 consumer가 in-tree 빌드에서 가려질 수 있는 어떤 dependency를 노출하는가?
- fixed-seed property와 large batch가 hand-written 테스트 준비 코드를 대체하지 않고 보완하는 이유는 무엇인가?
- portable baseline과 host-specific checks를 분리하지 않으면 어떤 지원 주장 왜곡이 생기는가?
- LP64 check와 compiler/platform matrix가 각각 어떤 portability claim을 executable하게 만드는가?

## 완료 기준

- [x] verification layer별 입력, 실패 조건, 증명하는 계약, 증명하지 않는 범위를 구분할 수 있습니다.
- [x] 공개 헤더 isolation과 external archive consumption을 실제 빌드 commands/테스트에서 확인할 수 있습니다.
- [x] fixed seed, sanitizer, ABI 검사문, CI matrix가 서로 중복되지 않는 증거를 제공하는 이유를 설명할 수 있습니다.
- [x] portable 대상과 platform 대상의 포함 관계 및 host prerequisite를 실제 Makefile/CI에서 복원할 수 있습니다.

## 원문에서 확인되는 불변식과 구현 난점

### 핵심 불변식

- 공개 headers는 비공개 include path 없이 컴파일되고 비공개 표현은 inaccessible해야 합니다.
- 지원하는 C++98 LP64 platform assumptions는 명시적으로 executable하게 검증됩니다.
- 결정적 출력과 owned-자원 release는 프로세스/release scope에서도 증거가 필요합니다.

### 주요 구현 난점

- portable verification과 host-specific archive/dependency/leak/sanitizer capability 분리.
- 컴파일 시점 API enforcement, external-consumer verification, reproducible property testing, compiler/platform matrix 구성.

위 항목은 원문이 확정한 범위입니다. 실제 코드에서 어떻게 구현되는지는 아래 학습 기록에서 직접 확인합니다.

## 커밋 목록

| 순서 | SHA | 커밋 제목 | 중요도 | 태그 | 원문에서 확정된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `6e78ced59357` | test(contact): 공개 계약과 명령행 세션 검증 | A | API, TEST, INTEGRATION | 단위 테스트, 컴파일 규약, CLI 통합 계층을 도입합니다. |
| 2 | `4bbbfd191669` | test(contracts): 공개 include와 소유권 규칙 검증 | A | API, TEST, OWNERSHIP | 라이브러리 전체의 성공·실패 공개 규약 검사를 확장합니다. |
| 3 | `01271d795d58` | test(consumer): 저장소 밖 공개 library 연결 검증 | A | API, INTEGRATION, PORTABILITY | 저장소 밖에서 공개 헤더와 아카이브만 사용해 외부 소비자 코드를 컴파일·실행합니다. |
| 4 | `9e07d3bc86d3` | test(boundary): 변환·배치 속성과 대용량 경계 검증 | A | TEST, DETERMINISM, EDGE | 고정 시드 속성 검사와 대규모 일괄 처리 스트레스 테스트를 추가합니다. |
| 5 | `45e9bbfd6b75` | build(check): sanitizer와 portable 검사 계층 구성 | A | ARCH, PORTABILITY, TEST | 빌드·이식성·플랫폼 검사를 분리하고 ASan/UBSan의 역할을 구분합니다. |
| 6 | `ab441fa8737c` | test(portability): 지원 LP64 데이터 모델 검증 | B | PORTABILITY, API | 지원하는 LP64 ABI 전제를 실행 가능한 검사로 만듭니다. |
| 7 | `50565bd67e03` | ci: 지원 compiler와 platform matrix 검증 | B | PORTABILITY, TEST | 확립한 지원 조건을 GCC/Clang, Linux/macOS 조합에서 실행합니다. |

## 커밋별 학습 기록

### `6e78ced59357` — test(contact): 공개 계약과 명령행 세션 검증

- 중요도: **A**
- 태그: **API, TEST, INTEGRATION**
- 원문에서 정한 역할: 단위 테스트, 컴파일 규약, CLI 통합 계층을 도입합니다.
<!-- 원문 분류 요약: Introduces unit, compile-contract, and CLI fixture layers for the contact subsystem. -->

#### 핵심 설계 / 실패 경계 확인
- [x] contact 구성 요소 테스트가 unit, 컴파일-규약, 명령-line 테스트 준비 코드로 분리되는 빌드·테스트 targets를 찾으세요.
- [x] positive 컴파일 테스트가 공개 헤더를 두 번 include하고 exported names만 사용하는 번역 단위를 확인하세요.
- [x] 비공개 표현 접근을 의도적으로 시도해 컴파일 실패를 요구하는 negative 규약을 확인하세요.
- [x] real 실행 파일 session 테스트 준비 코드가 ADD/LIST/invalid/QUIT transcript 전체 bytes를 비교하는 방식을 기록하세요.
- [x] 세 layer가 각각 domain 동작, API 도형, 프로세스 protocol 중 무엇을 검증하는지 구분하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **Contact 공개 API, domain order, 프로세스 protocol**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **비공개 표현 exposure, header isolation, session drift**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **unit + positive/negative 컴파일 + CLI transcript**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **Contact/ContactBook 공개 headers and real app**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **multi-layer verification pattern이 작동함**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **repository 전체 API/release packaging까지는 아직 아님**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **broad 통합/규약**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `4bbbfd191669`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `Makefile`의 contact unit/규약/통합 targets; `tests/compile/contact_headers.cpp`, contact 비공개-access negative 번역 단위; `tests/check_cli.sh`; contact unit 테스트와 실제 app 실행 파일.
- 핵심 코드 발췌 위치: `6e78ced59357`의 positive 컴파일 unit은 공개 contact header를 반복 include하고 exported API만 사용합니다. negative unit은 비공개 표현 접근을 시도하며 harness는 compiler rejection을 success로 취급합니다. CLI 스크립트는 scripted ADD/LIST/invalid/QUIT session의 status와 exact bytes를 비교합니다.
- 변경 전/후 차이: in-프로세스 domain unit 테스트 하나에서 공개 헤더 도형과 실제 프로세스 protocol까지 서로 다른 실패 조건으로 검사하는 세 층 구조가 생겼습니다.
- 직접 확인한 소유권·수명·상태 관계: unit layer는 `Contact`/`ContactBook` 값과 logical order를, 컴파일 layer는 const/비공개 소유권 경계를, CLI layer는 app이 입력을 읽고 상태를 갱신해 stdout/stderr로 내보내는 수명 전체를 관찰합니다.
- 직접 확인한 실패 처리: 비공개 member가 공개가 되거나 header가 self-contained하지 않으면 컴파일 규약이, 명령 parsing/출력이 바뀌면 transcript가 실패하도록 작성되어 있습니다. 이 커밋의 범위는 contact 구성 요소이며 repository 전체 archive packaging이나 sanitizer 근거는 아직 아닙니다.
- 실행한 테스트와 결과: 미실행. 대상 dependency, translation units, CLI 테스트 준비 코드를 검사했으나 compiler/app 명령은 수행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: contact 동작, 공개 API 도형, real CLI session을 분리해 검증하는 기본 패턴을 만들었습니다.

### `4bbbfd191669` — test(contracts): 공개 include와 소유권 규칙 검증

- 중요도: **A**
- 태그: **API, TEST, OWNERSHIP**
- 원문에서 정한 역할: 라이브러리 전체의 성공·실패 공개 규약 검사를 확장합니다.
<!-- 원문 분류 요약: Expands positive and negative compile contracts across every public header and ownership boundary. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `6e78ced59357`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] library 전체 공개 headers를 대상으로 하는 positive consumer translation units와 link step을 확인하세요.
- [x] abstract 인터페이스, protected/비공개 생성, explicit conversion, const access, utility non-생성, list-backed sorting, 포인터 signature를 각각 어떤 negative file이 거부하는지 분류하세요.
- [x] negative 컴파일 테스트가 "실행 결과"가 아니라 compiler rejection 자체를 expected success로 다루는 harness를 확인하세요.
- [x] 비공개 include path 없이 archive와 공개 include만으로 규약이 성립하는지 빌드 명령을 기록하세요.
- [x] 런타임 테스트가 통과해도 API widening/encapsulation 회귀를 이 layer가 어떻게 잡는지 예시 하나를 실제 테스트와 연결하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **repository-wide 공개 API 도형/소유권 경계**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **implicit conversion, accidental mutability/생성, 비공개 include leakage**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **positive/negative 컴파일-규약 suite**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **all installed 공개 headers + archive link**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **external consumer가 허용/금지된 API 도형을 compiler 수준에서 고정**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **런타임 behavioral correctness나 leak freedom 자체는 별도 근거가 필요**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **broad 컴파일 규약**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `01271d795d58`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `Makefile`의 `PUBLIC_CPPFLAGS := -Iinclude`, `test-contract`, 공개-규약 link rule; `tests/compile/public_headers.cpp`와 구성 요소 positive units; abstract/비공개/const/explicit/list-sort 등 negative units.
- 핵심 코드 발췌 위치: `4bbbfd191669:Makefile`의 positive commands는 `-Iinclude ... -fsyntax-only`만 사용하고, negative commands는 `@! $(CXX) ... -fsyntax-only <fail.cpp>`로 rejection을 기대합니다. 공개 통합 실행 파일도 공개 include path와 archive만 링크합니다.
- 변경 전/후 차이: contact 한 영역의 컴파일 규약이 모든 installed 공개 headers와 주요 소유권/API restrictions로 확대되었습니다.
- 직접 확인한 소유권·수명·상태 관계: 컴파일 suite는 `TextBuffer` 내부 storage/implicit conversion, 포매터 abstractness, creator/builder 생성, scalar/RPN utility 생성, 런타임 type access, serializer 포인터/const 도형, batch result mutability, template const 반복자와 컨테이너 requirement를 compiler-visible 경계로 고정합니다.
- 직접 확인한 실패 처리: 허용 API가 빠지거나 비공개 header가 필요하면 positive unit이 실패하고, 금지 API가 우연히 열리면 expected-fail unit이 성공해 Make 대상이 실패합니다. 이 layer는 compiler rejection을 증명할 뿐 런타임 출력, exception 정리, leak freedom은 별도 근거가 필요합니다.
- 실행한 테스트와 결과: 미실행. 컴파일 명령과 positive/negative file set을 검사했으나 compiler는 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 공개 include만으로 허용·금지 API 도형을 repository 전체에서 compiler 계약으로 만들었습니다.

### `01271d795d58` — test(consumer): 저장소 밖 공개 library 연결 검증

- 중요도: **A**
- 태그: **API, INTEGRATION, PORTABILITY**
- 원문에서 정한 역할: 저장소 밖에서 공개 헤더와 아카이브만 사용해 외부 소비자 코드를 컴파일·실행합니다.
<!-- 원문 분류 요약: Builds and runs a consumer outside the repository using only public headers and the archive. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `4bbbfd191669`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] 임시 객체 consumer directory가 repository tree 밖에 생성되는 setup과 정리 trap을 확인하세요.
- [x] compiler include path가 exported include directory만, linker 입력이 `libcpp_foundation.a`만 사용하도록 제한되는 명령을 기록하세요.
- [x] consumer가 테스트-only helper 없이 실제 공개 objects를 사용하는 원본을 확인하세요.
- [x] working-directory assumption, 비공개 include, unarchived object가 있으면 어느 컴파일/link/run 단계에서 실패하는지 추적하세요.
- [x] external location에서 executable을 실제 run하는 단계까지 포함되는지 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **공개 archive/header external consumability**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **in-tree-only include/path/object assumptions**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **out-of-tree 컴파일/link/run consumer**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **exported include + 정적 archive + 공개 objects**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **packaging 경계가 실제 external location에서 성립함**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **모든 downstream 빌드 system/platform을 증명하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **broad 통합/release 회귀**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `9e07d3bc86d3`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/check_external_consumer.sh`, `tests/consumer/external_main.cpp`, `Makefile`의 `test-consumer`와 `test-integration`, `libcpp_foundation.a`.
- 핵심 코드 발췌 위치: `01271d795d58:tests/check_external_consumer.sh`는 `${TMPDIR:-/tmp}` 아래 `mktemp -d`로 외부 디렉터리를 만들고 consumer 원본을 복사합니다. compiler에는 `-I"$project_root/include"`, copied `main.cpp`, 전달받은 absolute archive만 주고 그 디렉터리에서 executable을 실행합니다.
- 변경 전/후 차이: repository 내부 번역 단위/link에서 실제 out-of-tree consumer 컴파일/link/run으로 packaging 검증이 확장되었습니다.
- 직접 확인한 소유권·수명·상태 관계: 정리 trap이 copied 원본과 executable을 삭제하고 임시 객체 directory를 제거합니다. consumer는 테스트 support나 원본 object를 직접 소유·링크하지 않고 exported headers와 정적 archive만 사용합니다.
- 직접 확인한 실패 처리: 비공개 include, working-directory-relative asset, archive에 빠진 symbol/object가 있으면 컴파일/link/run 중 실패합니다. compiler 존재와 argument/archive file은 스크립트가 먼저 검사합니다. 한 compiler 명령/host의 consumer일 뿐 모든 downstream 빌드 system을 증명하지는 않습니다.
- 실행한 테스트와 결과: 미실행. 스크립트의 exact 명령과 정리/run scope를 검사했으나 외부 consumer를 실제 컴파일하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 공개 headers와 정적 archive만으로 repository 밖 consumer가 실제 실행되는 packaging 경계를 검사합니다.

### `9e07d3bc86d3` — test(boundary): 변환·배치 속성과 대용량 경계 검증

- 중요도: **A**
- 태그: **TEST, DETERMINISM, EDGE**
- 원문에서 정한 역할: 고정 시드 속성 검사와 대규모 일괄 처리 스트레스 테스트를 추가합니다.
<!-- 원문 분류 요약: Adds fixed-seed scalar/RPN properties and a 4,096-job batch stress check. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `01271d795d58`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] fixed linear-congruential seed와 first-counterexample reporting을 구현한 property 테스트 driver를 찾으세요.
- [x] generated integer literals가 4-line scalar 출력과 trailing invalid byte rejection을 어떻게 반복 검증하는지 확인하세요.
- [x] bounded 실행 파일 RPN expressions를 direct computation과 비교하는 oracle 구성과 오버플로 회피 범위를 확인하세요.
- [x] 4,096-job / 200KB 이상 batch의 생성, 독립 sort oracle, record-by-record comparison, repeated serialization 검증을 추적하세요.
- [x] 결정적 seed가 breadth를 늘리면서 flaky randomness를 피하는 구조를 실제 테스트 inputs와 counterexample 출력에서 확인하세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **scalar/RPN/batch 경계 breadth와 determinism**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **generated 경계 조건s and large 할당/sort/출력 growth**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **fixed-seed property-style generation + large stress case**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **scalar parser/렌더링, RPN checked ops, batch parse/sort/serialize**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **넓은 결정적 상태-space에서 established invariants 유지**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **formal exhaustive proof나 unfixed randomness를 제공하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **결정적 property/stress**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `45e9bbfd6b75`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/property/test_boundary_properties.cpp`의 fixed LCG, scalar/RPN properties, `ExpectedJob` oracle, large batch; `tests/run_with_timeout.sh`; `Makefile`의 `test-property`.
- 핵심 코드 발췌 위치: `9e07d3bc86d3:tests/property/test_boundary_properties.cpp`는 seed `0x13579BDF`, LCG `state * 1103515245 + 12345`, first-counterexample text를 사용합니다. scalar 2,048개, bounded 실행 파일 RPN 4,096개, 4,096-job·200KB 초과 batch를 생성합니다.
- 변경 전/후 차이: hand-written boundaries를 대체하지 않고 reproducible generated breadth와 large 할당/sort/출력 growth를 추가했습니다.
- 직접 확인한 소유권·수명·상태 관계: scalar는 exact int line·4 newlines·repeatability·trailing `x` rejection/empty 출력을 확인합니다. RPN은 오버플로를 피한 bounded direct arithmetic oracle와 비교하고, batch는 독립 `ExpectedJob` vector를 `std::sort`한 뒤 record-by-record 및 repeated bytes를 비교합니다.
- 직접 확인한 실패 처리: 실패 시 fixed seed와 첫 counterexample을 출력해 재현 가능하게 합니다. timeout wrapper는 hang/과도한 런타임을 별도 실패로 만듭니다. fixed finite sample이므로 formal exhaustive proof나 임의 seed 다양성은 제공하지 않습니다.
- 실행한 테스트와 결과: 미실행. generator, oracle, counts, timeout 대상을 검사했으나 property 실행 파일은 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: 고정 seed 속성 검사와 4,096-job stress로 결정적 검증 폭을 넓혔습니다.

### `45e9bbfd6b75` — build(check): sanitizer와 portable 검사 계층 구성

- 중요도: **A**
- 태그: **ARCH, PORTABILITY, TEST**
- 원문에서 정한 역할: 빌드·이식성·플랫폼 검사를 분리하고 ASan/UBSan의 역할을 구분합니다.
<!-- 원문 분류 요약: Separates build, portable, and platform verification while adding ASan and UBSan layers. -->

#### 핵심 설계 / 실패 경계 확인
- [x] 필요하면 직전 관련 SHA `9e07d3bc86d3`와 비교하여 책임, 상태 변경 순서, 테스트 경계가 어떻게 달라졌는지 확인하세요.
- [x] Makefile/check targets에서 빌드, portable, platform layer가 어떤 dependency graph로 재구성되는지 그리세요.
- [x] clean rebuild, 테스트, compiler contracts, archive, external consumer, properties, UBSan 중 portable baseline에 포함되는 항목을 실제 대상 prerequisite로 확인하세요.
- [x] ASan과 host-specific release inspection이 별도 platform checks로 분리되는 이유를 prerequisites/commands에서 확인하세요.
- [x] aggregate 대상이 unavailable host tool 때문에 portable checks까지 생략하지 않도록 구성된 브랜치/gating을 기록하세요.
- [x] reconstruction, incremental no-op, 결정적 artifact 검증이 어느 layer에서 실행되는지 찾으세요.
- [x] 이 커밋의 변경이 어떤 불변식/실패 처리/API 경계를 강화하는지 실제 코드와 테스트를 연결해 적으세요.
- [x] 이 커밋의 보장 범위를 넘는 항목은 무엇인지 원문에 근거해 별도로 적으세요.


#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `ab441fa8737c`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `Makefile`의 `test-asan`, `test-ubsan`, `test-sanitize`, `check-build`, `check-portable`, `check-platform`, `check`; archive/dependency/leak/determinism scripts.
- 핵심 코드 발췌 위치: `45e9bbfd6b75:Makefile`에서 `check-build`는 diff check, clean rebuild, `test`, 결정적 CLI 두 번, incremental `make -q all`을 수행합니다. `check-portable`은 여기에 UBSan을 추가하고 `check-platform`은 archive/dependency/leak scripts만 실행합니다.
- 변경 전/후 차이: 하나의 검사 묶음을 portable baseline과 host/tool-dependent release inspection으로 분리하고 ASan/UBSan 실행 파일/targets를 별도로 만들었습니다.
- 직접 확인한 소유권·수명·상태 관계: `test` 아래 unit, 결정적 실패 주입, no-elide, 컴파일 규약, CLI/공개 통합, external consumer, property 테스트가 모입니다. UBSan은 `check-portable` dependency이고, ASan은 standalone `test-asan`/`test-sanitize`로 남아 host capability에 따라 선택됩니다.
- 직접 확인한 실패 처리: portable checks는 host-specific `ar`/dependency/leak inspection 실패 때문에 생략되지 않고, timeout wrapper와 sanitizer halt options가 UB/memory error를 대상 실패로 만듭니다. **관찰된 차이:** scaffold 문구와 달리이 SHA의 `check-platform`에는 ASan이 포함되지 않으며 aggregate `check`도 ASan을 호출하지 않습니다. ASan matrix 실행은 후속 CI 커밋에서 추가됩니다.
- 실행한 테스트와 결과: 미실행. Make dependency graph와 commands를 검사했으나 sanitizer/host utility를 실행하지 않았습니다.
- 이 커밋을 한 문장으로 설명: portable 회귀·UBSan과 host-specific release inspection을 분리하고 ASan을 별도 capability 대상으로 제공했습니다.

### `ab441fa8737c` — test(portability): 지원 LP64 데이터 모델 검증

- 중요도: **B**
- 태그: **PORTABILITY, API**
- 원문에서 정한 역할: 지원하는 LP64 ABI 전제를 실행 가능한 검사로 만듭니다.
<!-- 원문 분류 요약: Adds compile-time LP64 data-model assertions. -->

#### 개발 흐름에서 확인할 구현 역할
- [x] 직전 관련 SHA `45e9bbfd6b75`와의 차이 중 이 개발 흐름의 흐름에 필요한 부분만 확인하세요.
- [x] CHAR_BIT/short/int/long/포인터/size_t 크기를 컴파일 시점에 assert하는 번역 단위를 확인하세요.
- [x] expected data model이 8-bit byte, 2-byte short, 4-byte int, 8-byte long/포인터/size_t임을 테스트 expression에서 확인하세요.
- [x] 이 check가 scalar limits, RPN `long`, 할당 size, 포인터 token 중 어떤 구성 요소 assumptions와 연결되는지 원본 references를 찾아 적으세요.
- [x] 지원하지 않는 ABI에서 런타임 오동작 대신 컴파일 시점 실패로 끝나는 harness 동작을 확인하세요.
- [x] 이 커밋이 다음 관련 커밋의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **supported LP64 data model**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **unsupported ABI silently compiling**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **컴파일 시점 portability 검사문**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **공개/platform 빌드 경계 before 런타임**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **declared ABI assumptions을 만족하지 않으면 조기 실패**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **LP64 내부의 모든 platform 차이를 증명하지는 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **컴파일 시점 portability 규약**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 다음 관련 커밋과 연결
- 다음 개발 흐름 SHA `50565bd67e03`를 읽기 전에, 이 SHA가 남긴 보장과 미해결 실패 경계를 2~4줄로 적으세요.

#### 학습자 기록
- 확인한 파일/심볼: `tests/portability/test_data_model.cpp`; `Makefile`의 `DATA_MODEL_BIN`, `check-data-model`, `check-build` dependency.
- 핵심 코드 발췌 위치: `ab441fa8737c:tests/portability/test_data_model.cpp`는 `CHAR_BIT == 8`, `sizeof(short)==2`, `sizeof(int)==4`, `sizeof(long)==8`, 포인터/`size_t` 8을 `const bool lp64`로 계산하고 false면 diagnostic 후 `return 1` 합니다.
- 변경 전/후 차이: 지원 data model 가정이 문서상의 전제에서 executable gate로 추가되어 `check-build`가 실제 실행 파일을 실행하게 되었습니다.
- 직접 확인한 소유권·수명·상태 관계: 이 check는 scalar/RPN의 `long` 범위, `size_t`/포인터 크기 등 빌드가 전제한 host 표현을 production 실행 전 verification 단계에서 판정합니다.
- 직접 확인한 실패 처리: unsupported model은 테스트 executable이 성공적으로 컴파일된 뒤 런타임에 exit 1로 끝납니다. **관찰된 차이:** scaffold에는 컴파일 시점 검사문/컴파일 시점 실패로 고정되어 있지만 해당 SHA의 실제 코드는 정적/typedef 검사문이 아니라 런타임 `bool`과 exit status를 사용합니다. 고정 scaffold text는 유지하고 이 불일치를 여기에 기록합니다.
- 실행한 테스트와 결과: 미실행. 원본과 Make 대상을 검사했으나 data-model 실행 파일을 컴파일/run하지 않았습니다.
- 이 커밋을 한 문장으로 설명: LP64 전제를 executable 런타임 gate로 만들었으며, scaffold의 컴파일 시점 설명과 실제 구현에는 차이가 있습니다.

### `50565bd67e03` — ci: 지원 compiler와 platform matrix 검증

- 중요도: **B**
- 태그: **PORTABILITY, TEST**
- 원문에서 정한 역할: 확립한 지원 조건을 GCC/Clang, Linux/macOS 조합에서 실행합니다.
<!-- 원문 분류 요약: Adds GCC/Clang Linux and Clang macOS CI with sanitizer coverage. -->

#### 개발 흐름에서 확인할 구현 역할
- [x] 직전 관련 SHA `ab441fa8737c`와의 차이 중 이 개발 흐름의 흐름에 필요한 부분만 확인하세요.
- [x] CI matrix에서 GCC/Clang Linux와 Clang macOS 조합을 실제 configuration으로 확인하세요.
- [x] 각 job이 어떤 established 빌드/check 대상을 실행하는지 기록하세요.
- [x] matrix fail-fast disabled 설정이 한 job 실패 시 다른 근거를 계속 수집하도록 하는지 확인하세요.
- [x] UBSan은 어디서 공통 실행되고 ASan은 어느 host로 제한되는지 configuration을 확인하세요.
- [x] minimal permissions와 explicit timeout가 실제 workflow에 선언되어 있는지 확인하세요.
- [x] 이 커밋이 다음 관련 커밋의 전제가 되는 상태/계약을 한 문단으로 기록하세요.

#### 테스트 커밋에서 확인할 내용
- 대상 실제 코드의 불변식: 원문이 확정한 방향은 **supported compiler/platform matrix**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 재현 실패 / 경계: 원문이 확정한 방향은 **compiler/OS-specific 회귀**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 테스트 방식: 원문이 확정한 방향은 **CI matrix with sanitizer/check targets**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 통과하는 실제 실행 경로: 원문이 확정한 방향은 **established 빌드 and verification stack**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하는 것: 원문이 확정한 방향은 **GCC/Clang Linux와 Clang macOS에서 지속적 근거 확보**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 이 테스트가 증명하지 않는 것: 원문이 확정한 방향은 **matrix 밖 compiler/OS support를 의미하지 않음**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.
- 성격: 원문이 확정한 방향은 **broad CI verification**입니다. 실제 테스트 코드/테스트 준비 코드를 읽고 구체적인 파일·case·검사문을 기록하세요.

#### 학습자 기록
- 확인한 파일/심볼: `.github/workflows/ci.yml`; `Makefile`의 `check-build`, `test-ubsan`, `test-asan`, `check-data-model`.
- 핵심 코드 발췌 위치: `50565bd67e03:.github/workflows/ci.yml`은 ubuntu-22.04 GCC, ubuntu-22.04 Clang, macOS latest Clang matrix를 정의하고 `fail-fast: false`, `timeout-minutes: 30`, `contents: read`를 설정합니다. 모든 job은 `make check-build`, UBSan을 실행하고 ASan은 Linux 두 job만 실행합니다.
- 변경 전/후 차이: local executable checks를 compiler/OS matrix에서 반복 실행하는 workflow가 추가되었습니다. `check-build`에는 런타임 LP64 gate가 포함됩니다.
- 직접 확인한 소유권·수명·상태 관계: 각 clean GitHub 체크아웃이 독립 workspace를 소유하고 selected compiler를 `CXX`로 Make targets에 전달합니다. matrix job별 artifact/상태는 공유하지 않으므로 compiler/OS-specific 실패가 다른 근거를 덮지 않습니다.
- 직접 확인한 실패 처리: compiler 선택, 빌드/회귀/data-model, UBSan, Linux ASan 중 하나가 nonzero면 해당 job이 실패합니다. `fail-fast: false`라 다른 matrix job은 계속됩니다. **보장 경계:** workflow push trigger는 `main`만이며 PR에도 실행됩니다. 또한 CI는 `make check-platform`/`make check`를 호출하지 않아 archive/dependency/leak scripts는 이 matrix 근거에 포함되지 않습니다. 실제 workflow run 결과는 검사하지 않았습니다.
- 실행한 테스트와 결과: 미실행. workflow configuration과 호출 대상만 검사했으며 GitHub Actions run 성공을 주장하지 않습니다.
- 이 커밋을 한 문장으로 설명: GCC/Clang Linux와 Clang macOS에서 빌드·회귀·LP64·UBSan 및 Linux ASan을 반복하도록 구성했습니다.

## 불변식 변화 기록

| SHA | 원문에서 확정된 불변식 변화 | 해당 SHA에서 직접 확인한 코드 근거 | 아직 남은 위험/미보장 |
| --- | --- | --- | --- | --- |
| `6e78ced59357` | unit + 컴파일-규약 + CLI의 multi-layer verification pattern 시작 | contact unit, repeated 공개 헤더 컴파일, 비공개-access expected 실패, exact CLI session 테스트 준비 코드가 동작/API/프로세스 층을 분리합니다. | repository 전체 공개 규약, external packaging, sanitizer/ABI matrix는 아직 없습니다. |
| `4bbbfd191669` | library 전체 공개 API/소유권 negative contracts로 확대 | `-Iinclude`만 사용하는 positive syntax units와 expected-fail API/소유권 units, 공개 archive link rule을 확인했습니다. | 런타임 동작과 leak/exception 정리는 compiler 규약만으로 증명되지 않습니다. |
| `01271d795d58` | repository 밖 consumer로 packaging/공개 헤더 isolation 검증 | 외부 임시 객체 directory에서 copied consumer를 공개 include + absolute 정적 archive만으로 컴파일/link/run하고 trap으로 정리합니다. | 한 host/compiler consumer이며 모든 downstream 빌드 system/platform을 포괄하지 않습니다. |
| `9e07d3bc86d3` | fixed-seed properties와 large-batch 결정적 stress 추가 | fixed LCG seed, scalar 2,048, RPN 4,096, 4,096-job large batch와 first-counterexample reporting을 확인했습니다. | 유한 fixed sample이므로 exhaustive/formal proof는 아닙니다. |
| `45e9bbfd6b75` | portable baseline과 platform checks 분리, ASan/UBSan prerequisite 구분 | `check-build`/`check-portable`/`check-platform` dependency와 UBSan/ASan standalone targets를 확인했습니다. | 실제 `check-platform`/aggregate `check`에는 ASan이 포함되지 않아 scaffold 표현과 차이가 있습니다. |
| `ab441fa8737c` | LP64 data model을 컴파일 시점 검사문으로 executable claim화 | LP64 sizes를 `const bool`로 검사하고 false에서 exit 1인 executable과 `check-data-model` 대상을 확인했습니다. | 컴파일 시점 검사문이 아니라 런타임 gate라는 scaffold/구현 불일치가 있으며 LP64 내부 차이는 남습니다. |
| `50565bd67e03` | GCC/Clang, Linux/macOS matrix에서 established claims 연속 실행 | ubuntu GCC/Clang, macOS Clang matrix가 check-빌드·UBSan·Linux ASan을 fail-fast false로 실행하도록 구성됩니다. | push는 main만이며 check-platform/archive/dependency/leak와 matrix 밖 host/compiler는 지원 증거가 아닙니다. |

## 문제 → 수정 → 검증 연결

- 이 개발 흐름은 한 개의 bug fix chain보다 verification blind spot을 단계적으로 줄이는 흐름입니다. 각 layer가 이전 layer가 증명하지 못한 무엇을 추가하는지 학습자가 기록하세요.

### 학습자 연결 기록
- 최초 위험/맹점: in-tree unit 테스트가 통과해도 공개 헤더가 비공개 include에 의존하거나, archive가 빠진 object를 숨기거나, API가 우연히 넓어지거나, 특정 compiler/ABI에서만 실패할 수 있습니다.
- 이를 드러낸 실제 실패 또는 테스트 gap: 런타임 출력은 abstractness/const/비공개 도형을 보지 못하고, in-tree link는 relative path와 loose objects를 숨기며, hand-written 테스트 준비 코드는 상태-space breadth와 large growth를 제한합니다. 단일 host 실행은 sanitizer/ABI/compiler 차이를 보여 주지 못합니다.
- 수정/강화된 결정: 동작, positive/negative 컴파일, CLI, out-of-tree consumer, fixed-seed property/stress, portable/host 대상, LP64 gate, CI matrix를 서로 다른 실패 조건으로 누적합니다.
- 해당 코드 위치: `6e78ced59357`~`50565bd67e03`의 `Makefile`, `tests/compile/`, `tests/check_external_consumer.sh`, `tests/property/test_boundary_properties.cpp`, `tests/portability/test_data_model.cpp`, `.github/workflows/ci.yml`.
- 이를 고정하는 회귀/근거: 각 layer 자체가 이전 layer가 볼 수 없는 실패를 nonzero 빌드·테스트/컴파일/run status로 드러냅니다. 다만 data-model check는 scaffold 설명과 달리 런타임이며 CI는 check-platform을 실행하지 않는다는 한계를 함께 기록합니다.

## Verification responsibility 변화

- Source 기준 흐름: 동작 → API 도형 → external packaging → breadth/stress → portable/platform separation → ABI → compiler/platform matrix로 검증 책임이 확장됩니다.
- [x] 각 layer가 실행되는 빌드/CI 대상과 prerequisite를 실제 코드에서 연결하세요.

### 코드 검사로 복원한 변화

1. `6e78ced59357`: verification 책임이 contact 동작에서 컴파일 시점 API 도형과 real 프로세스 transcript로 확장됩니다.
2. `4bbbfd191669`: 공개-only positive/negative compiler 계약이 library 전체 API와 소유권 restrictions를 다룹니다.
3. `01271d795d58`: packaging 책임이 repository 밖 컴파일/link/run으로 이동해 in-tree assumptions를 제거합니다.
4. `9e07d3bc86d3`: fixed-seed generated inputs와 large batch가 결정적 breadth와 growth를 검증합니다.
5. `45e9bbfd6b75`: portable baseline/UBSan과 host-specific archive/dependency/leak checks, standalone ASan capability가 분리됩니다.
6. `ab441fa8737c`: LP64 가정이 런타임 executable gate가 됩니다. scaffold의 컴파일 시점 표현과 실제 코드 차이를 명시했습니다.
7. `50565bd67e03`: clean Linux/macOS compiler jobs가 빌드/회귀/data-model/UBSan 및 Linux ASan을 반복합니다. check-platform은 CI 범위 밖입니다.

## 개발 흐름의 최종 상태

- 원문이 확정한 최종 흐름: `unit behavior → compile-time public contract → process fixture → external consumer → deterministic property/stress → portable/platform check layers → ABI gate → CI matrix`
- [x] 마지막 개발 흐름 SHA 시점에서 실제 type/function 호출 관계를 사용해 위 흐름을 다시 그리세요.
- [x] 개발 흐름 시작 시점과 비교해 새로 보장되는 불변식을 정리하세요.
- [x] 원문이 보장하지 않는 영역이나 외부 side effect/스트림 위치 등 남는 경계를 실제 코드 근거로 적으세요.

### 완성된 개발 흐름 해석

마지막 개발 흐름 SHA 기준으로 verification stack은 unit/실패/no-elide 테스트, 공개-only positive/negative compiler contracts, real CLI/공개 통합, out-of-tree consumer, fixed-seed property/stress, 결정적 rebuild/출력, 런타임 LP64 gate, UBSan/ASan targets, compiler/OS CI matrix로 구성됩니다. 각 층은 동일한 "테스트 통과"를 반복하는 것이 아니라 원본 동작, API 도형, packaging, 상태-space breadth, UB/memory fault, ABI, toolchain variation을 서로 다른 방식으로 관찰합니다.

시작 시점과 비교하면 support claim이 in-tree contact 동작에서 exported archive와 특정 GCC/Clang LP64 hosts까지 넓어졌습니다. 하지만 실제 증거는 구성 코드를 검사한 것이며 실행 결과는 없습니다. 또 data-model check는 컴파일 시점이 아니라 런타임이고, CI는 host-specific `check-platform`을 실행하지 않으며 push trigger는 main에 제한됩니다. 따라서 matrix 밖 compiler/OS와 archive/dependency/leak의 CI 지속 실행은 주장할 수 없습니다.

## 최종 설계와 실행 흐름

다음 항목은 학습자가 실제 커밋 코드를 읽은 뒤 완성합니다. 완성형 정답을 원문에 없는 내용을 바탕으로 추정해 채우지 않습니다.

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

- 실제 caller → callee 흐름: 원본/빌드 change → unit/실패/no-elide → 공개 positive/negative 컴파일 → CLI/공개 통합 → external consumer → fixed-seed property/stress → `check-build`/LP64 gate → UBSan/ASan → GitHub Actions compiler/OS jobs.
- 핵심 상태 필드: Make 대상 dependency graph, 컴파일 translation units, 테스트 준비 코드 expected bytes, property `random_state`/`first_failure`, 런타임 `lp64` bool, CI matrix entries and conditions.
- 자원 소유자 / 비소유 참조: each 테스트 프로세스 owns its 임시 객체 objects; external-consumer 스크립트 owns and traps a 임시 객체 directory; CI jobs own isolated checkouts. Public consumers borrow only installed headers/API and link the archive.
- 상태 확정 지점: 각 verification layer는 compiler/linker/프로세스 exit status와 exact 검사문/byte comparison으로 claim을 승인합니다. CI job은 모든 configured step success일 때만 green입니다.
- 정리 path: 컴파일/run 실패는 nonzero로 상위 Make/CI를 중단하고 scripts/traps가 임시 객체 files를 제거합니다. sanitizer는 configured halt option으로 첫 detected fault를 실패로 만듭니다.
- 최종 불변식 설명: supportable claim은 단일 unit result가 아니라 공개 API isolation, external packaging, 결정적 동작, LP64 런타임 gate, selected sanitizer와 compiler/OS configuration에서 실행 가능하도록 코드화되어 있습니다. 구성 밖 platform과 실행되지 않은 대상은 보장으로 확대하지 않습니다.

### 실행 검증 범위

이 문서의 구현·테스트 설명은 지정 SHA의 diff와 당시 파일을 GitHub 저장소에서 직접 검사해 복원했습니다. 현재 컨테이너에서는 GitHub 체크아웃에 필요한 네트워크 연결이 차단되어 빌드·테스트 명령를 실행하지 못했습니다. 따라서 아래 체크 표시는 코드·테스트 구현을 확인했다는 의미이며, 실행 결과를 의미하지 않습니다.

## 학습 완료 자가 점검

- [x] 커밋 목록의 SHA/순서를 그대로 따라 모든 관련 코드 tree를 확인했습니다.
- [x] 최종 HEAD를 과거 커밋 설명에 소급해서 사용하지 않았습니다.
- [x] S/A/B 중요도에 맞는 깊이로 코드/테스트 근거를 채웠습니다.
- [x] 원문이 확정한 불변식과 제가 실제 코드에서 확인한 증거를 구분했습니다.
- [x] 실패 처리에서 상태 변경 전후와 정리 소유자를 설명할 수 있습니다.
- [x] 테스트 커밋마다 실제 코드의 불변식, 검증 방식, 실제 실행 경로, 증명/비증명 범위를 구분했습니다.
- [x] 개발 흐름 마지막 상태를 커밋 history에 근거해 처음부터 끝까지 설명할 수 있습니다.
===== END FILE: 08-verification-supported-release-claims.md =====

===== BEGIN FILE: README.md =====
# cpp-foundation 개발 흐름 학습 안내

## 목적

이 디렉터리는 `commit-importance.md`에 정의된 개발 흐름을 따라 실제 커밋 history와 각 SHA의 코드를 직접 읽으며 프로젝트의 설계 → 구현 → 실패 → 수정 → 검증 과정을 복원하기 위한 학습 골격입니다.

완성형 프로젝트 해설서가 아닙니다. 미리 작성된 내용은 원본에서 확정된 thread 구조, 커밋 메타데이터, 역할, 불변식/실패 방향뿐이며, 실제 구현 해석과 코드 증거는 학습자가 채웁니다.

## 권장 학습 순서

1. [`01-direct-ownership-failure-safe-value.md`](01-direct-ownership-failure-safe-value.md)
2. [`02-polymorphic-cloning-owning-aggregate.md`](02-polymorphic-cloning-owning-aggregate.md)
3. [`03-factory-transaction-boundary.md`](03-factory-transaction-boundary.md)
4. [`04-scalar-text-target-projection.md`](04-scalar-text-target-projection.md)
5. [`05-checked-rpn-undefined-arithmetic.md`](05-checked-rpn-undefined-arithmetic.md)
6. [`06-generic-containers-transactional-batch.md`](06-generic-containers-transactional-batch.md)
7. [`07-contactbook-replacement-guarantee.md`](07-contactbook-replacement-guarantee.md)
8. [`08-verification-supported-release-claims.md`](08-verification-supported-release-claims.md)

순서는 원본의 개발 흐름 순서를 그대로 따릅니다. 동일 커밋이 여러 thread에 등장하는 경우 제거하지 않고 각 thread 관점에서 다시 확인합니다.

## 개발 흐름 문서 사용법

각 문서에서 먼저 개발 흐름 목표, 핵심 질문, 완료 기준, 커밋 목록을 읽습니다. 이후 커밋을 map 순서대로 진행합니다.

각 커밋에서는 다음 원칙을 지킵니다.

- 먼저 해당 SHA를 체크아웃하거나 `git show <sha>`로 정확한 시점의 diff와 파일을 확인합니다.
- 문서가 지목한 type/function/상태/테스트를 해당 SHA에서 직접 찾습니다.
- 필요하면 문서가 지정한 직전 관련 SHA와 비교합니다.
- 원본에 없는 파일명, 함수명, 소유권 관계를 추정해 채우지 않습니다.
- 학습 기록에는 실제 확인한 경로, 심볼, 코드 라인, 테스트 case를 근거로 남깁니다.

## 해당 SHA 코드 확인 원칙

**최종 HEAD의 코드를 과거 커밋 설명에 소급해서 사용하지 않습니다.**

후속 refactor나 fix가 이미 적용된 HEAD를 기준으로 과거 설계를 설명하면 실패 원인과 수정 경계가 사라집니다. 반드시 학습 대상 커밋의 tree를 기준으로 확인하고, 전후 비교가 필요할 때만 관련 SHA끼리 비교합니다.

권장 최소 기록 형식은 다음과 같습니다.

```text
SHA:
Path:
Symbol / test case:
직전 관련 SHA:
확인한 state / ownership / failure path:
이 코드가 증명하는 invariant:
```

## 중요도별 학습 깊이

### S

프로젝트의 핵심 architecture/불변식입니다. 직전 상태, problem, 실패 가능성, 결정, 핵심 코드, 소유권/lifecycle/상태 변화, 실패 처리, 보장/비보장 범위, 후속 fix/테스트까지 추적합니다.

### A

주요 구성 요소, 경계, 실패 처리, 통합 point입니다. 핵심 코드와 설계 판단, 전후 상태 변화, 테스트 근거까지 확인합니다.

### B

개발 흐름에서 맡는 구현 역할과 필요한 상태/API 변화를 확인합니다. S/A와 동일한 깊이의 분석란을 억지로 만들지 않습니다.

### C

개발 흐름 이해에 필요한 맥락으로만 사용합니다. 원본에 C 커밋이 개발 흐름에 포함되지 않았다면 별도 학습 항목을 추가하지 않습니다.

## 실제 코드 삽입 기준

문서에 코드를 붙일 때는 설명용으로 재작성하지 말고 **해당 SHA의 실제 코드**만 사용합니다.

- 핵심 불변식을 직접 만드는 상태 필드나 함수
- 소유권 transfer, 복제, delete, swap, 후보 반영 지점
- 실패/error 브랜치와 정리 path
- parser 경계, 오버플로 precondition, 스트림-상태 판정
- 실제 실행 경로를 실제 통과하는 회귀 테스트
- fix 전/후 차이를 보여 주는 최소 코드

코드 발췌에는 SHA, path, symbol을 함께 적습니다. 긴 파일 전체를 복사하지 않습니다.

## 테스트 커밋 학습 방법

테스트 커밋에서는 단순히 "테스트가 통과했다"고 기록하지 않습니다. 반드시 다음을 구분합니다.

- 어떤 실제 코드의 불변식를 대상으로 하는가
- 어떤 실패 또는 경계를 재현하는가
- 어떤 테스트 방식을 사용하는가
- 실제 어떤 production 코드 path를 통과하는가
- 이 테스트가 증명하는 것
- 이 테스트가 증명하지 않는 것
- broad 통합인지 결정적 회귀 테스트/실패 주입인지
- 후속 변경에서 어떤 회귀를 막는가

실행 결과는 사용한 compiler/빌드 대상과 함께 학습자가 직접 기록합니다.

## 문서 완료 기준

개발 흐름 하나는 다음 조건을 모두 만족해야 완료입니다.

- 커밋 목록의 모든 SHA를 원문 순서대로 확인했습니다.
- S/A/B/C 중요도에 맞는 깊이로 실제 코드 근거를 채웠습니다.
- Invariant ledger에 introduction/strengthening/실패/fix/테스트 흐름을 실제 코드 증거와 연결했습니다.
- fix 커밋은 기존 가정 → 실패/risk → root cause → 수정 결정 → 실제 수정 코드 → 회귀 테스트가 연결되어 있습니다.
- 테스트 커밋은 실제 코드의 불변식, 실패 경계, 검증 방식, 실제 실행 경로, 증명/비증명 범위가 구분되어 있습니다.
- 소유권/상태/responsibility 변화가 의미 있는 thread에서는 transition을 직접 설명할 수 있습니다.
- 개발 흐름 최종 상태와 execution/architecture flow를 최종 HEAD가 아닌 해당 커밋 순서를 근거로 설명할 수 있습니다.
===== END FILE: README.md =====

