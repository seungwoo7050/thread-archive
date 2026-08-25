# Thread: Scalar 문자열을 대상별 projection으로 바꾸기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

이 Thread는 입력 문자열 하나를 `char`, `int`, `float`, `double`로 각각 다시 해석하는 방식이 아니라, 먼저 **하나의 검증된 `ScalarLiteral`**로 만든 뒤 각 target의 표현 가능성을 독립적으로 판단하는 흐름을 완성합니다.

최종 처리 순서는 다음과 같습니다.

```text
raw text
  -> ASCII/grammar 검증
  -> ScalarLiteral(kind, double value, float_suffix, negative_zero)
  -> char/int/float/double projection
  -> local classic-locale buffer에 네 줄 완성
  -> caller stream에 한 번의 write
```

이 분리는 parsing 성공과 target conversion 성공을 구분합니다. 예를 들어 `128`은 유효한 finite literal이지만 `char`로는 impossible이고, `1e-50`은 `double`로는 가능해도 `float`로는 nonzero underflow 때문에 impossible입니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `6a3d0461faab` | `feat(scalar): scalar 리터럴 문법과 종류 분류` | A | `PARSING, ARCH, NUMERIC` | 문자열을 target-independent literal model로 분류 |
| `a863f4899a93` | `feat(scalar): locale 고정 수치 추출과 경계 보존` | A | `NUMERIC, PARSING, HARD` | suffix grammar, overflow/underflow, textual negative zero를 정교화 |
| `fc7faa10dc66` | `test(scalar): literal 문법과 수치 범위 검증` | A | `TEST, NUMERIC, EDGE` | accepted/rejected grammar와 numeric edge를 고정 |
| `7cdcec341fb1` | `feat(scalar): 부동소수점 표현과 원자 출력 구현` | A | `NUMERIC, DETERMINISM, EXCEPTION` | float/double projection과 local rendering commit 도입 |
| `afea789fd753` | `test(scalar): 변환 가능성·출력·CLI 오류 검증` | A | `TEST, NUMERIC, DETERMINISM` | projection matrix, stream-state 독립성, no-partial-output 검증 |

## `6a3d0461faab` — `feat(scalar): scalar 리터럴 문법과 종류 분류`

새 internal representation은 입력의 종류와 원문에서 보존해야 할 정보를 한곳에 모읍니다.

```cpp
enum LiteralKind
{
    literal_character,
    literal_finite,
    literal_nan,
    literal_positive_infinity,
    literal_negative_infinity
};

struct ScalarLiteral
{
    LiteralKind kind;
    double value;
    bool float_suffix;
    bool negative_zero;
};
```

`parseScalarLiteral()`은 먼저 raw byte contract를 검사합니다.

- 빈 문자열 거부
- embedded NUL 거부
- 7-bit ASCII 범위를 넘는 byte 거부
- 앞·뒤·중간 위치와 무관하게 ASCII whitespace 거부

그 뒤 special literal, 한 글자 character, finite number 순으로 분류합니다.

```cpp
if (text == "nan" || text == "nanf")
    /* nan kind */;
if (text == "+inf" || text == "+inff" ||
    text == "-inf" || text == "-inff")
    /* signed infinity kind */;
if (text.size() == 1 && !isDigit(text[0]))
    /* character kind */;
```

Finite grammar는 optional sign, mantissa, optional exponent, optional lowercase `f` suffix를 직접 순회하고 마지막 index가 정확히 `text.size()`인지 확인합니다. 숫자 추출은 `std::istringstream`에 `std::locale::classic()`을 적용해 현재 process locale의 decimal separator에 의존하지 않습니다.

이 commit의 역할은 “값을 출력하는 법”보다 **입력을 한 번만 해석하고 그 결과를 네 projection이 공유하게 하는 것**입니다. 다만 이 시점의 최초 구현에는 후속 commit이 교정할 한계가 남습니다.

- `42f`처럼 point/exponent 없는 정수형 spelling에도 suffix를 허용합니다.
- nonzero 값이 extraction 중 0으로 underflow한 경우를 textual zero와 구분하지 않습니다.
- `negative_zero`를 parsed `double == 0.0`에 기대므로 lexical 의미 보존이 extraction 결과에 종속됩니다.
- character 범위가 후속 grammar만큼 명확하지 않습니다.

따라서 이 SHA의 representation은 최종 model의 형태를 만들었지만, 모든 edge semantics까지 완성한 상태는 아닙니다.

## `a863f4899a93` — `feat(scalar): locale 고정 수치 추출과 경계 보존`

이 commit은 parser를 세 단계로 더 분명히 나눕니다.

1. Special literal을 explicit kind/suffix로 생성
2. Finite grammar를 구조적으로 검증
3. Textual mantissa가 zero인지 확인한 뒤 numeric extraction

### Float suffix grammar

Finite validator는 integer/fraction digit 수와 point/exponent 존재 여부를 따로 기록합니다.

```cpp
if (index != text.size() ||
    (float_suffix && !has_point && !has_exponent))
    throw ScalarParseError();
```

따라서 다음 구분이 생깁니다.

| 입력 | 결과 |
| --- | --- |
| `42` | finite double spelling |
| `42.` | finite double spelling |
| `1e2` | finite double spelling |
| `1e2f` | finite float spelling |
| `42f` | invalid — point/exponent 없는 suffix |
| `f` | 한 글자 character |

### Textual zero 보존

`allMantissaDigitsAreZero()`는 exponent와 suffix 이전의 mantissa digit이 모두 `0`인지 검사합니다. `-0`, `-0.0`, `-0e20`, `-0.0f`는 extraction 결과가 우연히 0이어서가 아니라 입력 spelling이 zero이므로 `negative_zero`를 보존합니다.

```cpp
all_zero = allMantissaDigitsAreZero(text);
literal.negative_zero = text[0] == '-' && all_zero;
if (all_zero)
    literal.value = literal.negative_zero ? -0.0 : 0.0;
```

이 별도 flag가 필요한 이유는 일반 비교에서 `-0.0 == 0.0`이 참이기 때문입니다. Output 단계가 원문의 부호 의도를 유지하려면 값 외의 정보가 필요합니다.

### Overflow와 nonzero underflow

Numeric extraction은 classic locale과 complete-consumption을 요구하고 finite double 범위를 벗어난 결과를 거부합니다.

```cpp
input >> value;
if (input.fail() || !input.eof() || value != value ||
    value > std::numeric_limits<double>::max() ||
    value < -std::numeric_limits<double>::max())
    throw ScalarParseError();
```

Textual mantissa가 zero가 아닌데 추출 결과가 `0.0`이면 underflow로 간주합니다.

```cpp
literal.value = extractFiniteValue(text, float_suffix);
if (literal.value == 0.0)
    throw ScalarParseError();
```

이 결정으로 `1e-9999` 같은 input은 유효한 zero spelling으로 조용히 바뀌지 않습니다. 반대로 `-0e20`은 zero임이 text에서 확인되므로 정확히 negative zero로 남습니다.

### 분류와 projection의 경계

Parser는 finite value가 `int`나 `float`에 들어가는지 결정하지 않습니다. 여기서 보장하는 것은 유효한 character/finite/special literal을 안정적인 intermediate form으로 만드는 것뿐입니다. 각 target의 range와 output wording은 `ScalarConverter`가 담당합니다.

## `fc7faa10dc66` — `test(scalar): literal 문법과 수치 범위 검증`

Literal test는 helper를 종류별로 나눠 `kind`, `value`, `float_suffix`, `negative_zero`를 직접 검사합니다.

### 허용 사례

- character: `a`, `f`, `+`, `-`, `.`
- finite: `0`, `9`, `+42`, `-42`, `42.`, `.5`, `1.e2`, `1e2`, `1e2f`
- negative zero: `-0`, `-0.0`, `-0e20`, `-0.0f`
- special: `nan`, `nanf`, `+inf`, `+inff`, `-inf`, `-inff`

### 거부 사례

- empty/whitespace/embedded NUL
- repeated sign/point
- exponent digit 누락
- `42f`, uppercase suffix `42F`
- hexadecimal, locale comma, trailing garbage
- unsigned infinity `inf`, signed nan `+nan`
- double overflow `1e309`
- nonzero underflow `1e-9999`

이 matrix는 parser의 accepted language를 문서가 아니라 executable examples로 정의합니다. 다만 수천 개의 임의 입력을 탐색하는 property test는 아니며, 선택된 대표 경계에 대한 deterministic unit regression입니다.

## `7cdcec341fb1` — `feat(scalar): 부동소수점 표현과 원자 출력 구현`

이 SHA 직전에는 character와 integer writer가 이미 존재합니다. 이 commit은 float/double writer를 추가하고, 네 줄을 caller stream에 직접 순차 출력하던 구조를 local rendering으로 바꿉니다.

### Character projection

```cpp
bool canProjectChar(const ScalarLiteral &literal)
{
    return isValue(literal) && literal.value > -1.0 &&
           literal.value < 128.0;
}
```

범위가 `[-0, 127]` 정수만을 검사하는 형태가 아니라 open interval `(-1, 128)`인 이유는 `static_cast<int>`가 소수부를 0 방향으로 버리기 때문입니다. 따라서 `-0.5`는 `0`으로 projection될 수 있지만 displayable character는 아닙니다. 정수화된 값이 32~126이면 quoted character, 그 밖이면 `Non displayable`, 범위 밖이나 special이면 `impossible`입니다. Quote와 backslash는 별도로 escape합니다.

### Integer projection

```cpp
const double lower =
    static_cast<double>(std::numeric_limits<int>::min()) - 1.0;
const double upper =
    static_cast<double>(std::numeric_limits<int>::max()) + 1.0;

return isValue(literal) && literal.value > lower &&
       literal.value < upper;
```

이 역시 truncation을 고려한 경계입니다. 예를 들어 `2147483647.9`는 0 방향으로 잘라 `INT_MAX`가 되지만 `2147483648.0`은 impossible입니다. Cast를 먼저 수행한 뒤 overflow를 확인하지 않으므로 범위 밖 floating-to-integer conversion을 피합니다.

### Float projection

```cpp
bool canProjectFloat(const ScalarLiteral &literal)
{
    const double maximum = std::numeric_limits<float>::max();

    if (!isValue(literal) || literal.value < -maximum ||
        literal.value > maximum)
        return false;
    const float value = static_cast<float>(literal.value);
    return literal.value == 0.0 || value != 0.0f;
}
```

두 가지 불가능 상태를 분리해 막습니다.

- `double` value가 finite float 최대 범위 밖임
- input은 nonzero인데 float cast가 0이 되는 underflow

NaN과 infinity는 range projection이 아니라 canonical spelling(`nanf`, `+inff`, `-inff`)으로 출력합니다.

### Stable finite representation

`finiteNumber()`는 caller locale 대신 classic locale을 사용하고, float/double에 각각 `digits10` precision을 적용합니다. Decimal point나 exponent가 없는 결과에는 `.0`을 붙입니다. Negative zero flag가 있으면 stream의 sign 처리에 의존하지 않고 `-0`에서 시작합니다.

```cpp
if (value == 0.0 && negative_zero)
    result = "-0";
/* ... */
if (result.find('.') == std::string::npos &&
    result.find('e') == std::string::npos &&
    result.find('E') == std::string::npos)
    result += ".0";
```

### Local render 후 commit

```cpp
std::ostringstream rendered;

rendered.imbue(std::locale::classic());
writeCharacter(literal, rendered);
writeInteger(literal, rendered);
writeFloating(literal, rendered);
writeDouble(literal, rendered);
const std::string result = rendered.str();

output.write(result.data(),
             static_cast<std::streamsize>(result.size()));
```

Parsing이나 네 projection을 구성하는 중간에 예외가 나면 caller stream에는 아무 byte도 전달되지 않습니다. Caller가 설정한 scientific/showpos/fill/width/precision/locale도 local stream에 전달되지 않으므로 converter output을 바꾸지 않습니다.

여기서 “원자 출력”은 **내부 계산 실패 전에 부분 line을 caller stream에 쓰지 않는다**는 의미입니다. 마지막 `output.write()`를 underlying streambuf가 물리적으로 전부 쓰는지까지 transaction으로 보장하는 것은 아닙니다.

## `afea789fd753` — `test(scalar): 변환 가능성·출력·CLI 오류 검증`

Test는 네 projection의 결과를 exact string으로 비교합니다.

```text
input: 42.5
char: '*'
int: 42
float: 42.5f
double: 42.5
```

주요 경계는 다음과 같습니다.

| 입력 | 확인하는 결정 |
| --- | --- |
| `-0` | float `-0.0f`, double `-0.0` |
| `31`, `32`, `126`, `127`, `128` | control/displayable/DEL/out-of-range character 구분 |
| `39`, `92` | quote와 backslash escaping |
| `-0.5`, `127.9` | truncation toward zero 후 char/int 판정 |
| `INT_MAX + fraction`, `INT_MIN - fraction` | cast 전 open-boundary range 검사 |
| `nanf`, `+inf`, `-inff` | canonical special spelling |
| float max 초과 | float만 impossible, double은 유지 |
| `1e-50` | float underflow만 impossible |
| `1.23456789` | float/double precision 차이 |

또한 caller `ostringstream`에 scientific, showpos, left alignment, fill, width, precision과 comma decimal locale을 미리 설정한 뒤에도 output byte가 동일하고 원래 formatting state가 보존되는지 검사합니다.

Invalid `42f`는 `InvalidScalar("invalid scalar literal")`로 변환되고 caller output은 빈 문자열이어야 합니다. CLI fixture도 성공 input의 네 줄과 invalid input의 stderr/status, 빈 stdout을 비교합니다.

이 commit은 출력 text와 state isolation을 강하게 검증하지만, 모든 floating implementation에서 decimal round-trip을 증명하는 테스트는 아닙니다. 사용 precision은 코드가 선택한 `digits10`이며 exact source value 복원을 목표로 하는 `max_digits10` 정책과는 다릅니다.

## 최종 projection table

| Literal kind | char | int | float | double |
| --- | --- | --- | --- | --- |
| character/finite, 범위 안 | truncation 후 displayability | truncation 후 decimal | finite text + `f` | finite text |
| finite, target overflow/underflow | `impossible` | `impossible` | `impossible` | parser가 finite double을 보장 |
| NaN | `impossible` | `impossible` | `nanf` | `nan` |
| +infinity | `impossible` | `impossible` | `+inff` | `+inf` |
| -infinity | `impossible` | `impossible` | `-inff` | `-inf` |
| finite but non-displayable char | `Non displayable` | 정상 가능 | 정상 가능 | 정상 가능 |

이 Thread의 최종 불변 조건은 “한 문자열을 네 번 parse한다”가 아니라 다음과 같습니다.

> Parsing은 입력의 의미를 한 번 고정하고, 각 target은 그 의미를 표현할 수 있는지만 판단한다. 네 결과가 모두 완성되기 전에는 caller output을 수정하지 않는다.

## Thread 경계와 검증 범위

- Accepted grammar는 decimal ASCII spelling과 명시된 special literal에 한정되며 hex, locale comma, whitespace는 지원하지 않습니다.
- Float/double text는 project가 선택한 precision 정책이며 모든 bit pattern의 round-trip serializer는 아닙니다.
- Underlying output stream의 I/O failure를 rollback하는 기능은 없습니다.
- 각 exact SHA의 diff/source/test를 검사했습니다. 이 환경에서는 branch를 로컬로 checkout하지 못해 build와 test 실행 성공을 주장하지 않습니다.
