# ft_printf Development Threads

이 디렉터리는 `42-archive` 저장소의 `c/ft_printf` 브랜치 이력을 여섯 개의 독립적인 개발 문제로 재구성한 문서 세트입니다. 각 문서는 커밋을 시간순 요약으로 소비하지 않고, 하나의 책임·실패·수정·검증이 어떤 계약을 확립했는지 설명합니다.

## 문서 목록

| 번호 | 문서 | 다루는 책임 |
| ---: | --- | --- |
| 01 | [`01-output-state-system-call-boundary.md`](01-output-state-system-call-boundary.md) | Descriptor, count, sticky error를 한 writer가 소유하고 partial write·`EINTR`·영구 실패를 처리하는 책임 |
| 02 | [`02-format-fields-typed-dispatch.md`](02-format-fields-typed-dispatch.md) | Raw field를 `t_format`으로 정규화하고 specifier별 `va_arg` type을 한 경계에서 소비하는 책임 |
| 03 | [`03-shared-numeric-layout.md`](03-shared-numeric-layout.md) | Decimal·hex·pointer의 prefix·precision·width·alignment를 하나의 layout 규칙으로 합성하는 책임 |
| 04 | [`04-string-precision-bounded-access.md`](04-string-precision-bounded-access.md) | `%s` precision을 emitted length뿐 아니라 caller object의 readable bound로 적용하는 책임 |
| 05 | [`05-whole-call-preflight.md`](05-whole-call-preflight.md) | 지원 문법과 총 결과 길이를 첫 `write` 전에 판정하는 책임 |
| 06 | [`06-runtime-artifact-verification.md`](06-runtime-artifact-verification.md) | Public bytes, syscall fault, fixed contract, archive shape, sanitizer runtime을 서로 다른 검증 경계로 다루는 책임 |

번호는 읽기 편의를 위한 배열이며 Thread 사이의 선형 선후관계를 뜻하지 않습니다. 같은 커밋이 여러 문서에 등장할 수 있습니다. 예를 들어 `c5f627099ad9`는 parser 관점에서는 flag normalization이고, numeric layout 관점에서는 downstream state 조합을 줄이는 변경입니다.

## 문서 작성 기준

- 각 commit section의 SHA, 제목, 중요도, 태그는 기존 Thread 정의 metadata를 유지했습니다.
- 설명과 코드 발췌는 표시된 exact SHA의 diff와 그 시점 source를 기준으로 작성했습니다.
- 같은 commit에 섞인 무관한 hunk는 해당 Thread에서 제외했습니다.
- Final HEAD의 후속 함수·test·보장을 과거 commit 설명에 소급하지 않았습니다.
- Diff와 source만으로 확정할 수 없는 실행 결과를 새 사실처럼 주장하지 않았습니다.
- 각 문서는 인접 관심사를 `이 Thread의 경계`에서 분리하고, 마지막 `검토 범위:` 줄에 확인한 source와 미실행 항목을 기록합니다.

## 검토 상태

Exact commit diff와 필요한 source/test/build 파일을 검사했습니다. 현재 환경에서는 branch를 로컬 checkout하여 build, functional test, release check, UBSan, Docker ASan을 실행하지 않았습니다. 따라서 test commit 문서는 test가 구성한 입력·assertion·production path와 증명 범위를 설명하며, 새 실행 성공을 주장하지 않습니다.
