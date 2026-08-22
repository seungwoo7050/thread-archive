===== BEGIN FILE: 01-non-overlap-copy-and-overlap-safe-movement.md =====
# Thread: Separating non-overlapping copy from overlap-safe movement

## Thread 목표

**Source significance**

> The sequence makes the precondition boundary explicit instead of hiding overlap handling inside every copy. The stronger operation is built only where needed, and the tests cover both possible overlap directions because the safe traversal direction changes with range placement.

이 문서에서는 source가 확정한 non-overlap precondition과 overlap-safe movement의 분리를 실제 SHA 코드에서 복원합니다.

### 이 Thread에 직접 연결된 source 항상 유지해야 하는 조건

> A length-bounded memory operation accesses only the specified valid byte range; zero-length operations perform no access, and overlapping movement preserves every source byte needed by later copies.

### 이 Thread에 직접 연결된 engineering difficulty

> Distinguishing the weaker non-overlap copy precondition from the direction-sensitive behavior required for overlapping ranges.

## 이 Thread를 이해하기 위한 핵심 질문

- `ft_memcpy`가 책임지지 않는 조건은 어디에 명시되거나 구현으로 드러나는가?
- overlap이 발생했을 때 어떤 배치에서 forward copy가 unread source를 파괴하는가?
- `ft_memmove`는 어떤 경우 직접 역방향 복사를 하고, 어떤 경우 기존 primitive를 재사용하는가?
- zero length와 identical pointer 경로는 실제로 memory access를 피하는가?
- 테스트는 두 overlap 방향과 disjoint case를 어떻게 구분해 검증하는가?

## 완료 기준

- `4873fb11ac60`의 non-overlap contract와 `f2c4c042b339`의 stronger contract 차이를 코드로 설명할 수 있습니다.
- `f2c4c042b339`에서 traversal direction 결정과 unread source 보존 관계를 직접 추적했습니다.
- `69853cd4d3ce`에서 same-position, forward-overlap, backward-overlap, disjoint case가 실제 production path를 어떻게 통과하는지 확인했습니다.
- whole-buffer 비교가 destination 밖 변경까지 잡는지 test code에서 확인했습니다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags | Source role |
| --- | --- | --- | --- | --- | --- |
| 1 | `4873fb11ac60` | `feat(memory): 겹치지 않는 메모리 복사 구현` | B | BYTE_RANGE, CORE | Establishes `ft_memcpy` as the primitive whose caller guarantees non-overlapping ranges. |
| 2 | `f2c4c042b339` | `feat(memory): 겹치는 메모리의 안전한 이동 구현` | A | BYTE_RANGE, CORE, RISK | Adds direction-sensitive movement so destination overlap cannot destroy unread source bytes. |
| 3 | `69853cd4d3ce` | `test(memory): 겹치는 메모리 이동 검증` | B | BYTE_RANGE, TEST | Differentially verifies same-position, forward-overlap, backward-overlap, and disjoint cases. |

## Commit별 학습 기록

### `4873fb11ac60` — `feat(memory): 겹치지 않는 메모리 복사 구현`

**Source 확정 역할:** non-overlapping byte-copy primitive를 도입하며, overlap 처리는 의도적으로 책임 범위 밖에 둡니다.

#### 해당 SHA에서 확인할 코드

- `libft.h`에서 `ft_memcpy` public declaration을 찾습니다.
- `ft_memcpy` 구현 translation unit에서 source와 destination을 byte 단위로 다루는 타입을 확인합니다.
- original destination을 보존해 반환하는 지점을 확인합니다.
- source가 `const` 역할을 유지하는지, 실제 write 대상이 destination에만 한정되는지 확인합니다.
- overlap detection 또는 backward-copy 처리가 없는지 확인하고, 그것이 precondition 경계와 어떻게 연결되는지 기록합니다.
- 이 SHA의 parent와 비교해 공개 API/build에 어떤 구현 단위가 추가되었는지 확인합니다.

#### 학습 기록

- 직전 상태: parent에는 `ft_memset`과 `ft_bzero`가 있었지만 byte range를 다른 위치로 복사하는 public API와 `src/memory/ft_memory_copy.c`는 없었습니다. 이 commit은 Makefile의 `SRC`, `libft.h`, 새 translation unit을 함께 추가했습니다.
- 해결하려던 문제: caller가 길이로 지정한 source bytes를 destination으로 옮기고, libc와 같은 방식으로 원래 destination pointer를 돌려주는 기본 primitive가 필요했습니다.
- 실제 loop와 byte-range 경계: `destination`은 `unsigned char *`, `source`는 `const unsigned char *`로 변환됩니다. `length > 0`인 동안 현재 byte 하나를 읽어 현재 destination에 쓰고 두 pointer를 1씩 전진시킨 뒤 `length`를 1 줄입니다. 따라서 정상 precondition 아래 정확히 `[0, original_length)`만 읽고 씁니다. 길이가 0이면 loop body에 들어가지 않아 pointer를 역참조하지 않습니다.
- caller가 지켜야 하는 precondition: 읽을 source range와 쓸 destination range가 각각 유효하고 서로 겹치지 않아야 합니다. 구현에는 overlap 판정, 임시 buffer, 역방향 traversal이 없으므로 겹치는 range를 넘기는 것은 이 함수가 책임지는 동작이 아닙니다.
- 반환값 계약: local byte pointer를 전진시키지만 마지막에는 변경하지 않은 매개변수 `destination`을 반환합니다. 따라서 반환값은 첫 destination address입니다.
- 이 commit이 보장하는 것: non-overlap precondition 아래 byte representation을 길이만큼 순서대로 복사하고 source를 수정하지 않으며, zero length에서는 memory access 없이 원래 destination을 반환합니다.
- 이 commit이 보장하지 않는 것: overlap에서 원본 byte 보존, invalid pointer의 방어, nonzero length의 `NULL` 처리, 범위 유효성 검사는 보장하지 않습니다.
- 다음 commit에서 강화되어야 하는 이유: destination write가 아직 읽지 않은 source 위치와 겹치면 앞에서부터 복사하는 현재 loop가 source 자체를 먼저 덮어쓸 수 있습니다. overlap을 허용하는 호출에는 별도의 traversal 결정을 가진 stronger operation이 필요합니다.

### `f2c4c042b339` — `feat(memory): 겹치는 메모리의 안전한 이동 구현`

**Source 확정 역할:** overlap 시 unread source byte가 destination write로 파괴되지 않도록 traversal direction을 선택합니다.

**Source 확정 결정:** same-position 또는 zero length는 즉시 반환하고, destination이 source interval 내부에서 시작하는 경우 backward copy를 사용하며, 나머지 경우 forward `ft_memcpy`를 재사용합니다.

#### 변경 전 상태 확인

- 먼저 `4873fb11ac60`에서 `ft_memcpy`가 non-overlap만 전제로 한다는 근거를 다시 확인합니다.
- `f2c4c042b339`의 parent에서 `ft_memmove`가 없던 상태 또는 이 commit이 추가하는 public/API/build 경계를 확인합니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

- `ft_memmove` declaration과 구현 translation unit을 찾습니다.
- identical pointers와 zero length early return이 memory access보다 앞에 있는지 확인합니다.
- overlap 판단에 사용되는 조건을 실제 코드로 기록합니다.
- destination이 source range 내부에서 시작할 때 index 또는 pointer가 어떤 방향으로 이동하는지 추적합니다.
- backward copy에서 **읽기 전에 덮어쓰면 안 되는 source byte**를 작은 예제로 직접 표시합니다.
- forward-safe case가 `ft_memcpy`로 위임되는 caller/callee 연결을 확인합니다.
- source 설명대로 unrelated object에 대한 단순 address ordering 대신 source range 내 위치 판정을 어떻게 구현했는지 확인합니다.

#### 상태 / 항상 유지해야 하는 조건 변화 기록

- 변경 전 precondition: `ft_memcpy` caller가 두 range의 non-overlap을 보장해야 했습니다. 구현은 항상 낮은 offset에서 높은 offset으로 진행했습니다.
- 새로 제공되는 stronger contract: 두 유효 range가 겹쳐도 `ft_memmove`가 아직 읽지 않은 source byte를 보존하는 방향으로 복사합니다. 성공 후 destination의 `length` bytes는 호출 전 source bytes와 같습니다.
- traversal direction 결정 조건: byte pointer를 만든 직후 `destination_byte == source_byte || length == 0`이면 반환합니다. 그 밖에는 `offset`을 1부터 `length - 1`까지 증가시키며 `destination_byte == source_byte + offset`인지 찾습니다. 이 조건이 참이면 destination 시작점이 source interval 내부에 있으므로 backward copy를 수행합니다. 끝까지 일치하지 않으면 `ft_memcpy`를 호출합니다.
- preserved source information: backward branch는 먼저 `length--`한 뒤 `destination_byte[length] = source_byte[length]`를 수행합니다. 가장 높은 offset부터 읽고 쓰므로 낮은 offset의 아직 읽지 않은 source가 높은 destination write로 덮이지 않습니다.
- zero-length no-access 근거: early return이 overlap 탐색과 `ft_memcpy` 호출보다 앞에 있습니다. 따라서 `ft_memmove(NULL, NULL, 0)`도 pointer arithmetic이나 dereference 없이 `NULL`을 반환합니다.

#### failure scenario 직접 복원

- forward copy가 실패하는 overlap 배치: 초기 bytes가 `A B C D E`이고 `source = &buffer[0]`, `destination = &buffer[1]`, `length = 4`인 경우입니다. destination은 `source + 1`이므로 source interval 내부에서 시작합니다.
- 각 iteration에서 손실되는 unread source byte: forward iteration 0이 `buffer[1] = buffer[0]`을 수행하면 아직 iteration 1에서 읽어야 할 원본 `B`가 `A`로 바뀝니다. 다음에는 바뀐 `A`가 다시 `buffer[2]`로 전달되어 결과가 연쇄적으로 오염됩니다.
- backward traversal이 이를 막는 이유: offset 3의 `D`를 destination offset 3으로 먼저 옮기고 2, 1, 0 순서로 진행합니다. 각 write는 이미 읽은 쪽의 높은 source 위치에만 영향을 주므로 다음에 읽을 낮은 source byte가 유지됩니다.
- 반대 overlap 방향에서 forward traversal이 안전한 이유: `destination = &buffer[0]`, `source = &buffer[1]`처럼 destination이 source보다 앞에 있으면 각 write는 다음 iteration에서 읽을 source보다 낮은 주소에 놓입니다. 구현은 이 경우 source interval 내부에서 destination을 찾지 못하고 `ft_memcpy`의 forward loop를 재사용합니다.

#### 보장 범위

- 이 commit이 새로 보장하는 것: same-position과 zero length의 no-access return, destination이 source 내부에서 시작하는 경우의 backward copy, destination-before-source overlap과 disjoint range의 forward copy를 하나의 API에서 제공합니다. 단순 relational pointer 비교 대신 `source + offset`과의 equality만 사용합니다.
- 아직 test commit 없이는 코드 inspection에 의존하는 부분: 두 overlap 방향에서 실제 bytes가 system `memmove`와 같은지, destination 밖 byte가 변하지 않는지, 여러 길이와 offset에서 반환 pointer가 맞는지는 이 commit 자체에 새 테스트가 없어 코드 검사에 의존합니다.
- 다음 `69853cd4d3ce`가 확인해야 할 위험: early return, forward-safe overlap, backward-required overlap, disjoint range, 반환값, destination 바깥 write를 서로 다른 배치와 길이에서 검증해야 합니다.

### `69853cd4d3ce` — `test(memory): 겹치는 메모리 이동 검증`

**Source 확정 역할:** system `memmove`를 oracle로 사용해 same-position, forward-overlap, backward-overlap, disjoint move를 비교합니다.

#### Test commit 학습

- production 항상 유지해야 하는 조건 대상:
  - 기록: 호출 전 source bytes가 overlap 방향과 무관하게 destination에 보존되고, 함수가 원래 destination을 반환하며, 지정 range 밖 buffer는 변하지 않아야 합니다. zero length는 `NULL` pointer에도 접근하지 않아야 합니다.
- 재현하는 boundary/failure:
  - same-position: `check_move(0, 0, length)`가 모든 길이에서 identical-pointer early return을 통과합니다.
  - forward-overlap: `check_move(0, 1, length)`는 길이 2 이상에서 destination-before-source overlap을 만들고 forward reuse path를 통과합니다. `(7, 19)`도 길이 15 이상에서는 같은 방향의 overlap입니다.
  - backward-overlap: `check_move(1, 0, length)`는 길이 2 이상에서 destination이 source interval 안에 놓여 backward branch를 통과합니다. `(23, 5)`는 길이 31과 63에서 같은 branch를 통과합니다.
  - disjoint: `(7, 19)`는 길이 12 이하, 실제 목록에서는 0·1·2·3·7·8에서 disjoint입니다. `(23, 5)`는 길이 18 이하, 실제 목록에서는 0·1·2·3·7·8·15·16에서 disjoint입니다. `(0, 1)`과 `(1, 0)`도 길이 1에서는 서로 다른 한 byte range입니다.
  - zero-length null-pointer: `CHECK(ft_memmove(NULL, NULL, 0) == NULL)`가 early return의 no-access 조건과 반환값을 직접 확인합니다.
- test technique:
  - source에 확정된 differential oracle 사용 위치를 실제 test code에서 찾습니다. `check_move`는 동일한 pattern으로 채운 `actual`과 `expected`를 준비하고, `actual`에는 `ft_memmove`, `expected`에는 system `memmove`를 적용합니다.
  - patterned buffer 초기화와 whole-buffer comparison 위치를 찾습니다. 128 bytes를 `(index * 29U + 17U)`로 채운 뒤 `memcmp(actual, expected, sizeof(actual))`를 수행하므로 destination 밖의 예상하지 않은 변경도 검출합니다.
- 실제 production path:
  - 각 case가 `ft_memmove`의 어느 분기로 들어가는지 기록합니다. `(0,0)` 또는 모든 zero length는 early return입니다. destination offset이 source offset보다 크고 그 차이가 length보다 작으면 backward branch이며, 그 밖에는 `ft_memcpy` branch입니다.
  - `ft_memcpy` reuse path를 통과하는 case를 확인합니다. destination-before-source overlap, disjoint range, destination이 source 끝 이후인 range가 이 path를 사용합니다.
- 테스트가 증명하는 것: 열 개의 길이와 다섯 offset 조합에서 반환값과 128-byte 전체 결과가 host `memmove`와 동일함을 결정적으로 확인합니다. 양쪽 overlap 방향, same-position, 여러 disjoint 구간, zero-length `NULL`을 포함합니다.
- 테스트가 증명하지 않는 것: 가능한 모든 길이와 offset, 유효하지 않은 nonzero pointer, 실제 object 범위를 벗어난 입력, 매우 큰 길이, 성능 특성은 증명하지 않습니다. oracle 자체와 같은 환경의 libc 동작을 기준으로 한 비교입니다.
- 테스트 성격:
  - [ ] broad integration
  - [x] deterministic regression
  - [x] differential boundary test
  - 선택 근거: 고정 pattern·길이·offset으로 항상 같은 production branches를 통과하고 system `memmove` 결과와 전체 buffer를 차등 비교하므로 deterministic differential boundary regression입니다. 여러 subsystem을 묶는 broad 통합 테스트는 아닙니다.
- 후속 변경에서 막아야 할 회귀: overlap 판정 조건 반전, backward index의 off-by-one, zero-length에서 pointer access, forward branch에서 잘못된 길이 전달, 원래 destination이 아닌 전진한 pointer 반환, destination 밖 write를 막습니다.
- 실행 근거: 저장소 checkout을 만들 수 없는 현재 환경에서는 test binary를 실행하지 않았습니다. 위 결과는 `69853cd4d3ce`의 test code와 해당 production SHA를 직접 검사한 내용이며 실행 성공을 주장하지 않습니다.

## 항상 유지해야 하는 조건 ledger

| 시점 | Commit | Source에 연결된 항상 유지해야 하는 조건 | 실제 코드에서 확인한 근거 |
| --- | --- | --- | --- |
| weaker primitive | `4873fb11ac60` | non-overlap precondition 아래 지정 byte range만 copy | `src/memory/ft_memory_copy.c`의 `length > 0` loop가 byte 하나씩 읽고 쓰며 overlap branch 없이 원래 `destination`을 반환합니다. |
| stronger operation | `f2c4c042b339` | overlap 시 later copy에 필요한 source byte 보존 | `destination == source + offset`을 source interval 안에서 찾으면 `length - 1`부터 0까지 복사하고, 나머지는 `ft_memcpy`로 위임합니다. |
| verification | `69853cd4d3ce` | 두 overlap 방향, same-position, disjoint, zero-length 경계 검증 | `tests/test_memory_move.c`가 고정 pattern과 여러 길이/offset을 system `memmove`와 whole-buffer 차등 비교하고 `NULL, NULL, 0`을 별도로 검사합니다. |

## Failure → Fix → Test 연결

이 thread는 source상 별도 `fix(...)` commit이 아니라 stronger feature 도입으로 위험을 해결합니다.

- 기존 가정: `ft_memcpy` caller가 non-overlap을 보장
- 위험: overlap에서 forward copy가 unread source를 덮어쓸 수 있음
- stronger decision: `f2c4c042b339`
- 실제 수정/추가 코드 근거: 새 `src/memory/ft_memory_move.c`가 source interval 안에서 destination 시작점을 탐색하고, 해당하면 backward loop를 수행하며, forward-safe case만 `ft_memcpy`에 넘깁니다.
- 검증: `69853cd4d3ce`
- regression으로 고정된 동작: same-position/zero-length no access, 두 방향 overlap의 byte 보존, disjoint copy, original destination return, destination 밖 byte 불변이 고정됩니다.

## Byte-range / responsibility 변화

- `ft_memcpy` caller responsibility: 두 유효 byte range가 겹치지 않는다는 사실과 각 range가 `length` bytes를 수용한다는 사실을 보장합니다.
- `ft_memmove` implementation responsibility: 유효 range가 겹치는지 방향상 필요한 만큼 판정하고 unread source를 보존하는 traversal을 선택합니다.
- 공통 byte-range 항상 유지해야 하는 조건: 정상 입력에서는 지정된 `length` bytes만 읽고 destination의 같은 길이만 쓰며, 길이 0이면 접근하지 않고 원래 destination을 반환합니다.
- 두 API를 분리했을 때 얻는 계약상의 차이: 일반 복사는 단순한 forward primitive와 명시적 non-overlap precondition을 유지하고, overlap 비용과 방향 결정은 이를 요구하는 `ft_memmove` 호출에만 부과됩니다.

## Thread 최종 상태

- 마지막 commit 시점에 이 thread가 보장하는 것:
  - 기록: 코드상 `ft_memcpy`와 `ft_memmove`의 책임이 분리되고, `ft_memmove`는 destination-inside-source에만 backward traversal을 적용합니다. 고정된 차등 테스트는 선택한 경계 조합에서 반환값과 전체 buffer가 libc oracle과 같음을 검사합니다.
- 이 thread만으로는 보장하지 않는 것:
  - 기록: invalid nonzero pointers, object bounds 밖 length, 모든 가능한 offset/length, concurrency, 성능을 보장하거나 검증하지 않습니다. 실행 결과도 현재 환경에서는 생산하지 않았습니다.
- source의 significance와 실제 코드 확인 결과가 연결되는 지점:
  - 기록: overlap 처리를 `ft_memcpy`에 숨기지 않고 stronger `ft_memmove`에만 두었고, destination 배치에 따라 traversal을 바꾸며, 테스트가 양쪽 방향을 구분해 oracle과 비교한다는 점이 source significance와 일치합니다.

## 최종 architecture 또는 실행 순서 정리

해당 thread의 commit history를 근거로 최종 흐름을 직접 작성합니다.

- 시작 조건 / 입력: caller가 유효한 destination, source, length를 전달합니다. `ft_memcpy`는 non-overlap을 추가 precondition으로 요구하고 `ft_memmove`는 overlap을 허용합니다.
- 핵심 분기 또는 각 부분이 맡는 일의 구분: `ft_memmove`는 same-position/zero length를 먼저 종료하고, destination이 `source + 1`부터 `source + length - 1` 중 하나인지 확인합니다. 해당하면 backward, 아니면 `ft_memcpy` forward path입니다.
- 상태 또는 소유권 변화: 별도 allocation이나 소유권 transfer는 없습니다. destination bytes만 변경되고 source는 read-only view로 취급됩니다.
- failure 처리: API에는 status가 없고 invalid range를 검증하지 않습니다. 안전성은 유효 범위 precondition과 overlap 방향 선택으로 확보합니다.
- verification 경로: `tests/test_memory_move.c`가 동일 초기 buffer에 project/system 구현을 적용한 뒤 전체 128 bytes와 반환 pointer를 비교합니다.
- 최종 설명: 기본 복사는 단순한 non-overlap primitive로 유지됩니다. overlap-safe API는 destination이 아직 읽지 않은 source의 뒤쪽을 덮을 때만 뒤에서 앞으로 복사하고, 그 외에는 기존 forward primitive를 재사용합니다. 이 분리는 caller precondition과 구현 책임을 명확히 하며, 고정된 differential test가 대표적인 양방향 overlap과 경계를 회귀로 묶습니다.

## 학습 완료 자가 점검

- [x] 모든 commit을 문서 순서대로 해당 SHA에서 확인했습니다.
- [x] 중요도와 tags를 source 그대로 유지했습니다.
- [x] 실제 코드 근거와 source 확정 설명을 구분했습니다.
- [x] 변경 전/후 비교가 필요한 commit은 이전 관련 SHA와 비교했습니다.
- [x] failure → fix → test 연결을 실제 코드와 test code로 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급하지 않았습니다.
- [x] 이 thread의 최종 항상 유지해야 하는 조건과 execution flow를 코드 근거로 설명할 수 있습니다.
===== END FILE: 01-non-overlap-copy-and-overlap-safe-movement.md =====

===== BEGIN FILE: 02-single-allocation-to-rollback-safe-소유권.md =====
# Thread: From single allocations to rollback-safe 소유권

## Thread 목표

**Source significance**

> The 소유권 model grows from one returned allocation to nested object graphs. `ft_split` establishes the decisive complete-or-rollback rule; list lifecycle callbacks generalize 소유권 to opaque caller data; `ft_lstmap` combines both concerns. The final failure harness demonstrates that cleanup holds at every intermediate acquisition point rather than only on successful examples.

### 이 Thread에 직접 연결된 source 항상 유지해야 하는 조건

> Allocation size arithmetic must be validated before multiplication or addition so wrapped sizes are never passed to `malloc`.

> Multi-allocation constructors either return a complete result or release every resource acquired for the partial result; no partially owned root escapes on failure.

> List nodes and list content have separate 수명. A content destructor is invoked exactly according to the caller-supplied policy, and completed cleanup leaves the caller's head `NULL`.

### 이 Thread에 직접 연결된 engineering difficulties

> Rolling back nested allocations in `ft_split` and callback-produced node/content pairs in `ft_lstmap`.

> Reproducing allocation and write failures deterministically without changing the production API.

## 이 Thread를 이해하기 위한 핵심 질문

- single allocation의 size arithmetic 검증이 이후 multi-allocation builder의 전제가 되는가?
- 새 allocation의 소유권은 어느 시점에 caller 또는 container로 이전됩니까?
- `ft_split`에서 root와 child field가 부분 생성된 상태는 어떻게 rollback됩니까?
- list node와 opaque content의 수명 책임은 어떻게 분리됩니까?
- `ft_lstmap`에서 callback 결과와 node allocation 사이 failure가 발생하면 누가 무엇을 해제하는가?
- deterministic allocator failure harness는 각 acquisition position을 어떻게 강제로 실패시키고 무엇을 측정하는가?

## 완료 기준

- multiplication/addition을 `malloc` 전에 검증하는 실제 코드를 찾고 이유를 설명할 수 있습니다.
- `ft_split`의 complete-or-rollback 경로를 allocation 순서대로 추적했습니다.
- `ft_lstclear`의 node/content 책임 분리와 `ft_lstmap` failure cleanup이 연결되는 지점을 확인했습니다.
- `fd3ae063139d`가 `ft_split`과 `ft_lstmap`의 각 allocation failure 위치를 실제로 강제하는 방법을 test code에서 확인했습니다.
- success 소유권 graph와 failure cleanup graph를 각각 그릴 수 있습니다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags | Source role |
| --- | --- | --- | --- | --- | --- |
| 1 | `3b1b30983876` | `feat(alloc): 0 초기화 메모리와 문자열 복제 추가` | A | SIZE_ARITH, OWNERSHIP, RISK | Establishes overflow-checked allocation sizes and the basic caller-owned allocation contract. |
| 2 | `6d076de7185e` | `feat(string): 부분 문자열 생성을 구현` | B | OWNERSHIP, SIZE_ARITH | Applies that contract to an exactly sized substring result. |
| 3 | `644b1c65444c` | `feat(string): 문자열 결합을 구현` | B | OWNERSHIP, SIZE_ARITH | Extends checked addition to joined strings. |
| 4 | `8c0a35a50878` | `feat(string): 실패 시 정리되는 문자열 분리 구현` | S | OWNERSHIP, CORE, RISK | Introduces a multi-allocation root whose partially built children are all released on failure. |
| 5 | `7a016ad8fd21` | `feat(list): 연결 리스트 순회와 삭제 구현` | A | LIST_LIFECYCLE, OWNERSHIP, RISK | Separates list-node lifetime from callback-defined content lifetime and provides complete clearing. |
| 6 | `6672ea67fae4` | `feat(list): 실패 시 정리되는 리스트 변환 구현` | A | LIST_LIFECYCLE, OWNERSHIP, RISK | Applies all-or-nothing ownership to callback-produced content and newly allocated list nodes. |
| 7 | `fd3ae063139d` | `test(alloc): 할당 실패와 rollback을 검증` | A | OWNERSHIP, TEST, RISK | Injects failure at each allocation position and measures leaks and invalid frees. |

## Commit별 학습 기록

### `3b1b30983876` — `feat(alloc): 0 초기화 메모리와 문자열 복제 추가`

**Source 확정 역할:** overflow-checked allocation size와 caller-owned single allocation contract의 기반을 만듭니다.

**Source 확정 위험:** `count * size` wrap은 논리 요청보다 작은 allocation을 만들 수 있으므로 multiplication 전에 검증해야 합니다.

#### 해당 SHA에서 확인할 코드

- `ft_calloc`에서 multiplication 가능 여부를 판단하는 조건과 그 조건이 `malloc`보다 먼저 실행되는지 확인합니다.
- zero product를 source 설명대로 one-byte allocation으로 처리하는 실제 branch를 찾습니다.
- allocation 성공 후 zeroing이 정확히 어떤 extent에 적용되는지 caller/callee를 추적합니다.
- `ft_strdup`에서 source length, terminator 포함 allocation size, copy, `NULL` failure return 순서를 확인합니다.
- 성공 시 반환 pointer의 소유권이 caller로 넘어가는 지점을 기록합니다.
- 이 commit의 parent와 비교해 public allocation API와 implementation module이 어떻게 추가되었는지 확인합니다.

#### 학습 기록

- 직전 상태: parent에는 character, memory, string search/bounds, `ft_atoi` 구현이 있었지만 allocation API와 `src/alloc/ft_allocate.c`는 없었습니다. 이 commit이 Makefile, `libft.h`, 새 module을 함께 추가했습니다.
- size arithmetic 위험: `count * size`가 `size_t` 최대값을 넘어 wrap되면 caller가 요구한 논리 크기보다 작은 block을 할당한 뒤 더 큰 범위로 사용할 위험이 있습니다. `ft_strdup`의 `length + 1`도 terminator 공간을 포함하므로 addition overflow를 피해야 합니다.
- pre-allocation 검증 코드: `ft_calloc`은 `count != 0 && size > (size_t)-1 / count`를 `count * size`보다 먼저 평가해 multiplication 가능 여부를 나눗셈으로 검사합니다. `ft_strdup`은 `ft_strlen` 결과가 `(size_t)-1`이면 `length + 1`을 계산하기 전에 `NULL`을 반환합니다.
- zero-size policy: product가 0이면 `allocation_size`를 1로 바꾸고 한 byte를 실제로 할당합니다. 이후 `ft_bzero(allocation, allocation_size)`가 그 한 byte를 0으로 만듭니다. 성공한 zero-size 호출도 caller가 `free`할 수 있는 non-`NULL` allocation을 받을 수 있습니다.
- `ft_calloc` 성공 소유권: `malloc`과 zeroing이 끝난 pointer를 반환하는 순간 해당 block의 해제 책임이 caller로 이전됩니다. 함수 내부에는 성공 후 retained reference가 없습니다.
- `ft_strdup` 성공 소유권: source는 `const`로 읽기만 하고 `length + 1` bytes의 새 block에 terminator까지 복사합니다. 반환된 duplicate는 source와 독립된 caller-owned allocation입니다.
- allocation 실패 처리: 두 함수 모두 `malloc`이 `NULL`이면 추가 write나 cleanup 없이 `NULL`을 반환합니다. 이 시점에는 함수가 이전에 획득한 다른 resource가 없으므로 rollback 대상도 없습니다.
- 이 commit이 이후 builder에 제공하는 전제: allocation 전에 크기를 확정하고, 성공 전에는 소유권을 publish하지 않으며, 실패 시 live allocation을 남기지 않는 single-resource pattern을 제공합니다. 이후 builder는 이 규칙을 root와 여러 child에 확장해야 합니다.

### `6d076de7185e` — `feat(string): 부분 문자열 생성을 구현`

**Source 확정 역할:** single allocation 소유권 contract를 exactly sized substring에 적용합니다.

#### 해당 SHA에서 확인할 코드

- null source failure branch를 확인합니다.
- `start >= source length`에서 독립적으로 free 가능한 empty string을 만드는 경로를 확인합니다.
- available suffix를 `text_length - start` 형태로 계산하는 지점을 찾습니다.
- requested length를 available span으로 clamp하는 순서를 확인합니다.
- allocation size와 explicit NUL termination을 확인합니다.
- 반환값이 source 내부 borrowed pointer가 아니라 새 allocation임을 코드에서 확인합니다.

#### 학습 기록

- 입력 범위 계산: source가 `NULL`이면 즉시 실패합니다. 그 밖에는 `text_length = ft_strlen(text)`를 구하고, `(size_t)start >= text_length`이면 empty result 경로로 이동합니다. 유효한 start에서는 available suffix를 `text_length - (size_t)start`로 계산합니다.
- overflow를 피하는 계산 형태: requested `length`가 available suffix보다 크면 먼저 suffix 길이로 clamp합니다. start를 더해 end index를 만드는 대신 전체 길이에서 start를 빼므로 end addition wrap을 피합니다. 이 SHA에는 `length + 1`의 별도 명시적 guard는 없지만, clamp된 length는 실제 유효 C string suffix보다 크지 않습니다.
- allocation 크기: 유효 범위는 `length + 1` bytes를 할당하고 payload `length` bytes를 복사한 뒤 `substring[length] = '\0'`로 종료합니다.
- 소유권 transfer: allocation과 terminator 작성이 완료된 pointer를 반환할 때 caller가 단독 소유합니다. source 내부 pointer를 반환하거나 source 소유권을 변경하지 않습니다.
- empty result 표현: start가 문자열 끝 이상이면 `ft_strdup("")`를 호출합니다. 따라서 empty string literal을 빌려주는 것이 아니라 caller가 자유롭게 `free`할 수 있는 one-byte NUL allocation을 반환합니다. 그 allocation이 실패하면 `NULL`입니다.
- 이 commit이 보장하는 것 / 보장하지 않는 것: valid source에서 start와 requested length를 suffix 안으로 제한한 독립 NUL-terminated 결과 또는 `NULL`을 보장합니다. invalid source pointer, 실제 object bounds를 벗어난 비정상 C string, allocation 성공을 보장하지 않습니다.

### `644b1c65444c` — `feat(string): 문자열 결합을 구현`

**Source 확정 역할:** joined string construction에 checked addition을 확장합니다.

#### 해당 SHA에서 확인할 코드

- null operand가 allocation 전에 거부되는지 확인합니다.
- left length + right length + terminator 계산이 wrap되지 않도록 검사하는 실제 조건을 기록합니다.
- left payload와 right payload/terminator가 어떤 순서로 copy되는지 확인합니다.
- 입력 두 문자열이 변경되지 않는지 write target을 기준으로 확인합니다.
- 성공 result의 단일 소유권과 failure 시 live allocation 유무를 기록합니다.

#### 학습 기록

- size addition 검증: 두 operand 중 하나가 `NULL`이면 길이를 읽기 전에 실패합니다. 길이를 구한 뒤 `right_length == (size_t)-1 || left_length > (size_t)-2 - right_length`이면 반환합니다. `(size_t)-2`는 `SIZE_MAX - 1`이므로 이 조건은 `left_length + right_length + 1`이 `SIZE_MAX`를 넘지 않는지 addition 전에 확인합니다.
- allocation 이후 copy 순서: 정확히 `left_length + right_length + 1` bytes를 할당한 뒤 left payload만 `left_length`만큼 복사하고, `joined + left_length`에 right의 `right_length + 1` bytes를 복사합니다.
- authoritative terminator 위치: 두 번째 copy가 right의 기존 NUL까지 포함하므로 최종 terminator는 `joined[left_length + right_length]`에 놓입니다. 별도 terminator write는 없습니다.
- input 소유권 / result 소유권: `left`와 `right`는 `const` input이며 write target은 새 `joined` block뿐입니다. 성공 시 caller가 하나의 result allocation을 소유하고, `malloc` 실패 전후에 함수가 보유한 다른 resource는 없습니다.
- 다음 multi-allocation commit과 다른 점: join은 하나의 size 검증과 하나의 allocation만 수행하므로 실패 시 partial child graph가 없습니다. `ft_split`은 root가 성공한 뒤 여러 field allocation이 이어져 중간 rollback이 필요합니다.

### `8c0a35a50878` — `feat(string): 실패 시 정리되는 문자열 분리 구현`

**Source 확정 역할:** 첫 project-defining multi-allocation transaction을 도입합니다. 성공 시 complete root를 publish하고, 실패 시 이미 획득한 모든 child와 root를 해제합니다.

#### 변경 전 상태 확인

- `3b1b30983876`, `6d076de7185e`, `644b1c65444c`에서 single allocation의 성공/실패 ownership이 어떻게 단순했는지 비교합니다.
- 이 SHA의 parent에서 multi-allocation root를 반환하는 기존 pattern이 있는지 직접 확인하되, source에 없는 결론은 추정하지 않습니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

- field count pass와 pointer-array allocation 순서를 확인합니다.
- root가 null-terminated representation이 되도록 초기화 또는 sentinel이 보장되는 지점을 확인합니다.
- consecutive delimiter를 skip하는 parsing 경로와 field span 계산을 찾습니다.
- 각 field가 언제 allocation되고 root의 어느 slot에 소유권이 연결되는지 추적합니다.
- field allocation 실패 branch에서 이미 생성된 field를 어떤 순서로 free하고 root를 언제 free하는지 확인합니다.
- cleanup helper가 partial result의 어느 상태를 입력으로 받는지 기록합니다.
- 실패 후 caller에게 partial root가 절대 반환되지 않는 return path를 확인합니다.
- 성공 시 caller가 sentinel을 기준으로 전체 graph를 해제할 수 있는 representation을 확인합니다.

#### 소유권 transaction 기록

- acquisition 1: root: `count_fields(text, delimiter) + 1`개의 pointer slot을 `ft_calloc`으로 확보합니다. 모든 slot이 0으로 초기화되므로 마지막 미사용 slot이 sentinel `NULL`입니다.
- acquisition 2..N: fields: parsing loop가 delimiter를 건너뛴 뒤 `[start, text_index)` span을 만들고, nonempty span마다 `copy_field`가 `length + 1` bytes를 할당해 복사·종료합니다.
- 각 child 소유권 transfer 시점: `copy_field`가 성공한 pointer를 `fields[field_index]`에 대입하는 순간 partial root가 그 field를 보유합니다. 그 뒤 `field_index`가 증가합니다.
- 성공 publish 조건: 입력 끝까지 처리해 모든 nonempty field가 연결되고, `ft_calloc`이 남겨 둔 다음 slot의 `NULL` sentinel이 유지된 상태에서만 `fields`를 반환합니다.
- failure detection 지점: root `ft_calloc`이 `NULL`인 경우와 각 `copy_field`가 `NULL`인 경우입니다. child failure는 slot 대입 직후 검사하지만 `field_index` 증가 전이므로 실패 slot은 cleanup count에 포함되지 않습니다.
- rollback 순서: `free_fields(fields, field_index)`가 성공해 연결된 child 수를 받아 높은 index부터 0까지 역순으로 free한 뒤 root pointer array를 free합니다. 이후 `ft_split`은 comma expression으로 `NULL`을 반환합니다.
- rollback 종료 후 live allocation 수: 해당 호출이 획득한 root와 모든 성공 child가 해제되므로 0입니다. caller에게 partial root가 전달되지 않습니다.

#### failure scenario 직접 복원

- 첫 field allocation 실패: root만 살아 있고 `field_index == 0`입니다. `free_fields`의 child loop는 실행되지 않고 root만 free한 뒤 `NULL`을 반환합니다.
- 중간 field allocation 실패: 앞선 slots에는 완성된 field들이 연결돼 있고 실패 slot에는 `NULL`이 대입됩니다. helper는 `field_index`개의 이전 child를 역순으로 해제하고 마지막에 root를 해제합니다.
- 마지막 field 근처 실패: 이전의 거의 모든 child가 root에 연결돼 있어도 동일한 count 기반 cleanup이 전부 회수합니다. sentinel 영역과 실패 slot을 free 대상으로 잘못 취급하지 않습니다.
- 각 경우 root/children의 상태: rollback 중에만 local `fields`가 partial graph를 가리킵니다. 함수 밖으로 나갈 때는 그 graph가 모두 해제되고 반환값은 `NULL`입니다.
- 잘못된 partial publish가 발생할 경우 caller가 겪는 문제: caller가 어느 slot까지 유효한지, 실패 slot 뒤 sentinel이 신뢰 가능한지 알 수 없고, 성공으로 오인해 불완전 데이터를 사용하거나 child/root를 누락·중복 해제할 수 있습니다. 현재 구현은 publish 자체를 금지합니다.

#### 보장 범위

- 이 commit이 보장하는 complete-or-rollback: valid input에서 모든 field와 `NULL` sentinel을 가진 complete array를 반환하거나, root/child acquisition 중 하나라도 실패하면 그 호출이 확보한 모든 allocation을 해제하고 `NULL`만 반환합니다. consecutive delimiter는 empty field를 만들지 않습니다.
- 성공-path test만으로는 증명되지 않는 부분: 두 번째 이후 field allocation 실패 시 이전 child가 실제로 모두 free되는지, root가 남지 않는지, untracked pointer를 free하지 않는지는 정상 실행만으로 확인되지 않습니다.
- 후속 `fd3ae063139d`에서 강제로 확인해야 하는 allocation 위치: 네 field 입력 기준 root 1회와 field 4회의 총 5개 위치 각각을 실패시켜야 합니다. 각 경우 반환 `NULL`, live allocation 0, invalid free 0을 확인해야 합니다.

### `7a016ad8fd21` — `feat(list): 연결 리스트 순회와 삭제 구현`

**Source 확정 역할:** list node 수명과 caller-defined content 수명을 분리하고 complete clearing을 제공합니다.

#### 해당 SHA에서 확인할 코드

- `ft_lstdelone`이 caller destructor와 node free를 어떤 순서로 수행하는지 확인합니다.
- destructor가 null일 때 partial destruction을 피하도록 어떤 guard가 있는지 확인합니다.
- `ft_lstclear`가 current node를 해제하기 전에 successor를 저장하는 지점을 찾습니다.
- caller head가 cleanup 완료 후 `NULL`이 되는 상태 전이를 추적합니다.
- iteration 함수가 content callback만 수행하고 link/free를 변경하지 않는지 확인합니다.
- node allocation 책임과 opaque content 수명 정책이 함수별로 어떻게 분리되는지 표로 정리합니다.

#### 학습 기록

- node owner: list를 만든 caller 또는 해당 list를 구축 중인 함수가 node allocation을 소유합니다. `ft_lstdelone`과 `ft_lstclear`는 전달받은 node를 `free`하는 lifecycle 연산입니다.
- content owner / destructor policy: library는 `void *content`의 실제 타입과 해제 방식을 알지 못합니다. caller가 전달한 `del` callback이 content 수명 policy를 정의하며, library는 이를 호출한 뒤 node를 해제합니다.
- `ft_lstdelone` transition: `node == NULL || del == NULL`이면 아무것도 하지 않습니다. 정상 경로에서는 먼저 `del(node->content)`로 opaque content를 정리하고 그 다음 `free(node)`로 node storage를 정리합니다.
- `ft_lstclear` loop state: `list == NULL || del == NULL`이면 반환합니다. 각 iteration에서 `next = (*list)->next`를 node free 전에 저장하고 `ft_lstdelone(*list, del)`을 호출한 뒤 `*list = next`로 caller head를 전진시킵니다.
- cleanup 완료 항상 유지해야 하는 조건: loop가 종료되면 `*list == NULL`입니다. successor를 먼저 보존하므로 freed node의 `next`를 읽지 않습니다.
- missing destructor에서 보장하는 것: content 정리 없이 node만 free하는 partial destruction을 수행하지 않고 list/head를 그대로 둡니다. 대신 cleanup을 대신해 주지 않으므로 소유권은 caller에게 남습니다.
- `ft_lstmap`이 이 lifecycle API에 의존할 이유: mapping 중 여러 node가 이미 연결된 뒤 새 node allocation이 실패할 수 있습니다. established `ft_lstclear`를 사용하면 각 mapped content에 같은 destructor policy를 적용하면서 partial node chain과 head를 한 번에 정리할 수 있습니다.

### `6672ea67fae4` — `feat(list): 실패 시 정리되는 리스트 변환 구현`

**Source 확정 역할:** callback-produced content와 새 list node에 all-or-nothing 소유권을 적용합니다.

#### 변경 전 상태 확인

- `7a016ad8fd21`에서 list clear가 node/content를 어떤 계약으로 정리하는지 다시 확인합니다.
- `8c0a35a50878`의 partial ownership rollback과 비교하되, array field와 opaque callback content의 차이를 직접 기록합니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

- source list를 순회하면서 callback이 mapped content를 생성하는 지점을 확인합니다.
- callback 결과가 생성된 뒤 node allocation이 발생하는 순서를 확인합니다.
- node allocation 실패 시 **아직 어느 node에도 연결되지 않은 최신 mapped content**를 즉시 destructor로 정리하는 branch를 찾습니다.
- 이미 만든 mapped list를 `ft_lstclear` 또는 동등한 established lifecycle 경로로 정리하는 호출을 확인합니다.
- maintained tail이 append마다 어떻게 갱신되는지 확인합니다.
- source list의 link/content를 변경하지 않는지 write target 기준으로 확인합니다.
- callback이 `NULL` content를 반환하는 경우와 node allocation failure가 코드에서 어떻게 구분되는지 확인합니다.

#### 소유권 / lifecycle 기록

- callback 직후 content owner: `mapped_content = function(list->content)`가 반환된 직후에는 아직 node에 연결되지 않았으므로 `ft_lstmap` local transaction이 그 content의 cleanup 책임을 가집니다.
- node 생성 성공 후 content owner: `ft_lstnew(mapped_content)`가 성공하면 새 node가 pointer를 보유합니다. 이후 cleanup은 node chain에 적용되는 `del` callback을 통해 수행됩니다.
- new node가 partial output에 연결되는 시점: 첫 node는 `mapped = node`, 이후 node는 `tail->next = node`로 연결합니다. 그 뒤 `tail = node`로 append cursor를 갱신합니다.
- node allocation failure 직후 cleanup 대상: node에 연결되지 않은 최신 `mapped_content`를 `del(mapped_content)`로 즉시 정리해야 합니다. 이어서 이전 iteration에서 연결된 `mapped` chain 전체를 `ft_lstclear(&mapped, del)`로 정리합니다.
- partial mapped list rollback: `ft_lstclear`가 각 기존 mapped content와 node를 정리하고 local head를 `NULL`로 만든 뒤 함수가 `NULL`을 반환합니다.
- source list 수명: loop는 local `list = list->next`만 수행합니다. source node의 content나 `next`에 write하지 않고 source allocation을 해제하지 않습니다.

#### failure scenario

- 첫 node allocation 실패: callback이 만든 첫 content는 아직 연결되지 않았으므로 `del(mapped_content)` 한 번만 호출됩니다. `mapped == NULL`이므로 `ft_lstclear`는 node를 해제하지 않고 결과는 `NULL`입니다.
- 중간 node allocation 실패: 최신 unlinked content를 먼저 `del`하고, 이전 성공 iteration의 mapped contents와 nodes를 `ft_lstclear`로 전부 정리합니다. source list는 그대로 남습니다.
- callback result가 `NULL`인 정상 case와 failure의 구분: 구현은 callback의 `NULL`을 별도 실패로 해석하지 않습니다. `ft_lstnew(NULL)`이 성공하면 `NULL` content를 가진 node가 정상 결과에 포함됩니다. failure 판단은 오직 `node == NULL`입니다. 따라서 callback 자체의 실패를 `NULL`로 전달하고 싶다면 이 API 구현만으로는 구분할 수 없습니다.
- cleanup 이후 남아야 하는 object: source list와 source contents만 남고, 해당 호출이 만든 mapped content 및 mapped nodes는 남지 않아야 합니다.

#### 다음 검증과 연결

- `fd3ae063139d`에서 주입해야 하는 실패 위치: 세 source node 각각에 대응하는 `ft_lstnew` production allocation 1·2·3번째를 실패시켜야 합니다. callback content 생성 직후 node failure path가 매 iteration에서 실행됩니다.
- leak / invalid free / source mutation 중 각각 확인할 항목: tracked node live count가 0인지, tracked node에 invalid free가 없는지, destructor 호출 수가 실패 index와 같은지, source content 값이 유지되는지 확인해야 합니다. source `next` pointer 자체는 해당 harness에서 명시적으로 비교하지 않는 한 코드 inspection 근거와 구분해야 합니다.

### `fd3ae063139d` — `test(alloc): 할당 실패와 rollback을 검증`

**Source 확정 역할:** tracked `malloc`/`free`와 선택적 allocation failure를 이용해 split과 list mapping의 각 acquisition position에서 rollback을 재현하고 측정합니다.

#### Test commit 학습

- production 항상 유지해야 하는 조건 대상:
  - pre-allocation size safety와 single-allocation failure: production source 전체를 substituted allocator로 다시 compile하고 첫 allocation을 실패시켜 `ft_calloc`, `ft_strdup`, `ft_substr`, `ft_strjoin`, `ft_strtrim`, `ft_itoa`, `ft_strmapi`, `ft_lstnew`가 `NULL`과 live count 0을 남기는지 검사합니다. overflow branch 자체를 이 failure suite가 직접 주입하는 것은 아닙니다.
  - `ft_split` complete-or-rollback: 네 field 입력의 정상 allocation count가 root 1 + fields 4 = 5인지 먼저 측정하고, failure index 1부터 5까지 각각 실패시켜 result `NULL`, live 0, invalid free 0을 검사합니다.
  - `ft_lstmap` callback content + node rollback: 세 source node에서 production `ft_lstnew` allocation 1·2·3번째를 각각 실패시키고, 최신 unlinked callback content와 이전 mapped chain에 대한 destructor 호출 수가 failure index와 같은지 검사합니다.
- failure injection technique:
  - production API를 바꾸지 않고 `malloc`/`free`를 substitute하는 build 경계를 찾습니다. Makefile은 모든 `SRC`를 `build/failure` 아래 별도 object로 만들 때 `-Dmalloc=test_malloc -Dfree=test_free`를 적용합니다. test/support source와 test driver는 이 macro 없이 link되어 실제 allocator를 구현·사용합니다.
  - 몇 번째 allocation attempt를 실패시킬지 설정하는 state를 찾습니다. `test_allocator_reset(failure_index)`가 attempt, selected index, live, invalid-free counters와 tracking slots를 초기화하고, `test_malloc`이 attempt를 증가시킨 뒤 정확히 같은 index에서 `NULL`을 반환합니다. 0은 failure 비활성화입니다.
  - live object / invalid free를 기록하는 support code를 찾습니다. 최대 4096개 pointer slot을 기록하고 successful allocation마다 `g_live`를 증가시킵니다. `test_free`가 tracked pointer를 찾으면 slot을 비우고 live를 줄이며, 찾지 못하면 실제 free 대신 `g_invalid_frees`를 증가시킵니다.
- 실제 production path:
  - `ft_split`의 root allocation 및 각 field allocation 실패가 어떤 식으로 순회되는지 기록합니다. 정상 호출로 5 attempts를 확인한 뒤 index 1은 root, 2~5는 각 field acquisition을 실패시킵니다. 각 호출은 `ft_split`의 실제 `free_fields` 경로를 통과합니다.
  - `ft_lstmap`의 각 node allocation 실패가 어떤 callback/content 상태에서 발생하는지 기록합니다. test driver의 `map_integer`는 macro 없이 compile되어 callback content를 실제 `malloc`으로 만듭니다. 주입 대상은 production `ft_lstnew`의 node allocation뿐이며, 그 실패 직전에 callback content는 이미 생성된 상태입니다. 이는 node-failure rollback을 결정적으로 검사하지만 callback allocator failure 자체는 주입하지 않습니다.
- 측정 항목:
  - live allocation: substituted production allocation의 현재 개수입니다. split root/fields와 list nodes가 rollback 후 0인지 확인합니다.
  - double/invalid free: tracked set에 없는 pointer를 `test_free`에 넘겼는지 측정합니다. callback content는 실제 `malloc/free` 경로라 이 counter의 추적 대상이 아니며 destructor 호출 수와 프로세스 동작으로만 간접 관찰됩니다.
  - source mutation: list-map failure마다 source integer 값 `1, 2, 3`이 유지되는지 검사합니다. source `next` links는 별도로 assertion하지 않으므로 link 불변은 production write-target inspection에 근거합니다. split input mutation도 별도 buffer comparison은 없습니다.
  - returned value: 각 forced failure에서 `ft_split`과 `ft_lstmap`이 `NULL`인지 검사합니다. 정상 control run은 non-`NULL`과 예상 allocation/deletion count를 확인합니다.
- 테스트가 증명하는 것: 지정 input에서 single-allocation 첫 실패, split의 다섯 acquisition 위치, list node의 세 acquisition 위치를 deterministic하게 재현하며, tracked production allocation이 남지 않고 tracked invalid free가 없으며 list callback destructor 수와 source values가 예상과 같음을 검사합니다.
- 테스트가 증명하지 않는 것: 모든 input 크기, allocator가 실제로 반환하는 모든 failure timing, callback 내부 allocation failure, callback content의 이중 해제를 tracking counter로 검출하는 것, source link 불변, arithmetic overflow branch, thread safety는 증명하지 않습니다.
- 테스트 성격:
  - [ ] broad integration
  - [x] deterministic regression
  - [x] deterministic failure-injection suite
  - 선택 근거: production allocator 호출을 compile-time substitution하고 선택한 N번째 acquisition을 정확히 실패시키며 rollback counters를 고정 assertion합니다. 여러 subsystem의 외부 통합보다 failure branch를 반복 가능하게 재현하는 regression suite입니다.
- 후속 변경에서 막아야 할 회귀: split partial child/root leak, 실패 slot까지 잘못 free, mapped latest content 누락, partial mapped list leak, tracked node invalid/이중 해제, failure 후 non-`NULL` partial publish, source content 변조를 막습니다.
- 실행 근거: 저장소 checkout을 만들 수 없는 현재 환경에서는 `make failure-test`를 실행하지 않았습니다. 이 절의 수치와 경로는 `fd3ae063139d`의 Makefile, `tests/failure/test_failure.c`, `tests/support/fail_alloc.c`를 직접 검사한 결과이며 실행 성공을 기록하지 않습니다.

## 항상 유지해야 하는 조건 ledger

| 단계 | Commit | Source에 연결된 항상 유지해야 하는 조건 | 실제 코드에서 확인한 근거 |
| --- | --- | --- | --- |
| size foundation | `3b1b30983876` | multiplication/addition wrap을 allocation 전에 차단 | `ft_calloc`이 division guard 후에만 product를 계산하고, `ft_strdup`이 `length == SIZE_MAX`를 `length + 1` 전에 거부합니다. |
| single owner | `6d076de7185e` | 성공 시 독립 allocation 하나를 caller가 소유 | `ft_substr`이 suffix를 clamp한 뒤 새 block을 만들고 명시적으로 NUL을 써서 반환하며 out-of-range start도 `ft_strdup("")`로 독립 allocation을 만듭니다. |
| checked builder | `644b1c65444c` | terminator 포함 결합 크기를 allocation 전에 검증 | `right_length == SIZE_MAX` guard와 `left_length > SIZE_MAX - 1 - right_length` guard를 평가한 뒤 한 번만 `malloc`합니다. |
| multi-allocation transaction | `8c0a35a50878` | complete result 또는 total rollback | zeroed root에 field를 하나씩 연결하고 failure 시 `free_fields(fields, field_index)`가 연결된 child를 역순 해제한 뒤 root를 해제합니다. |
| list lifecycle | `7a016ad8fd21` | node/content lifetime 분리, clear 후 head `NULL` | `ft_lstdelone`이 `del(content)` 후 node를 free하고, `ft_lstclear`가 successor 저장·삭제·head 전진을 반복해 `*list == NULL`로 끝납니다. |
| callback-produced graph | `6672ea67fae4` | unlinked content와 partial list를 모두 rollback | node failure branch가 `del(mapped_content)` 후 `ft_lstclear(&mapped, del)`을 호출하며 source에는 write하지 않습니다. |
| deterministic evidence | `fd3ae063139d` | 각 allocation failure 위치에서 leak/invalid free 없이 cleanup | substituted allocator가 exact attempt를 실패시키고 split 1~5, map node 1~3에서 `NULL`, live 0, invalid 0, 예상 destructor count를 assertion합니다. |

## Failure → Fix → Test 연결

### `ft_split`

- 기존 single-allocation 가정: allocation이 하나이므로 `malloc` 실패 시 `NULL`만 반환하면 함수가 회수할 이전 resource가 없었습니다.
- multi-allocation에서 새로 생긴 failure 위험: root가 성공한 뒤 여러 field 중 하나가 실패하면 이전 field와 root가 동시에 live 상태로 남습니다.
- root cause: acquisition이 여러 단계로 나뉘는데 partial 소유권 범위와 cleanup count를 추적하지 않으면 caller에게 incomplete graph가 새거나 함수 내부에서 leak됩니다.
- `8c0a35a50878`의 complete-or-rollback decision: root를 먼저 zero-initialize하고 성공 child 수를 `field_index`로 유지하며, 모든 field가 끝난 경우에만 root를 publish합니다.
- 실제 cleanup 코드: `copy_field` 실패 시 `free_fields(fields, field_index)`가 이미 연결된 fields를 역순 free하고 root를 free한 뒤 `NULL`을 반환합니다.
- `fd3ae063139d`의 failure injection: 정상 control run으로 allocation count 5를 얻고 1~5번째를 각각 실패시켜 모든 acquisition branch를 통과합니다.
- regression으로 고정된 항상 유지해야 하는 조건: 어떤 acquisition이 실패해도 partial root가 반환되지 않고 tracked live allocation과 invalid free가 0입니다.

### `ft_lstmap`

- `7a016ad8fd21`이 제공한 lifecycle dependency: opaque content는 caller destructor로, node는 library free로 정리하고 partial list head를 `NULL`로 만드는 `ft_lstclear`입니다.
- callback content 생성 후 node allocation failure 위험: callback이 이미 새 content를 반환했지만 node가 없으므로 그 content는 아직 partial list에 연결되지 않아 list clear만으로 회수되지 않습니다.
- root cause: 최신 content와 이전에 연결된 node chain이 서로 다른 소유권 상태에 있습니다.
- `6672ea67fae4`의 cleanup decision: 최신 unlinked content를 직접 `del`하고, 이미 연결된 chain은 `ft_lstclear`에 맡깁니다.
- 실제 cleanup 코드: `if (node == NULL) { del(mapped_content); ft_lstclear(&mapped, del); return (NULL); }` 순서입니다.
- `fd3ae063139d`의 failure injection: production node allocator 1~3번째를 실패시키고 destructor 호출 수가 각각 1~3인지, tracked node live/invalid count가 0인지 검사합니다.
- regression으로 고정된 항상 유지해야 하는 조건: 최신 unlinked content와 이전 linked contents/nodes가 모두 한 번씩 정리되고 source content 값은 유지됩니다. callback allocation failure 자체는 이 harness의 주입 범위가 아닙니다.

## 소유권 / state / responsibility 변화

| 단계 | 새 resource | 소유권이 확정되는 시점 | failure 시 책임 | 학습자 근거 |
| --- | --- | --- | --- | --- |
| single allocation | `ft_calloc` block, duplicate string | 완성된 pointer를 함수가 반환할 때 caller로 이전 | allocation 전/실패에는 live resource가 없고 `NULL` 반환 | `src/alloc/ft_allocate.c`의 guard → malloc → initialize/copy → return 순서 |
| substring/join | 독립 substring 또는 joined string 한 개 | NUL-terminated result를 반환할 때 caller로 이전 | `malloc` 실패 시 `NULL`; 입력은 계속 caller 소유 | `ft_substr`, `ft_strjoin`이 input에 write하지 않고 result만 할당 |
| split root + fields | pointer-array root와 각 field string | child는 slot 대입 시 partial root가 소유하고, 완성 root 반환 시 graph 전체가 caller로 이전 | builder가 성공 child와 root를 모두 rollback | `field_index`, `free_fields`, zeroed sentinel |
| list node + opaque content | node storage와 caller-defined content | node는 list owner, content policy는 `del` callback이 결정 | `ft_lstdelone`/`ft_lstclear`가 content 후 node를 정리; `del == NULL`이면 소유권 유지 | `src/list/ft_list_lifecycle.c` guard와 successor-before-free loop |
| mapped content + mapped node | callback result와 새 node chain | callback 직후 builder가 content 소유; node 성공·link 후 partial mapped list가 소유; 전체 성공 시 caller로 이전 | 최신 unlinked content는 직접 `del`, linked chain은 `ft_lstclear` | `src/list/ft_list_map.c` failure branch와 tail append |

## Thread 최종 상태

- 마지막 commit 시점에 이 thread가 보장하는 것:
  - 기록: single allocation은 size를 먼저 검증하고 완성된 pointer만 반환합니다. `ft_split`과 `ft_lstmap`은 partial 소유권 상태를 내부에 유지하다 성공 시에만 complete graph를 반환하고, inspected failure paths는 확보한 resource를 전부 회수합니다. list cleanup은 caller destructor와 node free를 분리합니다. deterministic harness는 지정 acquisition 위치의 rollback을 검사합니다.
- 이 thread만으로는 보장하지 않는 것:
  - 기록: 임의 callback의 의미·thread safety·모든 input 규모·callback 내부 allocation failure·source link 불변을 test assertion으로 증명하지 않습니다. 실행 환경에서 failure suite를 실제 실행한 결과도 없습니다.
- source의 significance와 실제 코드 확인 결과가 연결되는 지점:
  - 기록: 소유권이 단일 반환값에서 root/children 및 callback content/node graph로 확장되고, `ft_split`의 count 기반 rollback과 `ft_lstmap`의 두 단계 cleanup을 exact failure injection이 보호한다는 점에서 source significance와 일치합니다. 단, map harness의 macro substitution은 production node allocation에만 적용되어 callback allocator 자체는 주입하지 않습니다.

## 최종 architecture 또는 실행 순서 정리

해당 thread의 commit history를 근거로 최종 흐름을 직접 작성합니다.

- 시작 조건 / 입력: valid string 또는 source list와, list content를 변환·삭제할 caller callbacks를 받습니다. allocation size를 계산하는 함수는 먼저 wrap 가능성을 제거합니다.
- 핵심 분기 또는 각 부분이 맡는 일의 구분: single-resource 함수는 allocation 성공/실패만 나뉩니다. multi-resource builder는 root/partial children을 local transaction으로 관리하고 complete 여부에 따라 publish 또는 rollback합니다. opaque content cleanup은 `del` callback 경계에 둡니다.
- 상태 또는 소유권 변화: 새 allocation은 처음에는 생성 함수가 소유합니다. split child는 root slot 연결 시 partial graph로, mapped content는 node 성공·link 시 partial list로 이전됩니다. complete return에서 graph 전체가 caller에게 이전됩니다.
- failure 처리: split은 성공 child count로 child와 root를 회수합니다. list map은 최신 unlinked content를 직접 지우고 기존 linked chain은 `ft_lstclear`에 넘깁니다. 어떤 경우에도 partial root를 반환하지 않습니다.
- verification 경로: failure build가 production `malloc/free`를 tracked functions로 치환해 exact attempt를 실패시키고 live/invalid counters, 반환값, destructor count, 일부 source 값을 검사합니다.
- 최종 설명: 이 Thread의 핵심은 `malloc` 성공 여부만 확인하는 것이 아니라 소유권이 어느 statement에서 어느 container로 이동했는지를 추적하는 것입니다. complete graph가 되기 전에는 builder가 모든 cleanup 책임을 유지하며, list의 opaque content는 caller가 제공한 destructor policy에 따라 node와 별도로 정리됩니다. failure harness는 이 규칙을 선택한 acquisition마다 반복 가능하게 통과시킵니다.

## 학습 완료 자가 점검

- [x] 모든 commit을 문서 순서대로 해당 SHA에서 확인했습니다.
- [x] 중요도와 tags를 source 그대로 유지했습니다.
- [x] 실제 코드 근거와 source 확정 설명을 구분했습니다.
- [x] 변경 전/후 비교가 필요한 commit은 이전 관련 SHA와 비교했습니다.
- [x] failure → fix → test 연결을 실제 코드와 test code로 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급하지 않았습니다.
- [x] 이 thread의 최종 항상 유지해야 하는 조건과 execution flow를 코드 근거로 설명할 수 있습니다.
===== END FILE: 02-single-allocation-to-rollback-safe-소유권.md =====

===== BEGIN FILE: 03-fd-output-partial-system-calls.md =====
# Thread: Hardening file-descriptor output against partial system calls

## Thread 목표

**Source significance**

> The initial 공개 API cannot return status, so reliability has to be enforced internally and failure has to stop further composite output. The thread evolves from formatting correctness to a system-call progress 항상 유지해야 하는 조건, then proves that 항상 유지해야 하는 조건 with a deterministic substitute for `write`.

### 이 Thread에 직접 연결된 source 항상 유지해야 하는 조건

> File-descriptor output advances after every positive short write, retries `EINTR`, treats zero progress as failure, and stops composite output after a permanent error.

### 이 Thread에 직접 연결된 engineering difficulties

> Preserving output progress across short system calls while fitting a public `void` descriptor API that cannot return an error status.

> Reproducing allocation and write failures deterministically without changing the production API.

## 이 Thread를 이해하기 위한 핵심 질문

- 초기 one-shot `write`가 어떤 failure/partial-success 상태를 처리하지 못하는가?
- public API가 `void`일 때 내부 helper는 failure를 어떻게 전달해 composite output을 멈추는가?
- positive short write 이후 offset/remaining state는 어떤 순서로 갱신됩니까?
- `EINTR`, zero progress, permanent error는 각각 retry/stop 정책이 어떻게 다른가?
- number output refactor가 이후 공통 write path에 어떤 연결을 만드는가?
- deterministic scripted `write`가 실제 OS timing에 의존하지 않고 retry sequence를 어떻게 증명하는가?

## 완료 기준

- `26509fd54c3d`의 one-shot policy와 `3f2bfbf11e1f`의 write-until-complete policy를 실제 코드로 비교했습니다.
- short write 후 progress state, `EINTR` retry, zero-progress failure, permanent-error stop을 순서대로 추적했습니다.
- sign/prefix 또는 앞선 component failure 뒤 후속 출력이 중지되는 control flow를 확인했습니다.
- `b013c926ceb5`에서 scripted return/error sequence와 production call sequence가 1:1로 연결되는지 확인했습니다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags | Source role |
| --- | --- | --- | --- | --- | --- |
| 1 | `26509fd54c3d` | `feat(io): 파일 디스크립터 출력 함수 추가` | B | FD_OUTPUT, CORE | Adds the initial void-returning descriptor API and one-shot writes. |
| 2 | `60c35f2fb431` | `test(io): 파일 디스크립터 출력 검증` | B | FD_OUTPUT, TEST | Captures normal bytes and establishes the initial invalid-descriptor and broken-pipe observations. |
| 3 | `1077556d1c4b` | `refactor(io): 숫자 출력을 자릿수 helper로 분리` | B | FD_OUTPUT, REFACTOR | Routes integer digits through the common character-output path. |
| 4 | `3f2bfbf11e1f` | `fix(io): 파일 디스크립터 출력을 끝까지 재시도` | S | FD_OUTPUT, CORE, RISK | Introduces write-until-complete behavior, `EINTR` retry, zero-progress rejection, and permanent-error stopping. |
| 5 | `b013c926ceb5` | `test(io): 부분 쓰기와 EINTR 이후 진행을 검증` | A | FD_OUTPUT, TEST, RISK | Replaces nondeterministic operating-system timing with scripted write results and verifies the exact retry sequence. |

## Commit별 학습 기록

### `26509fd54c3d` — `feat(io): 파일 디스크립터 출력 함수 추가`

**Source 확정 역할:** arbitrary 파일 디스크립터를 대상으로 하는 initial void-returning output API를 one-shot writes로 도입합니다.

#### 해당 SHA에서 확인할 코드

- character, string, newline, signed-decimal descriptor helper의 public declarations와 implementations를 찾습니다.
- composite newline output이 string/character primitive를 어떤 순서로 재사용하는지 확인합니다.
- integer output이 fixed stack buffer를 만들고 한 번의 `write`로 제출하는 경로를 확인합니다.
- `write` 반환값을 버리는 지점을 찾고 public `void` API와 연결해 기록합니다.
- `INT_MIN` magnitude가 signed negation 없이 처리되는 코드를 확인합니다.
- short write 또는 `EINTR` 이후 남은 byte를 재시도하는 loop가 없는지 확인합니다.

#### 학습 기록

- 공개 API contract: `libft.h`에 `ft_putchar_fd`, `ft_putstr_fd`, `ft_putendl_fd`, `ft_putnbr_fd`가 모두 `void`로 추가됩니다. caller는 대상 fd와 값만 넘기며 함수 반환값으로 성공 byte 수나 error status를 받을 수 없습니다.
- normal formatting path: character는 주소와 길이 1을 직접 씁니다. string은 `ft_strlen(text)` bytes를 씁니다. newline 함수는 먼저 `ft_putstr_fd`, 다음 `ft_putchar_fd('\n', fd)`를 호출합니다. number 함수는 decimal text를 stack buffer 뒤에서부터 구성합니다.
- `write` 호출 단위: character와 string은 각각 한 번, newline은 내부 두 primitive 때문에 최대 두 번입니다. number는 sign과 digits를 모두 buffer에 만든 뒤 한 번의 `write(fd, buffer + index, sizeof(buffer) - index)`로 제출합니다.
- error observation 수단: 구현은 모든 반환값을 `(void)write(...)`로 버립니다. public status는 없고, caller가 관찰할 수 있는 것은 실제 fd에 남은 bytes, process signal, 또는 `write`가 설정한 `errno`와 같은 외부 효과뿐입니다.
- short write에서 아직 보장하지 않는 것: `write`가 요청보다 작은 양수를 반환해도 남은 bytes를 다시 제출하지 않으므로 정상 prefix만 출력되고 suffix가 누락될 수 있습니다.
- `EINTR`에서 아직 보장하지 않는 것: `write`가 `-1`과 `EINTR`을 반환해도 retry branch가 없어 해당 component 전체가 누락됩니다.
- composite output의 failure continuation 가능성: `ft_putendl_fd`는 string write 성공 여부를 알지 못한 채 newline을 이어서 출력합니다. number는 one-shot이므로 앞부분 성공/실패 이후 별도 component stop 제어가 없습니다.
- `INT_MIN` 처리 근거: negative number의 magnitude는 `-(number + 1)`을 먼저 계산한 뒤 `+ 1U`를 적용합니다. `-INT_MIN`을 signed `int`에서 직접 계산하지 않아 overflow를 피하고 `unsigned int`로 모든 digits를 구성합니다.

### `60c35f2fb431` — `test(io): 파일 디스크립터 출력 검증`

**Source 확정 역할:** pipe capture로 정상 byte sequence를 확인하고, invalid closed descriptor와 suppressed `SIGPIPE` 아래 broken pipe의 초기 observable 동작을 기록합니다.

#### Test commit 학습

- production 항상 유지해야 하는 조건/contract 대상:
  - 정상 formatting과 ordering: pipe write end에 네 helper를 조합해 `Afoundation\n0|-1|2147483647|-2147483648`을 출력하고, write end를 닫은 뒤 read end에서 예상 길이와 byte sequence를 비교합니다.
  - invalid descriptor control return: 새 pipe 양쪽을 닫은 뒤 closed fd에 네 helper를 호출합니다. 명시적 return/`errno` assertion은 없고, test process가 종료·hang하지 않고 함수 호출 뒤로 돌아오는지만 간접 관찰합니다.
  - broken pipe에서 `EPIPE` 관찰: `SIGPIPE` handler를 `SIG_IGN`으로 바꾸고 pipe read end를 닫은 뒤 `ft_putstr_fd("closed", pipe_fd[1])`를 호출해 `errno == EPIPE`인지 확인합니다. 마지막에 이전 시그널 처리 함수를 복구합니다.
- test technique:
  - pipe capture 구성 지점을 찾습니다. `pipe(pipe_fd)` 뒤 helper들이 write end를 사용하고, write end close 후 한 번의 `read`로 `actual`을 채워 exact byte count와 `memcmp`를 검사합니다.
  - `SIGPIPE` suppression과 `errno` 관찰 위치를 찾습니다. `signal(SIGPIPE, SIG_IGN)`, read end close, `errno = 0`, production call, `CHECK(errno == EPIPE)` 순서입니다.
- 실제 production code path:
  - 네 output helper 각각 어떤 write path를 통과하는지 기록합니다. character와 string은 one-shot direct write, newline은 string 다음 character, number는 stack buffer를 만든 뒤 one-shot write입니다. 정상 case는 숫자 0, -1, `INT_MAX`, `INT_MIN`을 모두 통과합니다.
- 이 테스트가 증명하는 것: 일반 pipe에서 선택한 formatting 조합의 byte 순서·길이, 극값 decimal formatting, closed read end에서 suppressed signal 아래 `EPIPE` 관찰을 확인합니다. closed fd 호출이 control을 반환하는 것도 실행 경로에 포함합니다.
- 이 테스트가 증명하지 않는 것:
  - source상 partial write / interrupted write completion은 아직 증명하지 않습니다. 실제 test code가 이를 주입하지 않는지 확인합니다. pipe가 실제로 short write나 `EINTR`을 반환하도록 제어하지 않으며, 한 번의 `read`가 충분하다는 작은 output 조건에 의존합니다. invalid fd에 대한 exact `errno`나 호출 횟수도 assertion하지 않습니다.
- 테스트 성격:
  - [ ] broad integration
  - [x] deterministic regression
  - [x] ordinary pipe/error observation
  - 선택 근거: 고정 byte sequence와 signal/error setup을 반복하는 regression이지만 system `write` 결과를 substitute하지 않고 실제 pipe와 host error observation에 의존합니다. 여러 subsystem release를 검증하는 broad integration은 아닙니다.
- 다음 refactor/fix와 연결: formatting은 확인됐지만 partial progress와 interruption은 통제되지 않습니다. number refactor는 digits를 common character path로 모으고, 이후 fix는 모든 helper가 completion helper를 사용하도록 바꿔야 합니다.
- 실행 근거: 현재 환경에서는 이 SHA의 binary를 실행하지 않았습니다. 위 내용은 `tests/test_fd_output.c`와 `src/io/ft_fd_output.c`를 해당 SHA에서 직접 검사한 결과입니다.

### `1077556d1c4b` — `refactor(io): 숫자 출력을 자릿수 helper로 분리`

**Source 확정 역할:** signed-decimal output을 sign handling + recursive unsigned digit emitter로 분리하고 각 digit을 공통 character-output primitive로 보냅니다.

#### 해당 SHA에서 확인할 코드

- 이전 `26509fd54c3d`의 fixed decimal buffer 경로와 이 SHA의 구현을 비교합니다.
- sign handling과 unsigned magnitude 계산을 찾습니다.
- recursive digit emitter의 base/recursive case를 실제 코드에서 기록합니다.
- 각 digit이 어떤 common character-output path를 통과하는지 caller/callee를 추적합니다.
- public 동작이 아직 short-write reliability를 보장하도록 바뀐 것이 아닌지 source role과 실제 코드로 구분합니다.
- one-byte writes 증가라는 감수할 점이 구현상 어떻게 나타나는지 확인합니다.

#### 학습 기록

- 변경 전 number-output path: `char buffer[11]`을 끝에서 앞으로 채우고 sign을 추가한 뒤 전체 decimal representation을 단일 `write`로 제출했습니다.
- 변경 후 path: `ft_putnbr_fd`가 negative sign을 `ft_putchar_fd`로 먼저 쓰고 같은 overflow-safe magnitude를 계산합니다. `put_unsigned`는 magnitude가 10 이상이면 quotient에 재귀 호출한 뒤 현재 remainder digit을 출력합니다.
- 공통 emission boundary: 모든 sign과 digit이 `ft_putchar_fd`를 거치므로 실제 system-call boundary가 character primitive 하나로 통일됩니다.
- public behavior 유지 근거: declaration과 반환형은 그대로 `void`이고 decimal order와 `INT_MIN` magnitude 계산도 유지됩니다. 그러나 `ft_putchar_fd` 자체가 여전히 one-shot `write` 결과를 버리므로 short-write reliability가 추가된 것은 아닙니다.
- 다음 fix가 이 구조를 이용할 수 있는 지점: character primitive에 completion 동작을 넣으면 숫자 각 digit도 그 path를 사용합니다. 다만 digit별 failure를 상위 recursion과 sign 이후 제어로 전달하려면 private helper 반환형을 `int` 등으로 바꿔야 합니다.
- 감수할 점: 이전 number당 한 번이던 system call이 sign과 각 digit마다 한 byte write로 늘어납니다. 이 commit은 구조 통일을 선택했으며 batching 성능을 개선하지 않습니다.

### `3f2bfbf11e1f` — `fix(io): 파일 디스크립터 출력을 끝까지 재시도`

**Source 확정 역할:** positive short write progress 보존, `EINTR` retry, zero progress rejection, permanent-error stop을 포함하는 project-defining system-call 항상 유지해야 하는 조건을 복구합니다.

#### 기존 가정 → 실제 failure 또는 위험

- 기존 one-shot assumption: 요청한 byte count가 한 번의 `write`로 모두 처리되거나, 실패하더라도 public `void` API에서는 더 할 일이 없다고 간주했습니다.
- positive short write에서 생기는 truncation: 반환값만큼은 정상 출력됐지만 아직 남은 suffix가 존재합니다. 기존 코드는 반환값을 버려 prefix progress와 remaining range를 잃습니다.
- `EINTR`을 completion으로 취급할 때의 문제: signal interruption은 byte가 기록되지 않은 transient error일 수 있는데 retry하지 않으면 component가 누락됩니다.
- 모든 error를 무조건 retry할 때의 문제: `EPIPE`, `EBADF`, `EIO` 같은 permanent error나 zero progress에서 같은 request를 반복하면 무한 loop 또는 오류 뒤의 부가 output이 발생할 수 있습니다.
- public `void` API가 직접 status를 돌려줄 수 없는 제약: caller에게 실패를 반환할 수 없으므로 private helper가 성공/실패를 반환하고 composite helper 내부에서 후속 write를 중지해야 합니다. 최종 `errno`는 system call 또는 zero-progress mapping으로 남습니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

- private `write_all` 또는 source가 설명한 completion helper를 찾습니다.
- 한 번의 request가 `SSIZE_MAX`를 넘지 않도록 제한하는 코드를 확인합니다.
- `write`가 양수를 반환했을 때 pointer/offset과 remaining byte count가 **반환된 수만큼만** 전진하는지 확인합니다.
- `write == -1`과 `errno == EINTR`인 branch가 progress를 중복 적용하지 않고 retry하는지 확인합니다.
- `write == 0`에서 `EIO`로 전환하고 종료하는 경로를 확인합니다.
- 다른 permanent error에서 즉시 stop하는 경로를 확인합니다.
- character/string/newline/integer helper가 새 completion path로 어떻게 route되는지 확인합니다.
- recursive number output에서 내부 failure status가 상위 호출로 어떻게 propagation되는지 추적합니다.
- sign emission 실패 후 digit emission이 중지되는지 확인합니다.
- composite newline 출력에서 앞 component failure 후 다음 write를 하지 않는지 확인합니다.

#### 상태 전이 기록

| 상황 | 이전 remaining/progress | system call 결과 | 다음 state | retry/stop | 실제 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| positive full write | `remaining = request`, `offset = k` | `written == request > 0` | `offset += written`, 전체 length에 도달 | loop 종료 / success | `if (written > 0) offset += (size_t)written` 후 `offset < length` 재평가 |
| positive short write | `remaining = length - offset` | `0 < written < request` | 정확히 `written`만큼 offset 증가, 새 remaining 계산 | 남은 range retry | 다음 iteration의 `request = length - offset` |
| `EINTR` | 기존 offset과 remaining 유지 | `written < 0`, `errno == EINTR` | state 변화 없음 | 같은 remaining retry | `continue` 전에 offset 갱신 없음 |
| zero progress | 기존 offset과 remaining 유지 | `written == 0` | `errno = EIO`, progress 없음 | stop / failure 0 | error branch의 explicit `written == 0` 처리 |
| permanent error | 기존 offset과 remaining 유지 | `written < 0`, `errno != EINTR` | system `errno` 유지 | 즉시 stop / failure 0 | `else` branch에서 추가 call 없이 return 0 |

#### 수정된 항상 유지해야 하는 조건

- progress가 보존되는 조건: system call이 양수를 반환한 경우에만 그 반환 byte 수만큼 offset을 증가시킵니다. 다음 request는 남은 길이이며 `buffer + offset`부터 시작합니다.
- retry 가능한 유일한 error: 이 구현에서 명시적으로 retry하는 error는 `EINTR`뿐입니다. progress를 적용하지 않고 같은 remaining range를 재요청합니다.
- zero progress 처리: length가 남았는데 `write`가 0이면 더 진행된다는 보장이 없으므로 `errno = EIO`로 바꾸고 실패합니다. 무한 retry하지 않습니다.
- composite stop 조건: `ft_putendl_fd`는 text `write_all`이 성공한 경우에만 newline을 씁니다. negative number는 sign write가 실패하면 return하고, recursive digit helper는 하위 digit 실패를 0으로 전파해 이후 digits를 중지합니다.
- public API가 `void`여도 내부적으로 확보되는 보장: caller는 status를 받지 못하지만, transient `EINTR`과 short write는 completion까지 처리되고, permanent failure 후 동일 composite operation의 추가 bytes는 쓰지 않습니다. 오류를 성공으로 바꾸거나 caller에게 report하는 보장은 없습니다.
- request bound: 매 iteration의 remaining이 `SSIZE_MAX`보다 크면 request를 `SSIZE_MAX`로 제한해 `write`의 representable return 범위 안에 둡니다.

#### 후속 검증 연결

- ordinary pipe test로 결정적으로 만들기 어려운 scenario: 정확히 2 bytes short write 후 `EINTR`, 다시 1 byte와 3 bytes 성공, zero return, 특정 digit 뒤 `EPIPE`처럼 OS scheduling에 좌우되는 sequence입니다.
- `b013c926ceb5`가 scripted result로 고정해야 하는 순서: request lengths와 fd, call count, copied output bytes를 모두 기록하며 positive progress 뒤 remaining 감소, `EINTR` 뒤 same request, zero/hard error 뒤 no further call을 assertion해야 합니다.

### `b013c926ceb5` — `test(io): 부분 쓰기와 EINTR 이후 진행을 검증`

**Source 확정 역할:** deterministic scripted `write` 결과를 사용해 partial progress, interruption, zero, permanent error의 정확한 retry/stop sequence를 검증합니다.

#### Test commit 학습

- production 항상 유지해야 하는 조건 대상:
  - positive short write 후 남은 byte만 재시도: `"abcdef"`에 첫 result 2를 주고 다음 request가 4인지 검사합니다.
  - `EINTR` retry: 두 번째 call이 `-1/EINTR`일 때 세 번째 request가 여전히 4이고 output progress가 중복되지 않는지 검사합니다.
  - zero progress rejection: 2 bytes 성공 뒤 0을 반환시켜 두 call에서 멈추고 output이 `"ab"`, `errno == EIO`인지 검사합니다.
  - permanent error 이후 composite stop: `ft_putendl_fd("error")`에서 1 byte 성공 뒤 `-1/EIO`를 반환시켜 newline과 남은 text를 쓰지 않는지 검사합니다. number에서도 `"-2"` 뒤 `EPIPE`가 나면 뒤 digits를 중지합니다.
- failure injection technique:
  - 실제 `write`를 substitute하는 test/build boundary를 찾습니다. Makefile은 `src/io/ft_fd_output.c`만 별도 `build/write-failure/ft_fd_output.o`로 compile하며 `-Dwrite=test_write`를 적용합니다. 이 object와 support/test source, 기존 archive를 link합니다.
  - scripted 반환값과 `errno`를 저장/소비하는 state를 찾습니다. `t_write_step { ssize_t result; int error_number; }` 배열을 reset 시 저장하고 `test_write`가 call index 순서대로 step을 소비합니다. negative result는 지정 `errno`와 `-1`을 반환합니다.
  - production이 요청한 buffer pointer/length 또는 call count를 기록하는 instrumentation을 확인합니다. fd와 requested length를 배열에 기록하고 positive result만큼 현재 buffer bytes를 output buffer에 복사합니다. script 고갈, result > request, output overflow는 `g_invalid`로 표시합니다.
- production path 추적:
  - short write → next request: 6-byte 요청에서 2를 반환해 offset 2, 다음 request 4가 됩니다.
  - `EINTR` → same remaining range retry: request 4에서 error가 나고 offset이 그대로라 다음 request도 4입니다.
  - zero → `EIO`/stop: 두 번째 call에서 0이면 third scripted success는 소비되지 않고 call count 2로 끝납니다.
  - hard error → no later component: text 중간 EIO 뒤 newline이 없고, number digit EPIPE 뒤 다음 one-byte step을 소비하지 않습니다.
- exact retry sequence:
  - test script: partial/interrupted case는 `{2, 0}, {-1, EINTR}, {1, 0}, {3, 0}`입니다.
  - 예상 call sequence: fd 91로 request lengths `6 → 4 → 4 → 3`, 총 4 calls이며 output은 순서대로 `ab`, 변화 없음, `c`, `def`입니다.
  - 실제 assertion: calls 4, 첫/마지막 fd 91, 각 request length, output size 6, `memcmp(..., "abcdef", 6)`, invalid 0을 각각 검사합니다.
- 테스트가 증명하는 것: 고정된 short write, `EINTR`, zero, EIO/EPIPE sequence에서 offset과 request length, call stop, accumulated output, `errno`가 예상과 같음을 production `write_all` 경로에 대해 결정적으로 검사합니다. `INT_MIN`의 sign/digits propagation도 포함합니다.
- 테스트가 증명하지 않는 것: 실제 kernel pipe/socket의 concurrency와 signal timing, `SSIZE_MAX`보다 큰 실사용 buffer, 모든 `errno`, nonblocking `EAGAIN` 정책, thread safety, public caller의 error reporting은 증명하지 않습니다.
- 테스트 성격:
  - [ ] broad integration
  - [x] deterministic regression
  - [x] deterministic system-call failure injection
  - 선택 근거: production system-call symbol을 compile-time substitute하고 exact return/error script, request lengths, output, call count를 assertion합니다. 실제 OS timing을 관찰하는 test가 아니라 progress 항상 유지해야 하는 조건 전용 regression입니다.
- 후속 변경에서 막아야 할 회귀: short write 뒤 original length 재요청, offset을 requested size만큼 잘못 증가, `EINTR`에서 중복 progress, zero에서 무한 loop, hard error 뒤 newline/digits 계속 출력, sign 실패 뒤 digits 출력, 잘못된 fd 전달을 막습니다.
- 실행 근거: 현재 환경에서는 `make write-failure-test`를 실행하지 않았습니다. 위 exact sequence는 `b013c926ceb5`의 test/support code와 Makefile을 검사한 결과이며 실제 실행 성공을 주장하지 않습니다.

## 항상 유지해야 하는 조건 ledger

| 단계 | Commit | Source에 연결된 항상 유지해야 하는 조건 상태 | 실제 코드에서 확인한 근거 |
| --- | --- | --- | --- |
| initial API | `26509fd54c3d` | one-shot write, completion invariant 미확립 | 네 public helper가 `write` 반환값을 버리고 short write/`EINTR` loop 없이 동작합니다. |
| initial observation | `60c35f2fb431` | 정상 bytes와 기본 error observation만 검증 | 실제 pipe로 expected bytes와 `EPIPE`를 확인하지만 write return sequence를 통제하지 않습니다. |
| common path preparation | `1077556d1c4b` | number digits가 common character path를 사용 | recursive `put_unsigned`와 sign이 모두 `ft_putchar_fd`를 호출합니다. |
| fix / invariant restoration | `3f2bfbf11e1f` | progress, `EINTR`, zero, permanent error 정책 확립 | `write_all`이 positive bytes만큼 offset을 갱신하고 EINTR만 retry하며 zero를 EIO, 다른 error를 stop으로 처리합니다. |
| deterministic regression | `b013c926ceb5` | exact retry/stop sequence 강제 검증 | substituted `test_write`가 scripted results를 반환하고 fd/request/output/call count를 기록해 `6→4→4→3` 및 stop cases를 assertion합니다. |

## Failure → Fix → Test 연결

- 기존 가정: one-shot `write`가 요청을 충분히 처리한다고 간주
- 실제 failure/위험: short write, `EINTR`, zero progress, permanent error 후 잘못된 continuation
- root cause: 반환값과 `errno`를 버려 이미 전송된 byte 수와 남은 range, retry 가능 여부를 state로 유지하지 않았습니다.
- 구조 준비: `1077556d1c4b`
- fix: `3f2bfbf11e1f`
- 실제 수정 코드: private `write_all`이 bounded request, offset, positive progress, EINTR retry, zero/hard error stop을 담당하고, newline/number private paths가 boolean success를 전파합니다.
- regression test: `b013c926ceb5`
- failure injection script: `write`를 `test_write`로 바꾸고 `{result, errno}` steps를 순서대로 제공하며 request/call/output를 기록합니다.
- 고정된 항상 유지해야 하는 조건: 양수 반환만큼만 전진하고, EINTR에는 같은 remaining range를 재시도하며, zero와 permanent error에서는 추가 component를 쓰지 않습니다.

## State / responsibility 변화

- 초기 public `void` API가 caller에게 제공하지 못하는 status: 성공 byte 수, partial progress, 최종 error를 반환값으로 전달하지 못합니다.
- 내부 helper가 새로 맡는 completion responsibility: 전체 length를 쓰거나 실패할 때까지 offset과 remaining을 유지하고 system-call 결과를 분류합니다.
- composite helper가 맡는 stop-on-error responsibility: 앞 component의 private failure를 검사해 newline, sign 이후 digits, recursive 후속 digits를 중지합니다.
- test harness가 맡는 deterministic system-call boundary: 실제 OS 대신 exact short/error results를 공급하고 production request와 output progression을 측정합니다.

## Thread 최종 상태

- 마지막 commit 시점에 이 thread가 보장하는 것:
  - 기록: valid text/value와 fd 입력에서 내부 completion path는 positive short write를 모두 이어 쓰고 EINTR을 재시도합니다. zero는 EIO, permanent error는 즉시 stop이며 composite helper는 앞선 failure 뒤 후속 bytes를 제출하지 않습니다. scripted test가 대표 sequence를 정확히 검사합니다.
- 이 thread만으로는 보장하지 않는 것:
  - 기록: public API가 caller에게 status를 반환하지 않고, `NULL` string 방어, nonblocking `EAGAIN` retry, 모든 kernel timing, atomic multi-byte output, thread safety를 보장하지 않습니다. 현재 작업 환경에서 test 명령은 실행하지 않았습니다.
- source의 significance와 실제 코드 확인 결과가 연결되는 지점:
  - 기록: API signature를 바꾸지 않고 private status propagation과 stop-on-error를 추가했으며, nondeterministic system call을 scripted substitute로 바꿔 exact progress 항상 유지해야 하는 조건을 검증한다는 점이 source significance와 일치합니다.

## 최종 architecture 또는 실행 순서 정리

해당 thread의 commit history를 근거로 최종 흐름을 직접 작성합니다.

- 시작 조건 / 입력: public helper가 fd와 character, NUL-terminated text, newline text, 또는 `int`를 받습니다. 문자열 pointer의 유효성은 caller precondition입니다.
- 핵심 분기 또는 각 부분이 맡는 일의 구분: public `void` wrapper는 private `write_all`/`put_unsigned` status를 내부 제어에만 사용합니다. `write_all`은 positive, EINTR, zero, permanent error를 분기합니다.
- 상태 또는 소유권 변화: allocation은 없습니다. local `offset`만 이미 처리된 prefix를 나타내며, remaining은 `length - offset`으로 매 iteration 재계산됩니다.
- failure 처리: EINTR만 state 변화 없이 retry합니다. zero는 EIO로 바꿔 실패하고 다른 error는 errno를 유지한 채 실패합니다. composite operation은 실패 status를 받으면 즉시 반환합니다.
- verification 경로: normal pipe test는 formatting과 기본 error를 관찰하고, special build는 `write`를 scripted function으로 치환해 request lengths, fd, accumulated bytes, call count를 검사합니다.
- 최종 설명: 이 Thread는 출력 문자열을 올바르게 만드는 문제에서 system call의 부분 성공을 올바르게 이어 가는 문제로 확장됩니다. public API는 여전히 `void`지만 private helper가 progress와 failure를 명시적으로 관리해 잘린 출력과 오류 뒤 continuation을 막습니다.

## 학습 완료 자가 점검

- [x] 모든 commit을 문서 순서대로 해당 SHA에서 확인했습니다.
- [x] 중요도와 tags를 source 그대로 유지했습니다.
- [x] 실제 코드 근거와 source 확정 설명을 구분했습니다.
- [x] 변경 전/후 비교가 필요한 commit은 이전 관련 SHA와 비교했습니다.
- [x] failure → fix → test 연결을 실제 코드와 test code로 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급하지 않았습니다.
- [x] 이 thread의 최종 항상 유지해야 하는 조건과 execution flow를 코드 근거로 설명할 수 있습니다.
===== END FILE: 03-fd-output-partial-system-calls.md =====

===== BEGIN FILE: 04-static-archive-release-verification.md =====
# Thread: Treating the static archive as a verified release artifact

## Thread 목표

**Source significance**

> The project stops treating a successful local compile as sufficient evidence. Compiler builtins are excluded, the archive and consumer boundary are inspected directly, independent defect detectors cover different failure classes, and the same evidence is reproduced across compiler families before being orchestrated into one release check.

### 이 Thread에 직접 연결된 source 항상 유지해야 하는 조건

> The archive contains the intended translation units and public symbols only, links from outside the repository, and depends only on the explicitly allowed runtime functions.

> Tests of reimplemented libc-style functions must not be invalidated by compiler builtin substitution.

### 이 Thread에 직접 연결된 engineering difficulty

> Inspecting archive symbols portably enough for both Darwin and Linux and validating the same source under Clang and GNU GCC.

## 이 Thread를 이해하기 위한 핵심 질문

- compiler builtin substitution이 libc reimplementation test의 신뢰성을 어떻게 훼손할 수 있으며 build flags는 이를 어디서 차단하는가?
- `libft.a`의 member, exported symbol, undefined external dependency는 어떤 manifest/inspection 단계로 검증됩니까?
- out-of-tree consumer는 in-tree path/include 의존성을 어떻게 드러내는가?
- UBSan, ASan, host leak checking은 서로 어떤 evidence 범위를 가지는가?
- Clang/GCC clean copied tree 실행은 어떤 compiler-specific assumption을 잡기 위한 것입니까?
- 최종 release target은 기존 검증을 어떤 순서와 경계로 orchestration하는가?

## 완료 기준

- build flags에서 strict C99 warning과 builtin policy를 실제로 확인했습니다.
- archive member/public symbol/allowed dependency/out-of-tree consumer 검증 경로를 `79c0dcefb590`에서 직접 추적했습니다.
- UBSan, ASan, host leak 검사 각각의 build/run target과 증명 범위를 구분했습니다.
- Clang/GCC 검증이 clean copied tree에서 full suite를 실행하는지 확인했습니다.
- 최종 orchestration target이 기존 evidence를 재사용하는 순서와 failure propagation을 확인했습니다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags | Source role |
| --- | --- | --- | --- | --- | --- |
| 1 | `4df8b23505b8` | `build(flags): C99 경고와 builtin 정책을 고정` | A | RELEASE, VERIFY, RISK | Locks C99 warnings and disables compiler builtin substitution. |
| 2 | `79c0dcefb590` | `test(release): archive와 consumer 경계를 검증` | A | RELEASE, ARCH, VERIFY | Verifies archive members, public definitions, allowed external dependencies, and an out-of-tree consumer. |
| 3 | `f5de4306ebcd` | `test(sanitize): undefined behavior 검사를 추가` | B | VERIFY, TEST | Adds undefined-behavior sanitizer execution. |
| 4 | `c625970fd211` | `test(sanitize): address sanitizer 검사를 추가` | B | VERIFY, TEST | Adds address-sanitizer execution. |
| 5 | `9f555c37a6d8` | `test(leak): host 누수 검사 경로를 추가` | B | VERIFY, TEST | Adds host leak checking. |
| 6 | `e31a2e748685` | `test(build): Clang과 GCC 호환성을 검증` | A | RELEASE, VERIFY | Runs the complete release-oriented suite under both Clang and GNU GCC in clean copied trees. |
| 7 | `b90fd748255a` | `test(release): 전체 검증 절차를 연결` | B | RELEASE, VERIFY | Connects clean build, functional, failure, sanitizer, archive, compiler, leak, and no-op rebuild checks. |

## Commit별 학습 기록

### `4df8b23505b8` — `build(flags): C99 경고와 builtin 정책을 고정`

**Source 확정 역할:** strict C99 warnings와 compiler builtin 비활성화를 고정해 low-level reimplementation 검증의 compiler boundary를 강화합니다.

#### 해당 SHA에서 확인할 코드 / build 설정

- Makefile 또는 실제 build configuration에서 language standard와 warning flags가 어디에 정의되는지 찾습니다.
- compiler builtins를 비활성화하는 flag가 production build와 test build 중 어디에 적용되는지 확인합니다.
- source가 언급한 bonus target exposure가 어떤 target dependency로 표현되는지 확인합니다.
- 이전 SHA와 비교해 flag 변경이 compile command에 실제 반영되는 지점을 확인합니다.
- test 대상 함수가 compiler builtin으로 대체될 가능성을 막는 설정이 전체 relevant object에 적용되는지 확인합니다.

#### 학습 기록

- 직전 build policy: `CFLAGS := -Wall -Wextra -Werror -std=c99 -pedantic`, `CPPFLAGS := -I.`였습니다. strict warning과 C99는 이미 있었지만 command-line override를 막지 않았고 `-fno-builtin`은 없었습니다. `bonus` target도 없었습니다.
- 추가/고정된 flags: `override CFLAGS := -Wall -Wextra -Werror -Wpedantic -std=c99 -fno-builtin`, `override CPPFLAGS := -I.`로 바뀝니다. GNU Make의 `override` 지시어 때문에 명령줄의 `CFLAGS`/`CPPFLAGS`만으로 이 정책을 제거할 수 없습니다. `bonus: all`도 추가돼 같은 archive build를 노출합니다.
- builtin substitution 위험: libc와 유사한 low-level 구현 또는 그 호출을 compiler가 builtin knowledge로 대체·접어 버리면 실제 project implementation을 통과하지 않은 결과를 test가 관찰할 수 있습니다. 이 commit은 relevant compile에서 builtin substitution을 허용하지 않는 정책을 선언합니다.
- 실제 compile command 근거: ordinary object rule과 test binary link/compile recipe가 모두 `$(CPPFLAGS) $(CFLAGS)`를 사용합니다. 별도 write-failure object rule도 같은 공통 flags에 `$(WRITE_DEFINES)`를 추가하므로 production, ordinary test, special I/O object에 정책이 전달됩니다.
- 이 commit이 보장하는 verification boundary: project가 의도한 C99 dialect, warning-as-error, pedantic diagnostics, no-builtin 설정으로 source와 tests를 compile하도록 Makefile 수준에서 고정합니다. `bonus`도 별도 구현이 아닌 `all`의 alias입니다.
- 아직 archive 자체에 대해 보장하지 않는 것: 생성된 `libft.a`에 어떤 object member가 들어갔는지, public symbol이 정확한지, 허용하지 않은 external symbol이 남는지, 외부 directory에서 header/archive만으로 link되는지는 검사하지 않습니다.

### `79c0dcefb590` — `test(release): archive와 consumer 경계를 검증`

**Source 확정 역할:** archive members, normalized public symbol sets, allowed external dependencies, out-of-tree consumer를 검증해 `libft.a`를 binary/release boundary로 취급합니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

- archive member manifest와 실제 archive member 목록을 비교하는 경로를 찾습니다.
- 공개 API/global symbol manifest와 symbol inspection 결과를 normalize/compare하는 경로를 찾습니다.
- allowed undefined external symbol set이 어디에 정의되고 platform-aware하게 적용되는지 확인합니다.
- Darwin/Linux 차이를 처리하는 symbol-inspection script의 분기와 normalization을 확인합니다.
- source tree 밖 temporary directory에서 consumer를 compile/link/run하는 절차를 찾습니다.
- consumer가 문서화된 header와 archive만으로 build되는지 include/library path를 실제 command로 확인합니다.
- Make target이 위 검증들을 어떤 순서로 실행하고 어느 failure에서 중단되는지 확인합니다.

#### Release boundary 기록

- archive member contract: `tests/archive-members.txt`의 17개 object 이름을 정렬한 값과 `ar t "$archive" | awk '/\.o$/' | sort` 결과를 `cmp`합니다. member 누락·추가·이름 변경은 실패입니다.
- exported API contract: `tests/api-symbols.txt`의 43개 `ft_` symbol과 `nm`으로 얻은 global defined identifiers를 정렬해 정확히 비교합니다. expected보다 적거나 많은 global definition 모두 실패합니다.
- allowed external dependency contract: undefined symbol 전체에서 library가 자체 정의하는 expected API symbols를 빼고, `tests/allowed-undefined.txt`의 `free`, `malloc`, `write`와 platform errno accessor만 남아야 합니다. Darwin은 `__error`, Linux는 `__errno_location`을 script가 명시적으로 추가합니다.
- out-of-tree linkage contract: `tests/smoke/consumer.c`를 `mktemp`로 만든 repository 외부 directory에 복사하고 그 directory에서 compile/link/run합니다. command는 project root header를 `-I"$project_root"`로, archive를 absolute path로 넘깁니다.
- platform-specific normalization: Darwin은 `nm -gU -j`와 `nm -u -j`를 사용하고 leading underscore를 `sed`로 제거합니다. Linux는 `nm -g --defined-only -j`와 `nm -u -j` 결과를 그대로 사용합니다. 그 밖의 OS는 `unsupported symbol tool platform`으로 실패합니다.
- 검증 실패가 의미하는 artifact defect: source list와 archive 불일치, 외부에 공개된 API drift, forbidden runtime dependency, platform normalization 실패, public header/archive만으로 external consumer를 build하거나 실행하지 못하는 문제 중 하나입니다.

#### Test commit 학습

- production/release 항상 유지해야 하는 조건 대상: intended translation units와 public globals만 가진 archive, 허용 external dependencies, repository 밖 current directory에서도 header와 archive를 명시해 compile/link/run할 수 있는 consumer boundary입니다.
- failure 또는 boundary:
  - missing/extra archive member: `members.expected`와 `members.actual`의 `cmp`가 nonzero입니다.
  - missing/extra global symbol: normalized identifier set과 API manifest의 `cmp`가 nonzero입니다.
  - unexpected undefined dependency: internal expected symbols을 제거한 뒤 `undefined.external`과 allowed/platform set이 다릅니다.
  - hidden in-tree include/path dependency: temporary consumer directory에서 compile 또는 link가 실패하거나 executable이 nonzero로 종료됩니다. 다만 header는 project root의 explicit `-I`를 사용하므로 system-installed layout까지 증명하지는 않습니다.
- test technique:
  - manifests: archive members, API symbols, allowed undefined symbols를 version-controlled text file로 고정합니다.
  - symbol inspection: `ar`, `nm`, `awk`, `sort`, `cmp`, `comm`을 사용해 archive structure와 normalized symbol sets를 exact 비교합니다.
  - external smoke consumer: `ft_strdup`, `ft_strlen`, `ft_lstnew`를 사용하고 allocation을 해제하는 small consumer를 temporary external cwd에서 strict C99/no-builtin flags로 build·run합니다.
- 이 검증이 증명하는 것: manifest와 일치하는 archive members/global symbols, 허용 set과 일치하는 external dependencies, 선택된 API를 사용한 external compile/link/runtime smoke가 script상 모두 성공해야 target이 성공합니다.
- 이 검증이 증명하지 않는 것: 모든 API의 functional correctness, ABI compatibility를 여러 compiler/version에 걸쳐 장기 보장하는 것, installed include/library layout, Darwin/Linux 외 platform, archive member 내부의 local symbols, consumer의 복잡한 usage는 증명하지 않습니다.
- 테스트 성격:
  - [x] broad integration
  - [x] deterministic regression
  - [x] release artifact contract test
  - 선택 근거: build product, symbol tools, manifests, external compile/link/run을 묶는 release boundary integration이며 exact text/set 비교로 deterministic regression입니다.
- 후속 변경에서 막아야 할 회귀: source를 archive에 누락하거나 stale object를 추가하는 문제, helper를 global로 노출하는 문제, 공개 API 누락, 새 forbidden libc dependency, project cwd에만 의존하는 consumer build를 막습니다.
- 실행 근거: 현재 환경에서는 `make check-archive`를 실행하지 않았습니다. 이 절은 `79c0dcefb590`의 Makefile, manifests, `tests/check_archive.sh`, smoke consumer를 검사한 결과이며 성공 output을 기록하지 않습니다.

### `f5de4306ebcd` — `test(sanitize): undefined behavior 검사를 추가`

**Source 확정 역할:** UBSan-specific objects와 execution target을 추가합니다.

#### Test commit 학습

- sanitizer용 object/build flags가 ordinary build와 어떻게 분리되는지 찾습니다. 모든 production `SRC`를 `build/ubsan` 아래 별도 object로 compile하며 `UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer`를 공통 strict flags에 추가합니다. ordinary `build/obj`와 섞지 않습니다.
- 어떤 test suite 또는 executable이 UBSan build에서 실행되는지 확인합니다. ordinary `TEST_SRC := $(wildcard tests/test_*.c)`와 instrumented production objects를 `tests/bin/test_ubsan`으로 link합니다. allocation/write failure-injection binaries는 이 target에 포함되지 않습니다.
- undefined-behavior sanitizer가 실패를 report했을 때 target이 어떻게 실패하는지 확인합니다. `UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 ./$(UBSAN_BIN)`으로 첫 report에서 process를 중단하고 nonzero status가 Make target에 전달됩니다.
- production path 중 실제로 통과하는 범위를 기록합니다. ordinary functional suite가 호출하는 memory, string, conversion, list, fd-output paths를 instrumented objects에서 통과합니다. forced allocator/write failures는 통과하지 않습니다.
- 이 테스트가 증명하는 것: 해당 functional inputs에서 UBSan이 감지하는 정의되지 않은 동작 report 없이 suite가 끝나야 `ubsan` target이 성공합니다.
- 이 테스트가 증명하지 않는 것: 실행되지 않은 branches, 메모리 누수, 해제 후 사용 등 ASan/host leak 범주, 모든 compiler sanitizer behavior, failure-injection paths는 증명하지 않습니다.
- 테스트 성격과 후속 회귀 방지 범위: instrumented dynamic regression입니다. signed overflow, invalid shift, misalignment 등 UBSan이 해당 실행에서 관찰하는 오류의 재도입을 막지만 완전한 정적 증명은 아닙니다.
- 실행 근거: target 정의만 검사했으며 현재 환경에서 `make ubsan` 또는 `make sanitize`를 실행하지 않았습니다.

### `c625970fd211` — `test(sanitize): address sanitizer 검사를 추가`

**Source 확정 역할:** ASan-specific builds와 runtime checks를 추가하며, source는 이것이 leak testing을 대체하지 않는다고 명시합니다.

#### Test commit 학습

- ASan용 compile/link flags와 object separation을 찾습니다. 모든 production `SRC`를 `build/asan` 아래 별도 object로 만들고 `-fsanitize=address -fno-omit-frame-pointer`를 compile/link에 적용해 `tests/bin/test_asan`을 만듭니다.
- 어떤 functional/failure paths가 ASan instrumented binary에서 실행되는지 확인합니다. UBSan과 마찬가지로 ordinary `TEST_SRC`만 link합니다. allocator/write failure test drivers는 ASan binary에 포함되지 않습니다.
- address error가 target failure로 연결되는 방법을 확인합니다. `ASAN_OPTIONS=$(ASAN_OPTIONS) ./$(ASAN_BIN)`을 실행하고 default가 `detect_leaks=0:halt_on_error=1`이므로 address error report에서 nonzero로 멈춥니다.
- host leak check와 책임 범위를 혼동하지 않도록 실제 설정 차이를 기록합니다. ASan의 leak detection을 명시적으로 0으로 두고, leak 검사는 다음 별도 host target에 맡깁니다.
- 이 테스트가 증명하는 것: 선택된 functional execution에서 ASan이 관찰하는 out-of-bounds, 해제 후 사용 등 address errors가 없어야 `asan` target이 성공합니다.
- 이 테스트가 증명하지 않는 것: leak-free 상태, 실행하지 않은 failure branches, callback 내부 모든 behavior, 전체 input space는 증명하지 않습니다.
- 테스트 성격과 후속 회귀 방지 범위: address-sanitized functional regression입니다. ordinary suite가 도달하는 memory access 회귀를 막습니다.
- 관찰된 orchestration 차이: 이 commit 뒤에도 `sanitize: ubsan`만 유지됩니다. `asan`은 독립 target으로 존재하지만 `sanitize`의 prerequisite가 아니며, 뒤의 최종 `check`가 `$(MAKE) sanitize`만 호출하므로 ASan은 자동 포함되지 않습니다. source의 ASan 추가 역할은 보존하되 실제 orchestration 범위를 이와 같이 제한해야 합니다.
- 실행 근거: target 정의만 검사했으며 현재 환경에서 `make asan`을 실행하지 않았습니다.

### `9f555c37a6d8` — `test(leak): host 누수 검사 경로를 추가`

**Source 확정 역할:** host에서 `leaks` 또는 Valgrind를 사용하는 leak-checking path를 추가합니다.

#### Test commit 학습

- platform/host에 따라 어떤 leak checker가 선택되는지 실제 target/script에서 확인합니다. shell이 먼저 `command -v leaks`를 확인해 있으면 `leaks --atExit -- ./$(TEST_BIN)`을 실행합니다. 없으면 `command -v valgrind`를 확인해 full leak check를 실행합니다. 둘 다 없으면 명시적으로 stderr와 exit 1을 반환합니다.
- 검사 대상 executable과 실행 범위를 기록합니다. ordinary `tests/bin/test_libft`만 검사합니다. failure-injection binaries와 sanitizer binaries는 leak target의 직접 대상이 아닙니다.
- leak checker의 exit/status가 build target 결과에 어떻게 반영되는지 확인합니다. Valgrind는 `--errors-for-leak-kinds=all --error-exitcode=1`을 사용하며, `leaks`의 종료 상태도 recipe status가 됩니다. 선택된 command가 실패하면 Make target이 실패합니다.
- ASan path와 별도로 유지되는 이유를 실제 target 구성과 source 역할을 기준으로 정리합니다. ASan default에서 `detect_leaks=0`이고 address errors와 host leak reporting은 다른 detector/옵션을 사용하므로 별도 `leak` target이 필요합니다.
- 이 테스트가 증명하는 것: host에 지원 checker가 있을 때 ordinary suite 실행 종료 시 checker가 보고하는 leak이 허용되지 않습니다. checker 부재를 성공으로 넘기지 않습니다.
- 이 테스트가 증명하지 않는 것: failure harness에서만 도달하는 allocations, 모든 input, checker별 동일한 탐지 범위, unsupported host에서의 leak-free 상태는 증명하지 않습니다.
- 테스트 성격과 후속 회귀 방지 범위: host-dependent dynamic leak regression입니다. ordinary tests가 도달하는 소유권 cleanup 회귀를 막습니다.
- 실행 근거: 현재 host에서 checker 존재 여부와 target 성공을 실행 확인하지 않았습니다.

### `e31a2e748685` — `test(build): Clang과 GCC 호환성을 검증`

**Source 확정 역할:** clean copied trees에서 complete release-oriented suite를 Clang과 GNU GCC 각각으로 실행합니다.

#### 해당 SHA에서 확인할 build/test flow

- source tree를 clean copy하는 경로와 복사 대상/제외 대상을 확인합니다.
- compiler 선택이 environment 또는 Make variable을 통해 어떻게 주입되는지 확인합니다.
- Clang run과 GCC run이 서로 독립된 clean tree를 사용하는지 확인합니다.
- 각 compiler에서 호출되는 "complete suite"가 실제로 어떤 target들을 포함하는지 추적합니다.
- builtin/extension/compiler-specific assumption이 한 compiler에서만 통과할 수 있는 지점을 떠올리되, 실제 defect 여부는 test result로만 기록합니다.
- failure가 상위 target에 어떻게 propagation되는지 확인합니다.

#### 학습 기록

- clean-copy 목적: existing object, dependency file, archive, test binary, repository metadata의 영향을 제거하고 같은 committed Makefile/header/src/tests만으로 compiler별 결과를 다시 만듭니다.
- compiler selection 경로: script는 `CLANG`/`GCC` environment 후보와 일반 command names를 조사하고 `--version` 첫 줄로 실제 Clang과 GNU GCC를 구분합니다. 각 suite는 `make ... CC="$compiler"`로 compiler를 주입합니다.
- Clang suite: `$scratch/clang`에 Makefile, `libft.h`, `src`, `tests`를 복사하고 `fclean` 후 `all test failure-test write-failure-test check-archive`를 실행합니다.
- GCC suite: `$scratch/gcc`라는 별도 directory에서 동일한 copy와 target sequence를 GNU GCC로 실행합니다.
- 공통 검증 범위: clean archive build, ordinary functional tests, allocator failure tests, scripted write tests, archive/consumer contract입니다. 실제 script는 sanitizer, host leak, no-op rebuild를 compiler matrix에 포함하지 않습니다. 따라서 source의 “complete release-oriented suite”는 이 script가 정의한 위 범위로 읽어야 하며 최종 `check` 전체와 동일하지 않습니다.
- compiler-specific failure가 드러나는 지점: strict C99/no-builtin compilation diagnostics, archive symbol format 처리, undefined symbol set, test runtime 중 어느 단계든 한 compiler tree에서 nonzero가 되면 드러납니다. 실제 defect가 있었다는 실행 결과는 현재 기록하지 않습니다.
- 이 commit이 release confidence에 추가하는 것: 같은 source/test/archive contract가 Clang과 GNU GCC 두 compiler family의 독립 clean tree에서 재현 가능해야 합니다. compiler를 찾지 못한 경우도 silent skip하지 않고 실패합니다.
- failure propagation: script는 `set -eu`이며 `make` command를 guard하지 않습니다. 한 compiler suite가 실패하면 script와 `check-compilers` target이 nonzero로 끝나고 다음 상위 release 단계로 진행하지 않습니다.
- 실행 근거: compiler availability와 실제 suite success를 현재 환경에서 실행 확인하지 않았습니다.

### `b90fd748255a` — `test(release): 전체 검증 절차를 연결`

**Source 확정 역할:** clean build, functional, failure, sanitizer, archive, compiler, leak, no-op rebuild 검증을 하나의 release procedure로 orchestration합니다.

#### Test / orchestration commit 학습

- top-level release verification target을 찾습니다.
- source가 열거한 각 하위 검증 target이 어떤 순서로 연결되는지 실제 dependency/recipe를 기록합니다.
- clean build가 어느 시점에 수행되는지 확인합니다.
- functional/failure/sanitizer/archive/compiler/leak/no-op rebuild 각 단계가 기존 target을 재사용하는지 확인합니다.
- 한 단계 실패 시 뒤 단계가 실행되는지 중단되는지 실제 Make/shell semantics로 확인합니다.
- no-op rebuild check가 무엇을 관찰하는지 해당 SHA에서 직접 확인합니다.
- orchestration 자체가 새 runtime 항상 유지해야 하는 조건을 만드는지, 기존 evidence를 묶는지 source role과 실제 코드를 구분합니다.

#### 검증 범위 기록

| 단계 | 하위 target/command | 증명 범위 | 증명하지 않는 범위 | 실패 propagation |
| --- | --- | --- | --- | --- |
| clean build | `git diff --check`, `$(MAKE) fclean`, `$(MAKE) all` | whitespace error가 없고 clean state에서 archive를 다시 만들 수 있음 | source correctness와 실행 중 동작 | 각 recipe line nonzero에서 `check` 중단 |
| functional | `$(MAKE) test` | ordinary test suite가 archive와 함께 성공 | forced allocation/write failures, sanitizer-only defects | submake status 전파 |
| failure | `$(MAKE) failure-test`, `$(MAKE) write-failure-test` | deterministic allocation rollback과 system-call progress branches | 실제 OS timing과 모든 allocator failures | 각 submake status 전파 |
| sanitizer | `$(MAKE) sanitize` | 실제 Makefile상 UBSan target만 실행 | ASan은 별도 `asan` target이며 이 orchestration에 포함되지 않음; leak도 별도 | UBSan binary/report status 전파 |
| archive | `$(MAKE) check-archive` | member/symbol/dependency/external consumer contract | 모든 API functional/ABI behavior | script/command status 전파 |
| compiler | `$(MAKE) check-compilers` | Clang/GCC clean copy에서 build, functional, failure, archive suite | sanitizer/leak/no-op의 compiler별 반복 | script와 nested make status 전파 |
| leak | `$(MAKE) leak` | available host checker로 ordinary test binary leak 검사 | failure binaries와 unsupported checker 없는 host의 성공 | checker 부재/검출/command failure 전파 |
| no-op rebuild | `$(MAKE) -q all` | 앞 단계 뒤 `all` target이 up-to-date여서 rebuild가 필요 없는지 관찰 | reproducible binary bytes, clean git tree 전체 | `make -q`의 1/2 status가 `check` 실패로 전파 |

- orchestration 순서: 위 표 앞에 `git diff --check`가 있고, clean build 이후 functional → allocator failure → write failure → `sanitize` → archive → compiler matrix → leak → no-op 순으로 recipe lines가 나열됩니다.
- failure semantics: 각 line은 별도 shell이지만 Make recipe에 `-` prefix나 error ignore가 없습니다. 어느 command든 nonzero면 target이 즉시 실패해 뒤 lines는 실행되지 않습니다.
- orchestration의 역할: 기존 target을 순서대로 호출할 뿐 production runtime 동작을 새로 구현하지 않습니다. release 판단에 필요한 evidence collection과 fail-fast order를 하나의 진입점으로 묶습니다.
- 관찰된 scaffold/implementation 차이 기록: source role은 sanitizer check를 연결한다고 고정하며 실제로 `sanitize`를 호출합니다. 그러나 해당 SHA의 dependency는 `sanitize: ubsan`이고 `asan`을 포함하지 않습니다. fixed role을 변경하지 않고, 학습자가 실제 실행 범위를 UBSan으로 제한해 이해해야 합니다.
- 실행 근거: 현재 환경에서는 Git checkout과 required host tools를 구성하지 못해 `make check`를 실행하지 않았습니다. 이 절은 Make recipe inspection이며 모든 단계 성공을 주장하지 않습니다.

## 항상 유지해야 하는 조건 ledger

| 단계 | Commit | Source에 연결된 항상 유지해야 하는 조건 / evidence | 실제 build/test 근거 |
| --- | --- | --- | --- |
| compiler honesty | `4df8b23505b8` | libc-style tests를 builtin substitution으로 무효화하지 않음 | `override CFLAGS`가 strict C99 warnings와 `-fno-builtin`을 ordinary/special compile commands에 전달합니다. |
| artifact contract | `79c0dcefb590` | intended members/symbols/dependencies/out-of-tree linkage | three manifests, `ar`/`nm` normalization, exact `cmp`, temporary consumer compile/link/run을 `check-archive`가 연결합니다. |
| UB detector | `f5de4306ebcd` | UBSan evidence 추가 | 별도 instrumented objects/binary와 halt-on-error execution target이 ordinary suite를 실행합니다. |
| address detector | `c625970fd211` | ASan evidence 추가 | 별도 ASan objects/binary와 halt-on-error target이 있고 leak detection은 0입니다. |
| leak detector | `9f555c37a6d8` | host leak evidence 추가 | `leaks` 우선, Valgrind fallback, checker 부재 explicit failure로 ordinary binary를 검사합니다. |
| compiler matrix | `e31a2e748685` | Clang/GCC clean-tree compatibility evidence | compiler를 version 문자열로 식별하고 별도 copied tree에서 `all test failure-test write-failure-test check-archive`를 실행합니다. |
| orchestration | `b90fd748255a` | established checks를 하나의 release procedure로 연결 | `check` recipe가 fail-fast 순서로 기존 targets와 마지막 `make -q all`을 호출합니다. 실제 `sanitize`는 UBSan만 포함합니다. |

## Failure → Fix → Test 연결

이 thread는 하나의 runtime fix chain보다 release evidence를 점층적으로 강화하는 구조입니다.

- 초기 위험: local compile 성공만으로 artifact boundary를 확정할 수 없음
- compiler substitution 위험 대응: `4df8b23505b8`
- archive/consumer contract 검증: `79c0dcefb590`
- 독립 defect detector 추가: `f5de4306ebcd`, `c625970fd211`, `9f555c37a6d8`
- compiler-family 재현: `e31a2e748685`
- 전체 procedure 연결: `b90fd748255a`
- 실제 각 단계에서 재현 가능한 failure: forbidden builtin policy 제거/compile warning, member·symbol·dependency manifest mismatch, external consumer compile/link/run failure, UBSan/ASan report, host leak report/checker 부재, compiler 한쪽의 build/test failure, stale or unnecessarily rebuilt target입니다.
- 각 failure가 막는 회귀: compiler가 project logic을 대체하는 검증 왜곡, malformed archive/API drift, 숨은 runtime/path dependency, UB/address/leak defect, compiler-family 종속성, 일부 release checks의 누락, dependency graph가 매번 rebuild하는 문제를 각각 막습니다. 단, 최종 `check`의 sanitizer 단계는 ASan 회귀를 자동으로 막지 않으므로 `make asan`을 별도로 실행해야 합니다.

## Verification responsibility 변화

- compiler flags가 책임지는 것: dialect, diagnostics, warning-as-error, builtin substitution 금지를 모든 relevant compile에 적용합니다.
- archive inspection이 책임지는 것: object membership, global 외부에 공개된 API, undefined external dependency allowlist를 binary artifact에서 직접 비교합니다.
- external consumer가 책임지는 것: repository 밖 cwd에서 public header와 archive path만으로 선택 API를 compile/link/run할 수 있는지 검사합니다.
- UBSan이 책임지는 것: ordinary suite가 도달한 execution에서 undefined-behavior reports를 fail-fast로 검출합니다.
- ASan이 책임지는 것: ordinary suite가 도달한 address errors를 별도 instrumented target에서 검출합니다. final `check`에는 자동 포함되지 않습니다.
- host leak checker가 책임지는 것: ordinary test process 종료 시 host tool이 관찰하는 leak을 검출하고 checker 부재를 실패로 처리합니다.
- compiler matrix가 책임지는 것: Clang과 GNU GCC 각각의 독립 clean copy에서 build, functional/failure, archive contract를 재현합니다.
- orchestration target이 책임지는 것: 설정된 하위 evidence를 fail-fast 순서로 호출하고 마지막에 no-op rebuild 상태를 확인합니다. 각 detector의 내부 범위를 확장하지는 않습니다.

## Thread 최종 상태

- 마지막 commit 시점에 이 thread가 보장하는 것:
  - 기록: Makefile과 scripts는 strict no-builtin build, archive manifests, external consumer, UBSan, 독립 ASan, host leak checker, Clang/GCC matrix, top-level fail-fast release procedure를 제공합니다. 각 target이 성공했을 때의 artifact/evidence 조건은 코드로 명시돼 있습니다.
- 이 thread만으로는 보장하지 않는 것:
  - 기록: 현재 환경에서 실제 target들이 성공했다는 runtime evidence, Darwin/Linux 외 symbol portability, 모든 API behavior, reproducible archive bytes, long-term ABI compatibility를 보장하지 않습니다. 최종 `check`는 ASan을 호출하지 않으며 compiler matrix도 sanitizer/leak을 반복하지 않습니다.
- source의 significance와 실제 코드 확인 결과가 연결되는 지점:
  - 기록: local compile을 넘어 compiler policy, binary artifact inspection, external consumer, 서로 다른 dynamic detector, compiler-family clean run을 단계적으로 도입하고 한 target에 연결한다는 source significance와 일치합니다. 실제 target dependency가 제한하는 detector 범위는 별도로 기록했습니다.

## 최종 architecture 또는 실행 순서 정리

해당 thread의 commit history를 근거로 최종 흐름을 직접 작성합니다.

- 시작 조건 / 입력: Git checkout, C toolchain, `ar`/`nm`/POSIX shell utilities, Clang과 GNU GCC, host leak checker 중 하나가 필요합니다. `check`는 current repository source와 Makefile targets를 입력으로 사용합니다.
- 핵심 분기 또는 각 부분이 맡는 일의 구분: compiler flags는 source compile honesty를, `check_archive.sh`는 artifact contract를, sanitizer/leak tools는 runtime defect classes를, `check_compilers.sh`는 compiler-family 재현을 각각 맡습니다.
- 상태 또는 소유권 변화: production 소유권 변화는 없습니다. build system은 ordinary, failure, UBSan, ASan object directories와 test binaries를 분리하고 temporary consumer/compiler directories는 trap으로 제거합니다.
- failure 처리: scripts는 `set -eu` 또는 unguarded commands와 exact comparisons를 사용하고 Make는 첫 nonzero recipe에서 중단합니다. unsupported platform, missing compiler/checker도 explicit failure입니다.
- verification 경로: `check`가 whitespace → clean build → functional → two failure suites → UBSan → archive → compiler matrix → host leak → no-op query를 순서대로 실행합니다. ASan은 별도 `asan` path입니다.
- 최종 설명: release artifact를 신뢰하려면 compile 성공 외에 archive contents, API/dependency surface, external linkage, runtime defect detectors, compiler portability, incremental build 상태가 각각 독립 evidence로 필요합니다. 이 Thread는 그 evidence를 scripts와 Make targets로 분리해 만들고 fail-fast release 진입점으로 조합합니다.

## 학습 완료 자가 점검

- [x] 모든 commit을 문서 순서대로 해당 SHA에서 확인했습니다.
- [x] 중요도와 tags를 source 그대로 유지했습니다.
- [x] 실제 코드 근거와 source 확정 설명을 구분했습니다.
- [x] 변경 전/후 비교가 필요한 commit은 이전 관련 SHA와 비교했습니다.
- [x] failure → fix → test 연결을 실제 코드와 test code로 확인했습니다.
- [x] final HEAD를 과거 commit 설명에 소급하지 않았습니다.
- [x] 이 thread의 최종 항상 유지해야 하는 조건과 execution flow를 코드 근거로 설명할 수 있습니다.
===== END FILE: 04-static-archive-release-verification.md =====

===== BEGIN FILE: README.md =====
# libft Development Thread 학습 골격

## 목적

이 문서 세트는 `commit-importance.md`에 정의된 Development Thread를 따라 실제 commit history와 해당 SHA의 코드를 직접 읽으면서 `libft`의 설계, 구현, 실패 처리, 수정, 검증 과정을 복원하기 위한 학습 골격입니다.

완성형 해설서가 아닙니다. source에 이미 확정된 thread 구조, commit metadata, 역할, 중요도와 연결 관계만 고정하고, 실제 구현 해석과 실행 결과는 학습자가 채웁니다.

## 권장 학습 순서

1. [`01-non-overlap-copy-and-overlap-safe-movement.md`](01-non-overlap-copy-and-overlap-safe-movement.md)
2. [`02-single-allocation-to-rollback-safe-ownership.md`](02-single-allocation-to-rollback-safe-ownership.md)
3. [`03-fd-output-partial-system-calls.md`](03-fd-output-partial-system-calls.md)
4. [`04-static-archive-release-verification.md`](04-static-archive-release-verification.md)

이 순서는 source의 Development Threads 순서를 그대로 따릅니다.

## Thread 문서 사용법

- 먼저 Commit map에서 thread의 commit 순서와 각 commit의 역할을 확인합니다.
- 각 commit은 반드시 해당 SHA로 checkout하거나 그 SHA의 tree를 직접 열어 확인합니다.
- 필요하면 문서가 지시하는 이전 관련 SHA와 비교합니다.
- source에 확정된 설명과 실제 코드에서 직접 확인한 사실을 구분해서 기록합니다.
- 구현 코드를 붙일 때는 전체 파일보다 판단에 필요한 최소 범위만 삽입하고, caller/callee, state mutation, 소유권 transfer, failure branch가 끊기지 않게 주변 문맥을 포함합니다.
- Thread 마지막에는 commit별 기록을 다시 연결하여 항상 유지해야 하는 조건, failure → fix → test, 책임 변화, 최종 execution flow를 학습자 자신의 설명으로 정리합니다.

## 해당 SHA 코드 확인 원칙

- final HEAD를 과거 commit 설명에 소급해서 사용하지 않습니다.
- 각 commit의 구현은 해당 SHA 시점의 코드로만 판단합니다.
- 변경 전 상태가 필요하면 해당 commit의 parent 또는 문서에 지정된 이전 관련 SHA를 확인합니다.
- 후속 fix/test는 후속 SHA에서 따로 확인하고, 이전 SHA의 구현에 소급 적용하지 않습니다.
- source에 없는 파일명, 함수 관계, 실패 처리를 추정해 확정 사실처럼 기록하지 않습니다.

## Importance별 학습 깊이

- **S**: 프로젝트 핵심 architecture/항상 유지해야 하는 조건으로 추적합니다. 문제, 직전 상태, 실패 가능성, 핵심 결정, 실제 핵심 코드, 소유권/lifecycle/상태 전이, 후속 fix/test까지 연결합니다.
- **A**: 주요 subsystem, boundary, 실패 처리, integration point를 중심으로 핵심 코드와 설계 판단까지 확인합니다.
- **B**: thread 흐름에서 맡는 구현 역할과 필요한 코드·상태 변화를 확인합니다. S/A와 같은 분량을 기계적으로 요구하지 않습니다.
- **C**: thread 이해에 필요한 맥락일 때만 사용합니다. 이 문서 세트의 source-defined Development Threads에는 C commit이 포함되어 있지 않습니다.

## 실제 코드 삽입 기준

- 함수 전체가 아니라 판단 근거가 되는 최소 코드 범위를 우선합니다.
- 조건문 하나만 떼어 의미가 사라지면 caller/callee 또는 초기화/cleanup 부분까지 함께 붙입니다.
- 소유권을 다룰 때는 획득, 이전, 실패 시 해제, 성공 시 반환 지점을 함께 보이게 합니다.
- system call을 다룰 때는 반환값 처리, progress 갱신, retry/stop 조건을 함께 보이게 합니다.
- test를 다룰 때는 failure 주입 지점, production 경로 진입 지점, assertion 또는 측정 지점을 함께 보이게 합니다.
- release 검증을 다룰 때는 build flag, archive/symbol/dependency 검사, external consumer 또는 compiler 실행을 실제로 연결하는 지점을 우선합니다.

## Test commit 학습 방법

각 test commit에서 다음을 구분하여 기록합니다.

- 대상으로 하는 production 항상 유지해야 하는 조건
- 재현하는 failure 또는 boundary
- 사용하는 test technique
- 실제로 통과하는 production 코드 경로
- 테스트가 증명하는 것
- 테스트가 증명하지 않는 것
- broad integration test인지 deterministic regression인지, 또는 그 외 성격인지
- 후속 변경에서 막아야 할 회귀

source에 test technique이 확정되어 있으면 그 사실은 고정하고, 실제 test code와 실행 결과는 해당 SHA에서 직접 확인합니다.

## 문서 완료 기준

모든 thread에서 다음 조건을 만족해야 완료로 봅니다.

- commit 순서를 따라 직전 상태 → 결정 → 구현 → failure/fix/test 연결을 설명할 수 있습니다.
- 중요한 항상 유지해야 하는 조건이 어느 commit에서 도입, 강화, 부족함 노출, 복구, 검증되었는지 실제 코드 근거와 함께 기록되어 있습니다.
- S/A commit은 핵심 코드와 실패 처리를 SHA 기준으로 직접 확인했습니다.
- test commit은 production 경로와 증명 범위를 분리해 기록했습니다.
- final HEAD를 과거 설명에 소급 사용한 기록이 없습니다.
- Thread 최종 상태와 architecture 또는 execution flow를 source 요약 복사가 아니라 학습자 자신의 코드 근거로 설명할 수 있습니다.
===== END FILE: README.md =====
