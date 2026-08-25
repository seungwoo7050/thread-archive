# Thread: Factory 교체를 transaction 경계로 만들기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

이 Thread는 문자열 specification으로 polymorphic formatter를 만들고 기존 `FormatPipeline`을 교체하는 작업을 다룹니다. Factory가 객체 하나를 생성하는 단계보다 어려운 지점은 여러 specification을 처리하는 중간에 parsing, creation, clone, allocation 중 하나가 실패할 수 있다는 사실입니다.

최종 규칙은 단순합니다.

> 새 pipeline의 모든 step이 성공적으로 만들어지기 전에는 기존 target을 바꾸지 않는다.

이를 위해 creator가 돌려준 raw pointer는 짧은 RAII owner가 즉시 맡고, 완성 중인 step은 local candidate가 소유하며, target mutation은 마지막 `swap()` 한 번으로 제한됩니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `4c34654a4602` | `feat(factory): 문자열 명세로 formatter 생성` | A | `ARCH, POLYMORPHISM, API` | specification grammar와 creator ownership contract 도입 |
| `fc0b8b7a40a0` | `feat(factory): formatter 임시 소유와 pipeline 교체 구현` | A | `OWNERSHIP, INTEGRATION, EXCEPTION` | raw factory result 누수를 막고 교체 흐름을 연결 |
| `907bfbd5c37c` | `fix(factory): 교체 실패에도 기존 파이프라인 보존` | S | `DEBUG, EXCEPTION, CORE` | early publish를 제거하고 candidate 완성 뒤 한 번만 commit |
| `466d7abdb60f` | `test(factory): 교체 실패 상태 보존과 CLI 검증` | B | `TEST, EXCEPTION` | 잘못된 specification과 CLI 결과를 회귀 고정 |
| `af4e35ca7d92` | `test(factory): 생성·복제·할당 실패 정리 검증` | A | `TEST, EXCEPTION, OWNERSHIP` | creation/clone/global allocation 실패를 각각 주입해 정리와 상태 보존 검증 |

## `4c34654a4602` — `feat(factory): 문자열 명세로 formatter 생성`

Public API는 creator와 builder를 분리합니다.

```cpp
class FormatterCreator
{
public:
    virtual ~FormatterCreator();
    virtual Formatter *create(const std::string &specification) const = 0;
};

class PipelineBuilder
{
public:
    static void replace(FormatPipeline &target,
                        const FormatterCreator &creator,
                        const std::string *specifications,
                        std::size_t count);

private:
    PipelineBuilder();
};
```

`DefaultFormatterCreator`가 인정하는 grammar는 제한적이고 완전합니다.

| specification | 결과 | 오류 |
| --- | --- | --- |
| `upper` | `new UppercaseFormatter()` | 없음 |
| `prefix=<text>` | nonempty `<text>`를 가진 `PrefixFormatter` | payload가 비면 `InvalidSpecification` |
| `suffix=<text>` | nonempty `<text>`를 가진 `SuffixFormatter` | payload가 비면 `InvalidSpecification` |
| 빈 문자열 | 없음 | `InvalidSpecification` |
| 그 밖의 문자열 | 없음 | `UnknownFormatter` |

```cpp
if (specification == "upper")
    return new UppercaseFormatter();
if (specification.compare(0, prefix_key.size(), prefix_key) == 0)
{
    if (specification.size() == prefix_key.size())
        throw InvalidSpecification();
    return new PrefixFormatter(
        TextBuffer(specification.substr(prefix_key.size()).c_str()));
}
```

`create()`의 반환값은 owning raw pointer입니다. 성공한 호출의 삭제 책임은 caller에게 넘어갑니다. 이 commit에서는 `PipelineBuilder::replace()`의 구현이 아직 없으므로, 여러 creator result를 어떤 순서로 소유하고 target에 공개할지는 정해지지 않았습니다.

## `fc0b8b7a40a0` — `feat(factory): formatter 임시 소유와 pipeline 교체 구현`

Builder 구현은 creator가 반환한 pointer를 즉시 `FormatterOwner`에 넣습니다.

```cpp
class FormatterOwner
{
public:
    explicit FormatterOwner(cppf::Formatter *formatter)
        : formatter_(formatter)
    {
    }

    ~FormatterOwner()
    {
        delete formatter_;
    }

    cppf::Formatter &get() const
    {
        return *formatter_;
    }

private:
    FormatterOwner(const FormatterOwner &other);
    FormatterOwner &operator=(const FormatterOwner &other);

    cppf::Formatter *formatter_;
};
```

따라서 `target.append()`가 clone 도중 예외를 던져도 현재 creator result는 owner destructor가 삭제합니다. 이미 append된 clone은 pipeline이 삭제합니다. **자원 누수 방지**만 보면 올바른 구조입니다.

하지만 초기 교체 순서는 다음과 같습니다.

```cpp
FormatPipeline empty;

if ((specifications == 0 && count != 0) ||
    count > FormatPipeline::max_steps)
    throw InvalidSpecification();
target.swap(empty);
for (std::size_t index = 0; index < count; ++index)
{
    FormatterOwner formatter(creator.create(specifications[index]));
    target.append(formatter.get());
}
```

문제는 첫 step을 만들기 전에 target을 비운다는 점입니다.

```text
기존 target
  -> target.swap(empty)     # 기존 값이 이미 외부에서 사라짐
  -> create/clone step 0
  -> create/clone step 1    # 여기서 실패 가능
  -> ...
```

중간 실패가 발생하면 자원은 정리되지만 target은 빈 상태 또는 일부 step만 가진 상태로 남습니다. 즉 이 구현은 다음 두 보장을 혼동합니다.

- **Leak safety**: 임시 객체와 이미 만든 clone이 남지 않는다.
- **Strong state guarantee**: 교체 전체가 실패하면 target이 호출 전과 동일하다.

`FormatterOwner`는 첫 번째 보장만 해결합니다. Target publish 시점은 여전히 너무 이릅니다.

## `907bfbd5c37c` — `fix(factory): 교체 실패에도 기존 파이프라인 보존`

수정 diff는 작지만 transaction의 의미를 바꿉니다.

```diff
-    FormatPipeline empty;
+    FormatPipeline candidate;
@@
-    target.swap(empty);
     for (index = 0; index < count; ++index)
     {
         FormatterOwner formatter(creator.create(specifications[index]));
-        target.append(formatter.get());
+        candidate.append(formatter.get());
     }
+    target.swap(candidate);
```

이제 모든 실패 가능 작업은 local candidate에서 끝납니다.

```text
입력 사전 검증
  -> candidate 생성
  -> specification[0] create -> owner -> candidate clone
  -> specification[1] create -> owner -> candidate clone
  -> ...
  -> candidate 완성
  -> target.swap(candidate)   # 유일한 commit
  -> candidate destructor     # 이전 target steps 정리
```

각 failure 지점의 owner도 명확합니다.

| 실패 지점 | 현재 raw creator result | 앞서 완성한 clone | 기존 target |
| --- | --- | --- | --- |
| `creator.create()` 내부 | 생성되지 않았거나 creator가 자체 정리 | `candidate`가 소유 | 미변경 |
| `FormatterOwner` 이후 `append()` clone | `FormatterOwner`가 delete | `candidate`가 소유 | 미변경 |
| candidate 내부 allocation/clone | scope unwind로 owner/candidate 정리 | `candidate` destructor | 미변경 |
| 모든 단계 성공 | owner는 각 원본을 이미 delete | candidate가 새 pipeline 소유 | final swap에서 교체 |

Swap 뒤 candidate는 target의 이전 step들을 소유합니다. 함수 종료 시 candidate destructor가 그 step들을 삭제하므로 성공 경로에서도 old target cleanup이 별도 branch 없이 완료됩니다.

이 fix는 `FormatPipeline::swap()`이 예외를 던지지 않는다는 앞 Thread의 contract에 의존합니다. Commit 단계가 다시 실패 가능해지면 같은 보장을 유지할 수 없습니다.

## `466d7abdb60f` — `test(factory): 교체 실패 상태 보존과 CLI 검증`

Unit test는 먼저 `prefix=[`, `upper`, `suffix=]`로 `[VALUE]` pipeline을 만듭니다. 그 뒤 가운데에 unknown formatter가 있는 배열을 교체 입력으로 사용합니다.

```cpp
std::string invalid_specifications[] = {
    "prefix=<", "reverse", "suffix=>"};

try
{
    PipelineBuilder::replace(
        pipeline, creator, invalid_specifications, 3);
}
catch (const UnknownFormatter &)
{
    threw = true;
}
```

Assertion은 예외 타입만 확인하지 않습니다.

- 기존 `pipeline.size() == 3`
- 기존 `pipeline.apply("value") == "[VALUE]"`
- null specification array와 nonzero count를 거부한 뒤에도 같은 값 보존
- `replace(target, creator, 0, 0)`은 유효한 empty replacement로 처리

CLI fixture도 두 경로를 구분합니다.

| 명령 | stdout | stderr / status |
| --- | --- | --- |
| `ex03_pipeline_factory mixed prefix=[ upper suffix=]` | `[MIXED]\n` | 성공 |
| `ex03_pipeline_factory mixed reverse` | 비어 있음 | `unknown formatter\n`, 실패 |

이 commit은 grammar failure와 public command behavior를 고정하지만, allocation이나 custom creator failure는 직접 주입하지 않습니다.

## `af4e35ca7d92` — `test(factory): 생성·복제·할당 실패 정리 검증`

### 1. Creator failure

`ControlledCreator`는 설정된 N번째 `create()` 호출에서 `CreationFailure`를 던집니다. 두 번째 creation을 실패시킨 경우 다음을 확인합니다.

- creation 시도는 정확히 2회
- 첫 specification의 clone은 1회 완료
- target의 기존 uppercase step과 `KEEP` 동작 보존
- candidate에 들어갔던 clone과 creator가 만든 원본의 live count가 모두 0으로 복귀

이는 이전 candidate가 stack unwinding으로 정리되는지 확인합니다.

### 2. Clone failure

Creator는 현재 specification의 `TestFormatter` 원본을 성공적으로 반환하지만, `candidate.append()`의 N번째 `clone()`이 실패합니다.

- 현재 raw object는 `FormatterOwner`가 삭제
- 이전 iteration의 clone은 `candidate` destructor가 삭제
- target은 기존 1-step pipeline 유지
- `CloneFailure` 타입이 다른 오류로 바뀌지 않고 전달

같은 예외 경로에서도 owner가 두 층으로 나뉜다는 점이 중요합니다.

### 3. 모든 allocation 위치 sweep

별도 failure binary는 정상적인 3-spec replacement에서 발생한 global allocation 횟수를 먼저 관찰합니다. 이후 1번부터 마지막 allocation까지 각각 `std::bad_alloc`을 주입합니다.

각 반복에서 다음 조건을 동시에 요구합니다.

```text
설정한 allocation index까지 실제 도달
std::bad_alloc 그대로 전달
unexpected exception 없음
기존 target size == 1
기존 target behavior == "KEEP"
live allocation blocks == 시도 전 baseline
```

따라서 test는 특정 `new` 하나뿐 아니라 specification substring, formatter 구성, `TextBuffer`, clone 등 실제 실행에서 관찰된 allocation 지점을 모두 대상으로 합니다.

## 최종 transaction

```text
validate(specifications, count)
  -> local FormatPipeline candidate
  -> for each specification
       creator.create() -> FormatterOwner
       candidate.append(owner.get()) -> owned clone
       owner scope ends -> creator result delete
  -> target.swap(candidate)
  -> candidate scope ends -> previous target delete
```

| 결과 | target | 임시 자원 |
| --- | --- | --- |
| validation 실패 | 호출 전 상태 | 없음 |
| creation 실패 | 호출 전 상태 | owner/candidate가 정리 |
| clone 실패 | 호출 전 상태 | current original + prior clones 정리 |
| allocation 실패 | 호출 전 상태 | stack unwinding으로 정리 |
| 성공 | 완성된 새 pipeline | 이전 target은 candidate destructor가 정리 |

이 Thread의 핵심은 RAII class를 하나 추가한 것이 아닙니다. **자원 정리와 상태 공개를 별개의 문제로 보고, target을 바꾸는 유일한 지점을 모든 실패 가능 작업 뒤로 옮긴 것**입니다.

## Thread 경계와 검증 범위

- `FormatterCreator::create()`는 성공 시 non-null owning pointer를 반환한다는 전제가 있으며 null 반환을 runtime에서 검사하지 않습니다.
- Specification escaping, quoting, 복합 grammar는 지원하지 않습니다.
- Pipeline clone 자체의 구현과 부분 생성자 정리는 앞 Thread의 책임입니다.
- 각 exact SHA의 diff/source/test를 확인했습니다. 이 환경에서는 branch를 로컬로 checkout하지 못해 test suite를 실행하지 않았으며, source에 존재하는 assertion만 기술했습니다.
