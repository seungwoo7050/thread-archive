# Thread: Generic container 검산을 transactional batch 교체로 연결하기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

이 Thread에는 서로 연결되지만 성격이 다른 두 층이 있습니다.

1. `RandomAccessBatch<T, Container>`가 vector와 deque에 공통으로 적용할 수 있는 값·iterator·정렬 contract를 제공합니다.
2. `BatchEngine::replace()`가 입력 전체를 local state에서 parse·계산·정렬·대조한 뒤, 마지막에만 이전 결과를 교체합니다.

최종 불변 조건은 다음과 같습니다.

- Public result는 완성된 batch 전체이거나 이전 batch 전체입니다. 부분 결과는 공개되지 않습니다.
- 이름 grammar, 중복, RPN 계산, stream 상태, allocation, container 대조 중 어느 단계가 실패해도 `results_`는 값과 storage identity를 유지합니다.
- 결과 순서는 input order가 아니라 `(value, name)`의 total order로 결정됩니다.
- Vector와 deque가 같은 값과 comparator로 정렬한 결과가 다르면 commit하지 않고 `logic_error`를 냅니다.
- Serialization은 local classic-locale buffer에서 완성한 뒤 caller stream에 전달됩니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `708c025ef2a0` | `feat(template): 임의 접근 container batch 추상화 추가` | S | `ARCH, GENERIC, CORE` | random-access container를 값처럼 감싸는 template 도입 |
| `aaeff163baf8` | `test(template): iterator·정렬·복사 실패 계약 검증` | A | `TEST, GENERIC, EXCEPTION` | vector/deque와 throwing value로 template contract 검증 |
| `d0295f82614b` | `feat(batch): 입력 문법과 원자 교체 구현` | S | `ARCH, EXCEPTION, INTEGRATION` | parse/evaluate를 local candidate에 모으고 final swap으로 공개 |
| `42d411e42268` | `feat(batch): 결과 정렬과 직렬화 제공` | A | `DETERMINISM, EXCEPTION, CORE` | `(value, name)` ordering과 local rendering 도입 |
| `af57a8f9c5fe` | `feat(batch): 두 container의 정렬 결과 대조` | A | `GENERIC, INTEGRATION, DETERMINISM` | vector/deque 양쪽 실행과 equality gate를 commit 전에 연결 |
| `9ba0e7c897ed` | `test(batch): 입력 순열과 출력 결정성 검증` | A | `TEST, DETERMINISM, EDGE` | permutation-independent exact output을 고정 |
| `ea23237ad506` | `fix(batch): 입력 stream 종료 상태를 명확히 구분` | A | `DEBUG, PARSING, EDGE` | newline, final unterminated line, EOF, read failure를 분리 |
| `b4ddd78fb9aa` | `test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증` | A | `TEST, EXCEPTION, RISK` | 모든 주요 failure phase 뒤 값·주소·allocation baseline 보존 검증 |

# 1. Generic container layer

## `708c025ef2a0` — `feat(template): 임의 접근 container batch 추상화 추가`

`RandomAccessBatch`는 value type과 backing container를 template parameter로 받습니다. Default는 `std::vector<T>`이고, 이후 실제 integration에서는 `std::deque<T>`도 사용합니다.

```cpp
template <class T, class Container = std::vector<T> >
class RandomAccessBatch
{
public:
    typedef typename Container::iterator iterator;
    typedef typename Container::const_iterator const_iterator;

    void push_back(const T &value);
    T &at(std::size_t index);
    const T &at(std::size_t index) const;
    iterator begin();
    iterator end();
    const_iterator begin() const;
    const_iterator end() const;

    template <class Compare>
    void sort(Compare compare)
    {
        std::sort(values_.begin(), values_.end(), compare);
    }

private:
    Container values_;
};
```

### 실제로 요구하는 container 능력

Template은 이름만 generic한 것이 아니라 body가 요구하는 연산으로 usable container 범위를 정합니다.

| 사용 지점 | Container 요구사항 |
| --- | --- |
| `push_back` | `push_back(const T&)` |
| `at` | `size()`와 `operator[]` |
| iterator 노출 | mutable/const `begin()`, `end()` |
| `sort` | iterator가 random-access이고 element가 comparator에 따라 교환 가능 |
| copy/assignment | backing container의 copy construction |
| `swap` | non-throwing 수준으로 사용할 수 있는 `Container::swap` |

따라서 vector와 deque는 맞지만 list는 `std::sort`에 필요한 random-access iterator가 없어 맞지 않습니다. 이 restriction은 후속 compile-fail contract에서 명시적으로 확인됩니다.

`at()`는 backing container의 `at()`에 의존하지 않고 size를 검사한 뒤 `operator[]`를 사용해 message를 `batch index`로 고정합니다.

```cpp
if (index >= values_.size())
    throw std::out_of_range("batch index");
return values_[index];
```

Assignment는 self-alias를 먼저 걸러낸 뒤 copy-and-swap을 사용합니다.

```cpp
if (this != &other)
{
    RandomAccessBatch copy(other);
    swap(copy);
}
```

Copy가 실패하면 backing container가 자신의 부분 element를 정리하고 destination은 아직 수정되지 않습니다. Self-assignment는 element copy를 아예 수행하지 않습니다.

### Container 종류와 무관한 range comparison

```cpp
template <class FirstIterator, class SecondIterator>
bool equal_ranges(FirstIterator first, FirstIterator last,
                  SecondIterator second, SecondIterator second_last)
{
    while (first != last && second != second_last)
    {
        if (!(*first == *second))
            return false;
        ++first;
        ++second;
    }
    return first == last && second == second_last;
}
```

Iterator subtraction이나 동일 iterator type을 요구하지 않고 두 range를 lockstep으로 순회합니다. 값 mismatch뿐 아니라 한쪽이 먼저 끝나는 length mismatch도 거부합니다.

이 commit은 `std::sort` 자체가 comparator/value exception 뒤 원래 순서를 보존한다고 약속하지 않습니다. Strong transaction은 이 batch를 local candidate로 사용하는 상위 `BatchEngine`에서 확보합니다.

## `aaeff163baf8` — `test(template): iterator·정렬·복사 실패 계약 검증`

Compile test는 header를 두 번 include하고 `RandomAccessBatch<int, std::deque<int> >`를 실제로 정렬합니다. 즉 template declaration만 parse되는 것이 아니라 non-default container에서 필요한 member가 instantiate되는지 확인합니다.

Unit test의 정상 범위는 다음과 같습니다.

- mutable/const `at`
- mutable/const iterator와 표준 algorithm 연결
- caller comparator에 따른 descending sort
- vector/deque 사이 `equal_ranges`
- empty, value mismatch, length mismatch
- non-scalar `std::string`
- copy independence, assignment, swap, self-assignment

더 중요한 부분은 copy constructor에서 설정된 위치에 예외를 던지는 `ThrowingValue`입니다.

### `push_back` 실패

빈 batch에 첫 값을 복사하는 시점에 `ValueCopyFailure`를 주입하고 다음을 확인합니다.

- batch는 empty
- 외부 source object 하나만 live
- scope 종료 뒤 live count 0

### Batch copy 실패

3개 value source를 복사하는 두 번째 element에서 실패시킵니다. 실패한 새 batch는 완성되지 않지만 backing `std::vector`가 앞서 복사한 element를 정리해야 합니다. Test는 source size 3과 live count baseline을 확인합니다.

### Assignment 실패

Destination에 값 9를 둔 뒤 source copy의 두 번째 element에서 실패시킵니다.

- destination size 1
- destination first value 9
- live count baseline 유지

Self-assignment는 `copyAttempts() == 0`을 요구해 early alias branch가 실제로 element copy를 건너뛰는지도 확인합니다.

이 test는 `ThrowingValue`와 사용 중인 standard containers에서 관찰되는 behavior를 고정합니다. 임의의 user-supplied container가 같은 exception guarantee를 제공한다고 일반화하지는 않습니다.

# 2. Transactional batch engine

## `d0295f82614b` — `feat(batch): 입력 문법과 원자 교체 구현`

`BatchEngine`은 public result를 `std::vector<JobResult>`로 보관하고 read-only reference로 노출합니다.

```cpp
class BatchEngine
{
public:
    void replace(std::istream &input);
    const std::vector<JobResult> &results() const;

private:
    std::vector<JobResult> results_;
};
```

### 한 line의 grammar

각 line은 separator `|`를 정확히 하나 가져야 합니다. 양쪽 field는 ASCII whitespace를 trim합니다.

```text
<name> | <rpn-expression>
```

Name은 첫 byte가 ASCII letter, 이후 byte가 letter/digit/underscore/hyphen이어야 합니다. Expression은 trim 뒤 비어 있으면 안 됩니다. Embedded NUL이나 non-ASCII byte는 이 문자 집합에 포함되지 않으므로 거부됩니다.

```cpp
if (separator == std::string::npos ||
    line.find('|', separator + 1) != std::string::npos)
    throw std::invalid_argument("invalid batch input");
```

### Replace transaction

```cpp
void BatchEngine::replace(std::istream &input)
{
    std::vector<JobResult> candidate;
    std::map<std::string, long> seen;
    std::string line;

    while (std::getline(input, line))
    {
        std::string name;
        std::string expression;

        parseLine(line, name, expression);
        if (seen.find(name) != seen.end())
            throw std::invalid_argument("invalid batch input");
        const long value = RpnEvaluator::evaluate(expression);

        seen.insert(std::make_pair(name, value));
        candidate.push_back(JobResult(name, value));
    }
    if (!input.eof() || candidate.empty())
        throw std::invalid_argument("invalid batch input");
    results_.swap(candidate);
}
```

`results_`는 마지막 줄까지 parse하고 모든 RPN 계산과 insertion이 끝난 뒤에만 바뀝니다. Name 오류, duplicate, `invalid rpn expression`, `rpn overflow`, `bad_alloc`, stream error는 모두 local `candidate`와 `seen`을 stack unwinding으로 정리하고 기존 결과를 보존합니다.

Empty input도 유효한 “결과를 비우는 요청”이 아니라 invalid batch입니다. 따라서 성공한 replace는 항상 하나 이상의 result를 publish합니다.

## `42d411e42268` — `feat(batch): 결과 정렬과 직렬화 제공`

결과 정렬 기준은 value ascending, 같은 value에서는 name ascending입니다.

```cpp
bool resultLess(const JobResult &left, const JobResult &right)
{
    if (left.value() != right.value())
        return left.value() < right.value();
    return left.name() < right.name();
}
```

Duplicate name은 이미 거부되므로 accepted batch 안에서 이 comparator는 각 result의 위치를 결정합니다. Input 순서, map iteration order, process locale에 의존하지 않습니다.

Sort도 `results_`가 아닌 local candidate에서 실행합니다.

```cpp
std::sort(candidate.begin(), candidate.end(), resultLess);
results_.swap(candidate);
```

Serialization 역시 caller stream에 line을 하나씩 직접 쓰지 않습니다.

```cpp
std::ostringstream rendered;
rendered.imbue(std::locale::classic());
for (std::size_t index = 0; index < results_.size(); ++index)
    rendered << results_[index].value() << " | "
             << results_[index].name() << '\n';
const std::string text = rendered.str();
output.write(text.data(), static_cast<std::streamsize>(text.size()));
```

Caller의 width, fill, locale, numeric flags가 output bytes를 바꾸지 않고, internal rendering 실패 전에 partial line을 caller에게 공개하지 않습니다. 마지막 unformatted write의 underlying I/O 자체를 rollback하는 것은 아닙니다.

## `af57a8f9c5fe` — `feat(batch): 두 container의 정렬 결과 대조`

Batch parse loop는 각 `JobResult`를 vector-backed batch와 deque-backed batch 양쪽에 넣습니다.

```cpp
RandomAccessBatch<JobResult> vector_batch;
RandomAccessBatch<JobResult, std::deque<JobResult> > deque_batch;
```

모든 input을 처리한 뒤 같은 comparator로 각각 정렬하고 range를 비교합니다.

```cpp
vector_batch.sort(resultLess);
deque_batch.sort(resultLess);
if (!equal_ranges(vector_batch.begin(), vector_batch.end(),
                  deque_batch.begin(), deque_batch.end()))
    throw std::logic_error("batch container disagreement");
```

합의한 경우에만 vector batch를 public representation으로 materialize하고 swap합니다.

```cpp
std::vector<JobResult> candidate(
    vector_batch.begin(), vector_batch.end());
results_.swap(candidate);
```

이 단계에서도 모든 allocation과 sort는 local입니다. Deque push/sort, vector push/sort, equality, final vector materialization 중 어느 단계가 실패해도 `results_`에는 도달하지 않습니다.

이 검사는 두 개의 독립적인 parser나 다른 sorting algorithm을 비교하는 것은 아닙니다. 같은 parsed `JobResult`와 같은 comparator를 서로 다른 random-access storage에 적용해 container-specific ordering disagreement를 검출하는 검산입니다.

## `9ba0e7c897ed` — `test(batch): 입력 순열과 출력 결정성 검증`

Test는 같은 일곱 job을 서로 다른 순서로 제공하고 exact output을 동일하게 요구합니다.

```text
-2 | neg
0 | zero
2 | pos
5 | Alpha
5 | alpha
5 | beta
5 | zeta
```

여기서 같은 value 5의 name 순서는 byte-wise string order입니다. Uppercase `Alpha`가 lowercase 이름들보다 먼저 옵니다. 두 번째 input은 마지막 newline이 없어 final unterminated line path도 함께 통과합니다.

추가로 one-element batch와 같은 engine의 반복 `write()`가 byte-identical인지 검사합니다. 이 commit은 selected permutations에 대한 deterministic regression이며 모든 순열을 exhaustive하게 생성하지는 않습니다.

## `ea23237ad506` — `fix(batch): 입력 stream 종료 상태를 명확히 구분`

기존 `std::getline` loop와 종료 후 `input.eof()` 확인은 newline 유무, EOF bit, fail/bad state를 한 표현에 묶어 해석해야 했습니다. Fix는 char-level `readLine()`으로 네 결과를 분리합니다.

```cpp
bool readLine(std::istream &input, std::string &line)
{
    char value;

    line.clear();
    while (input.get(value))
    {
        if (value == '\n')
            return true;
        line.push_back(value);
    }
    if (!input.eof())
        throw std::invalid_argument("invalid batch input");
    return !line.empty();
}
```

| Stream 상태 | `readLine()` 결과 |
| --- | --- |
| newline을 만남 | `true`, newline 제외 line 반환 |
| EOF 전에 문자를 읽었으나 final newline 없음 | `true`, 마지막 line 반환 |
| 아무 문자 없이 정상 EOF | `false`, loop 종료 |
| EOF가 아닌 fail/bad | `invalid batch input` |
| 빈 line 뒤 newline | `true` with empty line, `parseLine`이 거부 |

Helper가 non-EOF error를 즉시 처리하므로 replace 끝의 `!input.eof()` 조건은 제거되고 `vector_batch.empty()`만 남습니다. 이 change는 ordinary file뿐 아니라 미리 `badbit`가 설정된 stream에서 기존 result가 보존되는 경로를 명확하게 만듭니다.

## `b4ddd78fb9aa` — `test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증`

이 commit은 input, downstream arithmetic, stream, allocation failure를 하나의 strong-guarantee 관점으로 묶습니다.

### Invalid input와 RPN failure

Helper는 실패 전 다음 두 값을 저장합니다.

```cpp
const std::string before = writeBatch(engine);
const JobResult *first = &engine.results()[0];
```

실패 뒤에는 error type/message뿐 아니라 다음을 동시에 검사합니다.

```cpp
&engine.results()[0] == first
writeBatch(engine) == before
```

같은 byte를 다시 만들었다는 것만으로는 충분하지 않습니다. 첫 element 주소까지 같다는 assertion은 실패 경로에서 `results_`를 swap하거나 재할당하지 않았다는 증거입니다.

대상 입력에는 다음이 포함됩니다.

- spaces-only, blank middle/trailing line
- missing/extra separator
- empty name/expression
- digit-first, whitespace/punctuation, NUL, non-ASCII name
- duplicate name
- RPN division by zero와 arithmetic overflow
- CRLF field trimming과 final newline 없음
- 시작부터 `badbit`인 stream

### Allocation failure sweep

정상 3-job replacement에서 관찰한 global allocation 수를 얻고, 1번부터 마지막까지 각 allocation을 실패시킵니다. 매 반복마다 seeded result `7 | seed`가 value와 serialized form을 유지하고 live block count가 baseline으로 돌아오는지 검사합니다.

이 sweep은 name/string, map node, RPN token, vector/deque growth, sort-related copy, final vector materialization 등 실제 실행에서 관찰된 allocation 지점을 포괄합니다. 구현 내부 allocation 수를 하드코딩하지 않고 성공 경로에서 측정한 범위를 사용합니다.

### Compile-time mutation 차단

- `RpnEvaluator`를 instance로 만들 수 없어 static utility contract를 확인합니다.
- `engine.results().push_back(...)`가 compile되지 않아 public result reference가 const임을 확인합니다.

CLI invalid-RPN fixture는 stdout이 비어 있고 stderr가 정확히 `invalid rpn expression\n`인지도 고정합니다.

## 최종 transaction과 terminal state

```text
readLine
  -> parseLine(name, expression)
  -> duplicate name 검사
  -> RpnEvaluator::evaluate
  -> JobResult를 vector_batch/deque_batch에 append
(repeat until clean EOF)
  -> nonempty 확인
  -> 두 batch를 (value, name)으로 sort
  -> equal_ranges로 합의 확인
  -> vector candidate materialize
  -> results_.swap(candidate)       # 유일한 public commit
```

| 실패 지점 | Local state | Public `results_` |
| --- | --- | --- |
| line grammar / duplicate | stack containers가 정리 | 값·주소 미변경 |
| RPN invalid / overflow | 현재까지의 batch가 정리 | 값·주소 미변경 |
| stream non-EOF failure | candidate 정리 | 값·주소 미변경 |
| allocation/copy failure | vector/deque/map/string가 unwind | 값·주소 미변경 |
| container disagreement | 두 batch 정리 | 값·주소 미변경 |
| 성공 | candidate가 이전 results를 swap 후 소유 | 완성·정렬된 새 결과 |

이 Thread의 핵심은 vector와 deque를 사용했다는 사실이 아닙니다. **입력 전체의 의미와 두 storage의 합의를 local state에서 확정하고, public result를 바꾸는 문장을 마지막 swap 하나로 제한한 것**입니다.

## Thread 경계와 검증 범위

- RPN token과 checked arithmetic의 세부 규칙은 별도 RPN Thread에서 다룹니다.
- Dual-container 비교는 같은 comparator를 사용하므로 comparator 자체가 잘못된 경우를 독립적으로 발견하지 못합니다.
- `std::sort`에 부적합한 container나 throwing comparator의 standalone guarantee는 `RandomAccessBatch`가 보장하지 않습니다.
- 각 exact SHA의 diff/source/test를 확인했습니다. 이 환경에서는 branch를 로컬로 checkout하지 못해 test와 allocation sweep을 실행하지 않았습니다.
