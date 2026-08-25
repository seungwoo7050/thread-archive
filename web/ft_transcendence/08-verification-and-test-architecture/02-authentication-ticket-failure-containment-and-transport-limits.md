# Thread: 인증 ticket, 실패 격리, WebSocket transport 한계

이 Thread의 중심은 “로그인했다”는 사실이 아닙니다. HTTP session에서 WebSocket connection으로 권한을 안전하게 넘기는 동안 어떤 credential이 노출되고, 인증이 끝나기 전에 도착한 frame을 누가 소유하며, 내부 실패와 과대 payload가 어디에서 끊기는지를 다룹니다.

최종 인증 경로는 다음과 같습니다.

```text
HTTP-only session cookie
        ↓ POST /auth/ws-ticket
짧은 수명의 one-time ticket
        ↓ GET /ws?ticket=...&v=1
protocol version 확인 → ticket 원자 소비 → GameHub 연결
```

이 경로는 세 가지 실패를 별도로 다룹니다.

1. **credential transport 실패**: bearer/query session이나 재사용 ticket을 거부합니다.
2. **application failure**: repository·handler 내부 오류를 generic public event로 축약합니다.
3. **transport resource failure**: 너무 큰 frame은 JSON parse나 application buffer보다 앞선 decoder에서 끊습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `d0531791406bd9809f15363c8542b1583cf7f564` | `fix(auth): cookie-only session과 환경별 route 적용` | S | `AUTH, ARCH, RISK` | 장기 session credential을 HttpOnly cookie 하나로 제한하고 개발 route를 환경별로 닫는다. |
| 2 | `401cf13d9d17e5a6095e311f3564dba6d4ca0c46` | `test(auth): cookie session 경계 검증` | A | `AUTH, RISK, TEST` | bearer/query fallback 제거, cookie 속성, 권한·환경 경계를 회귀 테스트로 고정한다. |
| 3 | `306d1946afb76665fa42f5443cfd56d10516d83a` | `feat(auth): ticket 기반 WebSocket 인증 연결` | S | `AUTH, REALTIME, RISK` | cookie session을 30초 one-time ticket으로 교환하고 인증 전 frame을 제한적으로 보류한다. |
| 4 | `b0ee833313c1d1529828e4b1802207fc3bc08a88` | `test(auth): WebSocket ticket 경계 검증` | A | `AUTH, PROTOCOL, REALTIME` | replay·위조·만료·정지 사용자·version·인증 전 buffer 한계를 검증한다. |
| 5 | `fe62962d65d90c136f826a6b271d17c991a5ffd6` | `fix(api): 내부 WebSocket 오류 숨김` | A | `PROTOCOL, REALTIME, PERSISTENCE` | malformed event와 내부 처리 실패를 서로 다른 generic public error로 축약한다. |
| 6 | `20933b1393f3a49ff73138c3770d0abcf44d28ca` | `test(api): WebSocket repository error redaction 검증` | B | `PROTOCOL, REALTIME, PERSISTENCE` | 실제 내부 문자열이 client frame에 섞이지 않는지 고정한다. |
| 7 | `8ea18a1b92db8d3a92232a98aae22c0f3cd1a9a1` | `fix(realtime): WebSocket transport payload 상한 설정` | A | `AUTH, REALTIME, RISK` | `ws` decoder의 `maxPayload`를 application 상한과 동일하게 설정한다. |
| 8 | `1afec49052b60103508e22ca9ff57a87013566e5` | `test(realtime): oversized WebSocket frame 거부 검증` | B | `AUTH, REALTIME, TEST` | 인증된 connection의 과대 frame도 transport close 1009로 끝나는지 검증한다. |

## 1. 장기 credential은 cookie 밖으로 나오지 않는다

### `d0531791406b...` — bearer/query fallback 제거

이전 구현은 개발 로그인 응답의 JSON token, `Authorization: Bearer ...`, WebSocket query의 session token처럼 같은 장기 credential을 여러 transport에 실을 수 있었습니다. 기능상 편리하지만 노출 면적과 해석 경로가 늘어납니다.

이 fix는 session 조회를 cookie 하나로 수렴시킵니다.

- 로그인 성공은 session cookie를 설정합니다.
- protected HTTP route는 cookie에서만 session을 찾습니다.
- bearer header와 `?session=` fallback을 제거합니다.
- CORS 허용 header에서도 authorization 의존을 제거합니다.
- cookie는 `HttpOnly`, `SameSite=Lax`, `Path=/`, 14일 `Max-Age`를 갖습니다.
- production/demo에서는 `Secure`가 활성화됩니다.
- 개발 로그인 route는 development/test에서만 등록되고 production/demo에서는 존재하지 않습니다.

여기서 가장 중요한 결정은 “여러 credential 방식을 지원하되 우선순위를 정한다”가 아니라 **해석 가능한 credential 경로 자체를 하나로 줄이는 것**입니다.

```text
이전
  Cookie ─┐
  Bearer ─┼─> session lookup
  Query  ─┘

이후
  HttpOnly Cookie ─> session lookup
```

장기 token이 JSON body에 남지 않으므로 browser JavaScript가 읽어 localStorage 등에 복사할 이유도 사라집니다. 다만 cookie session을 WebSocket handshake query에 그대로 넣을 수는 없습니다. 그 credential handoff 문제는 `306d...`가 별도 ticket으로 해결합니다.

### `401cf13d9d17...` — 제거한 fallback도 테스트해야 한다

positive cookie test만 있으면 bearer/query path가 실수로 되살아나도 잡지 못합니다. 이 commit은 인증 경계를 양방향으로 고정합니다.

| 입력 | 기대 결과 |
| --- | --- |
| 유효 session cookie | protected route 접근 가능 |
| 같은 token을 Bearer header에만 전달 | 인증되지 않음 |
| 같은 token을 query에만 전달 | 인증되지 않음 |
| `admin`이라는 handle로 일반 로그인 | role이 자동 승격되지 않음 |
| production/demo에서 dev-login 요청 | route 없음(404) |

cookie 속성도 assertion 대상입니다. 즉 테스트는 “cookie가 하나 왔다”가 아니라 JavaScript 접근 차단, cross-site 기본 정책, 환경별 secure 정책을 public security contract로 다룹니다.

이 테스트가 증명하지 않는 것도 구분해야 합니다. browser의 실제 SameSite 전송 행태나 TLS termination은 browser/process E2E의 범위이며, 이 commit은 Fastify 응답과 인증 resolution 경계를 검증합니다.

## 2. WebSocket에는 session이 아니라 소모성 권한을 넘긴다

### `306d1946afb7...` — cookie → one-time ticket

WebSocket constructor는 일반 browser API에서 임의 Authorization header를 넣기 어렵고, cookie만으로 upgrade하면 cross-site·재연결·protocol negotiation을 세밀하게 제한하기 어렵습니다. 이 commit은 HTTP session과 WebSocket upgrade 사이에 짧은 교환 단계를 추가합니다.

#### ticket 발급

`POST /auth/ws-ticket`은 cookie session을 먼저 확인합니다. 유효하고 active인 사용자에게만 다음 형태의 응답을 돌려줍니다.

```json
{
  "ticket": "<43-character opaque value>",
  "expiresInSeconds": 30,
  "protocolVersion": 1
}
```

raw ticket은 client에게 한 번만 노출되고 repository에는 hash가 저장됩니다. 따라서 DB row가 유출되어도 raw query credential을 그대로 재사용할 수 없습니다.

#### upgrade 순서

connection은 `?ticket=<opaque>&v=1`로 열립니다. 서버는 다음 순서를 지킵니다.

```text
1. query의 protocol version이 정확히 1인지 확인
2. ticket 형식 검증
3. ticket hash를 repository에서 원자적으로 소비
4. ticket 사용자 상태 확인
5. socket이 여전히 OPEN인지 확인
6. GameHub.connect(...)
7. 인증 중 보류했던 frame을 순서대로 전달
```

version 검사를 ticket 소비보다 먼저 하는 이유는 중요합니다. 지원하지 않는 client가 유효한 one-time ticket을 소모해 버리면, 사용자는 올바른 protocol로 즉시 재연결할 수 없습니다.

#### 인증이 비동기인 동안의 frame 소유권

upgrade 직후 ticket DB 조회가 끝나기 전에 client frame이 도착할 수 있습니다. 이를 무제한 배열에 쌓으면 인증 전 socket 하나가 메모리를 점유할 수 있습니다. 이 commit은 pending frame에 세 가지 한계를 둡니다.

| 제한 | 값 | 초과 시 의미 |
| --- | ---: | --- |
| frame 하나의 최대 크기 | 8 KiB | close 1009 |
| 대기 frame 수 | 16개 | close 1009 |
| 누적 payload | 32 KiB | close 1009 |

이 버퍼는 인증 성공 시 GameHub에 전달될 frame의 임시 owner입니다. ticket 검증이 실패하거나 socket이 닫히면 버퍼를 domain layer로 publish하지 않습니다.

```text
upgrade accepted
  ├─ pending raw frames (bounded)
  └─ ticket consume promise
          ├─ fail → policy close, pending 폐기
          └─ success + socket OPEN
                    → GameHub 연결
                    → pending frame replay
```

이 구조가 S급인 이유는 credential뿐 아니라 **인증 완료 전의 비동기 상태와 메모리 소유권**까지 정의하기 때문입니다.

### `b0ee833313c1...` — one-time이라는 말을 observable하게 만든다

후속 테스트는 ticket happy path보다 failure matrix에 무게를 둡니다.

- 정상 ticket은 한 번만 연결됩니다.
- 같은 ticket replay는 실패합니다.
- 임의로 만든 ticket과 만료 ticket은 실패합니다.
- ticket 발급 뒤 사용자가 정지되면 연결할 수 없습니다.
- `v=1` 이외 version은 거부됩니다.
- 지나치게 긴 query credential은 정상 ticket으로 해석되지 않습니다.
- ticket consumption이 지연되는 동안 8KiB/16개/32KiB 경계를 넘으면 close됩니다.
- memory/PostgreSQL repository가 같은 single-consumption 결과를 내는지 검증합니다.

여기서 “한 번만 연결된다”는 것은 함수가 한 번만 호출된다는 뜻이 아닙니다. 여러 consume 시도가 경쟁할 수 있지만 durable ticket state가 정확히 한 caller에게만 user를 반환해야 한다는 뜻입니다.

## 3. 내부 실패는 public protocol이 아니다

### `fe62962d65d9...` — parse failure와 handler failure를 분리

WebSocket message 처리 중에는 서로 성격이 다른 두 실패가 발생합니다.

1. client가 보낸 JSON/event 자체가 잘못됨
2. event는 유효하지만 repository·domain handler가 내부에서 실패함

이 commit은 두 경우를 generic server event로 구분합니다.

```text
parseClientEvent 실패
    → code: invalid_event
    → "올바르지 않은 메시지입니다."

유효 event 처리 중 예외
    → code: internal_error
    → "메시지를 처리하지 못했습니다."
```

이전처럼 `error.message`를 그대로 보내면 SQL 문구, table/column 이름, 내부 host, secret-like fixture가 protocol payload가 됩니다. fix는 client가 복구 가능한 분기(`invalid_event`인지 `internal_error`인지)는 남기되 내부 원인은 노출하지 않습니다.

중요한 범위 제한이 있습니다. 이 diff는 client redaction을 구현하지만 별도의 server-side structured logging을 추가하지 않습니다. 따라서 “운영자가 내부 원인을 관찰할 수 있다”고 이 commit만으로 주장할 수는 없습니다.

### `20933b1393f...` — 실수로 leak하기 쉬운 문자열을 고정 fixture로 사용

회귀 테스트는 repository method를 다음과 같은 내부 정보가 포함된 error로 실패시킵니다.

```text
password_hash
내부 database hostname
repository-specific failure text
```

그 뒤 socket이 받는 것은 versioned `internal_error` event여야 하며, serialized payload 어디에도 원래 문자열이 없어야 합니다.

이 테스트가 B급인 이유는 새로운 정책을 만들지 않고 preceding fix의 정보 격리 경계를 보호하기 때문입니다. 그러나 단순 status assertion보다 강합니다. generic code가 맞더라도 원래 message가 다른 field에 섞이면 실패하기 때문입니다.

## 4. application limit만으로는 wire payload를 막을 수 없다

### `8ea18a1b92db...` — decoder 단계 hard limit

`306d...`의 pending buffer는 인증이 진행 중일 때 application code가 받은 `RawData`를 검사합니다. 하지만 `ws` library가 frame 전체를 decode한 뒤에야 message callback을 호출한다면, application size check 이전에 큰 payload를 메모리에 조립할 수 있습니다.

이 commit의 production 변경은 작지만 경계는 더 앞쪽으로 이동합니다.

```ts
realtime.register(websocket, {
  options: { maxPayload: PRE_AUTH_MESSAGE_MAX_BYTES }
});
```

결과적으로 8KiB 상한이 두 층에 존재합니다.

| 층 | 목적 | 관찰 가능한 결과 |
| --- | --- | --- |
| `ws` decoder `maxPayload` | frame 조립 단계의 hard limit | library가 close 1009, reason은 빈 문자열일 수 있음 |
| application pending buffer | 인증 완료 전 count/total/순서 소유권 | 정책 close와 pending 폐기 |

두 제한은 중복이 아닙니다. decoder는 한 frame의 wire 크기를 막고, application buffer는 작은 frame 여러 개가 인증 대기 중 누적되는 상황을 막습니다.

### `1afec49052b6...` — 인증 뒤에도 같은 transport 상한을 유지한다

이 test는 유효 cookie와 ticket으로 인증된 socket을 만든 뒤 8KiB를 1 byte 초과하는 frame을 전송합니다. 기대 결과는 application의 `invalid_event`가 아니라 transport close code 1009입니다. `maxPayload`가 decoder에서 동작하므로 close reason도 application이 작성한 설명이 아니라 빈 문자열입니다.

이 차이를 문서에서 유지해야 합니다.

```text
작지만 잘못된 JSON/event
    → message callback 도달
    → invalid_event

8KiB 초과 frame
    → decoder에서 차단
    → close 1009, application event 없음
```

## 최종 인증·실패 상태표

| 시점/실패 | 소유자 | 처리 | 외부 결과 |
| --- | --- | --- | --- |
| HTTP session 생성 | repository + cookie response | raw session은 HttpOnly cookie에만 전달 | JSON token 없음 |
| ticket 발급 | repository | raw ticket 반환, hash/expiry 저장 | 30초·v1 ticket |
| 잘못된 protocol version | upgrade boundary | ticket 소비 전 거부 | policy close |
| ticket replay/만료/위조/정지 사용자 | ticket repository | consume 실패, hub에 publish하지 않음 | 인증 실패 close |
| ticket 확인 중 작은 frame | pre-auth buffer | 16개/32KiB 안에서 임시 보유 | 성공 후 순서대로 전달 |
| single frame 8KiB 초과 | `ws` decoder | application 도달 전 종료 | close 1009 |
| malformed client event | GameHub parser | generic validation event | `invalid_event` |
| repository/domain 예외 | GameHub handler | 내부 message 폐기 | `internal_error` |

최종 불변 조건은 다음과 같습니다.

- 장기 session credential은 cookie-only입니다.
- WebSocket에는 짧고 한 번만 쓰는 ticket만 전달합니다.
- protocol compatibility를 확인하기 전에 ticket을 소모하지 않습니다.
- 인증 중 frame buffer는 단일 크기·개수·누적 크기로 제한됩니다.
- 내부 exception text는 server event가 되지 않습니다.
- transport hard limit은 application parser보다 앞에서 작동합니다.

## 이 Thread의 경계

- HTTP params/query/body의 일반 strict schema는 Thread 01입니다.
- ticket hash의 PostgreSQL 동시 소비 semantics는 Thread 05의 실제 DB 검증과 연결되지만, 이 문서는 인증 handshake 관점만 다룹니다.
- browser가 fresh ticket으로 재연결하는 실제 사용자 흐름은 Thread 06입니다.
- reconnect room ownership과 15초 lease는 Thread 04입니다.

## 조사·실행 기록

각 commit은 표시된 exact SHA의 diff와 그 시점 source/test code로 확인했습니다. 이 작성 환경에서는 unit/PostgreSQL/browser suite를 실행하지 않았으므로 실제 통과 결과를 주장하지 않습니다.
