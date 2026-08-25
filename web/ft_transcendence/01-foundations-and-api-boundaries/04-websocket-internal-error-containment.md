# Thread: WebSocket client 오류와 내부 처리 실패를 분리한다

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread가 고치는 문제는 WebSocket 명령 처리의 실패가 모두 같은 catch에 들어가던 구조다. 기존 `GameHub.receive`는 payload parse, 권한·guest 제한, repository 호출, broadcast를 한 try/catch에 두고 모든 예외를 `invalid_event`로 분류했다. 더 큰 문제는 `error.message`를 그대로 client event에 넣었다는 점이다. repository 예외에 SQL, column, 내부 hostname이 포함되면 그대로 wire에 노출될 수 있었다.

수정은 실패를 두 단계로 나눈다.

```text
payload parse 실패
  → invalid_event
  → 고정된 사용자 메시지
  → command dispatch 전에 반환

parse 성공 후 command/repository 실패
  → internal_error
  → 고정된 사용자 메시지
  → raw exception은 client event에 복사하지 않음
```

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `fe62962d65d9` | `fix(api): 내부 WebSocket 오류 숨김` | A | `PROTOCOL, REALTIME, PERSISTENCE` | client event parse failure와 command/repository processing failure를 분리하고 raw exception을 WebSocket response에서 제거한다. |
| 2 | `20933b1393f3` | `test(api): WebSocket repository error redaction 검증` | B | `PROTOCOL, REALTIME, PERSISTENCE` | chat repository method에 내부 SQL·host 문자열이 든 예외를 주입하고 client event가 generic error만 포함하는지 검증한다. |

## `fe62962d65d9` — 실패 소유자를 catch 구조로 표현한다

### 변경 전 가정

기존 코드는 `parseClientEvent(payload)`와 이후 command 처리를 한 try 안에서 수행했다. catch는 오류 원인과 관계없이 다음 event를 보냈다.

```ts
{
  type: "error",
  code: "invalid_event",
  message: error instanceof Error
    ? error.message
    : "메시지를 처리하지 못했습니다."
}
```

이 구조에는 두 문제가 있다.

1. 정상적으로 parse된 명령의 repository failure도 client 입력 오류로 오분류된다.
2. raw exception message가 public protocol field가 된다.

### parse와 processing을 분리한다

커밋은 두 public message를 상수로 둔다.

```ts
const INVALID_EVENT_MESSAGE = "올바르지 않은 메시지입니다.";
const INTERNAL_ERROR_MESSAGE = "메시지를 처리하지 못했습니다.";
```

그 다음 parse만 첫 try/catch에 둔다.

```ts
let event: ReturnType<typeof parseClientEvent>;

try {
  event = parseClientEvent(payload);
} catch {
  this.send(client, {
    type: "error",
    code: "invalid_event",
    message: INVALID_EVENT_MESSAGE
  });
  return;
}
```

parse에 성공한 뒤의 권한 확인과 command/repository 작업은 두 번째 try에 남는다.

```ts
try {
  /* guest 제한, queue/tournament/chat command 처리 */
} catch {
  this.send(client, {
    type: "error",
    code: "internal_error",
    message: INTERNAL_ERROR_MESSAGE
  });
}
```

따라서 public code가 실패 단계와 일치한다. malformed JSON·schema 불일치는 `invalid_event`, `createChatMessage` 같은 실행 중 실패는 `internal_error`다.

같은 커밋은 AI fallback promise의 catch도 고친다.

```diff
- .catch((error) => {
+ .catch(() => {
    this.send(entry.client, {
      type: "error",
      code: "internal_error",
-     message: error instanceof Error ? error.message : "AI 상대를 찾지 못했습니다."
+     message: INTERNAL_ERROR_MESSAGE
    });
  });
```

즉 receive path 한 곳만이 아니라 diff에 포함된 비동기 AI fallback도 raw message를 제거한다.

### 이 커밋이 보장하지 않는 것

- processing 중 일부 mutation이 끝난 뒤 broadcast가 실패했을 때 rollback
- internal exception을 별도 server log에 반드시 보존
- 모든 GameHub 비동기 callback의 전수 redaction
- error event 이후 socket을 닫거나 재시도하는 정책
- client가 generic message만으로 복구 방법을 알 수 있다는 보장

실제 catch는 error 변수를 받지 않으므로 이 코드 자체는 내부 원인을 log하지 않는다. 보안상 wire 노출은 막지만 관찰 가능성을 추가한 커밋은 아니다.

## `20933b1393f3` — 내부 문자열이 실제 wire에 없는지 확인한다

테스트는 memory repository의 `createChatMessage`만 결정적으로 실패시킨다.

```ts
vi.spyOn(repository, "createChatMessage").mockRejectedValue(
  new Error(
    "select password_hash from users failed: database.internal:5432"
  )
);
```

Fake WebSocket client를 `GameHub`에 연결하고 정상적으로 parse되는 chat event를 보낸다.

```ts
socket.receive({
  v: 1,
  type: "chat.send",
  scope: "lobby",
  body: "안전한 오류 응답"
});
```

그 뒤 error event가 정확히 하나의 public shape인지 기다린다.

```ts
{
  v: 1,
  type: "error",
  code: "internal_error",
  message: "메시지를 처리하지 못했습니다."
}
```

마지막 두 assertion이 이 test의 핵심이다.

```ts
expect(JSON.stringify(socket.events()))
  .not.toContain("password_hash");

expect(JSON.stringify(socket.events()))
  .not.toContain("database.internal");
```

단순히 code가 `internal_error`인지 확인하는 데서 끝나지 않고, 의도적으로 넣은 민감한 column과 내부 host가 event 전체에 없는지 검사한다.

## 보장 범위

| 상황 | client code | client message | 원문 노출 |
| --- | --- | --- | --- |
| JSON/schema parse 실패 | `invalid_event` | `올바르지 않은 메시지입니다.` | 없음 |
| chat repository 실패 | `internal_error` | `메시지를 처리하지 못했습니다.` | test로 부재 확인 |
| AI fallback 실패 | `internal_error` | 동일 고정 message | diff상 raw message 제거 |
| 부분 mutation 후 후속 실패 | `internal_error` 가능 | 고정 message | rollback은 보장하지 않음 |

후속 test가 직접 증명하는 path는 chat repository rejection 하나다. AI fallback, 다른 repository method, connection의 이후 사용 가능성, logging은 같은 수준으로 검증되지 않는다.

> 검사 범위: 지정 브랜치의 두 exact SHA diff와 `GameHub`/test source를 확인했다. 테스트는 실행하지 않았다.
