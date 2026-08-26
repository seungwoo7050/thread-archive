# get_next_line Development Threads

이 문서 세트는 `c/get_next_line` 브랜치에서 줄 단위 parser가 확립되고, descriptor별 상태·명시적 context·POSIX 복구 의미론으로 확장되는 개발 문제를 네 개의 독립적인 Thread로 설명합니다. 문서 번호는 읽기 편의를 위한 식별자이며, 모든 Thread가 하나의 선형 순서를 이룬다는 뜻은 아닙니다.

## 문서 구성

| 번호 | 문서 | 책임 |
| ---: | --- | --- |
| 01 | [`01-whole-stream-to-bounded-line-parser.md`](01-whole-stream-to-bounded-line-parser.md) | `read` 분할과 무관하게 한 줄을 반환하고 unread suffix를 보존하는 buffer/cursor 규칙 |
| 02 | [`02-singleton-to-descriptor-scoped-state.md`](02-singleton-to-descriptor-scoped-state.md) | descriptor마다 hidden reader state와 cleanup을 격리하는 규칙 |
| 03 | [`03-explicit-reader-lifetime-and-authoritative-engine.md`](03-explicit-reader-lifetime-and-authoritative-engine.md) | reader 수명과 결과 상태를 caller에게 드러내고 parser engine을 하나로 유지하는 규칙 |
| 04 | [`04-posix-transient-read-and-recovery.md`](04-posix-transient-read-and-recovery.md) | `EINTR`, `EAGAIN`, 일반 I/O 오류를 byte-preserving state transition으로 구분하는 규칙 |

## 문서 간 경계는 어떻게 읽어야 하는가

같은 commit이 둘 이상의 Thread에 등장할 수 있습니다. 예를 들어 `fd03a831686b`는 descriptor별 failure isolation을 설명할 때와 POSIX read fault harness의 기준점을 설명할 때 서로 다른 역할을 갖습니다. 각 문서는 자기 불변 조건과 직접 관련된 diff만 인용하며, 같은 commit에 섞인 다른 관심사는 해당 Thread의 경계에서 분리합니다.

각 Thread의 마지막 `검토 범위:` 줄에는 실제로 확인한 exact SHA와 실행하지 않은 항목을 명시했습니다. 따라서 소스에서 확인한 사실과 실행 결과를 혼동하지 않습니다.
