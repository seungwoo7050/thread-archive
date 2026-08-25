# Thread 05. 엄격한 실행 설정값 경계

## 무엇을 교정하는 Thread인가

command-line option은 문자열이지만 server 내부에서는 port, timeout, byte limit, connection count처럼 서로 다른 정수 타입과 의미를 가집니다. C library의 `strtol`/`strtoul`은 편리하지만 앞 공백과 `+`/`-` 부호를 허용하고, overflow를 errno와 포화값으로 표현합니다. 반환값을 곧바로 더 좁은 타입으로 cast하면 “문법상 순수한 10진수이며 대상 타입 안에 들어온다”는 runtime contract를 잃을 수 있습니다.

이 Thread의 최종 규칙은 다음과 같습니다.

```text
입력 문자열의 모든 문자가 '0'..'9'
AND 누적값이 대상 maximum을 넘지 않음
AND option별 semantic lower/upper bound를 만족
```

즉 parsing과 narrowing을 두 단계로 나누지 않고, 대상 타입의 범위 안에서 한 digit씩 값을 구성합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `52b6f1ce8f0f` | `feat(config): 기본 실행 인자 해석 모듈 구성` | B | `BUILD, RESILIENCE` | 기본값, port parser, option 진입점을 분리 |
| 2 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | `VERIFICATION, IRC_PROTOCOL, RISK` | public CLI의 exit/stdout/stderr 계약을 exact assertion으로 고정 |
| 3 | `b6c10bc51937` | `fix(config): 서버 크기 옵션을 오버플로 없이 해석` | A | `DEBUG, RESILIENCE, RISK` | target-aware digit parser로 문법·overflow·narrowing 문제 제거 |
| 4 | `5d1286620994` | `test(config): 크기 옵션 경계와 오류 입력 검증` | B | `VERIFICATION, RESILIENCE, RISK` | 부호·공백·대상 타입 최대값 초과를 회귀 사례로 고정 |

## 1. `52b6f1ce8f0f` — 설정값 owner를 만들다

`RuntimeConfig`는 application 정책의 기본값을 보관합니다.

```text
rateLimitCount = 24
rateLimitWindowSeconds = 3
idleTimeoutSeconds = 120
pingTimeoutSeconds = 30
registrationTimeoutSeconds = 30
```

transport에 직접 적용되는 `maxPendingBytes`, `maxConnections` 등은 `Server::Config&`를 통해 수정하는 형태로 확장됩니다. 이 분리는 option이 어느 계층의 동작을 바꾸는지 드러냅니다.

초기 port parser는 `std::strtol`을 사용합니다.

```cpp
char* end = NULL;
const long port = std::strtol(value, &end, 10);
if (!value[0] || *end != '\0' || port <= 0 || port > 65535)
    throw std::runtime_error("port must be an integer from 1 to 65535");
```

숫자가 아닌 suffix와 값 범위는 검사하지만 C 정수 parser가 허용하는 lexical grammar를 그대로 상속합니다. 따라서 `+6667`이나 앞 공백이 있는 입력도 값으로 해석될 수 있습니다.

초기 `parseOptions`는 세 번째 이후 인자를 아직 지원하지 않고 첫 unknown option을 오류로 만듭니다. 후속 operational commit들이 `--idle-timeout=`, `--rate-limit=`, `--max-pending-bytes=` 등을 이 진입점에 추가합니다.

## 2. `e5e6c57db80d` — CLI도 protocol처럼 public contract다

`tests/irc_contract.py`는 server process를 직접 실행하여 다음 세 요소를 함께 비교합니다.

- exit status
- stdout 전체
- stderr 전체 또는 명시된 허용 집합

예를 들어 인자 부족은 단순히 실패해야 하는 것이 아니라, 현재 option 목록을 포함한 정확한 usage를 stderr에 쓰고 exit 1이어야 합니다. invalid port, zero timeout, 잘못된 `COUNT:SECONDS`, unknown option도 각기 다른 오류 문구를 가집니다.

```python
check_cli(
    manifest,
    binary,
    "zero_timeout",
    ["6667", "contract-secret", "--idle-timeout=0"],
    expected_stderr=
        "irc-relay-server: idle timeout must be a positive integer\n",
)
```

이 commit은 기존 behavior를 characterization합니다. parser가 부호와 앞 공백을 거부한다는 요구까지는 아직 넣지 않았고, `--max-pending-bytes=abc`의 stderr에는 플랫폼별 errno suffix 차이를 허용합니다. 즉 test가 존재한다는 사실만으로 strict parsing이 완성된 것은 아닙니다.

## 3. 기존 정수 parser가 갖던 세 종류의 모호성

operational option이 늘어난 뒤 parsing helper는 대략 두 계열이었습니다.

```cpp
std::strtol(value.c_str(), &end, 10);   // timeout, port
std::strtoul(value.c_str(), &end, 10);  // size_t 계열
```

### lexical grammar가 넓다

C parser는 선행 whitespace와 optional sign을 허용합니다.

- `+1`은 양수 1
- ` 1`도 양수 1
- `-1`을 `strtoul`에 주면 unsigned 변환 규칙에 따른 큰 값

CLI option을 “ASCII decimal digits only”로 정의하려면 이 behavior는 너무 관대합니다.

### overflow를 반환값만으로 구별하기 어렵다

`strtoul` overflow는 `ULONG_MAX`를 반환하고 `errno=ERANGE`를 설정합니다. 기존 `parseSize`는 end pointer만 검사하고 errno를 확인하지 않았으므로 거대한 문자열을 정상적인 최대값처럼 받을 수 있습니다.

### source type과 destination type이 다를 수 있다

`unsigned long`의 범위가 `std::size_t`보다 넓은 플랫폼에서 정상적인 `strtoul` 결과라도 cast 시 narrowing될 수 있습니다. 반대로 port는 `long`으로 읽은 뒤 최종적으로 더 작은 범위만 허용합니다. parser가 destination maximum을 모르면 portable한 범위 검사를 끝낼 수 없습니다.

## 4. `b6c10bc51937` — 값을 만들기 전에 overflow를 판정하다

수정은 generic unsigned decimal parser를 도입합니다.

```cpp
template <typename Unsigned>
Unsigned parseUnsignedDecimal(const std::string& value,
                              Unsigned maximum,
                              const std::string& errorMessage) {
    if (value.empty())
        throw std::runtime_error(errorMessage);

    Unsigned parsed = 0;
    for (std::string::size_type i = 0; i < value.size(); ++i) {
        if (value[i] < '0' || value[i] > '9')
            throw std::runtime_error(errorMessage);

        const Unsigned digit = static_cast<Unsigned>(value[i] - '0');
        if (parsed > maximum / 10 ||
            (parsed == maximum / 10 && digit > maximum % 10))
            throw std::runtime_error(errorMessage);

        parsed = static_cast<Unsigned>(parsed * 10 + digit);
    }
    return parsed;
}
```

핵심은 `parsed * 10 + digit`을 수행하기 전에 maximum을 넘는지 판정한다는 것입니다.

```text
parsed < maximum / 10              → 다음 digit 안전
parsed > maximum / 10              → 반드시 overflow
parsed == maximum / 10             → digit <= maximum % 10일 때만 안전
```

따라서 검사 자체가 overflow하지 않습니다. 모든 문자를 직접 확인하므로 공백, `+`, `-`, locale 영향을 받는 다른 문자를 허용하지 않습니다.

### option별 target과 semantic bound

port는 `unsigned short`의 maximum으로 먼저 parse하고 0을 별도로 거부합니다.

```cpp
const unsigned short port = parseUnsignedDecimal<unsigned short>(
    text,
    std::numeric_limits<unsigned short>::max(),
    "port must be an integer from 1 to 65535");
if (port == 0)
    throw ...;
```

size option은 `std::size_t` 자체를 target으로 사용합니다.

```cpp
return parseUnsignedDecimal<std::size_t>(
    value,
    std::numeric_limits<std::size_t>::max(),
    name + " must be an unsigned integer");
```

timeout은 storage type maximum이 아니라 policy maximum 86400을 넘지 않도록 parse하고, 0을 거부합니다.

```cpp
const unsigned int parsed = parseUnsignedDecimal<unsigned int>(
    value, 86400U, name + " must be a positive integer");
```

이 구조는 lexical validation, arithmetic overflow, narrowing, business range를 각각 어느 지점에서 보장하는지 분명히 합니다.

### 보장하지 않는 것

- Unicode digit이나 locale 숫자는 지원하지 않습니다. 의도적으로 ASCII decimal만 허용합니다.
- option 간 관계, 예를 들어 `ping-timeout <= idle-timeout` 같은 교차 제약은 검사하지 않습니다.
- `rateLimitCount == 0`, `maxConnections == 0`처럼 0이 특별한 의미를 갖는지는 각 consumer의 policy입니다. parser는 size option의 0 자체를 허용합니다.

## 5. `5d1286620994` — host 크기에 맞춘 boundary test

회귀 test는 단순히 매우 큰 상수를 하나 넣지 않고 실행 플랫폼의 pointer width로 `size_t` 범위를 계산합니다.

```python
size_t_overflow = str(1 << (struct.calcsize("P") * 8))
```

64-bit 환경에서는 `2^64`, 32-bit 환경에서는 `2^32`가 되어 정확히 `size_t::max + 1`을 겨냥합니다.

검증 사례는 세 그룹입니다.

### port

```text
+6667
-6667
" 6667"
"6667 "
65536
```

모두 같은 port 오류와 exit 1을 요구합니다.

### positive timeout

```text
+1
-1
" 1"
"1 "
매우 긴 decimal overflow
```

모두 `must be a positive integer`로 수렴합니다.

### size 계열

```text
--max-connections=+1
--max-connections=-1
앞/뒤 공백
--max-pending-bytes=(size_t max + 1)
--rate-limit=(size_t max + 1):1
```

각 option 이름에 맞는 `must be an unsigned integer`를 exact stderr로 비교합니다. 이 test는 parser가 값을 잘못 받아 server start까지 진행하는 것을 막습니다.

## 최종 parsing 계약

| option 값 종류 | lexical rule | 산술 범위 | 추가 의미 규칙 |
| --- | --- | --- | --- |
| port | ASCII digit만 | `unsigned short` 최대값 | 1..65535 |
| timeout | ASCII digit만 | 최대 86400 | 0 금지 |
| byte/connection/count | ASCII digit만 | `size_t` 최대값 | 0의 의미는 consumer가 결정 |
| rate limit | `COUNT:SECONDS` 한 개의 구분 | count=`size_t`, seconds≤86400 | seconds는 양수 |
| unknown option | 일치하는 prefix 없음 | 해당 없음 | 즉시 오류 |

## Thread 경계

- 각 option이 server/application state에 어떤 resource limit을 적용하는지는 Thread 04입니다.
- 실제 Linux/macOS에서 같은 parser와 exact CLI test를 반복하는 것은 Thread 09의 CI 책임입니다.
- parser error 뒤 errno suffix를 어떻게 표시할지 같은 top-level diagnostic 정책은 이 Thread가 완전히 재설계하지 않습니다. fix 이후 parser 자체는 C conversion errno에 의존하지 않습니다.

> 검사 범위: `RuntimeConfig`의 각 SHA diff와 `tests/irc_contract.py`의 boundary case를 source로 확인했습니다. 플랫폼별 binary 실행 결과는 직접 생성하지 않았습니다.
