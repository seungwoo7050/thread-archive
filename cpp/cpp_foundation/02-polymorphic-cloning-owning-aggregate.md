# Thread: 다형 객체를 복제해 소유하는 aggregate 만들기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

이 Thread는 `Formatter`의 동적 타입을 보존하면서 여러 formatter를 순서대로 소유하는 `FormatPipeline`을 만듭니다. 단순한 `Formatter *` 배열이 아니라 다음 조건을 만족하는 owning aggregate가 목표입니다.

- base pointer로 삭제해도 실제 derived destructor가 호출됩니다.
- pipeline에 넘긴 formatter는 빌려 쓰지 않고 `clone()`한 별도 객체로 소유합니다.
- pipeline 복사는 각 step의 동적 타입과 내부 상태까지 다시 복제합니다.
- copy construction 도중 N번째 clone이 실패하면 앞서 만든 clone을 직접 정리합니다.
- assignment 도중 실패하면 destination의 기존 pipeline은 그대로 남습니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `835d87865762` | `feat(format): 다형적 formatter 인터페이스 정의` | S | `ARCH, POLYMORPHISM, OWNERSHIP` | virtual destructor와 `clone/apply/name` 계약 확립 |
| `62ed45f8adf9` | `feat(format): formatter 소유 pipeline 구현` | S | `ARCH, POLYMORPHISM, OWNERSHIP` | borrowed formatter를 owned clone으로 바꾸는 aggregate 구현 |
| `bf4d9bed705c` | `feat(format): pipeline 깊은 복사 구현` | S | `OWNERSHIP, EXCEPTION, POLYMORPHISM` | 실패한 copy construction의 부분 clone 정리와 strong assignment 구현 |
| `0427713637b8` | `test(format): 가상 소멸·추상 계약·CLI 검증` | A | `API, TEST, POLYMORPHISM` | type contract, virtual deletion, 정상 복사와 CLI를 다층 검증 |
| `2c99290b9268` | `test(format): 복제 실패 뒤 부분 객체 정리 검증` | A | `TEST, EXCEPTION, POLYMORPHISM` | clone 실패 위치별 copy/assignment 회귀 검증 |

## `835d87865762` — `feat(format): 다형적 formatter 인터페이스 정의`

`Formatter`는 직접 만들 수 없는 abstract base이며, 소유권과 실행에 필요한 세 동작을 virtual function으로 고정합니다.

```cpp
class Formatter
{
public:
    virtual ~Formatter();
    virtual Formatter *clone() const = 0;
    virtual TextBuffer apply(const TextBuffer &input) const = 0;
    virtual const char *name() const = 0;
};
```

각 함수의 역할은 겹치지 않습니다.

| 함수 | 호출자가 얻는 것 | ownership 의미 |
| --- | --- | --- |
| `clone()` | 같은 동적 타입과 상태를 가진 새 객체 | 반환 pointer의 삭제 책임이 호출자에게 이전됨 |
| `apply()` | 입력에 formatter를 적용한 새 `TextBuffer` | formatter 자체는 수정하지 않음 |
| `name()` | formatter 종류를 나타내는 비소유 문자열 | lifetime은 formatter가 보장 |
| virtual destructor | base pointer 삭제 시 derived destructor까지 실행 | owning container가 `delete Formatter *`를 안전하게 수행 |

`UppercaseFormatter`, `PrefixFormatter`, `SuffixFormatter`는 모두 `new Derived(*this)` 형태로 clone합니다. Prefix와 suffix formatter는 내부에 `TextBuffer`를 값으로 보관하므로, 앞 Thread에서 확립한 deep-copy semantics가 그대로 동적 상태 복제에 사용됩니다.

```cpp
Formatter *PrefixFormatter::clone() const
{
    return new PrefixFormatter(*this);
}
```

Uppercase 구현은 `std::toupper`에 plain signed `char`를 직접 전달하지 않고 unsigned byte로 변환합니다. 음수 `char`가 ctype 함수에 들어가는 undefined behavior를 피하기 위한 경계 처리입니다.

```cpp
result.at(index) = static_cast<char>(
    std::toupper(static_cast<unsigned char>(result.at(index))));
```

이 커밋만으로는 clone의 owner가 정해지지 않습니다. 반환된 pointer를 누가, 언제 삭제하는지는 다음 commit의 `FormatPipeline`이 담당합니다.

## `62ed45f8adf9` — `feat(format): formatter 소유 pipeline 구현`

`FormatPipeline`은 최대 8개의 `Formatter *`를 보관합니다. 외부 formatter의 주소를 저장하지 않고 `append()` 안에서 clone합니다.

```cpp
void FormatPipeline::append(const Formatter &formatter)
{
    if (size_ == max_steps)
        throw std::length_error("format pipeline capacity");
    steps_[size_] = formatter.clone();
    ++size_;
}
```

여기서 중요한 publish 순서는 `clone()` 성공 후 pointer 저장, 그 뒤 `size_` 증가입니다.

```text
borrowed const Formatter&
  -> formatter.clone()             # allocation/derived copy, 예외 가능
  -> steps_[size_] = returned ptr  # pipeline ownership 시작
  -> ++size_                       # valid owned range 공개
```

`clone()`이 예외를 던지면 `size_`가 증가하지 않으므로 destructor가 잘못된 slot을 소유한 것으로 해석하지 않습니다. 반대로 정상 반환한 pointer는 `[0, size_)` 범위에 들어가며, pipeline destructor가 정확히 한 번 삭제합니다.

```cpp
FormatPipeline::~FormatPipeline()
{
    for (std::size_t index = 0; index < size_; ++index)
        delete steps_[index];
}
```

`apply()`는 입력을 복사한 뒤 step 순서대로 결과를 넘깁니다.

```text
input copy
  -> steps_[0]->apply
  -> steps_[1]->apply
  -> ...
  -> steps_[size_-1]->apply
```

빈 pipeline은 입력 copy를 그대로 반환하므로 identity입니다. Pipeline의 step들은 실행 중 바뀌지 않으며, derived formatter의 mutation도 parent object에 노출되지 않습니다.

이 시점의 제한도 분명합니다. Copy constructor와 assignment가 private이어서 owning aggregate 자체를 값처럼 복사할 수 없습니다.

## `bf4d9bed705c` — `feat(format): pipeline 깊은 복사 구현`

Pipeline copy는 source의 pointer 값을 복사하면 안 됩니다. 그렇게 하면 두 pipeline이 같은 formatter를 삭제합니다. 따라서 각 step에 다시 `append()`를 적용해 polymorphic clone을 만들어야 합니다.

문제는 N번째 clone이 실패하는 경우입니다. C++에서 constructor body가 예외로 끝나면 **그 객체의 destructor는 호출되지 않습니다.** 이미 완성된 data member의 destructor는 호출되지만, 이 class가 raw pointer 배열에 넣은 pointee는 자동으로 삭제되지 않습니다.

이 commit은 copy constructor 안에서 성공한 prefix를 직접 정리합니다.

```cpp
FormatPipeline::FormatPipeline(const FormatPipeline &other)
    : size_(0)
{
    std::size_t index;

    for (index = 0; index < max_steps; ++index)
        steps_[index] = 0;
    try
    {
        for (index = 0; index < other.size_; ++index)
            append(*other.steps_[index]);
    }
    catch (...)
    {
        for (index = 0; index < size_; ++index)
            delete steps_[index];
        throw;
    }
}
```

정리 범위는 전체 배열이 아니라 실제로 ownership이 공개된 `[0, size_)`입니다. `append()`가 clone 성공 뒤에만 `size_`를 증가시키므로 이 범위가 곧 유효 pointer 집합입니다.

Assignment는 완전한 copy를 먼저 만든 뒤 swap합니다.

```cpp
FormatPipeline &FormatPipeline::operator=(const FormatPipeline &other)
{
    FormatPipeline copy(other);

    swap(copy);
    return *this;
}
```

두 failure path의 정리 주체는 서로 다릅니다.

| 경로 | 실패 시점 | 정리 주체 | destination 상태 |
| --- | --- | --- | --- |
| copy construction | N번째 clone | constructor catch가 성공한 clone 삭제 | 새 객체가 존재하지 않음 |
| assignment | 임시 copy의 N번째 clone | 임시 copy constructor의 catch | 기존 destination 그대로 |
| assignment 성공 | 모든 clone 완료 후 | swap 뒤 임시 객체 destructor가 이전 destination 삭제 | deep-copied source 상태 |

Self-assignment도 임시 deep copy를 만들 수 있지만 aliasing이나 double delete는 없습니다. 이 구현은 성능보다 같은 단일 경로로 correctness를 유지하는 선택입니다.

## `0427713637b8` — `test(format): 가상 소멸·추상 계약·CLI 검증`

이 commit은 하나의 “unit test 추가”보다 넓습니다.

### Compile-time contract

- public header를 두 번 include해 include guard와 독립 선언을 확인합니다.
- `Formatter formatter;`가 compile되지 않아 abstract contract를 확인합니다.
- public API만으로 formatter와 pipeline을 사용할 수 있는지 확인합니다.

### Virtual lifetime과 dynamic state

`TestFormatter`는 live/destroyed counter와 prefix 상태를 가집니다.

```cpp
cppf::Formatter *owned =
    new test_support::TestFormatter(cppf::TextBuffer("!"));
delete owned;
```

Base pointer 삭제 뒤 live count가 0이고 derived destructor count가 1인지 검사합니다. 별도 clone에는 prefix와 `name()`이 보존되는지도 확인합니다. 이는 단순히 “virtual destructor가 선언되어 있다”보다 실제 dynamic dispatch 결과를 관찰합니다.

### Pipeline behavior

해당 SHA의 unit test는 다음을 확인합니다.

- empty pipeline은 identity
- prefix → uppercase → suffix 순서가 `[VALUE]`를 만듦
- original에 step을 더한 뒤에도 copy는 기존 3-step 상태 유지
- assignment가 각 step을 clone
- self-assignment 뒤 size와 behavior 보존
- capacity 초과가 `length_error`를 던지고 full pipeline 보존

CLI fixture는 `ex02_format_pipeline mixed`의 stdout이 정확히 `[MIXED]\n`인지 byte 단위로 비교합니다.

이 commit은 정상 clone과 virtual deletion은 검증하지만 “몇 번째 clone이 실패했을 때 부분 객체가 남지 않는가”는 아직 다루지 않습니다.

## `2c99290b9268` — `test(format): 복제 실패 뒤 부분 객체 정리 검증`

`TestFormatter::clone()`에 호출 횟수 기반 failure seam이 추가되어 있습니다.

```cpp
Formatter *TestFormatter::clone() const
{
    ++clone_attempts_;
    if (clone_failure_attempt_ != 0 &&
        clone_attempts_ == clone_failure_attempt_)
        throw CloneFailure();
    return new TestFormatter(*this);
}
```

Failure test는 네 step source에 대해 1~4번째 clone을 각각 실패시킵니다.

### Copy construction failure

각 위치마다 다음을 검사합니다.

- 예정된 `CloneFailure`가 그대로 전달됨
- clone attempt가 설정한 위치에서 정확히 멈춤
- `TestFormatter::liveCount()`가 시도 전과 같음
- source의 size와 `>>>>value` 동작이 보존됨

Live count가 같다는 것은 실패 전에 성공한 clone들이 constructor catch에서 모두 삭제되었음을 직접 관찰하는 지표입니다.

### Assignment failure

Destination에는 먼저 `!` prefix step을 넣습니다. Source copy 도중 clone이 실패한 뒤에도 다음을 요구합니다.

- destination size는 1
- `target.apply("value") == "!value"`
- live formatter 수가 baseline과 같음

즉 “누수가 없다”와 “기존 destination이 보존된다”를 별도 assertion으로 확인합니다. 앞의 것은 cleanup 책임, 뒤의 것은 copy-before-swap commit 순서를 겨냥합니다.

이 테스트는 정확히 네 step과 설정된 clone failure만 다룹니다. Formatter의 `apply()` 자체가 예외를 던지는 경우나 creator가 null pointer를 반환하는 경우는 이 Thread의 증명 범위가 아닙니다.

## 최종 ownership 모델

| 대상 | owner가 되는 시점 | owner 책임이 끝나는 시점 |
| --- | --- | --- |
| 외부 formatter 인자 | pipeline이 소유하지 않음 | 호출자가 원래 lifetime 관리 |
| `clone()` 반환 pointer | `steps_[size_]`에 저장되고 `size_`가 증가한 뒤 pipeline | pipeline destructor 또는 swap 뒤 임시 destructor |
| copy constructor의 부분 clone | 각 `append()` 성공 직후 생성 중 객체 | 다음 clone 실패 시 catch에서 직접 delete |
| assignment용 임시 pipeline | 임시 copy가 모든 clone을 소유 | swap 뒤 scope 종료 시 이전 destination steps delete |

```text
append: borrowed formatter -> clone -> owned slot publish
copy:   source steps -> clone prefix -> (실패 시 prefix delete) -> complete copy
assign: complete copy -> swap -> old destination destruction
```

이 Thread의 핵심은 polymorphism 자체가 아니라, **동적 타입을 보존하는 복제와 raw pointer ownership의 종료 지점을 같은 `size_` 불변 조건으로 묶은 것**입니다.

## Thread 경계와 검증 범위

- `FormatPipeline::max_steps`는 고정 8이며 동적 capacity 확장은 다루지 않습니다.
- `clone()`은 성공 시 non-null owning pointer를 반환한다는 contract를 전제로 하며 null 반환 방어는 없습니다.
- `apply()` 중 formatter가 예외를 던졌을 때 pipeline 자체는 수정되지 않지만, 그 예외 의미는 이 Thread의 주제가 아닙니다.
- 각 exact SHA의 diff와 source/test를 검사했습니다. 이 환경에서는 branch를 로컬로 checkout하지 못해 test binary 실행 결과는 주장하지 않습니다.
