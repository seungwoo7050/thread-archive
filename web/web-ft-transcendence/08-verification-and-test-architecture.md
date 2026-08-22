===== BEGIN FILE: 01-executable-protocol-and-http-contract-verification.md =====
# Development Thread 01: 실행 가능한 프로토콜·HTTP 계약 검증

English title: **Executable Protocol and HTTP Contract Verification**

## Thread 목표

공유 schema가 실제 API와 realtime 신뢰 경계에서 fail-closed로 적용되고, 구형·unknown·불완전 입력이 비즈니스 로직에 도달하지 않는 과정을 복원한다.

## 핵심 질문

- compile-time TypeScript 타입과 runtime parser의 보장 범위는 어떻게 다른가?
- strict schema를 정의하는 것과 모든 route가 그 schema를 실제 호출하는 것은 왜 별도 단계인가?
- version, sequence, persisted-result discriminator가 어떤 잘못된 상태 조합을 막는가?
- table-driven route regression은 적용 누락을 어떤 방식으로 탐지하며 무엇을 자동으로 포괄하지 못하는가?

## 완료 기준

- WebSocket·HTTP schema 단위 테스트와 실제 route integration 테스트의 역할을 구분한다.
- 각 SHA에서 허용·거부되는 concrete shape를 해당 시점 코드로 설명한다.
- Contract → Apply → Route-wide regression의 순서를 재구성한다.
- unknown field, invalid path, unversioned event, incomplete persistence metadata의 실패 위치를 설명한다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags |
| --- | --- | --- | --- | --- |
| 1 | `60c38090effc` | `test(shared): WebSocket 프로토콜 검증` | B | PROTOCOL, SIMULATION, REALTIME |
| 2 | `78cf83f29e80` | `test(shared): HTTP contract 검증` | B | AUTH, PROTOCOL, TEST |
| 3 | `f655969b0d36` | `test(protocol): versioned realtime contract 검증` | A | PROTOCOL, REALTIME, PERSISTENCE |
| 4 | `d07056e11871` | `feat(shared): 모든 HTTP request schema를 strict하게 정의` | A | PROTOCOL |
| 5 | `59d75fddcaa6` | `fix(api): 모든 route input을 runtime 검증` | A | PROTOCOL, RISK |
| 6 | `1abbf7dcdde4` | `test(api): strict request contract 검증` | B | TEST |

> Commit 순서·subject·importance·tags는 Phase 1 감사 후 동결된 정보입니다. Phase 2에서는 변경하지 않습니다.

## Commit `60c38090effc` — test(shared): WebSocket 프로토콜 검증

- Full SHA: `60c38090effc2af3df2670d4d96d3d30a49e95cc`
- Subject: `test(shared): WebSocket 프로토콜 검증`
- Importance: **B**
- Tags: PROTOCOL, SIMULATION, REALTIME
- Source-defined role: 초기 WebSocket 런타임 파서의 정상·거부 입력 범위를 table-driven 테스트로 고정한다.

### 정확한 조사 대상

- `packages/shared/src/ws.test.ts`에서 `parseClientEvent`를 호출하는 표 기반 사례를 확인한다.
- `queue.join`, `game.ready`, `game.input`, `game.pause`, `game.resume`, `chat.send`의 필수 필드와 enum 범위를 대조한다.
- 방향 값이 정확히 `-1 | 0 | 1`인지, 채팅 본문이 trim되고 1~240자로 제한되는지 확인한다.
- 잘못된 JSON과 서버 이벤트 encode/decode 왕복이 어떤 방식으로 검증되는지 확인한다.

### 학습자 복원

<!-- ANSWER:60c38090effc:begin -->
#### 역사적 복원

이 커밋 이전에는 공유 WebSocket 스키마가 존재했지만, 허용 이벤트 전체와 기본값·경계값을 한 자리에서 회귀 검증하는 증거가 부족했다.

`ws.test.ts`는 각 이벤트를 실제 `parseClientEvent` 경계에 넣는다. 특히 `queue.join`의 기본 mode, room 식별자와 입력 방향의 도메인, 채팅 공백 제거와 길이 제한을 정상·오류 사례로 분리한다. 따라서 TypeScript 타입이 아니라 실행 시 파서가 계약을 강제한다는 점을 확인한다.

서버 이벤트는 encode 후 다시 parse하는 왕복을 사용한다. 이는 생산자와 소비자가 동일한 공유 스키마를 사용한다는 기본 호환성 증거지만, 네트워크 연결·순서 뒤바뀜·버전 전환까지 재현하지는 않는다.

#### 이 커밋이 보장하는 것

- 지원하는 초기 client-event 형태와 값 범위를 파서 수준에서 고정한다.
- 잘못된 JSON과 계약 위반 객체가 처리기로 넘어가지 않음을 증명한다.

#### 이 커밋이 보장하지 않는 것

- 실제 소켓의 인증·전송·순서 보장은 검증하지 않는다.
- 뒤에 도입되는 `v: 1`, snapshot sequence, persistence discriminator는 아직 대상이 아니다.

#### 전후 관계

`f655969b0d36`이 이 기초 계약을 versioned realtime 형태로 확장하고, 구형 shape를 명시적으로 거부한다.

#### 증거 성격

- 공유 패키지 단위 경계 테스트
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:60c38090effc:end -->

## Commit `78cf83f29e80` — test(shared): HTTP contract 검증

- Full SHA: `78cf83f29e80fb9fde1cb8ec73382d832c7abd56`
- Subject: `test(shared): HTTP contract 검증`
- Importance: **B**
- Tags: AUTH, PROTOCOL, TEST
- Source-defined role: 공유 HTTP 요청·응답 스키마의 보안 및 값 경계를 실행 가능한 단위 테스트로 고정한다.

### 정확한 조사 대상

- `packages/shared/src/http.test.ts`에서 session user, login, profile update, UUID params, error envelope를 확인한다.
- 로그인 입력의 알 수 없는 privilege 필드와 금지된 handle 형태가 거부되는지 확인한다.
- 프로필 수정이 비어 있는 객체를 허용하지 않는지, 문자열 trim이 언제 일어나는지 확인한다.
- WebSocket ticket 응답의 TTL 30초와 protocol version 1 조건을 확인한다.

### 학습자 복원

<!-- ANSWER:78cf83f29e80:begin -->
#### 역사적 복원

HTTP DTO와 Zod 스키마가 추가된 뒤, 이 커밋은 계약의 의미를 값 수준에서 고정한다. 로그인 입력에 임의 role·권한 필드를 섞거나 허용되지 않은 handle을 보내도 스키마가 이를 받아들이지 않아야 한다.

테스트는 session user projection, UUID 경로 변수, 비어 있지 않은 profile update, 공통 오류 envelope, 30초짜리 version-1 WebSocket ticket 응답을 직접 parse한다. 요청과 응답 모두 실제 런타임 스키마를 통과해야 한다.

다만 이 시점의 테스트는 공유 스키마 자체만 검증한다. API의 모든 라우트가 해당 스키마를 실제로 호출하는지는 `59d75fddcaa6`과 `1abbf7dcdde4`에서 별도로 확인된다.

#### 이 커밋이 보장하는 것

- 공유 HTTP schema가 권한 주입 필드, 잘못된 UUID, 빈 mutation을 거부한다.
- 공통 오류와 ticket 응답의 외부 shape가 고정된다.

#### 이 커밋이 보장하지 않는 것

- Fastify 라우트의 적용 누락을 탐지하지 않는다.
- DB나 인증 세션의 실제 상태 변화는 검증하지 않는다.

#### 전후 관계

`d07056e11871`이 모든 JSON 라우트용 strict request contract를 정의하고, 이후 API 적용·회귀 테스트가 이 단위 계약을 실행 경계로 확장한다.

#### 증거 성격

- 공유 HTTP schema 단위 테스트
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:78cf83f29e80:end -->

## Commit `f655969b0d36` — test(protocol): versioned realtime contract 검증

- Full SHA: `f655969b0d36b23b95aad350c6500dbfa3c3a8e8`
- Subject: `test(protocol): versioned realtime contract 검증`
- Importance: **A**
- Tags: PROTOCOL, REALTIME, PERSISTENCE
- Source-defined role: version-1 realtime migration이 구형·불완전 shape를 다시 허용하지 않도록 negative protocol 사례를 추가한다.

### 정확한 조사 대상

- `packages/shared/src/ws.test.ts`에서 version 없는 presence/event가 거부되는지 확인한다.
- snapshot의 sequence 누락·잘못된 값과 중첩된 `snapshot.state` 구조를 검사한다.
- `persisted: true` 결과가 유효한 `matchId` 없이 통과할 수 없는 discriminator를 확인한다.
- 이전 이벤트 형식과 version-1 형식의 차이를 해당 SHA에서 직접 비교한다.

### 학습자 복원

<!-- ANSWER:f655969b0d36:begin -->
#### 역사적 복원

versioned codec 도입 직후의 위험은 새 형식을 정상적으로 encode하는 것보다, 이전 형식이 느슨한 union이나 optional 필드 때문에 계속 통과하는 것이다. 이 커밋은 그 호환성 구멍을 negative test로 막는다.

테스트는 `v`가 없는 presence 이벤트, sequence가 없는 snapshot, 영속 결과를 주장하면서 `matchId`가 없는 완료 결과를 실제 파서에 넣어 실패를 요구한다. snapshot은 단순 상태 객체가 아니라 versioned envelope와 중첩 state를 가져야 하며, 영속성 discriminator는 결과 식별자와 결합된다.

이 증거는 wire shape의 fail-closed 성질을 보여 준다. 그러나 같은 sequence가 실제 클라이언트 reducer에서 역행을 막는지, 서버가 stale input을 버리는지는 이 커밋만으로 증명되지 않는다.

#### 이 커밋이 보장하는 것

- 구형 unversioned 이벤트와 불완전 persistence metadata가 다시 수용되지 않는다.
- version, snapshot sequence, persisted-result discriminator가 독립 optional 값이 아니라 결합된 계약임을 고정한다.

#### 이 커밋이 보장하지 않는 것

- 실제 WebSocket 연결이나 재전송 순서를 시험하지 않는다.
- 프로토콜 버전 2와 같은 향후 호환 정책은 정의하지 않는다.

#### 전후 관계

초기 `60c38090effc`의 이벤트 범위를 versioned·monotonic·persistence-aware 계약으로 강화한다.

#### 증거 성격

- 고위험 negative protocol regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:f655969b0d36:end -->

## Commit `d07056e11871` — feat(shared): 모든 HTTP request schema를 strict하게 정의

- Full SHA: `d07056e118716fe7537dd92636d1740867f486b9`
- Subject: `feat(shared): 모든 HTTP request schema를 strict하게 정의`
- Importance: **A**
- Tags: PROTOCOL
- Source-defined role: 모든 JSON HTTP 라우트에 params/query/body의 strict 런타임 계약을 한 곳에서 정의한다.

### 정확한 조사 대상

- `packages/shared/src/http.ts`의 `defineHttpRequestContract`, `emptyHttpRequestContract`, `idHttpRequestContract`를 확인한다.
- 각 라우트가 params, query, body를 모두 명시하며 입력이 없는 위치도 strict empty object로 표현되는지 확인한다.
- unknown key가 Zod 기본 strip으로 사라지는 대신 거부되는지 확인한다.
- 라우트 목록과 request-contract export 사이의 누락 여부를 검사한다.

### 학습자 복원

<!-- ANSWER:d07056e11871:begin -->
#### 역사적 복원

이전 공유 HTTP 스키마는 주요 DTO와 일부 요청을 검증했지만, 모든 라우트가 동일한 params/query/body 틀을 갖지는 않았다. 입력이 없다고 가정한 위치는 unknown query나 body를 조용히 무시할 여지가 있었다.

이 커밋은 `defineHttpRequestContract`를 통해 세 입력 영역을 하나의 계약으로 만들고, 입력이 없는 라우트도 `emptyHttpRequestContract`로 명시한다. UUID 경로를 쓰는 라우트는 `idHttpRequestContract`를 재사용한다. 각 object가 strict이므로 정의되지 않은 키는 strip되지 않고 validation failure가 된다.

소유권은 공유 패키지에 있다. API는 라우트별 schema를 재정의하지 않고 이 계약을 소비해야 한다. 그러나 이 SHA는 계약 정의만 추가하므로 실제 라우트 적용 누락 가능성은 남는다.

#### 이 커밋이 보장하는 것

- 모든 JSON 라우트의 외부 입력 표면이 열거되고 unknown field 정책이 fail-closed가 된다.
- 입력이 없는 params/query/body도 검증 대상이라는 규칙을 세운다.

#### 이 커밋이 보장하지 않는 것

- API handler가 계약을 반드시 호출한다는 보장은 아직 없다.
- 응답 schema 적용이나 비즈니스 authorization까지 대신하지 않는다.

#### 전후 관계

`59d75fddcaa6`이 이 공유 계약을 Fastify 라우트 전면에 연결하며, `1abbf7dcdde4`가 누락 여부를 표 기반으로 보호한다.

#### 증거 성격

- 실행 계약 정의; 후속 integration 검증 필요
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:d07056e11871:end -->

## Commit `59d75fddcaa6` — fix(api): 모든 route input을 runtime 검증

- Full SHA: `59d75fddcaa66da998e23ae6f47c373652143ba2`
- Subject: `fix(api): 모든 route input을 runtime 검증`
- Importance: **A**
- Tags: PROTOCOL, RISK
- Source-defined role: 정의만 존재하던 strict HTTP request contract를 모든 API 라우트의 실제 신뢰 경계에 적용한다.

### 정확한 조사 대상

- `apps/api/src/httpBoundary.ts`의 `parseHttpRequest`가 params/query/body를 함께 parse하는지 확인한다.
- `apps/api/src/app.ts`의 각 JSON 라우트가 비즈니스 로직 전에 올바른 contract를 호출하는지 확인한다.
- Zod 실패가 공통 `validation_error` envelope로 변환되는 경로를 추적한다.
- unknown query가 handler에서 무시되지 않고 400으로 종료되는지 해당 SHA의 테스트 변경과 비교한다.

### 학습자 복원

<!-- ANSWER:59d75fddcaa6:begin -->
#### 역사적 복원

직전 상태에서는 모든 strict contract가 공유 패키지에 있었지만 라우트가 개별적으로 body만 parse하거나 입력을 직접 읽을 수 있었다. 즉 schema 정의와 실제 trust boundary 사이에 적용 공백이 있었다.

`parseHttpRequest`는 Fastify request의 params, query, body를 라우트별 공유 contract에 한 번에 넣고, 성공한 값만 handler에 반환한다. `app.ts`의 JSON 라우트들은 저장소나 GameHub를 호출하기 전에 이 함수를 거친다. 파싱 실패는 공통 typed 오류 경계에서 `validation_error`와 400 응답으로 정규화된다.

이 수정으로 검증 책임은 handler 내부의 임시 검사에서 하나의 HTTP boundary helper로 이동한다. unknown query/body가 조용히 버려지지 않으며 invalid path parameter가 저장소 호출 전에 차단된다.

#### 이 커밋이 보장하는 것

- 모든 당시 JSON 라우트의 params/query/body가 공유 strict contract를 통과한 뒤에만 비즈니스 로직에 도달한다.
- 검증 실패 응답이 공통 오류 envelope로 수렴한다.

#### 이 커밋이 보장하지 않는 것

- handler 이후의 도메인 authorization이나 DB constraint를 대체하지 않는다.
- 새 라우트가 후속에 추가될 때 자동으로 contract 적용을 강제하는 타입 수준 증명은 아니다.

#### 전후 관계

`d07056e11871`의 정의-적용 공백을 수정하고, `1abbf7dcdde4`가 라우트 목록 전체를 회귀 테스트한다.

#### 증거 성격

- 실제 API 신뢰 경계 수정
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:59d75fddcaa6:end -->

## Commit `1abbf7dcdde4` — test(api): strict request contract 검증

- Full SHA: `1abbf7dcdde497172790fe23a057a8b326417793`
- Subject: `test(api): strict request contract 검증`
- Importance: **B**
- Tags: TEST
- Source-defined role: API 전 라우트가 strict request contract를 실제로 사용하는지 table-driven regression으로 확인한다.

### 정확한 조사 대상

- `apps/api/src/http-contract.test.ts`의 라우트 표와 요청 생성 방식을 확인한다.
- 각 JSON 라우트에 unknown query/body를 주입하고 공통 400 `validation_error`를 요구하는지 확인한다.
- 잘못된 path parameter가 비즈니스 저장소 호출 전에 차단되는지 확인한다.
- untrusted `X-Forwarded-For`가 guest/client-address 경계에 영향을 주지 않는 별도 사례를 확인한다.

### 학습자 복원

<!-- ANSWER:1abbf7dcdde4:begin -->
#### 역사적 복원

이 테스트는 특정 helper의 단위 동작이 아니라 라우트 등록 결과를 Fastify injection으로 통과한다. 각 JSON endpoint에 unknown query 또는 body field를 넣고 동일한 validation envelope를 요구한다.

invalid UUID와 같은 path 오류도 handler의 저장소 호출보다 먼저 실패해야 한다. 이 방식은 `parseHttpRequest`를 추가하고도 특정 라우트에서 호출을 빠뜨리는 회귀를 잡는다.

`X-Forwarded-For` 사례는 proxy trust가 없는 환경에서 외부 헤더를 주소 권한의 근거로 삼지 않는다는 별도 입력 경계도 고정한다.

#### 이 커밋이 보장하는 것

- 당시 JSON route set 전체의 strict 적용 누락을 탐지한다.
- unknown field와 invalid path가 공통 400 오류로 수렴함을 증명한다.

#### 이 커밋이 보장하지 않는 것

- 라우트의 성공 응답 의미나 DB side effect 전체를 검증하지 않는다.
- 후속 신규 라우트는 표에 추가되지 않으면 자동으로 포함되지 않는다.

#### 전후 관계

`d07056e11871` → `59d75fddcaa6`의 Contract → Apply 흐름을 회귀 증거로 닫는다.

#### 증거 성격

- table-driven API boundary integration regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:1abbf7dcdde4:end -->


## Thread-level invariant evolution

다음 표에서 불변식이 도입·확장·교정·검증된 SHA를 구분합니다.

<!-- ANSWER:thread-01-invariants:begin -->
### 복원 결과

| SHA | 변화 | 불변식 |
| --- | --- | --- |
| `60c38090effc` | 도입 | 초기 WebSocket client-event vocabulary가 runtime parser와 표 기반 사례로 고정된다. |
| `78cf83f29e80` | 확장 | HTTP 요청·응답의 인증·식별자·mutation 값 경계가 executable schema test가 된다. |
| `f655969b0d36` | 강화 | version·snapshot sequence·persisted result 조합이 구형 shape를 fail-closed로 거부한다. |
| `d07056e11871` | 정의 확대 | 모든 JSON route의 params/query/body가 strict contract로 열거된다. |
| `59d75fddcaa6` | 적용 수정 | 공유 contract가 실제 Fastify handler 앞의 단일 `parseHttpRequest` 경계로 이동한다. |
| `1abbf7dcdde4` | 회귀 보호 | route table 전체가 unknown/invalid input을 공통 400 envelope로 거부한다. |

각 변화는 이전 커밋의 최종 상태를 다음 커밋이 단순 반복한 것이 아니라, 새 경계를 도입·확장·교정하거나 regression으로 보호한 시점을 나타냅니다.
<!-- ANSWER:thread-01-invariants:end -->

## Failure → Fix → Test 관계

구현 추가를 독립 기능으로 나열하지 말고, 이전 가정과 실제 실패 위험, 수정된 결정, 회귀 증거를 연결합니다.

<!-- ANSWER:thread-01-failure-relations:begin -->
### 복원 결과

| Failure / 이전 가정 | Fix / 결정 | Verification |
| --- | --- | --- |
| strict schema 정의만 존재 | `59d75fddcaa6` — route 전면 runtime 적용 | `1abbf7dcdde4` — table-driven route regression |
| version migration 중 구형 shape 재수용 위험 | `f655969b0d36`의 negative cases로 직접 차단 | 동일 커밋의 parser regression |
| 입력이 없는 route가 unknown query/body를 무시할 가능성 | `d07056e11871`의 strict empty contract | `1abbf7dcdde4`의 unknown field 주입 |
<!-- ANSWER:thread-01-failure-relations:end -->

## Ownership·state·responsibility 변화

자원과 상태를 누가 만들고, 보유하고, 이전하고, 정리하는지 커밋 순서에 따라 정리합니다.

<!-- ANSWER:thread-01-ownership:begin -->
### 복원 결과

| 대상 | 소유자 | 책임·수명 |
| --- | --- | --- |
| Wire/HTTP contract | `packages/shared` | request·response·client/server event의 executable shape와 version을 소유한다. |
| HTTP trust boundary | `apps/api/src/httpBoundary.ts` | Fastify raw params/query/body를 parse한 값으로 변환하거나 handler 진입 전에 종료한다. |
| Route coverage evidence | API contract test table | 새·기존 JSON route의 적용 누락을 회귀로 탐지한다. |
| Domain logic | route handler/repository/GameHub | 이미 검증된 값을 소비하되 authorization·transaction invariant는 별도로 소유한다. |
<!-- ANSWER:thread-01-ownership:end -->

## 최종 Thread 상태

최종 HEAD를 과거 커밋에 투영하지 말고, 위 SHA 순서에서 도달한 보장을 요약합니다.

<!-- ANSWER:thread-01-final-state:begin -->
### 최종 상태

- WebSocket와 HTTP의 외부 shape는 공유 runtime schema가 소유한다.
- 모든 당시 JSON API route는 strict params/query/body를 business logic 전에 parse한다.
- version 없는 realtime event, sequence 없는 snapshot, 불완전 persisted result, unknown HTTP field는 fail-closed로 종료된다.
- schema unit test와 route integration test가 분리되어 contract 자체 오류와 적용 누락을 각각 탐지한다.
<!-- ANSWER:thread-01-final-state:end -->

## 최종 architecture / execution flow

실제 caller→boundary→state transition→cleanup 흐름을 순서대로 작성합니다.

<!-- ANSWER:thread-01-flow:begin -->
### 최종 실행 흐름

1. 외부 JSON/HTTP 입력 수신
2. 공유 strict schema 선택
3. version·discriminator·params/query/body runtime parse
4. 실패 시 공통 protocol/HTTP validation error
5. 성공한 typed value만 handler로 전달
6. route-wide regression이 누락·unknown field를 보호
<!-- ANSWER:thread-01-flow:end -->

## 학습 완료 확인

<!-- ANSWER:thread-01-checks:begin -->
### 완료 확인

- [x] 초기 unversioned event와 version-1 event의 차이를 예로 설명할 수 있다.
- [x] Zod object가 strict하지 않을 때 unknown key가 왜 위험한지 해당 route 적용 흐름으로 설명할 수 있다.
- [x] `d070...`만으로 완료가 아니며 `59d...`가 필요한 이유를 설명할 수 있다.
- [x] `1abb...`가 무엇을 증명하고 신규 route 자동 포괄은 왜 보장하지 않는지 설명할 수 있다.

체크는 repository code inspection을 통해 설명이 문서에 작성되었음을 뜻합니다. 실제 repository test suite 실행 완료를 뜻하지 않습니다.
<!-- ANSWER:thread-01-checks:end -->

## 실행 증거 기록

<!-- ANSWER:thread-01-execution:begin -->
- Historical inspection: 각 참조 SHA의 GitHub commit diff와 변경 파일을 개별 조회했습니다.
- Repository test execution: 실행하지 않았습니다. 로컬 환경에서 지정 branch의 전체 checkout을 materialize하지 못해 pnpm, PostgreSQL, Playwright, k6, Toxiproxy 명령 결과를 만들 수 없었습니다.
- Runtime claims: 기록하지 않았습니다. 위의 `증거 성격`은 test 구현과 production code path에 대한 정적·역사적 검사입니다.
- Artifact validation: scaffold/completed 대응, fixed metadata, answer completion, SHA uniqueness, Markdown fence, ZIP 구조는 로컬 검증 스크립트로 검사했습니다.
<!-- ANSWER:thread-01-execution:end -->
===== END FILE: 01-executable-protocol-and-http-contract-verification.md =====

===== BEGIN FILE: 02-authentication-ticket-failure-containment-and-transport-limits.md =====
# Development Thread 02: 인증·ticket·실패 격리·transport 상한 검증

English title: **Authentication, Ticket, Failure Containment, and Transport Limits**

## Thread 목표

지속 session을 cookie로 제한하고 one-time ticket으로 realtime 권한을 이전하며, 오류 정보와 미인증·oversized 입력의 자원 사용을 제한하는 검증 구조를 복원한다.

## 핵심 질문

- 왜 HttpOnly cookie를 WebSocket URL credential로 재사용하지 않고 ticket handoff를 두는가?
- ticket single-use는 application check가 아니라 어떤 저장소 원자성으로 증명되는가?
- 비동기 인증 중 메시지 보존과 memory bound를 어떻게 동시에 만족하는가?
- client-caused parse 오류와 repository/internal 오류의 외부 정보 수준은 왜 달라야 하는가?
- application buffer limit만으로 인증 후 oversized frame을 막지 못한 이유는 무엇인가?

## 완료 기준

- cookie→HTTP ticket issuance→hash storage→atomic consume→GameHub ownership transfer를 설명한다.
- pre-auth message/개수/누적 상한과 transport `maxPayload`의 계층 차이를 설명한다.
- replay·expiry·suspension·unsupported version·동시 consume의 상태 결과를 재구성한다.
- internal error redaction과 deterministic failure injection을 연결한다.
- S/A/B 중요도에 맞춰 credential ownership과 cleanup을 깊이 구분한다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags |
| --- | --- | --- | --- | --- |
| 1 | `d0531791406b` | `fix(auth): cookie-only session과 환경별 route 적용` | S | AUTH, ARCH, RISK |
| 2 | `401cf13d9d17` | `test(auth): cookie session 경계 검증` | A | AUTH, RISK, TEST |
| 3 | `306d1946afb7` | `feat(auth): ticket 기반 WebSocket 인증 연결` | S | AUTH, REALTIME, RISK |
| 4 | `b0ee833313c1` | `test(auth): WebSocket ticket 경계 검증` | A | AUTH, PROTOCOL, REALTIME |
| 5 | `fe62962d65d9` | `fix(api): 내부 WebSocket 오류 숨김` | A | PROTOCOL, REALTIME, PERSISTENCE |
| 6 | `20933b1393f3` | `test(api): WebSocket repository error redaction 검증` | B | PROTOCOL, REALTIME, PERSISTENCE |
| 7 | `8ea18a1b92db` | `fix(realtime): WebSocket transport payload 상한 설정` | A | AUTH, REALTIME, RISK |
| 8 | `1afec49052b6` | `test(realtime): oversized WebSocket frame 거부 검증` | B | AUTH, REALTIME, TEST |

> Commit 순서·subject·importance·tags는 Phase 1 감사 후 동결된 정보입니다. Phase 2에서는 변경하지 않습니다.

## Commit `d0531791406b` — fix(auth): cookie-only session과 환경별 route 적용

- Full SHA: `d0531791406bd9809f15363c8542b1583cf7f564`
- Subject: `fix(auth): cookie-only session과 환경별 route 적용`
- Importance: **S**
- Tags: AUTH, ARCH, RISK
- Source-defined role: 내구성 세션 credential을 HttpOnly cookie 하나로 제한하고 개발 전용 인증 기능을 실행 모드로 격리한다.

### 정확한 조사 대상

- `apps/api/src/app.ts`에서 세션 읽기가 `pp_session` cookie만 허용하도록 바뀐 지점을 확인한다.
- Authorization bearer와 URL query session fallback 제거, CORS 허용 header 축소를 함께 추적한다.
- development/test에서만 dev-login route가 등록되고 production/demo에서는 노출되지 않는지 확인한다.
- cookie의 HttpOnly, SameSite, Secure, Path, expiry 속성이 실행 모드별로 어떻게 설정되는지 확인한다.
- 이 변경 전 브라우저·smoke 코드가 토큰을 직접 보관하던 상태와 비교한다.

### 학습자 복원

<!-- ANSWER:d0531791406b:begin -->
#### 역사적 복원

이전 구조는 동일한 장기 세션을 cookie, bearer header, URL query에서 받아들였다. 이 구조에서는 JavaScript 저장소, access log, referrer, 프록시 기록 등 여러 경로로 재사용 가능한 credential이 노출될 수 있고, 개발 로그인도 배포 모드에서 실수로 살아남을 수 있었다.

이 커밋은 인증 credential의 소유권을 브라우저의 HttpOnly `pp_session` cookie로 단일화한다. request 인증은 cookie만 읽고 bearer/query fallback과 CORS Authorization 허용을 제거한다. 외부 모드의 cookie는 Secure가 적용되며, dev-login route는 development/test에서만 등록된다.

상태 전이는 로그인 성공 시 서버가 cookie를 설정하고, 이후 HTTP 요청은 브라우저가 cookie를 자동 전송하는 방식으로 바뀐다. JavaScript는 장기 세션 값을 읽거나 URL에 붙일 책임을 잃는다. production/demo에서 개발용 route를 아예 등록하지 않는 것은 handler 내부 권한 검사보다 강한 비노출 경계다.

그러나 WebSocket 업그레이드에 장기 cookie 값을 그대로 URL에 넣을 수는 없다. 이 커밋은 그 문제를 일부러 남기며, `306d1946afb7`의 one-time ticket handoff가 이를 해결한다.

#### 이 커밋이 보장하는 것

- 지속 세션은 HttpOnly cookie로만 운반된다.
- 개발 로그인은 development/test 모드에만 존재한다.
- bearer·query 기반 재사용 토큰 경로와 관련 CORS 표면이 제거된다.

#### 이 커밋이 보장하지 않는 것

- WebSocket 인증 자체는 아직 완성하지 않는다.
- cookie를 발급한 세션의 서버 측 폐기·계정 정지 후 열린 연결 회수까지 모두 보장하는 커밋은 아니다.

#### 전후 관계

`401cf13d9d17`이 cookie 속성·금지 transport·모드 격리를 검증하고, `306d1946afb7`이 cookie에서 one-time WebSocket ticket으로 권한을 넘긴다.

#### 증거 성격

- 핵심 credential ownership 전환
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:d0531791406b:end -->

## Commit `401cf13d9d17` — test(auth): cookie session 경계 검증

- Full SHA: `401cf13d9d17e5a6095e311f3564dba6d4ca0c46`
- Subject: `test(auth): cookie session 경계 검증`
- Importance: **A**
- Tags: AUTH, RISK, TEST
- Source-defined role: cookie-only 전환의 허용·금지 credential transport와 실행 모드 격리를 Fastify 경계에서 검증한다.

### 정확한 조사 대상

- Fastify injection으로 dev-login 응답의 `Set-Cookie` 속성을 검사한다.
- cookie 없는 bearer header와 query token이 인증되지 않는지 확인한다.
- `admin` handle만으로 role을 획득하지 못하는지 확인한다.
- production/demo에서 dev-login route가 404인지, 공통 오류 envelope가 유지되는지 확인한다.

### 학습자 복원

<!-- ANSWER:401cf13d9d17:begin -->
#### 역사적 복원

테스트는 내부 cookie helper를 직접 호출하지 않고 실제 라우트 등록과 Fastify 응답을 통해 cookie 속성과 인증 결과를 관찰한다. 개발 모드 로그인은 HttpOnly session cookie를 설정하지만 bearer/query만 제시한 요청은 인증되지 않는다.

또한 `admin`이라는 문자열을 handle로 선택하는 것과 실제 관리 role은 분리된다. production/demo에서는 dev-login handler가 실패하는 것이 아니라 route 자체가 등록되지 않아야 한다.

이 테스트가 보호하는 핵심은 허용 경로 한 개와 금지 경로 여러 개를 동시에 고정하는 것이다. 단순 로그인 성공 테스트만으로는 credential 표면 축소를 증명할 수 없다.

#### 이 커밋이 보장하는 것

- cookie 속성, 금지 bearer/query transport, 환경별 route 비노출을 통합적으로 증명한다.
- handle과 authorization role이 분리되어 있음을 확인한다.

#### 이 커밋이 보장하지 않는 것

- 브라우저에서 HttpOnly가 실제로 JavaScript 접근을 막는지 직접 실행하지 않는다.
- WebSocket ticket이나 동시 소비는 대상이 아니다.

#### 전후 관계

`d0531791406b`의 보안 경계를 고정하고, 뒤의 ticket 테스트와 결합되어 HTTP→WebSocket trust chain을 구성한다.

#### 증거 성격

- Fastify authentication boundary regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:401cf13d9d17:end -->

## Commit `306d1946afb7` — feat(auth): ticket 기반 WebSocket 인증 연결

- Full SHA: `306d1946afb76665fa42f5443cfd56d10516d83a`
- Subject: `feat(auth): ticket 기반 WebSocket 인증 연결`
- Importance: **S**
- Tags: AUTH, REALTIME, RISK
- Source-defined role: HttpOnly cookie로 인증된 HTTP 세션을 짧고 단일 사용인 WebSocket ticket으로 원자적으로 넘긴다.

### 정확한 조사 대상

- `apps/api/src/app.ts`의 `/auth/ws-ticket` 발급 route와 `/ws` upgrade/handshake 경로를 추적한다.
- raw ticket 생성 길이·TTL·protocol version과 repository의 hash 저장/consume 계약을 확인한다.
- 인증 전 message buffer의 8KiB per-message, 16개, 총 32KiB 제한과 1008/1009 close code를 확인한다.
- 비동기 consume 중 도착한 메시지를 보존한 뒤 인증 성공 시 순서대로 넘기고 listener를 제거하는지 확인한다.
- socket이 인증 완료 전에 닫힌 경우 GameHub에 client를 등록하지 않는 cleanup 경로를 확인한다.

### 학습자 복원

<!-- ANSWER:306d1946afb7:begin -->
#### 역사적 복원

cookie-only 모델은 HTTP에는 적합하지만 WebSocket URL에서 지속 credential을 노출하거나 JavaScript가 cookie 값을 읽게 만들 수 없다. 또한 ticket 검증이 비동기인 동안 초기 client message를 잃거나 무제한 버퍼링하면 기능·메모리 안전성이 깨진다.

발급 경로는 HttpOnly cookie로 HTTP 사용자를 인증한 뒤 cryptographically random raw ticket을 반환하고 repository에는 hash만 저장한다. upgrade 경로는 `ticket`과 protocol version을 strict하게 검증하고 repository의 consume 작업으로 한 번만 사용자 identity를 얻는다. raw ticket은 장기 세션이 아니며 TTL 이후 또는 한 번 소비된 뒤 재사용할 수 없다.

pre-auth 단계는 별도 수명주기를 가진다. 메시지 하나는 8KiB, 개수는 16, 누적은 32KiB로 제한된다. 초과 시 정책 위반 또는 payload-too-large close code로 닫고 listener와 임시 버퍼를 정리한다. 성공하면 임시 listener를 제거한 뒤 버퍼를 도착 순서대로 GameHub 경계에 재생한다. socket이 먼저 닫혔다면 인증 결과가 돌아와도 client ownership을 만들지 않는다.

이 설계에서 HTTP 세션은 ticket 발급 권한만 소유하고, repository는 ticket single-use 상태를 소유하며, GameHub는 consume 성공 후의 realtime connection만 소유한다. 권한과 자원 수명이 단계별로 분리된다.

#### 이 커밋이 보장하는 것

- 장기 session credential을 URL/JavaScript에 노출하지 않고 WebSocket identity를 확립한다.
- ticket은 짧은 수명·hash-at-rest·single-use이며 인증 전 메모리 사용은 명시적으로 제한된다.
- 비동기 인증 중 초기 메시지의 순서와 cleanup이 정의된다.

#### 이 커밋이 보장하지 않는 것

- 네트워크 중간자의 TLS 보호 자체를 제공하지 않는다.
- repository 구현이 atomic consume을 지키는지는 별도 테스트가 필요하다.
- 인증 후의 모든 frame 크기 제한은 이 SHA의 application buffer만으로 완전하지 않아 `8ea18a1b92db`가 transport 상한을 추가한다.

#### 전후 관계

`b0ee833313c1`이 replay·expiry·suspension·동시 소비·pre-auth 한계를 실제 경계에서 검증하고, `8ea18a1b92db`이 underlying ws transport에도 같은 8KiB 상한을 적용한다.

#### 증거 성격

- 핵심 HTTP-to-realtime authentication handoff
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:306d1946afb7:end -->

## Commit `b0ee833313c1` — test(auth): WebSocket ticket 경계 검증

- Full SHA: `b0ee833313c1d1529828e4b1802207fc3bc08a88`
- Subject: `test(auth): WebSocket ticket 경계 검증`
- Importance: **A**
- Tags: AUTH, PROTOCOL, REALTIME
- Source-defined role: ticket 발급·저장·소비·handshake·pre-auth 제한을 실제 server/socket와 memory/PostgreSQL 저장소에서 검증한다.

### 정확한 조사 대상

- ticket 길이, 30초 TTL, protocol version 1 응답을 검사한다.
- DB에는 raw ticket이 아니라 hash만 저장되는지 SQL 조회로 확인한다.
- replay, forged, expired, suspended ticket이 거부되고 unsupported version이 ticket을 소비하지 않는지 확인한다.
- memory와 PostgreSQL에서 동일 ticket의 동시 consume 중 정확히 한 호출만 성공하는지 확인한다.
- 실제 WebSocket에서 durable session만으로 인증되지 않으며 pre-auth count/bytes/message 제한이 정확한지 확인한다.

### 학습자 복원

<!-- ANSWER:b0ee833313c1:begin -->
#### 역사적 복원

이 테스트 묶음은 ticket helper의 정상 결과만 보는 것이 아니라 trust chain 전체의 실패 모델을 재현한다. 발급 응답은 43자 raw ticket, 30초 만료, protocol version 1을 가져야 하고, 저장소 조회에서는 raw 값이 발견되지 않고 hash만 남아야 한다.

replay·위조·만료·정지 계정은 handshake를 얻지 못한다. 지원되지 않는 protocol version은 consume 전에 거부되어 같은 유효 ticket으로 올바른 version handshake를 다시 시도할 수 있다. 반대로 한 번 성공한 ticket은 재사용할 수 없다.

memory와 PostgreSQL 구현 모두 동일 ticket을 병렬 consume하며 fulfilled 결과가 정확히 하나여야 한다. 이는 single-use가 프로세스 로컬 check-then-delete가 아니라 repository 원자성으로 보호된다는 핵심 증거다.

실제 socket 테스트는 cookie/session token만 query에 넣는 구형 경로를 거부하고, 인증 전 메시지 수·누적 바이트·단일 메시지 한계를 close code와 함께 확인한다.

#### 이 커밋이 보장하는 것

- 발급부터 socket admission까지 ticket security invariant를 통합적으로 증명한다.
- memory와 PostgreSQL이 동시 소비에서도 one winner semantics를 보존한다.
- version 검증이 ticket 상태를 불필요하게 소모하지 않는다.

#### 이 커밋이 보장하지 않는 것

- TLS, 프록시 로그 정책, 브라우저 재연결 UX는 검증하지 않는다.
- 인증 성공 뒤 8KiB를 넘는 단일 transport frame은 이후 별도 fix/test가 필요했다.

#### 전후 관계

`306d1946afb7`의 설계를 고위험 regression evidence로 닫고, 후속 `8ea18a1b92db`/`1afec49052b6`가 인증 후 transport payload 공백을 보완한다.

#### 증거 성격

- real socket + repository concurrency security regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:b0ee833313c1:end -->

## Commit `fe62962d65d9` — fix(api): 내부 WebSocket 오류 숨김

- Full SHA: `fe62962d65d90c136f826a6b271d17c991a5ffd6`
- Subject: `fix(api): 내부 WebSocket 오류 숨김`
- Importance: **A**
- Tags: PROTOCOL, REALTIME, PERSISTENCE
- Source-defined role: client parse 오류와 내부 처리·저장소 오류를 분리하여 내부 상세가 realtime 응답으로 노출되지 않게 한다.

### 정확한 조사 대상

- `apps/api/src/gameHub.ts`의 message 처리 try/catch와 오류 분류를 확인한다.
- protocol validation 실패가 안정된 client error code/message로 변환되는지 확인한다.
- repository/내부 예외가 generic Korean message와 `internal_error`로 변환되는지 확인한다.
- 로그와 client response가 서로 다른 정보 경계를 갖는지 확인한다.

### 학습자 복원

<!-- ANSWER:fe62962d65d9:begin -->
#### 역사적 복원

이전에는 WebSocket command 처리 중 발생한 예외 문자열이 동일한 오류 응답 경로를 통해 client로 전달될 수 있었다. 저장소 예외에는 SQL, host, schema, 내부 필드명이 포함될 수 있어 protocol 입력 오류와 같은 방식으로 노출하면 안 된다.

수정은 파서가 보고하는 client-caused validation failure와 handler/repository가 던지는 internal failure를 분리한다. 전자는 안정된 protocol error code로 설명하지만, 후자는 client에 generic message와 `internal_error`만 보낸다. 내부 진단은 server-side 관측 경계에 남는다.

오류 소유권을 나눔으로써 protocol은 예상 가능한 외부 실패만 표현하고 infrastructure detail은 운영 로그의 책임으로 유지된다.

#### 이 커밋이 보장하는 것

- repository·내부 예외 문자열이 client realtime payload로 직접 새지 않는다.
- client validation 오류와 server failure의 외부 code가 구분된다.

#### 이 커밋이 보장하지 않는 것

- 로그 자체의 credential redaction 전체를 검증하지 않는다.
- 모든 비-WebSocket HTTP 오류 노출 정책까지 변경하지 않는다.

#### 전후 관계

`20933b1393f3`이 SQL과 내부 hostname을 포함한 강제 repository 실패로 이 비노출 경계를 검증한다.

#### 증거 성격

- failure containment and error redaction fix
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:fe62962d65d9:end -->

## Commit `20933b1393f3` — test(api): WebSocket repository error redaction 검증

- Full SHA: `20933b1393f3a49ff73138c3770d0abcf44d28ca`
- Subject: `test(api): WebSocket repository error redaction 검증`
- Importance: **B**
- Tags: PROTOCOL, REALTIME, PERSISTENCE
- Source-defined role: 민감한 내부 문자열을 가진 repository 실패를 실제 WebSocket command 경로에 주입해 redaction을 검증한다.

### 정확한 조사 대상

- 테스트 repository가 어떤 오류 문자열(SQL·내부 host)을 던지는지 확인한다.
- protocol-valid command를 통해 해당 production handler가 실행되는지 확인한다.
- client가 받는 event의 `code`, message, version을 확인한다.
- raw payload 어디에도 `password_hash`나 내부 hostname이 없는지 확인한다.

### 학습자 복원

<!-- ANSWER:20933b1393f3:begin -->
#### 역사적 복원

테스트는 parse 단계에서 실패시키지 않는다. 유효한 realtime command가 repository를 호출한 뒤 `password_hash`와 내부 host 정보를 포함한 오류를 던지게 하여 내부 failure branch를 직접 통과한다.

client는 versioned error event와 generic `internal_error`만 받아야 하며, 원래 예외 텍스트는 직렬화된 payload 어디에도 나타나지 않아야 한다. 따라서 단순 정상 경로가 아니라 실제 정보 노출 회귀를 재현한다.

#### 이 커밋이 보장하는 것

- 내부 repository 오류 상세가 WebSocket client 응답으로 노출되지 않음을 증명한다.

#### 이 커밋이 보장하지 않는 것

- server log에 기록되는 메타데이터의 전체 redaction 규칙은 이 테스트 범위가 아니다.
- 모든 종류의 infrastructure error를 열거하지 않는다.

#### 전후 관계

`fe62962d65d9`의 내부/외부 오류 경계에 대한 deterministic regression이다.

#### 증거 성격

- deterministic failure injection / information-disclosure regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:20933b1393f3:end -->

## Commit `8ea18a1b92db` — fix(realtime): WebSocket transport payload 상한 설정

- Full SHA: `8ea18a1b92db8d3a92232a98aae22c0f3cd1a9a1`
- Subject: `fix(realtime): WebSocket transport payload 상한 설정`
- Importance: **A**
- Tags: AUTH, REALTIME, RISK
- Source-defined role: application pre-auth buffer 제한과 동일한 8KiB 상한을 underlying `ws` server transport에도 적용한다.

### 정확한 조사 대상

- Fastify WebSocket plugin/server 생성 옵션의 `maxPayload` 값을 확인한다.
- 기존 `PRE_AUTH_MESSAGE_MAX_BYTES` 상수를 재사용하는지 확인한다.
- 인증 전 application 검사와 인증 후 transport 검사 사이의 이전 공백을 비교한다.
- oversized frame이 application JSON parser 전에 어떤 close path를 타는지 확인한다.

### 학습자 복원

<!-- ANSWER:8ea18a1b92db:begin -->
#### 역사적 복원

기존 코드에는 인증 전 수동 buffer가 8KiB 제한을 적용했지만, 인증이 끝난 뒤에는 underlying WebSocket transport가 더 큰 frame을 받아 메모리에 조립할 수 있었다. 같은 시스템에 두 개의 서로 다른 payload 한계가 존재한 셈이다.

수정은 `ws` server의 `maxPayload`를 기존 8KiB 상수에 맞춘다. 제한은 JSON parse나 GameHub dispatch 이전의 transport 계층에서 적용되므로, 인증 상태와 무관하게 oversized frame이 전체 수신·할당되는 것을 막는다.

자원 경계의 소유권은 application callback이 아니라 실제 frame decoder로 내려간다. 결과적으로 pre-auth와 authenticated transport가 동일한 최대 메시지 크기를 공유한다.

#### 이 커밋이 보장하는 것

- 인증 전후 모든 WebSocket frame에 8KiB hard limit가 적용된다.
- oversized payload가 business handler에 도달하기 전에 transport에서 종료된다.

#### 이 커밋이 보장하지 않는 것

- 메시지 빈도·누적 처리량·정상 크기 frame의 rate limit까지 해결하지 않는다.
- 프록시가 더 큰 raw network buffer를 가지는 문제와는 별개다.

#### 전후 관계

`1afec49052b6`이 인증된 실제 socket에서 8,193-byte frame을 보내 close code 1009를 요구한다.

#### 증거 성격

- transport-layer resource limit fix
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:8ea18a1b92db:end -->

## Commit `1afec49052b6` — test(realtime): oversized WebSocket frame 거부 검증

- Full SHA: `1afec49052b60103508e22ca9ff57a87013566e5`
- Subject: `test(realtime): oversized WebSocket frame 거부 검증`
- Importance: **B**
- Tags: AUTH, REALTIME, TEST
- Source-defined role: 인증된 실제 WebSocket에서도 8KiB를 넘는 frame이 transport close 1009로 거부되는지 검증한다.

### 정확한 조사 대상

- 테스트가 정상 ticket handshake를 완료한 뒤 frame을 보내는 순서를 확인한다.
- payload 길이가 정확히 8,193 bytes인지 확인한다.
- server가 보내는 application error가 아니라 socket close code 1009를 관찰하는지 확인한다.
- GameHub/repository side effect가 발생하지 않는 경로인지 확인한다.

### 학습자 복원

<!-- ANSWER:1afec49052b6:begin -->
#### 역사적 복원

테스트는 pre-auth 상태가 아니라 유효 ticket으로 인증된 실제 socket을 사용한다. 이후 8,193-byte frame을 보내 underlying `maxPayload` 경계를 한 바이트 초과한다.

기대 결과는 application-level validation error가 아니라 WebSocket close code 1009다. 이 차이는 제한이 handler가 아니라 transport decoder에서 작동함을 보여 준다.

#### 이 커밋이 보장하는 것

- authenticated connection에도 hard payload limit가 동일하게 적용됨을 증명한다.

#### 이 커밋이 보장하지 않는 것

- 8KiB 이하 payload의 의미 검증이나 rate limit은 증명하지 않는다.

#### 전후 관계

`8ea18a1b92db`의 transport-level fix를 실제 socket 회귀로 고정한다.

#### 증거 성격

- real transport boundary regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:1afec49052b6:end -->


## Thread-level invariant evolution

다음 표에서 불변식이 도입·확장·교정·검증된 SHA를 구분합니다.

<!-- ANSWER:thread-02-invariants:begin -->
### 복원 결과

| SHA | 변화 | 불변식 |
| --- | --- | --- |
| `d0531791406b` | 도입 | 지속 credential은 HttpOnly `pp_session` cookie 한 경로로만 운반되고 dev-login은 실행 모드로 격리된다. |
| `401cf13d9d17` | 회귀 보호 | cookie 속성·금지 bearer/query transport·route 비노출·role 분리가 검증된다. |
| `306d1946afb7` | 소유권 이전 | HTTP session은 ticket 발급만 허용하고 repository가 hash·single-use를 소유하며 consume 뒤 GameHub가 connection을 소유한다. |
| `b0ee833313c1` | 경계 검증 | replay·expiry·suspension·version·동시 소비·pre-auth limits가 실제 socket/DB에서 검증된다. |
| `fe62962d65d9` | 실패 격리 | client validation과 internal/repository failure의 외부 오류 정보가 분리된다. |
| `20933b1393f3` | 정보 노출 회귀 | SQL/내부 host를 가진 강제 실패에서도 generic event만 보인다. |
| `8ea18a1b92db` | 상한 교정 | 8KiB 제한이 pre-auth application buffer에서 underlying transport 전체로 내려간다. |
| `1afec49052b6` | transport 회귀 | 인증 후 8,193-byte frame이 1009로 종료된다. |

각 변화는 이전 커밋의 최종 상태를 다음 커밋이 단순 반복한 것이 아니라, 새 경계를 도입·확장·교정하거나 regression으로 보호한 시점을 나타냅니다.
<!-- ANSWER:thread-02-invariants:end -->

## Failure → Fix → Test 관계

구현 추가를 독립 기능으로 나열하지 말고, 이전 가정과 실제 실패 위험, 수정된 결정, 회귀 증거를 연결합니다.

<!-- ANSWER:thread-02-failure-relations:begin -->
### 복원 결과

| Failure / 이전 가정 | Fix / 결정 | Verification |
| --- | --- | --- |
| 재사용 session을 header/query/URL에서 수용 | `d0531791406b` — cookie-only + mode-scoped route | `401cf13d9d17` — 금지 transport 검증 |
| WebSocket identity에 지속 credential 노출 위험 | `306d1946afb7` — short-lived hash-only single-use ticket | `b0ee833313c1` — replay/expiry/concurrency/socket regression |
| repository 예외 상세가 client로 노출 | `fe62962d65d9` — internal error genericization | `20933b1393f3` — SQL/host failure injection |
| pre-auth에는 8KiB 제한이 있지만 인증 후 transport에는 공백 | `8ea18a1b92db` — `ws.maxPayload` 적용 | `1afec49052b6` — authenticated 8,193-byte frame 1009 |
<!-- ANSWER:thread-02-failure-relations:end -->

## Ownership·state·responsibility 변화

자원과 상태를 누가 만들고, 보유하고, 이전하고, 정리하는지 커밋 순서에 따라 정리합니다.

<!-- ANSWER:thread-02-ownership:begin -->
### 복원 결과

| 대상 | 소유자 | 책임·수명 |
| --- | --- | --- |
| Durable browser identity | HttpOnly `pp_session` cookie | 브라우저가 자동 전송하며 JavaScript·URL은 값을 소유하지 않는다. |
| Ticket issuance authority | Authenticated HTTP route | 현재 session을 확인한 뒤 raw one-time ticket을 잠시 client에 반환한다. |
| Ticket state | Memory/PostgreSQL repository | hash, expiry, user, consumed 상태와 atomic one-winner consume을 소유한다. |
| Pre-auth buffer | WebSocket route | consume 완료 전 message 순서와 8KiB/16개/32KiB cleanup을 소유한다. |
| Authenticated connection | GameHub | consume 성공·socket open 후 command lifecycle을 소유한다. |
| Frame hard limit | underlying `ws` server | 인증 상태와 무관하게 decoder 단계에서 8KiB를 강제한다. |
| Internal diagnostics | server observability boundary | client에게는 generic code만 주고 내부 상세를 외부 protocol에서 분리한다. |
<!-- ANSWER:thread-02-ownership:end -->

## 최종 Thread 상태

최종 HEAD를 과거 커밋에 투영하지 말고, 위 SHA 순서에서 도달한 보장을 요약합니다.

<!-- ANSWER:thread-02-final-state:begin -->
### 최종 상태

- 지속 credential은 cookie-only이며 개발 인증 route는 production/demo에 존재하지 않는다.
- WebSocket은 HTTP session으로 발급한 짧은 raw ticket을 hash-at-rest로 저장하고 원자적으로 한 번 소비한다.
- 인증 전 메시지는 손실 없이 bounded buffer에 보관되며 인증 후 전체 frame도 transport 8KiB 상한을 따른다.
- protocol 오류와 internal failure가 분리되고 repository 상세는 client payload로 노출되지 않는다.
<!-- ANSWER:thread-02-final-state:end -->

## 최종 architecture / execution flow

실제 caller→boundary→state transition→cleanup 흐름을 순서대로 작성합니다.

<!-- ANSWER:thread-02-flow:begin -->
### 최종 실행 흐름

1. 브라우저가 HttpOnly cookie로 ticket HTTP 요청
2. 서버가 random raw ticket 반환·repository에 hash/expiry 저장
3. client가 `ticket` + supported version으로 WebSocket handshake
4. route가 bounded pre-auth buffer를 유지하며 atomic consume
5. 성공 시 listener 교체·버퍼 순서 재생·GameHub ownership 생성
6. protocol 오류는 stable client error, 내부 오류는 generic `internal_error`
7. 모든 frame은 underlying transport 8KiB 상한 적용
<!-- ANSWER:thread-02-flow:end -->

## 학습 완료 확인

<!-- ANSWER:thread-02-checks:begin -->
### 완료 확인

- [x] unsupported protocol version이 유효 ticket을 소비하면 안 되는 이유를 설명할 수 있다.
- [x] 동시 consume에서 exactly one success를 memory와 PostgreSQL 모두 확인해야 하는 이유를 설명할 수 있다.
- [x] pre-auth cumulative limit와 `maxPayload`가 해결하는 공격 표면의 차이를 설명할 수 있다.
- [x] send/parse/repository 실패 중 어떤 정보가 client와 server에 각각 남아야 하는지 설명할 수 있다.

체크는 repository code inspection을 통해 설명이 문서에 작성되었음을 뜻합니다. 실제 repository test suite 실행 완료를 뜻하지 않습니다.
<!-- ANSWER:thread-02-checks:end -->

## 실행 증거 기록

<!-- ANSWER:thread-02-execution:begin -->
- Historical inspection: 각 참조 SHA의 GitHub commit diff와 변경 파일을 개별 조회했습니다.
- Repository test execution: 실행하지 않았습니다. 로컬 환경에서 지정 branch의 전체 checkout을 materialize하지 못해 pnpm, PostgreSQL, Playwright, k6, Toxiproxy 명령 결과를 만들 수 없었습니다.
- Runtime claims: 기록하지 않았습니다. 위의 `증거 성격`은 test 구현과 production code path에 대한 정적·역사적 검사입니다.
- Artifact validation: scaffold/completed 대응, fixed metadata, answer completion, SHA uniqueness, Markdown fence, ZIP 구조는 로컬 검증 스크립트로 검사했습니다.
<!-- ANSWER:thread-02-execution:end -->
===== END FILE: 02-authentication-ticket-failure-containment-and-transport-limits.md =====

===== BEGIN FILE: 03-deterministic-simulation-timing-and-snapshot-delivery-verification.md =====
# Development Thread 03: 결정적 simulation·시간·snapshot 전달 검증

English title: **Deterministic Simulation, Timing, and Snapshot Delivery Verification**

## Thread 목표

순수 simulation 결정성에서 fixed-step 시간 변환, versioned replay, latest snapshot backpressure와 callback 오판 교정까지의 검증 관계를 복원한다.

## 핵심 질문

- 동일 입력 결정성에는 state immutability, seeded randomness, fixed delta가 왜 함께 필요한가?
- wall-clock 지연을 simulation work로 그대로 변환하지 않고 accumulator와 catch-up cap을 두는 이유는 무엇인가?
- versioned replay fixture가 짧은 단위 테스트와 다른 호환성 증거를 어떻게 제공하는가?
- snapshot은 왜 reliable FIFO가 아니라 latest-value buffer이며, 어떤 control event와 구분되는가?
- send callback 지연과 실제 transport congestion을 동일시한 가정은 왜 틀렸는가?

## 완료 기준

- pure simulation, scheduler, replay fixture, delivery buffer의 책임을 분리한다.
- 1,000-tick digest와 versioned fixture가 증명하는 것과 못 하는 것을 구분한다.
- 50ms step, remainder, 250ms catch-up cap을 수치로 설명한다.
- soft/hard backpressure와 callback-latency fix/test 관계를 재구성한다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags |
| --- | --- | --- | --- | --- |
| 1 | `4ef4beeb8611` | `test(game): 결정적 simulation 검증` | A | SIMULATION, REALTIME, TEST |
| 2 | `0888e119036d` | `test(game): fixed-step 보정 범위 검증` | B | SIMULATION, REALTIME, TEST |
| 3 | `125aa113a01c` | `test(game): snapshot replacement와 congestion 검증` | A | REALTIME, PERF, RISK |
| 4 | `37a7b2e4611b` | `test(game): versioned match replay fixture 추가` | A | SIMULATION, REALTIME, TEST |
| 5 | `d90f17fa765d` | `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음` | A | REALTIME, PERF, RISK |
| 6 | `5cd54767858f` | `test(game): callback 지연과 실제 congestion 구분` | A | REALTIME, PERF, TEST |

> Commit 순서·subject·importance·tags는 Phase 1 감사 후 동결된 정보입니다. Phase 2에서는 변경하지 않습니다.

## Commit `4ef4beeb8611` — test(game): 결정적 simulation 검증

- Full SHA: `4ef4beeb8611a17f47764065a9b4da2fc16cd463`
- Subject: `test(game): 결정적 simulation 검증`
- Importance: **A**
- Tags: SIMULATION, REALTIME, TEST
- Source-defined role: 추출된 Pong simulation과 AI가 동일 입력에서 동일 결과를 만들고 이전 상태를 변경하지 않는다는 핵심 결정성 계약을 검증한다.

### 정확한 조사 대상

- `apps/api/src/pongSimulation.test.ts`와 `pongAi.test.ts`의 seeded 실행을 확인한다.
- integer PRNG·동일 seed가 동일 AI command와 snapshot sequence를 만드는지 확인한다.
- 이전 state 및 중첩 객체를 변경하지 않고 새 state를 반환하는 deep immutability 검사를 확인한다.
- delta scaling, paddle clamp, collision/scoring/termination 경계와 1,000-tick digest 검사를 확인한다.
- simulation source에 `Math.random` 또는 시간 의존 호출이 없는지 source guard를 확인한다.

### 학습자 복원

<!-- ANSWER:4ef4beeb8611:begin -->
#### 역사적 복원

simulation을 GameHub에서 분리했더라도 내부에서 전역 난수나 wall clock을 읽거나 이전 객체를 mutate하면 replay와 다중 실행 비교가 깨진다. 이 커밋은 구조 분리만으로는 보장되지 않는 결정성·불변성 조건을 실제 테스트로 고정한다.

동일 seed와 입력 스트림으로 AI command와 simulation state를 반복 생성하여 결과가 동일한지 비교한다. PRNG와 AI source에는 `Math.random`·삼각함수 기반 임의성이 들어오지 못하도록 source guard가 있고, delta는 기준 timestep에 맞게 scale되며 paddle은 경기장 범위에 clamp된다.

각 step 뒤 이전 state와 중첩 값이 그대로인지 확인해 caller가 과거 snapshot을 안전하게 보관할 수 있음을 검증한다. 1,000 tick 실행의 digest는 짧은 단위 사례가 놓칠 누적 차이를 잡는 장기 regression 역할을 한다.

이 테스트는 순수 simulation을 대상으로 하므로 socket scheduling이나 serialization 차이는 포함하지 않는다. 그 공백은 fixed-step, replay fixture, GameHub integration test에서 다뤄진다.

#### 이 커밋이 보장하는 것

- 같은 초기 상태·seed·입력·delta는 같은 결과를 만든다.
- step과 AI 계산은 입력 state를 변경하지 않는다.
- 기본 역학 경계와 장기 실행 결과가 회귀로 고정된다.

#### 이 커밋이 보장하지 않는 것

- 다른 JS 엔진·런타임 버전 사이의 영구 bit-for-bit 호환성을 단독으로 보장하지 않는다.
- GameHub timer drift, network delivery, DB persistence는 대상이 아니다.

#### 전후 관계

`0888e119036d`이 wall-clock을 bounded fixed-step work로 변환하는 규칙을 검증하고, `37a7b2e4611b`이 versioned replay fixture로 장기 호환성을 명시한다.

#### 증거 성격

- deterministic simulation / immutability / long-run regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:4ef4beeb8611:end -->

## Commit `0888e119036d` — test(game): fixed-step 보정 범위 검증

- Full SHA: `0888e119036df23deeb961b9e26da305310da391`
- Subject: `test(game): fixed-step 보정 범위 검증`
- Importance: **B**
- Tags: SIMULATION, REALTIME, TEST
- Source-defined role: monotonic elapsed time을 50ms simulation step으로 변환할 때 remainder와 catch-up 상한을 결정적으로 검증한다.

### 정확한 조사 대상

- `apps/api/src/fixedStepScheduler.test.ts`의 fake clock/fake timer 구성을 확인한다.
- 정확히 50ms에서 한 step, 부족한 시간은 remainder로 남는지 확인한다.
- 긴 지연 뒤 최대 5 step/250ms만 처리하는 catch-up cap을 확인한다.
- clock이 뒤로 이동한 경우 음수 elapsed를 무시하는지, `stop()`이 timer를 정리하는지 확인한다.

### 학습자 복원

<!-- ANSWER:0888e119036d:begin -->
#### 역사적 복원

테스트는 실제 시간 대기 대신 제어 가능한 clock을 사용해 50ms 경계와 remainder를 정확히 만든다. 여러 짧은 elapsed가 누적되어 한 step이 되고, 긴 pause가 발생해도 한 callback에서 최대 5 step만 수행한다.

뒤로 간 clock은 음수 work를 만들지 않으며 stop 뒤에는 더 이상 callback이 실행되지 않는다. 따라서 wall-clock 이상과 event-loop stall이 simulation을 무한 catch-up 상태로 밀어 넣지 않는다.

#### 이 커밋이 보장하는 것

- 50ms fixed-step, remainder 보존, 250ms catch-up cap, timer cleanup을 고정한다.

#### 이 커밋이 보장하지 않는 것

- 운영 부하에서 실제 event-loop lag 수치를 측정하지 않는다.
- 여러 room의 스케줄러 topology는 별도 benchmark/통합 테스트 범위다.

#### 전후 관계

`4ef4beeb8611`의 pure deterministic step을 실제 시간과 연결할 때 생기는 보정 규칙을 보호한다.

#### 증거 성격

- fake-clock boundary testing
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:0888e119036d:end -->

## Commit `125aa113a01c` — test(game): snapshot replacement와 congestion 검증

- Full SHA: `125aa113a01c38605372c490637e1bfa4e86a009`
- Subject: `test(game): snapshot replacement와 congestion 검증`
- Importance: **A**
- Tags: REALTIME, PERF, RISK
- Source-defined role: latest-value snapshot buffer가 backlog를 누적하지 않고 실제 congestion에서만 drop·close하는 규칙을 fake socket과 clock으로 검증한다.

### 정확한 조사 대상

- `apps/api/src/latestSnapshotBuffer.test.ts`의 fake socket state와 `bufferedAmount` 조작을 확인한다.
- 전송 중 새 snapshot이 도착하면 오래된 pending 값 대신 최신 값 하나만 남기는지 확인한다.
- soft congestion에서 retry 간격 50ms와 5초 지속 기준을 확인한다.
- hard buffered amount 1MiB 이상에서 즉시 종료되는지 확인한다.
- `onDropped`와 close callback이 어떤 사건에서 몇 번 호출되는지 확인한다.

### 학습자 복원

<!-- ANSWER:125aa113a01c:begin -->
#### 역사적 복원

고주파 snapshot을 일반 queue에 쌓으면 느린 client 하나가 과거 상태를 끝없이 보유하고 메모리와 지연을 함께 키운다. 이 테스트는 snapshot을 reliable history가 아니라 replaceable latest value로 다루는 설계를 검증한다.

fake socket의 전송과 `bufferedAmount`를 제어해, 새 snapshot이 도착할 때 pending queue가 여러 개로 늘지 않고 가장 최신 값 하나로 대체되는지 확인한다. soft pressure에서는 50ms 뒤 다시 시도하되 5초 이상 지속되면 연결을 종료하고, hard pressure 1MiB 이상이면 즉시 종료한다.

drop callback과 close 횟수까지 검증해 observable side effect의 중복도 막는다. 하지만 이 초기 테스트와 구현은 outstanding send callback 자체를 congestion 신호로 해석했고, `d90f17fa765d`에서 그 가정이 잘못임이 드러난다.

#### 이 커밋이 보장하는 것

- snapshot backlog는 latest one으로 bounded된다.
- soft/hard congestion의 retry·termination 기준이 결정적으로 고정된다.
- drop/close side effect가 관찰 가능하고 중복되지 않는다.

#### 이 커밋이 보장하지 않는 것

- control event의 reliable delivery를 대신하지 않는다.
- send callback 지연이 실제 kernel/socket pressure와 동일하다는 초기 가정은 후속 fix에서 폐기된다.

#### 전후 관계

`d90f17fa765d` → `5cd54767858f`가 callback latency와 measurable `bufferedAmount` pressure를 분리해 이 invariant를 교정한다.

#### 증거 성격

- controlled backpressure primitive regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:125aa113a01c:end -->

## Commit `37a7b2e4611b` — test(game): versioned match replay fixture 추가

- Full SHA: `37a7b2e4611b02d077e98c0e136f2b97313ff807`
- Subject: `test(game): versioned match replay fixture 추가`
- Importance: **A**
- Tags: SIMULATION, REALTIME, TEST
- Source-defined role: 결정성 검사를 코드 내부 상수에서 versioned fixture 계약으로 승격해 장기 simulation 호환성을 검증한다.

### 정확한 조사 대상

- `apps/api/src/fixtures/replay-v1.json`의 protocolVersion, seed, timestep, tick count, input encoding을 확인한다.
- `replayFixture.test.ts`가 fixture를 parse하고 1,000 ticks를 정확히 재생하는지 확인한다.
- 최종 canonical state를 어떤 방식으로 SHA-256 digest로 만드는지 확인한다.
- fixture version과 현재 parser/simulation version 불일치가 어떻게 실패하는지 확인한다.

### 학습자 복원

<!-- ANSWER:37a7b2e4611b:begin -->
#### 역사적 복원

단위 테스트 안에 직접 작성된 반복 실행은 결정성을 보여 주지만 입력 데이터와 기대 결과가 호환성 자산으로 분리되어 있지 않다. 이 커밋은 초기 상태, seed, 50ms timestep, 1,000 tick 입력 스트림, 기대 SHA-256을 `replay-v1.json`에 명시한다.

테스트는 fixture version을 확인한 뒤 encoded input을 tick별로 적용하고 final simulation state의 canonical digest를 계산한다. 따라서 알고리즘·상수·입력 해석 중 어느 하나가 바뀌면 version-1 replay 결과 차이가 명시적으로 드러난다.

fixture는 재현 가능한 compatibility evidence이지만, 실제 경기 패킷 캡처나 모든 가능한 물리 상태를 포괄하지는 않는다.

#### 이 커밋이 보장하는 것

- version-1 replay 입력과 최종 simulation digest의 장기 호환성을 고정한다.
- 변경이 의도적이면 fixture/version 갱신이 필요하다는 review 경계를 만든다.

#### 이 커밋이 보장하지 않는 것

- 브라우저 렌더링, socket cadence, persistence 결과를 검증하지 않는다.
- 한 fixture가 전체 상태 공간을 증명하지 않는다.

#### 전후 관계

`4ef4beeb8611`의 장기 digest를 독립 versioned artifact로 발전시킨다.

#### 증거 성격

- versioned deterministic replay regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:37a7b2e4611b:end -->

## Commit `d90f17fa765d` — fix(game): callback 지연을 snapshot congestion으로 오판하지 않음

- Full SHA: `d90f17fa765d4e9548b8f52c850b7605698644eb`
- Subject: `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`
- Importance: **A**
- Tags: REALTIME, PERF, RISK
- Source-defined role: WebSocket `send` callback 지연을 transport congestion으로 간주하던 잘못된 가정을 제거하고 `bufferedAmount`만 pressure 근거로 사용한다.

### 정확한 조사 대상

- `apps/api/src/latestSnapshotBuffer.ts`에서 `sending` 상태와 callback gating이 제거된 diff를 확인한다.
- snapshot 전송 결정이 `bufferedAmount`의 soft/hard threshold에만 의존하도록 바뀌는지 확인한다.
- callback이 늦어도 socket buffer가 건강하면 다음 snapshot을 보낼 수 있는지 확인한다.
- latest-value replacement가 실제 pressure 상황에서만 적용되는지 확인한다.

### 학습자 복원

<!-- ANSWER:d90f17fa765d:begin -->
#### 역사적 복원

초기 구현은 `send` callback이 돌아오기 전의 상태를 in-flight congestion으로 보았다. 그러나 callback scheduling은 event-loop 지연이나 라이브러리 callback 시점의 영향을 받으며, 실제 socket send buffer가 비어 있어도 늦을 수 있다. 이 오판은 정상 client에게 snapshot을 불필요하게 drop하고 5초 후 연결을 종료할 수 있었다.

수정은 `sending` 상태를 제거하고 `bufferedAmount`를 유일한 transport pressure 신호로 사용한다. buffer가 정상 범위라면 callback이 아직 돌아오지 않았어도 새 snapshot을 보낸다. soft/hard pressure와 latest replacement 규칙은 유지되지만 measurable backlog가 있을 때만 작동한다.

교정된 invariant는 callback completion과 network congestion을 동일시하지 않는 것이다. callback은 notification일 뿐 flow-control ownership을 갖지 않는다.

#### 이 커밋이 보장하는 것

- 정상 `bufferedAmount`에서 callback 지연만으로 drop·close가 발생하지 않는다.
- 실제 measurable transport backlog에서만 latest-value/congestion 정책이 작동한다.

#### 이 커밋이 보장하지 않는 것

- `bufferedAmount`가 모든 네트워크 상태를 완벽히 예측한다는 보장은 아니다.
- OS·프록시 바깥의 지연 측정은 포함하지 않는다.

#### 전후 관계

`125aa113a01c`의 초기 가정을 교정하고 `5cd54767858f`가 delayed callback과 실제 pressure를 분리해 회귀를 막는다.

#### 증거 성격

- backpressure root-cause correction
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:d90f17fa765d:end -->

## Commit `5cd54767858f` — test(game): callback 지연과 실제 congestion 구분

- Full SHA: `5cd54767858f60f0f10b9c86e680a9922c361fac`
- Subject: `test(game): callback 지연과 실제 congestion 구분`
- Importance: **A**
- Tags: REALTIME, PERF, TEST
- Source-defined role: 지연된 send callback과 `bufferedAmount` pressure를 독립적으로 제어해 교정된 snapshot delivery invariant를 검증한다.

### 정확한 조사 대상

- `latestSnapshotBuffer.test.ts`의 fake socket callback queue를 확인한다.
- callback 세 개를 보류한 상태에서도 healthy `bufferedAmount`이면 세 snapshot이 모두 전송되는지 확인한다.
- soft pressure를 설정했을 때만 pending 최신값 replacement와 `onDropped`가 발생하는지 확인한다.
- callback을 나중에 flush해도 중복 전송·drop·close가 없는지 확인한다.

### 학습자 복원

<!-- ANSWER:5cd54767858f:begin -->
#### 역사적 복원

fake socket은 send callback을 즉시 실행하지 않고 queue에 보관한다. 이 상태에서 `bufferedAmount`를 0으로 유지한 채 여러 snapshot을 보내 모두 전송되었음을 확인한다. 따라서 callback delay만으로 congestion branch가 실행되지 않는다.

별도 사례에서는 `bufferedAmount`를 soft threshold 위로 올려 최신 snapshot replacement와 drop 관찰을 요구한다. 두 원인을 독립적으로 조작하기 때문에 수정이 단순히 테스트 타이밍을 느슨하게 만든 것이 아니라 판단 신호를 바꿨음을 증명한다.

#### 이 커밋이 보장하는 것

- callback delay와 실제 buffered pressure가 별도 상태로 취급됨을 결정적으로 증명한다.
- healthy transport에서 `onDropped`가 호출되지 않는다.

#### 이 커밋이 보장하지 않는 것

- 실제 인터넷 경로의 latency나 browser별 `bufferedAmount` 구현 차이는 측정하지 않는다.

#### 전후 관계

`d90f17fa765d`의 corrected invariant를 deterministic regression으로 닫는다.

#### 증거 성격

- root-cause-specific regression test
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:5cd54767858f:end -->


## Thread-level invariant evolution

다음 표에서 불변식이 도입·확장·교정·검증된 SHA를 구분합니다.

<!-- ANSWER:thread-03-invariants:begin -->
### 복원 결과

| SHA | 변화 | 불변식 |
| --- | --- | --- |
| `4ef4beeb8611` | 도입 | 동일 state·seed·input·delta는 동일 결과를 만들며 입력 state를 mutate하지 않는다. |
| `0888e119036d` | 시간 경계 | monotonic elapsed는 50ms step과 remainder로 변환되고 한 callback catch-up은 250ms로 제한된다. |
| `125aa113a01c` | 전달 상한 | snapshot backlog는 latest one으로 bounded되고 soft/hard congestion에 retry/close 규칙이 있다. |
| `37a7b2e4611b` | 호환성 고정 | version-1 1,000-tick replay와 최종 SHA-256이 독립 fixture가 된다. |
| `d90f17fa765d` | 가정 교정 | outstanding callback은 congestion 증거가 아니며 `bufferedAmount`만 pressure 판단에 사용된다. |
| `5cd54767858f` | 회귀 보호 | callback delay와 실제 pressure가 독립적으로 조작되어 drop 여부가 검증된다. |

각 변화는 이전 커밋의 최종 상태를 다음 커밋이 단순 반복한 것이 아니라, 새 경계를 도입·확장·교정하거나 regression으로 보호한 시점을 나타냅니다.
<!-- ANSWER:thread-03-invariants:end -->

## Failure → Fix → Test 관계

구현 추가를 독립 기능으로 나열하지 말고, 이전 가정과 실제 실패 위험, 수정된 결정, 회귀 증거를 연결합니다.

<!-- ANSWER:thread-03-failure-relations:begin -->
### 복원 결과

| Failure / 이전 가정 | Fix / 결정 | Verification |
| --- | --- | --- |
| simulation이 전역 난수·mutable state에 의존할 위험 | seeded pure transition | `4ef4beeb8611`의 repeatability/immutability/digest |
| event-loop stall을 무제한 catch-up | 50ms accumulator + 5-step cap | `0888e119036d` fake-clock boundary |
| snapshot FIFO backlog | latest-value + soft/hard pressure | `125aa113a01c` controlled socket test |
| send callback 지연을 congestion으로 오판 | `d90f17fa765d` — `sending` 제거, measurable pressure만 사용 | `5cd54767858f` — delayed callbacks vs buffered pressure |
<!-- ANSWER:thread-03-failure-relations:end -->

## Ownership·state·responsibility 변화

자원과 상태를 누가 만들고, 보유하고, 이전하고, 정리하는지 커밋 순서에 따라 정리합니다.

<!-- ANSWER:thread-03-ownership:begin -->
### 복원 결과

| 대상 | 소유자 | 책임·수명 |
| --- | --- | --- |
| Game mechanics | `PongSimulation` | state transition, collision, score, phase와 결과 결정성을 소유한다. |
| AI randomness | seeded integer PRNG / `PongAi` | 같은 seed·rating에서 같은 command stream을 소유한다. |
| Wall-clock conversion | fixed-step scheduler | elapsed, remainder, catch-up cap, timer cleanup을 소유한다. |
| Compatibility evidence | `replay-v1.json` + replay test | versioned input stream과 기대 digest를 소유한다. |
| Snapshot pressure | `LatestSnapshotBuffer` | replaceable latest state, retry, drop/close와 measurable backlog를 소유한다. |
| Send callback | notification only | flow-control authority를 갖지 않는다. |
<!-- ANSWER:thread-03-ownership:end -->

## 최종 Thread 상태

최종 HEAD를 과거 커밋에 투영하지 말고, 위 SHA 순서에서 도달한 보장을 요약합니다.

<!-- ANSWER:thread-03-final-state:begin -->
### 최종 상태

- simulation은 transport·clock에서 분리된 pure deterministic transition으로 검증된다.
- wall-clock은 bounded 50ms work로 변환되며 장기 결과는 versioned replay digest로 보호된다.
- snapshot 전달은 latest-value로 bounded되고 실제 `bufferedAmount` pressure에서만 drop/close한다.
- callback scheduling 지연은 network congestion으로 취급되지 않는다.
<!-- ANSWER:thread-03-final-state:end -->

## 최종 architecture / execution flow

실제 caller→boundary→state transition→cleanup 흐름을 순서대로 작성합니다.

<!-- ANSWER:thread-03-flow:begin -->
### 최종 실행 흐름

1. monotonic clock에서 elapsed 수집
2. 50ms fixed step으로 최대 5회 simulation 실행
3. seeded AI/input으로 immutable next state 생성
4. versioned snapshot projection
5. `bufferedAmount`가 정상일 때 즉시 전송
6. soft pressure면 latest pending 하나로 대체·retry
7. 지속 soft/hard pressure면 관찰 후 연결 종료
<!-- ANSWER:thread-03-flow:end -->

## 학습 완료 확인

<!-- ANSWER:thread-03-checks:begin -->
### 완료 확인

- [x] 동일 seed만으로 결정성이 충분하지 않은 이유를 state mutation과 delta 관점에서 설명할 수 있다.
- [x] replay SHA-256이 protocol/network 전체를 증명하지 않는 이유를 설명할 수 있다.
- [x] latest snapshot drop이 데이터 손실 버그가 아니라 의도된 delivery policy인 조건을 설명할 수 있다.
- [x] `d90...` 이전/이후 pressure 판단식을 비교할 수 있다.

체크는 repository code inspection을 통해 설명이 문서에 작성되었음을 뜻합니다. 실제 repository test suite 실행 완료를 뜻하지 않습니다.
<!-- ANSWER:thread-03-checks:end -->

## 실행 증거 기록

<!-- ANSWER:thread-03-execution:begin -->
- Historical inspection: 각 참조 SHA의 GitHub commit diff와 변경 파일을 개별 조회했습니다.
- Repository test execution: 실행하지 않았습니다. 로컬 환경에서 지정 branch의 전체 checkout을 materialize하지 못해 pnpm, PostgreSQL, Playwright, k6, Toxiproxy 명령 결과를 만들 수 없었습니다.
- Runtime claims: 기록하지 않았습니다. 위의 `증거 성격`은 test 구현과 production code path에 대한 정적·역사적 검사입니다.
- Artifact validation: scaffold/completed 대응, fixed metadata, answer completion, SHA uniqueness, Markdown fence, ZIP 구조는 로컬 검증 스크립트로 검사했습니다.
<!-- ANSWER:thread-03-execution:end -->
===== END FILE: 03-deterministic-simulation-timing-and-snapshot-delivery-verification.md =====

===== BEGIN FILE: 04-gamehub-lifecycle-reconnect-matchmaking-and-finalization-recovery.md =====
# Development Thread 04: GameHub lifecycle·재연결·매칭·결과 저장 복구 검증

English title: **GameHub Lifecycle, Reconnect, Matchmaking, and Finalization Recovery**

## Thread 목표

RoomSession, connection replacement, drain, Matchmaker reservation, room rollback, persistence retry가 하나의 GameHub ownership story로 결합되는 과정을 복원한다.

## 핵심 질문

- connection replacement와 실제 disconnect는 왜 서로 다른 상태 전이인가?
- room side, reconnect timer, queue/reservation, finalization retry의 소유자는 각각 누구인가?
- queued 상태와 matched reservation을 별도 상태로 두지 않으면 어떤 누수가 생기는가?
- room publish 도중 실패한 경우 어떤 자원을 역순으로 rollback해야 다시 matchable한가?
- simulation 종료와 durable finalization 완료를 왜 분리하며, 외부 `game.finished`는 언제 보내야 하는가?
- drain은 신규 작업 거부와 기존 room 기다리기를 어떻게 동시에 수행하는가?

## 완료 기준

- 순수 RoomSession 전이와 GameHub integration evidence를 구분한다.
- replacement/reconnect/expiry/forfeit의 timer·socket·side ownership을 설명한다.
- Matchmaker queue/reservation release 경로를 성공·실패·포기·drain별로 추적한다.
- transient finalization failure에서 room을 유지해야 하는 이유와 stable key/backoff를 설명한다.
- shutdown single-entry와 bounded wait를 설명한다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags |
| --- | --- | --- | --- | --- |
| 1 | `4026c3bf72ad` | `test(game): 게임 방 상태 전이 검증` | B | REALTIME, OPERATIONS, TEST |
| 2 | `113e39acc85c` | `test(game): reconnect 복구 동작 검증` | A | REALTIME, PERSISTENCE, RISK |
| 3 | `9d05f47e7f4b` | `test(ops): GameHub drain과 graceful shutdown 검증` | A | REALTIME, OPERATIONS, RISK |
| 4 | `fc7da13e935d` | `test(game): matchmaking 규칙 검증` | A | REALTIME, TEST |
| 5 | `112228db8878` | `test(game): matchmaking lifecycle 검증` | A | REALTIME, RISK, TEST |
| 6 | `e939a50948b2` | `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지` | A | REALTIME, RISK |
| 7 | `8f5b2e86f69b` | `test(game): 일시적인 경기 결과 저장 실패 복구 검증` | A | REALTIME, OBSERVABILITY, RISK |

> Commit 순서·subject·importance·tags는 Phase 1 감사 후 동결된 정보입니다. Phase 2에서는 변경하지 않습니다.

## Commit `4026c3bf72ad` — test(game): 게임 방 상태 전이 검증

- Full SHA: `4026c3bf72adfd97868a9f8296c8899ce4d0ff44`
- Subject: `test(game): 게임 방 상태 전이 검증`
- Importance: **B**
- Tags: REALTIME, OPERATIONS, TEST
- Source-defined role: 순수 `RoomSession` 상태 기계의 준비·일시정지·재연결·만료·종료 전이를 결정적으로 고정한다.

### 정확한 조사 대상

- `apps/api/src/roomSession.test.ts`에서 initial/ready/playing/paused/reconnecting/finished 전이를 확인한다.
- 양쪽 ready 전에는 playing이 되지 않는지 확인한다.
- pause 이전 상태를 기억하고 resume이 올바른 상태로 돌아가는지 확인한다.
- 한쪽 disconnect의 15초 deadline, in-window reconnect, expiry forfeit, 양쪽 부재 시 winner 없음, finish idempotency를 확인한다.

### 학습자 복원

<!-- ANSWER:4026c3bf72ad:begin -->
#### 역사적 복원

테스트는 socket이나 repository 없이 `RoomSession`만 구동하여 legal transition을 고정한다. 두 player가 모두 ready여야 playing이 되고, pause/resume은 이전 진행 상태와 일치해야 한다.

disconnect는 즉시 종료가 아니라 15초 reconnect 예약으로 바뀐다. 한쪽만 만료되면 상대가 forfeit winner가 되지만, 양쪽 모두 부재하면 persisted winner를 만들지 않는다. 동일 expiry/finish 호출은 두 번째 terminal effect를 만들지 않는다.

#### 이 커밋이 보장하는 것

- room lifecycle의 합법 전이와 15초 복구 경계가 고정된다.
- terminal 전이는 idempotent이고 양쪽 부재 시 임의 승자를 만들지 않는다.

#### 이 커밋이 보장하지 않는 것

- 실제 socket 교체, timer ownership, DB finalization은 검증하지 않는다.

#### 전후 관계

`113e39acc85c`가 이 순수 상태 기계를 GameHub의 실제 connection replacement와 persistence 경계에서 검증한다.

#### 증거 성격

- pure state-machine boundary testing
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:4026c3bf72ad:end -->

## Commit `113e39acc85c` — test(game): reconnect 복구 동작 검증

- Full SHA: `113e39acc85c9779c29eb62339bafa884e95b7e5`
- Subject: `test(game): reconnect 복구 동작 검증`
- Importance: **A**
- Tags: REALTIME, PERSISTENCE, RISK
- Source-defined role: GameHub에서 동일 사용자 연결 교체와 실제 disconnect 복구가 서로 다른 lifecycle을 갖고 한 번만 finalization되는지 검증한다.

### 정확한 조사 대상

- `apps/api/src/gameHub.reconnect.test.ts`의 fake socket, fake clock, repository spy를 확인한다.
- 동일 사용자 새 연결이 old socket을 code 4001로 닫고 room side를 이전하는지 확인한다.
- replacement가 reconnect timer나 forfeit를 만들지 않고 최신 snapshot을 새 socket에 보내는지 확인한다.
- 실제 disconnect만 15초 예약을 만들며 in-window reconnect가 같은 room/side로 복구되는지 확인한다.
- deadline expiry 뒤 `finalizeMatch`와 game-finished effect가 정확히 한 번인지 확인한다.

### 학습자 복원

<!-- ANSWER:113e39acc85c:begin -->
#### 역사적 복원

같은 identity의 새 socket은 장애가 아니라 ownership handoff다. 테스트는 기존 socket을 4001 `connection replaced`로 닫고 새 socket이 기존 room side와 최신 snapshot을 받는지 확인한다. old socket의 이후 message는 stale-client guard로 무시되고 reconnect deadline은 생기지 않는다.

반대로 실제 disconnect는 side를 비운 채 15초 timer를 room이 소유한다. 같은 사용자가 새 ticket으로 돌아오면 matchmaking을 다시 실행하지 않고 reserved room/side로 복구된다. deadline이 지나면 state machine이 forfeit 결과를 만들고 repository finalization을 한 번만 호출한다.

이 테스트는 one user → one authoritative transport, one room → one reserved side, one terminal result라는 세 invariant의 상호작용을 검증한다.

#### 이 커밋이 보장하는 것

- connection replacement가 room을 보존하며 duplicate authority를 만들지 않는다.
- 실제 disconnect만 bounded reservation을 만들고 같은 identity가 같은 room으로 복구된다.
- expiry가 중복 persisted result를 만들지 않는다.

#### 이 커밋이 보장하지 않는 것

- 브라우저 retry backoff나 network proxy 장애를 직접 검증하지 않는다.
- repository transaction 내부 원자성은 별도 DB 테스트 범위다.

#### 전후 관계

`4026c3bf72ad`의 pure transition을 실제 GameHub ownership에 연결하며, `9d05f47e7f4b`의 drain과 `e939a50948b2`의 finalization retry가 같은 room lifecycle을 확장한다.

#### 증거 성격

- deterministic GameHub integration regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:113e39acc85c:end -->

## Commit `9d05f47e7f4b` — test(ops): GameHub drain과 graceful shutdown 검증

- Full SHA: `9d05f47e7f4bcbf12e8ff9ac36d446acd865a691`
- Subject: `test(ops): GameHub drain과 graceful shutdown 검증`
- Importance: **A**
- Tags: REALTIME, OPERATIONS, RISK
- Source-defined role: drain 시작 즉시 신규 작업을 차단하면서 기존 room은 제한된 예산 안에서 마치고 process shutdown은 단 한 번만 실행되는지 검증한다.

### 정확한 조사 대상

- `apps/api/src/gameHub.drain.test.ts`에서 queued waiter/timer 정리와 신규 queue/tournament 거부를 확인한다.
- 기존 active room이 최대 60초 동안 completion을 기다리는지 fake clock으로 확인한다.
- readiness endpoint가 drain 시작 즉시 503으로 바뀌는지 health test를 확인한다.
- `gracefulShutdown.test.ts`에서 SIGTERM/SIGINT 반복이 한 shutdown promise만 실행하는지 확인한다.
- drain/close 실패가 report되지만 두 번째 shutdown을 시작하지 않는지 확인한다.

### 학습자 복원

<!-- ANSWER:9d05f47e7f4b:begin -->
#### 역사적 복원

즉시 process exit는 owned room, retry timer, DB 작업을 끊을 수 있고, 반대로 readiness를 유지한 채 오래 기다리면 새로운 트래픽이 계속 들어온다. 테스트는 이 두 경계를 분리한다.

drain 전이는 GameHub를 accepting에서 draining으로 바꾸고 waiting queue와 AI fallback timer를 제거한다. 신규 queue/tournament work는 즉시 거부되며 readiness는 503이 된다. 이미 소유한 room은 완료를 기다리되 60초 예산을 넘기지 않는다.

signal handler는 여러 SIGTERM/SIGINT가 와도 하나의 shutdown promise만 소유한다. drain, server close, repository close 중 오류는 관측되지만 재진입이나 두 번째 cleanup 실행으로 이어지지 않는다.

이 검증은 process lifecycle과 domain lifecycle의 ownership을 맞춘다. 다만 container가 실제로 60초보다 긴 stop grace를 주는지는 다른 category의 deployment contract가 맡는다.

#### 이 커밋이 보장하는 것

- drain은 신규 작업 거부와 기존 작업 bounded completion을 동시에 보장한다.
- readiness가 lifecycle 상태와 즉시 정렬된다.
- shutdown cleanup은 single-entry이고 실패도 중복 실행되지 않는다.

#### 이 커밋이 보장하지 않는 것

- 운영 container signal delivery와 실제 60초 wall-clock 실행을 이 테스트에서 수행하지 않는다.
- 모든 외부 dependency가 예산 안에 종료된다고 보장하지 않는다.

#### 전후 관계

RoomSession/GameHub의 장기 소유 작업을 process termination에 연결한다. category 09의 container grace/CI wiring은 의도적으로 제외한다.

#### 증거 성격

- fake-time lifecycle and shutdown integration testing
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:9d05f47e7f4b:end -->

## Commit `fc7da13e935d` — test(game): matchmaking 규칙 검증

- Full SHA: `fc7da13e935d4206b743e1eaf706ea9c3b942578`
- Subject: `test(game): matchmaking 규칙 검증`
- Importance: **A**
- Tags: REALTIME, TEST
- Source-defined role: socket과 분리된 `Matchmaker` 상태 기계의 rating 선택·pool 격리·AI fallback·reservation 규칙을 제어 가능한 clock으로 검증한다.

### 정확한 조사 대상

- `apps/api/src/matchmaker.test.ts`에서 compatible candidate 중 rating 차가 가장 작은 pair를 고르는지 확인한다.
- 허용 rating gap 밖 사용자는 queue에 남는지 확인한다.
- guest와 registered pool이 절대 교차하지 않는지 확인한다.
- AI fallback이 정확히 6초 전에는 unavailable이고 deadline 이후 claim되는지 확인한다.
- duplicate join, `leaveQueue`, `release`가 queued와 reserved 상태에서 어떻게 다른지 확인한다.

### 학습자 복원

<!-- ANSWER:fc7da13e935d:begin -->
#### 역사적 복원

이 테스트는 매칭 규칙을 socket callback이나 GameHub map에서 떼어 내어 pure state machine으로 검증한다. compatible 범위 안에서는 rating 차가 가장 작은 상대를 선택하고 범위 밖 사용자는 기다린다. identity kind가 다른 guest/registered는 rating이 맞아도 pair가 되지 않는다.

제어 clock으로 6초 전후를 나눠 AI fallback이 정확한 deadline 뒤에만 claim되는지 확인한다. queue 취소는 아직 배정되지 않은 상태만 제거하고, match reservation은 별도 `release`가 필요하다. duplicate join은 queued/reserved identity를 다시 삽입하지 않는다.

핵심은 availability가 단순 배열 존재 여부가 아니라 queued 또는 reserved라는 명시적 소유 상태라는 점이다.

#### 이 커밋이 보장하는 것

- closest compatible rating, identity-pool isolation, exact fallback deadline을 고정한다.
- queued와 reserved lifecycle 및 중복 가입 의미를 구분한다.

#### 이 커밋이 보장하지 않는 것

- socket disconnect·room publish·rollback에서 release가 실제 호출되는지는 검증하지 않는다.

#### 전후 관계

`112228db8878`이 Matchmaker 규칙을 GameHub room creation/finalization/rollback lifecycle과 결합해 검증한다.

#### 증거 성격

- deterministic matchmaking state-machine testing
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:fc7da13e935d:end -->

## Commit `112228db8878` — test(game): matchmaking lifecycle 검증

- Full SHA: `112228db8878078fae6e9322e258a646e9642e7a`
- Subject: `test(game): matchmaking lifecycle 검증`
- Importance: **A**
- Tags: REALTIME, RISK, TEST
- Source-defined role: GameHub가 Matchmaker의 queue·reservation을 room 성공·종료·포기·부분 실패의 모든 경로에서 정확히 해제하는지 검증한다.

### 정확한 조사 대상

- `apps/api/src/gameHub.matchmaking.test.ts`의 rating gap과 successful room 사례를 확인한다.
- finalized forfeit와 empty-room abandonment 후 두 사용자가 다시 매칭 가능한지 확인한다.
- room construction 중 observer `roomCreated` 실패를 주입하는 방식을 확인한다.
- 실패 뒤 scheduler entry, room map, client room assignment, reservation이 모두 rollback되는지 확인한다.
- cleanup 후 같은 identity가 duplicate로 남지 않고 재가입 가능한지 확인한다.

### 학습자 복원

<!-- ANSWER:112228db8878:begin -->
#### 역사적 복원

Matchmaker 자체가 올바르더라도 GameHub가 room을 반쯤 공개한 뒤 실패하거나 terminal cleanup에서 reservation을 놓치면 사용자는 영구적으로 매칭 불가능해진다. 이 테스트는 integration ownership 누수를 겨냥한다.

정상 사례에서는 rating window에 따라 pair되고 room 종료/forfeit 뒤 reservation이 풀린다. 양쪽 connection이 사라진 room도 persisted winner 없이 정리되며 identity는 다시 queue에 들어갈 수 있다.

실패 주입 사례는 room state, scheduler registration, client assignment를 만든 뒤 observer `roomCreated`가 예외를 던지게 한다. GameHub는 방, timer, scheduler, client room 참조, Matchmaker reservation을 모두 역순으로 되돌려야 한다. 이후 동일 사용자들이 다시 매칭되는 것으로 cleanup의 완전성을 관찰한다.

#### 이 커밋이 보장하는 것

- match reservation은 성공한 room이 소유하고 모든 terminal/rollback 경로에서 해제된다.
- 부분 room publish 실패가 stale room·scheduler·client assignment를 남기지 않는다.
- 실패 뒤 사용자는 다시 matchable하다.

#### 이 커밋이 보장하지 않는 것

- DB tournament start 실패와 match finalization retry까지 모두 포함하지 않는다.
- 프로세스 강제 종료에서 memory cleanup을 관찰하지 않는다.

#### 전후 관계

`fc7da13e935d`의 순수 reservation 규칙을 GameHub integration에서 보호하며, `e939a50948b2`는 persistence 실패 동안 reservation을 너무 일찍 해제하지 않는 반대 경계를 추가한다.

#### 증거 성격

- rollback/failure-injection integration regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:112228db8878:end -->

## Commit `e939a50948b2` — fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지

- Full SHA: `e939a50948b2934e5087ccc8b9169faf91c968e1`
- Subject: `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지`
- Importance: **A**
- Tags: REALTIME, RISK
- Source-defined role: 일시적 persistence 실패 시 finished room과 reservation을 보존하고 안정된 idempotency key로 재시도한 뒤에만 외부 종료 효과를 노출한다.

### 정확한 조사 대상

- `apps/api/src/gameHub.ts`의 room-owned `finalizationRetryTimer`와 cleanup 경로를 확인한다.
- 재시도 delay가 250ms에서 시작해 지수 증가하고 5초로 cap되는지 확인한다.
- `room:<roomId>:finished`와 같은 result key가 재시도마다 동일한지 확인한다.
- 실패 중 room, reservation, drain pending 상태가 유지되고 `game.finished`가 아직 broadcast되지 않는지 확인한다.
- 성공·abandon·close·remove 시 retry timer가 한 번만 정리되는지 확인한다.

### 학습자 복원

<!-- ANSWER:e939a50948b2:begin -->
#### 역사적 복원

이전 finalization 경로는 simulation이 끝난 뒤 repository 저장이 실패하면 room cleanup과 reservation release가 진행될 수 있었다. 그러면 결과를 다시 저장할 소유자가 사라지고 client는 종료를 보았지만 durable match가 없는 split state가 생긴다.

수정은 terminal simulation state와 durable finalization 완료를 구분한다. 저장 실패 시 room은 map과 Matchmaker reservation을 계속 소유하고 `finalizationRetryTimer`를 예약한다. retry는 250ms부터 지수 증가해 최대 5초이며 모든 시도에 안정된 `room:<id>:finished` idempotency key를 사용한다.

`game.finished`, room removal, reservation release 같은 observable terminal effect는 repository success 뒤에만 일어난다. drain도 이 room을 pending work로 본다. 성공·abandonment·shutdown cleanup은 timer를 취소해 orphan retry가 남지 않게 한다.

보정된 invariant는 'simulation finished'가 곧 'match finalized'가 아니라는 점과, 재시도 중 결과 ownership이 room에 남아 있어야 한다는 점이다.

#### 이 커밋이 보장하는 것

- 일시적 저장 실패가 room ownership과 idempotency context를 잃지 않는다.
- 외부 종료 통지는 durable success 이후 한 번만 발생한다.
- retry work와 reservation의 cleanup owner가 room으로 명시된다.

#### 이 커밋이 보장하지 않는 것

- 영구 장애가 자동으로 해결된다고 보장하지 않는다.
- 재시도 횟수에 최종 상한은 없으며 process drain 예산이 종료 경계를 제공한다.
- repository 내부 transaction 원자성은 별도 persistence invariant다.

#### 전후 관계

`8f5b2e86f69b`이 첫 finalization 실패 후 성공을 fake time으로 재현하고, 동일 key·비조기 cleanup·observer 결과를 확인한다.

#### 증거 성격

- persistence-failure recovery and ownership fix
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:e939a50948b2:end -->

## Commit `8f5b2e86f69b` — test(game): 일시적인 경기 결과 저장 실패 복구 검증

- Full SHA: `8f5b2e86f69bfa9faad1d942d097e2c6f9961df5`
- Subject: `test(game): 일시적인 경기 결과 저장 실패 복구 검증`
- Importance: **A**
- Tags: REALTIME, OBSERVABILITY, RISK
- Source-defined role: 첫 `finalizeMatch` 실패와 다음 성공을 결정적으로 주입해 room-owned retry와 외부 효과 순서를 검증한다.

### 정확한 조사 대상

- `apps/api/src/gameHub.finalization.test.ts`의 fake timer와 repository mock call sequence를 확인한다.
- 첫 호출이 reject하고 두 번째 호출이 success하도록 failure injection을 구성한 방식을 확인한다.
- 두 호출의 result key가 동일한지, retry delay가 250ms부터 시작하는지 확인한다.
- 첫 실패 후 room·reservation·drain pending이 남고 `game.finished`가 전송되지 않는지 확인한다.
- 성공 후 한 번만 broadcast/remove/release되고 observer가 failure 후 success를 기록하는지 확인한다.

### 학습자 복원

<!-- ANSWER:8f5b2e86f69b:begin -->
#### 역사적 복원

테스트 repository는 첫 `finalizeMatch`만 의도적으로 reject하고 두 번째에 성공 결과를 반환한다. fake timer를 진행해 실제 backoff callback을 실행하며 두 call의 idempotency key가 동일함을 비교한다.

첫 실패 시 client는 아직 `game.finished`를 받지 않고 room과 reservation은 유지된다. observer에는 persistence failure가 기록되고 drain은 room을 pending으로 센다. retry 성공 후에만 persisted result event, room removal, reservation release가 한 번 발생하며 observer도 success를 기록한다.

이 테스트는 '재시도 함수가 호출된다'보다 강하게, 실패 전후 상태와 observable effect 순서를 함께 고정한다.

#### 이 커밋이 보장하는 것

- transient failure 후 안정된 key로 복구하며 duplicate terminal effect가 없음을 증명한다.
- failure/success observability와 drain accounting이 실제 상태와 정렬된다.

#### 이 커밋이 보장하지 않는 것

- 실제 PostgreSQL outage나 process restart 뒤 재개를 검증하지 않는다.
- 무한 장애에서의 운영 정책은 load/fault 및 shutdown 범위다.

#### 전후 관계

`e939a50948b2`의 Failure → Retryable ownership → Success 흐름에 대한 deterministic regression이다.

#### 증거 성격

- deterministic transient-failure injection
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:8f5b2e86f69b:end -->


## Thread-level invariant evolution

다음 표에서 불변식이 도입·확장·교정·검증된 SHA를 구분합니다.

<!-- ANSWER:thread-04-invariants:begin -->
### 복원 결과

| SHA | 변화 | 불변식 |
| --- | --- | --- |
| `4026c3bf72ad` | 상태 기계 검증 | 준비·pause·15초 reconnect·expiry·finish의 legal transition이 순수 모델로 고정된다. |
| `113e39acc85c` | connection ownership | 동일 사용자 replacement는 room을 보존하고 실제 disconnect만 reservation/forfeit timer를 만든다. |
| `9d05f47e7f4b` | process lifecycle | drain은 신규 work를 즉시 거부하고 기존 room을 60초 내 기다리며 shutdown은 한 번만 실행된다. |
| `fc7da13e935d` | 매칭 규칙 | closest rating, guest/registered 격리, 6초 fallback, queued/reserved 의미가 고정된다. |
| `112228db8878` | integration cleanup | finalization·abandonment·partial room publish 실패 뒤 reservation과 room 자원이 완전히 해제된다. |
| `e939a50948b2` | persistence ownership | 저장 실패 중에는 room/reservation/retry ownership을 유지하고 success 전 외부 종료를 노출하지 않는다. |
| `8f5b2e86f69b` | 복구 회귀 | 첫 실패→stable-key retry→성공의 상태·observer·drain 효과가 결정적으로 검증된다. |

각 변화는 이전 커밋의 최종 상태를 다음 커밋이 단순 반복한 것이 아니라, 새 경계를 도입·확장·교정하거나 regression으로 보호한 시점을 나타냅니다.
<!-- ANSWER:thread-04-invariants:end -->

## Failure → Fix → Test 관계

구현 추가를 독립 기능으로 나열하지 말고, 이전 가정과 실제 실패 위험, 수정된 결정, 회귀 증거를 연결합니다.

<!-- ANSWER:thread-04-failure-relations:begin -->
### 복원 결과

| Failure / 이전 가정 | Fix / 결정 | Verification |
| --- | --- | --- |
| socket 교체를 disconnect로 처리해 불필요한 forfeit | user-indexed atomic handoff + reserved side recovery | `113e39acc85c` |
| GameHub와 queue가 중복 availability를 소유해 stale reservation | Matchmaker queued/reserved state와 중앙 release | `fc7da13e935d` + `112228db8878` |
| room 생성 중 observer/assignment 실패로 partial publication | scheduler/room/client/timer/reservation 역순 rollback | `112228db8878` failure injection |
| finalize 실패 후 room을 제거해 결과 owner 상실 | `e939a50948b2` room-owned retry + stable key | `8f5b2e86f69b` transient failure regression |
| signal 반복·무기한 종료 대기 | single-entry shutdown + 60초 drain | `9d05f47e7f4b` |
<!-- ANSWER:thread-04-failure-relations:end -->

## Ownership·state·responsibility 변화

자원과 상태를 누가 만들고, 보유하고, 이전하고, 정리하는지 커밋 순서에 따라 정리합니다.

<!-- ANSWER:thread-04-ownership:begin -->
### 복원 결과

| 대상 | 소유자 | 책임·수명 |
| --- | --- | --- |
| Legal room transition | `RoomSession` | ready/play/pause/reconnect/finish 상태와 deadline 결과를 소유한다. |
| Current socket authority | GameHub user-indexed client map | 동일 identity의 현재 transport 한 개와 replacement handoff를 소유한다. |
| Room side/reconnect timer | GameHub room | disconnect 중 side reservation과 15초 timer cleanup을 소유한다. |
| Queued/reserved identity | `Matchmaker` | availability, pair, fallback, release를 소유한다. |
| Transport-specific waiter/timer | GameHub matchmaking metadata | socket/client와 AI timeout callback을 소유하되 reservation truth는 소유하지 않는다. |
| Finalization retry | finished GameHub room | stable result key, backoff timer, pending drain 상태를 durable success까지 소유한다. |
| Process shutdown | single shutdown promise | drain→server close→repository close의 한 번 실행을 소유한다. |
<!-- ANSWER:thread-04-ownership:end -->

## 최종 Thread 상태

최종 HEAD를 과거 커밋에 투영하지 말고, 위 SHA 순서에서 도달한 보장을 요약합니다.

<!-- ANSWER:thread-04-final-state:begin -->
### 최종 상태

- 한 사용자에는 한 authoritative realtime connection만 있고 replacement는 room을 잃지 않는다.
- 실제 disconnect는 15초 reserved-side recovery를 만들며 expiry만 terminal forfeit가 된다.
- Matchmaker가 queued/reserved identity의 단일 owner이고 모든 room terminal/rollback 경로가 release에 수렴한다.
- durable finalization 실패 시 room은 stable key로 retry하며 성공 전에는 client 종료·room 제거·reservation release가 일어나지 않는다.
- drain은 신규 작업을 차단하고 owned room/retry를 bounded하게 기다린 뒤 single-entry shutdown으로 정리한다.
<!-- ANSWER:thread-04-final-state:end -->

## 최종 architecture / execution flow

실제 caller→boundary→state transition→cleanup 흐름을 순서대로 작성합니다.

<!-- ANSWER:thread-04-flow:begin -->
### 최종 실행 흐름

1. client가 Matchmaker queue 또는 tournament room 요청
2. Matchmaker가 pair/reservation 또는 6초 fallback claim
3. GameHub가 room·scheduler·client assignment를 publish; 실패 시 역순 rollback
4. RoomSession이 ready/play/pause/disconnect/reconnect/finish 전이 결정
5. replacement는 authority handoff, 실제 disconnect는 15초 side reservation
6. simulation terminal 후 repository `finalizeMatch`
7. 실패 시 room-owned exponential retry·stable key·drain pending 유지
8. 성공 시 한 번의 finished event·room cleanup·reservation release
9. drain은 신규 진입 거부 후 기존 room을 최대 60초 기다림
<!-- ANSWER:thread-04-flow:end -->

## 학습 완료 확인

<!-- ANSWER:thread-04-checks:begin -->
### 완료 확인

- [x] replacement socket이 reconnect timer를 만들면 왜 잘못인지 설명할 수 있다.
- [x] `leaveQueue`와 matched reservation `release`의 차이를 설명할 수 있다.
- [x] room publish 실패 후 재매칭 가능 여부가 cleanup 완전성의 관찰값인 이유를 설명할 수 있다.
- [x] 저장 실패 직후 `game.finished`를 보내면 생기는 split state를 설명할 수 있다.
- [x] drain readiness 503 시점과 기존 room 종료 대기 시점이 왜 분리되는지 설명할 수 있다.

체크는 repository code inspection을 통해 설명이 문서에 작성되었음을 뜻합니다. 실제 repository test suite 실행 완료를 뜻하지 않습니다.
<!-- ANSWER:thread-04-checks:end -->

## 실행 증거 기록

<!-- ANSWER:thread-04-execution:begin -->
- Historical inspection: 각 참조 SHA의 GitHub commit diff와 변경 파일을 개별 조회했습니다.
- Repository test execution: 실행하지 않았습니다. 로컬 환경에서 지정 branch의 전체 checkout을 materialize하지 못해 pnpm, PostgreSQL, Playwright, k6, Toxiproxy 명령 결과를 만들 수 없었습니다.
- Runtime claims: 기록하지 않았습니다. 위의 `증거 성격`은 test 구현과 production code path에 대한 정적·역사적 검사입니다.
- Artifact validation: scaffold/completed 대응, fixed metadata, answer completion, SHA uniqueness, Markdown fence, ZIP 구조는 로컬 검증 스크립트로 검사했습니다.
<!-- ANSWER:thread-04-execution:end -->
===== END FILE: 04-gamehub-lifecycle-reconnect-matchmaking-and-finalization-recovery.md =====

===== BEGIN FILE: 05-postgresql-integration-concurrency-migration-and-failure-injection.md =====
# Development Thread 05: PostgreSQL 통합·동시성·migration·실패 주입 검증

English title: **PostgreSQL Integration, Concurrency, Migration, and Failure Injection**

## Thread 목표

schema-isolated PostgreSQL harness를 기반으로 idempotent finalization, friendship/tournament 경쟁, 역사적 migration 보존, destructive reset guard, 감사 기록 atomicity를 실제 DB semantics로 검증하는 구조를 복원한다.

## 핵심 질문

- memory repository 테스트만으로 row lock·unique constraint·transaction rollback을 증명할 수 없는 이유는 무엇인가?
- schema-per-test 격리와 역순 cleanup은 어떤 test contamination과 failure masking을 막는가?
- 동일 resultKey의 20개 동시 finalization은 exactly-once execution이 아니라 무엇을 증명하는가?
- friendship canonical pair와 tournament capacity serialization은 어떤 race를 막는가?
- migration 검증이 final HEAD schema가 아니라 이전 migration state에서 시작해야 하는 이유는 무엇인가?
- 두 번째 DB write만 실패시키는 CHECK constraint가 atomicity root cause를 어떻게 재현하는가?

## 완료 기준

- Testcontainers·schema/search_path·migration·cleanup ownership을 설명한다.
- memory/PostgreSQL differential test의 공통 contract와 엔진별 근거를 구분한다.
- idempotency, row lock, uniqueness, rollback, migration preservation을 concrete rows로 설명한다.
- destructive reset guard와 admin audit failure injection의 안전 경계를 설명한다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags |
| --- | --- | --- | --- | --- |
| 1 | `c43b87694b29` | `test(db): PostgreSQL integration 환경과 계약 추가` | A | PERSISTENCE, RISK, TEST |
| 2 | `582a1615a2c6` | `test(db): 경기 결과 단일 확정 조건 검증` | A | PERSISTENCE, TOURNAMENT, RISK |
| 3 | `cdaca35ccf7f` | `test(db): friendship와 tournament 경쟁 상태 검증` | A | PERSISTENCE, TOURNAMENT, RISK |
| 4 | `0649b63a1ca9` | `test(db): 인증 migration 중 데이터 보존 검증` | A | AUTH, PERSISTENCE, TEST |
| 5 | `527b5f137425` | `test(db): test database reset guard 검증` | B | PERSISTENCE, TEST |
| 6 | `d0137660cd9f` | `fix(db): 차단 감사 기록을 원자적으로 저장` | A | AUTH, PERSISTENCE, RISK |
| 7 | `9106abc10d0e` | `test(db): 차단 감사 기록 atomicity 검증` | A | AUTH, PERSISTENCE, TEST |

> Commit 순서·subject·importance·tags는 Phase 1 감사 후 동결된 정보입니다. Phase 2에서는 변경하지 않습니다.

## Commit `c43b87694b29` — test(db): PostgreSQL integration 환경과 계약 추가

- Full SHA: `c43b87694b29bcee15291c6a752ecf8db77cb356`
- Subject: `test(db): PostgreSQL integration 환경과 계약 추가`
- Importance: **A**
- Tags: PERSISTENCE, RISK, TEST
- Source-defined role: 실제 PostgreSQL 16에서 migration·seed·repository를 격리 실행할 수 있는 schema-per-test harness와 실패-safe cleanup 계약을 구축한다.

### 정확한 조사 대상

- `packages/db/src/postgres.integration.test.ts`의 Testcontainers PostgreSQL 16 lifecycle을 확인한다.
- 각 테스트가 `test_<32hex>` schema와 `search_path`를 만들고 migration/seed를 그 안에서 실행하는지 확인한다.
- pool/repository/schema/container resource를 어떤 순서로 등록하고 역순 정리하는지 확인한다.
- test callback이 실패해도 schema drop과 client close가 실행되는지 확인한다.
- cleanup 오류가 원래 test 오류를 덮지 않는지, 임시 container 종료가 보장되는지 확인한다.

### 학습자 복원

<!-- ANSWER:c43b87694b29:begin -->
#### 역사적 복원

memory repository 테스트만으로는 PostgreSQL의 transaction, unique constraint, row lock, migration 동작을 증명할 수 없다. 반대로 테스트마다 전체 DB를 공유하면 병렬 실행과 실패 잔여 상태 때문에 결과가 비결정적이 된다.

harness는 한 PostgreSQL 16 Testcontainer를 세션 범위에서 띄우고, 각 테스트에 무작위 `test_<32hex>` schema를 만든다. connection의 `search_path`를 해당 schema로 제한한 뒤 migration과 선택적 seed를 실행하므로 같은 container 안에서도 schema와 migration table이 분리된다.

생성된 pool, repository, schema cleanup, 임시 container 같은 자원은 등록 순서의 역순으로 해제된다. 테스트 callback이 실패해도 finally cleanup을 수행하며, cleanup 중 추가 오류가 생겨도 원래 assertion/error를 잃지 않도록 보존한다. 이는 test harness 자체의 ownership·failure semantics를 명시한다.

이 기반이 있어야 뒤의 concurrency·migration·atomicity 테스트가 실제 PostgreSQL semantics를 신뢰할 수 있다.

#### 이 커밋이 보장하는 것

- 각 integration test가 독립 schema와 migration state를 가진다.
- 실패 경로에서도 DB 자원이 역순으로 정리되고 원래 실패 원인이 보존된다.
- memory 구현이 숨기는 PostgreSQL 고유 동작을 실제 engine에서 검증할 수 있다.

#### 이 커밋이 보장하지 않는 것

- 운영 DB의 데이터 규모·권한·replication을 재현하지 않는다.
- Testcontainers를 실행할 수 없는 환경에서는 suite가 실제로 수행되었다는 증거가 되지 않는다.

#### 전후 관계

이 harness 위에서 `582a1615a2c6`, `cdaca35ccf7f`, `0649b63a1ca9`, `9106abc10d0e`가 실제 transaction/constraint/migration 실패를 검증한다.

#### 증거 성격

- PostgreSQL integration test infrastructure and cleanup contract
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:c43b87694b29:end -->

## Commit `582a1615a2c6` — test(db): 경기 결과 단일 확정 조건 검증

- Full SHA: `582a1615a2c6de131dffee725f4a17e2223f86c8`
- Subject: `test(db): 경기 결과 단일 확정 조건 검증`
- Importance: **A**
- Tags: PERSISTENCE, TOURNAMENT, RISK
- Source-defined role: 동일 logical match의 반복·동시 finalization과 tournament partial failure가 memory/PostgreSQL 양쪽에서 한 결과로 수렴하는지 검증한다.

### 정확한 조사 대상

- memory와 PostgreSQL에 공통 적용되는 repository contract test 구성을 확인한다.
- 동일 `resultKey`로 20개 concurrent `finalizeMatch`를 실행하는 방식을 확인한다.
- match row, participant counters, ratings, rating-history row가 한 번만 반영되는지 확인한다.
- tournament match linkage가 없을 때 전체 transaction이 rollback되는 사례를 확인한다.
- 두 semifinal이 동시에 끝나도 final bracket row가 하나만 만들어지는지 확인한다.

### 학습자 복원

<!-- ANSWER:582a1615a2c6:begin -->
#### 역사적 복원

atomic finalization 구현은 unique key와 transaction을 사용하지만, 반복 호출·동시 호출·중간 domain failure에서 실제로 수렴하는지 별도 증거가 필요하다. 이 테스트는 memory와 PostgreSQL에 같은 시나리오를 적용해 repository abstraction의 의미를 비교한다.

동일 `resultKey`로 20개 `finalizeMatch`를 병렬 실행한다. 성공 응답은 기존 결과를 되돌려줄 수 있지만, durable match row와 두 participant의 통계·현재 rating·rating history는 한 세트만 생성되어야 한다. PostgreSQL에서는 unique result key와 transaction/row locks가 이를 보장하고 memory 구현은 같은 domain 결과를 모사한다.

tournament match 연결이 유효하지 않은 경우 match와 rating side effect 모두 rollback되어야 한다. 두 semifinal을 동시에 확정하는 사례에서는 finalist 조회와 final insertion이 경쟁해도 final row가 하나만 남고 bracket가 일관되어야 한다.

이 테스트는 exactly-once execution을 주장하지 않는다. 여러 호출이 실행되지만 observable durable effects가 idempotency key로 한 번에 수렴함을 증명한다.

#### 이 커밋이 보장하는 것

- 동일 logical result의 반복·동시 요청이 하나의 durable match와 한 세트의 rating effects로 수렴한다.
- tournament linkage failure가 partial match/rating state를 남기지 않는다.
- 동시 semifinal completion이 duplicate final을 만들지 않는다.

#### 이 커밋이 보장하지 않는 것

- 분산 DB나 process crash 직후 transaction outcome 불확실성까지 재현하지 않는다.
- 서로 다른 resultKey로 같은 실제 경기를 제출하는 semantic duplicate는 별도 상위 invariant가 필요하다.

#### 전후 관계

`e939a50948b2`의 stable retry key가 이 repository idempotency boundary를 사용한다.

#### 증거 성격

- memory/PostgreSQL differential concurrency and rollback regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:582a1615a2c6:end -->

## Commit `cdaca35ccf7f` — test(db): friendship와 tournament 경쟁 상태 검증

- Full SHA: `cdaca35ccf7fa9f86e7a89c16686d852d210fdae`
- Subject: `test(db): friendship와 tournament 경쟁 상태 검증`
- Importance: **A**
- Tags: PERSISTENCE, TOURNAMENT, RISK
- Source-defined role: unordered friendship identity와 tournament 마지막 슬롯이 반복·역방향·동시 요청에서도 memory/PostgreSQL 양쪽에서 일관되는지 검증한다.

### 정확한 조사 대상

- self-friend, repeated request, reversed pair request의 기대 상태를 확인한다.
- canonical `(min_user_id, max_user_id)` 또는 동등한 memory key가 사용되는지 확인한다.
- 마지막 tournament slot에 10개 caller를 동시에 넣어 one success/nine full을 요구하는지 확인한다.
- 최종 entry가 정확히 4명·seed가 unique인지, semifinal이 정확히 2개 생성되는지 확인한다.
- 이미 참가한 사용자의 재요청이 idempotent한지와 DB constraint 직접 위반 사례를 확인한다.

### 학습자 복원

<!-- ANSWER:cdaca35ccf7f:begin -->
#### 역사적 복원

friendship을 방향성 row 두 개로 취급하거나 tournament capacity를 read-then-insert로 구현하면 reversed 요청과 마지막 슬롯 경쟁에서 duplicate/overflow가 생긴다. 테스트는 두 문제를 같은 'domain identity + serialized capacity' 관점에서 검증한다.

friendship은 self 관계를 거부하고, A→B와 B→A를 같은 canonical pair로 본다. 반복·역방향 요청 후 row는 하나이며 상태 전이는 repository 계약에 맞게 수렴한다. memory 구현도 unordered pair key를 사용해 동일 의미를 보존해야 한다.

tournament에는 3명이 먼저 들어간 뒤 10개 caller가 마지막 슬롯을 동시에 요청한다. 정확히 하나만 성공하고 나머지는 full이어야 하며 최종 entry는 4개, seed는 unique, semifinal은 2개다. row lock/constraint가 없는 check-then-insert 구현이면 이 사례에서 실패한다.

이 테스트는 DB constraint와 repository transaction을 모두 관찰하며, 단순 sequential happy path가 잡지 못하는 경쟁 조건을 재현한다.

#### 이 커밋이 보장하는 것

- friendship identity가 방향과 무관하게 하나로 수렴한다.
- tournament capacity와 seed uniqueness가 동시 admission에서도 보존된다.
- memory/PostgreSQL 구현이 같은 domain 결과를 낸다.

#### 이 커밋이 보장하지 않는 것

- 높은 contention에서의 latency나 starvation을 측정하지 않는다.
- 네트워크 retry를 포함한 HTTP layer idempotency는 별도 범위다.

#### 전후 관계

category의 concurrency 검증 축을 match finalization 외 friendship/tournament admission까지 확장한다.

#### 증거 성격

- differential concurrency and constraint regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:cdaca35ccf7f:end -->

## Commit `0649b63a1ca9` — test(db): 인증 migration 중 데이터 보존 검증

- Full SHA: `0649b63a1ca9ee4fb171e5dfc0fdb35802747d14`
- Subject: `test(db): 인증 migration 중 데이터 보존 검증`
- Importance: **A**
- Tags: AUTH, PERSISTENCE, TEST
- Source-defined role: 인증 계약 변경 migration이 기존 session을 의도적으로 폐기하면서 사용자·경기·rating history를 보존하는지 이전 schema 상태에서 검증한다.

### 정확한 조사 대상

- integration harness가 migration 004까지만 적용하도록 구성되는지 확인한다.
- 그 상태에서 users, active session, finalized match, rating history를 삽입하는지 확인한다.
- migration 전 보존 대상 snapshot을 어떤 query로 캡처하는지 확인한다.
- 전체 migration 적용 뒤 session row는 없어지고 users/matches/history는 동일한지 확인한다.
- 최종 HEAD schema를 직접 만든 뒤 결과만 비교하지 않는지 확인한다.

### 학습자 복원

<!-- ANSWER:0649b63a1ca9:begin -->
#### 역사적 복원

authentication transport가 바뀔 때 기존 session을 그대로 유지하면 새 보안 규칙을 우회하는 credential이 남을 수 있다. 그러나 migration이 session과 무관한 사용자·경기 기록까지 삭제하면 데이터 손실이다.

테스트는 최신 schema에서 시작하지 않고 migration 004까지만 적용한 실제 이전 상태를 만든다. users, active session, finalized match, rating history를 넣고 보존 대상 projection을 snapshot한 뒤 나머지 migration을 순서대로 적용한다.

결과는 session row가 0이 되어 강제 재인증되지만 사용자, match, rating-history 데이터는 migration 전과 같아야 한다. 이는 'credential invalidation'과 'domain data preservation'을 동시에 검증한다.

역사적 migration을 실제 순서로 실행하므로 final HEAD schema를 보고 과거 상태를 추정하는 오류를 피한다.

#### 이 커밋이 보장하는 것

- legacy session은 안전하게 만료되고 관련 없는 durable domain data는 보존된다.
- migration set이 실제 이전 상태에서 forward 적용 가능함을 증명한다.

#### 이 커밋이 보장하지 않는 것

- 대규모 운영 데이터에서 migration 시간·lock 영향을 측정하지 않는다.
- rollback/down migration을 검증하지 않는다.

#### 전후 관계

`c43b87694b29`의 schema-isolated PostgreSQL harness를 사용하며 cookie-only 인증 전환의 durable cleanup을 검증한다.

#### 증거 성격

- historical migration preservation regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:0649b63a1ca9:end -->

## Commit `527b5f137425` — test(db): test database reset guard 검증

- Full SHA: `527b5f137425adadc486a6b071581f72c0005b16`
- Subject: `test(db): test database reset guard 검증`
- Importance: **B**
- Tags: PERSISTENCE, TEST
- Source-defined role: 파괴적 test reset이 test 환경과 명확한 대상에서만 실행되고 sibling schema를 보존하는지 검증한다.

### 정확한 조사 대상

- `packages/db/src/testReset.test.ts`에서 `NODE_ENV=test`, `TEST_DATABASE_URL`, PostgreSQL scheme 요구를 확인한다.
- 일반 운영형 database name과 모호한 `search_path`가 거부되는지 확인한다.
- 명확한 test database public schema 또는 exact isolated schema만 허용되는지 확인한다.
- PostgreSQL integration에서 target schema reset·remigrate 후 sibling schema가 유지되는지 확인한다.

### 학습자 복원

<!-- ANSWER:527b5f137425:begin -->
#### 역사적 복원

테스트 편의를 위한 schema drop/reset은 잘못된 URL이나 환경 변수에서 실행되면 운영 데이터 손실로 이어진다. 이 테스트는 destructive operation의 precondition을 fail-closed로 고정한다.

`NODE_ENV=test`, 별도 `TEST_DATABASE_URL`, PostgreSQL URL, 명확한 test database 또는 exact isolated schema를 모두 확인한다. integration 사례는 target만 reset하고 migration을 다시 적용한 뒤 sibling schema의 marker가 남아 있는지 확인한다.

#### 이 커밋이 보장하는 것

- 파괴적 reset target이 명시적 test 범위 밖으로 확장되지 않는다.
- isolated schema reset이 sibling schema를 손상하지 않는다.

#### 이 커밋이 보장하지 않는 것

- DB 사용자의 실제 권한 최소화까지 검증하지 않는다.
- 오퍼레이터가 잘못된 test 명명 규칙을 사용하는 모든 가능성을 제거하지 않는다.

#### 전후 관계

PostgreSQL integration harness의 반복 실행을 안전하게 만드는 test-infrastructure guard다.

#### 증거 성격

- destructive-operation guard regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:527b5f137425:end -->

## Commit `d0137660cd9f` — fix(db): 차단 감사 기록을 원자적으로 저장

- Full SHA: `d0137660cd9feb48e594a78b8db23eb8633d23aa`
- Subject: `fix(db): 차단 감사 기록을 원자적으로 저장`
- Importance: **A**
- Tags: AUTH, PERSISTENCE, RISK
- Source-defined role: 사용자 상태 변경과 대응하는 관리자 감사 기록을 하나의 PostgreSQL transaction으로 묶는다.

### 정확한 조사 대상

- `packages/db/src/index.ts`의 관리자 상태 변경 repository method를 확인한다.
- user `status`/`banned_at` update와 `admin_actions` insert가 같은 `this.db.transaction` callback 안에 있는지 확인한다.
- 두 statement 사이 실패 시 외부로 어떤 오류가 전달되는지 확인한다.
- memory implementation의 동등한 domain behavior와 비교한다.

### 학습자 복원

<!-- ANSWER:d0137660cd9f:begin -->
#### 역사적 복원

이전 경로는 사용자 상태를 먼저 update하고 감사 row를 나중에 insert할 수 있었다. 두 번째 쓰기가 실패하면 계정은 차단되었지만 누가 왜 변경했는지 기록이 없는 비감사 상태가 남는다.

수정은 user update와 `admin_actions` insert를 하나의 `this.db.transaction` 안에서 실행한다. 둘 중 하나라도 실패하면 transaction 전체가 rollback되어 account state와 audit trail이 함께 이전 상태로 돌아간다.

원자성의 기준은 '두 API 호출이 모두 성공했다'가 아니라 같은 DB transaction commit에 포함되는 것이다. 감사 기록은 부가 로그가 아니라 authorization state transition의 일부로 승격된다.

#### 이 커밋이 보장하는 것

- 차단/해제 상태와 감사 row가 함께 commit되거나 함께 rollback된다.
- 감사 불가능한 account state transition을 만들지 않는다.

#### 이 커밋이 보장하지 않는 것

- 외부 로그나 알림 전송까지 같은 transaction에 포함하지 않는다.
- 관리자 권한 자체의 검증은 상위 API 경계 책임이다.

#### 전후 관계

`9106abc10d0e`가 두 번째 insert만 deterministic하게 실패시켜 실제 rollback을 검증한다.

#### 증거 성격

- transaction-boundary atomicity fix
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:d0137660cd9f:end -->

## Commit `9106abc10d0e` — test(db): 차단 감사 기록 atomicity 검증

- Full SHA: `9106abc10d0e9e191ed388d7879012664f0136bf`
- Subject: `test(db): 차단 감사 기록 atomicity 검증`
- Importance: **A**
- Tags: AUTH, PERSISTENCE, TEST
- Source-defined role: 감사 insert만 실패하도록 임시 DB constraint를 주입해 선행 user update까지 rollback되는지 검증한다.

### 정확한 조사 대상

- PostgreSQL integration test가 `admin_actions.reason`의 특정 값만 거부하는 temporary CHECK constraint를 설치하는지 확인한다.
- user update가 먼저 실행된 뒤 audit insert에서 실패하도록 입력을 구성하는지 확인한다.
- repository call이 reject하는지와 transaction 종료 뒤 user status/banned_at을 재조회하는지 확인한다.
- `admin_actions` row count가 0인지 확인하고 constraint cleanup을 추적한다.

### 학습자 복원

<!-- ANSWER:9106abc10d0e:begin -->
#### 역사적 복원

단순히 transaction API를 사용했다는 코드 검사는 실제 statement 순서와 rollback 결과를 증명하지 못한다. 테스트는 임시 CHECK constraint로 특정 reason의 `admin_actions` insert만 실패시키고, user update는 정상 실행 가능한 입력을 사용한다.

repository call은 constraint violation으로 reject해야 한다. 새 connection/query로 user를 다시 읽었을 때 status는 active, `banned_at`은 null이며 관련 admin action은 0개여야 한다. 즉 두 번째 write의 실패가 첫 번째 write까지 rollback했다.

이 방식은 mock throw보다 실제 PostgreSQL constraint와 transaction semantics를 통과하는 deterministic failure injection이다.

#### 이 커밋이 보장하는 것

- 감사 insert 실패가 사용자 상태 변경까지 실제 DB에서 rollback함을 증명한다.
- partial authorization state가 commit되지 않는다.

#### 이 커밋이 보장하지 않는 것

- DB crash recovery나 network disconnect 직후 commit outcome을 재현하지 않는다.
- 모든 가능한 audit schema constraint를 열거하지 않는다.

#### 전후 관계

`d0137660cd9f`의 previous split-write assumption → transaction fix → DB constraint regression 관계를 완성한다.

#### 증거 성격

- real-PostgreSQL deterministic failure injection
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:9106abc10d0e:end -->


## Thread-level invariant evolution

다음 표에서 불변식이 도입·확장·교정·검증된 SHA를 구분합니다.

<!-- ANSWER:thread-05-invariants:begin -->
### 복원 결과

| SHA | 변화 | 불변식 |
| --- | --- | --- |
| `c43b87694b29` | 검증 기반 | 각 test는 독립 schema/migration state를 가지며 실패에서도 자원을 역순 정리한다. |
| `582a1615a2c6` | 원자·멱등 결과 | 반복·동시 finalization이 하나의 match/rating/bracket effect로 수렴하고 partial failure는 rollback된다. |
| `cdaca35ccf7f` | 도메인 경쟁 | unordered friendship identity와 tournament capacity/seed가 concurrency에서도 유지된다. |
| `0649b63a1ca9` | 역사적 migration | legacy session은 폐기하되 user/match/rating data는 이전 schema에서 forward migration 후 보존된다. |
| `527b5f137425` | 파괴 작업 guard | reset은 명시 test target에만 적용되고 sibling schema를 보존한다. |
| `d0137660cd9f` | transaction 수정 | user status와 admin audit row가 한 commit/rollback 단위가 된다. |
| `9106abc10d0e` | 실패 주입 회귀 | audit insert constraint failure가 선행 user update까지 실제 PostgreSQL에서 rollback한다. |

각 변화는 이전 커밋의 최종 상태를 다음 커밋이 단순 반복한 것이 아니라, 새 경계를 도입·확장·교정하거나 regression으로 보호한 시점을 나타냅니다.
<!-- ANSWER:thread-05-invariants:end -->

## Failure → Fix → Test 관계

구현 추가를 독립 기능으로 나열하지 말고, 이전 가정과 실제 실패 위험, 수정된 결정, 회귀 증거를 연결합니다.

<!-- ANSWER:thread-05-failure-relations:begin -->
### 복원 결과

| Failure / 이전 가정 | Fix / 결정 | Verification |
| --- | --- | --- |
| 공유 DB/schema로 테스트가 서로 오염되고 cleanup 실패가 원래 오류를 가림 | schema-per-test + tracked reverse cleanup | `c43b87694b29` harness contract |
| 동시 match finalize가 duplicate match/rating/final 생성 | unique result key + transaction + locks | `582a1615a2c6` 20-way concurrency/rollback |
| 역방향 friendship duplicate·last-slot tournament overflow | canonical pair + row-locked admission/constraints | `cdaca35ccf7f` |
| 인증 migration이 session 외 domain data까지 손상 | legacy session delete를 좁은 migration으로 수행 | `0649b63a1ca9` pre-state snapshot comparison |
| audit insert 실패 뒤 user만 차단됨 | `d0137660cd9f` transaction | `9106abc10d0e` temporary CHECK failure injection |
<!-- ANSWER:thread-05-failure-relations:end -->

## Ownership·state·responsibility 변화

자원과 상태를 누가 만들고, 보유하고, 이전하고, 정리하는지 커밋 순서에 따라 정리합니다.

<!-- ANSWER:thread-05-ownership:begin -->
### 복원 결과

| 대상 | 소유자 | 책임·수명 |
| --- | --- | --- |
| PostgreSQL process | Testcontainers session harness | 한 container의 시작·종료를 소유한다. |
| Per-test namespace | generated schema + `search_path` | migration table, data, constraints의 격리를 소유한다. |
| Test resources | tracked cleanup stack | pool/repository/schema/container를 생성 역순으로 해제한다. |
| Logical match identity | `resultKey` unique boundary | 중복 실행을 하나의 durable 결과로 수렴시킨다. |
| Concurrent rows/capacity | PostgreSQL transaction + row locks + constraints | participant/tournament/friendship 경쟁을 직렬화한다. |
| Historical schema evolution | ordered migration set | 이전 state에서 현재 state로 데이터 보존·폐기 정책을 소유한다. |
| Admin status transition | single DB transaction | user row와 audit row를 하나의 commit 단위로 소유한다. |
<!-- ANSWER:thread-05-ownership:end -->

## 최종 Thread 상태

최종 HEAD를 과거 커밋에 투영하지 말고, 위 SHA 순서에서 도달한 보장을 요약합니다.

<!-- ANSWER:thread-05-final-state:begin -->
### 최종 상태

- 고위험 persistence invariant는 실제 PostgreSQL 16의 migration·constraint·transaction·lock semantics에서 검증된다.
- memory와 PostgreSQL은 같은 repository domain 결과를 내지만 엔진 고유 원자성은 PostgreSQL integration으로 별도 증명된다.
- match finalization, friendship, tournament admission이 concurrency에서 중복·초과·partial state를 만들지 않는다.
- 역사적 auth migration은 session만 폐기하고 domain data를 보존하며 reset은 명시 test target 밖으로 확장되지 않는다.
- 관리자 상태와 감사 기록은 deterministic DB failure에서도 함께 rollback된다.
<!-- ANSWER:thread-05-final-state:end -->

## 최종 architecture / execution flow

실제 caller→boundary→state transition→cleanup 흐름을 순서대로 작성합니다.

<!-- ANSWER:thread-05-flow:begin -->
### 최종 실행 흐름

1. PostgreSQL 16 container 시작
2. 테스트별 random schema 생성·`search_path` 설정
3. 필요 migration state까지 적용·fixture 입력
4. repository operation 또는 동시 호출 실행
5. DB row/constraint/transaction 결과를 별도 query로 관찰
6. 실패 포함 callback 종료
7. pool/repository/schema를 역순 cleanup
8. 원래 assertion error를 cleanup error보다 우선 보존
<!-- ANSWER:thread-05-flow:end -->

## 학습 완료 확인

<!-- ANSWER:thread-05-checks:begin -->
### 완료 확인

- [x] schema-per-test가 database-per-test보다 선택된 실용적 이유와 격리 한계를 설명할 수 있다.
- [x] 20개 concurrent call이 '함수가 한 번 실행됨'을 증명하지 않는 이유를 설명할 수 있다.
- [x] migration 004 상태를 직접 만들지 않고 final schema에서 테스트하면 놓치는 것을 설명할 수 있다.
- [x] temporary CHECK constraint가 mock throw보다 강한 atomicity 증거인 이유를 설명할 수 있다.
- [x] reset guard가 환경 변수·URL·schema 이름을 모두 확인해야 하는 이유를 설명할 수 있다.

체크는 repository code inspection을 통해 설명이 문서에 작성되었음을 뜻합니다. 실제 repository test suite 실행 완료를 뜻하지 않습니다.
<!-- ANSWER:thread-05-checks:end -->

## 실행 증거 기록

<!-- ANSWER:thread-05-execution:begin -->
- Historical inspection: 각 참조 SHA의 GitHub commit diff와 변경 파일을 개별 조회했습니다.
- Repository test execution: 실행하지 않았습니다. 로컬 환경에서 지정 branch의 전체 checkout을 materialize하지 못해 pnpm, PostgreSQL, Playwright, k6, Toxiproxy 명령 결과를 만들 수 없었습니다.
- Runtime claims: 기록하지 않았습니다. 위의 `증거 성격`은 test 구현과 production code path에 대한 정적·역사적 검사입니다.
- Artifact validation: scaffold/completed 대응, fixed metadata, answer completion, SHA uniqueness, Markdown fence, ZIP 구조는 로컬 검증 스크립트로 검사했습니다.
<!-- ANSWER:thread-05-execution:end -->
===== END FILE: 05-postgresql-integration-concurrency-migration-and-failure-injection.md =====

===== BEGIN FILE: 06-process-smoke-and-browser-end-to-end-verification.md =====
# Development Thread 06: process smoke·browser end-to-end 검증

English title: **Process Smoke and Browser End-to-End Verification**

## Thread 목표

실행 중 API/WebSocket/browser를 잇는 검증 경계가 초기 bearer/query 방식에서 최종 cookie-ticket-v1 방식으로 교정되고, guest browser 복구까지 확장되는 과정을 복원한다.

## 핵심 질문

- unit/Fastify injection 테스트와 실제 process smoke는 어떤 서로 다른 실패를 잡는가?
- 초기 smoke가 기능상 유효해도 최종 인증 계약과 어긋나면 왜 수정되어야 하는가?
- 두 socket이 같은 room을 관찰한다는 조건을 어떻게 확인하는가?
- Canvas element 존재가 아니라 pixel alpha를 확인하는 이유는 무엇인가?
- guest AI fallback 시간과 reconnect를 UI만이 아니라 WebSocket frame·routed socket으로 어떻게 관찰하는가?

## 완료 기준

- 초기 HTTP/WS smoke의 당시 credential/protocol을 역사적으로 정확히 설명한다.
- `17e7...`가 테스트를 cookie/ticket/v1로 migration한 이유와 구체 변경을 설명한다.
- Playwright desktop/mobile config와 Canvas paint evidence의 범위를 설명한다.
- 두 BrowserContext, frame timestamp, `routeWebSocket`을 사용한 guest flow를 설명한다.
- 실제 실행하지 않은 결과를 문서에서 주장하지 않는다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags |
| --- | --- | --- | --- | --- |
| 1 | `9a0562d395db` | `test(smoke): HTTP API 실행 검사 추가` | B | AUTH, OPERATIONS, TEST |
| 2 | `8a462f6f05b3` | `test(smoke): WebSocket 경기 실행 검사 추가` | B | AUTH, PROTOCOL, SIMULATION |
| 3 | `d755b8dae2c1` | `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성` | B | PERSISTENCE, TOURNAMENT, WEB |
| 4 | `17e7dab21a55` | `test(smoke): cookie 기반 realtime protocol 검증` | B | AUTH, PROTOCOL, SIMULATION |
| 5 | `1abda1299ad8` | `test(e2e): 비회원 체험 브라우저 흐름 검증` | A | AUTH, REALTIME, WEB |

> Commit 순서·subject·importance·tags는 Phase 1 감사 후 동결된 정보입니다. Phase 2에서는 변경하지 않습니다.

## Commit `9a0562d395db` — test(smoke): HTTP API 실행 검사 추가

- Full SHA: `9a0562d395db8a2a9d37fa8496c33207913f5576`
- Subject: `test(smoke): HTTP API 실행 검사 추가`
- Importance: **B**
- Tags: AUTH, OPERATIONS, TEST
- Source-defined role: 빌드된 실행 시스템에 대해 개발 로그인과 주요 HTTP read path가 응답하는지 확인하는 초기 process smoke를 추가한다.

### 정확한 조사 대상

- `tests/smoke-api.mjs`의 base URL, request helper, 실패 시 status/body 보고 방식을 확인한다.
- dev-login 후 `/me`, `/lobby`, `/dashboard`와 public leaderboard를 호출하는 순서를 확인한다.
- 당시 인증이 bearer token을 사용했다는 historical state를 그대로 기록한다.
- 비-2xx가 즉시 process failure로 변환되는지 확인한다.

### 학습자 복원

<!-- ANSWER:9a0562d395db:begin -->
#### 역사적 복원

unit/injection 테스트만으로는 실제로 빌드된 API process가 listen하고 route·repository composition을 완료했는지 알 수 없다. 이 커밋은 외부 HTTP client로 실행 중 service를 두드리는 최소 smoke boundary를 만든다.

스크립트는 개발 로그인 응답의 bearer token을 받아 `/me`, `/lobby`, `/dashboard`에 전달하고 public leaderboard도 호출한다. 비-2xx면 path, status, response text를 포함해 실패한다.

중요한 역사적 한계는 이 smoke가 당시의 bearer credential transport를 전제로 했다는 점이다. cookie-only 전환 뒤에는 보안 계약과 어긋나며 `17e7dab21a55`에서 수정된다.

#### 이 커밋이 보장하는 것

- 실행 중 HTTP process의 기본 인증·조회 경로가 연결되었는지 빠르게 확인한다.
- 실패가 shell/CI에서 non-zero로 드러나는 실행 경계를 제공한다.

#### 이 커밋이 보장하지 않는 것

- 응답의 모든 schema·side effect를 깊게 검증하지 않는다.
- 초기 bearer 인증 방식은 최종 보안 invariant가 아니다.

#### 전후 관계

`17e7dab21a55`가 이 smoke를 cookie-only session과 no-token response로 교정한다.

#### 증거 성격

- running-process HTTP smoke
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:9a0562d395db:end -->

## Commit `8a462f6f05b3` — test(smoke): WebSocket 경기 실행 검사 추가

- Full SHA: `8a462f6f05b3612a8058cd8d4d474c9910d2b487`
- Subject: `test(smoke): WebSocket 경기 실행 검사 추가`
- Importance: **B**
- Tags: AUTH, PROTOCOL, SIMULATION
- Source-defined role: 두 실행 중 WebSocket client가 로그인·매칭·준비·playing snapshot·room chat까지 통과하는 초기 realtime process smoke를 추가한다.

### 정확한 조사 대상

- `tests/smoke-ws.mjs`와 Makefile `smoke` target을 확인한다.
- 두 사용자 로그인 후 query session token으로 socket을 여는 당시 방식을 기록한다.
- 양쪽 `queue.join`, 동일 room의 `queue.matched`, 양쪽 `game.ready`, playing snapshot 대기를 확인한다.
- match-scope chat 전송·수신과 10초 polling timeout, finally socket close를 확인한다.

### 학습자 복원

<!-- ANSWER:8a462f6f05b3:begin -->
#### 역사적 복원

HTTP smoke만으로는 WebSocket upgrade, hub registration, matchmaking, room scheduling, simulation snapshot 경로가 실제 process에서 연결되었는지 알 수 없다.

스크립트는 두 사용자를 로그인하고 두 socket을 열어 양쪽이 같은 room assignment를 받는지 확인한다. 이어 ready를 보내 playing snapshot과 room chat을 기다리고, 모든 대기에는 bounded timeout이 있으며 finally에서 socket을 닫는다.

당시에는 session token을 query에 넣고 unversioned shape를 사용했다. 기능 연결 증거로는 유효하지만 최종 auth/protocol 계약으로는 obsolete이며 `17e7dab21a55`가 교정한다.

#### 이 커밋이 보장하는 것

- 실행 중 API의 socket upgrade→matchmaking→simulation→chat 경로가 최소한 연결됨을 확인한다.
- 양쪽 client가 같은 room을 관찰하도록 요구한다.

#### 이 커밋이 보장하지 않는 것

- 결정적 physics, reconnect, persistence atomicity를 증명하지 않는다.
- 초기 query-token/unversioned protocol은 후속 보안·프로토콜 전환을 반영하지 않는다.

#### 전후 관계

`17e7dab21a55`가 이 시나리오를 one-time ticket과 version-1 event로 다시 작성한다.

#### 증거 성격

- running-process WebSocket smoke
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:8a462f6f05b3:end -->

## Commit `d755b8dae2c1` — test(e2e): 한국어 내비게이션과 캔버스 흐름 구성

- Full SHA: `d755b8dae2c1a73507bc3526a510f51159fd2759`
- Subject: `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성`
- Importance: **B**
- Tags: PERSISTENCE, TOURNAMENT, WEB
- Source-defined role: Playwright 기반 desktop/mobile browser runner와 한국어 핵심 navigation·Canvas paint 검증을 도입한다.

### 정확한 조사 대상

- `playwright.config.ts`의 testDir, timeout, baseURL, trace/screenshot 정책을 확인한다.
- desktop Chromium viewport와 Pixel 7 mobile project를 확인한다.
- `tests/e2e/pong-pong.spec.ts`에서 개발 로그인 후 dashboard/leaderboard/tournament 이동을 확인한다.
- Canvas의 2D pixel alpha를 직접 읽어 실제 paint가 존재하는지 확인한다.
- package script와 Makefile `e2e` target을 확인한다.

### 학습자 복원

<!-- ANSWER:d755b8dae2c1:begin -->
#### 역사적 복원

process smoke는 API와 protocol 연결을 확인하지만 실제 browser rendering, accessible name, client navigation, Canvas drawing을 보지 않는다. 이 커밋은 Playwright를 repository-level E2E runner로 도입한다.

첫 시나리오는 한국어 heading/label/role을 사용해 로그인과 주요 화면 이동을 수행한다. 두 번째는 경기 화면의 canvas context에서 pixel data를 읽고 alpha가 0이 아닌 픽셀이 존재하는지 확인한다. 단순 element 존재가 아니라 renderer가 실제로 draw했다는 최소 증거다.

desktop과 mobile Chromium project, failure trace, failure screenshot, configurable base URL이 실행·진단 계약을 만든다.

#### 이 커밋이 보장하는 것

- 핵심 한국어 navigation과 Canvas paint가 실제 browser에서 연결됨을 검증한다.
- desktop/mobile viewport에서 같은 시나리오를 실행할 수 있는 runner를 제공한다.

#### 이 커밋이 보장하지 않는 것

- pixel 내용이 물리 상태와 정확히 일치하는지 비교하지 않는다.
- 다른 browser engine과 전체 접근성 규칙을 검증하지 않는다.

#### 전후 관계

후속 E2E가 API action, 사용자 격리, guest/reconnect 흐름을 추가하지만 이 category thread는 최초 browser execution boundary와 최종 guest high-risk flow를 대표로 연결한다.

#### 증거 성격

- browser end-to-end rendering/navigation evidence
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:d755b8dae2c1:end -->

## Commit `17e7dab21a55` — test(smoke): cookie 기반 realtime protocol 검증

- Full SHA: `17e7dab21a550d4bcd85fa1c4f93a57bf4381942`
- Subject: `test(smoke): cookie 기반 realtime protocol 검증`
- Importance: **B**
- Tags: AUTH, PROTOCOL, SIMULATION
- Source-defined role: 기존 process smoke와 E2E를 cookie-only session, one-time WebSocket ticket, version-1 realtime protocol에 맞게 교정한다.

### 정확한 조사 대상

- `tests/smoke-api.mjs`가 `Set-Cookie`를 추출하고 JSON token 노출을 명시적으로 거부하는지 확인한다.
- `/me`, `/lobby`, chat, dashboard, tournament 요청이 cookie를 사용하고 admin handle이 403인지 확인한다.
- `tests/smoke-ws.mjs`가 cookie로 `/auth/ws-ticket`을 호출한 뒤 `?ticket=...&v=1`로 연결하는지 확인한다.
- 모든 outbound command에 `v: 1`을 붙이고 nested snapshot state/sequence를 parse하는지 확인한다.
- presence, 동일 room match, ready, ball acceleration, pause/resume, match chat, AI fallback을 확인한다.

### 학습자 복원

<!-- ANSWER:17e7dab21a55:begin -->
#### 역사적 복원

초기 smoke는 실행 연결을 보여 주었지만 bearer/query session과 구형 event shape에 고정되어 최종 보안·프로토콜 계약을 오히려 우회했다. 이 커밋은 테스트 자체를 production boundary에 맞게 수정한다.

HTTP smoke는 dev-login의 `Set-Cookie`를 사용하고 response body에 `token` 필드가 있으면 실패한다. 모든 protected request는 cookie를 보내며, `admin` handle만으로 admin route를 호출하면 403을 요구한다.

WebSocket smoke는 각 cookie로 one-time ticket을 발급하고 version 1 handshake를 연다. command는 `v: 1`을 가지며 snapshot은 `snapshot.state`와 sequence 구조로 읽는다. 두 client의 presence, same-room matchmaking, readiness, 공 가속, pause/resume, match chat과 한 명 queue의 AI fallback까지 실제 process에서 관찰한다.

따라서 이 커밋은 기능 추가가 아니라 obsolete verification transport를 교정한다. 테스트가 제품 보안 경계를 우회하지 않아야 한다는 원칙을 보여 준다.

#### 이 커밋이 보장하는 것

- process smoke가 cookie-only/no-JSON-token/one-time-ticket/v1 protocol을 실제로 따른다.
- handle 기반 권한 획득이 실행 시스템에서도 거부된다.
- 핵심 realtime state와 simulation behavior를 end-to-end로 관찰한다.

#### 이 커밋이 보장하지 않는 것

- 모든 negative protocol·concurrency·DB rollback을 process 수준에서 반복하지 않는다.
- 실제 browser reconnect UX는 `1abda1299ad8` 범위다.

#### 전후 관계

`9a0562d395db`과 `8a462f6f05b3`의 obsolete credential/protocol 가정을 수정하고 final auth/realtime execution evidence로 정렬한다.

#### 증거 성격

- historical smoke-contract migration and running-system regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:17e7dab21a55:end -->

## Commit `1abda1299ad8` — test(e2e): 비회원 체험 브라우저 흐름 검증

- Full SHA: `1abda1299ad80e299e79297060531eee90112fb8`
- Subject: `test(e2e): 비회원 체험 브라우저 흐름 검증`
- Importance: **A**
- Tags: AUTH, REALTIME, WEB
- Source-defined role: demo mode에서 guest 진입·capability 제한·두 browser PvP·6초 AI fallback·fresh-ticket reconnect를 실제 browser lifecycle로 검증한다.

### 정확한 조사 대상

- `tests/e2e/guest-demo.spec.ts`의 demo-mode skip과 serial execution 설정을 확인한다.
- 입력 없이 guest로 진입하고 navigation이 lobby/play로 제한되는지 확인한다.
- 독립 BrowserContext 두 개를 생성해 두 guest가 같은 PvP room에 들어가 ready/playing까지 진행하는지 확인한다.
- WebSocket frame observer로 `queue.join`부터 `queue.matched`까지 5.5~10초 범위를 측정하는지 확인한다.
- `page.routeWebSocket`으로 진행 중 socket을 code 1012로 닫고 두 번째 연결·재개 상태를 확인한다.
- 각 context와 routed socket의 cleanup을 확인한다.

### 학습자 복원

<!-- ANSWER:1abda1299ad8:begin -->
#### 역사적 복원

unit guest 테스트는 서명, lease, capability를 검증하지만 browser가 실제 demo policy를 소비하고 fresh ticket reconnect를 수행하는지 보지 못한다. 이 suite는 `E2E_APP_MODE=demo`일 때만 serial로 실행된다.

첫 시나리오는 credential field 없이 guest session을 만들고 제한된 navigation만 보여 주는지 확인한다. 두 개의 독립 BrowserContext 시나리오는 cookie/session 격리를 유지한 두 guest가 같은 PvP room에 배정되고 양쪽 ready 뒤 playing이 되는지 검증한다.

AI fallback은 UI text만 기다리지 않고 WebSocket frame의 `queue.join`과 `queue.matched` timestamp 차이를 기록해 5.5초 이상 10초 미만을 요구한다. reconnect 시나리오는 `routeWebSocket`으로 진행 중 connection을 1012로 닫고, 재연결 대기 표시·두 번째 socket 생성·fresh ticket을 통한 동일 경기 복구·command 활성화를 확인한다.

각 browser context는 finally에서 닫혀 session/socket ownership을 정리한다. 테스트는 demo mode의 실제 API, browser policy, transport client, GameHub recovery를 한 흐름으로 연결한다.

#### 이 커밋이 보장하는 것

- guest가 credential 없이 진입하되 등록 사용자 기능을 보지 않는다.
- guest PvP pool, bounded AI fallback, fresh-ticket reconnect가 browser에서 작동한다.
- 서로 다른 browser context의 identity/cookie가 격리된다.

#### 이 커밋이 보장하지 않는 것

- 실제 외부 네트워크 단절 패턴 전체나 장시간 load를 재현하지 않는다.
- Chromium 외 browser engine을 검증하지 않는다.
- CI job wiring 자체는 category 09 범위다.

#### 전후 관계

process smoke보다 높은 사용자 흐름 증거이며, auth/reconnect/matchmaking invariant를 browser boundary에서 결합한다.

#### 증거 성격

- multi-browser end-to-end recovery regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:1abda1299ad8:end -->


## Thread-level invariant evolution

다음 표에서 불변식이 도입·확장·교정·검증된 SHA를 구분합니다.

<!-- ANSWER:thread-06-invariants:begin -->
### 복원 결과

| SHA | 변화 | 불변식 |
| --- | --- | --- |
| `9a0562d395db` | 초기 HTTP process 경계 | 빌드된 API의 로그인·주요 read path가 외부 client에서 실패를 드러낸다. |
| `8a462f6f05b3` | 초기 realtime process 경계 | 두 socket이 same-room match→ready→playing snapshot→chat을 통과한다. |
| `d755b8dae2c1` | browser 경계 | desktop/mobile Chromium에서 한국어 navigation과 실제 Canvas paint가 검증된다. |
| `17e7dab21a55` | 계약 교정 | process tests가 bearer/query token을 버리고 cookie→one-time ticket→v1 protocol을 따른다. |
| `1abda1299ad8` | guest high-risk E2E | 제한된 guest UI, 두 browser PvP, bounded fallback, fresh-ticket reconnect가 결합 검증된다. |

각 변화는 이전 커밋의 최종 상태를 다음 커밋이 단순 반복한 것이 아니라, 새 경계를 도입·확장·교정하거나 regression으로 보호한 시점을 나타냅니다.
<!-- ANSWER:thread-06-invariants:end -->

## Failure → Fix → Test 관계

구현 추가를 독립 기능으로 나열하지 말고, 이전 가정과 실제 실패 위험, 수정된 결정, 회귀 증거를 연결합니다.

<!-- ANSWER:thread-06-failure-relations:begin -->
### 복원 결과

| Failure / 이전 가정 | Fix / 결정 | Verification |
| --- | --- | --- |
| unit test는 통과하지만 production process composition/listen/build가 깨질 수 있음 | 외부 HTTP/WS smoke | `9a0562...` + `8a462...` |
| 초기 smoke가 obsolete bearer/query credential로 보안 경계를 우회 | `17e7dab21a55` cookie/ticket/v1 migration | 동일 smoke의 no-JSON-token·admin 403·nested snapshot assertions |
| DOM에 Canvas만 있고 실제 draw가 없음 | pixel alpha 관찰 | `d755b8dae2c1` browser test |
| guest reconnect가 새 matchmaking intent를 만들거나 같은 room을 잃음 | fresh-ticket connection recovery | `1abda1299ad8` routed socket close/second connection |
<!-- ANSWER:thread-06-failure-relations:end -->

## Ownership·state·responsibility 변화

자원과 상태를 누가 만들고, 보유하고, 이전하고, 정리하는지 커밋 순서에 따라 정리합니다.

<!-- ANSWER:thread-06-ownership:begin -->
### 복원 결과

| 대상 | 소유자 | 책임·수명 |
| --- | --- | --- |
| HTTP process smoke | `tests/smoke-api.mjs` | 실행 중 API origin·cookie jar·핵심 request 결과를 소유한다. |
| WebSocket process smoke | `tests/smoke-ws.mjs` | 두 client의 ticket/socket/event timeout과 finally cleanup을 소유한다. |
| Browser runner | Playwright config | project viewport, base URL, timeout, trace/screenshot lifecycle을 소유한다. |
| Browser identity isolation | separate BrowserContext | guest cookie/session/socket 수명을 각 context에 격리한다. |
| Reconnect fault | `page.routeWebSocket` | 진행 중 connection을 의도적으로 닫고 두 번째 연결을 관찰한다. |
| Fallback timing evidence | WebSocket frame observer | `queue.join`/`queue.matched` timestamp 차이를 소유한다. |
<!-- ANSWER:thread-06-ownership:end -->

## 최종 Thread 상태

최종 HEAD를 과거 커밋에 투영하지 말고, 위 SHA 순서에서 도달한 보장을 요약합니다.

<!-- ANSWER:thread-06-final-state:begin -->
### 최종 상태

- process smoke는 실제 빌드된 API의 HTTP와 versioned WebSocket 경로를 cookie/ticket 인증으로 통과한다.
- browser E2E는 한국어 navigation, Canvas paint, desktop/mobile execution boundary를 제공한다.
- guest demo E2E는 capability 제한, 두 context PvP, 약 6초 AI fallback, 진행 중 socket 단절 후 fresh-ticket 복구를 사용자 흐름으로 검증한다.
- 초기 bearer/query smoke는 역사적 단계로 보존되지만 최종 보장으로 오해하지 않는다.
<!-- ANSWER:thread-06-final-state:end -->

## 최종 architecture / execution flow

실제 caller→boundary→state transition→cleanup 흐름을 순서대로 작성합니다.

<!-- ANSWER:thread-06-flow:begin -->
### 최종 실행 흐름

1. 실행 중 API/Web process readiness 확보
2. 브라우저/스크립트가 dev 또는 guest login으로 cookie 획득
3. HTTP read/action을 cookie로 호출
4. cookie로 one-time WebSocket ticket 발급
5. version-1 socket에서 queue/ready/input/chat/pause/reconnect 수행
6. smoke는 protocol/state를, Playwright는 UI/rendering/context lifecycle을 관찰
7. timeout/finally에서 socket·context 정리
<!-- ANSWER:thread-06-flow:end -->

## 학습 완료 확인

<!-- ANSWER:thread-06-checks:begin -->
### 완료 확인

- [x] Fastify injection과 실제 process smoke의 failure surface를 비교할 수 있다.
- [x] 초기 smoke의 bearer/query 방식을 최종 invariant로 서술하면 왜 역사 왜곡인지 설명할 수 있다.
- [x] Canvas pixel alpha가 무엇을 증명하고 무엇을 증명하지 않는지 설명할 수 있다.
- [x] guest fallback의 5.5~10초 범위와 reconnect second-connection 관찰 방식을 설명할 수 있다.
- [x] CI job 실행 여부와 테스트 구현 존재를 구분할 수 있다.

체크는 repository code inspection을 통해 설명이 문서에 작성되었음을 뜻합니다. 실제 repository test suite 실행 완료를 뜻하지 않습니다.
<!-- ANSWER:thread-06-checks:end -->

## 실행 증거 기록

<!-- ANSWER:thread-06-execution:begin -->
- Historical inspection: 각 참조 SHA의 GitHub commit diff와 변경 파일을 개별 조회했습니다.
- Repository test execution: 실행하지 않았습니다. 로컬 환경에서 지정 branch의 전체 checkout을 materialize하지 못해 pnpm, PostgreSQL, Playwright, k6, Toxiproxy 명령 결과를 만들 수 없었습니다.
- Runtime claims: 기록하지 않았습니다. 위의 `증거 성격`은 test 구현과 production code path에 대한 정적·역사적 검사입니다.
- Artifact validation: scaffold/completed 대응, fixed metadata, answer completion, SHA uniqueness, Markdown fence, ZIP 구조는 로컬 검증 스크립트로 검사했습니다.
<!-- ANSWER:thread-06-execution:end -->
===== END FILE: 06-process-smoke-and-browser-end-to-end-verification.md =====

===== BEGIN FILE: 07-benchmark-load-and-fault-recovery-verification.md =====
# Development Thread 07: benchmark·load·fault recovery 검증

English title: **Benchmark, Load, and Fault-Recovery Verification**

## Thread 목표

격리 benchmark, executable load SLO, Toxiproxy fault injection, 부하 중 snapshot cadence 수정, machine-readable recovery report가 서로 다른 증거 층으로 결합되는 과정을 복원한다.

## 핵심 질문

- scheduler microbenchmark와 실제 k6 service load는 어떤 변수를 각각 격리·포함하는가?
- measurement report에 runtime/CPU/settings를 포함해도 결과가 universal해지지 않는 이유는 무엇인가?
- 500 connections·50 rooms·99%·p95/p99·duplicate zero가 어떻게 executable contract가 되는가?
- DB와 edge fault path를 별도 proxy로 분리해야 원인과 recovery를 어떻게 구분할 수 있는가?
- 부하 병목을 simulation tick 저하가 아니라 20Hz state/10Hz staggered delivery로 수정한 이유는 무엇인가?
- fault runner의 always-reset와 loopback-only guard가 test harness 안전성에 왜 필수인가?

## 완료 기준

- microbenchmark, static harness contract, actual load capability, production fix, fault orchestration test를 구분한다.
- 모든 수치 threshold와 측정 metric을 concrete하게 설명한다.
- authoritative simulation cadence와 snapshot delivery cadence를 분리한다.
- fault scenario의 ordered state와 readiness 기대값·timeout·cleanup을 설명한다.
- 실제 load/fault 실행 결과가 없음을 명확히 구분한다.

## Commit map

| 순서 | Commit | Subject | Importance | Tags |
| --- | --- | --- | --- | --- |
| 1 | `aed88c8a93e0` | `perf(game): scheduler benchmark 실행 경계 추가` | B | REALTIME, OBSERVABILITY, PERF |
| 2 | `8d24b5e70837` | `perf(game): scheduler benchmark 측정 결과 출력` | B | REALTIME, OBSERVABILITY, PERF |
| 3 | `ff1bffcd5296` | `test(load): 실시간 부하 임계값 정의` | B | REALTIME, PERSISTENCE, OPERATIONS |
| 4 | `7b0b5f086b41` | `test(load): 실시간 fault injection 도구 추가` | A | AUTH, REALTIME, PERSISTENCE |
| 5 | `ad482c200cea` | `fix(game): 부하 중 snapshot cadence 안정화` | A | SIMULATION, REALTIME, PERF |
| 6 | `84bec3bf57ae` | `test(load): fault recovery 검사 자동화` | A | PERSISTENCE, OPERATIONS, PERF |
| 7 | `335565908920` | `test(load): fault scenario 설정과 report 검증` | B | PERSISTENCE, OPERATIONS, PERF |

> Commit 순서·subject·importance·tags는 Phase 1 감사 후 동결된 정보입니다. Phase 2에서는 변경하지 않습니다.

## Commit `aed88c8a93e0` — perf(game): scheduler benchmark 실행 경계 추가

- Full SHA: `aed88c8a93e0c9d987e80a23d5180b85f07e2b78`
- Subject: `perf(game): scheduler benchmark 실행 경계 추가`
- Importance: **B**
- Tags: REALTIME, OBSERVABILITY, PERF
- Source-defined role: room별 timer와 shared timer topology를 동일한 합성 room-step 작업·50ms cadence에서 비교하는 독립 benchmark 경계를 만든다.

### 정확한 조사 대상

- `tests/load/scheduler-benchmark.mjs`의 `TIMESTEP_MS`, `ROOM_COUNTS`, warmup, duration, repeats 설정을 확인한다.
- `room` 전략과 `shared` 전략이 같은 `simulateRoomStep` 함수를 수행하는지 확인한다.
- expected timestamp와 `performance.now()` 차이로 lag sample을 만드는 방식을 확인한다.
- 1, 20, 50, 100 room에서 p95/p99를 계산하기 위한 sample 수집·timer cleanup을 확인한다.

### 학습자 복원

<!-- ANSWER:aed88c8a93e0:begin -->
#### 역사적 복원

스케줄러 topology를 바꾸려면 실제 서비스 전체 부하보다 먼저 timer 수 자체의 영향을 격리할 필요가 있다. 이 script는 room별 `setInterval`과 하나의 shared `setInterval`을 비교하되 두 전략 모두 50ms cadence와 동일한 `simulateRoomStep` CPU work를 사용한다.

250ms warmup 뒤 expected fire time보다 늦은 정도를 sample로 모으고 room 수 1·20·50·100에서 p95/p99를 계산한다. duration 종료 시 생성한 모든 timer를 해제한다.

이 단계는 측정 경계를 만든 것이며 아직 결과 출력·선택 기준이 완성되지 않았다. 실제 PongSimulation이나 network broadcast를 수행하지 않으므로 topology 비교를 위한 microbenchmark다.

#### 이 커밋이 보장하는 것

- 두 scheduler topology를 같은 합성 작업과 cadence로 비교할 수 있다.
- room 수 증가에 따른 timer lag sample을 수집한다.

#### 이 커밋이 보장하지 않는 것

- 실제 GameHub·socket·DB 부하를 대표한다고 보장하지 않는다.
- 측정 환경과 선택 기준은 이 커밋만으로 기록되지 않는다.

#### 전후 관계

`8d24b5e70837`이 runtime metadata, 반복 median, 50-room 5% 선택 기준을 포함한 재현 가능한 report로 완성한다.

#### 증거 성격

- isolated scheduler microbenchmark boundary
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:aed88c8a93e0:end -->

## Commit `8d24b5e70837` — perf(game): scheduler benchmark 측정 결과 출력

- Full SHA: `8d24b5e70837afe3762622a29e802e40bc66071e`
- Subject: `perf(game): scheduler benchmark 측정 결과 출력`
- Importance: **B**
- Tags: REALTIME, OBSERVABILITY, PERF
- Source-defined role: scheduler benchmark를 runtime 환경·설정·반복 median·선택 결정을 담은 JSON report로 만든다.

### 정확한 조사 대상

- room count와 strategy별 `REPEATS` 실행 및 p95/p99 median 집계를 확인한다.
- Node version, platform/release, CPU model/count, total/free memory를 report에 포함하는지 확인한다.
- settings에 timestep, duration, warmup, repeats, room counts가 기록되는지 확인한다.
- 50-room shared p95가 room p95의 105% 이내일 때 shared를 선택하는 기준을 확인한다.
- stdout JSON이 후속 비교 가능한 구조인지 확인한다.

### 학습자 복원

<!-- ANSWER:8d24b5e70837:begin -->
#### 역사적 복원

이 커밋은 단일 실행 숫자만 출력하는 대신 strategy/room count별 반복 결과의 median을 계산하고 측정 환경과 설정을 함께 JSON으로 기록한다.

의사결정은 50-room에서 shared p95가 room-per-timer p95의 105% 이내인지로 명시된다. 따라서 selected strategy가 임의 인상이 아니라 저장 가능한 threshold 계산으로 나온다.

보고서가 환경 메타데이터를 포함해도 서로 다른 machine의 수치를 자동으로 동일하게 만들지는 않는다. 이는 해석 가능성을 높이는 것이지 universal benchmark 결과를 주장하는 것은 아니다.

#### 이 커밋이 보장하는 것

- benchmark 결과와 실행 환경·설정·선택 기준이 한 JSON artifact로 결합된다.
- 반복 run의 일회성 outlier를 median으로 완화한다.

#### 이 커밋이 보장하지 않는 것

- 이 커밋 자체가 특정 machine에서 실제 실행된 결과를 repository에 고정하지 않는다.
- 5% 기준이 사용자 latency SLO를 직접 의미하지 않는다.

#### 전후 관계

`aed88c8a93e0`의 측정 경계를 review 가능한 decision report로 완성한다.

#### 증거 성격

- reproducible benchmark reporting contract
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:8d24b5e70837:end -->

## Commit `ff1bffcd5296` — test(load): 실시간 부하 임계값 정의

- Full SHA: `ff1bffcd5296ed89fca6f42b070f6ecac4c0eb65`
- Subject: `test(load): 실시간 부하 임계값 정의`
- Importance: **B**
- Tags: REALTIME, PERSISTENCE, OPERATIONS
- Source-defined role: k6/Toxiproxy load harness의 기본 규모, metric 집합, SLO threshold, overlay wiring을 정적 실행 테스트로 고정한다.

### 정확한 조사 대상

- `tests/load/load-harness.test.mjs`에서 default 500 connections, 50 rooms, 100 player connections를 확인한다.
- 최소 성공 연결 495, optional 1,000 connection profile, connections >= rooms*2 guard를 확인한다.
- connection/reconnect 99%, snapshot p95 150ms·p99 250ms, drop <1%, finalize failure/duplicate 0 threshold를 확인한다.
- k6 source에 required metric과 dev-login→ticket→v1 queue/input 경로가 실제로 있는지 source contract를 확인한다.
- Toxiproxy PostgreSQL/edge plan과 Compose route가 별도 proxy를 통과하는지 확인한다.

### 학습자 복원

<!-- ANSWER:ff1bffcd5296:begin -->
#### 역사적 복원

load script가 존재하는 것만으로는 어떤 규모와 실패 기준이 release evidence인지 알 수 없다. 이 테스트는 default profile을 500 연결·50 active room·100 player connection으로 고정하고 99% 이상 연결 성공을 요구한다.

metric threshold는 reconnect success 99%, snapshot delay p95 150ms/p99 250ms, normal snapshot drop rate 1% 미만, finalization failure/duplicate 0, 결과 50개 이상, online 495 이상, active room 50 이상이다. 1,000 연결은 `EXTENDED_LOAD=1` 또는 명시 설정에서만 선택된다.

테스트는 profile object만 비교하지 않고 k6 source에 ticket issuance, v1 queue/input, inputSeq/serverTimeMs metric 경로가 있는지 확인하고, Toxiproxy 계획과 Compose overlay가 DB와 public edge를 별도 proxy로 라우팅하는지 검사한다.

이는 harness contract를 검증하지만 실제 k6 run의 SLO 통과 결과는 아니다.

#### 이 커밋이 보장하는 것

- 부하 규모·metric·threshold·fault overlay가 의도와 다르게 drift하는 것을 막는다.
- 연결 수와 room 수가 논리적으로 맞지 않는 profile을 사전에 거부한다.

#### 이 커밋이 보장하지 않는 것

- 500/1,000 실제 연결을 이 단위 테스트에서 열지 않는다.
- 환경별 capacity나 장기 soak stability를 증명하지 않는다.

#### 전후 관계

`7b0b5f086b41`의 load/fault harness를 executable contract로 고정하고, `ad482c200cea`의 cadence fix를 평가할 기준을 제공한다.

#### 증거 성격

- static/executable load-harness contract test
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:ff1bffcd5296:end -->

## Commit `7b0b5f086b41` — test(load): 실시간 fault injection 도구 추가

- Full SHA: `7b0b5f086b4193b9585e5a8bf6e55ec6e52fb447`
- Subject: `test(load): 실시간 fault injection 도구 추가`
- Importance: **A**
- Tags: AUTH, REALTIME, PERSISTENCE
- Source-defined role: k6 realtime load와 Toxiproxy DB/edge fault 주입을 결합해 연결·재연결·snapshot·finalization invariant를 운영 경계에서 측정할 수 있게 한다.

### 정확한 조사 대상

- `docker-compose.load.yml`의 Toxiproxy service, bootstrap, API DATABASE_URL, edge port wiring을 확인한다.
- `tests/load/load-profile.mjs`의 profile/threshold/hold/reconnect timing을 확인한다.
- `pong-load.js`에서 dev-login cookie, one-time ticket, version-1 WebSocket, queue/ready/input sequence를 추적한다.
- snapshot `serverTimeMs` delay, sequence gap drop rate, online/active room metric 수집을 확인한다.
- finished result의 persisted flag, matchId/roomId, duplicate set 검사와 reconnect same-room recovery를 확인한다.
- `toxiproxy-control.mjs`의 postgres/edge proxy와 latency/down/reset/up command, argument validation을 확인한다.

### 학습자 복원

<!-- ANSWER:7b0b5f086b41:begin -->
#### 역사적 복원

unit·process smoke는 하나 또는 몇 개 연결의 correctness를 보여 주지만 수백 connection, 다수 room, reconnect storm, DB/edge degradation에서 invariant가 유지되는지는 측정하지 못한다. 이 커밋은 별도 load runtime을 정의한다.

k6 VU는 dev-login cookie로 one-time ticket을 발급하고 version-1 socket을 연다. player VU는 queue에 들어가 ready/inputSeq를 보내고 initial connection을 닫은 뒤 새 ticket으로 같은 room에 복구한다. snapshot의 `serverTimeMs`와 sequence gap으로 delay/drop을 기록한다.

완료 결과는 left side에서 `persisted: true`, 유효한 matchId, 기대 roomId를 요구하고 per-VU set으로 duplicate match/room 결과를 센다. connection/reconnect/online/active-room/finalization metric은 threshold 평가에 사용된다.

Toxiproxy는 PostgreSQL과 edge를 별도 proxy로 만들며 API DB traffic과 public edge를 overlay를 통해 통과시킨다. control script는 latency, down/up, peer reset을 명시적 command로 제공하고 잘못된 숫자를 거부한다.

이 도구는 fault를 주입할 능력을 제공하지만 recovery 절차와 report를 자동으로 순서화하는 것은 `84bec3bf57ae`에서 완성된다.

#### 이 커밋이 보장하는 것

- 대규모 realtime traffic에서 auth ticket, same-room reconnect, snapshot cadence, single finalization을 측정할 수 있다.
- DB와 edge fault path가 분리되어 원인별 degradation을 주입할 수 있다.
- load profile과 fault topology가 versioned protocol/credential 경계를 우회하지 않는다.

#### 이 커밋이 보장하지 않는 것

- repository에 기록된 실제 부하 실행 결과가 아니다.
- k6 VU model이 실제 browser rendering 비용을 포함하지 않는다.
- process crash·multi-region failure까지 주입하지 않는다.

#### 전후 관계

`ff1bffcd5296`이 harness contract를 고정하고, `ad482c200cea`가 부하 병목으로 드러난 snapshot burst를 수정하며, `84bec3bf57ae`가 fault-recovery sequence를 자동화한다.

#### 증거 성격

- load and network/database fault-injection architecture
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:7b0b5f086b41:end -->

## Commit `ad482c200cea` — fix(game): 부하 중 snapshot cadence 안정화

- Full SHA: `ad482c200cea41429c48419fc564069751505d5c`
- Subject: `fix(game): 부하 중 snapshot cadence 안정화`
- Importance: **A**
- Tags: SIMULATION, REALTIME, PERF
- Source-defined role: authoritative simulation은 20Hz로 유지하면서 snapshot 전송은 room별 교대 slot을 가진 10Hz로 줄여 동시 burst를 분산한다.

### 정확한 조사 대상

- `apps/api/src/gameHub.ts`의 `SIMULATION_TIMESTEP_MS`, `SNAPSHOT_DELIVERY_DIVISOR`, `snapshotDeliverySlot`을 확인한다.
- 새 room이 0/1 slot을 round-robin으로 배정받는지 확인한다.
- simulation `step`과 snapshot sync는 매 50ms 수행되지만 broadcast는 `(tick + slot) % 2 === 0`일 때만 실행되는지 확인한다.
- room authoritative state와 client delivery cadence가 분리되었는지 확인한다.
- finalization observer의 `created` metadata와 duplicate metric 연결도 같은 diff에서 확인한다.

### 학습자 복원

<!-- ANSWER:ad482c200cea:begin -->
#### 역사적 복원

부하 profile에서 room별 20Hz snapshot을 같은 tick에 전송하면 simulation 자체보다 serialization/send burst가 event loop와 transport buffer를 압박할 수 있었다. 단순히 simulation tick을 낮추면 게임 규칙과 입력 반응을 바꾸므로 올바른 수정이 아니다.

수정은 simulation timestep 50ms, 즉 20Hz를 유지한다. 각 room은 생성 시 0 또는 1의 `snapshotDeliverySlot`을 round-robin으로 받고, snapshot broadcast는 divisor 2 조건에서만 수행해 10Hz가 된다. slot이 번갈아 있으므로 여러 room의 전송이 같은 tick에 몰리지 않는다.

authoritative state는 매 tick 갱신되고 client는 더 낮은 cadence의 최신 projection을 받는다. 같은 커밋은 finalization observer에 `created`를 전달해 DB가 기존 idempotent 결과를 반환한 경우 duplicate metric으로 관찰할 수 있게 한다.

핵심은 simulation correctness와 delivery capacity를 별도 조절하는 것이다.

#### 이 커밋이 보장하는 것

- authoritative mechanics와 input processing은 20Hz를 유지한다.
- snapshot delivery는 10Hz로 bounded되고 room slot으로 burst가 분산된다.
- idempotent existing-result 반환이 별도 metric으로 관찰된다.

#### 이 커밋이 보장하지 않는 것

- 모든 환경에서 p95/p99 threshold를 자동으로 통과한다고 보장하지 않는다.
- 10Hz가 모든 클라이언트 UX에 최적이라는 일반 결론은 아니다.

#### 전후 관계

load harness가 드러낸 병목을 timing ownership을 깨지 않고 수정한다. 후속 load regression은 20Hz simulation/10Hz staggered delivery를 함께 확인한다.

#### 증거 성격

- load-induced cadence correction
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:ad482c200cea:end -->

## Commit `84bec3bf57ae` — test(load): fault recovery 검사 자동화

- Full SHA: `84bec3bf57ae15af185ed17ce935008438461758`
- Subject: `test(load): fault recovery 검사 자동화`
- Importance: **A**
- Tags: PERSISTENCE, OPERATIONS, PERF
- Source-defined role: Toxiproxy command와 readiness polling을 순서화하여 baseline→degradation→outage/reset→recovery를 versioned JSON report로 자동 검증한다.

### 정확한 조사 대상

- `tests/load/fault-scenario.mjs`의 `createFaultScenarioConfig`, `runFaultScenario`, `observeStep`을 확인한다.
- control/API/edge URL이 HTTP loopback으로 제한되는지 확인한다.
- baseline, DB latency 300ms, DB down, DB up/recovery, edge latency 150ms, edge reset, edge recovery 순서를 확인한다.
- DB down이 HTTP 503 + `not_ready` + `checks.database=down`으로 관찰되는지 확인한다.
- request/recovery timeout과 poll interval, JSON `schemaVersion: 1` report를 확인한다.
- 성공·실패 어느 경우에도 마지막 `reset`이 실행되고 cleanup error가 원래 error와 어떻게 결합되는지 확인한다.

### 학습자 복원

<!-- ANSWER:84bec3bf57ae:begin -->
#### 역사적 복원

Toxiproxy control command가 있어도 사람이 수동으로 fault를 넣고 readiness를 눈으로 확인하면 순서·timeout·cleanup·증거 형식이 재현되지 않는다. 이 커밋은 fault scenario 자체를 실행 가능한 state machine으로 만든다.

설정은 control, API readiness, edge readiness URL을 loopback HTTP로 제한해 우발적으로 외부 target을 변조하지 못하게 한다. 기본 DB latency 300ms, edge latency 150ms, request timeout 5초, recovery timeout 15초, polling 250ms가 명시된다.

runner는 모든 proxy를 reset하고 baseline ready를 확인한 뒤 DB latency에서도 ready, DB down에서는 503/not_ready/database down, DB up 후 ready를 기다린다. 선택적으로 edge latency, peer reset으로 network failure, edge up 후 recovery를 관찰한다. 각 단계는 status, duration, body/error를 report에 남긴다.

try/catch 뒤 별도 cleanup block에서 항상 proxy reset을 시도한다. scenario와 cleanup이 모두 실패하면 원래 오류를 유지하면서 cleanup error를 cause로 보존한다. 성공 report는 schemaVersion 1, settings, target, ordered steps, timestamps를 가진다.

#### 이 커밋이 보장하는 것

- DB/edge degradation과 readiness failure/recovery가 bounded polling으로 검증된다.
- fault target은 loopback으로 제한되고 proxy state는 성공·실패 뒤 reset된다.
- 결과가 versioned machine-readable report로 남는다.

#### 이 커밋이 보장하지 않는 것

- 실제 production network나 원격 host에 fault를 주입하지 않는다.
- DB가 down인 동안 in-flight business transaction의 모든 결과를 검증하지 않는다.
- 이 커밋 자체에 실제 실행 report가 포함된 것은 아니다.

#### 전후 관계

`335565908920`이 dependency injection으로 command/probe/sleep 순서와 cleanup을 deterministic하게 검증한다.

#### 증거 성격

- automated operational fault-recovery scenario
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:84bec3bf57ae:end -->

## Commit `335565908920` — test(load): fault scenario 설정과 report 검증

- Full SHA: `335565908920371a20f939ae27df5bcd985792c9`
- Subject: `test(load): fault scenario 설정과 report 검증`
- Importance: **B**
- Tags: PERSISTENCE, OPERATIONS, PERF
- Source-defined role: fault runner의 설정 안전성, 명령 순서, polling, JSON report, 실패 cleanup을 외부 Toxiproxy 없이 결정적으로 검증한다.

### 정확한 조사 대상

- `tests/load/fault-scenario.test.mjs`의 dependency injection 지점을 확인한다.
- 기본 loopback URL·latency·timeout과 non-loopback 거부 사례를 확인한다.
- 가짜 probe sequence가 ready→latency ready→DB down→recovery→edge reset→recovery를 만드는지 확인한다.
- 명령 배열과 sleep 배열이 정확한 순서/횟수인지 확인한다.
- report의 schemaVersion/timestamp/targets/steps/duration/error가 기대와 일치하는지 확인한다.
- `db-down` control 실패 뒤에도 마지막 `reset`이 호출되는지 확인한다.

### 학습자 복원

<!-- ANSWER:335565908920:begin -->
#### 역사적 복원

외부 Toxiproxy를 매번 띄우지 않고도 runner의 제어 논리를 검증하기 위해 command, readiness probe, sleep, clock을 주입한다. 가짜 probe 결과는 DB down 전 두 번의 not-ready polling과 edge socket reset을 포함한다.

테스트는 `reset → db-latency → db-down → db-up → edge-latency → edge-reset → edge-up → reset` 명령 순서와 두 번의 250ms sleep을 정확히 비교한다. report는 시작/종료 시각, target, settings, ordered step, latency duration, DB down body, socket-reset error를 보존해야 한다.

별도 실패 사례에서 `db-down` command가 예외를 던져도 마지막 reset이 호출되는지 확인한다. 또한 non-loopback URL은 runner가 어떤 command도 수행하기 전에 거부된다.

#### 이 커밋이 보장하는 것

- fault orchestration과 report 형식이 외부 환경 없이 결정적으로 회귀 검증된다.
- 중간 control failure에서도 cleanup reset이 보장된다.

#### 이 커밋이 보장하지 않는 것

- 실제 Toxiproxy·PostgreSQL·edge process가 예상 상태로 변하는지는 이 단위 테스트가 증명하지 않는다.
- 실제 recovery duration 수치를 제공하지 않는다.

#### 전후 관계

`84bec3bf57ae`의 operational runner를 설정/제어/cleanup contract 수준에서 고정한다.

#### 증거 성격

- deterministic orchestration and cleanup regression
- 저장소 코드·diff 검사로 확인했습니다. 이 workbook 작성 환경에서는 해당 SHA의 repository test command를 실행하지 않았습니다.
<!-- ANSWER:335565908920:end -->


## Thread-level invariant evolution

다음 표에서 불변식이 도입·확장·교정·검증된 SHA를 구분합니다.

<!-- ANSWER:thread-07-invariants:begin -->
### 복원 결과

| SHA | 변화 | 불변식 |
| --- | --- | --- |
| `aed88c8a93e0` | benchmark 경계 | room timer와 shared timer를 동일 50ms 합성 work에서 비교한다. |
| `8d24b5e70837` | 측정 보고 | 반복 median, runtime metadata, 50-room 5% decision rule이 JSON으로 기록된다. |
| `ff1bffcd5296` | load 계약 | 500 connections·50 rooms와 연결/재연결/snapshot/finalization threshold가 executable profile이 된다. |
| `7b0b5f086b41` | fault/load 도구 | k6가 ticket/v1/reconnect/snapshot/finalization을 측정하고 Toxiproxy가 DB/edge를 분리한다. |
| `ad482c200cea` | 부하 수정 | simulation 20Hz를 유지하면서 snapshot 10Hz와 교대 slot으로 burst를 분산한다. |
| `84bec3bf57ae` | 복구 자동화 | baseline→DB latency/down/up→edge latency/reset/up가 bounded readiness polling과 versioned report가 된다. |
| `335565908920` | orchestrator 회귀 | loopback guard, command/probe/sleep 순서, report와 실패 후 reset이 결정적으로 검증된다. |

각 변화는 이전 커밋의 최종 상태를 다음 커밋이 단순 반복한 것이 아니라, 새 경계를 도입·확장·교정하거나 regression으로 보호한 시점을 나타냅니다.
<!-- ANSWER:thread-07-invariants:end -->

## Failure → Fix → Test 관계

구현 추가를 독립 기능으로 나열하지 말고, 이전 가정과 실제 실패 위험, 수정된 결정, 회귀 증거를 연결합니다.

<!-- ANSWER:thread-07-failure-relations:begin -->
### 복원 결과

| Failure / 이전 가정 | Fix / 결정 | Verification |
| --- | --- | --- |
| timer topology 선택이 인상·단일 숫자에 의존 | 동일 work benchmark + environment/report + 5% rule | `aed88...` → `8d24...` |
| load script 존재하지만 규모·SLO·metric drift | executable profile/threshold/source contract | `ff1bffcd5296` |
| 20Hz snapshot burst가 event loop/transport를 압박 | `ad482c200cea` — 20Hz simulation 유지, 10Hz staggered delivery | 후속 load regression과 metric 관찰 |
| 수동 fault 주입이 순서·cleanup·증거를 보장하지 못함 | `84bec3bf57ae` automated runner | `335565908920` dependency-injected regression |
<!-- ANSWER:thread-07-failure-relations:end -->

## Ownership·state·responsibility 변화

자원과 상태를 누가 만들고, 보유하고, 이전하고, 정리하는지 커밋 순서에 따라 정리합니다.

<!-- ANSWER:thread-07-ownership:begin -->
### 복원 결과

| 대상 | 소유자 | 책임·수명 |
| --- | --- | --- |
| Topology microbenchmark | `scheduler-benchmark.mjs` | 합성 work, timer topology, lag sample과 environment report를 소유한다. |
| Load profile | `load-profile.mjs` | connections, rooms, hold/reconnect timing, SLO threshold를 소유한다. |
| Realtime load client | `pong-load.js` | cookie/ticket/v1 socket, queue/input/reconnect, metric observation을 소유한다. |
| Fault topology | Toxiproxy + Compose overlay | PostgreSQL·edge의 별도 degradation 경로를 소유한다. |
| Delivery cadence | GameHub room `snapshotDeliverySlot` | 20Hz authoritative state와 10Hz staggered projection을 분리한다. |
| Fault orchestration | `fault-scenario.mjs` | ordered commands, readiness expectations, polling, report, final reset을 소유한다. |
| Harness regression | `fault-scenario.test.mjs` | 외부 dependency 없이 orchestration contract를 결정적으로 보호한다. |
<!-- ANSWER:thread-07-ownership:end -->

## 최종 Thread 상태

최종 HEAD를 과거 커밋에 투영하지 말고, 위 SHA 순서에서 도달한 보장을 요약합니다.

<!-- ANSWER:thread-07-final-state:begin -->
### 최종 상태

- scheduler topology는 격리 benchmark와 환경 metadata가 있는 JSON decision report로 비교된다.
- load harness는 500 connection·50 room 기본 규모와 구체적 connection/reconnect/snapshot/finalization threshold를 가진다.
- k6와 Toxiproxy는 최종 auth/protocol을 우회하지 않고 DB/edge degradation을 별도로 주입할 수 있다.
- 부하 대응은 simulation authority를 20Hz로 유지하면서 snapshot 전달만 10Hz로 낮추고 room별 burst를 분산한다.
- fault recovery는 loopback-only, bounded polling, always-reset, versioned JSON report로 자동화되며 orchestration 자체도 단위 회귀를 가진다.
<!-- ANSWER:thread-07-final-state:end -->

## 최종 architecture / execution flow

실제 caller→boundary→state transition→cleanup 흐름을 순서대로 작성합니다.

<!-- ANSWER:thread-07-flow:begin -->
### 최종 실행 흐름

1. microbenchmark로 room/shared timer topology의 lag 비교
2. load profile에서 connections/rooms/SLO 확정
3. k6가 cookie→ticket→v1 socket으로 500 connections·50 rooms 구동
4. snapshot delay/drop, reconnect, online/rooms, finalization duplicate/failure 측정
5. Toxiproxy로 DB latency/down 및 edge latency/reset 주입
6. readiness에서 failure/recovery state bounded polling
7. GameHub는 20Hz simulation·10Hz staggered snapshots로 delivery burst 완화
8. runner가 schemaVersion 1 JSON report 생성 후 항상 proxy reset
<!-- ANSWER:thread-07-flow:end -->

## 학습 완료 확인

<!-- ANSWER:thread-07-checks:begin -->
### 완료 확인

- [x] microbenchmark 수치와 actual service SLO를 동일시하면 안 되는 이유를 설명할 수 있다.
- [x] 500/50 profile에서 player connection이 100이고 최소 성공 연결이 495인 계산을 설명할 수 있다.
- [x] snapshot p95 150ms·p99 250ms·drop 1% 미만·final duplicate 0을 열거할 수 있다.
- [x] 20Hz simulation과 10Hz delivery가 모순되지 않는 state projection 흐름을 설명할 수 있다.
- [x] fault runner의 non-loopback 거부와 finally reset이 어떤 위험을 줄이는지 설명할 수 있다.
- [x] 이 workbook이 실제 load pass 결과를 기록하지 않는 이유를 설명할 수 있다.

체크는 repository code inspection을 통해 설명이 문서에 작성되었음을 뜻합니다. 실제 repository test suite 실행 완료를 뜻하지 않습니다.
<!-- ANSWER:thread-07-checks:end -->

## 실행 증거 기록

<!-- ANSWER:thread-07-execution:begin -->
- Historical inspection: 각 참조 SHA의 GitHub commit diff와 변경 파일을 개별 조회했습니다.
- Repository test execution: 실행하지 않았습니다. 로컬 환경에서 지정 branch의 전체 checkout을 materialize하지 못해 pnpm, PostgreSQL, Playwright, k6, Toxiproxy 명령 결과를 만들 수 없었습니다.
- Runtime claims: 기록하지 않았습니다. 위의 `증거 성격`은 test 구현과 production code path에 대한 정적·역사적 검사입니다.
- Artifact validation: scaffold/completed 대응, fixed metadata, answer completion, SHA uniqueness, Markdown fence, ZIP 구조는 로컬 검증 스크립트로 검사했습니다.
<!-- ANSWER:thread-07-execution:end -->
===== END FILE: 07-benchmark-load-and-fault-recovery-verification.md =====

===== BEGIN FILE: README.md =====
# 08 — Verification and Test Architecture

Repository: `seungwoo7050/42-archive`  
Branch: `web/ft_transcendence`  
Category: `08-verification-and-test-architecture`

## Category boundary

이 category는 **제품 동작을 검증하는 test architecture와 evidence mechanism**을 다룹니다.

포함 범위:

- 공유 HTTP/WebSocket runtime contract와 실제 route 적용 검증
- cookie·one-time ticket·error redaction·transport limit 검증
- 결정적 simulation, fixed-step timing, replay, snapshot backpressure 검증
- GameHub room/reconnect/matchmaking/rollback/finalization-retry 검증
- 실제 PostgreSQL의 migration·transaction·constraint·concurrency·failure injection
- 실행 중 process smoke와 browser end-to-end evidence
- scheduler benchmark, k6 load contract, Toxiproxy fault/recovery report

제외 범위:

- production artifact 생성, Dockerfile/Compose 배포 계약, CI workflow wiring은 category 09의 delivery/release boundary입니다.
- runtime health/metric 구현 자체는 category 07의 observability/service-health boundary입니다.
- browser state/query-cache architecture 자체는 category 06에 남기고, 여기서는 실행 evidence만 다룹니다.
- core simulation·GameHub·auth·persistence 기능의 구현 서사는 각 기능 category에 남기고, 여기서는 해당 invariant를 검증하는 구조와 필요한 fix/test 관계만 추적합니다.

## Phase 1 audit result

초기 draft는 2개 Thread와 31개 참조 SHA로 구성되어 있었습니다. 실제 branch history와 source classification을 다시 대조한 결과 다음 문제가 확인되었습니다.

- deterministic unit tests, PostgreSQL concurrency, process/browser evidence, load/fault evidence가 두 문서에 과도하게 결합되어 있었습니다.
- strict HTTP route 적용, protocol negative regression, auth ticket concurrency, transport hard limit, migration 보존, audit atomicity, GameHub rollback·retry가 누락되거나 구체성이 부족했습니다.
- initial smoke의 bearer/query credential을 최종 auth evidence처럼 읽을 위험이 있었으며, cookie/ticket/v1 교정 관계가 분리되어 있지 않았습니다.
- generic 조사 지시가 exact file/function/test/failure path를 지정하지 않았습니다.

Phase 1에서 이를 7개 독립 engineering story와 46개 SHA로 재구성했습니다. 단순 테스트 개수 확장이 아니라 다음 경계를 기준으로 분리했습니다.

1. executable wire/HTTP contract
2. auth trust handoff와 transport containment
3. deterministic simulation/time/delivery primitive
4. GameHub lifecycle와 recovery ownership
5. PostgreSQL engine-level verification
6. running process와 real browser evidence
7. benchmark/load/fault recovery evidence

모든 참조 SHA는 branch의 433-commit linear classification 범위에 존재하는 항목과 exact commit 조회를 대조했습니다. Commit subject, importance, tags는 `commit/commit-importance.md` 값을 유지했습니다.

## Frozen Thread index

| No. | File | Thread | Commits | Historical range |
| --- | --- | --- | --- | --- |
| 01 | [01-executable-protocol-and-http-contract-verification.md](01-executable-protocol-and-http-contract-verification.md) | 실행 가능한 프로토콜·HTTP 계약 검증 | 6 | `60c38090effc` → `1abbf7dcdde4` |
| 02 | [02-authentication-ticket-failure-containment-and-transport-limits.md](02-authentication-ticket-failure-containment-and-transport-limits.md) | 인증·ticket·실패 격리·transport 상한 검증 | 8 | `d0531791406b` → `1afec49052b6` |
| 03 | [03-deterministic-simulation-timing-and-snapshot-delivery-verification.md](03-deterministic-simulation-timing-and-snapshot-delivery-verification.md) | 결정적 simulation·시간·snapshot 전달 검증 | 6 | `4ef4beeb8611` → `5cd54767858f` |
| 04 | [04-gamehub-lifecycle-reconnect-matchmaking-and-finalization-recovery.md](04-gamehub-lifecycle-reconnect-matchmaking-and-finalization-recovery.md) | GameHub lifecycle·재연결·매칭·결과 저장 복구 검증 | 7 | `4026c3bf72ad` → `8f5b2e86f69b` |
| 05 | [05-postgresql-integration-concurrency-migration-and-failure-injection.md](05-postgresql-integration-concurrency-migration-and-failure-injection.md) | PostgreSQL 통합·동시성·migration·실패 주입 검증 | 7 | `c43b87694b29` → `9106abc10d0e` |
| 06 | [06-process-smoke-and-browser-end-to-end-verification.md](06-process-smoke-and-browser-end-to-end-verification.md) | process smoke·browser end-to-end 검증 | 5 | `9a0562d395db` → `1abda1299ad8` |
| 07 | [07-benchmark-load-and-fault-recovery-verification.md](07-benchmark-load-and-fault-recovery-verification.md) | benchmark·load·fault recovery 검증 | 7 | `aed88c8a93e0` → `335565908920` |

## Commit profile

- Total referenced commits: **46**
- Unique full SHA: **46**
- Importance: **S 2 / A 28 / B 16 / C 0**
- S-level: cookie-only session boundary와 one-time WebSocket ticket handoff
- A-level: 고위험 contract, concurrency, rollback, recovery, resource-limit, deterministic evidence
- B-level: 구체적 단위·smoke·browser·harness 계약과 supporting evidence

## Source-of-truth inspection

Phase 1/2에서 다음 branch-local 자료만 사용했습니다.

- `commit/commit-importance.md`
- `commit/commit-bodies.md`
- `development-thread-workbook/COVERAGE.md`
- 초기 category scaffold 2개
- 각 참조 commit의 exact SHA diff와 해당 시점 변경 파일

다른 branch의 코드·테스트·문서·build logic은 사용하지 않았습니다. Final HEAD 구현을 이전 SHA 설명에 역투영하지 않았습니다.

## Phase 1 freeze

Frozen thread-document set digest:

`53bf51050e35afbebded2bf0e192786d733caf3521cc9d41ccb015befbe4213e`

Digest 입력은 아래 7개 Thread Markdown의 정렬된 `filename:sha256` 목록입니다. README는 digest를 기록하므로 digest 입력에서 제외합니다.

| Frozen Thread file | SHA-256 |
| --- | --- |
| `01-executable-protocol-and-http-contract-verification.md` | `1e94322035cf842a5f94b6f8fb24333e5bd2dc62ddde4501f884df644cb6865b` |
| `02-authentication-ticket-failure-containment-and-transport-limits.md` | `a93b2211242fe80510fe842fcea5dfccc41c62489a73a443e8c5e9c765fac47b` |
| `03-deterministic-simulation-timing-and-snapshot-delivery-verification.md` | `0fbc31fd6061420ecc04bd08ae76dbd647a0433847fe793164117bf54b43f659` |
| `04-gamehub-lifecycle-reconnect-matchmaking-and-finalization-recovery.md` | `beb74a01ddfeec881aa6d53caa7f88fc844feedb7f1f670a4481916a395ecdd0` |
| `05-postgresql-integration-concurrency-migration-and-failure-injection.md` | `058b3460fdc91a5f46c6644a6814defe98bd388005dcccaadf4bf4fd13a39385` |
| `06-process-smoke-and-browser-end-to-end-verification.md` | `63c515fe3438d3c44821e05a12195f311eefb89d3921a37dc3759d1b9435b123` |
| `07-benchmark-load-and-fault-recovery-verification.md` | `4072bcce5acda1f463aa78af65590a4dfcab1ff3c5847f9f3665e003b16901ba` |

Phase 2에서는 이 scaffold의 answer block 내부만 completed 내용으로 채웁니다. Commit map, fixed prompts, source roles, filenames, structure는 변경하지 않습니다.

## Workbook completion and validation

<!-- ANSWER:readme-completion:begin -->
## Phase 2 완료 결과

- Frozen scaffold Thread: 7
- Completed counterpart: 7
- Referenced commits: 46개, 중복 없음
- Importance distribution: S 2 / A 28 / B 16 / C 0
- Relative file set: scaffold와 completed가 정확히 일치
- Fixed commit metadata: SHA, subject, order, importance, tags 일치
- Unfinished answer block: 0
- Frozen scaffold integrity: Phase 2 전후 SHA-256 manifest 일치
- Repository runtime tests: 실행하지 않음
- Artifact-only validation: 실행함
- Packaging: `08-verification-and-test-architecture/scaffold/`와 `completed/`만 포함

### 실행하지 않은 검증

지정 branch의 전체 checkout을 로컬에 materialize하지 못했으므로 다음 명령군은 실행하지 않았습니다.

- `pnpm` unit/integration/smoke test
- Testcontainers PostgreSQL integration
- Playwright browser E2E
- k6 load test
- Toxiproxy fault scenario
- scheduler benchmark

따라서 completed 문서의 test 결과 설명은 각 exact SHA의 test code와 production diff가 구성한 검증 메커니즘에 대한 역사적 검사이며, 실제 pass 기록이 아닙니다.

### 실제로 실행한 artifact 검증

- scaffold/completed 상대 경로 집합 비교
- answer block을 제외한 고정 텍스트 byte comparison
- commit SHA/subject/importance/tags/order comparison
- 46개 full SHA 형식·short SHA uniqueness 검사
- completed placeholder/미완료 marker 검사
- Markdown code fence 균형 검사
- scaffold Phase 2 전후 SHA-256 manifest 비교
- ZIP member allowlist·top-level 구조·CRC 검사
<!-- ANSWER:readme-completion:end -->
===== END FILE: README.md =====

