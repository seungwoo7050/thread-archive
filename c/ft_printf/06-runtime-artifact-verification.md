# Thread: 검증은 출력 동작·실패 시퀀스·배포 아카이브·계측 실행을 분리한다
> Project: `ft_printf` · Branch: `c/ft_printf` · 문서 번호: 06

## 개요

`ft_printf`가 “테스트를 통과한다”는 말은 하나의 보장을 뜻하지 않습니다. Public return과 raw bytes가 맞는지, rare `write` sequence를 상태 머신이 처리하는지, libc가 oracle이 될 수 없는 repository 규칙이 유지되는지, 실제 `.a`가 기대한 member·symbol·dependency를 갖는지, 실행된 source에서 sanitizer 진단이 없는지는 서로 다른 질문입니다.

이 Thread는 하나의 거대한 test suite로 합치지 않고 관찰 경계를 나눕니다. 각 계층이 강하게 보는 대상과 보지 못하는 대상이 다르기 때문에, 하나의 성공을 다른 계층의 증거로 소급하지 않습니다.

| 검증 계층 | 강하게 관찰하는 것 | 단독으로 관찰하지 못하는 것 |
| --- | --- | --- |
| stdout capture + differential | 실제 bytes, captured length, public return | 결정적 syscall 순서, archive shape |
| scripted writer + real broken pipe | partial/`EINTR`/zero/`EPIPE` 상태 전이와 `SIGPIPE` policy | 전체 format surface, artifact symbols |
| fixed expectations | repository가 선택한 null·pointer·percent 결과 | 그 선택의 표준성·이식성 |
| release check | archive member·global symbol·외부 dependency·out-of-tree consumer | sanitizer 진단, 모든 runtime branch |
| UBSan/ASan build | 실행된 normal/fault source 경로의 UB·invalid access | 미실행 경로, prebuilt archive packaging |

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1b8049e411bb` | test(printf): 기본 변환과 포맷 경계 검증 | A | `FORMAT, TEST, VERIFY` | stdout capture, return-count 비교, libc differential 기반 구축 |
| 2 | `1223518652bd` | test(output): 쓰기 실패 시퀀스와 채움 전략 검증 | A | `OUTPUT, TEST, RISK` | scripted syscall result와 실제 `SIGPIPE` policy 검증 |
| 3 | `12d715eba77d` | test(printf): 공개 계약 경계 사례 확대 | A | `FORMAT, TEST, EDGE` | libc에 맡길 수 없는 repository 결과를 fixed expectation으로 기록 |
| 4 | `a87bcf560789` | test(release): 아카이브와 외부 소비자 검증 | A | `RELEASE, ARCH, VERIFY` | archive members, global definitions, external dependencies, out-of-tree consumer 확인 |
| 5 | `1b474fa2a5e3` | build(sanitize): UBSan과 Linux ASan 검증 추가 | B | `VERIFY, TEST` | normal/fault source를 UBSan과 Linux GCC ASan으로 실행하는 target 추가 |

## Public behavior를 바이트와 고정 기대값으로 관찰한다

## 1b8049e411bb — test(printf): 기본 변환과 포맷 경계 검증
**중요도** `A` · **태그** `FORMAT, TEST, VERIFY`

### 왜 다른 기법이 필요한가

일반적인 C string 비교만으로는 embedded NUL 뒤의 bytes나 실제 출력 길이를 확인하기 어렵습니다. 이 commit은 stdout을 pipe에 연결해 `ft_printf`가 기록한 raw bytes를 직접 수집하고, public return과 capture length를 함께 비교합니다.

### 무엇을 추가했는가 (diff)

```diff
+typedef struct s_capture
+{
+	int	saved_stdout;
+	int	pipe_fd[2];
+}	t_capture;
```

```diff
+static void	check_case(int line, const char *format, const char *expected,
+		int expected_ret, const char *actual, ssize_t actual_len,
+		int actual_ret)
+{
+	if (actual_ret != expected_ret || actual_len != expected_ret
+		|| memcmp(expected, actual, (size_t)expected_ret) != 0)
+	{
+		/* mismatch diagnostic 출력 생략 */
+		fail_test(line, "ft_printf output mismatch");
+	}
+}
```

Portable behavior로 취급하는 case는 `snprintf`가 만든 expected bytes와 return을 사용하고, null string이나 pointer처럼 fixed representation이 필요한 case는 `EXPECT_OUTPUT`으로 직접 적습니다. 초기 suite는 literal, `%%`, NUL 문자 `%c`, 문자열, signed/unsigned/hex/pointer, width·precision·flags, parser field overflow를 public entry를 통해 실행합니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

열거된 format surface에서 actual bytes, captured length, return이 기대값과 일치함을 증명합니다. Raw `write`를 제어하지 않으므로 첫 호출 `EINTR`, 다음 호출 partial 같은 rare sequence를 결정적으로 만들 수 없고, late invalid field의 whole-call atomicity나 archive symbol 구성도 확인하지 않습니다.

### 관련 커밋

`1223518652bd`은 같은 production writer의 syscall 결과를 script로 만들어 이 harness가 재현하지 못한 rare transition을 검증합니다. `12d715eba77d`은 differential과 fixed expectation의 구분을 더 넓은 public contract에 적용합니다.

## 1223518652bd — test(output): 쓰기 실패 시퀀스와 채움 전략 검증
**중요도** `A` · **태그** `OUTPUT, TEST, RISK`

### 왜 다른 기법이 필요한가

Kernel timing에 의존하면 short write, `EINTR`, zero progress, `EPIPE`의 호출 순서를 안정적으로 재현하기 어렵습니다. Test build는 `FT_PRINTF_TEST_WRITE`로 호출 target만 교체하고, 결과를 해석하는 `ft_printf_write` loop는 production source를 그대로 사용합니다.

### 어떤 실패 시퀀스를 만드는가 (diff)

```diff
+typedef enum e_write_action
+{
+	WRITE_ALL,
+	WRITE_PART,
+	WRITE_EINTR,
+	WRITE_EPIPE,
+	WRITE_ZERO
+}	t_write_action;
```

```diff
+	reset_writer();
+	add_step(WRITE_PART, 2);
+	add_step(WRITE_ALL, 0);
+	expect_success("partial", 2, 7);
+
+	reset_writer();
+	add_step(WRITE_EINTR, 0);
+	add_step(WRITE_PART, 3);
+	add_step(WRITE_EINTR, 0);
+	add_step(WRITE_ALL, 0);
+	expect_success("interrupt", 4, 9);
```

Failure matrix는 first-call `EPIPE`, partial 3바이트 뒤 `EPIPE`, zero-byte return을 구분합니다. `%1000d` case는 total 1000바이트, writer call 17회, largest request 64바이트를 요구해 padding chunk 전략도 관찰합니다.

Scripted `EPIPE`는 signal delivery를 만들지 않으므로 별도의 archive-linked test가 읽기 끝을 닫은 실제 pipe와 caller-installed `SIGPIPE` handler를 사용합니다. Handler가 한 번 실행되고, `ft_printf`가 `-1`을 반환하며, 호출 뒤 handler가 그대로 설치되어 있어야 합니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

열거된 sequence에서 pointer·remaining·count·sticky error 전이가 맞고, partial failure에서 이미 accepted된 prefix가 남으며, library가 caller의 `SIGPIPE` disposition을 변경하지 않음을 증명합니다. 모든 errno와 real kernel scheduling을 포괄하지 않고, default `SIGPIPE` policy에서 process가 생존함을 보장하지도 않습니다.

### 관련 커밋

이 test는 `8a3ec50cb689`의 retry/request policy와 `22e65c176b5d`의 chunk strategy를 고정합니다. 그 production 결정은 `01-output-state-system-call-boundary.md`가 설명합니다.

## 12d715eba77d — test(printf): 공개 계약 경계 사례 확대
**중요도** `A` · **태그** `FORMAT, TEST, EDGE`

### 무엇을 검증하는가

Signed, unsigned, hex의 boundary matrix는 libc-comparable case로 유지하고, repository가 선택한 null pointer·null string·formatted percent 결과는 fixed expectation으로 분리합니다.

```diff
+	EXPECT_OUTPUT("0x", "%.0p", (void *)0);
+	EXPECT_OUTPUT("      0x", "%8.0p", (void *)0);
+	EXPECT_OUTPUT("  (null)", "%8s", (char *)0);
+	EXPECT_OUTPUT("", "%.0s", (char *)0);
+	EXPECT_OUTPUT("0000%|%    |%", "%05%|%-5%|%.%");
```

Host libc 결과와 무조건 비교하면 platform-dependent pointer/null 표현이나 project extension을 잘못된 것으로 판정할 수 있습니다. Fixed expectation은 “표준과 같다”가 아니라 “이 repository의 public contract는 이것이다”를 기록합니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

열거된 fixed bytes와 numeric matrix가 유지됨을 증명합니다. 그 contract가 모든 플랫폼에서 바람직하거나 표준적으로 유일한 선택임을 평가하지 않으며, 모든 width·precision 조합을 exhaustive하게 검증하지도 않습니다.

### 관련 커밋

`1b8049e411bb`이 만든 `EXPECT_PRINTF`와 `EXPECT_OUTPUT`의 역할 구분을 확대합니다. Numeric layout과 bounded string semantics의 구현 근거는 `03`과 `04` 문서가 각각 다룹니다.

## Source-tree 내부 성공을 배포 artifact와 계측 runtime 경계로 확장한다

## a87bcf560789 — test(release): 아카이브와 외부 소비자 검증
**중요도** `A` · **태그** `RELEASE, ARCH, VERIFY`

### 왜 다른 기법이 필요한가

Source tree 안에서 test binary가 link되고 동작해도 실제 `libftprintf.a`에 object가 빠졌거나, 의도하지 않은 global symbol·외부 dependency가 들어갔거나, public header만 가진 외부 consumer가 link하지 못할 수 있습니다. Release check는 runtime formatting과 다른 artifact boundary를 검사합니다.

### 무엇을 검사하는가 (diff)

```diff
+release-check: $(NAME)
+	CC="$(CC)" sh tests/check_release.sh $(NAME) include \
+		tests/test_consumer.c
```

Script는 `ar t`의 exact member 목록을 비교하고, `nm -g`를 platform별로 정규화해 global definition과 archive 내부에서 해결되지 않는 external symbol을 계산합니다. Linux에서는 `__errno_location`, `write`, Darwin에서는 `__error`, `write`와 compiler에 따른 stack-check symbol을 기대합니다.

마지막에는 임시 디렉터리에 public header, archive, consumer source만 복사해 compile하고 실행합니다.

```diff
+#include "ft_printf.h"
+
+int	main(void)
+{
+	if (ft_printf("consumer:%d:%s\n", 17, "ok") != 15)
+		return (1);
+	return (0);
+}
```

Expected output을 확인한 뒤 temporary directory cleanup까지 검사합니다.

### 이 check가 증명하는 것 / 증명하지 않는 것

현재 repository가 선언한 archive members, global definitions, external dependencies와 public-header-only consumer 경계를 검증합니다. ABI가 미래 compiler/version에서도 안정적이라는 장기 보장, shared-object 사용 가능성, runtime formatting의 모든 edge case는 증명하지 않습니다.

### 관련 커밋

`1b474fa2a5e3`은 artifact shape가 아니라 실행된 source의 UB·invalid access를 관찰합니다. 두 계층은 서로 대체하지 않습니다.

## 1b474fa2a5e3 — build(sanitize): UBSan과 Linux ASan 검증 추가
**중요도** `B` · **태그** `VERIFY, TEST`

### 왜 별도의 빌드 경계가 필요한가

Prebuilt archive만 test에 link하면 implementation object에 sanitizer instrumentation이 들어가지 않을 수 있습니다. 이 commit은 normal suite와 output fault suite를 `$(SRC)`와 함께 다시 compile해 production과 injected failure 경로를 모두 계측합니다.

### 무엇이 바뀌었는가 (diff)

```diff
+UBSAN_FLAGS := -g -fno-omit-frame-pointer -fsanitize=undefined
+SANITIZER_FLAGS := -g -fno-omit-frame-pointer -fsanitize=address,undefined
@@
+sanitize-undefined:
+	mkdir -p tests/bin
+	$(CC) $(CFLAGS) $(CPPFLAGS) $(UBSAN_FLAGS) \
+		tests/test_ft_printf.c $(SRC) -o $(UBSAN_TEST_BIN)
+	UBSAN_OPTIONS=halt_on_error=1 ./$(UBSAN_TEST_BIN)
+	$(CC) $(CFLAGS) $(CPPFLAGS) $(UBSAN_FLAGS) \
+		-DFT_PRINTF_TEST_WRITE tests/test_output_faults.c $(SRC) \
+		-o $(UBSAN_FAULT_BIN)
+	UBSAN_OPTIONS=halt_on_error=1 ./$(UBSAN_FAULT_BIN)
```

Linux AddressSanitizer는 GCC 14 Bookworm container에서 별도 target으로 실행합니다.

```diff
+sanitize-linux:
+	docker run --rm -v "$(CURDIR):/source:ro" gcc:14-bookworm \
+		sh -c 'cp -R /source /tmp/format-printer-fix && \
+		cd /tmp/format-printer-fix && \
+		make sanitize-address SANITIZER_CC=gcc'
```

`make sanitize`는 UBSan만 실행하고 `sanitize-linux`는 별도입니다. `make check`도 `test`, `release-check`, `sanitize`, `git diff --check`를 묶지만 Docker ASan을 포함하지 않습니다.

### 이 target이 증명하는 것 / 증명하지 않는 것

실제로 실행된 normal/fault inputs에서 instrumented source가 UBSan 또는 ASan 진단 없이 끝나는지를 관찰하도록 구성합니다. 실행하지 않은 branch와 모든 가능한 format의 안전성, prebuilt archive의 member·symbol shape는 증명하지 않습니다.

### 관련 커밋

`9ac825379180`의 bounded string scan과 `e040e69db535`의 non-NUL object case 같은 memory boundary를 계측된 suite에서 관찰할 수 있고, `1223518652bd`의 fault path도 별도 sanitizer binary에 포함됩니다. Artifact 구성은 `a87bcf560789`의 release check가 계속 담당합니다.

## 이 Thread의 경계

- Partial write, `EINTR`, sticky error의 production contract는 `01-output-state-system-call-boundary.md`가 다룹니다.
- Numeric prefix·precision·width 순서는 `03-shared-numeric-layout.md`가 다룹니다.
- `%s` bounded read는 `04-string-precision-bounded-access.md`가 다룹니다.
- Late format error의 무출력 보장은 `05-whole-call-preflight.md`가 다룹니다.

> 검토 범위: `1b8049e411bb`, `1223518652bd`, `12d715eba77d`, `a87bcf560789`, `1b474fa2a5e3`의 exact diff와 해당 시점의 functional/fault tests, release script, consumer source, Makefile sanitizer targets를 확인했습니다. `test`, `release-check`, UBSan, Docker ASan target은 이 환경에서 실행하지 않았습니다.
