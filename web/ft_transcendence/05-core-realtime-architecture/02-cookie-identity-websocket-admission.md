# 장기 세션을 쿠키에 가두고 WebSocket 입장을 단발 ticket으로 제한하기

HTTP 세션 토큰을 브라우저 JavaScript와 WebSocket URL에 재사용하면 하나의 장기 비밀값이 local storage, response body, query string, reverse-proxy log까지 퍼집니다. 이 Thread는 인증 정보를 다음 두 수명으로 분리합니다.

- **장기 identity**: 브라우저가 읽지 못하는 `HttpOnly` cookie
- **짧은 WebSocket admission**: 30초 동안 한 번만 소비할 수 있는 32-byte 난수 ticket

여기에 handshake가 인증을 기다리는 동안 받을 수 있는 message 수와 byte 수를 제한하고, request log에서 cookie·authorization·ticket을 제거합니다.

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `1779df300611` | `feat(api): 로그인과 로비 HTTP 경계 구현` | B | `AUTH, PERSISTENCE, WEB` | session token을 body·cookie·Bearer·query로 넓게 노출한 초기 상태 |
| `d0531791406b` | `fix(auth): cookie-only session과 환경별 route 적용` | S | `AUTH, ARCH, RISK` | 장기 세션의 유일한 browser transport를 HttpOnly cookie로 제한 |
| `353ca9a17415` | `fix(web): browser token 저장 제거` | A | `AUTH, PROTOCOL, REALTIME` | local storage와 session query 전달을 제거하고 ticket 요청으로 전환 |
| `d9bde7485719` | `feat(auth): WebSocket ticket 생성과 HTTP 계약 정의` | A | `AUTH, PROTOCOL, REALTIME` | 32-byte·30초·protocol-version ticket 계약 정의 |
| `c89a455fee06` | `feat(db): PostgreSQL WebSocket ticket 저장 추가` | A | `AUTH, SIMULATION, REALTIME` | raw ticket 대신 hash를 저장하고 `DELETE … RETURNING`으로 한 번만 소비 |
| `306d1946afb7` | `feat(auth): ticket 기반 WebSocket 인증 연결` | S | `AUTH, REALTIME, RISK` | version 확인, bounded pre-auth buffer, ticket 소비, GameHub handoff 순서 확정 |
| `b0ee833313c1` | `test(auth): WebSocket ticket 경계 검증` | A | `AUTH, PROTOCOL, REALTIME` | 일회성·동시 소비·만료·정지 사용자·버전·버퍼 한계를 검증 |
| `ec9cb39babef` | `fix(log): 요청 비밀 정보 redaction 적용` | A | `AUTH, REALTIME, OBSERVABILITY` | request log에서 인증 cookie/header/query와 URL query 제거 |

## 1. 초기 HTTP 경계는 장기 토큰을 너무 많은 곳에 허용했다

### `1779df300611`

초기 로그인 응답은 사용자와 token을 함께 반환하고 cookie도 설정했습니다. 이후 current-user 해석은 cookie뿐 아니라 `Authorization: Bearer …`, query token 등 여러 입력을 허용했습니다.

이 방식은 개발 초기 연결에는 편리하지만 browser threat surface가 큽니다.

```text
로그인 response body
  ├─ JavaScript가 token을 읽을 수 있음
  ├─ localStorage 등에 보관 가능
  ├─ WebSocket URL query로 재전달 가능
  └─ URL/access log/referrer에 남을 수 있음
```

이 커밋은 문제의 원인이 아니라 비교 기준입니다. 아직 “HTTP 요청을 인증한다”와 “WebSocket 한 번을 허가한다”가 같은 credential 수명으로 묶여 있습니다.

## 2. 장기 identity의 browser transport를 cookie 하나로 줄였다

### `d0531791406b` — cookie-only boundary

이 커밋은 로그인 response에서 raw token을 제거하고, 서버의 session reader도 cookie만 인정하도록 바꿉니다. 브라우저에서 직접 다룰 credential이 사라지며 CORS 허용 header에서도 `Authorization`이 빠집니다.

환경별 route도 함께 정리됩니다.

- 개발용 로그인 route는 development/test에서만 노출됩니다.
- production/demo cookie는 `Secure`를 사용합니다.
- browser identity는 `HttpOnly` cookie로만 전달됩니다.

핵심 결정은 “token을 안전하게 저장하는 JavaScript 코드”를 만드는 것이 아니라 **JavaScript가 장기 token을 소유하지 않게 하는 것**입니다. XSS가 발생했을 때 cookie를 이용한 same-origin 요청까지 자동으로 막는 것은 아니지만, raw credential을 읽어 외부로 반출하거나 URL에 붙이는 경로는 제거됩니다.

### `353ca9a17415` — browser 저장·query 전달 제거

web 앱은 local storage에서 session token을 읽거나 쓰지 않고, credential 포함 HTTP 요청으로 짧은 WebSocket ticket을 발급받습니다. socket URL에는 장기 세션이 아니라 ticket과 protocol version만 들어갑니다.

```text
브라우저
  ├─ cookie: 브라우저가 자동 전송, JS는 값에 접근하지 않음
  └─ POST /auth/ws-ticket
         └─ 30초짜리 raw ticket을 메모리에서 바로 WebSocket URL에 사용
```

이 커밋에 다른 API 변경도 섞여 있지만, 이 Thread에서는 token 저장 제거와 ticket 요청 전환만 다룹니다.

## 3. ticket은 raw 값과 저장 값을 분리하고, DB가 소비 권위를 갖는다

### `d9bde7485719` — ticket 형식과 수명

raw ticket은 crypto RNG의 32 bytes를 base64url로 표현합니다. 결과는 43자 URL-safe 문자열이며 TTL은 30초입니다.

```ts
export const WS_TICKET_TTL_SECONDS = 30;

export function createRawWsTicket(): string {
  return randomBytes(32).toString("base64url");
}

export function hashWsTicket(ticket: string): string {
  return createHash("sha256").update(ticket, "utf8").digest("hex");
}
```

HTTP response는 raw ticket, 만료 시간, 지원 protocol version을 반환합니다. raw 값은 실제 client가 한 번 연결하는 데 필요하지만 repository에는 hash만 넘깁니다.

### `c89a455fee06` — 한 번만 성공하는 DB operation

PostgreSQL의 `ws_tickets`는 `ticket_hash`를 primary key로 두고 사용자와 만료 시각을 연결합니다. 소비는 먼저 조회하고 나중에 삭제하는 두 단계가 아니라, 하나의 statement에서 row를 제거하며 반환하는 방식입니다.

```sql
DELETE FROM ws_tickets
WHERE ticket_hash = $1
RETURNING user_id, expires_at;
```

실제 repository는 반환된 사용자 상태와 만료를 확인해 active user만 돌려줍니다. 중요한 성질은 **row 삭제가 소비 시도와 결합**되어 있다는 점입니다.

- 같은 raw ticket을 두 연결이 동시에 사용해도 한 DELETE만 row를 얻습니다.
- 만료되었거나 정지된 사용자 ticket도 재사용 가능한 상태로 남지 않습니다.
- DB에는 raw credential이 없으므로 저장소 유출이 즉시 WebSocket 입장권 유출로 이어지지 않습니다.

## 4. handshake는 버전을 먼저 확인하고, 인증 대기 자원에 상한을 둔다

### `306d1946afb7` — admission 순서

upgrade 이후 repository 응답을 기다리는 동안 client가 message를 보낼 수 있습니다. 이를 무제한으로 배열에 쌓으면 아직 인증되지 않은 연결이 메모리를 소비할 수 있습니다. 이 커밋은 임시 listener가 보관할 수 있는 범위를 세 겹으로 제한합니다.

| 제한 | 값 | 초과 시 |
| --- | ---: | --- |
| message 하나 | 8 KiB | close code 1009 |
| message 개수 | 16 | close code 1009 |
| 합계 | 32 KiB | close code 1009 |

handshake 순서도 보안 의미가 있습니다.

```text
upgrade request
  → query 형식 검사
  → protocol version 확인
  → bounded pre-auth listener 설치
  → raw ticket hash 계산
  → repository에서 ticket 단발 소비
  → 사용자 active 여부 확인
  → 임시 listener 제거
  → GameHub.connect(socket, user, pendingPayloads)
```

지원하지 않는 version은 ticket을 소비하기 **전에** 거부합니다. 따라서 client가 실수로 `v=2`를 보냈다가 지원되는 `v=1`으로 재시도할 때, 유효한 단발 ticket이 이미 소진되지 않습니다.

인증 실패는 policy violation(1008), message/byte 초과는 1009, repository 내부 오류는 1011로 분리됩니다. socket이 인증 완료 전에 닫히면 GameHub로 넘기지 않습니다.

## 5. 테스트가 고정하는 admission 경계

### `b0ee833313c1`

테스트는 단순 “ticket으로 연결된다”보다 다음 negative path를 겨냥합니다.

- cookie가 없고 `Authorization`만 있는 요청은 장기 세션 인증에 실패합니다.
- raw ticket이 아니라 SHA-256 hash만 persistence에 전달됩니다.
- 같은 ticket의 반복 사용과 동시 사용에서 한 연결만 성공합니다.
- 만료 ticket과 suspended user ticket은 거부되고 재사용되지 않습니다.
- 지원하지 않는 version은 ticket을 소비하지 않습니다.
- pre-auth 단계에서 message 1개, 개수, 누적 byte 각각의 상한을 넘으면 1009로 닫힙니다.
- 인증이 끝난 뒤에만 pending payload가 GameHub 처리 경로로 넘어갑니다.

이 테스트는 PostgreSQL transaction isolation 전반을 증명하는 것이 아니라, repository의 atomic consume operation과 WebSocket admission 순서를 고정합니다.

## 6. 비밀값이 request log로 되돌아 나오지 않게 했다

### `ec9cb39babef`

credential을 cookie와 짧은 query ticket으로 줄여도 logger가 request 전체를 직렬화하면 다시 노출될 수 있습니다. 이 커밋은 Fastify logger serializer/redaction에 다음 값을 포함합니다.

- cookie
- authorization header
- query의 ticket과 기타 비밀값
- request URL의 query 문자열

따라서 로그에는 route/path와 운영에 필요한 request metadata는 남지만 `?ticket=…` 원문은 남지 않습니다. 범위는 **설정된 request logger 경로**입니다. 애플리케이션 코드가 별도 문자열로 비밀값을 직접 기록하는 경우까지 자동으로 제거한다는 뜻은 아닙니다.

## 최종 trust boundary

```text
[HttpOnly session cookie]
        │  same-origin HTTP 인증
        ▼
POST /auth/ws-ticket
        │  raw 32-byte ticket은 응답으로 한 번 노출
        ├─ DB: SHA-256 hash + user + expiry
        └─ browser: 메모리에서 즉시 사용
                 │
                 ▼
/ws?ticket=...&v=1
  version check → bounded pre-auth buffer → atomic consume → GameHub handoff
```

최종적으로 다음이 성립합니다.

- browser JavaScript는 장기 session token을 저장하거나 URL에 싣지 않습니다.
- WebSocket query에 들어가는 값은 30초짜리 단발 ticket입니다.
- persistence에는 raw ticket이 아니라 hash만 남습니다.
- 동시 소비에서도 한 연결만 인증됩니다.
- 잘못된 protocol version은 유효 ticket을 불필요하게 소진하지 않습니다.
- 인증 전 socket이 점유할 수 있는 message/byte 수는 고정 상한을 가집니다.
- request log는 cookie·authorization·ticket query를 기록하지 않습니다.

이 Thread는 인증된 이후의 message shape와 sequence를 다루지 않습니다. 그 경계는 Thread 03입니다. 같은 사용자의 socket을 하나로 교체하거나 끊긴 경기 좌석을 복구하는 문제는 Thread 05가 담당합니다.
