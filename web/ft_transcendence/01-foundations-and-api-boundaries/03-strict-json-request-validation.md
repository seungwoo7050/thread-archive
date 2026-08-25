# Thread: 국소 body 방어에서 모든 JSON 입력의 strict 검증까지

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread는 한 route에서 `request.body`가 `undefined`일 수 있다는 문제로 시작한다. 첫 수정은 누락된 body를 `{}`로 바꿔 기존 수동 검증까지 안전하게 도달하게 할 뿐이다. 이후 작업은 이 국소 방어를 다음 규칙으로 일반화한다.

> 모든 JSON route는 `params`, `query`, `body` 세 입력을 명시적으로 소유하고, 인증·권한 확인이나 repository 호출보다 먼저 shared strict schema로 검사한다.

이 규칙은 body를 쓰지 않는 GET이나 POST에도 적용된다. “사용하지 않으니 무시한다”가 아니라 `z.object({}).strict()`를 적용해 unknown query/body를 400으로 거부한다. 따라서 transport에 들어왔지만 handler가 읽지 않는 값도 허용 목록에 없는 입력으로 취급된다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `8ce1199ffd12` | `fix(api): body 없는 로비 채팅 요청 처리` | B | `-` | 누락된 chat body를 `{}`로 정규화해 property access 예외 대신 기존 400 validation 분기로 보낸다. |
| 2 | `d07056e11871` | `feat(shared): 모든 HTTP request schema를 strict하게 정의` | A | `PROTOCOL` | 모든 JSON route에 params/query/body 세 schema를 가진 strict request contract map을 정의한다. |
| 3 | `59d75fddcaa6` | `fix(api): 모든 route input을 runtime 검증` | A | `PROTOCOL, RISK` | 공통 `parseHttpRequest`를 모든 JSON handler의 business logic 전에 호출해 params/query/body strictness를 실제로 집행한다. |
| 4 | `1abbf7dcdde4` | `test(api): strict request contract 검증` | B | `TEST` | 전체 JSON route의 unknown query/body와 invalid path parameter, untrusted forwarded address를 table-driven Fastify injection으로 검증한다. |

## `8ce1199ffd12` — 예외를 피했지만 route 하나만 고쳤다

변경은 한 줄이다.

```diff
- const body = request.body as { body?: string };
+ const body = (request.body ?? {}) as { body?: string };
```

그 다음 코드는 그대로다.

```ts
const messageBody = body.body?.trim() ?? "";
if (!messageBody) {
  return reply.code(400).send({ message: "메시지를 입력해주세요." });
}
if (messageBody.length > 240) {
  return reply.code(400).send({
    message: "메시지는 240자 이내로 입력해주세요."
  });
}
```

이 커밋은 body 없는 채팅 전송을 성공시키지 않는다. 이전에는 `undefined.body` 접근 전에 문제가 생길 수 있었지만, 이후에는 `{}`에서 빈 문자열을 얻어 의도한 400 응답을 반환한다. schema도 공통 parser도 없고, 다른 route의 `params`나 `query`에는 아무 변화가 없다.

이 국소 수정이 보여주는 문제는 “optional transport field를 handler마다 알아서 정규화하면 누락과 strictness 규칙이 흩어진다”는 것이다.

## `d07056e11871` — route마다 세 입력을 빠짐없이 선언한다

shared package에 작은 조합 helper와 route map이 추가된다.

```ts
function defineHttpRequestContract<
  Params extends z.ZodTypeAny,
  Query extends z.ZodTypeAny,
  Body extends z.ZodTypeAny
>(params: Params, query: Query, body: Body) {
  return { params, query, body } as const;
}
```

사용하지 않는 입력도 `emptyParamsSchema`로 명시한다.

```ts
const emptyHttpRequestContract = defineHttpRequestContract(
  emptyParamsSchema,
  emptyParamsSchema,
  emptyParamsSchema
);

const idHttpRequestContract = defineHttpRequestContract(
  idParamsSchema,
  emptyParamsSchema,
  emptyParamsSchema
);
```

`jsonHttpRequestContracts`는 health, login/logout, WebSocket ticket 발급, 사용자·로비·채팅·순위·대시보드·profile·friendship·tournament·admin route를 모두 이름 있는 entry로 나열한다.

```ts
export const jsonHttpRequestContracts = {
  health: emptyHttpRequestContract,
  devLogin: defineHttpRequestContract(
    emptyParamsSchema,
    emptyParamsSchema,
    devLoginBodySchema
  ),
  userById: idHttpRequestContract,
  lobbyChat: defineHttpRequestContract(
    emptyParamsSchema,
    emptyParamsSchema,
    chatBodySchema
  ),
  profileByHandle: defineHttpRequestContract(
    handleParamsSchema,
    emptyParamsSchema,
    emptyParamsSchema
  ),
  adminBan: defineHttpRequestContract(
    idParamsSchema,
    emptyParamsSchema,
    adminBanBodySchema
  )
  /* ... */
} as const;
```

이 커밋이 중요한 이유는 개별 schema를 새로 만든 데 있지 않다. 이미 존재하던 strict body/params schema를 **route별 complete input set**으로 묶는다. bodyless GET도 empty body contract를 갖고, query를 읽지 않는 route도 empty query contract를 갖는다.

그러나 map은 data일 뿐이다. handler가 이를 호출하지 않으면 요청은 여전히 통과한다.

## `59d75fddcaa6` — contract map을 실제 entry gate로 만든다

### 공통 parser

```ts
export function parseHttpRequest<Params, Query, Body>(
  contract: {
    params: ZodType<Params>;
    query: ZodType<Query>;
    body: ZodType<Body>;
  },
  request: FastifyRequest
): { params: Params; query: Query; body: Body } {
  return {
    params: parseInput(contract.params, request.params ?? {}),
    query: parseInput(contract.query, request.query ?? {}),
    body: parseInput(contract.body, request.body ?? {})
  };
}
```

`?? {}`는 첫 커밋의 국소 normalization을 params/query/body 전체에 적용한다. 차이는 그 다음 단계다. 이제 빈 객체가 strict empty schema를 통과하거나, unknown field 때문에 공통 validation error가 된다.

이 커밋은 validation machine code도 `validation_failed`에서 `validation_error`로 바꾼다. 따라서 이 SHA 이후의 기대값은 다음 envelope다.

```json
{
  "error": {
    "code": "validation_error",
    "message": "입력값을 확인해주세요.",
    "requestId": "...",
    "fieldErrors": {
      "request": ["..."]
    }
  }
}
```

### handler ordering

각 JSON handler는 business logic 전에 contract를 호출한다.

```ts
app.post("/chat/lobby", async (request) => {
  const { body } = parseHttpRequest(
    http.jsonHttpRequestContracts.lobbyChat,
    request
  );

  const user = await getCurrentUser(request);
  if (!user) unauthorized();
  requireRegistered(user);
  if (!isActive(user)) suspended();

  return parseOutput(http.chatResponseSchema, {
    message: await repo.createChatMessage({
      scope: "lobby",
      roomId: null,
      senderId: user.id,
      body: body.body
    })
  });
});
```

여기서는 parsed `body.body`만 repository에 들어간다. path route도 동일하다.

```ts
const {
  params: { id }
} = parseHttpRequest(
  http.jsonHttpRequestContracts.joinTournament,
  request
);
```

주요 순서는 다음과 같다.

```text
request.params/query/body
  → missing input은 {}
  → 세 strict schema parse
  → parsed value 반환
  → auth/mode/permission check
  → repository read 또는 mutation
```

validation을 인증보다 먼저 수행하기 때문에 observable status가 바뀌는 사례도 있다. 이전에는 `?session=...`을 인증 source로 읽거나 인증 실패로 보냈지만, empty query contract가 먼저 실행된 뒤에는 허용되지 않은 query field로 400 `validation_error`가 된다. 같은 시점의 auth test는 cookie 요청 200, Authorization 요청 401, query session 요청 400을 기대하도록 바뀐다.

### 보장과 한계

이 커밋이 보장하는 것:

- map에 등록되고 `parseHttpRequest`를 호출하는 현재 JSON route는 세 입력 모두 검사한다.
- invalid path parameter와 unknown query/body는 repository 호출 전에 종료된다.
- handler가 사용하는 값은 parse/trim/refine을 통과한 결과다.
- bodyless route에도 explicit empty contract가 있다.

이 커밋이 자동으로 보장하지 않는 것:

- 새 route가 map과 parser 호출을 빠뜨리지 않는다는 compile-time 강제
- multipart, raw WebSocket frame 등 JSON route 밖의 입력
- authorization보다 validation을 먼저 하는 순서가 모든 API에서 바람직하다는 일반 명제
- repository가 별도로 읽는 환경·DB 값의 검증

## `1abbf7dcdde4` — route-wide regression을 만든다

새 `http-contract.test.ts`는 문자열 하나를 검사하는 단위 test가 아니라 route 목록을 직접 열거한다.

### unknown query matrix

27개 JSON route에 `?unexpected=1`을 붙이고 모두 공통 400 envelope인지 확인한다. public health부터 authenticated profile, tournament, admin route까지 포함된다. auth나 not-found보다 먼저 strict query parse가 실행되므로 test credential이 없어도 validation branch를 관찰할 수 있다.

### unknown body matrix

body를 사용하는 route뿐 아니라 body가 비어 있어야 하는 POST에도 `{ unexpected: true }`를 넣는다. 총 12개 case가 login, logout, WS ticket, chat, profile patch, friend, tournament, admin mutation을 포괄한다.

### path parameter matrix

다음과 같은 잘못된 path가 route work 전에 400인지 검사한다.

- `/users/not-a-uuid`
- 65자 handle
- 잘못된 friendship/tournament/admin UUID

### demo route

demo guest login에도 unknown body와 query를 각각 보내 strict contract가 mode-specific route에도 적용되는지 확인한다.

공통 assertion은 status 400, `apiErrorBodySchema` parse, code `validation_error`, non-empty request ID, field errors 존재를 확인한다.

같은 커밋에는 `trustProxy`가 설정되지 않은 Fastify instance에서 서로 다른 `x-forwarded-for`를 보내도 guest rate-limit key가 바뀌지 않는 test도 추가된다. 이는 JSON schema 자체가 아니라 **untrusted transport metadata를 신뢰하지 않는다는 인접 경계**다. 이 test는 network trust Thread와 맞닿지만, 여기서는 “forwarded header도 임의 client input이며 기본값에서 identity를 바꾸지 못한다”는 범위만 연결한다.

## 최종 상태

```text
[Fastify JSON request]
  ├─ params ─┐
  ├─ query  ─┼─ parseHttpRequest(route contract)
  └─ body   ─┘
               ├─ success: parsed values only
               └─ failure: 400 validation_error + requestId + fieldErrors
                         ↓
              auth / mode / permission
                         ↓
                 repository operation
```

| 실패 입력 | repository 도달 여부 | 최종 분류 |
| --- | --- | --- |
| 누락된 optional transport object | `{}`로 정규화 후 schema 판단 | contract에 따라 성공 또는 400 |
| unknown query | 도달하지 않음 | 400 `validation_error` |
| unknown body | 도달하지 않음 | 400 `validation_error` |
| invalid UUID/handle | 도달하지 않음 | 400 `validation_error` |
| 유효 입력, 인증 없음 | 입력 parse 후 도달하지 않음 | 401 |
| 유효 입력, 권한 없음 | 입력 parse 후 도달하지 않음 | 403 |

route matrix는 당시 목록에 대한 강한 회귀 방지책이지만, 새 route 추가 시 contract map과 test case를 사람이 함께 갱신해야 한다. “모든 미래 route”를 자동으로 발견하는 registry introspection이나 lint rule은 이 Thread에 없다.

> 검사 범위: 지정 브랜치의 exact SHA diff와 해당 SHA source/test를 확인했다. 테스트 suite는 실행하지 않았다.
