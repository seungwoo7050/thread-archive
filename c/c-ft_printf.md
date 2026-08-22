===== BEGIN FILE: 01-output-state-system-call-boundary.md =====
# A single output state becomes a robust system-call boundary

## 1. Thread 목표

초기 local write/count 처리에서 출발해, 모든 conversion이 공유하는 `t_printf` 출력 상태와 최종 POSIX `write` 진행 정책이 어떤 commit을 거쳐 형성되는지 복원합니다.

### Source에서 확정된 significance

출력 계층은 편의성 helper에서 모든 conversion이 공유하는 상태 머신으로 발전합니다. count 범위, progress, interruption, permanent failure, system-call 비용, process signal policy가 명시되고 운영체제 타이밍에 의존하지 않는 방식으로 검증됩니다.

### 이 Thread에 명시적으로 연결되는 source invariant / engineering difficulty

- Invariant: 누적 count는 `INT_MAX`를 넘어 narrow/overflow하지 않습니다.
- Invariant: positive short write는 buffer를 전진시키고, `EINTR`는 retry하며, request는 `SSIZE_MAX`를 넘지 않고, non-retryable 또는 zero-byte result는 output을 error로 중단합니다.
- Invariant: library는 process의 `SIGPIPE` disposition을 바꾸지 않으며, 이미 OS가 받아들인 byte를 rollback할 수 없습니다.
- Engineering difficulty: partial write, interruption, zero progress, `EPIPE`, `SIGPIPE`를 처리하면서 process-wide signal policy까지 library가 소유하지 않는 경계를 유지하는 문제입니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 출력 descriptor, 누적 count, sticky error의 ownership은 언제 하나의 상태로 묶이는가?
- positive short write, `EINTR`, zero progress, permanent error는 각각 state와 buffer offset을 어떻게 바꾸는가?
- `ssize_t` 결과와 public `int` return count 사이의 범위 경계는 어디에서 보장되는가?
- wide padding 성능 개선은 공통 failure/count path를 우회하지 않고 어떻게 이루어지는가?
- `EPIPE`가 발생해도 library가 process-wide `SIGPIPE` policy를 소유하지 않는다는 사실은 코드와 테스트에서 어떻게 드러나는가?

## 3. 완료 기준

- `1d6a5cee3041`부터 `1223518652bd`까지 write state transition을 실제 함수와 branch로 설명할 수 있습니다.
- count overflow, short write, `EINTR`, zero return, `EPIPE` 각각에 대해 state 변화와 public return 결과를 구분할 수 있습니다.
- 64-byte padding chunk가 성능만 바꾸고 공통 output invariant는 바꾸지 않는다는 근거를 해당 SHA 코드와 테스트에서 제시할 수 있습니다.
- caller-owned `SIGPIPE` disposition과 library-owned error propagation의 경계를 설명할 수 있습니다.

## 4. Commit map

| SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- |
| `1d6a5cee3041` | feat(core): 리터럴과 퍼센트 출력 구현 | `B` | `CORE, OUTPUT` | Starts with a local write-and-count helper that rejects short writes. |
| `3f7b0ab926d0` | feat(output): 출력 컨텍스트와 쓰기 API 추가 | `S` | `ARCH, OUTPUT, CORE` | Introduces shared descriptor, count, and sticky-error state and resumes after positive short writes. |
| `78e5d25d7df6` | refactor(core): 리터럴 출력을 컨텍스트 API로 이관 | `B` | `OUTPUT, REFACTOR` | Migrates literal output to the shared context, eliminating duplicate accounting. |
| `c627bd1f85bb` | fix(output): 쓰기 결과를 집계하기 전에 범위 검증 | `A` | `OUTPUT, RISK, DEBUG` | Validates a wide `ssize_t` result before narrowing it into the public `int` count. |
| `8a3ec50cb689` | fix(output): 중단된 쓰기 재시도와 요청 크기 제한 | `S` | `OUTPUT, CORE, RISK` | Caps requests at `SSIZE_MAX`, retries `EINTR`, and creates a deterministic write seam. |
| `22e65c176b5d` | perf(output): 반복 채움을 묶어서 출력 | `A` | `OUTPUT, PERF` | Emits wide padding in bounded chunks rather than one system call per byte. |
| `1223518652bd` | test(output): 쓰기 실패 시퀀스와 채움 전략 검증 | `A` | `OUTPUT, TEST, RISK` | Scripts partial progress, interruption, zero writes, `EPIPE`, and verifies `SIGPIPE` and chunking policy. |

## 5. Commit별 학습 기록

> 원칙: 아래 기록은 final HEAD가 아니라 각 항목의 정확한 SHA에서 작성합니다. source가 확정하지 않은 파일명/함수명은 현재 골격에서 추측하지 않습니다.

## 5.1 `1d6a5cee3041` — feat(core): 리터럴과 퍼센트 출력 구현

- Importance: `B`
- Tags: `CORE, OUTPUT`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Starts with a local write-and-count helper that rejects short writes.
- Commit Classification summary: Creates the public entry point, archive, literal loop, percent escape, and initial counting.
- Importance 근거: This is necessary project bootstrap, but its one-byte writes and short-write rejection are an initial implementation later replaced by the defining output architecture.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - 이전 parent에는 이 library 구현이 없었고, 이 SHA에서 public `ft_printf`, archive build, literal/`%%` 출력의 최소 실행 경로가 처음 생겼습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - `src/ft_printf.c`의 `ft_printf`가 format cursor와 local `count`를 소유하고, `ft_write_count`가 fd 1 write와 누적 길이 검사를 함께 수행합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - literal과 escaped percent를 모두 한 byte씩 `ft_write_count`에 넘깁니다. helper는 `length <= 0`이면 no-op, 합계 범위를 먼저 검사하고, `write(1, ...)`가 정확히 요청 길이를 반환한 경우에만 count를 증가시킵니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - positive short write를 실패로 처리하는 정책과 entry point에 붙어 있는 count/error 책임은 `3f7b0ab926d0`의 context API 및 `78e5d25d7df6`의 migration으로 대체됩니다.

### 해당 SHA에서 확인할 코드
- 해당 SHA에서 public `ft_printf` entry point와 local write/count helper의 실제 이름과 위치를 찾습니다.
- literal byte와 `%%`가 main format loop에서 어떤 branch를 거쳐 fd 1로 전달되는지 추적합니다.
- null format, `write` failure, count + requested length의 `INT_MAX` overflow check 순서를 실제 조건식으로 기록합니다.
- positive short write가 “일부 progress”가 아니라 failure로 판정되는 조건을 확인합니다.
- 직후 `3f7b0ab926d0`과 비교하여 descriptor/count/error 책임 중 무엇이 local implementation에서 context로 이동했는지 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf.c`: `ft_write_count`, `ft_printf`. `format == 0`은 `va_start` 전에 `-1`; `*format == '%' && *(format + 1) == '%'`이면 `%` 한 byte; 그 외에는 현재 literal 한 byte를 기록합니다.
  - `ft_write_count`: `*count > INT_MAX - length`, `written < 0 || written != length`가 실패 branch입니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 1d6a5cee3041, src/ft_printf.c, ft_write_count */
written = (int)write(1, buffer, (size_t)length);
if (written < 0 || written != length)
    return (-1);
*count += written;
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - public formatter의 최소 골격을 만든 commit입니다. 다만 출력 상태는 local 변수와 helper에 흩어져 있고, 성공을 “요청 길이 전체가 한 번에 기록됨”으로 정의하므로 POSIX가 허용하는 short write를 진행 상태로 보존하지 못합니다.

## 5.2 `3f7b0ab926d0` — feat(output): 출력 컨텍스트와 쓰기 API 추가

- Importance: `S`
- Tags: `ARCH, OUTPUT, CORE`
- Most Important Commits 목록: 포함
- Thread 내 역할: Introduces shared descriptor, count, and sticky-error state and resumes after positive short writes.
- Commit Classification summary: Introduces a shared output context with descriptor, count, sticky error, and short-write progress.
- Importance 근거: This abstraction determines how every later formatter accounts for bytes and propagates failure. Removing it would leave a fundamental gap in the project's responsibility boundaries and output correctness story.

### 학습 깊이
- 이 commit은 architecture/invariant의 핵심으로 취급합니다.
- 학습자 기록 — 직전 상태:
  - `ft_printf`의 local `count`와 `ft_write_count`가 fd 1, count, 실패를 함께 다뤘고, 한 번의 short write도 즉시 실패했습니다.
- 학습자 기록 — 해결하려던 문제:
  - 이후 모든 conversion이 같은 descriptor/count/error 규칙을 써야 하며, positive short write 이후 아직 쓰지 못한 suffix를 계속 처리해야 했습니다.
- 학습자 기록 — 기존 설계가 충분하지 않았던 이유:
  - entry point 전용 helper에는 공유 가능한 sticky failure state가 없고, caller마다 count와 write 결과를 다시 처리하게 될 위험이 있었습니다.
- 학습자 기록 — 선택한 핵심 decision:
  - private `t_printf { fd, count, error }`와 `ft_printf_init`, `ft_printf_write`, `ft_printf_putchar`를 도입하고, `ft_printf_write`가 남은 buffer 전체를 처리하는 단일 loop를 소유하도록 했습니다.
- 학습자 기록 — ownership / lifecycle / state transition:
  - caller가 stack context를 생성하고 initializer가 세 field를 설정합니다. output API만 `count`와 `error`를 mutate합니다. positive result마다 pointer와 remaining length를 진행시키며, failure 후 `error == 1`은 이후 write를 차단합니다.
- 학습자 기록 — failure scenario와 public consequence:
  - 이 SHA의 API 자체는 `written <= 0` 또는 count overflow에서 `error = 1`, `-1`을 반환합니다. 다만 이 commit 시점의 `ft_printf`는 아직 새 context를 호출하지 않으므로 public entry와의 통합은 다음 SHA에서 이루어집니다.
- 학습자 기록 — 이 SHA가 보장하는 것:
  - 새 output API를 사용하는 caller에서는 positive short write가 suffix 재시도로 이어지고, 모든 성공 byte가 한 count에 집계되며 error가 sticky해집니다.
- 학습자 기록 — 아직 보장하지 않는 것:
  - `EINTR` 재시도, `SSIZE_MAX` request cap, `ssize_t`를 `int`로 narrow하기 전 개별 범위 검사, 실제 entry point migration은 아직 없습니다.
- 학습자 기록 — 후속 fix/test로 이어지는 지점:
  - `78e5d25d7df6`이 entry를 migration하고, `c627bd1f85bb`가 narrowing 순서를 고치며, `8a3ec50cb689`와 `1223518652bd`가 POSIX retry 정책과 deterministic proof를 완성합니다.

### 해당 SHA에서 확인할 코드
- 해당 SHA의 private `t_printf` 정의에서 descriptor, accumulated count, sticky error에 대응하는 실제 field를 기록합니다.
- context initializer, `ft_printf_write`, `ft_printf_putchar`의 caller/callee 관계를 추적합니다.
- `ft_printf_write`에서 positive short write 후 buffer pointer/remaining length/count가 각각 언제 갱신되는지 순서대로 기록합니다.
- 한 번 error state에 들어간 뒤 subsequent write가 어떻게 차단되는지 모든 early-return branch를 확인합니다.
- count overflow 검사 위치와 `write` return type 처리 방식을 기록하고, 이후 `c627bd1f85bb`가 왜 이 경계를 다시 수정하는지 비교할 근거를 남깁니다.
- parent SHA와 diff하여 이전 local state가 제거/대체된 정확한 코드 지점을 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf_internal.h`: `t_printf`의 `fd`, `count`, `error`; output API 선언.
  - `src/ft_output.c`: `ft_printf_init`, `ft_printf_write`, `ft_printf_putchar`. `ctx->error` early return, `written <= 0`, count overflow branch, positive progress mutation이 있습니다.
  - `Makefile`: `src/ft_output.c`를 archive source에 추가합니다. `src/ft_printf.c`의 기존 local helper는 이 SHA에 아직 남아 있습니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 3f7b0ab926d0, src/ft_output.c, ft_printf_write */
while (length > 0)
{
    written = write(ctx->fd, buffer, length);
    if (written <= 0)
    {
        ctx->error = 1;
        return (-1);
    }
    if (ctx->count > INT_MAX - (int)written)
    {
        ctx->error = 1;
        return (-1);
    }
    ctx->count += (int)written;
    buffer += written;
    length -= (size_t)written;
}
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `src/ft_printf.c`의 local `int count`와 `ft_write_count`가 fd 1을 고정하고 short write를 거부했습니다.
  - 이후: 별도 private context가 fd/count/error를 소유하고, `ft_printf_write`가 positive result만큼 suffix를 전진시킵니다. entry point migration은 아직 일어나지 않았습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - formatter 전반이 공유할 output state machine의 핵심을 도입한 commit입니다. descriptor, 누적 count, sticky error를 하나의 private context에 모으고 short write를 정상 progress로 바꾸었지만, 이 시점에는 새 API가 아직 public loop에 연결되지 않았고 POSIX interruption/range 세부 규칙도 미완성입니다.

## 5.3 `78e5d25d7df6` — refactor(core): 리터럴 출력을 컨텍스트 API로 이관

- Importance: `B`
- Tags: `OUTPUT, REFACTOR`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Migrates literal output to the shared context, eliminating duplicate accounting.
- Commit Classification summary: Migrates literal and percent output to the shared context.
- Importance 근거: The migration removes duplicate accounting and is required integration work, but the decisive architecture was established by the preceding context commit.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - shared output API는 존재했지만 `ft_printf`가 여전히 local helper와 local count를 사용해 두 구현이 병존했습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - public loop의 literal/escaped-percent 출력을 `ft_printf_putchar(&ctx, ...)`로 이관하고 local helper를 삭제합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - `ft_printf`가 stack `t_printf ctx`를 초기화하고 traversal 동안 같은 context를 전달합니다. loop 종료 뒤 `ctx.error`이면 `-1`, 아니면 `ctx.count`를 반환합니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - 이후 conversion dispatch도 이 context를 공유하며, output loop의 range/EINTR/request 정책은 후속 fix에서 강화됩니다.

### 해당 SHA에서 확인할 코드
- 해당 SHA의 `ft_printf`에서 literal/escaped-percent가 shared output API를 호출하는 지점을 찾습니다.
- entry point가 format traversal, variadic traversal init/close, final context-to-public-result translation만 담당하는지 실제 코드로 확인합니다.
- 직전 SHA에 남아 있던 local write/count implementation이 완전히 제거되었는지 diff로 확인합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf.c`: local `ft_write_count` 삭제, `t_printf ctx`, `ft_printf_init(&ctx, 1)`, 두 `ft_printf_putchar` branch, final `ctx.error` translation.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 78e5d25d7df6, src/ft_printf.c, ft_printf */
ft_printf_init(&ctx, 1);
while (*format)
{
    if (*format == '%' && *(format + 1) == '%')
    {
        if (ft_printf_putchar(&ctx, '%') < 0)
            break ;
        format += 2;
    }
    else
    {
        if (ft_printf_putchar(&ctx, *format) < 0)
            break ;
        format++;
    }
}
va_end(args);
if (ctx.error)
    return (-1);
return (ctx.count);
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 새 architecture를 실제 public 경로에 연결한 integration commit입니다. entry point에서 중복 count/write 처리를 없애고, 모든 현재 출력이 같은 context의 count와 sticky error를 통과하도록 만들었습니다.

## 5.4 `c627bd1f85bb` — fix(output): 쓰기 결과를 집계하기 전에 범위 검증

- Importance: `A`
- Tags: `OUTPUT, RISK, DEBUG`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Validates a wide `ssize_t` result before narrowing it into the public `int` count.
- Commit Classification summary: Rejects a write result wider than int before narrowing and adding it.
- Importance 근거: The small fix restores the public count invariant at the exact conversion boundary and avoids implementation-defined narrowing. Its impact is significant despite the one-line diff.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - `written`은 `ssize_t`인데 guard가 `(int)written`을 먼저 만들었습니다. 성공값이 `INT_MAX`보다 크면 검증 자체가 implementation-defined narrowing 결과에 의존했습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - system-call result의 representability를 wide type 상태에서 먼저 검사하고, 통과한 뒤에만 public `int` count 계산에 참여시킵니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - `written > INT_MAX` 또는 `count > INT_MAX - (int)written`이면 count/pointer/remaining을 mutate하지 않고 sticky error로 전환합니다.
- 학습자 기록 — failure 또는 edge case:
  - 단일 successful write가 public return type보다 큰 경우와 기존 count와의 합이 `INT_MAX`를 넘는 경우를 구분합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: `ssize_t` 성공값을 안전하게 `int`로 표현할 수 있을 때만 count에 더하며, 누적 count도 `INT_MAX`를 넘지 않습니다.
  - 미보장: oversized request 사전 제한, `EINTR` retry, 해당 거대 successful result를 실제로 발생시키는 deterministic test는 이 commit에 없습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `8a3ec50cb689`가 request 자체를 `SSIZE_MAX`로 cap하고 interruption 정책을 추가합니다. `1223518652bd`는 output state를 광범위하게 검증하지만 이 exact `written > INT_MAX` branch를 직접 scripted하지는 않습니다.

### 해당 SHA에서 확인할 코드
- fix 직전 SHA에서 `write`의 `ssize_t` result가 `int`로 cast되는 위치와 overflow guard의 평가 순서를 기록합니다.
- `ssize_t` successful result가 `INT_MAX`보다 큰 경우 이전 식이 어떤 narrowing 위험을 갖는지 type 단위로 설명합니다.
- fix SHA에서 “개별 `written` representability 확인 → accumulated sum 확인”의 실제 조건 순서를 기록합니다.
- 범위 위반 시 sticky error와 public `-1`까지 어떤 경로로 전달되는지 추적합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_output.c`, `ft_printf_write`: 하나의 compound condition 앞쪽에 `written > INT_MAX`를 추가합니다. 실패 시 `ctx->error = 1`, caller chain을 따라 `ft_printf`가 `-1`을 반환합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* c627bd1f85bb, src/ft_output.c, ft_printf_write */
if (written > INT_MAX || ctx->count > INT_MAX - (int)written)
{
    ctx->error = 1;
    return (-1);
}
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `ctx->count > INT_MAX - (int)written`
  - 이후: `written > INT_MAX`를 먼저 평가한 뒤 동일 누적 합 검사를 수행합니다.

### Failure → Fix 추적
- 기존 가정/상태: `write` result를 `int`로 먼저 cast해도 overflow guard가 안전하다는 암묵적 가정
- 실제 failure 또는 위험: `ssize_t` successful result > `INT_MAX`일 때 narrowing이 implementation-defined/negative가 될 수 있음
- source가 지목한 root cause: 범위 검증보다 cast가 먼저 일어나는 순서
- 수정된 decision/invariant: 각 `written`이 `int`에 representable한지 먼저 확인하고, 그 다음 accumulated sum을 `INT_MAX`에 대해 검증
- 학습자 기록 — 실제 수정 코드:
  - `if (written > INT_MAX || ctx->count > INT_MAX - (int)written)` 순서로 바뀌며, 실패 branch는 count를 갱신하기 전에 sticky error를 설정합니다.
- 학습자 기록 — regression test 연결:
  - source가 이 exact representability branch에 직접 연결한 deterministic case는 없습니다. `1223518652bd`는 partial/EINTR/zero/EPIPE/chunking을 검증하여 주변 state machine을 보호하지만, `written > INT_MAX` 반환을 생성하지 않습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 한 줄이지만 public return invariant를 지키는 type boundary fix입니다. wide syscall result를 좁힌 뒤 검사하던 순서를 뒤집어, representable한 성공값만 `int` count에 반영합니다.

## 5.5 `8a3ec50cb689` — fix(output): 중단된 쓰기 재시도와 요청 크기 제한

- Importance: `S`
- Tags: `OUTPUT, CORE, RISK`
- Most Important Commits 목록: 포함
- Thread 내 역할: Caps requests at `SSIZE_MAX`, retries `EINTR`, and creates a deterministic write seam.
- Commit Classification summary: Caps write requests at SSIZE_MAX, retries EINTR, and exposes a deterministic write test seam.
- Importance 근거: This completes the project-defining POSIX output policy beyond simple short-write handling. It establishes progress, interruption, and request-range invariants used by every conversion and enables direct failure-path proof.

### 학습 깊이
- 이 commit은 architecture/invariant의 핵심으로 취급합니다.
- 학습자 기록 — 직전 상태:
  - loop는 positive short write를 처리했지만 `write`에 남은 `size_t length`를 그대로 넘겼고, 모든 nonpositive result를 동일한 permanent error로 처리했습니다.
- 학습자 기록 — 해결하려던 문제:
  - `write`가 표현 가능한 최대 successful count를 넘는 request를 피하고, signal interruption을 실제 output failure와 구분하며, 드문 분기를 재현 가능하게 만들어야 했습니다.
- 학습자 기록 — 기존 설계가 충분하지 않았던 이유:
  - `EINTR`는 아무 byte도 소비하지 않았는데도 sticky failure가 되었고, production timing만으로 partial/EINTR/zero/EPIPE 순서를 안정적으로 시험할 수 없었습니다.
- 학습자 기록 — 선택한 핵심 decision:
  - 각 iteration의 `request`를 `min(length, SSIZE_MAX)`로 정하고, `written < 0 && errno == EINTR`이면 mutation 없이 continue합니다. compile-time macro로 system call 한 지점만 test writer로 대체합니다.
- 학습자 기록 — ownership / lifecycle / state transition:
  - `ft_printf_write`가 request sizing과 retry를 소유합니다. positive result만 count/buffer/length를 mutate하고, EINTR은 상태를 보존하며, zero 또는 non-EINTR negative는 `error = 1`로 terminal state를 만듭니다.
- 학습자 기록 — failure scenario와 public consequence:
  - EPIPE 등 permanent error와 zero progress는 이미 accepted된 prefix를 되돌리지 못한 채 public `-1`로 끝납니다. 이후 write 호출은 sticky early return으로 차단됩니다.
- 학습자 기록 — 이 SHA가 보장하는 것:
  - 요청 크기 제한, transparent EINTR retry, positive progress, zero/hard-error stop, deterministic seam이 한 output loop에 존재합니다.
- 학습자 기록 — 아직 보장하지 않는 것:
  - 이 commit 자체에는 scripted assertions가 없고, SIGPIPE disposition을 변경하지 않는 실제 integration 증명과 padding chunk cost 검증은 후속 test에 있습니다.
- 학습자 기록 — 후속 fix/test로 이어지는 지점:
  - `1223518652bd`가 `FT_PRINTF_TEST_WRITE`를 이용해 partial/EINTR/EPIPE/zero를 순서대로 주입하고, real broken pipe와 64-byte chunk까지 검증합니다.

### 해당 SHA에서 확인할 코드
- fix 직전 output loop에서 `EINTR`가 어떤 branch로 permanent failure가 되는지 확인합니다.
- fix SHA의 `ft_printf_write`에서 request length를 `SSIZE_MAX`로 cap하는 계산, `write` call, return-value 분기를 순서대로 추적합니다.
- positive return, `EINTR`, zero return, other error 각각에 대해 buffer/remaining/count/error가 변하는지 state table로 기록합니다.
- `EINTR` retry에서는 count와 remaining range가 유지되는지 실제 mutation 위치로 확인합니다.
- production build와 deterministic test build가 system-call boundary만 바꾸도록 `FT_PRINTF_TEST_WRITE` seam이 적용되는 위치를 확인합니다.
- 후속 `1223518652bd`의 scripted writer가 이 seam을 통해 어떤 production branch를 통과하는지 연결합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_output.c`: `<errno.h>`, `<stdint.h>`/range macro 사용, `FT_PRINTF_TEST_WRITE`에 따른 `FT_PRINTF_SYSTEM_WRITE`, `request`, EINTR branch, terminal nonpositive branch.
  - 상태표: positive=`count/pointer/remaining` 진행, EINTR=모두 유지, zero/other negative=`error=1`, 이후 호출은 첫 guard에서 실패합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 8a3ec50cb689, src/ft_output.c, ft_printf_write */
request = length;
if (request > (size_t)SSIZE_MAX)
    request = (size_t)SSIZE_MAX;
written = FT_PRINTF_SYSTEM_WRITE(ctx->fd, buffer, request);
if (written < 0 && errno == EINTR)
    continue ;
if (written <= 0)
{
    ctx->error = 1;
    return (-1);
}
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `write(ctx->fd, buffer, length)`와 `written <= 0` 하나로 interruption도 terminal failure였습니다.
  - 이후: bounded request와 `errno == EINTR` retry가 positive/terminal branch보다 앞서며, build macro가 call site만 test seam으로 바꿉니다.

### Failure → Fix 추적
- 기존 가정/상태: positive short write만 resume하면 충분하고 interrupted call은 ordinary failure로 처리해도 된다는 상태
- 실제 failure 또는 위험: `EINTR`가 progress 없이 permanent failure가 되고 `size_t` remaining request가 `SSIZE_MAX`보다 클 수 있음
- source가 지목한 root cause: interruption과 permanent error를 구분하지 않고 system call signed result range를 request size에 반영하지 않음
- 수정된 decision/invariant: `SSIZE_MAX` request cap + `EINTR` transparent retry + nonpositive non-`EINTR` sticky failure + deterministic seam
- 학습자 기록 — 실제 수정 코드:
  - 위 발췌처럼 request cap과 EINTR branch가 mutation 전에 추가되고, 실제 call은 `FT_PRINTF_SYSTEM_WRITE` macro를 통합니다.
- 학습자 기록 — regression test 연결:
  - `1223518652bd`의 partial case는 `WRITE_PART(2) → WRITE_ALL`, interrupt case는 `EINTR → PART(3) → EINTR → ALL`, failure case는 immediate/partial-then-`EPIPE`와 zero를 주입합니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - POSIX write boundary를 완성한 핵심 fix입니다. “성공 byte만 state를 진행시키고, interruption은 그대로 재시도하며, progress 불가능한 결과만 terminal error”라는 규칙과 이를 검증할 seam을 같은 call site에 고정했습니다.

## 5.6 `22e65c176b5d` — perf(output): 반복 채움을 묶어서 출력

- Importance: `A`
- Tags: `OUTPUT, PERF`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Emits wide padding in bounded chunks rather than one system call per byte.
- Commit Classification summary: Emits repeated padding through bounded 64-byte chunks instead of one write per character.
- Importance 근거: The change materially reduces system-call count for wide fields while preserving bounded stack use and failure propagation. Later fault tests make the cost model observable.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - `ft_printf_putnchar`가 길이만큼 `ft_printf_putchar`를 호출하여 wide width가 한 byte당 한 output call/system call 비용으로 이어졌습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - producer는 고정 64-byte local buffer를 한 번 채우고 남은 길이에 따라 64 이하 chunk를 `ft_printf_write`에 넘깁니다. system-call semantics의 owner는 바꾸지 않습니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - 성능 모델만 O(width) one-byte calls에서 약 `ceil(width/64)` shared writes로 바뀌고, count/error/progress는 기존 context에서 동일하게 처리됩니다.
- 학습자 기록 — failure 또는 edge case:
  - negative/zero length는 loop를 실행하지 않습니다. 마지막 chunk는 64보다 작을 수 있으며, 어느 chunk에서든 shared write가 실패하면 즉시 `-1`입니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: stack 사용량은 width와 무관하게 64 bytes로 제한되고, 각 요청은 64 이하이며 공통 failure path를 유지합니다.
  - 미보장: 정확한 OS syscall 횟수 자체는 short write/EINTR에 따라 늘 수 있습니다. commit 자체에는 call-count assertion이 없습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `1223518652bd`가 width 1000에서 scripted writer call 수 17과 largest request 64를 확인합니다.

### 해당 SHA에서 확인할 코드
- 이 SHA에서 repeated-character output이 one-byte loop에서 bounded stack chunk로 바뀐 함수와 diff를 찾습니다.
- 64-byte buffer 준비, 남은 padding에 따른 chunk length 결정, 반복 호출 순서를 기록합니다.
- 각 chunk가 직접 `write`하지 않고 shared `ft_printf_write`를 통과하는지 확인하여 count/error semantics가 유지되는 근거를 기록합니다.
- width가 매우 커도 stack usage가 user width와 비례하지 않는 이유를 실제 local storage 크기로 확인합니다.
- 후속 `1223518652bd`의 width 1000 test가 call count와 64-byte maximum을 어떻게 관찰하는지 연결합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_output.c`, `ft_printf_putnchar`: `char buffer[64]`, fill loop, `chunk = min(length, 64)`, `ft_printf_write`, `length -= chunk`.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 22e65c176b5d, src/ft_output.c, ft_printf_putnchar */
chunk = length;
if (chunk > (int)sizeof(buffer))
    chunk = (int)sizeof(buffer);
if (ft_printf_write(ctx, buffer, (size_t)chunk) < 0)
    return (-1);
length -= chunk;
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `while (length-- > 0) ft_printf_putchar(ctx, c)` 형태의 one-byte 반복이었습니다.
  - 이후: 고정 buffer와 bounded chunk 반복으로 바뀌되, 최종 callee는 계속 `ft_printf_write`입니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - output invariant를 건드리지 않고 padding producer의 단위만 키운 성능 개선입니다. 고정 크기 stack buffer이므로 width가 커져도 메모리 사용은 증가하지 않고, failure/count는 기존 state machine을 그대로 통과합니다.

## 5.7 `1223518652bd` — test(output): 쓰기 실패 시퀀스와 채움 전략 검증

- Importance: `A`
- Tags: `OUTPUT, TEST, RISK`
- Most Important Commits 목록: 포함
- Thread 내 역할: Scripts partial progress, interruption, zero writes, `EPIPE`, and verifies `SIGPIPE` and chunking policy.
- Commit Classification summary: Injects partial writes, EINTR, EPIPE, zero progress, and verifies SIGPIPE and padding chunks.
- Importance 근거: This provides deterministic evidence for the S-level output state machine and confirms that the library does not mutate process signal policy. It is unusually strong failure-path verification.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - retry/chunk 정책은 구현돼 있었지만 실제 kernel timing으로는 원하는 partial/EINTR/EPIPE 순서를 재현하기 어렵고, SIGPIPE ownership과 accepted-prefix 보존을 명시적으로 증명하지 못했습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - production source를 `FT_PRINTF_TEST_WRITE`로 다시 compile한 별도 fault binary와 실제 archive를 쓰는 normal test를 함께 둡니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production code는 바뀌지 않습니다. 테스트가 scripted step, call count, 최대 request, accepted output을 관찰 가능한 상태로 만들어 기존 invariant를 고정합니다.
- 학습자 기록 — failure 또는 edge case:
  - PART→ALL, EINTR/PART/EINTR/ALL, immediate EPIPE, PART→EPIPE, zero result, width 1000, real broken pipe를 각각 분리합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 지정한 deterministic sequence에서 offset/retry/stop/chunk behavior와 caller handler 보존이 assertion과 일치해야 suite가 통과합니다.
  - 미보장: 모든 OS scheduler 조합, 비동기 signal 전체, 이미 kernel이 받은 bytes의 rollback 가능성은 증명하지 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - 이후 sanitizer/release targets가 이 fault suite도 다시 compile/run하여 runtime와 artifact 검증 축에 포함합니다.

### 해당 SHA에서 확인할 코드
- fault binary가 `FT_PRINTF_TEST_WRITE` seam을 사용하도록 build되는 실제 Makefile rule/compile definition을 기록합니다.
- scripted writer가 configured return sequence, request length, accepted bytes, call record를 어떤 상태로 보관하는지 확인합니다.
- full write, short write, `EINTR`, zero, `EPIPE` case마다 production `ft_printf_write`의 어떤 branch를 통과하는지 매핑합니다.
- partial failure 이전에 accepted된 bytes가 exact하게 남고 이후 write가 중단되는지 assertion을 확인합니다.
- width 1000 padding case의 request count와 maximum chunk 64 assertion을 확인합니다.
- real broken-pipe test에서 caller-owned `SIGPIPE` handler 설치/복원과 `ft_printf` return을 함께 확인합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `Makefile`: `tests/test_output_faults.c $(SRC)`를 `-DFT_PRINTF_TEST_WRITE`로 compile하여 `tests/bin/test_output_faults`를 만들고 실행합니다.
  - `tests/test_output_faults.c`: `t_write_step`, global step/call/output state, `ft_printf_test_write`, retry/failure/padding case 함수가 production seam을 구동합니다.
  - `tests/test_ft_printf.c`: pipe/dup2 기반 capture와 real broken-pipe `SIGPIPE` handler case가 archive path를 검사합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 1223518652bd, tests/test_output_faults.c, run_retry_cases */
reset_writer();
add_step(WRITE_PART, 2);
add_step(WRITE_ALL, 0);
expect_success("partial", 2, 7);
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: normal differential/error test만 있었고 writer return sequence 및 request 크기를 통제하지 못했습니다.
  - 이후: 별도 fault executable이 production sources를 seam-enabled로 compile하고, normal suite에는 real SIGPIPE policy case가 추가됩니다.

### Test commit 학습 기록
- production invariant 대상: positive short write progress, `EINTR` retry, zero/permanent failure stop, `SIGPIPE` ownership boundary, 64-byte padding chunk policy
- 재현하는 failure / boundary: scripted partial write, `EINTR`, zero-byte result, `EPIPE`; 별도의 real broken pipe
- test technique: compile-time write seam을 통한 deterministic fault injection + call/emitted-byte recording + real signal-policy case
- 통과하는 production path: `ft_printf` → padding/conversion output → `ft_printf_write` → substituted writer 또는 real `write`
- 이 test가 source상 증명하려는 것: retry offset, no-progress handling, hard-failure stop, prior accepted bytes, chunk bound, caller-owned signal disposition
- 이 test가 증명하지 않는 것: 모든 OS scheduling/timing behavior나 이미 OS가 받아들인 byte의 rollback을 증명하지 않습니다.
- 분류: deterministic regression/fault-injection 중심이며 signal policy는 real integration boundary case를 포함합니다.
- 후속 회귀 방지 역할: output loop, retry policy, padding optimization이 바뀌어도 동일 state transition과 signal boundary를 유지하도록 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - `ft_printf_test_write`는 step마다 call 수와 largest request를 기록하고, PART/ALL에서 실제 accepted bytes를 capture buffer에 복사합니다. retry cases는 최종 exact bytes와 call 수, failure cases는 `-1`과 accepted prefix, padding case는 length 1000·17 calls·largest 64를 검사합니다. normal suite의 signal case는 caller handler 호출 수와 설치 상태를 모두 검사합니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: 현재 실행 환경에서 GitHub host DNS/checkout이 차단되어 exact SHA source tree를 로컬로 구성할 수 없었습니다.
  - result: 실행 결과를 주장하지 않습니다. `1223518652bd`의 Makefile과 두 test source는 exact commit diff/code inspection으로 확인했습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 드문 write 결과를 운에 맡기지 않고 production call boundary에 순서대로 주입하는 회귀 suite입니다. accepted bytes와 요청 단위까지 기록해 state transition을 검증하며, 실제 broken pipe에서는 library가 caller의 SIGPIPE disposition을 바꾸지 않는다는 별도 책임 경계도 확인합니다.

## 6. Invariant ledger

Source가 확정한 변화 축을 아래에 배치했습니다. “실제 코드 근거”는 학습자가 해당 SHA를 읽고 채웁니다.

| Invariant / concern | 도입 또는 초기 상태 | 강화 / 수정 | 고정한 검증 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| 공유 count/error state | `3f7b0ab926d0`에서 `t_printf`로 중앙화 | `c627bd1f85bb`에서 wide `ssize_t`를 `int`로 narrow하기 전 검증을 강화 | `1223518652bd`에서 fault sequence로 state transition 검증 | `src/ft_printf_internal.h`의 `t_printf`; `src/ft_output.c`의 init/write 및 `written > INT_MAX` guard; fault writer의 result/output assertions |
| write progress policy | `1d6a5cee3041`은 short write를 failure로 처리 | `3f7b0ab926d0`은 positive short write 진행을 도입하고 `8a3ec50cb689`는 `EINTR`/`SSIZE_MAX` 정책을 완성 | `1223518652bd`에서 partial/interrupt/zero/permanent failure를 deterministic하게 검증 | write loop의 pointer/length mutation은 positive branch 뒤에만 있고 EINTR은 `continue`; PART/EINTR/EPIPE/ZERO scripted cases가 각 branch를 통과 |
| padding cost model | `22e65c176b5d`에서 64-byte bounded chunk 도입 | 공통 `ft_printf_write` path 유지 | `1223518652bd`에서 width 1000의 call count와 최대 chunk 확인 | `ft_printf_putnchar`의 `char buffer[64]` 및 min chunk; fault test의 1000 bytes, 17 calls, largest request 64 assertions |
| signal ownership | `8a3ec50cb689`의 output error policy와 연결 | library가 `SIGPIPE` handler를 대체하지 않는 boundary 유지 | `1223518652bd`의 real broken-pipe test로 확인 | production output은 `write` error만 sticky state로 변환하며 signal API를 호출하지 않음; test가 caller handler 설치·호출·유지·복원을 검사 |

### 학습자 추가 기록

- source가 명시한 invariant 범위 안에서만 필요한 행을 추가합니다. 새 invariant를 확정 사실처럼 만들지 않습니다.
- 추가 기록:
  - 별도 행 추가는 필요하지 않습니다. 이미 네 행이 이 Thread의 source-defined count/progress/cost/signal 축을 모두 포함합니다.

## 7. Failure → Fix → Test 연결

| 기존 failure / risk | Fix / change | 수정 decision | Test / 학습 확인 |
| --- | --- | --- | --- |
| short write를 progress로 다루지 못함 | `3f7b0ab926d0` | positive short write 이후 남은 suffix 재시도 | `1223518652bd`에서 scripted partial write로 최종 검증 |
| wide `ssize_t`를 먼저 `int`로 cast할 위험 | `c627bd1f85bb` | narrowing 전에 개별 result 범위를 검증 | 학습자가 후속 test suite에서 이 경계의 직접/간접 검증 범위를 구분해 기록 |
| `EINTR`를 permanent failure로 취급하고 request가 `SSIZE_MAX`를 넘을 수 있음 | `8a3ec50cb689` | `EINTR` retry + request cap + test seam | `1223518652bd`에서 scripted `EINTR`, partial, zero, `EPIPE` 검증 |
| padding byte마다 system call 발생 | `22e65c176b5d` | stack의 bounded 64-byte chunk로 반복 출력 | `1223518652bd`에서 chunk size와 call count 검증 |

- 학습자 기록 — 실제 failure branch와 regression assertion을 연결한 추가 설명:
  - `written <= 0`는 EINTR branch 뒤에 있어 zero/EPIPE만 terminal error가 되고, partial result는 pointer/remaining을 진행시킵니다. fault suite는 exact final output 또는 accepted prefix와 call count를 검사합니다. `written > INT_MAX` guard는 code inspection으로 확인했으나 direct injected regression은 없습니다.

## 8. Ownership / state / responsibility 변화

| 시점 | Source상 owner / boundary | Source상 responsibility 변화 | 해당 SHA 코드 근거 |
| --- | --- | --- | --- |
| 초기 | `ft_printf` 내부/local helper | descriptor write, count, literal/percent 처리의 일부가 entry point 주변에 함께 존재 | `1d6a5cee3041`의 `src/ft_printf.c`: local `count`, `ft_write_count`, fd 1 고정 |
| `3f7b0ab926d0` | `t_printf` + output API | descriptor, count, sticky error를 하나의 private context가 소유 | `src/ft_printf_internal.h`의 세 field와 `src/ft_output.c`의 initializer/write API |
| `78e5d25d7df6` | `ft_printf` vs output context | entry point는 traversal/variadic lifecycle/final result translation에 집중하고 output semantics는 context로 이동 | `src/ft_printf.c`에서 local helper 삭제, `ft_printf_putchar(&ctx, ...)`, final `ctx.error/count` translation |
| `8a3ec50cb689` 이후 | `ft_printf_write` system-call boundary | request sizing, retry, progress, permanent failure의 책임이 한 write loop에 모임 | `request = min(length, SSIZE_MAX)`, EINTR continue, positive-only mutation, terminal sticky error |
| `22e65c176b5d` 이후 | padding producer vs output state | padding producer는 chunk를 만들고 실제 progress/count/failure는 기존 output API가 계속 소유 | `ft_printf_putnchar`의 64-byte local buffer가 각 chunk를 `ft_printf_write`에 전달 |

## 9. Thread 최종 상태

- Source가 확정한 도달점: 모든 conversion이 공유하는 output state machine에서 count range, progress, interruption, permanent failure, syscall cost, signal policy가 명시되고 별도로 검증된 상태입니다.
- 학습자 기록 — 마지막 commit 기준 실제 코드에서 확인한 최종 state:
  - `t_printf`가 fd/count/error를 소유하고 모든 producer가 `ft_printf_write` 또는 그 wrapper를 사용합니다. write loop는 bounded request, EINTR retry, positive-only progress, representability/누적 count 검사, zero/hard-error sticky stop을 적용합니다. padding은 64-byte chunk로 들어오며 fault test가 sequence와 request 크기를 관찰합니다.
- 학습자 기록 — 이 Thread 밖에서만 해결되는 남은 문제를 source 범위 안에서 구분:
  - format 전체의 사전 유효성/총 길이 원자성은 Thread 5의 preflight에서 다룹니다. numeric/text formatting 의미와 release/sanitizer artifact 검증도 각각 다른 Thread의 책임입니다. output failure 뒤 이미 accepted된 bytes는 어떤 후속 Thread도 rollback하지 않습니다.

## 10. 최종 architecture 또는 execution flow 정리

실제 SHA 코드를 읽은 뒤 아래 흐름을 완성합니다. source 설명만 복사하지 말고 함수/상태/branch를 연결합니다.

```text
[caller / ft_printf]
    -> [stack t_printf 초기화, conversion/literal producer]
    -> [ft_printf_putchar / ft_printf_putnchar / ft_printf_write]
    -> [request <= SSIZE_MAX, EINTR retry, positive progress 또는 sticky error]
    -> [ctx.error ? -1 : ctx.count]
```

- 각 단계에 대응하는 SHA / file / function:
  - `78e5d25d7df6` `src/ft_printf.c::ft_printf` → `3f7b0ab926d0`/`8a3ec50cb689` `src/ft_output.c::ft_printf_write` → `22e65c176b5d` `ft_printf_putnchar`; `1223518652bd`의 fault/real-write tests가 경계를 검증합니다.
- 핵심 state transition:
  - 초기 `{count=0,error=0}`에서 positive result마다 `{count += written, buffer += written, remaining -= written}`; EINTR은 동일 상태; zero/non-EINTR negative/range 위반은 `{error=1}` terminal state입니다.
- failure가 끊기는 지점:
  - `ft_printf_write`가 sticky error를 설정하고 `-1`을 반환하며, caller가 연쇄 반환합니다. 이후 output call은 `ctx->error` 첫 guard에서 system call 없이 실패합니다.
- 후속 fix/test가 보장한 지점:
  - narrowing 순서, request cap/EINTR policy, 64-byte padding unit, accepted-prefix/no-rollback, SIGPIPE disposition 비소유가 각각 fix와 deterministic/real tests로 고정됩니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 정확한 시점의 코드로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 source와 그대로 유지했습니다.
- [x] final HEAD의 코드를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] 필요한 parent/직전 관련 SHA 비교를 실제 diff로 수행했습니다.
- [x] source가 확정한 사실과 내가 코드에서 확인한 사실을 구분했습니다.
- [x] fix의 기존 가정 → failure/risk → root cause → decision → code → test 연결을 필요한 곳에서 완성했습니다.
- [x] test commit의 target invariant, technique, production path, proves/not-proves를 구분했습니다.
- [x] Invariant ledger에 실제 코드 근거를 채웠습니다.
- [x] 이 Thread의 최종 architecture/execution flow를 commit history 순서로 설명할 수 있습니다.
===== END FILE: 01-output-state-system-call-boundary.md =====

===== BEGIN FILE: 02-format-fields-typed-dispatch.md =====
# From format fields to typed conversion dispatch

## 1. Thread 목표

raw format text를 `t_format`으로 정규화하고, main traversal, typed `va_arg` dispatch, conversion renderer 사이의 책임 경계가 형성되는 과정을 복원합니다.

### Source에서 확정된 significance

parsing, argument extraction, rendering이 분리된 책임이 됩니다. 정규화된 field가 각 conversion의 raw format 재해석을 막고, dispatch가 specifier별 정확한 promoted type 소비를 중앙화합니다. parser 단계의 flag normalization은 renderer가 처리해야 할 충돌 상태를 줄입니다.

### 이 Thread에 명시적으로 연결되는 source invariant / engineering difficulty

- Invariant: `t_format`은 parser에서 measurement와 rendering으로 전달되는 normalized field representation입니다.
- Invariant: variadic argument는 conversion이 정의한 promoted type과 호환되는 방식으로 소비되어야 합니다.
- Invariant: format grammar와 overflow behavior는 main loop, 이후 measurement, renderer 사이에서 일관되어야 합니다.
- Engineering difficulty: normalized field grammar와 overflow behavior를 여러 소비자 사이에서 일관되게 유지하고, specifier마다 정확한 promoted type을 소비하도록 책임을 분리하는 문제입니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `t_format`에는 어떤 상태가 저장되고 omitted precision과 `.0`은 어떻게 구분되는가?
- parser는 field의 어느 부분까지 소비하고 main loop는 parser가 반환한 cursor를 어떻게 사용하는가?
- `va_arg`의 promoted type 선택은 왜 main loop가 아니라 dispatch가 소유하는가?
- decimal, hex, pointer conversion은 기존 parser/dispatch/output boundary 안에 어떻게 추가되는가?
- 충돌 flag를 한 번 정규화하면 renderer의 상태 공간이 어떻게 줄어드는가?

## 3. 완료 기준

- 해당 SHA의 실제 `t_format` field와 parser 흐름을 raw format grammar 순서대로 추적할 수 있습니다.
- main traversal, parser, dispatch, renderer의 caller/callee 관계를 실제 함수명으로 그릴 수 있습니다.
- 각 specifier가 어떤 promoted type으로 `va_arg`를 소비하는지 해당 dispatch 코드에서 확인해 기록할 수 있습니다.
- `-`/`0`, `+`/space 충돌과 signed/alternate prefix 선택이 어느 책임 경계에서 해결되는지 설명할 수 있습니다.

## 4. Commit map

| SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- |
| `7984ddf2dd57` | feat(parser): 포맷 필드 모델과 해석기 추가 | `S` | `ARCH, PARSER, CORE` | Establishes the normalized `t_format` representation and overflow-checked field parser. |
| `9e6d785628f3` | feat(core): 포맷 필드 해석을 출력 루프에 연결 | `B` | `PARSER, INTEGRATION` | Connects field parsing to the main format traversal. |
| `03c3e6e09fa1` | feat(text): 문자·문자열·퍼센트 변환 추가 | `A` | `ARCH, FORMAT, VARARGS` | Introduces a dispatcher that owns `va_arg` type selection and routes to conversion renderers. |
| `95d6613a1c72` | feat(decimal): 부호 있는·없는 10진수 출력 추가 | `B` | `FORMAT, VARARGS` | Adds signed and unsigned decimal conversions inside that boundary. |
| `93c883070a1b` | feat(hex): 16진수와 포인터 출력 추가 | `B` | `FORMAT, VARARGS` | Adds hexadecimal and pointer conversions inside the same boundary. |
| `c5f627099ad9` | feat(flags): 숫자 플래그 우선순위 정규화 | `A` | `PARSER, FORMAT, LAYOUT` | Normalizes conflicting flags once and applies signed and alternate-form prefixes. |

## 5. Commit별 학습 기록

> 원칙: 아래 기록은 final HEAD가 아니라 각 항목의 정확한 SHA에서 작성합니다. source가 확정하지 않은 파일명/함수명은 현재 골격에서 추측하지 않습니다.

## 5.1 `7984ddf2dd57` — feat(parser): 포맷 필드 모델과 해석기 추가

- Importance: `S`
- Tags: `ARCH, PARSER, CORE`
- Most Important Commits 목록: 포함
- Thread 내 역할: Establishes the normalized `t_format` representation and overflow-checked field parser.
- Commit Classification summary: Defines t_format and parses flags, width, precision, and specifiers with decimal overflow checks.
- Importance 근거: The parser creates the durable representation through which all conversions and both later passes communicate. It is indispensable to explaining the formatter's grammar and field-processing architecture.

### 학습 깊이
- 이 commit은 architecture/invariant의 핵심으로 취급합니다.
- 학습자 기록 — 직전 상태:
  - main loop는 literal과 `%%`만 직접 구분했으며, `%` 뒤의 flags/width/precision/specifier를 표현하는 공통 상태가 없었습니다.
- 학습자 기록 — 해결하려던 문제:
  - 각 conversion이 raw format을 다시 읽지 않도록 한 field를 한 번 해석해 전달하고, decimal field가 `int` 범위를 넘는 경우 cursor/state가 잘못 진행되지 않도록 해야 했습니다.
- 학습자 기록 — 기존 설계가 충분하지 않았던 이유:
  - raw cursor만으로는 omitted precision과 `.0`을 구분할 수 없고, 이후 renderer와 measurement가 같은 grammar를 재구현하면 서로 다른 결과와 다른 argument consumption이 생길 수 있습니다.
- 학습자 기록 — 선택한 핵심 decision:
  - private `t_format`에 `flags`, `width`, `precision`, `has_precision`, `spec`를 저장하고 `ft_printf_parse`가 flags→width→optional precision→specifier 순서로 한 field를 소비한 뒤 next cursor를 반환하도록 했습니다.
- 학습자 기록 — ownership / lifecycle / state transition:
  - caller가 stack `t_format`을 넘기고 parser가 먼저 zero/default state로 초기화합니다. flag는 OR로 누적되고, `.`를 만나면 값이 0이어도 `has_precision = 1`이 됩니다. parse 성공 시 `spec`와 next unread pointer가 확정됩니다.
- 학습자 기록 — failure scenario와 public consequence:
  - width 또는 precision 누적 전에 `value > (INT_MAX - digit) / 10`이면 parser가 null을 반환합니다. 이 SHA에서는 parser가 아직 public loop에 연결되지 않아 public consequence는 다음 integration commit에서 생깁니다.
- 학습자 기록 — 이 SHA가 보장하는 것:
  - parser API를 사용하는 caller는 repeated flags를 idempotent bit set으로 받고, omitted precision과 explicit zero를 구분하며, field integer overflow를 검출할 수 있습니다.
- 학습자 기록 — 아직 보장하지 않는 것:
  - main traversal integration, supported-specifier validation, flag conflict normalization, typed `va_arg` dispatch는 아직 없습니다.
- 학습자 기록 — 후속 fix/test로 이어지는 지점:
  - `9e6d785628f3`이 cursor/error를 main loop에 연결하고, `03c3e6e09fa1`이 dispatch를 추가합니다. `c5f627099ad9`이 parsed flag를 canonicalize하며, Thread 5의 preflight가 동일 parser를 measurement에도 사용합니다.

### 해당 SHA에서 확인할 코드
- 해당 SHA의 `t_format` 정의에서 flag bit set, width, precision value, `has_precision`, specifier에 대응하는 실제 field를 기록합니다.
- parser가 flags → width → optional precision → specifier 순서로 cursor를 이동하는 실제 함수 호출 흐름을 추적합니다.
- repeated flag가 bitwise OR로 idempotent하게 누적되는 지점을 확인합니다.
- width/precision decimal parsing에서 multiply/add 전에 `INT_MAX` overflow를 차단하는 조건식을 기록합니다.
- precision omitted와 `.0`을 `has_precision`로 구분하는 state transition을 확인합니다.
- parser가 반환하는 next unread position을 caller가 아직 사용하지 않는 이 SHA의 boundary와, 직후 `9e6d785628f3`에서의 integration을 비교할 준비를 합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf_internal.h`: flag constants와 `t_format` 다섯 field, parser declarations.
  - `src/ft_parse.c`: `ft_flag_value`, decimal parser, `ft_printf_init_format`, `ft_printf_parse`. `while (ft_flag_value(*format))`에서 repeated flag를 OR하고, decimal helper를 width와 precision에 재사용합니다.
  - `Makefile`: `src/ft_parse.c`를 archive source에 추가하지만 `src/ft_printf.c`는 이 commit에서 parser를 호출하지 않습니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 7984ddf2dd57, src/ft_parse.c, ft_parse_decimal */
while (ft_is_digit(**format))
{
    digit = **format - '0';
    if (*value > (INT_MAX - digit) / 10)
        return (-1);
    *value = *value * 10 + digit;
    (*format)++;
}
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `src/ft_printf.c`가 raw `%`/`%%` branch만 가졌고 field state가 없었습니다.
  - 이후: 별도 parser module과 normalized `t_format`이 생겼지만 public traversal은 아직 기존 동작을 유지합니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - formatter grammar의 공통 데이터 모델을 만든 S-level commit입니다. 한 field의 raw bytes를 bounded integer와 명시적 optional-precision 상태로 변환하지만, 이 시점에는 독립 모듈 도입 단계라 public 출력 경로를 아직 바꾸지 않습니다.

## 5.2 `9e6d785628f3` — feat(core): 포맷 필드 해석을 출력 루프에 연결

- Importance: `B`
- Tags: `PARSER, INTEGRATION`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Connects field parsing to the main format traversal.
- Commit Classification summary: Connects parsed fields to the main traversal with temporary fallback output.
- Importance 근거: This is normal integration of the parser into the loop; actual conversion responsibility and final invalid-format policy are established later.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - parser가 archive에는 들어갔지만 호출되지 않아 normalized field와 overflow failure가 public behavior에 영향을 주지 않았습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - `%`를 만난 main loop가 `ft_printf_parse(format + 1, &fmt)`를 호출하고, 성공 시 반환 cursor를 다음 traversal 위치로 사용합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - null parser result는 `ctx.error = 1` 후 loop를 중단합니다. 성공 field는 임시 fallback으로 `%`와 specifier를 다시 출력하며, 기존 special `%%` branch는 parser보다 먼저 처리됩니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - temporary echo는 `03c3e6e09fa1`의 typed dispatch로 대체됩니다. unsupported/trailing field를 whole-call no-output error로 확정하는 정책은 Thread 5의 preflight 이전에는 없습니다.

### 해당 SHA에서 확인할 코드
- main loop가 `%` field를 parser에 넘기고 parser가 반환한 cursor로 traversal을 전진시키는 지점을 기록합니다.
- parse failure가 output context의 sticky error로 승격되는 path를 확인합니다.
- dedicated dispatch가 아직 없기 때문에 temporary rendering이 percent와 available specifier를 echo하는 코드를 찾고, 무엇이 아직 final syntax policy가 아닌지 기록합니다.
- parser와 main loop의 responsibility를 “field consumption”과 “overall sequencing/termination”으로 실제 code boundary에 대응시킵니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf.c`, `ft_printf`: `t_format fmt`; `%` branch에서 parser 호출, null이면 `ctx.error = 1`; 성공하면 `format`을 returned pointer로 바꾸고 `%`/`fmt.spec`를 shared output으로 임시 출력합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 9e6d785628f3, src/ft_printf.c */
format = ft_printf_parse(format + 1, &fmt);
if (format == 0)
{
    ctx.error = 1;
    break ;
}
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - parser를 main sequencing에 연결한 integration commit입니다. parser가 한 field의 소비 범위와 실패를 결정하고 main loop는 반환 cursor와 sticky error를 관리하지만, conversion semantics는 아직 echo fallback뿐입니다.

## 5.3 `03c3e6e09fa1` — feat(text): 문자·문자열·퍼센트 변환 추가

- Importance: `A`
- Tags: `ARCH, FORMAT, VARARGS`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Introduces a dispatcher that owns `va_arg` type selection and routes to conversion renderers.
- Commit Classification summary: Introduces conversion dispatch and concrete c, s, and percent renderers.
- Importance 근거: Separating va_arg selection from rendering becomes the integration boundary used by every later conversion. It is significant architecture, though subordinate to the parser and two-pass core.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - main loop가 parsed specifier를 직접 echo했고, 어떤 specifier가 argument를 어떤 type으로 소비하는지 소유하는 경계가 없었습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - `ft_printf_dispatch(ctx, fmt, va_list *)`가 specifier selection과 `va_arg`를 소유하고, text renderer는 이미 추출된 값과 normalized field만 받도록 분리했습니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - `c`는 default promotion에 맞춰 `int`, `s`는 `char *`를 소비하고 `%`는 argument를 소비하지 않습니다. renderer는 모두 같은 output context를 사용합니다.
- 학습자 기록 — failure 또는 edge case:
  - null string은 renderer에서 `(null)`로 치환됩니다. unknown/trailing specifier는 아직 `%`와 spec을 literal로 쓰는 fallback이므로 final validity rejection은 아닙니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 지원된 text conversion의 type extraction과 rendering 책임이 분리되고 main entry에는 type-specific `va_arg`가 남지 않습니다.
  - 미보장: decimal/hex/pointer, width/precision layout 전체, unsupported syntax의 whole-call rejection은 아직 없습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `95d6613a1c72`와 `93c883070a1b`이 같은 dispatch boundary에 numeric types를 추가하고, 후속 text/layout commits가 renderer semantics를 확장합니다.

### 해당 SHA에서 확인할 코드
- conversion dispatcher의 실제 함수와 main loop call site를 찾습니다.
- `c`, `s`, `%` 각각에서 `va_arg` 수행 여부를 확인하고, argument를 소비하는 경우 어떤 promoted type/argument form으로 renderer에 전달되는지 기록합니다.
- null string이 `(null)` representation으로 mapping되는 branch와 shared output path를 확인합니다.
- unknown field가 아직 prior literal fallback을 유지하는 branch를 찾아 final syntax validation이 아직 아님을 기록합니다.
- entry point에 type-specific `va_arg`가 남아 있는지 diff로 확인합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf.c`: parse 성공 뒤 `ft_printf_dispatch(&ctx, &fmt, &args)`만 호출합니다.
  - `src/ft_dispatch.c`: `c`, `s`, `%`, fallback branches와 두 `va_arg` call.
  - `src/ft_text.c`: `ft_printf_print_char`, `ft_printf_print_string`, `ft_printf_print_percent`; null string mapping 및 shared write.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 03c3e6e09fa1, src/ft_dispatch.c */
if (fmt->spec == 'c')
    return (ft_printf_print_char(ctx, fmt, va_arg(*args, int)));
if (fmt->spec == 's')
    return (ft_printf_print_string(ctx, fmt, va_arg(*args, char *)));
if (fmt->spec == '%')
    return (ft_printf_print_percent(ctx, fmt));
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `ft_printf`가 parsed field를 `%`와 specifier로 직접 출력했습니다.
  - 이후: main은 dispatcher만 호출하고, dispatcher가 promoted type을 추출해 conversion renderer로 전달합니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - variadic extraction을 conversion routing 지점에 중앙화한 A-level boundary commit입니다. parser는 field를 만들고, dispatch는 type을 소비하며, renderer는 값 표현과 output만 처리하는 기본 분리가 이때 성립합니다.

## 5.4 `95d6613a1c72` — feat(decimal): 부호 있는·없는 10진수 출력 추가

- Importance: `B`
- Tags: `FORMAT, VARARGS`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Adds signed and unsigned decimal conversions inside that boundary.
- Commit Classification summary: Adds d, i, and u dispatch and decimal digit emission.
- Importance 근거: This is normal core-feature implementation within the established dispatch and output model.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - dispatch와 text conversions만 있어 `d`, `i`, `u`가 fallback으로 처리됐고 numeric argument를 올바른 type으로 소비하지 않았습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - dispatch에 signed `int`와 `unsigned int` extraction을 추가하고, 공통 unsigned decimal digit routine으로 값을 출력합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - local fixed buffer에 least-significant digit부터 저장한 뒤 index를 역방향으로 `ft_printf_putchar`에 넘깁니다. signed renderer는 negative sign을 먼저 쓰고 magnitude를 unsigned routine에 전달합니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - width/prefix/layout은 후속 commits에서 sign을 prefix representation으로 바꾸며, `ed3750fd081a`가 `INT_MIN` magnitude 계산의 `long` width 가정을 제거합니다.

### 해당 SHA에서 확인할 코드
- dispatch에 `d`, `i`, `u` case가 추가되는 위치와 각 `va_arg` type을 확인합니다.
- 하나의 unsigned digit routine이 signed magnitude와 unsigned value 모두에 사용되는 caller/callee 관계를 기록합니다.
- least-significant-first로 fixed local buffer에 digits를 만든 뒤 reverse emission하는 loop를 추적합니다.
- negative `int`를 `long`으로 widen한 뒤 negation하는 magnitude path와 sign emission 분리를 확인합니다.
- 이 시점의 `INT_MIN` 처리에 `long`이 `int`보다 넓다는 portability assumption이 남아 있음을 코드와 type model로 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_dispatch.c`: `d`/`i`에서 `va_arg(*args, int)`, `u`에서 `va_arg(*args, unsigned int)`.
  - `src/ft_number.c`: unsigned digit helper, `ft_printf_print_signed`, `ft_printf_print_unsigned`. negative path는 `(long)number`, `-value`를 사용합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 95d6613a1c72, src/ft_number.c */
value = (long)number;
if (value < 0)
{
    if (ft_printf_putchar(ctx, '-') < 0)
        return (-1);
    value = -value;
}
return (ft_print_unsigned_digits(ctx, (unsigned long)value));
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - established dispatch/output 경계 안에 decimal conversion을 추가한 구현 commit입니다. type extraction은 정확하지만, `long`이 `int`보다 넓지 않은 구현에서는 `INT_MIN` negation이 안전하지 않다는 후속 portability 과제가 남습니다.

## 5.5 `93c883070a1b` — feat(hex): 16진수와 포인터 출력 추가

- Importance: `B`
- Tags: `FORMAT, VARARGS`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Adds hexadecimal and pointer conversions inside the same boundary.
- Commit Classification summary: Adds x, X, and p formatting with uintptr_t conversion.
- Importance 근거: The feature completes the base conversion set but follows existing dispatch and output boundaries.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - `x`, `X`, `p`가 지원되지 않았고, unsigned integer와 pointer를 base 16 representation으로 변환하는 renderer가 없었습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - dispatch가 `x`/`X`의 `unsigned int`, `p`의 `void *`를 소비하고, hex renderer가 alphabet과 pointer prefix를 처리합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - hex digits는 fixed buffer에 reverse order로 생성됩니다. `X`는 uppercase alphabet을 선택하고, pointer는 `void * → uintptr_t → unsigned long` 변환 후 `0x`를 먼저 출력합니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - width/precision와 prefix placement는 후속 numeric layout commits에서 통합되고, alternate `#` prefix는 `c5f627099ad9`에서 nonzero 조건과 함께 추가됩니다.

### 해당 SHA에서 확인할 코드
- dispatch의 `x`, `X`, `p` case와 specifier별 `va_arg` type을 기록합니다.
- base-16 routine이 lowercase/uppercase digit alphabet을 선택하는 지점을 확인합니다.
- pointer가 `uintptr_t`로 conversion된 뒤 numeric formatting으로 넘어가는 경로를 추적합니다.
- `0x` prefix가 address digits와 별도 representation으로 출력되는 코드와 이후 width/padding에 사용할 수 있는 boundary를 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_dispatch.c`: `x`/`X`는 `unsigned int`, `p`는 `void *`를 `va_arg`로 소비합니다.
  - `src/ft_hex.c`: hex digit routine, `ft_printf_print_hex`, `ft_printf_print_pointer`; pointer cast는 `(unsigned long)(uintptr_t)pointer`입니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 93c883070a1b, src/ft_dispatch.c */
if (fmt->spec == 'x' || fmt->spec == 'X')
    return (ft_printf_print_hex(ctx, fmt,
            va_arg(*args, unsigned int)));
if (fmt->spec == 'p')
    return (ft_printf_print_pointer(ctx, fmt,
            va_arg(*args, void *)));
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 같은 typed dispatch boundary를 hex와 pointer로 확장한 commit입니다. pointer는 `uintptr_t`를 거쳐 정수 representation으로 바뀌고, prefix와 digit alphabet은 renderer가 소유합니다.

## 5.6 `c5f627099ad9` — feat(flags): 숫자 플래그 우선순위 정규화

- Importance: `A`
- Tags: `PARSER, FORMAT, LAYOUT`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Normalizes conflicting flags once and applies signed and alternate-form prefixes.
- Commit Classification summary: Normalizes conflicting flags and adds signed and alternate-form prefixes.
- Importance 근거: Resolving '-' over '0' and '+' over space at the parser boundary simplifies every renderer, while nonzero-only hexadecimal prefixes fix public semantics. This is significant cross-conversion judgment.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - parser는 raw flags를 모두 보존하여 `-`와 `0`, `+`와 space가 동시에 설정될 수 있었고, renderer마다 충돌 우선순위를 다시 판단해야 했습니다. positive sign/alternate prefix도 없었습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - parse 완료 직후 한 번 flag bit를 canonicalize하고, numeric renderer는 canonical flags에서 prefix string만 선택합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - `LEFT`가 있으면 `ZERO` bit를 제거하고 `PLUS`가 있으면 `SPACE` bit를 제거합니다. signed positive는 `+`/space/empty 중 하나, negative는 `-`; hex `#`는 nonzero일 때만 case에 맞는 prefix를 선택합니다.
- 학습자 기록 — failure 또는 edge case:
  - repeated flags는 기존 OR semantics를 유지합니다. zero hex에서 alternate prefix가 생기지 않으며, negative signed 값은 plus/space 요청보다 `-`가 우선합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: renderer가 `LEFT|ZERO` 또는 `PLUS|SPACE` 충돌을 받지 않고, signed/hex prefix 선택이 conversion value와 normalized flags에 맞습니다.
  - 미보장: precision과 zero padding의 전체 배치, zero-value suppression, shared layout 중복 제거는 이후 Thread 3에서 해결됩니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `1fa064ca9d79`가 precision/zero layout과 prefix 순서를 구현하고 `177c8d03b353`이 decimal/hex 배치 코드를 공유 helper로 통합합니다.

### 해당 SHA에서 확인할 코드
- parse 이후 flag normalization이 수행되는 정확한 함수/위치를 찾습니다.
- left alignment가 `0` padding을 제거하고 explicit `+`가 space-sign request를 제거하는 bit mutation을 기록합니다.
- positive signed value의 prefix 선택(`+`, space, none)과 negative value의 `-` 유지 branch를 확인합니다.
- hex alternate form이 nonzero value에만 `0x`/`0X`를 추가하는 조건을 기록합니다.
- 이 normalized field/prefix decision이 decimal/hex shared layout 또는 각 renderer에 어떻게 전달되는지 caller/callee path로 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_parse.c`: parse 끝에서 호출되는 `ft_normalize_flags`; bit clear operations.
  - `src/ft_number.c`: signed renderer가 negative/plus/space/none prefix를 선택합니다.
  - `src/ft_hex.c`: hash와 `number != 0`, `X` 여부에 따라 `0X`/`0x`/empty를 선택합니다. 이 SHA에는 아직 shared layout helper가 없고 각 renderer의 기존 output path로 전달됩니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* c5f627099ad9, src/ft_parse.c */
if (fmt->flags & FT_FLAG_LEFT)
    fmt->flags &= ~FT_FLAG_ZERO;
if (fmt->flags & FT_FLAG_PLUS)
    fmt->flags &= ~FT_FLAG_SPACE;
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: parser가 충돌 flag bit를 그대로 넘겼고 signed/hex prefix가 없었습니다.
  - 이후: parser가 canonical flag set을 만들고 renderer가 value-dependent prefix string을 선택합니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 여러 renderer에 흩어질 우선순위 판단을 parser boundary에서 한 번 끝내고, 숫자 prefix semantics를 추가한 cross-conversion commit입니다. 이후 layout은 충돌하지 않는 flag와 이미 결정된 prefix를 입력으로 받을 수 있습니다.

## 6. Invariant ledger

Source가 확정한 변화 축을 아래에 배치했습니다. “실제 코드 근거”는 학습자가 해당 SHA를 읽고 채웁니다.

| Invariant / concern | 도입 또는 초기 상태 | 강화 / 수정 | 고정한 검증 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| field representation | `7984ddf2dd57`에서 `t_format` 도입 | `9e6d785628f3`에서 main traversal과 연결 | 이후 dispatch/rendering이 raw text 대신 normalized field 사용 | `src/ft_printf_internal.h`의 다섯 field; `ft_printf_parse` 반환 cursor; main의 `t_format fmt`와 dispatch 전달 |
| argument extraction | `03c3e6e09fa1`에서 dispatch가 `va_arg` type selection 소유 | `95d6613a1c72`와 `93c883070a1b`에서 decimal/hex/pointer로 확대 | 학습자가 specifier별 정확한 promoted type을 코드에서 기록 | `c:int`, `s:char *`, `d/i:int`, `u/x/X:unsigned int`, `p:void *`, `%`: no argument가 모두 `src/ft_dispatch.c`에 위치 |
| flag conflict states | parser가 flag bit를 idempotent하게 수집 | `c5f627099ad9`에서 conflicting flag normalization 추가 | renderer가 canonical flag set을 받는지 실제 호출 흐름으로 확인 | parser 끝의 LEFT→clear ZERO, PLUS→clear SPACE; main은 같은 `fmt`를 dispatch에 넘기고 numeric renderer가 prefix를 선택 |

### 학습자 추가 기록

- source가 명시한 invariant 범위 안에서만 필요한 행을 추가합니다. 새 invariant를 확정 사실처럼 만들지 않습니다.
- 추가 기록:
  - 추가 행은 만들지 않았습니다. overflow는 field representation 행의 parser 근거와 Failure 표에서 구체화했습니다.

## 7. Failure → Fix → Test 연결

| 기존 failure / risk | Fix / change | 수정 decision | Test / 학습 확인 |
| --- | --- | --- | --- |
| width/precision decimal accumulation overflow | `7984ddf2dd57` | `INT_MAX` 기준 pre-multiplication check로 field parse 실패 처리 | 이 Thread에서는 parser code를 확인하고 whole-call no-output 검증은 Thread 5에서 다시 추적 |
| 각 renderer가 raw field를 독립 해석할 경우 grammar/type 책임이 분산될 위험 | `7984ddf2dd57` + `03c3e6e09fa1` | normalized field + typed dispatch로 책임 분리 | 후속 conversion commits에서 동일 boundary 재사용 여부 확인 |
| conflicting flag를 renderer마다 다시 해석할 위험 | `c5f627099ad9` | `-`가 `0`을, `+`가 space를 제거하도록 한 번 정규화 | numeric layout Thread와 public-boundary tests에서 상호작용을 다시 확인 |

- 학습자 기록 — 실제 failure branch와 regression assertion을 연결한 추가 설명:
  - parser overflow branch는 null을 반환하고 main integration 이후 sticky error가 됩니다. typed dispatch 재사용은 decimal/hex commits의 diff에서 확인했습니다. flag normalization의 조합은 `1b8049e411bb`의 `"norm:'%-05d' '%+ d'"` differential case 및 후속 numeric boundary matrix가 간접적으로 보호합니다.

## 8. Ownership / state / responsibility 변화

| 시점 | Source상 owner / boundary | Source상 responsibility 변화 | 해당 SHA 코드 근거 |
| --- | --- | --- | --- |
| raw format cursor | main traversal + parser | main loop는 전체 sequencing, parser는 한 field 소비와 next unread position을 책임 | `9e6d785628f3` `ft_printf`가 `%` 뒤 pointer를 넘기고 parser 반환값을 다음 `format`으로 사용 |
| normalized field state | `t_format` | flags, width, optional precision, specifier를 renderer가 다시 parse하지 않도록 전달 | `7984ddf2dd57`의 `t_format`과 `ft_printf_init_format`; 이후 dispatch signatures가 `t_format *`를 받음 |
| variadic extraction | dispatch | specifier별 promoted type 선택과 renderer routing을 한 경계에서 수행 | `03c3e6e09fa1` 이후 `src/ft_dispatch.c`의 모든 `va_arg`; entry에는 type-specific extraction 없음 |
| conversion-specific representation | text/decimal/hex renderer | 각 conversion은 자신의 text/digit 생성에 집중하고 shared output path를 사용 | `src/ft_text.c`, `src/ft_number.c`, `src/ft_hex.c`가 extracted value와 `fmt`로 representation을 만들고 `ft_printf_*output` API 호출 |

## 9. Thread 최종 상태

- Source가 확정한 도달점: parsing, argument extraction, rendering이 분리되고, normalized field와 typed dispatch가 공통 boundary가 되며 parser-side flag normalization이 renderer의 conflicting states를 줄인 상태입니다.
- 학습자 기록 — 마지막 commit 기준 실제 코드에서 확인한 최종 state:
  - main loop는 전체 cursor/variadic lifetime/output result를 관리하고, parser는 overflow-checked `t_format`과 next cursor를 만들며, parser 끝에서 conflict flags를 제거합니다. dispatch는 specifier별 promoted type을 정확히 소비하고 text/decimal/hex renderer에 전달합니다.
- 학습자 기록 — 이 Thread 밖에서만 해결되는 남은 문제를 source 범위 안에서 구분:
  - unsupported/trailing field를 전체 호출 전에 거부하는 정책과 measurement/rendering의 argument 동기화는 Thread 5에서 완성됩니다. 숫자 prefix/precision/width의 공유 배치와 bounded string scan은 각각 Thread 3·4의 책임입니다.

## 10. 최종 architecture 또는 execution flow 정리

실제 SHA 코드를 읽은 뒤 아래 흐름을 완성합니다. source 설명만 복사하지 말고 함수/상태/branch를 연결합니다.

```text
[ft_printf의 전체 format cursor]
    -> [ft_printf_parse: flags/width/precision/spec + normalization]
    -> [ft_printf_dispatch: specifier별 va_arg promoted type 소비]
    -> [text/decimal/hex renderer: representation 생성, shared output 호출]
    -> [parse/output failure는 sticky error와 public -1, 성공은 다음 cursor]
```

- 각 단계에 대응하는 SHA / file / function:
  - `7984ddf2dd57` `src/ft_parse.c::ft_printf_parse`; `9e6d785628f3` `src/ft_printf.c::ft_printf`; `03c3e6e09fa1` `src/ft_dispatch.c::ft_printf_dispatch`; `95d6613a1c72`/`93c883070a1b` numeric renderers; `c5f627099ad9` normalization/prefix paths입니다.
- 핵심 state transition:
  - raw pointer → initialized `t_format` → OR된 flags와 parsed integer fields → canonical flags/spec → 해당 conversion type만큼 `va_list` 진행 → output context mutation입니다.
- failure가 끊기는 지점:
  - decimal overflow로 parser가 null이면 main이 `ctx.error`를 설정합니다. renderer/output 실패는 dispatch return `< 0`으로 main loop를 끊습니다. 이 Thread 시점의 unknown fallback은 실패로 끊기지 않습니다.
- 후속 fix/test가 보장한 지점:
  - later preflight가 supported syntax와 total length를 사전 확인하고, numeric/text differential/public-boundary suites가 normalized field와 type dispatch 결과를 `snprintf` 또는 explicit bytes와 비교합니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 정확한 시점의 코드로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 source와 그대로 유지했습니다.
- [x] final HEAD의 코드를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] 필요한 parent/직전 관련 SHA 비교를 실제 diff로 수행했습니다.
- [x] source가 확정한 사실과 내가 코드에서 확인한 사실을 구분했습니다.
- [x] fix의 기존 가정 → failure/risk → root cause → decision → code → test 연결을 필요한 곳에서 완성했습니다.
- [x] test commit의 target invariant, technique, production path, proves/not-proves를 구분했습니다.
- [x] Invariant ledger에 실제 코드 근거를 채웠습니다.
- [x] 이 Thread의 최종 architecture/execution flow를 commit history 순서로 설명할 수 있습니다.
===== END FILE: 02-format-fields-typed-dispatch.md =====

===== BEGIN FILE: 03-shared-numeric-layout.md =====
# Numeric formatting converges on one layout model

## 1. Thread 목표

decimal, hexadecimal, pointer에서 중복되던 prefix/precision/zero/width/alignment 배치 규칙이 하나의 numeric layout responsibility로 수렴하는 과정을 복원합니다.

### Source에서 확정된 significance

초기 decimal/hex 구현은 spaces, prefixes, field zeros, precision zeros, digits, trailing padding 순서를 중복 구현합니다. shared layout은 이 순서를 하나의 invariant로 만들고, 이후 portability 및 boundary 작업이 signed endpoint와 프로젝트 고유 pointer semantics에서도 이 공통 모델을 유지합니다.

### 이 Thread에 명시적으로 연결되는 source invariant / engineering difficulty

- Invariant: numeric output은 decimal, hexadecimal, pointer 전반에서 spaces, prefixes, field zeros, precision zeros, digits, trailing spaces를 일관된 순서로 출력합니다.
- Invariant: measurement와 rendering은 prefix, zero suppression, precision zeros, field width를 포함한 effective length에 동의해야 합니다.
- Engineering difficulty: prefix selection, zero precision, alternate form, zero padding, width, left alignment를 올바른 순서와 precedence로 합성하는 문제입니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- spaces, prefix, field zeroes, precision zeroes, digits, trailing spaces의 정확한 순서는 각 단계에서 어떻게 표현되는가?
- precision이 명시되면 field-level `0` flag가 왜/어디서 무효화되는가?
- zero value + precision zero의 digit suppression과 alternate-form prefix는 어떻게 결합되는가?
- decimal/hex duplication은 어떤 공통 입력으로 추상화되어 `ft_printf_write_numeric_layout`에 모이는가?
- `INT_MIN` magnitude 계산은 왜 signed type에서 직접 negation하면 안 되는가?
- libc differential oracle와 project-specific fixed expectation은 어느 경계에서 나뉘는가?

## 3. 완료 기준

- 각 layout component의 길이 계산과 emission 순서를 실제 code branch로 설명할 수 있습니다.
- `177c8d03b353` 전후로 decimal/hex 중복 코드가 어떤 shared responsibility로 이동했는지 비교할 수 있습니다.
- `ed3750fd081a`에서 `INT_MIN`을 signed overflow 없이 magnitude로 바꾸는 식을 실제 코드로 설명할 수 있습니다.
- `12d715eba77d`에서 zero precision, prefix, null pointer, narrow width가 어떤 public contract로 고정되는지 test case와 production path를 연결할 수 있습니다.

## 4. Commit map

| SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- |
| `ac27a26affaa` | feat(decimal): 10진수 너비와 정렬 적용 | `B` | `FORMAT, LAYOUT` | Adds prefix-aware width and alignment to decimal output. |
| `c5ef742b84de` | feat(hex): 16진수와 포인터 너비와 정렬 적용 | `B` | `FORMAT, LAYOUT` | Repeats the model for hexadecimal and pointer output. |
| `1fa064ca9d79` | feat(numeric): 숫자 정밀도와 0 채움 적용 | `A` | `FORMAT, LAYOUT, RISK` | Adds zero suppression, precision zeros, and zero-field padding rules. |
| `c5f627099ad9` | feat(flags): 숫자 플래그 우선순위 정규화 | `A` | `PARSER, FORMAT, LAYOUT` | Establishes prefix selection and flag precedence. |
| `f276ee73087c` | test(numeric): 접두사와 정밀도 배치 회귀 검증 | `B` | `FORMAT, TEST, EDGE` | Locks down representative prefix, precision, zero, and left-alignment interactions. |
| `177c8d03b353` | refactor(output): 숫자 출력 배치 로직 통합 | `A` | `ARCH, LAYOUT, REFACTOR` | Extracts one shared numeric layout writer for decimal, hexadecimal, and pointer conversions. |
| `ed3750fd081a` | fix(decimal): INT_MIN 크기를 unsigned 범위에서 계산 | `A` | `FORMAT, EDGE, RISK` | Removes signed-overflow dependence when formatting `INT_MIN`. |
| `12d715eba77d` | test(printf): 공개 계약 경계 사례 확대 | `A` | `FORMAT, TEST, EDGE` | Expands public boundary matrices for zero precision, prefixes, null pointers, and narrow fields. |

## 5. Commit별 학습 기록

> 원칙: 아래 기록은 final HEAD가 아니라 각 항목의 정확한 SHA에서 작성합니다. source가 확정하지 않은 파일명/함수명은 현재 골격에서 추측하지 않습니다.

## 5.1 `ac27a26affaa` — feat(decimal): 10진수 너비와 정렬 적용

- Importance: `B`
- Tags: `FORMAT, LAYOUT`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Adds prefix-aware width and alignment to decimal output.
- Commit Classification summary: Builds decimal digit buffers and applies prefix-aware width and alignment.
- Importance 근거: The commit is necessary layout implementation but still local to decimal rendering and later consolidated.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - 직전 `src/ft_number.c`는 숫자를 역순 임시 배열에 만든 뒤 한 글자씩 바로 출력했고, 음수 부호도 digit 출력 전에 별도 `putchar`로 처리했습니다. 따라서 width가 부호를 포함한 전체 표현에 적용될 수 없었습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - decimal digit를 출력 순서의 buffer로 먼저 완성하고, 부호를 `prefix` 문자열로 취급해 `width - prefix_len - digit_len`만큼의 padding을 계산합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - `ft_decimal_digits`가 `reversed[20]`에서 `digits[20]`으로 순서를 뒤집어 반환합니다. `ft_write_decimal`은 right-aligned이면 spaces→prefix→digits, left-aligned이면 prefix→digits→spaces 순서로 shared output API를 호출합니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - `1fa064ca9d79`가 digit suppression, precision zero, field zero를 같은 함수에 추가합니다. `177c8d03b353`은 이 배치 책임을 decimal module에서 공통 numeric layout으로 옮깁니다.

### 해당 SHA에서 확인할 코드
- decimal conversion이 먼저 magnitude digits를 materialize하고 length를 결정한 뒤 layout을 수행하도록 바뀐 diff를 찾습니다.
- minus sign이 별도 preliminary write가 아니라 explicit prefix로 표현되는 지점을 확인합니다.
- width 계산에서 prefix length + digit length가 어떻게 반영되는지 기록합니다.
- right alignment와 left alignment에서 spaces가 complete signed representation 앞/뒤 어디에 배치되는지 확인합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_number.c`: `ft_decimal_digits`, `ft_write_decimal`, `ft_printf_print_signed`, `ft_printf_print_unsigned`.
  - signed renderer는 negative value에 `"-"`, nonnegative value에 `""`를 전달합니다. 모든 실제 byte 출력은 `ft_printf_putnchar` 또는 `ft_printf_write`를 통과합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* ac27a26affaa, src/ft_number.c, ft_write_decimal */
padding = fmt->width - prefix_len - digit_len;
if (!(fmt->flags & FT_FLAG_LEFT)
    && ft_printf_putnchar(ctx, ' ', padding) < 0)
    return (-1);
if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
    return (-1);
if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
    return (-1);
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - decimal value를 먼저 완전한 digit representation으로 만든 뒤 prefix를 포함해 field width를 계산한 commit입니다. 배치 규칙은 아직 decimal 내부 구현이며 precision과 zero padding은 다루지 않습니다.

## 5.2 `c5ef742b84de` — feat(hex): 16진수와 포인터 너비와 정렬 적용

- Importance: `B`
- Tags: `FORMAT, LAYOUT`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Repeats the model for hexadecimal and pointer output.
- Commit Classification summary: Applies width and alignment to hexadecimal and pointer output.
- Importance 근거: This repeats the established layout model for another conversion family and is normal supporting work.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - hex는 digit를 역순 buffer에서 한 글자씩 출력했고 pointer는 먼저 `0x`를 써 버렸기 때문에 width가 prefix와 digits 전체에 적용되지 않았습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - `x`, `X`, `p` 모두 digit를 materialize하고 explicit prefix와 함께 하나의 field로 배치합니다. 대소문자는 base alphabet이, pointer 여부는 caller가 넘기는 `"0x"`가 결정합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - `ft_hex_digits`가 lowercase/uppercase alphabet으로 출력 순서의 digits를 만들고, `ft_write_hex`가 `prefix_len + digit_len`을 기준으로 leading/trailing spaces를 배치합니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - `1fa064ca9d79`에서 decimal과 같은 precision/zero 규칙이 복제됩니다. `c5f627099ad9`에서 `#` prefix가 값에 따라 선택되고, `177c8d03b353`에서 중복 배치가 제거됩니다.

### 해당 SHA에서 확인할 코드
- hex/pointer가 digit materialization 이후 prefix + digit length로 width를 계산하는 path를 추적합니다.
- `x`, `X`, `p`가 동일 placement logic을 공유하면서 digit alphabet과 pointer prefix responsibility를 분리하는 코드를 기록합니다.
- 직전 decimal layout과 비교하여 중복된 placement sequence를 구체적으로 표시합니다. 이 비교는 이후 `177c8d03b353`의 refactor 근거가 됩니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_hex.c`: `ft_hex_digits`, `ft_write_hex`, `ft_printf_print_hex`, `ft_printf_print_pointer`.
  - decimal의 `ft_write_decimal`과 마찬가지로 prefix length를 순회해 구하고 `padding = width - prefix_len - digit_len`을 계산한 뒤 spaces/prefix/digits/spaces를 직접 호출합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* c5ef742b84de, src/ft_hex.c, ft_write_hex */
if (fmt->spec == 'X')
    digit_len = ft_hex_digits(digits, number, "0123456789ABCDEF");
else
    digit_len = ft_hex_digits(digits, number, "0123456789abcdef");
prefix_len = 0;
while (prefix[prefix_len])
    prefix_len++;
padding = fmt->width - prefix_len - digit_len;
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - decimal에서 도입한 prefix-aware width/alignment 방식을 hex와 pointer에 반복 적용한 commit입니다. 기능은 완성되지만 동일한 placement sequence가 두 module에 중복된 상태가 됩니다.

## 5.3 `1fa064ca9d79` — feat(numeric): 숫자 정밀도와 0 채움 적용

- Importance: `A`
- Tags: `FORMAT, LAYOUT, RISK`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Adds zero suppression, precision zeros, and zero-field padding rules.
- Commit Classification summary: Adds zero suppression, precision zeros, and zero-padding order for decimal and hexadecimal output.
- Importance 근거: The interaction among prefix, precision, field width, left alignment, and the zero flag is one of the formatter's hardest local correctness problems. It is significant, though the duplicated implementation is later centralized.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - width/alignment만으로는 `%.0d`, `%08d`, `%08.5d`를 구분할 수 없습니다. 특히 prefix를 field zero보다 먼저 출력해야 하고, 명시적 precision이 있으면 `0` flag를 field padding에 사용하면 안 됩니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - decimal과 hex writer가 각각 `digit_len`, `prefix_len`, `zero_len`, `padding`, `pad_char`를 계산하고 같은 component 순서를 직접 구현합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - zero value이면서 explicit precision이 0이면 effective `digit_len = 0`; precision이 digit보다 크면 `zero_len = precision - digit_len`; field width는 prefix, precision zeros, digits를 뺀 값입니다. field `0`은 LEFT가 없고 precision도 없을 때만 선택됩니다.
- 학습자 기록 — failure 또는 edge case:
  - prefix 뒤에 field zero가 와야 하는 signed/alternate form, zero digit suppression, precision이 `0` flag를 무효화하는 경우가 핵심입니다. 각 output call 실패는 즉시 `-1`로 반환됩니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 이 SHA의 decimal/hex writer는 leading spaces→prefix→field zeros→precision zeros→digits→trailing spaces 순서를 유지합니다.
  - 미보장: 동일 규칙이 두 파일에 복제되어 있어 후속 수정 시 drift할 가능성이 남습니다. flag conflict와 `+`/space/`#` prefix 선택은 다음 commit 범위입니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `c5f627099ad9`가 canonical flags와 value-dependent prefix를 공급하고, `f276ee73087c`가 조합을 고정합니다. `177c8d03b353`은 중복 계산을 하나의 helper로 이동합니다.

### 해당 SHA에서 확인할 코드
- numeric layout에서 prefix bytes, precision zeroes, field padding, value digits를 각각 어떤 변수로 계산하는지 기록합니다.
- `has_precision`이 true일 때 field-level `0` flag가 억제되는 조건을 확인합니다.
- zero value + precision zero에서 effective digit count가 0이 되는 branch를 추적합니다.
- leading spaces → prefix → field zeroes → precision zeroes → digits → trailing spaces의 실제 emission call 순서를 기록합니다.
- decimal/hex가 같은 rule을 각자 구현하는 중복 지점을 표시하여 이후 shared layout과 비교합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_number.c`의 `ft_write_decimal`과 `src/ft_hex.c`의 `ft_write_hex`에 사실상 같은 branch와 emission calls가 추가됩니다.
  - `ft_printf_putnchar`는 spaces/zeros를, `ft_printf_write`는 prefix/digits를 출력하므로 기존 shared count/error state는 우회하지 않습니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 1fa064ca9d79, src/ft_number.c, ft_write_decimal */
if (fmt->has_precision && fmt->precision == 0 && number == 0)
    digit_len = 0;
prefix_len = 0;
while (prefix[prefix_len])
    prefix_len++;
zero_len = 0;
if (fmt->has_precision && fmt->precision > digit_len)
    zero_len = fmt->precision - digit_len;
padding = fmt->width - prefix_len - zero_len - digit_len;
pad_char = ' ';
if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
    && !fmt->has_precision)
    pad_char = '0';
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: width는 prefix와 raw digit length만 빼고 spaces로 채웠습니다.
  - 이후: effective digits, precision zeros, field padding을 별도 길이로 계산하며 field zero와 precision zero를 서로 다른 emission 단계로 둡니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 숫자 field를 여섯 component로 분해해 precision과 zero padding의 precedence를 구현한 A-level commit입니다. 올바른 visible order를 만들지만 decimal과 hex에 같은 correctness logic이 중복됩니다.

## 5.4 `c5f627099ad9` — feat(flags): 숫자 플래그 우선순위 정규화

- Importance: `A`
- Tags: `PARSER, FORMAT, LAYOUT`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Establishes prefix selection and flag precedence.
- Commit Classification summary: Normalizes conflicting flags and adds signed and alternate-form prefixes.
- Importance 근거: Resolving '-' over '0' and '+' over space at the parser boundary simplifies every renderer, while nonzero-only hexadecimal prefixes fix public semantics. This is significant cross-conversion judgment.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - parser가 LEFT+ZERO, PLUS+SPACE를 모두 보존해 renderer마다 precedence를 다시 판단해야 했고, signed positive prefix와 hex alternate prefix도 아직 선택되지 않았습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - parser가 한 field를 다 읽은 뒤 LEFT이면 ZERO를, PLUS이면 SPACE를 지웁니다. renderer는 canonical flags만 받고 값과 specifier에 따라 하나의 prefix 문자열을 고릅니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - signed는 negative `-`, otherwise PLUS `+`, SPACE ` `, none 순서입니다. hex는 HASH이면서 number가 0이 아닐 때만 specifier case에 맞는 `0x`/`0X`를 선택합니다.
- 학습자 기록 — failure 또는 edge case:
  - `%#x`의 zero에는 prefix가 없어야 하며, `%-05d`는 left-aligned space padding이어야 하고 `%+ d`는 plus만 남아야 합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: renderer는 conflicting flag 조합을 받지 않고 prefix가 width/precision 계산 전에 명시됩니다.
  - 미보장: placement 계산은 여전히 decimal/hex 내부에 중복되어 있습니다. project-specific pointer precision 계약과 broad matrix는 후속 test에서 고정됩니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `f276ee73087c`가 prefix와 zero/precision 조합을 differential test로 고정하고, `177c8d03b353`가 선택된 prefix를 공통 layout input으로 받습니다.

### 해당 SHA에서 확인할 코드
- parse 이후 flag normalization이 수행되는 정확한 함수/위치를 찾습니다.
- left alignment가 `0` padding을 제거하고 explicit `+`가 space-sign request를 제거하는 bit mutation을 기록합니다.
- positive signed value의 prefix 선택(`+`, space, none)과 negative value의 `-` 유지 branch를 확인합니다.
- hex alternate form이 nonzero value에만 `0x`/`0X`를 추가하는 조건을 기록합니다.
- 이 normalized field/prefix decision이 decimal/hex shared layout 또는 각 renderer에 어떻게 전달되는지 caller/callee path로 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_parse.c`: `ft_normalize_flags`가 parse 종료 전에 bit를 제거합니다.
  - `src/ft_number.c`: `ft_printf_print_signed`가 value/sign flags로 prefix를 선택해 local decimal writer에 전달합니다.
  - `src/ft_hex.c`: `ft_printf_print_hex`가 HASH, nonzero value, `X` 여부로 prefix를 선택합니다. 이 SHA의 배치 대상은 아직 각 module의 local writer입니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* c5f627099ad9, src/ft_parse.c */
if (fmt->flags & FT_FLAG_LEFT)
    fmt->flags &= ~FT_FLAG_ZERO;
if (fmt->flags & FT_FLAG_PLUS)
    fmt->flags &= ~FT_FLAG_SPACE;
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: conflicting bits가 함께 남았고 numeric prefix는 negative sign/pointer를 제외하면 비어 있었습니다.
  - 이후: parser가 canonical state를 만들고 renderer가 signed/alternate prefix를 explicit string으로 배치 함수에 넘깁니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - flag conflict를 parser에서 한 번 해소하고 prefix 선택을 value-aware renderer 책임으로 둔 commit입니다. 공통 layout 추출 전에 입력 상태를 정규화한 단계입니다.

## 5.5 `f276ee73087c` — test(numeric): 접두사와 정밀도 배치 회귀 검증

- Importance: `B`
- Tags: `FORMAT, TEST, EDGE`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Locks down representative prefix, precision, zero, and left-alignment interactions.
- Commit Classification summary: Adds focused prefix, precision, zero, and alignment regression cases.
- Importance 근거: The cases protect tricky established layout semantics but do not introduce the shared layout mechanism.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - layout 구현은 존재했지만 sign/alternate prefix와 두 종류의 zero padding이 결합되는 대표 사례를 별도로 묶어 보호하지 않았습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - focused numeric cases를 `run_numeric_layout_cases`로 묶어 `snprintf` 결과와 return/byte sequence를 비교합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - `%+08d`, `%#08x`, `%-#10.4x`, `% 08.5d`, `%#.0x` 등의 포맷이 public `ft_printf`에서 당시 중복 decimal/hex writer까지 통과합니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - `177c8d03b353` 이후 동일 test가 shared helper refactor의 external behavior regression gate가 됩니다. `12d715eba77d`는 값을 배열로 순회하는 더 넓은 boundary matrix를 추가합니다.

### 해당 SHA에서 확인할 코드
- 각 regression case가 sign, alternate prefix, field zero, precision zero, left alignment 중 어떤 조합을 만드는지 표로 기록합니다.
- expected value가 `snprintf` differential인지 explicit expected output인지 test code에서 확인합니다.
- sign/`0x`가 zero padding 앞에 오는지, precision이 field `0`을 무효화하는지, zero hex + precision zero에서 alternate prefix가 사라지는지, trailing spaces가 complete prefixed value 밖에 오는지 assertion으로 연결합니다.
- 이 test가 shared layout 도입 전의 duplicated implementation을 검증하는지, 이후 같은 test가 regression gate로 남는지 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `tests/test_ft_printf.c`: `run_numeric_layout_cases`, `EXPECT_PRINTF` macro, stdout capture/check helper.
  - 모든 추가 case는 explicit output이 아니라 `snprintf` differential입니다.
  - 조합 대응: `%+08d`=sign before field zeros, `%#08x`=`0x` before field zeros, `%-#10.4x`=prefix+precision zeros then trailing spaces, `% 08.5d`=precision disables field zero, `%#.0x` with zero=digits/prefix both absent.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* f276ee73087c, tests/test_ft_printf.c, run_numeric_layout_cases */
EXPECT_PRINTF("empty:'%#.0x' '%#.0X' '% .0d'", 0u, 0u, 0);
EXPECT_PRINTF("signed-zero:'%+08d'", 42);
EXPECT_PRINTF("hex-zero:'%#08x'", 42u);
EXPECT_PRINTF("hex-left-precision:'%-#10.4x'", 42u);
```

### Test commit 학습 기록
- production invariant 대상: numeric prefix와 field/precision zero, left alignment의 ordering
- 재현하는 failure / boundary: sign/`0x` 앞뒤 zero placement 오류, precision과 `0` 충돌, zero-precision alternate-form 오류, trailing-space 위치 오류
- test technique: `snprintf`와 focused differential regression cases
- 통과하는 production path: numeric conversion → 당시 decimal/hex placement implementation → shared output context
- 이 test가 source상 증명하려는 것: 대표적인 다중-flag 조합의 visible ordering이 intended semantics와 일치함
- 이 test가 증명하지 않는 것: 모든 값/format 조합이나 이후 shared-layout refactor의 내부 구조 자체를 증명하지 않습니다.
- 분류: focused deterministic regression입니다.
- 후속 회귀 방지 역할: layout 중앙화/수정 시 prefix–padding–precision 조합의 회귀를 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - `EXPECT_PRINTF`는 `snprintf`로 expected bytes/return을 만들고 pipe로 `ft_printf` stdout을 캡처해 return, captured length, `memcmp`를 모두 확인합니다. `run_numeric_layout_cases`는 여섯 focused format을 이 fixture로 실행합니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: 실행 가능한 exact checkout을 만들 수 없는 환경이어서 GitHub connector로 해당 SHA의 test와 production diff만 검사했습니다.
  - result: 실행 결과를 주장하지 않습니다. test mechanism과 대상 production branch만 코드로 확인했습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - shared layout 도입 전 중복 구현이 보여 주어야 할 핵심 ordering을 focused differential cases로 고정한 commit입니다. 이후 구조가 바뀌어도 동일 public bytes를 요구합니다.

## 5.6 `177c8d03b353` — refactor(output): 숫자 출력 배치 로직 통합

- Importance: `A`
- Tags: `ARCH, LAYOUT, REFACTOR`
- Most Important Commits 목록: 포함
- Thread 내 역할: Extracts one shared numeric layout writer for decimal, hexadecimal, and pointer conversions.
- Commit Classification summary: Extracts one numeric layout writer shared by decimal, hexadecimal, and pointer conversions.
- Importance 근거: Centralizing prefix, precision-zero, field-padding, and digit ordering removes duplicated correctness logic and creates one responsibility boundary later mirrored by measurement. This is a significant structural improvement, though not independently project-defining.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - `ft_write_decimal`과 `ft_write_hex`가 suppression, `zero_len`, `padding`, `pad_char`, 여섯 단계 emission과 실패 반환을 각각 보유했습니다. 어느 한쪽만 수정되면 visible semantics가 달라질 수 있었습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - 새 `src/ft_numeric_layout.c`의 `ft_printf_write_numeric_layout`이 normalized `fmt`, prefix, prepared digits, digit length, `is_zero`를 입력으로 받아 모든 placement를 소유합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - decimal/hex/pointer가 같은 함수에서 zero suppression, precision zero, field padding과 output ordering을 수행합니다. 각 conversion은 value를 digits로 바꾸고 prefix/zero 사실만 제공합니다.
- 학습자 기록 — failure 또는 edge case:
  - helper의 각 `putnchar`/`write`가 실패하면 즉시 `-1`이므로 중간 실패 이후 후속 component가 출력되지 않습니다. 이미 accepted된 bytes는 rollback하지 않습니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 세 numeric family의 layout 판단과 failure propagation이 한 구현을 공유합니다.
  - 미보장: digit conversion, signed magnitude portability, prefix selection은 각 renderer에 남고 measurement와 rendering 일치는 아직 별도 선검증 도입 전입니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `ed3750fd081a`가 layout 앞의 signed magnitude 변환을 수정합니다. 후속 `2d773acc5bd6` measurement는 동일 component length 계산을 mirror해야 하며, `12d715eba77d` matrix가 public semantics를 확대 검증합니다.

### 해당 SHA에서 확인할 코드
- parent SHA에서 decimal과 hex가 각각 갖고 있던 placement sequence를 나란히 비교합니다.
- `ft_printf_write_numeric_layout`의 실제 signature에서 prefix, prepared digits, digit length, zero-value fact, normalized field 중 어떤 정보가 전달되는지 기록합니다.
- shared helper가 effective digit length, precision zeroes, field padding, emission order, error propagation을 소유하는 코드 지점을 추적합니다.
- decimal/hex/pointer conversion 쪽에서 제거된 책임과 여전히 남은 responsibility를 diff로 구분합니다.
- 후속 measurement pass가 이 one layout model을 mirror해야 하는 이유를 실제 component 계산을 기준으로 메모합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `Makefile`: `src/ft_numeric_layout.c` 추가.
  - `src/ft_numeric_layout.c`: `ft_printf_write_numeric_layout`이 prefix length, suppression, precision zero, padding, emission을 담당합니다.
  - `src/ft_number.c`: decimal digits와 signed/unsigned prefix/magnitude 선택 후 helper 호출.
  - `src/ft_hex.c`: hex digits/alphabet, alternate/pointer prefix 선택 후 helper 호출.
  - `src/ft_printf_internal.h`: shared helper prototype.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 177c8d03b353, src/ft_numeric_layout.c,
 * ft_printf_write_numeric_layout */
if (fmt->has_precision && fmt->precision == 0 && is_zero)
    digit_len = 0;
prefix_len = ft_prefix_length(prefix);
zero_len = 0;
if (fmt->has_precision && fmt->precision > digit_len)
    zero_len = fmt->precision - digit_len;
padding = fmt->width - prefix_len - zero_len - digit_len;
pad_char = ' ';
if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
    && !fmt->has_precision)
    pad_char = '0';
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: decimal/hex local writer가 각각 suppression부터 trailing spaces까지 전부 수행했습니다.
  - 이후: local writer 자체가 제거되고 conversion은 prepared representation을 공통 helper에 넘깁니다. 숫자 base 변환과 prefix 선택은 원래 module에 남습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 중복된 visible-layout correctness를 하나의 함수로 중앙화한 refactor입니다. 입력은 conversion-specific representation이고, 출력 component 계산·순서·실패 전파는 shared layout의 단일 책임이 됩니다.

## 5.7 `ed3750fd081a` — fix(decimal): INT_MIN 크기를 unsigned 범위에서 계산

- Importance: `A`
- Tags: `FORMAT, EDGE, RISK`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Removes signed-overflow dependence when formatting `INT_MIN`.
- Commit Classification summary: Forms negative magnitude as -(value + 1) + 1 before unsigned conversion.
- Importance 근거: The correction removes signed-overflow dependence on platforms where long cannot represent -INT_MIN. It restores portable numeric correctness with a small but material fix.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - negative `int`를 `long`으로 옮긴 뒤 `-value`를 먼저 signed domain에서 계산했습니다. `long`과 `int`의 폭이 같은 data model에서는 `INT_MIN`의 양수 counterpart가 `long`에 표현되지 않습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - sign/prefix는 그대로 결정하되 magnitude만 `-(value + 1)`로 representable한 signed 값까지 계산한 후 `unsigned long`으로 바꾸고 마지막 1을 unsigned domain에서 더합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - 모든 `int` negative endpoint의 magnitude가 signed overflow 없이 `unsigned long`에 형성됩니다. shared layout이 받는 digits/prefix interface는 바뀌지 않습니다.
- 학습자 기록 — failure 또는 edge case:
  - 위험 대상은 정확히 minimum signed value입니다. 일반 negative 값에서는 이전과 새 식이 같은 magnitude를 만듭니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: `int`와 `long`이 같은 폭인 구현에서도 `INT_MIN` magnitude 계산 자체가 unrepresentable signed positive value를 만들지 않습니다.
  - 미보장: 이 fix는 출력 retry, layout ordering, parser 또는 pointer conversion을 바꾸지 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - 기존 core suite의 `INT_MIN` case와 후속 sanitizer targets가 이 path를 통과할 수 있고, `12d715eba77d`의 signed matrix가 추가 조합을 보호합니다. 다만 이 commit에 전용 신규 test는 포함되지 않았습니다.

### 해당 SHA에서 확인할 코드
- fix 직전 signed negative magnitude 계산에서 `-value` 또는 equivalent positive counterpart가 어떤 signed type에서 형성되는지 확인합니다.
- `INT_MIN`의 positive counterpart가 같은 signed type에 representable하지 않을 수 있는 이유를 해당 platform-independent integer range로 설명합니다.
- fix SHA에서 `value + 1`을 먼저 negation하고 `unsigned long`으로 변환한 뒤 마지막 1을 unsigned domain에서 더하는 정확한 expression을 기록합니다.
- sign selection과 shared decimal layout은 바뀌지 않았는지 diff로 확인하여 fix scope를 분리합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_number.c`, `ft_printf_print_signed`: negative branch가 `ft_write_decimal`에 넘기는 magnitude expression만 의미 있게 바뀝니다. prefix `"-"`, digit creation, `ft_printf_write_numeric_layout` 호출은 유지됩니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* ed3750fd081a, src/ft_number.c, ft_printf_print_signed */
if (value < 0)
    return (ft_write_decimal(ctx, fmt, "-",
            (unsigned long)(-(value + 1)) + 1));
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `(unsigned long)(-value)`처럼 positive magnitude를 signed `long`에서 먼저 만들었습니다.
  - 이후: representable한 `-(value + 1)`까지만 signed로 계산하고 최종 `+ 1`은 unsigned arithmetic으로 수행합니다.

### Failure → Fix 추적
- 기존 가정/상태: negative `int`를 더 넓은 `long`으로 옮겨 negation하면 `INT_MIN` magnitude를 항상 안전하게 만들 수 있다는 assumption
- 실제 failure 또는 위험: `long`이 `int`보다 넓지 않은 data model에서는 positive counterpart가 signed type에 representable하지 않을 수 있음
- source가 지목한 root cause: signed domain에서 unrepresentable positive magnitude를 직접 형성하려는 계산
- 수정된 decision/invariant: `-(value + 1)`의 representable result를 unsigned로 변환하고 마지막 1을 unsigned domain에서 더함
- 학습자 기록 — 실제 수정 코드:
  - 실제 expression은 `(unsigned long)(-(value + 1)) + 1`입니다. cast 뒤의 `+ 1`은 usual arithmetic conversion에 따라 unsigned domain에서 수행됩니다.
- 학습자 기록 — regression test 연결:
  - 이 fix commit 자체에는 전용 test 추가가 없습니다. 이전 `1b8049e411bb`의 core `INT_MIN` differential case가 해당 path를 통과하고, 후속 `12d715eba77d` signed matrix와 `1b474fa2a5e3` sanitizer suite가 broader coverage를 제공합니다. 동일 폭 `int`/`long` 플랫폼 전용 실행 증거는 repository test code에서 확인되지 않았습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - layout이 아니라 value-to-magnitude boundary를 수정한 portability fix입니다. minimum signed value의 양수 counterpart를 signed domain에 만들지 않고 unsigned representation으로 넘긴 뒤 기존 shared layout을 그대로 사용합니다.

## 5.8 `12d715eba77d` — test(printf): 공개 계약 경계 사례 확대

- Importance: `A`
- Tags: `FORMAT, TEST, EDGE`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Expands public boundary matrices for zero precision, prefixes, null pointers, and narrow fields.
- Commit Classification summary: Locks down zero precision, prefixes, null values, percent extensions, and width/precision boundary matrices.
- Importance 근거: Several expectations are deliberate project contracts rather than portable libc behavior. Fixing them explicitly is significant for preserving the library's actual public semantics.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - focused cases만으로는 width가 content와 같거나 작을 때, zero precision에서 value가 0/비0일 때, sign/hash가 있는 여러 value를 체계적으로 교차하지 못했습니다. null pointer/string과 formatted percent는 libc oracle에 맡길 수 없는 프로젝트 계약입니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - signed, unsigned, hex에 format 배열×value 배열 differential matrix를 추가하고, null format/empty/null pointer/null string/percent는 explicit expected output으로 분리합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production code는 바뀌지 않습니다. tests가 shared layout의 suppression/prefix/precision/width semantics와 project-specific pointer representation을 public return/bytes 계약으로 고정합니다.
- 학습자 기록 — failure 또는 edge case:
  - `%.0p` null이 `"0x"`, `%8.4p` null이 `"  0x0000"`; `%#.0x` zero는 empty digits/prefix; narrow width는 content를 truncate하지 않고 padding만 0 이하가 됩니다. null string precision은 `(null)`을 같은 string 규칙으로 제한합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 명시된 matrices와 fixed cases에서 output bytes와 return count가 oracle/프로젝트 expectation과 같습니다.
  - 미보장: 모든 가능한 width/precision/value 조합, system-call fault behavior, archive ABI/dependency를 증명하지 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - 이 Thread의 마지막 test입니다. 이후 release/sanitizer verification은 같은 functional suite를 산출물·runtime 계층에서 다시 실행하지만 numeric contract 자체를 재정의하지 않습니다.

### 해당 SHA에서 확인할 코드
- precision zero, digit-count transition, signs, alternate prefix, content-width boundary에 대한 differential matrix를 찾아 입력 축과 expected oracle을 기록합니다.
- null format, empty format, null string, null pointer, formatted percent처럼 fixed project expectation으로 분리된 case를 식별합니다.
- 이 Thread 관점에서는 특히 null pointer의 `0x` prefix와 precision-zero digit suppression, width가 content보다 작은/큰 경우의 production layout path를 추적합니다.
- libc와 비교하지 않는 fixed expectation이 “왜 project contract인지” source description과 test implementation을 대응시킵니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `tests/test_ft_printf.c`: `run_signed_boundary_matrix`, `run_unsigned_boundary_matrix`, `run_hex_boundary_matrix`, `run_public_contract_boundary_cases`.
  - signed matrix: 12 formats×5 values; unsigned: 10×5; hex: 11×5를 `EXPECT_PRINTF`/`snprintf`와 비교합니다.
  - fixed `EXPECT_OUTPUT`은 null pointer/string과 `%` extension의 repository-defined bytes를 검사합니다. `EXPECT_FORMAT_ERROR(NULL)`은 public null-format failure를 검사합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 12d715eba77d, tests/test_ft_printf.c,
 * run_public_contract_boundary_cases */
EXPECT_OUTPUT("0x", "%.0p", (void *)0);
EXPECT_OUTPUT("      0x", "%8.0p", (void *)0);
EXPECT_OUTPUT("0x      ", "%-8.0p", (void *)0);
EXPECT_OUTPUT("  0x0000", "%8.4p", (void *)0);
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: 개별 focused/differential cases가 있었지만 systematic signed/unsigned/hex boundary cross-product와 fixed public-contract group은 없었습니다.
  - 이후: matrices가 shared numeric path를 넓게 반복하고 libc 비호환/비이식 expectation은 explicit bytes로 분리됩니다.

### Test commit 학습 기록
- production invariant 대상: public formatting boundary에서 zero precision, prefixes, null values, percent extension, width/precision edge semantics 유지
- 재현하는 failure / boundary: libc-comparable boundary interaction 오류 또는 project-specific null/pointer/percent semantics drift
- test technique: differential matrices + fixed project-specific expectations
- 통과하는 production path: public `ft_printf` → parser/dispatch → text 또는 numeric layout/output path
- 이 test가 source상 증명하려는 것: source에 명시된 public boundary cases가 byte/return contract대로 유지됨
- 이 test가 증명하지 않는 것: system-call retry state machine이나 archive symbol/dependency boundary까지 증명하지 않습니다.
- 분류: broad boundary regression이며 일부는 deterministic fixed-contract regression입니다.
- 후속 회귀 방지 역할: numeric/text edge semantics와 deliberate project extensions가 후속 refactor에서 libc behavior와 혼동되어 바뀌는 것을 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - 세 matrix는 static format/value arrays의 모든 조합을 `EXPECT_PRINTF`로 실행합니다. fixed group은 null pointer의 `0x` 유지, precision zero digits suppression, width/alignment, null string truncation, percent width/precision extension을 `EXPECT_OUTPUT`으로 exact byte 비교합니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: exact SHA checkout을 로컬에 구성할 수 없어 connector로 commit patch와 당시 test implementation을 검사했습니다.
  - result: runtime pass/fail을 기록하지 않았습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - libc와 비교 가능한 numeric 경계는 matrix로, repository가 의도적으로 정한 null pointer/string/percent semantics는 fixed expected bytes로 검증한 public-contract commit입니다.

## 6. Invariant ledger

Source가 확정한 변화 축을 아래에 배치했습니다. “실제 코드 근거”는 학습자가 해당 SHA를 읽고 채웁니다.

| Invariant / concern | 도입 또는 초기 상태 | 강화 / 수정 | 고정한 검증 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| width/alignment layout | `ac27a26affaa`에서 decimal에 도입 | `c5ef742b84de`에서 hex/pointer에 반복 | 중복의 실제 형태를 두 SHA에서 비교 | 두 local writer가 모두 `padding = width - prefix_len - digit_len`과 spaces→prefix→digits→spaces를 구현 |
| precision/zero composition | `1fa064ca9d79`에서 zero suppression, precision zeros, field-zero ordering 도입 | `c5f627099ad9`의 prefix/flag precedence와 결합 | `f276ee73087c`에서 대표 상호작용 회귀 검증 | local decimal/hex의 `digit_len`, `zero_len`, `padding`, `pad_char`; parser normalization과 renderer prefix; focused `EXPECT_PRINTF` cases |
| shared layout responsibility | `177c8d03b353`에서 공통 numeric layout writer로 중앙화 | decimal/hex/pointer가 prepared digits/prefix/zero fact를 공급 | `12d715eba77d`에서 boundary matrix로 public semantics 확대 검증 | `src/ft_numeric_layout.c` helper와 세 conversion caller; signed/unsigned/hex matrices 및 fixed null-pointer cases |
| signed endpoint portability | 초기 decimal magnitude는 `long`이 더 넓다는 가정에 의존 | `ed3750fd081a`에서 unsigned domain 계산으로 복구 | 학습자가 sanitizer/functional coverage와 연결 여부를 별도 기록 | `src/ft_number.c`의 `(unsigned long)(-(value + 1)) + 1`; 기존 INT_MIN functional case는 path를 통과하지만 same-width model 전용 test는 없음 |

### 학습자 추가 기록

- source가 명시한 invariant 범위 안에서만 필요한 행을 추가합니다. 새 invariant를 확정 사실처럼 만들지 않습니다.
- 추가 기록:
  - 추가 행은 만들지 않았습니다. measurement와 rendering의 길이 일치는 Thread 5의 preflight에서 별도로 추적합니다.

## 7. Failure → Fix → Test 연결

| 기존 failure / risk | Fix / change | 수정 decision | Test / 학습 확인 |
| --- | --- | --- | --- |
| decimal과 hex가 동일한 placement rule을 중복 구현하여 edge case drift 가능 | `177c8d03b353` | shared `ft_printf_write_numeric_layout`에 authoritative ordering 집중 | `12d715eba77d` 및 기존 focused regressions가 어떤 공통 path를 통과하는지 확인 |
| zero precision/prefix/zero-padding 조합이 순서를 깨뜨릴 위험 | `1fa064ca9d79` + `c5f627099ad9` | component count 분리 + normalized prefix/flag precedence | `f276ee73087c`와 `12d715eba77d`에서 서로 다른 범위로 검증 |
| `INT_MIN`을 signed type에서 직접 양수 magnitude로 만들 수 없는 portability 위험 | `ed3750fd081a` | `-(value + 1)`을 representable하게 계산한 뒤 unsigned로 전환하고 1을 더함 | 학습자가 해당 fix를 통과하는 endpoint test를 실제 suite에서 찾아 기록 |

- 학습자 기록 — 실제 failure branch와 regression assertion을 연결한 추가 설명:
  - `177c8d03b353` 이후 focused/matrix cases는 모두 한 helper의 component ordering을 통과합니다. zero+precision zero는 helper의 `digit_len = 0` branch와 `%#.0x`, `%1.0d/u` cases가 연결됩니다. signed minimum fix는 기존 core `INT_MIN` differential을 통과하지만 해당 portability data model을 강제하는 deterministic fixture는 없습니다.

## 8. Ownership / state / responsibility 변화

| 시점 | Source상 owner / boundary | Source상 responsibility 변화 | 해당 SHA 코드 근거 |
| --- | --- | --- | --- |
| 초기 decimal/hex renderer | 각 conversion module | digit 생성과 field placement를 함께 소유하여 유사 rule이 중복 | `src/ft_number.c::ft_write_decimal`과 `src/ft_hex.c::ft_write_hex`에 동일 length variables/emission branches 존재 |
| `177c8d03b353` 이후 | conversion module vs shared layout | conversion은 prefix/prepared digits/length/zero fact를 공급하고 shared layout이 suppression, precision, padding, ordering, failure propagation을 소유 | 두 renderer가 `ft_printf_write_numeric_layout`을 호출하고 새 `src/ft_numeric_layout.c`가 여섯 component와 return checks를 보유 |
| signed magnitude | decimal conversion | layout과 분리된 value-to-magnitude 경계에서 `INT_MIN` portability를 책임 | `src/ft_number.c::ft_printf_print_signed`가 prefix/magnitude를 정한 뒤 digits와 shared layout으로 넘기며 `ed3750fd081a`가 그 계산만 수정 |

## 9. Thread 최종 상태

- Source가 확정한 도달점: numeric placement가 하나의 shared layout invariant로 수렴하고, signed endpoint 및 project-specific pointer/precision boundary까지 후속 fix와 tests로 보강된 상태입니다.
- 학습자 기록 — 마지막 commit 기준 실제 코드에서 확인한 최종 state:
  - decimal/hex/pointer는 각자 typed value를 digits와 prefix로 변환한 뒤 `ft_printf_write_numeric_layout` 하나를 통과합니다. helper가 zero suppression, precision zeros, field padding, byte 순서와 실패 반환을 소유합니다. signed minimum magnitude는 unsigned-safe 식을 사용하며, public matrices/fixed cases가 대표 경계를 고정합니다.
- 학습자 기록 — 이 Thread 밖에서만 해결되는 남은 문제를 source 범위 안에서 구분:
  - measurement가 같은 effective length를 계산하고 전체 call을 선검증하는 문제는 Thread 5, write retry/count/signal은 Thread 1, archive와 sanitizer 실행 경계는 Thread 6의 범위입니다.

## 10. 최종 architecture 또는 execution flow 정리

실제 SHA 코드를 읽은 뒤 아래 흐름을 완성합니다. source 설명만 복사하지 말고 함수/상태/branch를 연결합니다.

```text
[ft_printf / ft_printf_dispatch]
    -> [decimal 또는 hex/pointer renderer가 typed value를 digits·prefix·is_zero로 변환]
    -> [ft_printf_write_numeric_layout가 suppression·zero_len·padding 계산]
    -> [spaces -> prefix -> field zeroes -> precision zeroes -> digits -> trailing spaces]
    -> [각 output 실패 시 -1, 성공 시 shared context count가 public return으로 전달]
```

- 각 단계에 대응하는 SHA / file / function:
  - `ac27a26affaa` `src/ft_number.c`, `c5ef742b84de` `src/ft_hex.c`에서 local field model이 시작됩니다. `1fa064ca9d79`가 precision/zero components를 추가하고 `177c8d03b353` `src/ft_numeric_layout.c`로 중앙화합니다. `ed3750fd081a`가 decimal magnitude를 고칩니다.
- 핵심 state transition:
  - raw numeric value→prepared digits/prefix/is_zero→effective digit suppression→precision/field padding lengths→순차 output calls입니다. prefix와 precision zero는 field zero와 서로 다른 component입니다.
- failure가 끊기는 지점:
  - shared layout의 각 `ft_printf_putnchar`/`ft_printf_write` 반환을 즉시 검사해 후속 component를 중단합니다. output context의 sticky error가 최종 public `-1`을 만듭니다.
- 후속 fix/test가 보장한 지점:
  - `ed3750fd081a`가 minimum signed magnitude 경계를 수정하고, `f276ee73087c`와 `12d715eba77d`가 focused ordering 및 broad/fixed public boundary를 각각 보호합니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 정확한 시점의 코드로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 source와 그대로 유지했습니다.
- [x] final HEAD의 코드를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] 필요한 parent/직전 관련 SHA 비교를 실제 diff로 수행했습니다.
- [x] source가 확정한 사실과 내가 코드에서 확인한 사실을 구분했습니다.
- [x] fix의 기존 가정 → failure/risk → root cause → decision → code → test 연결을 필요한 곳에서 완성했습니다.
- [x] test commit의 target invariant, technique, production path, proves/not-proves를 구분했습니다.
- [x] Invariant ledger에 실제 코드 근거를 채웠습니다.
- [x] 이 Thread의 최종 architecture/execution flow를 commit history 순서로 설명할 수 있습니다.
===== END FILE: 03-shared-numeric-layout.md =====

===== BEGIN FILE: 04-string-precision-bounded-access.md =====
# String precision changes from output truncation to bounded access

## 1. Thread 목표

`%s` precision이 “출력 후 truncation”이 아니라 “caller object를 읽을 수 있는 최대 범위”까지 제한하는 memory-access contract로 교정되는 과정을 복원합니다.

### Source에서 확정된 significance

이 history는 출력 길이 제한과 메모리 접근 제한이 다르다는 점을 드러냅니다. precision은 단순한 사후 truncation이 아니라 renderer가 caller object를 얼마나 멀리 읽을 수 있는지 제한하며, focused regression이 다시 full `strlen` 방식으로 돌아가는 것을 방지합니다.

### 이 Thread에 명시적으로 연결되는 source invariant / engineering difficulty

- Invariant: `%s`는 precision이 있을 때 그 limit보다 더 멀리 읽지 않으며, 그 readable range 밖의 NUL terminator를 요구하지 않습니다.
- Engineering difficulty: `%s` precision을 단순 output truncation rule로 구현하지 않고 memory access 자체를 bound하는 문제입니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 implementation은 string의 full NUL-terminated length를 언제 계산하고 precision을 언제 적용하는가?
- 왜 `%.3s`는 NUL terminator가 없는 정확히 3-byte readable object에도 안전해야 하는가?
- fix는 출력 call만 막는 것이 아니라 length discovery 자체를 어떻게 바꾸는가?
- non-NUL three-byte regression은 ordinary NUL-terminated string test가 잡지 못한 무엇을 드러내는가?

## 3. 완료 기준

- `8e1cee3ed7f0`과 `9ac825379180`의 string length discovery를 실제 코드로 비교할 수 있습니다.
- precision이 width 계산과 output length뿐 아니라 memory read bound에도 사용됨을 설명할 수 있습니다.
- `e040e69db535`의 source object, precision, expected bytes, production path를 추적할 수 있습니다.
- functional regression만으로 보장되는 것과 sanitizer 아래에서 추가로 관찰 가능한 것을 구분할 수 있습니다.

## 4. Commit map

| SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- |
| `8e1cee3ed7f0` | feat(text): 문자열 정밀도와 퍼센트 0 채움 적용 | `B` | `FORMAT, EDGE` | Initially computes the full string length and truncates the emitted length afterward. |
| `9ac825379180` | fix(text): 문자열 정밀도 범위까지만 읽기 | `A` | `FORMAT, EDGE, RISK` | Moves the precision bound into the scan itself so memory beyond the permitted object range is never read. |
| `e040e69db535` | test(text): NUL 없는 제한 문자열 회귀 검증 | `B` | `FORMAT, TEST, EDGE` | Uses a non-NUL-terminated three-byte object to prove that `%.3s` requires only those three readable bytes. |

## 5. Commit별 학습 기록

> 원칙: 아래 기록은 final HEAD가 아니라 각 항목의 정확한 SHA에서 작성합니다. source가 확정하지 않은 파일명/함수명은 현재 골격에서 추측하지 않습니다.

## 5.1 `8e1cee3ed7f0` — feat(text): 문자열 정밀도와 퍼센트 0 채움 적용

- Importance: `B`
- Tags: `FORMAT, EDGE`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Initially computes the full string length and truncates the emitted length afterward.
- Commit Classification summary: Adds string truncation and zero padding for the project's percent extension.
- Importance 근거: The semantics matter, but the initial string implementation still scans to NUL before truncating and is corrected later; this is an intermediate feature step.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - string renderer는 전체 문자열을 출력했으므로 `%.Ns`가 요구하는 최대 출력 길이와 그 길이를 반영한 field width를 처리하지 못했습니다. formatted percent에도 repository가 허용한 zero padding 규칙이 없었습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - `%s`의 emitted `length`를 precision 이하로 줄이고, 그 effective length로 leading/trailing spaces를 계산합니다. percent renderer는 LEFT가 없고 ZERO가 있으면 padding character로 `'0'`을 선택합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - null string은 먼저 `"(null)"`로 치환됩니다. 이어서 `ft_local_strlen(string)`이 NUL까지 전부 읽고, 그 뒤에만 `precision < length`이면 `length = precision`으로 줄입니다. width와 `ft_printf_write`에는 줄어든 length가 전달됩니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - `9ac825379180`은 precision을 post-scan truncation에서 scan 종료 조건으로 이동합니다. percent zero padding은 이 fix의 대상이 아니며 그대로 유지됩니다.

### 해당 SHA에서 확인할 코드
- string renderer가 full NUL-terminated length를 먼저 계산하는 helper/call을 찾습니다.
- precision을 full length 계산 이후 emitted length에 적용하는 조건을 기록합니다.
- post-precision length가 width calculation에 사용되는 지점을 확인합니다.
- 이 SHA에서 precision이 output limit이지만 memory-read bound는 아닌 이유를 실제 scan loop로 설명합니다.
- 직후 `9ac825379180`과 비교할 수 있도록 scan termination condition을 그대로 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_text.c`: `ft_local_strlen(const char *string)`의 `while (string[length])`, `ft_printf_print_string`의 null mapping, post-scan precision branch, width/padding/output calls.
  - 같은 파일의 percent renderer에서 `pad_char`가 ZERO/LEFT flags로 선택됩니다. 이 Thread의 memory risk는 string branch에 한정됩니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 8e1cee3ed7f0, src/ft_text.c */
length = ft_local_strlen(string);
if (fmt->has_precision && fmt->precision < (int)length)
    length = (size_t)fmt->precision;
padding = fmt->width - (int)length;
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - visible `%s` precision과 width는 구현했지만 읽기 범위는 바꾸지 않은 중간 단계입니다. 출력은 최대 precision byte이지만 length discovery는 여전히 precision 밖의 NUL을 요구합니다.

## 5.2 `9ac825379180` — fix(text): 문자열 정밀도 범위까지만 읽기

- Importance: `A`
- Tags: `FORMAT, EDGE, RISK`
- Most Important Commits 목록: 포함
- Thread 내 역할: Moves the precision bound into the scan itself so memory beyond the permitted object range is never read.
- Commit Classification summary: Stops the string scan at precision instead of finding NUL first and truncating afterward.
- Importance 근거: The previous approach could read beyond the caller's valid bounded object even when the requested output was safe. This root-cause fix restores the memory-access contract at the renderer boundary.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - output length는 precision으로 줄였지만 `ft_local_strlen`은 먼저 NUL까지 진행했습니다. caller가 정확히 precision byte만 readable한 object를 넘기면 visible output은 유효해도 scan은 object 밖을 읽을 수 있었습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - length helper가 `t_format *fmt`를 받아 precision 유무와 현재 `length`를 scan condition에 직접 포함합니다. renderer의 별도 post-scan truncation은 제거됩니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - 각 반복 전에 `!has_precision || length < precision`을 먼저 확인하고 그 범위 안에서만 `string[length]`를 읽습니다. 반환된 하나의 bounded length가 padding 계산과 실제 write 길이에 모두 사용됩니다.
- 학습자 기록 — failure 또는 edge case:
  - precision 0이면 `string[0]`조차 평가하지 않고 길이 0을 반환합니다. precision 전에 NUL을 만나면 NUL에서 멈추고, NUL이 없어도 precision에 도달하면 종료합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: precision이 명시된 `%s`는 index `[0, precision)` 밖의 byte를 length discovery에서 요구하지 않습니다.
  - 미보장: precision이 생략된 `%s`는 여전히 정상적인 NUL-terminated string을 요구합니다. caller가 precision 범위 자체도 readable하지 않은 invalid pointer를 넘기는 문제까지 방어하지 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `e040e69db535`가 NUL 없는 `char[3]`와 `%.3s`를 사용해 정확히 이 종료 조건을 regression target으로 만듭니다. 후속 sanitizer target은 같은 test를 instrumented binary에서 실행할 수 있게 합니다.

### 해당 SHA에서 확인할 코드
- fix 직전 full scan → truncate 순서를 다시 확인하고 failure/risk의 root cause를 “length discovery 자체가 unbounded”로 기록합니다.
- fix SHA에서 local length scan이 NUL 또는 precision limit 중 먼저 만나는 조건에서 멈추는 loop를 기록합니다.
- 같은 bounded length가 width calculation과 output length 양쪽에 전달되는 path를 추적합니다.
- precision outside의 byte가 readable하지 않아도 되는 contract가 어떤 memory access 조건으로 보장되는지 설명합니다.
- 후속 `e040e69db535`가 이 exact boundary를 어떤 object로 재현하는지 연결합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_text.c`: `ft_local_strlen(const char *string, t_format *fmt)`와 `ft_printf_print_string`.
  - C의 `&&` left-to-right short-circuit 때문에 `length == precision`이면 `string[length]`가 평가되지 않습니다. null string은 scan 전에 `"(null)"`로 치환됩니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 9ac825379180, src/ft_text.c */
while ((!fmt->has_precision || length < (size_t)fmt->precision)
    && string[length])
    length++;
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `while (string[length])`로 full length를 구한 뒤 renderer에서 length를 precision으로 줄였습니다.
  - 이후: helper signature에 `fmt`가 추가되고 scan 자체가 precision에서 멈추며 별도 truncation branch가 삭제됩니다.

### Failure → Fix 추적
- 기존 가정/상태: precision은 full string length를 구한 뒤 emitted length만 줄여도 된다는 구현
- 실제 failure 또는 위험: precision 범위만 readable한 caller object에서 NUL을 찾기 위해 범위 밖을 읽을 수 있음
- source가 지목한 root cause: precision이 length discovery가 아니라 post-scan truncation에만 적용됨
- 수정된 decision/invariant: NUL 또는 precision 중 먼저 도달하면 scan을 멈추고 같은 bounded length를 width/output에 사용
- 학습자 기록 — 실제 수정 코드:
  - `while ((!fmt->has_precision || length < (size_t)fmt->precision) && string[length])`로 종료 조건을 바꾸고, `length = ft_local_strlen(string, fmt)` 뒤의 post-scan truncation을 제거했습니다.
- 학습자 기록 — regression test 연결:
  - 직접 연결된 후속 commit은 `e040e69db535`입니다. `char bounded[3]`에 NUL을 넣지 않고 `EXPECT_PRINTF("%.3s", bounded)`를 실행해 full NUL scan으로 되돌아가는 회귀를 겨냥합니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - `%s` precision의 의미를 output-size 조정에서 caller memory access bound로 바로잡은 root-cause fix입니다. width와 write가 사용하는 길이도 같은 bounded scan 결과이므로 관찰되는 출력과 실제 읽기 범위가 일치합니다.

## 5.3 `e040e69db535` — test(text): NUL 없는 제한 문자열 회귀 검증

- Importance: `B`
- Tags: `FORMAT, TEST, EDGE`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Uses a non-NUL-terminated three-byte object to prove that `%.3s` requires only those three readable bytes.
- Commit Classification summary: Formats a three-byte non-NUL array with a matching string precision.
- Importance 근거: The focused regression proves the bounded-read fix, but it is supporting evidence for the preceding A-level correction.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - ordinary string literals와 NUL-terminated arrays는 잘못된 full scan도 안전하게 끝나므로 memory-access bug를 구별하지 못합니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - 정확히 세 byte만 갖고 NUL은 없는 local object를 만들고 matching precision 3으로 formatting해 bounded-scan contract의 최소 재현 사례를 추가합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - production code는 바뀌지 않습니다. `run_text_differential_cases`가 `bounded[0..2] = {'a','b','c'}`를 구성하고 `EXPECT_PRINTF("%.3s", bounded)`를 호출합니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - 후속 `1b474fa2a5e3` sanitizer targets는 이 functional suite를 ASan/UBSan binary로 다시 compile해 out-of-bounds read를 더 직접적으로 탐지할 수 있게 합니다. 이 commit 자체에는 instrumentation이 없습니다.

### 해당 SHA에서 확인할 코드
- test source가 NUL terminator 없는 정확히 3-byte array를 어떻게 구성하는지 기록합니다.
- format `%.3s`와 expected `snprintf` result를 만드는 code를 확인합니다.
- project implementation이 exactly three bytes를 emit하는 assertion과 captured return/count를 확인합니다.
- 이 case가 `9ac825379180`의 bounded scan production path를 통과하는지 call path를 기록합니다.
- sanitizer와 함께 실행할 때 out-of-bounds regression이 더 직접적으로 관찰될 수 있다는 범위와, 이 한 case가 모든 string precision 조합을 증명하지는 않는다는 한계를 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `tests/test_ft_printf.c`: `run_text_differential_cases`, `EXPECT_PRINTF`, stdout capture/compare fixture.
  - call path는 `ft_printf`→parser→dispatch `%s`→`ft_printf_print_string`→precision-aware `ft_local_strlen`→padding/write입니다.
  - `EXPECT_PRINTF`는 `snprintf` expected return/bytes와 captured `ft_printf` return/length/bytes를 비교합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* e040e69db535, tests/test_ft_printf.c */
char bounded[3];

bounded[0] = 'a';
bounded[1] = 'b';
bounded[2] = 'c';
EXPECT_PRINTF("%.3s", bounded);
```

### Test commit 학습 기록
- production invariant 대상: `%s` precision이 memory access와 emitted length를 같은 bound로 제한
- 재현하는 failure / boundary: NUL 없는 3-byte object에서 precision 3인데 full NUL scan이 object 밖으로 진행하는 regression
- test technique: non-NUL bounded object + `snprintf` expected result + functional comparison; memory instrumentation과 결합 시 OOB 관찰 강화
- 통과하는 production path: string conversion → precision-aware length discovery → width/output
- 이 test가 source상 증명하려는 것: 이 exact bounded object case에서 exactly three bytes를 요구하고 bounded-scan fix의 regression target을 제공함
- 이 test가 증명하지 않는 것: 모든 string length/precision/width 조합의 memory safety를 단독으로 증명하지 않습니다.
- 분류: focused deterministic regression입니다.
- 후속 회귀 방지 역할: string length helper가 다시 full NUL scan 방식으로 돌아가는 회귀를 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - `run_text_differential_cases`의 유일한 case가 non-NUL stack array를 전달합니다. macro가 expected return 3, captured length 3, bytes `abc`를 간접적으로 exact 비교합니다. test source는 array 뒤 byte를 직접 guard/canary로 검사하지 않습니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: exact SHA checkout이 로컬에서 불가능해 connector로 test와 production 코드를 검사했습니다.
  - result: functional 또는 sanitizer 실행 결과를 주장하지 않습니다. 후속 Makefile이 이 suite를 sanitizer instrumentation으로 compile한다는 사실만 별도 SHA에서 확인했습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - NUL-terminated 입력으로는 드러나지 않는 bounded-read invariant를 세 byte non-NUL object로 고정한 focused regression입니다. functional comparison은 visible contract를 확인하고, sanitizer와 결합하면 범위 밖 읽기의 관찰력이 강화됩니다.

## 6. Invariant ledger

Source가 확정한 변화 축을 아래에 배치했습니다. “실제 코드 근거”는 학습자가 해당 SHA를 읽고 채웁니다.

| Invariant / concern | 도입 또는 초기 상태 | 강화 / 수정 | 고정한 검증 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| string precision 의미 | `8e1cee3ed7f0`에서 full length를 먼저 찾고 emitted length를 이후 truncate | `9ac825379180`에서 scan 자체를 precision으로 bound | `e040e69db535`에서 non-NUL 3-byte object로 회귀 고정 | `src/ft_text.c`의 full `while (string[length])`→precision-first short-circuit loop; `tests/test_ft_printf.c`의 `char bounded[3]`/`%.3s` |

### 학습자 추가 기록

- source가 명시한 invariant 범위 안에서만 필요한 행을 추가합니다. 새 invariant를 확정 사실처럼 만들지 않습니다.
- 추가 기록:
  - 추가 행은 만들지 않았습니다. null-string representation과 general width semantics는 이 Thread의 bounded-access 변화와 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 failure / risk | Fix / change | 수정 decision | Test / 학습 확인 |
| --- | --- | --- | --- |
| visible output은 3 byte인데 full NUL scan이 caller object 밖을 읽을 수 있음 | `9ac825379180` | length discovery가 NUL 또는 precision에서 즉시 종료 | `e040e69db535`의 non-NUL bounded object case |

- 학습자 기록 — 실제 failure branch와 regression assertion을 연결한 추가 설명:
  - 이전 helper는 precision을 알지 못해 NUL을 만날 때까지 `string[length]`를 평가했습니다. fix의 left operand가 `length < precision`을 먼저 검사하므로 index 3은 읽지 않습니다. regression은 NUL이 없는 길이 3 object를 넘겨 return/bytes가 `snprintf`와 같은지 검사하며, 후속 sanitizer 실행은 잘못된 full scan이 복구될 경우 OOB 진단 가능성을 추가합니다.

## 8. Ownership / state / responsibility 변화

| 시점 | Source상 owner / boundary | Source상 responsibility 변화 | 해당 SHA 코드 근거 |
| --- | --- | --- | --- |
| 초기 | string renderer의 length discovery | full logical string length를 먼저 구한 뒤 output만 truncate | `8e1cee3ed7f0` `ft_local_strlen(string)`의 unbounded NUL loop와 renderer의 별도 precision branch |
| fix 이후 | precision-aware length discovery | 읽기 범위와 width/output에 쓰일 effective length를 같은 bounded scan이 결정 | `9ac825379180` `ft_local_strlen(string, fmt)`의 short-circuit condition; 반환 length를 padding과 `ft_printf_write`가 그대로 사용 |

## 9. Thread 최종 상태

- Source가 확정한 도달점: string precision이 emitted-byte 제한과 memory-access 제한을 동시에 정의하고, precision 범위 밖 NUL terminator를 요구하지 않는 상태입니다.
- 학습자 기록 — 마지막 commit 기준 실제 코드에서 확인한 최종 state:
  - `%s` renderer는 null mapping 후 NUL과 precision 중 먼저 도달하는 곳까지 길이를 구합니다. 동일 length로 width를 계산하고 정확히 그 byte만 출력합니다. non-NUL `char[3]`/`%.3s` regression이 이 구현 경계를 직접 통과합니다.
- 학습자 기록 — 이 Thread 밖에서만 해결되는 남은 문제를 source 범위 안에서 구분:
  - 전체 format을 출력 전에 검증하는 문제는 Thread 5, 실제 output syscall 실패는 Thread 1, ASan/UBSan 실행 target과 그 환경 범위는 Thread 6에 속합니다.

## 10. 최종 architecture 또는 execution flow 정리

실제 SHA 코드를 읽은 뒤 아래 흐름을 완성합니다. source 설명만 복사하지 말고 함수/상태/branch를 연결합니다.

```text
[ft_printf dispatch의 %s]
    -> [ft_printf_print_string: null이면 "(null)"로 mapping]
    -> [ft_local_strlen(string, fmt): NUL 또는 precision에서 먼저 종료]
    -> [bounded length로 leading/trailing padding과 write length 결정]
    -> [output 실패 시 -1, 성공 시 bounded bytes/count 반환]
```

- 각 단계에 대응하는 SHA / file / function:
  - `8e1cee3ed7f0` `src/ft_text.c`가 initial precision output을 도입하고, `9ac825379180`의 같은 파일/helper가 access bound를 수정합니다. `e040e69db535` `tests/test_ft_printf.c::run_text_differential_cases`가 이를 고정합니다.
- 핵심 state transition:
  - raw pointer+normalized precision→bounded effective length→width padding→exact length write입니다. precision 0은 첫 byte를 읽지 않는 length 0 state입니다.
- failure가 끊기는 지점:
  - scan에는 error return이 없고 valid readable range를 caller contract로 전제합니다. 이후 leading padding, string write, trailing padding 각각의 negative return에서 renderer가 즉시 `-1`을 반환합니다.
- 후속 fix/test가 보장한 지점:
  - fix는 access condition 자체를 바꾸고, test는 termination byte가 없는 object를 사용해 post-scan truncation으로 돌아가는 회귀를 겨냥합니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 정확한 시점의 코드로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 source와 그대로 유지했습니다.
- [x] final HEAD의 코드를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] 필요한 parent/직전 관련 SHA 비교를 실제 diff로 수행했습니다.
- [x] source가 확정한 사실과 내가 코드에서 확인한 사실을 구분했습니다.
- [x] fix의 기존 가정 → failure/risk → root cause → decision → code → test 연결을 필요한 곳에서 완성했습니다.
- [x] test commit의 target invariant, technique, production path, proves/not-proves를 구분했습니다.
- [x] Invariant ledger에 실제 코드 근거를 채웠습니다.
- [x] 이 Thread의 최종 architecture/execution flow를 commit history 순서로 설명할 수 있습니다.
===== END FILE: 04-string-precision-bounded-access.md =====

===== BEGIN FILE: 05-whole-call-preflight.md =====
# Per-field validation becomes whole-call preflight

## 1. Thread 목표

한 field의 parser overflow check에서 출발해, 전체 format과 total output length를 첫 write 전에 검증하는 two-pass `va_list` architecture로 발전하는 과정을 복원합니다.

### Source에서 확정된 significance

한 field의 local parse validation만으로는 뒤쪽 invalid field가 있을 때 public no-output guarantee를 만들 수 없습니다. 두 번째 pass가 architecture를 바꾸어 format/length error는 preflight failure가 되고, device error만 partial external output을 남길 수 있게 됩니다.

### 이 Thread에 명시적으로 연결되는 source invariant / engineering difficulty

- Invariant: unsupported specifier, unterminated field, field-number overflow, total result > `INT_MAX`는 output 없이 `-1`을 반환합니다.
- Invariant: 시작되거나 copy된 각 `va_list`는 독립적으로 traversal되고 정확히 한 번 종료되며 measurement와 output은 호환되는 promoted type으로 argument를 소비합니다.
- Invariant: measurement와 rendering은 prefixes, zero suppression, precision zeros, field width를 포함한 effective length에 동의해야 합니다.
- Engineering difficulty: total output length를 미리 계산하여 late format error를 첫 write 전에 거부하면서도, variadic sequence를 두 번 독립적으로 소비하고 runtime device failure의 non-atomicity는 별도로 인정하는 문제입니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- field width/precision overflow를 한 field에서 막는 것만으로 왜 late invalid format의 no-output guarantee를 만들 수 없는가?
- `va_copy` measurement pass와 original `va_list` rendering pass는 각각 어떤 argument state를 소유하는가?
- measurement는 parser/rendering과 어떤 semantics를 반드시 동일하게 재현해야 하는가?
- format/length error와 runtime device/write error의 atomicity 경계는 어디인가?
- late invalid specifier나 total `INT_MAX` overflow test는 “renderer가 도달해서 실패”하는 구현과 어떻게 구별되는가?

## 3. 완료 기준

- `7984ddf2dd57`의 local field overflow validation과 `2d773acc5bd6`의 whole-call preflight 차이를 설명할 수 있습니다.
- `ft_printf`의 두 traversal과 모든 `va_start`/`va_copy`/`va_end` ownership을 success/failure path별로 기록할 수 있습니다.
- measurement와 rendering의 length model이 prefix, zero suppression, precision, width, bounded string read까지 일치하는지 실제 코드를 대조할 수 있습니다.
- `14059bd24f3e`의 late-failure cases가 zero emitted bytes를 요구한다는 점을 capture harness 결과와 연결할 수 있습니다.
- preflight error는 atomic하지만 runtime write failure는 이미 accepted byte를 rollback하지 못한다는 한계를 구분할 수 있습니다.

## 4. Commit map

| SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- |
| `7984ddf2dd57` | feat(parser): 포맷 필드 모델과 해석기 추가 | `S` | `ARCH, PARSER, CORE` | Rejects decimal width or precision values that overflow `int` while parsing one field. |
| `1b8049e411bb` | test(printf): 기본 변환과 포맷 경계 검증 | `A` | `FORMAT, TEST, VERIFY` | Adds parser-boundary tests and a differential harness, but errors later in a format can still follow earlier output. |
| `2d773acc5bd6` | fix(format): 지원 문법과 전체 출력 크기 선검증 | `S` | `ARCH, VARARGS, ATOMIC` | Adds a `va_copy` measurement pass that validates every field and the total result before any write. |
| `14059bd24f3e` | test(format): 잘못된 포맷의 무출력 실패 검증 | `A` | `ATOMIC, TEST, RISK` | Verifies that late unsupported fields and unrepresentable total lengths fail with zero emitted bytes. |

## 5. Commit별 학습 기록

> 원칙: 아래 기록은 final HEAD가 아니라 각 항목의 정확한 SHA에서 작성합니다. source가 확정하지 않은 파일명/함수명은 현재 골격에서 추측하지 않습니다.

## 5.1 `7984ddf2dd57` — feat(parser): 포맷 필드 모델과 해석기 추가

- Importance: `S`
- Tags: `ARCH, PARSER, CORE`
- Most Important Commits 목록: 포함
- Thread 내 역할: Rejects decimal width or precision values that overflow `int` while parsing one field.
- Commit Classification summary: Defines t_format and parses flags, width, precision, and specifiers with decimal overflow checks.
- Importance 근거: The parser creates the durable representation through which all conversions and both later passes communicate. It is indispensable to explaining the formatter's grammar and field-processing architecture.

### 학습 깊이
- 이 commit은 architecture/invariant의 핵심으로 취급합니다.
- 학습자 기록 — 직전 상태:
  - public loop는 literal과 `%%`만 직접 처리했고 flags, width, optional precision, specifier를 보존하는 field state가 없었습니다.
- 학습자 기록 — 해결하려던 문제:
  - raw decimal text를 `int` field로 변환할 때 overflow를 일으키지 않고, omitted precision과 `.0`을 구분하며, 이후 여러 consumer가 같은 grammar 결과를 사용해야 했습니다.
- 학습자 기록 — 기존 설계가 충분하지 않았던 이유:
  - conversion별 raw parsing은 grammar와 overflow behavior를 중복시키고, width/precision을 이미 overflow한 뒤 검사하면 잘못된 state/cursor를 만들 수 있습니다.
- 학습자 기록 — 선택한 핵심 decision:
  - `t_format`을 normalized representation으로 두고 decimal digit를 더하기 전에 `value > (INT_MAX - digit) / 10`을 검사하는 parser를 별도 module로 추가했습니다.
- 학습자 기록 — ownership / lifecycle / state transition:
  - caller가 stack `t_format`을 넘기며 parser가 초기화합니다. flags는 OR로 누적되고 width를 읽은 뒤 `.`가 있으면 `has_precision = 1`과 precision을 설정합니다. 마지막 specifier와 next cursor가 반환됩니다.
- 학습자 기록 — failure scenario와 public consequence:
  - 한 field의 width/precision이 `INT_MAX`를 넘으면 parser가 null을 반환합니다. 이 정확한 SHA에서는 parser가 아직 `ft_printf`에 연결되지 않았으므로 public no-output 결과는 아직 형성되지 않습니다.
- 학습자 기록 — 이 SHA가 보장하는 것:
  - parser API를 사용하는 consumer는 한 field의 decimal overflow를 state mutation 전에 검출하고 normalized field를 받을 수 있습니다.
- 학습자 기록 — 아직 보장하지 않는 것:
  - supported specifier 검증, 전체 format 검사, 모든 conversion length의 합, late failure 전 zero output, independent variadic traversal은 없습니다.
- 학습자 기록 — 후속 fix/test로 이어지는 지점:
  - `9e6d785628f3`에서 parser가 single-pass output loop에 연결되고 `1b8049e411bb`가 field-at-start overflow를 검사합니다. `2d773acc5bd6`가 동일 parser를 whole-call measurement에 재사용합니다.

### 해당 SHA에서 확인할 코드
- 해당 SHA의 `t_format` 정의에서 flag bit set, width, precision value, `has_precision`, specifier에 대응하는 실제 field를 기록합니다.
- parser가 flags → width → optional precision → specifier 순서로 cursor를 이동하는 실제 함수 호출 흐름을 추적합니다.
- repeated flag가 bitwise OR로 idempotent하게 누적되는 지점을 확인합니다.
- width/precision decimal parsing에서 multiply/add 전에 `INT_MAX` overflow를 차단하는 조건식을 기록합니다.
- precision omitted와 `.0`을 `has_precision`로 구분하는 state transition을 확인합니다.
- parser가 반환하는 next unread position을 caller가 아직 사용하지 않는 이 SHA의 boundary와, 직후 `9e6d785628f3`에서의 integration을 비교할 준비를 합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf_internal.h`: flag constants, `t_format`, parser prototypes.
  - `src/ft_parse.c`: `ft_printf_init_format`, flag parser, decimal parser, `ft_printf_parse`.
  - `Makefile`: parser source를 archive에 추가하지만 `src/ft_printf.c`는 이 commit에서 호출하지 않습니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 7984ddf2dd57, src/ft_parse.c, ft_parse_decimal */
while (ft_is_digit(**format))
{
    digit = **format - '0';
    if (*value > (INT_MAX - digit) / 10)
        return (-1);
    *value = *value * 10 + digit;
    (*format)++;
}
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: format field를 나타내는 state와 decimal overflow guard가 없었습니다.
  - 이후: parser module은 한 field를 안전하게 normalize하지만, public loop에는 아직 integration되지 않았습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - whole-call preflight의 공통 grammar 기반을 만든 commit입니다. 다만 이 단계의 validation 단위는 한 field이고, 외부 효과 전 전체 call을 판정하는 architecture는 아직 아닙니다.

## 5.2 `1b8049e411bb` — test(printf): 기본 변환과 포맷 경계 검증

- Importance: `A`
- Tags: `FORMAT, TEST, VERIFY`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Adds parser-boundary tests and a differential harness, but errors later in a format can still follow earlier output.
- Commit Classification summary: Creates stdout capture, libc differential checks, fixed expectations, and parser-overflow tests.
- Importance 근거: The harness materially changes confidence in all core conversions and return counts and becomes the basis for later regression matrices. It is significant verification rather than a defining runtime mechanism.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - conversion과 flag 기능은 구현됐지만 public return, embedded NUL을 포함한 exact bytes, parser boundary를 한 fixture에서 비교할 repository test target이 없었습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - stdout을 pipe로 redirect해 `ft_printf` bytes를 캡처하고, portable supported cases는 `snprintf`, repository-specific null/pointer behavior는 explicit output과 비교합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production code는 바뀌지 않습니다. test가 return count, captured length, byte sequence를 동시에 검사하며 width/precision parser overflow가 `-1`과 zero output인지 확인합니다.
- 학습자 기록 — failure 또는 edge case:
  - literal/escaped percent, `%c`의 embedded NUL, null string/pointer, signed extrema, all integer bases, flags/width/precision/mixed normalization, `2147483648` width/precision가 포함됩니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 실행에 성공한다면 이 suite의 broad supported surface와 field-at-start overflow contract가 일치합니다.
  - 미보장: format 앞부분이 이미 valid output을 만든 뒤 뒤쪽 unsupported/incomplete field가 나오는 경우 zero-output을 요구하지 않습니다. total length overflow를 whole-call 수준에서 검증하지도 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `2d773acc5bd6`이 preflight를 도입하고, `14059bd24f3e`가 기존 capture fixture에 late-fault 전용 macro/cases를 추가합니다.

### 해당 SHA에서 확인할 코드
- stdout redirection/pipe capture harness가 output bytes, captured byte count, `ft_printf` return을 어떤 순서로 수집/비교하는지 기록합니다.
- `snprintf`를 independent behavioral oracle로 사용하는 supported cases와 project-defined null string/pointer fixed expectations를 구분합니다.
- literal, escaped percent, embedded NUL, integer extrema, width/alignment/precision/zero/alternate/sign/mixed flag coverage를 test grouping으로 기록합니다.
- width/precision > `INT_MAX` field가 `-1`과 zero output을 요구하는 case를 확인합니다.
- 이 시점의 harness가 late invalid field 앞에서 이미 출력된 bytes까지 막는 whole-call preflight를 증명하지는 못함을 Thread 5 관점에서 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `Makefile`: test binary/`test` target 추가.
  - `tests/test_ft_printf.c`: `t_capture`, `capture_begin`, `capture_end`, `check_case`, `EXPECT_PRINTF`, `EXPECT_OUTPUT`, `expect_field_error`, grouped case runners.
  - capture는 stdout을 pipe write end로 바꾸고 호출 뒤 원래 descriptor를 복원한 후 read end에서 bytes를 수집합니다. comparison은 public return, captured byte count, exact memory bytes를 분리합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 1b8049e411bb, tests/test_ft_printf.c */
expect_field_error(__LINE__, "%2147483648d");
expect_field_error(__LINE__, "%.2147483648d");
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: repository 안에서 전체 public pipeline을 재현하는 automated harness가 없었습니다.
  - 이후: broad differential/fixed verification이 생겼지만 parser overflow cases는 invalid field가 출력의 첫 위치에 있어 late-error atomicity를 구별하지 않습니다.

### Test commit 학습 기록
- production invariant 대상: supported formatting의 exact bytes, captured byte count, public return count 및 parser field-overflow behavior
- 재현하는 failure / boundary: conversion/flag/width/precision 조합의 visible mismatch, count mismatch, tested field overflow의 partial/invalid result
- test technique: stdout pipe capture + `snprintf` differential oracle + project-defined fixed expectations
- 통과하는 production path: public `ft_printf` 전체 conversion pipeline
- 이 test가 source상 증명하려는 것: 넓은 정상/edge formatting surface와 return-count consistency, tested parser overflow boundary
- 이 test가 증명하지 않는 것: late invalid field의 whole-call no-output preflight, deterministic syscall retry sequence, release artifact boundary를 아직 증명하지 않습니다.
- 분류: broad integration/differential harness입니다.
- 후속 회귀 방지 역할: 후속 conversion/layout changes가 기존 public formatting surface를 깨는 회귀를 막는 기반 harness가 됩니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - `EXPECT_PRINTF`는 `snprintf` expected buffer/return과 captured implementation bytes를 `check_case`에서 비교합니다. `EXPECT_OUTPUT`은 libc와 다를 수 있는 explicit bytes를 사용합니다. `expect_field_error`는 `-1`과 captured length 0을 요구합니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: exact SHA의 repository checkout을 로컬에 확보하지 못해 GitHub connector로 Makefile/test/production diff를 검사했습니다.
  - result: 실행 결과를 기록하지 않습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 후속 모든 regression의 기반이 되는 broad public harness입니다. field 자체가 처음부터 invalid한 경우는 확인하지만, single-pass가 이미 쓴 앞부분까지 되돌리지 못하는 문제는 아직 노출하지 않습니다.

## 5.3 `2d773acc5bd6` — fix(format): 지원 문법과 전체 출력 크기 선검증

- Importance: `S`
- Tags: `ARCH, VARARGS, ATOMIC`
- Most Important Commits 목록: 포함
- Thread 내 역할: Adds a `va_copy` measurement pass that validates every field and the total result before any write.
- Commit Classification summary: Adds a va_copy measurement pass that validates grammar and total int length before output.
- Importance 근거: This changes the formatter from incremental discovery to a two-pass architecture: malformed, unsupported, or unrepresentable output is rejected with no external effect. The final correctness and failure contract cannot be explained without it.

### 학습 깊이
- 이 commit은 architecture/invariant의 핵심으로 취급합니다.
- 학습자 기록 — 직전 상태:
  - `ft_printf`는 original `va_list` 하나로 parse→dispatch→write를 반복했습니다. unsupported specifier는 dispatcher fallback으로 literal처럼 출력될 수 있었고, late parse/length failure는 earlier bytes가 이미 external fd에 전달된 뒤 발견될 수 있었습니다.
- 학습자 기록 — 해결하려던 문제:
  - supported grammar와 모든 conversion의 exact effective length를 첫 write 전에 확정하고, total public `int` count가 표현 가능한지 검증하면서 original variadic position을 rendering용으로 보존해야 했습니다.
- 학습자 기록 — 기존 설계가 충분하지 않았던 이유:
  - per-field parser guard는 현재 field만 판정합니다. single pass에서는 뒤쪽 `%q`, trailing `%`, 전체 합 overflow를 알 때 이미 literal/valid conversion을 썼으며, 한 `va_list`를 사전 순회하면 rendering argument cursor가 소진됩니다.
- 학습자 기록 — 선택한 핵심 decision:
  - `va_start(args)` 뒤 `va_copy(measure_args, args)`를 만들고 `ft_printf_measure`로 whole format을 순회합니다. 성공한 경우에만 original `args`와 output context로 기존 rendering loop를 실행합니다.
- 학습자 기록 — ownership / lifecycle / state transition:
  - null format은 variadic initialization 전에 `-1`입니다. 그 외 original `args`는 `va_start`로 생성되고 copied `measure_args`는 independent cursor입니다. preflight failure는 copy와 original을 각각 한 번 `va_end`하고 반환합니다. 성공은 copy를 끝낸 뒤 original만 rendering에서 소비하고 loop 종료 후 한 번 끝냅니다.
- 학습자 기록 — failure scenario와 public consequence:
  - parser null, unsupported/incomplete specifier, conversion measurement failure, component/total `INT_MAX` overflow는 measurement에서 `-1`; output context가 아직 초기화되거나 호출되지 않아 emitted byte 0입니다. preflight 성공 뒤 `write` 실패는 이미 accepted bytes가 남을 수 있고 public return만 `-1`입니다.
- 학습자 기록 — 이 SHA가 보장하는 것:
  - stable input과 defined variadic arguments를 전제로 grammar/length failure는 첫 write 전 판정되고, measurement/rendering은 specifier별 호환되는 promoted type으로 독립 소비됩니다.
- 학습자 기록 — 아직 보장하지 않는 것:
  - runtime device failure의 transactional rollback, 모든 OS timing, mutable input이 두 pass 사이에서 외부에 의해 바뀌는 상황은 보장하지 않습니다. measurement와 renderer는 별도 구현이므로 semantic parity를 tests로 계속 보호해야 합니다.
- 학습자 기록 — 후속 fix/test로 이어지는 지점:
  - `14059bd24f3e`가 literal/valid conversion 뒤 fault와 total overflow를 배치해 zero-output contract를 직접 검증합니다. `12d715eba77d` matrices와 release/sanitizer targets도 two-pass 경로를 통과합니다.

### 해당 SHA에서 확인할 코드
- fix 직전 single-pass `ft_printf`가 parse/render를 진행하며 late invalid field를 언제 발견하는지 parent code로 확인합니다.
- fix SHA에서 `va_copy`로 measurement traversal을 만들고 original `va_list`를 output용으로 보존하는 entry-point flow를 기록합니다.
- measurement module이 field parser를 재사용/호출하는 방식과 unsupported/incomplete syntax rejection 지점을 추적합니다.
- specifier별 argument를 rendering과 동일한 promoted type으로 소비하는 measurement dispatch를 기록합니다.
- string precision bound, numeric prefix, zero suppression, precision, width를 반영해 exact length를 계산하는 component를 실제 함수별로 기록합니다.
- component와 total을 `INT_MAX` 범위에서 누적하는 guard를 추적합니다.
- 성공/실패 모든 path에서 copied/original variadic traversal의 `va_end`가 balanced되는지 ownership table로 기록합니다.
- preflight failure에는 write가 시작되지 않고 runtime `write` failure만 partial external output이 가능한 execution split을 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `src/ft_printf.c`: `va_start`, `va_copy`, measurement failure cleanup, copy cleanup, rendering loop, original cleanup.
  - `src/ft_measure.c`: `ft_is_supported_specifier`, `ft_add_length`, digit-length helpers, `ft_measure_numeric`, `ft_measure_string`, signed/hex helpers, `ft_measure_conversion`, `ft_printf_measure`.
  - `src/ft_printf_internal.h`: measurement prototype. `Makefile`: new source inclusion.
  - type map: `c:int`, `s:char *`, `d/i:int`, `u/x/X:unsigned int`, `p:void *`, `%`: no `va_arg`. Renderer dispatch와 동일합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 2d773acc5bd6, src/ft_printf.c */
va_start(args, format);
va_copy(measure_args, args);
if (ft_printf_measure(format, &measure_args) < 0)
{
    va_end(measure_args);
    va_end(args);
    return (-1);
}
va_end(measure_args);
ft_printf_init(&ctx, 1);
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: original `args`를 한 번 순회하며 field를 발견하는 즉시 rendering/output했습니다.
  - 이후: copied `measure_args`가 whole grammar와 exact total을 먼저 검증하고, 성공한 경우에만 untouched original `args`가 같은 format을 rendering합니다.

### Failure → Fix 추적
- 기존 가정/상태: single pass에서 field를 발견하는 즉시 validate/render해도 public failure contract를 충분히 만족할 수 있다는 상태
- 실제 failure 또는 위험: late invalid syntax 또는 total `INT_MAX` overflow가 earlier output 이후에 발견되어 format/length error가 partial output을 남길 수 있음
- source가 지목한 root cause: whole-call grammar/length를 external effect 전에 알 수 있는데도 rendering과 discovery가 같은 pass에 묶여 있음
- 수정된 decision/invariant: copied `va_list` measurement pass로 전체 grammar/argument length를 먼저 검증하고, 성공 후 original `va_list`로 output
- 학습자 기록 — 실제 수정 코드:
  - entry의 `va_copy`/early cleanup과 `src/ft_measure.c` 전체가 fix입니다. `ft_add_length`는 `amount > INT_MAX || total > INT_MAX - amount`를 검사하고, `ft_printf_measure`가 parse/support/conversion/total 중 하나라도 실패하면 즉시 `-1`을 반환합니다.
- 학습자 기록 — regression test 연결:
  - `14059bd24f3e`가 직접 연결됩니다. `EXPECT_FORMAT_ERROR`는 return `-1`과 captured length 0을 함께 요구하며 prefix/valid conversion 뒤 invalid syntax와 component 합 overflow를 검사합니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - formatter를 incremental single pass에서 measure-then-render two pass로 바꾼 S-level architecture fix입니다. format/length 오류와 device 오류를 서로 다른 시점과 atomicity class로 분리합니다.

## 5.4 `14059bd24f3e` — test(format): 잘못된 포맷의 무출력 실패 검증

- Importance: `A`
- Tags: `ATOMIC, TEST, RISK`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Verifies that late unsupported fields and unrepresentable total lengths fail with zero emitted bytes.
- Commit Classification summary: Verifies late invalid fields and INT_MAX length failures produce no bytes.
- Importance 근거: These cases specifically prove the new preflight atomicity guarantee, including errors after otherwise valid prefixes and conversions. They materially protect an S-level contract.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - existing `expect_field_error`는 invalid field가 첫 위치에 있어 single-pass implementation도 zero output으로 통과할 수 있었습니다. preflight의 distinguishing property를 검증하려면 fault 앞에 valid output-producing content가 있어야 했습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - capture fixture 위에 `EXPECT_FORMAT_ERROR`를 추가해 actual return이 `-1`이고 captured length가 정확히 0인지 한 번에 검사합니다. cases는 late syntax와 total-length overflow를 별도로 구성합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production state는 바뀌지 않습니다. test는 preflight가 entire format을 통과하기 전에는 rendering pass가 시작되지 않는다는 observable invariant를 고정합니다.
- 학습자 기록 — failure 또는 edge case:
  - literal 뒤 oversized width, `%q`, trailing `%`; valid `%d` argument consumption 뒤 `%q`; `INT_MAX` width 앞/뒤 literal; sign/hash prefix와 `INT_MAX` precision이 total에 더해지는 경우입니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 열거된 malformed/unsupported/total-overflow calls는 public `-1`과 zero captured bytes를 함께 만족해야 합니다.
  - 미보장: preflight를 통과한 뒤 실제 `write`가 partial progress 후 실패할 때 bytes가 0이어야 한다는 보장은 하지 않습니다. variadic misuse나 concurrent source mutation도 검사하지 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - 이 Thread의 마지막 직접 regression입니다. 후속 functional matrices와 release/sanitizer targets가 same entry를 실행하지만 no-output contract를 새로 정의하지 않습니다.

### 해당 SHA에서 확인할 코드
- invalid specifier, trailing percent, width/precision > `INT_MAX`, total formatted length > `INT_MAX` case를 test file에서 식별합니다.
- fault가 literal text 뒤 또는 valid argument-consuming conversion 뒤에 위치하도록 만든 format을 기록합니다.
- each case가 return `-1`뿐 아니라 captured emitted bytes = 0을 요구하는 assertion을 확인합니다.
- overflow cases에서 literal, suffix, sign, alternate prefix, precision이 total length에 기여하는 test를 분류합니다.
- 이 test가 format/length preflight atomicity를 증명하지만 runtime device failure rollback은 증명하지 않는다는 경계를 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `tests/test_ft_printf.c`: `EXPECT_FORMAT_ERROR` macro와 `run_parser_boundary_cases`의 여덟 추가 calls.
  - field/local syntax: `prefix:%2147483648d`, `prefix:%q`, `prefix:%`.
  - valid argument 뒤 syntax: `value:%d bad:%q`.
  - total overflow: `x%2147483647d`, `%2147483647dX`, `%+.2147483647d`, `%#.2147483647x`.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 14059bd24f3e, tests/test_ft_printf.c, EXPECT_FORMAT_ERROR */
#define EXPECT_FORMAT_ERROR(FORMAT, ...) do { \
    char        actual[16]; \
    t_capture   capture; \
    int         actual_ret; \
    ssize_t     actual_len; \
    capture_begin(&capture, __LINE__); \
    actual_ret = ft_printf(FORMAT, ##__VA_ARGS__); \
    actual_len = capture_end(&capture, actual, sizeof(actual), __LINE__); \
    if (actual_ret != -1 || actual_len != 0) \
        fail_test(__LINE__, "invalid format produced output"); \
} while (0)
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: field-at-start overflow만 zero-output fixture로 검사했습니다.
  - 이후: earlier literal/valid conversion이 있는 late faults와 individually valid `INT_MAX` component에 추가 byte/prefix가 결합되는 total overflow를 검사합니다.

### Test commit 학습 기록
- production invariant 대상: format/length error는 whole-call preflight에서 첫 output 전에 실패해야 함
- 재현하는 failure / boundary: late invalid/unsupported/incomplete field 또는 total `INT_MAX` overflow가 earlier valid bytes 뒤에 발견되어 partial output을 남기는 regression
- test technique: late-fault format + stdout capture + zero-byte assertion
- 통과하는 production path: public `ft_printf` → copied-`va_list` measurement/preflight; failure 시 rendering pass 미진입
- 이 test가 source상 증명하려는 것: source에 열거된 malformed/overflow cases가 `-1`과 zero emitted bytes를 만족함
- 이 test가 증명하지 않는 것: runtime `write` failure의 rollback/atomicity는 증명하지 않으며 source도 이를 보장하지 않습니다.
- 분류: deterministic regression focused on S-level atomic preflight contract입니다.
- 후속 회귀 방지 역할: measurement/parser/length model 변경이 late error를 rendering 중 발견하도록 퇴행하는 것을 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - macro는 16-byte capture buffer면 충분합니다. invariant가 맞으면 output은 0이고, 틀린 single-pass implementation이라도 prefix 몇 byte를 관찰할 수 있습니다. 모든 case는 `capture_begin` 전후 actual return/length를 직접 검사합니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: exact SHA checkout을 로컬에 만들 수 없어 connector로 test macro/cases와 production preflight를 검사했습니다.
  - result: test pass를 주장하지 않습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - 단순 `-1` 검사가 아니라 fault 앞의 valid content가 한 byte도 나오지 않았는지를 검사해 two-pass architecture를 single-pass late failure와 구별하는 deterministic regression입니다.

## 6. Invariant ledger

Source가 확정한 변화 축을 아래에 배치했습니다. “실제 코드 근거”는 학습자가 해당 SHA를 읽고 채웁니다.

| Invariant / concern | 도입 또는 초기 상태 | 강화 / 수정 | 고정한 검증 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| field-local validation | `7984ddf2dd57`에서 width/precision decimal overflow를 parse 시 거부 | `1b8049e411bb`에서 parser boundary tests 추가 | late invalid field 앞의 earlier output까지 막지는 못하는 한계 기록 | decimal precheck와 field-at-start `expect_field_error`; parser는 당시 single-pass에서 field 도달 시점에만 호출 |
| whole-call preflight | `2d773acc5bd6`에서 `va_copy` measurement pass 추가 | grammar + promoted types + exact effective lengths + total `INT_MAX`를 첫 write 전 검증 | `14059bd24f3e`에서 late invalid/overflow zero-output 보장 고정 | `src/ft_measure.c` support/type/length functions, `src/ft_printf.c` measure-before-init/write, eight `EXPECT_FORMAT_ERROR` cases |
| atomicity boundary | format/length error는 preflight에서 external effect 이전에 판정 | runtime `write` failure는 output context가 처리하지만 이미 accepted bytes rollback 불가 | 학습자가 두 failure class를 execution flow에서 분리 | preflight failure branch는 `ft_printf_init`보다 앞; rendering의 `ft_printf_write`는 positive progress를 count/offset에 반영하고 later failure 시 public `-1` |

### 학습자 추가 기록

- source가 명시한 invariant 범위 안에서만 필요한 행을 추가합니다. 새 invariant를 확정 사실처럼 만들지 않습니다.
- 추가 기록:
  - 추가 행은 만들지 않았습니다. copied/original `va_list` ownership은 아래 responsibility 표에 구체화했습니다.

## 7. Failure → Fix → Test 연결

| 기존 failure / risk | Fix / change | 수정 decision | Test / 학습 확인 |
| --- | --- | --- | --- |
| 한 field는 유효해도 뒤쪽 unsupported/incomplete field가 earlier bytes 뒤에 발견될 수 있음 | `2d773acc5bd6` | whole-format measurement before rendering | `14059bd24f3e`에서 literal/valid conversion 뒤에 fault를 배치하여 zero-output 검증 |
| 각 field 길이는 `int`여도 전체 합이 `INT_MAX`를 넘을 수 있음 | `2d773acc5bd6` | component/total length accumulation을 preflight에서 검증 | `14059bd24f3e`에서 literal/suffix/sign/prefix/precision 기여까지 포함한 overflow cases |
| measurement가 original `va_list`를 소비하면 output pass argument state가 깨질 위험 | `2d773acc5bd6` | `va_copy`로 independent traversal 생성 후 balanced `va_end` | 학습자가 success/failure 모든 cleanup path를 실제 코드에서 확인 |

- 학습자 기록 — 실제 failure branch와 regression assertion을 연결한 추가 설명:
  - support/parser/conversion/`ft_add_length` 중 하나가 measurement에서 실패하면 entry가 두 lists를 종료하고 output context 초기화 전에 반환합니다. tests의 prefix/valid conversion bytes가 0이라는 assertion이 rendering 미진입을 관찰합니다. copy는 measurement에서 소비되고 성공 시 끝나며, original은 untouched 상태로 dispatch가 동일 type sequence를 소비합니다.

## 8. Ownership / state / responsibility 변화

| 시점 | Source상 owner / boundary | Source상 responsibility 변화 | 해당 SHA 코드 근거 |
| --- | --- | --- | --- |
| single-pass 이전 | original variadic traversal | parse/render가 진행되며 late error가 이미 출력된 byte 뒤에 발견될 수 있음 | parent `src/ft_printf.c`가 `va_start(args)` 후 바로 output context/loop에 진입하고 dispatch가 `args`를 소비 |
| `2d773acc5bd6` 이후 measurement | copied `va_list` | whole-format grammar/length validation과 동일 promoted type 소비 | `va_copy(measure_args, args)`와 `ft_printf_measure(format, &measure_args)`; failure/success 모두 copy를 한 번 `va_end` |
| `2d773acc5bd6` 이후 rendering | original `va_list` | preflight 성공 후 실제 output만 수행 | measurement 성공 뒤에만 `ft_printf_init`; dispatch는 original `args`, loop 후 `va_end(args)` |
| parser/measure/render | shared semantics contract | 같은 normalized field와 conversion length model을 서로 일치시켜야 함 | measurement와 rendering 모두 `ft_printf_parse`; type map 동일; bounded string scan과 numeric prefix/suppression/precision/width 계산이 renderer model을 mirror |

## 9. Thread 최종 상태

- Source가 확정한 도달점: final entry point가 `va_copy` measurement와 original-argument rendering의 두 traversal을 수행하여 format/length error를 output 전에 거부하고, runtime device failure만 non-atomic하게 남기는 상태입니다.
- 학습자 기록 — 마지막 commit 기준 실제 코드에서 확인한 최종 state:
  - entry는 null format을 즉시 거부하고, copied list로 complete grammar/type/effective length/total count를 검사합니다. 실패하면 두 list를 정리하고 zero output으로 `-1`; 성공하면 copy를 끝내고 original list로 rendering합니다. late-fault tests는 valid prefix/conversion이 있어도 captured length 0을 요구합니다.
- 학습자 기록 — 이 Thread 밖에서만 해결되는 남은 문제를 source 범위 안에서 구분:
  - actual syscall의 short write/EINTR/permanent failure와 accepted-byte nonrollback은 Thread 1, numeric layout authority는 Thread 3, archive/sanitizer execution 범위는 Thread 6에 속합니다.

## 10. 최종 architecture 또는 execution flow 정리

실제 SHA 코드를 읽은 뒤 아래 흐름을 완성합니다. source 설명만 복사하지 말고 함수/상태/branch를 연결합니다.

```text
[ft_printf: null check -> va_start(args) -> va_copy(measure_args)]
    -> [ft_printf_measure: parser + supported grammar + matching va_arg types + exact lengths]
    -> [failure: both va_end, no output, -1 / success: va_end(copy)]
    -> [original args로 parser -> dispatch -> renderer -> output state]
    -> [format/length 성공 count 또는 runtime write failure -1]
```

- 각 단계에 대응하는 SHA / file / function:
  - `7984ddf2dd57` `src/ft_parse.c`가 field grammar를 제공합니다. `2d773acc5bd6` `src/ft_measure.c::ft_printf_measure`와 `src/ft_printf.c::ft_printf`가 two-pass를 구성합니다. `14059bd24f3e` test macro/cases가 observable no-output을 고정합니다.
- 핵심 state transition:
  - original list initialized→copied list independently consumed→preflight result→copy ended→original list rendered/ended입니다. total은 `size_t`로 계산하되 매 add마다 `INT_MAX` representability를 유지합니다.
- failure가 끊기는 지점:
  - format/length failure는 `ft_printf_init`과 모든 output call 전에 끊깁니다. runtime failure는 rendering의 shared output loop에서 끊기므로 앞선 accepted bytes는 남을 수 있습니다.
- 후속 fix/test가 보장한 지점:
  - late `%q`, trailing `%`, oversized field와 literal/sign/hash/precision 때문에 total이 넘는 cases에서 return `-1`뿐 아니라 captured bytes 0을 확인하도록 설계됐습니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 정확한 시점의 코드로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 source와 그대로 유지했습니다.
- [x] final HEAD의 코드를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] 필요한 parent/직전 관련 SHA 비교를 실제 diff로 수행했습니다.
- [x] source가 확정한 사실과 내가 코드에서 확인한 사실을 구분했습니다.
- [x] fix의 기존 가정 → failure/risk → root cause → decision → code → test 연결을 필요한 곳에서 완성했습니다.
- [x] test commit의 target invariant, technique, production path, proves/not-proves를 구분했습니다.
- [x] Invariant ledger에 실제 코드 근거를 채웠습니다.
- [x] 이 Thread의 최종 architecture/execution flow를 commit history 순서로 설명할 수 있습니다.
===== END FILE: 05-whole-call-preflight.md =====

===== BEGIN FILE: 06-runtime-artifact-verification.md =====
# Verification reaches runtime and artifact boundaries

## 1. Thread 목표

verification이 단순 format 결과 비교에서 deterministic syscall failure, project-specific contract, distributable archive, sanitizer runtime까지 어떻게 확장되는지 복원합니다.

### Source에서 확정된 significance

각 verification layer는 서로 다른 질문에 답합니다. bytes 일치 여부, failure sequence에서 output contract 유지 여부, libc를 portable oracle로 쓸 수 없는 프로젝트 semantics의 고정 여부, distributable archive boundary, 실행된 경로의 UB/invalid memory access 여부를 각각 분리해 검증합니다.

### 이 Thread에 명시적으로 연결되는 source invariant / engineering difficulty

- Invariant: built archive는 expected definitions와 external dependencies를 노출하고 public header만 보는 consumer와 link됩니다.
- Invariant: output fault verification은 partial, interrupted, zero-progress, permanent failure를 deterministic하게 재현해야 합니다.
- Invariant: project-specific semantics는 portable libc oracle가 아닌 fixed expectations로 구분되어야 합니다.
- Engineering difficulty: system-call sequence를 deterministic하게 검증하면서 portable libc behavior와 formatted percent 같은 explicit project extension을 구분하고, runtime뿐 아니라 release artifact boundary까지 증거를 확장하는 문제입니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `snprintf`를 oracle로 사용할 수 있는 case와 fixed project expectation이 필요한 case는 어떻게 구분되는가?
- stdout capture는 emitted bytes, captured length, return value를 어떻게 함께 검증하는가?
- scripted writer는 nondeterministic OS behavior 없이 어떤 output state transition을 재현하는가?
- release test는 source-tree 내부 성공이 아니라 어떤 archive/public API/dependency/consumer boundary를 검증하는가?
- UBSan과 Linux ASan 경로는 library source와 fault binary까지 실제 instrument하는가?
- 각 test layer가 증명하지 않는 범위는 무엇인가?

## 3. 완료 기준

- 각 verification commit의 대상 invariant, failure/boundary, technique, production path, proved/not-proved 범위를 구분해 설명할 수 있습니다.
- libc differential test와 fixed expectation의 기준을 실제 test case로 분리할 수 있습니다.
- deterministic fault injection과 real broken-pipe signal test가 서로 다른 것을 검증함을 설명할 수 있습니다.
- archive member/global symbol/external dependency/out-of-tree consumer check를 실제 release script 단계와 연결할 수 있습니다.
- sanitizer target이 implementation 자체와 fault path를 instrument하는지 build command/object graph로 확인할 수 있습니다.

## 4. Commit map

| SHA | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- |
| `1b8049e411bb` | test(printf): 기본 변환과 포맷 경계 검증 | `A` | `FORMAT, TEST, VERIFY` | Establishes byte capture, return-count comparison, and libc differential testing. |
| `1223518652bd` | test(output): 쓰기 실패 시퀀스와 채움 전략 검증 | `A` | `OUTPUT, TEST, RISK` | Adds deterministic system-call and signal-policy verification. |
| `12d715eba77d` | test(printf): 공개 계약 경계 사례 확대 | `A` | `FORMAT, TEST, EDGE` | Records fixed expectations for deliberate project semantics that libc cannot serve as a portable oracle for. |
| `a87bcf560789` | test(release): 아카이브와 외부 소비자 검증 | `A` | `RELEASE, ARCH, VERIFY` | Verifies archive members, global definitions, external dependencies, and an out-of-tree consumer. |
| `1b474fa2a5e3` | build(sanitize): UBSan과 Linux ASan 검증 추가 | `B` | `VERIFY, TEST` | Runs normal and fault binaries under UBSan and a Linux GCC AddressSanitizer environment. |

## 5. Commit별 학습 기록

> 원칙: 아래 기록은 final HEAD가 아니라 각 항목의 정확한 SHA에서 작성합니다. source가 확정하지 않은 파일명/함수명은 현재 골격에서 추측하지 않습니다.

## 5.1 `1b8049e411bb` — test(printf): 기본 변환과 포맷 경계 검증

- Importance: `A`
- Tags: `FORMAT, TEST, VERIFY`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Establishes byte capture, return-count comparison, and libc differential testing.
- Commit Classification summary: Creates stdout capture, libc differential checks, fixed expectations, and parser-overflow tests.
- Importance 근거: The harness materially changes confidence in all core conversions and return counts and becomes the basis for later regression matrices. It is significant verification rather than a defining runtime mechanism.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - implementation은 있었지만 stdout의 raw bytes와 public return을 함께 수집하고, portable printf semantics와 repository-specific semantics를 구분해 자동 비교하는 in-tree harness가 없었습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - pipe와 `dup2`로 stdout을 캡처하고, supported portable cases는 `snprintf`가 만든 expected bytes/return과 비교합니다. null pointer/string처럼 libc 표현이 이식 가능한 단일 oracle이 아닌 case는 explicit expected output으로 분리합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production code는 바뀌지 않습니다. verification이 textual equality뿐 아니라 embedded NUL을 포함한 captured length, exact bytes, `ft_printf` return count를 독립적으로 확인합니다.
- 학습자 기록 — failure 또는 edge case:
  - literals, `%%`, `%c`와 NUL, strings/null, signed extrema, unsigned/hex/pointer, width/alignment/precision/zero/hash/plus/space와 mixed flags, parser width/precision overflow를 그룹별로 실행합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 실제 실행 시 열거된 public formatting surface의 bytes/length/return과 field-at-start parser overflow behavior를 검증합니다.
  - 미보장: deterministic short-write/EINTR sequence, late invalid field의 whole-call atomicity, archive contents, sanitizer diagnostics는 이 layer의 대상이 아닙니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `1223518652bd`가 system-call fault layer를 추가하고, `12d715eba77d`가 differential matrix와 fixed project contracts를 넓힙니다. release/sanitizer commits는 동일 functional suite를 다른 boundary에서 사용합니다.

### 해당 SHA에서 확인할 코드
- stdout redirection/pipe capture harness가 output bytes, captured byte count, `ft_printf` return을 어떤 순서로 수집/비교하는지 기록합니다.
- `snprintf`를 independent behavioral oracle로 사용하는 supported cases와 project-defined null string/pointer fixed expectations를 구분합니다.
- literal, escaped percent, embedded NUL, integer extrema, width/alignment/precision/zero/alternate/sign/mixed flag coverage를 test grouping으로 기록합니다.
- width/precision > `INT_MAX` field가 `-1`과 zero output을 요구하는 case를 확인합니다.
- 이 시점의 harness가 late invalid field 앞에서 이미 출력된 bytes까지 막는 whole-call preflight를 증명하지는 못함을 Thread 5 관점에서 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `Makefile`: archive를 link한 normal test binary와 `test` target 도입.
  - `tests/test_ft_printf.c`: `t_capture`, `capture_begin`, `capture_end`, `check_case`, `EXPECT_PRINTF`, `EXPECT_OUTPUT`, `expect_field_error`, case runners.
  - `capture_begin`은 stdout을 pipe write end로 교체하고, `capture_end`는 stdout을 복원한 뒤 read end에서 bytes를 읽습니다. `check_case`가 expected/actual return과 length, `memcmp` 결과를 검사합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 1b8049e411bb, tests/test_ft_printf.c, check_case */
if (actual_ret != expected_ret || actual_len != expected_ret
    || memcmp(expected, actual, (size_t)expected_ret) != 0)
{
    dprintf(STDERR_FILENO, "format: %s\n", format);
    dprintf(STDERR_FILENO, "expected ret: %d\n", expected_ret);
    dprintf(STDERR_FILENO, "actual ret: %d, actual bytes: %zd\n",
        actual_ret, actual_len);
    dump_bytes("expected", expected, expected_ret);
    dump_bytes("actual", actual, (int)actual_len);
    fail_test(line, "ft_printf output mismatch");
}
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: source/build만 있고 repository가 소유하는 broad public regression runner가 없었습니다.
  - 이후: normal conversion pipeline을 byte/return 기준으로 검증하지만 `write`를 대체하지 않으므로 rare syscall transition은 OS timing에 맡겨집니다.

### Test commit 학습 기록
- production invariant 대상: supported formatting의 exact bytes, captured byte count, public return count 및 parser field-overflow behavior
- 재현하는 failure / boundary: conversion/flag/width/precision 조합의 visible mismatch, count mismatch, tested field overflow의 partial/invalid result
- test technique: stdout pipe capture + `snprintf` differential oracle + project-defined fixed expectations
- 통과하는 production path: public `ft_printf` 전체 conversion pipeline
- 이 test가 source상 증명하려는 것: 넓은 정상/edge formatting surface와 return-count consistency, tested parser overflow boundary
- 이 test가 증명하지 않는 것: late invalid field의 whole-call no-output preflight, deterministic syscall retry sequence, release artifact boundary를 아직 증명하지 않습니다.
- 분류: broad integration/differential harness입니다.
- 후속 회귀 방지 역할: 후속 conversion/layout changes가 기존 public formatting surface를 깨는 회귀를 막는 기반 harness가 됩니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - `EXPECT_PRINTF`가 libc-comparable cases를, `EXPECT_OUTPUT`이 fixed bytes를, `expect_field_error`가 `-1`/zero capture를 담당합니다. embedded NUL은 C string 비교가 아니라 captured length와 memory bytes를 검사하므로 누락되지 않습니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: exact SHA checkout을 로컬에 확보하지 못해 GitHub connector로 Makefile, test fixture, production diff를 검사했습니다.
  - result: 실행 성공을 주장하지 않습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - visible output contract를 자동 검증하는 첫 broad layer입니다. portable semantics에는 libc differential oracle을 쓰고, 이식 가능한 비교 대상이 아닌 repository-defined 표현은 fixed bytes로 분리하는 기준을 만듭니다.

## 5.2 `1223518652bd` — test(output): 쓰기 실패 시퀀스와 채움 전략 검증

- Importance: `A`
- Tags: `OUTPUT, TEST, RISK`
- Most Important Commits 목록: 포함
- Thread 내 역할: Adds deterministic system-call and signal-policy verification.
- Commit Classification summary: Injects partial writes, EINTR, EPIPE, zero progress, and verifies SIGPIPE and padding chunks.
- Importance 근거: This provides deterministic evidence for the S-level output state machine and confirms that the library does not mutate process signal policy. It is unusually strong failure-path verification.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - ordinary pipe capture는 kernel이 보통 full write를 받아들이므로 positive short write, 정확한 EINTR 위치, zero return, partial-then-EPIPE sequence를 재현성 있게 만들 수 없었습니다. padding chunk size도 visible bytes만으로는 알 수 없습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - `FT_PRINTF_TEST_WRITE`로 production `ft_printf_write`의 system-call 한 지점만 test writer로 바꾸고, scripted actions와 accepted bytes/call statistics를 global fixture에 기록합니다. real `write`가 필요한 SIGPIPE policy는 normal suite에서 별도로 검사합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production code는 바뀌지 않습니다. test layer가 requested length, call count, largest request, accepted output prefix를 관찰해 retry offset과 stop behavior를 직접 판정합니다.
- 학습자 기록 — failure 또는 edge case:
  - PART 2→ALL, EINTR→PART 3→EINTR→ALL, immediate EPIPE, PART 3→EPIPE, ZERO, `%1000d`가 포함됩니다. real broken pipe는 caller가 설치한 SIGPIPE handler가 호출되고 유지되는지 확인합니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: scripted sequence에 대해 production write loop의 offset/count/error transitions와 64-byte chunk policy를 deterministic하게 검사하고, real pipe에서 library가 signal disposition을 덮지 않는지 검사합니다.
  - 미보장: 모든 kernel scheduling, `SSIZE_MAX` 크기의 실제 allocation/output, OS가 이미 accepted한 byte의 rollback, 다른 asynchronous signals의 전체 조합은 증명하지 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `1b474fa2a5e3`가 normal suite뿐 아니라 이 fault source와 implementation sources도 sanitizer flags로 다시 compile합니다.

### 해당 SHA에서 확인할 코드
- fault binary가 `FT_PRINTF_TEST_WRITE` seam을 사용하도록 build되는 실제 Makefile rule/compile definition을 기록합니다.
- scripted writer가 configured return sequence, request length, accepted bytes, call record를 어떤 상태로 보관하는지 확인합니다.
- full write, short write, `EINTR`, zero, `EPIPE` case마다 production `ft_printf_write`의 어떤 branch를 통과하는지 매핑합니다.
- partial failure 이전에 accepted된 bytes가 exact하게 남고 이후 write가 중단되는지 assertion을 확인합니다.
- width 1000 padding case의 request count와 maximum chunk 64 assertion을 확인합니다.
- real broken-pipe test에서 caller-owned `SIGPIPE` handler 설치/복원과 `ft_printf` return을 함께 확인합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `Makefile`: normal archive suite 실행 뒤 `-DFT_PRINTF_TEST_WRITE tests/test_output_faults.c $(SRC)`로 fault binary를 build/run합니다. archive를 link하지 않고 implementation source를 seam과 함께 compile합니다.
  - `tests/test_output_faults.c`: `t_write_action`, `t_write_step`, `g_steps`, step/call/output statistics, `ft_printf_test_write`, `run_retry_cases`, `run_failure_cases`, `run_padding_case`.
  - `tests/test_ft_printf.c`: `run_sigpipe_policy_case`가 real pipe, caller handler, stdout redirection/복원을 소유합니다.
  - production branch mapping: PART=positive progress; EINTR=`errno == EINTR` continue; ZERO/EPIPE=sticky error; subsequent renderer/output calls stop.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 1223518652bd, tests/test_output_faults.c */
add_step(WRITE_PART, 3);
add_step(WRITE_EPIPE, 0);
result = ft_printf("%s", "partial failure");
if (result != -1 || g_output_length != 3
    || memcmp(g_output, "par", 3) != 0)
    fail_test(__LINE__, "failure after a partial write was not preserved");
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: functional suite는 real stdout byte result만 관찰했습니다.
  - 이후: compile-time seam이 요청/응답 sequence를 test가 소유하고, normal suite에는 real SIGPIPE integration case가 추가됩니다.

### Test commit 학습 기록
- production invariant 대상: positive short write progress, `EINTR` retry, zero/permanent failure stop, `SIGPIPE` ownership boundary, 64-byte padding chunk policy
- 재현하는 failure / boundary: scripted partial write, `EINTR`, zero-byte result, `EPIPE`; 별도의 real broken pipe
- test technique: compile-time write seam을 통한 deterministic fault injection + call/emitted-byte recording + real signal-policy case
- 통과하는 production path: `ft_printf` → padding/conversion output → `ft_printf_write` → substituted writer 또는 real `write`
- 이 test가 source상 증명하려는 것: retry offset, no-progress handling, hard-failure stop, prior accepted bytes, chunk bound, caller-owned signal disposition
- 이 test가 증명하지 않는 것: 모든 OS scheduling/timing behavior나 이미 OS가 받아들인 byte의 rollback을 증명하지 않습니다.
- 분류: deterministic regression/fault-injection 중심이며 signal policy는 real integration boundary case를 포함합니다.
- 후속 회귀 방지 역할: output loop, retry policy, padding optimization이 바뀌어도 동일 state transition과 signal boundary를 유지하도록 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - `ft_printf_test_write`는 default ALL 또는 다음 scripted step을 소비합니다. PART는 request 이하 amount를 `g_output`에 복사하고, EINTR/EPIPE는 errno와 `-1`, ZERO는 0을 반환합니다. every call은 `g_write_calls`와 `g_largest_write`를 갱신합니다.
  - retry assertions: `partial`은 2 calls/max 7, `interrupt`는 4 calls/max 9. failure assertions은 immediate EPIPE/zero의 output 0과 partial failure의 exact `par`를 확인합니다. padding은 return/output 1000, calls 17, max 64, 마지막 bytes를 확인합니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: exact SHA checkout을 로컬에 구성할 수 없어 connector로 test seam, Makefile, production output loop를 검사했습니다.
  - result: runtime 결과를 주장하지 않습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - visible final bytes만 보던 검증을 system-call transition까지 내린 commit입니다. fault seam은 deterministic state evidence를, real broken pipe는 process-wide signal policy를 library가 소유하지 않는 integration evidence를 제공합니다.

## 5.3 `12d715eba77d` — test(printf): 공개 계약 경계 사례 확대

- Importance: `A`
- Tags: `FORMAT, TEST, EDGE`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Records fixed expectations for deliberate project semantics that libc cannot serve as a portable oracle for.
- Commit Classification summary: Locks down zero precision, prefixes, null values, percent extensions, and width/precision boundary matrices.
- Importance 근거: Several expectations are deliberate project contracts rather than portable libc behavior. Fixing them explicitly is significant for preserving the library's actual public semantics.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - focused cases는 있었지만 zero precision/width/prefix를 값 배열과 체계적으로 교차하지 않았고, null format/string/pointer 및 formatted-percent extension을 하나의 explicit contract group으로 충분히 고정하지 않았습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - standard-comparable signed/unsigned/hex combinations는 `EXPECT_PRINTF` matrices로 `snprintf`와 비교합니다. implementation이 의도적으로 정한 null pointer/string/percent bytes는 `EXPECT_OUTPUT`으로 분리합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production code는 바뀌지 않습니다. test oracle의 책임이 명확해집니다. libc에 위임할 부분과 repository가 직접 소유할 부분을 case 단위로 나눕니다.
- 학습자 기록 — failure 또는 edge case:
  - signed 12 formats×5 values, unsigned 10×5, hex 11×5; null format, empty format, null pointer precision/width, null string width/precision, `%05%|%-5%|%.%` extension이 포함됩니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: 열거된 differential matrix와 fixed expected bytes가 public return/output contract로 유지됩니다.
  - 미보장: syscall transition, archive ABI/dependencies, unexecuted value space, memory instrumentation은 다른 layer의 대상입니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `a87bcf560789`은 functional semantics에서 distributable archive로 검증 범위를 이동하고, `1b474fa2a5e3`은 이 expanded suite를 sanitizer build에 포함합니다.

### 해당 SHA에서 확인할 코드
- precision zero, digit-count transition, signs, alternate prefix, content-width boundary에 대한 differential matrix를 찾아 입력 축과 expected oracle을 기록합니다.
- null format, empty format, null string, null pointer, formatted percent처럼 fixed project expectation으로 분리된 case를 식별합니다.
- 이 Thread 관점에서는 특히 null pointer의 `0x` prefix와 precision-zero digit suppression, width가 content보다 작은/큰 경우의 production layout path를 추적합니다.
- libc와 비교하지 않는 fixed expectation이 “왜 project contract인지” source description과 test implementation을 대응시킵니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `tests/test_ft_printf.c`: `run_signed_boundary_matrix`, `run_unsigned_boundary_matrix`, `run_hex_boundary_matrix`, `run_public_contract_boundary_cases`.
  - `EXPECT_PRINTF`: matrices의 libc-comparable results. `EXPECT_FORMAT_ERROR(NULL)`과 `EXPECT_OUTPUT`: repository-defined null/empty/pointer/string/percent cases.
- 학습자 기록 — 필요한 최소 코드 발췌:
```c
/* 12d715eba77d, tests/test_ft_printf.c,
 * run_public_contract_boundary_cases */
EXPECT_OUTPUT("0x", "%.0p", (void *)0);
EXPECT_OUTPUT("      0x", "%8.0p", (void *)0);
EXPECT_OUTPUT("0x      ", "%-8.0p", (void *)0);
EXPECT_OUTPUT("  0x0000", "%8.4p", (void *)0);
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: broad/focused cases가 있었으나 oracle classification과 boundary cross-product가 작았습니다.
  - 이후: portable behavior는 matrices로 확장되고 deliberate extension/representation은 fixed exact bytes로 명시됩니다.

### Test commit 학습 기록
- production invariant 대상: public formatting boundary에서 zero precision, prefixes, null values, percent extension, width/precision edge semantics 유지
- 재현하는 failure / boundary: libc-comparable boundary interaction 오류 또는 project-specific null/pointer/percent semantics drift
- test technique: differential matrices + fixed project-specific expectations
- 통과하는 production path: public `ft_printf` → parser/dispatch → text 또는 numeric layout/output path
- 이 test가 source상 증명하려는 것: source에 명시된 public boundary cases가 byte/return contract대로 유지됨
- 이 test가 증명하지 않는 것: system-call retry state machine이나 archive symbol/dependency boundary까지 증명하지 않습니다.
- 분류: broad boundary regression이며 일부는 deterministic fixed-contract regression입니다.
- 후속 회귀 방지 역할: numeric/text edge semantics와 deliberate project extensions가 후속 refactor에서 libc behavior와 혼동되어 바뀌는 것을 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - matrices는 static arrays의 Cartesian product를 `EXPECT_PRINTF`로 반복합니다. fixed group은 literal expected buffer를 capture fixture에 넘겨 return/length/bytes를 검사하므로 host libc의 null pointer 표기나 formatted-percent 해석에 의존하지 않습니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: exact SHA checkout 부재로 connector를 통해 test implementation을 검사했습니다.
  - result: 실제 pass/fail을 기록하지 않습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - oracle의 적용 범위를 명확히 한 public-contract commit입니다. standard-comparable 조합은 differential evidence를 얻고, null representation과 formatted percent처럼 repository가 결정한 semantics는 fixed expectation으로 직접 소유합니다.

## 5.4 `a87bcf560789` — test(release): 아카이브와 외부 소비자 검증

- Importance: `A`
- Tags: `RELEASE, ARCH, VERIFY`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Verifies archive members, global definitions, external dependencies, and an out-of-tree consumer.
- Commit Classification summary: Checks archive order, global definitions, external dependencies, and a header-only external consumer.
- Importance 근거: The commit establishes the distributable artifact and consumer boundary as reproducible contracts, adding significant release-level evidence beyond in-tree tests.

### 학습 깊이
- 이 commit은 주요 subsystem/boundary/failure path/integration point 수준으로 추적합니다.
- 학습자 기록 — 직전 상태와 문제:
  - in-tree test는 source tree의 include paths, source/object availability 또는 accidental globals/dependencies에 의존해도 통과할 수 있습니다. archive가 실제 배포 단위로 올바른지는 별도 증거가 없었습니다.
- 학습자 기록 — 설계 판단 / boundary 변화:
  - `release-check`가 built archive를 manifest와 비교하고 `nm -g` output을 정의/미해결 symbol로 분류합니다. 임시 directory에는 public header, archive, consumer source만 복사해 독립 compile/run합니다.
- 학습자 기록 — 핵심 state/invariant 변화:
  - production/archive 내용은 바뀌지 않습니다. verification contract가 source behavior에서 archive composition, global namespace, external runtime dependencies, consumer-visible packaging까지 확장됩니다.
- 학습자 기록 — failure 또는 edge case:
  - missing/extra/reordered object, unexpected global definition, undeclared unresolved symbol, unsupported OS/compiler normalization, repository-relative include/link assumption, wrong consumer output/return, temp cleanup 실패가 즉시 script failure입니다.
- 학습자 기록 — 보장하는 것 / 보장하지 않는 것:
  - 보장: Linux 또는 지원된 Darwin/compiler에서 manifest와 일치하는 archive가 public header만으로 external consumer에 link되고 expected output을 냅니다.
  - 미보장: 다른 플랫폼/toolchain, shared-library ABI, binary compatibility across versions, 전체 formatter functional path나 sanitizer safety를 증명하지 않습니다.
- 학습자 기록 — 다음 관련 commit 연결:
  - `1b474fa2a5e3`의 aggregate `check`가 `release-check`를 behavioral tests와 UBSan 사이에 포함합니다.

### 해당 SHA에서 확인할 코드
- release script의 expected archive member manifest와 actual archive member extraction/comparison 단계를 기록합니다.
- globally defined API symbol manifest와 undeclared global symbol rejection logic을 확인합니다.
- unresolved dependency allowlist에서 `write`, platform errno accessor, compiler stack-protector symbols가 어떻게 normalize/허용되는지 기록합니다.
- isolated temporary directory consumer가 copied public header + archive 외에 repository-relative input을 사용하지 않는지 build command로 확인합니다.
- consumer의 expected output/return behavior와 temporary-state cleanup assertion을 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `Makefile`: `release-check`가 `CC=... sh tests/check_release.sh $(NAME) include tests/test_consumer.c`를 실행합니다.
  - `tests/check_release.sh`: `set -eu`, `mktemp`, cleanup trap, archive/global/external manifests, temp consumer build/run, explicit cleanup assertion.
  - expected members: `ft_printf.o`, `ft_output.o`, `ft_parse.o`, `ft_measure.o`, `ft_dispatch.o`, `ft_text.o`, `ft_numeric_layout.o`, `ft_number.o`, `ft_hex.o` 순서.
  - Linux external set: `__errno_location`, `write`; Darwin: `__error`, `write`, Clang이면 `__stack_chk_fail`, `__stack_chk_guard` 추가. Darwin leading underscore는 normalize합니다.
  - `tests/test_consumer.c`: public header만 include하고 `ft_printf("consumer:%d:%s\n", 17, "ok") == 15`를 요구합니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```sh
# a87bcf560789, tests/check_release.sh
actual_members=$(ar t "$archive" | sed '/^__.SYMDEF/d')

if [ "$actual_members" != "$expected_members" ]; then
    printf '%s\n' "archive member list mismatch" >&2
    exit 1
fi
```
- 학습자 기록 — 직전 관련 SHA와 비교한 핵심 diff:
  - 이전: `make test`는 source tree 안에서 archive/implementation을 실행했습니다.
  - 이후: `make release-check`가 archive 자체를 inspect하고 최소 배포 inputs만 가진 temporary consumer를 compile/run합니다.

### Test commit 학습 기록
- production invariant 대상: built archive의 members/global definitions/external dependencies/public-header-only consumer boundary
- 재현하는 failure / boundary: missing/extra object, accidental symbol export, undeclared dependency, source-tree-only link assumption, temporary-state pollution
- test technique: artifact manifest comparison + unresolved-symbol normalization/allowlist + isolated out-of-tree consumer build/run
- 통과하는 production path: archive packaging/public header/linker/runtime consumer boundary
- 이 test가 source상 증명하려는 것: delivered static archive가 source에 정의된 artifact contract로 외부 consumer에게 link/run 가능함
- 이 test가 증명하지 않는 것: 모든 internal formatting path나 sanitizer memory safety를 증명하지 않습니다.
- 분류: release/artifact integration regression입니다.
- 후속 회귀 방지 역할: build/object/symbol/header changes가 distributable boundary를 깨는 회귀를 막습니다.
- 학습자 기록 — 실제 test 함수/fixture/seam/assertion:
  - `ar t`의 exact multiline string, `nm -g`에서 만든 sorted unique definitions/external files, `cmp`/`diff`, OS/compiler case statement, isolated compiler command, output `consumer:17:ok`, cleanup 후 path nonexistence가 각각 독립 assertion입니다. expected global definitions는 17개 이름으로 고정됩니다.
- 학습자 기록 — 직접 실행했다면 command / 환경 / 결과:
  - command: 미실행
  - environment: repository archive와 exact checkout이 로컬에 없어 script source/Makefile/consumer만 connector로 검사했습니다.
  - result: archive manifest나 consumer runtime 성공을 실행 결과처럼 주장하지 않습니다.

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - source tree 내부 기능 검증과 배포 산출물 검증을 분리한 commit입니다. archive의 object/symbol/dependency manifest와 public-header-only consumer를 함께 검사해 실제 전달 단위의 계약을 고정합니다.

## 5.5 `1b474fa2a5e3` — build(sanitize): UBSan과 Linux ASan 검증 추가

- Importance: `B`
- Tags: `VERIFY, TEST`
- Most Important Commits 목록: 미포함
- Thread 내 역할: Runs normal and fault binaries under UBSan and a Linux GCC AddressSanitizer environment.
- Commit Classification summary: Adds UBSan targets and a Dockerized GCC AddressSanitizer path.
- Importance 근거: The sanitizer matrix is useful safety infrastructure, but it applies standard verification without changing the formatter's architecture or contract.

### 학습 깊이
- 이 commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/code 변화에 집중합니다.
- 학습자 기록 — 직전 상태 대비 필요한 변화:
  - value/byte assertions는 실행 중 UB와 invalid memory access가 결과를 우연히 맞추더라도 진단하지 못할 수 있고, archive가 uninstrumented objects로 이미 build된 상태라 단순 link만으로는 implementation 전체를 sanitizer 처리할 수 없습니다.
- 학습자 기록 — 이 commit이 맡는 구현 책임:
  - normal functional source와 deterministic fault source를 각각 모든 `$(SRC)`와 직접 compile해 UBSan 또는 ASan+UBSan instrumentation을 적용하고 실행합니다. Linux GCC path는 pinned Docker image로 제공합니다.
- 학습자 기록 — 해당 SHA에서 확인한 핵심 상태/flow 변화:
  - `sanitize-address`는 `-fsanitize=address,undefined`, `sanitize-undefined`는 `-fsanitize=undefined`로 normal/fault 두 binaries를 만들고 halt-on-error로 실행합니다. fault build에는 동일 `FT_PRINTF_TEST_WRITE` define이 적용됩니다.
- 학습자 기록 — 이후 commit이 보강하거나 대체하는 부분:
  - scaffold에 연결된 후속 verification commit은 없습니다. 이 commit은 existing contracts에 runtime diagnostics layer를 더하며 architecture를 바꾸지 않습니다.

### 해당 SHA에서 확인할 코드
- UBSan target이 library implementation과 functional suite를 별도 instrumented build로 compile하는 command/flags를 기록합니다.
- normal functional binary와 deterministic output-fault binary 모두 instrument되는지 object/source list로 확인합니다.
- Linux GCC container의 pinned environment에서 combined ASan/UBSan build가 어떻게 실행되는지 기록합니다.
- object-free test binary가 uninstrumented archive를 재사용하지 않는다는 근거를 build graph에서 확인합니다.
- aggregate `check`가 behavioral tests, release boundary, UB detection, whitespace 중 무엇을 포함하고 `sanitize-linux`/leak coverage는 무엇을 암시하지 않는지 기록합니다.
- 학습자 기록 — 실제 파일 경로 / 함수 / 구조체 / branch:
  - `Makefile`: sanitizer binary/flag variables, `sanitize-address`, `sanitize-undefined`, `sanitize`, `sanitize-linux`, `check`, cleanup targets.
  - normal commands는 `tests/test_ft_printf.c $(SRC)`, fault commands는 `-DFT_PRINTF_TEST_WRITE tests/test_output_faults.c $(SRC)`입니다. `$(NAME)` archive를 재사용하지 않습니다.
  - `sanitize-linux`는 read-only source mount의 `gcc:14-bookworm` container에서 `/source`를 writable temp로 복사하고 `make sanitize-address SANITIZER_CC=gcc`를 실행합니다.
  - `sanitize`는 UBSan target만 prerequisite로 두고 ASan은 `sanitize-linux`에서 별도 실행하라는 메시지를 출력합니다. `check: test release-check sanitize` 후 `git diff --check`; Linux ASan은 aggregate `check`에 포함되지 않습니다.
- 학습자 기록 — 필요한 최소 코드 발췌:
```make
# 1b474fa2a5e3, Makefile
$(SANITIZER_CC) $(CFLAGS) $(CPPFLAGS) $(SANITIZER_FLAGS) \
    tests/test_ft_printf.c $(SRC) -o $(SANITIZER_TEST_BIN)
$(SANITIZER_CC) $(CFLAGS) $(CPPFLAGS) $(SANITIZER_FLAGS) \
    -DFT_PRINTF_TEST_WRITE tests/test_output_faults.c $(SRC) \
    -o $(SANITIZER_FAULT_BIN)
```

### Commit 설명 완성란
- 학습자가 해당 SHA 코드와 필요한 비교 SHA를 읽은 뒤, 이 commit을 자신의 말로 설명합니다.
- 최종 설명:
  - normal과 fault paths의 implementation source를 직접 instrument하는 verification infrastructure입니다. `check`는 UBSan까지만 자동 포함하며 pinned Linux ASan/UBSan은 별도 target입니다. 실행된 test paths의 진단력을 높이지만 unexecuted paths나 explicit leak-proof contract를 뜻하지는 않습니다.

## 6. Invariant ledger

Source가 확정한 변화 축을 아래에 배치했습니다. “실제 코드 근거”는 학습자가 해당 SHA를 읽고 채웁니다.

| Invariant / concern | 도입 또는 초기 상태 | 강화 / 수정 | 고정한 검증 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| behavioral oracle | `1b8049e411bb`에서 stdout capture + `snprintf` differential + fixed expectation 기반 생성 | `12d715eba77d`에서 deliberate project contracts를 더 명확히 분리 | 학습자가 어떤 case를 libc와 비교하고 어떤 case는 fixed expectation인지 표로 기록 | `EXPECT_PRINTF` matrices는 `snprintf`; `EXPECT_OUTPUT`/`EXPECT_FORMAT_ERROR`는 null format/string/pointer와 formatted-percent exact bytes를 repository가 직접 정의 |
| failure state machine evidence | `1223518652bd`에서 scripted writer + real broken pipe | partial/EINTR/zero/EPIPE/SIGPIPE/chunking 확인 | production write seam과 signal boundary를 함께 추적 | `ft_printf_test_write` actions/statistics, retry/failure/padding assertions; normal suite의 caller-installed SIGPIPE handler case |
| artifact boundary | `a87bcf560789`에서 archive members/global symbols/dependency allowlist/out-of-tree consumer 검증 | source tree 밖 소비 가능성까지 release contract로 확장 | 학습자가 manifest와 consumer build input을 실제 script에서 기록 | `ar t` 9-member manifest, 17 defined globals, Linux/Darwin external set, temp header/archive/consumer only compile and output/cleanup checks |
| runtime instrumentation | `1b474fa2a5e3`에서 UBSan + pinned Linux GCC ASan/UBSan 추가 | normal + fault binaries의 implementation source까지 instrument | 실행 환경과 실제 수행 결과는 학습자가 기록 | sanitizer commands가 `$(SRC)`를 두 suite에 직접 compile; `sanitize-linux` Docker path; 이번 환경에서는 실행하지 않음 |

### 학습자 추가 기록

- source가 명시한 invariant 범위 안에서만 필요한 행을 추가합니다. 새 invariant를 확정 사실처럼 만들지 않습니다.
- 추가 기록:
  - 추가 행은 만들지 않았습니다. 각 layer의 미보장 범위는 commit별 test 기록에 명시했습니다.

## 7. Failure → Fix → Test 연결

| 기존 failure / risk | Fix / change | 수정 decision | Test / 학습 확인 |
| --- | --- | --- | --- |
| ordinary output comparison만으로는 retry/state transition을 증명할 수 없음 | `1223518652bd` | compile-time write seam + scripted returns + call recording | partial/EINTR/zero/EPIPE 및 64-byte padding assertions |
| libc behavior가 project extension의 portable oracle가 아닐 수 있음 | `12d715eba77d` | differential cases와 project-specific fixed expectations 분리 | null format/string/pointer/percent extension 등 실제 fixed cases를 test에서 분류 |
| in-tree test 성공만으로 archive/package boundary를 증명할 수 없음 | `a87bcf560789` | artifact manifests + dependency check + isolated consumer | consumer가 public header와 archive만 사용해 build/run하는지 확인 |
| value-based tests만으로 UB/address fault를 모두 관찰할 수 없음 | `1b474fa2a5e3` | implementation과 fault suite를 sanitizer build로 별도 compile | UBSan과 Linux ASan/UBSan의 실제 coverage와 미포함 leak 범위를 기록 |

- 학습자 기록 — 실제 failure branch와 regression assertion을 연결한 추가 설명:
  - write-loop risks는 scripted return이 production seam을 통과한 뒤 exact call/output statistics로 판정합니다. oracle risk는 matrix/fixed macro 분리로 해결합니다. packaging risk는 source가 아닌 built archive와 isolated directory를 대상으로 합니다. sanitizer는 두 suite에 source를 직접 instrument하지만 실제 실행된 inputs에 한해 진단하며, repository는 별도 leak-sanitizer target이나 exhaustive path proof를 선언하지 않습니다.

## 8. Ownership / state / responsibility 변화

| 시점 | Source상 owner / boundary | Source상 responsibility 변화 | 해당 SHA 코드 근거 |
| --- | --- | --- | --- |
| functional suite | visible output/public return contract | byte sequence, captured count, return value 및 supported libc-comparable semantics | `tests/test_ft_printf.c` pipe fixture, `EXPECT_PRINTF`, `EXPECT_OUTPUT`, grouped runners |
| fault suite | output system-call boundary | scripted `write` result와 call sequence를 소유하여 state transition 관찰 | `tests/test_output_faults.c` global script/statistics와 `ft_printf_test_write`; Makefile seam build |
| fixed project cases | project public semantics | portable libc behavior가 아닌 explicit expectation을 test가 고정 | `run_public_contract_boundary_cases`의 null/pointer/string/percent `EXPECT_OUTPUT` calls |
| release script | distributable archive boundary | members, symbols, unresolved dependencies, public-header-only consumer를 확인 | `tests/check_release.sh` manifests, `nm` normalization, temp copies/compiler/run/cleanup |
| sanitizer build | runtime language/address diagnostics | normal/fault path의 implementation source를 instrumentation 대상으로 포함 | Makefile sanitizer commands가 archive 대신 `$(SRC)`를 각각 compile하고 halt-on-error로 실행 |

## 9. Thread 최종 상태

- Source가 확정한 도달점: verification이 visible formatting에서 failure state machine, project-specific semantics, archive consumer boundary, sanitizer runtime까지 계층별로 확장된 상태입니다.
- 학습자 기록 — 마지막 commit 기준 실제 코드에서 확인한 최종 state:
  - normal suite는 bytes/count/return, fault suite는 scripted syscall transitions와 real SIGPIPE ownership, fixed cases는 repository-defined semantics, release script는 static archive/consumer packaging, sanitizer targets는 normal/fault implementation runtime diagnostics를 각각 담당합니다. 어느 한 layer도 나머지 layer의 증거를 대신하지 않습니다.
- 학습자 기록 — 이 Thread 밖에서만 해결되는 남은 문제를 source 범위 안에서 구분:
  - 실제 모든 target의 CI matrix/실행 이력, unsupported platforms, exhaustive input space, transactional device output, explicit leak proof는 scaffold와 inspected commits가 보장하지 않습니다. 이번 작업 환경에서도 commands를 실행하지 않았으므로 code-defined verification과 runtime evidence를 구분했습니다.

## 10. 최종 architecture 또는 execution flow 정리

실제 SHA 코드를 읽은 뒤 아래 흐름을 완성합니다. source 설명만 복사하지 말고 함수/상태/branch를 연결합니다.

```text
[functional suite: stdout capture]
    -> [snprintf differential 또는 fixed expected bytes]
    -> [fault suite: substituted write sequence / real SIGPIPE]
    -> [release-check: archive manifests -> isolated consumer]
    -> [sanitizer builds: normal/fault sources 직접 instrument 및 실행]
```

- 각 단계에 대응하는 SHA / file / function:
  - `1b8049e411bb` `tests/test_ft_printf.c`; `1223518652bd` `tests/test_output_faults.c`와 normal signal case; `12d715eba77d` public contract runners; `a87bcf560789` `tests/check_release.sh`/consumer; `1b474fa2a5e3` Makefile sanitizer targets입니다.
- 핵심 state transition:
  - verification 대상이 visible result→system-call progress/error→oracle ownership→archive composition/link boundary→instrumented runtime으로 확장됩니다. 각 layer는 별도 fixture와 failure condition을 갖습니다.
- failure가 끊기는 지점:
  - functional/fault tests는 assertion에서 process exit, release script는 manifest/dependency/build/output/cleanup mismatch에서 exit 1, sanitizer targets는 diagnostics와 `halt_on_error=1`에서 command failure가 됩니다.
- 후속 fix/test가 보장한 지점:
  - deterministic seam이 retry invariants를, fixed cases가 project semantics를, release script가 배포 단위를, sanitizer source builds가 실행 경로의 UB/address diagnostics를 보호합니다. 실제 pass 여부는 command 실행이 있어야만 별도 runtime evidence가 됩니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 정확한 시점의 코드로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 source와 그대로 유지했습니다.
- [x] final HEAD의 코드를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] 필요한 parent/직전 관련 SHA 비교를 실제 diff로 수행했습니다.
- [x] source가 확정한 사실과 내가 코드에서 확인한 사실을 구분했습니다.
- [x] fix의 기존 가정 → failure/risk → root cause → decision → code → test 연결을 필요한 곳에서 완성했습니다.
- [x] test commit의 target invariant, technique, production path, proves/not-proves를 구분했습니다.
- [x] Invariant ledger에 실제 코드 근거를 채웠습니다.
- [x] 이 Thread의 최종 architecture/execution flow를 commit history 순서로 설명할 수 있습니다.
===== END FILE: 06-runtime-artifact-verification.md =====

===== BEGIN FILE: README.md =====
# ft_printf Development Thread 학습 골격

## 목적

이 디렉터리는 `commit-importance.md`와 `commit-bodies.md`에서 확정된 Development Thread를 따라 실제 commit history와 해당 SHA의 코드를 직접 읽으며 프로젝트의 설계, 구현, failure handling, 수정, 검증 과정을 복원하기 위한 학습 골격입니다.

완성형 해설서가 아닙니다. 각 Thread 문서는 source에서 이미 확정된 제목, significance, SHA, subject, importance, tags, commit 관계와 역할만 미리 제공합니다. 실제 함수 동작, 변경 전후 코드, ownership/lifetime, failure path, test 결과, 최종 설명은 학습자가 repository의 정확한 SHA를 확인한 뒤 채웁니다.

## 권장 학습 순서

1. [`01-output-state-system-call-boundary.md`](01-output-state-system-call-boundary.md)
2. [`02-format-fields-typed-dispatch.md`](02-format-fields-typed-dispatch.md)
3. [`03-shared-numeric-layout.md`](03-shared-numeric-layout.md)
4. [`04-string-precision-bounded-access.md`](04-string-precision-bounded-access.md)
5. [`05-whole-call-preflight.md`](05-whole-call-preflight.md)
6. [`06-runtime-artifact-verification.md`](06-runtime-artifact-verification.md)

이 순서는 `commit-importance.md`의 Development Threads 순서를 그대로 따릅니다. 동일 commit이 여러 Thread에 등장하는 경우 의도적인 중복입니다. 각 Thread에서 서로 다른 학습 관점으로 다시 확인합니다.

## Thread 문서 사용법

- 먼저 Thread 목표, 핵심 질문, 완료 기준을 읽습니다.
- Commit map의 순서를 바꾸지 않고 각 SHA를 차례로 checkout 또는 `git show`로 확인합니다.
- 각 commit의 “해당 SHA에서 확인할 코드” 항목에 따라 실제 파일 경로, 함수, 구조체, branch, caller/callee를 직접 찾아 기록합니다.
- fix commit은 직전 관련 상태와 실제 수정 diff를 함께 확인합니다.
- test commit은 production invariant와 failure/boundary, test technique, 통과하는 production path를 분리해서 기록합니다.
- Thread가 끝나면 Invariant ledger와 Failure → Fix → Test 연결을 실제 코드 근거로 완성합니다.
- 마지막에 Thread 최종 상태와 architecture/execution flow를 자신의 설명으로 정리합니다.

## 해당 SHA 코드 확인 원칙

- 반드시 문서에 적힌 정확한 SHA 시점의 코드를 확인합니다.
- 현재 checkout이 다른 commit이라면 그 상태의 코드로 대신 판단하지 않습니다.
- 변경 전 상태가 필요하면 해당 commit의 parent 또는 문서가 지목한 직전 관련 SHA를 확인합니다.
- source 문서가 실제 파일명이나 함수명을 확정하지 않은 경우, 이 골격이 이름을 추측하지 않습니다. 학습자가 해당 SHA의 diff에서 정확한 이름을 찾아 기록합니다.
- 코드 발췌는 학습 근거가 되는 최소 범위만 넣고, 발췌한 SHA와 경로를 함께 기록합니다.

## final HEAD 소급 사용 금지

final HEAD의 구현을 과거 commit의 정답처럼 소급해서 사용하지 않습니다. 후속 refactor/fix에서 함수 경계, state representation, failure behavior가 바뀔 수 있으므로 각 학습 기록은 반드시 해당 SHA 시점의 코드에 근거해야 합니다.

## Importance별 학습 깊이

- `S`: project-defining architecture/invariant입니다. 직전 상태, problem, 기존 설계의 한계, failure possibility, 핵심 decision, 실제 핵심 코드, ownership/lifecycle/state transition, 후속 fix/test까지 깊게 추적합니다.
- `A`: 주요 subsystem, responsibility boundary, failure path, integration point입니다. 핵심 코드와 설계 판단, edge case, 보장/미보장 범위를 확인합니다.
- `B`: Thread 흐름에서 맡는 구현 역할과 필요한 코드/state 변화까지 확인합니다. S/A와 같은 분석란을 기계적으로 반복하지 않습니다.
- `C`: Thread 이해에 필요한 맥락만 확인합니다. 이 source의 Development Threads에는 C-level commit이 포함되어 있지 않지만, 학습 원칙은 동일합니다.

## 실제 코드 삽입 기준

- source에 이미 확정된 설명을 다시 장문의 해설로 복사하지 않습니다.
- 학습자가 직접 확인한 핵심 함수/구조체/조건식/state mutation/test seam만 최소한으로 발췌합니다.
- fix는 가능하면 parent 또는 직전 관련 SHA의 대응 코드와 수정 SHA의 코드를 함께 기록합니다.
- ownership/lifecycle은 선언만 보지 말고 생성/초기화, 전달, mutation, cleanup/종료 지점을 함께 확인합니다.
- failure path는 정상 경로와 별도로 branch 조건, state mutation, 이후 호출 억제, public consequence를 추적합니다.
- test code는 fixture 전체를 복사하기보다 failure injection 지점과 핵심 assertion을 중심으로 기록합니다.

## Test commit 학습 방법

각 test commit에서는 다음을 반드시 구분합니다.

- 어떤 production invariant를 대상으로 하는가
- 어떤 failure 또는 boundary를 재현하는가
- 어떤 test technique을 사용하는가
- 실제 어떤 production code path를 통과하는가
- 이 test가 증명하는 것은 무엇인가
- 이 test가 증명하지 않는 것은 무엇인가
- broad integration인지 deterministic regression인지
- 후속 변경에서 어떤 회귀를 막는가

테스트를 직접 실행한 경우 command, compiler/runtime environment, 실제 result를 추가합니다. 실행하지 않았다면 source description을 실행 결과처럼 작성하지 않습니다.

## 문서 완료 기준

- 모든 Development Thread 문서를 source 순서대로 완료했습니다.
- 각 Thread의 모든 commit을 문서에 적힌 SHA에서 확인했습니다.
- SHA, subject, importance, tags를 source와 다르게 바꾸지 않았습니다.
- S/A/B/C에 따른 학습 깊이를 구분했습니다.
- 각 중요한 commit에서 실제 파일/함수/상태/branch를 직접 찾아 기록했습니다.
- fix마다 가능한 범위에서 기존 가정 → failure/risk → root cause → 수정 decision → 실제 코드 → regression test 연결을 완성했습니다.
- test commit마다 production invariant와 test technique, production path, proved/not-proved 범위를 구분했습니다.
- Invariant ledger와 Failure → Fix → Test 표를 실제 코드 근거로 완성했습니다.
- Thread 마지막에서 final architecture/execution flow를 commit history에 근거해 설명할 수 있습니다.
===== END FILE: README.md =====

