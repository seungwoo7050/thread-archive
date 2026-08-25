# NPC를 저장된 사용자로 만들고 대기열 fallback까지 연결하기

원문 Thread: `NPC AI policy and fallback journey`

## 이 Thread가 다루는 문제

AI 상대를 room snapshot의 `"ai"` 문자열로만 표현하면 화면에는 이름을 보여 줄 수 있어도 다음 질문에 답할 수 없습니다.

- 어떤 NPC가 상대였는가?
- 그 NPC의 rating은 얼마였고 어떤 AI policy를 써야 하는가?
- 일반 match 결과의 winner/loser ID에 누구를 연결하는가?
- profile·leaderboard에서 human과 어떻게 구분하는가?

반대로 NPC를 저장된 사용자로 만들더라도, queue fallback timer를 취소하지 않으면 이미 human과 매칭된 client에게 몇 초 뒤 두 번째 AI room이 생길 수 있습니다.

이 Thread는 durable NPC identity와 queue timer ownership을 연결합니다. 다만 exact source를 대조한 결과, 기존 문서가 암시한 것과 달리 **명시적 `mode="ai"` 경로는 마지막 포함 SHA에서도 persisted NPC를 전달하지 않습니다.** persisted NPC가 실제 room에 들어가는 경로는 `1122e6a4b901`의 6초 human-queue fallback입니다. importance와 tags는 유지하되 역할 설명은 이 사실에 맞게 좁혔습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `72d23baefc3c` | `feat(db): NPC 사용자 contract와 schema 추가` | B | REALTIME, PERSISTENCE, TOURNAMENT | `is_npc`를 user storage와 모든 공유 user projection에 전파한다. |
| 2 | `b3239bae51e5` | `feat(db): rating 구간별 NPC 상대 저장` | B | REALTIME, PERSISTENCE | 1100–1400 NPC seed와 별도 opponent 조회를 PostgreSQL/memory에 구현한다. |
| 3 | `dec431822873` | `test(db): NPC seed와 leaderboard 분리 검증` | B | REALTIME, PERSISTENCE, TEST | rating 순 NPC 목록이 NPC·offline projection으로 노출되는지 검증한다. |
| 4 | `87b38e2f23c8` | `feat(game): NPC 상대를 경기 방에 연결` | B | REALTIME | room이 optional persisted NPC identity를 표시·결과에 사용할 수 있게 준비한다. 이 SHA의 direct AI caller는 아직 NPC를 전달하지 않는다. |
| 5 | `1122e6a4b901` | `feat(game): 대기 플레이어 NPC fallback 구성` | B | REALTIME | 6초 queue fallback에서 가장 가까운 NPC를 실제 room에 전달하고 timer lifetime을 관리한다. |
| 6 | `b159bcda3b83` | `feat(game): rating 기반 NPC AI policy 구현` | B | SIMULATION, REALTIME, WEB | NPC rating을 deterministic 반응·예측·실수·속도·dead-zone profile로 바꾼다. |
| 7 | `afd0a97c5c1c` | `feat(web): 대기열에서 NPC 상대 표시` | B | REALTIME, WEB | queue fallback과 AI identity를 home·play·profile·leaderboard에 표시한다. |
| 8 | `cfb15fc84dee` | `test(app): NPC fallback matching 검증` | B | PROTOCOL, REALTIME, PERSISTENCE | solo queue가 6초 뒤 `npc-*` identity를 가진 AI room snapshot으로 이어지는지 smoke 경로에서 검증한다. |

## 1. NPC 여부를 user identity의 일부로 만들기

### `72d23baefc3c` — `is_npc`의 전파

users schema에 다음 column이 추가됩니다.

```sql
is_npc boolean not null default false
```

기존 DB에도 적용할 수 있도록 `alter table ... add column if not exists`가 함께 들어갑니다. shared `PublicUser`에는 `isNpc`가 추가되고 row mapper, chat sender projection, tournament creator/participant projection도 `is_npc`를 선택·변환합니다.

이 변경의 의미는 NPC 여부가 room 내부의 일시적 flag가 아니라 user row의 속성이 됐다는 것입니다. 기존 human row는 default false이며 memory dev user도 false로 생성됩니다.

아직 실제 NPC row는 없고, 상대 선택이나 AI policy도 없습니다.

### `b3239bae51e5` — rating band별 seed와 별도 조회

네 NPC가 seed됩니다.

| handle | 표시 이름 | rating |
| --- | --- | ---: |
| `npc-rally-1100` | AI 랠리 1100 | 1100 |
| `npc-block-1200` | AI 블록 1200 | 1200 |
| `npc-spin-1300` | AI 스핀 1300 | 1300 |
| `npc-smash-1400` | AI 스매시 1400 | 1400 |

PostgreSQL `upsertNpc`는 email null, active, `is_npc = true`로 저장하고 충돌 시 display/avatar/rating/status/marker를 NPC 값으로 갱신합니다. 반대 방향도 중요합니다. dev login이 같은 handle을 사용하면 upsert가 `is_npc = false`로 되돌립니다. handle 충돌로 실제 로그인 사용자가 NPC로 오염되는 것을 막는 규칙입니다.

`listNpcOpponents`는 active NPC만 rating 오름차순으로 반환하며 `online: false` projection을 사용합니다. NPC가 게임에 사용할 수 있는 identity라는 사실과 WebSocket presence를 가진 사용자는 아니라는 사실을 분리합니다. memory repository도 같은 네 seed와 정렬을 구현합니다.

### `dec431822873` — seed contract 검증

memory test는 ratings가 `[1100, 1200, 1300, 1400]` 순서인지, 모두 `isNpc === true`, `online === false`인지 확인합니다.

commit 제목의 “leaderboard 분리”를 NPC가 leaderboard에서 제외된다는 뜻으로 읽으면 안 됩니다. 실제 `listLeaderboard`는 이후에도 users 전체를 읽고, UI는 NPC에 AI badge를 표시합니다. 이 test가 직접 고정하는 것은 **상대 선택용 NPC 목록과 presence projection**입니다.

## 2. room이 persisted NPC를 받을 준비를 하다

### `87b38e2f23c8` — optional NPC identity와 결과 attribution 기반

`createRoom` options에 `npc?: PublicUser | null`이 추가되고 right player는 human socket user, supplied NPC, generic fallback 순서로 결정됩니다.

```ts
const rightPlayer = right?.user ?? options.npc ?? null;
```

snapshot의 right player ID/handle/displayName과 `queue.matched.opponent`가 이 identity를 사용합니다. finish path도 AI opponent를 winner/loser로 연결할 수 있도록 user-like ID를 다루는 형태로 바뀝니다. `findClosestNpc`는 rating 절대 차이가 가장 작은 첫 NPC를 찾습니다. 목록이 rating 오름차순이므로 동률이면 낮은 rating 쪽이 먼저 선택됩니다.

그러나 **이 exact SHA에서 `findClosestNpc`는 호출되지 않습니다.** `joinQueue`의 direct AI branch는 여전히 다음과 같습니다.

```ts
if (mode === "ai") {
  this.createRoom(client, null, { ai: true, mode: "ai" });
  return;
}
```

따라서 이 commit은 persisted NPC를 실제 room에 배정한 완성점이 아니라, room과 result path가 optional NPC를 받을 수 있게 준비한 단계입니다. `NPC_QUEUE_FALLBACK_MS` 상수도 이 시점에 생기지만 아직 timer가 없습니다.

## 3. queue entry와 fallback timer를 하나의 수명으로 묶기

### `1122e6a4b901` — 6초 뒤에도 같은 대기 상태일 때만 NPC 배정

human queue에서 적합한 상대가 없으면 `QueueEntry`에 timer를 붙입니다.

```ts
const entry: QueueEntry = {
  client,
  queuedAt: Date.now(),
  npcFallbackTimer: null
};

entry.npcFallbackTimer = setTimeout(() => {
  this.matchQueuedClientWithNpc(entry).catch((error) => {
    this.send(client, {
      type: "error",
      message: error instanceof Error
        ? error.message
        : "AI 상대를 찾지 못했습니다."
    });
  });
}, 6000);

this.queue.push(entry);
```

timer callback은 자신이 예약될 때의 상태를 신뢰하지 않습니다. 발화한 시점에 다시 확인합니다.

```ts
const index = this.queue.findIndex(
  (queued) => queued.client.id === entry.client.id
);
if (index < 0
    || entry.client.socket.readyState !== WebSocket.OPEN
    || entry.client.roomId) return;
```

그 뒤 가장 가까운 NPC가 있을 때만 queue에서 entry를 제거하고 timer를 clear하고 wait sample을 기록한 뒤 다음처럼 room을 만듭니다.

```ts
this.createRoom(queued.client, null, {
  ai: true,
  mode: "queue",
  npc
});
```

이 호출이 포함 history에서 persisted NPC가 실제 room에 전달되는 최초 지점입니다. room은 `npcUser`를 보관하고 finish 결과의 right user로 사용합니다.

### timer cancellation path

| queue entry가 끝나는 이유 | 처리 |
| --- | --- |
| human opponent가 도착 | queue에서 제거한 opponent의 timer clear |
| 사용자가 queue.leave | entry 제거 후 timer clear |
| socket close/prune | entry 제거 후 timer clear |
| fallback callback이 NPC를 배정 | entry 제거 후 timer clear |
| callback 전에 다른 room 보유 | callback revalidation에서 no-op |

`clearQueueTimer`는 clearTimeout 뒤 field를 null로 바꿉니다. 이 정리가 없으면 human room이 이미 만들어진 뒤 오래된 callback이 다시 AI room을 만들거나, disconnect된 socket을 대상으로 repository 조회와 error send를 수행할 수 있습니다.

### direct AI 경로의 남은 차이

같은 SHA에서 `mode === "ai"`는 여전히 `npc` 없이 generic AI room을 즉시 만듭니다. room의 `npcUser`가 null이므로 finish 시 AI 쪽 user ID도 null로 처리됩니다. queue fallback과 direct AI는 둘 다 AI simulation을 사용하지만 identity attribution은 같지 않습니다.

## 4. rating을 재현 가능한 AI policy로 바꾸기

### `b159bcda3b83` — profile table

기존 AI는 매 tick ball Y와 paddle center만 비교했습니다. 새 policy는 NPC rating에 따라 다섯 값을 정합니다.

| rating | reaction ticks | prediction noise | mistake chance | speed multiplier | dead zone |
| ---: | ---: | ---: | ---: | ---: | ---: |
| ≥ 1400 | 3 | 20 | 0.04 | 1.05 | 10 |
| ≥ 1300 | 4 | 34 | 0.08 | 0.96 | 14 |
| ≥ 1200 | 6 | 54 | 0.12 | 0.86 | 18 |
| < 1200 | 8 | 78 | 0.18 | 0.74 | 24 |

`updateAiPaddleIntent`는 reaction tick에서만 target을 다시 계산합니다. 공이 오른쪽으로 갈 때는 오른쪽 벽까지 이동한 뒤 위·아래 벽에 반사될 예상 Y를 구하고, 왼쪽으로 갈 때는 중앙으로 복귀합니다. prediction noise와 occasional mistake offset을 더한 뒤 playable range로 clamp합니다.

난수처럼 보이는 variation은 ambient `Math.random()`이 아닙니다. room ID를 hash하고 tick과 salt를 섞은 deterministic 값입니다.

```ts
function deterministicUnit(
  seed: string,
  tick: number,
  salt: number
): number {
  const value = Math.sin(
    hashString(seed) * 0.001
      + tick * 12.9898
      + salt * 78.233
  ) * 43758.5453;
  return value - Math.floor(value);
}
```

같은 room·tick·salt면 같은 선택을 하므로 replay와 test에서 reasoning이 가능합니다.

queue fallback room은 selected NPC rating을 사용합니다. direct AI room은 `npcUser`가 null이므로 default rating 1200 profile을 사용합니다. 이 차이도 포함 commit 범위 안의 실제 최종 상태입니다.

## 5. 사용자에게 AI identity와 fallback을 숨기지 않기

### `afd0a97c5c1c` — queue entry point와 AI 표시

홈의 human queue 링크는 `/play?mode=queue`로 바뀌고 “상대가 없으면 AI를 배정한다”고 설명합니다. play page는 queue mode도 자동 연결하고 snapshot right player의 `ai` flag를 보고 AI 상대임을 표시합니다.

`PublicUser.isNpc`를 소비하는 화면도 바뀝니다.

- leaderboard: 이름 옆 `AI` badge
- public profile: `AI 상대` badge
- NPC profile: 친구 추가 버튼 disabled

이 commit은 server authorization이 아닙니다. 버튼 disabled는 UX 제약이고, friend request API가 NPC 요청을 반드시 거부한다는 증거는 이 Thread commit에 없습니다. 또한 leaderboard에서 NPC를 숨기지 않고 명시적으로 표시합니다.

### `cfb15fc84dee` — solo queue end-to-end smoke

smoke script는 새 사용자를 로그인시키고 WebSocket으로 `queue.join`, mode `queue`를 보냅니다. 최대 8초 안에 opponent 문자열에 `AI`가 들어간 `queue.matched`를 기다린 뒤 ready를 보내고, snapshot에서 다음을 확인합니다.

- `players.some(player => player.ai)`
- 그 AI player의 handle이 `npc-`로 시작

6초 fallback delay보다 긴 8초 timeout을 별도로 사용합니다. 이 테스트는 login → socket → queue timer → repository NPC lookup → room snapshot 연결을 통과합니다.

증명하지 않는 것도 명확합니다.

- explicit `mode="ai"`가 persisted NPC를 쓰는지
- closest rating이 정확히 선택됐는지
- timer cancellation의 모든 race
- AI profile이 실제로 의도한 난이도 차이를 만드는지
- match 종료 뒤 rating/result attribution

source는 확인했지만 smoke environment는 실행하지 않았습니다.

## 최종 경로 비교

| 경로 | room 생성 시 NPC | AI policy rating | 결과의 AI user ID | smoke coverage |
| --- | --- | ---: | --- | --- |
| human queue, 6초 안에 human 발견 | 없음 | 해당 없음 | human ID | 기존 human match 경로 |
| human queue, 6초 후 fallback | 가장 가까운 persisted NPC | NPC rating | NPC ID | `cfb15fc84dee` |
| explicit `mode="ai"` | 없음, generic AI | default 1200 | null | 이 Thread에서 별도 검증 없음 |

최종적으로 보장되는 것은 **queue fallback의 persisted identity와 timer 소유권**입니다. explicit AI mode까지 같은 identity 경로로 일원화됐다고 주장하면 exact source를 넘어섭니다.

## 조사 범위

각 설명은 `web/ft_transcendence`의 표시 SHA diff와 해당 시점 source를 기준으로 작성했습니다. repository/unit/smoke test는 실행하지 않았습니다.
