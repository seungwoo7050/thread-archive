# libft Development Threads — 재작성본

> Branch: `c/libft`  
> 대상 폴더: `development-thread-workbook/completed`

이 폴더는 `libft`의 commit history를 기능 목록이 아니라 **개발 문제 단위**로 재구성한 문서 세트다. 각 Thread는 exact SHA의 diff와 그 시점 source를 기준으로 작성했으며, final HEAD의 구현을 과거 commit에 소급하지 않았다.

## 문서

1. [`01-non-overlap-copy-and-overlap-safe-movement.md`](01-non-overlap-copy-and-overlap-safe-movement.md)  
   `ft_memcpy`의 비중첩 전제와 `ft_memmove`의 방향 선택을 분리한다.

2. [`02-single-allocation-to-rollback-safe-ownership.md`](02-single-allocation-to-rollback-safe-ownership.md)  
   단일 allocation에서 `ft_split`과 `ft_lstmap`의 complete-or-rollback ownership으로 확장되는 과정을 다룬다.

3. [`03-fd-output-partial-system-calls.md`](03-fd-output-partial-system-calls.md)  
   one-shot `write`를 short write, `EINTR`, zero progress, permanent error에 대응하는 progress loop로 교정한다.

4. [`04-static-archive-release-verification.md`](04-static-archive-release-verification.md)  
   build 성공을 넘어 archive member·symbol·dependency·외부 consumer·sanitizer·compiler·leak evidence를 구성한다.

## 작성 원칙

- Thread/commit/SHA/title/importance/tags는 기존 metadata를 출발점으로 사용했다.
- 설명과 코드 발췌는 각 exact SHA의 실제 diff/source를 근거로 했다.
- 같은 commit에 섞인 Thread 밖 변경은 제외했다.
- importance와 tags는 기존 체계를 유지했다.
- AFTER 예시처럼 작은 commit은 짧게, invariant나 failure path를 바꾸는 commit은 필요한 만큼 깊게 설명했다.
- 테스트는 “무엇을 증명하는가”와 “무엇을 증명하지 않는가”를 구분했다.

## repository 대조로 바로잡은 범위

- allocation failure harness의 `ft_lstmap` case는 callback 내부의 `malloc`까지 실패 주입하지 않는다. production `ft_lstnew`의 node allocation을 sweep하고 destructor 호출과 tracked node cleanup을 확인한다.
- `b90fd748255a`의 `make check`는 `sanitize: ubsan`을 호출한다. 별도 `asan` target은 존재하지만 `check` sequence에는 포함되지 않는다.
- 문서에 적힌 test/build 결과는 실행 결과가 아니라 exact source와 script가 의도·구성한 검증 범위다.

## 검토 상태

GitHub의 `c/libft` branch에서 표시된 commit diff와 필요한 exact-SHA source를 검사했다. 현재 실행 환경에서는 branch checkout과 binary build가 불가능해 test suite는 실행하지 않았으며, 실행 성공을 주장하지 않는다.
