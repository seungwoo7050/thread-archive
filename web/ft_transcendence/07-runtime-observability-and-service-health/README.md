# Runtime observability and service health — Development Thread 재작성본

대상은 `web/ft_transcendence` 브랜치의 `development-thread-workbook/07-runtime-observability-and-service-health/completed`에 있던 6개 Development Thread다.

이 재작성본은 기존 문서에서 Thread 구분, commit SHA·제목·importance·tags를 가져오되, 본문은 각 SHA의 실제 diff를 다시 확인해 구성했다. 시간순 변경 목록이 아니라 하나의 개발 문제를 해결하는 단위로 설명하며, 같은 commit에 섞인 변경 중 Thread와 무관한 부분은 제외했다.

## 문서

1. [기동 상태와 트래픽 수용 계약](./01-startup-liveness-readiness-and-storage-state.md)
2. [관측 경계와 metric cardinality](./02-metrics-observer-boundaries-and-cardinality.md)
3. [런타임 limiter primitive와 유계 작업](./03-runtime-limiter-primitives-and-bounded-work.md)
4. [GameHub 통합, shared scheduling, congestion](./04-gamehub-runtime-integration-shared-scheduling-and-congestion.md)
5. [draining readiness와 graceful shutdown](./05-draining-readiness-and-graceful-shutdown.md)
6. [load·fault recovery와 pool error containment](./06-load-fault-recovery-and-pool-error-containment.md)

## 검증 범위

- 조사한 branch: `web/ft_transcendence`만 사용했다.
- 선택된 57개 commit은 각 exact SHA의 GitHub diff를 확인했다.
- 후대 HEAD 구현을 과거 commit 설명에 소급하지 않았다.
- 이 환경에서는 repository checkout, build, unit/integration/load/fault 실행을 수행하지 않았다. 따라서 문서의 테스트 설명은 **test source가 무엇을 검증하도록 작성됐는지**를 뜻하며, 실제 실행 통과를 주장하지 않는다.

## 기존 본문에서 바로잡은 주요 과장

- `85ac2a949439`의 환경 테스트는 명시값과 로컬 기본값 두 경우만 검사한다. 잘못된 port·URL·mode를 거부하는 negative test는 이 commit에 없다.
- `5cac4843fd9b`의 startup seed 금지 검증은 `index.ts`에 `.ensureSeedData(` 문자열이 없는지 확인하는 정적 source regression이다. 실제 프로세스를 띄워 DB mutation 부재를 관찰하는 테스트는 아니다.
- scheduler benchmark와 k6/fault 관련 commit들은 실행 경계·threshold·report 형식을 제공하지만, commit 자체에 실제 부하 측정 결과나 fault run 결과를 저장하지 않는다.
- 초기 `LatestSnapshotBuffer`는 send callback이 끝나지 않은 상태까지 congestion으로 취급했지만, `d90f17fa765d`가 이를 실제 `bufferedAmount` 기반 congestion과 분리한다.
