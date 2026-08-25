# get_next_line Development Thread 재작성본

이 문서 세트는 `c/get_next_line` 브랜치의 `development-thread-workbook/completed`에 있던 네 Development Thread를 실제 커밋 diff와 해당 SHA의 소스 코드에 맞춰 다시 작성한 결과입니다.

기존 문서의 Thread 구분, 포함 커밋, SHA, 제목, importance, tags, 역할은 메타데이터로 유지했습니다. 반면 반복되는 학습용 빈칸, 자가 점검표, 같은 내용을 되풀이하는 ledger는 제거하고, 실제 개발 문제와 코드 변화가 드러나는 부분만 남겼습니다.

## 문서 구성

1. [`01-whole-stream-to-bounded-line-parser.md`](01-whole-stream-to-bounded-line-parser.md)  
   EOF까지 전체 입력을 모으던 구현이 unread window와 scan cursor를 갖춘 줄 단위 parser로 바뀌고, direct-tail read와 작업량 계측으로 성능 특성이 고정되는 과정입니다.

2. [`02-singleton-to-descriptor-scoped-state.md`](02-singleton-to-descriptor-scoped-state.md)  
   하나뿐이던 hidden reader 상태가 descriptor별 linked node로 분리되고, 한 descriptor의 오류가 다른 descriptor의 unread bytes를 지우지 않도록 만드는 과정입니다.

3. [`03-explicit-reader-lifetime-and-authoritative-engine.md`](03-explicit-reader-lifetime-and-authoritative-engine.md)  
   hidden lifetime을 caller가 직접 관리하는 opaque context로 바꾸고, `get_next_line`과 explicit API가 하나의 authoritative engine을 공유하게 되는 과정입니다.

4. [`04-posix-transient-read-and-recovery.md`](04-posix-transient-read-and-recovery.md)  
   short read, `EINTR`, `EAGAIN`/`EWOULDBLOCK`, 일반 I/O 오류, EOF를 서로 다른 상태 전이로 처리하면서 이미 읽은 bytes를 보존하는 과정입니다.

## 읽는 방법

네 Thread는 프로젝트의 서로 다른 문제 축입니다. 문서 번호가 곧 강제적인 개발 선후 관계를 뜻하지 않으며, 같은 커밋이 둘 이상의 Thread에 등장할 수 있습니다. 예를 들어 `fd03a831686b`는 descriptor별 실패 격리를 설명할 때와 POSIX read failure harness를 설명할 때 각각 다른 역할을 합니다.

각 문서는 다음 원칙으로 작성했습니다.

- 설명은 표시된 exact SHA 또는 그 parent의 diff와 source를 근거로 합니다.
- final HEAD의 필드나 helper를 과거 커밋에 소급하지 않습니다.
- 같은 커밋에 섞여 있어도 해당 Thread와 무관한 변경은 제외합니다.
- 작은 보조 커밋은 필요한 만큼만 설명하고, 핵심 architecture·failure fix는 상태 변화와 보장 범위까지 깊게 다룹니다.
- 테스트는 “무엇을 증명하는가”와 “무엇까지는 증명하지 않는가”를 구분합니다.

## 검증 범위

GitHub에서 대상 브랜치의 exact SHA diff와 해당 SHA의 파일 내용을 확인했습니다. 현재 실행 환경에서는 저장소를 checkout하여 빌드하거나 test target을 실행하지 못했으므로, 문서에는 실행 성공을 주장하지 않습니다. 테스트 설명은 커밋에 포함된 harness, fixture, assertion과 production call path를 근거로 합니다.
