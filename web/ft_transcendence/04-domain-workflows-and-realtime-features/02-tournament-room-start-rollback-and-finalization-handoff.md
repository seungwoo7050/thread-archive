# 토너먼트 경기방과 저장소를 하나의 전이처럼 다루기

원문 Thread: `Tournament room start rollback and finalization handoff`

## 이 Thread가 다루는 문제

토너먼트 경기는 두 종류의 상태를 동시에 바꿉니다.

- GameHub 안에서는 두 WebSocket client를 room에 배치하고 scheduler/timer를 소유합니다.
- repository 안에서는 특정 tournament match를 `ready → running → finished`로 바꾸고, 일반 경기 결과·rating·다음 라운드·우승자를 저장합니다.

두 상태를 하나의 DB transaction으로 묶을 수는 없습니다. GameHub room은 process memory이고 tournament match는 PostgreSQL row이기 때문입니다. 따라서 순차 작업 중간에 실패하면 한쪽만 성공한 “ghost room”이나 “stuck match”가 생길 수 있습니다.

이 Thread는 두 방향의 부분 성공을 닫습니다.

1. **시작:** room을 먼저 만든 뒤 DB start가 실패하면 room publication을 보상 롤백한다.
2. **종료:** 일반 경기 저장과 bracket 진행을 하나의 idempotent `finalizeMatch` transaction으로 인계한다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `33b6dfc5df7a` | `feat(tournament): 토너먼트 경기 방 진행` | A | REALTIME, TOURNAMENT, RISK | 저장된 대진의 두 참가자를 realtime room에 배치한다. |
| 2 | `e338ea32b2a6` | `feat(db): PostgreSQL tournament 경기 확정을 연결` | S | PERSISTENCE, TOURNAMENT, RISK | 일반 경기·rating·대진 연결·다음 라운드·우승 확정을 하나의 잠금 transaction으로 묶는다. |
| 3 | `582a1615a2c6` | `test(db): 경기 결과 단일 확정 조건 검증` | A | PERSISTENCE, TOURNAMENT, RISK | 반복·동시 호출·중간 실패에서도 결과가 한 번만 확정되는지 검증한다. |
| 4 | `10bf15723591` | `refactor(game): 경기 결과 확정 boundary 사용` | A | REALTIME, PERSISTENCE, TOURNAMENT | room 종료를 canonical `finalizeMatch` 명령과 하나의 in-flight promise로 인계한다. |
| 5 | `916683099ecd` | `fix(db): tournament start 상태 갱신 여부 확인` | A | REALTIME, PERSISTENCE, TOURNAMENT | 0-row start update를 성공으로 취급하지 않고 명시적으로 실패시킨다. |
| 6 | `480e2dc48028` | `test(db): tournament match 미갱신 거부 검증` | B | REALTIME, PERSISTENCE, TOURNAMENT | 존재하지 않거나 갱신 불가능한 대진 시작이 거부되는지 고정한다. |
| 7 | `38312bcaf632` | `fix(game): tournament 시작 실패 시 room 상태 복원` | A | REALTIME, TOURNAMENT, RISK | room 공개 뒤 저장소 start가 실패하면 room·scheduler·client 연결을 되돌린다. |
| 8 | `4e2cb4ae702d` | `test(game): tournament start rollback 검증` | B | REALTIME, PERSISTENCE, TOURNAMENT | start 실패 후 잔여 room이 없고 같은 대진을 다시 시작할 수 있는지 검증한다. |
| 9 | `25a495d2cd43` | `refactor(db): 경기 결과 확정 boundary 일원화` | A | PERSISTENCE, TOURNAMENT, RISK | 우회 가능한 별도 대진 완료 API를 제거하고 모든 결과를 `finalizeMatch`로 통과시킨다. |
| 10 | `1646034acd9f` | `test(db): 경기 결과 확정 boundary 적용 검증` | B | PERSISTENCE, TOURNAMENT, TEST | repository surface에 `finalizeMatch`만 남았음을 검증한다. |

## 1. 최초 연결은 정상 경로만 이어 붙였다

### `33b6dfc5df7a` — persisted match를 realtime room으로 전환

`joinTournamentMatch`는 match가 `ready`인지, 요청자가 left/right 참가자인지, 이미 room에 들어가 있지 않은지 확인합니다. 같은 match ID의 waiter 두 명이 모이면 persisted left/right 위치에 맞춰 client를 배치하고 `createRoom`을 호출한 뒤 `repo.startTournamentMatch(matchId, roomId)`를 실행합니다.

종료 경로도 연결됐지만 두 단계였습니다.

```text
room finish
  → 일반 match 생성 및 rating 반영
  → completeTournamentMatch로 bracket row 연결
```

이 구조에는 서로 다른 두 결함이 있었습니다.

- `createRoom`은 client에 room ID를 보내고 내부 map에 등록한 뒤 repository start를 호출합니다. start가 실패하면 공개된 room이 남습니다.
- 일반 match 저장이 성공하고 tournament 연결이 실패하면 rating과 match만 반영된 채 bracket은 running에 머뭅니다.

즉 이 commit은 통합 지점을 만들었지만, 그 지점을 하나의 전이처럼 다루지는 못했습니다.

## 2. 종료는 저장소 transaction 하나로 수렴한다

### `e338ea32b2a6` — `finalizeMatch` 안에서 tournament 진행까지 잠그기

이 S급 commit은 기존 `finalizeMatch` transaction에 tournament 연동을 포함합니다. transaction은 result key에 의한 중복 결과 방지와 일반 match/rating 반영뿐 아니라, tournament match와 tournament row를 `FOR UPDATE`로 잠급니다.

대진 연동의 핵심은 다음 순서입니다.

```ts
select id, tournament_id, round, match_id, left_user_id, right_user_id
from tournament_matches
where id = ${command.tournament.tournamentMatchId}
for update
```

그 뒤 이미 `match_id`가 있으면 중복 확정을 거부하고, winner/loser가 실제 left/right 참가자인지 검사합니다. 연결 update도 `match_id is null` 조건과 `returning id`를 사용합니다.

```ts
update tournament_matches
set status = 'finished',
    room_id = ${command.tournament.roomId},
    match_id = ${matchId},
    winner_id = ${command.winnerId},
    score_left = ${command.scoreLeft},
    score_right = ${command.scoreRight},
    updated_at = now()
where id = ${command.tournament.tournamentMatchId}
  and match_id is null
returning id
```

준결승이면 같은 transaction 안에서 finished semifinal winner 두 명을 slot 순서로 읽고 final slot 1을 `on conflict ... do nothing`으로 만듭니다. 결승이면 tournament row를 finished/winner로 바꿉니다.

따라서 다음 변경은 함께 commit되거나 함께 rollback됩니다.

- generic match row
- 양 참가자의 rating·승패
- tournament match의 score/winner/match/room 연결
- final 생성 또는 tournament 우승 확정

이 commit이 해결하는 것은 저장소 내부 원자성입니다. 같은 room에서 완료 처리가 여러 번 호출되는 process-local 중복은 GameHub 쪽에서 별도로 막아야 합니다.

### `582a1615a2c6` — 세 종류의 중복과 rollback을 분리해 검증

PostgreSQL integration test는 서로 다른 실패를 한 테스트로 뭉개지 않습니다.

- 같은 finalize command를 20회 동시에 호출해 generic match와 rating이 한 번만 반영되는지 확인
- tournament linkage가 성립하지 않는 명령에서 앞서 만든 match/rating까지 rollback되는지 확인
- 두 준결승을 동시에 끝내도 final row가 하나만 생기는지 확인

첫 번째는 result key/idempotency, 두 번째는 transaction 원자성, 세 번째는 row lock+unique slot을 겨냥합니다. 테스트 source를 검사했지만 실제 DB test runner는 실행하지 않았습니다.

### `10bf15723591` — GameHub의 in-flight 중복을 하나의 promise로 합치기

room 종료 이벤트가 여러 tick·socket 경로에서 겹칠 수 있으므로 room에 `finishing` promise를 보관하고 동일한 완료 작업을 공유합니다. repository에 전달하는 result key는 room을 기준으로 고정됩니다.

```text
resultKey = room:<roomId>:finished
```

GameHub promise는 **한 process에서 동시에 시작되는 호출을 합치는 역할**, PostgreSQL result key와 transaction은 **재시도·중복 process를 포함해 저장 결과를 한 번만 만드는 역할**입니다. 둘은 대체 관계가 아닙니다.

## 3. 시작 실패를 성공처럼 보이지 않게 만들기

### `916683099ecd` — zero-row update를 실패로 승격

기존 `startTournamentMatch`는 조건에 맞는 row가 없어도 SQL 실행 자체가 성공하면 반환했습니다. 존재하지 않는 ID, 이미 running/finished인 row, ready가 아닌 row가 모두 “시작 성공”처럼 보일 수 있었습니다.

수정은 update에 `returning id`를 붙이고 결과 row가 없으면 예외를 발생시킵니다. 이제 GameHub는 durable start의 실패를 관찰할 수 있습니다. 이 관찰 가능성이 없으면 보상 롤백도 실행할 수 없습니다.

### `480e2dc48028` — 미갱신 거부를 integration regression으로 고정

임의의 존재하지 않는 tournament match ID로 start를 호출해 rejection을 기대합니다. 작은 테스트지만 “SQL이 오류 없이 실행됐다”와 “요구한 상태 전이가 실제 발생했다”를 구분합니다.

## 4. 이미 공개한 room을 역순으로 되돌리기

### `38312bcaf632` — room creation 뒤 start 실패 보상

두 참가자를 room에 넣은 뒤 repository start가 실패할 수 있으므로 호출부가 room을 다시 찾아 `abandonRoom`에 넘깁니다.

```ts
const roomId = this.createRoom(left, right, {
  ai: false,
  mode: "tournament",
  tournamentMatchId: matchId
});
try {
  await this.repo.startTournamentMatch(matchId, roomId);
} catch (error) {
  const room = this.rooms.get(roomId);
  if (room) this.abandonRoom(room);
  throw error;
}
```

보상 순서가 중요한 이유는 `createRoom`이 단순 객체 생성이 아니기 때문입니다. room map 등록, client의 `roomId`, scheduler/timer 등록, presence와 초기 event publication이 함께 일어날 수 있습니다. 실패 후 이 소유권을 남기면 두 client는 존재하지 않는 durable match에 묶이고 재시도도 막힙니다.

`abandonRoom`은 완료 결과를 저장하지 않고 room scheduler/timer를 제거하고, room map과 client association을 정리합니다. 예외는 다시 던져 호출자가 시작 실패를 성공으로 오해하지 않게 합니다.

### `4e2cb4ae702d` — cleanup뿐 아니라 retryability를 검증

fake socket과 repository spy를 이용해 첫 `startTournamentMatch`만 실패시킵니다. 테스트는 다음을 함께 관찰합니다.

- 실패 뒤 active room/scheduler가 0
- client가 stale room을 소유하지 않음
- 같은 persisted match에 다시 두 참가자가 들어오면 두 번째 start가 성공

“room이 없어졌다”만 확인하면 충분하지 않습니다. waiter/client 상태가 잘못 남으면 cleanup 뒤에도 재시작할 수 없기 때문입니다. 이 테스트 역시 source만 검사했고 실행하지 않았습니다.

## 5. 우회 경로 제거

### `25a495d2cd43` — 별도 `completeTournamentMatch` 삭제

원자적 `finalizeMatch`가 생긴 뒤에도 repository interface에 별도 대진 완료 함수가 남아 있으면 caller가 generic match/rating transaction을 거치지 않고 bracket만 진행시킬 수 있습니다. 또는 반대로 일반 결과만 저장한 뒤 별도 호출에 실패하는 옛 패턴이 재도입될 수 있습니다.

이 commit은 PostgreSQL·memory 구현과 interface에서 escape hatch를 제거합니다. 리팩터링의 목적은 이름 정리가 아니라 **불변 조건을 API 표면으로 강제하는 것**입니다.

### `1646034acd9f` — 좁아진 surface를 테스트

repository가 `finalizeMatch`를 제공하고 `completeTournamentMatch`는 제공하지 않는다는 구조적 regression을 추가합니다. 런타임 결과 테스트보다 가볍지만, 잘못된 우회 API가 다시 생기는 것을 조기에 드러냅니다.

## 최종 전이

```text
[두 tournament participant가 같은 match waiter에 모임]
        ↓
[GameHub가 room 생성·client 연결·publication]
        ↓
[repository start: ready row를 running+room으로 갱신]
   ├─ 성공 → room 사용 가능
   └─ 실패 → abandonRoom으로 scheduler/client/map 역순 정리 → 예외
        ↓
[room 종료]
        ↓
[한 in-flight promise가 finalizeMatch command 생성]
        ↓
[PostgreSQL transaction:
 result-key 중복 확인
 → 일반 match/rating
 → tournament match lock·participant 검증·연결
 → final 생성 또는 tournament winner 확정]
        ↓
[전체 commit 또는 전체 rollback]
```

| 실패 지점 | 최종 상태 |
| --- | --- |
| persisted start가 0-row | room 보상 폐기, match는 ready, 재시도 가능 |
| finalize 중 tournament match 없음/이미 연결됨 | generic match와 rating도 rollback |
| 같은 room 완료가 중복 호출 | GameHub promise 공유, 저장소 result key로 한 결과 |
| 두 준결승이 동시에 완료 | tournament lock과 unique final slot으로 final 하나 |

이 Thread는 process memory와 DB 사이에 분산 transaction을 만든 것이 아닙니다. 시작에는 명시적 보상, 종료에는 durable owner로의 단일 인계를 사용해 같은 사용자 관찰 결과로 수렴시킨 것입니다.

## 조사 범위

각 설명은 `web/ft_transcendence`의 표시 SHA diff와 해당 시점 source를 기준으로 작성했습니다. integration/unit/smoke test는 실행하지 않았습니다.
