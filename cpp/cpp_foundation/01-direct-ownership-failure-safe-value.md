# Thread: 직접 소유권을 실패 안전한 값으로 만들기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

이 Thread는 `TextBuffer`가 단순히 `char *`를 감싸는 클래스에서, 복사·대입·결합 실패에도 기존 값을 보존하는 **독립적인 값 타입**으로 완성되는 과정을 다룹니다.

최종 불변 조건은 다음과 같습니다.

- 살아 있는 `TextBuffer`는 항상 자신만의 NUL 종료 저장소를 하나 소유합니다.
- 복사된 객체끼리는 저장소를 공유하지 않습니다.
- 대입과 문자열 결합은 새 저장소를 먼저 완성한 뒤, 예외를 던지지 않는 교체 연산으로 결과를 공개합니다.
- 할당이 실패하면 source와 destination을 포함해 이미 존재하던 객체의 값은 바뀌지 않습니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `aa3b5ba6c3c4` | `feat(buffer): 종료 문자를 포함한 문자열 저장소 소유` | A | `OWNERSHIP, CORE` | 단일 owner와 기본 저장소 불변 조건을 확립 |
| `0bc528c7d58e` | `feat(buffer): 깊은 복사와 정규 대입 구현` | S | `OWNERSHIP, EXCEPTION, CORE` | 복사 생성과 copy-and-swap 대입으로 strong guarantee를 도입 |
| `93faed0d67a2` | `feat(buffer): 결합·비교·출력 연산 제공` | A | `OWNERSHIP, EXCEPTION, CORE` | 결합까지 prepare-then-commit 규칙을 확장 |
| `47134f9e3b29` | `test(buffer): 할당 실패와 복사 생략 비활성화 검증` | A | `TEST, EXCEPTION, OWNERSHIP` | 모든 관찰된 할당 지점과 no-elide 경로를 회귀 테스트로 고정 |

## `aa3b5ba6c3c4` — `feat(buffer): 종료 문자를 포함한 문자열 저장소 소유`

초기 구현은 복사 생성자와 대입 연산자를 private으로 두고, 한 객체가 한 allocation을 소유하는 최소 규칙부터 세웁니다.

```cpp
class TextBuffer
{
public:
    TextBuffer();
    explicit TextBuffer(const char *text);
    ~TextBuffer();

    std::size_t size() const;
    bool empty() const;
    const char *c_str() const;
    char &at(std::size_t index);
    const char &at(std::size_t index) const;
    void swap(TextBuffer &other) throw();

private:
    TextBuffer(const TextBuffer &other);
    TextBuffer &operator=(const TextBuffer &other);

    char *data_;
    std::size_t size_;
};
```

생성자는 null pointer를 빈 문자열로 정규화하고, 표시할 문자 수에 terminator 한 byte를 더해 저장소를 만듭니다.

```cpp
TextBuffer::TextBuffer(const char *text) : data_(0), size_(0)
{
    if (text == 0)
        text = "";
    size_ = std::strlen(text);
    data_ = new char[size_ + 1];
    std::memcpy(data_, text, size_ + 1);
}
```

여기서 정해진 표현 규칙은 이후 모든 커밋의 전제가 됩니다.

| 상태 | 의미 |
| --- | --- |
| `data_ != 0` | 정상 생성된 객체는 빈 문자열도 실제 저장소를 소유함 |
| `data_[size_] == '\0'` | `size_`는 terminator를 제외한 길이 |
| `delete[] data_` | 저장소 해제 책임은 해당 `TextBuffer` 하나에만 있음 |
| `at(index)` | `index >= size_`이면 `std::out_of_range("buffer index")` |

`swap()`은 pointer와 길이만 교환하며 `throw()`로 선언됩니다. 아직 복사는 금지되어 있지만, 나중에 값을 안전하게 공개할 수 있는 비예외 commit primitive는 이미 마련된 셈입니다.

이 커밋이 보장하지 않는 것은 명확합니다. 객체를 복사해 독립적인 값을 만들 수 없고, 기존 객체를 새 값으로 교체하는 실패 경로도 아직 존재하지 않습니다.

## `0bc528c7d58e` — `feat(buffer): 깊은 복사와 정규 대입 구현`

이 커밋에서 복사 생성자와 대입 연산자가 public이 됩니다. 핵심은 단순히 “복사가 가능해졌다”가 아니라, **실패할 수 있는 작업과 기존 상태를 바꾸는 작업을 분리했다**는 점입니다.

### 복사 생성

```cpp
TextBuffer::TextBuffer(const TextBuffer &other)
    : data_(new char[other.size_ + 1]), size_(other.size_)
{
    std::memcpy(data_, other.data_, size_ + 1);
}
```

새 객체는 source와 같은 크기의 별도 저장소를 만듭니다. `new[]`가 실패하면 객체 생성 자체가 끝나지 않으므로 source에는 아무 변화가 없습니다. 성공하면 두 객체가 같은 byte를 갖더라도 소유하는 pointer는 다릅니다.

### 대입

```cpp
TextBuffer &TextBuffer::operator=(const TextBuffer &other)
{
    TextBuffer copy(other);

    swap(copy);
    return *this;
}
```

상태 변화는 다음 순서로 일어납니다.

```text
other 읽기
  -> 임시 copy의 저장소 할당 및 복사   # 예외 가능, *this 미변경
  -> swap(copy)                         # 비예외 commit
  -> 임시 copy 소멸                     # 이전 *this 저장소 해제
```

이 구조가 주는 보장은 두 가지입니다.

1. **할당 실패 시 기존 destination 보존**: `TextBuffer copy(other)`가 끝나기 전에는 `*this`를 건드리지 않습니다.
2. **self-assignment 안전성**: `value = value`도 먼저 별도 copy를 만든 뒤 swap하므로 alias를 특별 취급하지 않아도 됩니다.

대입 성공 뒤에는 임시 객체가 destination의 이전 저장소를 소유하게 되고, 함수가 끝날 때 그 저장소를 정확히 한 번 해제합니다. 기존 pointer를 먼저 지운 뒤 새로 할당하는 구현과 달리, 실패로 인해 destination이 빈 값이나 dangling pointer가 되는 구간이 없습니다.

## `93faed0d67a2` — `feat(buffer): 결합·비교·출력 연산 제공`

`operator+=`는 현재 객체의 저장소를 실제로 교체하므로 대입과 같은 실패 안전성이 필요합니다. 이 커밋은 결합 결과를 새 저장소에 완성한 뒤에만 기존 저장소를 버립니다.

```cpp
TextBuffer &TextBuffer::operator+=(const TextBuffer &other)
{
    const std::size_t maximum =
        std::numeric_limits<std::size_t>::max();

    if (other.size_ > maximum - size_ - 1)
        throw std::length_error("buffer length");

    const std::size_t joined_size = size_ + other.size_;
    char *joined = new char[joined_size + 1];

    std::memcpy(joined, data_, size_);
    std::memcpy(joined + size_, other.data_, other.size_ + 1);
    delete[] data_;
    data_ = joined;
    size_ = joined_size;
    return *this;
}
```

### 길이 계산부터 검사하는 이유

`size_ + other.size_ + 1`을 먼저 계산하면 unsigned overflow가 이미 발생한 뒤입니다. 조건은 덧셈을 수행하기 전에 오른쪽 길이가 남은 표현 범위를 넘는지 확인합니다.

```text
other.size_ <= SIZE_MAX - size_ - 1
```

이 조건이 참일 때만 `joined_size + 1`이 유효한 allocation 크기입니다.

### self-concatenation이 안전한 이유

`value += value`에서 `other.data_`와 `data_`는 같은 pointer입니다. 구현은 두 source range를 모두 `joined`로 복사한 **뒤에** `data_`를 해제합니다. 따라서 두 번째 `memcpy`가 해제된 저장소를 읽는 순서 역전이 없습니다.

### `operator+`의 실패 경로

```cpp
TextBuffer operator+(const TextBuffer &left, const TextBuffer &right)
{
    TextBuffer result(left);

    result += right;
    return result;
}
```

이 연산은 최소한 다음 두 allocation 단계가 있을 수 있습니다.

1. `left`를 `result`로 복사
2. `result += right`의 결합 저장소 생성

어느 단계에서 실패하더라도 `left`와 `right`는 수정되지 않습니다. 두 번째 단계에서 실패하면 임시 `result`가 stack unwinding 중 자신의 첫 저장소를 해제합니다. 반환값 복사 생략은 성능 최적화일 뿐, 정리의 정확성을 위한 전제가 아닙니다.

같은 커밋의 비교 연산과 stream 출력은 저장소 소유권을 바꾸지 않습니다. 비교는 NUL 종료 표현을 사용하고, `operator<<`는 `c_str()`을 출력 대상으로 노출할 뿐 ownership을 이전하지 않습니다.

## `47134f9e3b29` — `test(buffer): 할당 실패와 복사 생략 비활성화 검증`

이 테스트 커밋은 전역 `operator new`/`new[]`를 test binary 안에서 교체해 다음 값을 기록합니다.

- 현재까지의 allocation 시도 횟수
- 지정된 N번째 allocation 실패
- 아직 해제되지 않은 allocation block 수

그리고 정상 실행에서 관찰한 allocation 수를 기준으로 실패 지점을 1번부터 끝까지 순회합니다. 특정 구현에서 “첫 번째 `new`만 실패”시키는 테스트보다, 복사 생략 여부나 내부 allocation 수가 달라져도 실제로 도달한 모든 지점을 검사할 수 있습니다.

| 대상 경로 | 실패 뒤 확인하는 상태 |
| --- | --- |
| `TextBuffer(const char *)` | live block 수가 baseline으로 복귀 |
| copy construction | source의 byte와 길이 보존, leak 없음 |
| `destination = source` | destination과 source 모두 기존 값 보존 |
| `value = value` | alias 상태 보존, double free 없음 |
| `left += right` | left/right 보존, 임시 allocation 누수 없음 |
| `left + right` | 관찰된 모든 allocation 지점에서 두 operand 보존 |

별도 `-fno-elide-constructors` binary도 같은 unit suite를 실행합니다. 따라서 `operator+`와 반환값 경로의 정확성을 컴파일러의 copy elision에 의존해 설명할 수 없습니다.

이 테스트가 직접 증명하는 범위는 test harness가 관찰한 `new`/`new[]` 경로와 객체 상태입니다. 임의의 allocator 구현, 메모리 손상 전체, concurrency는 이 Thread의 검증 범위가 아닙니다.

## 최종 상태

| 연산 | 준비 단계 | commit 지점 | 실패 시 외부에서 보이는 상태 |
| --- | --- | --- | --- |
| 복사 생성 | 새 `char[]` 할당 후 byte 복사 | 생성 완료 | source 그대로, 새 객체 없음 |
| 대입 | 임시 deep copy 완성 | `swap(copy)` | source/destination 모두 그대로 |
| `+=` | overflow 검사 후 joined buffer 완성 | 기존 pointer 해제 후 `data_`/`size_` 갱신 | 양 operand 그대로 |
| `+` | left copy 후 임시 객체에 `+=` | 반환 객체 완성 | 두 operand 그대로, 임시 자원 정리 |

이 Thread에서 가장 중요한 변화는 raw pointer를 숨겼다는 사실이 아닙니다. **실패할 수 있는 allocation과 기존 값을 바꾸는 시점을 분리하고, 모든 공개 연산이 같은 prepare-then-commit 규칙을 따르게 한 것**이 핵심입니다.

## Thread 경계와 검증 범위

- C++98 프로젝트이므로 move constructor나 move assignment는 다루지 않습니다.
- `TextBuffer`의 저장소는 항상 `new[]`/`delete[]` 조합이며 custom allocator 정책은 없습니다.
- stream 자체의 write failure 처리와 동시 접근 안전성은 이 네 커밋의 주제가 아닙니다.
- 각 SHA의 diff와 해당 시점 source/test를 확인해 작성했습니다. 이 환경에서는 branch를 로컬로 checkout하지 못해 binary build와 test 실행 결과를 주장하지 않습니다.
