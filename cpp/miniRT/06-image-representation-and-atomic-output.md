# Thread 6. 이미지 표현과 원자적 출력 — 계산, 검증, checksum, 파일 게시

Renderer가 `Image`를 만들었다고 해서 안전하게 파일로 게시할 수 있는 것은 아닙니다. dimensions의
곱은 overflow할 수 있고, public pixel vector는 생성 후 크기가 바뀔 수 있으며, checksum 상수가
틀릴 수 있고, 최종 파일을 직접 열어 쓰면 중간 실패가 기존 결과를 파괴할 수 있습니다.

이 Thread는 네 단계의 불변 조건을 연결합니다.

```text
checked dimensions
  -> exact pixel storage
  -> standard deterministic checksum / serialization
  -> complete temporary output만 final path에 publish
```

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
| 1 | `71096cd311d5` | `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지` | A | `OUTPUT, RISK, EDGE` | positive dimensions와 `width×height×3` checked arithmetic 도입 |
| 2 | `3d2e6a5becb7` | `test(image): 잘못된 차원과 저장 크기 계산 검증` | B | `TEST, OUTPUT` | 정상 exact size와 zero/negative rejection 고정 |
| 3 | `89c3c7269877` | `fix(output): 표준 FNV-1a 기준값 적용` | B | `DEBUG, OUTPUT` | 잘못된 64-bit offset basis를 표준 상수로 교정 |
| 4 | `eac2ecd13c33` | `test(output): PPM과 렌더링 체크섬 기준 고정` | A | `TEST, DETERMINISM, OUTPUT` | 작은 image와 full render의 exact checksum golden 고정 |
| 5 | `4eb50073bc3e` | `fix(output): 불일치한 이미지 저장소 거부` | A | `OUTPUT, RISK, EDGE` | public `Image`의 dimensions와 pixels exact-size를 소비 직전 재검증 |
| 6 | `918dd1efeaf3` | `test(output): 잘못된 이미지 저장소 처리 검증` | B | `TEST, OUTPUT, RISK` | short/oversized storage rejection과 기존 destination 보존 검증 |
| 7 | `053235a7a5e1` | `fix(output): PPM 출력 실패 시 기존 파일 보존` | A | `OUTPUT, RISK, PRACTICAL` | checked stream→same-directory temp→flush/close→replace publication |
| 8 | `c6a6a7562a4d` | `test(output): 출력 실패의 대상 보존과 정리 검증` | A | `TEST, OUTPUT, RISK` | serialization/replacement 실패에서 destination/temp 상태를 직접 검증 |

## 1. 생성 시점의 크기 계산부터 checked operation으로 바꾼다

### `71096cd311d5` — cast가 늦으면 overflow를 막지 못한다

**중요도** `A` · **태그** `OUTPUT, RISK, EDGE`

최초 구현은 다음과 같은 형태였습니다.

```cpp
static_cast<std::size_t>(width * height * 3)
```

`width`, `height`가 `int`이므로 곱셈은 먼저 `int`에서 일어나고, 이미 overflow한 결과를 나중에
`size_t`로 바꿉니다. 이 commit은 dimensions를 먼저 검증하고 multiplication 전에 각 operand를
`size_t`로 변환합니다.

```cpp
std::size_t pixelStorageSize(int width, int height) {
    if (width <= 0 || height <= 0)
        throw std::invalid_argument("image dimensions must be positive");

    const std::size_t w = static_cast<std::size_t>(width);
    const std::size_t h = static_cast<std::size_t>(height);
    const std::size_t channels = 3;

    if (w > std::numeric_limits<std::size_t>::max() / h)
        throw std::overflow_error("image dimensions overflow");
    const std::size_t pixels = w * h;
    if (pixels > std::numeric_limits<std::size_t>::max() / channels)
        throw std::overflow_error("image storage size overflow");
    return pixels * channels;
}
```

PPM traversal의 offset도 `y`, `width`, `x`를 먼저 `size_t`로 올려 계산합니다. 따라서 allocation
size와 pixel access가 같은 arithmetic domain을 사용합니다.

이 commit이 실제 allocation 성공까지 보장하는 것은 아닙니다. 계산 결과가 `size_t`에 들어가더라도
메모리가 부족하면 vector allocation이 실패할 수 있습니다. 보장하는 것은 **signed intermediate
overflow나 wraparound 때문에 실제 필요량보다 작은 buffer를 만드는 일이 없다**는 것입니다.

### `3d2e6a5becb7` — exact storage와 invalid dimensions

**중요도** `B` · **태그** `TEST, OUTPUT`

테스트는 `Image(2,3)`이 정확히 `2×3×3 = 18` bytes를 갖는지 확인하고 width/height가 0 또는
음수인 생성을 거부합니다. 최대값 근처의 모든 overflow 조합을 exhaustive하게 테스트하지는
않지만, public constructor가 positive/exact-size contract를 사용한다는 기본 회귀를 고정합니다.

## 2. checksum을 표준 정의와 golden으로 고정한다

### `89c3c7269877` — offset basis 한 자리 오류

**중요도** `B` · **태그** `DEBUG, OUTPUT`

초기 checksum 구현의 FNV-1a 64-bit offset basis는 다음처럼 한 자리 짧았습니다.

```diff
-std::uint64_t hash = 1469598103934665603ULL;
+std::uint64_t hash = 14695981039346656037ULL;
```

곱셈 prime과 byte 순회가 같아도 initial state가 다르면 모든 checksum이 다른 custom hash가
됩니다. 이 수정은 rendering pixels를 바꾸지 않고 public identifier의 정의를 표준 FNV-1a에
맞춥니다.

“기존 checksum과 호환”보다 “알려진 표준과 일치”를 선택한 breaking correction입니다. 따라서
다음 commit이 exact golden을 새로 고정합니다.

### `eac2ecd13c33` — 형태가 아니라 값을 검증하다

**중요도** `A` · **태그** `TEST, DETERMINISM, OUTPUT`

최초 smoke는 checksum이 16자리 hex이고 반복 실행에서 같다는 것만 봤습니다. 잘못된 offset
basis도 이 조건은 통과합니다. 이 commit은 두 exact value를 고정합니다.

- 작은 synthetic image: `0fde7b4d509f1daf`
- 기본 full render: `456dc8d87ebf194f`

작은 image는 hash algorithm 자체를 좁게 검증하고, full render checksum은 camera·geometry·shading·
quantization까지 포함한 넓은 golden입니다. 둘을 분리해야 rendering 변경 때문에 hash test가
실패한 것인지, hash 구현이 바뀌어 full render golden이 실패한 것인지 해석할 수 있습니다.

golden은 의도적인 rendering contract 변경 시 함께 갱신될 수 있습니다. “어떤 변경도 checksum을
바꾸면 안 된다”는 뜻이 아니라, 변경 이유 없이 bytes가 바뀌는 회귀를 드러냅니다.

## 3. constructor가 맞았어도 public storage는 다시 틀릴 수 있다

### `4eb50073bc3e` — 사용 직전 representation 검증

**중요도** `A` · **태그** `OUTPUT, RISK, EDGE`

`Image`의 `width`, `height`, `pixels`는 public이므로 정상 constructor 뒤 caller가 vector를
resize하거나 dimensions를 바꿀 수 있습니다. 생성 시점 검증만 믿고 writer가 expected index를
읽으면 short storage에서는 out-of-bounds read가, oversized storage에서는 의미가 불분명한
trailing bytes가 생깁니다.

`Image::validate()`는 checked `pixelStorageSize(width,height)`를 다시 계산하고
`pixels.size() == expected`를 요구합니다. PPM writer와 checksum 모두 동작 전에 validate를
호출합니다.

```cpp
void Image::validate() const {
    const std::size_t expected = pixelStorageSize(width, height);
    if (pixels.size() != expected)
        throw std::invalid_argument(
            "image pixel storage does not match dimensions");
}
```

`>= expected`가 아니라 exact equality인 이유는 representation을 하나로 만들기 위해서입니다.
trailing bytes를 checksum에는 포함하지만 PPM에는 무시하는 식의 이중 의미를 허용하지 않습니다.

### `918dd1efeaf3` — malformed object와 destination side effect를 함께 본다

**중요도** `B` · **태그** `TEST, OUTPUT, RISK`

테스트는 정상 Image의 `pixels`를 하나 줄인 short case와 하나 늘린 oversized case를 만듭니다.
둘 다 serialization/checksum에서 거부되어야 합니다. 파일 writer를 호출하기 전에 기존 destination에
`preserve me`를 쓰고, invalid image가 실패한 뒤 내용이 그대로인지 확인합니다.

이 commit 시점의 writer가 완전한 temporary publication을 이미 갖는 것은 아닙니다. 하지만
validation이 file side effect보다 앞에 있어 malformed in-memory object가 기존 결과를 손상하지
않는 순서를 고정합니다.

## 4. final path를 작업 파일로 사용하지 않는다

### `053235a7a5e1` — serialization과 publication을 분리

**중요도** `A` · **태그** `OUTPUT, RISK, PRACTICAL`

기존 방식처럼 final path를 truncate하고 직접 쓰면 write/flush/close 중 실패할 때 이전 파일도
새 파일도 잃습니다. 수정된 경로는 같은 directory의 unique temporary path를 사용합니다.

```text
Image::validate
  -> final path와 같은 directory에 temp path 생성
  -> checked stream으로 P3 전체 serialization
  -> stream state 확인
  -> flush 확인
  -> close 확인
  -> platform replacement
  -> temp owner commit
```

`TemporaryOutput` RAII object는 commit 전 scope가 끝나면 temp file 제거를 시도합니다. path에는
timestamp와 process-local sequence가 들어가 충돌 가능성을 줄입니다.

stream overload는 모든 insertion 뒤 stream state를 확인하고 실패를 exception으로 바꿉니다.
file wrapper는 serialization 성공뿐 아니라 flush/close 성공까지 확인한 뒤에만 replace합니다.

### Replacement 경계

- POSIX: same-directory temporary path를 final path로 `rename`
- Windows: replace-existing와 write-through flag를 사용하는 `MoveFileEx`

같은 directory를 쓰는 이유는 cross-filesystem rename 문제를 피하고, 일반적인 same-filesystem
replacement semantics를 사용하기 위해서입니다.

이 commit은 crash-consistent filesystem transaction이나 directory fsync까지 보장하지 않습니다.
프로세스가 정상적인 API failure를 관찰하는 범위에서 “완성되지 않은 새 파일이 기존 destination을
대체하지 않는다”는 contract입니다.

### `c6a6a7562a4d` — 실패 지점의 filesystem 상태를 직접 검증

**중요도** `A` · **태그** `TEST, OUTPUT, RISK`

두 종류의 failure technique이 추가됩니다.

1. **serialization failure**: custom failing stream buffer가 일정 byte 뒤 write를 거부합니다.
   writer는 성공을 반환하지 않고 stream failure를 전파해야 합니다.
2. **replacement failure**: final destination을 일반 파일이 아닌 교체 불가능한 directory 상태로
   만들어 replace가 실패하게 합니다.

테스트는 단순히 exception이 발생했는지만 보지 않습니다.

- 기존 destination/sentinel이 보존되는가
- 성공 case의 final bytes가 expected PPM과 정확히 같은가
- failure 뒤 temporary artifact가 남지 않는가
- replacement가 성공한 경우에만 old destination이 새 content로 바뀌는가

replacement failure test가 모든 OS/filesystem 조합의 atomicity를 증명하지는 않습니다. repository가
지원하는 implementation에서 cleanup/preservation branch를 직접 재현하는 regression입니다.

## 상태 전이

| 단계 | 유효 상태 | 실패 시 결과 |
| --- | --- | --- |
| dimensions 계산 | positive, `size_t` 안의 exact byte count | Image 생성 실패, storage 없음 |
| Image 소비 전 | `pixels.size() == width×height×3` | serialization/checksum 시작 전 실패 |
| checksum | 표준 FNV-1a initial state와 고정 byte order | exception이면 identifier 없음 |
| temp serialization | complete P3 bytes, stream good | final 유지, temp cleanup 시도 |
| flush/close | kernel/API에 성공 보고 | final 유지, temp cleanup 시도 |
| replace | 완성된 temp만 final name으로 이동 | replace 실패 시 final 유지, temp cleanup 시도 |
| commit | RAII cleanup에서 temp 제외 | 새 final artifact 공개 |

## 최종 보장

- dimension multiplication과 pixel index 계산은 변환 전에 signed overflow하지 않습니다.
- Image의 dimensions와 storage는 생성 시점과 소비 시점 모두 exact하게 일치해야 합니다.
- checksum은 표준 FNV-1a basis와 repository의 byte ordering을 사용합니다.
- malformed image나 serialization failure는 기존 destination을 변경하기 전에 실패합니다.
- final path에는 complete serialization과 flush/close가 끝난 결과만 replace됩니다.
- failure path는 temporary file을 제거하려고 하며, tests는 대표 실패에서 잔여 temp가 없음을 봅니다.

## Thread 경계

pixel을 어떤 값으로 채우는지는 Thread 1·4·5, Image에 쓰는 worker ownership은 Thread 5,
CTest/CI 실행 환경은 Thread 7의 문제입니다. power loss에 대한 directory durability, fsync policy,
network filesystem의 rename semantics, multi-process writer 간 lock은 이 Thread의 보장 범위가
아닙니다.
