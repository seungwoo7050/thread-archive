# 전체 입력 누적에서 제한된 스트리밍 줄 파서까지

## 1. 개발 흐름 목표

전체 입력을 EOF까지 모으는 초기 reader가, 임의의 `read` 분할과 무관하게 한 번에 정확히 한 줄을 반환하고 남은 입력을 보존하는 bounded streaming parser로 발전하는 과정을 복원합니다. 이후 direct tail read와 구조적 성능 측정이 이 parser의 cursor·소유권 불변 조건을 약화시키지 않는지도 확인합니다.

### 원자료에서 확인된 프로젝트 항목

- **Project profile:** POSIX C buffered record reading과 static-library API 설계 중, incremental line framing과 dynamic buffer-window representation을 담당하는 Thread입니다.
- **Critical 불변 조건:** 한 번의 성공 결과는 정확히 한 logical record이며, newline이 있으면 포함하고 caller가 소유하는 독립 allocation이어야 합니다.
- **Critical 불변 조건:** EOF의 unterminated suffix는 한 번만 반환되고 이후 EOF는 안정적으로 유지되어야 하며, empty stream은 empty line을 만들어서는 안 됩니다.
- **Critical 불변 조건:** 최종 architecture의 buffer state는 `0 <= begin <= scan <= end < capacity`와 `bytes[end]`의 NUL sentinel을 유지하며 capacity arithmetic은 wrap되지 않아야 합니다.
- **Major engineering difficulty:** kernel이 입력을 어떤 크기로 나누어 반환하더라도 observable record가 달라지지 않는 parser를 설계하는 문제입니다.
- **Major engineering difficulty:** append-only accumulator를 unread-window로 바꾸면서 allocation rollback과 caller 소유권을 유지하는 문제입니다.
- **Major engineering difficulty:** wall-clock 대신 operation count로 반복 scan, linear growth, per-chunk copy 회귀를 재현 가능하게 검출하는 문제입니다.

### 원자료에서 확인된 중요성

이 개발 흐름은 세 가지 결정을 구분합니다. 첫째, bytes를 실패 시 손상 없이 누적하는 방법입니다. 둘째, logical record를 표현하고 소비하는 방법입니다. 셋째, parser 비용이 repeated append copy, repeated scan, linear-capacity growth로 되돌아가지 않도록 검증하는 방법입니다. 초기 구현은 소유권과 growth의 기반을 제공하고, unread interval과 line extractor가 durable architecture를 확립합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- EOF까지 누적하는 상태와 한 줄씩 소비하는 상태는 어떤 필드 차이로 표현됩니까?
- `begin`, `scan`, `end`, `capacity`는 각각 어떤 byte 영역을 뜻하며 어느 함수가 변경하는가?
- newline이 한 read 안, read 경계, 여러 line이 섞인 read에 있을 때 같은 결과가 나오는 이유는 무엇입니까?
- result allocation, reserve, compaction, growth가 실패할 때 기존 unread bytes는 어디에 남는가?
- 최종 unterminated suffix와 empty stream은 어떤 조건으로 구분됩니까?
- direct tail read가 기존 append-copy 경로를 어떻게 대체하면서 rollback 순서를 유지하는가?
- 4 MiB metric test의 숫자는 어떤 알고리즘적 선택을 고정하며 무엇을 고정하지 않는가?

## 3. 완료 기준

- 초기 accumulator의 allocation·append·EOF return 경로를 실제 코드로 설명할 수 있습니다.
- unread interval의 각 index와 NUL sentinel이 모든 reserve branch에서 어떻게 유지되는지 입증했습니다.
- newline extraction과 EOF-tail extraction의 cursor mutation 순서를 해당 SHA 코드로 추적했습니다.
- 여러 `BUFFER_SIZE`에서 같은 logical line sequence가 나오는 이유를 테스트 항목과 production path로 연결했습니다.
- scratch-buffer 제거 전후의 read destination과 copy 횟수 차이를 비교했습니다.
- 4 MiB test의 고정 수치가 어떤 regression을 검출하는지, wall-clock benchmark와 어떻게 다른지 설명할 수 있습니다.

## 4. 커밋 목록
| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 Thread 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `85e4c2a41a4c` | `feat(reader): 파일 끝의 마지막 줄 반환` | **A** | `CORE`, `LINE_STATE`, `RISK` | 기하급수적 누적, ownership rollback, EOF tail 규칙을 확립합니다. |
| 2 | `7e64d3d79ad4` | `refactor(buffer): 읽지 않은 입력을 구간으로 표현` | **S** | `ARCH`, `LINE_STATE`, `HARD` | 소비한 prefix를 안전하게 제외하기 위한 unread-window 표현을 도입합니다. |
| 3 | `39a2b9055728` | `feat(reader): 줄을 분리하고 남은 입력 보존` | **S** | `CORE`, `LINE_STATE`, `HARD` | 한 줄 추출, newline 보존, scan 진행, unread suffix 보존을 구현합니다. |
| 4 | `656528529ade` | `test(reader): BUFFER_SIZE 경계값 검증` | **A** | `TEST`, `LINE_STATE`, `EDGE` | 여러 chunk 크기와 어려운 경계에서도 framing과 ownership이 유지됨을 검증합니다. |
| 5 | `dbf1abd21121` | `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용` | **A** | `PERF`, `LINE_STATE`, `REFACTOR` | reserved tail로 직접 읽어 매 read마다 발생하던 scratch-buffer 복사를 제거합니다. |
| 6 | `a0654d9de446` | `test(perf): 4 MiB 입력의 작업량 기준 고정` | **A** | `PERF`, `TEST`, `LINE_STATE` | 대용량 입력의 재현 가능한 작업량을 고정해 multiplicative work 회귀를 막습니다. |

## 5. 커밋별 학습 기록
### 5.1 `85e4c2a41a4c` — `feat(reader): 파일 끝의 마지막 줄 반환`

- **Commit:** `85e4c2a41a4c`
- **Subject:** `feat(reader): 파일 끝의 마지막 줄 반환`
- **중요도:** **A**
- **태그:** `CORE`, `LINE_STATE`, `RISK`

#### 원자료에서 확인된 역할

이 commit은 여러 `read`에 걸친 persistent accumulation, geometric buffer growth, descriptor probing, allocation/I/O failure cleanup을 도입합니다. EOF가 오면 누적한 bytes를 caller가 독립적으로 소유하는 result로 복사하고 internal state를 정리합니다. trailing newline이 없는 마지막 byte sequence도 data이며 버리면 안 된다는 규칙을 확립합니다.

이 시점에는 embedded newline을 분리하지 않으며, 전체 파일을 하나의 final record처럼 처리합니다. 따라서 완성된 line reader가 아니라 이후 line extraction이 재사용할 소유권·growth·rollback 기반입니다.

#### 직전 상태와 비교할 지점

- 공개 계약이 도입된 직전 관련 SHA `466cfcbd3525`와 비교해, header contract가 실제 reader state와 어떤 함수로 연결되는지 확인합니다.
- commit parent와 diff해 최초로 생긴 allocation, reserve/growth, descriptor probe, EOF copy, cleanup 경로를 구분합니다.
- 아직 존재하지 않는 line split, unread suffix, persistent scan cursor를 이후 구현의 관점에서 소급해 설명하지 않습니다.

#### 해당 SHA에서 확인할 실제 코드

1. persistent accumulation state를 보관하는 객체 또는 file-scope storage와 각 필드의 초기값을 찾습니다.
2. descriptor를 실제 read 전에 검사하거나 probe하는 caller와 callee를 찾고, invalid/closed descriptor가 어느 cleanup branch로 이동하는지 추적합니다.
3. capacity가 부족할 때 새 allocation을 얻고 기존 allocation을 교체하는 순서를 확인합니다. 새 allocation 실패 전에 기존 pointer가 덮어써지지 않는지 기록합니다.
4. positive short read를 append하는 코드, EOF를 판정하는 코드, I/O error를 처리하는 코드를 분리해 발췌합니다.
5. EOF에서 accumulated bytes를 caller-owned NUL-terminated result로 복사하는 지점과 internal state release 순서를 확인합니다.
6. empty input과 unterminated nonempty input이 서로 다른 return 결과가 되는 조건을 찾습니다.

#### 코드 근거 기록

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| accumulation state 정의와 초기화 | `get_next_line.c`, `t_reader`와 `static t_reader g_reader = {-1, NULL, 0, 0}` | `fd`, `bytes`, `length`, `capacity`가 process 수명의 singleton state입니다. 첫 호출 전에는 descriptor가 없고 allocation도 없습니다. |
| geometric capacity 계산과 overflow 방어 | `reserve_bytes`: `capacity *= 2`; `capacity > (size_t)-1 / 2`이면 `required`; `append_bytes`: `length > SIZE_MAX - current - 1` 검사 | NUL 한 바이트를 포함한 `required` 계산이 wrap되기 전에 실패하며, capacity는 필요 크기 이상이 될 때까지 배가됩니다. |
| old allocation을 보존하는 replacement 순서 | `reserve_bytes`: `malloc(capacity)` → 기존 bytes 복사 → `free(g_reader.bytes)` → pointer/capacity 교체 | 새 allocation 실패 시 기존 pointer, length, capacity는 바뀌지 않습니다. 교체가 성공한 뒤에만 이전 allocation의 소유권을 놓습니다. |
| read progress / EOF / error 분기 | `get_next_line`: zero-length probe, `while (read_size > 0)`, `read_size < 0`, `length == 0` | 양수는 append되는 progress, 0은 EOF, 음수는 오류입니다. short positive read도 EOF로 취급하지 않고 다음 read를 계속합니다. |
| caller-visible result allocation·copy·NUL 종료 | `append_bytes`가 매 append 뒤 `bytes[length]='\0'`; `release_final_line`은 `line = g_reader.bytes` | 실제 SHA에는 별도 result allocation/copy가 없습니다. 내부 allocation 자체를 caller에게 이전하므로 결과는 독립 소유가 되지만 source의 “복사” 설명과 구현 방식은 다릅니다. |
| failure 및 EOF cleanup | `reset_reader`; EOF nonempty의 `release_final_line` | invalid fd, append 실패, read 오류, empty EOF는 allocation을 free합니다. nonempty EOF는 pointer를 넘긴 뒤 singleton의 pointer와 길이만 초기화해 caller가 free할 owner가 됩니다. |

**최소 코드 근거**

`85e4c2a41a4c`, `get_next_line.c`, `release_final_line`:

```c
line = g_reader.bytes;
g_reader.bytes = NULL;
g_reader.length = 0;
g_reader.capacity = 0;
return (line);
```

이 코드는 result copy가 아니라 소유권 transfer입니다. 고정된 Source 역할은 변경하지 않았으며, 저장소에서 관찰된 차이를 이 학습 기록에 명시합니다.

#### 설계 결정과 한계

- **해결하려던 문제:** `read` 한 번으로 전체 입력이 오지 않더라도 bytes를 누적하고, EOF 직전의 newline 없는 suffix를 data로 반환해야 했습니다. `append_bytes`가 여러 positive read를 이어 붙이고 `length != 0`일 때 `release_final_line`으로 반환합니다.
- **기존 설계가 충분하지 않았던 이유:** `466cfcbd3525`에는 header의 `get_next_line(int fd)` 선언과 `BUFFER_SIZE` 검증만 있고 persistent bytes, growth, failure rollback을 수행하는 구현이 없었습니다.
- **선택한 결정:** singleton accumulator와 geometric growth를 사용합니다. realloc을 쓰지 않고 새 allocation을 먼저 확보해 실패 시 기존 state를 보존합니다. EOF에서는 내부 allocation을 직접 넘겨 복사 비용 없이 caller 소유권으로 전환합니다.
- **이 commit이 보장하는 것:** valid descriptor의 empty stream은 `NULL`, nonempty stream은 전체 bytes 한 덩어리, invalid/closed fd와 I/O·allocation 오류는 `NULL`이며 해당 singleton allocation을 정리합니다. 여러 read에 걸친 입력과 short read도 누적합니다.
- **아직 보장하지 않는 것:** `find_line_end`나 `scan`이 없으므로 embedded newline을 분리하지 않습니다. `"a\nb\n"`도 한 번에 전체 문자열로 반환됩니다. 또한 singleton이므로 fd를 바꾸면 이전 fd의 state를 버립니다.
- **다음 commit과의 연결:** 하나의 `length`는 “전체 allocation 중 유효한 끝”만 나타냅니다. 반환한 prefix와 다음 호출에 남길 suffix를 동시에 표현하려면 소비 시작점인 `begin`이 필요하므로 `7e64d3d79ad4`가 `[begin, end)`를 도입합니다.

### 5.2 `7e64d3d79ad4` — `refactor(buffer): 읽지 않은 입력을 구간으로 표현`

- **Commit:** `7e64d3d79ad4`
- **Subject:** `refactor(buffer): 읽지 않은 입력을 구간으로 표현`
- **중요도:** **S**
- **태그:** `ARCH`, `LINE_STATE`, `HARD`

#### 원자료에서 확인된 Problem

하나의 active length만으로는 accumulated bytes, 이미 반환한 prefix, 아직 읽지 않은 bytes, scan progress, unused capacity를 구분하기 어렵습니다. Incremental line consumption은 이 영역들이 서로 다른 의미를 가져야 하며, allocation failure가 기존 data를 손상시키지 않아야 합니다.

#### 원자료에서 확인된 Decision

buffer를 capacity-managed allocation 내부의 unread interval `[begin, end)`로 표현합니다. 소비한 prefix는 `begin`을 전진시켜 제외하고, remaining bytes는 필요할 때만 compact하며, geometric growth는 allocation과 copy가 성공한 뒤에만 기존 allocation을 교체합니다.

#### 원자료에서 확인된 중요성

이 표현은 이후 newline scan, suffix preservation, explicit context, failure retry, direct tail read가 공유하는 기반입니다. 최종 reader는 repeated string concatenation이 아니라 bytes 위의 stateful window로 이해해야 합니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

1. reader state 정의에서 `begin`, `end`, `capacity` 또는 같은 의미의 필드가 어떻게 추가·변경되었는지 직전 관련 SHA `85e4c2a41a4c`와 비교합니다.
2. logical unread length를 계산하는 코드와 allocation 전체 capacity를 계산하는 코드를 구분합니다.
3. free tail space만으로 요청을 만족하는 branch, consumed prefix를 compact하는 branch, 새 allocation으로 grow하는 branch를 각각 찾습니다.
4. compaction 전후에 unread bytes의 source/destination range와 index reset 순서를 기록합니다.
5. growth size 계산에서 overflow를 막는 조건과 필요한 NUL sentinel 공간을 포함하는지 확인합니다.
6. 새 allocation 또는 copy 준비가 실패했을 때 old allocation, `begin`, `end`가 그대로 유지되는지 failure return 직전 상태를 확인합니다.
7. reserve 완료 후 `bytes[end]`에 NUL sentinel을 유지하는 지점과 모든 mutation path에서 sentinel이 유효한지 확인합니다.
8. 이 commit이 line extraction 자체를 도입하지 않고 representation만 바꾸는지 caller 흐름으로 확인합니다.

#### State representation 복원

| 영역 | index/field | 해당 SHA의 실제 의미 | 변경 주체 | 실패 시 유지 조건 |
| --- | --- | --- | --- | --- |
| 소비 완료 prefix | `[0, begin)` | 논리적으로 읽은 것으로 간주해 다음 결과에서 제외할 영역입니다. 이 SHA에는 extraction이 없어 정상 경로에서 아직 `begin`이 0에 머뭅니다. | 이후 extraction 또는 `compact_bytes`가 사용하도록 준비됨 | growth allocation 실패 전에는 `begin`을 변경하지 않습니다. |
| unread bytes | `[begin, end)` | caller에게 아직 전달하지 않은 유효 byte 구간이며 길이는 `unread_length() == end - begin`입니다. | `append_bytes`, 향후 extraction, `compact_bytes` | growth 실패 시 pointer와 두 index가 그대로입니다. compaction은 실패하지 않는 in-place mutation입니다. |
| append 가능한 tail | `[end, capacity)` 중 sentinel 이후 공간 | `capacity - end >= appended + 1`이면 allocation·compaction 없이 새 bytes와 NUL을 둘 수 있습니다. | `reserve_bytes`, `append_bytes` | tail branch는 state를 바꾸지 않고 성공만 반환합니다. |
| allocation 전체 크기 | `capacity` | `bytes`가 가리키는 block의 크기입니다. logical unread length와 별개입니다. | `reserve_bytes` growth success, `reset_reader` | 새 allocation 실패 시 기존 capacity 유지 |
| NUL sentinel | `bytes[end]` | unread interval의 물리적 끝 바로 뒤에 둔 검사·문자열 편의용 byte입니다. record 내용에는 포함되지 않습니다. | `compact_bytes`, growth commit, `append_bytes` | 성공한 mutation마다 다시 기록하며 실패 path는 기존 sentinel을 건드리지 않습니다. |

#### Reserve branch별 코드 근거

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| tail 재사용 branch | `reserve_bytes`: `capacity - end >= appended + 1` | 현재 `end` 뒤에 data와 sentinel을 둘 수 있으면 pointer와 index를 전혀 바꾸지 않습니다. |
| unread suffix compaction branch | `begin > 0 && required <= capacity` → `compact_bytes()` | allocation 전체에는 충분하지만 앞쪽 consumed 공간 때문에 연속 tail이 부족한 경우 `[begin,end)`를 0으로 당깁니다. |
| geometric growth branch | capacity 1부터 doubling, `malloc(capacity)`, unread copy, old free, index commit | 필요한 크기가 allocation 전체보다 크면 새 block을 먼저 얻습니다. 성공 후 unread bytes만 복사하고 `begin=0`, `end=length`로 정규화합니다. |
| capacity arithmetic overflow branch | `appended > SIZE_MAX - length - 1`; doubling saturation 검사 | `length + appended + 1`과 doubling이 wrap되기 전에 0을 반환합니다. sentinel 공간까지 계산합니다. |
| allocation failure rollback branch | `allocation = malloc(capacity); if (allocation == NULL) return (0);`가 old free와 index commit보다 앞 | 실패 시 old allocation, `begin`, `end`, `capacity`, sentinel이 모두 그대로입니다. |
| NUL sentinel 복구 지점 | `compact_bytes`, growth success, `append_bytes`의 `bytes[end] = '\0'` | 모든 실제 byte/index mutation이 끝난 뒤 새 logical end에 sentinel을 씁니다. |

**최소 코드 근거**

`7e64d3d79ad4`, `get_next_line.c`, `reserve_bytes`:

```c
allocation = malloc(capacity);
if (allocation == NULL)
    return (0);
if (length > 0)
    copy_bytes(allocation, g_reader.bytes + g_reader.begin, length);
free(g_reader.bytes);
```

실패 가능한 획득을 먼저 수행하므로 growth branch는 non-destructive입니다.

#### 소유권·failure 분석

- 기존 allocation은 함수 진입 시 `g_reader`가 소유합니다. 새 block을 받은 직후부터 교체 전까지는 지역 변수 `allocation`이 임시 owner이며, unread copy가 끝난 뒤 기존 block을 free하고 `g_reader.bytes = allocation`으로 소유권을 넘깁니다.
- compaction은 allocation 소유권을 바꾸지 않습니다. `copy_bytes`가 작은 주소 방향으로 왼쪽 이동하며 index를 0부터 증가시키므로 source와 destination이 겹쳐도 아직 읽지 않은 source byte를 덮지 않습니다. 일반적인 양방향 overlap을 지원하는 함수는 아니지만 이 호출 방향에서는 안전합니다.
- growth failure는 `malloc` 직후 반환하므로 state mutation이 없습니다. overflow도 allocation·index mutation 전에 반환합니다. 반면 compaction은 성공을 전제로 실제 위치를 바꾸며 별도 failure branch가 없습니다.
- source가 최종 불변 조건으로 제시한 `0 <= begin <= scan <= end < capacity` 중 이 SHA에는 `scan`이 없습니다. 실제 불변 조건은 allocation이 있을 때 `0 <= begin <= end < capacity`, 그리고 `bytes[end] == '\0'`입니다.

#### 이 commit의 보장과 다음 연결

- **보장:** logical unread bytes와 allocation capacity를 분리하고, reserve가 tail reuse, left compaction, geometric growth를 선택할 수 있습니다. growth failure는 old unread interval을 보존합니다.
- **보장하지 않는 것:** `get_next_line`은 여전히 EOF까지 읽고 `release_final_line` 한 번만 반환합니다. `scan`과 one-line extraction은 없습니다.
- **다음 연결:** `39a2b9055728`은 `[begin,end)` 안에서 `scan`으로 newline을 찾고 `[begin,line_end)`만 결과로 복사한 뒤 `begin=line_end`를 commit하여 suffix를 남깁니다.

### 5.3 `39a2b9055728` — `feat(reader): 줄을 분리하고 남은 입력 보존`

- **Commit:** `39a2b9055728`
- **Subject:** `feat(reader): 줄을 분리하고 남은 입력 보존`
- **중요도:** **S**
- **태그:** `CORE`, `LINE_STATE`, `HARD`

#### 원자료에서 확인된 Problem

EOF까지 누적하는 구현은 library의 중심 계약을 만족하지 못합니다. caller는 delimiter가 read 경계를 가로지르거나 하나의 read에 여러 line이 들어 있어도 한 call에서 정확히 한 logical line을 받아야 합니다.

#### 원자료에서 확인된 Decision

reader는 persistent scan cursor에서 newline을 찾고, newline까지의 prefix를 독립 allocation으로 반환하며, 해당 record만큼 unread start를 전진시킵니다. 뒤의 모든 bytes는 다음 call을 위해 보존합니다. EOF에서는 nonempty suffix를 한 번 반환한 뒤 completion으로 이동합니다.

#### 원자료에서 확인된 중요성

이 commit이 whole-stream accumulator를 reusable streaming line reader로 바꿉니다. kernel read partitioning을 caller에게 보이지 않게 만들고, 한 성공 call이 정확히 한 record만 소비한다는 핵심 consumption 불변 조건을 확립합니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

1. `scan` cursor 또는 같은 역할의 필드가 state에 추가된 diff와 초기값을 찾습니다.
2. newline search helper가 `[scan, end)` 중 어디를 검사하고, delimiter를 못 찾았을 때 `scan`을 어디까지 전진시키는지 추적합니다.
3. newline을 찾은 경우 result length에 newline byte가 포함되는 계산을 확인합니다.
4. caller-owned result allocation, bytes copy, NUL termination의 순서를 발췌합니다.
5. 성공한 extraction 뒤 `begin`과 `scan`이 각각 어떤 값으로 이동하며 unread suffix가 어떻게 보존되는지 확인합니다.
6. 연속 newline이 각각 별도 empty line result가 되는 조건을 실제 index 계산으로 설명합니다.
7. EOF이며 unread suffix가 nonempty인 branch와 EOF이며 unread bytes가 없는 branch를 구분합니다.
8. 한 read가 첫 line의 newline 뒤에 다음 line 일부까지 가져온 경우, 두 번째 prefix가 내부 buffer에 남는 경로를 추적합니다.

#### 핵심 execution trace

아래 세 입력을 해당 SHA의 실제 index 값으로 추적합니다.

| 사례 | 각 단계의 `begin` | 각 단계의 `scan` | 각 단계의 `end` | 반환 result | 남은 unread bytes |
| --- | --- | --- | --- | --- | --- |
| newline이 read 경계 앞에 있음 | 예: 첫 read `"ab"`: `0`; 다음 read `"\ncd"`: `0`; extraction 후 `3` | 첫 scan 후 `2`; newline 발견 시 `3`; extraction 후 `3` | 첫 read `2`; 다음 read `5`; extraction 후 `5` | `"ab\n"` | `[3,5)`의 `"cd"` |
| 한 read에 두 line이 들어옴 | `"a\nb\n"` read 후 `0`; 첫 extraction 후 `2`; 둘째 extraction 후 state 폐기 | 첫 newline에서 `2`; commit `2`; 다음 newline에서 `4` | read 후와 두 extraction 전까지 `4` | 첫 call `"a\n"`, 다음 call `"b\n"` | 첫 call 뒤 `[2,4)`의 `"b\n"`; 둘째 뒤 없음 |
| newline 없는 EOF tail | data read 후 `0`; EOF transfer 후 singleton 초기화 | data scan 후 `4`; 초기화 후 0 | data read 후 `4`; 초기화 후 0 | `"tail"` | 없음; 다음 call은 empty EOF로 `NULL` |

연속 newline `"\n\n"`에서는 첫 `find_line_end`가 1을 반환해 길이 1인 `"\n"`을 만들고 `begin=scan=1`로 이동합니다. 다음 호출은 index 1의 newline을 찾아 다시 길이 1인 `"\n"`을 만듭니다. 빈 logical line은 NUL-only 문자열이 아니라 newline을 포함한 한 바이트 result입니다.

#### 코드 근거 기록

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| newline scan 시작/종료 조건 | `find_line_end`: `while (scan < end)`, byte가 `\n`이면 `scan++` 후 반환, 없으면 `scan=end` | 이미 확인한 prefix를 재검색하지 않으며 반환 index는 delimiter 다음 위치입니다. |
| delimiter 포함 result length 계산 | `extract_line`: `length = line_end - begin` | `line_end`가 newline 다음 index이므로 newline 자체가 result에 포함됩니다. |
| result allocation·copy·NUL termination | `malloc(length + 1)` → `copy_bytes` → `line[length]='\0'` | 각 성공 result는 internal buffer와 별도 allocation이며 caller가 free해야 합니다. |
| 성공 후 `begin` commit | `reader->begin = line_end` | 방금 반환한 record만 consumed prefix가 되고 그 뒤 bytes는 남습니다. |
| 성공 후 `scan` 재설정 또는 보정 | `reader->scan = reader->begin` | 다음 record의 시작에서 다시 탐색합니다. compaction 시에는 기존 scan에서 old begin을 빼 상대 위치를 유지합니다. |
| EOF tail 반환과 이후 completion | `unread_length()!=0`이면 `release_final_line`; 없으면 `reset_reader`/`NULL` | nonempty suffix는 한 번 반환되고 singleton state가 비워집니다. 이후 EOF call은 `NULL`입니다. |

**최소 코드 근거**

`39a2b9055728`, `get_next_line.c`, `find_line_end`와 `extract_line`:

```c
if (reader->bytes[reader->scan] == '\n')
{
    reader->scan++;
    return (reader->scan);
}
```

```c
reader->begin = line_end;
reader->scan = reader->begin;
```

첫 부분이 delimiter를 result 범위에 포함하고, 둘째 부분이 성공한 record만 소비합니다.

#### Failure와 아직 남은 범위

- result allocation이 실패하면 이 SHA의 `extract_line`은 `discard_reader(reader)`를 호출합니다. 따라서 `begin`과 `scan`을 보존해 재시도하는 것이 아니라 해당 descriptor의 전체 state를 해제합니다. 이후 `9bd6ebf429e2`의 transactional extraction을 이 SHA에 소급할 수 없습니다.
- read error도 `discard_reader`를 호출해 unread bytes를 버립니다. EOF는 오류와 달리 nonempty unread suffix를 반환합니다.
- 이 SHA는 여전히 하나의 file-scope singleton만 사용합니다. 다른 fd가 들어오면 기존 state를 reset하므로 descriptor별 isolation이나 explicit result taxonomy는 없습니다.

#### 이 commit이 보장하는 것

- 성공한 `extract_line`이 소비하는 범위는 정확히 `[begin,line_end)`이며 newline이 발견된 경우 newline까지 포함합니다.
- `[line_end,end)`는 다음 호출용 unread suffix로 남고, EOF의 `[begin,end)` nonempty tail은 마지막 result가 됩니다.
- kernel chunk boundary가 logical record boundary와 분리되는 이유는 매 read 뒤 기존 `scan`부터 이어서 검사하고, newline이 없으면 더 읽되 이미 받은 bytes와 새 bytes를 같은 interval에 유지하기 때문입니다.

### 5.4 `656528529ade` — `test(reader): BUFFER_SIZE 경계값 검증`

- **Commit:** `656528529ade`
- **Subject:** `test(reader): BUFFER_SIZE 경계값 검증`
- **중요도:** **A**
- **태그:** `TEST`, `LINE_STATE`, `EDGE`

#### 원자료에서 확인된 역할

동일한 reader 동작을 `BUFFER_SIZE` 1, 2, default, much larger value로 실행합니다. chunk boundary 주변 data, 큰 adjacent lines, pipe input, high-numbered descriptors, repeated EOF, 반환 buffer의 독립성, descriptor를 닫지 않는 borrowing 동작을 검증합니다.

#### 해당 SHA에서 확인할 테스트 코드

1. 각 `BUFFER_SIZE` 변형을 어떤 build target, compile definition, test loop로 실행하는지 찾습니다.
2. delimiter가 chunk 앞·경계·뒤에 놓이는 fixture와 expected line sequence를 확인합니다.
3. large adjacent lines가 growth, scan, suffix retention을 동시에 통과하도록 구성된 입력을 찾습니다.
4. pipe input과 high-numbered descriptor를 생성·정리하는 helper를 확인합니다.
5. repeated EOF call의 expected value와 몇 번 반복하는지 기록합니다.
6. 반환된 line을 보관한 뒤 reader를 계속 사용해 internal buffer와 alias하지 않음을 확인하는 assertion을 찾습니다.
7. 호출 뒤 descriptor가 여전히 열려 있음을 어떤 system call 또는 assertion으로 확인하는지 찾습니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | `tests/test_boundaries.c`의 `check_single_line`, large adjacent line, storage independence case가 한 call당 한 line, suffix 보존, caller-owned result를 각각 확인합니다. |
| **Failure / boundary** | body 길이를 `BUFFER_SIZE-1`, `BUFFER_SIZE`, `BUFFER_SIZE+1`, `3*BUFFER_SIZE+7`로 배치하고, newline 유무·연속 line·empty input·pipe·high fd를 구분합니다. |
| **Test technique** | Makefile의 `MATRIX_SIZES := 1 2 42 1024`와 size별 object/bin 경로를 사용하는 compile-time behavioral matrix입니다. 각 size는 별도 `build/obj/<size>`와 `tests/bin/test_reader_<size>`를 사용합니다. |
| **Production path** | fixture 생성 → file/pipe/fd 준비 → `get_next_line` → selected descriptor state의 scan/read/extract → result 비교/free → descriptor close 순입니다. |
| **증명하는 것** | 같은 fixture가 네 chunk 크기에서 같은 line sequence를 내고, repeated EOF는 `NULL`, first result는 subsequent read 뒤에도 내용이 유지되며 두 result pointer가 다름을 assertion으로 고정합니다. |
| **증명하지 않는 것** | allocation/read fault injection, `EINTR`/`EAGAIN`, explicit context enum, 구조적 copy·allocation count는 이 commit의 범위가 아닙니다. |
| **분류** | 여러 compile-time chunk 크기와 실제 descriptor 형태를 포괄하는 broad boundary regression입니다. |
| **막는 회귀** | delimiter alignment 의존은 size matrix, suffix 손실은 adjacent lines, alias는 retained first line/pointer inequality, unstable EOF는 반복 `NULL`, fd를 닫는 동작은 test가 명시적으로 descriptor를 계속 사용·정리하는 흐름이 검출합니다. |

**실제 fixture 근거**

- `tests/test_boundaries.c`의 large case는 첫 body 32,768 bytes 뒤 newline과 이어지는 32,771-byte tail을 사용해 grow, first extraction, suffix retention, EOF tail을 한 sequence로 통과시킵니다.
- high-numbered descriptor는 `fcntl(..., F_DUPFD, 128)`로 생성합니다.
- empty input은 `get_next_line`의 `NULL`을 세 번 확인하고, single-line helper도 line 반환 뒤 EOF `NULL`을 두 번 확인합니다.
- descriptor borrowing은 일반 case에서 library가 `close`를 호출하지 않고 test가 마지막에 직접 닫는 방식과, high-fd/pipe를 후속 operation에 사용하는 동작으로 간접 검증됩니다. 이 SHA에 context API의 명시적 `F_GETFD` assertion은 없습니다.

#### 결과 기록

| `BUFFER_SIZE` | 실행 명령 | 통과/실패 | 실패 시 최초 assertion | 확인한 production path |
| ---: | --- | --- | --- | --- |
| 1 | `make --no-print-directory test-run BUFFER_SIZE=1` | **미실행** — 이 환경에서는 branch checkout을 만들 수 없어 binary를 빌드·실행하지 못했습니다. | 실행 결과 없음 | Makefile과 test source inspection으로 size별 object/archive, scan/extract path를 확인했습니다. |
| 2 | `make --no-print-directory test-run BUFFER_SIZE=2` | **미실행** — 같은 환경 제한 | 실행 결과 없음 | 동일 |
| default | `make --no-print-directory test-run BUFFER_SIZE=42` | **미실행** — 같은 환경 제한 | 실행 결과 없음 | 동일 |
| large | `make --no-print-directory test-run BUFFER_SIZE=1024` | **미실행** — 같은 환경 제한 | 실행 결과 없음 | 동일 |

실행 결과를 통과로 간주하지 않았습니다. 확인한 것은 `656528529ade`의 Makefile·test implementation과 production code뿐입니다.

### 5.5 `dbf1abd21121` — `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용`

- **Commit:** `dbf1abd21121`
- **Subject:** `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용`
- **중요도:** **A**
- **태그:** `PERF`, `LINE_STATE`, `REFACTOR`

#### 원자료에서 확인된 역할

기존 stack scratch buffer에 `read`한 뒤 internal buffer로 append-copy하던 경로를 제거합니다. reserve가 확보한 internal unread buffer의 tail을 system call destination으로 직접 사용하고, positive read만큼 `end`를 늘린 뒤 NUL sentinel을 복구합니다.

#### 변경 전후 비교

- **변경 전:** 직전 구현의 `get_next_line`은 `char buffer[BUFFER_SIZE]`를 destination으로 사용하고, `append_bytes(reader, buffer, read_size)`가 다시 `copy_bytes`로 persistent allocation에 복사했습니다. scratch storage는 한 public call의 stack 수명만 가집니다.
- **변경 후:** `reserve_bytes(reader, BUFFER_SIZE)` 뒤 `read(fd, reader->bytes + reader->end, BUFFER_SIZE)`를 호출합니다. 양수일 때만 `end += read_size` 후 `bytes[end]='\0'`을 기록합니다.
- 이 비교는 각각 해당 historical SHA의 `get_next_line.c` symbol을 사용했으며 후속 context API symbol을 끌어오지 않았습니다.

#### 해당 SHA에서 확인할 실제 코드

1. read caller가 필요한 tail capacity를 계산하고 reserve를 먼저 호출하는 순서를 확인합니다.
2. reserve 성공 전에는 `read`가 실행되지 않는지 control flow를 추적합니다.
3. system call destination이 internal allocation의 어느 offset인지 pointer expression을 발췌합니다.
4. short positive read에서 실제 반환 byte 수만큼 `end`가 증가하는지 확인합니다.
5. zero read와 negative read에서는 `end`가 증가하지 않는지 각 branch를 확인합니다.
6. 새 bytes 뒤 NUL sentinel을 복구하는 정확한 지점을 기록합니다.
7. compaction 또는 growth가 발생한 뒤 read destination이 stale pointer를 사용하지 않는지 caller/callee 순서로 확인합니다.

#### 코드 근거 기록

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| 변경 전 scratch-buffer read | 직전 `get_next_line.c`: `char buffer[BUFFER_SIZE]`; `read(fd, buffer, BUFFER_SIZE)` | kernel bytes가 먼저 stack에 저장됐습니다. |
| 변경 전 append copy | 직전 `append_bytes`: reserve 후 `copy_bytes(reader->bytes + reader->end, bytes, length)` | 모든 positive read마다 stack → persistent buffer 복사가 하나 추가됐습니다. |
| 변경 후 reserve-before-read | `while (1)` 첫 분기 `if (!reserve_bytes(...))` 다음에만 `read` | 연속 공간을 확보하지 못하면 system call을 수행하지 않고 해당 legacy node를 정리합니다. |
| 변경 후 direct tail destination | `reader->bytes + reader->end` | unread interval 끝의 예약된 tail이 곧 system-call destination입니다. |
| positive read 뒤 `end` mutation | `if (read_size <= 0) break; reader->end += (size_t)read_size` | 요청량이 아니라 실제 반환량만 state에 commit합니다. short read는 정상 progress입니다. |
| NUL sentinel 복구 | `reader->bytes[reader->end] = '\0'` | 새 `end`가 정해진 직후 sentinel을 복구하고 scan합니다. |
| reserve/read failure에서 unread interval 유지 | reserve growth 실패 자체는 non-destructive이나 caller는 `discard_reader`; read error도 `discard_reader` | reserve/read helper 내부의 interval은 commit 전 안전하지만 이 시점 legacy API 정책은 실패 시 selected node를 폐기하므로 caller 재시도용 state는 보존하지 않습니다. |

**최소 코드 근거**

`dbf1abd21121`, `get_next_line.c`, read loop:

```c
if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
{
    discard_reader(reader);
    return (NULL);
}
read_size = read(fd, reader->bytes + reader->end,
        (size_t)BUFFER_SIZE);
```

reserve가 compaction/growth로 pointer를 확정한 뒤 destination을 계산하므로 stale pre-reserve pointer를 사용하지 않습니다.

#### 성능 결정과 불변 조건

- 변경 전에는 모든 successful read마다 `append_bytes`의 copy가 한 번 발생했습니다. 변경 후에는 kernel이 persistent tail에 직접 쓰므로 그 copy가 사라집니다.
- zero-copy 전체 구현은 아닙니다. capacity growth 때 unread bytes를 새 allocation으로 복사하고, newline result는 caller-owned allocation으로 복사합니다. 이 SHA의 EOF tail은 내부 allocation을 직접 이전하지만 후속 authoritative context engine에서는 별도 result copy로 통일됩니다.
- reserve의 arithmetic/allocation failure는 old allocation을 부분 교체하지 않지만 compatibility caller가 node를 폐기합니다. read error도 이 SHA에서는 selected node의 unread bytes를 정리합니다. 후속 explicit 문맥의 retry semantics를 소급하지 않습니다.
- observable line semantics는 바뀌지 않도록 기존 `656528529ade` matrix가 같은 production public API를 계속 통과하도록 Makefile에 포함됩니다. 실제 실행은 이 환경에서 수행하지 못했습니다.

### 5.6 `a0654d9de446` — `test(perf): 4 MiB 입력의 작업량 기준 고정`

- **Commit:** `a0654d9de446`
- **Subject:** `test(perf): 4 MiB 입력의 작업량 기준 고정`
- **중요도:** **A**
- **태그:** `PERF`, `TEST`, `LINE_STATE`

#### 원자료에서 확인된 역할

4 MiB newline 없는 입력을 `BUFFER_SIZE=4096`으로 읽고 system call, allocation, release, internal copy volume을 test hook으로 셉니다. manifest는 checksum과 함께 **1025 reads, 13 allocations, 11 copy operations, 12,533,760 copied bytes**를 고정하며 wall-clock은 정보로만 남기고 pass/fail 조건으로 사용하지 않습니다.

#### 해당 SHA에서 확인할 테스트·계측 코드

1. 4 MiB fixture가 생성되는 방식과 newline이 없음을 보장하는 코드를 찾습니다.
2. `BUFFER_SIZE=4096` build가 실제 production implementation에 적용되는 지점을 확인합니다.
3. read, allocation, release, copy를 세는 hook 또는 wrapper와 production 호출 지점의 연결을 추적합니다.
4. 1025 reads가 data reads와 EOF read를 어떻게 합산한 값인지 실제 counter increment 위치로 계산합니다.
5. 13 allocations, 11 copy operations, 12,533,760 bytes가 각각 어떤 growth/result-copy 단계에서 발생하는지 기록합니다.
6. line checksum이 caller-visible result의 정확성을 어떻게 확인하는지 expected value와 계산 함수를 찾습니다.
7. wall-clock 결과가 assertion에 들어가지 않고 informational output에만 쓰이는지 확인합니다.
8. manifest mismatch가 어떤 failure message와 종료 상태를 만드는지 확인합니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | geometric growth, direct tail read, persistent `scan`이 하나의 4 MiB EOF-tail에서 per-chunk append copy와 repeated full scan 없이 진행된다는 구조적 기준입니다. |
| **Failure / boundary** | newline이 전혀 없는 매우 긴 record는 매 chunk마다 전체 buffer를 복사하거나 처음부터 재검색하는 회귀를 가장 크게 드러냅니다. |
| **Test technique** | Makefile이 production을 `metric_malloc`, `metric_free`, `metric_read`, `BLR_COPY_OBSERVER=metric_copy_observer`로 다시 컴파일하고, output manifest를 `diff -u`로 비교하는 deterministic operation counting입니다. |
| **Production path** | 4 MiB fixture → explicit context create → reserve/direct read/scan 반복 → EOF → final result allocation/copy → FNV-1a checksum → counters 출력 → manifest diff입니다. |
| **증명하는 것** | 이 입력과 ABI/configuration에서 read·allocation·copy 수와 result checksum이 기준과 같음을 증명합니다. |
| **증명하지 않는 것** | 모든 입력 크기의 수학적 상한, zero-copy, 실제 latency 상한, thread safety는 증명하지 않습니다. allocation bytes에는 `sizeof(t_blr_reader)`가 들어가므로 ABI 변화에도 민감합니다. |
| **분류** | wall-clock 대신 재현 가능한 구조적 작업량을 고정한 deterministic performance regression입니다. |
| **막는 회귀** | linear growth는 allocation/copy 수·bytes를, scratch append-copy 재도입은 copy count/bytes를, repeated scan은 해당 observer가 직접 scan을 세지는 않지만 전체 구조와 wall output/후속 설계 검토에서 드러납니다. manifest는 주로 read/allocation/copy 회귀를 직접 고정합니다. |

#### 고정 수치 재구성

| Metric | 원자료 기준 | 해당 SHA의 counter 증가 지점 | 직접 계산한 이유 |
| --- | ---: | --- | --- |
| reads | 1025 | `metric_read`가 production의 data/EOF read마다 증가 | 4,194,304 / 4,096 = 1,024 data reads와 EOF 확인 1회입니다. zero-length descriptor probe의 계수 여부는 metric wrapper 구현에서 별도로 다루며 manifest의 data path 합계는 1025입니다. |
| allocations | 13 | `metric_malloc`: context, buffer growth, final line | context 1회 + capacities 8,192부터 8,388,608까지 11회 + 4,194,305-byte caller line 1회입니다. |
| copy operations | 11 | production `copy_bytes`가 length > 0일 때 `BLR_COPY_OBSERVER` 호출 | 11번의 growth 중 최초 8,192 allocation은 old length 0이라 관찰되지 않습니다. 이후 growth copy 10회와 final result copy 1회입니다. |
| copied bytes | 12,533,760 | `metric_copy_observer(length)` 누적 | growth 길이 `4096+12288+28672+61440+126976+258048+520192+1044480+2093056+4190208 = 8,339,456`; final 4,194,304를 더하면 12,533,760입니다. |
| line checksum | `790796585941148453` | `tests/metrics/test_metrics.c`의 caller-visible line FNV-1a 계산 | manifest가 길이 4,194,304와 checksum을 함께 비교해 작업량만 맞고 내용이 틀린 구현을 거부합니다. |

allocation byte 합계 `20,963,393`은 buffer capacities 합, final line allocation, 측정 ABI의 context object 크기를 포함합니다. wall-clock은 stderr 정보로만 출력되고 `tests/manifests/metrics-4mib.txt`에 없으므로 pass/fail 조건이 아닙니다. `metrics` target은 실제 output을 파일에 쓰고 `diff`가 다르면 nonzero로 종료합니다.

**실행 상태**

- 예상 명령: `make metrics` at `a0654d9de446`.
- **미실행:** 이 작업 환경에서는 해당 SHA의 checkout을 로컬에 구성하지 못해 binary 실행 및 manifest diff를 수행하지 않았습니다.
- 위 수치는 `Makefile`, `tests/metrics/*`, manifest, 해당 SHA의 production code를 대조해 재구성한 code-inspection 결과입니다.

## 6. 불변 조건 ledger

| 불변 조건 | 최초로 확인할 commit | 강화 또는 표현 변경 | 검증 commit | 학습자가 남길 코드 근거 |
| --- | --- | --- | --- | --- |
| trailing newline이 없는 nonempty EOF suffix도 data로 반환합니다. | `85e4c2a41a4c` | `39a2b9055728`에서 line sequence의 마지막 record로 통합됩니다. | `656528529ade` | `length != 0`/`unread_length()!=0` EOF branch와 unterminated boundary fixtures. |
| caller-visible line은 internal mutable buffer와 독립된 allocation입니다. | `85e4c2a41a4c` | `39a2b9055728`에서 record 단위 copy-out으로 사용됩니다. | `656528529ade` | 초기 SHA는 buffer ownership transfer로 독립성을 얻고, 39a는 `malloc+copy`; test는 first result 유지와 pointer inequality를 확인합니다. |
| consumed prefix와 unread suffix를 index로 구분합니다. | `7e64d3d79ad4` | `39a2b9055728`에서 `scan`과 one-record consumption으로 확장됩니다. | `656528529ade` | `unread_length=end-begin`, extraction의 `begin=line_end`, adjacent line fixture. |
| reserve failure는 old unread interval을 부분 교체하지 않습니다. | `7e64d3d79ad4` | `dbf1abd21121`에서 reserve-before-direct-read 순서로 유지됩니다. | 해당 failure test는 다른 Thread의 commit과 연결해 확인 | allocation 전 state와 `malloc` 성공 뒤 old free 순서; `fd03a831686b` fault harness는 legacy cleanup까지 검증합니다. |
| read chunk boundary는 logical record boundary가 아닙니다. | `39a2b9055728` | 여러 `BUFFER_SIZE`와 pipe input에서 검증됩니다. | `656528529ade` | persistent `[begin,scan,end)`와 size matrix의 동일 expected lines. |
| 이미 scan한 bytes를 반복해 전체 검색하지 않습니다. | `39a2b9055728` | direct tail read와 함께 대용량 operation count로 간접 고정됩니다. | `a0654d9de446` | `find_line_end`가 current `scan`부터 시작하고 no-delimiter 시 `scan=end`; metric은 대용량 구조 회귀를 감시합니다. |
| per-read scratch append-copy가 없습니다. | `dbf1abd21121` | 구조적 copy count로 회귀를 감시합니다. | `a0654d9de446` | `read(..., bytes+end, ...)`와 11-copy manifest. |

## 7. 실패 → 수정 → 검증 연결

이 개발 흐름에는 subject가 `fix`인 commit이 없습니다. 대신 초기 위험을 representation·parser·performance refactor로 제거하고 test가 이를 고정하는 흐름을 기록합니다.

| 기존 상태 또는 위험 | 원자료에서 확인된 원인 | 설계/변경 commit | 검증 commit | 실제 실패 처리와 assertion |
| --- | --- | --- | --- | --- |
| newline 없는 마지막 bytes를 버릴 위험 | EOF를 record content와 구분하지 못함 | `85e4c2a41a4c` | `656528529ade` | EOF에서 nonempty length/unread를 result로 이전하며 tests는 newline 없는 body와 EOF `NULL` 순서를 확인합니다. |
| 반환한 prefix와 남은 suffix를 한 length로 관리 | 소비 위치와 allocation capacity가 같은 개념으로 묶임 | `7e64d3d79ad4` | `656528529ade` | `[begin,end)`와 compaction/growth가 suffix를 보존하고 adjacent-line expected sequence가 손실을 검출합니다. |
| whole stream을 한 record로 반환 | persistent delimiter scan과 prefix retirement가 없음 | `39a2b9055728` | `656528529ade` | `find_line_end` + `extract_line`; multiple/empty lines fixtures가 한 call 한 record와 newline retention을 확인합니다. |
| 매 read마다 scratch → internal append copy | system call destination과 persistent storage가 분리됨 | `dbf1abd21121` | `a0654d9de446` | direct tail destination으로 변경하고 metric copy observer가 scratch copy 재도입 시 count/bytes 증가를 검출합니다. |
| wall-clock만으로 performance를 판단 | scheduling과 machine load 때문에 재현성이 낮음 | `a0654d9de446` | 같은 commit의 manifest | wall time은 assertion에서 제외하고 read/allocation/copy/checksum의 exact manifest를 `diff`합니다. |

## 8. 소유권 / state / responsibility 변화

| 단계 | Internal allocation owner | Caller result owner | Active state 표현 | read 책임 | extraction 책임 |
| --- | --- | --- | --- | --- | --- |
| `85e4c2a41a4c` | 읽는 동안 `g_reader`; EOF nonempty에서 pointer를 넘길 때 ownership 종료 | `release_final_line` 반환 이후 caller | append-only `length/capacity` accumulator | `get_next_line`이 stack scratch로 반복 read | EOF에서 internal allocation 자체를 이전 |
| `7e64d3d79ad4` | 여전히 `g_reader` | EOF transfer 이후 caller | unread `[begin, end)` window | stack scratch → `append_bytes` | 아직 line split 없음 |
| `39a2b9055728` | singleton allocation; record extraction 후 suffix가 있으면 유지 | 각 newline result allocation과 EOF-transferred tail | `begin/scan/end` 기반 record state | stack scratch read와 append | `find_line_end` + `extract_line`, EOF `release_final_line` |
| `dbf1abd21121` | descriptor node가 buffer 소유 | newline result 또는 EOF tail caller | 같은 window 유지 | reserved tail로 direct read | 기존 의미 유지 |
| `a0654d9de446` | production 변화 없음; metric wrappers가 획득·해제를 관찰 | production 변화 없음 | operation hooks로 관찰 | `metric_read` call/byte count 측정 | `copy_bytes` observer와 final checksum 측정 |

## 9. 개발 흐름의 최종 상태

원자료 기준으로 이 개발 흐름이 끝났을 때 다음이 확립되어 있습니다.

- whole-stream accumulator가 unread-window 기반 one-line streaming parser로 바뀌었습니다.
- newline이 있으면 result에 포함하며, 다음 record의 bytes는 internal suffix로 남습니다.
- EOF의 nonempty unterminated suffix는 final line으로 반환됩니다.
- parser 동작은 여러 `BUFFER_SIZE`와 어려운 descriptor/input boundary에서 검증됩니다.
- read는 reserved internal tail로 직접 수행됩니다.
- 4 MiB workload의 structural operation count가 manifest로 고정됩니다.

### 최종 상태 설명

- **최종 state fields와 각 의미:** `fd`는 borrowed descriptor number, `bytes`는 context/node-owned allocation, `begin`은 unread 시작, `scan`은 다음 검사 위치, `end`는 valid byte exclusive end, `capacity`는 allocation 크기, 후속 context engine 시점의 `reached_eof`는 terminal EOF 기억, `next`는 legacy list link입니다.
- **한 line을 반환할 때 commit되는 mutation:** `find_line_end`가 exclusive end를 찾고 result allocation/copy/NUL이 성공한 뒤 `begin=line_end`, `scan=begin`을 commit합니다. `[begin,end)`의 다음 bytes는 남습니다.
- **EOF tail을 반환할 때 commit되는 mutation:** EOF flag를 세우고 unread nonempty이면 `extract_line(reader,end)`로 별도 caller allocation을 만든 뒤 `begin=scan=end`가 됩니다. 다음 call은 EOF입니다. 초기 SHA의 direct transfer와 최종 context copy를 구분했습니다.
- **reserve/read/result allocation failure에서 유지되는 state:** reserve growth는 새 allocation 성공 전 old interval을 유지합니다. 최종 explicit engine에서는 read error와 result allocation failure도 logical unread를 유지하고 scan을 begin으로 복구합니다. 그러나 `39a2b9055728`와 `dbf1abd21121`의 legacy path는 실패 시 node를 폐기했으므로 시점별 차이가 있습니다.
- **이 개발 흐름이 다루지 않고 이후 Thread로 넘기는 범위:** descriptor별 hidden state isolation, caller-controlled context lifecycle, explicit result enum, `EINTR`/`EAGAIN` semantics와 retry는 다른 Thread에서 확립됩니다.

## 10. 최종 architecture 또는 실행 순서 정리

해당 SHA의 실제 symbol로 아래 흐름을 완성합니다. 최종 HEAD symbol을 대신 넣지 않습니다.

```text
public reader call
    → [find_line_end / reader->scan < reader->end]
        → found: [extract_line의 malloc, copy_bytes, NUL]
        → [reader->begin = line_end; reader->scan = reader->begin]
        → caller-owned line 반환
    → not found: [reserve_bytes(reader, BUFFER_SIZE)]
        → [tail reuse / compact_bytes / geometric allocation]
        → read into [reader->bytes + reader->end]
        → positive read: [end += read_size; bytes[end] = '\0'] 후 재검색
        → EOF + unread bytes: [해당 시점 release_final_line 또는 context의 extract_line(end)]
        → EOF + no unread bytes: completion
        → failure: [historical legacy는 discard, 최종 explicit context는 state 보존]
```

### 코드 근거가 포함된 최종 설명

1. **Entry와 state 접근:** 초기에는 `g_reader`, descriptor-state 이후에는 `find_reader(fd)`가 선택한 `t_reader`, explicit API 이후에는 caller가 넘긴 `t_blr_reader`가 state owner입니다.
2. **Scan 범위와 delimiter 판정:** `find_line_end`는 `[scan,end)`만 검사하고 newline을 발견하면 `scan`을 newline 다음으로 증가시켜 exclusive `line_end`를 반환합니다.
3. **Reserve branch와 overflow 방어:** `appended > SIZE_MAX - unread_length - 1`을 먼저 검사하며, tail reuse → compact → geometric grow 순으로 선택합니다. growth는 `malloc` 성공 뒤에만 old allocation을 교체합니다.
4. **Direct read와 index mutation:** `dbf1abd21121`부터 reserve 뒤 `bytes+end`로 읽고 positive actual count만 `end`에 더한 뒤 sentinel을 복구합니다.
5. **Line extraction과 suffix preservation:** caller result가 준비된 뒤 `begin=line_end`, `scan=begin`; `end`는 유지되므로 뒤 bytes가 다음 call의 unread suffix입니다.
6. **EOF tail과 stable completion:** 초기 legacy는 nonempty tail allocation을 이전하고 state를 제거합니다. explicit engine은 `reached_eof`를 기록하고 tail을 copy-out한 뒤 repeated `BLR_EOF`를 반환합니다.
7. **Operation count가 bounded되는 이유:** capacity가 geometric하게 증가하고, scan은 이미 검사한 bytes를 건너뛰며, read가 persistent tail로 직접 들어갑니다. 따라서 4 MiB case의 read 1025·allocation 13·copy 11/12,533,760 bytes 기준이 유지됩니다.

## 11. 학습 완료 자가 점검

- [x] `85e4c2a41a4c`가 finished line parser가 아닌 이유를 실제 코드로 보였습니다.
- [x] `[begin, end)`와 allocation capacity를 혼동하지 않습니다.
- [x] `scan`이 반복 full-buffer scan을 피하는 방법을 index trace로 설명할 수 있습니다.
- [x] newline 포함 result length와 NUL termination 위치를 코드로 확인했습니다.
- [x] one successful call이 다음 line의 byte를 소비하지 않는다는 근거가 있습니다.
- [x] empty stream과 empty line과 unterminated tail을 구분할 수 있습니다.
- [x] reserve failure 시 old allocation이 유지되는 mutation 순서를 확인했습니다.
- [x] scratch-buffer 제거 전후의 copy path를 실제 diff로 비교했습니다.
- [x] 1025/13/11/12,533,760 수치를 counter 위치로 재구성했습니다.
- [x] 이 개발 흐름의 동작을 최종 HEAD 없이 각 SHA 기준으로 설명할 수 있습니다.

---

# 단일 전역 상태에서 파일 디스크립터별 호환 상태까지

## 1. 개발 흐름 목표

한 개의 hidden singleton reader가 explicit state parameter를 거쳐 descriptor number별 linked state로 바뀌는 과정을 복원합니다. interleaved call, invalid descriptor, fd number reuse, allocation/read failure에서 lookup·mutation·cleanup이 정확히 한 node에만 적용되는지 실제 코드와 테스트로 확인합니다.

### 원자료에서 확인된 프로젝트 항목

- **Project profile:** compatibility API의 hidden descriptor-indexed state와 stream별 소유권/isolation을 담당하는 Thread입니다.
- **Core architecture:** finished compatibility layer는 descriptor number를 key로 하는 file-scope 연결 리스트에 reader 문맥을 보관합니다.
- **Critical 불변 조건:** state와 cleanup은 한 descriptor 또는 문맥에만 scoped되어야 하며, 한 stream의 failure가 다른 stream의 unread bytes를 지우면 안 됩니다.
- **Critical 불변 조건:** failed/completed descriptor의 stale hidden state가 나중에 재사용된 같은 integer fd에 붙으면 안 됩니다.
- **Major engineering difficulty:** interleaved descriptors를 지원하면서 broad cleanup, stale integer-fd state, shared scan/buffer를 방지하는 문제입니다.
- **Practical engineering area:** deterministic replacement of allocation, release, read를 통해 partial construction과 cross-descriptor failure를 재현하는 문제입니다.

### 원자료에서 확인된 중요성

helper에 state를 명시적으로 전달하는 변화는 준비 단계입니다. 결정적인 변화는 active descriptor마다 독립 buffer, cursor, disposal path를 소유하게 하는 것입니다. 이후 테스트는 global cursor, broad reset, retained failed node가 stream contamination 또는 fd reuse contamination을 일으킬 수 있음을 검증하며, fault harness는 이 isolation 불변 조건을 partial construction과 I/O failure까지 확장합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- singleton state는 어느 helper에서 암묵적으로 읽고 수정되었으며 explicit parameter가 이를 어떻게 드러내는가?
- descriptor node는 어떤 필드를 소유하고, list와 node의 allocation/cleanup 책임은 어디에 있는가?
- lookup, create, mutation, EOF removal, error removal은 어떤 caller chain으로 한 fd에만 적용됩니까?
- invalid descriptor call이 이미 buffer를 가진 valid descriptor를 건드리지 않는 이유는 무엇입니까?
- closed/write-only fd의 stale node가 남으면 같은 integer fd가 재사용될 때 어떤 data contamination이 가능한가?
- allocation/read fault가 node 생성 전, 생성 중, unread bytes 보유 후에 발생할 때 각각 어떤 owner가 cleanup하는가?

## 3. 완료 기준

- singleton helper와 explicit state helper의 signature/call graph 차이를 설명할 수 있습니다.
- descriptor-indexed linked collection의 node fields, lookup/create/remove 경로를 실제 코드로 복원했습니다.
- EOF/error cleanup이 affected node 하나만 제거한다는 근거가 있습니다.
- interleaved test의 call sequence와 각 node의 unread state를 단계별로 추적했습니다.
- fd reuse hazard가 단순 leak 문제가 아니라 correctness 문제인 이유를 테스트 준비 코드로 설명할 수 있습니다.
- deterministic fault injection이 cross-node 소유권과 cleanup을 어떻게 검사하는지 확인했습니다.

## 4. 커밋 목록
| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 Thread 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `fc01012e8521` | `refactor(state): reader 상태를 helper 인자로 전달` | **B** | `REFACTOR`, `READER_LIFECYCLE` | helper mutation이 hidden singleton field가 아니라 explicit reader object를 대상으로 하게 합니다. |
| 2 | `a4f41cbf2cf0` | `feat(state): 디스크립터별 읽기 상태 분리` | **S** | `ARCH`, `READER_LIFECYCLE`, `RISK` | file descriptor를 key로 하는 독립 linked reader node를 생성합니다. |
| 3 | `61f8b9858672` | `test(state): 교차 디스크립터 상태 격리 검증` | **A** | `TEST`, `READER_LIFECYCLE`, `RISK` | interleaved stream과 unrelated invalid descriptor에서 state isolation을 검증합니다. |
| 4 | `d3e2b37fca03` | `test(error): 오류 발생 시 디스크립터 상태 정리 검증` | **A** | `TEST`, `READER_LIFECYCLE`, `RISK` | unusable descriptor cleanup과 descriptor-number reuse hazard를 고정합니다. |
| 5 | `fd03a831686b` | `test(failure): 메모리 할당과 읽기 실패 처리 검증` | **A** | `TEST`, `POSIX_IO`, `RISK` | 정확한 allocation/read transition에 fault를 주입하고 다른 reader node가 생존하는지 검증합니다. |

## 5. 커밋별 학습 기록
### 5.1 `fc01012e8521` — `refactor(state): reader 상태를 helper 인자로 전달`

- **Commit:** `fc01012e8521`
- **Subject:** `refactor(state): reader 상태를 helper 인자로 전달`
- **중요도:** **B**
- **태그:** `REFACTOR`, `READER_LIFECYCLE`

#### 원자료에서 확인된 역할

buffering과 extraction helper가 hidden singleton storage를 직접 참조하지 않고 explicit reader-state object를 인자로 받도록 바꿉니다. observable 동작은 유지하지만 모든 mutation이 어느 state instance에 속하는지 signature에 드러납니다. reserve, scan, extraction, cleanup helper를 독립 state에 재사용할 준비 단계입니다.

#### 해당 SHA에서 확인할 코드

1. 직전 commit에서 file-scope singleton을 직접 참조하던 helper 목록을 찾습니다.
2. 이 SHA에서 state pointer/reference가 추가된 helper signature와 모든 호출 지점을 비교합니다.
3. reserve, scan, extraction, cleanup 중 어떤 helper가 state를 mutate하고 어떤 helper가 read-only인지 기록합니다.
4. explicit state object를 생성하거나 보유하는 상위 caller를 찾고, 아직 singleton 수명이 남아 있는지 확인합니다.
5. behavior-preserving refactor임을 test 결과와 return/상태 전이의 동일성으로 확인합니다.

#### 간결한 학습 기록

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| 변경 전 implicit singleton access | 직전 `get_next_line.c`의 `reset_reader`, `unread_length`, `compact_bytes`, `reserve_bytes`, `append_bytes`, `find_line_end`, `extract_line`, `release_final_line`이 `g_reader`를 직접 사용 | helper가 어떤 reader instance를 변경하는지 signature에 드러나지 않았습니다. |
| 변경 후 helper signature | `t_reader *reader` 또는 `const t_reader *reader`가 각 helper 인자로 추가됨 | mutation 대상이 caller가 넘긴 object로 한정됩니다. `unread_length(const t_reader *)`는 read-only이고 나머지 reserve/scan/extract/cleanup은 state를 변경합니다. |
| 상위 caller가 전달하는 state instance | `get_next_line`이 `reader = &g_reader`로 singleton address를 얻어 각 helper에 전달 | helper는 instance-agnostic해졌지만 public entry는 여전히 하나의 process-wide state만 선택합니다. |
| cleanup helper의 대상 state | `reset_reader(reader)` 또는 extraction failure에서 전달받은 `reader` | cleanup 범위가 함수 인자로 표현되지만 실제 호출 대상은 아직 `g_reader` 하나입니다. |
| 여전히 남은 singleton 소유권 | file-scope `static t_reader g_reader` | persistent allocation과 indices의 owner는 여전히 singleton이며 multi-descriptor isolation은 아직 없습니다. |

- 이 refactor가 직접 해결한 coupling은 helper 내부의 전역 이름 의존입니다. 테스트 가능한 동작과 상태 전이는 그대로 두면서 여러 reader instance에 재사용할 수 있는 함수 형태를 만듭니다.
- 해결하지 않은 문제는 state selection입니다. `get_next_line`이 계속 `&g_reader` 하나만 전달하므로 fd를 바꾸면 기존 bytes를 버리고, 교차 호출은 서로의 state를 유지할 수 없습니다.
- 다음 `a4f41cbf2cf0`은 같은 helper들을 `find_reader(fd)` 또는 `create_reader(fd)`로 선택한 node에 전달합니다. 이 때문에 refactor 자체를 multi-fd feature로 과대평가하면 안 됩니다.
- 해당 SHA의 runtime test는 이 환경에서 실행하지 못했습니다. behavior-preserving 여부는 commit diff의 return branch와 helper body/call-site 대응으로 확인했습니다.

### 5.2 `a4f41cbf2cf0` — `feat(state): 디스크립터별 읽기 상태 분리`

- **Commit:** `a4f41cbf2cf0`
- **Subject:** `feat(state): 디스크립터별 읽기 상태 분리`
- **중요도:** **S**
- **태그:** `ARCH`, `READER_LIFECYCLE`, `RISK`

#### 원자료에서 확인된 Problem

persistent read-ahead state를 unrelated descriptor가 공유할 수 없습니다. interleaved calls에서는 각 stream이 자신의 unread bytes, cursor positions, cleanup state를 유지해야 하며, 한 descriptor의 EOF/error가 다른 descriptor를 reset하면 안 됩니다.

#### 원자료에서 확인된 Decision

compatibility layer가 descriptor number를 key로 하는 linked collection of reader nodes를 보관합니다. lookup, creation, mutation, EOF disposal, error cleanup은 한 node를 대상으로 수행합니다.

#### 원자료에서 확인된 중요성

이 commit은 hidden legacy state의 authoritative 소유권 boundary를 확립합니다. multi-descriptor 사용을 가능하게 하고 cross-stream data leakage를 막지만, integer fd reuse가 lifecycle hazard가 된다는 점도 함께 만듭니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

1. linked node type과 list head의 storage duration을 찾습니다.
2. 각 node가 descriptor number, unread interval, scan cursor, allocation 중 무엇을 직접 소유하는지 필드별로 기록합니다.
3. fd lookup helper의 traversal 조건과 found/not-found return을 확인합니다.
4. 새 node allocation, reader state initialization, list insertion 순서를 추적합니다.
5. partial construction 중 allocation failure가 발생할 때 아직 list에 삽입되지 않은 object를 누가 해제하는지 확인합니다.
6. public `get_next_line(fd)` call이 lookup/create된 node를 parser helper에 전달하는 caller chain을 작성합니다.
7. EOF에서 affected node를 unlink하고 release하는 순서와 head/middle/tail removal을 확인합니다.
8. unrecoverable descriptor error가 broad list reset이 아니라 one-node removal을 호출하는지 확인합니다.
9. 다른 node의 `begin/scan/end/allocation`에 접근할 수 있는 global mutation이 남아 있는지 검색합니다.

#### Node 소유권 map

| Resource / state | 생성 지점 | owner | mutation 지점 | release 조건 | release 함수 |
| --- | --- | --- | --- | --- | --- |
| list head/link | `create_reader`: `reader->next = g_readers; g_readers = reader` | file-scope `g_readers`가 head를 보관하고 각 node가 자신의 `next` link를 보관 | head insertion, `discard_reader`의 pointer-to-pointer unlink | selected node의 EOF/error/consumed-buffer cleanup | `discard_reader`가 link를 우회시킴 |
| descriptor number | `create_reader(fd)`의 `reader->fd = fd` | node가 key 값을 보관하지만 OS descriptor 자체는 caller 소유 | 생성 후 변경하지 않음 | node 수명 종료 | 별도 release 없음; `close`하지 않음 |
| byte allocation | `reserve_bytes`의 lazy `malloc` | 해당 node | append, compact, growth, scan/extract | error/EOF/node 제거; EOF tail transfer 시 caller로 이전 | 보통 `discard_reader`의 `free(reader->bytes)`; transfer 전 `bytes=NULL` |
| unread indices | node 생성 시 0 | 해당 node | `reserve_bytes`, `compact_bytes`, `find_line_end`, `extract_line` | node 수명 종료 | 메모리와 함께 소멸 |
| node allocation | `create_reader`의 `malloc(sizeof(*reader))` | insertion 전 local `reader`, insertion 뒤 연결 리스트 | insertion/removal | EOF, selected error, buffer fully consumed policy | `discard_reader`의 `free(reader)` |

`create_reader`는 node 한 번만 allocation하고 internal buffer는 `NULL`로 시작합니다. node allocation 실패 시 insertion 전에 `NULL`을 반환하므로 해제할 부분 구성물도 없습니다. internal buffer allocation은 이후 `reserve_bytes`가 수행하며 실패 시 public caller가 selected node를 `discard_reader`로 정리합니다.

#### Lookup → create → use → remove 흐름

```text
get_next_line(fd)
    → find_reader(fd)
        → found: existing t_reader node
        → not found: create_reader(fd)로 malloc/init 후 head insertion
    → find_line_end / reserve_bytes / read / extract_line on exactly one node
        → line: unread suffix가 있으면 node retained
        → EOF: discard_reader 또는 release_final_line 후 selected node 제거
        → unrecoverable error: discard_reader(selected node)
```

`find_reader`는 `reader != NULL && reader->fd != fd` 동안 `next`를 따라갑니다. `discard_reader`는 `t_reader **link = &g_readers`로 시작해 `*link = reader->next`를 수행하므로 head, middle, tail을 같은 코드로 처리합니다.

**최소 코드 근거**

`a4f41cbf2cf0`, `get_next_line.c`, `discard_reader`:

```c
link = &g_readers;
while (*link != NULL && *link != reader)
    link = &(*link)->next;
if (*link == NULL)
    return ;
*link = reader->next;
free(reader->bytes);
free(reader);
```

이 함수는 selected node의 predecessor link만 바꾸며 다른 node의 buffer나 indices를 순회·초기화하지 않습니다.

#### Failure와 descriptor reuse 분석

- stale node가 남은 위험 sequence는 다음과 같습니다. old fd가 첫 line을 반환한 뒤 read-ahead suffix를 node에 보유하고, caller가 fd를 close하며, OS가 새 file을 같은 integer로 열고, `find_reader(new_fd)`가 old node를 찾으면 새 file의 첫 read 전에 old suffix를 반환할 수 있습니다. 이는 leak이 아니라 새 stream에 old data가 섞이는 correctness violation입니다.
- 실제 key는 `reader->fd == fd`라는 integer 비교뿐입니다. open file description identity, inode, offset identity는 확인하지 않습니다. 따라서 timely disposal과 caller lifecycle discipline가 필요합니다.
- probe/read 오류와 EOF branch는 `find_reader(fd)`가 선택한 node만 `discard_reader`에 전달합니다. unlink code는 다른 link를 보존하고 다른 node의 allocation에 접근하지 않습니다.
- `extract_line` allocation 실패도 이 시점에는 selected node 전체를 폐기합니다. 비파괴 retry는 후속 context engine의 변화이며 이 SHA에 소급하지 않습니다.

#### 이 commit의 보장과 한계

- **보장:** 동시에 active한 여러 integer fd가 각각 독립 `bytes`, `begin`, `scan`, `end`, `capacity`를 소유하고 public call은 하나의 node만 mutate·remove합니다.
- **한계:** state 수명은 hidden list와 EOF/error에 묶여 있습니다. caller가 명시적으로 reset/cancel/destroy하거나 EOF/error/AGAIN을 구분할 API는 없습니다. integer reuse 자체를 감지하는 identity check도 없습니다.
- **후속 검증:** `61f8b9858672`는 정상 interleaving과 invalid call isolation, `d3e2b37fca03`은 write-only/closed descriptor의 local cleanup, `fd03a831686b`은 exact allocation/read failure와 cross-node survival을 나누어 확인합니다.

### 5.3 `61f8b9858672` — `test(state): 교차 디스크립터 상태 격리 검증`

- **Commit:** `61f8b9858672`
- **Subject:** `test(state): 교차 디스크립터 상태 격리 검증`
- **중요도:** **A**
- **태그:** `TEST`, `READER_LIFECYCLE`, `RISK`

#### 원자료에서 확인된 역할

여러 descriptor를 각각 끝까지 읽은 뒤 비교하는 대신 call을 교차시킵니다. 각 descriptor가 자신의 unread suffix와 line order를 유지하는지, invalid descriptor call이 이미 buffered data를 가진 valid reader를 방해하지 않는지 검증합니다.

#### 해당 SHA에서 확인할 테스트 코드

1. 두 개 이상 descriptor의 input fixture와 expected line sequence를 찾습니다.
2. call order가 A1 → B1 → A2 → invalid → B2처럼 실제로 interleave되는지 순서를 기록합니다.
3. 각 call 직전 production list에 어떤 node/state가 존재할 것으로 기대하는지 test setup으로 추론한 뒤 코드로 확인합니다.
4. invalid descriptor call의 expected result와 그 뒤 valid descriptor의 다음 expected line을 연결합니다.
5. shared buffer, global scan cursor, broad cleanup이 있었다면 어느 assertion이 처음 실패하는지 기록합니다.
6. descriptor와 returned line의 cleanup이 test 자체에서 정확히 수행되는지 확인합니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | `find_reader`가 fd별 node를 선택하고 line success 뒤 selected node의 suffix만 유지하며, invalid call은 matching node가 없으면 다른 node를 건드리지 않아야 합니다. |
| **Failure / boundary** | `test_alternating_descriptors`의 서로 다른 길이·line 수를 가진 두 pipe와 `test_invalid_fd_preserves_other_state`의 `-1` call을 구분합니다. |
| **Test technique** | real pipe descriptors를 A/B/A/B 순서로 호출하고 exact result 문자열을 비교하는 deterministic state-isolation regression입니다. |
| **Production path** | 각 `get_next_line(fd)` → list lookup → selected node의 buffered scan/read/extract → result free; EOF에서 selected node removal 순입니다. |
| **증명하는 것** | left/right line order, 각 suffix의 보존, unrelated `-1` call 뒤 valid fd의 `"second"` 반환을 증명합니다. |
| **증명하지 않는 것** | forced integer fd reuse, every allocation failure, same-context thread synchronization은 다루지 않습니다. |
| **분류** | descriptor-indexed architecture의 핵심 isolation을 좁게 고정하는 focused deterministic regression입니다. |
| **막는 회귀** | singleton/global cursor면 left 첫 call 뒤 right state로 교체되어 다음 left assertion이 실패합니다. broad invalid cleanup이면 `"second"` assertion이 실패합니다. shared buffer면 expected stream 문자열이 교차 오염됩니다. |

#### Interleaving state trace

실제 commit에는 alternating case와 invalid-fd case가 별도 test 함수로 존재합니다. 스캐폴드의 A1 → B1 → A2 → invalid → B2 예시는 하나의 함수에 그대로 구현된 순서가 아닙니다.

| 호출 순서 | fd | 예상 선택 node | 호출 전 unread bytes | 반환 | 호출 후 node 유지/제거 |
| ---: | ---: | --- | --- | --- | --- |
| 1 | `left` | 새 left node | 없음 | `"left one\n"` | read-ahead `"left two"`가 있어 유지 |
| 2 | `right` | 새 right node | 없음 | `"right one\n"` | `"right two\nright three"` suffix로 유지 |
| 3 | `left` | 기존 left node | `"left two"` | EOF tail `"left two"` | node 제거 |
| 4 | `right` | 기존 right node | `"right two\nright three"` | `"right two\n"` | `"right three"`로 유지 |
| 5 | `right` | 기존 right node | `"right three"` | EOF tail `"right three"` | node 제거 |
| invalid call | `-1` | matching node 없음 | 별도 valid fd는 `"second"` 보유 | `NULL` | valid node 유지 |
| 이후 valid call | valid fd | 기존 valid node | `"second"` | `"second"` | EOF tail 반환 후 제거 |

모든 returned line은 test에서 `free`하며 pipe read descriptor는 마지막에 `close`합니다. 이 환경에서는 해당 test binary를 실행하지 않았고 source assertion만 확인했습니다.

### 5.4 `d3e2b37fca03` — `test(error): 오류 발생 시 디스크립터 상태 정리 검증`

- **Commit:** `d3e2b37fca03`
- **Subject:** `test(error): 오류 발생 시 디스크립터 상태 정리 검증`
- **중요도:** **A**
- **태그:** `TEST`, `READER_LIFECYCLE`, `RISK`

#### 원자료에서 확인된 역할

write-only descriptor, closed descriptor, state creation 이후 발생한 failure가 affected buffer와 metadata를 제거하는지 검증합니다. 동시에 한 descriptor의 error가 다른 descriptor의 unread bytes를 보존하는지 확인합니다. stale node가 같은 integer fd에 붙는 위험을 cleanup correctness로 다룹니다.

#### 해당 SHA에서 확인할 테스트 코드

1. write-only descriptor를 생성하고 reader가 unusable state를 판정하게 만드는 setup을 찾습니다.
2. closed descriptor의 integer value를 보관한 뒤 call하는 순서를 확인합니다.
3. state가 먼저 생성된 뒤 descriptor가 unusable해지는 scenario가 있는지 찾고, failure가 발생하는 production transition을 기록합니다.
4. error call 전후 list/node 존재 여부를 직접 검사하는 hook, allocation count, behavior-based indirect assertion 중 어떤 방식을 쓰는지 확인합니다.
5. affected fd number가 실제로 재사용되도록 새 descriptor를 여는 test가 있는지 확인하고 expected first line을 기록합니다.
6. 다른 valid descriptor가 보유한 unread suffix를 error 전후로 확인하는 call sequence를 추적합니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | unusable descriptor의 matching hidden node만 제거하고 borrowed fd는 library가 닫지 않으며 unrelated node suffix는 남아야 합니다. |
| **Failure / boundary** | write-only pipe end, 첫 line 뒤 외부에서 close된 fd, 동시에 read-ahead를 가진 다른 valid fd를 다룹니다. |
| **Test technique** | 내부 hook 없이 실제 descriptor lifecycle과 result 동작만 관찰하는 focused regression입니다. |
| **Production path** | write-only는 zero-length probe 실패 전에/중 matching lookup·cleanup, closed-state case는 기존 node lookup → probe failure → `discard_reader` → survivor node lookup/extract입니다. |
| **증명하는 것** | write-only fd call이 `NULL`이고 그 write end가 여전히 `write` 가능함, closed selected node의 실패 뒤 다른 fd가 `"survive"`를 반환함을 증명합니다. |
| **증명하지 않는 것** | 모든 allocation rollback, ordered `EINTR`/`EAGAIN`, 실제 동일 integer fd 강제 재사용은 이 commit의 test에 없습니다. |
| **분류** | descriptor 소유권과 local cleanup을 고정하는 real-descriptor focused regression입니다. |
| **막는 회귀** | library가 borrowed fd를 close하면 후속 `write` assertion이 실패하고, global list clear이면 survivor의 `"survive"` assertion이 실패합니다. 이중 해제는 이 test가 직접 계측하지 않습니다. |

#### 기존 가정 → 실제 위험 → 검증 연결

- **기존 가정:** node key가 integer fd이므로 그 integer가 같은 active stream을 계속 나타낸다는 암묵적 가정이 생길 수 있습니다. production `find_reader`는 숫자만 비교합니다.
- **실제 위험:** old node가 suffix를 보유한 채 fd가 close되고 같은 숫자가 새 file에 재사용되면 새 stream이 old suffix를 먼저 받을 수 있습니다.
- **필요한 cleanup decision:** probe/read 오류와 EOF에서 selected node를 즉시 unlink하고 owned buffer/node를 해제해야 합니다. `discard_reader`의 link commit 뒤 free 순서가 이를 수행합니다.
- **regression evidence:** 이 SHA의 실제 evidence는 closed selected fd의 `NULL`과 survivor의 `"survive"`, write-only fd의 후속 `write` 성공입니다.

**관찰된 범위 차이**

고정된 Source 역할은 descriptor-number reuse hazard를 포함하지만, `d3e2b37fca03`의 실제 `tests/test_reader.c`는 새 descriptor가 같은 integer를 쓰도록 `dup2`하거나 반복 open하지 않습니다. 따라서 이 commit은 **stale node를 제거해야 하는 전제와 local cleanup을 간접 검증하지만, integer reuse 후 새 stream 첫 line을 직접 검증하지는 않습니다.** 강제 reuse fixture는 후속 `249093ba477a`의 explicit-context test에서 확인됩니다. 고정 역할은 변경하지 않고 실제 test 범위를 이곳에 기록했습니다.

### 5.5 `fd03a831686b` — `test(failure): 메모리 할당과 읽기 실패 처리 검증`

- **Commit:** `fd03a831686b`
- **Subject:** `test(failure): 메모리 할당과 읽기 실패 처리 검증`
- **중요도:** **A**
- **태그:** `TEST`, `POSIX_IO`, `RISK`

#### 원자료에서 확인된 역할

테스트 build에서 allocation, release, read를 deterministic replacement로 바꿉니다. 각 allocation point를 실패시키고, short read, partial input 전후 error, invalid/duplicate free를 제어·기록합니다. 이 개발 흐름에서는 특히 한 descriptor의 partial construction 또는 read failure가 다른 descriptor node를 손상시키지 않는지 확인합니다.

#### 해당 SHA에서 확인할 harness 코드

1. `malloc`, `free`, `read` replacement가 production code에 연결되는 compile definition, macro, link substitution 방식을 찾습니다.
2. allocation call index를 세고 특정 n번째 call을 실패시키는 control state를 확인합니다.
3. read script가 positive short read, zero, negative/error를 어떤 entry로 표현하는지 기록합니다.
4. invalid free와 duplicate free를 판단하기 위해 allocation 소유권을 어떤 table/list로 추적하는지 확인합니다.
5. 한 descriptor node가 unread bytes를 가진 상태에서 다른 descriptor에 fault를 주입하는 fixture를 찾습니다.
6. fault 뒤 valid descriptor를 다시 호출해 기존 unread bytes가 남아 있음을 확인하는 assertion을 기록합니다.
7. 각 scenario 종료 시 outstanding allocation, release count, node cleanup을 확인하는 공통 검사를 찾습니다.

#### 테스트 커밋 분석 — descriptor isolation 관점

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | selected descriptor의 construction/read failure cleanup이 다른 node의 unread state에 영향을 주지 않고, 모든 allocation에 단일 owner가 있어야 합니다. |
| **Failure / boundary** | exhaustive n번째 allocation 실패, `read_limit(3)` short reads, first read EIO, third read EIO after prior progress, right fd failure while left node has suffix를 구분합니다. |
| **Test technique** | Makefile의 `-Dmalloc=test_malloc -Dfree=test_free -Dread=test_read`로 production object를 대체 runtime에 연결하고 allocation table·call-index failure·read limit/fail index를 사용합니다. |
| **Production path** | control 설정 → replacement call → production node/buffer/result failure branch → selected node cleanup → `check_clean_runtime`; cross-fd case는 survivor node를 다시 호출합니다. |
| **증명하는 것** | tested allocation indices에서 no leak, no invalid/이중 해제, short-read progress, selected read failure cleanup, right failure 뒤 left `"left two"` suffix 생존을 증명합니다. |
| **증명하지 않는 것** | `BLR_AGAIN`, ordered errno sequence, same-context retry는 아직 구현·테스트되지 않았습니다. middle EIO case는 이 SHA에서 accepted prefix를 보존하지 않고 selected node를 폐기합니다. |
| **분류** | exact acquisition/failure transition을 반복하는 deterministic failure-injection regression harness입니다. |
| **막는 회귀** | partially retained failed node, broad list cleanup, leaked buffer/node, invalid/duplicate free, survivor suffix loss를 각각 counter와 expected line으로 검출합니다. |

**Harness 실제 구조**

- `test_malloc`은 allocation attempt를 증가시키고 configured n번째 call에서 `NULL`을 반환합니다. 성공 pointer는 최대 4096-entry table에 `live` 상태로 기록됩니다.
- `test_free`는 live pointer면 해제·dead 표시, 이미 dead인 known pointer면 이중 해제 counter, table에 없는 pointer면 invalid-free counter를 증가시킵니다.
- `test_read`는 target fd의 positive-count call을 세며 `fault_read_limit`으로 실제 read 요청량을 줄이고 `fault_read_fail_on(fd,index)`에서 `errno=EIO`, `-1`을 반환합니다. zero-length descriptor probe는 fault 대상에서 제외됩니다.
- 이 SHA에는 errno/return/bytes를 임의 entry 배열로 공급하는 일반 ordered “read script”가 없습니다. 그 기능은 `11033bd85c59`에서 `fault_read_script`로 추가됩니다. 고정된 scaffold 표현은 유지하고 실제 mechanism 차이를 기록합니다.
- `check_clean_runtime`은 live allocation 0, invalid free 0, 이중 해제 0을 확인합니다. harness 자체가 duplicate/invalid pointer를 일부러 free하는 guard test도 있어 detector가 실제로 증가하는지 검증합니다.

#### Fault point별 소유권 기록

| 주입 지점 | 실패 직전 owner/resource | production return | affected node 상태 | 다른 node 상태 | leak/free 검사 |
| --- | --- | --- | --- | --- | --- |
| node allocation | local `create_reader`가 아직 node를 얻지 못함 | `NULL` | list insertion 없음 | 기존 nodes 무변경 | live 0, invalid/double 0 |
| buffer allocation/growth | selected node가 node와 old buffer/unread를 소유 | legacy `NULL` | reserve 실패 후 selected node를 `discard_reader`; old owned resources 해제 | fault가 해당 node에만 적용되므로 기존 other node link 유지 | exhaustive allocation loop 뒤 clean runtime |
| caller result allocation | selected node가 delimiter와 buffer를 보유 | legacy `NULL` | 이 SHA의 `extract_line` failure가 selected node 전체를 폐기 | 다른 node는 직접 건드리지 않음 | allocation index sweep과 clean runtime |
| read before progress | selected node가 빈/초기 buffer를 소유 | `NULL` | EIO branch에서 selected node 제거 | cross-fd case에서 left node 유지 | `fault_read_calls()==1`, clean runtime |
| read after partial progress | selected node가 이미 읽은 bytes를 소유 | `NULL` | third read EIO에서 accepted bytes와 함께 selected node 제거 | 별도 survivor case는 other fd에 first-read fault를 주어 left suffix를 확인 | exact call count와 clean runtime |

실행 명령은 Makefile상 `make failure-test`이며 sizes 1, 2, 42, 1024를 순회합니다. 이 환경에서는 checkout을 만들지 못해 실행하지 않았고, 테스트 통과를 주장하지 않습니다.

## 6. 불변 조건 ledger

| 불변 조건 | 도입/준비 commit | 위험이 드러나는 commit | 검증/강화 commit | 학습자가 남길 코드 근거 |
| --- | --- | --- | --- | --- |
| helper mutation은 전달받은 reader state에만 적용됩니다. | `fc01012e8521` | singleton이 multi-fd를 지원하지 못하는 한계 | `a4f41cbf2cf0` | pointer-parameter helper signatures와 `get_next_line`의 selected node 전달. |
| active descriptor마다 독립 unread buffer와 cursor를 소유합니다. | `a4f41cbf2cf0` | global buffer/cursor가 interleaving을 오염시킬 위험 | `61f8b9858672` | node fields, `find_reader(fd)`, alternating expected sequence. |
| EOF/error cleanup은 affected descriptor node 하나만 제거합니다. | `a4f41cbf2cf0` | invalid fd가 다른 node까지 지우는 위험 | `61f8b9858672`, `d3e2b37fca03` | pointer-to-pointer `discard_reader`; invalid/closed call 뒤 survivor result. |
| failed/completed fd의 stale state는 reused integer에 붙지 않습니다. | `a4f41cbf2cf0`의 timely disposal decision | descriptor-number reuse hazard | `d3e2b37fca03` | production error/EOF unlink는 확인되지만 이 SHA test는 forced reuse가 없다는 한계를 함께 기록했습니다. |
| allocation/read failure가 다른 node의 unread bytes를 지우지 않습니다. | node-local ownership model | exact fault transition | `fd03a831686b` | right fd read fault 뒤 left node의 `"left two"` 반환. |
| acquired allocation마다 owner가 하나이며 invalid/duplicate free가 없습니다. | production ownership paths | partial construction/failure | `fd03a831686b` | live allocation table, `test_free` classification, scenario별 `check_clean_runtime`. |

## 7. 실패 → 수정 → 검증 연결

이 개발 흐름에는 subject가 `fix`인 commit이 없습니다. architecture change와 회귀 테스트가 failure risk를 닫는 구조입니다.

| 기존 가정 | 실제 failure 또는 위험 | root cause | 수정된 decision/architecture | 검증 commit | 학습자 근거 |
| --- | --- | --- | --- | --- | --- |
| 하나의 persistent state면 충분함 | interleaved descriptor의 suffix/cursor contamination | hidden singleton ownership | explicit state parameter 후 descriptor별 node | `61f8b9858672` | `fc010` helper parameter와 `a4f` list; left/right alternating exact lines. |
| 한 fd failure 때 전체 state를 정리해도 됨 | unrelated valid stream의 unread data 손실 | cleanup scope가 global | affected node만 unlink/release | `61f8b9858672`, `d3e2b37fca03` | invalid `-1`/closed fd 뒤 valid suffix 반환. |
| failed fd number는 다시 쓰이지 않음 | 새 stream이 stale bytes를 상속 | integer fd를 state key로 사용하면서 timely disposal 누락 | EOF/error 즉시 node disposal | `d3e2b37fca03` | production unlink는 직접 확인; actual test는 forced reuse를 하지 않는다는 범위 제한 포함. |
| normal path test면 ownership이 충분히 검증됨 | partial construction, double free, cross-node corruption | failure transition이 비결정적 | deterministic allocation/read replacement | `fd03a831686b` | n번째 allocation sweep, EIO call index, live/invalid/double counters, survivor call. |

## 8. 소유권 / state / responsibility 변화

| 단계 | persistent state owner | state 선택 방식 | cleanup 범위 | 다른 descriptor와의 관계 |
| --- | --- | --- | --- | --- |
| refactor 전 | file-scope singleton | implicit | global 또는 singleton | 독립성 없음 |
| `fc01012e8521` | 여전히 상위 singleton owner | caller가 `&g_reader`를 helper에 전달 | 전달된 state | multi-fd는 아직 없음 |
| `a4f41cbf2cf0` | linked list의 각 node | descriptor number lookup | affected node | 각 node가 독립 state 소유 |
| `61f8b9858672` 이후 검증 | production 변화 없음 | interleaved lookup 관찰 | invalid call의 local effect | expected sequence로 isolation 고정 |
| `fd03a831686b` 이후 검증 | production 변화 없음 | fault 시 selected node 관찰 | partial construction까지 검사 | survivor node state 확인 |

### 실제 코드로 채울 responsibility map

- **List head owner와 storage duration:** `static t_reader *g_readers`가 process 수명 동안 hidden list head를 보관합니다.
- **Node creation 책임 함수:** `create_reader(fd)`가 node를 allocation·초기화하고 head insertion을 commit합니다.
- **Parser mutation 책임 함수:** `find_line_end`, `reserve_bytes`, `compact_bytes`, `extract_line`과 public `get_next_line` read loop가 selected node만 변경합니다.
- **Node unlink 책임 함수:** `discard_reader(t_reader *reader)`가 pointer-to-pointer traversal로 정확한 link를 우회합니다.
- **Node/internal buffer release 책임 함수:** `discard_reader`가 `free(reader->bytes)` 뒤 `free(reader)`합니다. EOF tail 소유권 transfer에서는 먼저 `reader->bytes=NULL`로 분리합니다.
- **Public compatibility adapter의 책임:** descriptor를 검증하고 node를 lookup/create하며 parser result를 `char *` 또는 `NULL`로 반환하고, 이 시점에는 EOF/error에서 hidden node cleanup policy를 적용합니다.

## 9. 개발 흐름의 최종 상태

원자료 기준으로 이 개발 흐름이 끝났을 때 compatibility reader는 descriptor number로 persistent state를 분리하며, 각 node는 자신의 unread interval, scan cursor, allocation, 정리 과정을 가집니다. interleaving, invalid descriptor, fd reuse hazard, allocation/read fault에서 state와 cleanup의 범위가 검증됩니다.

### 최종 상태 설명

- **list와 node의 실제 자료구조:** file-scope singly 연결 리스트가며 각 node는 `fd`, `bytes`, `begin`, `scan`, `end`, `capacity`, `next`를 갖습니다.
- **lookup/create/remove의 정확한 call graph:** `get_next_line` → zero-read validation → `find_reader` → missing이면 `create_reader` → selected parser helpers; terminal/error 또는 buffer fully consumed policy에서 `discard_reader`.
- **line result 뒤 node가 유지되는 이유:** `[begin,end)`에 read-ahead suffix가 있으면 다음 call에서 같은 fd가 `find_reader`로 같은 cursor/buffer를 이어 써야 합니다. 모든 bytes를 소비한 경우 해당 historical implementation은 node를 제거할 수 있습니다.
- **EOF/error 뒤 node가 제거되는 조건:** read 0에 unread가 없거나 EOF tail 소유권을 넘긴 뒤, probe/read/reserve/result-allocation의 unrecoverable legacy failure에서 selected node를 unlink/free합니다.
- **fd reuse contamination을 막는 실제 코드:** 정상적으로 오류/EOF를 관찰한 경로의 `discard_reader`가 old numeric key를 list에서 제거합니다. 외부 close 뒤 library를 다시 호출하지 않으면 hidden node를 자동 감지할 수 없으므로 caller discipline와 후속 explicit 문맥이 필요합니다.
- **explicit context API가 아직 필요한 이유:** hidden list는 caller가 EOF 전에 buffered state를 취소·reset·destroy할 수 없고, `NULL`이 EOF/error/wait를 구분하지 못하며 integer fd identity 변화도 handle과 분리할 수 없습니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
get_next_line(fd)
    → [read(fd, ..., 0) descriptor validation/probe]
    → [find_reader(fd) on file-scope g_readers]
        → existing node: reuse its unread state
        → missing node: [create_reader allocation/init/head insertion]
    → [find_line_end/reserve_bytes/read/extract_line with selected node]
        → line: keep node for suffix/read-ahead
        → EOF: unlink selected node → release owned buffer/node
        → unrecoverable error: unlink selected node → release owned buffer/node
    → map result to compatibility return
```

### 실제 symbol과 mutation을 넣어 완성

1. **List traversal:** `find_reader`가 `reader = g_readers`부터 `reader->fd == fd`를 찾을 때까지 `next`를 이동합니다.
2. **Node allocation과 insertion commit point:** `create_reader`가 모든 scalar/pointer field를 초기화한 뒤 `reader->next=g_readers; g_readers=reader`로 list 소유권을 넘깁니다.
3. **Helper state parameter 전달:** public caller가 lookup/create한 `reader`를 `find_line_end`, `reserve_bytes`, `extract_line`, `unread_length`에 넘깁니다.
4. **Line 뒤 retained state:** result copy 성공 뒤 `begin=line_end`, `scan=begin`; `begin<end`이면 node와 suffix가 남습니다.
5. **EOF/error unlink 순서:** `discard_reader`가 먼저 predecessor link를 `reader->next`로 바꾸고 그 다음 buffer와 node를 free합니다. EOF transfer는 buffer pointer를 caller에게 분리한 뒤 discard합니다.
6. **Head/middle/tail removal 처리:** pointer-to-pointer traversal이 head에는 `&g_readers`, 이후에는 `&previous->next`를 사용하므로 별도 special case가 없습니다.
7. **다른 node를 보존하는 근거:** unlink는 selected pointer를 찾은 한 link만 갱신하고 다른 node의 fields를 읽거나 초기화하지 않습니다. interleaving/closed/fault tests가 survivor result를 확인합니다.

## 11. 학습 완료 자가 점검

- [x] `fc01012e8521`이 multi-descriptor feature 자체가 아니라 준비 refactor인 이유를 설명할 수 있습니다.
- [x] node가 소유하는 resource와 단순히 보관하는 fd integer를 구분했습니다.
- [x] list lookup/create/insert의 failure rollback을 실제 코드로 확인했습니다.
- [x] interleaved call마다 선택되는 node와 unread suffix를 추적했습니다.
- [x] invalid descriptor가 다른 node를 reset하지 않는 근거가 있습니다.
- [x] stale node와 fd number reuse의 관계를 concrete scenario로 설명할 수 있습니다.
- [x] write-only/closed descriptor의 정리 과정을 확인했습니다.
- [x] fault harness가 invalid/duplicate free를 검출하는 방법을 확인했습니다.
- [x] `fd03a831686b`를 POSIX Thread와 중복 학습하되 여기서는 node isolation 관점으로 기록했습니다.
- [x] 최종 HEAD의 context object를 이 시점 node 구현에 소급하지 않았습니다.

---

# 명시적 리더 수명과 하나의 기준 처리 엔진

## 1. 개발 흐름 목표

EOF/error에 묶인 hidden 수명에서 벗어나 caller가 생성·reset·destroy하는 opaque reader 문맥과 명시적 result state를 도입하는 과정을 복원합니다. 이후 `get_next_line`이 별도 parser를 유지하지 않고 같은 engine을 사용하는지, line allocation failure가 input consumption을 commit하지 않는지, descriptor borrowing과 kernel offset coupling이 API 사용 규칙으로 어떻게 검증되는지 확인합니다.

### 원자료에서 확인된 프로젝트 항목

- **Core architecture:** `t_blr_reader`는 heap object와 internal buffer를 소유하고 supplied descriptor는 빌립니다.
- **Core architecture:** `blr_reader_create`, `blr_reader_next`, `blr_reader_reset`, `blr_reader_destroy`가 explicit 수명/result semantics를 제공합니다.
- **Core architecture:** `blr_reader_next`는 authoritative state-transition engine이고 `get_next_line(fd)`는 그 위의 compatibility adapter입니다.
- **Critical 불변 조건:** successful line은 caller-owned independent allocation이며, non-line result는 valid output pointer를 `NULL`로 둡니다.
- **Critical 불변 조건:** reset/destroy는 owned memory를 해제하지만 borrowed descriptor를 닫지 않습니다.
- **Critical 불변 조건:** allocation/read failure가 explicit 문맥의 unread input을 부분 소비하면 안 되며, line extraction은 caller-visible allocation 성공 뒤에만 cursor movement를 commit합니다.
- **Major engineering difficulty:** explicit 문맥을 추가하면서 compatibility API와 parsing implementation이 중복되거나 diverge하지 않도록 하는 문제입니다.
- **Practical engineering area:** descriptor borrowing, offset coupling, fd reuse, dup aliases, reset requirement를 테스트로 명시하는 문제입니다.

### 원자료에서 확인된 중요성

프로젝트는 hidden 수명을 explicit state object로 바꾸고 caller가 cancel, reset, destroy할 수 있게 합니다. result enumeration은 data와 status를 분리하고, adapter는 compatibility function과 explicit API가 다른 parser로 갈라지는 것을 막습니다. 테스트는 borrowed descriptor의 read-ahead가 kernel offset에 결합되고, integer reuse에는 새 문맥이 필요하며, output allocation failure가 input consumption을 commit하면 안 된다는 비자명한 결과를 확립합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- 문맥은 어떤 resource를 소유하고 descriptor는 왜 borrowed resource입니까?
- create/reset/destroy는 buffer, indices, EOF flag, descriptor에 각각 어떤 mutation을 수행하는가?
- `blr_reader_next`의 result enum과 output pointer rule은 `char *`/`NULL` ambiguity를 어떻게 제거하는가?
- repeated EOF가 new read 없이 stable terminal이 되는 상태는 어디에 저장됩니까?
- legacy adapter는 문맥을 어떻게 lookup하고 result taxonomy를 어떻게 축소해 반환하는가?
- allocation failure에서 delimiter 또는 EOF tail을 소비하지 않으려면 cursor commit은 어느 시점 이후여야 하는가?
- external seek, close/reuse, `dup` aliases가 문맥의 buffered read-ahead와 어떤 관계를 갖는가?

## 3. 완료 기준

- public opaque type과 lifecycle functions의 선언·구현·소유권을 확인했습니다.
- create/reset/destroy가 descriptor를 닫지 않는다는 코드와 test 근거가 있습니다.
- `blr_reader_next`의 every result branch와 output pointer mutation을 추적했습니다.
- stable EOF flag와 repeated call 동작을 실제 코드로 설명할 수 있습니다.
- `get_next_line`이 authoritative engine을 호출하고 return을 map하는 call graph를 복원했습니다.
- newline result와 EOF-tail result의 allocation failure가 non-consuming이라는 근거가 있습니다.
- seek/fd reuse/dup alias test의 의미와 API 사용자가 지켜야 할 lifecycle rule을 구분했습니다.

## 4. 커밋 목록
| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 Thread 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `903768a43bf4` | `feat(context): 명시적 reader 수명 API 추가` | **A** | `ARCH`, `READER_LIFECYCLE`, `API_CONTRACT` | opaque create/reset/destroy를 공개하고 descriptor ownership은 caller에게 남깁니다. |
| 2 | `2e681112b304` | `feat(reader): 명시적 결과 상태 API 추가` | **S** | `ARCH`, `API_CONTRACT`, `CORE` | explicit line/EOF/error result-state API를 정의합니다. |
| 3 | `9bd6ebf429e2` | `refactor(reader): legacy API를 context reader에 연결` | **A** | `REFACTOR`, `INTEGRATION`, `API_CONTRACT` | legacy function을 context engine에 연결하고 allocation failure의 non-consuming extraction을 확립합니다. |
| 4 | `249093ba477a` | `test(context): 결과 상태와 컨텍스트 수명 검증` | **A** | `TEST`, `READER_LIFECYCLE`, `API_CONTRACT` | descriptor borrowing, seek 후 reset, fd reuse, dup alias, stable result를 검증합니다. |
| 5 | `a24ad4e49cc4` | `test(failure): 컨텍스트의 line 할당 재시도 검증` | **A** | `TEST`, `READER_LIFECYCLE`, `RISK` | newline-delimited line과 EOF-tail allocation failure가 input loss 없이 재시도됨을 증명합니다. |

## 5. 커밋별 학습 기록
### 5.1 `903768a43bf4` — `feat(context): 명시적 reader 수명 API 추가`

- **Commit:** `903768a43bf4`
- **Subject:** `feat(context): 명시적 reader 수명 API 추가`
- **중요도:** **A**
- **태그:** `ARCH`, `READER_LIFECYCLE`, `API_CONTRACT`

#### 원자료에서 확인된 역할

opaque `t_blr_reader`와 create, reset, destroy operation을 공개해 reader 수명을 caller가 관리하게 합니다. 문맥은 heap object와 internal buffer를 소유하지만 supplied descriptor는 빌립니다. reset/destroy는 buffered state를 버리지만 descriptor를 닫지 않으며, legacy descriptor list도 같은 lifecycle primitive를 사용하도록 적응합니다.

#### 해당 SHA에서 확인할 실제 코드

1. public header에서 opaque type declaration과 create/reset/destroy signature를 찾습니다.
2. context implementation type의 fields를 찾되 public header에 layout이 노출되지 않는지 확인합니다.
3. create가 context object와 internal buffer를 언제 allocation하는지, descriptor와 indices를 어떻게 초기화하는지 추적합니다.
4. create 중 partial allocation failure의 rollback owner와 return rule을 기록합니다.
5. reset이 buffer capacity를 release하는지 또는 재사용하는지 해당 SHA 코드로 확인하고, indices/EOF state를 어떤 값으로 되돌리는지 적습니다.
6. destroy가 NULL-safe인지, internal buffer와 object release 순서가 무엇인지 확인합니다.
7. reset/destroy path에 `close` call이 없는지 symbol search와 test로 확인합니다.
8. legacy descriptor-list node가 새 lifecycle primitive를 어디서 호출하는지 caller/callee를 기록합니다.

#### 소유권 ledger

| Resource | 획득 지점 | owner | reset 시 | destroy 시 | descriptor close 여부 |
| --- | --- | --- | --- | --- | --- |
| context heap object | `blr_reader_create`: fd probe 뒤 `malloc(sizeof(*reader))` | explicit API에서는 caller가 handle을 보유; legacy insertion 뒤 hidden list | 유지 | internal buffer 해제 후 object 해제 | 해당 없음 |
| internal byte buffer | create 시에는 `NULL`; 첫 `reserve_bytes`에서 lazy allocation | 해당 context | `free(reader->bytes)`, pointer/indices/capacity를 0으로 초기화 | `free(reader->bytes)` 후 object 해제 | 해당 없음 |
| supplied 파일 디스크립터 | caller가 create 전에 열어 전달 | caller | `reader->fd`는 유지되고 fd는 열려 있음 | fd는 열려 있음 | reset/destroy 모두 `close`하지 않음 |
| legacy list node/context | `create_legacy_reader`가 `blr_reader_create` 결과를 head에 insertion | `g_readers` hidden list | public reset을 자동 호출하지 않음 | `discard_legacy_reader`가 unlink 후 `blr_reader_destroy` | borrowed fd 유지 |

**실제 상태와 초기화**

- public header는 `typedef struct s_blr_reader t_blr_reader;`만 노출하고 field layout은 `get_next_line.c`에 둡니다.
- private object는 이 SHA에서 `fd`, `bytes`, `begin`, `scan`, `end`, `capacity`, `next`를 갖습니다. `reached_eof`는 아직 없습니다.
- `blr_reader_create`는 zero-length `read`로 fd를 검사한 뒤 context object 하나만 allocation합니다. internal buffer는 `NULL`이므로 create 중 “object 성공 후 buffer 실패”라는 두 단계 partial construction은 이 SHA에 존재하지 않습니다.
- object allocation 실패 시 list insertion이나 다른 resource 획득 전에 `NULL`을 반환합니다. explicit caller나 hidden list 어느 쪽에도 owner가 생기지 않습니다.
- reset은 buffer를 free하고 `bytes=NULL`, `begin=scan=end=capacity=0`으로 만듭니다. `fd`와 legacy `next` link는 바꾸지 않습니다.
- destroy는 `reader==NULL`이면 즉시 반환하며, non-NULL이면 buffer를 먼저 free하고 object를 free합니다.

**최소 코드 근거**

`903768a43bf4`, `get_next_line.c`, lifecycle:

```c
void blr_reader_reset(t_blr_reader *reader)
{
    if (reader == NULL)
        return ;
    free(reader->bytes);
    reader->bytes = NULL;
    reader->begin = 0;
    reader->scan = 0;
    reader->end = 0;
    reader->capacity = 0;
}
```

`close(reader->fd)`가 없으므로 reset은 parser state만 폐기합니다.

#### 학습자가 복원할 API decision

- hidden list 수명만으로는 caller가 EOF 전에 stream을 포기하거나 `lseek` 후 old read-ahead를 제거할 수 없습니다. explicit handle은 caller가 그 시점에 reset/destroy할 수 있게 합니다.
- opacity는 buffer pointer, indices, linked-list link 같은 불변 조건-bearing fields를 caller가 임의 변경하지 못하게 합니다. public API는 대신 “context-owned memory, borrowed descriptor”라는 소유권만 노출합니다.
- reset은 같은 context·same fd를 유지한 채 buffered bytes와 cursor를 폐기하는 operation이고, destroy는 context allocation까지 끝냅니다. 둘 다 OS descriptor lifecycle operation이 아닙니다.
- 이 commit 시점에는 explicit result enum이 없고 read outcome을 richer status로 전달하는 `blr_reader_next`도 아직 없습니다. 다음 `2e681112b304`가 handle을 실제 state-machine API로 완성합니다.

### 5.2 `2e681112b304` — `feat(reader): 명시적 결과 상태 API 추가`

- **Commit:** `2e681112b304`
- **Subject:** `feat(reader): 명시적 결과 상태 API 추가`
- **중요도:** **S**
- **태그:** `ARCH`, `API_CONTRACT`, `CORE`

#### 원자료에서 확인된 Problem

historical `char *` interface는 clean EOF, allocation/I/O error, temporary incompleteness를 `NULL` 하나로 겹치며, persistent stream state와 repeated terminal result를 명시적으로 다룰 handle이 없습니다.

#### 원자료에서 확인된 Decision

`blr_reader_next`가 explicit result enumeration을 반환하고 successful line은 output pointer를 통해 전달합니다. 문맥이 EOF state를 기록해 repeated call이 new read 없이 terminal로 유지되며, non-line result는 supplied output pointer를 null로 둡니다.

#### 원자료에서 확인된 중요성

이 commit은 finished library의 richer state-machine contract를 만듭니다. caller는 data 소유권과 control status를 분리하고, null data pointer를 서로 다른 outcome으로 추측하지 않아도 됩니다. 이 engine은 이후 `get_next_line`의 authoritative implementation이 됩니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

1. public header의 result enum 정의와 이 SHA에 실제 존재하는 enumerator를 정확히 기록합니다. 후속 `BLR_AGAIN`을 소급하지 않습니다.
2. `blr_reader_next` signature에서 context, output line pointer, return type의 역할을 구분합니다.
3. 함수 entry에서 invalid argument를 검사하고 output pointer를 `NULL`로 초기화하는 순서를 확인합니다.
4. buffered newline, need-more-read, EOF-tail, clean EOF, error branch가 각각 어떤 enum을 반환하는지 control flow를 작성합니다.
5. successful line에서 output pointer 소유권이 caller로 넘어가는 지점을 확인합니다.
6. 문맥의 EOF flag가 최초 EOF에서 설정되는 위치와 repeated call에서 read를 건너뛰는 조건을 찾습니다.
7. empty input이 `LINE`이 아니라 EOF로 가는 조건과 nonempty EOF tail이 먼저 `LINE`이 되는 조건을 비교합니다.
8. error branch가 output pointer를 stale caller value로 남기지 않는지 entry/exit mutation을 확인합니다.

#### Result-state table

| 상황 | 해당 SHA enum | `*line` 값 | context unread state | EOF flag | 다음 call behavior |
| --- | --- | --- | --- | --- | --- |
| buffered newline 발견 | `BLR_LINE` | 새 caller-owned NUL-terminated allocation | 성공 뒤 `begin=line_end`, `scan=begin`; suffix 유지 | 기존 값 유지 | suffix scan/read 계속 |
| EOF + nonempty tail | `BLR_LINE` | `[begin,end)` copy 결과 | 성공 뒤 `begin=scan=end` | `1` | 다음 call은 clean EOF path |
| clean EOF / repeated EOF | `BLR_EOF` | `NULL` | empty interval 유지 | 최초 EOF에서 `1`, 이후 유지 | positive data read 없이 다시 `BLR_EOF` |
| invalid argument | `BLR_ERROR` | output pointer가 valid하면 entry에서 `NULL`; `line==NULL`이면 쓸 곳 없음 | reader가 valid한 경우 state mutation 없음 | 유지 | caller가 argument를 고쳐 재호출 가능 |
| allocation 또는 I/O error | `BLR_ERROR` | `NULL` | read error는 accepted unread 유지; result malloc 실패는 `begin` 유지, `scan=begin` 복구 | read error 전 값 또는 EOF-tail failure면 `1` | 같은 문맥으로 retry 가능 |

#### 코드 근거 기록

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| public enum과 function declaration | `BLR_ERROR=-1`, `BLR_EOF=0`, `BLR_LINE=1`; `t_blr_result blr_reader_next(t_blr_reader *, char **)` | 이 SHA에는 세 상태만 있고 `BLR_AGAIN`은 없습니다. |
| output pointer 초기화 rule | entry의 `if (line != NULL) *line = NULL;`가 argument validation보다 앞 | stale caller value는 valid output pointer를 넘긴 모든 non-line path에서 제거됩니다. |
| successful line 소유권 transfer | `malloc(length+1)`, copy/NUL 후 `*line = ...`, `BLR_LINE` | caller가 반환 allocation을 free합니다. context buffer와 alias하지 않습니다. |
| EOF flag set/check | data read가 0일 때 `reached_eof=1`; buffered scan 뒤 `if (reader->reached_eof)` | EOF-tail과 clean EOF를 context state로 기억합니다. |
| repeated EOF 빠른 처리 경로 | `reached_eof && unread_length()==0`에서 `BLR_EOF` | 후속 positive-count data read loop에는 들어가지 않습니다. 다만 entry의 zero-length fd probe는 여전히 호출됩니다. |
| error return과 state 처리 | probe/read negative는 `BLR_ERROR`; allocation 실패는 `scan=begin` 후 error | explicit context 자체와 logical unread bytes를 free하지 않습니다. |

**최소 코드 근거**

`2e681112b304`, `get_next_line.c`, `blr_reader_next` entry:

```c
if (line != NULL)
    *line = NULL;
if (reader == NULL || line == NULL)
    return (BLR_ERROR);
```

result data와 status가 분리되며 non-line result가 stale pointer를 남기지 않습니다.

#### 이 commit이 보장하는 것과 이후 변화

- 실제 enumerator는 `LINE`, `EOF`, `ERROR`입니다. `EAGAIN`/`EWOULDBLOCK`은 아직 별도 결과가 아니며 후속 `f0055ae5cf19`에서 `BLR_AGAIN`이 추가됩니다.
- empty stream은 첫 positive-count read가 0이고 unread length가 0이므로 `BLR_EOF`; unterminated nonempty tail은 EOF flag 설정 후 먼저 `BLR_LINE`, 다음 call에 `BLR_EOF`입니다.
- source의 “repeated call이 new read 없이 terminal”은 data-read loop 기준으로 성립하지만 실제 code는 매 call entry에서 `read(fd,&probe,0)` 검증을 수행합니다. 따라서 **positive-count stream read는 재실행하지 않지만 read system call 자체가 완전히 0회인 것은 아닙니다.**
- allocation failure의 non-consuming behavior는 이 SHA의 inline extraction path에 이미 있습니다. `9bd6ebf429e2`는 이를 공통 `extract_line`으로 옮겨 legacy adapter까지 같은 rule을 쓰게 합니다.
- legacy `get_next_line`은 이 시점에도 별도 parsing path를 유지하므로 engine divergence 위험이 남습니다.

### 5.3 `9bd6ebf429e2` — `refactor(reader): legacy API를 context reader에 연결`

- **Commit:** `9bd6ebf429e2`
- **Subject:** `refactor(reader): legacy API를 context reader에 연결`
- **중요도:** **A**
- **태그:** `REFACTOR`, `INTEGRATION`, `API_CONTRACT`

#### 원자료에서 확인된 역할

`get_next_line`을 context reader 위의 adapter로 줄입니다. descriptor lookup이 문맥을 제공하고, `blr_reader_next`가 buffering, extraction, EOF, failure 상태 전이를 수행하며, adapter는 richer result를 historical `char *`/`NULL`로 mapping합니다. line extraction은 allocation/copy가 성공하기 전까지 consuming하지 않도록 바뀝니다.

#### 변경 전후 authoritative engine 확인

1. 직전 SHA에서 legacy function과 context API가 각각 어떤 parsing path를 사용했는지 call graph를 그립니다.
2. 이 SHA에서 duplicated scan/read/extract code가 제거되고 `blr_reader_next` 호출로 대체된 diff를 찾습니다.
3. descriptor-indexed compatibility state가 raw reader state인지 `t_blr_reader` context인지 확인합니다.
4. adapter가 `LINE`, `EOF`, `ERROR`를 각각 `char *` 또는 `NULL`로 mapping하는 switch/branch를 발췌합니다.
5. EOF/error에서 hidden context/node를 retain 또는 remove하는 정책을 해당 SHA 기준으로 기록합니다.
6. newline-delimited extraction에서 result allocation/copy 성공 전 `begin`이 움직이지 않는지 mutation 순서를 확인합니다.
7. allocation failure 뒤 `scan` cursor가 same delimiter를 다시 찾을 수 있도록 어떤 값으로 복구되는지 확인합니다.
8. EOF tail을 internal buffer 자체로 transfer하지 않고 caller-owned storage로 copy하는 지점을 확인합니다.

**변경 전후 call graph**

```text
2e681112b304
explicit caller → blr_reader_next (inline scan/read/result allocation)
get_next_line(fd) → hidden list → 별도 legacy scan/read/extract/EOF code

9bd6ebf429e2
explicit caller ─┐
                 ├→ blr_reader_next → extract_line
get_next_line ───┘        ↑
  hidden fd→context lookup┘
```

compatibility list의 node type은 raw 별도 type이 아니라 같은 `t_blr_reader` 문맥이며 `next`를 private field로 사용합니다.

#### Transactional extraction trace

| 단계 | result allocation 상태 | `begin` | `scan` | `end` | caller output | retry 시 기대 |
| --- | --- | ---: | ---: | ---: | --- | --- |
| delimiter 발견 직후 | 미시도 | old `begin` | `line_end` | unchanged | `NULL` | same interval still logically unread |
| allocation 실패 | 실패 | unchanged | `begin`으로 복구 | unchanged | `NULL`, `BLR_ERROR` | same delimiter를 다시 찾아 exact line retry |
| allocation/copy 성공 직전 | temp allocation 성공 | unchanged | `line_end` | unchanged | temp line 준비 | 아직 consumption commit 전 |
| cursor commit 후 | 성공 | `line_end` | new `begin` | unchanged | caller-owned line, `BLR_LINE` | next record/suffix |

EOF tail도 `line_end=end`, `reached_eof=1`인 점만 다릅니다. allocation 실패 시 `begin`과 `end`는 유지되고 `scan=begin`; retry는 EOF flag를 보고 같은 `[begin,end)`를 다시 `extract_line`합니다. 성공 뒤 `begin=scan=end`입니다.

#### 코드 근거 기록

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| legacy adapter entry | `char *get_next_line(int fd)` | historical signature는 유지됩니다. |
| descriptor → context lookup | `find_reader(fd)`; missing이면 `create_legacy_reader(fd)` | hidden list도 explicit context object를 소유합니다. |
| `blr_reader_next` authoritative call | `result = blr_reader_next(reader, &line)` | buffering/scan/read/EOF/failure decision이 한 engine에 모입니다. |
| result enum → `char *`/`NULL` mapping | `result == BLR_LINE`이면 line; 그 외 context discard 후 `NULL` | EOF와 ERROR 정보는 compatibility return에서 합쳐집니다. 이 SHA에는 AGAIN이 없습니다. |
| newline result allocation failure rollback | `extract_line`: malloc 실패 시 `reader->scan=reader->begin; return NULL` | begin/end를 소비하지 않고 exact retry state를 남깁니다. |
| `scan` restoration | 위 `scan=begin` | `find_line_end`가 같은 delimiter를 다시 발견할 수 있습니다. |
| EOF tail copy-out | `extract_line(reader, reader->end)`가 malloc/copy/NUL | 초기 legacy의 internal buffer transfer를 제거하고 모든 line을 독립 allocation으로 통일합니다. |
| context/node cleanup policy | adapter가 LINE 외 result에서 `discard_legacy_reader` | explicit 문맥은 caller가 보유하지만 hidden legacy 문맥은 EOF/ERROR에 제거됩니다. |

#### 기존 가정 → 실제 위험 → 수정된 decision

- **기존 가정:** `2e681112b304` 직전 legacy path에서는 `extract_line` allocation 실패 시 selected hidden node를 폐기하고, EOF tail은 internal buffer를 직접 transfer했습니다. explicit path는 별도 inline code로 rollback했습니다.
- **실제 failure:** 두 parser가 유지되면 explicit API는 retry 가능하지만 legacy API는 같은 allocation failure에서 line을 잃는 등 동작이 갈라질 수 있습니다.
- **root cause:** 동일한 scan/read/extract 상태 전이를 두 public entry가 중복 구현하고, legacy extraction이 allocation 성공 전에 recovery 가능 state를 버립니다.
- **수정된 불변 조건:** result allocation과 copy가 성공한 뒤에만 `begin`을 전진시키며, allocation 실패 시 `scan=begin`; 두 API가 `blr_reader_next` 하나를 호출합니다.
- **후속 regression:** `a24ad4e49cc4`는 explicit context에서 newline과 EOF tail의 same-context retry를 deterministic하게 확인합니다.

**관찰된 도입 시점 차이**

고정 Source 역할은 이 commit에서 non-consuming extraction을 확립한다고 설명합니다. 실제 repository inspection상 explicit `blr_reader_next`의 inline path는 `2e681112b304`에서 이미 allocation 실패 시 `scan=begin`, `begin` 유지 동작을 갖고 있었습니다. `9bd6ebf429e2`의 정확한 변화는 그 rule을 공통 `extract_line`으로 centralize하고 legacy adapter까지 적용한 것입니다. Source text는 변경하지 않고 실제 도입·통합 시점을 구분했습니다.

### 5.4 `249093ba477a` — `test(context): 결과 상태와 컨텍스트 수명 검증`

- **Commit:** `249093ba477a`
- **Subject:** `test(context): 결과 상태와 컨텍스트 수명 검증`
- **중요도:** **A**
- **태그:** `TEST`, `READER_LIFECYCLE`, `API_CONTRACT`

#### 원자료에서 확인된 역할

ordered `LINE`, repeated `EOF`, empty input, invalid arguments, descriptor reposition 뒤 reset, destroy without close를 검증합니다. descriptor-number reuse, 같은 open file description을 공유하는 duplicated descriptors, 문맥이 buffer한 read-ahead와 kernel offset의 결합도 다룹니다.

#### 해당 SHA에서 확인할 테스트 코드

1. context create → multiple `blr_reader_next` → destroy의 기본 sequence와 expected enum/output을 찾습니다.
2. repeated EOF에서 read call count 또는 동작이 stable terminal임을 어떻게 확인하는지 기록합니다.
3. invalid context/output argument마다 expected enum과 output pointer 값이 무엇인지 확인합니다.
4. descriptor를 seek/reposition한 뒤 reset 전후 result 차이를 만드는 fixture를 찾습니다.
5. destroy 이후 같은 descriptor를 계속 사용할 수 있음을 read/lseek/close 중 어떤 operation으로 검증하는지 확인합니다.
6. close 후 같은 integer fd가 재사용되는 scenario에서 old 문맥을 버려야 하는 test를 찾습니다.
7. `dup` 또는 equivalent로 같은 open file description을 공유하는 descriptors를 만들고 offset/read-ahead 관계를 어떻게 검증하는지 확인합니다.
8. returned lines가 independent caller-owned allocations임을 release/수명 assertion으로 확인합니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | caller가 context state를 소유하고 fd는 borrow하며, `LINE`/`EOF`/`ERROR`, output NULL rule, reset/destroy semantics가 public contract와 일치해야 합니다. |
| **Failure / boundary** | ordered two-line/EOF, empty, invalid args, external `lseek`, destroy-before-EOF, close+`dup2` reuse, duplicated descriptor alias를 분리합니다. |
| **Test technique** | real pipe/파일 디스크립터와 public context API만 사용하는 broad lifecycle/contract integration tests입니다. |
| **Production path** | create → next 반복 → optional external fd operation → reset/destroy/new create → result/free/close 순이며, private fields를 직접 검사하지 않습니다. |
| **증명하는 것** | result order, output clearing, borrowed fd, reset requirement, new context on reused integer, one context through surviving dup alias를 assertion으로 확인합니다. |
| **증명하지 않는 것** | same-context concurrent synchronization, nonblocking AGAIN, every allocation point, repeated EOF의 exact syscall count는 증명하지 않습니다. |
| **분류** | 여러 lifecycle boundary를 포괄하는 broad public-contract regression입니다. |
| **막는 회귀** | destroy가 fd를 close, reset이 stale read-ahead를 유지, reused fd에 old context 사용, non-line result가 stale output을 남기는 회귀를 각 scenario가 검출합니다. |

#### Descriptor / context 관계 기록

| Scenario | kernel offset 변화 주체 | context buffer 상태 | reset 필요 여부 | 기대 result | 근거 test |
| --- | --- | --- | --- | --- | --- |
| normal sequential read | `blr_reader_next`의 positive reads | read-ahead suffix와 cursors 유지 | 아니요 | `"first\n"` → `"last"` → EOF → EOF | result-state case |
| external seek | caller의 `lseek(fd,0,SEEK_SET)` | seek 전 buffer는 old offset에서 가져온 stale read-ahead | **필요** | `blr_reader_reset` 후 다시 `"repeat\n"` | reset-after-external-seek |
| close 후 fd number reuse | caller `close(first)` 후 `dup2(replacement, first)` | old 문맥은 destroy해야 하며 새 stream에는 새 context 필요 | old context reset보다 destroy/new create가 사용됨 | new 문맥에서 `"new\n"` | reused-descriptor test |
| duplicated descriptor alias | `dup`가 same open file description/offset을 공유 | test는 surviving alias 하나에 context 하나를 붙임 | 해당 sequence에서는 아니요 | original close 뒤 alias 문맥이 두 line을 순서대로 읽음 | single-context-on-dup-alias |
| destroy before EOF | context read-ahead가 있을 수 있으나 `blr_reader_destroy`가 폐기 | library buffer 없음; fd 자체는 open | 새 문맥을 만들거나 caller가 직접 fd 사용 | `fcntl(F_GETFD)>=0`, `lseek` 후 새 문맥으로 first line | cancel-without-closing |

**실제 assertion 범위**

- repeated EOF는 `BLR_EOF`가 두 번 반환되고 output이 `NULL`임을 확인합니다. read counter는 없으므로 “positive data read가 재실행되지 않습니다”는 것은 production code inspection으로 보완했습니다.
- destroy borrowing은 `fcntl(fd, F_GETFD) >= 0`과 후속 `lseek`/새 context read로 직접 확인합니다.
- reuse test는 `dup2(replacement, first) == first`로 같은 integer를 강제하고 **old 문맥을 destroy한 뒤** new 문맥이 `"new\n"`을 반환하는지 확인합니다.
- dup alias test는 두 문맥이 같은 open file description을 경쟁하는 경우를 만들지 않습니다. original을 닫고 alias 하나에 context 하나를 사용합니다. 따라서 shared-offset hazard 전부를 증명하지는 않습니다.
- returned line은 매 success 뒤 test가 `free`합니다. pointer independence는 production allocation path와 lifecycle 결과를 함께 근거로 판단합니다.
- 해당 suite는 이 환경에서 실행하지 않았습니다.

### 5.5 `a24ad4e49cc4` — `test(failure): 컨텍스트의 line 할당 재시도 검증`

- **Commit:** `a24ad4e49cc4`
- **Subject:** `test(failure): 컨텍스트의 line 할당 재시도 검증`
- **중요도:** **A**
- **태그:** `TEST`, `READER_LIFECYCLE`, `RISK`

#### 원자료에서 확인된 역할

caller-visible line allocation을 newline이 이미 buffered된 경우와 EOF가 unterminated tail을 남긴 경우에 각각 강제로 실패시킵니다. 같은 문맥을 다시 호출했을 때 original line이 skip, truncate, EOF 전환 없이 정확히 반환되어야 하며 temporary storage leak도 없어야 합니다.

#### 해당 SHA에서 확인할 테스트 코드

1. newline-delimited line이 이미 internal buffer에 있는 상태를 만드는 fixture와 fault activation 시점을 찾습니다.
2. EOF tail이 internal buffer에 남은 상태에서 result allocation만 실패시키는 sequence를 찾습니다.
3. 첫 failed call의 expected enum과 output pointer가 무엇인지 확인합니다.
4. fault를 해제한 뒤 같은 문맥을 그대로 재호출하는 코드와 exact expected bytes를 기록합니다.
5. retry call 전 context reset/recreate가 없음을 확인합니다.
6. failed attempt 뒤 allocation/release ledger가 leak 또는 invalid free 없이 정리되는 assertion을 찾습니다.
7. 두 scenario가 production extraction의 서로 다른 branch를 통과하는지 call path를 비교합니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | result allocation failure는 buffered input에 대해 non-consuming이며 allocation/copy success 뒤에만 `begin`을 commit해야 합니다. |
| **Failure / boundary** | already-buffered newline extraction과 reached-EOF unterminated tail extraction을 별도 scenario로 다룹니다. |
| **Test technique** | `fault_allocation_fail_at(n)`로 exact allocation attempt를 `NULL`로 만들고 failure를 해제한 뒤 same 문맥을 재호출합니다. |
| **Production path** | `find_line_end` 또는 EOF-tail branch → `extract_line` malloc failure → `scan=begin`, ERROR/NULL → same context retry → malloc/copy/commit → exact line입니다. |
| **증명하는 것** | newline `"\n"`와 tail `"tail"`이 실패 뒤 한 번 정확히 반환되고, skip/truncate/early EOF·leak·invalid/이중 해제가 없음을 확인합니다. |
| **증명하지 않는 것** | read failure recovery, EINTR/EAGAIN, thread safety는 다루지 않습니다. |
| **분류** | transactional extraction commit point를 고정하는 narrow deterministic regression입니다. |
| **막는 회귀** | pre-allocation `begin` advance, EOF-tail clear-before-copy, scan cursor loss는 retry result 또는 allocation ledger assertion을 실패시킵니다. |

#### Failure → retry state trace

| Scenario | failed call 전 `begin/scan/end/EOF` | failure return 후 state | retry result | successful commit 후 state |
| --- | --- | --- | --- | --- |
| buffered newline | `0/1/1/0` — delimiter를 찾아 `scan`이 newline 다음 | `begin=0`, `scan=0`, `end=1`, EOF 0; `BLR_ERROR`, line NULL | same 문맥에서 `BLR_LINE`, `"\n"` | `begin=scan=end=1`, EOF 0; 다음 data read에서 EOF 확인 |
| EOF tail | `0/end/end/1` — EOF를 이미 보고 tail이 unread | `begin=0`, `scan=0`, `end` 유지, EOF 1; `BLR_ERROR`, line NULL | same 문맥에서 `BLR_LINE`, exact `"tail"` | `begin=scan=end`, EOF 1; 다음 call `BLR_EOF` |

**Fault activation과 범위**

- newline case는 input `"\n"`의 context 생성과 buffer 확보 뒤 caller-line allocation이 되는 특정 attempt를 실패시키고, 즉시 ERROR/NULL과 retry success를 확인합니다.
- tail helper는 기준 상태 allocation count를 얻은 뒤 index 2부터 각 allocation attempt를 실패시키는 loop를 사용합니다. 따라서 commit subject와 Source 역할의 중심은 final line allocation retry이지만, 실제 loop에는 buffer growth allocation 실패 지점도 포함될 수 있습니다. final line allocation이 실패하는 index에서는 위 non-consuming tail trace가 검증됩니다.
- retry 전 `blr_reader_reset`, destroy, create를 호출하지 않습니다. fault만 disable하고 같은 handle을 넘깁니다.
- scenario 종료 후 live allocation, invalid free, 이중 해제가 0인지 확인합니다.
- 실행 명령은 Makefile의 `failure-test` 계열이지만 이 환경에서는 실행하지 않았습니다.

## 6. 불변 조건 ledger

| 불변 조건 | 최초/강화 commit | 부족함 또는 위험 | 고정한 test | 학습자가 남길 코드 근거 |
| --- | --- | --- | --- | --- |
| context는 own heap/buffer를 관리하고 descriptor는 borrow합니다. | `903768a43bf4` | destroy/reset이 fd를 close할 위험 | `249093ba477a` | lifecycle code에 close 없음; destroy 후 F_GETFD/lseek/new reader. |
| caller가 reset/destroy로 reader lifetime을 제어할 수 있습니다. | `903768a43bf4` | external seek 또는 abandon 뒤 hidden state mismatch | `249093ba477a` | reset frees buffer/indices; seek+reset과 destroy-before-EOF tests. |
| data result와 EOF/error status를 구분합니다. | `2e681112b304` | `char *`/`NULL` ambiguity | `249093ba477a` | enum declarations와 ordered LINE/EOF/error cases. |
| non-line result는 output pointer를 `NULL`로 둡니다. | `2e681112b304` | stale caller pointer 오해/잘못된 free | `249093ba477a` | entry output clear와 deliberately seeded pointer assertions. |
| EOF는 context에 기록되어 repeated call에서 stable terminal입니다. | `2e681112b304` | repeated read 또는 accidental one-shot completion | `249093ba477a` | `reached_eof`, EOF twice; zero-length probe nuance 기록. |
| explicit API와 legacy API는 one authoritative engine을 사용합니다. | `9bd6ebf429e2` | duplicated parser divergence | explicit/legacy 관련 test를 해당 SHA에서 연결 | `get_next_line`의 `blr_reader_next` call과 shared `extract_line`. |
| result allocation failure는 input을 소비하지 않습니다. | `9bd6ebf429e2` | cursor advance 후 allocation failure | `a24ad4e49cc4` | 실제 explicit inline origin은 2e; 9bd shared helper의 scan rollback과 same-context retry. |
| read-ahead state는 descriptor의 current stream position과 결합됩니다. | context design의 consequence | external seek, close/reuse, dup alias | `249093ba477a` | lseek+reset, dup2 reuse+new context, surviving alias test. |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 | 실제 failure 또는 위험 | root cause | 수정된 불변 조건/decision | 실제 수정 commit | 회귀 테스트 | 학습자 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| hidden state는 EOF까지 두면 충분함 | caller가 stream을 abandon/reset하거나 fd position을 바꿀 수 없음 | lifetime control 부재 | opaque create/reset/destroy | `903768a43bf4` | `249093ba477a` | lifecycle functions와 seek/reset, destroy/reuse cases. |
| `NULL` 하나면 모든 non-line outcome을 표현 가능 | EOF와 error를 구분할 수 없고 stale output 위험 | data와 status가 같은 return에 겹침 | enum result + output pointer rule + EOF state | `2e681112b304` | `249093ba477a` | enum/entry clear와 empty-vs-error assertions. |
| legacy와 context parser를 따로 유지해도 됨 | behavior와 failure semantics가 diverge할 위험 | duplicated state-transition engine | legacy를 `blr_reader_next` adapter로 축소 | `9bd6ebf429e2` | `249093ba477a` 및 후속 tests | adapter call graph와 shared extractor. |
| interval을 먼저 소비하고 result를 만들 수 있음 | allocation 실패 시 line skip/shorten/early EOF | commit point가 allocation 성공보다 앞섬 | copy 성공 뒤 cursor commit, scan restore | `9bd6ebf429e2` | `a24ad4e49cc4` | same-context newline/tail retry. Explicit inline behavior는 2e부터 존재함. |

## 8. 소유권 / state / responsibility 변화

| 단계 | State owner | Descriptor owner | Reading engine | Result 표현 | Cleanup/cancel |
| --- | --- | --- | --- | --- | --- |
| descriptor-list model | hidden compatibility list/node | caller | legacy path | `char *`/`NULL` | EOF/error 중심 |
| `903768a43bf4` | explicit `t_blr_reader` caller handle + legacy adaptation | caller | context lifecycle 준비 | 기존 reading result 확인 | reset/destroy 가능 |
| `2e681112b304` | explicit context | caller | `blr_reader_next` | enum + output pointer | stable EOF state |
| `9bd6ebf429e2` | explicit context와 hidden context 모두 같은 engine 사용 | caller | one authoritative engine | adapter가 축소 mapping | allocation failure non-consuming |

### 실제 responsibility map

- **Public header가 노출하는 것:** opaque type name, create/reset/destroy/next function signatures, result enum과 borrowed-fd/output 소유권 contract입니다.
- **Opaque implementation이 숨기는 것:** fd value, buffer allocation, `begin/scan/end/capacity`, EOF flag, legacy list link입니다.
- **Caller가 반드시 release할 것:** every `BLR_LINE`의 `*line` allocation과 explicit context handle입니다.
- **Library가 절대 close하지 않는 것:** `blr_reader_create`에 supplied된 descriptor입니다.
- **Reset이 폐기하는 state:** internal bytes, capacity, begin/scan/end, EOF flag이며 context object와 fd association은 유지합니다.
- **Destroy가 폐기하는 resource:** internal allocation과 context object입니다. fd는 남습니다.
- **Adapter가 잃는 result information:** EOF와 ERROR를 모두 `NULL`로 축소합니다. 후속 AGAIN도 `NULL`로 축소되지만 hidden context retention policy로 일부 resumability를 보완합니다.

## 9. 개발 흐름의 최종 상태

원자료 기준으로 이 개발 흐름이 끝났을 때 explicit 문맥은 caller-controlled 수명, borrowed descriptor, `LINE`/`EOF`/`ERROR` result contract와 stable EOF를 제공합니다. `get_next_line`은 같은 engine을 사용하는 compatibility adapter이며, result allocation failure는 buffered input을 소비하지 않아 same-context retry가 가능합니다.

이 Thread 종료 시점에는 후속 `f0055ae5cf19`의 `BLR_AGAIN`을 아직 포함하지 않습니다.

### 최종 상태 설명

- **context 내부 state와 public opacity:** private struct가 fd, owned bytes, unread/scan indices, capacity, EOF flag, optional legacy link를 보관하고 public header는 incomplete type만 노출합니다.
- **create/reset/destroy의 정확한 소유권 변화:** create 성공 시 caller 또는 hidden list가 object owner, reserve 후 문맥이 buffer owner가 됩니다. reset은 buffer만 놓고 object/fd를 유지하며 destroy는 object까지 해제하되 fd는 caller 소유입니다.
- **`blr_reader_next` result별 상태 전이:** LINE은 result allocation 성공 뒤 begin/scan commit, EOF는 empty unread와 reached_eof 유지, ERROR는 output NULL이고 accepted unread를 소비하지 않습니다. 이 Thread 시점엔 AGAIN 없음입니다.
- **legacy adapter mapping과 정보 손실:** hidden 문맥을 lookup/create해 same engine을 부르고 LINE만 pointer로 반환합니다. EOF/ERROR는 모두 NULL이므로 caller는 구분하지 못하며 hidden 문맥은 제거됩니다.
- **transactional extraction commit point:** malloc과 copy/NUL이 성공한 다음에만 `begin=line_end`, `scan=begin`; failure는 `scan=begin`으로 복구합니다.
- **descriptor offset coupling과 reset 규칙:** 문맥의 buffer는 prior kernel offset에서 read-ahead한 bytes를 포함합니다. caller가 lseek하거나 fd integer의 target을 바꾸면 reset 또는 old context destroy/new 문맥이 필요합니다. dup aliases는 offset을 공유하므로 context 사용을 임의로 섞으면 안 됩니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
caller
    → blr_reader_create(fd)
        → context owns [heap object; first reserve부터 internal buffer]
        → context borrows fd
    → blr_reader_next(context, &line)
        → clear output
        → stable EOF check
        → authoritative scan/read/extract engine
            → LINE: caller owns independent line
            → EOF: no line, terminal state retained
            → ERROR: no line, unread state retained by explicit context
    → optional blr_reader_reset(context)
        → discard buffered state, keep fd open
    → blr_reader_destroy(context)
        → release owned state, keep fd open

get_next_line(fd)
    → hidden descriptor → context lookup
    → blr_reader_next
    → map rich result to line or NULL
```

### 해당 SHA symbol로 완성

1. **Context allocation과 initialization:** `blr_reader_create` validates fd, allocates object, sets bytes NULL, indices/capacity/EOF 0, fd copied, next NULL.
2. **Output pointer 초기화:** `blr_reader_next` first clears `*line` when pointer is non-NULL, then validates arguments.
3. **EOF flag 빠른 처리 경로:** buffered scan after entry probe; `reached_eof` with empty unread returns `BLR_EOF`, nonempty invokes tail extraction. Positive-count data read is skipped.
4. **LINE 소유권 transfer:** `extract_line` allocates length+1, copies, NUL-terminates, then commits cursors and stores pointer for caller.
5. **ERROR state preservation/cleanup:** explicit API returns ERROR without destroying context; line allocation resets scan, read/probe errors leave unread logical data. Legacy adapter may destroy hidden context according to result.
6. **Reset after external reposition:** caller performs `lseek`, then `blr_reader_reset` discards read-ahead before next call.
7. **Legacy adapter mapping:** `find_reader/create_legacy_reader` → `blr_reader_next`; LINE returns pointer, other result returns NULL and this Thread's policy discards hidden context.
8. **Allocation failure retry path:** same context, begin/end/EOF unchanged, scan restored to begin; fault disabled; next call rebuilds same result and commits once.

## 11. 학습 완료 자가 점검

- [x] opaque type 선언과 private layout을 구분했습니다.
- [x] 문맥이 descriptor를 소유하지 않는다는 코드와 test 근거가 있습니다.
- [x] reset과 destroy가 각각 어떤 allocation/state를 폐기하는지 설명할 수 있습니다.
- [x] 이 Thread 시점의 enum에 `BLR_AGAIN`을 소급하지 않았습니다.
- [x] non-line result의 output pointer rule을 모든 branch에서 확인했습니다.
- [x] repeated EOF가 positive-count data read 없이 terminal이 되는 코드와 zero-length probe 예외를 찾았습니다.
- [x] legacy parser가 제거되고 authoritative engine으로 연결되는 diff를 확인했습니다.
- [x] newline과 EOF tail의 allocation failure에서 cursor가 유지되는 근거가 있습니다.
- [x] external seek, fd reuse, dup alias를 서로 다른 lifecycle 문제로 설명할 수 있습니다.
- [x] broad context tests와 narrow allocation retry test의 증명 범위를 구분했습니다.

---

# POSIX 일시적 읽기 오류와 복구 규칙

## 1. 개발 흐름 목표

short read, `EINTR`, `EAGAIN`/`EWOULDBLOCK`, terminal I/O error, EOF를 서로 다른 상태 전이로 처리하는 최종 POSIX streaming semantics를 복원합니다. 이미 받아들인 partial bytes가 wait/error 뒤에도 보존되고, explicit API와 compatibility adapter가 각각 표현 가능한 범위 안에서 resume하는지 realistic nonblocking pipe와 scripted fault sequence로 검증합니다.

### 원자료에서 확인된 프로젝트 항목

- **Core technical area:** POSIX descriptor와 system-call semantics—short reads, EOF, invalid descriptors, `EINTR`, `EAGAIN`, `EWOULDBLOCK`, recoverable I/O failure를 담당합니다.
- **Critical 불변 조건:** `BLR_LINE`, `BLR_EOF`, `BLR_AGAIN`, `BLR_ERROR`는 구분되어야 하며 non-line outcome은 output pointer를 `NULL`로 둡니다.
- **Critical 불변 조건:** `EINTR`는 같은 logical operation으로 retry하고, `EAGAIN`/`EWOULDBLOCK`은 partial input을 보존한 채 temporary incompleteness를 노출합니다.
- **Critical 불변 조건:** allocation/read failure가 explicit 문맥의 unread input을 부분 소비하면 안 됩니다.
- **Major engineering difficulty:** allocation failure, short reads, interruption, nonblocking wait, later retry 사이에서 이미 accepted bytes를 잃거나 중복하지 않는 문제입니다.
- **Major engineering difficulty:** `NULL`로 EOF/error/wait를 구분하지 못하는 compatibility API에도 useful resumability를 제공하는 문제입니다.
- **Practical engineering area:** deterministic read scripting과 actual nonblocking pipe를 결합해 OS scheduling에 의존하지 않는 검증을 만드는 문제입니다.

### 원자료에서 확인된 중요성

중요한 progression은 단순 errno mapping이 아닙니다. implementation은 interruption, temporary lack of readiness, terminal failure, EOF를 record boundary로 오해하지 않고 이미 읽은 bytes를 유지해야 합니다. fault harness가 transition을 제어하고, fix가 state policy를 정의하며, 실제 nonblocking test와 ordered adversarial test가 각각 현실적인 동작과 정확한 cursor mutation을 검증합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- positive short read, zero read, `-1/EINTR`, `-1/EAGAIN`, 다른 `-1/error`는 각각 `end`, `scan`, EOF flag에 어떤 영향을 주는가?
- `EINTR` retry는 어떤 wrapper/loop에서 이루어지고 caller에게 왜 보이지 않는가?
- `BLR_AGAIN`은 어떤 조건에서 반환되며 unread fragment와 output pointer는 어떻게 유지됩니까?
- terminal I/O error 뒤에도 explicit 문맥의 accepted bytes가 남는다는 것은 다음 call에서 어떤 path로 resume한다는 뜻입니까?
- compatibility adapter가 `BLR_AGAIN`을 `NULL`로 map하면서도 hidden 문맥을 제거하지 않는 조건은 무엇입니까?
- actual nonblocking pipe test와 scripted fault test가 각각 증명하는 범위는 어떻게 다른가?
- progress와 failure가 번갈아 나올 때 `end`와 `scan`이 duplicate/skip 없이 유지되는지 어떻게 입증하는가?

## 3. 완료 기준

- read wrapper와 parser loop의 result mapping을 실제 코드로 복원했습니다.
- `EINTR`가 retry되고 `EAGAIN`이 `BLR_AGAIN`으로 나오는 정확한 조건을 확인했습니다.
- partial bytes가 `AGAIN`과 terminal error 뒤 어떻게 문맥에 남는지 state trace가 있습니다.
- legacy adapter가 wait 뒤 hidden 문맥을 retain하는 cleanup decision을 설명할 수 있습니다.
- nonblocking pipe fixture의 staged write/close sequence와 expected results를 추적했습니다.
- scripted harness의 ordered entries와 production `end/scan` mutation을 연결했습니다.
- realistic 통합 테스트와 deterministic regression의 증명 범위·비증명 범위를 구분했습니다.

## 4. 커밋 목록
| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 Thread 역할 |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `fd03a831686b` | `test(failure): 메모리 할당과 읽기 실패 처리 검증` | **A** | `TEST`, `POSIX_IO`, `RISK` | deterministic short-read/read-error injection과 allocation ownership tracking 기반을 제공합니다. |
| 2 | `f0055ae5cf19` | `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존` | **S** | `CORE`, `POSIX_IO`, `RISK` | `EINTR` 재시도, `BLR_AGAIN` 도입, transient/terminal failure에서 accepted bytes 보존을 확립합니다. |
| 3 | `f3504f674c73` | `test(reader): 비차단 부분 입력 보존 검증` | **A** | `TEST`, `POSIX_IO`, `EDGE` | staged nonblocking input으로 wait, resume, suffix, EOF tail, legacy compatibility를 검증합니다. |
| 4 | `11033bd85c59` | `test(failure): EINTR·EAGAIN·I/O 오류 순서 검증` | **A** | `TEST`, `POSIX_IO`, `RISK` | progress와 failure가 섞인 scripted sequence에서 cursor와 result taxonomy를 검증합니다. |

## 5. 커밋별 학습 기록
### 5.1 `fd03a831686b` — `test(failure): 메모리 할당과 읽기 실패 처리 검증`

- **Commit:** `fd03a831686b`
- **Subject:** `test(failure): 메모리 할당과 읽기 실패 처리 검증`
- **중요도:** **A**
- **태그:** `TEST`, `POSIX_IO`, `RISK`

#### 원자료에서 확인된 역할 — POSIX transition 관점

allocation/release/read replacement를 통해 short read, partial input 전후 read error, exact allocation failure를 deterministic하게 만듭니다. 이 개발 흐름에서는 node isolation보다 read script가 production loop에 어떤 outcome 순서를 공급하고, positive progress와 negative failure 사이 state를 어떻게 관찰하는지에 집중합니다.

#### 해당 SHA에서 확인할 harness 코드

1. read script entry가 반환값, bytes, errno를 어떤 자료구조로 표현하는지 찾습니다.
2. positive short read가 caller-provided destination에 몇 bytes를 복사하고 어떤 반환값을 주는지 확인합니다.
3. error entry가 partial data를 같은 call에서 함께 반환하는지, 또는 이전 positive entry 뒤 별도 negative call로 표현되는지 구분합니다.
4. script cursor가 각 replacement read마다 어떻게 advance되는지 기록합니다.
5. production read caller가 replacement를 실제 `read`와 같은 contract로 사용하는지 compile/link 경계를 확인합니다.
6. error 발생 전후 `end`, unread bytes, allocation 소유권을 test가 직접 또는 간접으로 어떻게 관찰하는지 찾습니다.
7. short read를 EOF로 오해하지 않는 expected sequence와 assertion을 확인합니다.

#### 테스트 커밋 분석 — fault harness 기반

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | positive short read는 valid progress이며, exact failure transition에서도 selected node의 allocation 소유권과 cleanup이 일관되어야 합니다. |
| **Failure / boundary** | global positive-read cap에 의한 short read, first positive-count read EIO, prior short progress 뒤 third read EIO, allocation failure를 구분합니다. |
| **Test technique** | production object를 `test_read`/`test_malloc`/`test_free`로 macro-substitute하고 target fd, read-call index, maximum read size, allocation attempt index를 설정합니다. |
| **Production path** | zero-length probe는 fault를 우회 → positive-count `test_read` → real read 또는 exact EIO → legacy parser append/scan/cleanup → 다음 call/assertion입니다. |
| **증명하는 것** | read cap 3에서도 complete line/tail이 정확히 반환되고, error 전후 selected state가 당시 cleanup policy대로 해제되며 leak/invalid/이중 해제가 없음을 증명합니다. |
| **증명하지 않는 것** | `EINTR` retry와 `BLR_AGAIN`은 아직 production에 없고, 이 SHA harness도 errno sequence 배열을 제공하지 않습니다. |
| **분류** | 후속 POSIX state-machine test를 위한 deterministic failure infrastructure와 기준 상태 regression입니다. |
| **막는 회귀** | short read를 EOF로 오해하면 expected complete lines가 실패하고, EIO cleanup이 잘못되면 live/invalid/double counters 또는 survivor assertion이 실패합니다. |

**Harness와 기준 상태의 실제 모습**

- `fault_read_limit(3)`은 target fd의 요청 `count`를 3 이하로 줄인 뒤 real `read`를 호출합니다. 따라서 returned bytes는 real descriptor에서 오며 harness가 별도 byte array를 destination에 복사하지 않습니다.
- `fault_read_fail_on(fd, n)`은 target fd의 n번째 positive-count read에서 `errno=EIO`, `-1`을 반환합니다. 한 system call에서 “일부 bytes + error”를 동시에 반환하지 않습니다. partial progress 뒤 error는 이전 positive read entry와 다음 negative call로 분리됩니다.
- target fd의 positive-count call마다 `read_call_count`가 증가합니다. zero-length descriptor probe는 counter/fault에서 제외됩니다.
- 이 SHA에는 `{return, bytes, errno}` 구조체 배열과 script cursor가 없습니다. Source가 표현한 일반 scripted ordering은 후속 `11033bd85c59`에서 errno script 배열로 추가됩니다. 고정 Source 문구는 유지하고 실제 SHA의 제한을 명시합니다.

**실제 기준 상태 scenarios**

- short-read case: `read_limit=3`, source `"short reads still work\nlast"`; complete first line, EOF tail `"last"`, then `NULL`을 기대하고 read calls가 2보다 큰지 확인합니다.
- error-before-progress: first positive-count read를 EIO로 만들어 `NULL`, calls 1, clean allocation ledger를 확인합니다.
- error-after-progress: `read_limit=4`, third read EIO, source `"partial bytes must be discarded"`; 이 SHA의 legacy policy는 selected node와 accepted prefix를 폐기하고 `NULL`을 반환합니다. **accepted bytes preservation은 아직 기준 상태가 아닙니다.**
- cross-descriptor case: left node가 first line 뒤 suffix를 갖고, right fd first read에 EIO를 주입한 뒤 left가 `"left two"`를 반환하는지 확인합니다.

#### 후속 fix를 위한 기준 상태

- 이 SHA production code는 negative read의 errno를 분류하지 않습니다. read error면 selected legacy node를 `discard_reader`하고 `NULL`을 반환합니다.
- `EINTR`, `EAGAIN`, `EWOULDBLOCK`도 모두 같은 terminal 오류 처리에 들어갑니다.
- 따라서 partial fragment 뒤 EAGAIN은 state loss, EINTR은 caller-visible failure, terminal EIO 뒤 explicit retry는 불가능한 baseline입니다. `f0055ae5cf19` diff는 이 cleanup/error taxonomy를 바꿉니다.
- 같은 commit을 descriptor-state Thread에서 다룬 기록과 달리 이곳에서는 read return ordering과 accepted-byte policy만 사용했습니다.
- Makefile의 `failure-test`는 이 환경에서 실행하지 않았습니다.

### 5.2 `f0055ae5cf19` — `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존`

- **Commit:** `f0055ae5cf19`
- **Subject:** `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존`
- **중요도:** **S**
- **태그:** `CORE`, `POSIX_IO`, `RISK`

#### 원자료에서 확인된 Problem

POSIX `read`는 interrupt되거나, future line의 일부를 이미 받은 뒤 temporary no-data를 보고할 수 있습니다. 이를 EOF, cleanup, destructive error로 처리하면 partial input을 잃고 nonblocking stream을 사용할 수 없습니다.

#### 원자료에서 확인된 Decision

low-level read path가 `EINTR`를 내부에서 retry합니다. `EAGAIN`/`EWOULDBLOCK`은 explicit API의 `BLR_AGAIN`이 되며 accumulated bytes를 그대로 보존합니다. 다른 I/O error는 `BLR_ERROR`로 보고하지만 explicit 문맥이 이미 받아들인 unread bytes를 버리지 않습니다. compatibility adapter는 wait를 `NULL`로 축소하되 descriptor 문맥을 retain해 later call이 resume하게 합니다.

#### 원자료에서 확인된 중요성

이 commit은 final POSIX streaming semantics를 확립합니다. scheduling/readiness는 record를 지연할 수 있지만 record boundary를 바꾸거나 partial bytes를 소비할 수 없습니다. result taxonomy가 EOF/error distinction을 넘어 complete POSIX state model이 되는 지점입니다.

#### Fix commit 연결 구조

##### 기존 가정

- 직전 code는 `read_size < 0`을 errno에 관계없이 `BLR_ERROR` 또는 legacy cleanup으로 취급했습니다.
- compatibility adapter는 `LINE` 이외의 result에서 hidden 문맥을 제거하는 정책이었습니다.

##### 실제 failure 또는 위험

- fragment `"par"`를 accepted한 다음 EAGAIN이면 기존 policy가 node를 제거해 later `"tial\n"`과 결합할 prefix를 잃습니다.
- EINTR를 ERROR로 노출하면 caller가 같은 operation을 수동 재개해야 하고 hidden adapter는 문맥을 버릴 수 있습니다.
- short positive read 뒤 terminal EIO에서 explicit 문맥까지 clear하면 later success가 prior bytes와 결합되지 않습니다.

##### Root cause

- syscall negative result를 interruption, temporary readiness, permanent failure로 분류하지 않고 하나의 cleanup branch로 묶었습니다.
- rich result enum에 temporary incompleteness가 없었고 compatibility `NULL`과 cleanup decision을 같은 것으로 취급했습니다.

##### 수정된 불변 조건 / decision

- `EINTR`는 동일 destination/count의 `read`를 내부 반복합니다.
- `EAGAIN`/`EWOULDBLOCK`은 `BLR_AGAIN`이며 unread bytes와 EOF flag를 유지합니다.
- 다른 error는 `BLR_ERROR`이지만 explicit 문맥의 accepted unread bytes를 유지합니다.
- compatibility adapter는 `BLR_AGAIN`을 `NULL`로 반환하되 hidden 문맥은 제거하지 않습니다.

#### 해당 SHA에서 확인할 실제 핵심 코드

1. public result enum에 `BLR_AGAIN`이 추가된 declaration과 enum ordering을 확인합니다.
2. low-level read wrapper 또는 loop가 `errno == EINTR`에서 같은 destination/count로 재호출하는지 확인합니다.
3. `EAGAIN`과 `EWOULDBLOCK`을 portable하게 비교하는 조건을 찾습니다.
4. read wrapper의 internal outcome이 `blr_reader_next`의 `BLR_AGAIN`/`BLR_ERROR`로 mapping되는 caller chain을 작성합니다.
5. positive read에서만 `end`가 실제 byte count만큼 증가하는지 확인합니다.
6. `BLR_AGAIN` return 직전 `begin/scan/end`, EOF flag, output pointer가 어떻게 유지되는지 기록합니다.
7. terminal error return 직전 explicit 문맥의 unread interval을 free/clear하지 않는지 확인합니다.
8. compatibility adapter switch에서 `BLR_AGAIN`이 `NULL`로 map되지만 context/node removal branch로 가지 않는지 확인합니다.
9. descriptor probing에도 `EINTR` retry가 적용되는지 source가 언급한 actual/probe read 경로를 각각 찾습니다.

#### Result taxonomy와 state mutation

| Low-level outcome | retry 여부 | explicit result | `end` 변화 | unread bytes | hidden legacy context | output pointer |
| --- | --- | --- | --- | --- | --- | --- |
| positive short read | retry가 아니라 valid progress; parser loop가 scan 후 더 읽을 수 있음 | newline 발견 시 `BLR_LINE`, 아니면 loop 계속 | 실제 `n`만큼 증가 | append되어 유지 | 유지 | line 전에는 `NULL` |
| zero / EOF | 재시도하지 않음 | unread tail이면 `BLR_LINE`, 없으면 `BLR_EOF` | 변화 없음 | tail은 성공 copy 후 소비 | adapter는 EOF에서 제거; tail LINE이면 suffix 정책에 따라 유지 | tail line 또는 NULL |
| `-1`, `EINTR` | **같은 read 즉시 retry** | caller에게 별도 result 없음 | 변화 없음 | 그대로 | 그대로 | 그대로 NULL |
| `-1`, `EAGAIN` | 즉시 syscall retry하지 않음 | `BLR_AGAIN` | 변화 없음 | 보존 | retained | NULL |
| `-1`, `EWOULDBLOCK` | 즉시 syscall retry하지 않음 | `BLR_AGAIN` | 변화 없음 | 보존 | retained | NULL |
| `-1`, other error | 즉시 retry하지 않음 | `BLR_ERROR` | 변화 없음 | explicit 문맥에서 보존 | adapter는 result!= AGAIN이므로 selected hidden context 제거 | NULL |

#### 코드 근거 기록

| 확인 대상 | 해당 SHA에서 남길 근거 | 학습자가 정리할 결론 |
| --- | --- | --- |
| `BLR_AGAIN` public declaration | result enum에 `BLR_AGAIN = 2` 추가 | LINE/EOF/ERROR와 temporary wait를 구분합니다. |
| `EINTR` retry loop | `read_retrying`: first `read`; `while (read_size < 0 && errno == EINTR) read(...)` | same fd/destination/count가 유지되고 caller에게 interruption이 보이지 않습니다. |
| `EAGAIN`/`EWOULDBLOCK` classification | `errno == EAGAIN` 또는 `errno == EWOULDBLOCK`이면 `BLR_AGAIN` | 두 macro가 같은 값인 platform에서도 논리 OR은 올바르게 동작합니다. |
| positive read의 `end` mutation | `if (read_size <= 0) break; end += read_size; bytes[end]='\0'` | 음수/0에서는 end가 증가하지 않고 actual positive bytes만 accepted됩니다. |
| `BLR_AGAIN` return 전 state preservation | error branch가 free/reset/index clear 없이 return | reserve가 representation을 compact/grow했을 수 있지만 logical unread content와 cursor 의미는 유지됩니다. |
| terminal error 뒤 accepted bytes preservation | other negative branch도 `BLR_ERROR`만 반환 | explicit handle은 살아 있고 next call이 same begin/scan/end에서 이어갑니다. |
| legacy adapter retain-vs-cleanup decision | LINE return; `if (result != BLR_AGAIN) discard_legacy_reader(reader); return NULL` | AGAIN만 hidden 문맥을 유지하고 EOF/ERROR는 historical NULL cleanup policy를 적용합니다. |

**최소 코드 근거**

`f0055ae5cf19`, `get_next_line.c`, low-level wrapper와 adapter:

```c
read_size = read(fd, buffer, count);
while (read_size < 0 && errno == EINTR)
    read_size = read(fd, buffer, count);
```

```c
if (result != BLR_AGAIN)
    discard_legacy_reader(reader);
return (NULL);
```

첫 코드는 interruption을 parser 상태 전이로 만들지 않고, 둘째 코드는 compatibility `NULL`과 state disposal을 분리합니다.

#### 후속 regression 연결

- `f3504f674c73`는 actual nonblocking pipe에서 `"part"` → AGAIN → `"ial\nnext"` → first LINE → AGAIN → writer close → EOF-tail LINE → EOF를 검증합니다.
- `11033bd85c59`는 deterministic errno script로 EINTR → real short read → EAGAIN, 그리고 real short read → EIO → later success를 재현합니다.
- real pipe test는 actual kernel O_NONBLOCK 동작을 다루지만 EINTR/EIO ordering을 강제하지 못합니다. scripted test는 exact ordering/call count를 다루지만 실제 scheduler/readiness를 증명하지 않습니다.

### 5.3 `f3504f674c73` — `test(reader): 비차단 부분 입력 보존 검증`

- **Commit:** `f3504f674c73`
- **Subject:** `test(reader): 비차단 부분 입력 보존 검증`
- **중요도:** **A**
- **태그:** `TEST`, `POSIX_IO`, `EDGE`

#### 원자료에서 확인된 역할

nonblocking pipe에 line을 여러 단계로 공급합니다. 첫 fragment는 `BLR_AGAIN`, 이후 bytes는 첫 line을 완성하고 다음 line의 prefix를 시작하며, 다음 wait는 두 번째 fragment를 보존합니다. writer close 후 remaining bytes가 unterminated tail이 되고 그 다음 stable EOF가 됩니다. compatibility function도 wait를 거쳐 prefix를 보존하는지 확인합니다.

#### 해당 SHA에서 확인할 테스트 코드

1. pipe 생성과 read end를 nonblocking으로 바꾸는 system call/flag를 찾습니다.
2. 각 stage에서 writer가 보내는 exact bytes와 newline 위치를 기록합니다.
3. 첫 fragment 뒤 explicit API expected result가 `BLR_AGAIN`이고 output pointer가 null인지 확인합니다.
4. 다음 write가 first line을 완성하면서 second line prefix도 함께 전달하는지 fixture bytes를 확인합니다.
5. first `BLR_LINE` 뒤 internal suffix가 다음 call에서 `BLR_AGAIN`까지 유지되는 expected sequence를 기록합니다.
6. writer close가 EOF를 만들고 remaining fragment가 EOF tail `BLR_LINE`으로 나온 뒤 stable `BLR_EOF`가 되는 assertion을 찾습니다.
7. compatibility call이 wait에서 `NULL`을 반환한 뒤 later write에서 prefix를 포함한 line을 반환하는 sequence를 확인합니다.
8. reader/pipe descriptor와 returned lines를 test가 정리하는 순서를 확인합니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | readiness boundary는 record boundary가 아니며 `AGAIN`은 `[begin,end)` fragment를 소비·반환·cleanup하지 않아야 합니다. |
| **Failure / boundary** | actual EAGAIN, fragmented delimiter completion, same read의 following suffix, writer-close EOF tail, repeated EOF, legacy NULL wait를 구분합니다. |
| **Test technique** | `pipe` 후 read end `fcntl(F_GETFL/F_SETFL, O_NONBLOCK)`, staged `write`, explicit/legacy calls를 사용하는 realistic POSIX integration/boundary test입니다. |
| **Production path** | nonblocking setup → fragment write → `blr_reader_next`/adapter → `read_retrying` → AGAIN/scan/extract → writer close → read 0/EOF-tail path입니다. |
| **증명하는 것** | actual OS EAGAIN에서 prefix preservation, exact line 결합, suffix retention, close 전 tail 미반환, close 후 tail/EOF, legacy hidden-context retention을 증명합니다. |
| **증명하지 않는 것** | EINTR retry count, arbitrary errno order, progress 뒤 terminal EIO recovery는 다루지 않습니다. |
| **분류** | broad realistic POSIX integration regression입니다. |
| **막는 회귀** | EAGAIN cleanup은 first line/legacy line mismatch, duplicate fragment는 exact strcmp, premature EOF-tail은 stage 3 AGAIN assertion, unstable EOF는 final EOF assertion이 검출합니다. |

#### Stage별 state trace

| Stage | writer action | expected explicit result | returned bytes | context unread bytes | EOF state | legacy result |
| ---: | --- | --- | --- | --- | --- | --- |
| 1 | `write("part", 4)` | `BLR_AGAIN` | NULL | `"part"` (`begin=0`, `scan=end=4`) | 0 | 별도 legacy scenario에서는 `write("leg")` 뒤 `NULL`, hidden context retained |
| 2 | `write("ial\nnext", 8)` | `BLR_LINE` | `"partial\n"` | `"next"` suffix | 0 | legacy는 다음 `write("acy\n")` 뒤 `"legacy\n"` |
| 3 | no new data | `BLR_AGAIN` | NULL | `"next"` 그대로 | 0 | explicit scenario만 확인 |
| 4 | writer close | EOF-tail `BLR_LINE` | `"next"` | empty | 1 | legacy writer close 뒤 `NULL`로 EOF mapping/cleanup |
| 5 | repeated call | `BLR_EOF` | NULL | empty | terminal | legacy hidden 문맥은 이미 EOF NULL에서 제거됨 |

**Exact setup과 cleanup**

- helper는 `pipe(fds)`, `fcntl(read_fd, F_GETFL)`, `fcntl(read_fd, F_SETFL, flags | O_NONBLOCK)` 순으로 설정합니다.
- explicit scenario의 first reader call은 writer가 열려 있고 newline 없는 `"part"`만 있으므로 `read`가 fragment를 받은 뒤 다음 read에서 EAGAIN을 보고 tail을 반환하지 않습니다.
- stage 2의 `"ial\nnext"`는 first delimiter와 next record prefix를 한 번에 전달해 first LINE 뒤 suffix가 남는지 동시에 확인합니다.
- returned line은 각 success 뒤 free하고 explicit 문맥을 destroy합니다. writer/read end는 test가 직접 close합니다. library가 fd를 닫는다는 가정은 하지 않습니다.
- suite는 이 환경에서 실행하지 않았습니다.

### 5.4 `11033bd85c59` — `test(failure): EINTR·EAGAIN·I/O 오류 순서 검증`

- **Commit:** `11033bd85c59`
- **Subject:** `test(failure): EINTR·EAGAIN·I/O 오류 순서 검증`
- **중요도:** **A**
- **태그:** `TEST`, `POSIX_IO`, `RISK`

#### 원자료에서 확인된 역할

scripted read harness가 isolated fault가 아니라 ordered sequence를 공급합니다. interruption 뒤 short read와 `EAGAIN`, short read 뒤 terminal I/O error와 later successful call을 다룹니다. interruption은 caller에게 보이지 않고, temporary wait는 data loss 없이 보고되며, terminal error도 이미 accepted된 bytes를 지우지 않아야 합니다.

#### 해당 SHA에서 확인할 테스트 코드

1. `EINTR` → positive short read → `EAGAIN` script의 exact entries와 expected public results를 기록합니다.
2. low-level replacement read call count로 `EINTR`가 내부 retry되었음을 확인하는 assertion을 찾습니다.
3. short read bytes가 `end`에 반영된 뒤 `EAGAIN` result가 나오는 production path를 추적합니다.
4. short read → terminal error → later success script에서 error call 후 같은 문맥을 재호출하는지 확인합니다.
5. later success가 prior accepted bytes와 new bytes를 정확히 결합한 line을 반환하는 expected value를 기록합니다.
6. scan cursor가 error/again 뒤 possible delimiter를 skip하거나 같은 bytes를 duplicate하지 않는지 검출하는 fixture를 찾습니다.
7. 각 sequence 종료 시 allocation/free ledger와 unread state를 확인하는 assertion을 기록합니다.

#### 테스트 커밋 분석

| 구분 | 해당 SHA에서 기록할 내용 |
| --- | --- |
| **Production 불변 조건** | `end`는 positive real read에만 증가하고, EINTR는 public transition이 아니며, AGAIN/ERROR 뒤 accepted `[begin,end)`와 `scan`이 retry 가능한 상태로 남아야 합니다. |
| **Failure / boundary** | script A `EINTR → success(short) → EAGAIN`, script B `success(short) → EIO → later success`입니다. |
| **Test technique** | target fd의 각 positive-count replacement read에 errno script entry를 순서대로 적용하고 zero entry는 read-limit이 적용된 real read를 수행합니다. exact call/result/line과 allocation ledger를 확인합니다. |
| **Production path** | `fault_read_script` → `test_read` → `read_retrying`의 EINTR loop → parser end/scan mutation → public result → same-context next call입니다. |
| **증명하는 것** | interruption invisibility/call count, `BLR_AGAIN` fragment preservation, terminal `BLR_ERROR` 뒤 same-context recovery, exact no-skip/no-duplicate line을 증명합니다. |
| **증명하지 않는 것** | actual kernel O_NONBLOCK readiness와 scheduling은 scripted replacement가 아니라 `f3504f674c73`이 보완합니다. |
| **분류** | ordered state-machine transition을 고정하는 narrow deterministic regression입니다. |
| **막는 회귀** | EINTR 노출, EAGAIN cleanup, EIO에서 accepted bytes erase, end 증가 오류, scan reset/skip은 expected public result와 exact final strings를 실패시킵니다. |

**Ordered script mechanism**

- `fault_read_script(fd, errors, length)`는 최대 64개의 **errno integer 배열**을 복사합니다. 각 target positive-count read에서 next entry를 소비합니다.
- entry가 nonzero면 그 errno를 설정하고 `-1`; entry가 0이면 configured `fault_read_limit`으로 count를 줄인 뒤 real `read`를 호출합니다.
- 따라서 script가 successful bytes 자체를 저장하지는 않습니다. bytes는 실제 fixture fd에서 오고 script는 success/error 순서만 정합니다.
- script cursor는 target read call마다 한 칸 전진하며 EINTR retry도 replacement read를 다시 호출하므로 다음 entry를 소비합니다.

#### Ordered transition trace

| Sequence | Low-level event | read wrapper 동작 | public result | `begin/scan/end` 변화 | 다음 call의 시작 state |
| --- | --- | --- | --- | --- | --- |
| A | `EINTR` | 같은 destination/count로 즉시 replacement read 재호출 | caller-visible result 없음 | `0/0/0` 유지 | same logical read 내부 |
| A | short read | script 0 → limit 3 real read로 `"par"` | parser loop/scan 계속 | `begin=0`, `end=3`, scan `0→3` | same public call에서 다음 read |
| A | `EAGAIN` | wrapper가 EINTR가 아니므로 `-1` 반환; engine이 AGAIN 분류 | `BLR_AGAIN`, line NULL | `0/3/3` 유지 | same 문맥에 `"par"` 보존 |
| A retry | later real short reads | `"tia"`, 이어 `"l\nl"` 등 limit 3 chunks | first `BLR_LINE` | newline exclusive end 8에서 `begin=scan=8`, 당시 `end=9` | suffix `"l"`과 remaining source에서 `"last"` 완성 |
| B | short read | script 0 → `"rec"` 3 bytes | loop 계속 | `0/3/3` | 같은 public call next read |
| B | terminal `EIO` | retry하지 않고 engine에 negative 전달 | `BLR_ERROR`, line NULL | `0/3/3` 유지 | same 문맥에 `"rec"` 보존 |
| B | later success | script exhausted, real reads continue | `BLR_LINE` | prior prefix와 new bytes scan; success 뒤 begin/scan commit | exact `"recoverable\n"`, then EOF |

**실제 assertions**

- A는 `errors = {EINTR, 0, EAGAIN}`, read limit 3, source `"partial\nlast"`를 사용합니다. first public result는 `BLR_AGAIN`, output NULL, `fault_read_calls()==3`로 EINTR가 내부 retry됐음을 고정합니다. 이후 exact `"partial\n"`, `"last"`, EOF를 확인합니다.
- B는 `errors = {0, EIO}`, limit 3, source `"recoverable\n"`를 사용합니다. 첫 call ERROR/NULL 후 reset/recreate 없이 같은 문맥을 재호출해 full line과 EOF를 확인합니다.
- `scan`을 직접 expose하지 않지만 exact line bytes가 prior+later data를 한 번씩 포함하는지로 skip/duplicate를 간접 관찰합니다.
- destroy 뒤 allocation ledger의 live/invalid/이중 해제 상태가 clean인지 확인합니다.
- 해당 failure binary는 이 환경에서 실행하지 않았습니다.

## 6. 불변 조건 ledger

| 불변 조건 | 기반/도입 commit | fix에서 확정 | 현실적 검증 | deterministic 검증 | 학습자가 남길 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| short read는 EOF가 아니라 valid progress입니다. | `fd03a831686b` harness로 제어 가능 | `f0055ae5cf19` production path 확인 | `f3504f674c73`의 staged input | `11033bd85c59` | positive `read_size`만큼 end 증가, read-limit fixtures의 complete lines. |
| `EINTR`는 같은 logical operation으로 retry합니다. | harness baseline | `f0055ae5cf19` | actual pipe test는 직접 `EINTR`를 보장하지 않음 | `11033bd85c59` | `read_retrying` loop와 script A calls 3/first AGAIN. |
| `EAGAIN`/`EWOULDBLOCK`은 EOF가 아니라 `BLR_AGAIN`입니다. | harness baseline | `f0055ae5cf19` | `f3504f674c73` | `11033bd85c59` | errno condition, writer-open fragment tests, script A. |
| wait는 accumulated partial input을 소비하거나 cleanup하지 않습니다. | fault harness로 관찰 가능 | `f0055ae5cf19` | `f3504f674c73` | `11033bd85c59` | AGAIN branch no mutation/free; `part`+`ial`, `par`+later exact results. |
| terminal I/O error도 explicit context의 accepted bytes를 지우지 않습니다. | `fd03a831686b` | `f0055ae5cf19` | 실제 pipe test 범위 확인 | `11033bd85c59` | ERROR branch no clear; `rec` + later bytes = `recoverable\n`. |
| compatibility adapter는 wait를 `NULL`로 map해도 hidden context를 보존합니다. | adapter architecture | `f0055ae5cf19` | `f3504f674c73` | 필요 시 scripted legacy case 확인 | `result != BLR_AGAIN`일 때만 discard; `leg`+`acy\n` test. |
| `end`는 positive read byte 수만큼만 증가합니다. | buffer state model | `f0055ae5cf19` | staged behavior | `11033bd85c59` | `read_size <=0` break 이후에만 `end += read_size`; state traces. |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 | 실제 failure 또는 위험 | root cause | 수정된 불변 조건/decision | 실제 수정 코드 확인 | 회귀 테스트 |
| --- | --- | --- | --- | --- | --- |
| negative read는 모두 terminal error | `EINTR`가 logical read를 불필요하게 중단 | errno category 미분리 | `EINTR` internal retry | `f0055ae5cf19`의 `read_retrying` | `11033bd85c59` |
| no data면 EOF/cleanup | partial fragment 뒤 `EAGAIN`에서 data loss | readiness와 stream completion 혼동 | `BLR_AGAIN` + state preservation | `f0055ae5cf19`의 errno mapping/return | `f3504f674c73`, `11033bd85c59` |
| error면 buffered bytes도 폐기 | prior positive read 뒤 terminal error에서 prefix 손실 | accepted data와 current syscall failure를 한 상태로 처리 | explicit context unread bytes retained | `f0055ae5cf19` error branch | `11033bd85c59` |
| compatibility `NULL`이면 hidden state 제거 | wait 뒤 later call이 prefix 없이 재개 | API 표현 한계와 cleanup policy 혼동 | `BLR_AGAIN` mapping은 `NULL`, context는 retain | `f0055ae5cf19` adapter branch | `f3504f674c73` |

### Fix commit 실제 코드 기록

- **직전 behavior:** negative read는 errno와 무관하게 selected legacy state discard 또는 explicit ERROR로 끝났고 AGAIN enum이 없었습니다.
- **failure를 유발하는 concrete sequence:** `"part"` accepted → EAGAIN → state discard → later `"ial\n"`만 반환하거나 line 손실; `EINTR` 한 번으로 caller-visible error; `"rec"` accepted → EIO → prefix loss입니다.
- **errno classification 함수/조건:** `read_retrying`의 EINTR loop, `blr_reader_next`의 `errno == EAGAIN || errno == EWOULDBLOCK` branch입니다.
- **state mutation 전후 순서:** reserve → read; only positive result commits `end` and sentinel → scan. negative wait/error는 indices/EOF를 commit하지 않습니다.
- **cleanup을 하지 않는 branch:** explicit AGAIN/ERROR returns, legacy adapter의 AGAIN mapping입니다.
- **explicit result mapping:** wait는 `BLR_AGAIN`, other negative는 `BLR_ERROR`, line/EOF 기존 taxonomy 유지입니다.
- **legacy mapping:** LINE은 pointer, AGAIN은 NULL+retain, EOF/ERROR는 NULL+discard입니다.
- **후속 test assertion:** actual `"partial\n"`/`"next"`, legacy `"legacy\n"`, script A call count/result, script B `"recoverable\n"`입니다.

## 8. 소유권 / state / responsibility 변화

이 개발 흐름의 핵심은 allocation owner 변경보다 read outcome을 해석하는 책임의 분리입니다.

| 책임 | fix 전 실제 위치 | `f0055ae5cf19` 이후 위치 | 보존해야 하는 state | caller가 보는 결과 |
| --- | --- | --- | --- | --- |
| raw `read`와 errno 해석 | parser/public path의 direct `read`와 generic negative branch | `read_retrying`이 EINTR 처리, engine이 wait/error 분류 | destination/count와 preexisting accepted bytes | EINTR는 직접 보이지 않음 |
| parser loop progress | direct loop | authoritative `blr_reader_next` | `begin/scan/end`, sentinel, EOF flag | LINE/EOF/AGAIN/ERROR |
| wait state 표현 | 없음 또는 error/NULL로 혼합 | explicit `BLR_AGAIN` | unread fragment | `BLR_AGAIN` |
| legacy compatibility | `NULL` 의미와 cleanup 결합 | rich result 축소 mapping | AGAIN일 때 hidden context retain 여부 | `char *`/`NULL` |
| terminal error 이후 retry | legacy는 selected node 폐기, explicit 기준 상태는 정책 미완성 | explicit context ERROR return without unread clear | accepted unread bytes | `BLR_ERROR`, same handle later call 가능 |

## 9. 개발 흐름의 최종 상태

원자료 기준으로 이 개발 흐름이 끝났을 때 reader는 다음을 구분합니다.

- `EINTR`: same logical read를 내부 retry합니다.
- positive short read: valid progress로 받아들이고 실제 byte 수만큼 state를 늘립니다.
- `EAGAIN`/`EWOULDBLOCK`: `BLR_AGAIN`으로 보고하며 partial bytes를 유지합니다.
- terminal I/O error: `BLR_ERROR`로 보고하되 explicit 문맥이 이미 accepted한 bytes를 지우지 않습니다.
- EOF: remaining unterminated suffix를 line으로 반환한 뒤 stable EOF로 이동합니다.
- legacy adapter: wait를 `NULL`로 표현하지만 hidden 문맥을 유지해 later call이 resume할 수 있습니다.

### 최종 상태 설명

- **raw read outcome taxonomy:** positive n, 0/EOF, -1/EINTR, -1/EAGAIN-or-EWOULDBLOCK, -1/other를 각각 progress, completion, internal retry, temporary wait, explicit error로 해석합니다.
- **각 outcome의 exact state mutation:** positive만 end 증가/sentinel/scan, 0만 reached_eof set, EINTR/AGAIN/ERROR는 current call에서 index·EOF를 바꾸지 않습니다. reserve compaction/growth는 logical unread를 보존합니다.
- **explicit API의 output/result rule:** entry에서 output NULL; complete record only LINE+allocation, terminal empty EOF, wait AGAIN, other failure ERROR입니다. non-line output은 NULL입니다.
- **legacy API의 정보 손실과 보완 정책:** EOF/ERROR/AGAIN 모두 NULL이지만 AGAIN에서는 hidden 문맥을 유지해 later call이 accepted prefix를 이어갑니다.
- **partial fragment가 여러 call을 통과하는 실제 trace:** `part` → AGAIN, `ial\nnext` → `partial\n`, no data → AGAIN with `next`, close → `next`, EOF입니다.
- **real pipe test와 scripted test의 상호 보완:** real pipe는 actual O_NONBLOCK/EOF behavior, script는 EINTR/EIO exact order와 call count/state recovery를 고정합니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
blr_reader_next(context, &line)
    → output = NULL
    → buffered delimiter scan
        → found: transactional line extraction → BLR_LINE
    → need more bytes
        → reserve tail
        → read_retrying
            → EINTR: retry same read
            → positive n: end += n, restore sentinel, scan again
            → EAGAIN/EWOULDBLOCK: preserve state → BLR_AGAIN
            → other error: preserve accepted unread state → BLR_ERROR
            → 0/EOF:
                → unread tail exists: copy tail → BLR_LINE
                → no unread bytes: stable BLR_EOF

get_next_line(fd)
    → hidden context lookup
    → blr_reader_next
        → BLR_LINE: return line
        → BLR_AGAIN: return NULL, retain context
        → EOF/error: return NULL and discard selected hidden context
```

### 실제 symbol과 조건으로 완성

1. **errno retry/classification:** `read_retrying` loops only on EINTR; `blr_reader_next` checks EAGAIN/EWOULDBLOCK after a negative result.
2. **positive read state commit:** `reader->end += (size_t)read_size`, `bytes[end]='\0'`, then `find_line_end`.
3. **`BLR_AGAIN` return 전 state:** output NULL, begin/scan/end logical state unchanged from accepted fragment, reached_eof false, context alive.
4. **terminal error return 전 state:** explicit context remains allocated and accepted bytes remain; output NULL. legacy adapter later discards its hidden context because result is not AGAIN.
5. **EOF와 wait 구분:** read 0 sets `reached_eof=1`; EAGAIN does not. EOF can authorize unterminated tail, wait cannot.
6. **legacy context retention condition:** exactly `result == BLR_AGAIN`; LINE returns line, EOF/ERROR discard.
7. **next call resume point:** `find_line_end` begins from preserved `scan`; if no delimiter, reserve/read appends at preserved `end`.
8. **test harness가 이 흐름을 관찰하는 hook:** `test_read` errno script/call counter/read limit, allocation ledger, public enum/output and exact string assertions입니다.

## 11. 학습 완료 자가 점검

- [x] short read와 EOF를 반환값으로 정확히 구분합니다.
- [x] `EINTR` retry loop가 같은 logical operation을 유지하는 근거가 있습니다.
- [x] `EAGAIN`과 `EWOULDBLOCK`이 `BLR_AGAIN`으로 연결되는 코드를 확인했습니다.
- [x] `BLR_AGAIN`에서 output pointer가 null이고 unread fragment가 남는 것을 확인했습니다.
- [x] terminal error 뒤 explicit 문맥의 accepted bytes가 보존되는 path를 추적했습니다.
- [x] legacy adapter가 wait 뒤 문맥을 제거하지 않는 근거가 있습니다.
- [x] writer close 전에는 unterminated fragment를 line으로 반환하지 않는 이유를 설명할 수 있습니다.
- [x] actual nonblocking pipe test와 scripted error test의 차이를 구분했습니다.
- [x] ordered sequence에서 `end`가 positive read에만 증가함을 확인했습니다.
- [x] `fd03a831686b`를 descriptor-state Thread와 중복 유지하고 여기서는 POSIX transition 관점으로 작성했습니다.

---

# get_next_line 개발 흐름 학습 기록

## 목적

이 문서 세트는 완성된 프로젝트 해설서가 아닙니다. 학습자가 실제 commit history와 각 SHA의 코드를 직접 읽고, 설계 → 구현 → 실패 가능성 → 수정 → 검증의 변화를 근거와 함께 복원하기 위한 기록 골격입니다.

문서에 미리 적힌 commit 역할, 중요도, tags, 순서와 불변 조건 연결은 제공된 source를 그대로 따릅니다. 함수별 동작, 변경 전후 코드 차이, 실제 소유권과 수명, 실패 처리, 테스트 결과, 최종 설명은 학습자가 해당 SHA의 코드로 완성합니다.

## 권장 학습 순서

1. [`01-whole-stream-to-bounded-line-parser.md`](01-whole-stream-to-bounded-line-parser.md)
2. [`02-singleton-to-descriptor-scoped-state.md`](02-singleton-to-descriptor-scoped-state.md)
3. [`03-explicit-reader-lifetime-and-authoritative-engine.md`](03-explicit-reader-lifetime-and-authoritative-engine.md)
4. [`04-posix-transient-read-and-recovery.md`](04-posix-transient-read-and-recovery.md)

각 문서는 source에 정의된 개발 흐름 하나와 정확히 대응합니다. 같은 commit이 여러 Thread에 등장하면 제거하지 않고, 해당 개발 흐름이 요구하는 관점으로 다시 확인합니다.

## Thread 문서 사용법

1. `Commit map`에서 Thread의 순서와 각 commit의 역할을 먼저 확인합니다.
2. 각 commit으로 checkout하거나 해당 SHA의 tree를 직접 열어, 문서에 지정된 상태 필드·함수·caller/callee·failure branch·테스트를 찾습니다.
3. 직전 관련 SHA와 비교해 무엇이 새로 생겼고 무엇이 그대로 유지되었는지 기록합니다.
4. 코드 발췌에는 파일 경로, symbol, 해당 SHA를 함께 남깁니다.
5. commit별 기록이 끝나면 `Invariant ledger`, `Failure → Fix → Test 연결`, `Thread 최종 상태`를 다시 작성합니다.
6. 마지막에는 코드 없이도 Thread의 변화 과정을 순서대로 설명할 수 있는지 자가 점검합니다.

## 해당 SHA 코드 확인 원칙

- 모든 판단은 반드시 문서에 적힌 **해당 SHA 시점의 코드**를 기준으로 합니다.
- 변경 전 상태가 필요하면 immediate parent 또는 문서가 지정한 직전 관련 SHA를 비교합니다.
- commit subject만 읽고 구현을 추정하지 않습니다. 실제 변경 파일, 함수, 상태 mutation, cleanup, 테스트 진입점을 확인합니다.
- source가 파일명이나 symbol을 확정하지 않은 항목은 저장소 tree에서 직접 찾아 기록합니다.
- 코드 확인에 사용할 수 있는 기본 형태는 다음과 같습니다.

```sh
git show <sha> --stat
git show <sha> -- <path>
git diff <previous-related-sha>..<sha> -- <path>
git show <sha>:<path>
```

## 최종 HEAD 소급 사용 금지

최종 HEAD의 함수명, 구조체 배치, helper 분리, 테스트 harness를 과거 commit에 소급해서 설명하지 않습니다. 현재 코드에서 익숙한 symbol을 발견했더라도 해당 SHA에 실제로 존재하는지 먼저 확인합니다. 이후 commit에서 수정된 불변 조건은 이전 commit이 이미 보장했다고 기록하지 않습니다.

## Importance별 학습 깊이

### S

프로젝트를 설명하는 핵심 architecture 또는 불변 조건으로 다룹니다. 문제, 기존 상태, 실패 가능성, 결정, 핵심 코드, 소유권/lifecycle/상태 전이, 후속 fix와 회귀 테스트까지 연결합니다. 코드 근거 없이 요약만 남기면 완료로 보지 않습니다.

### A

주요 subsystem, API/lifecycle boundary, 실패 처리, integration point 또는 강한 검증 근거를 확인합니다. 핵심 함수와 상태 변화, 선택한 설계 판단, 해당 commit이 보장하는 범위를 기록합니다.

### B

Thread의 흐름에서 맡는 준비·지원·검증 역할을 확인합니다. 변경된 helper, build/test 진입점, 필요한 상태 변화와 전후 연결을 중심으로 기록하며 S 수준의 전체 architecture 분석을 반복하지 않습니다.

### C

Thread 이해에 필요한 경우에만 맥락으로 사용합니다. 문서 전용 또는 기계적 변경에 S/A 수준의 분석란을 만들지 않습니다.

## 실제 코드 삽입 기준

- 상태 필드, 핵심 조건문, 소유권 이전, cursor commit, cleanup, error mapping처럼 설명의 근거가 되는 최소 범위만 발췌합니다.
- 발췌마다 `<sha>`, 파일 경로, symbol을 적습니다.
- caller와 callee의 관계가 중요하면 양쪽을 각각 발췌합니다.
- 변경 전후 비교는 같은 책임을 수행하는 코드끼리 나란히 기록합니다.
- 긴 함수 전체, 관련 없는 boilerplate, 최종 HEAD의 대체 코드는 삽입하지 않습니다.
- 코드 아래에는 “무엇을 합니다”뿐 아니라 “어떤 상태를 언제 바꾸며, 실패하면 무엇이 유지되는가”를 작성합니다.

## Test commit 학습 방법

각 test commit에서는 다음을 구분해 기록합니다.

- 대상으로 삼은 production 불변 조건
- 재현하는 failure 또는 boundary
- 실제 descriptor, pipe, fault injection, build matrix, operation counting 등 사용한 test technique
- 테스트가 통과하는 production 코드 경로
- assertion과 expected result가 증명하는 것
- 테스트가 증명하지 않는 것
- broad integration test인지 deterministic regression인지
- 이후 어떤 회귀를 막는지

테스트 이름과 기대값만 옮기지 않습니다. failure가 어느 시점에 주입되고, production state가 그 전후에 어떻게 유지되는지까지 추적합니다.

## 문서 완료 기준

- 네 개발 흐름의 commit 순서를 source와 동일하게 설명할 수 있습니다.
- 각 S commit의 핵심 state representation, decision, failure risk와 후속 검증을 실제 코드로 입증했습니다.
- A/B commit은 importance에 맞는 깊이로 Thread 내 역할과 코드 근거가 채워져 있습니다.
- `Invariant ledger`에서 불변 조건이 도입·강화·위험 노출·검증된 시점을 구분했습니다.
- fix를 기존 가정 → failure/risk → root cause → 수정된 decision → 회귀 테스트로 연결했습니다.
- test commit마다 증명 범위와 비증명 범위를 구분했습니다.
- 소유권, descriptor borrowing, context 수명, state mutation, cleanup 경로를 서로 혼동하지 않습니다.
- 최종 HEAD를 과거 SHA의 근거로 사용한 부분이 없습니다.
- 각 Thread의 최종 execution flow를 코드 없이 순서대로 설명할 수 있습니다.
