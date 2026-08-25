# Thread: Resource API가 실행 가능한 HTTP 계약을 갖기까지

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread는 이미 동작하는 Fastify route에 나중에 runtime contract를 입히는 과정이다. 최초 API는 `request.body as ...`, 누락값의 기본값, route마다 다른 오류 body로 repository를 호출한다. 초기 통합 테스트도 “로그인 후 조회가 된다”, “관리자가 ban할 수 있다”, “토너먼트를 만든 뒤 목록에서 보인다”는 기능 기준선만 고정한다.

그 뒤 shared package의 TypeScript interface가 Zod schema로 바뀌고, 요청·응답·오류 envelope가 추가된다. 하지만 schema를 정의하는 것만으로 HTTP 경계가 바뀌지는 않는다. `parseInput`, `parseOutput`, `ApiHttpError`와 Fastify error handler가 생기고, 실제 route가 이 helper를 호출한 뒤에야 다음 순서가 집행된다.

```text
인증·권한 확인 또는 공개 route 진입
  → params/body runtime parse
  → repository 호출
  → response runtime parse
  → 성공 body 반환

예상된 실패
  → ApiHttpError
  → status/code/public message/requestId envelope

예상 밖 실패
  → server log
  → 500 internal_error의 고정된 public message
```

이 Thread는 “계약 정의”와 “계약 집행”을 분리해 읽어야 한다. 또한 response validation은 잘못된 wire body를 막지만, 그 전에 수행된 repository mutation을 되돌리는 transaction은 아니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1779df300611` | `feat(api): 로그인과 로비 HTTP 경계 구현` | B | `AUTH, PERSISTENCE, WEB` | 저장소 기반 개발 로그인, 현재 사용자, 로비, 순위 endpoint를 최초 Fastify route로 노출한다. |
| 2 | `0bcc487d949f` | `feat(api): 프로필과 친구 리소스 라우트 추가` | B | `PERSISTENCE` | 공개 profile/dashboard와 사용자별 profile update·friendship mutation route를 추가한다. |
| 3 | `e8bb6a4bf68b` | `feat(api): 토너먼트와 관리자 라우트 추가` | B | `AUTH, PERSISTENCE, TOURNAMENT` | 토너먼트 생성·참가와 관리자 사용자 조회·상태 변경 route를 추가한다. |
| 4 | `fb1c287d9e79` | `test(api): 로그인과 로비 조회 검증` | B | `PERSISTENCE, TEST` | 초기 로그인→현재 사용자와 leaderboard/lobby read 경로를 Fastify injection으로 검증한다. |
| 5 | `1395d45a3665` | `test(api): 관리자 사용자 상태 변경 검증` | B | `AUTH, PERSISTENCE, TEST` | 개발 로그인부터 admin ban route와 memory repository 상태 변경까지 초기 관리자 경로를 검증한다. |
| 6 | `5088099d1e7d` | `test(api): 토너먼트 생성 흐름 검증` | B | `AUTH, PERSISTENCE, TOURNAMENT` | 인증 사용자 tournament 생성 후 목록에서 다시 읽는 write→read 경로를 검증한다. |
| 7 | `0c5c27c8c3df` | `feat(shared): 사용자 HTTP runtime contract 정의` | B | `PROTOCOL, TOURNAMENT` | compile-time user interface를 실행 가능한 Zod schema와 inferred type으로 교체한다. |
| 8 | `6704f37ca6a3` | `feat(shared): 경기·대시보드 runtime contract 정의` | B | `PROTOCOL` | match summary, dashboard, leaderboard payload를 실행 가능한 schema로 정의한다. |
| 9 | `4bace138f188` | `feat(shared): 친구·채팅·로비 runtime contract 정의` | B | `PROTOCOL, REALTIME` | friendship, chat, lobby stats/aggregate를 runtime schema로 확장한다. |
| 10 | `7d0793a23f5d` | `feat(shared): 토너먼트·관리 runtime contract 정의` | B | `PROTOCOL, TOURNAMENT, OPERATIONS` | 토너먼트 aggregate와 관리자 audit response를 runtime schema로 정의한다. |
| 11 | `282a9d0beb47` | `feat(shared): HTTP 요청·오류 schema 정의` | B | `-` | route params/body와 공통 API error envelope를 strict schema로 정의한다. |
| 12 | `e226b68fe235` | `feat(shared): HTTP 응답 runtime contract 정의` | B | `AUTH, PROTOCOL, REALTIME` | domain validator를 health/auth/user/lobby/tournament/admin endpoint별 response schema로 조합한다. |
| 13 | `78cf83f29e80` | `test(shared): HTTP contract 검증` | B | `AUTH, PROTOCOL, TEST` | HTTP schema의 positive/negative shape와 normalization 규칙을 package unit test로 고정한다. |
| 14 | `ac85316bb0cb` | `feat(api): typed HTTP 오류 boundary 추가` | A | `AUTH, PROTOCOL, RISK` | Zod 입력 오류, 예상된 API 오류, not-found, 예상 밖 실패를 하나의 Fastify error boundary로 통합한다. |
| 15 | `c4cba7d3f871` | `feat(api): 인증·사용자 HTTP contract 적용` | B | `AUTH, PROTOCOL, WEB` | health/auth/user/profile route에 shared input/output schema와 typed error boundary를 적용한다. |
| 16 | `05e3ecfa2a2d` | `feat(api): 로비·친구 HTTP contract 적용` | B | `AUTH, PROTOCOL, PERSISTENCE` | lobby/chat/leaderboard/dashboard/friendship route에 runtime input/output validation과 typed failure를 적용한다. |
| 17 | `24f99345452d` | `feat(api): 토너먼트·관리 HTTP contract 적용` | B | `AUTH, TOURNAMENT` | 토너먼트와 관리자 route에 request/response schema 및 공통 authorization/error helper를 적용한다. |
| 18 | `b2a8de5a0027` | `refactor(api): HTTP boundary helper 통합` | B | `AUTH` | 남은 route-local unauthorized/suspended helper를 제거하고 공통 typed boundary 함수를 직접 사용한다. |
| 19 | `50caaf5c7c49` | `test(api): typed HTTP boundary 기대값 정렬` | B | `AUTH, TOURNAMENT, WEB` | 기존 API·관리·토너먼트 통합 test를 HttpOnly cookie와 typed response/error contract에 맞춘다. |

## 1. 기능이 먼저 생겼고, 경계는 아직 느슨했다

### `1779df300611` — 로그인·세션·로비의 최초 HTTP 연결

`buildApp`은 CORS와 cookie plugin을 등록하고 개발 로그인, logout, 현재 사용자, 로비, 순위표 route를 만든다. 입력은 runtime validator가 아니라 type assertion과 기본값으로 처리된다.

```ts
const body = request.body as {
  handle?: string;
  displayName?: string;
  email?: string;
};

const user = await repo.upsertDevUser({
  handle: body.handle ?? "player",
  displayName: body.displayName ?? body.handle ?? "플레이어",
  email: body.email
});
```

로그인은 session token을 HttpOnly cookie에 넣는 동시에 JSON body에도 `{ user, token }`으로 반환한다. 인증 조회는 다음 네 출처를 우선순위대로 허용한다.

```text
pp_session cookie
  → Authorization: Bearer ...
  → parsed query.session
  → raw URL의 session query
```

따라서 이 단계의 인증 계약은 cookie 전용이 아니다. logout도 cookie만 지우고 server-side session을 삭제하지 않는다. unauthorized 응답은 공통 envelope가 아니라 route-local `{ message: ... }`다.

### `0bcc487d949f` — profile과 friendship route 확장

`/users/:id`, `/dashboard`, 공개 profile, 자신의 profile 조회·수정, 친구 목록·요청·수락이 추가된다. `request.params as { id: string }`, `request.body as ...`가 그대로 repository 입력으로 넘어가므로 UUID, 길이, unknown field는 transport 경계에서 검사되지 않는다.

공개 read와 authenticated mutation이 구분되기 시작하지만 오류 표현은 여전히 route마다 다르다. 예를 들어 없는 사용자는 직접 `reply.code(404).send(...)`하고, 인증 실패는 별도 helper가 응답을 만든다.

### `e8bb6a4bf68b` — tournament와 admin route 확장

토너먼트 목록·생성·참가, 관리자 사용자·감사 목록, ban/status 변경이 추가된다. 관리자 route마다 다음 패턴을 반복한다.

```text
currentUser 조회
  → 없으면 401
  → role !== "admin"이면 403
  → params/body assertion
  → repository mutation
```

`name`, `banned`, `reason`에는 route 안의 fallback이 사용된다. 이 기본값은 API 정책이지만 shared contract에는 아직 표현되지 않는다.

## 2. 초기 테스트는 기능 기준선이지 계약 검증이 아니다

### `fb1c287d9e79`

Fastify `inject`로 개발 로그인 후 JSON에서 token을 꺼내 Bearer header로 `/me`를 호출한다. leaderboard와 lobby는 200인지, leaderboard가 비어 있지 않은지만 확인한다.

### `1395d45a3665`

두 사용자를 개발 로그인으로 만들고 JSON token을 Bearer로 전달해 `/admin/users/:id/ban`을 호출한 뒤 대상 상태가 `banned`인지 본다. 이 테스트는 당시의 권한 준비 방식과 memory repository mutation을 고정하지만, 입력 오류나 공통 error body는 검사하지 않는다.

### `5088099d1e7d`

로그인 token으로 토너먼트를 만들고 목록 첫 항목의 이름을 비교한다. write 후 read-back은 검증하지만 response 전체 shape나 잘못된 이름을 다루지 않는다.

세 테스트가 공통으로 증명하지 않는 것은 다음과 같다.

- unknown body field 거부
- UUID·길이·enum 검증
- 오류 code/request ID
- response object의 실행 시점 검증
- cookie만으로 동작하는 browser session 흐름

이 제한은 결함이 아니라 이후 retrofit 전의 기준선이다.

## 3. TypeScript interface가 실행 가능한 schema로 바뀐다

### Domain value schema — `0c5c27c8c3df`부터 `7d0793a23f5d`까지

네 커밋은 기존 interface를 한 번에 교체하지 않고 영역별로 Zod schema와 `z.infer` type으로 전환한다.

**`0c5c27c8c3df` — 사용자**

```ts
export const publicUserSchema = z.object({
  id: z.string().uuid(),
  handle: z.string().min(1),
  displayName: z.string().min(1),
  avatarKey: z.string(),
  role: userRoleSchema,
  status: userStatusSchema,
  rating: z.number().int(),
  wins: z.number().int().nonnegative(),
  losses: z.number().int().nonnegative(),
  online: z.boolean(),
  isNpc: z.boolean()
});

export type PublicUser = z.infer<typeof publicUserSchema>;
```

UUID, 정수, 음수 금지, enum 같은 조건이 실행 시점에 검사 가능해진다. `SessionUser`는 이를 확장해 nullable email을 추가한다.

**`6704f37ca6a3` — 경기·대시보드**

경기 종료 시각은 `datetime`, 점수는 nonnegative integer, win rate는 0~100, 순위는 positive integer로 제한된다.

**`4bace138f188` — 친구·채팅·로비**

채팅 본문은 1~240자, room ID는 nullable UUID, 로비 통계는 음수가 아닌 수로 정의된다. `LobbyResponse`는 사용자·경기·채팅·통계를 기존 schema로 조합한다.

**`7d0793a23f5d` — 토너먼트·관리**

토너먼트 match의 round/status, nullable participant·winner·score, aggregate의 capacity와 audit action이 schema가 된다.

이 네 커밋이 만드는 것은 **검증 가능한 값의 정의**다. API route가 `.parse()`나 `.safeParse()`를 호출하지 않으면 외부 요청과 실제 응답은 아직 이 계약을 통과하지 않는다.

### Request와 error schema — `282a9d0beb47`

요청 schema는 domain schema와 목적이 다르다. 외부에서 신뢰할 수 없는 params/body를 정규화하고 제한한다.

```ts
export const idParamsSchema =
  z.object({ id: z.string().uuid() }).strict();

export const devLoginBodySchema = z.object({
  handle: z.string().trim().min(2).max(24)
    .regex(/^[a-z0-9][a-z0-9-]*$/),
  displayName: z.string().trim().min(1).max(40),
  email: z.string().trim().email().optional()
}).strict();
```

`profileUpdateBodySchema`는 `displayName`과 `avatarKey`가 모두 없는 요청을 `refine`으로 거부한다. `chatBodySchema`, 친구 요청, 토너먼트 생성, 관리자 ban/status도 모두 `.strict()`라 unknown field를 허용하지 않는다.

동시에 오류 body가 하나의 envelope로 정의된다.

```ts
{
  error: {
    code: string;
    message: string;
    requestId: string;
    fieldErrors?: Record<string, string[]>;
  }
}
```

### Response schema — `e226b68fe235`

domain schema를 endpoint별 wrapper로 조합한다. 예를 들면 `{ user: sessionUserSchema }`, `{ entries: leaderboardEntrySchema[] }`, `{ tournament: tournamentSummarySchema }`처럼 transport envelope까지 검사할 수 있다. WebSocket ticket 응답은 `expiresInSeconds: 30`, `protocolVersion: 1`을 literal로 고정한다.

### `78cf83f29e80` — schema 단위 테스트

shared test는 다음 규칙을 직접 검사한다.

- 정상 session user shape 수용
- login의 unknown field와 잘못된 handle 거부
- display name과 chat body trim
- route ID에 UUID 요구
- 빈 profile patch 거부
- error envelope 유지
- WebSocket ticket TTL과 protocol version 고정

이 테스트는 schema 자체를 검증한다. Fastify route가 실제로 어느 schema를 호출하는지는 아직 증명하지 않는다.

## 4. `ac85316bb0cb` — typed HTTP error boundary

이 Thread에서 유일한 `A` 커밋이다. 여러 route에 흩어진 응답 생성을 다음 세 종류로 분류한다.

### 입력 오류

```ts
export function parseInput<T>(schema: ZodType<T>, input: unknown): T {
  const result = schema.safeParse(input);
  if (result.success) return result.data;

  const fieldErrors: Record<string, string[]> = {};
  for (const issue of result.error.issues) {
    const field = issue.path.join(".") || "request";
    (fieldErrors[field] ??= []).push(issue.message);
  }

  throw new ApiHttpError(
    400,
    "validation_failed",
    "입력값을 확인해주세요.",
    Object.keys(fieldErrors).length > 0 ? fieldErrors : undefined
  );
}
```

Zod issue path를 field 이름으로 모으고, handler가 직접 reply를 보내는 대신 typed exception을 던진다.

### 예상된 API 실패

`unauthorized()`, `suspended()`, `forbidden()`, `notFound()`는 각각 status와 machine-readable code, 공개 가능한 message를 가진 `ApiHttpError`를 던진다. route가 동일한 분기에서 다른 JSON 모양을 만들지 않게 한다.

### 예상 밖 실패

```ts
app.setErrorHandler((error, request, reply) => {
  if (error instanceof ApiHttpError) {
    sendApiError(
      reply,
      request,
      error.statusCode,
      error.code,
      error.message,
      error.fieldErrors
    );
    return;
  }

  request.log.error({ err: error }, "request failed");
  sendApiError(
    reply,
    request,
    500,
    "internal_error",
    "요청을 처리하지 못했습니다."
  );
});
```

예상 밖 error의 원문은 server log로 보내고, client에는 고정된 message만 보낸다. 모든 오류 body에는 Fastify `request.id`가 문자열로 들어간다. not-found handler도 같은 envelope를 사용한다.

`parseOutput`은 schema 불일치를 `ApiHttpError`로 바꾸지 않고 generic `Error`로 던진다. 따라서 response contract 위반은 예상 밖 server failure로 기록되고 client에는 `internal_error`가 보인다.

중요한 시점 구분이 있다. 이 커밋은 `httpBoundary.ts`를 추가하지만 `buildApp`에 설치하거나 route에서 호출하지 않는다. 실제 집행은 다음 세 커밋에서 시작된다.

## 5. 기존 route를 계약 경계로 교체한다

### `c4cba7d3f871` — auth·user·profile

`installHttpErrorBoundary(app)`가 실제로 설치되고 health, login, logout, current user, user lookup, profile route가 shared schema를 사용한다.

개발 로그인은 더 이상 fallback으로 임의 값을 만들지 않는다.

```ts
const body = parseInput(http.devLoginBodySchema, request.body);
const user = await repo.upsertDevUser(body);
const token = await repo.createSession(user.id);

reply.setCookie("pp_session", token, {
  path: "/",
  sameSite: "lax",
  httpOnly: true,
  maxAge: 60 * 60 * 24 * 14
});

return parseOutput(http.userResponseSchema, { user });
```

응답에서 raw token이 제거되고 HttpOnly cookie가 session 전달 수단이 된다. logout은 `repo.deleteSession(readSessionToken(request))`을 호출한 뒤 cookie를 지워 server-side session까지 무효화한다.

`/users/:id`는 UUID를 parse한 뒤 조회하고, 없는 사용자는 typed 404를 던진다. profile patch는 정규화된 parsed body만 repository에 전달한다.

다만 이 SHA와 Thread 마지막 `50caaf5c7c49`에서도 `readSessionToken`은 다음 legacy source를 계속 허용한다.

```ts
return cookieToken ?? header ?? queryToken ?? rawQueryToken;
```

즉 **로그인 응답과 대표 테스트는 cookie 중심으로 바뀌지만, production credential reader가 cookie 전용으로 축소된 것은 아니다.** Bearer와 query session을 제거하는 일은 이 Thread의 커밋이 보장하지 않는다.

### `05e3ecfa2a2d` — lobby·chat·friends

로비·순위·대시보드 response가 schema를 통과한다. 채팅은 인증 및 계정 상태 확인 뒤 `chatBodySchema`로 trim/길이 검증을 하고, repository 결과를 `chatResponseSchema`로 검사한다.

두 친구 요청 endpoint는 하나의 `requestFriend` handler로 합쳐져 동일한 auth·suspension·input/output 흐름을 사용한다. 친구 수락의 path ID도 UUID schema를 통과한다.

### `24f99345452d` — tournament·admin

토너먼트 이름과 ID, 관리자 ban/status body가 schema를 거친다. 반복되던 관리자 검사는 `requireAdmin`으로 모인다.

```ts
async function requireAdmin(
  repo: AppRepository,
  request: FastifyRequest
): Promise<SessionUser> {
  const user = await currentUser(repo, request);
  if (!user) raiseUnauthorized();
  if (user.role !== "admin") raiseForbidden();
  return user;
}
```

관리자 mutation의 실제 순서는 다음과 같다.

```text
requireAdmin
  → id params parse
  → body parse
  → repository mutation
  → public-user response parse
```

토너먼트 생성과 참가도 authenticated/active check 뒤 parsed input만 repository에 보낸다.

### `b2a8de5a0027` — 남은 local helper 제거

이 커밋은 semantics를 새로 만들지 않는다. `raiseUnauthorized` 같은 alias와 `reply.code(...).send(...)` 방식의 local helper를 제거하고, `httpBoundary.ts`의 `unauthorized`, `suspended`, `forbidden`을 직접 사용한다. 결과적으로 권한 실패가 우회 경로 없이 동일 error handler로 모인다.

## 6. 출력 검증의 보장과 한계

route의 전형적인 mutation 코드는 다음 형태다.

```ts
const body = parseInput(requestSchema, request.body);

return parseOutput(responseSchema, {
  profile: await repo.updateProfile(user.id, body)
});
```

입력 검증은 repository call **전**에 있으므로 잘못된 body가 mutation에 도달하는 것을 막는다. 반면 출력 검증은 `await repo.updateProfile(...)` **후**에 있다. repository가 상태를 바꾼 뒤 잘못된 shape를 반환하면 다음 일이 일어난다.

1. mutation은 이미 수행된다.
2. `parseOutput`이 실패한다.
3. error boundary가 500 `internal_error`를 반환한다.
4. 이 코드만으로 mutation은 rollback되지 않는다.

따라서 `parseOutput`의 계약은 “잘못된 응답을 정상 성공으로 내보내지 않는다”이지 “응답 검증 실패 시 저장소 상태까지 원복한다”가 아니다. transaction이 필요한 operation은 repository 계층에서 별도로 제공해야 한다.

## 7. `50caaf5c7c49` — 통합 테스트의 기대값을 새 경계에 맞춘다

기존 세 suite가 다음과 같이 바뀐다.

- login JSON에 `token` property가 없음을 확인
- `set-cookie`에서 `pp_session`만 추출해 다음 요청에 전달
- logout 전 `/me`는 200, logout 후 같은 cookie는 401
- 관리자 fixture는 development seed의 admin으로 server session을 직접 만들고 cookie를 사용
- ban된 사용자의 채팅이 403인지 확인
- 토너먼트 생성·참가도 cookie session을 사용

이 변화는 browser와 가까운 대표 경로를 고정한다. 그러나 앞서 확인한 대로 token reader의 Bearer/query fallback까지 제거됐음을 증명하지 않는다. 또한 이 커밋은 기존 성공 흐름의 기대값을 정렬하는 작업이지, 모든 route의 unknown query/body를 체계적으로 열거하는 negative matrix는 아니다. 그 역할은 별도의 strict request Thread가 맡는다.

## 최종 계약

| 경계 | owner | 성공 시 | 실패 시 |
| --- | --- | --- | --- |
| domain value | `@pong-pong/shared` Zod schema | 검증된 사용자·경기·로비·토너먼트 값 | Zod issue |
| request params/body | shared request schema + API `parseInput` | trim/default가 적용된 typed value | 400 `validation_failed` + field errors |
| authorization | API helper | handler 계속 | 401/403 typed `ApiHttpError` |
| repository | `AppRepository` | 조회 또는 mutation 결과 | 예상 밖 error는 중앙 500 |
| response | shared endpoint schema + `parseOutput` | 검증된 wire body | 500 `internal_error` |
| not-found | Fastify boundary | 해당 없음 | 공통 404 envelope |
| test credential | 대표 integration suite | HttpOnly session cookie | raw login token 미사용 |

Thread 마지막 상태의 핵심은 “TypeScript가 맞다고 가정하는 객체”에서 “request entry와 response exit에서 실제로 검사하는 객체”로의 전환이다. 그럼에도 query/Bearer credential 제거, route 전체의 params/query/body strictness, mutation rollback은 이 Thread만으로 완료되지 않는다.

> 검사 범위: 지정 브랜치의 exact SHA diff와 해당 SHA source를 확인했다. 로컬 checkout, test 실행, 실제 browser CORS/session 검증은 수행하지 않았다.
