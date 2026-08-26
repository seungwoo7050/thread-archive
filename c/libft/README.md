# libft Development Threads

> Branch: `c/libft` · 대상 폴더: `development-thread-workbook/completed`

이 문서 세트는 `libft`의 commit history를 기능 목록이 아니라 개발 문제와 불변 조건 단위로 재구성한다. 각 Thread는 exact SHA의 diff와 그 시점 source만 사용하며, final HEAD의 구현을 과거 커밋에 소급하지 않는다.

## 문서

1. [`01-non-overlap-copy-and-overlap-safe-movement.md`](01-non-overlap-copy-and-overlap-safe-movement.md)  
   비중첩 복사의 caller precondition과 겹침 안전 이동의 방향 선택을 분리한다.

2. [`02-single-allocation-to-rollback-safe-ownership.md`](02-single-allocation-to-rollback-safe-ownership.md)  
   단일 allocation에서 `ft_split`과 `ft_lstmap`의 complete-or-rollback ownership으로 확장한다.

3. [`03-fd-output-partial-system-calls.md`](03-fd-output-partial-system-calls.md)  
   one-shot `write`를 short write, `EINTR`, zero progress, permanent error에 대응하는 progress loop로 교정한다.

4. [`04-static-archive-release-verification.md`](04-static-archive-release-verification.md)  
   build 성공을 넘어 archive surface, 외부 consumer, sanitizer, compiler, leak evidence를 구성한다.

## 문서 구조

각 Thread는 개요의 서사와 commit map으로 시작하고, exact subject·importance·tags를 보존한 커밋 섹션을 실제 성격에 맞게 전개한다. 일반 변경은 diff를 기본 증거로 사용하고, 상태 전이가 조밀한 로직만 주석이 있는 최종 코드로 설명한다. 마지막에는 인접 관심사를 `이 Thread의 경계`로 분리하고 실제 확인·미실행 범위를 명시한다.

> 전체 검토 범위: 네 Thread의 commit table에 표시된 exact SHA diff와 문서에서 지목한 source·test·build script를 확인했다. repository binary, ordinary/failure test suite, sanitizer, leak checker, compiler matrix는 실행하지 않았다.
