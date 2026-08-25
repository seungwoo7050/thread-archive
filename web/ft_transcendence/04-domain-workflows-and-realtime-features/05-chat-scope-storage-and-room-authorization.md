# 채팅의 scope·room·audience를 같은 규칙으로 묶기

원문 Thread: `Chat scope storage and room authorization`

## 이 Thread가 다루는 문제

`scope: "match"`와 `roomId`가 있다는 사실만으로는 사용자가 그 방의 참가자라는 뜻이 아닙니다. 반대로 서버가 정상적으로 audience를 제한하더라도 잘못된 row가 DB에 직접 들어가거나, stale event가 브라우저 reducer에 섞일 수 있습니다.

따라서 채팅 경계는 한 곳의 `if`로 끝나지 않습니다.

1. wire schema가 가능한 payload 형태를 줄입니다.
2. repository와 DB가 저장 가능한 row 형태를 강제합니다.
3. GameHub가 sender의 현재 좌석을 저장 전에 확인합니다.
4. broadcast는 room audience 또는 global lobby audience를 선택합니다.
5. browser는 현재 active room에 맞는 message만 reducer에 반영합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `00d0d7941382` | `fix(protocol): 채팅 scope와 room 식별자 조합 제한` | A | PROTOCOL, REALTIME, WEB | lobby는 room 필드가 없고 match는 UUID room이 필수인 payload를 정의한다. |
| 2 | `5a3819aec8d0` | `test(protocol): 채팅 scope와 room 조합 검증` | B | AUTH, PROTOCOL, REALTIME | wire parse boundary의 허용·거부 조합을 고정한다. |
| 3 | `2ff750fa4ff8` | `fix(db): 채팅 행의 scope와 room 불변식 강제` | A | REALTIME, PERSISTENCE, RISK | legacy row를 정리하고 repository와 PostgreSQL CHECK에서 같은 불변식을 강제한다. |
| 4 | `1cead7cc9f35` | `test(db): 채팅 저장 불변식 검증` | B | REALTIME, PERSISTENCE, TEST | application rejection·migration cleanup·CHECK·migration idempotency를 검증한다. |
| 5 | `7759eef59b67` | `fix(game): 매치 채팅의 좌석과 audience 검증` | A | REALTIME, PERSISTENCE, RISK | 현재 그 room 좌석을 점유한 sender만 저장·room broadcast할 수 있게 한다. |
| 6 | `4a98bd1e4f22` | `test(game): 타 경기방 채팅 주입 차단 검증` | B | REALTIME, TEST | cross-room 요청이 persistence 전에 멈추고 audience가 섞이지 않는지 검증한다. |
| 7 | `85edd6d1e26a` | `fix(web): 현재 경기방의 채팅만 표시` | B | REALTIME, WEB | client reducer가 active room의 match message만 수용하게 한다. |
| 8 | `02775797ab63` | `test(web): 매치 채팅 room filtering 검증` | B | REALTIME, WEB, TEST | active·other·lobby·no-room 조합을 pure helper에서 고정한다. |

## 1. wire contract에서 잘못된 조합을 표현하지 못하게 하기

### `00d0d7941382` — `scope`별 payload 분리

이전 `chat.send`는 scope와 optional/nullable room을 독립적으로 받았습니다. lobby+room, match+missing room, match+임의 문자열이 모두 parser를 통과할 수 있었습니다.

수정은 chat event를 scope로 나눈 strict union으로 바꿉니다.

```ts
const chatClientEventSchema = z.discriminatedUnion("scope", [
  z.object({
    ...version,
    type: z.literal("chat.send"),
    scope: z.literal("lobby"),
    body: chatBodySchema
  }).strict(),
  z.object({
    ...version,
    type: z.literal("chat.send"),
    scope: z.literal("match"),
    roomId: z.string().uuid(),
    body: chatBodySchema
  }).strict()
]);
```

lobby payload는 `roomId: null`조차 허용하지 않고 필드 자체가 없어야 합니다. match는 non-null UUID가 필수입니다. GameHub는 lobby를 저장할 때 `roomId = null`로 정규화합니다.

이 계약은 형식만 증명합니다. 그 UUID room이 실제 존재하는지, sender가 참가자인지는 아직 모릅니다.

### `5a3819aec8d0` — invalid state space를 표로 고정

parser test는 다음을 모두 거부합니다.

- lobby + `roomId: null`
- lobby + UUID room
- match + room 필드 없음
- match + null room
- match + non-UUID string

정상 match fixture도 UUID로 바뀝니다. 이 테스트는 application/repository를 우회한 DB write까지 막지는 않습니다.

## 2. 저장 경계를 protocol과 같은 규칙으로 맞추기

### `2ff750fa4ff8` — migration cleanup, application assertion, DB CHECK

새 constraint를 설치하기 전에 기존 데이터를 처리합니다.

```sql
update chat_messages
set room_id = null
where scope = 'lobby' and room_id is not null;

delete from chat_messages
where scope not in ('lobby', 'match')
   or (
     scope = 'match'
     and (
       room_id is null
       or room_id !~* '^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$'
     )
   );
```

그 뒤 `chat_messages_scope_room_check`를 추가합니다.

```sql
check (
  (scope = 'lobby' and room_id is null)
  or (
    scope = 'match'
    and room_id is not null
    and room_id ~* '^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$'
  )
)
```

PostgreSQL과 memory repository의 `createChatMessage`도 공통 `assertChatRoom`을 호출합니다. protocol parser를 거치지 않는 내부 caller, test, admin script도 같은 application rule을 받습니다. DB CHECK는 application code를 우회한 write의 마지막 방어선입니다.

migration의 선택도 구분됩니다.

- lobby row의 불필요한 room은 안전하게 null로 정규화
- match의 room 부재/비UUID, unknown scope는 의미를 복구할 수 없어 삭제

### `1cead7cc9f35` — 저장 불변식의 네 증거

memory test는 잘못된 세 조합을 repository가 거부하는지 확인합니다. PostgreSQL integration test는 migration 005 상태에 legacy row를 직접 넣은 뒤 migration 006을 두 번 실행합니다.

검증 대상은 다음 네 가지입니다.

1. lobby+room이 null로 정규화됨
2. invalid match/unknown scope row가 삭제됨
3. 이후 raw invalid insert가 SQLSTATE `23514`로 거부됨
4. migration 기록이 한 번만 남음

테스트 source는 확인했지만 실제 PostgreSQL 환경에서 실행하지 않았습니다.

## 3. 저장 가능성과 전송 권한은 다른 문제다

### `7759eef59b67` — seat authorization을 persistence 전에 수행

UUID가 맞고 row 형태가 유효해도 다른 방 ID를 아는 사용자가 message를 보낼 수 있었습니다. GameHub는 match chat에서 다음 조건을 모두 확인합니다.

```ts
const room = this.rooms.get(event.roomId);
if (!room
    || client.roomId !== room.id
    || !sideFor(room, client)) {
  this.send(client, {
    type: "error",
    code: "forbidden",
    message: "현재 경기방에만 채팅을 보낼 수 있습니다."
  });
  return;
}
```

검사가 repository 호출보다 앞에 있으므로 unauthorized message는 저장 이력에도 남지 않습니다. 통과한 message만 exact room에 broadcast합니다.

lobby branch는 별도로 `scope: "lobby", roomId: null`을 저장하고 전체 client에 broadcast합니다. 한 generic branch에서 room 유무로 분기하지 않고 두 audience를 코드 구조로 분리했습니다.

### `4a98bd1e4f22` — persistence와 delivery를 동시에 관찰

fake WebSocket client 네 명으로 room A와 room B를 만들고 room B 참가자가 room A에 message를 주입합니다. test는 하나만 보지 않습니다.

- repository `createChatMessage`가 호출되지 않음
- sender가 `forbidden` error를 받음
- 어떤 socket도 `chat.message`를 받지 않음

정상 room A message는 A의 두 socket만 받고 B의 두 socket은 받지 않습니다. lobby message는 room과 무관하게 모든 socket이 받고 저장 input의 room은 null입니다.

따라서 이 test는 “거부 response가 왔다”뿐 아니라 **저장 전 차단과 audience 격리**를 함께 고정합니다.

## 4. 브라우저의 마지막 수용 경계

### `85edd6d1e26a` — current room filter

서버가 올바르게 동작하더라도 reconnect·stale handler·future routing bug로 다른 message가 client에 도착할 수 있습니다. `useGameConnection`은 reducer dispatch 전에 pure helper를 호출합니다.

```ts
export function isChatForActiveRoom(
  message: ChatMessage,
  activeRoomId: string | null
): boolean {
  return activeRoomId !== null
    && message.scope === "match"
    && message.roomId === activeRoomId;
}
```

active room이 없으면 이전 room의 message도 표시하지 않습니다. lobby message 역시 play 화면의 match transcript에 들어오지 않습니다.

이 filter는 보안 권한을 대체하지 않습니다. unauthorized row가 이미 저장된 뒤 client에서 숨기는 것은 올바른 해결이 아닙니다. 서버가 저장 전 차단하고 client는 UI state 오염을 추가로 막습니다.

### `02775797ab63` — pure rule regression

테스트는 current room match만 true이고, other room match·lobby·active room 없음은 false임을 한 함수에서 고정합니다. network 없이 가능한 규칙을 pure helper로 분리해 room reducer 전체를 띄우지 않고도 경계를 검사합니다.

## 최종 허용 행렬

| 입력/상태 | wire | repository/DB | GameHub 저장·전송 | play client 표시 |
| --- | :---: | :---: | :---: | :---: |
| lobby, room 필드 없음 | 허용 | room null로 저장 | 전체 audience | match transcript에는 제외 |
| lobby, room/null 필드 포함 | 거부 | 직접 호출 시 거부/DB CHECK | 도달 안 함 | 제외 |
| match, UUID, sender가 현재 좌석 | 허용 | 허용 | exact room만 | active room이면 표시 |
| match, UUID, sender가 다른 room | 허용 | 형태만 보면 허용 | 저장 전 `forbidden` | 도달해도 room mismatch로 제외 |
| match, room 없음/null/nonUUID | 거부 | 직접 호출 시 거부/DB CHECK | 도달 안 함 | 제외 |

이 Thread의 핵심은 defense-in-depth라는 추상어가 아니라 각 층이 서로 다른 질문에 답한다는 점입니다.

- schema: 이 payload 형태가 가능한가?
- storage: 이 row가 영구히 존재해도 되는가?
- hub: 이 사용자가 지금 이 room에서 말할 권한이 있는가?
- audience: 누가 받아야 하는가?
- browser: 현재 화면 상태에 포함할 message인가?

## 조사 범위

각 설명은 `web/ft_transcendence`의 표시 SHA diff와 해당 시점 source를 기준으로 작성했습니다. protocol/repository/GameHub/web test는 실행하지 않았습니다.
