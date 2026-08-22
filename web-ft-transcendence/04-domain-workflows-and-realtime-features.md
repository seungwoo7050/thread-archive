===== BEGIN FILE: 01-tournament-contract-schema-and-bracket-construction.md =====
# 토너먼트 계약·스키마와 대진 구성

원문 Development Thread: `Tournament contract, schema, and bracket construction`

## 1. Thread 목표

- 단순 참가자 목록에서 독립적인 tournament-match read model과 영속 상태로 이동하는 과정을 추적합니다.
- 4인 대회의 seed 배치, 준결승 생성, 결승 생성 조건과 PostgreSQL/memory 구현의 동작 일치를 복원합니다.
- 화면에서 추정한 대진표가 persisted match를 소비하는 UI로 교체되는 지점을 확인합니다.

### Source에서 확정된 significance

> Entries alone cannot represent round, slot, room assignment, persisted game linkage, score, winner, or independent match lifecycle. The history introduces those facts as stored tournament-match state and then makes both realtime play and the web bracket consume that state.

### 직접 연결되는 Critical Invariants

> Tournament bracket state is persisted once and read by every consumer instead of being independently reconstructed from entry order.
>
> A four-player bracket is seeded as 1–4 and 2–3; the final exists only after both semifinals have finished with winners.

### 직접 연결되는 Major Engineering Difficulties

> Keeping shared contracts, SQL schema, row mapping, PostgreSQL behavior, memory behavior, and web rendering aligned while the model expands.
>
> Distinguishing participant admission from bracket construction and distinguishing a tournament match from the generic persisted game result linked later.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 repository와 화면은 참가 순서에서 seed와 bracket을 어떻게 추정했으며 어떤 정보를 표현하지 못했습니까?
- `TournamentMatchSummary`, `tournament_matches`, row mapper는 각각 어떤 형태의 데이터를 소유합니까?
- 4번째 참가자가 들어왔을 때 1–4, 2–3 준결승이 어떤 SQL/메서드에서 생성됩니까?
- 준결승 두 경기가 끝나기 전후 결승 row 생성 조건은 PostgreSQL과 memory에서 어떻게 맞춰집니까?
- UI는 언제부터 `entries.slice(...)` 대신 `TournamentSummary.matches`를 신뢰합니까?

## 3. 완료 기준

- entry-only 모델과 persisted tournament-match 모델의 표현 차이를 실제 타입·컬럼·mapper로 설명할 수 있습니다.
- 4인 bracket 생성과 결승 생성의 선행 조건을 PostgreSQL과 memory 구현에서 각각 추적할 수 있습니다.
- tournament-match ID, room ID, generic match ID가 서로 다른 수명과 역할을 갖는 이유를 설명할 수 있습니다.
- 초기 placeholder bracket이 persisted match 기반 화면으로 교체되는 호출 흐름을 그릴 수 있습니다.
- Commit map의 모든 SHA를 지정 브랜치 ancestry와 source classification에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 실행한 명령과 코드 검사만으로 확인한 사실을 구분하고 실행하지 않은 test를 통과했다고 기록하지 않습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `34c80874f13f` | `feat(db): 토너먼트 row contract 정의` | B | PERSISTENCE, TOURNAMENT | Defines typed persistence representations needed to map tournaments into the shared application contract. |
| 2 | `9b1dabcc4bb4` | `feat(db): 토너먼트 참가 저장 구현` | B | PERSISTENCE, TOURNAMENT | Implements tournament creation, listing, and joining through the common repository interface. |
| 3 | `4370ac3162b2` | `feat(web): 토너먼트 대진표 화면 추가` | B | TOURNAMENT, WEB | Adds the first tournament page and connects tournament listing and creation to the HTTP API. |
| 4 | `11e4c3dda1aa` | `feat(tournament): 대진 경기 contract 정의` | B | REALTIME, TOURNAMENT, WEB | Adds a shared tournament-match summary with bracket position, lifecycle, participants, winner, score, room, and persisted-match identifiers. |
| 5 | `138e5b8590b6` | `feat(tournament): 대진 경기 schema 추가` | B | REALTIME, PERSISTENCE, TOURNAMENT | Introduces a dedicated `tournament_matches` persistence model instead of deriving every round from entries or generic game records. |
| 6 | `4021a437e7e0` | `feat(tournament): 대진 row mapper 정의` | B | REALTIME, PERSISTENCE, TOURNAMENT | Adds explicit mapping from database tournament-match rows to application records and public summaries. |
| 7 | `53579ad0f0bf` | `feat(tournament): 대진 경기 lifecycle 저장 구현` | A | REALTIME, PERSISTENCE, TOURNAMENT | Adds tournament-match read/start/complete operations to `AppRepository` and both repository implementations. |
| 8 | `0d6824683677` | `feat(tournament): 준결승 대진 생성과 조회 구현` | A | TOURNAMENT | Creates semifinal bracket rows at four-player capacity and includes persisted matches in tournament summaries. |
| 9 | `b01adf728ca0` | `feat(tournament): memory 대진 진행 구현` | B | PERSISTENCE, TOURNAMENT | Aligns the in-memory tournament flow behaviorally with PostgreSQL. |
| 10 | `b0a1505c6a0f` | `feat(tournament): 플레이 가능한 대진 UI 연결` | B | PROTOCOL, REALTIME, TOURNAMENT | Replaces the placeholder bracket with persisted matches and links eligible participants directly to realtime play. |

## 5. Commit별 학습 기록

### 5.1. `feat(db): 토너먼트 row contract 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `34c80874f13f` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Defines typed persistence representations needed to map tournaments into the shared application contract.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/schema.ts`의 tournament/entry row 타입과 projection
- `packages/db/src/rowMappers.ts`의 tournament 변환 경계
- `packages/shared/src/http.ts`의 `TournamentSummary`와 사용자 projection
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:34c80874f13f -->
- **직전 상태:** repository에는 사용자·세션·경기 등은 있었지만 tournament row를 공유 HTTP 모델로 바꾸는 typed 표현이 없었습니다.
- **구현 결정:** DB row의 snake_case 필드와 application의 camelCase summary를 분리하고, creator와 entries를 명시적으로 projection할 기반을 추가했습니다.
- **상태/소유권 변화:** 이 commit은 lifecycle 동작을 만들지 않고 “DB 형태를 application contract로 변환하는 책임”을 database package에 둡니다.
- **실패/edge:** 참가자 목록이 없거나 creator join projection이 불완전한 경우를 이 commit 자체가 해결하지는 않습니다.
- **보장/비보장:** typed row와 mapping 경계는 보장하지만 생성·참가·capacity·bracket 생성은 아직 없습니다.
- **다음 연결:** `9b1dabcc4bb4`가 이 표현 위에 실제 tournament 생성·목록·참가 저장을 추가합니다.
<!-- LEARNER-ANSWER END commit:34c80874f13f -->

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 Thread 관련 SHA: `9b1dabcc4bb4` — `feat(db): 토너먼트 참가 저장 구현`

### 5.2. `feat(db): 토너먼트 참가 저장 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `9b1dabcc4bb4` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Implements tournament creation, listing, and joining through the common repository interface.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `AppRepository` tournament 메서드
- PostgreSQL `createTournament`, `listTournaments`, `joinTournament` SQL
- memory repository의 tournament 배열과 seed 계산
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:9b1dabcc4bb4 -->
- **직전 상태:** typed row는 있었지만 application이 대회를 만들거나 참가자를 저장할 repository operation은 없었습니다.
- **구현 결정:** 생성 시 tournament row를 넣은 뒤 creator를 참가시키고, 참가 시 현재 entry 수를 읽어 `count + 1`을 seed로 저장합니다. memory 구현도 같은 순서로 summary를 갱신합니다.
- **상태/소유권 변화:** 참가 순서와 seed는 repository가 소유하며, capacity 4에 도달하면 memory 상태를 `running`으로 바꿉니다.
- **실패/edge:** count 조회와 insert가 하나의 원자적 capacity claim은 아니므로 concurrent admission 안전성은 이 commit의 보장이 아닙니다. 중복 사용자는 unique/conflict 경로로 제한됩니다.
- **보장/비보장:** 단일 호출 흐름의 생성·목록·참가와 seed 순서는 제공하지만 round/slot별 match 상태는 아직 없습니다.
- **다음 연결:** `4370ac3162b2`는 entries만으로 화면 대진을 만들며, 그 한계가 이후 persisted match 모델의 필요성을 드러냅니다.
<!-- LEARNER-ANSWER END commit:9b1dabcc4bb4 -->

비교 기준:
- 직전 Thread 관련 SHA: `34c80874f13f` — `feat(db): 토너먼트 row contract 정의`
- 다음 Thread 관련 SHA: `4370ac3162b2` — `feat(web): 토너먼트 대진표 화면 추가`

### 5.3. `feat(web): 토너먼트 대진표 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `4370ac3162b2` |
| Importance | B |
| Tags | TOURNAMENT, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds the first tournament page and connects tournament listing and creation to the HTTP API.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/tournaments/page.tsx`의 초기 state와 API load/create
- 대진표 렌더링에서 `entries.slice(index, index + 2)` 사용 지점
- sample tournament fallback과 선택 상태
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:4370ac3162b2 -->
- **직전 상태:** tournament 저장은 있었지만 사용자가 목록·생성·대진을 보는 route가 없었습니다.
- **구현 결정:** 화면은 API 목록과 생성을 연결하되, 선택 대회의 `entries`를 두 명씩 잘라 round card처럼 렌더링했습니다.
- **상태/소유권 변화:** UI가 실제 match 상태를 소유하지 않으면서도 entry order에서 bracket을 계산하는 임시 read model 역할을 맡았습니다.
- **실패/edge:** round, slot, winner, score, room, generic match link가 없고 1–4/2–3 seed 규칙도 표현하지 못합니다. API 실패 시 sample state가 실제 데이터처럼 남을 수 있습니다.
- **보장/비보장:** tournament 목록/생성 UI는 제공하지만 대진표는 영속 사실이 아니라 화면 추정입니다.
- **다음 연결:** `11e4c3dda1aa`가 이 추정을 제거할 tournament-match contract를 정의합니다.
<!-- LEARNER-ANSWER END commit:4370ac3162b2 -->

비교 기준:
- 직전 Thread 관련 SHA: `9b1dabcc4bb4` — `feat(db): 토너먼트 참가 저장 구현`
- 다음 Thread 관련 SHA: `11e4c3dda1aa` — `feat(tournament): 대진 경기 contract 정의`

### 5.4. `feat(tournament): 대진 경기 contract 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `11e4c3dda1aa` |
| Importance | B |
| Tags | REALTIME, TOURNAMENT, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds a shared tournament-match summary with bracket position, lifecycle, participants, winner, score, room, and persisted-match identifiers.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/http.ts`의 `TournamentMatchSummary`와 tournament summary 확장
- `packages/shared/src/ws.ts`의 `tournament.join` client event
- status, round, slot, participant, winner, score, `roomId`, `matchId` 필드
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:11e4c3dda1aa -->
- **직전 상태:** 화면과 repository는 entries만 공유하므로 독립 경기 lifecycle을 전달할 수 없었습니다.
- **구현 결정:** round/slot과 `pending|ready|running|finished`, 양 참가자, 승자, 점수, 선택적 room/generic match ID를 공유 계약으로 올렸습니다.
- **상태/소유권 변화:** tournament summary 안에 “계산된 pair”가 아니라 독립적인 match summary collection이 들어갈 자리가 생겼고, realtime 참가 명령은 tournament match ID를 사용합니다.
- **실패/edge:** 타입은 허용 상태를 표현할 뿐 DB 제약이나 상태 전이 자체를 강제하지 않습니다.
- **보장/비보장:** API·web·WebSocket이 같은 식별자와 필드를 말하게 하지만 persistence는 다음 commit 전까지 없습니다.
- **다음 연결:** `138e5b8590b6`이 같은 모델을 `tournament_matches` 테이블로 저장합니다.
<!-- LEARNER-ANSWER END commit:11e4c3dda1aa -->

비교 기준:
- 직전 Thread 관련 SHA: `4370ac3162b2` — `feat(web): 토너먼트 대진표 화면 추가`
- 다음 Thread 관련 SHA: `138e5b8590b6` — `feat(tournament): 대진 경기 schema 추가`

### 5.5. `feat(tournament): 대진 경기 schema 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `138e5b8590b6` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Introduces a dedicated `tournament_matches` persistence model instead of deriving every round from entries or generic game records.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/migrations.ts`의 `tournament_matches` DDL
- `packages/db/src/schema.ts`의 tournament-match table/row 타입
- `unique(tournament_id, round, slot)`과 tournament cascade FK
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:138e5b8590b6 -->
- **직전 상태:** shared contract는 있었지만 restart 후에도 round/slot/lifecycle을 보존할 테이블이 없었습니다.
- **구현 결정:** tournament FK, round, slot, nullable left/right/winner/room/generic match IDs, scores, status를 가진 전용 row와 `(tournament_id, round, slot)` unique key를 추가했습니다.
- **상태/소유권 변화:** bracket 위치와 lifecycle의 authority가 entry order나 UI에서 database row로 이동합니다.
- **실패/edge:** nullable participant와 status 조합의 의미는 application logic이 책임지며, 이 DDL만으로 합법 상태 전이를 완전히 제약하지는 않습니다.
- **보장/비보장:** 동일 대회의 같은 round/slot 중복 row를 막고 대회 삭제 시 종속 row를 정리하지만 bracket 생성 조건은 아직 없습니다.
- **다음 연결:** `4021a437e7e0`이 row를 내부 record와 public summary로 변환합니다.
<!-- LEARNER-ANSWER END commit:138e5b8590b6 -->

비교 기준:
- 직전 Thread 관련 SHA: `11e4c3dda1aa` — `feat(tournament): 대진 경기 contract 정의`
- 다음 Thread 관련 SHA: `4021a437e7e0` — `feat(tournament): 대진 row mapper 정의`

### 5.6. `feat(tournament): 대진 row mapper 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `4021a437e7e0` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds explicit mapping from database tournament-match rows to application records and public summaries.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/rowMappers.ts`의 `toTournamentMatchRecord`
- `toTournamentMatchSummary`의 사용자 projection과 nullable 필드 처리
- slot 숫자 변환과 ISO timestamp 변환
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:4021a437e7e0 -->
- **직전 상태:** 테이블 row는 생겼지만 DB 이름·Date·nullable ID를 shared summary로 바꾸는 단일 경계가 없었습니다.
- **구현 결정:** raw row를 내부 record로 먼저 정규화하고, participant/winner `PublicUser` projection을 주입해 외부 summary를 구성합니다.
- **상태/소유권 변화:** SQL 조회와 HTTP 응답 사이의 이름 변환·날짜 직렬화·사용자 결합 책임이 mapper로 모입니다.
- **실패/edge:** mapper는 존재하지 않는 사용자나 잘못된 상태를 복구하지 않으며 caller가 올바른 projection을 제공해야 합니다.
- **보장/비보장:** 동일 row가 PostgreSQL 경로에서 일관된 application 형태가 되지만 lifecycle mutation은 다음 commit이 담당합니다.
- **다음 연결:** `53579ad0f0bf`가 조회·시작·완료 operation을 양 repository에 추가합니다.
<!-- LEARNER-ANSWER END commit:4021a437e7e0 -->

비교 기준:
- 직전 Thread 관련 SHA: `138e5b8590b6` — `feat(tournament): 대진 경기 schema 추가`
- 다음 Thread 관련 SHA: `53579ad0f0bf` — `feat(tournament): 대진 경기 lifecycle 저장 구현`

### 5.7. `feat(tournament): 대진 경기 lifecycle 저장 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `53579ad0f0bf` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds tournament-match read/start/complete operations to `AppRepository` and both repository implementations.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `getTournamentMatch`, `startTournamentMatch`, `completeTournamentMatch`
- PostgreSQL complete SQL과 `ensureFinalMatch` 호출
- memory implementation의 같은 operation 및 당시 parity 차이
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:53579ad0f0bf -->
- **직전 상태:** 전용 row와 mapper는 있었지만 realtime hub가 match를 조회·running 전환·finished 전환할 repository API가 없었습니다.
- **구현 결정:** `AppRepository`에 조회/시작/완료를 추가하고 PostgreSQL은 completion에서 winner·scores·room/generic match link를 저장한 뒤 semifinal이면 결승 생성 검사를, final이면 tournament 완료 갱신을 수행합니다.
- **상태/소유권 변화:** tournament match lifecycle mutation은 repository가 소유하며 GameHub는 구체 SQL을 알지 않습니다. PostgreSQL과 memory가 같은 interface 뒤에 놓입니다.
- **실패/edge:** 당시 memory completion은 PostgreSQL과 완전히 같은 final 생성 동작을 아직 갖지 않았고, 일반 match 저장과 tournament completion도 별도 호출이었습니다.
- **보장/비보장:** 기본 조회·시작·완료 경로는 생기지만 transaction 단위의 generic match/rating/bracket 원자성은 Thread 02의 후속 `finalizeMatch`가 완성합니다.
- **다음 연결:** `0d6824683677`이 4번째 entry 시 semifinal row를 생성하고 summary에 포함합니다.
<!-- LEARNER-ANSWER END commit:53579ad0f0bf -->

비교 기준:
- 직전 Thread 관련 SHA: `4021a437e7e0` — `feat(tournament): 대진 row mapper 정의`
- 다음 Thread 관련 SHA: `0d6824683677` — `feat(tournament): 준결승 대진 생성과 조회 구현`

### 5.8. `feat(tournament): 준결승 대진 생성과 조회 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `0d6824683677` |
| Importance | A |
| Tags | TOURNAMENT |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Creates semifinal bracket rows at four-player capacity and includes persisted matches in tournament summaries.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 PostgreSQL `joinTournament`
- `ensureTournamentBracket`의 seed 1/4 및 2/3 insert
- `tournamentFromRow`의 match load, 사용자 resolution, round/slot 정렬
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:0d6824683677 -->
- **직전 상태:** lifecycle operation은 있었지만 tournament join이 entries만 저장해 실제 첫 round row가 자동 생성되지 않았습니다.
- **구현 결정:** 4명일 때 seed 1–4를 semifinal slot 1, seed 2–3을 slot 2로 insert하고 unique conflict 시 중복 생성을 피합니다. summary 조회는 matches와 participant/winner 사용자를 결합해 round/slot 순으로 반환합니다.
- **상태/소유권 변화:** bracket topology가 web 계산이 아니라 repository가 생성한 row 집합으로 확정됩니다.
- **실패/edge:** 참가 인원 count와 insert의 concurrent capacity 경쟁은 이 commit만으로 안전하지 않습니다. 결승은 두 semifinal 완료 전 생성되지 않습니다.
- **보장/비보장:** 정상적인 4인 순차 참가에서 정확한 seed pair와 persisted summary를 보장하지만 concurrent admission invariant는 category 02에서 보강됩니다.
- **다음 연결:** `b01adf728ca0`이 memory repository에도 동일한 topology와 progression을 구현합니다.
<!-- LEARNER-ANSWER END commit:0d6824683677 -->

비교 기준:
- 직전 Thread 관련 SHA: `53579ad0f0bf` — `feat(tournament): 대진 경기 lifecycle 저장 구현`
- 다음 Thread 관련 SHA: `b01adf728ca0` — `feat(tournament): memory 대진 진행 구현`

### 5.9. `feat(tournament): memory 대진 진행 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `b01adf728ca0` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Aligns the in-memory tournament flow behaviorally with PostgreSQL.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 memory `joinTournament` capacity guard
- `ensureMemoryBracket`의 semifinal 생성
- memory completion의 두 semifinal winner 확인과 final/tournament 완료
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:b01adf728ca0 -->
- **직전 상태:** PostgreSQL은 persisted semifinal/final progression을 가졌지만 memory backend는 entry와 부분 lifecycle만 구현해 테스트·개발 환경이 달랐습니다.
- **구현 결정:** memory join도 capacity를 검사하고 4명 시 1–4/2–3 semifinal을 한 번만 만들며, 두 semifinal이 finished이고 winner가 있을 때 final을 만듭니다. final 완료는 tournament status/winner를 갱신합니다.
- **상태/소유권 변화:** backend 선택과 무관하게 tournament summary와 progression이 같은 repository contract를 따릅니다.
- **실패/edge:** memory는 process 수명 안의 단일 자료구조이므로 PostgreSQL transaction/동시성 특성을 검증하지 않습니다.
- **보장/비보장:** 기능 parity는 높이지만 durable persistence나 row lock 보장은 없습니다.
- **다음 연결:** `b0a1505c6a0f`이 web과 play flow를 이 공통 persisted model에 연결합니다.
<!-- LEARNER-ANSWER END commit:b01adf728ca0 -->

비교 기준:
- 직전 Thread 관련 SHA: `0d6824683677` — `feat(tournament): 준결승 대진 생성과 조회 구현`
- 다음 Thread 관련 SHA: `b0a1505c6a0f` — `feat(tournament): 플레이 가능한 대진 UI 연결`

### 5.10. `feat(tournament): 플레이 가능한 대진 UI 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `b0a1505c6a0f` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Replaces the placeholder bracket with persisted matches and links eligible participants directly to realtime play.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/tournaments/page.tsx`의 `selected.matches` 렌더링
- ready match와 현재 사용자 participant 여부에 따른 play link
- `apps/web/src/app/play/page.tsx`의 `tournamentMatchId` parsing 및 `tournament.join` 전송
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:b0a1505c6a0f -->
- **직전 상태:** UI는 entries를 잘라 대진을 추정했고 play route는 일반 queue/AI만 시작했습니다.
- **구현 결정:** round/slot/status/score/winner를 persisted match summary에서 표시하고, 현재 사용자가 ready match의 참가자일 때만 match ID를 포함한 play link를 제공합니다. play page는 query의 ID로 `tournament.join`을 보냅니다.
- **상태/소유권 변화:** 화면은 bracket을 만들지 않고 repository-backed summary를 표시하며, realtime server가 실제 좌석·room 생성 권한을 유지합니다.
- **실패/edge:** 링크 가시성은 편의 제어일 뿐 보안 경계가 아니며 server가 participant/status를 다시 검증해야 합니다.
- **보장/비보장:** 정상 UI journey는 persisted bracket과 연결되지만 room start rollback과 finalization 원자성은 Thread 02에서 다룹니다.
- **다음 연결:** 다음 Thread의 `33b6dfc5df7a`가 tournament match admission을 GameHub room lifecycle에 실제 통합합니다.
<!-- LEARNER-ANSWER END commit:b0a1505c6a0f -->

비교 기준:
- 직전 Thread 관련 SHA: `b01adf728ca0` — `feat(tournament): memory 대진 진행 구현`
- 이 commit을 Thread의 마지막 상태로 사용합니다.

## 6. Thread 종합

다음 항목을 commit 순서에서 재구성합니다: invariant evolution, Failure → Fix → Test 관계, ownership/state 변화, 최종 실행 흐름, 비보장, 실제 실행 증거.

<!-- LEARNER-ANSWER START thread:01-tournament-contract-schema-and-bracket-construction.md:synthesis -->
- **불변식 진화:** 초기 `TournamentSummary.entries`는 참가 순서만 보존했고, `4370ac3162b2`의 화면은 이를 두 명씩 잘라 대진을 추정했습니다. `11e4c3dda1aa`–`4021a437e7e0`이 round/slot/participant/winner/score/room/match 식별자를 가진 별도 계약·테이블·mapper를 만들었고, `0d6824683677`과 `b01adf728ca0`이 4명 충원 시 1–4, 2–3 준결승과 두 준결승 승자 기반 결승을 각각 PostgreSQL과 memory에 고정했습니다. `b0a1505c6a0f`에서 UI도 이 저장 상태만 렌더링합니다.
- **소유권과 상태:** 참가자 집합은 tournament entry가, 각 경기의 round/slot/lifecycle은 `tournament_matches`가, 실제 종료 경기 기록은 별도 generic match가 소유합니다. `room_id`는 진행 중 realtime 방 연결이고 `match_id`는 종료 후 영속 결과 연결이므로 같은 식별자가 아닙니다.
- **Failure → Fix → Verification:** 이 Thread의 핵심 실패는 데이터 손상보다 read-model fabrication입니다. `entries.slice(...)`와 backend별 불완전한 bracket progression을 persisted matches 및 memory parity가 교정했습니다. 원자적 결과 확정과 동시성 검증은 다음 Thread가 주 소유자입니다.
- **최종 흐름:** 대회 생성/참가 → 4번째 참가 시 준결승 두 row 생성 → summary가 matches를 round/slot 순서로 반환 → 참가 가능한 ready match를 UI가 play route에 연결 → 준결승 완료 두 건이 확인되면 결승 생성 → 최종 match 완료 후 tournament winner 확정입니다.
- **비보장:** 이 Thread만으로 concurrent admission의 capacity 안전성이나 match 결과 확정 transaction의 원자성이 보장되지는 않습니다. 전자는 category 02, 후자는 이 category의 Thread 02에서 다룹니다.
- **실행 증거:** 로컬 checkout을 만들 수 없어 테스트 명령은 실행하지 않았습니다. 위 내용은 각 exact SHA의 diff, 파일, 테스트 구현을 검사한 결과이며 runtime 성공으로 표시하지 않았습니다.
<!-- LEARNER-ANSWER END thread:01-tournament-contract-schema-and-bracket-construction.md:synthesis -->

## 7. 학습 완료 확인

<!-- LEARNER-ANSWER START thread:01-tournament-contract-schema-and-bracket-construction.md:checklist -->
- [x] 모든 SHA를 지정 브랜치의 exact commit으로 검사했습니다.
- [x] fixed commit map과 source classification을 보존했습니다.
- [x] earlier commit을 later HEAD 코드로 설명하지 않았습니다.
- [x] fix와 test를 원래 failure/production path에 연결했습니다.
- [x] 실제 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
- [x] Thread 최종 owner, invariant, flow, non-guarantee를 작성했습니다.
<!-- LEARNER-ANSWER END thread:01-tournament-contract-schema-and-bracket-construction.md:checklist -->
===== END FILE: 01-tournament-contract-schema-and-bracket-construction.md =====

===== BEGIN FILE: 02-tournament-room-start-rollback-and-finalization-handoff.md =====
# 토너먼트 경기방 시작 롤백과 결과 확정 인계

원문 Development Thread: `Tournament room start rollback and finalization handoff`

## 1. Thread 목표

- persisted tournament match를 두 참가자의 realtime room으로 전환하는 admission·pairing·publication 순서를 복원합니다.
- room publication 후 DB start가 실패하는 부분 성공을 어떻게 감지하고 역순 롤백하는지 확인합니다.
- generic match, rating, tournament progression을 하나의 `finalizeMatch` boundary로 합친 뒤 legacy escape hatch를 제거하는 과정을 추적합니다.

### Source에서 확정된 significance

> Tournament play crosses an in-memory room boundary and a durable tournament boundary. Publishing one without the other creates ghost rooms or stuck matches; persisting a generic result separately from bracket progression creates duplicate or partially applied outcomes. The history first exposes these split transitions, then consolidates and verifies them.

### 직접 연결되는 Critical Invariants

> A tournament room is usable only when the exact persisted tournament match has been marked running; failure restores room, timer, client association, and retryability.
>
> A finished room hands one idempotent command to persistence, where generic match, rating, tournament match linkage, next-round creation, and tournament completion are atomic.

### 직접 연결되는 Major Engineering Difficulties

> Coordinating two connected participants, an in-memory room, scheduler/timer state, and a PostgreSQL row without distributed transaction support.
>
> Making completion idempotent under repeated/concurrent calls while preserving rollback on tournament linkage failure.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `joinTournamentMatch`는 어떤 persisted status와 participant 조건을 검사하고 두 waiter를 어떤 좌우 좌석으로 배치합니까?
- 초기 구현은 room publication과 `startTournamentMatch`를 어떤 순서로 수행해 부분 성공을 만들었습니까?
- `UPDATE ... RETURNING` zero-row 검사가 없으면 어떤 존재하지 않는 match가 성공처럼 보입니까?
- `abandonRoom`은 room, timer/scheduler, clients, waiters 중 무엇을 정리해 재입장을 가능하게 합니까?
- `finalizeMatch` transaction은 어떤 row를 잠그고 어떤 결과를 한 번만 적용합니까?
- legacy `completeTournamentMatch` 제거가 왜 단순 API 정리가 아니라 invariant 강제입니까?

## 3. 완료 기준

- tournament waiter → room publication → persistent start의 정상/실패 경로를 순서대로 그릴 수 있습니다.
- rollback test의 실패 주입 위치와 관찰한 room/timer/client 상태를 설명할 수 있습니다.
- 20개 concurrent finalize 호출과 동시 semifinal 완료 테스트가 각각 어떤 중복을 막는지 구분할 수 있습니다.
- GameHub의 in-flight completion promise와 repository transaction이 담당하는 서로 다른 idempotency 범위를 설명할 수 있습니다.
- Commit map의 모든 SHA를 지정 브랜치 ancestry와 source classification에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 실행한 명령과 코드 검사만으로 확인한 사실을 구분하고 실행하지 않은 test를 통과했다고 기록하지 않습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `33b6dfc5df7a` | `feat(tournament): 토너먼트 경기 방 진행` | A | REALTIME, TOURNAMENT, RISK | Integrates persisted tournament bracket matches with the realtime game hub. |
| 2 | `e338ea32b2a6` | `feat(db): PostgreSQL tournament 경기 확정을 연결` | S | PERSISTENCE, TOURNAMENT, RISK | Persists match creation, rating changes, tournament linkage, finalist creation, and tournament completion in one locked transaction. |
| 3 | `582a1615a2c6` | `test(db): 경기 결과 단일 확정 조건 검증` | A | PERSISTENCE, TOURNAMENT, RISK | Verifies idempotent result finalization, rollback, and single final creation under repetition and concurrency. |
| 4 | `10bf15723591` | `refactor(game): 경기 결과 확정 boundary 사용` | A | REALTIME, PERSISTENCE, TOURNAMENT | Makes room completion use the canonical atomic repository command and shares one in-flight completion promise. |
| 5 | `916683099ecd` | `fix(db): tournament start 상태 갱신 여부 확인` | A | REALTIME, PERSISTENCE, TOURNAMENT | Turns a zero-row tournament-start update into an explicit failure. |
| 6 | `480e2dc48028` | `test(db): tournament match 미갱신 거부 검증` | B | REALTIME, PERSISTENCE, TOURNAMENT | Adds a PostgreSQL integration regression for a zero-row tournament-start update. |
| 7 | `38312bcaf632` | `fix(game): tournament 시작 실패 시 room 상태 복원` | A | REALTIME, TOURNAMENT, RISK | Treats in-memory room creation and persistent tournament-start marking as one logical transition. |
| 8 | `4e2cb4ae702d` | `test(game): tournament start rollback 검증` | B | REALTIME, PERSISTENCE, TOURNAMENT | Reproduces persistence failure after in-memory tournament-room publication and verifies cleanup/retry. |
| 9 | `25a495d2cd43` | `refactor(db): 경기 결과 확정 boundary 일원화` | A | PERSISTENCE, TOURNAMENT, RISK | Removes the separate tournament-completion operation so every result passes through `finalizeMatch`. |
| 10 | `1646034acd9f` | `test(db): 경기 결과 확정 boundary 적용 검증` | B | PERSISTENCE, TOURNAMENT, TEST | Pins the narrowed repository surface: `finalizeMatch` exists and `completeTournamentMatch` does not. |

## 5. Commit별 학습 기록

### 5.1. `feat(tournament): 토너먼트 경기 방 진행`

| 항목 | 값 |
| --- | --- |
| SHA | `33b6dfc5df7a` |
| Importance | A |
| Tags | REALTIME, TOURNAMENT, RISK |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Integrates persisted tournament bracket matches with the realtime game hub.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `tournamentWaiters`
- `joinTournamentMatch`, `createRoom`, `finishRoom`, disconnect cleanup
- participant 좌우 배치와 `repo.startTournamentMatch`/`completeTournamentMatch` 호출 순서
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:33b6dfc5df7a -->
- **직전 상태:** persisted bracket과 play event는 있었지만 GameHub는 tournament match 두 참가자를 한 room으로 묶지 않았습니다.
- **구현 결정:** match를 조회해 `ready`, participant, 현재 room 부재를 확인하고 match ID별 waiter에 모은 뒤 left/right participant 순서로 room을 만듭니다. 생성 후 `startTournamentMatch`를 호출하고 종료 시 generic match 저장 뒤 별도 tournament completion을 호출합니다.
- **상태/소유권 변화:** GameHub가 tournament waiter와 room/client association을 소유하고 repository가 durable match status를 소유합니다. 두 소유자가 순차 호출로만 결합됩니다.
- **실패/edge:** room을 먼저 publish한 뒤 DB start가 실패하면 room·timer·client `roomId`가 남을 수 있습니다. 종료도 generic match가 성공하고 bracket update가 실패하는 부분 적용 가능성이 있습니다.
- **보장/비보장:** 정상 경로의 admission과 play는 연결하지만 start/finish의 cross-boundary atomicity는 보장하지 않습니다.
- **다음 연결:** `e338ea32b2a6`이 종료 결과를 하나의 persistence transaction으로 통합하고, 뒤의 `916...`–`4e2...`가 start rollback을 닫습니다.
<!-- LEARNER-ANSWER END commit:33b6dfc5df7a -->

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 Thread 관련 SHA: `e338ea32b2a6` — `feat(db): PostgreSQL tournament 경기 확정을 연결`

### 5.2. `feat(db): PostgreSQL tournament 경기 확정을 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `e338ea32b2a6` |
| Importance | S |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| 학습 깊이 | major architecture와 핵심 invariant를 previous state, transaction/ownership, failure, later verification까지 깊게 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Persists match creation, rating changes, tournament linkage, finalist creation, and tournament completion in one locked transaction.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 PostgreSQL `finalizeMatch`
- result key/idempotency row와 generic match/history/rating 처리
- `SELECT ... FOR UPDATE` tournament match/tournament lock
- semifinal two-winner final insert와 final tournament completion
- parent 상태와 비교해 transaction 시작 전 획득 상태, lock 순서, write 순서, rollback 범위를 단계별로 기록합니다.
- process-local idempotency와 durable idempotency를 구분하고, 반복·동시 호출에서 어떤 key/row가 serialization 지점인지 확인합니다.
- 정상 결과뿐 아니라 중간 failure가 이미 수행한 generic match, history, rating, tournament write를 어떻게 되돌리는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:e338ea32b2a6 -->
- **직전 상태:** GameHub는 generic `createMatch` 후 `completeTournamentMatch`를 따로 호출해 첫 단계만 commit될 수 있었습니다. 반복 finish도 동일 결과를 중복 적용할 위험이 있었습니다.
- **구현 결정:** `finalizeMatch`가 하나의 transaction 안에서 result key를 기준으로 기존 결과를 확인하고, generic match와 history/rating을 만든 뒤 tournament match 및 aggregate row를 잠급니다. tournament 참가자·winner/loser 관계를 검증하고 `match_id is null`인 row만 finished로 연결합니다. 두 semifinal이 끝났을 때 final을 만들고 final이면 tournament winner/status를 갱신합니다.
- **상태/소유권 변화:** durable completion의 유일한 owner가 repository transaction이 됩니다. GameHub는 tournament progression 단계를 조합하지 않고 command와 결과만 주고받습니다.
- **실패/edge:** 존재하지 않는 tournament match, 이미 다른 match에 연결된 row, 참가자가 아닌 winner/loser, linkage SQL 실패는 transaction 전체를 rollback해야 합니다. lock 순서는 같은 tournament aggregate를 통한 동시 semifinal serialization에 사용됩니다.
- **보장/비보장:** 동일 result key의 반복/동시 호출에서 한 generic match·한 rating 적용·한 tournament link를 목표로 하며, linkage 실패 시 앞선 match/history/rating도 남기지 않습니다. in-memory room cleanup은 이 transaction의 범위가 아닙니다.
- **다음 연결:** `582a1615a2c6`이 20회 동시 호출, 강제 rollback, 동시 semifinal 완료로 이 불변식을 직접 검증하고 `10bf...`가 GameHub를 새 boundary에 연결합니다.
<!-- LEARNER-ANSWER END commit:e338ea32b2a6 -->

비교 기준:
- 직전 Thread 관련 SHA: `33b6dfc5df7a` — `feat(tournament): 토너먼트 경기 방 진행`
- 다음 Thread 관련 SHA: `582a1615a2c6` — `test(db): 경기 결과 단일 확정 조건 검증`

### 5.3. `test(db): 경기 결과 단일 확정 조건 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `582a1615a2c6` |
| Importance | A |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies idempotent result finalization, rollback, and single final creation under repetition and concurrency.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 memory 20-call/concurrent semifinal test
- `packages/db/src/postgres.integration.test.ts`의 isolated database cases
- 강제 tournament linkage failure와 match/history/rating rollback assertion
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:582a1615a2c6 -->
- **직전 상태:** transaction 구현은 있었지만 반복 호출·동시 호출·중간 SQL 실패 시 결과가 하나만 남는다는 실행 가능한 회귀 근거가 없었습니다.
- **구현 결정/기법:** 같은 command를 20개 병렬 호출해 모든 응답이 같은 match ID를 가리키고 `created`가 한 번뿐인지 확인합니다. PostgreSQL에서는 match/history 개수와 rating 적용을 확인하고, tournament linkage 실패를 주입해 전체 rollback을 검사합니다. 두 semifinal을 동시에 확정해 final row가 정확히 하나인지도 확인합니다.
- **생산 경로:** `AppRepository.finalizeMatch`의 result-key claim, row lock, generic match/history/rating, tournament link/final insert를 실제 repository 경계로 통과합니다.
- **증명:** 반복·동시 호출의 단일 적용, transaction rollback, concurrent semifinal의 single final을 증명하도록 작성됐습니다.
- **비증명:** 여러 process의 실제 부하·장애 복구 시간이나 GameHub room cleanup을 증명하지는 않습니다.
- **다음 연결:** `10bf15723591`이 이 검증된 boundary를 realtime completion의 유일한 호출로 사용합니다.
<!-- LEARNER-ANSWER END commit:582a1615a2c6 -->

비교 기준:
- 직전 Thread 관련 SHA: `e338ea32b2a6` — `feat(db): PostgreSQL tournament 경기 확정을 연결`
- 다음 Thread 관련 SHA: `10bf15723591` — `refactor(game): 경기 결과 확정 boundary 사용`

### 5.4. `refactor(game): 경기 결과 확정 boundary 사용`

| 항목 | 값 |
| --- | --- |
| SHA | `10bf15723591` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes room completion use the canonical atomic repository command and shares one in-flight completion promise.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 Room `finishing` 필드
- `finishRoom`의 promise reuse/reset
- `finalizeRoom`의 `repo.finalizeMatch` command와 `resultKey`
- 성공 후 finished event/broadcast/cleanup 순서
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:10bf15723591 -->
- **직전 상태:** GameHub가 `createMatch`와 `completeTournamentMatch`를 직접 조합해 transaction boundary를 우회했습니다.
- **구현 결정:** room에 `finishing` promise를 저장해 동시에 들어온 finish가 같은 작업을 공유하게 하고, `finalizeRoom`은 `room:<id>:finished` result key와 선택적 tournament context를 한 번의 `repo.finalizeMatch`로 넘깁니다. rejection 시 promise를 비워 재시도를 허용합니다.
- **상태/소유권 변화:** in-flight 중복 억제는 room이, durable idempotency와 결과 생성은 repository가 담당합니다. 성공한 persistence 결과를 받은 뒤에만 finished event를 방송합니다.
- **실패/edge:** repository 실패는 finished로 발표되지 않고 promise가 reset됩니다. 이미 commit된 뒤 response만 실패하는 경우도 result key가 재시도를 안전하게 수용합니다.
- **보장/비보장:** 한 process room의 중복 finish와 durable repeated command를 함께 방어하지만 process crash 후 in-memory room 복구 자체는 다루지 않습니다.
- **다음 연결:** start 경로의 부분 성공은 여전히 남아 있어 `916...`–`4e2...`가 별도로 수정합니다.
<!-- LEARNER-ANSWER END commit:10bf15723591 -->

비교 기준:
- 직전 Thread 관련 SHA: `582a1615a2c6` — `test(db): 경기 결과 단일 확정 조건 검증`
- 다음 Thread 관련 SHA: `916683099ecd` — `fix(db): tournament start 상태 갱신 여부 확인`

### 5.5. `fix(db): tournament start 상태 갱신 여부 확인`

| 항목 | 값 |
| --- | --- |
| SHA | `916683099ecd` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Turns a zero-row tournament-start update into an explicit failure.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 PostgreSQL `startTournamentMatch`
- `UPDATE ... RETURNING id`와 result row count 검사
- ready/running condition과 not-found error
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:916683099ecd -->
- **직전 가정:** SQL `UPDATE`가 예외를 내지 않으면 match가 running으로 바뀌었다고 간주했습니다. 존재하지 않거나 조건에 맞지 않는 ID도 zero-row success처럼 통과할 수 있었습니다.
- **실제 위험:** GameHub는 durable match가 시작되지 않았는데 room을 유효하다고 유지해 두 상태가 갈라질 수 있습니다.
- **교정:** update에 `RETURNING id`를 붙이고 반환 row가 정확히 하나가 아니면 `tournament match not found` 오류를 던집니다.
- **상태/소유권 변화:** repository가 “상태 전이가 실제 row에 적용됐는지”를 확인하고 caller에 실패를 전달합니다.
- **보장/비보장:** zero-row silent success를 제거하지만 caller가 이미 만든 room을 되돌리는 일은 다음 commit의 GameHub 책임입니다.
- **다음 연결:** `480e2dc48028`이 임의 UUID regression을 추가하고 `38312bcaf632`가 이 오류를 rollback trigger로 사용합니다.
<!-- LEARNER-ANSWER END commit:916683099ecd -->

비교 기준:
- 직전 Thread 관련 SHA: `10bf15723591` — `refactor(game): 경기 결과 확정 boundary 사용`
- 다음 Thread 관련 SHA: `480e2dc48028` — `test(db): tournament match 미갱신 거부 검증`

### 5.6. `test(db): tournament match 미갱신 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `480e2dc48028` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds a PostgreSQL integration regression for a zero-row tournament-start update.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/postgres.integration.test.ts`의 존재하지 않는 UUID start case
- isolated PostgreSQL repository와 rejection assertion
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:480e2dc48028 -->
- **직전 상태:** zero-row 검사는 구현됐지만 실제 PostgreSQL driver/result 형태에서 오류가 발생하는지 고정한 test가 없었습니다.
- **기법:** isolated DB에서 존재하지 않는 random UUID로 `startTournamentMatch`를 호출하고 rejection을 요구합니다.
- **생산 경로:** mock이 아니라 PostgreSQL `UPDATE ... RETURNING`과 repository row-count 검사를 통과합니다.
- **증명/비증명:** 없는 match를 성공으로 처리하지 않음을 증명하도록 작성됐지만 room rollback은 호출하지 않습니다.
- **보장:** DB boundary regression을 고정합니다.
- **다음 연결:** `38312bcaf632`이 이 rejection을 받은 GameHub의 역순 cleanup을 구현합니다.
<!-- LEARNER-ANSWER END commit:480e2dc48028 -->

비교 기준:
- 직전 Thread 관련 SHA: `916683099ecd` — `fix(db): tournament start 상태 갱신 여부 확인`
- 다음 Thread 관련 SHA: `38312bcaf632` — `fix(game): tournament 시작 실패 시 room 상태 복원`

### 5.7. `fix(game): tournament 시작 실패 시 room 상태 복원`

| 항목 | 값 |
| --- | --- |
| SHA | `38312bcaf632` |
| Importance | A |
| Tags | REALTIME, TOURNAMENT, RISK |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Treats in-memory room creation and persistent tournament-start marking as one logical transition.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `joinTournamentMatch` try/catch
- `abandonRoom(room)` 호출과 rethrow
- room map, timer/scheduler, client room association cleanup
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:38312bcaf632 -->
- **직전 가정:** room 생성 후 `startTournamentMatch`가 실패하지 않거나 실패해도 caller 오류만 보내면 충분하다고 봤습니다.
- **실제 실패:** room map과 양 client `roomId`, timer/scheduler가 남아 참가자가 다시 join할 수 없고 live stats에도 ghost room이 나타날 수 있습니다.
- **교정:** start를 try/catch로 감싸고 실패한 room을 찾아 `abandonRoom`으로 publication을 역순 취소한 뒤 원래 오류를 다시 던집니다.
- **상태/소유권 변화:** GameHub는 자신이 획득한 in-memory 자원을 직접 rollback하고 repository는 durable row 전이만 책임집니다.
- **보장/비보장:** start failure 뒤 room/timer/client association을 제거해 retry 가능한 상태를 목표로 하지만, cleanup 누락 여부는 다음 deterministic test가 확인합니다.
- **다음 연결:** `4e2cb4ae702d`이 한 번 실패 후 같은 참가자들이 다시 join해 정상 room을 얻는지 검증합니다.
<!-- LEARNER-ANSWER END commit:38312bcaf632 -->

비교 기준:
- 직전 Thread 관련 SHA: `480e2dc48028` — `test(db): tournament match 미갱신 거부 검증`
- 다음 Thread 관련 SHA: `4e2cb4ae702d` — `test(game): tournament start rollback 검증`

### 5.8. `test(game): tournament start rollback 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4e2cb4ae702d` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Reproduces persistence failure after in-memory tournament-room publication and verifies cleanup/retry.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.tournament.test.ts`의 fake sockets
- `startTournamentMatch` `mockRejectedValueOnce` 후 성공 설정
- `activeRooms`, `scheduledRoomCount`, matched events, second join assertions
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:4e2cb4ae702d -->
- **직전 상태:** rollback 코드는 있었지만 room과 timer가 모두 사라지고 참가자가 재시도할 수 있다는 근거가 없었습니다.
- **실패 주입:** repository `startTournamentMatch`를 첫 호출에만 reject하도록 하고 두 fake socket을 같은 ready match에 pair합니다.
- **관찰:** 첫 실패 뒤 start 호출 1회, active room 0, scheduled room 0을 확인합니다. 같은 두 참가자가 다시 join하면 두 번째 start가 성공하고 room 1개와 좌우 matched event가 생기는지 확인합니다.
- **생산 경로:** 실제 `GameHub.connect`/event receive/pair/create/abandon 흐름을 사용하고 persistence만 spy/mock으로 제어합니다.
- **증명/비증명:** partial publication cleanup과 retryability를 결정적으로 검증하도록 작성됐지만 PostgreSQL transaction 자체는 이 test 범위가 아닙니다.
- **다음 연결:** start/finish 양쪽 교정 후 `25a495d2cd43`이 legacy completion escape hatch를 제거합니다.
<!-- LEARNER-ANSWER END commit:4e2cb4ae702d -->

비교 기준:
- 직전 Thread 관련 SHA: `38312bcaf632` — `fix(game): tournament 시작 실패 시 room 상태 복원`
- 다음 Thread 관련 SHA: `25a495d2cd43` — `refactor(db): 경기 결과 확정 boundary 일원화`

### 5.9. `refactor(db): 경기 결과 확정 boundary 일원화`

| 항목 | 값 |
| --- | --- |
| SHA | `25a495d2cd43` |
| Importance | A |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Removes the separate tournament-completion operation so every result passes through `finalizeMatch`.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `AppRepository` surface
- PostgreSQL/memory `completeTournamentMatch` 및 old final helper 제거
- observability operation 목록과 caller compile surface
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:25a495d2cd43 -->
- **직전 상태:** GameHub는 새 boundary를 사용했지만 repository interface에 `completeTournamentMatch`가 남아 다른 caller가 generic match/rating transaction을 우회할 수 있었습니다.
- **구현 결정:** 양 repository 구현과 interface, 관련 helper/observability 이름에서 별도 operation을 제거하고 tournament progression을 `finalizeMatch` 내부로만 남깁니다.
- **상태/소유권 변화:** “결과 확정” 책임이 명시적으로 단일 method에 귀속되며, 잘못된 호출 순서를 타입/API surface에서 만들기 어려워집니다.
- **실패/edge:** 기존 외부 caller가 있었다면 compile/runtime migration이 필요하지만 같은 branch의 caller는 이미 `10bf...`에서 전환됐습니다.
- **보장/비보장:** application-level escape hatch를 제거하지만 직접 SQL로 제약을 우회하는 권한까지 없애는 것은 아닙니다.
- **다음 연결:** `1646034acd9f`이 method 존재/부재를 contract test로 고정합니다.
<!-- LEARNER-ANSWER END commit:25a495d2cd43 -->

비교 기준:
- 직전 Thread 관련 SHA: `4e2cb4ae702d` — `test(game): tournament start rollback 검증`
- 다음 Thread 관련 SHA: `1646034acd9f` — `test(db): 경기 결과 확정 boundary 적용 검증`

### 5.10. `test(db): 경기 결과 확정 boundary 적용 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1646034acd9f` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Pins the narrowed repository surface: `finalizeMatch` exists and `completeTournamentMatch` does not.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 repository contract assertion
- memory repository object의 method presence/absence
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:1646034acd9f -->
- **직전 상태:** 구현과 interface에서 legacy method를 지웠지만 후속 변경이 다시 escape hatch를 추가해도 behavioral tests가 눈치채지 못할 수 있었습니다.
- **기법:** 생성된 repository에 `finalizeMatch`가 함수인지, `completeTournamentMatch` property가 없는지 직접 검사합니다.
- **증명:** 결과 확정 API surface가 하나라는 구조적 회귀를 고정하도록 작성됐습니다.
- **비증명:** transaction 내부 원자성이나 동시성은 `582a...` 테스트가 담당합니다.
- **최종 보장:** caller가 공식 repository contract를 사용할 때 별도 tournament completion 경로를 선택할 수 없습니다.
- **Thread 종료:** room start rollback과 durable finalization handoff의 양쪽 logical transition이 각각 실패를 소유하는 계층에서 닫힙니다.
<!-- LEARNER-ANSWER END commit:1646034acd9f -->

비교 기준:
- 직전 Thread 관련 SHA: `25a495d2cd43` — `refactor(db): 경기 결과 확정 boundary 일원화`
- 이 commit을 Thread의 마지막 상태로 사용합니다.

## 6. Thread 종합

다음 항목을 commit 순서에서 재구성합니다: invariant evolution, Failure → Fix → Test 관계, ownership/state 변화, 최종 실행 흐름, 비보장, 실제 실행 증거.

<!-- LEARNER-ANSWER START thread:02-tournament-room-start-rollback-and-finalization-handoff.md:synthesis -->
- **불변식 진화:** `33b6dfc5df7a`는 tournament waiter를 pair하고 room을 만든 뒤 DB start를 호출했으므로 in-memory publication과 persistence가 갈라질 수 있었습니다. `916683099ecd`가 zero-row update를 오류로 바꾸고, `38312bcaf632`가 실패 시 `abandonRoom`을 호출하며, `4e2cb4ae702d`가 재입장까지 검증했습니다. 종료 쪽은 `e338ea32b2a6`–`10bf15723591`에서 generic match·rating·bracket progression을 하나의 transaction command로 통합했습니다.
- **소유권과 인계:** GameHub는 waiter, room, client association, scheduler/timer와 현재 in-flight completion promise를 소유합니다. repository는 match result key, generic match/history/rating, tournament row/match row, final 생성/우승 확정을 소유합니다. GameHub는 결과를 계산한 뒤 한 번의 `finalizeMatch` command로 durable ownership을 넘깁니다.
- **Failure → Fix → Test:** zero-row start → `RETURNING id` 검사 → PostgreSQL regression. room publish 후 start 실패 → `abandonRoom` rollback → fake-socket 재입장 test. 두 단계 completion → transactional finalize + in-flight promise → 20-call idempotency/rollback/concurrent-semifinal tests → legacy boundary 제거와 surface test로 닫힙니다.
- **최종 흐름:** 참가자 두 명이 동일 ready match에 join → 좌석 결정과 room publish → DB running 전환; 실패하면 room 전체 폐기 → 경기 종료 시 동일 room completion은 같은 promise 공유 → repository transaction이 result key와 tournament rows를 잠그고 한 번만 결과 적용 → 성공 후 finished broadcast/cleanup입니다.
- **비보장:** 이 Thread는 later Matchmaker reservation refactor 전체나 process drain을 소유하지 않습니다. 그 ownership 통합은 category 05 core realtime Thread가 주 소유자입니다.
- **실행 증거:** 테스트 파일과 실패 주입 코드는 exact SHA에서 검사했으나 로컬 checkout 부재로 실행하지 않았습니다. 따라서 “테스트가 통과했다”가 아니라 “해당 regression을 검증하도록 구현되어 있다”고 기록합니다.
<!-- LEARNER-ANSWER END thread:02-tournament-room-start-rollback-and-finalization-handoff.md:synthesis -->

## 7. 학습 완료 확인

<!-- LEARNER-ANSWER START thread:02-tournament-room-start-rollback-and-finalization-handoff.md:checklist -->
- [x] 모든 SHA를 지정 브랜치의 exact commit으로 검사했습니다.
- [x] fixed commit map과 source classification을 보존했습니다.
- [x] earlier commit을 later HEAD 코드로 설명하지 않았습니다.
- [x] fix와 test를 원래 failure/production path에 연결했습니다.
- [x] 실제 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
- [x] Thread 최종 owner, invariant, flow, non-guarantee를 작성했습니다.
<!-- LEARNER-ANSWER END thread:02-tournament-room-start-rollback-and-finalization-handoff.md:checklist -->
===== END FILE: 02-tournament-room-start-rollback-and-finalization-handoff.md =====

===== BEGIN FILE: 03-profile-friendship-dashboard-and-ranking-journeys.md =====
# 프로필·친구·대시보드·순위표 여정

원문 Development Thread: `Profile, friendship, dashboard, and ranking journeys`

## 1. Thread 목표

- profile·leaderboard·recent-match·dashboard read model이 repository와 HTTP route로 추가되는 과정을 추적합니다.
- sample user, 고정 그래프, 추정 연승처럼 서버 사실이 아닌 표시가 실제 데이터·명시적 empty/error state로 교체되는 과정을 확인합니다.
- React Query helper/key/invalidation이 identity-bound mutation 뒤 관련 read model을 어떻게 무효화하는지 복원합니다.

### Source에서 확정된 significance

> These screens look read-oriented, but they define which data is treated as fact. The history shows several fabricated fallbacks and metrics that could misrepresent authenticated state. Later commits replace them with repository-derived projections, explicit failure states, and cache ownership rules.

### 직접 연결되는 Critical Invariants

> Authenticated and public screens do not substitute sample identities, matches, rankings, or metrics when server reads fail.
>
> Dashboard metrics and charts are derived from the bounded recent-match read model and disclose that boundary instead of claiming broader history.

### 직접 연결되는 Major Engineering Difficulties

> Reconstructing chronological rating/streak state from a newest-first bounded match collection without inventing missing history.
>
> Keeping profile, session, lobby, dashboard, friends, leaderboard, tournament, and admin caches coherent after identity-affecting mutations.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- repository profile update는 현재 row와 optional input을 어떻게 결합하며 public/session projection을 어떻게 나눕니까?
- leaderboard rank와 win rate는 어떤 정렬·반올림·zero-game 규칙으로 계산됩니까?
- 초기 dashboard `bestStreak`와 rating chart는 실제 match history가 아니라 무엇에서 만들어졌습니까?
- `bestWinningStreak`는 newest-first 결과를 왜 reverse하고 loss에서 어떤 상태를 reset합니까?
- sample fallback 제거 후 loading, request failure, empty result는 화면에서 어떻게 구분됩니까?
- profile mutation은 어떤 query key들을 invalidate하며 session expiry는 어떤 cache를 제거합니까?

## 3. 완료 기준

- profile, leaderboard, recent matches, dashboard의 repository → route → browser adapter → 화면 흐름을 설명할 수 있습니다.
- 고정/sample/fabricated 데이터가 실제 서버 결과처럼 보였던 각 지점을 파일과 함수로 지적할 수 있습니다.
- 최고 연승과 rating chart가 recent-match 범위에서 계산되는 정확한 순서와 비보장을 설명할 수 있습니다.
- 친구 관계의 canonical persistence invariant가 이 Thread가 아니라 category 02에 있다는 경계를 구분할 수 있습니다.
- Commit map의 모든 SHA를 지정 브랜치 ancestry와 source classification에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 실행한 명령과 코드 검사만으로 확인한 사실을 구분하고 실행하지 않은 test를 통과했다고 기록하지 않습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `c5b96a06925c` | `feat(db): 프로필 조회와 변경 저장 구현` | B | PERSISTENCE | Extends identity access with public profile reads, authenticated profile updates, and active-user listing. |
| 2 | `0364c42f776b` | `feat(db): 순위 조회 구현` | B | PERSISTENCE | Adds a leaderboard projection to the repository contract. |
| 3 | `c7ea1ff241c8` | `feat(db): 최근 경기와 대시보드 조회 구현` | B | PERSISTENCE | Implements recent-match and dashboard reads behind the repository contract. |
| 4 | `0bcc487d949f` | `feat(api): 프로필과 친구 리소스 라우트 추가` | B | PERSISTENCE | Extends HTTP resources to profile, dashboard, and friendship while separating public reads from identity-bound mutations. |
| 5 | `cbe876359d31` | `feat(web): 플레이어 대시보드 구현` | B | WEB | Adds a dashboard route consuming the shared `DashboardSummary` read model. |
| 6 | `cb295396771f` | `feat(web): 순위표 화면 추가` | B | WEB | Renders rank, player record, rating, and win rate from the shared leaderboard contract. |
| 7 | `0afc0a0694bd` | `feat(web): 공개 프로필 화면 추가` | B | WEB | Adds a dynamic public-profile route keyed by handle. |
| 8 | `051eac1b4aee` | `feat(profile): 친구 요청 동작 연결` | B | AUTH, WEB | Connects the dynamic profile route to public-profile lookup and authenticated friend-request mutation. |
| 9 | `51e66cf1df80` | `fix(profile): 공개 프로필 상태 표현 개선` | B | PERSISTENCE, WEB | Stores and renders the recent-match collection returned with a public profile. |
| 10 | `8d79139a32da` | `fix(dashboard): 경기 상태 표현 개선` | B | WEB | Replaces a fixed chart with points reconstructed from current rating and recent match deltas. |
| 11 | `be31566ac0fd` | `fix(web): 로그인 화면의 sample fallback 제거` | A | AUTH, TOURNAMENT, WEB | Stops authenticated and server-backed screens from displaying sample data after request failure. |
| 12 | `035b97ca7c58` | `fix(db): 최근 경기에서 최고 연승 계산` | B | PERSISTENCE | Calculates dashboard best winning streak from actual recent match results instead of formulas/constants. |
| 13 | `6b661420e060` | `test(db): 최고 연승 계산 검증` | B | PERSISTENCE, TEST | Pins the winning-streak ordering and reset rules. |
| 14 | `7fe29f991a9b` | `fix(dashboard): 연승 지표 설명 정정` | C | - | Changes the dashboard hint from “this season” to “recent matches” so the label matches the data boundary. |
| 15 | `3c6c9134ee94` | `fix(dashboard): 빈 rating history를 정확히 표시` | B | PERSISTENCE, WEB | Treats empty match history as no rating evidence instead of fabricating a two-point chart. |
| 16 | `c17e7ad0fd84` | `feat(web): profile과 friend 조회 query 추가` | B | WEB | Adds schema-validated profile/friend browser helpers and scoped React Query options. |
| 17 | `8bc4d0cc32bd` | `test(web): profile과 friend 조회 규칙 검증` | B | AUTH, WEB, TEST | Verifies own-profile/friend request helpers and React Query ownership rules. |

## 5. Commit별 학습 기록

### 5.1. `feat(db): 프로필 조회와 변경 저장 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `c5b96a06925c` |
| Importance | B |
| Tags | PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Extends identity access with public profile reads, authenticated profile updates, and active-user listing.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `getUserById`, `getUserByHandle`, `updateProfile`, `listOnlineUsers`
- PostgreSQL normalized-handle lookup과 optional display/avatar update
- memory repository의 같은 contract
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:c5b96a06925c -->
- **직전 상태:** repository identity access는 development login/session 중심이어서 public handle 조회와 인증된 profile mutation이 없었습니다.
- **구현 결정:** ID/normalized handle 조회, 기존 값을 유지하는 optional displayName/avatar update, active user의 rating 내림차순 목록을 공통 interface에 추가했습니다.
- **상태/소유권 변화:** profile row mutation과 public/session projection은 repository가 소유하고 route는 이후 caller가 됩니다.
- **실패/edge:** 존재하지 않는 사용자와 비활성 상태 처리는 caller-visible null/error로 남고, `online` projection은 실제 WebSocket presence가 아닙니다.
- **보장/비보장:** PostgreSQL/memory의 기본 profile 동작은 맞추지만 friendship과 realtime presence는 별도 책임입니다.
- **다음 연결:** `0364c42f776b`이 같은 사용자 projection 위에 leaderboard read model을 추가합니다.
<!-- LEARNER-ANSWER END commit:c5b96a06925c -->

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 Thread 관련 SHA: `0364c42f776b` — `feat(db): 순위 조회 구현`

### 5.2. `feat(db): 순위 조회 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `0364c42f776b` |
| Importance | B |
| Tags | PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds a leaderboard projection to the repository contract.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `listLeaderboard`
- PostgreSQL `ORDER BY rating DESC, wins DESC LIMIT 20`
- rank index와 win-rate 반올림/zero-game 처리
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:0364c42f776b -->
- **직전 상태:** active user 목록은 있었지만 순위 번호·승률을 포함한 전용 read model이 없었습니다.
- **구현 결정:** rating 내림차순, 동률 시 wins 내림차순으로 정렬하고 index+1을 rank로, `round(wins / total * 1000) / 10`을 승률로 계산합니다. 경기가 없으면 0입니다.
- **상태/소유권 변화:** 순위 계산은 web이 아니라 repository projection이 담당합니다.
- **실패/edge:** PostgreSQL은 20명 제한이고 memory 구현은 당시 전체 배열을 map해 backend별 limit 차이가 남을 수 있습니다.
- **보장/비보장:** 반환된 collection 안의 deterministic ordering/rank는 보장하지만 시즌·pagination 개념은 없습니다.
- **다음 연결:** `c7ea1ff241c8`이 match history와 dashboard projection을 추가합니다.
<!-- LEARNER-ANSWER END commit:0364c42f776b -->

비교 기준:
- 직전 Thread 관련 SHA: `c5b96a06925c` — `feat(db): 프로필 조회와 변경 저장 구현`
- 다음 Thread 관련 SHA: `c7ea1ff241c8` — `feat(db): 최근 경기와 대시보드 조회 구현`

### 5.3. `feat(db): 최근 경기와 대시보드 조회 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `c7ea1ff241c8` |
| Importance | B |
| Tags | PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Implements recent-match and dashboard reads behind the repository contract.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `listRecentMatches`와 `getDashboard`
- winner/loser join, optional user filter, `ended_at desc limit 8`
- 초기 PostgreSQL/memory `bestStreak` 계산 차이
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:c7ea1ff241c8 -->
- **직전 상태:** match 저장은 있어도 사용자 관점의 상대·결과·점수·rating delta를 묶은 최근 경기와 dashboard summary가 없었습니다.
- **구현 결정:** newest-first 최대 8경기를 사용자 관점으로 projection하고 dashboard에 me, wins/losses, winRate, bestStreak, recentMatches를 넣었습니다.
- **상태/소유권 변화:** match row를 사용자별 read model로 바꾸는 책임이 repository로 이동했습니다.
- **실패/edge:** 최고 연승은 실제 history가 아니라 PostgreSQL `max(1,min(12,wins-losses+3))`, memory 고정 `3`이어서 fabricated metric입니다.
- **보장/비보장:** 최근 경기 ordering과 기본 dashboard 구조는 제공하지만 bestStreak 정확성은 후속 fix 전까지 보장하지 않습니다.
- **다음 연결:** `0bcc487d949f`가 이 read model을 HTTP resource로 노출합니다.
<!-- LEARNER-ANSWER END commit:c7ea1ff241c8 -->

비교 기준:
- 직전 Thread 관련 SHA: `0364c42f776b` — `feat(db): 순위 조회 구현`
- 다음 Thread 관련 SHA: `0bcc487d949f` — `feat(api): 프로필과 친구 리소스 라우트 추가`

### 5.4. `feat(api): 프로필과 친구 리소스 라우트 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `0bcc487d949f` |
| Importance | B |
| Tags | PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Extends HTTP resources to profile, dashboard, and friendship while separating public reads from identity-bound mutations.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`의 `/users/:id`, `/dashboard`, `/profile/:handle`, `/profile/me`
- `/friends`와 request/accept route
- `currentUser` 사용 여부와 public/authenticated response
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:0bcc487d949f -->
- **직전 상태:** repository operation은 있었지만 browser가 사용할 profile/dashboard/friend HTTP route가 없었습니다.
- **구현 결정:** public profile read와 인증된 own-profile read/update, dashboard, friends/list/request/accept를 별도 route로 연결했습니다.
- **상태/소유권 변화:** route는 인증 identity를 mutation target에 결합하고 repository는 저장/조회 로직을 유지합니다.
- **실패/edge:** 화면이 실패를 sample로 대체하면 route 권한·오류가 숨겨질 수 있으며, friendship canonicalization은 이후 category 02에서 강화됩니다.
- **보장/비보장:** 접근 형태는 구분하지만 web의 cache coherence와 honest failure display는 아직 없습니다.
- **다음 연결:** `cbe876359d31`–`051eac1b4aee`가 이 API를 화면 journey에 연결합니다.
<!-- LEARNER-ANSWER END commit:0bcc487d949f -->

비교 기준:
- 직전 Thread 관련 SHA: `c7ea1ff241c8` — `feat(db): 최근 경기와 대시보드 조회 구현`
- 다음 Thread 관련 SHA: `cbe876359d31` — `feat(web): 플레이어 대시보드 구현`

### 5.5. `feat(web): 플레이어 대시보드 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `cbe876359d31` |
| Importance | B |
| Tags | WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds a dashboard route consuming the shared `DashboardSummary` read model.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/dashboard/page.tsx`의 sample 초기 state와 `getDashboard`
- 고정 SVG `polyline`
- stats와 recent match rendering
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:cbe876359d31 -->
- **직전 상태:** dashboard API는 있었지만 사용자 화면이 없었습니다.
- **구현 결정:** sample dashboard를 초기값으로 두고 실제 API 결과로 교체하며 승/패/승률/연승과 최근 경기를 표시했습니다.
- **상태/소유권 변화:** 화면이 서버 read model을 소비하지만 고정 SVG와 sample 초기값으로 일부 사실을 자체 생성합니다.
- **실패/edge:** request 실패 시 sample이 남고 chart는 실제 rating history와 무관한 고정 상승선입니다.
- **보장/비보장:** 기본 dashboard journey는 제공하지만 표시가 전부 서버 사실이라는 보장은 없습니다.
- **다음 연결:** `8d79139a32da`가 chart를 match delta에서 계산하고 `be31566ac0fd`가 sample fallback을 제거합니다.
<!-- LEARNER-ANSWER END commit:cbe876359d31 -->

비교 기준:
- 직전 Thread 관련 SHA: `0bcc487d949f` — `feat(api): 프로필과 친구 리소스 라우트 추가`
- 다음 Thread 관련 SHA: `cb295396771f` — `feat(web): 순위표 화면 추가`

### 5.6. `feat(web): 순위표 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `cb295396771f` |
| Importance | B |
| Tags | WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Renders rank, player record, rating, and win rate from the shared leaderboard contract.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/leaderboard/page.tsx`의 sample 초기 state
- `getLeaderboard` 호출과 entry rendering
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:cb295396771f -->
- **직전 상태:** repository/API가 순위를 계산해도 전용 web route가 없었습니다.
- **구현 결정:** `LeaderboardEntry`를 받아 rank, identity, wins, rating, winRate를 표로 표시했습니다.
- **상태/소유권 변화:** ranking 계산은 repository에 남고 web은 projection을 렌더링합니다.
- **실패/edge:** API 실패 시 sample leaderboard가 실제 현재 순위처럼 보일 수 있습니다.
- **보장/비보장:** 정상 응답 rendering은 제공하지만 실패의 진실성은 `be31566ac0fd` 전까지 부족합니다.
- **다음 연결:** sample fallback 제거 commit이 empty/error 상태로 바꿉니다.
<!-- LEARNER-ANSWER END commit:cb295396771f -->

비교 기준:
- 직전 Thread 관련 SHA: `cbe876359d31` — `feat(web): 플레이어 대시보드 구현`
- 다음 Thread 관련 SHA: `0afc0a0694bd` — `feat(web): 공개 프로필 화면 추가`

### 5.7. `feat(web): 공개 프로필 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `0afc0a0694bd` |
| Importance | B |
| Tags | WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds a dynamic public-profile route keyed by handle.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/profile/[handle]/page.tsx`의 sample user selection
- handle 길이 기반 선수 번호, 정적 play style, 초기 friend/share controls
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:0afc0a0694bd -->
- **직전 상태:** public profile API는 있었지만 handle route가 없었습니다.
- **구현 결정:** route handle에 따라 sample user를 선택하고 전적·rating·정적 style과 action UI를 표시했습니다.
- **상태/소유권 변화:** 초기 화면은 profile 사실을 서버가 아니라 sample module과 handle 길이에서 만들어 냅니다.
- **실패/edge:** 존재하지 않는 handle도 가짜 사용자로 표현되고 선수 번호·style에 repository 근거가 없습니다.
- **보장/비보장:** route shell만 제공하며 실제 identity/read-model 정확성은 보장하지 않습니다.
- **다음 연결:** `051eac1b4aee`가 API와 친구 요청을 연결하지만 실패 시 sample은 아직 남습니다.
<!-- LEARNER-ANSWER END commit:0afc0a0694bd -->

비교 기준:
- 직전 Thread 관련 SHA: `cb295396771f` — `feat(web): 순위표 화면 추가`
- 다음 Thread 관련 SHA: `051eac1b4aee` — `feat(profile): 친구 요청 동작 연결`

### 5.8. `feat(profile): 친구 요청 동작 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `051eac1b4aee` |
| Importance | B |
| Tags | AUTH, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Connects the dynamic profile route to public-profile lookup and authenticated friend-request mutation.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/profile/[handle]/page.tsx`의 `getProfile`, `requestFriend`
- route handle을 target으로 사용하는 friend action
- fetch/mutation catch와 sample fallback
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:051eac1b4aee -->
- **직전 상태:** profile page와 버튼은 정적 sample 동작이었습니다.
- **구현 결정:** resolved route handle로 public profile을 요청하고 같은 handle을 friend-request target으로 사용합니다.
- **상태/소유권 변화:** identity target은 URL과 server response로 이동하지만 initial/failure state는 sample user를 유지합니다.
- **실패/edge:** profile fetch failure가 sample 표시로 숨겨지고 friend mutation 실패도 일반 메시지로만 처리됩니다.
- **보장/비보장:** 정상 요청의 target 연결은 보장하지만 실패 화면이 실제 identity를 반영한다는 보장은 없습니다.
- **다음 연결:** `51e66cf1df80`이 최근 경기 표시를 실제 response에 연결하고 `be31566ac0fd`가 sample을 제거합니다.
<!-- LEARNER-ANSWER END commit:051eac1b4aee -->

비교 기준:
- 직전 Thread 관련 SHA: `0afc0a0694bd` — `feat(web): 공개 프로필 화면 추가`
- 다음 Thread 관련 SHA: `51e66cf1df80` — `fix(profile): 공개 프로필 상태 표현 개선`

### 5.9. `fix(profile): 공개 프로필 상태 표현 개선`

| 항목 | 값 |
| --- | --- |
| SHA | `51e66cf1df80` |
| Importance | B |
| Tags | PERSISTENCE, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Stores and renders the recent-match collection returned with a public profile.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/profile/[handle]/page.tsx`의 `recentMatches` state
- result/opponent/score list와 empty state
- 당시 fetch failure message와 남아 있는 sample user
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:51e66cf1df80 -->
- **직전 상태:** 공개 프로필은 정적 play style을 보여 실제 match history를 사용하지 않았습니다.
- **구현 결정:** profile response의 `recentMatches`를 보관해 승패·상대·점수를 렌더링하고 없으면 명시적 empty state를 표시합니다.
- **상태/소유권 변화:** 플레이 상태 설명이 정적 문구에서 repository-backed match read model로 이동합니다.
- **실패/edge:** fetch 실패 시 메시지는 sample 표시를 인정하지만 sample user 자체는 여전히 남습니다.
- **보장/비보장:** 정상 response의 recent-match rendering은 정확하지만 실패 identity는 후속 fix 전까지 부정확합니다.
- **다음 연결:** `8d79139a32da`가 dashboard chart도 실제 match delta로 전환합니다.
<!-- LEARNER-ANSWER END commit:51e66cf1df80 -->

비교 기준:
- 직전 Thread 관련 SHA: `051eac1b4aee` — `feat(profile): 친구 요청 동작 연결`
- 다음 Thread 관련 SHA: `8d79139a32da` — `fix(dashboard): 경기 상태 표현 개선`

### 5.10. `fix(dashboard): 경기 상태 표현 개선`

| 항목 | 값 |
| --- | --- |
| SHA | `8d79139a32da` |
| Importance | B |
| Tags | WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Replaces a fixed chart with points reconstructed from current rating and recent match deltas.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/dashboard/page.tsx`의 `buildRatingPoints`
- `toChartPoints`의 min/max/range 및 640×150 scaling
- empty history 당시 `[currentRating - 1, currentRating]` fallback
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:8d79139a32da -->
- **직전 상태:** chart는 고정 좌표라 사용자 rating history와 무관했습니다.
- **구현 결정:** newest-first matches를 reverse하고 delta 합을 현재 rating에서 빼 시작 rating을 구한 뒤 시간순으로 누적합니다. 화면 좌표는 min/max range를 640×150 영역으로 정규화합니다.
- **상태/소유권 변화:** chart shape는 실제 recent-match delta에서 유도됩니다.
- **실패/edge:** match가 없으면 `[current-1,current]` 두 점을 만들어 여전히 근거 없는 상승선을 표시합니다.
- **보장/비보장:** bounded recent history의 변화는 반영하지만 전체 rating history와 empty truthfulness는 보장하지 않습니다.
- **다음 연결:** `3c6c9134ee94`가 empty history fabrication을 제거합니다.
<!-- LEARNER-ANSWER END commit:8d79139a32da -->

비교 기준:
- 직전 Thread 관련 SHA: `51e66cf1df80` — `fix(profile): 공개 프로필 상태 표현 개선`
- 다음 Thread 관련 SHA: `be31566ac0fd` — `fix(web): 로그인 화면의 sample fallback 제거`

### 5.11. `fix(web): 로그인 화면의 sample fallback 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `be31566ac0fd` |
| Importance | A |
| Tags | AUTH, TOURNAMENT, WEB |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Stops authenticated and server-backed screens from displaying sample data after request failure.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- admin/dashboard/leaderboard/lobby/profile/tournaments page의 sample import 제거
- nullable dashboard/profile와 empty arrays
- loading/error/empty message 및 mutation error handling
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:be31566ac0fd -->
- **직전 가정:** 개발 편의를 위해 request가 실패해도 sample user·match·chat·ranking·tournament를 보여 주는 것이 허용됐습니다.
- **실제 위험:** 인증 실패, 권한 거부, 서버 장애, 빈 데이터가 모두 성공한 제품 상태처럼 보이고 사용자가 가짜 identity/action target을 볼 수 있었습니다.
- **교정:** server-backed state를 null/empty로 시작하고 성공 시에만 채웁니다. 실패는 명시적 메시지, empty는 별도 문구로 표시하며 profile은 user가 없으면 본문을 렌더링하지 않습니다.
- **상태/소유권 변화:** sample module이 read-model authority에서 제거되고 server response만 제품 데이터가 됩니다. web은 loading/error/empty 표현만 소유합니다.
- **보장/비보장:** 실패 은폐는 제거하지만 server 데이터 자체의 정확성은 repository invariant와 tests에 의존합니다.
- **다음 연결:** `035b97ca7c58`이 여전히 fabricated였던 backend 최고 연승을 실제 recent matches에서 계산합니다.
<!-- LEARNER-ANSWER END commit:be31566ac0fd -->

비교 기준:
- 직전 Thread 관련 SHA: `8d79139a32da` — `fix(dashboard): 경기 상태 표현 개선`
- 다음 Thread 관련 SHA: `035b97ca7c58` — `fix(db): 최근 경기에서 최고 연승 계산`

### 5.12. `fix(db): 최근 경기에서 최고 연승 계산`

| 항목 | 값 |
| --- | --- |
| SHA | `035b97ca7c58` |
| Importance | B |
| Tags | PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Calculates dashboard best winning streak from actual recent match results instead of formulas/constants.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 양 repository `getDashboard`
- `bestWinningStreak` helper
- newest-first collection reverse, win increment, loss reset
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:035b97ca7c58 -->
- **직전 가정:** 누적 wins/losses formula나 고정 3이 최고 연승을 근사할 수 있다고 봤습니다.
- **실제 오류:** 동일 누적 전적이라도 경기 순서에 따라 연승이 달라지며 PostgreSQL/memory가 서로 다른 값을 반환했습니다.
- **교정:** `listRecentMatches` 결과를 reverse해 시간순으로 순회하고 win이면 current/max를 증가, loss이면 current를 0으로 reset합니다.
- **상태/소유권 변화:** metric은 repository별 임의 값이 아니라 공유 result sequence에서 계산됩니다.
- **보장/비보장:** 반환된 최근 경기 범위 안의 최고 연승만 정확하며 전체 시즌 기록은 아닙니다.
- **다음 연결:** `6b661420e060`이 ordering과 loss reset을 W/L/W/W sequence로 고정합니다.
<!-- LEARNER-ANSWER END commit:035b97ca7c58 -->

비교 기준:
- 직전 Thread 관련 SHA: `be31566ac0fd` — `fix(web): 로그인 화면의 sample fallback 제거`
- 다음 Thread 관련 SHA: `6b661420e060` — `test(db): 최고 연승 계산 검증`

### 5.13. `test(db): 최고 연승 계산 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `6b661420e060` |
| Importance | B |
| Tags | PERSISTENCE, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Pins the winning-streak ordering and reset rules.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 memory repository case
- chronological W,L,W,W 생성과 newest-first 반환 assertion
- `bestStreak === 2` assertion
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:6b661420e060 -->
- **직전 상태:** helper는 구현됐지만 newest-first read model을 잘못 순회하거나 loss reset을 빠뜨리는 regression을 막을 test가 없었습니다.
- **기법:** 시간순 W,L,W,W를 저장하고 read model이 newest-first W,W,L,W인지 확인한 뒤 dashboard 최고 연승이 2인지 검사합니다.
- **생산 경로:** memory repository의 match 저장·recent projection·dashboard calculation을 함께 통과합니다.
- **증명/비증명:** ordering과 reset rule을 검증하도록 작성됐지만 PostgreSQL query ordering은 별도 integration 근거가 필요합니다.
- **보장:** formula/constant로 돌아가는 regression을 감지합니다.
- **다음 연결:** `7fe29f991a9b`이 화면 설명을 실제 bounded metric에 맞춥니다.
<!-- LEARNER-ANSWER END commit:6b661420e060 -->

비교 기준:
- 직전 Thread 관련 SHA: `035b97ca7c58` — `fix(db): 최근 경기에서 최고 연승 계산`
- 다음 Thread 관련 SHA: `7fe29f991a9b` — `fix(dashboard): 연승 지표 설명 정정`

### 5.14. `fix(dashboard): 연승 지표 설명 정정`

| 항목 | 값 |
| --- | --- |
| SHA | `7fe29f991a9b` |
| Importance | C |
| Tags | - |
| 학습 깊이 | Thread 이해에 필요한 제한된 문맥만 기록합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Changes the dashboard hint from “this season” to “recent matches” so the label matches the data boundary.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/dashboard/page.tsx`의 최고 연승 `StatCard` hint 한 줄
- 이 한정된 변경이 Thread의 실제 의미 또는 설명 정확성에 기여하는 부분만 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:7fe29f991a9b -->
- **변경:** 최고 연승 설명을 `이번 시즌`에서 `최근 경기`로 바꿨습니다.
- **이유:** repository는 최대 최근 8경기만 계산하므로 시즌 전체라는 주장은 근거가 없습니다.
- **범위:** 계산·저장·API 동작은 바꾸지 않는 표시 정확성 보정입니다.
<!-- LEARNER-ANSWER END commit:7fe29f991a9b -->

비교 기준:
- 직전 Thread 관련 SHA: `6b661420e060` — `test(db): 최고 연승 계산 검증`
- 다음 Thread 관련 SHA: `3c6c9134ee94` — `fix(dashboard): 빈 rating history를 정확히 표시`

### 5.15. `fix(dashboard): 빈 rating history를 정확히 표시`

| 항목 | 값 |
| --- | --- |
| SHA | `3c6c9134ee94` |
| Importance | B |
| Tags | PERSISTENCE, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Treats empty match history as no rating evidence instead of fabricating a two-point chart.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/dashboard/page.tsx`의 `hasRatingHistory`
- empty chart message와 polyline 조건
- `buildRatingPoints`의 가짜 `[current-1,current]` 제거
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:3c6c9134ee94 -->
- **직전 가정:** chart component를 항상 그리기 위해 empty history에도 1점 상승한 두 좌표를 만들었습니다.
- **실제 위험:** 경기가 없는 사용자에게 rating 상승 이력이 존재하는 것처럼 보였습니다.
- **교정:** history 존재 여부를 별도로 계산하고 없으면 polyline 대신 “표시할 경기 이력 없음” 상태를 렌더링하며 helper는 실제 point만 반환합니다.
- **상태/소유권 변화:** presentation이 데이터 부재를 숨기지 않습니다.
- **보장/비보장:** empty truthfulness는 보장하지만 bounded recent history 이전의 변화는 여전히 표시하지 않습니다.
- **다음 연결:** `c17e7ad0fd84`이 profile/friend reads와 mutation invalidation을 React Query로 구조화합니다.
<!-- LEARNER-ANSWER END commit:3c6c9134ee94 -->

비교 기준:
- 직전 Thread 관련 SHA: `7fe29f991a9b` — `fix(dashboard): 연승 지표 설명 정정`
- 다음 Thread 관련 SHA: `c17e7ad0fd84` — `feat(web): profile과 friend 조회 query 추가`

### 5.16. `feat(web): profile과 friend 조회 query 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `c17e7ad0fd84` |
| Importance | B |
| Tags | WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds schema-validated profile/friend browser helpers and scoped React Query options.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/lib/api.ts`의 `getFriends`, `getOwnProfile`, `updateOwnProfile`
- `apps/web/src/lib/query.ts`의 keys/options
- profile update 성공 시 me/own/public/lobby/dashboard/friends/leaderboard/tournaments/admin invalidation
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:c17e7ad0fd84 -->
- **직전 상태:** profile/friend 요청은 page effect와 ad-hoc state에 흩어져 cache ownership과 mutation 후 refresh 범위가 명시적이지 않았습니다.
- **구현 결정:** typed API helper와 query key factory/options를 추가하고 own-profile mutation 뒤 identity가 반영될 수 있는 모든 read model을 invalidate합니다. session expiry는 own-profile cache도 제거합니다.
- **상태/소유권 변화:** server state freshness는 React Query cache/key가, 화면 local state는 presentation만 담당합니다.
- **실패/edge:** 넓은 invalidation은 추가 요청 비용이 있지만 stale display를 방지합니다. 서버 mutation 자체의 원자성은 이 layer 범위가 아닙니다.
- **보장/비보장:** 공식 helper를 쓰는 화면의 cache coherence를 높이지만 임의 direct fetch caller까지 강제하지는 않습니다.
- **다음 연결:** `8bc4d0cc32bd`가 URL/credentials/body/key/invalidation/session-expiry 규칙을 테스트합니다.
<!-- LEARNER-ANSWER END commit:c17e7ad0fd84 -->

비교 기준:
- 직전 Thread 관련 SHA: `3c6c9134ee94` — `fix(dashboard): 빈 rating history를 정확히 표시`
- 다음 Thread 관련 SHA: `8bc4d0cc32bd` — `test(web): profile과 friend 조회 규칙 검증`

### 5.17. `test(web): profile과 friend 조회 규칙 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `8bc4d0cc32bd` |
| Importance | B |
| Tags | AUTH, WEB, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies own-profile/friend request helpers and React Query ownership rules.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- web API helper test의 URL, credentials, AbortSignal, PATCH body assertion
- query key/options test
- profile update invalidation과 session expiry cache removal
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:8bc4d0cc32bd -->
- **직전 상태:** helper와 invalidation 목록은 구현됐지만 URL·credential·body·cache 범위가 후속 refactor에서 바뀌어도 잡을 test가 없었습니다.
- **기법:** fetch를 대체해 own profile/friends 요청의 URL, credential, signal과 PATCH JSON body를 확인하고 query client spy로 invalidate/remove 호출을 검사합니다.
- **생산 경로:** browser adapter와 React Query option/mutation lifecycle을 직접 실행합니다.
- **증명/비증명:** client request/cache ownership을 검증하도록 작성됐지만 실제 HTTP 서버와 DB friendship invariant는 검증하지 않습니다.
- **보장:** profile mutation/session expiry 뒤 stale identity 관련 cache가 남는 regression을 감지합니다.
- **Thread 종료:** read model의 사실성, bounded metric 설명, client cache ownership이 연결됩니다.
<!-- LEARNER-ANSWER END commit:8bc4d0cc32bd -->

비교 기준:
- 직전 Thread 관련 SHA: `c17e7ad0fd84` — `feat(web): profile과 friend 조회 query 추가`
- 이 commit을 Thread의 마지막 상태로 사용합니다.

## 6. Thread 종합

다음 항목을 commit 순서에서 재구성합니다: invariant evolution, Failure → Fix → Test 관계, ownership/state 변화, 최종 실행 흐름, 비보장, 실제 실행 증거.

<!-- LEARNER-ANSWER START thread:03-profile-friendship-dashboard-and-ranking-journeys.md:synthesis -->
- **불변식 진화:** 초기 repository는 profile/leaderboard/recent-match/dashboard를 제공했지만 `getDashboard`의 최고 연승은 PostgreSQL에서 `wins - losses + 3`을 제한한 값, memory에서 고정 3이었습니다. web은 sample user와 고정 SVG 그래프를 사용하고 API 실패를 sample로 숨겼습니다. `be31566ac0fd`가 이 실패 은폐를 제거하고, `035b97ca7c58`–`7fe29f991a9b`가 실제 recent-match 순서에서 연승을 계산하고 범위를 “최근 경기”로 설명하며, `3c6c9134ee94`가 빈 이력에 가짜 그래프를 만들지 않게 했습니다.
- **소유권:** repository는 read model의 계산·정렬·projection을, HTTP route는 public/authenticated 접근 구분을, browser adapter와 React Query는 parsing·cache key·invalidation을, 화면은 loading/error/empty/rendering을 소유합니다. sample data는 더 이상 실패 대체 authority가 아닙니다.
- **Failure → Fix → Test:** 추정 최고 연승 → 실제 chronological scan → W/L/W/W test. 고정 chart → rating delta 역산 → empty history를 별도 상태로 수정. sample fallback → nullable/empty/error 화면. query ownership → helper/key/invalidation test로 연결됩니다.
- **최종 흐름:** repository가 public profile, leaderboard, newest-first recent matches, dashboard를 생성 → API가 공개/인증 route로 노출 → typed browser helper와 query option이 읽음 → mutation은 관련 identity/read-model cache를 무효화 → 화면은 실제 결과 또는 명시적 실패/empty state만 표시합니다.
- **비보장:** 최고 연승과 chart는 repository가 반환한 최근 최대 8경기 범위입니다. 전체 시즌/전체 계정 이력을 의미하지 않습니다. friendship canonical pair/동시성은 category 02 Thread 04가 주 소유자입니다.
- **실행 증거:** exact SHA의 repository·route·web helper·test 구현을 검사했으며 로컬 test runner는 실행하지 않았습니다.
<!-- LEARNER-ANSWER END thread:03-profile-friendship-dashboard-and-ranking-journeys.md:synthesis -->

## 7. 학습 완료 확인

<!-- LEARNER-ANSWER START thread:03-profile-friendship-dashboard-and-ranking-journeys.md:checklist -->
- [x] 모든 SHA를 지정 브랜치의 exact commit으로 검사했습니다.
- [x] fixed commit map과 source classification을 보존했습니다.
- [x] earlier commit을 later HEAD 코드로 설명하지 않았습니다.
- [x] fix와 test를 원래 failure/production path에 연결했습니다.
- [x] 실제 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
- [x] Thread 최종 owner, invariant, flow, non-guarantee를 작성했습니다.
<!-- LEARNER-ANSWER END thread:03-profile-friendship-dashboard-and-ranking-journeys.md:checklist -->
===== END FILE: 03-profile-friendship-dashboard-and-ranking-journeys.md =====

===== BEGIN FILE: 04-lobby-presence-chat-and-live-statistics.md =====
# 로비 접속 상태·채팅과 실시간 지표

원문 Development Thread: `Lobby presence, chat, and live statistics`

## 1. Thread 목표

- 저장되는 로비 채팅과 GameHub가 계산하는 접속·대기열·room 지표의 서로 다른 authority를 추적합니다.
- HTTP 채팅 쓰기, WebSocket 실시간 반영, presence refresh가 web 화면에 결합되는 순서를 복원합니다.
- sample fallback, body 누락, 저장 계정 기반 online 표시, WebSocket open 직후 즉시 가시성 가정을 각각 어떻게 교정했는지 확인합니다.

### Source에서 확정된 significance

> The lobby combines durable chat history with ephemeral process state. Treating the database as presence authority, treating sample data as fallback truth, or assuming immediate propagation after WebSocket open all produce misleading state. The history separates these owners and adapts the tests to eventual visibility.

### 직접 연결되는 Critical Invariants

> Lobby chat history is repository-backed, while online users, queued users, active rooms, and wait samples are derived from the live GameHub process.
>
> UI and tests tolerate asynchronous presence propagation without replacing failed server reads with fabricated data.

### 직접 연결되는 Major Engineering Difficulties

> Merging an initial HTTP snapshot with later WebSocket events without duplicating chat or leaking socket lifecycle.
>
> Testing presence that becomes visible asynchronously while still using deterministic time bounds and explicit failure.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `listLobbyChat`는 최신 20개를 SQL에서 어떤 순서로 읽고 caller에는 어떤 순서로 반환합니까?
- GameHub는 connected/playing/queued/room/wait sample을 어떤 in-memory 자료구조에서 계산합니까?
- HTTP `/chat/lobby`는 body 누락, trim 후 empty, 240자 초과를 각각 어떻게 처리합니까?
- web lobby는 HTTP initial load와 WebSocket `chat.message`/`presence.changed`를 어떻게 합칩니까?
- `onlinePlayers` authority가 repository에서 GameHub로 이동하면서 비접속 저장 계정이 왜 사라집니까?
- smoke test는 WebSocket open과 HTTP presence visibility 사이의 eventual delay를 어떻게 기다립니까?

## 3. 완료 기준

- durable chat와 ephemeral presence/statistics의 owner를 구분해 호출 흐름을 그릴 수 있습니다.
- HTTP chat write와 WebSocket chat delivery가 같은 repository record를 어떻게 공유하는지 설명할 수 있습니다.
- body 없는 요청의 실제 failure 원인과 nullish normalization fix를 지적할 수 있습니다.
- presence test가 즉시 assertion에서 bounded polling으로 바뀐 이유와 비보장을 설명할 수 있습니다.
- Commit map의 모든 SHA를 지정 브랜치 ancestry와 source classification에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 실행한 명령과 코드 검사만으로 확인한 사실을 구분하고 실행하지 않은 test를 통과했다고 기록하지 않습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a6fa5a187eec` | `feat(db): 채팅 메시지 저장 구현` | B | REALTIME, PERSISTENCE | Extends the repository with persistent lobby and match chat messages. |
| 2 | `dabd8d5c2a49` | `feat(game): 실시간 경기 채팅 전달` | B | PROTOCOL, REALTIME, PERSISTENCE | Persists WebSocket chat commands and broadcasts the resulting message. |
| 3 | `1d9aa3902614` | `feat(lobby): 실시간 로비 지표 API 추가` | B | REALTIME | Makes the GameHub expose live connected, playing, queued, room, and recent wait metrics. |
| 4 | `de9a173e6eb1` | `feat(chat): 쓰기 가능한 로비 채팅 API 추가` | B | REALTIME, PERSISTENCE | Adds an authenticated HTTP endpoint for validated lobby chat writes. |
| 5 | `e0ef3fec89a6` | `feat(chat): 로비 채팅 입력 화면 추가` | B | WEB | Adds a controlled lobby-chat form and appends accepted messages to bounded history. |
| 6 | `4f9b3b312d0e` | `fix(lobby): 로비 상태 표현 개선` | B | REALTIME | Displays server-provided lobby statistics instead of fixed activity and wait-time claims. |
| 7 | `8ce1199ffd12` | `fix(api): body 없는 로비 채팅 요청 처리` | B | - | Normalizes a missing lobby-chat request body to an empty object before reading the optional message. |
| 8 | `8078ac6f92ba` | `test(app): 실시간 지표·채팅·경기 기록 검증` | B | REALTIME, PERSISTENCE, WEB | Expands application coverage for lobby metrics, authenticated chat persistence, chat attribution, and recent-match ordering. |
| 9 | `cd3787eefd6a` | `feat(chat): 로비 채팅과 접속 상태 실시간 반영` | B | AUTH, REALTIME, WEB | Opens an authenticated lobby WebSocket and merges chat/presence events with the HTTP snapshot. |
| 10 | `8debb1ea3ad3` | `feat(lobby): 연결 중인 WebSocket 사용자 목록 추가` | B | REALTIME | Makes the realtime hub, not persistent account storage, the authority for lobby presence. |
| 11 | `c3ff9ed2402f` | `test(lobby): WebSocket 사용자 목록 검증` | B | REALTIME, PERSISTENCE, TEST | Verifies that the lobby reports no online players when the realtime hub has no WebSocket clients. |
| 12 | `23a978879b81` | `test(smoke): WebSocket 접속 상태 반영 대기` | B | REALTIME, OPERATIONS, TEST | Changes the smoke test from an immediate presence assertion to bounded asynchronous polling. |

## 5. Commit별 학습 기록

### 5.1. `feat(db): 채팅 메시지 저장 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `a6fa5a187eec` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Extends the repository with persistent lobby and match chat messages.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `createChatMessage`, `listLobbyChat`
- PostgreSQL sender join과 latest-20-desc 후 reverse
- memory chat collection/filter/slice
- `packages/db/src/rowMappers.ts`의 `toChatMessage`
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:a6fa5a187eec -->
- **직전 상태:** chat은 realtime 화면/event 개념만 있고 sender와 scope를 가진 durable history가 없었습니다.
- **구현 결정:** lobby/match scope, optional room, sender ID, body, timestamp를 repository에 저장하고 lobby read는 최신 20개를 고른 뒤 오래된 것부터 보이도록 reverse합니다.
- **상태/소유권 변화:** chat identity/body/history는 repository가 소유하고 realtime hub는 전달자가 됩니다.
- **실패/edge:** 이 시점에는 scope와 room 조합을 강제하지 않아 lobby row에 room이 있거나 match row에 room이 없는 입력을 막지 못합니다.
- **보장/비보장:** sender projection과 bounded lobby history는 제공하지만 authorization과 DB invariant는 Thread 05 전까지 없습니다.
- **다음 연결:** `dabd8d5c2a49`가 WebSocket `chat.send`를 이 저장 경계에 연결합니다.
<!-- LEARNER-ANSWER END commit:a6fa5a187eec -->

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 Thread 관련 SHA: `dabd8d5c2a49` — `feat(game): 실시간 경기 채팅 전달`

### 5.2. `feat(game): 실시간 경기 채팅 전달`

| 항목 | 값 |
| --- | --- |
| SHA | `dabd8d5c2a49` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Persists WebSocket chat commands and broadcasts the resulting message.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 async `receive`
- `chat.send` 분기와 `repo.createChatMessage`
- `broadcastRoom`/`broadcastAll` 선택
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:dabd8d5c2a49 -->
- **직전 상태:** repository chat API는 있었지만 WebSocket client event가 이를 호출하지 않았습니다.
- **구현 결정:** `chat.send`를 받으면 인증된 client user ID로 먼저 message를 저장하고, match면 room, lobby면 모든 연결에 persisted message를 broadcast합니다.
- **상태/소유권 변화:** message ID/timestamp/sender는 repository 결과가 authority이고 hub는 그 결과를 전달합니다.
- **실패/edge:** match branch는 event가 말한 room을 그대로 사용해 current seat/room authorization이 없고 scope-room 조합도 느슨합니다.
- **보장/비보장:** 저장 후 broadcast 순서는 갖지만 cross-room injection 방지는 Thread 05에서 추가됩니다.
- **다음 연결:** `1d9aa3902614`가 같은 GameHub의 live lobby stats를 노출합니다.
<!-- LEARNER-ANSWER END commit:dabd8d5c2a49 -->

비교 기준:
- 직전 Thread 관련 SHA: `a6fa5a187eec` — `feat(db): 채팅 메시지 저장 구현`
- 다음 Thread 관련 SHA: `1d9aa3902614` — `feat(lobby): 실시간 로비 지표 API 추가`

### 5.3. `feat(lobby): 실시간 로비 지표 API 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `1d9aa3902614` |
| Importance | B |
| Tags | REALTIME |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes the GameHub expose live connected, playing, queued, room, and recent wait metrics.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `QueueEntry.queuedAt`, wait sample collection
- `lobbyStats`/live stats getter
- 최근 최대 20개 wait sample과 rounded average seconds
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:1d9aa3902614 -->
- **직전 상태:** lobby는 저장 데이터 중심이라 현재 process의 queue와 room 상태를 수치로 제공하지 못했습니다.
- **구현 결정:** queue entry timestamp를 기록하고 match 시 wait sample을 남겨 connected clients, playing participants, queued clients, active rooms, 평균 대기 시간을 계산합니다.
- **상태/소유권 변화:** 실시간 지표 authority는 database가 아니라 GameHub in-memory maps/queue/wait samples입니다.
- **실패/edge:** sample이 없으면 average wait는 null이며 process restart 시 history는 사라집니다. 장기 통계가 아닙니다.
- **보장/비보장:** 현재 process snapshot과 bounded recent wait 평균만 보장합니다.
- **다음 연결:** `de9a173e6eb1`이 로비 채팅의 HTTP 쓰기 경로를 추가합니다.
<!-- LEARNER-ANSWER END commit:1d9aa3902614 -->

비교 기준:
- 직전 Thread 관련 SHA: `dabd8d5c2a49` — `feat(game): 실시간 경기 채팅 전달`
- 다음 Thread 관련 SHA: `de9a173e6eb1` — `feat(chat): 쓰기 가능한 로비 채팅 API 추가`

### 5.4. `feat(chat): 쓰기 가능한 로비 채팅 API 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `de9a173e6eb1` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds an authenticated HTTP endpoint for validated lobby chat writes.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- API route의 `/chat/lobby` handler
- request body cast, trim, empty/240-char 검사
- `repo.createChatMessage({ scope: "lobby", ... })`
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:de9a173e6eb1 -->
- **직전 상태:** lobby chat write는 WebSocket 경로에만 있고 socket이 없는 client의 HTTP fallback이 없었습니다.
- **구현 결정:** 인증 사용자의 body를 trim하고 empty 또는 240자 초과를 400으로 거부한 뒤 room 없는 lobby message를 저장해 반환합니다.
- **상태/소유권 변화:** HTTP와 WS가 같은 repository message record를 만들 수 있게 됩니다.
- **실패/edge:** `request.body` 자체가 undefined이면 `body.message` 접근 전에 TypeError가 나 intended 400이 아니라 내부 오류가 될 수 있습니다.
- **보장/비보장:** body가 객체인 정상 입력의 validation은 제공하지만 missing-body normalization은 `8ce1199ffd12`에서 수정됩니다.
- **다음 연결:** `e0ef3fec89a6`이 web form과 HTTP fallback을 연결합니다.
<!-- LEARNER-ANSWER END commit:de9a173e6eb1 -->

비교 기준:
- 직전 Thread 관련 SHA: `1d9aa3902614` — `feat(lobby): 실시간 로비 지표 API 추가`
- 다음 Thread 관련 SHA: `e0ef3fec89a6` — `feat(chat): 로비 채팅 입력 화면 추가`

### 5.5. `feat(chat): 로비 채팅 입력 화면 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e0ef3fec89a6` |
| Importance | B |
| Tags | WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds a controlled lobby-chat form and appends accepted messages to bounded history.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 `chatInput`/submit handler
- `sendLobbyChat` 호출
- 최대 20개 history 유지와 당시 sample initial/failure state
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:e0ef3fec89a6 -->
- **직전 상태:** lobby는 chat history를 표시만 하고 사용자가 입력할 form이 없었습니다.
- **구현 결정:** controlled input을 trim해 empty submit을 무시하고 HTTP API 반환 message를 history 뒤에 붙인 뒤 20개로 제한합니다.
- **상태/소유권 변화:** UI는 pending input과 local render list를 소유하지만 message 사실은 server response에서 옵니다.
- **실패/edge:** 당시 initial players/chat는 sample이며 load failure에도 sample 표시 메시지를 사용했습니다.
- **보장/비보장:** HTTP chat journey는 연결하지만 실시간 broadcast와 honest failure state는 후속 commit이 담당합니다.
- **다음 연결:** `4f9b3b312d0e`이 server stats를 화면에 반영하고 `cd3787eefd6a`가 WebSocket을 연결합니다.
<!-- LEARNER-ANSWER END commit:e0ef3fec89a6 -->

비교 기준:
- 직전 Thread 관련 SHA: `de9a173e6eb1` — `feat(chat): 쓰기 가능한 로비 채팅 API 추가`
- 다음 Thread 관련 SHA: `4f9b3b312d0e` — `fix(lobby): 로비 상태 표현 개선`

### 5.6. `fix(lobby): 로비 상태 표현 개선`

| 항목 | 값 |
| --- | --- |
| SHA | `4f9b3b312d0e` |
| Importance | B |
| Tags | REALTIME |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Displays server-provided lobby statistics instead of fixed activity and wait-time claims.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 lobby stats cards
- online/playing/queued/active room count
- averageWaitSeconds null 처리와 고정 30초/주간 문구 제거
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:4f9b3b312d0e -->
- **직전 상태:** lobby는 server stats가 있어도 일부 고정 수치·“30초 미만” 같은 근거 없는 문구를 표시했습니다.
- **구현 결정:** API가 준 connected/playing/queued/room/average wait를 직접 표시하고 값이 없으면 측정 전 상태를 보여 줍니다.
- **상태/소유권 변화:** live metric 값은 GameHub가, 화면은 format만 소유합니다.
- **실패/edge:** 이 시점에도 sample initial state가 남아 request failure truthfulness는 완전하지 않습니다.
- **보장/비보장:** 성공 응답 시 고정 통계 fabrication을 제거합니다.
- **다음 연결:** `8ce1199ffd12`가 HTTP missing-body bug를 고치고 `8078ac6f92ba`가 지표/채팅 저장을 테스트합니다.
<!-- LEARNER-ANSWER END commit:4f9b3b312d0e -->

비교 기준:
- 직전 Thread 관련 SHA: `e0ef3fec89a6` — `feat(chat): 로비 채팅 입력 화면 추가`
- 다음 Thread 관련 SHA: `8ce1199ffd12` — `fix(api): body 없는 로비 채팅 요청 처리`

### 5.7. `fix(api): body 없는 로비 채팅 요청 처리`

| 항목 | 값 |
| --- | --- |
| SHA | `8ce1199ffd12` |
| Importance | B |
| Tags | - |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Normalizes a missing lobby-chat request body to an empty object before reading the optional message.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- 로비 chat route의 `(request.body ?? {})` destructuring
- trim/empty 400 validation path
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:8ce1199ffd12 -->
- **직전 가정:** JSON request에는 항상 object body가 있다고 봤습니다.
- **실제 실패:** body가 없으면 validation 전에 property access TypeError가 나 400 계약을 우회했습니다.
- **교정:** nullish body를 빈 객체로 정규화해 `message`가 undefined가 되고 기존 trim/empty 검사로 400을 반환하게 합니다.
- **보장/비보장:** missing body가 의도한 client error로 수렴하지만 content-type parser 전체 동작을 바꾸지는 않습니다.
- **다음 연결:** `8078ac6f92ba`의 application tests가 chat write/read와 lobby contract를 함께 검증합니다.
<!-- LEARNER-ANSWER END commit:8ce1199ffd12 -->

비교 기준:
- 직전 Thread 관련 SHA: `4f9b3b312d0e` — `fix(lobby): 로비 상태 표현 개선`
- 다음 Thread 관련 SHA: `8078ac6f92ba` — `test(app): 실시간 지표·채팅·경기 기록 검증`

### 5.8. `test(app): 실시간 지표·채팅·경기 기록 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `8078ac6f92ba` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Expands application coverage for lobby metrics, authenticated chat persistence, chat attribution, and recent-match ordering.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- API injection/e2e test의 초기 lobby stats
- authenticated `/chat/lobby` write-to-read
- memory repository chat sender/room 및 recent match 시간순 assertions
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:8078ac6f92ba -->
- **직전 상태:** stats/chat 기능은 여러 layer에 걸쳤지만 초기 contract와 저장 결과를 연결한 regression coverage가 부족했습니다.
- **기법:** 초기 hub의 zero/empty stats를 확인하고 dev login 후 HTTP lobby chat을 쓰고 다시 읽습니다. memory repository에서는 lobby/match message의 sender·room attribution과 recent-match ordering을 검사합니다.
- **생산 경로:** API routing/auth/repository와 memory domain projection을 통과합니다.
- **증명/비증명:** 기능 조합을 검증하도록 작성됐지만 WebSocket presence의 eventual visibility와 cross-room auth는 별도 tests가 담당합니다.
- **보장:** 로비 read/write contract와 ordering regression을 넓게 감지합니다.
- **다음 연결:** `cd3787eefd6a`가 browser lobby를 WebSocket event에 연결합니다.
<!-- LEARNER-ANSWER END commit:8078ac6f92ba -->

비교 기준:
- 직전 Thread 관련 SHA: `8ce1199ffd12` — `fix(api): body 없는 로비 채팅 요청 처리`
- 다음 Thread 관련 SHA: `cd3787eefd6a` — `feat(chat): 로비 채팅과 접속 상태 실시간 반영`

### 5.9. `feat(chat): 로비 채팅과 접속 상태 실시간 반영`

| 항목 | 값 |
| --- | --- |
| SHA | `cd3787eefd6a` |
| Importance | B |
| Tags | AUTH, REALTIME, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Opens an authenticated lobby WebSocket and merges chat/presence events with the HTTP snapshot.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 user/token 준비 후 WebSocket open
- `chat.message` ID dedupe와 append
- `presence.changed` 시 `loadLobby`
- unmount/identity change socket cleanup 및 HTTP fallback
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:cd3787eefd6a -->
- **직전 상태:** lobby는 HTTP load/write만 사용해 다른 사용자의 chat과 presence 변화를 새로고침 없이 보지 못했습니다.
- **구현 결정:** 현재 user와 token이 준비되면 socket을 열고 chat event는 ID 중복을 제거해 병합하며 presence event는 authoritative HTTP lobby를 다시 읽습니다. socket이 열리지 않으면 chat submit은 HTTP로 fallback합니다.
- **상태/소유권 변화:** socket lifecycle은 page effect가, durable message는 repository가, presence snapshot은 GameHub/API가 소유합니다.
- **실패/edge:** 당시 token query 연결 방식 자체의 보안 개선은 다른 category에 속합니다. reconnect/backoff도 이 commit의 핵심은 아닙니다.
- **보장/비보장:** 단일 page lifetime의 실시간 반영과 cleanup은 제공하지만 presence source는 아직 repository 기반일 수 있습니다.
- **다음 연결:** `8debb1ea3ad3`이 `/lobby.onlinePlayers`를 hub-connected clients로 바꿉니다.
<!-- LEARNER-ANSWER END commit:cd3787eefd6a -->

비교 기준:
- 직전 Thread 관련 SHA: `8078ac6f92ba` — `test(app): 실시간 지표·채팅·경기 기록 검증`
- 다음 Thread 관련 SHA: `8debb1ea3ad3` — `feat(lobby): 연결 중인 WebSocket 사용자 목록 추가`

### 5.10. `feat(lobby): 연결 중인 WebSocket 사용자 목록 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `8debb1ea3ad3` |
| Importance | B |
| Tags | REALTIME |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes the realtime hub, not persistent account storage, the authority for lobby presence.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `onlinePlayers`/client map projection
- user ID dedupe, `online: true`, rating/name sort
- `/lobby` route의 repository `listOnlineUsers` 대체
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:8debb1ea3ad3 -->
- **직전 상태:** `/lobby`는 repository의 active user 목록을 online처럼 반환해 실제 socket 연결이 없는 저장 계정도 접속자로 보였습니다.
- **구현 결정:** GameHub client map에서 user ID를 dedupe하고 online true projection을 만든 뒤 rating/name으로 정렬해 route가 사용합니다.
- **상태/소유권 변화:** account 활성 상태는 DB가, 현재 접속 presence는 GameHub가 소유하도록 분리됩니다.
- **실패/edge:** 한 process의 client map만 보므로 다중 instance 전체 presence aggregation은 보장하지 않습니다.
- **보장/비보장:** 현재 hub에 연결된 사용자만 반환합니다.
- **다음 연결:** `c3ff9ed2402f`가 socket이 없을 때 empty list를 요구합니다.
<!-- LEARNER-ANSWER END commit:8debb1ea3ad3 -->

비교 기준:
- 직전 Thread 관련 SHA: `cd3787eefd6a` — `feat(chat): 로비 채팅과 접속 상태 실시간 반영`
- 다음 Thread 관련 SHA: `c3ff9ed2402f` — `test(lobby): WebSocket 사용자 목록 검증`

### 5.11. `test(lobby): WebSocket 사용자 목록 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `c3ff9ed2402f` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies that the lobby reports no online players when the realtime hub has no WebSocket clients.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- API route/injection test의 empty hub setup
- `/lobby` response `onlinePlayers: []` assertion
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:c3ff9ed2402f -->
- **직전 상태:** authority 전환은 구현됐지만 seed/active users가 다시 online list에 섞이는 regression을 막을 test가 없었습니다.
- **기법:** repository seed는 존재하되 WebSocket client를 연결하지 않은 app에서 `/lobby`를 조회해 empty online list를 요구합니다.
- **생산 경로:** route가 repository가 아니라 hub projection을 호출하는지를 간접 검증합니다.
- **증명/비증명:** no-connection case를 고정하지만 connect/disconnect propagation timing은 smoke test가 담당합니다.
- **보장:** 저장 계정 존재만으로 online이 되는 regression을 감지합니다.
- **다음 연결:** `23a978879b81`이 실제 socket open 후 presence가 eventual하게 반영되는 시간을 기다립니다.
<!-- LEARNER-ANSWER END commit:c3ff9ed2402f -->

비교 기준:
- 직전 Thread 관련 SHA: `8debb1ea3ad3` — `feat(lobby): 연결 중인 WebSocket 사용자 목록 추가`
- 다음 Thread 관련 SHA: `23a978879b81` — `test(smoke): WebSocket 접속 상태 반영 대기`

### 5.12. `test(smoke): WebSocket 접속 상태 반영 대기`

| 항목 | 값 |
| --- | --- |
| SHA | `23a978879b81` |
| Importance | B |
| Tags | REALTIME, OPERATIONS, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Changes the smoke test from an immediate presence assertion to bounded asynchronous polling.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `tests/smoke-ws.mjs`의 async `waitFor`
- 두 socket open 뒤 `/lobby` polling
- 10초 timeout, 50ms `delay`, predicate가 두 handle을 포함할 때 종료
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:23a978879b81 -->
- **직전 가정:** WebSocket `open` promise가 resolve되면 HTTP `/lobby`도 같은 순간 두 사용자를 반드시 포함한다고 봤습니다.
- **실제 실패:** socket open event와 server-side connect/presence publication, 별도 HTTP read 사이에는 scheduling delay가 있어 정상 시스템도 flaky할 수 있습니다.
- **교정/기법:** async predicate를 최대 10초 동안 50ms 간격으로 호출하고 두 handle이 모두 보일 때만 다음 smoke 단계로 진행합니다. timeout이면 명시적으로 실패합니다.
- **생산 경로:** 실제 WebSocket 연결과 HTTP lobby read를 함께 사용하도록 작성됐습니다.
- **증명/비증명:** bounded eventual visibility를 검증하지만 즉시 consistency나 10초 이내라는 제품 SLA를 공식 보장하지는 않습니다.
- **Thread 종료:** durable chat와 live presence의 서로 다른 authority 및 asynchronous observation 모델이 테스트에도 반영됩니다.
<!-- LEARNER-ANSWER END commit:23a978879b81 -->

비교 기준:
- 직전 Thread 관련 SHA: `c3ff9ed2402f` — `test(lobby): WebSocket 사용자 목록 검증`
- 이 commit을 Thread의 마지막 상태로 사용합니다.

## 6. Thread 종합

다음 항목을 commit 순서에서 재구성합니다: invariant evolution, Failure → Fix → Test 관계, ownership/state 변화, 최종 실행 흐름, 비보장, 실제 실행 증거.

<!-- LEARNER-ANSWER START thread:04-lobby-presence-chat-and-live-statistics.md:synthesis -->
- **불변식 진화:** `a6fa5a187eec`은 채팅을 repository에 저장했고 `dabd8d5c2a49`는 WebSocket send를 저장 후 broadcast했습니다. `1d9aa3902614`는 connected/playing/queued/room/wait를 GameHub에서 계산했습니다. 초기 UI는 sample 상태와 repository online projection을 섞었지만 `be31566ac0fd`의 sample 제거, `8debb1ea3ad3`의 GameHub presence authority 전환, `c3ff9ed2402f`의 empty presence test로 정리됐습니다.
- **소유권:** repository는 chat rows와 sender projection을 소유합니다. GameHub는 live client/queue/room/wait samples를 소유합니다. API `/lobby`는 두 source를 조합하고 web은 HTTP snapshot에 WebSocket event를 병합합니다.
- **Failure → Fix → Test:** missing body에서 property access 예외 → `(request.body ?? {})` → intended 400 경로. 저장 사용자 목록을 online으로 표시 → `hub.onlinePlayers()` → no-socket empty test. WebSocket open 직후 presence 즉시 조회 → 최대 10초/50ms async polling smoke test.
- **최종 흐름:** HTTP `/lobby`가 최근 chat와 hub stats/presence를 반환 → 로그인 사용자는 ticket/token 기반 WebSocket 연결 → lobby chat은 socket이 열리면 WS, 아니면 HTTP write → repository가 message를 만들고 hub가 broadcast → `presence.changed`는 HTTP lobby refresh를 유도 → message ID로 중복을 제거합니다.
- **비보장:** 이 Thread는 match chat의 scope/room 조합과 cross-room authorization을 완성하지 않습니다. 그 보안 invariant는 Thread 05가 소유합니다. Presence는 process-local이며 WebSocket open 순간과 HTTP read 순간 사이에 eventual delay가 있을 수 있습니다.
- **실행 증거:** repository/API/web/smoke test 구현은 검사했지만 서비스를 실행하지 않았습니다.
<!-- LEARNER-ANSWER END thread:04-lobby-presence-chat-and-live-statistics.md:synthesis -->

## 7. 학습 완료 확인

<!-- LEARNER-ANSWER START thread:04-lobby-presence-chat-and-live-statistics.md:checklist -->
- [x] 모든 SHA를 지정 브랜치의 exact commit으로 검사했습니다.
- [x] fixed commit map과 source classification을 보존했습니다.
- [x] earlier commit을 later HEAD 코드로 설명하지 않았습니다.
- [x] fix와 test를 원래 failure/production path에 연결했습니다.
- [x] 실제 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
- [x] Thread 최종 owner, invariant, flow, non-guarantee를 작성했습니다.
<!-- LEARNER-ANSWER END thread:04-lobby-presence-chat-and-live-statistics.md:checklist -->
===== END FILE: 04-lobby-presence-chat-and-live-statistics.md =====

===== BEGIN FILE: 05-chat-scope-storage-and-room-authorization.md =====
# 채팅 scope 저장 불변식과 경기방 권한

원문 Development Thread: `Chat scope storage and room authorization`

## 1. Thread 목표

- lobby와 match chat의 scope/room 조합을 wire schema, repository validation, migration/DB CHECK에서 같은 규칙으로 강제하는 과정을 추적합니다.
- match chat을 보내는 client가 실제 현재 room 좌석인지 저장 전 검사하고 audience를 해당 room으로 제한하는 경로를 복원합니다.
- 브라우저가 현재 active room과 정확히 일치하는 match message만 받아들이는 마지막 방어선을 확인합니다.

### Source에서 확정된 significance

> Chat scope is a security and data-integrity boundary, not a display label. A syntactically valid room ID does not prove membership, and server broadcast discipline does not eliminate the need for storage constraints or client filtering. The history layers the invariant across wire, repository, database, hub authorization, and browser state.

### 직접 연결되는 Critical Invariants

> Lobby chat has no room; match chat has a non-null UUID room. Invalid combinations are rejected before or at persistence and legacy rows are cleaned before the constraint is installed.
>
> A match message is persisted and broadcast only when the sender currently occupies a seat in that exact room; a client reducer accepts only messages for its active room.

### 직접 연결되는 Major Engineering Difficulties

> Expressing a discriminated payload rule in JSON schema/TypeScript, application validation, and SQL CHECK without allowing different edge cases.
>
> Proving rejection occurs before persistence and proving normal delivery is limited to one room while lobby delivery remains global.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- wire schema는 lobby payload의 `roomId` 부재와 match payload의 UUID를 어떤 discriminated union으로 표현합니까?
- migration 006은 기존 lobby/match/unknown row를 어떤 순서로 정규화·삭제한 뒤 CHECK를 설치합니까?
- `assertChatRoom`이 PostgreSQL과 memory 양쪽에 필요한 이유는 무엇입니까?
- GameHub는 room 존재, `client.roomId`, `sideFor`를 저장 호출 전에 어떤 순서로 검사합니까?
- cross-room test는 persistence spy, error event, audience event를 어떻게 함께 관찰합니까?
- `isChatForActiveRoom`은 다른 room, lobby scope, active room 없음 각각을 어떻게 처리합니까?

## 3. 완료 기준

- wire→repository→database의 scope-room invariant를 허용/거부 표로 정리할 수 있습니다.
- storage validity와 sender authorization이 다른 문제인 이유를 설명할 수 있습니다.
- cross-room 주입이 persistence 전에 멈추고 정상 match/lobby delivery가 다른 audience를 갖는 근거를 제시할 수 있습니다.
- client filter가 server authorization을 대체하지 않지만 stale/misrouted event의 UI 오염을 막는 이유를 설명할 수 있습니다.
- Commit map의 모든 SHA를 지정 브랜치 ancestry와 source classification에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 실행한 명령과 코드 검사만으로 확인한 사실을 구분하고 실행하지 않은 test를 통과했다고 기록하지 않습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `00d0d7941382` | `fix(protocol): 채팅 scope와 room 식별자 조합 제한` | A | PROTOCOL, REALTIME, WEB | Defines scope-specific chat payloads so lobby messages cannot identify a room and match messages require a UUID room. |
| 2 | `5a3819aec8d0` | `test(protocol): 채팅 scope와 room 조합 검증` | B | AUTH, PROTOCOL, REALTIME | Pins valid scope-room combinations at the WebSocket parse boundary. |
| 3 | `2ff750fa4ff8` | `fix(db): 채팅 행의 scope와 room 불변식 강제` | A | REALTIME, PERSISTENCE, RISK | Cleans legacy rows and enforces the chat scope-room invariant in both repository implementations and PostgreSQL. |
| 4 | `1cead7cc9f35` | `test(db): 채팅 저장 불변식 검증` | B | REALTIME, PERSISTENCE, TEST | Verifies repository rejection, migration cleanup, SQL constraint enforcement, and migration idempotency. |
| 5 | `7759eef59b67` | `fix(game): 매치 채팅의 좌석과 audience 검증` | A | REALTIME, PERSISTENCE, RISK | Rejects match chat unless the sender currently occupies a seat in that exact room, before persistence. |
| 6 | `4a98bd1e4f22` | `test(game): 타 경기방 채팅 주입 차단 검증` | B | REALTIME, TEST | Verifies cross-room rejection before persistence and room-scoped versus global delivery. |
| 7 | `85edd6d1e26a` | `fix(web): 현재 경기방의 채팅만 표시` | B | REALTIME, WEB | Filters incoming chat so the game reducer accepts only match messages for the current active room. |
| 8 | `02775797ab63` | `test(web): 매치 채팅 room filtering 검증` | B | REALTIME, WEB, TEST | Pins the pure active-room chat filter. |

## 5. Commit별 학습 기록

### 5.1. `fix(protocol): 채팅 scope와 room 식별자 조합 제한`

| 항목 | 값 |
| --- | --- |
| SHA | `00d0d7941382` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, WEB |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Defines scope-specific chat payloads so lobby messages cannot identify a room and match messages require a UUID room.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/ws.ts`의 `chat.send` discriminated variants
- lobby strict object와 match `roomId: z.string().uuid()`
- body trim/min/max 규칙
- GameHub/web caller payload 수정
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:00d0d7941382 -->
- **직전 상태:** 하나의 chat event가 `scope`와 optional `roomId`를 독립 필드로 가져 lobby+room, match+missing room 같은 조합이 타입/parse를 통과할 수 있었습니다.
- **구현 결정:** `scope: "lobby"` variant는 room 필드를 갖지 않는 strict object로, `scope: "match"` variant는 UUID room을 필수로 하는 discriminated union으로 분리합니다. body는 trim 후 1–240자입니다.
- **상태/소유권 변화:** wire contract가 허용 상태 공간을 줄이고 caller는 lobby에 room을 보내지 않으며 GameHub는 저장 시 null로 정규화합니다.
- **실패/edge:** UUID 형식은 room 존재나 sender membership을 증명하지 않습니다. 저장 layer를 직접 호출하는 코드도 protocol parser를 거치지 않을 수 있습니다.
- **보장/비보장:** parse된 payload의 scope-room 형태는 보장하지만 storage/auth는 후속 commits가 담당합니다.
- **다음 연결:** `5a3819aec8d0`이 모든 invalid 조합을 table-driven test로 고정합니다.
<!-- LEARNER-ANSWER END commit:00d0d7941382 -->

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 Thread 관련 SHA: `5a3819aec8d0` — `test(protocol): 채팅 scope와 room 조합 검증`

### 5.2. `test(protocol): 채팅 scope와 room 조합 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `5a3819aec8d0` |
| Importance | B |
| Tags | AUTH, PROTOCOL, REALTIME |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Pins valid scope-room combinations at the WebSocket parse boundary.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/ws.test.ts`의 `it.each` invalid payload table
- lobby null/present room, match missing/null/non-UUID cases
- `parseClientEvent(JSON.stringify(payload))` rejection
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:5a3819aec8d0 -->
- **직전 상태:** schema는 분리됐지만 optional/unknown-key behavior가 바뀌어 invalid 조합이 다시 허용돼도 감지할 test가 없었습니다.
- **기법:** lobby에 null room, lobby에 UUID room, match에 room 없음/null/non-UUID를 각각 serialize해 parser가 throw하는지 확인합니다.
- **생산 경로:** 실제 client-event parser와 Zod schema를 사용합니다.
- **증명/비증명:** wire rejection을 검증하지만 repository direct call, DB row, current room membership은 검증하지 않습니다.
- **보장:** protocol layer의 허용 상태 공간을 고정합니다.
- **다음 연결:** `2ff750fa4ff8`이 같은 invariant를 저장 경계와 기존 데이터에 적용합니다.
<!-- LEARNER-ANSWER END commit:5a3819aec8d0 -->

비교 기준:
- 직전 Thread 관련 SHA: `00d0d7941382` — `fix(protocol): 채팅 scope와 room 식별자 조합 제한`
- 다음 Thread 관련 SHA: `2ff750fa4ff8` — `fix(db): 채팅 행의 scope와 room 불변식 강제`

### 5.3. `fix(db): 채팅 행의 scope와 room 불변식 강제`

| 항목 | 값 |
| --- | --- |
| SHA | `2ff750fa4ff8` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, RISK |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Cleans legacy rows and enforces the chat scope-room invariant in both repository implementations and PostgreSQL.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/migrations/006_chat_invariants.sql`의 update/delete/check 순서
- `packages/db/src/index.ts`의 `assertChatRoom`
- PostgreSQL/memory `createChatMessage` 호출 위치
- `chat_messages_scope_room_check` UUID regex
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:2ff750fa4ff8 -->
- **직전 가정:** 모든 chat write가 protocol parser를 통과하므로 DB에는 올바른 조합만 들어간다고 봤습니다.
- **실제 위험:** HTTP/내부 repository caller나 기존 데이터는 lobby+room, match+null/non-UUID, unknown scope를 저장할 수 있었습니다.
- **교정:** migration은 lobby row의 room을 null로 정규화하고 unknown scope 및 invalid match rows를 삭제한 뒤 `(lobby and room is null) or (match and UUID room)` CHECK를 설치합니다. 양 repository는 SQL 전 `assertChatRoom`을 호출합니다.
- **상태/소유권 변화:** application과 DB가 같은 invariant를 독립적으로 강제합니다. migration은 constraint 설치가 실패하지 않도록 기존 데이터의 운명도 명시합니다.
- **보장/비보장:** 공식 repository와 direct SQL 모두 invalid row를 거부하지만 sender가 그 room에 속하는지는 저장 형태와 별도 authorization 문제입니다.
- **다음 연결:** `1cead7cc9f35`가 memory rejection, legacy cleanup, migration idempotency, DB CHECK를 실제 test로 묶습니다.
<!-- LEARNER-ANSWER END commit:2ff750fa4ff8 -->

비교 기준:
- 직전 Thread 관련 SHA: `5a3819aec8d0` — `test(protocol): 채팅 scope와 room 조합 검증`
- 다음 Thread 관련 SHA: `1cead7cc9f35` — `test(db): 채팅 저장 불변식 검증`

### 5.4. `test(db): 채팅 저장 불변식 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1cead7cc9f35` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies repository rejection, migration cleanup, SQL constraint enforcement, and migration idempotency.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 invalid lobby/match cases
- `packages/db/src/postgres.integration.test.ts`의 005→006 legacy fixture
- migrate twice, expected surviving rows, SQLSTATE `23514`, migration count 1
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:1cead7cc9f35 -->
- **직전 상태:** 저장 invariant는 구현됐지만 legacy row 정리와 CHECK가 실제 PostgreSQL에서 의도대로 함께 작동한다는 근거가 없었습니다.
- **기법:** memory repository에 세 invalid 조합을 직접 전달해 reject를 확인합니다. PostgreSQL은 migration 005 상태에 good/bad rows를 넣고 006을 두 번 실행한 뒤 정규화된 lobby 2개와 valid match 1개만 남는지 확인합니다. 이후 invalid direct INSERT가 `23514`인지, migration record가 한 번인지 검사합니다.
- **생산 경로:** repository validation과 실제 migration/constraint를 모두 통과합니다.
- **증명/비증명:** stored-row invariant와 migration 재실행 안전성을 검증하지만 room membership authorization은 검증하지 않습니다.
- **보장:** protocol을 우회한 write와 legacy 오염에 대한 방어를 회귀로 고정합니다.
- **다음 연결:** `7759eef59b67`이 유효한 UUID room을 악용한 cross-room sender를 차단합니다.
<!-- LEARNER-ANSWER END commit:1cead7cc9f35 -->

비교 기준:
- 직전 Thread 관련 SHA: `2ff750fa4ff8` — `fix(db): 채팅 행의 scope와 room 불변식 강제`
- 다음 Thread 관련 SHA: `7759eef59b67` — `fix(game): 매치 채팅의 좌석과 audience 검증`

### 5.5. `fix(game): 매치 채팅의 좌석과 audience 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `7759eef59b67` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, RISK |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Rejects match chat unless the sender currently occupies a seat in that exact room, before persistence.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `chat.send` match branch
- `rooms.get`, `client.roomId`, `sideFor` authorization
- `forbidden` error와 early return
- lobby null-room write와 global broadcast
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:7759eef59b67 -->
- **직전 가정:** parser가 UUID를 요구하고 repository가 valid row를 저장하면 match chat은 안전하다고 봤습니다.
- **실제 공격/오류:** 다른 active room의 UUID를 아는 사용자가 자신의 room이 아닌 곳에 message를 저장·broadcast할 수 있었습니다.
- **교정:** event room을 조회하고 room 존재, `client.roomId === room.id`, `sideFor(room, client)`를 모두 만족할 때만 `createChatMessage`를 호출합니다. 실패는 `forbidden`을 sender에게 보내고 return합니다.
- **상태/소유권 변화:** current seat membership은 GameHub의 room/client state가 authority이며 repository는 형태만 검증합니다. match audience는 `broadcastRoom`, lobby audience는 `broadcastAll`입니다.
- **보장/비보장:** cross-room write와 delivery를 server에서 막지만 browser가 stale room event를 local state에 넣는 문제는 별도입니다.
- **다음 연결:** `4a98bd1e4f22`가 rejection-before-persistence와 audience 범위를 두 room으로 검증합니다.
<!-- LEARNER-ANSWER END commit:7759eef59b67 -->

비교 기준:
- 직전 Thread 관련 SHA: `1cead7cc9f35` — `test(db): 채팅 저장 불변식 검증`
- 다음 Thread 관련 SHA: `4a98bd1e4f22` — `test(game): 타 경기방 채팅 주입 차단 검증`

### 5.6. `test(game): 타 경기방 채팅 주입 차단 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4a98bd1e4f22` |
| Importance | B |
| Tags | REALTIME, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies cross-room rejection before persistence and room-scoped versus global delivery.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.chat.test.ts`의 two-room fake socket setup
- `createChatMessage` spy
- cross-room forbidden/no-call/no-chat assertions
- normal room A only 및 lobby all-client cases
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:4a98bd1e4f22 -->
- **직전 상태:** authorization fix는 있었지만 저장 전 차단인지, 정상 room audience가 다른 room과 분리되는지, lobby가 여전히 global인지 함께 증명한 test가 없었습니다.
- **실패 주입:** room B 좌석의 client가 room A UUID로 match message를 보냅니다.
- **관찰:** room IDs가 다름, repository spy 미호출, sender의 forbidden, 전체 socket의 chat event 0을 확인합니다. 정상 room A message는 A의 두 socket만 받고 B는 0, lobby message는 null room으로 저장되고 모든 socket이 받습니다.
- **생산 경로:** 실제 GameHub queue pairing, room creation, event parser/handler/broadcast를 fake WebSocket으로 실행합니다.
- **증명/비증명:** server authorization/audience를 결정적으로 검증하지만 browser reducer filtering은 포함하지 않습니다.
- **다음 연결:** `85edd6d1e26a`가 client active-room filter를 추가합니다.
<!-- LEARNER-ANSWER END commit:4a98bd1e4f22 -->

비교 기준:
- 직전 Thread 관련 SHA: `7759eef59b67` — `fix(game): 매치 채팅의 좌석과 audience 검증`
- 다음 Thread 관련 SHA: `85edd6d1e26a` — `fix(web): 현재 경기방의 채팅만 표시`

### 5.7. `fix(web): 현재 경기방의 채팅만 표시`

| 항목 | 값 |
| --- | --- |
| SHA | `85edd6d1e26a` |
| Importance | B |
| Tags | REALTIME, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Filters incoming chat so the game reducer accepts only match messages for the current active room.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/chatScope.ts`의 `isChatForActiveRoom`
- `apps/web/src/game/useGameConnection.ts`의 `chat.message` 분기
- dispatch 전 `stateRef.current.roomId` 비교
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:85edd6d1e26a -->
- **직전 상태:** server가 room broadcast를 지켜도 reconnect/race/misrouted event가 들어오면 hook이 모든 `chat.message`를 현재 panel에 넣었습니다.
- **구현 결정:** active room이 non-null이고 message scope가 match이며 `message.roomId === activeRoomId`일 때만 reducer dispatch를 허용하는 pure function을 추가했습니다.
- **상태/소유권 변화:** browser connection state가 현재 화면 audience의 마지막 수용 기준이 됩니다.
- **실패/edge:** client filter는 보안 경계가 아니며 악성 sender의 저장을 막지 못합니다. lobby message도 game panel에서는 의도적으로 버립니다.
- **보장/비보장:** 현재 room panel의 local contamination을 막지만 server auth를 대체하지 않습니다.
- **다음 연결:** `02775797ab63`이 current/other/lobby/no-active 네 조합을 고정합니다.
<!-- LEARNER-ANSWER END commit:85edd6d1e26a -->

비교 기준:
- 직전 Thread 관련 SHA: `4a98bd1e4f22` — `test(game): 타 경기방 채팅 주입 차단 검증`
- 다음 Thread 관련 SHA: `02775797ab63` — `test(web): 매치 채팅 room filtering 검증`

### 5.8. `test(web): 매치 채팅 room filtering 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `02775797ab63` |
| Importance | B |
| Tags | REALTIME, WEB, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Pins the pure active-room chat filter.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/game/chatScope.test.ts`
- current match true
- other match, lobby, no active room false
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:02775797ab63 -->
- **직전 상태:** pure filter는 있었지만 조건 하나가 느슨해지는 regression을 막을 test가 없었습니다.
- **기법:** 고정 UUID 두 개와 helper로 만든 `ChatMessage`를 사용해 현재 match만 true이고 다른 room/lobby/active null은 false인지 검사합니다.
- **생산 경로:** network 없이 production pure predicate를 직접 실행합니다.
- **증명/비증명:** browser 수용 규칙을 정확히 검증하지만 server broadcast/authorization과 storage는 별도 tests가 담당합니다.
- **보장:** scope·room·active state 세 조건을 모두 요구합니다.
- **Thread 종료:** wire, storage, authorization, audience, browser state가 같은 room semantics를 각 책임 범위에서 강제합니다.
<!-- LEARNER-ANSWER END commit:02775797ab63 -->

비교 기준:
- 직전 Thread 관련 SHA: `85edd6d1e26a` — `fix(web): 현재 경기방의 채팅만 표시`
- 이 commit을 Thread의 마지막 상태로 사용합니다.

## 6. Thread 종합

다음 항목을 commit 순서에서 재구성합니다: invariant evolution, Failure → Fix → Test 관계, ownership/state 변화, 최종 실행 흐름, 비보장, 실제 실행 증거.

<!-- LEARNER-ANSWER START thread:05-chat-scope-storage-and-room-authorization.md:synthesis -->
- **불변식 진화:** 초기 chat contract는 `scope`와 optional room의 잘못된 조합을 표현할 수 있었습니다. `00d0d7941382`가 wire union을 lobby/no-room과 match/UUID-room으로 분리하고 `5a3819aec8d0`이 반례를 고정했습니다. `2ff750fa4ff8`은 같은 규칙을 repository와 DB CHECK로 내리고 legacy rows를 정리했으며 `1cead7cc9f35`가 memory/실제 migration을 검증했습니다. `7759eef59b67`은 syntactically valid room ID만으로 부족하므로 실제 seat membership을 저장 전에 확인합니다. web은 `85edd6d1e26a`에서 active room filter를 추가합니다.
- **소유권:** shared protocol은 wire shape, repository/DB는 stored row validity, GameHub는 current room membership과 broadcast audience, browser connection state는 현재 화면에 수용할 room을 소유합니다. 어느 한 layer도 다른 layer의 책임을 대신하지 않습니다.
- **Failure → Fix → Test:** invalid scope-room payload → discriminated schema → table-driven negative test. legacy/우회 저장 → migration + `assertChatRoom` + CHECK → cleanup/idempotent migration/SQLSTATE 23514 test. cross-room sender → seat check before persistence → two-room fake-socket test. stale/misrouted client event → pure room filter → four-case unit test.
- **최종 흐름:** parser가 payload 조합을 검증 → GameHub가 current room/seat를 authorization → repository가 동일 invariant를 다시 검사 → DB CHECK가 direct/legacy invalid row를 거부 → 성공 record만 해당 room audience에 broadcast → browser가 active room 일치 여부를 확인 후 reducer에 넣습니다. lobby는 room null로 저장하고 global audience에 전달됩니다.
- **비보장:** 이 Thread는 message content moderation, rate limiting, end-to-end encryption, multi-process room routing을 제공하지 않습니다.
- **실행 증거:** exact test/migration code를 검사했지만 Vitest/PostgreSQL integration을 실행하지 않았습니다.
<!-- LEARNER-ANSWER END thread:05-chat-scope-storage-and-room-authorization.md:synthesis -->

## 7. 학습 완료 확인

<!-- LEARNER-ANSWER START thread:05-chat-scope-storage-and-room-authorization.md:checklist -->
- [x] 모든 SHA를 지정 브랜치의 exact commit으로 검사했습니다.
- [x] fixed commit map과 source classification을 보존했습니다.
- [x] earlier commit을 later HEAD 코드로 설명하지 않았습니다.
- [x] fix와 test를 원래 failure/production path에 연결했습니다.
- [x] 실제 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
- [x] Thread 최종 owner, invariant, flow, non-guarantee를 작성했습니다.
<!-- LEARNER-ANSWER END thread:05-chat-scope-storage-and-room-authorization.md:checklist -->
===== END FILE: 05-chat-scope-storage-and-room-authorization.md =====

===== BEGIN FILE: 06-pause-resume-and-input-neutralization.md =====
# 일시정지·재개와 입력 무효화

원문 Development Thread: `Pause, resume, and input neutralization`

## 1. Thread 목표

- pause/resume를 화면 효과가 아니라 server-owned game phase transition으로 추가하는 과정을 추적합니다.
- timer/scheduler 중단·재등록과 participant/phase authorization을 복원합니다.
- pause 직전 paddle direction이 resume 뒤 남는 결함을 snapshot과 simulation 양쪽에서 어떻게 neutralize했는지 확인합니다.

### Source에서 확정된 significance

> Stopping ticks is not enough to pause a stateful simulation. Input intent can outlive the timer and become active again on resume. The history first introduces the protocol and server transition, then corrects the hidden input state and protects the boundary with fake-time regression coverage.

### 직접 연결되는 Critical Invariants

> Only a seated participant may transition a playing room to paused or a paused room to playing, and simulation ticks/input application occur only in playing.
>
> Entering paused neutralizes both externally visible paddle velocity and the internal simulation direction so resume starts without carried intent.

### 직접 연결되는 Major Engineering Difficulties

> Keeping protocol phase, GameHub session state, scheduler registration, snapshot state, simulation state, and UI controls synchronized.
>
> Testing temporal behavior deterministically without depending on wall-clock timing or a real network.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- shared `GamePhase`와 client event union은 pause/resume를 어떻게 표현합니까?
- 초기 `pauseRoom`/`resumeRoom`은 어떤 phase와 seat 조건을 검사하고 timer를 어떻게 관리합니까?
- 왜 timer를 멈춰도 `paddles[side].dy`와 simulation direction이 남으면 문제가 됩니까?
- later refactor 뒤 pause는 scheduler와 session state를 어떤 순서로 전환합니까?
- fake timer test는 pause 전 input, paused snapshot, resume 후 tick을 어떤 순서로 재현합니까?

## 3. 완료 기준

- protocol→hub phase transition→UI control의 호출 흐름을 설명할 수 있습니다.
- pause가 획득/해제하는 timer 또는 scheduler 자원과 illegal transition no-op 조건을 지적할 수 있습니다.
- visible snapshot와 internal simulation의 입력 상태를 둘 다 0으로 만들어야 하는 이유를 설명할 수 있습니다.
- fake timer regression이 증명하는 것과 실제 장시간 pause/reconnect에서 증명하지 않는 것을 구분할 수 있습니다.
- Commit map의 모든 SHA를 지정 브랜치 ancestry와 source classification에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 실행한 명령과 코드 검사만으로 확인한 사실을 구분하고 실행하지 않은 test를 통과했다고 기록하지 않습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `655bc7bd8df7` | `feat(protocol): 일시정지 WebSocket 계약 추가` | B | PROTOCOL, REALTIME, WEB | Adds the paused game phase and room-scoped pause/resume client commands. |
| 2 | `d93612c18e6f` | `feat(game): 서버 주도 일시정지 기능 추가` | B | SIMULATION, REALTIME | Handles pause and resume as server-owned room transitions. |
| 3 | `e4e2dec55805` | `feat(play): 일시정지와 재개 UI 연결` | B | REALTIME, WEB | Derives pause/resume controls from server snapshots and sends room-scoped commands. |
| 4 | `f46bbab95ea5` | `fix(game): 일시정지 시 paddle 입력 상태 초기화` | A | SIMULATION, REALTIME, RISK | Neutralizes both snapshot and internal simulation paddle directions when a room enters paused. |
| 5 | `632cbf13b616` | `test(game): pause 전 입력이 재개 뒤 남지 않음 검증` | B | SIMULATION, REALTIME, TEST | Verifies that a pre-pause paddle direction does not survive pause and resume. |

## 5. Commit별 학습 기록

### 5.1. `feat(protocol): 일시정지 WebSocket 계약 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `655bc7bd8df7` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds the paused game phase and room-scoped pause/resume client commands.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/game.ts`의 `GamePhase`
- `packages/shared/src/ws.ts`의 `game.pause`/`game.resume` variants
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:655bc7bd8df7 -->
- **직전 상태:** phase는 waiting/countdown/playing/finished뿐이고 client가 server에 pause/resume 의도를 보낼 계약이 없었습니다.
- **구현 결정:** `paused`를 공유 phase union에 추가하고 두 command가 `roomId`를 필수로 갖게 합니다.
- **상태/소유권 변화:** protocol은 요청과 관찰 가능한 phase를 표현하지만 전이 권한은 server에 남습니다.
- **실패/edge:** 타입만으로 sender가 room 참가자인지, 현재 phase가 legal한지는 확인하지 않습니다.
- **보장/비보장:** wire vocabulary는 통일하지만 timer와 simulation 동작은 다음 commit 전까지 없습니다.
- **다음 연결:** `d93612c18e6f`가 GameHub transition과 tick/input gating을 구현합니다.
<!-- LEARNER-ANSWER END commit:655bc7bd8df7 -->

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 Thread 관련 SHA: `d93612c18e6f` — `feat(game): 서버 주도 일시정지 기능 추가`

### 5.2. `feat(game): 서버 주도 일시정지 기능 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `d93612c18e6f` |
| Importance | B |
| Tags | SIMULATION, REALTIME |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Handles pause and resume as server-owned room transitions.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `pauseRoom`, `resumeRoom`
- playing/paused phase와 `sideFor` 조건
- timer clear/null 및 single `setInterval` restart
- `applyInput`/`tick`의 playing-only guard
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:d93612c18e6f -->
- **직전 상태:** protocol command는 parse돼도 server room과 simulation을 바꾸지 않았습니다.
- **구현 결정:** seated participant만 playing→paused, paused→playing을 요청할 수 있게 하고 pause는 interval을 clear해 null로, resume은 timer가 없을 때만 새 interval을 만듭니다. snapshot phase/time을 갱신해 room에 broadcast합니다. input/tick은 playing일 때만 작동합니다.
- **상태/소유권 변화:** server room이 phase와 timer authority가 되고 client는 snapshot을 따라갑니다.
- **실패/edge:** 잘못된 room/phase/non-participant 요청은 no-op입니다. 당시 pause가 기존 paddle direction 자체를 reset하지는 않습니다.
- **보장/비보장:** tick 중단과 중복 timer 방지는 제공하지만 stale input neutralization은 후속 fix가 필요합니다.
- **다음 연결:** `e4e2dec55805`가 UI를 authoritative snapshot phase에 연결합니다.
<!-- LEARNER-ANSWER END commit:d93612c18e6f -->

비교 기준:
- 직전 Thread 관련 SHA: `655bc7bd8df7` — `feat(protocol): 일시정지 WebSocket 계약 추가`
- 다음 Thread 관련 SHA: `e4e2dec55805` — `feat(play): 일시정지와 재개 UI 연결`

### 5.3. `feat(play): 일시정지와 재개 UI 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `e4e2dec55805` |
| Importance | B |
| Tags | REALTIME, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Derives pause/resume controls from server snapshots and sends room-scoped commands.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/play/page.tsx`의 `canPause`, `canResume`
- `togglePause`의 command send
- snapshot phase에 따른 status/button label
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:e4e2dec55805 -->
- **직전 상태:** play 화면에는 disabled “일시정지 예정” 버튼만 있고 server command와 연결되지 않았습니다.
- **구현 결정:** current room과 snapshot phase에서 pause/resume 가능 여부를 계산하고 같은 버튼이 해당 command를 전송합니다. phase snapshot 수신 시 상태 문구도 갱신합니다.
- **상태/소유권 변화:** UI는 server snapshot을 근거로 control을 활성화하며 click 직후 local phase를 임의 변경하지 않습니다.
- **실패/edge:** command rejection/no-op을 별도 acknowledgement로 표시하지 않으며 다음 snapshot이 authority입니다.
- **보장/비보장:** 정상 interaction은 연결하지만 server input carryover를 해결하지 않습니다.
- **다음 연결:** `f46bbab95ea5`가 pause 전 direction이 resume에 남는 simulation bug를 교정합니다.
<!-- LEARNER-ANSWER END commit:e4e2dec55805 -->

비교 기준:
- 직전 Thread 관련 SHA: `d93612c18e6f` — `feat(game): 서버 주도 일시정지 기능 추가`
- 다음 Thread 관련 SHA: `f46bbab95ea5` — `fix(game): 일시정지 시 paddle 입력 상태 초기화`

### 5.4. `fix(game): 일시정지 시 paddle 입력 상태 초기화`

| 항목 | 값 |
| --- | --- |
| SHA | `f46bbab95ea5` |
| Importance | A |
| Tags | SIMULATION, REALTIME, RISK |
| 학습 깊이 | 주요 subsystem, 구현 경로, ownership/failure/non-guarantee를 구체적으로 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Neutralizes both snapshot and internal simulation paddle directions when a room enters paused.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 later `pauseRoom`
- `roomScheduler.unregister`, `room.session.pause()`
- left/right `snapshot.state.paddles[side].dy = 0`
- `room.simulation.paddles[side].direction = 0`
- parent 상태와 비교해 이전 가정, 새 boundary, caller/callee, ownership 또는 failure path를 기록합니다.
- 이 commit이 보장하지 않는 상태와 다음 fix/test가 보강하는 지점을 구분합니다.
- Fix chain을 `이전 가정 → 실제 실패/위험 → root cause → 교정된 결정 → 남는 비보장` 순서로 복원합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:f46bbab95ea5 -->
- **직전 가정:** scheduler를 중단하고 phase를 paused로 바꾸면 게임 상태가 완전히 정지한다고 봤습니다.
- **실제 실패:** pause 직전 눌린 방향이 snapshot `dy` 또는 internal simulation direction에 남아 resume 첫 tick에서 새 input 없이 패들이 계속 움직일 수 있었습니다.
- **교정:** scheduler unregister와 성공한 `session.pause()` 뒤 양쪽 paddle의 외부 snapshot velocity와 내부 simulation direction을 모두 0으로 설정한 다음 paused snapshot을 broadcast합니다.
- **상태/소유권 변화:** pause는 단순 clock transition이 아니라 현재 input intent의 terminal boundary가 됩니다. 서로 다른 두 state representation을 함께 갱신합니다.
- **보장/비보장:** resume는 neutral intent에서 시작하지만 키가 실제로 계속 눌린 client가 resume 후 새 input을 보내는 것은 정상 새 의도입니다.
- **다음 연결:** `632cbf13b616`이 fake timer로 pre-pause direction이 resume 뒤 재생되지 않는지 검증합니다.
<!-- LEARNER-ANSWER END commit:f46bbab95ea5 -->

비교 기준:
- 직전 Thread 관련 SHA: `e4e2dec55805` — `feat(play): 일시정지와 재개 UI 연결`
- 다음 Thread 관련 SHA: `632cbf13b616` — `test(game): pause 전 입력이 재개 뒤 남지 않음 검증`

### 5.5. `test(game): pause 전 입력이 재개 뒤 남지 않음 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `632cbf13b616` |
| Importance | B |
| Tags | SIMULATION, REALTIME, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies that a pre-pause paddle direction does not survive pause and resume.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.pause.test.ts`의 fake timers/fake socket
- AI room 생성, ready, input sequence 0 direction 1
- pause snapshot phase/dy 0
- resume 후 100ms advance와 playing/dy 0
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:632cbf13b616 -->
- **직전 상태:** neutralization fix는 있었지만 scheduler 재등록과 다음 tick까지 통과한 뒤 상태가 0인지 보장하는 test가 없었습니다.
- **기법:** fake timers를 사용해 AI room을 만들고 ready→direction 1→pause를 보내 paused snapshot과 left `dy=0`을 확인합니다. paused 중 neutral input event를 보낸 뒤 resume하고 100ms를 진행해 playing snapshot에서도 `dy=0`인지 검사합니다.
- **생산 경로:** 실제 GameHub event handling, room/session/scheduler, snapshot parsing을 fake WebSocket으로 실행합니다.
- **증명/비증명:** pause boundary가 stale direction을 지우고 resume ticks가 이를 되살리지 않음을 검증하도록 작성됐습니다. 실제 브라우저 keyup, 장시간 pause, reconnect는 검증하지 않습니다.
- **보장:** snapshot과 simulation neutralization 중 하나가 빠지는 regression을 감지합니다.
- **Thread 종료:** pause/resume는 server phase·scheduler·input intent를 함께 전환하는 lifecycle이 됩니다.
<!-- LEARNER-ANSWER END commit:632cbf13b616 -->

비교 기준:
- 직전 Thread 관련 SHA: `f46bbab95ea5` — `fix(game): 일시정지 시 paddle 입력 상태 초기화`
- 이 commit을 Thread의 마지막 상태로 사용합니다.

## 6. Thread 종합

다음 항목을 commit 순서에서 재구성합니다: invariant evolution, Failure → Fix → Test 관계, ownership/state 변화, 최종 실행 흐름, 비보장, 실제 실행 증거.

<!-- LEARNER-ANSWER START thread:06-pause-resume-and-input-neutralization.md:synthesis -->
- **불변식 진화:** `655bc7bd8df7`은 `paused` phase와 pause/resume commands를 공유 계약에 추가했습니다. `d93612c18e6f`는 GameHub가 participant/phase를 검사하고 timer를 clear/restart하며 playing일 때만 input/tick을 허용하게 했습니다. `e4e2dec55805`는 snapshot phase로 UI 가능 여부를 계산했습니다. 그러나 later simulation/session refactor 상태에서 pause 전 direction이 남았고, `f46bbab95ea5`가 snapshot `dy`와 internal simulation direction을 모두 0으로 만들었습니다.
- **소유권:** server session/room이 phase authority이며 scheduler/timer는 tick ownership을 표현합니다. snapshot은 client-visible state, simulation object는 다음 tick에 사용할 internal intent를 소유합니다. UI는 command를 요청할 뿐 phase를 낙관적으로 바꾸지 않습니다.
- **Failure → Fix → Test:** timer 중단만 수행 → resume 첫 tick에서 stale direction 재적용 위험 → 두 state representation 모두 neutralize → fake timers와 fake socket으로 input 1→pause→dy 0→resume/tick→dy 0을 검증합니다.
- **최종 흐름:** participant가 playing room에 pause 전송 → scheduler unregister/session pause → 양 paddle의 snapshot/internal direction 0 → paused snapshot broadcast → paused 중 input은 적용되지 않음 → participant resume → session playing/scheduler register → neutral state에서 ticks 재개입니다.
- **비보장:** pause timeout, 합의형 pause, reconnect 후 pause ownership, process restart persistence는 이 Thread의 기능이 아닙니다.
- **실행 증거:** test 구현은 exact SHA에서 검사했으나 Vitest fake-timer suite를 실행하지 않았습니다.
<!-- LEARNER-ANSWER END thread:06-pause-resume-and-input-neutralization.md:synthesis -->

## 7. 학습 완료 확인

<!-- LEARNER-ANSWER START thread:06-pause-resume-and-input-neutralization.md:checklist -->
- [x] 모든 SHA를 지정 브랜치의 exact commit으로 검사했습니다.
- [x] fixed commit map과 source classification을 보존했습니다.
- [x] earlier commit을 later HEAD 코드로 설명하지 않았습니다.
- [x] fix와 test를 원래 failure/production path에 연결했습니다.
- [x] 실제 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
- [x] Thread 최종 owner, invariant, flow, non-guarantee를 작성했습니다.
<!-- LEARNER-ANSWER END thread:06-pause-resume-and-input-neutralization.md:checklist -->
===== END FILE: 06-pause-resume-and-input-neutralization.md =====

===== BEGIN FILE: 07-npc-ai-policy-and-fallback-journey.md =====
# NPC AI 정책과 fallback 여정

원문 Development Thread: `NPC AI policy and fallback journey`

## 1. Thread 목표

- NPC를 익명 문자열이 아니라 `isNpc`가 저장된 사용자 identity로 도입하고 rating band별 상대를 구성하는 과정을 추적합니다.
- 직접 AI 모드와 6초 human-queue fallback이 같은 persisted NPC를 room participant와 match result에 연결하는 경로를 복원합니다.
- rating 기반 반응 주기·예측 오차·실수 확률·속도·dead zone을 deterministic policy로 구현하고 UI/smoke test에 노출하는 과정을 확인합니다.

### Source에서 확정된 significance

> An AI opponent affects identity, matchmaking, room membership, simulation policy, persistence, ranking/profile presentation, and cleanup. Treating it as a label loses result attribution; treating fallback as an uncancelled timer can create duplicate matches. The history gives NPCs durable identity and explicit timer/state ownership.

### 직접 연결되는 Critical Invariants

> NPC opponents are persisted users marked `isNpc`, remain offline as presence, and are selected by rating without being confused with authenticated human identities.
>
> A queued player is matched at most once: human pairing, leave, disconnect/prune, or NPC fallback all clear the queue timer and revalidate membership before room creation.

### 직접 연결되는 Major Engineering Difficulties

> Maintaining PostgreSQL/memory parity for seeded NPC identity while avoiding handle collisions that leave a real login marked as NPC.
>
> Making AI variation reproducible from room/tick state so policy can be reasoned about and tested without ambient randomness.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `is_npc`는 schema, shared `PublicUser`, row mapper, chat/tournament projections에 어떻게 전파됩니까?
- dev login과 NPC upsert가 같은 handle을 만날 때 어떤 값을 다시 설정해 identity 오염을 막습니까?
- `findClosestNpc`는 rating distance 동률과 empty NPC list를 어떻게 처리합니까?
- 6초 fallback callback은 timer가 발화한 뒤 queue membership, socket open, existing room을 왜 다시 확인합니까?
- human match, leave, prune에서 `clearQueueTimer`를 빠뜨리면 어떤 duplicate room 위험이 생깁니까?
- AI profile의 reaction/noise/mistake/speed/dead-zone와 `predictedBallY` 반사가 gameplay에 어떻게 반영됩니까?
- smoke test는 solo queue가 실제 persisted NPC handle을 가진 snapshot으로 이어졌음을 어떻게 확인합니까?

## 3. 완료 기준

- NPC identity가 DB row에서 room snapshot과 persisted match result까지 이동하는 경로를 설명할 수 있습니다.
- queue entry와 fallback timer의 lifetime 및 모든 cancellation path를 나열할 수 있습니다.
- rating band별 AI profile과 deterministic pseudo-random 입력의 seed를 실제 함수로 설명할 수 있습니다.
- UI의 AI badge/친구 버튼 제한과 server-side identity rule의 차이를 구분할 수 있습니다.
- smoke test가 검증하는 end-to-end 연결과 AI 품질 자체를 검증하지 않는다는 한계를 설명할 수 있습니다.
- Commit map의 모든 SHA를 지정 브랜치 ancestry와 source classification에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 실행한 명령과 코드 검사만으로 확인한 사실을 구분하고 실행하지 않은 test를 통과했다고 기록하지 않습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `72d23baefc3c` | `feat(db): NPC 사용자 contract와 schema 추가` | B | REALTIME, PERSISTENCE, TOURNAMENT | Adds a persisted NPC marker to user storage and every shared projection that can carry a user. |
| 2 | `b3239bae51e5` | `feat(db): rating 구간별 NPC 상대 저장` | B | REALTIME, PERSISTENCE | Seeds rating-banded NPC users and exposes active NPC opponents separately from presence. |
| 3 | `dec431822873` | `test(db): NPC seed와 leaderboard 분리 검증` | B | REALTIME, PERSISTENCE, TEST | Verifies the memory repository exposes rating-banded NPC opponents as NPC and offline projections. |
| 4 | `87b38e2f23c8` | `feat(game): NPC 상대를 경기 방에 연결` | B | REALTIME | Uses persisted NPC identity in AI rooms and match-result attribution. |
| 5 | `1122e6a4b901` | `feat(game): 대기 플레이어 NPC fallback 구성` | B | REALTIME | Adds six-second NPC fallback with explicit timer cleanup and asynchronous revalidation. |
| 6 | `b159bcda3b83` | `feat(game): rating 기반 NPC AI policy 구현` | B | SIMULATION, REALTIME, WEB | Maps NPC rating bands to deterministic reaction, prediction, mistake, speed, and dead-zone policy. |
| 7 | `afd0a97c5c1c` | `feat(web): 대기열에서 NPC 상대 표시` | B | REALTIME, WEB | Exposes NPC identity and fallback behavior in queue, play, profile, and ranking presentation. |
| 8 | `cfb15fc84dee` | `test(app): NPC fallback matching 검증` | B | PROTOCOL, REALTIME, PERSISTENCE | Extends the WebSocket smoke test through solo queue fallback into an NPC-backed room snapshot. |

## 5. Commit별 학습 기록

### 5.1. `feat(db): NPC 사용자 contract와 schema 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `72d23baefc3c` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds a persisted NPC marker to user storage and every shared projection that can carry a user.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/http.ts`의 `PublicUser.isNpc`
- `packages/db/src/migrations.ts`의 `users.is_npc boolean not null default false`
- `packages/db/src/schema.ts`의 user/projection row 타입
- `packages/db/src/rowMappers.ts` 및 chat/tournament query projection
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:72d23baefc3c -->
- **직전 상태:** AI 상대는 room 안의 특수 문자열/flag로 표현돼 사용자·chat·tournament projection과 같은 identity contract를 갖지 못했습니다.
- **구현 결정:** users row에 `is_npc`를 추가하고 shared `PublicUser.isNpc`로 mapping하며 chat sender와 tournament creator/participant query도 해당 column을 선택합니다. memory dev user는 false입니다.
- **상태/소유권 변화:** NPC 여부가 room 임시 label이 아니라 durable user identity 속성이 됩니다.
- **실패/edge:** marker만 추가했으므로 실제 NPC row와 선택 정책은 아직 없습니다. 기존 row는 default false입니다.
- **보장/비보장:** user가 전달되는 contract 전반에서 NPC 여부를 보존하지만 NPC가 online인지, 어떤 rating인지, 누구와 match할지는 정하지 않습니다.
- **다음 연결:** `b3239bae51e5`가 rating band별 NPC row를 seed하고 별도 조회를 구현합니다.
<!-- LEARNER-ANSWER END commit:72d23baefc3c -->

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 Thread 관련 SHA: `b3239bae51e5` — `feat(db): rating 구간별 NPC 상대 저장`

### 5.2. `feat(db): rating 구간별 NPC 상대 저장`

| 항목 | 값 |
| --- | --- |
| SHA | `b3239bae51e5` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Seeds rating-banded NPC users and exposes active NPC opponents separately from presence.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `NPC_PLAYERS` 1100/1200/1300/1400
- PostgreSQL `upsertNpc`와 `listNpcOpponents`
- memory seed/list parity
- dev user upsert의 `is_npc = false` 복구
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:b3239bae51e5 -->
- **직전 상태:** `isNpc` 필드는 있었지만 사용할 NPC row가 없고 repository에서 NPC 후보를 가져올 API가 없었습니다.
- **구현 결정:** 네 고정 handle/display/avatar/rating seed를 active `is_npc=true`, email null로 upsert하고 rating 오름차순 `listNpcOpponents`를 추가합니다. projection은 online false입니다. 같은 handle로 dev login하면 `is_npc=false`로 되돌립니다.
- **상태/소유권 변화:** NPC identity/rating band는 repository seed가 소유하며 realtime presence와 분리됩니다.
- **실패/edge:** 고정 seed 정책은 운영 콘텐츠 변경 시 migration/seed update가 필요하고, 단순 rating distance 외 gameplay skill calibration은 아직 없습니다.
- **보장/비보장:** 양 backend에서 같은 후보 집합과 human/NPC marker 복구를 제공하지만 match selection은 다음 commit입니다.
- **다음 연결:** `dec431822873`이 seed ratings/marker/offline projection을 고정합니다.
<!-- LEARNER-ANSWER END commit:b3239bae51e5 -->

비교 기준:
- 직전 Thread 관련 SHA: `72d23baefc3c` — `feat(db): NPC 사용자 contract와 schema 추가`
- 다음 Thread 관련 SHA: `dec431822873` — `test(db): NPC seed와 leaderboard 분리 검증`

### 5.3. `test(db): NPC seed와 leaderboard 분리 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `dec431822873` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TEST |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies the memory repository exposes rating-banded NPC opponents as NPC and offline projections.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 `ensureSeedData` 후 `listNpcOpponents`
- ratings `[1100, 1200, 1300, 1400]`
- 모두 `isNpc === true`, `online === false`
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:dec431822873 -->
- **직전 상태:** seed/list 구현은 있었지만 order, marker, presence projection이 바뀌어도 감지할 regression이 없었습니다.
- **기법:** memory repository를 seed한 뒤 NPC 목록 rating 배열과 모든 `isNpc`, 모든 offline 값을 직접 검사합니다.
- **생산 경로:** 실제 memory seed와 public projection/list sorting을 통과합니다.
- **증명/비증명:** 후보 집합과 identity/presence 분리를 검증하도록 작성됐지만 PostgreSQL seed, leaderboard 표시, match 결과 저장은 포함하지 않습니다.
- **보장:** 개발/test backend에서 stable rating bands를 제공합니다.
- **다음 연결:** `87b38e2f23c8`이 후보를 실제 room과 result persistence에 연결합니다.
<!-- LEARNER-ANSWER END commit:dec431822873 -->

비교 기준:
- 직전 Thread 관련 SHA: `b3239bae51e5` — `feat(db): rating 구간별 NPC 상대 저장`
- 다음 Thread 관련 SHA: `87b38e2f23c8` — `feat(game): NPC 상대를 경기 방에 연결`

### 5.4. `feat(game): NPC 상대를 경기 방에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `87b38e2f23c8` |
| Importance | B |
| Tags | REALTIME |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Uses persisted NPC identity in AI rooms and match-result attribution.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 async `joinQueue`와 `findClosestNpc`
- absolute rating distance selection
- `createRoom`의 `npc` option/right player snapshot
- `finishRoom`의 winner/loser ID에 NPC identity 사용
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:87b38e2f23c8 -->
- **직전 상태:** AI room의 오른쪽 참가자는 `ai-opponent`/`ai`/`연습 AI` 같은 익명 fallback이라 generic match가 실제 상대 user를 참조하지 못했습니다.
- **구현 결정:** repository NPC 목록에서 `abs(npc.rating - player.rating)`이 가장 작은 후보를 선택하고 room snapshot의 오른쪽 participant와 opponent label에 사용합니다. 종료 시 room이 보관한 NPC user ID를 winner/loser로 넘깁니다.
- **상태/소유권 변화:** room은 선택된 persisted NPC identity를 lifetime 동안 보존하고 result persistence도 같은 ID를 사용합니다.
- **실패/edge:** 후보가 없으면 기존 anonymous fallback이 남을 수 있고 동률은 먼저 나온 rating-order 후보가 선택됩니다. 일반 human queue의 지연 fallback은 아직 없습니다.
- **보장/비보장:** 직접 AI room의 identity attribution은 제공하지만 queue timeout ownership은 다음 commit입니다.
- **다음 연결:** `1122e6a4b901`이 human 상대가 없을 때 같은 NPC를 6초 fallback으로 배정합니다.
<!-- LEARNER-ANSWER END commit:87b38e2f23c8 -->

비교 기준:
- 직전 Thread 관련 SHA: `dec431822873` — `test(db): NPC seed와 leaderboard 분리 검증`
- 다음 Thread 관련 SHA: `1122e6a4b901` — `feat(game): 대기 플레이어 NPC fallback 구성`

### 5.5. `feat(game): 대기 플레이어 NPC fallback 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `1122e6a4b901` |
| Importance | B |
| Tags | REALTIME |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds six-second NPC fallback with explicit timer cleanup and asynchronous revalidation.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `QueueEntry.npcFallbackTimer`
- `NPC_QUEUE_FALLBACK_MS = 6000`
- `matchQueuedClientWithNpc` membership/socket/room checks
- human match, `leaveQueue`, `pruneQueue`의 `clearQueueTimer`
- Room `npcUser`와 result attribution
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:1122e6a4b901 -->
- **직전 상태:** 일반 queue에 human이 없으면 무기한 기다렸고 NPC는 명시적 AI mode에서만 즉시 배정됐습니다.
- **구현 결정:** unmatched queue entry마다 6초 timer를 만들고 callback에서 entry가 아직 queue에 있는지, socket이 OPEN인지, 이미 room이 없는지 재확인한 뒤 closest NPC로 room을 만듭니다. human pair, leave, closed-socket prune는 timer를 clear/null합니다.
- **상태/소유권 변화:** fallback timer는 queue entry의 자원이며 queue membership이 lifetime을 결정합니다. room은 선택된 `npcUser`를 별도로 보관합니다.
- **실패/edge:** callback의 NPC lookup은 async이므로 timer 발화 뒤 상태가 바뀔 수 있어 재검사가 필수입니다. 정리를 빠뜨리면 human room 뒤 두 번째 NPC room이 생길 수 있습니다.
- **보장/비보장:** 단일 GameHub process에서 queue entry당 한 fallback path를 목표로 하지만 multi-process reservation은 later Matchmaker(category 05)가 담당합니다.
- **다음 연결:** `b159bcda3b83`이 NPC rating을 실제 AI 행동 profile로 사용합니다.
<!-- LEARNER-ANSWER END commit:1122e6a4b901 -->

비교 기준:
- 직전 Thread 관련 SHA: `87b38e2f23c8` — `feat(game): NPC 상대를 경기 방에 연결`
- 다음 Thread 관련 SHA: `b159bcda3b83` — `feat(game): rating 기반 NPC AI policy 구현`

### 5.6. `feat(game): rating 기반 NPC AI policy 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `b159bcda3b83` |
| Importance | B |
| Tags | SIMULATION, REALTIME, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Maps NPC rating bands to deterministic reaction, prediction, mistake, speed, and dead-zone policy.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.ts`의 `AiProfile`, `aiProfileFor`
- `updateAiPaddleIntent`, `predictedBallY`
- `deterministicUnit`, `signedDeterministic`, `hashString`
- Room `aiTargetY`와 right paddle speed multiplier
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:b159bcda3b83 -->
- **직전 상태:** AI는 매 tick 현재 ball Y를 단순 추적해 persisted rating이 gameplay에 영향을 주지 않았습니다.
- **구현 결정:** 1400/1300/1200/lower band에 reaction ticks, prediction noise, mistake chance, paddle speed multiplier, dead zone을 지정합니다. 공이 AI 쪽으로 갈 때 벽 반사를 포함한 도착 Y를 예측하고 room ID·tick·salt 기반 deterministic noise/mistake를 적용합니다. 공이 반대 방향이면 중앙을 목표로 합니다.
- **상태/소유권 변화:** room이 `aiTargetY`를 유지하고 rating이 simulation policy를 선택합니다. ambient `Math.random` 대신 재현 가능한 pseudo-random 값을 사용합니다.
- **실패/edge:** policy 값은 경험적 설정이며 난이도·공정성 통계가 없습니다. `predictedBallY`는 현재 단순 직선/벽 반사 모델을 전제로 합니다.
- **보장/비보장:** 동일 room/tick/state에서 같은 intent를 계산하고 rating band별 차이를 만들지만 인간 수준 성능은 보장하지 않습니다.
- **다음 연결:** `afd0a97c5c1c`이 queue/NPC identity를 UI에 명시합니다.
<!-- LEARNER-ANSWER END commit:b159bcda3b83 -->

비교 기준:
- 직전 Thread 관련 SHA: `1122e6a4b901` — `feat(game): 대기 플레이어 NPC fallback 구성`
- 다음 Thread 관련 SHA: `afd0a97c5c1c` — `feat(web): 대기열에서 NPC 상대 표시`

### 5.7. `feat(web): 대기열에서 NPC 상대 표시`

| 항목 | 값 |
| --- | --- |
| SHA | `afd0a97c5c1c` |
| Importance | B |
| Tags | REALTIME, WEB |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Exposes NPC identity and fallback behavior in queue, play, profile, and ranking presentation.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/page.tsx`의 `/play?mode=queue`와 fallback 설명
- `apps/web/src/app/play/page.tsx`의 auto queue 및 opponent `ai` 설명
- leaderboard/profile AI badge
- NPC profile의 friend button disable
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:afd0a97c5c1c -->
- **직전 상태:** server가 NPC를 배정해도 home/play/profile/leaderboard는 human과 구분되지 않거나 queue route가 자동 참가하지 않았습니다.
- **구현 결정:** queue CTA가 queue mode query로 play를 열어 자동 join하고 human이 없으면 AI 배정됨을 설명합니다. snapshot `player.ai`로 상대 문구를 바꾸고 `isNpc` badge를 profile/leaderboard에 표시하며 NPC 친구 추가를 비활성화합니다.
- **상태/소유권 변화:** web은 server-provided `isNpc`/`ai`를 표시할 뿐 NPC를 추정하지 않습니다.
- **실패/edge:** UI disable은 보안 경계가 아니므로 friend API가 NPC 관계를 허용하지 않는 invariant는 server가 별도로 책임져야 합니다. 당시 profile의 handle-length 선수 번호 같은 unrelated placeholder는 남을 수 있습니다.
- **보장/비보장:** 사용자 journey의 명시성은 높이지만 fallback 성공 자체는 server/smoke가 검증합니다.
- **다음 연결:** `cfb15fc84dee`가 solo queue에서 AI-labelled match와 persisted NPC handle snapshot을 기다립니다.
<!-- LEARNER-ANSWER END commit:afd0a97c5c1c -->

비교 기준:
- 직전 Thread 관련 SHA: `b159bcda3b83` — `feat(game): rating 기반 NPC AI policy 구현`
- 다음 Thread 관련 SHA: `cfb15fc84dee` — `test(app): NPC fallback matching 검증`

### 5.8. `test(app): NPC fallback matching 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `cfb15fc84dee` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, PERSISTENCE |
| 학습 깊이 | Thread 안에서 필요한 실제 구현 역할과 상태 변화를 복원합니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Extends the WebSocket smoke test through solo queue fallback into an NPC-backed room snapshot.
- 이 역할·제목·Importance·Tags는 지정 브랜치의 source classification과 exact commit diff를 대조해 고정했습니다.

#### 해당 SHA에서 확인할 실제 코드

- `tests/smoke-ws.mjs`의 solo login/socket
- `queue.join` mode queue
- 8초 안 `queue.matched` opponent가 `AI` 포함
- `game.ready` 후 snapshot의 `player.ai` 및 `handle.startsWith("npc-")`
- finally socket close
- parent 또는 직전 Thread SHA와 비교해 concrete state/data/caller 변화와 edge path를 기록합니다.
- Test technique, failure/boundary fixture, production path, proves/does-not-prove를 실제 assertion으로 구분합니다.

#### 학습자 기록

<!-- LEARNER-ANSWER START commit:cfb15fc84dee -->
- **직전 상태:** seed/unit 검사와 GameHub 구현은 있었지만 실제 login→WebSocket→queue timer→room snapshot의 서비스 경로를 연결한 smoke 근거가 없었습니다.
- **기법:** 별도 solo 사용자를 로그인/연결하고 일반 queue에 참가시킵니다. 6초 fallback을 고려해 8초 timeout으로 AI opponent match를 기다린 뒤 ready를 보내고 AI participant handle이 `npc-`로 시작하는 snapshot을 기다립니다.
- **생산 경로:** HTTP login, WebSocket auth/parse, queue timer, repository NPC lookup, room creation, snapshot encoding을 통과하도록 작성됐습니다.
- **증명/비증명:** end-to-end fallback과 persisted NPC identity 노출을 검증하지만 AI paddle policy 품질, match completion/result row, 동시 queue race를 검증하지 않습니다.
- **cleanup:** `finally`에서 solo socket을 닫고 기존 PvP socket cleanup도 유지합니다.
- **Thread 종료:** 저장 identity, delayed matching, simulation policy, UI disclosure, service smoke가 하나의 journey로 연결됩니다.
<!-- LEARNER-ANSWER END commit:cfb15fc84dee -->

비교 기준:
- 직전 Thread 관련 SHA: `afd0a97c5c1c` — `feat(web): 대기열에서 NPC 상대 표시`
- 이 commit을 Thread의 마지막 상태로 사용합니다.

## 6. Thread 종합

다음 항목을 commit 순서에서 재구성합니다: invariant evolution, Failure → Fix → Test 관계, ownership/state 변화, 최종 실행 흐름, 비보장, 실제 실행 증거.

<!-- LEARNER-ANSWER START thread:07-npc-ai-policy-and-fallback-journey.md:synthesis -->
- **불변식 진화:** `72d23baefc3c`은 user contract/schema에 `isNpc`를 추가했고 `b3239bae51e5`는 1100/1200/1300/1400 rating NPC를 PostgreSQL/memory에 seed하며 별도 조회를 제공했습니다. `87b38e2f23c8`은 closest NPC를 room participant·result identity로 연결했습니다. `1122e6a4b901`은 human queue에서 6초 뒤 fallback하되 queue/socket/room을 재검사하고 모든 terminal path에서 timer를 정리합니다. `b159bcda3b83`은 rating을 simulation policy로 바꿉니다.
- **소유권:** repository는 NPC identity와 rating band를, queue entry는 fallback timer를, GameHub room은 선택된 `npcUser`와 result attribution을, simulation은 `aiTargetY`와 profile 기반 intent를, web은 AI 표시와 action 제한을 소유합니다.
- **Failure → Fix → Test:** 익명 `ai-opponent` 결과 attribution → persisted NPC user. 대기 timer 미정리/async stale callback 위험 → entry-owned timer와 membership/socket/room 재검사. NPC seed가 human identity를 오염할 위험 → dev login upsert가 `is_npc=false`. memory seed/rating/offline test와 실제 WebSocket solo fallback smoke가 경계를 검증합니다.
- **최종 흐름:** seed 시 NPC users 저장 → queue/AI 요청 시 현재 user rating과 가장 가까운 NPC 조회 → 직접 AI는 즉시, 일반 queue는 human이 없으면 6초 후 재검사해 room 생성 → snapshot participant에 NPC identity/`ai` 표시 → deterministic policy가 paddle intent 계산 → 종료 결과는 NPC user ID로 저장 → web은 badge와 AI 상대 설명을 표시합니다.
- **비보장:** rating policy가 공정하거나 인간과 동일한 난이도를 제공한다는 통계적 증거는 없습니다. multi-instance queue timer coordination과 later Matchmaker reservation ownership은 category 05가 주 소유자입니다.
- **실행 증거:** seed/unit/smoke test 구현을 검사했지만 실제 repository seed나 8초 WebSocket smoke를 실행하지 않았습니다.
<!-- LEARNER-ANSWER END thread:07-npc-ai-policy-and-fallback-journey.md:synthesis -->

## 7. 학습 완료 확인

<!-- LEARNER-ANSWER START thread:07-npc-ai-policy-and-fallback-journey.md:checklist -->
- [x] 모든 SHA를 지정 브랜치의 exact commit으로 검사했습니다.
- [x] fixed commit map과 source classification을 보존했습니다.
- [x] earlier commit을 later HEAD 코드로 설명하지 않았습니다.
- [x] fix와 test를 원래 failure/production path에 연결했습니다.
- [x] 실제 실행하지 않은 test를 통과했다고 기록하지 않았습니다.
- [x] Thread 최종 owner, invariant, flow, non-guarantee를 작성했습니다.
<!-- LEARNER-ANSWER END thread:07-npc-ai-policy-and-fallback-journey.md:checklist -->
===== END FILE: 07-npc-ai-policy-and-fallback-journey.md =====

===== BEGIN FILE: README.md =====
# 04 — Domain Workflows and Realtime Features

Repository: `seungwoo7050/42-archive`  
Branch: `web/ft_transcendence`  
Category path: `development-thread-workbook/04-domain-workflows-and-realtime-features`

## Phase 1 감사 결과와 동결 범위

- 이 category boundary는 유지했습니다. 제품 사용자가 거치는 tournament, profile/dashboard/ranking, lobby/chat, pause/resume, NPC/AI journey와 realtime 기능 통합이 대상입니다.
- canonical friendship pair 및 tournament admission/capacity concurrency는 `02-persistence-and-data-integrity`가 주 소유자입니다.
- simulation loop, connection lifecycle, Matchmaker reservation ownership, scheduler, finalization retry와 process drain은 `05-core-realtime-architecture`가 주 소유자입니다.
- 이 category는 위 subsystem을 중복 재구성하지 않고 제품 workflow에서 호출·표시·rollback·handoff되는 지점만 포함합니다.
- Thread 수와 파일명은 7개로 유지했습니다. 독립 Thread를 합치거나 분리하거나 다른 category commit을 흡수하지 않았습니다.

### Scaffold 보정 내역

- 기존 60개 reference에 repository evidence상 필요한 10개 intermediate/fix/test commit을 추가해 총 70개를 동결했습니다.
- Thread 01에 `9b1dabcc4bb4`, `4370ac3162b2`를 추가해 entry-only repository와 UI-fabricated bracket이라는 선행 상태를 보존했습니다.
- Thread 02에 `e338ea32b2a6`, `582a1615a2c6`, `10bf15723591`을 추가해 atomic finalization 구현·동시성/rollback 검증·GameHub handoff를 포함했습니다.
- Thread 03에 `8d79139a32da`, `be31566ac0fd`, `7fe29f991a9b`을 추가하고 실제 시간순으로 재배치해 fixed/sample read model 교정 chain을 복원했습니다.
- Thread 04에 `8ce1199ffd12`, `23a978879b81`을 추가하고 실제 시간순으로 재배치해 missing-body fix와 eventual presence smoke regression을 포함했습니다.
- commit을 삭제하거나 다른 Thread로 이동하지 않았습니다. cross-cutting `be31566ac0fd`는 profile/dashboard/ranking read-model 정직성 Thread에 한 번만 두고 lobby/tournament Thread에서 참조합니다.
- generic investigation 문구를 exact file, function, SQL, schema, test, timer, state owner, cleanup/failure/non-guarantee 질문으로 교체했습니다.

### Thread boundary와 ordering 판단

- Thread 01과 02는 분리했습니다. 전자는 tournament-match contract/schema/bracket topology의 생성과 소비를, 후자는 그 persisted match를 realtime room으로 publish하고 durable result로 인계하는 cross-boundary lifecycle을 소유합니다.
- Thread 03은 분리하지 않았습니다. profile, dashboard, leaderboard, friendship journey가 동일한 user/recent-match read model과 identity/cache invalidation에 연결되기 때문입니다. 단, friendship canonical pair와 동시성은 category 02를 참조합니다.
- Thread 04와 05는 분리했습니다. Thread 04는 `/lobby`와 한 화면에서 durable chat history, process-local presence, live statistics, HTTP/WebSocket 반영을 합치는 제품 journey를 소유합니다. Thread 05는 lobby/match scope-room 저장 불변식과 current-seat authorization이라는 독립 보안 story를 소유합니다.
- Thread 06은 scheduler/phase/input state를 함께 전환하는 독립 temporal lifecycle이고, Thread 07은 persisted NPC identity, queue timer, simulation policy, UI disclosure를 잇는 독립 fallback journey입니다.
- 전체 Thread 순서는 contract/storage → realtime handoff → user read journey → lobby composition → chat hardening → pause lifecycle → NPC fallback 순으로 유지했습니다. 서로 다른 기능의 commit은 branch에서 교차하지만 각 Thread 내부 commit map은 actual commit chronology로 정렬했습니다.

### 동결된 파일과 commit 수

| 파일 | Commit 수 |
| --- | ---: |
| `01-tournament-contract-schema-and-bracket-construction.md` | 10 |
| `02-tournament-room-start-rollback-and-finalization-handoff.md` | 10 |
| `03-profile-friendship-dashboard-and-ranking-journeys.md` | 17 |
| `04-lobby-presence-chat-and-live-statistics.md` | 12 |
| `05-chat-scope-storage-and-room-authorization.md` | 8 |
| `06-pause-resume-and-input-neutralization.md` | 5 |
| `07-npc-ai-policy-and-fallback-journey.md` | 8 |
| **합계** | **70** |

- Frozen commit manifest SHA-256: `16394a851a8ac4eae148544ffed672a2f47f717ec21af4b7bee53244da9fbf00`
- Phase 2에서는 위 commit map, 제목, 순서, Importance, Tags, source-defined 역할과 문서 구조를 변경하지 않습니다.

## 읽는 순서

1. [`01-tournament-contract-schema-and-bracket-construction.md`](01-tournament-contract-schema-and-bracket-construction.md) — 토너먼트 계약·스키마와 대진 구성
2. [`02-tournament-room-start-rollback-and-finalization-handoff.md`](02-tournament-room-start-rollback-and-finalization-handoff.md) — 토너먼트 경기방 시작 롤백과 결과 확정 인계
3. [`03-profile-friendship-dashboard-and-ranking-journeys.md`](03-profile-friendship-dashboard-and-ranking-journeys.md) — 프로필·친구·대시보드·순위표 여정
4. [`04-lobby-presence-chat-and-live-statistics.md`](04-lobby-presence-chat-and-live-statistics.md) — 로비 접속 상태·채팅과 실시간 지표
5. [`05-chat-scope-storage-and-room-authorization.md`](05-chat-scope-storage-and-room-authorization.md) — 채팅 scope 저장 불변식과 경기방 권한
6. [`06-pause-resume-and-input-neutralization.md`](06-pause-resume-and-input-neutralization.md) — 일시정지·재개와 입력 무효화
7. [`07-npc-ai-policy-and-fallback-journey.md`](07-npc-ai-policy-and-fallback-journey.md) — NPC AI 정책과 fallback 여정

## Evidence discipline

- 모든 구현 설명은 해당 SHA의 commit diff와 그 시점 파일/심볼을 기준으로 작성합니다.
- later refactor가 있는 경우 earlier section에 역투영하지 않고, 해당 commit에서 실제로 존재한 표현과 비보장을 기록합니다.
- test commit은 fixture/failure injection, production path, assertion, 증명/비증명 범위를 분리합니다.
- 로컬 checkout을 만들 수 없어 repository command나 test runner를 실행하지 않았습니다. 실행 결과를 만들지 않았으며 코드·test 구현 검사와 runtime evidence를 구분합니다.

## Phase 2 완료 및 검증 기록

<!-- LEARNER-ANSWER START readme:phase2-validation -->
- [x] frozen scaffold 8개 파일(README + 7 Threads)에 정확히 대응하는 completed 8개 파일을 생성했습니다.
- [x] 총 70개 commit reference의 SHA/subject/order/Importance/Tags/source role이 scaffold와 completed에서 일치합니다.
- [x] 모든 referenced SHA는 branch source classification과 exact commit 조회에서 `web/ft_transcendence` 이력의 commit으로 확인했습니다.
- [x] completed의 learner-facing placeholder를 모두 채웠으며 S/A/B/C 깊이를 구분했습니다.
- [x] fix/test 관계, historical SHA, 실행하지 않은 test의 비실행 표기를 검사했습니다.
- [x] Phase 2 전후 frozen scaffold tree SHA-256이 `cc53210489e51fe33c556628828cb23e680e7e35a4c7ce59b5c555889364629c`로 동일합니다.
- [x] remote repository에는 commit/push/PR/파일 변경을 수행하지 않았습니다.
<!-- LEARNER-ANSWER END readme:phase2-validation -->
===== END FILE: README.md =====

