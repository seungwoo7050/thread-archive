===== BEGIN FILE: 01-server-session-logout-and-auth-migrations.md =====
# Server session·logout·인증 migration

- 카테고리: `03-identity-authorization-and-account-lifecycle` — 신원·권한·계정 수명주기
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

등록 사용자 session이 repository에 처음 저장되는 시점부터 HTTP login/logout에 연결되고, client-side cookie 삭제만으로는 끝나지 않던 logout이 server-side revocation으로 수정된 뒤, 인증 모델 변경 시 기존 session만 안전하게 만료하는 migration으로 마무리되는 과정을 복원합니다.

이 문서는 exact SHA를 순서대로 확인해 설계 → 구현 → 실패 → 수정 → 검증을 복원합니다.

### 직접 연결되는 불변식

- 등록 사용자 session의 생성·조회·폐기는 `AppRepository`가 소유합니다.
- logout 이후 동일한 server session credential은 다시 인증에 성공하지 않아야 합니다.
- 인증 migration은 legacy session을 폐기하되 사용자·경기·rating history를 변경하지 않아야 합니다.

## 2. 핵심 질문

- PostgreSQL과 memory repository가 session identity를 각각 어디에 저장하며 만료를 어떻게 판단합니까?
- 초기 logout이 cookie만 지웠을 때 bearer/query credential이 계속 유효했던 이유는 무엇입니까?
- `readSessionToken`과 `deleteSession`이 어떤 순서로 호출되며 DB 삭제 실패 시 response는 어디까지 진행됩니까?
- 005 migration이 왜 session 전체 삭제를 선택했고, 보존 대상 테이블은 테스트에서 어떻게 고정됩니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 지정 브랜치 ancestry에 속하는지 확인합니다.
- 각 SHA의 diff와 parent 또는 직전 관련 SHA의 실제 state를 구분합니다.
- 아래 commit-specific 파일·함수·SQL·테스트를 확인하고 generic 설명으로 대체하지 않습니다.
- Fix는 이전 가정 → 실제 실패 → root cause → corrected invariant → regression evidence로 연결합니다.
- 실행하지 않은 test/command는 실행한 것처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread의 최종 owner, state transition, guarantee와 non-guarantee를 정리합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `4f65c6214321` | `feat(db): 사용자 session 저장 구현` | B | AUTH, REALTIME, PERSISTENCE | session creation/resolution을 repository abstraction에 추가합니다. |
| 2 | `1779df300611` | `feat(api): 로그인과 로비 HTTP 경계 구현` | B | AUTH, PERSISTENCE, WEB | repository-backed identity와 초기 HTTP login/logout 경계를 구성합니다. |
| 3 | `252befef9527` | `fix(api): logout 시 server session 폐기` | A | AUTH, REALTIME, PERSISTENCE | logout을 server-side security transition으로 수정합니다. |
| 4 | `bc789124b20b` | `test(api): logout session invalidation 검증` | B | AUTH, SIMULATION, TEST | logout의 server-side session invalidation을 API integration 수준에서 검증합니다. |
| 5 | `b93910708330` | `feat(db): legacy session을 안전하게 만료` | A | AUTH, PERSISTENCE | 인증 계약 전환 뒤 기존 session credential을 명시적 migration으로 만료합니다. |
| 6 | `0649b63a1ca9` | `test(db): 인증 migration 중 데이터 보존 검증` | A | AUTH, PERSISTENCE, TEST | legacy session migration이 credential만 지우고 durable domain history를 보존하는지 PostgreSQL에서 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(db): 사용자 session 저장 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `4f65c6214321` |
| Importance | B |
| Tags | AUTH, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | session creation/resolution을 repository abstraction에 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `AppRepository.createSession`, `getSessionUser` 추가 전후를 비교합니다.
- `PostgresRepository.createSession`의 `randomUUID` 생성, `sessions` insert, 14일 `expires_at` 계산을 추적합니다.
- `PostgresRepository.getSessionUser`의 sessions/users join, token equality, `expires_at > now()` 조건과 `toSessionUser` 호출을 확인합니다.
- `MemoryRepository.sessions` Map의 key/value와 memory 구현에 만료 시간이 없는 차이를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `AppRepository`는 개발 사용자 upsert까지만 제공했고, 여러 HTTP 요청에 걸쳐 동일 사용자를 복원할 저장된 session identity가 없었습니다. |
| 해결하려던 문제 | API가 login 뒤의 요청을 사용자와 연결하려면 concrete DB에 직접 의존하지 않는 session 생성·해석 연산이 필요했습니다. |
| 핵심 결정 | 공통 interface에 `createSession(userId)`와 `getSessionUser(token)`을 추가하고 PostgreSQL과 memory 구현을 동시에 제공했습니다. PostgreSQL은 UUID token과 14일 만료를 저장하고, 조회 시 users와 join합니다. |
| 입력 → 상태 전이 → 출력 | caller가 user id로 `createSession` 호출 → UUID 생성 → PostgreSQL `sessions(token,user_id,expires_at)` insert 또는 memory Map 저장 → 이후 token으로 `getSessionUser` 호출 → 유효 row/user를 `SessionUser`로 변환하거나 `null` 반환. |
| ownership/lifetime/cleanup | repository가 token→user 연결을 보유합니다. caller는 raw token만 받습니다. PostgreSQL row는 DB 수명주기를 따르고 memory Map은 repository instance가 닫힐 때 함께 소멸합니다. |
| failure/rollback/retry | token이 없으면 `null`입니다. PostgreSQL insert/query 오류는 caller로 전파됩니다. memory에서 user가 사라졌거나 mapping이 없으면 `null`입니다. 이 SHA에는 session 삭제 API가 없습니다. |
| 보장하는 것 | PostgreSQL에서는 만료 전 token만 사용자로 해석하며, 두 repository가 같은 interface를 제공합니다. |
| 보장하지 않는 것 | token hash 저장, server-side logout, session row 청소, memory session 만료, 사용자 status 기반 거부는 아직 보장하지 않습니다. |
| 후속 연결 | `1779df300611`에서 HTTP login이 session을 만들고 여러 credential transport로 이를 읽습니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `1779df300611` — `feat(api): 로그인과 로비 HTTP 경계 구현`

### 5.2. `feat(api): 로그인과 로비 HTTP 경계 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `1779df300611` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, WEB |
| Source에서 확정된 역할 | repository-backed identity와 초기 HTTP login/logout 경계를 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`의 `buildApp`, `/auth/dev-login`, `/auth/logout`, `/me`, `/auth/me`, `/lobby` route를 확인합니다.
- dev login에서 `repo.upsertDevUser` → `repo.createSession` → `reply.setCookie('pp_session', ...)` 순서를 추적합니다.
- `currentUser`가 cookie, bearer header, parsed query, raw query를 어떤 우선순위로 선택하는지 확인합니다.
- logout route가 이 SHA에서는 `clearCookie`만 수행하고 repository를 호출하지 않는 점을 parent/후속 fix와 연결합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | session 저장 연산은 존재했지만 HTTP caller가 이를 생성·전달·해석하는 route가 없었습니다. |
| 해결하려던 문제 | 브라우저 login과 인증된 resource read를 repository session에 연결할 API composition이 필요했습니다. |
| 핵심 결정 | Fastify cookie/CORS 설정과 개발 login, logout, me/lobby route를 추가했습니다. login은 HttpOnly `pp_session` cookie를 설정하는 동시에 response body에도 raw token을 반환했습니다. |
| 입력 → 상태 전이 → 출력 | POST `/auth/dev-login` → body 기본값 적용 → user upsert → session 생성 → 14일 cookie 설정 → `{ user, token }` 반환. 인증 read는 `currentUser`가 cookie→bearer→query 순으로 token을 골라 repository 조회를 호출합니다. |
| ownership/lifetime/cleanup | repository가 session row를 소유하고 Fastify reply가 browser cookie 설정을 소유합니다. request별 identity 해석은 `currentUser` helper가 담당합니다. |
| failure/rollback/retry | body는 runtime schema로 검증되지 않고 기본값으로 대체됩니다. repository 오류는 route 오류로 전파됩니다. logout은 cookie만 지우므로 다른 transport나 보관된 token은 계속 유효합니다. |
| 보장하는 것 | login이 만든 repository session을 cookie·header·query를 통해 후속 route가 해석할 수 있습니다. |
| 보장하지 않는 것 | credential transport 단일화, secure cookie, server-side logout, active status 검사, production에서 dev login 차단은 아직 없습니다. |
| 후속 연결 | `252befef9527`이 logout의 client-only 삭제를 server-side revocation으로 수정합니다. |

비교 기준:
- 직전 관련 SHA: `4f65c6214321` — `feat(db): 사용자 session 저장 구현`
- 다음 관련 SHA: `252befef9527` — `fix(api): logout 시 server session 폐기`

### 5.3. `fix(api): logout 시 server session 폐기`

| 항목 | 값 |
| --- | --- |
| SHA | `252befef9527` |
| Importance | A |
| Tags | AUTH, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | logout을 server-side security transition으로 수정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- parent의 `/auth/logout`가 `clearCookie`만 호출하는 상태와 이 SHA의 `repo.deleteSession(readSessionToken(request))`를 비교합니다.
- `apps/api/src/app.ts`의 `readSessionToken`이 기존 cookie/header/query 우선순위를 그대로 추출 helper로 분리하는 이유를 확인합니다.
- `packages/db/src/index.ts`의 interface, PostgreSQL `delete from sessions where token = ...`, memory `Map.delete`를 함께 추적합니다.
- DB delete await가 cookie clear보다 먼저 실행되므로 삭제 실패 시 어떤 response/cookie 상태가 남는지 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | logout은 browser의 `pp_session` cookie만 제거했습니다. response body나 storage에 남은 bearer/query token은 repository row가 유지되는 동안 `/me`에 재사용할 수 있었습니다. |
| 해결하려던 문제 | 사용자가 logout을 수행해도 server-side credential이 유효한 보안 결함이 있었습니다. |
| 핵심 결정 | credential 추출을 `readSessionToken`으로 공통화하고, logout에서 해당 token을 repository에서 삭제한 뒤 cookie를 지우도록 변경했습니다. 두 repository에 `deleteSession`을 추가했습니다. |
| 입력 → 상태 전이 → 출력 | POST `/auth/logout` → request에서 한 개 token 선택 → `repo.deleteSession` await → PostgreSQL DELETE 또는 Map.delete → `clearCookie` → `{ ok: true }`. |
| ownership/lifetime/cleanup | logout route가 revocation 명령을 시작하지만 실제 session state 변경은 repository가 소유합니다. cookie 정리는 DB 삭제 성공 뒤 Fastify reply가 수행합니다. |
| failure/rollback/retry | token이 없으면 repository 삭제는 no-op입니다. PostgreSQL DELETE가 실패하면 route는 cookie clear와 성공 response에 도달하지 않습니다. 선택된 한 token만 폐기합니다. |
| 보장하는 것 | 성공한 logout 뒤 그 요청에 제시된 server session token은 repository에서 더 이상 사용자로 해석되지 않습니다. |
| 보장하지 않는 것 | 동일 사용자의 다른 session 전체 폐기, 이미 열린 WebSocket 회수, 만료 row 일괄 정리, DB와 cookie 삭제의 원자성은 보장하지 않습니다. |
| 후속 연결 | `bc789124b20b`이 동일 bearer token의 logout 전 200·후 401 전이를 고정합니다. |

#### 최소 코드 근거

- SHA: `252befef9527`
- 파일: `apps/api/src/app.ts`
- 위치: `POST /auth/logout`
- 확인 목적: server revocation이 cookie clear보다 먼저 수행되는 순서

```ts
app.post("/auth/logout", async (request, reply) => {
  await repo.deleteSession(readSessionToken(request));
  reply.clearCookie("pp_session", { path: "/" });
  return { ok: true };
});
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | cookie를 지우면 logout이 완료된다는 가정이었습니다. |
| 실제 실패 또는 위험 | cookie 밖에 복사된 token을 다시 보내면 server session이 계속 인증되었습니다. |
| Root cause | logout route가 repository의 session row/Map entry를 변경하지 않았습니다. |
| 수정된 invariant | logout 성공은 client cookie 정리뿐 아니라 제시된 server session의 폐기를 포함합니다. |
| 변경 코드 | `apps/api/src/app.ts` `/auth/logout`, `readSessionToken`; `packages/db/src/index.ts` `deleteSession` implementations. |
| Regression evidence | `bc789124b20b`의 `invalidates the server session on logout` Fastify injection test. |

비교 기준:
- 직전 관련 SHA: `1779df300611` — `feat(api): 로그인과 로비 HTTP 경계 구현`
- 다음 관련 SHA: `bc789124b20b` — `test(api): logout session invalidation 검증`

### 5.4. `test(api): logout session invalidation 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `bc789124b20b` |
| Importance | B |
| Tags | AUTH, SIMULATION, TEST |
| Source에서 확정된 역할 | logout의 server-side session invalidation을 API integration 수준에서 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.test.ts`의 `invalidates the server session on logout` test를 확인합니다.
- login response의 raw token을 bearer header로 `/me` → `/auth/logout` → `/me`에 동일하게 재사용하는 순서를 기록합니다.
- test fixture가 memory repository와 `app.inject`를 사용하며 실제 browser cookie·network·PostgreSQL을 사용하지 않는 범위를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production fix는 추가됐지만 같은 token을 logout 뒤 재사용하는 회귀를 자동으로 막는 테스트가 없었습니다. |
| 해결하려던 문제 | client cookie 상태와 무관하게 server credential 자체가 폐기되는지를 직접 검증해야 했습니다. |
| 핵심 결정 | dev login으로 받은 token을 bearer로 재사용해 logout 전 `/me` 200, logout 200, logout 후 `/me` 401을 확인하는 Fastify injection test를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | login injection → token 추출 → bearer `/me` 200 → bearer logout → 같은 bearer `/me` 401. |
| ownership/lifetime/cleanup | test lifecycle은 기존 suite의 app/repository fixture가 소유합니다. 별도의 socket·DB container resource는 만들지 않습니다. |
| failure/rollback/retry | status code 하나라도 기대와 다르면 test가 실패합니다. delete 호출 횟수나 cookie header는 직접 관찰하지 않습니다. |
| 보장하는 것 | memory repository를 통과하는 실제 route path에서 동일 server token이 logout 뒤 인증되지 않음을 증명합니다. |
| 보장하지 않는 것 | PostgreSQL DELETE, real HTTP transport, browser cookie 제거, 여러 session 동시 revocation은 증명하지 않습니다. |
| 후속 연결 | `b93910708330`은 개별 logout이 아니라 인증 계약 변경 시 모든 legacy session을 migration으로 폐기합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | logout 성공 뒤 제시한 server session token은 재사용할 수 없습니다. |
| 재현하는 failure/boundary | cookie 삭제와 무관하게 동일 bearer token을 다시 보내는 경계입니다. |
| test technique | memory repository를 포함한 Fastify injection integration regression입니다. |
| 통과하는 production path | `/auth/dev-login` → `createSession` → `/me` → `/auth/logout` → `deleteSession` → `/me`. |
| 증명하는 것 | route와 memory repository가 200→401 state transition을 만듭니다. |
| 증명하지 않는 것 | PostgreSQL, 브라우저, 실제 network, 전체 사용자 session 폐기는 검증하지 않습니다. |
| 후속 회귀 방지 | logout을 다시 cookie-only 처리로 퇴행시키는 변경을 막습니다. |

비교 기준:
- 직전 관련 SHA: `252befef9527` — `fix(api): logout 시 server session 폐기`
- 다음 관련 SHA: `b93910708330` — `feat(db): legacy session을 안전하게 만료`

### 5.5. `feat(db): legacy session을 안전하게 만료`

| 항목 | 값 |
| --- | --- |
| SHA | `b93910708330` |
| Importance | A |
| Tags | AUTH, PERSISTENCE |
| Source에서 확정된 역할 | 인증 계약 전환 뒤 기존 session credential을 명시적 migration으로 만료합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/migrations/005_expire_legacy_sessions.sql`의 유일한 statement와 영향 table을 확인합니다.
- `packages/db/src/migrator.ts`의 `migrateDatabase(databaseUrl, targetMigration?)`와 `migrateTo`/`migrateToLatest` 분기를 확인합니다.
- migration이 users, matches, rating_history를 수정하지 않는다는 점을 SQL 자체와 후속 integration test로 연결합니다.
- 부분 session 선택이 아니라 `delete from sessions` 전체 삭제를 택한 운영 의미를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | cookie-only·ticket 기반 인증으로 계약이 바뀌었지만 이전 방식으로 발급된 session rows는 DB에 남아 계속 해석될 가능성이 있었습니다. |
| 해결하려던 문제 | credential transport·trust model 변경 뒤 legacy session을 선택적으로 판별할 metadata가 없어, 기존 credential을 안전하게 계속 신뢰할 수 없었습니다. |
| 핵심 결정 | 005 migration에서 `sessions` table 전체를 비우고 모든 등록 사용자가 재인증하도록 했습니다. 테스트가 이전 migration 상태를 만들 수 있도록 migrator에 optional target을 추가했습니다. |
| 입력 → 상태 전이 → 출력 | migration provider가 005를 선택 → `delete from sessions` 실행 → Kysely migration metadata에 적용 기록. 테스트용 호출은 `migrateTo('004_...')`로 pre-migration state를 구성할 수 있습니다. |
| ownership/lifetime/cleanup | schema evolution은 migrator가 소유하고 session invalidation 범위는 SQL migration이 소유합니다. 사용자·경기 데이터의 owner table은 건드리지 않습니다. |
| failure/rollback/retry | migration 실행 오류는 migrator error로 보고되고 적용 완료로 기록되지 않습니다. 전체 삭제이므로 정상·비정상 legacy session을 구분하지 않습니다. |
| 보장하는 것 | 005가 성공한 PostgreSQL database에서는 기존 session row가 하나도 남지 않습니다. |
| 보장하지 않는 것 | memory repository session, 사용자별 선택적 유지, 무중단 reauthentication, concurrent request와의 application-level coordination은 보장하지 않습니다. |
| 후속 연결 | `0649b63a1ca9`이 004 상태의 실제 PostgreSQL에 session과 경기 이력을 만든 뒤 005의 최소 삭제 범위를 검증합니다. |

#### 최소 코드 근거

- SHA: `b93910708330`
- 파일: `packages/db/migrations/005_expire_legacy_sessions.sql`
- 위치: `migration body`
- 확인 목적: credential state만 전부 폐기하는 최소 migration

```sql
delete from sessions;
```

비교 기준:
- 직전 관련 SHA: `bc789124b20b` — `test(api): logout session invalidation 검증`
- 다음 관련 SHA: `0649b63a1ca9` — `test(db): 인증 migration 중 데이터 보존 검증`

### 5.6. `test(db): 인증 migration 중 데이터 보존 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `0649b63a1ca9` |
| Importance | A |
| Tags | AUTH, PERSISTENCE, TEST |
| Source에서 확정된 역할 | legacy session migration이 credential만 지우고 durable domain history를 보존하는지 PostgreSQL에서 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/postgres.integration.test.ts`의 `expires legacy sessions without changing users or match history` test를 확인합니다.
- `migrateDatabase(..., '004_friendship_tournament_invariants')`로 pre-005 schema를 만든 뒤 users/session/finalized match를 구성하는 순서를 추적합니다.
- `authMigrationSnapshot`의 users, matches, rating_history SQL column 집합과 정렬 기준을 기록합니다.
- 005 적용 뒤 `getSessionUser` null, sessions count 0, before/after snapshot equality, migration metadata 포함을 각각 어떤 assertion이 증명하는지 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 005 SQL은 한 줄로 최소 범위를 보였지만 실제 migration lifecycle에서 다른 durable rows가 보존되는 자동 증거가 없었습니다. |
| 해결하려던 문제 | 인증 credential 폐기가 사용자 rating·경기 결과·rating history까지 손상시키지 않는지 실제 PostgreSQL에서 재현해야 했습니다. |
| 핵심 결정 | 격리 database를 004까지만 migration하고 두 사용자, 유효 session, finalized match를 만든 뒤 snapshot을 저장합니다. latest migration 후 session invalidation과 세 domain query 결과의 완전 동일성을 함께 검증합니다. |
| 입력 → 상태 전이 → 출력 | isolated DB → migrate to 004 → users/session/finalizeMatch 생성 → snapshot 및 session 유효성 확인 → migrate latest → session null/count 0 → users/matches/rating_history snapshot equality → migration name 확인. |
| ownership/lifetime/cleanup | `withIsolatedDatabase` fixture가 database/pool/repository cleanup을 소유합니다. test가 생성한 pre-migration state는 해당 isolated schema에 한정됩니다. |
| failure/rollback/retry | 005가 session을 남기거나 보존 column 중 하나를 바꾸거나 migration metadata를 기록하지 않으면 각각 독립 assertion이 실패합니다. |
| 보장하는 것 | 실제 PostgreSQL migration 경로에서 session rows만 제거되고 선택한 사용자·경기·rating history columns는 그대로임을 증명합니다. |
| 보장하지 않는 것 | 테스트에 포함되지 않은 모든 table/column, production traffic 중 migration, backup/restore, memory repository는 증명하지 않습니다. |
| 후속 연결 | 이 Thread의 최종 검증입니다. cookie-only transport와 WebSocket ticket 상세는 별도 core realtime Thread의 범위입니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | auth migration은 legacy credential을 폐기하면서 durable account/game history를 보존합니다. |
| 재현하는 failure/boundary | 구버전 schema에 유효 session과 이미 확정된 경기 이력이 함께 존재하는 migration 경계입니다. |
| test technique | 격리된 실제 PostgreSQL을 사용하는 migration integration regression입니다. |
| 통과하는 production path | `migrateDatabase(target=004)` → repository writes/finalizeMatch → `migrateDatabase(latest)` → repository/SQL reads. |
| 증명하는 것 | session 0개, old token 무효, users/matches/rating_history selected state 보존, 005 적용 기록입니다. |
| 증명하지 않는 것 | 온라인 migration concurrency, 모든 schema object, memory backend, 운영 backup은 검증하지 않습니다. |
| 후속 회귀 방지 | 인증 migration이 account/history table을 함께 삭제하거나 005를 누락하는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `b93910708330` — `feat(db): legacy session을 안전하게 만료`
- 이 Thread의 최종 상태와 비교합니다.

## 6. Invariant ledger

| Invariant | 도입/수정 SHA | 변화 | 검증/후속 보호 |
| --- | --- | --- | --- |
| session identity는 repository가 소유한다 | `4f65c6214321` | PostgreSQL row와 memory Map으로 도입 | `252befef9527`에서 삭제 연산까지 확장 |
| logout은 server credential을 폐기한다 | `252befef9527` | client cookie-only logout 결함을 수정 | `bc789124b20b`에서 동일 token 재사용으로 검증 |
| auth migration은 credential만 만료한다 | `b93910708330` | 005가 sessions 전체 삭제 | `0649b63a1ca9`에서 users/matches/rating_history 보존 검증 |

## 7. Failure → Fix → Test

| Failure/Risk | Fix | Test | 연결 근거 |
| --- | --- | --- | --- |
| cookie만 삭제해 bearer/query token이 계속 유효 | `252befef9527` | `bc789124b20b` | 동일 token으로 `/me` 200→logout→401 |
| 새 인증 계약 뒤 legacy session이 DB에 잔존 | `b93910708330` | `0649b63a1ca9` | 005 적용 전후 실제 PostgreSQL snapshot 비교 |

## 8. Ownership·state·responsibility 변화

| 관심사 | 최종 owner와 변화 |
| --- | --- |
| session state | `AppRepository`와 concrete repository가 token→user mapping 및 삭제를 소유합니다. |
| HTTP credential 전달 | Fastify route/helper가 request token 선택과 cookie 설정·삭제를 소유하지만 session validity는 결정하지 않습니다. |
| schema 전환 | Kysely migrator와 005 SQL이 legacy credential 전체 폐기를 소유합니다. |
| test resource | API test는 suite fixture, migration test는 isolated PostgreSQL fixture가 lifecycle을 소유합니다. |

## 9. Thread 최종 상태

### 최종 owner

등록 session의 authoritative state는 repository입니다. HTTP layer는 token을 전달하고 login/logout 명령을 조정하며, migration layer는 trust model 변경 시 기존 credential을 일괄 폐기합니다.

### 최종 execution flow

login에서 user upsert와 session 생성 후 cookie를 설정합니다. 인증 요청은 repository가 token을 현재 사용자로 해석합니다. logout은 제시된 token을 repository에서 먼저 삭제하고 cookie를 지웁니다. 인증 계약 전환 시 005 migration이 남은 legacy sessions를 전부 제거합니다.

### 최종 guarantee

성공한 logout token의 재사용 차단과, 005 적용 시 모든 PostgreSQL legacy session 폐기 및 핵심 durable history 보존이 코드와 regression test로 연결됩니다.

### 최종 non-guarantee

사용자 전체 device logout, session row TTL 청소, DB 삭제와 client cookie 삭제의 분산 원자성, 온라인 migration 무중단성은 이 Thread가 보장하지 않습니다.

## 10. Learning completion check

- [x] 모든 Commit map SHA의 exact diff와 관련 code path를 확인했습니다.
- [x] Importance별로 A commit은 failure/ownership을 깊게, B commit은 Thread 역할 중심으로 기록했습니다.
- [x] Fix와 regression test를 이전 failure에 연결했습니다.
- [x] 실제 실행 증거와 code inspection을 구분했습니다.
- [ ] Runtime test command 실행 — 로컬 checkout을 만들 GitHub network/DNS가 차단되어 실행하지 못했습니다. 따라서 문서에는 실행 성공을 주장하지 않습니다.
===== END FILE: 01-server-session-logout-and-auth-migrations.md =====

===== BEGIN FILE: 02-explicit-role-and-administrator-authorization.md =====
# 명시적 role과 관리자 권한

- 카테고리: `03-identity-authorization-and-account-lifecycle` — 신원·권한·계정 수명주기
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

초기 administrator user listing/status mutation과 inline role check에서 시작해, 사용자 handle이 암묵적으로 privilege를 부여하던 규칙을 제거하고 repository/CLI의 명시적 role assignment와 현재 account status를 함께 검사하는 administrator authorization으로 발전하는 과정을 복원합니다.

이 문서는 exact SHA를 순서대로 확인해 설계 → 구현 → 실패 → 수정 → 검증을 복원합니다.

### 직접 연결되는 불변식

- 사용자가 선택하는 handle은 administrator privilege source가 될 수 없습니다.
- administrator route는 현재 session user가 존재하고 active이며 role이 `admin`인 경우에만 통과합니다.
- role assignment는 ordinary login과 분리된 repository/CLI operation을 통해서만 수행됩니다.

## 2. 핵심 질문

- 초기 `upsertDevUser`가 `admin` handle을 어떤 방식으로 role에 연결했고 테스트가 그 가정을 어떻게 사용했습니까?
- `setUserRoleByHandle`은 handle normalization, NPC 제외, not-found를 어떻게 처리합니까?
- dev seed의 admin 승격과 ordinary dev login의 role reset은 서로 어떤 차이가 있습니까?
- `requireAdmin`의 authentication → active status → role 검사 순서가 어떤 error code를 만듭니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 지정 브랜치 ancestry에 속하는지 확인합니다.
- 각 SHA의 diff와 parent 또는 직전 관련 SHA의 실제 state를 구분합니다.
- 아래 commit-specific 파일·함수·SQL·테스트를 확인하고 generic 설명으로 대체하지 않습니다.
- Fix는 이전 가정 → 실제 실패 → root cause → corrected invariant → regression evidence로 연결합니다.
- 실행하지 않은 test/command는 실행한 것처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread의 최종 owner, state transition, guarantee와 non-guarantee를 정리합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `fa6e7de259cf` | `feat(db): 관리자 상태 변경 저장 구현` | B | AUTH, PERSISTENCE | administrator user listing과 ban-state mutation을 repository에 추가합니다. |
| 2 | `e8bb6a4bf68b` | `feat(api): 토너먼트와 관리자 라우트 추가` | B | AUTH, PERSISTENCE, TOURNAMENT | administrator resource에 authentication과 inline role authorization을 적용합니다. |
| 3 | `1395d45a3665` | `test(api): 관리자 사용자 상태 변경 검증` | B | AUTH, PERSISTENCE, TEST | login부터 admin route와 memory repository status mutation까지 검증합니다. |
| 4 | `45225adcfcd9` | `feat(db): 명시적 사용자 role 할당 추가` | A | AUTH, PERSISTENCE | handle 기반 privilege를 제거하고 repository/CLI role assignment를 도입합니다. |
| 5 | `dae31d4a223c` | `test(auth): 명시적 role assignment 검증` | B | AUTH, PERSISTENCE, TEST | `admin` handle만으로 privilege가 생기지 않는지 PostgreSQL에서 검증합니다. |
| 6 | `c577fe2603e3` | `fix(auth): 정지된 관리자 login 거부` | A | AUTH, RISK | administrator authorization에 현재 account status를 포함합니다. |
| 7 | `aa037c5291fe` | `test(auth): 정지된 관리자 session 거부 검증` | B | AUTH, TEST | 정지 후 기존 administrator session의 privilege가 거부되는지 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(db): 관리자 상태 변경 저장 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `fa6e7de259cf` |
| Importance | B |
| Tags | AUTH, PERSISTENCE |
| Source에서 확정된 역할 | administrator user listing과 ban-state mutation을 repository에 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `AppRepository.listAdminUsers`, `setUserBan` 추가와 PostgreSQL/memory 구현을 비교합니다.
- PostgreSQL의 users UPDATE와 admin_actions INSERT가 이 SHA에서는 별도 `this.db` statement인 점을 확인합니다.
- `packages/db/src/schema.ts`의 `AdminActionTable` actor/target nullability, action enum, reason, created_at을 확인합니다.
- memory 구현이 actor/reason을 저장하지 않고 user status만 바꾸는 backend 차이를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | repository에는 administrator 전용 사용자 조회나 account status mutation operation이 없었습니다. |
| 해결하려던 문제 | 관리 기능이 concrete storage를 우회하지 않고 user status와 audit row를 변경할 공통 persistence API가 필요했습니다. |
| 핵심 결정 | `listAdminUsers`와 `setUserBan`을 interface에 추가했습니다. PostgreSQL은 최근 사용자 50명을 조회하고 status update 뒤 admin_actions insert를 실행하며, memory는 status만 변경했습니다. |
| 입력 → 상태 전이 → 출력 | caller가 actor/target/banned/reason 전달 → PostgreSQL users status/banned_at UPDATE returning → admin_actions INSERT → mapped target 반환. memory는 target Map row를 찾아 status만 변경합니다. |
| ownership/lifetime/cleanup | repository가 user status mutation을 소유합니다. PostgreSQL은 audit row도 저장하지만 이 SHA의 memory backend는 audit identity를 소유하지 않습니다. |
| failure/rollback/retry | target이 없으면 PostgreSQL update result/foreign-key 또는 `firstRow` 경로에서 실패할 수 있고 memory는 명시적으로 throw합니다. 두 SQL이 별도이므로 audit insert 실패 뒤 status만 반영될 위험이 있습니다. |
| 보장하는 것 | 두 backend에서 target status를 active/banned로 바꾸는 공통 호출 지점을 제공합니다. |
| 보장하지 않는 것 | route authorization, 명시적 role provisioning, PostgreSQL atomicity, memory audit parity는 아직 없습니다. |
| 후속 연결 | `e8bb6a4bf68b`이 repository operation을 role-gated HTTP routes에 연결합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `e8bb6a4bf68b` — `feat(api): 토너먼트와 관리자 라우트 추가`

### 5.2. `feat(api): 토너먼트와 관리자 라우트 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e8bb6a4bf68b` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, TOURNAMENT |
| Source에서 확정된 역할 | administrator resource에 authentication과 inline role authorization을 적용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`의 `/admin/users`, `/admin/users/:id/ban`, `/admin/users/:id/status` route를 확인합니다.
- 각 route의 `currentUser` null 검사와 `user.role !== 'admin'` 403 분기를 순서대로 기록합니다.
- request body의 optional `banned/status/reason` cast와 `manual review` 기본값이 runtime 검증 없이 repository로 전달되는 점을 확인합니다.
- 이 시점의 `upsertDevUser`가 handle `admin`에 role을 부여하는 전제와 route authorization을 연결합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | repository operation은 존재했지만 HTTP caller가 administrator identity를 증명하고 호출할 route가 없었습니다. |
| 해결하려던 문제 | 관리자 사용자 목록과 상태 변경을 authentication/authorization 뒤에 노출해야 했습니다. |
| 핵심 결정 | 세 admin route를 추가하고 `currentUser`가 없으면 401, role이 admin이 아니면 403을 반환한 뒤 repository를 호출하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | request → session user resolve → null이면 401 → role 비교 → non-admin이면 403 → params/body cast → `listAdminUsers` 또는 `setUserBan` → response. |
| ownership/lifetime/cleanup | HTTP layer가 route-level authorization을 소유하고 repository가 durable mutation을 소유합니다. 검사 로직은 각 route에 중복돼 있습니다. |
| failure/rollback/retry | active/banned status는 검사하지 않습니다. malformed params/body도 strict schema로 막지 않습니다. repository 오류는 route error로 전파됩니다. |
| 보장하는 것 | 인증된 `admin` role 사용자만 admin resources를 호출하는 최초의 server-side gate를 제공합니다. |
| 보장하지 않는 것 | handle과 role 분리, suspended admin 차단, centralized authorization helper, audit atomicity는 아직 보장하지 않습니다. |
| 후속 연결 | `1395d45a3665`이 초기 handle-derived admin 경로의 happy path를 검증합니다. |

비교 기준:
- 직전 관련 SHA: `fa6e7de259cf` — `feat(db): 관리자 상태 변경 저장 구현`
- 다음 관련 SHA: `1395d45a3665` — `test(api): 관리자 사용자 상태 변경 검증`

### 5.3. `test(api): 관리자 사용자 상태 변경 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1395d45a3665` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, TEST |
| Source에서 확정된 역할 | login부터 admin route와 memory repository status mutation까지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/admin.test.ts`의 suite setup/teardown과 memory repository seed를 확인합니다.
- handle `admin`으로 dev login한 뒤 response token을 bearer로 보내는 초기 privilege 가정을 기록합니다.
- target login → `/admin/users/:id/ban` → status `banned` assertion을 추적합니다.
- non-admin 거부, audit row, PostgreSQL, active status를 검증하지 않는 범위를 명시합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 관리 route가 추가됐지만 login부터 repository mutation까지 이어지는 자동 회귀 증거가 없었습니다. |
| 해결하려던 문제 | 초기 administrator happy path가 실제 Fastify route와 memory backend를 통과하는지 확인해야 했습니다. |
| 핵심 결정 | seeded memory repository에서 handle `admin`으로 login하고 target을 생성한 뒤 bearer token으로 ban route를 호출해 200과 `banned` status를 확인했습니다. |
| 입력 → 상태 전이 → 출력 | memory seed → admin dev login → target dev login → admin token/target id 추출 → POST ban → response status assertion. |
| ownership/lifetime/cleanup | test suite가 app/repository lifecycle을 setup/teardown합니다. authorization credential은 login response token입니다. |
| failure/rollback/retry | route가 200이 아니거나 returned status가 banned가 아니면 실패합니다. role의 출처 자체는 검증하지 않습니다. |
| 보장하는 것 | 초기 handle-derived admin happy path와 memory status mutation이 연결됩니다. |
| 보장하지 않는 것 | non-admin denial, explicit role assignment, audit persistence, PostgreSQL, suspended admin denial은 증명하지 않습니다. |
| 후속 연결 | `45225adcfcd9`이 이 테스트의 handle privilege 전제를 제거하고 fixture에 명시적 role assignment를 추가합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | admin role 사용자만 상태 변경 route의 성공 경로를 사용할 수 있습니다. |
| 재현하는 failure/boundary | 초기 구현의 login→authorization→memory mutation 통합 경계입니다. |
| test technique | Fastify injection과 memory repository를 사용하는 API integration happy-path test입니다. |
| 통과하는 production path | dev login → session lookup → inline role check → `setUserBan` → response mapping. |
| 증명하는 것 | 당시 `admin` handle fixture가 ban route를 성공시키고 target status를 변경합니다. |
| 증명하지 않는 것 | privilege source의 안전성, denial path, audit/transaction, PostgreSQL은 검증하지 않습니다. |
| 후속 회귀 방지 | 기본 admin route wiring과 status response가 끊기는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `e8bb6a4bf68b` — `feat(api): 토너먼트와 관리자 라우트 추가`
- 다음 관련 SHA: `45225adcfcd9` — `feat(db): 명시적 사용자 role 할당 추가`

### 5.4. `feat(db): 명시적 사용자 role 할당 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `45225adcfcd9` |
| Importance | A |
| Tags | AUTH, PERSISTENCE |
| Source에서 확정된 역할 | handle 기반 privilege를 제거하고 repository/CLI role assignment를 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 PostgreSQL/memory `upsertDevUser`에서 handle 비교가 제거되고 role이 `user`로 설정·reset되는 diff를 확인합니다.
- `setUserRoleByHandle`의 `normalizeHandle`, `is_npc = false`, `returning *`, not-found error를 두 backend에서 비교합니다.
- development seed가 ordinary upsert 뒤 seed admin을 별도로 `admin`으로 승격하는 memory/PostgreSQL 경로를 확인합니다.
- `packages/db/src/cli.ts`와 package script `user:set-role`의 argument validation 및 repository close를 추적합니다.
- `apps/api/src/admin.test.ts` fixture가 login 뒤 `repo.setUserRoleByHandle`을 명시적으로 호출하도록 바뀐 이유를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | ordinary dev login/upsert가 handle이 `admin`인지 검사해 role과 rating을 자동 부여했습니다. 사용자가 선택 가능한 문자열이 privilege source였습니다. |
| 해결하려던 문제 | `admin` handle을 선택하는 것만으로 administrator 권한을 얻는 privilege escalation 경로가 있었습니다. |
| 핵심 결정 | 모든 ordinary upsert를 `user` role로 고정하고, non-NPC user만 대상으로 하는 `setUserRoleByHandle` repository operation과 검증된 CLI command를 추가했습니다. development seed 승격은 별도 provisioning 단계로 남겼습니다. |
| 입력 → 상태 전이 → 출력 | ordinary login/upsert → role `user` 저장/갱신. 운영 provisioning은 CLI args 검증 → repository open → normalized handle로 non-NPC UPDATE/Map mutation → result 출력 → repository close. |
| ownership/lifetime/cleanup | ordinary identity creation은 privilege를 소유하지 않습니다. role state는 users row/MemoryUserRow가 소유하고, role 변경 권한은 application login과 분리된 repository/CLI 실행 경계에 있습니다. |
| failure/rollback/retry | handle 누락·role enum 위반은 CLI에서 즉시 실패합니다. 대상이 없거나 NPC뿐이면 repository가 `user not found`를 throw합니다. role 변경 자체의 actor/audit authorization은 이 operation에 없습니다. |
| 보장하는 것 | 사용자 입력 handle만으로 admin role이 생기지 않으며, 명시적 operation을 호출해야 role이 바뀝니다. |
| 보장하지 않는 것 | role-change audit, CLI 실행자의 OS-level 권한, concurrent provisioning, ordinary upsert가 기존 role을 유지하는 정책은 보장하지 않습니다. 실제로 dev upsert는 role을 `user`로 reset합니다. |
| 후속 연결 | `dae31d4a223c`이 실제 PostgreSQL에서 `admin` handle login은 user이고 명시적 assignment 뒤에만 admin이 됨을 검증합니다. |

#### 최소 코드 근거

- SHA: `45225adcfcd9`
- 파일: `packages/db/src/index.ts`
- 위치: `PostgresRepository.upsertDevUser / setUserRoleByHandle`
- 확인 목적: identity creation과 privilege assignment의 분리

```ts
values (..., 'user', false)
...
update users
set role = ${role}
where handle = ${normalizeHandle(handle)} and is_npc = false
returning *
```

비교 기준:
- 직전 관련 SHA: `1395d45a3665` — `test(api): 관리자 사용자 상태 변경 검증`
- 다음 관련 SHA: `dae31d4a223c` — `test(auth): 명시적 role assignment 검증`

### 5.5. `test(auth): 명시적 role assignment 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `dae31d4a223c` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, TEST |
| Source에서 확정된 역할 | `admin` handle만으로 privilege가 생기지 않는지 PostgreSQL에서 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/postgres.integration.test.ts`의 `grants administrator access only through an explicit role assignment` test를 확인합니다.
- development seed 뒤 `upsertDevUser({ handle: 'admin' })`가 role `user`를 반환하는 assertion을 기록합니다.
- `setUserRoleByHandle('admin','admin')` 뒤 role `admin` assertion과 실제 route/CLI를 통과하지 않는 범위를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 코드는 handle privilege를 제거했지만 실제 PostgreSQL upsert와 role update가 그 invariant를 보존하는 integration evidence가 없었습니다. |
| 해결하려던 문제 | schema/query 차이로 PostgreSQL에서만 handle-based privilege가 남거나 assignment가 적용되지 않는 회귀를 막아야 했습니다. |
| 핵심 결정 | 격리 PostgreSQL에서 development seed를 만들고 같은 `admin` handle로 ordinary upsert한 결과가 user인지 확인한 뒤 explicit role operation 결과가 admin인지 확인했습니다. |
| 입력 → 상태 전이 → 출력 | isolated repository → development seed → `upsertDevUser(admin)` → expect user → `setUserRoleByHandle` → expect admin. |
| ownership/lifetime/cleanup | isolated PostgreSQL fixture가 DB lifecycle을 소유합니다. test는 repository interface만 사용합니다. |
| failure/rollback/retry | 두 role assertion 중 하나라도 다르면 실패합니다. HTTP authorization 또는 CLI parser는 실행하지 않습니다. |
| 보장하는 것 | PostgreSQL repository에서 handle과 role이 분리되고 explicit operation이 role을 변경함을 증명합니다. |
| 보장하지 않는 것 | memory parity, CLI authorization, role-change audit, admin route 성공/거부는 증명하지 않습니다. |
| 후속 연결 | `c577fe2603e3`은 role만 맞으면 정지된 account도 admin route를 통과하던 별도 authorization 결함을 수정합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | handle 문자열은 privilege source가 아니며 role 변경에는 explicit assignment가 필요합니다. |
| 재현하는 failure/boundary | `admin`이라는 공격 가능한 handle과 stored role 사이의 PostgreSQL boundary입니다. |
| test technique | 실제 PostgreSQL repository integration regression입니다. |
| 통과하는 production path | development seed/upsert SQL → role mapping → explicit role UPDATE → mapping. |
| 증명하는 것 | ordinary `admin` upsert는 user, explicit operation 뒤에는 admin입니다. |
| 증명하지 않는 것 | HTTP/CLI 실행 권한, audit, memory backend는 검증하지 않습니다. |
| 후속 회귀 방지 | handle 비교를 privilege assignment에 다시 도입하는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `45225adcfcd9` — `feat(db): 명시적 사용자 role 할당 추가`
- 다음 관련 SHA: `c577fe2603e3` — `fix(auth): 정지된 관리자 login 거부`

### 5.6. `fix(auth): 정지된 관리자 login 거부`

| 항목 | 값 |
| --- | --- |
| SHA | `c577fe2603e3` |
| Importance | A |
| Tags | AUTH, RISK |
| Source에서 확정된 역할 | administrator authorization에 현재 account status를 포함합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`의 `requireAdmin` parent와 이 SHA를 비교해 `isActive(user)` 검사가 추가된 정확한 위치를 확인합니다.
- 검사 순서가 no user→`unauthorized`, inactive→`suspended`, non-admin→`forbidden`임을 error helper와 연결합니다.
- 기존 session cookie가 있어도 `currentUser`가 현재 users row status를 다시 mapping하는지 해당 SHA의 session lookup을 확인합니다.
- 이 fix가 session row 자체를 삭제하거나 이미 열린 WebSocket을 닫지는 않는 범위를 후속 live-revocation Thread와 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `requireAdmin`은 session user 존재와 role만 검사했습니다. 계정이 banned로 바뀌어도 기존 session이 admin role을 계속 포함하면 admin route에 접근할 수 있었습니다. |
| 해결하려던 문제 | account suspension이 ordinary mutations에는 적용돼도 administrator capability에는 적용되지 않는 authorization bypass가 있었습니다. |
| 핵심 결정 | `requireAdmin`에서 role 검사 전에 `isActive(user)`를 호출하고 inactive user에는 typed `account_suspended` 403을 발생시키도록 했습니다. |
| 입력 → 상태 전이 → 출력 | admin request → `currentUser`로 현재 user projection 해석 → 없으면 401 → inactive면 403 account_suspended → role이 admin 아니면 403 forbidden → handler 실행. |
| ownership/lifetime/cleanup | current account status는 repository user row가 소유하고 `requireAdmin`이 매 요청마다 capability 결정을 소유합니다. session cookie는 identity pointer일 뿐 privilege snapshot이 아닙니다. |
| failure/rollback/retry | repository/session lookup 오류는 request failure로 전파됩니다. banned admin은 role이 맞아도 handler에 도달하지 않습니다. |
| 보장하는 것 | 기존 session을 보유한 suspended administrator도 다음 admin HTTP request에서 거부됩니다. |
| 보장하지 않는 것 | session 삭제, 이미 열린 realtime channel 즉시 회수, DB status change와 request race의 전역 직렬화는 보장하지 않습니다. |
| 후속 연결 | `aa037c5291fe`이 기존 admin cookie를 그대로 사용해 `/admin/actions`가 typed 403을 반환하는지 검증합니다. |

#### 최소 코드 근거

- SHA: `c577fe2603e3`
- 파일: `apps/api/src/app.ts`
- 위치: `requireAdmin`
- 확인 목적: authentication·status·role의 fail-closed 검사 순서

```ts
const user = await currentUser(repo, request);
if (!user) unauthorized();
if (!isActive(user)) suspended();
if (user.role !== "admin") forbidden();
return user;
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | admin role이 있으면 현재 account status와 무관하게 administrator capability가 유효하다는 가정이었습니다. |
| 실제 실패 또는 위험 | banned administrator의 이미 발급된 session이 admin actions에 계속 접근할 수 있었습니다. |
| Root cause | `requireAdmin`이 `currentUser`와 role만 검사하고 status를 누락했습니다. |
| 수정된 invariant | administrator capability는 active status와 admin role을 동시에 요구합니다. |
| 변경 코드 | `apps/api/src/app.ts` `requireAdmin`의 `if (!isActive(user)) suspended();`. |
| Regression evidence | `aa037c5291fe`의 existing administrator session test. |

비교 기준:
- 직전 관련 SHA: `dae31d4a223c` — `test(auth): 명시적 role assignment 검증`
- 다음 관련 SHA: `aa037c5291fe` — `test(auth): 정지된 관리자 session 거부 검증`

### 5.7. `test(auth): 정지된 관리자 session 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `aa037c5291fe` |
| Importance | B |
| Tags | AUTH, TEST |
| Source에서 확정된 역할 | 정지 후 기존 administrator session의 privilege가 거부되는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/admin.test.ts`의 `rejects an existing administrator session after the account is banned` test를 확인합니다.
- suite가 미리 만든 `adminCookie`를 ban 전후 동일하게 사용한다는 점을 기록합니다.
- repository에서 admin self-ban → `/admin/actions` injection → 403 typed error envelope의 code/message/requestId assertion을 추적합니다.
- WebSocket·session deletion·PostgreSQL을 검증하지 않는 범위를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `requireAdmin` status check가 추가됐지만 already-issued cookie가 실제 route에서 current status를 반영하는 회귀 테스트가 없었습니다. |
| 해결하려던 문제 | ban 이후 새 login만 막고 기존 session은 통과하는 퇴행을 방지해야 했습니다. |
| 핵심 결정 | 기존 admin cookie를 유지한 채 repository로 admin account를 ban하고 `/admin/actions`를 호출해 `account_suspended` 403 envelope를 확인했습니다. |
| 입력 → 상태 전이 → 출력 | seed admin/session cookie → `setUserBan(admin, admin, true)` → same cookie로 GET `/admin/actions` → requireAdmin status gate → 403. |
| ownership/lifetime/cleanup | suite fixture가 app/repository/cookie를 소유합니다. status mutation은 repository operation으로 수행합니다. |
| failure/rollback/retry | status code나 typed error fields가 다르면 실패합니다. session row가 남아 있는지는 의도적으로 허용합니다. |
| 보장하는 것 | 기존 administrator session도 현재 user status가 banned면 admin HTTP capability를 얻지 못합니다. |
| 보장하지 않는 것 | real PostgreSQL, network, 열린 WebSocket, 모든 non-admin route는 검증하지 않습니다. |
| 후속 연결 | 이 Thread의 최종 검증입니다. ordinary suspended user의 audit atomicity와 live socket 회수는 다음 Thread가 다룹니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | active status와 admin role을 모두 만족해야 administrator route에 접근할 수 있습니다. |
| 재현하는 failure/boundary | 이미 발급된 admin session cookie를 account ban 뒤 재사용하는 boundary입니다. |
| test technique | memory repository를 포함한 Fastify injection authorization regression입니다. |
| 통과하는 production path | `setUserBan` → same-cookie `/admin/actions` → `currentUser` → `requireAdmin` → typed error boundary. |
| 증명하는 것 | old session이 403 `account_suspended`로 거부됩니다. |
| 증명하지 않는 것 | session row 삭제, WebSocket revocation, PostgreSQL은 검증하지 않습니다. |
| 후속 회귀 방지 | `requireAdmin`에서 status check가 빠지는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `c577fe2603e3` — `fix(auth): 정지된 관리자 login 거부`
- 이 Thread의 최종 상태와 비교합니다.

## 6. Invariant ledger

| Invariant | 도입/수정 SHA | 변화 | 검증/후속 보호 |
| --- | --- | --- | --- |
| administrator operation은 server-side role gate 뒤 실행된다 | `e8bb6a4bf68b` | inline currentUser/role check로 도입 | `c577fe2603e3`에서 active status까지 확장 |
| handle은 privilege source가 아니다 | `45225adcfcd9` | ordinary upsert role을 user로 고정하고 explicit assignment 추가 | `dae31d4a223c` PostgreSQL test로 검증 |
| 기존 session도 현재 status를 따른다 | `c577fe2603e3` | requireAdmin이 status를 매 요청 검사 | `aa037c5291fe` same-cookie regression으로 검증 |

## 7. Failure → Fix → Test

| Failure/Risk | Fix | Test | 연결 근거 |
| --- | --- | --- | --- |
| `admin` handle만으로 role 획득 | `45225adcfcd9` | `dae31d4a223c` | ordinary upsert와 explicit assignment 결과를 분리 |
| banned administrator의 기존 session이 admin route 통과 | `c577fe2603e3` | `aa037c5291fe` | same cookie로 `/admin/actions` 403 검증 |

## 8. Ownership·state·responsibility 변화

| 관심사 | 최종 owner와 변화 |
| --- | --- |
| identity creation | ordinary `upsertDevUser`는 user role만 만들며 privilege를 소유하지 않습니다. |
| role state | users row/MemoryUserRow가 authoritative하고 `setUserRoleByHandle`이 명시적으로 변경합니다. |
| administrator capability | HTTP `requireAdmin`이 현재 identity, active status, role을 순서대로 판정합니다. |
| provisioning lifecycle | CLI가 args validation, repository open/call/close를 소유하지만 actor audit은 제공하지 않습니다. |

## 9. Thread 최종 상태

### 최종 owner

stored role과 account status는 repository user row가 소유합니다. ordinary login은 privilege를 만들지 않고, administrator capability는 request 시점의 `requireAdmin`이 current status와 role을 함께 읽어 결정합니다.

### 최종 execution flow

사용자는 ordinary login으로 user role session을 얻습니다. 별도 provisioning operation/CLI가 non-NPC user role을 변경합니다. admin request는 current user를 해석한 뒤 unauthenticated, suspended, non-admin을 차례로 거부하고 handler가 repository operation을 호출합니다.

### 최종 guarantee

handle-based privilege가 제거되고, explicit role assignment와 active-status-aware admin authorization이 PostgreSQL/API regression으로 보호됩니다.

### 최종 non-guarantee

role change audit·approval, CLI 호출자 인증, role 변경과 active session의 즉시 강제 logout, 이미 열린 realtime connection 회수는 이 Thread 범위 밖입니다.

## 10. Learning completion check

- [x] 모든 Commit map SHA의 exact diff와 관련 code path를 확인했습니다.
- [x] Importance별로 A commit은 failure/ownership을 깊게, B commit은 Thread 역할 중심으로 기록했습니다.
- [x] Fix와 regression test를 이전 failure에 연결했습니다.
- [x] 실제 실행 증거와 code inspection을 구분했습니다.
- [ ] Runtime test command 실행 — 로컬 checkout을 만들 GitHub network/DNS가 차단되어 실행하지 못했습니다. 따라서 문서에는 실행 성공을 주장하지 않습니다.
===== END FILE: 02-explicit-role-and-administrator-authorization.md =====

===== BEGIN FILE: 03-suspension-audit-atomicity-and-live-revocation.md =====
# 계정 정지 audit atomicity와 live revocation

- 카테고리: `03-identity-authorization-and-account-lifecycle` — 신원·권한·계정 수명주기
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

account status를 HTTP mutation과 WebSocket admission에 실제 authorization state로 적용하고 감사 기록을 노출한 초기 구현에서 출발해, status update와 audit insert를 한 PostgreSQL transaction으로 수정하고, ban 직후 이미 열린 authoritative WebSocket과 관련 runtime resource까지 회수하는 과정을 복원합니다.

이 문서는 exact SHA를 순서대로 확인해 설계 → 구현 → 실패 → 수정 → 검증을 복원합니다.

### 직접 연결되는 불변식

- suspended user는 보호된 HTTP mutation과 새 realtime admission을 수행할 수 없습니다.
- user status 변경과 대응하는 administrator action은 PostgreSQL에서 함께 commit되거나 함께 rollback됩니다.
- ban 성공 뒤 현재 authoritative realtime connection은 즉시 제거되고 동일 session으로 새 ticket도 발급받지 못합니다.

## 2. 핵심 질문

- 초기 suspension enforcement는 어떤 HTTP routes와 WebSocket handshake에 적용됐고 어떤 route에는 빠져 있었습니까?
- 초기 PostgreSQL `setUserBan`의 두 statement 사이에서 audit insert가 실패하면 어떤 partial state가 남습니까?
- CHECK constraint failure injection은 어느 call을 실패시키고 rollback 후 어떤 rows를 직접 조회합니까?
- `GameHub.revokeUser`가 heartbeat, snapshot buffer, queue, tournament waiter, input gate, room side, socket, presence를 어떤 순서로 정리합니까?
- DB commit과 in-memory revocation 사이에는 왜 하나의 transaction이 존재할 수 없으며 남는 failure window는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 지정 브랜치 ancestry에 속하는지 확인합니다.
- 각 SHA의 diff와 parent 또는 직전 관련 SHA의 실제 state를 구분합니다.
- 아래 commit-specific 파일·함수·SQL·테스트를 확인하고 generic 설명으로 대체하지 않습니다.
- Fix는 이전 가정 → 실제 실패 → root cause → corrected invariant → regression evidence로 연결합니다.
- 실행하지 않은 test/command는 실행한 것처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread의 최종 owner, state transition, guarantee와 non-guarantee를 정리합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `42033a6f2f3a` | `feat(admin): 감사 가능한 사용자 상태 API 추가` | A | AUTH, REALTIME, PERSISTENCE | account suspension을 presentation flag가 아닌 enforced authorization state와 audit read model로 만듭니다. |
| 2 | `bf797871007c` | `feat(admin): 감사 기록과 상태 변경 UI 추가` | B | AUTH, WEB | administrator interface를 reason-bearing status mutation과 audit-capable API에 연결합니다. |
| 3 | `e07726592df5` | `test(app): 전체 서비스 흐름 검증` | B | AUTH, REALTIME, PERSISTENCE | 전체 서비스 회귀 중 administrator audit와 suspension enforcement 경로를 함께 검증합니다. |
| 4 | `d0137660cd9f` | `fix(db): 차단 감사 기록을 원자적으로 저장` | A | AUTH, PERSISTENCE, RISK | user status update와 administrator-action insert를 하나의 PostgreSQL transaction으로 이동합니다. |
| 5 | `9106abc10d0e` | `test(db): 차단 감사 기록 atomicity 검증` | A | AUTH, PERSISTENCE, TEST | audit insert를 deterministic constraint violation으로 실패시켜 user status rollback을 검증합니다. |
| 6 | `40e5c520d49c` | `fix(auth): 정지된 사용자의 열린 연결 폐기` | A | AUTH, REALTIME, TOURNAMENT | ban persistence 직후 `GameHub.revokeUser`로 current socket과 realtime ownership을 회수합니다. |
| 7 | `454cbf2c95e0` | `test(auth): 계정 정지의 기존 WebSocket 차단 검증` | A | AUTH, REALTIME, TEST | 실제 server/socket에서 suspension 뒤 기존 connection close와 새 ticket 거부를 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(admin): 감사 가능한 사용자 상태 API 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `42033a6f2f3a` |
| Importance | A |
| Tags | AUTH, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | account suspension을 presentation flag가 아닌 enforced authorization state와 audit read model로 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts` WebSocket session resolution 뒤 `user.status !== 'active'` close branch를 확인합니다.
- lobby chat, friend request, tournament create/join에 추가된 `isActive`/`suspended` guard와 빠진 read-only/admin route를 구분합니다.
- `/admin/actions` → `repo.listAdminActions`와 PostgreSQL row mapper의 actor/target lookup을 추적합니다.
- PostgreSQL `setUserBan`이 여전히 users UPDATE와 admin_actions INSERT를 별도 statement로 실행하는 점을 이후 atomicity fix와 연결합니다.
- memory `adminActions.unshift`와 target status mutation 순서 및 backend parity 한계를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | user status와 admin_actions schema/insert는 존재했지만 suspended 상태는 대부분 presentation data였고, protected mutations와 WebSocket admission에 일관되게 적용되지 않았습니다. |
| 해결하려던 문제 | ban된 account가 session을 이용해 chat/friend/tournament write나 새 realtime connection을 계속 시작할 수 있고, 운영자가 감사 기록을 조회할 API도 없었습니다. |
| 핵심 결정 | WebSocket authentication 직후 active status를 검사하고 주요 state-changing HTTP routes에 status guard를 추가했습니다. admin actions list API, repository read, row mapper, memory action storage도 연결했습니다. |
| 입력 → 상태 전이 → 출력 | HTTP mutation은 currentUser→active check→repository mutation으로 진행합니다. WebSocket은 session resolve→active check→inactive면 close 1008, active면 hub.connect입니다. admin audit read는 role check→listAdminActions→actor/target mapping입니다. |
| ownership/lifetime/cleanup | repository user row가 status를 소유하고 HTTP/WS admission boundary가 capability를 판정합니다. audit rows는 PostgreSQL 또는 memory list가 소유합니다. |
| failure/rollback/retry | PostgreSQL status UPDATE 뒤 audit INSERT가 실패하면 별도 autocommit 때문에 status만 남을 수 있습니다. 이미 hub에 연결된 socket은 이 handshake guard를 다시 통과하지 않으므로 계속 살아 있습니다. |
| 보장하는 것 | 이 SHA 이후 suspended user는 지정된 HTTP writes와 새 WebSocket admission을 통과하지 못하고 admin은 감사 목록을 읽을 수 있습니다. |
| 보장하지 않는 것 | admin status check, audit atomicity, 이미 열린 socket 회수, 모든 route의 중앙화된 status policy는 아직 보장하지 않습니다. |
| 후속 연결 | `bf797871007c`이 조치 사유 입력과 audit list를 browser admin 화면에 연결하고, `e07726592df5`가 초기 enforcement를 API test로 확인합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `bf797871007c` — `feat(admin): 감사 기록과 상태 변경 UI 추가`

### 5.2. `feat(admin): 감사 기록과 상태 변경 UI 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `bf797871007c` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | administrator interface를 reason-bearing status mutation과 audit-capable API에 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/src/app/admin/page.tsx`의 users/actions/reason/message local state와 initial `Promise.all` load를 확인합니다.
- `toggleUser`가 trimmed reason을 `setUserStatus`에 전달하고 users state를 교체한 뒤 `getAdminActions`로 refresh하는 순서를 추적합니다.
- `apps/web/src/lib/api.ts`의 `getAdminActions`, reason parameter 추가와 request body를 확인합니다.
- abort/cancellation, optimistic rollback, transaction enforcement가 browser에 없다는 점을 server invariant와 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | server는 reason을 저장하고 actions를 읽을 수 있었지만 admin UI는 고정 reason으로 상태만 바꾸고 감사 목록을 표시하지 않았습니다. |
| 해결하려던 문제 | 운영자가 조치 사유를 명시하고 결과 audit trail을 확인할 browser workflow가 필요했습니다. |
| 핵심 결정 | admin page에 reason input과 action list state를 추가하고 users/actions를 함께 load합니다. 상태 변경 성공 뒤 target row를 갱신하고 audit list를 다시 가져옵니다. |
| 입력 → 상태 전이 → 출력 | page mount → users/actions Promise.all → local state. toggle → status/reason PATCH → users map replace → actions GET refresh → result message render. |
| ownership/lifetime/cleanup | 이 SHA에서는 component local state가 browser projection을 소유합니다. authoritative status/audit는 여전히 server repository입니다. |
| failure/rollback/retry | 초기 load 중 하나라도 실패하면 공통 권한 메시지로 처리됩니다. mutation 성공 뒤 audit refresh 실패는 catch로 들어가 이미 반영된 server status와 UI message가 어긋날 수 있습니다. |
| 보장하는 것 | 사용자 입력 reason이 API request에 포함되고 returned audit actions를 화면에 표시합니다. |
| 보장하지 않는 것 | server atomicity, stale request cancellation, optimistic rollback, live connection revocation은 보장하지 않습니다. |
| 후속 연결 | `e07726592df5`의 admin API test가 reason 저장과 suspended target의 write 거부를 확인합니다. |

비교 기준:
- 직전 관련 SHA: `42033a6f2f3a` — `feat(admin): 감사 가능한 사용자 상태 API 추가`
- 다음 관련 SHA: `e07726592df5` — `test(app): 전체 서비스 흐름 검증`

### 5.3. `test(app): 전체 서비스 흐름 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `e07726592df5` |
| Importance | B |
| Tags | AUTH, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | 전체 서비스 회귀 중 administrator audit와 suspension enforcement 경로를 함께 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 이 Thread에서는 `apps/api/src/admin.test.ts`에 추가된 actions GET와 blocked lobby chat assertion만 범위로 삼습니다.
- ban request reason `smoke`가 `/admin/actions` 첫 row에 나타나는 경로를 추적합니다.
- ban된 target의 기존 bearer token으로 `/chat/lobby`를 호출해 403을 확인하는 순서를 기록합니다.
- 같은 commit의 tournament, browser, smoke 변경은 이 Thread의 직접 근거와 분리합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 suspension/audit 구현은 있었지만 ban→audit visibility→기존 session write denial을 한 test에서 연결하지 않았습니다. |
| 해결하려던 문제 | status가 단순 response field로만 바뀌고 실제 후속 mutation을 막지 못하는 회귀를 검출해야 했습니다. |
| 핵심 결정 | 기존 admin route test에 audit actions 조회와 banned target의 lobby chat 시도를 추가해 reason과 403을 확인했습니다. |
| 입력 → 상태 전이 → 출력 | admin ban(reason smoke) → admin token으로 `/admin/actions` → first reason assertion → target의 기존 token으로 `/chat/lobby` → 403 assertion. |
| ownership/lifetime/cleanup | memory repository와 Fastify app fixture가 test state를 소유합니다. broad commit의 다른 tests와 독립된 admin test block입니다. |
| failure/rollback/retry | audit reason이 저장되지 않거나 active guard가 빠지면 실패합니다. status/audit SQL partial failure는 주입하지 않습니다. |
| 보장하는 것 | memory/API 경로에서 ban audit가 읽히고 ban 전 발급된 target credential의 lobby write가 즉시 거부됩니다. |
| 보장하지 않는 것 | PostgreSQL transaction, WebSocket, 모든 protected route, process/network는 증명하지 않습니다. |
| 후속 연결 | `d0137660cd9f`은 initial PostgreSQL implementation의 partial-write 위험을 transaction으로 수정합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | ban action은 audit reason을 남기고 기존 session의 protected HTTP write를 거부합니다. |
| 재현하는 failure/boundary | ban 전 발급된 target bearer token의 ban 후 재사용입니다. |
| test technique | Fastify injection과 memory repository를 사용하는 broad integration regression의 admin 부분입니다. |
| 통과하는 production path | admin ban route → memory `setUserBan`/adminActions → actions route → target lobby-chat active guard. |
| 증명하는 것 | reason visibility와 HTTP write 403을 연결합니다. |
| 증명하지 않는 것 | PostgreSQL atomicity, real socket, 모든 capabilities는 검증하지 않습니다. |
| 후속 회귀 방지 | status가 UI field로만 남고 HTTP mutation enforcement가 사라지는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `bf797871007c` — `feat(admin): 감사 기록과 상태 변경 UI 추가`
- 다음 관련 SHA: `d0137660cd9f` — `fix(db): 차단 감사 기록을 원자적으로 저장`

### 5.4. `fix(db): 차단 감사 기록을 원자적으로 저장`

| 항목 | 값 |
| --- | --- |
| SHA | `d0137660cd9f` |
| Importance | A |
| Tags | AUTH, PERSISTENCE, RISK |
| Source에서 확정된 역할 | user status update와 administrator-action insert를 하나의 PostgreSQL transaction으로 이동합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts` `PostgresRepository.setUserBan`의 parent에서 두 `.execute(this.db)`가 분리돼 있던 상태를 확인합니다.
- 이 SHA에서 `this.db.transaction().execute` callback과 동일 `transaction` object로 UPDATE/INSERT를 실행하는지 확인합니다.
- `return toPublicUser(firstRow(result))`가 transaction callback 안에서 실행되고 성공 시에만 commit되는 순서를 기록합니다.
- memory implementation에는 같은 변경이 없고 failure rollback 모델이 다르다는 점을 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | PostgreSQL `setUserBan`은 users UPDATE를 commit한 뒤 admin_actions INSERT를 별도로 실행했습니다. 두 번째 statement 실패 시 banned status만 남았습니다. |
| 해결하려던 문제 | 감사 기록 없이 account status만 바뀌는 durable partial state가 생겨 운영 조치의 추적 가능성과 복구 판단이 깨질 수 있었습니다. |
| 핵심 결정 | 두 SQL을 하나의 Kysely transaction callback 안으로 옮기고 동일 transaction handle로 실행했습니다. |
| 입력 → 상태 전이 → 출력 | transaction begin → target users UPDATE returning → admin_actions INSERT → mapped user return → callback 성공 시 commit. 어느 statement든 throw하면 callback reject와 rollback. |
| ownership/lifetime/cleanup | PostgreSQL transaction이 status row와 audit row의 commit/rollback을 함께 소유합니다. application caller는 하나의 Promise 결과만 봅니다. |
| failure/rollback/retry | UPDATE/INSERT/mapping 중 예외가 발생하면 transaction이 rollback됩니다. target-not-found의 정확한 error 표현은 별도 typed domain error로 정리되지 않았습니다. |
| 보장하는 것 | PostgreSQL에서 한 `setUserBan` 호출의 status와 audit insert는 함께 commit되거나 함께 rollback됩니다. |
| 보장하지 않는 것 | memory backend의 failure atomicity, DB commit 뒤 realtime revoke, audit read consistency across replicas는 보장하지 않습니다. |
| 후속 연결 | `9106abc10d0e`가 audit insert만 의도적으로 실패시켜 users update도 rollback되는지 실제 PostgreSQL에서 검증합니다. |

#### 최소 코드 근거

- SHA: `d0137660cd9f`
- 파일: `packages/db/src/index.ts`
- 위치: `PostgresRepository.setUserBan`
- 확인 목적: status와 audit write의 단일 transaction

```ts
return this.db.transaction().execute(async (transaction) => {
  const result = await sql<UserRow>`update users ... returning *`.execute(transaction);
  await sql`insert into admin_actions ...`.execute(transaction);
  return toPublicUser(firstRow(result));
});
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | 순차 await한 두 SQL이면 하나의 논리적 operation으로 충분하다는 가정이었습니다. |
| 실제 실패 또는 위험 | audit INSERT 실패 뒤 users status UPDATE는 이미 commit돼 감사 없는 ban이 남을 수 있었습니다. |
| Root cause | 두 statement가 동일 transaction/connection 범위에 없었습니다. |
| 수정된 invariant | status mutation과 audit action은 하나의 PostgreSQL transaction outcome을 공유합니다. |
| 변경 코드 | `packages/db/src/index.ts` `PostgresRepository.setUserBan`의 `this.db.transaction().execute`. |
| Regression evidence | `9106abc10d0e`의 CHECK constraint failure injection test. |

비교 기준:
- 직전 관련 SHA: `e07726592df5` — `test(app): 전체 서비스 흐름 검증`
- 다음 관련 SHA: `9106abc10d0e` — `test(db): 차단 감사 기록 atomicity 검증`

### 5.5. `test(db): 차단 감사 기록 atomicity 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `9106abc10d0e` |
| Importance | A |
| Tags | AUTH, PERSISTENCE, TEST |
| Source에서 확정된 역할 | audit insert를 deterministic constraint violation으로 실패시켜 user status rollback을 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/postgres.integration.test.ts`의 `rolls back a ban when its audit record cannot be written` test를 확인합니다.
- test 전용 CHECK constraint `reason <> 'force audit failure'`를 `admin_actions`에 추가하는 SQL을 기록합니다.
- `repository.setUserBan`이 constraint name으로 reject되는 assertion과, 이후 users status/banned_at 및 admin_actions count 직접 조회를 추적합니다.
- 실패가 users UPDATE가 아니라 두 번째 audit INSERT에서 발생한다는 점을 statement 순서로 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | transaction fix는 있었지만 실제 두 번째 write 실패 시 첫 번째 update가 rollback되는 deterministic evidence가 없었습니다. |
| 해결하려던 문제 | happy-path test만으로는 transaction handle 누락이나 한 statement의 잘못된 connection 사용을 검출할 수 없었습니다. |
| 핵심 결정 | 격리 PostgreSQL의 admin_actions에 특정 reason을 거부하는 CHECK constraint를 추가하고 그 reason으로 `setUserBan`을 호출했습니다. reject 뒤 target row와 action count를 SQL로 직접 확인했습니다. |
| 입력 → 상태 전이 → 출력 | actor/target 생성 → test CHECK constraint 설치 → `setUserBan(...,'force audit failure')` → INSERT constraint error → users row query active/null → admin_actions count 0. |
| ownership/lifetime/cleanup | isolated database fixture가 임시 constraint와 rows의 lifecycle을 소유합니다. production repository method를 그대로 호출합니다. |
| failure/rollback/retry | 예상 constraint name이 아니거나 promise가 resolve되거나 status/action rows가 남으면 test가 실패합니다. |
| 보장하는 것 | 실제 PostgreSQL에서 두 번째 statement의 deterministic failure가 첫 번째 users update까지 rollback함을 증명합니다. |
| 보장하지 않는 것 | process crash timing, network disconnect, memory backend, DB commit 후 realtime side effect는 검증하지 않습니다. |
| 후속 연결 | `40e5c520d49c`은 durable ban이 성공한 뒤 이미 열린 realtime control channel을 회수합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | status update와 audit insert는 원자적입니다. |
| 재현하는 failure/boundary | 첫 SQL 성공 뒤 두 번째 audit INSERT만 CHECK constraint로 실패하는 경계입니다. |
| test technique | 실제 PostgreSQL schema mutation을 이용한 deterministic failure-injection integration test입니다. |
| 통과하는 production path | `setUserBan` transaction → users UPDATE → failing admin_actions INSERT → rollback → direct SQL observation. |
| 증명하는 것 | target은 active/banned_at null이고 audit rows는 0입니다. |
| 증명하지 않는 것 | memory backend, crash recovery, realtime revocation은 검증하지 않습니다. |
| 후속 회귀 방지 | 둘 중 하나가 transaction handle 대신 base DB handle을 사용하는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `d0137660cd9f` — `fix(db): 차단 감사 기록을 원자적으로 저장`
- 다음 관련 SHA: `40e5c520d49c` — `fix(auth): 정지된 사용자의 열린 연결 폐기`

### 5.6. `fix(auth): 정지된 사용자의 열린 연결 폐기`

| 항목 | 값 |
| --- | --- |
| SHA | `40e5c520d49c` |
| Importance | A |
| Tags | AUTH, REALTIME, TOURNAMENT |
| Source에서 확정된 역할 | ban persistence 직후 `GameHub.revokeUser`로 current socket과 realtime ownership을 회수합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts` 두 admin status routes에서 `repo.setUserBan` await 뒤 `if (banned) hub.revokeUser(id)`가 호출되는 순서를 확인합니다.
- `apps/api/src/gameHub.ts`의 `clientsByUser` lookup이 current authoritative client만 선택하는지 확인합니다.
- `revokeUser`의 heartbeat stop, snapshot buffer close, queue/tournament waiter leave, clients maps delete, input gate release를 순서대로 추적합니다.
- room client인 경우 `sideFor`와 `reserveRoomSide`로 reconnect/forfeit lifecycle에 넘기는 이유를 확인합니다.
- WebSocket OPEN일 때 close code 4003/reason과 마지막 presence broadcast를 확인하고, no-client/unban 분기를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | ban 뒤 새 HTTP write와 새 WebSocket admission은 거부됐지만 ban 전에 이미 `GameHub`에 연결된 socket은 status를 다시 검사하지 않아 명령을 계속 보낼 수 있었습니다. |
| 해결하려던 문제 | account suspension과 realtime authority 사이에 revocation gap이 있어 suspended user가 열린 control channel을 유지했습니다. |
| 핵심 결정 | durable ban 성공 후 `hub.revokeUser`를 호출하고, user-indexed current client의 모든 runtime ownership을 해제한 뒤 4003으로 socket을 닫도록 했습니다. |
| 입력 → 상태 전이 → 출력 | admin request → requireAdmin → repository transaction commit → banned이면 `revokeUser` → current client lookup → timers/buffers/queue/waiters/maps/input gate cleanup → room side reservation → socket 4003 close → presence broadcast → response. |
| ownership/lifetime/cleanup | repository가 durable status/audit를 먼저 확정합니다. `GameHub`가 current socket, heartbeat, snapshots, matchmaking/tournament memberships, input budget와 room-side handoff를 소유합니다. |
| failure/rollback/retry | 대상 connection이 없으면 no-op입니다. unban은 revoke하지 않습니다. DB commit과 in-memory cleanup은 하나의 transaction이 아니므로 process failure가 그 사이에 발생하면 durable ban만 남고 현재 socket cleanup은 완료되지 않을 수 있습니다. |
| 보장하는 것 | 정상 실행 경로에서 ban 성공 직후 current authoritative connection이 hub indexes와 input/queue ownership을 잃고 4003으로 닫힙니다. |
| 보장하지 않는 것 | DB와 realtime side effect의 분산 원자성, socket close frame 전달 보장, 다른 process/instance의 connection revocation은 보장하지 않습니다. |
| 후속 연결 | `454cbf2c95e0`이 실제 server와 `ws` client로 open socket close와 새 ticket 거부를 함께 검증합니다. |

#### 최소 코드 근거

- SHA: `40e5c520d49c`
- 파일: `apps/api/src/gameHub.ts`
- 위치: `GameHub.revokeUser`
- 확인 목적: 현재 realtime client가 보유한 resource의 회수 순서

```ts
client.heartbeat.stop();
client.snapshots.close();
this.leaveQueue(client);
this.leaveTournamentWaiters(client);
this.clients.delete(client.id);
this.clientsByUser.delete(userId);
this.inputGate.releaseUser(userId);
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | 새 요청/handshake에서 status를 검사하면 suspension enforcement가 충분하다는 가정이었습니다. |
| 실제 실패 또는 위험 | ban 전에 열린 WebSocket은 다시 인증하지 않아 계속 game commands를 보낼 수 있었습니다. |
| Root cause | account status mutation path가 `GameHub`의 current connection ownership에 알리지 않았습니다. |
| 수정된 invariant | ban 성공은 durable status change와 현재 realtime authority 회수를 순서대로 수행합니다. |
| 변경 코드 | `apps/api/src/app.ts` admin handlers; `apps/api/src/gameHub.ts` `revokeUser`. |
| Regression evidence | `454cbf2c95e0`의 real WebSocket close 4003 및 new-ticket 403 test. |

비교 기준:
- 직전 관련 SHA: `9106abc10d0e` — `test(db): 차단 감사 기록 atomicity 검증`
- 다음 관련 SHA: `454cbf2c95e0` — `test(auth): 계정 정지의 기존 WebSocket 차단 검증`

### 5.7. `test(auth): 계정 정지의 기존 WebSocket 차단 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `454cbf2c95e0` |
| Importance | A |
| Tags | AUTH, REALTIME, TEST |
| Source에서 확정된 역할 | 실제 server/socket에서 suspension 뒤 기존 connection close와 새 ticket 거부를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/admin.test.ts` setup이 `app.ready` 대신 loopback ephemeral port `app.listen`을 사용하도록 바뀐 이유를 확인합니다.
- target dev login → cookie로 `/auth/ws-ticket` → real `ws` client open → close promise 등록 순서를 추적합니다.
- admin ban injection 뒤 close `{ code: 4003, reason: 'account suspended' }`와 동일 old cookie의 새 ticket 403 assertion을 구분합니다.
- afterEach가 sockets를 terminate하고 app/repository를 닫는 cleanup ownership을 확인합니다.
- queue/room 내부 상태, PostgreSQL transaction, DB→hub failure window를 직접 검증하지 않는 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | live revocation code는 추가됐지만 실제 WebSocket transport가 close event를 받고 old session이 새 ticket을 얻지 못하는 end-to-end regression evidence가 없었습니다. |
| 해결하려던 문제 | mock socket이나 method-level assertion만으로는 HTTP ban, ticket auth, hub connection index, close frame의 통합을 검증할 수 없었습니다. |
| 핵심 결정 | Fastify를 실제 loopback port에 listen하고 `ws` client를 ticket으로 연결한 뒤 admin ban route를 호출했습니다. close code/reason과 새 ticket 403을 모두 확인했습니다. |
| 입력 → 상태 전이 → 출력 | target login → ws-ticket issue → real socket open → close listener → admin ban → repository ban + hub revoke → socket close 4003 → old cookie로 ws-ticket request 403. |
| ownership/lifetime/cleanup | test의 sockets 배열이 client cleanup을 소유하고 Fastify listen/close와 memory repository lifecycle을 suite가 소유합니다. |
| failure/rollback/retry | socket open/error와 close promise가 transport failure를 드러냅니다. cleanup은 closed가 아니면 terminate합니다. |
| 보장하는 것 | 한 process의 실제 HTTP/WebSocket 통합에서 ban이 open socket을 4003으로 닫고 기존 session의 신규 realtime admission을 403으로 차단함을 증명합니다. |
| 보장하지 않는 것 | PostgreSQL atomicity, multi-instance revocation, 모든 room/queue cleanup field, network packet delivery 보장은 증명하지 않습니다. |
| 후속 연결 | 이 Thread의 최종 회귀 증거입니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | suspended user는 현재 realtime control channel과 새 WebSocket admission을 모두 잃습니다. |
| 재현하는 failure/boundary | ban 전에 이미 ticket-authenticated socket이 OPEN인 상태입니다. |
| test technique | loopback Fastify server와 real `ws` client를 사용하는 process-local integration regression입니다. |
| 통과하는 production path | login/session → ws-ticket → WebSocket handshake/GameHub connect → admin ban → repository → `revokeUser` → close; 이후 ticket route active-status gate. |
| 증명하는 것 | close code 4003/reason과 old-cookie new-ticket 403입니다. |
| 증명하지 않는 것 | PostgreSQL, multi-process, queue/room 모든 internal state, DB-hub atomicity는 검증하지 않습니다. |
| 후속 회귀 방지 | ban route에서 `hub.revokeUser` 호출이나 cleanup/close code가 빠지는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `40e5c520d49c` — `fix(auth): 정지된 사용자의 열린 연결 폐기`
- 이 Thread의 최종 상태와 비교합니다.

## 6. Invariant ledger

| Invariant | 도입/수정 SHA | 변화 | 검증/후속 보호 |
| --- | --- | --- | --- |
| suspended user는 protected write와 새 WS admission을 잃는다 | `42033a6f2f3a` | HTTP active guards와 handshake close로 도입 | `e07726592df5`에서 old token write 403 검증 |
| status와 audit action은 원자적이다 | `d0137660cd9f` | 동일 PostgreSQL transaction으로 수정 | `9106abc10d0e`에서 second-write failure injection으로 검증 |
| ban은 현재 realtime authority를 회수한다 | `40e5c520d49c` | GameHub resource cleanup과 close 4003 도입 | `454cbf2c95e0` real socket/ticket regression으로 검증 |

## 7. Failure → Fix → Test

| Failure/Risk | Fix | Test | 연결 근거 |
| --- | --- | --- | --- |
| ban 후 기존 HTTP credential이 write를 계속 수행 | `42033a6f2f3a` | `e07726592df5` | target old token의 lobby chat 403 |
| users UPDATE만 commit되고 audit INSERT 실패 | `d0137660cd9f` | `9106abc10d0e` | CHECK constraint로 audit insert 실패 후 active/0 actions 확인 |
| ban 전에 열린 WebSocket이 계속 명령 가능 | `40e5c520d49c` | `454cbf2c95e0` | real socket close 4003와 new ticket 403 |

## 8. Ownership·state·responsibility 변화

| 관심사 | 최종 owner와 변화 |
| --- | --- |
| durable account status/audit | PostgreSQL `setUserBan` transaction이 status와 action row의 commit/rollback을 소유합니다. |
| HTTP/WS capability decision | API active guards와 ticket/handshake boundary가 current status를 읽어 새 work를 거부합니다. |
| live realtime authority | `GameHub.clientsByUser`가 current client를 선택하고 heartbeat/snapshot/queue/waiter/input/room/socket cleanup을 소유합니다. |
| browser projection | admin page local state는 users/actions/reason presentation만 소유하며 authoritative state가 아닙니다. |
| failure boundary | DB commit과 GameHub revoke는 순차 side effects이며 하나의 distributed transaction으로 묶이지 않습니다. |

## 9. Thread 최종 상태

### 최종 owner

repository transaction이 durable status와 audit record를 소유하고, API가 새 HTTP/WS capability를 판정하며, GameHub가 이미 열린 current realtime connection과 관련 in-memory resource를 회수합니다.

### 최종 execution flow

active administrator request가 ban을 요청하면 repository transaction이 target status와 audit action을 함께 commit합니다. ban인 경우 handler가 `GameHub.revokeUser`를 호출해 current client의 timers, buffers, memberships, indexes, input state를 해제하고 room side를 예약한 뒤 socket을 4003으로 닫습니다. 이후 같은 session의 새 ticket request는 current banned status 때문에 403입니다.

### 최종 guarantee

정상 단일-process 경로에서 감사 없는 partial ban을 막고, protected HTTP/WS admission과 이미 열린 authoritative connection을 모두 차단하는 회귀 증거가 연결됩니다.

### 최종 non-guarantee

DB commit과 in-memory revocation의 분산 원자성, multi-instance connection broadcast, close frame 전달, process crash 직후 자동 보상은 이 Thread가 보장하지 않습니다.

## 10. Learning completion check

- [x] 모든 Commit map SHA의 exact diff와 관련 code path를 확인했습니다.
- [x] Importance별로 A commit은 failure/ownership을 깊게, B commit은 Thread 역할 중심으로 기록했습니다.
- [x] Fix와 regression test를 이전 failure에 연결했습니다.
- [x] 실제 실행 증거와 code inspection을 구분했습니다.
- [ ] Runtime test command 실행 — 로컬 checkout을 만들 GitHub network/DNS가 차단되어 실행하지 못했습니다. 따라서 문서에는 실행 성공을 주장하지 않습니다.
===== END FILE: 03-suspension-audit-atomicity-and-live-revocation.md =====

===== BEGIN FILE: README.md =====
# 신원·권한·계정 수명주기

등록 사용자 session, server-side logout/revocation, explicit role assignment, administrator authorization, suspension audit, 이미 열린 realtime connection 회수까지 등록 계정의 수명주기를 다룹니다.

## 범위

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- 상태: Phase 1 repository audit 후 동결된 authoritative scaffold
- 제품 전달 제외: Docker/Caddy/Compose image, CI delivery job, release artifact, dependency patch, media asset 생성은 이 카테고리의 학습 대상에서 제외합니다.

## Phase 1 category audit 결과

- 카테고리 경계는 적절합니다. durable registered-account session, role, suspension과 revocation에 한정합니다.
- cookie-only session·one-time WebSocket ticket과 guest transient trust domain은 이미 `05-core-realtime-architecture`의 source-defined Thread이므로 이 카테고리에 복제하지 않습니다.
- 기존 Thread 수는 3개로 유지합니다. 별도 독립 story를 추가할 repository evidence는 없었습니다.
- `42033a6f2f3a`와 `bf797871007c`는 role provisioning보다 suspension/audit lifecycle에 직접 속하므로 Thread 2에서 Thread 3으로 이동했습니다.
- 초기 ban→audit visibility→protected-write denial을 검증하는 `e07726592df5`를 Thread 3에 추가했습니다.
- 각 Thread의 commit 순서를 실제 branch chronology에 맞추고 generic 조사 문장을 concrete file/function/SQL/test task로 교체했습니다.
- source classification의 subject, importance, tags와 role은 변경하지 않았습니다.

## Thread

1. [Server session·logout·인증 migration](01-server-session-logout-and-auth-migrations.md)
2. [명시적 role과 관리자 권한](02-explicit-role-and-administrator-authorization.md)
3. [계정 정지 audit atomicity와 live revocation](03-suspension-audit-atomicity-and-live-revocation.md)

## 사용 원칙

- 각 문서의 frozen Commit map 순서를 유지합니다.
- exact SHA의 코드와 parent 또는 직전 관련 state를 확인합니다.
- 다른 카테고리에서 같은 SHA를 다루더라도 이 문서의 account-lifecycle 질문에 필요한 근거만 기록합니다.
- final HEAD code를 과거 SHA 설명에 소급하지 않습니다.
- 실행하지 않은 command는 성공한 것으로 기록하지 않습니다.

## Phase 2 completion evidence

| 검증 항목 | 결과 |
| --- | --- |
| Exact SHA inspection | 20개 frozen commit을 각 exact SHA의 GitHub commit diff로 확인했습니다. |
| Branch ancestry | source classification이 `72ac4c1870f`→`71c5c13480f0`의 complete linear branch history임을 확인했고, `71c5c13480f0`이 현재 `web/ft_transcendence` head의 ancestor임을 compare로 확인했습니다. |
| Runtime commands | 실행하지 않았습니다. sandbox의 GitHub DNS/network 제한으로 checkout과 dependency installation이 불가능했습니다. |
| Evidence discipline | runtime 성공을 주장하지 않고 code inspection과 repository test implementation만 기록했습니다. |
| Structure | frozen scaffold 4개 파일과 completed 4개 파일이 같은 상대 경로·Commit map을 가집니다. |
===== END FILE: README.md =====

