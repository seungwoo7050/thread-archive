# Thread: RPN 계산에서 정의되지 않은 signed 산술을 실행 전에 차단하기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

이 Thread는 space-separated RPN expression을 `long` stack으로 계산하면서, token parsing과 `+ - * /`의 모든 위험한 경계를 **실제 signed 연산 전에** 검사하는 과정을 다룹니다.

핵심 불변 조건은 다음과 같습니다.

- 숫자 token은 `LONG_MIN`부터 `LONG_MAX`까지 정확히 읽고 범위 밖 magnitude를 거부합니다.
- Operator는 두 operand가 있을 때만 실행하며 오른쪽 operand를 먼저 pop해 순서를 보존합니다.
- Overflow 가능성이 있는 signed expression을 평가하지 않습니다.
- Division by zero는 잘못된 expression, `LONG_MIN / -1`은 arithmetic overflow로 구분합니다.
- 모든 token을 처리한 뒤 stack에 값이 정확히 하나만 남아야 합니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `57a25e8475ab` | `feat(rpn): signed token과 stack 문법 처리` | A | `PARSING, NUMERIC, CORE` | decimal token 범위 검사와 stack grammar 도입 |
| `e1641a714172` | `feat(rpn): overflow 검사 산술 연산 구현` | S | `NUMERIC, HARD, CORE` | 네 연산을 precondition 기반 checked arithmetic으로 구현 |
| `aa0cc5e3e063` | `test(rpn): 산술 경계와 잘못된 token 검증` | A | `TEST, NUMERIC, EDGE` | literal/operation boundary와 invalid grammar를 회귀 고정 |

## `57a25e8475ab` — `feat(rpn): signed token과 stack 문법 처리`

초기 evaluator는 expression을 ASCII space로 나누고 각 token을 `long`으로 parse합니다. Tab이나 newline은 separator가 아니라 token 내부 byte가 되므로 numeric parser가 거부합니다.

```cpp
while (index < expression.size() && expression[index] == ' ')
    ++index;
start = index;
while (index < expression.size() && expression[index] != ' ')
    ++index;
```

`parseLong()`은 부호를 떼어낸 뒤 `unsigned long magnitude`를 누적합니다. Negative 범위는 positive 범위보다 한 칸 넓으므로 limit을 다르게 둡니다.

```cpp
limit = static_cast<unsigned long>(
    std::numeric_limits<long>::max());
if (negative)
    ++limit;
```

Digit를 더하기 전에 다음 조건을 검사합니다.

```cpp
if (magnitude > (limit - digit) / 10)
    throw std::overflow_error("rpn overflow");
magnitude = magnitude * 10 + digit;
```

따라서 `magnitude * 10 + digit` 자체가 unsigned 범위를 넘기거나 허용된 signed magnitude를 초과하기 전에 중단됩니다. Negative magnitude가 정확히 `LONG_MAX + 1`이면 signed cast나 unary minus로 표현하려 하지 않고 `LONG_MIN`을 직접 선택합니다.

```cpp
if (!negative)
    value = static_cast<long>(magnitude);
else if (magnitude == limit)
    value = std::numeric_limits<long>::min();
else
    value = -static_cast<long>(magnitude);
```

이 commit의 evaluator는 아직 operator를 처리하지 않습니다. 모든 token을 number로 push한 뒤 최종 stack size가 1인지 검사하므로, 단일 signed number만 유효한 상태입니다. 다음 commit이 이 stack skeleton에 연산을 연결합니다.

## `e1641a714172` — `feat(rpn): overflow 검사 산술 연산 구현`

Signed overflow는 C++에서 “잘못된 값이 나온다”가 아니라 undefined behavior입니다. 따라서 `left + right`를 먼저 수행하고 범위를 확인하는 방식은 사용할 수 없습니다. 각 helper는 실제 연산이 안전하다는 것을 먼저 증명합니다.

### Addition

```cpp
if ((right > 0 &&
     left > std::numeric_limits<long>::max() - right) ||
    (right < 0 &&
     left < std::numeric_limits<long>::min() - right))
    throw std::overflow_error("rpn overflow");
return left + right;
```

`right > 0`일 때는 upper bound, `right < 0`일 때는 lower bound만 검사합니다. 비교식 안의 `MAX - right`, `MIN - right`는 각 branch의 sign 조건 때문에 representable합니다.

### Subtraction

```cpp
if ((right > 0 &&
     left < std::numeric_limits<long>::min() + right) ||
    (right < 0 &&
     left > std::numeric_limits<long>::max() + right))
    throw std::overflow_error("rpn overflow");
return left - right;
```

Subtraction은 right의 부호에 따라 반대쪽 경계를 넘는지 검사합니다. `right`를 먼저 `-right`로 바꾸지 않으므로 `-LONG_MIN` 같은 undefined intermediate도 만들지 않습니다.

### Multiplication

가장 까다로운 경우는 `LONG_MIN`의 절댓값입니다. `-LONG_MIN`은 표현할 수 없으므로 `magnitudeOf()`는 한 칸 이동한 뒤 unsigned로 복원합니다.

```cpp
unsigned long magnitudeOf(long value)
{
    if (value >= 0)
        return static_cast<unsigned long>(value);
    return static_cast<unsigned long>(-(value + 1)) + 1;
}
```

두 magnitude가 0이 아니면 곱셈 전에 division 형태로 한계를 검사합니다.

```cpp
unsigned long limit =
    static_cast<unsigned long>(std::numeric_limits<long>::max());
if (negative)
    ++limit;
if (left_magnitude > limit / right_magnitude)
    throw std::overflow_error("rpn overflow");
```

결과가 음수이고 magnitude가 정확히 `LONG_MAX + 1`이면 `LONG_MIN`을 직접 반환합니다. 그 외에만 안전한 범위의 unsigned 값을 `long`으로 바꾸고 부호를 붙입니다.

### Division

```cpp
if (right == 0)
    throw std::invalid_argument("invalid rpn expression");
if (left == std::numeric_limits<long>::min() && right == -1)
    throw std::overflow_error("rpn overflow");
return left / right;
```

`LONG_MIN / -1`은 수학적으로 `LONG_MAX + 1`이므로 유일한 signed division overflow입니다. Division by zero와 오류 종류를 분리해 caller가 malformed expression과 numeric overflow를 구분할 수 있습니다.

### Stack 적용 순서

```cpp
right = stack.back();
stack.pop_back();
left = stack.back();
stack.pop_back();
stack.push_back(applyOperator(left, right, token[0]));
```

Subtraction과 division에서 operand 순서를 보존하려면 먼저 pop한 값을 right로 사용해야 합니다. 연산이 예외를 던지면 local stack은 이미 pop된 상태지만 evaluator가 즉시 종료되고 외부에 공개된 mutable state가 없으므로 rollback 대상은 없습니다.

`+`와 `-` 한 글자 token은 `isOperator()`가 먼저 인식합니다. Signed number는 `+2`, `-3`처럼 부호 뒤에 최소 한 digit이 있어야 합니다.

## `aa0cc5e3e063` — `test(rpn): 산술 경계와 잘못된 token 검증`

Unit test는 정상 산술뿐 아니라 `numeric_limits<long>`에서 직접 만든 boundary expression을 사용합니다.

| 대상 | 대표 검증 |
| --- | --- |
| literal parsing | `LONG_MAX`, `LONG_MIN`, 양·음 범위 초과 |
| addition | `MAX 1 +`, `MIN -1 +`, 경계에 0 더하기 |
| subtraction | `MIN 1 -`, `MAX -1 -`, 경계에서 0 빼기 |
| multiplication | 네 sign 조합, `MIN 1 *`, `MAX 2 *` |
| division | truncation toward zero, `MIN 1 /`, `MIN -1 /`, zero divisor |
| stack grammar | missing operand, leftover operand, empty/spaces-only |
| token grammar | `%`, `--2`, decimal point, exponent, suffix, tab/newline, NUL, non-ASCII |

오류 message도 contract로 검사합니다.

- 문법·stack·zero divisor: `std::invalid_argument("invalid rpn expression")`
- literal/operation overflow: `std::overflow_error("rpn overflow")`

Header compile translation unit은 `cppf/RpnEvaluator.hpp`를 반복 include하고 `RpnEvaluator::evaluate("2 3 +")`를 호출합니다. 이 시점에는 compiler command의 include path를 `include/`로만 격리하지 않았으며, 그 stronger public-only 검사는 verification Thread의 `4bbbfd191669`에서 추가됩니다.

이 테스트는 대표 경계와 sign 조합을 폭넓게 다루지만, 임의 길이 expression 전체를 생성하는 exhaustive proof는 아닙니다. 후속 verification Thread의 fixed-seed property test가 작은 범위의 이항 연산을 추가로 반복합니다.

## 최종 실행 규칙

```text
expression을 ASCII space로 token화
  -> operator token?
       yes: stack size >= 2 확인
            right pop -> left pop
            checked operation
            result push
       no:  signed decimal parse
            범위 안 long push
  -> 모든 token 처리 뒤 stack size == 1 확인
  -> stack.back() 반환
```

| 연산 | 실행 전 검사 | 실패 종류 |
| --- | --- | --- |
| `+` | sign별 `MAX-right` / `MIN-right` | overflow_error |
| `-` | sign별 `MIN+right` / `MAX+right` | overflow_error |
| `*` | unsigned magnitude와 sign별 limit division | overflow_error |
| `/` | zero divisor, `MIN/-1` | invalid_argument / overflow_error |

이 Thread의 핵심은 boundary case를 나중에 보정하는 것이 아니라, **undefined signed expression 자체를 프로그램이 한 번도 평가하지 않도록 조건을 배치한 것**입니다.

## Thread 경계와 검증 범위

- Token separator는 ASCII space 하나이며 tab/newline을 whitespace로 일반화하지 않습니다.
- 지원 연산자는 `+ - * /` 네 개뿐입니다.
- `long`의 실제 폭을 포함한 지원 platform 선언은 verification Thread의 LP64 check가 담당합니다.
- Exact SHA의 diff/source/test를 확인했으며, 이 환경에서는 test binary를 실행하지 않았습니다.
