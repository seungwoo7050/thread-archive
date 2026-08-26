# Thread: 정적 archive를 검증된 배포 산출물로 다루기

> Project: `libft` · Branch: `c/libft` · 문서 번호: 04

## 개요

이 Thread는 “local compile이 성공했다”를 release evidence로 충분하다고 보지 않는다. 검증을 compiler policy, artifact surface, instrumented runtime, host detector, clean-environment reproducibility로 나누고, 마지막에 선택된 evidence만 하나의 fail-fast entry point로 연결한다. 각 detector의 존재와 최종 orchestration 포함 여부는 별개의 사실로 취급한다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4df8b23505b8` | `build(flags): C99 경고와 builtin 정책을 고정` | A | `RELEASE, VERIFY, RISK` | strict C99 warning과 no-builtin 정책을 build 전체에 고정한다. |
| 2 | `79c0dcefb590` | `test(release): archive와 consumer 경계를 검증` | A | `RELEASE, ARCH, VERIFY` | archive member, 공개 symbol, 허용 외부 의존성, 외부 consumer를 검사한다. |
| 3 | `f5de4306ebcd` | `test(sanitize): undefined behavior 검사를 추가` | B | `VERIFY, TEST` | production source와 functional suite를 UBSan instrumentation으로 build/run한다. |
| 4 | `c625970fd211` | `test(sanitize): address sanitizer 검사를 추가` | B | `VERIFY, TEST` | 별도 ASan object tree와 실행 target을 추가한다. |
| 5 | `9f555c37a6d8` | `test(leak): host 누수 검사 경로를 추가` | B | `VERIFY, TEST` | `leaks` 또는 Valgrind로 ordinary functional binary의 leak을 검사한다. |
| 6 | `e31a2e748685` | `test(build): Clang과 GCC 호환성을 검증` | A | `RELEASE, VERIFY` | clean copied tree에서 두 compiler family로 core release suite를 다시 실행한다. |
| 7 | `b90fd748255a` | `test(release): 전체 검증 절차를 연결` | B | `RELEASE, VERIFY` | clean build, functional/failure, UBSan, archive, compiler, leak, no-op 검사를 한 target에 연결한다. |

---

**역할군 1 — compiler와 archive의 배포 경계를 명시적으로 검사한다.**

## 4df8b23505b8 — build(flags): C99 경고와 builtin 정책을 고정
**중요도** `A` · **태그** `RELEASE, VERIFY, RISK`

### 무엇이 바뀌었는가 (diff)

```diff
-CFLAGS := -Wall -Wextra -Werror -std=c99 -pedantic
-CPPFLAGS := -I.
+override CFLAGS := -Wall -Wextra -Werror -Wpedantic -std=c99 \
+	-fno-builtin
+override CPPFLAGS := -I.

-.PHONY: all clean fclean re test write-failure-test
+.PHONY: all bonus clean fclean re test write-failure-test

+bonus: all
```

### 왜 build policy를 고정하는가

libc와 유사한 함수를 재구현할 때 compiler가 builtin knowledge로 호출을 대체하면 test가 archive의 실제 implementation을 통과하지 않을 수 있다. `-fno-builtin`은 이 우회를 차단하고, `override`는 command line의 `CFLAGS`·`CPPFLAGS` 지정만으로 해당 정책을 제거하지 못하게 한다. ordinary production object와 test-related object rule이 공통 flags를 사용하므로 C99, warning-as-error, pedantic diagnostics, no-builtin이 함께 적용된다.

`bonus: all`은 별도 artifact가 아니라 같은 `libft.a` build를 노출하는 alias다.

### 무엇을 아직 검증하지 않는가

이 커밋은 compile policy만 고정한다. archive에 들어간 object와 global symbol이 정확한지, 허용하지 않은 external dependency가 남는지, 외부 directory의 consumer가 link되는지는 확인하지 않는다.

### 어떤 커밋과 왜 연결되는가

`79c0dcefb590`이 build 결과인 `libft.a` 자체를 member·symbol·dependency·consumer 관점에서 검사한다. 이후 sanitizer와 compiler matrix도 이 고정된 compile policy 위에서 source를 다시 build한다.

## 79c0dcefb590 — test(release): archive와 consumer 경계를 검증
**중요도** `A` · **태그** `RELEASE, ARCH, VERIFY`

### 왜 다른 기법이 필요한가

archive 생성이 성공해도 잘못된 object가 포함되거나 public symbol surface가 바뀌고, 예상하지 않은 runtime dependency가 생기거나 in-tree path에만 기대는 consumer가 남을 수 있다. 이 커밋은 version-controlled manifest와 binary inspection, repository 밖 smoke consumer로 그 경계를 직접 관찰한다.

### archive와 consumer를 어떻게 검사하는가 (diff)

```diff
+check-archive: $(NAME)
+	CC="$(CC)" sh tests/check_archive.sh $(NAME)
```

새 script의 핵심 단계는 다음과 같다.

```diff
+ar t "$archive" | awk '/\.o$/' | sort > "$output_dir/members.actual"
+sort tests/archive-members.txt > "$output_dir/members.expected"
+cmp "$output_dir/members.expected" "$output_dir/members.actual"
+
+case "$(uname -s)" in
+	Darwin)
+		nm -gU -j "$archive" > "$output_dir/defined.raw"
+		nm -u -j "$archive" > "$output_dir/undefined.raw"
+		sed -E 's/^_//' "$output_dir/defined.raw" \
+			> "$output_dir/defined.normalized"
+		;;
+	Linux)
+		nm -g --defined-only -j "$archive" > "$output_dir/defined.raw"
+		nm -u -j "$archive" > "$output_dir/undefined.raw"
+		;;
+esac
+
+sort tests/api-symbols.txt > "$output_dir/symbols.expected"
+cmp "$output_dir/symbols.expected" "$output_dir/symbols.actual"
+
+comm -23 "$output_dir/undefined.all" "$output_dir/symbols.expected" \
+	> "$output_dir/undefined.external"
+cmp "$output_dir/undefined.expected" "$output_dir/undefined.external"
+
+cp tests/smoke/consumer.c "$consumer_dir/consumer.c"
+(
+	cd "$consumer_dir"
+	"$compiler" -I"$project_root" -Wall -Wextra -Werror -Wpedantic \
+		-std=c99 -fno-builtin consumer.c "$archive_path" -o consumer
+	./consumer
+)
```

member manifest는 17개 object를, API manifest는 43개 `ft_*` global definition을 exact set으로 고정한다. external undefined set에는 `free`, `malloc`, `write`와 platform별 errno accessor만 남아야 한다. Darwin과 Linux의 `nm` 출력 차이는 별도 branch에서 정규화하며 그 밖의 OS는 unsupported로 실패한다.

외부 consumer는 temporary directory로 이동한 뒤 project root header와 absolute archive path만 받아 `ft_strdup`, `ft_strlen`, `ft_lstnew`를 호출한다. current working directory의 우연한 relative path에 기대는 build를 드러내는 방식이다.

### 무엇을 증명하고 무엇은 증명하지 않는가

manifest에 적힌 member·public symbol·허용 dependency가 실제 archive와 같고, 외부 cwd에서 명시적 header/archive로 smoke consumer가 build·run됨을 증명한다. package installation layout, shared-library ABI, 모든 platform의 symbol format, 각 함수의 semantic correctness는 증명하지 않는다.

### 어떤 커밋과 왜 연결되는가

`4df8b23505b8`이 compiler가 project implementation을 우회하지 못하게 하고, 이 커밋은 그 결과 artifact의 외형과 소비 경계를 검사한다. `e31a2e748685`은 같은 `check-archive`를 Clang과 GNU GCC clean-copy suite 안에서도 실행한다.

---

**역할군 2 — 서로 다른 runtime 결함군을 독립된 detector로 관찰한다.**

## f5de4306ebcd — test(sanitize): undefined behavior 검사를 추가
**중요도** `B` · **태그** `VERIFY, TEST`

### 무엇을 검증하는가

```diff
+UBSAN_OBJ_DIR := build/ubsan
+UBSAN_OBJ := $(SRC:%.c=$(UBSAN_OBJ_DIR)/%.o)
+UBSAN_BIN := tests/bin/test_ubsan
+UBSAN_FLAGS := -fsanitize=undefined -fno-omit-frame-pointer
+
+$(UBSAN_BIN): $(UBSAN_OBJ) $(TEST_SRC) tests/test.h
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(UBSAN_FLAGS) $(TEST_SRC) $(UBSAN_OBJ) -o $@
+
+ubsan: $(UBSAN_BIN)
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 ./$(UBSAN_BIN)
+
+sanitize: ubsan
```

production source를 별도 object tree에서 UBSan instrumentation으로 compile하고 ordinary functional tests와 link한다. runtime diagnostic이 나오면 즉시 실패하도록 `halt_on_error=1`을 사용한다.

### 무엇을 증명하고 무엇은 증명하지 않는가

ordinary functional suite가 실행한 경로에서 UBSan이 보고하는 undefined behavior가 없음을 관찰한다. allocation-failure와 write-failure binary의 경로, address error, leak, 실행되지 않은 branch는 이 target 하나로 증명하지 않는다.

### 어떤 커밋과 왜 연결되는가

`c625970fd211`은 같은 별도-object-tree 패턴으로 address detector를 추가하고, `b90fd748255a`는 `sanitize` alias를 최종 orchestration에 연결한다.

## c625970fd211 — test(sanitize): address sanitizer 검사를 추가
**중요도** `B` · **태그** `VERIFY, TEST`

### 무엇을 검증하는가

```diff
+ASAN_OBJ_DIR := build/asan
+ASAN_OBJ := $(SRC:%.c=$(ASAN_OBJ_DIR)/%.o)
+ASAN_BIN := tests/bin/test_asan
+ASAN_FLAGS := -fsanitize=address -fno-omit-frame-pointer
+ASAN_OPTIONS ?= detect_leaks=0:halt_on_error=1
+
+asan: $(ASAN_BIN)
+	ASAN_OPTIONS=$(ASAN_OPTIONS) ./$(ASAN_BIN)
```

ASan도 production source와 ordinary functional tests를 별도 instrumented binary로 만든다. 기본 `ASAN_OPTIONS`가 `detect_leaks=0`이므로 이 target의 직접 책임은 실행된 경로의 address error다.

이 diff는 `asan` target을 추가하지만 기존 `sanitize: ubsan` dependency를 바꾸지 않는다. 따라서 `make asan`과 `make sanitize`는 서로 다른 entry point다.

### 무엇을 증명하고 무엇은 증명하지 않는가

ASan binary가 실행한 functional path에서 sanitizer가 보고하는 address diagnostics를 관찰한다. 기본 설정의 leak과 failure-specific binaries는 포함하지 않으며, target이 존재한다는 사실만으로 다른 orchestration에서 자동 실행된다고 볼 수 없다.

### 어떤 커밋과 왜 연결되는가

`9f555c37a6d8`은 leak detection을 host tool로 분리한다. `b90fd748255a`의 final target을 읽을 때는 이 커밋의 `asan`이 실제 dependency로 연결되는지 별도로 확인해야 한다.

## 9f555c37a6d8 — test(leak): host 누수 검사 경로를 추가
**중요도** `B` · **태그** `VERIFY, TEST`

### 왜 다른 기법이 필요한가

UBSan은 leak detector가 아니고, 앞선 ASan target도 기본값으로 leak detection을 끈다. 이 커밋은 ordinary functional binary를 host에서 제공하는 dedicated leak checker 아래 실행한다.

```diff
+leak: $(TEST_BIN)
+	@if command -v leaks >/dev/null 2>&1; then \
+		leaks --atExit -- ./$(TEST_BIN); \
+	elif command -v valgrind >/dev/null 2>&1; then \
+		valgrind --leak-check=full --errors-for-leak-kinds=all \
+			--error-exitcode=1 ./$(TEST_BIN); \
+	else \
+		echo "no supported leak checker found" >&2; \
+		exit 1; \
+	fi
```

macOS `leaks`가 있으면 우선 사용하고, 없으면 Valgrind를 찾는다. 둘 다 없을 때 silent skip하지 않고 target이 실패한다.

### 무엇을 증명하고 무엇은 증명하지 않는가

선택된 host tool 아래 ordinary `TEST_BIN` 실행에서 보고되는 leak을 검사한다. allocation-failure binary와 write-failure binary, 모든 allocator call sequence, tool이 없는 환경에서의 portability는 증명하지 않는다.

### 어떤 커밋과 왜 연결되는가

`c625970fd211`이 기본 ASan leak detection을 끈 상태이므로 이 target이 leak evidence를 별도 담당한다. `b90fd748255a`는 host tool이 실제로 존재해야 성공하는 이 target을 final sequence에 포함한다.

---

**역할군 3 — 깨끗한 compiler matrix와 최종 orchestration으로 evidence를 재현한다.**

## e31a2e748685 — test(build): Clang과 GCC 호환성을 검증
**중요도** `A` · **태그** `RELEASE, VERIFY`

### 왜 다른 기법이 필요한가

현재 working tree의 기존 object와 dependency file을 재사용하면 compiler family별 차이가 가려질 수 있다. 이 커밋은 source와 test input만 temporary tree로 복사하고 Clang과 GNU GCC를 각각 탐색해 독립된 clean suite를 실행한다.

```diff
+run_suite()
+{
+	label=$1
+	compiler=$2
+	work=$scratch/$label
+
+	mkdir -p "$work"
+	cp "$project_root/Makefile" "$project_root/libft.h" "$work/"
+	cp -R "$project_root/src" "$project_root/tests" "$work/"
+	make -s -C "$work" CC="$compiler" fclean
+	make -s -C "$work" CC="$compiler" all test failure-test \
+		write-failure-test check-archive
+}
+
+clang_compiler=$(find_clang || true)
+gcc_compiler=$(find_gcc || true)
+
+if [ -z "$clang_compiler" ]
+then
+	echo "Clang compiler not found" >&2
+	exit 1
+fi
+if [ -z "$gcc_compiler" ]
+then
+	echo "GNU GCC compiler not found" >&2
+	exit 1
+fi
+
+run_suite clang "$clang_compiler"
+run_suite gcc "$gcc_compiler"
```

compiler 후보는 `--version` 첫 줄로 Clang과 GNU GCC를 구분한다. 둘 중 하나라도 찾지 못하면 검사 자체가 실패한다. 각 clean tree에서 archive build, ordinary tests, allocation/write failure tests, archive/consumer inspection을 실행한다.

### 무엇을 증명하고 무엇은 증명하지 않는가

두 compiler family가 같은 copied source로 core suite를 compile·run할 수 있음을 검증하도록 구성된다. ASan, UBSan, host leak checker, top-level `check`까지 compiler별로 반복하는 것은 아니다.

### 어떤 커밋과 왜 연결되는가

`79c0dcefb590`의 archive/consumer 검사도 각 compiler의 `CC` 아래 재사용된다. `b90fd748255a`는 이 compiler matrix를 전체 sequence의 한 단계로 호출한다.

## b90fd748255a — test(release): 전체 검증 절차를 연결
**중요도** `B` · **태그** `RELEASE, VERIFY`

### 무엇을 연결하는가 (diff)

```diff
+check:
+	git diff --check
+	$(MAKE) fclean
+	$(MAKE) all
+	$(MAKE) test
+	$(MAKE) failure-test
+	$(MAKE) write-failure-test
+	$(MAKE) sanitize
+	$(MAKE) check-archive
+	$(MAKE) check-compilers
+	$(MAKE) leak
+	$(MAKE) -q all
```

각 recipe line이 nonzero로 끝나면 Make가 중단되므로 앞 단계의 failure를 숨긴 채 `check`가 성공할 수 없다. sequence는 whitespace 검사, clean archive build, ordinary/failure tests, `sanitize`, artifact inspection, compiler matrix, host leak, no-op rebuild 확인을 순서대로 호출한다.

`sanitize`는 `f5de4306ebcd`에서 정의된 `sanitize: ubsan` 그대로이므로 이 final sequence는 UBSan을 실행하지만 독립 `asan` target은 호출하지 않는다. `make -q all`은 마지막 artifact가 추가 rebuild 없이 up-to-date인지 확인한다.

### 무엇을 증명하고 무엇은 증명하지 않는가

선택된 target들이 한 entry point에서 fail-fast로 연결되고, 성공하려면 host leak tool과 두 compiler family를 포함한 모든 단계가 통과해야 함을 build graph로 고정한다. 이 커밋 자체는 각 binary를 실행한 결과를 기록하지 않으며, ASan 자동 실행, package installation, CI environment 재현성까지 포함하지 않는다.

### 어떤 커밋과 왜 연결되는가

앞선 여섯 커밋이 만든 compile policy, archive inspection, UBSan, compiler matrix, leak target을 조합한다. `c625970fd211`의 ASan은 의도 여부를 추정하지 않고, 이 exact recipe에 dependency가 없다는 사실만 final boundary로 남긴다.

## 이 Thread의 경계

이 Thread는 `libft.a`의 build policy, archive surface, 외부 consumer, sanitizer·leak·compiler 검증 entry point를 다룬다. byte-copy semantics는 [`01-non-overlap-copy-and-overlap-safe-movement`](01-non-overlap-copy-and-overlap-safe-movement.md), allocation rollback은 [`02-single-allocation-to-rollback-safe-ownership`](02-single-allocation-to-rollback-safe-ownership.md), partial descriptor output은 [`03-fd-output-partial-system-calls`](03-fd-output-partial-system-calls.md)에서 다룬다. CI workflow, package installation, versioned ABI, cross-compilation, Windows toolchain, 모든 OS의 symbol normalization은 별개의 문제다.

> 검토 범위: `4df8b23505b8`, `79c0dcefb590`, `f5de4306ebcd`, `c625970fd211`, `9f555c37a6d8`, `e31a2e748685`, `b90fd748255a`의 commit diff와 해당 SHA의 `Makefile`, `tests/check_archive.sh`, manifests, smoke consumer, sanitizer/leak rules, `tests/check_compilers.sh`를 확인했다. `make check`, compiler matrix, sanitizer, leak checker, ordinary/failure test binaries는 실행하지 않았다.
