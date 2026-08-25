# Thread: 검증 경계 — 출력 바이트에서 배포 아카이브와 sanitizer까지

## 개요

`ft_printf`가 “테스트를 통과한다”는 말은 하나의 사실이 아닙니다. 이 history는 서로 다른 결함을 서로 다른 관찰 방법으로 확인합니다.

| 검증 계층 | 묻는 질문 |
| --- | --- |
| 기능·차등 비교 | 반환값과 실제 출력 바이트가 기대 결과와 같은가? |
| 결정적 output fault | 부분 쓰기·`EINTR`·0 진행·영구 실패 순서를 production 상태 머신이 올바르게 처리하는가? |
| 고정된 프로젝트 규칙 | libc가 이식 가능한 oracle이 아닌 경우 repository가 선택한 결과가 유지되는가? |
| release artifact | 실제 `.a`의 member·symbol·외부 의존성과 public-header-only consumer가 기대한 형태인가? |
| sanitizer runtime | 실행된 production/fault 경로에서 ASan/UBSan이 메모리 오류나 UB를 보고하는가? |

이 계층들은 서로 대체할 수 없습니다. Source를 sanitizer로 직접 compile한 테스트가 통과해도 archive의 member/symbol 구성이 맞다는 뜻은 아니고, out-of-tree consumer가 link된다고 해서 `EINTR` 재시도가 맞다는 뜻도 아닙니다.

| SHA | 제목 | 중요도 | 태그 | 역할 |
| --- | --- | :---: | --- | --- |
| `1b8049e411bb` | test(printf): 기본 변환과 포맷 경계 검증 | A | `FORMAT, TEST, VERIFY` | stdout capture, return-count 비교, libc differential 기반 구축 |
| `1223518652bd` | test(output): 쓰기 실패 시퀀스와 채움 전략 검증 | A | `OUTPUT, TEST, RISK` | scripted syscall result와 실제 `SIGPIPE` policy 검증 |
| `12d715eba77d` | test(printf): 공개 계약 경계 사례 확대 | A | `FORMAT, TEST, EDGE` | libc에 맡길 수 없는 프로젝트 고유 결과를 fixed expectation으로 기록 |
| `a87bcf560789` | test(release): 아카이브와 외부 소비자 검증 | A | `RELEASE, ARCH, VERIFY` | archive members, global definitions, external dependencies, out-of-tree consumer 확인 |
| `1b474fa2a5e3` | build(sanitize): UBSan과 Linux ASan 검증 추가 | B | `VERIFY, TEST` | normal/fault source를 UBSan과 Linux GCC ASan으로 실행하는 target 추가 |

## `1b8049e411bb` — visible contract를 raw bytes로 관찰

**중요도** A · **태그** `FORMAT, TEST, VERIFY`

### stdout capture

테스트는 `pipe`, `dup`, `dup2`로 stdout을 임시 pipe에 연결합니다.

```c
typedef struct s_capture
{
	int	saved_stdout;
	int	pipe_fd[2];
}	t_capture;
```

`capture_begin`이 원래 stdout을 저장하고 pipe write end를 1번 FD에 복제한 뒤, `ft_printf`를 호출합니다. `capture_end`는 stdout을 복원하고 pipe read end를 EOF까지 읽습니다. 따라서 테스트는 C string으로 보이는 값뿐 아니라 실제 byte count를 얻습니다.

```c
if (actual_ret != expected_ret || actual_len != expected_ret
	|| memcmp(expected, actual, (size_t)expected_ret) != 0)
	fail_test(...);
```

이 세 비교는 서로 다른 회귀를 잡습니다.

- bytes는 맞지만 반환 count가 틀린 경우
- 반환 count는 맞지만 실제로 덜/더 출력한 경우
- embedded NUL 뒤의 byte가 달라 C string 비교로는 놓치는 경우

### 차등 oracle와 fixed expectation의 분리

Portable behavior로 취급하는 case는 `snprintf`가 같은 format과 argument로 만든 expected bytes/return을 사용합니다.

```c
expected_ret = snprintf(expected, sizeof(expected), FORMAT, ...);
actual_ret = ft_printf(FORMAT, ...);
```

반면 null pointer/string처럼 플랫폼별 libc representation이 달라질 수 있거나 repository가 명시적으로 다른 규칙을 택한 case는 `EXPECT_OUTPUT`으로 expected bytes를 직접 적습니다. 이 구분은 이후 public-contract test가 확장되는 기반입니다.

초기 suite는 literal, `%%`, embedded-NUL `%c`, string/null string, signed extrema, unsigned/hex, pointer, width/alignment/precision/flags, parser field overflow를 통과합니다. Production source는 이 commit에서 바뀌지 않습니다.

### 증명 범위

이 harness는 실제 public `ft_printf` 전체 경로를 link해 bytes와 count를 검증합니다. 그러나 raw `write`를 제어하지 않으므로 정확히 첫 호출이 `EINTR`, 두 번째가 short write 같은 rare sequence는 운영체제 타이밍에 맡겨집니다. Late invalid field의 whole-call preflight, archive symbol 구성, sanitizer 진단도 아직 범위 밖입니다.

## `1223518652bd` — 시스템 호출 결과를 script로 만들기

**중요도** A · **태그** `OUTPUT, TEST, RISK`

### Production writer를 그대로 통과하는 test seam

Makefile은 `FT_PRINTF_TEST_WRITE`를 정의한 별도 fault binary를 production source와 함께 compile합니다. 이 macro 아래에서 `ft_printf_write`가 호출하는 함수만 test writer로 치환되고, 결과를 해석하는 loop는 동일합니다.

```c
typedef enum e_write_action
{
	WRITE_ALL,
	WRITE_PART,
	WRITE_EINTR,
	WRITE_EPIPE,
	WRITE_ZERO
}	t_write_action;
```

Scripted writer는 호출 수, 가장 큰 요청 길이, 실제로 받아들인 bytes를 기록합니다. 각 action은 production loop에 다음 상태를 만듭니다.

| action | 반환/errno | 관찰 대상 |
| --- | --- | --- |
| `WRITE_ALL` | 요청 길이 전부 | 정상 완료 |
| `WRITE_PART(n)` | 양의 n | pointer·remaining·count 전진 |
| `WRITE_EINTR` | -1 / `EINTR` | 상태 변경 없는 재시도 |
| `WRITE_EPIPE` | -1 / `EPIPE` | sticky error와 `-1` |
| `WRITE_ZERO` | 0 | 무한 루프 방지를 위한 실패 처리 |

### 순서가 중요한 regression

```text
PART(2) → ALL
EINTR → PART(3) → EINTR → ALL
PART(3) → EPIPE
```

첫 두 sequence는 성공 뒤 exact bytes, return, write call count, largest request를 검사합니다. 세 번째는 public return이 `-1`이더라도 writer가 이미 받아들인 앞 3바이트 `"par"`가 남는다는 non-atomic runtime contract를 확인합니다.

Padding case는 `%1000d`에서 총 1000바이트, 17회 호출, 최대 요청 64바이트를 요구합니다. 이 assertion은 `ft_printf_putnchar`가 문자별 writer 호출로 회귀하지 않고 64바이트 bounded chunk를 사용한다는 구현 전략까지 고정합니다.

### Scripted `EPIPE`와 실제 `SIGPIPE`는 다른 검증

Mock writer가 `EPIPE`를 반환하는 것만으로는 process signal policy를 확인할 수 없습니다. 그래서 normal archive-linked suite는 읽기 끝이 닫힌 실제 pipe에 stdout을 연결하고, caller가 설치한 `SIGPIPE` handler를 사용합니다.

테스트는 다음을 요구합니다.

- handler가 한 번 실행됨
- 살아남은 `write`가 실패해 `ft_printf`가 `-1`을 반환함
- 호출 뒤에도 caller의 handler가 그대로 설치됨

이는 library가 process-wide `SIGPIPE` disposition을 변경하지 않는다는 검증입니다. 기본 disposition인 프로세스가 실제로 종료되지 않는다는 보장은 하지 않습니다. 그 정책은 caller/process가 소유합니다.

## `12d715eba77d` — libc 비교와 프로젝트 고유 계약을 명시적으로 분리

**중요도** A · **태그** `FORMAT, TEST, EDGE`

이 커밋은 signed/unsigned/hex의 boundary matrix를 `EXPECT_PRINTF`로 넓히는 한편, portable libc oracle에 맡기지 않을 결과를 `EXPECT_OUTPUT`으로 고정합니다.

대표적인 fixed expectation은 다음과 같습니다.

```c
EXPECT_OUTPUT("0x", "%.0p", (void *)0);
EXPECT_OUTPUT("      0x", "%8.0p", (void *)0);
EXPECT_OUTPUT("  (null)", "%8s", (char *)0);
EXPECT_OUTPUT("", "%.0s", (char *)0);
EXPECT_OUTPUT("0000%|%    |%", "%05%|%-5%|%.%");
```

여기에는 세 종류의 repository 결정이 섞여 있습니다.

1. null pointer를 `0x` 계열로 표현
2. null string을 `(null)`로 대체한 뒤 width/precision 적용
3. formatted percent에 width, LEFT, ZERO를 적용

이 값들을 host libc 결과와 비교하면 플랫폼에 따라 테스트 자체가 불안정하거나 프로젝트 extension을 잘못된 것으로 판정할 수 있습니다. Fixed expectation은 “표준과 같다”가 아니라 “이 repository가 선택한 public contract가 이것이다”를 기록합니다.

이 commit의 숫자 행렬은 많은 경계 조합을 늘리지만 exhaustive proof는 아닙니다. 또한 fixed output이 합리적인 specification인지 평가하는 테스트가 아니라, 이미 선택한 결과의 회귀를 막는 테스트입니다.

## `a87bcf560789` — source tree 밖의 실제 archive를 검사

**중요도** A · **태그** `RELEASE, ARCH, VERIFY`

`release-check`는 `libftprintf.a`를 단순히 link해보는 데서 끝나지 않습니다.

### 1. archive member 목록

`ar t` 결과에서 Darwin index member를 제외한 뒤 exact object 목록과 순서를 비교합니다.

```text
ft_printf.o
ft_output.o
ft_parse.o
ft_measure.o
ft_dispatch.o
ft_text.o
ft_numeric_layout.o
ft_number.o
ft_hex.o
```

Source file이 Makefile에서 빠지거나 의도하지 않은 object가 archive에 들어가면 consumer test가 우연히 link되더라도 여기서 실패합니다.

### 2. global definition과 unresolved dependency

`nm -g` 결과를 Darwin/Linux 차이에 맞게 정규화하고, archive가 정의하는 global symbol 집합을 exact 목록과 비교합니다. 이 목록에는 public `ft_printf`뿐 아니라 repository가 global definition으로 빌드한 내부 함수들도 포함됩니다.

정의되지 않은 symbol 중 archive 내부 정의로 해결되지 않는 외부 의존성도 별도로 계산합니다.

- Linux: `__errno_location`, `write`
- Darwin: `__error`, `write`, compiler에 따라 stack-check symbol

따라서 source에 새 libc 호출이 들어가거나 platform/compiler 가정을 벗어나면 명시적으로 드러납니다. 이 check는 지원 대상을 Linux/Darwin과 인식 가능한 compiler 조합으로 제한하며, 알 수 없는 platform에서 조용히 통과하지 않습니다.

### 3. public-header-only out-of-tree consumer

Script는 임시 디렉터리에 다음 세 파일만 복사합니다.

- `include/ft_printf.h`
- `libftprintf.a`
- `tests/test_consumer.c`

그 위치에서 consumer를 compile하고 실행합니다.

```c
#include "ft_printf.h"

int	main(void)
{
	if (ft_printf("consumer:%d:%s\n", 17, "ok") != 15)
		return (1);
	return (0);
}
```

Shell command substitution 결과가 `consumer:17:ok`인지 확인하고, 임시 디렉터리 cleanup까지 검사합니다. 이는 repository 내부 include path나 object file에 우연히 의존하지 않고 배포 artifact와 public header만으로 소비할 수 있음을 겨냥합니다.

### 이 check가 증명하지 않는 것

- ABI가 미래 compiler/version에서도 안정적이라는 장기 보장
- 내부 함수가 global로 노출되는 설계가 이상적인지
- archive를 shared object 또는 다른 linker mode에서 사용할 수 있는지
- runtime formatting의 모든 경계

Release check는 현재 repository가 선언한 artifact shape를 검증합니다.

## `1b474fa2a5e3` — 실행된 source에 instrumentation을 추가

**중요도** B · **태그** `VERIFY, TEST`

Makefile에 normal suite와 output fault suite를 sanitizer flags로 직접 compile하는 target이 추가됩니다.

### UBSan

```make
UBSAN_FLAGS := -g -fno-omit-frame-pointer -fsanitize=undefined

sanitize-undefined:
	$(CC) ... $(UBSAN_FLAGS) tests/test_ft_printf.c $(SRC) ...
	UBSAN_OPTIONS=halt_on_error=1 ./tests/bin/test_ft_printf_ubsan
	$(CC) ... $(UBSAN_FLAGS) -DFT_PRINTF_TEST_WRITE \
		tests/test_output_faults.c $(SRC) ...
	UBSAN_OPTIONS=halt_on_error=1 ./tests/bin/test_output_faults_ubsan
```

Normal conversion 경로뿐 아니라 injected partial/EINTR/error path도 implementation source와 함께 instrument합니다.

### Linux GCC ASan + UBSan

```make
SANITIZER_FLAGS := -g -fno-omit-frame-pointer \
	-fsanitize=address,undefined

sanitize-linux:
	docker run --rm -v "$(CURDIR):/source:ro" gcc:14-bookworm \
		sh -c 'cp -R /source /tmp/format-printer-fix && \
		cd /tmp/format-printer-fix && \
		make sanitize-address SANITIZER_CC=gcc'
```

Read-only mounted source를 container 내부 임시 디렉터리로 복사하고 GCC 14 환경에서 AddressSanitizer와 UndefinedBehaviorSanitizer를 함께 실행합니다. `sanitize-address`도 normal/fault 두 binary를 각각 compile합니다.

`make sanitize`는 UBSan target만 실행하고 Linux ASan은 별도의 `make sanitize-linux`입니다. `make check` 역시 `test`, `release-check`, `sanitize`와 `git diff --check`를 묶지만 Docker ASan까지 자동 포함하지는 않습니다.

### Source instrumentation과 release artifact의 차이

Sanitizer target은 prebuilt `libftprintf.a`를 link하지 않고 `$(SRC)`를 test와 함께 compile합니다. 그래야 implementation 자체에 instrumentation이 들어갑니다. 대신 이 결과는 archive member/symbol/dependency shape를 검증하지 않으므로 release check와 보완 관계입니다.

Sanitizer 성공은 실행된 입력과 경로에서 진단이 없었다는 뜻입니다. 모든 가능한 format, 모든 allocator/OS behavior, 실행되지 않은 branch의 메모리 안전성을 증명하지 않습니다.

## 최종 검증 지도

| 계층 | 실제 production 경로 | 강하게 관찰하는 것 | 관찰하지 못하는 것 |
| --- | --- | --- | --- |
| stdout capture + differential | archive-linked public `ft_printf` | bytes, captured length, public return | rare syscall sequence, artifact symbols |
| fixed expectation | archive-linked public `ft_printf` | repository-specific output rules | 그 규칙의 표준성·이식성 |
| scripted writer | source-built `ft_printf_write` 포함 전체 path | partial/EINTR/zero/permanent failure와 chunk call shape | real kernel timing과 signal delivery |
| real broken pipe | archive-linked public path + OS pipe | caller `SIGPIPE` policy 보존과 write failure | default policy에서 process 생존 |
| release check | built `.a`, public header, external consumer | member/symbol/dependency/link/run artifact | sanitizer 진단과 전체 runtime surface |
| UBSan/ASan | instrumented normal/fault source binary | 실행 경로의 UB·invalid memory access | 미실행 경로와 archive packaging |

## Thread 경계

이 Thread는 검증 방법과 각 방법이 볼 수 있는 경계를 설명합니다. 다음 production decision 자체는 다른 문서가 source of truth입니다.

- 부분 쓰기와 `EINTR` 처리: output-state Thread
- 숫자 prefix/precision/width 순서: numeric layout Thread
- `%s` bounded read: string precision Thread
- late format error의 no-output 보장: whole-call preflight Thread

검사 과정에서는 test/release/sanitizer source와 Makefile diff를 exact SHA에서 확인했습니다. 해당 target을 이 환경에서 직접 실행한 결과를 주장하지 않습니다.
