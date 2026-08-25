# ft_printf Development Threads

이 디렉터리는 `c/ft_printf` 브랜치의 개발 이력을 여섯 개의 개발 문제로 나누어 복원한 문서입니다. 커밋을 단순히 시간순으로 요약하지 않고, 하나의 기능·실패·수정·검증이 어떤 불변 조건을 만들었는지를 중심으로 설명합니다.

## 문서 범위

| 문서 | 개발 문제 |
| --- | --- |
| [`01-output-state-system-call-boundary.md`](01-output-state-system-call-boundary.md) | 출력 길이와 오류를 한 상태로 모으고, POSIX `write`의 부분 진행·중단·영구 실패를 처리하는 문제 |
| [`02-format-fields-typed-dispatch.md`](02-format-fields-typed-dispatch.md) | raw 포맷 문자열을 `t_format`으로 정규화하고, 변환별 `va_arg` 타입 선택을 한곳에 모으는 문제 |
| [`03-shared-numeric-layout.md`](03-shared-numeric-layout.md) | 10진수·16진수·포인터에 중복된 접두사·정밀도·너비 배치를 하나의 규칙으로 합치는 문제 |
| [`04-string-precision-bounded-access.md`](04-string-precision-bounded-access.md) | `%s` 정밀도를 출력 길이 제한뿐 아니라 읽기 범위 제한으로 만드는 문제 |
| [`05-whole-call-preflight.md`](05-whole-call-preflight.md) | 뒤쪽의 잘못된 필드와 전체 길이 초과를 첫 출력 전에 거부하는 문제 |
| [`06-runtime-artifact-verification.md`](06-runtime-artifact-verification.md) | 바이트 비교, 시스템 호출 실패, 프로젝트 고유 규칙, 배포 아카이브, sanitizer를 서로 다른 검증 계층으로 다루는 문제 |

이 순서는 읽기 편의를 위한 것입니다. 각 Thread는 서로 다른 개발 문제를 설명하며, 반드시 앞 문서의 결론이 다음 문서의 시작이라는 뜻은 아닙니다. 같은 커밋이 여러 문서에 등장하는 경우도 의도적입니다. 예를 들어 `c5f627099ad9`는 parser 관점에서는 flag 정규화이고, 숫자 배치 관점에서는 출력 함수가 처리해야 할 상태 조합을 줄이는 변경입니다.

## 근거 사용 원칙

- 각 설명은 문서에 표시된 정확한 SHA의 diff와 그 시점의 source를 기준으로 작성했습니다.
- 같은 커밋에 섞인 변경이라도 해당 Thread와 관계없는 부분은 제외했습니다.
- 후속 커밋이나 최종 HEAD에서 생긴 함수·테스트·보장을 과거 SHA의 동작으로 소급하지 않았습니다.
- 기존 문서의 Thread 구분, SHA, 제목, importance, tags는 유지했습니다. `04` 문서에서 `8e1cee3ed7f0`의 중요도가 목록에서는 `B`, 기존 본문에서는 `A`였던 불일치는 목록의 원래 metadata에 맞춰 `B`로 통일했습니다.

## 확인 상태

GitHub에서 `c/ft_printf` 브랜치에 포함된 exact SHA의 commit diff와 필요한 source/test 파일을 검사했습니다. 이 작업 환경에서는 브랜치를 로컬 checkout하여 build·test·sanitizer를 실행하지 못했으므로, 문서에서 테스트 결과를 새로 실행한 사실처럼 서술하지 않습니다. 테스트 커밋은 코드가 구성한 입력, 통과하는 production path, assertion, 그리고 그 assertion이 증명하지 못하는 범위를 구분해 설명합니다.
