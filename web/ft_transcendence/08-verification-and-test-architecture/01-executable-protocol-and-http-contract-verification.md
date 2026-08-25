# Thread: 실행 가능한 프로토콜과 HTTP 신뢰 경계

공유 TypeScript type만으로는 외부 입력을 보호할 수 없습니다. 이 Thread는 WebSocket/HTTP schema를 테스트 가능한 계약으로 만든 뒤, 그 계약을 모든 JSON route의 실제 runtime boundary에 적용하고, route 전수 회귀 테스트로 우회를 막는 과정을 다룹니다.

핵심 변화는 다음 한 문장으로 요약됩니다.

> “올바른 객체를 만들 수 있다”는 compile-time 편의에서, “외부에서 들어온 어떤 값도 공유 schema를 통과하지 않고 domain code에 도달하지 않는다”는 runtime 불변 조건으로 이동했다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `60c38090effc2af3df2670d4d96d3d30a49e95cc` | `test(shared): WebSocket 프로토콜 검증` | B | `PROTOCOL, SIMULATION, REALTIME` | client event parsing과 server event 직렬화 계약을 실행 가능한 테스트로 만든다. |
| 2 | `78cf83f29e80fb9fde1cb8ec73382d832c7abd56` | `test(shared): HTTP contract 검증` | B | `AUTH, PROTOCOL, TEST` | HTTP payload·error envelope·ticket 응답의 shared schema 경계를 검증한다. |
| 3 | `f655969b0d36b23b95aad350c6500dbfa3c3a8e8` | `test(protocol): versioned realtime contract 검증` | A | `PROTOCOL, REALTIME, PERSISTENCE` | version 누락과 서로 모순되는 server event를 명시적으로 거부한다. |
| 4 | `d07056e118716fe7537dd92636d1740867f486b9` | `feat(shared): 모든 HTTP request schema를 strict하게 정의` | A | `PROTOCOL` | method/path별 params·query·body 계약을 하나의 strict registry로 정의한다. |
| 5 | `59d75fddcaa66da998e23ae6f47c373652143ba2` | `fix(api): 모든 route input을 runtime 검증` | A | `PROTOCOL, RISK` | 공유 request contract를 실제 Fastify route 진입점에 적용한다. |
| 6 | `1abbf7dcdde497172790fe23a057a8b326417793` | `test(api): strict request contract 검증` | B | `TEST` | JSON route 전체에 unknown field·invalid path·bodyless GET 회귀 검증을 건다. |

## 1. 공유 schema는 테스트를 통해 wire contract가 된다

### `60c38090effc...` — WebSocket 입력과 출력의 경계

이 commit은 production parser를 새로 만들기보다, 이미 존재하던 `parseClientEvent`와 server event schema가 어떤 값을 허용하는지 처음으로 촘촘하게 고정합니다. 변경의 중심은 `packages/shared/src/ws.test.ts`입니다.

검증 범위는 세 층으로 나뉩니다.

1. **명령별 필수 필드**
   - `queue.join`은 지원되는 mode만 받습니다.
   - `game.ready`, `game.input`, `game.pause`, `game.resume`은 room 식별자와 명령별 필드를 요구합니다.
   - paddle 방향은 `-1`, `0`, `1`만 허용됩니다.
2. **문자열 정규화와 크기 제한**
   - 채팅 body는 trim된 뒤 비어 있으면 거부됩니다.
   - 허용 길이는 1~240자로 고정됩니다.
3. **wire format 자체의 실패**
   - 깨진 JSON은 domain event로 복구하지 않고 즉시 parse failure가 됩니다.
   - server event는 JSON stringify/parse 뒤에도 schema를 다시 통과해야 합니다.

테스트가 겨냥하는 경계는 단순한 타입 호환이 아닙니다. 예를 들어 다음 둘은 TypeScript object를 직접 만들 때는 비슷해 보이지만 wire contract에서는 완전히 다릅니다.

```json
{ "v": 1, "type": "queue.join", "mode": "queue" }
```

```json
{ "type": "queue.join", "mode": "queue" }
```

두 번째 값은 version이 없으므로 이후 protocol evolution에서 의미를 확정할 수 없습니다. 이 commit의 테스트는 parser가 이런 차이를 실제 실행 시점에 유지하는지 확인하는 기반이 됩니다.

이 단계의 한계도 분명합니다. parser 자체를 통과하는 shared package의 계약을 검증할 뿐, API가 모든 WebSocket frame을 반드시 이 parser로 보낸다는 사실이나 HTTP route가 같은 수준으로 강제된다는 사실까지 증명하지는 않습니다.

### `78cf83f29e80...` — HTTP 응답과 오류도 계약의 일부가 된다

다음 commit은 shared HTTP schema의 성공/실패 양쪽을 테스트합니다. 주요 검증 대상은 다음과 같습니다.

- session user가 공개 가능한 필드만 가진 형태인지
- login handle/role처럼 제한된 값이 unknown variant를 거부하는지
- login·chat 문자열의 trim 결과가 계약과 일치하는지
- UUID path parameter가 실제 UUID인지
- profile update가 빈 변경 객체를 허용하지 않는지
- 오류 응답이 안정된 envelope를 가지는지
- WebSocket ticket 응답이 43자 ticket, 30초 TTL, `protocolVersion: 1`을 갖는지
- 지원하지 않는 protocol version이 거부되는지

여기서 오류 envelope는 부가적인 편의가 아닙니다. route마다 `{message}`, `{error}`, 문자열 body를 제각각 반환하면 client는 status code 외에 안정적인 분기 기준을 가질 수 없습니다. shared schema 테스트는 실패도 다음처럼 구조화된 public contract여야 한다는 기준을 고정합니다.

```json
{
  "error": {
    "code": "validation_error",
    "message": "..."
  }
}
```

정확한 message 문구보다 중요한 것은 client가 `error.code`와 envelope 형태를 신뢰할 수 있다는 점입니다.

### `f655969b0d36...` — “v1처럼 보이는 값”이 아니라 일관된 v1만 허용한다

이 commit은 넓은 새 suite를 추가하지 않습니다. 대신 기존 server-event 검증에 **오래된 shape와 내부적으로 모순된 shape**를 겨냥한 세 가지 negative case를 추가합니다.

- version이 없는 presence event
- 음수 snapshot sequence
- `persisted: true`인데 `matchId`가 `null`인 경기 결과

세 번째 사례가 중요합니다. 각 field를 독립적으로만 검증하면 boolean은 boolean이고 nullable ID는 nullable ID이므로 통과할 수 있습니다. 그러나 둘의 조합은 “저장되었다고 말하면서 저장된 row를 식별할 수 없음”이라는 모순입니다.

```text
persisted = false  → matchId가 없을 수 있음
persisted = true   → 유효한 matchId가 필요함
```

따라서 이 commit은 단순 field validation에서 **event 의미의 일관성**으로 검증 수준을 높입니다. 중요도 A인 이유도 여기에 있습니다. stale client와 신규 server가 섞였을 때 조용히 잘못 해석되는 것보다 boundary에서 명확히 실패하는 편이 안전합니다.

## 2. 선언된 계약과 강제되는 계약은 다르다

### `d07056e11871...` — method/path별 request registry

앞의 테스트는 shared schema가 기대한 대로 동작한다는 사실을 보여 주지만, route handler가 임의로 `request.body as ...`를 사용하면 언제든 우회할 수 있습니다. 이 commit은 그 간극을 줄이기 위해 `packages/shared/src/http.ts`에 JSON request contract registry를 추가합니다.

핵심 표현은 endpoint마다 다음 세 입력을 분리하는 것입니다.

```text
(method, path)
    ├─ params schema
    ├─ query schema
    └─ body schema
```

body가 없는 GET도 “검증 생략”으로 표현하지 않고 strict empty object 계약을 가집니다. ID path는 UUID schema를 재사용하고, body/query object는 unknown key를 허용하지 않습니다. registry에는 health, auth, user, lobby, profile, friendship, tournament, admin route가 포함됩니다.

구조적으로 중요한 점은 contract가 route handler 내부에 흩어지지 않는다는 것입니다. 하나의 registry가 다음 질문에 답할 수 있게 됩니다.

- 이 endpoint는 body를 받는가?
- query에 허용된 key는 무엇인가?
- path ID의 형식은 무엇인가?
- 빈 object에 unknown field를 넣으면 거부되는가?

개념적인 사용 형태는 다음과 같습니다.

```ts
const contract = jsonHttpRequestContracts[routeKey];
const params = contract.params.parse(request.params);
const query = contract.query.parse(request.query);
const body = contract.body.parse(request.body);
```

이 코드는 contract 정의가 제공하는 의도를 보여 주지만, **이 commit만으로 실제 route가 반드시 registry를 사용한다고 보장되지는 않습니다.** 바로 그 연결이 다음 fix의 책임입니다.

## 3. `59d75fddcaa6...` — shared contract를 API 신뢰 경계에 연결한다

이전 상태의 문제는 schema 부재가 아니라 적용 지점의 불균일함입니다. 일부 handler는 Fastify generic이나 수동 check에 의존하고, 일부는 shared schema를 사용하더라도 params/query/body 가운데 일부만 검사할 수 있었습니다.

이 fix는 `apps/api/src/app.ts`의 JSON route들을 공통 request parsing boundary로 보냅니다. helper는 route key로 shared registry를 찾고 세 영역을 함께 parse합니다.

```text
Fastify request
    ↓
route별 shared request contract 선택
    ↓
params + query + body runtime parse
    ↓ 성공
handler/domain/repository 호출
    ↓ 실패
공통 validation error envelope
```

이 순서가 중요한 이유는 side effect보다 검증이 먼저 일어나야 하기 때문입니다. unknown body field를 handler가 무시한 뒤 write를 수행하거나, 잘못된 path ID가 repository query까지 내려간 뒤 DB 오류로 바뀌면 public contract가 불안정해집니다.

이 commit에서 눈여겨볼 변화는 다음과 같습니다.

- body가 없는 GET도 query/body의 unknown key를 거부합니다.
- path parameter가 shared UUID contract를 통과해야 handler가 실행됩니다.
- auth, lobby, profile, friendship, tournament, admin 등 JSON route가 같은 boundary를 사용합니다.
- validation error code가 shared error contract와 맞도록 `validation_error`로 정렬됩니다.

이 fix가 복구한 불변 조건은 다음과 같습니다.

> route handler는 raw `request.params`, `request.query`, `request.body`를 domain input으로 신뢰하지 않는다.

### 보장 범위

보장하는 것:

- registry에 포함된 JSON route는 shared schema를 runtime에서 통과합니다.
- params/query/body의 unknown key와 invalid value가 side effect 전에 실패합니다.
- validation failure가 공통 public envelope로 수렴합니다.

보장하지 않는 것:

- WebSocket frame은 별도의 realtime parser 경계를 사용합니다.
- binary/file upload처럼 JSON registry 밖의 transport는 이 commit의 범위가 아닙니다.
- 올바른 형식의 요청이 authorization/domain rule까지 만족한다는 뜻은 아닙니다.

## 4. `1abbf7dcdde4...` — route가 늘어나도 boundary를 우회하지 못하게 한다

공통 helper를 도입해도 새 route가 이를 호출하지 않으면 계약은 다시 새기 시작합니다. 이 test commit은 `apps/api/src/http-contract.test.ts`에서 route matrix를 만들고, JSON endpoint에 공통 negative case를 적용합니다.

검증 방식은 endpoint마다 happy path를 장황하게 반복하는 것이 아닙니다. contract 종류에 따라 다음과 같은 잘못된 입력을 기계적으로 주입합니다.

| 대상 | 주입하는 실패 | 기대 결과 |
| --- | --- | --- |
| bodyless GET | 임의 query/body key | handler 진입 전 4xx validation error |
| body가 있는 route | 정의되지 않은 body key | strict object failure |
| path route | 잘못된 UUID/ID | params parse failure |
| demo guest route | 명시되지 않은 입력 | 동일 strict boundary 유지 |

이 commit에는 forwarded-address 관련 테스트도 함께 섞여 있지만, 그것은 HTTP request schema Thread와 직접 관련이 없으므로 여기서는 제외합니다.

이 회귀 테스트가 제공하는 가치는 “현재 route 몇 개가 올바르다”보다 큽니다. route 목록과 contract registry가 어긋나거나 handler가 공통 parser를 건너뛰는 순간, unknown-field matrix가 실패합니다.

## 최종 불변 조건

| 경계 | 최종 규칙 | 근거가 된 commit |
| --- | --- | --- |
| WebSocket client event | version과 명령별 필수 field·범위를 runtime parse한다. | `60c380...` |
| WebSocket server event | JSON round-trip 뒤에도 v1 schema와 cross-field 의미를 만족한다. | `60c380...`, `f65596...` |
| HTTP shared model | 성공 payload와 오류 envelope를 같은 shared package에서 검증한다. | `78cf83...` |
| HTTP request declaration | endpoint별 params/query/body가 strict registry에 존재한다. | `d07056...` |
| API runtime enforcement | handler 전에 세 입력 영역을 shared contract로 parse한다. | `59d75f...` |
| 회귀 방지 | route matrix가 unknown field·invalid path·bodyless GET 우회를 검출한다. | `1abbf7...` |

최종 흐름은 다음과 같습니다.

```text
[외부 JSON]
    ↓
[method/path 또는 event type/version 선택]
    ↓
[shared strict schema parse]
    ├─ 실패: 안정된 public validation/protocol failure
    └─ 성공: 정규화된 typed value
                    ↓
          [authorization/domain/repository]
```

## 이 Thread의 경계

이 Thread는 “입력과 출력의 shape를 실행 가능한 계약으로 만들고 모든 JSON route에 강제한다”는 문제에 한정됩니다.

- cookie·WebSocket ticket의 실제 credential lifecycle은 Thread 02가 다룹니다.
- GameHub가 protocol error와 internal error를 어떻게 분리하는지도 Thread 02 범위입니다.
- 실제 process와 browser가 이 계약을 통과하는지는 Thread 06이 다룹니다.
- database transaction·concurrency 의미는 Thread 05가 다룹니다.

## 조사·실행 기록

각 설명은 표시된 exact SHA의 GitHub diff와 해당 시점 source/test code를 기준으로 작성했습니다. 이 작성 환경에서는 repository checkout과 test command를 실행하지 않았으므로, suite 통과 결과는 주장하지 않습니다.
