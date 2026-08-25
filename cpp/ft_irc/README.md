# ft_irc Development Threads — 재작성본

대상은 `seungwoo7050/42-archive` 저장소의 `cpp/ft_irc` 브랜치입니다. 이 폴더는 기존 `development-thread-workbook/completed`의 Thread 구분과 commit metadata를 출발점으로 삼되, 각 SHA의 실제 diff와 그 시점의 source를 다시 확인하여 본문을 재작성한 결과입니다.

## 작성 원칙

- Thread는 단순한 시간순 commit 묶음이 아니라 하나의 개발 문제, 불변 조건, 실패와 수정, 검증 범위를 설명하는 단위로 유지했습니다.
- commit subject, SHA, importance, tags는 기존 문서의 연동 metadata를 보존했습니다.
- 설명은 해당 SHA의 diff와 그 시점의 코드에 한정했습니다. 후대 구현을 과거 commit에 소급하지 않았습니다.
- 같은 commit이 여러 Thread에 등장할 때에는 중복을 제거하지 않고, 각 Thread의 문제와 직접 관련된 변경만 다뤘습니다.
- 작은 B급 commit은 역할과 전후 상태만 짧게 설명하고, S/A급 commit은 실패 조건, 원인, 상태 전이, 소유권, 후속 검증까지 깊게 추적했습니다.
- 테스트가 source상 무엇을 겨냥하는지는 기록했지만, 이 작업 환경에서는 브랜치를 checkout하여 빌드·실행하지 못했으므로 실제 통과 결과를 주장하지 않습니다.

## 문서 목록

1. [이식 가능한 준비 이벤트와 논블로킹 전송](01-portable-readiness-and-nonblocking-transport.md)
2. [프로토콜 경계, 식별자와 등록 전이](02-protocol-boundary-identity-and-registration.md)
3. [채널 권한, 팬아웃과 종료 정리](03-channel-authority-fanout-and-cleanup.md)
4. [운영 보호 장치와 제어된 종료](04-operational-protections-and-controlled-shutdown.md)
5. [엄격한 실행 설정값 경계](05-strict-runtime-configuration-boundaries.md)
6. [heartbeat 생존 판정의 정확성](06-heartbeat-liveness-correctness.md)
7. [부분 실패에서의 송신 대기열 정확성](07-output-queue-correctness-under-partial-failure.md)
8. [재진입 가능한 서버와 application 정리](08-reentrant-server-and-application-cleanup.md)
9. [검증 체계의 성숙과 이식성 강제](09-verification-maturation-and-portability-enforcement.md)

## 전체 연결 관계

```text
준비 이벤트 추상화와 Connection 수명
        ↓
IRC line parsing과 등록 상태
        ↓
채널 권한·팬아웃·정리
        ├─ 운영 한도·관측·종료
        ├─ heartbeat 정확성
        └─ 송신 대기열의 부분 실패
                  ↓
callback 재진입과 상태 재조회
                  ↓
단위·계약·공정성·플랫폼 CI 검증
```

이 화살표는 문서의 읽기 순서를 돕기 위한 것이며, 모든 Thread가 실제 개발 이력에서 선형으로 종속된다는 뜻은 아닙니다. 예를 들어 설정값 파싱, heartbeat 교정, 송신 대기열 검증은 공통 runtime 위에서 병렬로 발전한 문제입니다.
