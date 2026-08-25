# 토너먼트 대진은 참가자 목록이 아니라 저장된 경기다

원문 Thread: `Tournament contract, schema, and bracket construction`

## 이 Thread가 다루는 문제

초기 토너먼트는 `tournaments`와 `tournament_entries`만으로도 참가 신청과 정원 마감까지는 표현할 수 있었습니다. 그러나 참가자 목록만으로는 “누가 누구와 준결승을 치르는가”, “어느 경기가 준비·진행·종료 상태인가”, “어떤 realtime room과 일반 경기 결과가 이 대진에 연결됐는가”를 저장할 수 없습니다.

첫 화면은 이 공백을 참가 순서와 sample 데이터로 메웠습니다. 브라우저가 `entries.slice(...)`로 준결승처럼 보이는 배열을 만들었기 때문에 같은 토너먼트를 읽는 다른 소비자가 다른 대진을 계산할 수도 있었고, 새로고침 뒤 room·score·winner를 복원할 방법도 없었습니다.

이 Thread의 도달점은 명확합니다.

> 대진은 화면이 참가자 목록에서 추정하는 표현이 아니라, repository가 한 번 생성하고 모든 소비자가 읽는 `tournament_matches`의 영속 상태다.

4인 대회는 4번째 참가가 확정된 뒤 seed 1–4, 2–3으로 준결승 두 경기를 만들고, 결승은 두 준결승이 모두 winner를 가진 `finished` 상태가 된 뒤에만 생성됩니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `34c80874f13f` | `feat(db): 토너먼트 row contract 정의` | B | PERSISTENCE, TOURNAMENT | DB row를 공유 토너먼트 요약으로 바꿀 typed 표현을 마련한다. |
| 2 | `9b1dabcc4bb4` | `feat(db): 토너먼트 참가 저장 구현` | B | PERSISTENCE, TOURNAMENT | 생성·목록·참가를 공통 repository에 구현한다. |
| 3 | `4370ac3162b2` | `feat(web): 토너먼트 대진표 화면 추가` | B | TOURNAMENT, WEB | 첫 토너먼트 화면을 만들지만 대진은 참가 순서에서 임시로 구성한다. |
| 4 | `11e4c3dda1aa` | `feat(tournament): 대진 경기 contract 정의` | B | REALTIME, TOURNAMENT, WEB | 라운드·슬롯·참가자·점수·방·결과 연결을 가진 대진 경기 요약을 정의한다. |
| 5 | `138e5b8590b6` | `feat(tournament): 대진 경기 schema 추가` | B | REALTIME, PERSISTENCE, TOURNAMENT | `tournament_matches`를 독립 영속 모델로 추가한다. |
| 6 | `4021a437e7e0` | `feat(tournament): 대진 row mapper 정의` | B | REALTIME, PERSISTENCE, TOURNAMENT | DB 대진 row를 내부 record와 공개 summary로 변환한다. |
| 7 | `53579ad0f0bf` | `feat(tournament): 대진 경기 lifecycle 저장 구현` | A | REALTIME, PERSISTENCE, TOURNAMENT | 대진 조회·시작·완료 연산을 PostgreSQL과 memory repository에 추가한다. |
| 8 | `0d6824683677` | `feat(tournament): 준결승 대진 생성과 조회 구현` | A | TOURNAMENT | 4번째 참가 시 1–4·2–3 준결승을 한 번만 만들고 요약에서 저장된 대진을 읽는다. |
| 9 | `b01adf728ca0` | `feat(tournament): memory 대진 진행 구현` | B | PERSISTENCE, TOURNAMENT | memory 구현의 대진 생성·결승 진출·우승 처리를 PostgreSQL과 맞춘다. |
| 10 | `b0a1505c6a0f` | `feat(tournament): 플레이 가능한 대진 UI 연결` | B | PROTOCOL, REALTIME, TOURNAMENT | 임시 대진을 제거하고 저장된 match와 실시간 경기 진입을 연결한다. |

## 1. 참가 저장은 있었지만 대진 상태는 없었다

### `34c80874f13f` — DB row와 공개 요약의 변환 경계

`packages/db/src/schema.ts`와 `rowMappers.ts`에 tournament/entry row 타입과 `TournamentWithCreatorRow`가 생기고, shared HTTP contract의 `TournamentSummary`로 바꾸는 경계가 마련됐습니다. 이 commit은 lifecycle을 구현하지 않습니다. DB의 snake_case row와 application의 camelCase 객체를 혼용하지 않도록 **표현 책임을 database package에 둔 것**이 핵심입니다.

당시 요약이 표현할 수 있는 것은 토너먼트 ID·이름·상태·정원·생성자·참가자·우승자 정도입니다. round, slot, room, score, 일반 match 연결은 아직 존재하지 않습니다.

### `9b1dabcc4bb4` — 생성·목록·참가의 저장

공통 `AppRepository`에 토너먼트 생성·목록·참가가 추가되고 PostgreSQL과 memory 구현이 이를 따릅니다. 생성자는 seed 1 참가자로 들어가고, 이후 참가자는 현재 entry 수를 기준으로 다음 seed를 받습니다. 정원에 도달하면 status가 `running`으로 바뀝니다.

여기서 `running`은 “실제 준결승 row가 준비됐다”는 뜻이 아니었습니다. 저장된 사실은 여전히 참가자와 seed뿐입니다. 따라서 이 상태만으로는 경기 단위의 독립 lifecycle을 시작할 수 없습니다.

### `4370ac3162b2` — 화면이 공백을 대진처럼 꾸몄다

첫 토너먼트 페이지는 API 목록·생성 기능을 연결했지만, 대진은 저장소에서 오지 않았습니다. 참가자 배열 일부를 잘라 카드에 배치하고 초기 state에는 sample 토너먼트를 넣었습니다. 결과적으로 화면은 “대진표가 있다”고 보이지만 실제로는 다음 사실을 저장하지 못했습니다.

- round와 slot
- 경기별 status
- left/right participant의 고정된 배치
- room ID와 일반 match ID
- 점수와 winner

이 commit은 UI 기능의 시작점이지만, 이 Thread에서는 이후 교정되는 **독립 재구성의 선행 상태**로 다룹니다.

## 2. 대진 경기를 독립 모델로 만들기

### `11e4c3dda1aa` — 공유 contract가 대진의 수명을 표현하기 시작했다

`TournamentSummary`에 `matches`가 추가되고, 각 match는 tournament ID, round, slot, status, left/right user, winner, score, room ID, persisted match ID를 가집니다. 또한 realtime 진입을 위한 `tournament.join` client event가 추가됩니다.

중요한 구분은 세 식별자의 수명입니다.

| 식별자 | 의미 |
| --- | --- |
| tournament match ID | bracket 안의 한 경기 자체. 준비 전부터 종료 뒤까지 유지 |
| room ID | 실제 realtime 세션. 경기가 시작될 때 생기고 실패 시 폐기될 수 있음 |
| generic match ID | 최종 점수와 rating 반영을 가진 일반 경기 결과. 종료 확정 때 연결 |

이 구분이 없으면 “방은 만들어졌지만 저장 경기는 시작되지 않은 상태”나 “일반 결과는 저장됐지만 bracket에는 연결되지 않은 상태”를 탐지하기 어렵습니다.

### `138e5b8590b6` — `tournament_matches` schema

전용 table이 round, slot, status, 양쪽 참가자, winner, room, 일반 match, 점수를 소유합니다. `(tournament_id, round, slot)` unique 제약은 같은 준결승·결승 자리가 중복 생성되는 것을 막습니다.

이 schema는 참가자 admission과 경기 lifecycle을 분리합니다. entry는 “대회에 누가 어떤 seed로 참가했는가”를, match row는 “그 seed들이 어떤 경기로 진행되고 있는가”를 답합니다.

### `4021a437e7e0` — row mapper로 storage와 public model을 맞추기

DB row의 nullable user ID와 timestamp, status를 내부 `TournamentMatchRecord`와 공개 `TournamentMatchSummary`로 변환하는 mapper가 추가됩니다. left/right/winner user projection을 mapper 호출자가 제공하게 해, SQL row와 사용자 공개 정보의 결합 위치를 명시했습니다.

작은 commit이지만 이 경계가 없으면 PostgreSQL과 memory 구현, HTTP 응답이 각자 필드 이름과 null 처리 규칙을 다시 작성하게 됩니다.

## 3. 대진의 lifecycle을 repository가 소유하게 만들기

### `53579ad0f0bf` — 조회·시작·완료 연산

`AppRepository`와 두 구현에 다음 책임이 추가됩니다.

- ID로 tournament match 조회
- ready match를 room에 연결하고 running으로 변경
- 점수·winner·일반 match ID를 기록하고 finished로 변경
- 두 준결승이 끝난 경우 결승 준비
- 결승 종료 시 tournament winner와 finished 상태 반영

이 commit은 match row가 단순 read model이 아니라 **독립적으로 상태가 전이되는 aggregate**가 된 지점입니다. 다만 당시 PostgreSQL start update가 실제 row를 바꿨는지 확인하지 않았고, 종료는 일반 match 저장과 별도 대진 완료 호출로 나뉘어 있었습니다. 이 두 결함은 다음 Thread에서 각각 `916683099ecd`, `e338ea32b2a6` 계열로 다뤄집니다.

### `0d6824683677` — 4번째 참가 시 준결승을 한 번만 생성

PostgreSQL 참가 경로는 정원과 기존 참가 여부를 검사하고, entry 저장 뒤 bracket 생성을 호출합니다. 실제 핵심 코드는 다음과 같습니다.

```ts
private async ensureTournamentBracket(tournamentId: string): Promise<void> {
  const entries = await sql<{ user_id: string; seed: number }>`
    select user_id, seed from tournament_entries
    where tournament_id = ${tournamentId}
    order by seed asc
  `.execute(this.db);
  if (entries.rows.length < 4) return;

  await sql`
    insert into tournament_matches (
      tournament_id, round, slot, left_user_id, right_user_id, status
    )
    values
      (${tournamentId}, 'semifinal', 1,
       ${entries.rows[0].user_id}, ${entries.rows[3].user_id}, 'ready'),
      (${tournamentId}, 'semifinal', 2,
       ${entries.rows[1].user_id}, ${entries.rows[2].user_id}, 'ready')
    on conflict (tournament_id, round, slot) do nothing
  `.execute(this.db);
}
```

여기에는 세 가지 결정이 함께 있습니다.

1. 참가자가 4명보다 적으면 대진 row를 만들지 않습니다.
2. seed 순서를 기준으로 1–4, 2–3을 고정합니다.
3. unique key와 `on conflict ... do nothing`으로 재호출해도 같은 slot을 중복 생성하지 않습니다.

토너먼트 조회도 이제 `tournament_matches`를 round/slot 순서로 읽고 각 row의 left/right/winner를 user projection과 결합합니다. 화면이나 GameHub가 참가자 순서를 다시 해석할 필요가 없어졌습니다.

### `b01adf728ca0` — memory 구현의 동작 일치

memory repository도 4번째 참가에서 같은 두 준결승을 만들고, 준결승 winner 두 명으로 결승을 만들며, 결승 종료 시 tournament winner를 기록합니다. PostgreSQL과 memory가 같은 interface를 구현한다는 사실만으로는 충분하지 않습니다. **같은 입력에서 같은 상태 전이와 같은 대진 배치**가 나와야 테스트 대역이 실제 동작을 왜곡하지 않습니다.

이 commit은 그 동작 일치를 채웁니다. 다만 동시 transaction과 row lock은 memory 구현이 재현하지 못하는 PostgreSQL 전용 성질입니다.

## 4. 소비자가 저장된 대진만 읽게 만들기

### `b0a1505c6a0f` — placeholder bracket 제거와 play 진입

토너먼트 페이지는 참가자 배열을 잘라 만든 카드를 버리고 `TournamentSummary.matches`를 round/slot 기준으로 렌더링합니다. 현재 사용자가 ready match의 실제 left/right 참가자인 경우에만 그 match ID를 가진 play 링크를 노출합니다. play 화면은 해당 ID로 `tournament.join`을 보내 realtime 대기 경로에 들어갑니다.

이 변경으로 다음 세 소비자가 같은 사실을 공유합니다.

- repository: bracket 생성과 lifecycle 저장
- web: 저장된 match의 상태·참가자·점수 표시
- GameHub: 동일한 tournament match ID로 두 참가자를 room에 결합

화면에서 그럴듯한 대진을 만드는 책임은 완전히 사라집니다.

## 최종 상태와 보장 범위

| 상태 | 저장된 대진 |
| --- | --- |
| 참가자 1–3명 | 없음. entry와 seed만 존재 |
| 4번째 참가 확정 | semifinal slot 1 = seed 1 vs 4, slot 2 = seed 2 vs 3 |
| 준결승 하나 종료 | 해당 row만 finished. 결승 없음 |
| 준결승 둘 모두 종료 | 두 winner로 final slot 1을 한 번만 생성 |
| 결승 종료 | final row와 tournament에 winner/finished 반영 |

이 Thread가 보장하는 핵심은 **대진의 단일 출처**입니다. 참가자 admission의 강한 동시성 보장, room publication 실패 롤백, 결과·rating·다음 라운드의 원자적 확정은 이 문서의 모델 위에서 동작하지만 각각 별도 Thread의 책임입니다.

## 조사 범위

각 설명은 `web/ft_transcendence`의 표시 SHA diff와 해당 시점 source를 기준으로 작성했습니다. repository command와 test runner는 실행하지 않았습니다.
