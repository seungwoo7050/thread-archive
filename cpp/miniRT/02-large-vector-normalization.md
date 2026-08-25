# Thread 2. 큰 유한 벡터 정규화 — 값은 유효하지만 중간 계산이 넘치는 경우

`Vec3`의 세 성분이 모두 유한하더라도 제곱의 합을 먼저 계산하면 중간값이 `double` 범위를
넘을 수 있습니다. 이 Thread는 길이 계산식을 바꿔 **유한 입력의 방향을 잃지 않는 것**과,
그 위험을 정확히 재현하는 한 개의 회귀 사례를 다룹니다.

> **조사 범위**
>
> 이 문서는 `cpp/miniRT` 브랜치의 표시된 exact SHA와 그 parent diff만을 근거로 작성했습니다.
> 다른 브랜치나 최종 HEAD의 구현을 과거 커밋에 소급하지 않았습니다. GitHub에서 diff와
> 해당 SHA의 source/test를 확인했으며, 로컬 checkout을 확보하지 못해 build·test·benchmark는
> 실행하지 않았습니다. 따라서 아래의 실행 결과는 repository에 들어 있는 assertion과
> expected value의 의미를 설명한 것이며, 이 작업 환경에서 새로 측정한 결과가 아닙니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `aa92a87c98a3` | `fix(math): 큰 유한 벡터를 안정적으로 정규화` | A | `DEBUG, EDGE, RAY_PIPELINE` | `sqrt(x²+y²+z²)`의 중간 overflow를 `std::hypot`으로 제거 |
| 2 | `ff18d1cc3afc` | `test(math): 큰 유한 벡터 정규화 검증` | B | `TEST, EDGE` | `(1e308,0,0)`이 단위 x축으로 정규화되는 경계를 회귀 테스트로 고정 |

## `aa92a87c98a3` — overflow가 발생하는 위치를 없애다

**중요도** `A` · **태그** `DEBUG, EDGE, RAY_PIPELINE`

수정 전 `Vec3::length()`는 `lengthSquared()`를 먼저 계산했습니다.

```cpp
double Vec3::lengthSquared() const {
    return x * x + y * y + z * z;
}

double Vec3::length() const {
    return std::sqrt(lengthSquared());
}
```

이 방식은 수학적으로는 맞지만 floating-point 실행 순서에서는 안전하지 않습니다.
예를 들어 `x = 1e308`은 유한하지만 `x * x`는 `double` 최대값을 넘으므로 `inf`가 됩니다.
그 뒤 `sqrt(inf)`도 `inf`이고, 정규화는 각 성분을 `inf`로 나눕니다. 방향이 명확한
`(1e308, 0, 0)`조차 `(0, 0, 0)`에 가까운 잘못된 결과나 유효하지 않은 값으로 무너질 수
있습니다.

커밋은 길이 계산만 다음과 같이 바꿉니다.

```diff
 double Vec3::length() const {
-    return std::sqrt(lengthSquared());
+    return std::hypot(x, y, z);
 }
```

`std::hypot`은 성분을 적절히 scaling해 제곱합의 중간 overflow와 불필요한 underflow를 피한
뒤 길이를 계산합니다. `normalize`의 zero/near-zero 처리나 public API는 바꾸지 않습니다.
따라서 수정의 핵심은 “정규화 알고리즘 전체를 교체”하는 것이 아니라, 그 알고리즘이 의존하는
길이 값이 유한 입력에서 불필요하게 `inf`가 되지 않도록 만드는 것입니다.

### 바뀐 불변 조건

```text
모든 성분이 유한하고 벡터가 0이 아니라면
길이 계산의 중간 제곱 때문에 방향 정보가 사라져서는 안 된다.
```

이 커밋이 보장하지 않는 범위도 분명합니다.

- 입력 자체에 `NaN`이나 `inf`가 들어 있는 경우까지 유효한 방향을 만들어 주지 않습니다.
- `isNearZero()`의 tolerance 정책을 바꾸지 않습니다.
- ray/normal을 어느 call site에서 정규화해야 하는지는 별도의 책임입니다.

## `ff18d1cc3afc` — 일반적인 단위 테스트가 놓치는 정확한 값을 고정하다

**중요도** `B` · **태그** `TEST, EDGE`

작은 벡터만 검사하면 수정 전후가 모두 통과합니다. 회귀 테스트는 중간 제곱이 넘치지만 입력
자체는 유한한 값을 직접 사용합니다.

```cpp
const ray::Vec3 huge(1.0e308, 0.0, 0.0);
const ray::Vec3 unit = ray::normalize(huge);

require(unit == ray::Vec3(1.0, 0.0, 0.0),
        "large finite vector normalization");
```

이 case의 힘은 결과가 애매하지 않다는 데 있습니다. 원래 방향이 정확히 x축이므로 정상 결과도
`(1,0,0)`이어야 합니다. 상대 오차가 큰 임의의 세 성분을 비교하는 대신, overflow 제거 여부를
명확한 축 방향으로 판별합니다.

이 테스트는 다음을 증명하려고 합니다.

- `1e308`이 유한한 상태에서 길이 계산이 `inf`로 무너지지 않는다.
- 정규화 결과가 방향을 보존한다.
- 이후 누군가 `length()`를 다시 `sqrt(lengthSquared())`로 되돌리면 해당 case가 회귀를 드러낸다.

반면 모든 크기·방향 조합의 수치 오차나 subnormal 영역까지 포괄하지 않습니다. 이 Thread의
목표는 특정 root cause인 **제곱합 중간 overflow**를 제거하고 그 재현값을 고정하는 것입니다.

## 최종 상태와 Thread 경계

```text
Vec3 components
    -> std::hypot(x, y, z)
    -> finite magnitude for a large finite nonzero vector
    -> normalize by that magnitude
    -> direction preserved
```

카메라 frame, surface normal, reflection ray가 이 함수의 소비자일 수 있지만, 각각의 기하
의미와 call-site validation은 이 Thread에서 다루지 않습니다. 여기서 확립한 계약은 더 작고
명확합니다. **유효한 큰 벡터가 구현상의 중간 overflow 때문에 무효한 벡터로 변하지 않는다.**
