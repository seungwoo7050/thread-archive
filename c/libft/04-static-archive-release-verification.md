# Thread: 정적 archive를 검증된 배포 산출물로 다루기

> Project: `libft`  
> Branch: `c/libft`

## 개요

이 Thread는 “현재 directory에서 compile된다”는 사실을 release evidence로 충분하다고 보지 않는다. 검증 대상은 source뿐 아니라 실제 `libft.a`의 구성과 외부 소비 경계다.

검증 축은 서로 대체되지 않는다.

- build flags는 compiler가 project 함수를 builtin으로 바꾸지 못하게 한다.
- archive inspection은 member, 공개 symbol, undefined dependency를 manifest와 비교한다.
- out-of-tree smoke test는 header와 archive만으로 외부 consumer가 link/run되는지 확인한다.
- UBSan, ASan, host leak checker는 서로 다른 결함군을 관찰한다.
- Clang과 GNU GCC 검사는 compiler-family별 가정을 찾는다.
- 마지막 `check` target은 이들 중 선택된 절차를 순서대로 연결하고 no-op rebuild까지 확인한다.

중요한 실제 범위가 하나 있다. `c625970fd211`에서 `asan` target은 추가되지만 `sanitize` aggregate는 계속 `ubsan`만 의존한다. 따라서 `b90fd748255a`의 `check`는 `$(MAKE) sanitize`를 통해 UBSan은 실행하지만 ASan은 실행하지 않는다. ASan은 별도로 `make asan`을 호출해야 한다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4df8b23505b8` | `build(flags): C99 경고와 builtin 정책을 고정` | A | `RELEASE`, `VERIFY`, `RISK` | strict C99 warning과 no-builtin 정책을 build 전체에 고정한다. |
| 2 | `79c0dcefb590` | `test(release): archive와 consumer 경계를 검증` | A | `RELEASE`, `ARCH`, `VERIFY` | archive member, 공개 symbol, 허용 외부 의존성, 외부 consumer를 검사한다. |
| 3 | `f5de4306ebcd` | `test(sanitize): undefined behavior 검사를 추가` | B | `VERIFY`, `TEST` | production source와 functional suite를 UBSan instrumentation으로 build/run한다. |
| 4 | `c625970fd211` | `test(sanitize): address sanitizer 검사를 추가` | B | `VERIFY`, `TEST` | 별도 ASan object tree와 실행 target을 추가한다. |
| 5 | `9f555c37a6d8` | `test(leak): host 누수 검사 경로를 추가` | B | `VERIFY`, `TEST` | `leaks` 또는 Valgrind로 ordinary functional binary의 leak을 검사한다. |
| 6 | `e31a2e748685` | `test(build): Clang과 GCC 호환성을 검증` | A | `RELEASE`, `VERIFY` | clean copied tree에서 두 compiler family로 core release suite를 다시 실행한다. |
| 7 | `b90fd748255a` | `test(release): 전체 검증 절차를 연결` | B | `RELEASE`, `VERIFY` | clean build, functional/failure, UBSan, archive, compiler, leak, no-op 확인을 한 target에 연결한다. |

## `4df8b23505b8` — compiler가 검증 대상을 우회하지 못하게 한다

```make
override CFLAGS := -Wall -Wextra -Werror -Wpedantic -std=c99 \
	-fno-builtin
override CPPFLAGS := -I.
```

`override`는 command line에서 `CFLAGS`나 `CPPFLAGS`를 다시 지정해도 이 Makefile 정책이 제거되지 않게 한다. 일반 production object, ordinary test, 별도 write-failure object의 compile recipe가 공통 `$(CPPFLAGS) $(CFLAGS)`를 사용하므로 다음 조건이 함께 적용된다.

- C99 dialect
- warning-as-error
- pedantic diagnostics
- compiler builtin substitution 비활성화

libc 유사 함수를 재구현하는 project에서 `-fno-builtin`은 단순 최적화 취향이 아니다. compiler가 호출을 builtin knowledge로 대체하면 test가 archive의 실제 implementation을 통과하지 않을 수 있다.

`bonus: all`도 이 커밋에 추가된다. 별도 bonus archive를 만드는 것이 아니라 같은 `libft.a` build를 노출하는 alias다.

이 단계는 compile policy만 고정한다. archive에 어떤 object가 들어갔는지나 외부에서 link되는지는 아직 검사하지 않는다.

## `79c0dcefb590` — archive 내부와 외부 소비 경계를 동시에 검사한다

`check-archive`는 version-controlled manifest 세 개와 shell script, 외부 consumer를 추가한다.

### 1. archive member 목록

```sh
ar t "$archive" | awk '/\.o$/' | sort > "$output_dir/members.actual"
sort tests/archive-members.txt > "$output_dir/members.expected"
cmp "$output_dir/members.expected" "$output_dir/members.actual"
```

manifest에는 17개 object member가 있다. 누락, 예상하지 않은 member, 이름 변경은 exact `cmp`에서 실패한다.

### 2. 공개 global symbol 목록

Darwin과 Linux의 `nm` 출력 차이를 나눠 처리한다.

```text
Darwin: nm -gU -j / nm -u -j, leading '_' 제거
Linux:  nm -g --defined-only -j / nm -u -j
```

정규화한 defined identifiers를 43개 `ft_*` API manifest와 정렬 비교한다. expected symbol이 빠진 경우뿐 아니라 예상하지 않은 global definition이 추가된 경우도 실패한다.

### 3. 허용된 undefined dependency

archive member 사이의 `ft_*` reference는 public symbol manifest를 사용해 undefined set에서 제외한다. 남은 외부 symbol은 다음 기본 목록과 platform errno accessor만 허용한다.

```text
free
malloc
write
Darwin: __error
Linux:  __errno_location
```

따라서 source가 모르는 runtime dependency를 새로 끌어오면 archive build가 성공하더라도 release 검사에서 드러난다.

### 4. repository 밖 consumer

script는 `mktemp`로 repository 외부 directory를 만든 뒤 consumer source만 복사한다. 그 directory로 이동해 project root의 header와 absolute archive path를 명시적으로 넘긴다.

```sh
"$compiler" -I"$project_root" -Wall -Wextra -Werror -Wpedantic \
	-std=c99 -fno-builtin consumer.c "$archive_path" -o consumer
./consumer
```

consumer는 `ft_strdup`, `ft_strlen`, `ft_lstnew`를 사용하고 직접 cleanup한다. 이 검사는 current working directory나 in-tree relative path에 우연히 의존하는 build를 찾는다.

다만 installed include layout, shared-library ABI, package manager 배포를 증명하는 것은 아니다. header와 archive의 absolute path를 알고 있는 외부 consumer를 검증한다.

## 서로 다른 결함군을 보는 세 검사

### `f5de4306ebcd` — UBSan

모든 production source를 별도 `build/ubsan` object tree로 compile하고 ordinary functional test source와 함께 sanitizer flags로 link한다.

```make
UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer

ubsan: $(UBSAN_BIN)
	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 ./$(UBSAN_BIN)

sanitize: ubsan
```

runtime undefined-behavior 진단이 하나라도 나오면 즉시 실패하도록 한다. functional suite가 실행하는 경로만 관찰하며, allocation/write failure binaries는 이 target에 포함되지 않는다.

### `c625970fd211` — ASan

```make
ASAN_FLAGS := -fsanitize=address -fno-omit-frame-pointer
ASAN_OPTIONS ?= detect_leaks=0:halt_on_error=1

asan: $(ASAN_BIN)
	ASAN_OPTIONS=$(ASAN_OPTIONS) ./$(ASAN_BIN)
```

ASan도 별도 production object tree와 ordinary functional suite를 사용한다. `detect_leaks=0`이므로 이 target의 기본 책임은 address error이며 leak 검사는 다음 target과 분리된다.

이 커밋은 `.PHONY`에 `asan`을 추가하지만 다음 line을 바꾸지 않는다.

```make
sanitize: ubsan
```

따라서 `make sanitize`는 ASan을 실행하지 않는다.

### `9f555c37a6d8` — host leak checker

```make
leak: $(TEST_BIN)
	@if command -v leaks >/dev/null 2>&1; then \
		leaks --atExit -- ./$(TEST_BIN); \
	elif command -v valgrind >/dev/null 2>&1; then \
		valgrind --leak-check=full --errors-for-leak-kinds=all \
			--error-exitcode=1 ./$(TEST_BIN); \
	else \
		echo "no supported leak checker found" >&2; \
		exit 1; \
	fi
```

macOS `leaks`가 있으면 우선 사용하고, 아니면 Valgrind를 찾는다. 둘 다 없으면 skip하지 않고 target 자체가 실패한다.

검사 대상은 ordinary `TEST_BIN`이다. deterministic allocation-failure binary나 write-failure binary의 leak까지 이 target 하나로 증명하지 않는다.

## `e31a2e748685` — compiler family 검사는 깨끗한 복사본에서 수행한다

`check_compilers.sh`는 version text를 확인해 Clang과 GNU GCC를 각각 찾는다. 둘 중 하나라도 없으면 실패한다.

각 compiler마다 임시 directory에 다음만 복사한다.

```text
Makefile
libft.h
src/
tests/
```

그 뒤 해당 compiler를 `CC`로 넘겨 실행한다.

```sh
make -s -C "$work" CC="$compiler" fclean
make -s -C "$work" CC="$compiler" all test failure-test \
	write-failure-test check-archive
```

clean copied tree를 쓰는 이유는 현재 build artifact나 dependency file이 다른 compiler 결과를 가리는 것을 막기 위해서다. `check-archive`의 외부 consumer도 같은 `CC`를 전달받는다.

이 compiler matrix가 실행하는 범위는 명확하다.

- 실행: archive build, ordinary functional tests, allocation failure tests, write failure tests, archive/consumer inspection
- 미실행: ASan, UBSan, host leak checker, 최상위 `check`

따라서 “두 compiler에서 모든 Make target을 실행한다”는 의미는 아니다.

## `b90fd748255a` — 최종 `check`가 실제로 연결하는 것

```make
check:
	git diff --check
	$(MAKE) fclean
	$(MAKE) all
	$(MAKE) test
	$(MAKE) failure-test
	$(MAKE) write-failure-test
	$(MAKE) sanitize
	$(MAKE) check-archive
	$(MAKE) check-compilers
	$(MAKE) leak
	$(MAKE) -q all
```

각 command가 nonzero로 끝나면 Make가 다음 line으로 진행하지 않으므로 앞선 evidence가 실패한 상태에서 release check가 성공할 수 없다.

단계별 의미는 다음과 같다.

| 단계 | 확인하는 것 |
| --- | --- |
| `git diff --check` | working-tree diff의 whitespace 오류 |
| `fclean` → `all` | 이전 artifact를 버린 clean archive build |
| `test` | ordinary functional behavior |
| `failure-test` | production allocator 치환 경로 |
| `write-failure-test` | scripted partial write/error 경로 |
| `sanitize` | 이 SHA에서는 **UBSan만** |
| `check-archive` | member, symbol, external dependency, external consumer |
| `check-compilers` | Clang/GNU GCC clean-copy core suite |
| `leak` | host leak checker로 ordinary test binary 검사 |
| `make -q all` | 최종 archive가 추가 build 없이 up-to-date인지 확인 |

`asan`은 독립 target으로 존재하지만 이 sequence에는 없다. 따라서 이 커밋의 정확한 최종 상태는 다음과 같다.

```text
make check
  = functional + deterministic failures + UBSan
    + archive/consumer + compiler matrix + host leak + no-op rebuild

make asan
  = 별도로 실행해야 하는 address-sanitized functional suite
```

## evidence 범위 정리

| 검사 | 직접 관찰하는 범위 | 직접 증명하지 않는 범위 |
| --- | --- | --- |
| no-builtin build | project implementation을 compile 대상으로 유지 | runtime memory correctness |
| archive manifest | member/API/dependency drift | 함수별 semantic correctness |
| external consumer | 외부 cwd에서 header+archive link/run | 설치 패키지와 ABI 안정성 |
| UBSan | 실행된 functional path의 UB 진단 | address leak 및 미실행 failure path |
| ASan | 실행된 functional path의 address error | 기본 설정상 leak, `make check` 포함 여부 |
| host leak | ordinary test process의 leak | failure binaries와 모든 호출 조합 |
| compiler matrix | 두 compiler family의 core suite | sanitizer/leak의 compiler별 실행 |
| final `check` | 위 표의 선택된 절차 orchestration | ASan 자동 실행 |

## Thread의 범위

이 Thread는 static archive의 build·inspection·consumer·host verification을 다룬다. 이후 CI workflow, package installation, versioned ABI, cross-compilation, Windows toolchain, 모든 OS의 symbol normalization은 포함하지 않는다.

> 검토 범위: 표시된 exact SHA의 commit diff와 `b90fd748255a` 시점의 Makefile/scripts를 확인했다. release target과 sanitizer/test suite는 실행하지 않았다.
