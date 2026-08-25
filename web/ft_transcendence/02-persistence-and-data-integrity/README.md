# 영속성·데이터 무결성

`web/ft_transcendence`의 데이터 저장 계층이 schema·repository·read model·migration·동시성 제약으로 발전한 과정을 exact commit diff로 복원합니다.

이 디렉터리의 문서는 시간순 commit 요약이 아닙니다. 각 문서는 하나의 개발 문제와 그 불변 조건을 다루며, 같은 commit에 섞인 다른 관심사는 해당 Thread에서 제외합니다.

## Thread

1. [저장소 추상화와 조회 모델 — 공통 인터페이스에서 실제 backend 증거까지](01-repository-abstraction-backend-parity-and-read-models.md)
2. [Migration·seed·readiness·reset — schema 변경과 데이터 준비의 수명주기 분리](02-migration-seed-readiness-and-reset-lifecycle.md)
3. [Row mapping과 backend contract 정렬 — 저장 표현과 조회 표현을 분리하기](03-row-mapping-and-backend-contract-alignment.md)
4. [Canonical friendship — 방향이 다른 두 요청을 하나의 관계로 수렴시키기](04-canonical-friendship-and-concurrent-requests.md)
5. [Tournament admission — 마지막 한 자리와 대진 생성을 하나의 transaction으로 묶기](05-tournament-admission-and-capacity-concurrency.md)

## Category 경계

포함하는 문제는 다음과 같습니다.

- PostgreSQL schema와 `AppRepository` lifetime
- memory/PostgreSQL의 공통 surface와 실제 의미 차이
- row type·mapper·aggregate 조립 경계
- migration과 seed의 분리, readiness, startup mutation 금지
- destructive test reset의 대상 guard와 schema 격리
- unordered friendship identity와 역방향 요청 수렴
- tournament capacity·seed·bracket 생성의 transaction 경계

다음 문제는 이 카테고리 밖에 둡니다.

- session/token 보안과 WebSocket ticket 소비
- match result finalization과 rating update의 원자성
- chat room 접근·메시지 무결성
- admin audit 기록의 원자성
- tournament match 진행과 winner propagation
- Docker/Compose/Caddy image, 배포 job, release artifact

## 문서 작성 기준

- Commit map의 SHA·제목·importance·tags는 기존 metadata를 유지했습니다.
- 설명과 code excerpt는 표시된 exact SHA 또는 그 parent diff를 기준으로 작성했습니다.
- final HEAD의 helper·field·test를 과거 commit에 소급하지 않았습니다.
- 같은 commit의 무관한 변경은 해당 Thread에서 제외했습니다.
- 작은 B급 commit은 역할과 실제 변화만 설명하고, A급 failure/constraint commit은 이전 가정·실패 원인·결정·검증 범위를 깊게 추적했습니다.
- Test source가 존재한다는 사실과 실제 실행 결과를 구분했습니다. 이 재작성 과정에서는 repository를 checkout해 suite를 실행하지 않았습니다.

## 감사 과정에서 바로잡은 핵심 사항

- `1140fb868714`은 SQL asset을 읽는 migration loader가 아니라 schema를 TypeScript 문자열로 복제한 초기 단계입니다.
- `compareMigrationSets()`는 적용 순서 prefix나 checksum을 확인하지 않고 expected/applied 이름의 집합 차이만 비교합니다.
- API startup seed 시험은 entry source의 직접 `.ensureSeedData()` 호출을 막는 정규식 guard이며, 모든 간접 mutation을 증명하지 않습니다.
- Friendship 시험은 canonical/idempotent 결과와 DB 단일 row를 검증하지만 두 방향 요청을 실제 동시에 충돌시키지는 않습니다.
- Memory tournament의 `Promise.allSettled` case는 같은 process의 순차 mutation 결과를 검증하고, 실제 DB 경쟁은 PostgreSQL integration case가 담당합니다.
- `cdaca35ccf7f`의 friendship과 tournament assertion은 각 불변 조건에 맞는 두 Thread에서 나누어 설명했습니다.
