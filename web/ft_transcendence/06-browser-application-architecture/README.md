# Browser application architecture Development Threads — 재작성본

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Thread: 8개
- Commit mapping: 83개
- 고유 SHA: 82개

## 이 재작성본의 기준

기존 `completed` 문서의 Thread 구분, commit/SHA, subject, importance, tags, 역할을 초기 metadata로 사용했습니다. 본문은 기존 “완료 기준 → 확인할 코드 → 학습자 기록 → 비교 기준” 반복 형식을 계승하지 않고, 각 Thread가 실제로 해결하는 개발 문제에 맞춰 다시 구성했습니다.

- 작은 feature/refactor commit은 맡은 역할과 남긴 한계만 짧게 설명했습니다.
- A급 fix·test는 이전 가정, 실패 가능성, root cause, 수정된 불변 조건, 코드 경계, 증명/비증명 범위를 연결했습니다.
- 같은 commit에 섞인 다른 subsystem 변경은 해당 Thread에서 제외했습니다.
- final HEAD 구현을 과거 SHA에 소급하지 않았습니다.
- 실제로 실행하지 않은 build/test 결과는 기록하지 않았습니다.

## 카테고리 경계

이 카테고리는 browser application이 직접 소유하는 다음 항목을 다룹니다.

- Next.js runtime, document root, shared application shell
- navigation identity와 current-session 소비
- HTTP adapter, runtime response validation, cookie credential
- server-backed resource screen과 mutation feedback
- game reducer, WebSocket client, hook migration
- React Query cache ownership와 session invalidation
- authoritative snapshot projection, canvas interpolation, input intent
- guest browser capability와 transient result/recovery presentation

server simulation, room lifecycle producer, database transaction, guest session/ticket store, deployment orchestration 전체는 browser가 소비하는 contract를 설명할 때만 언급합니다.

## Thread 목록

| 순서 | 문서 | 핵심 문제 | Commit 수 |
| ---: | --- | --- | ---: |
| 1 | [`01-application-shell-auth-entry-and-navigation-identity.md`](01-application-shell-auth-entry-and-navigation-identity.md) | runtime·shell·auth entry와 변하는 profile URL에서 stable navigation identity 유지 | 11 |
| 2 | [`02-browser-http-adapter-runtime-validation-and-cookie-credentials.md`](02-browser-http-adapter-runtime-validation-and-cookie-credentials.md) | unchecked JSON/localStorage token을 cookie·schema·AbortSignal 경계로 교정 | 6 |
| 3 | [`03-resource-screens-actions-and-truthful-server-state.md`](03-resource-screens-actions-and-truthful-server-state.md) | server failure를 sample success로 위장하지 않는 화면 상태 | 14 |
| 4 | [`04-game-connection-reducer-and-transport-client.md`](04-game-connection-reducer-and-transport-client.md) | reducer와 ticket/socket/reconnect resource owner 분리 | 9 |
| 5 | [`05-game-connection-hook-migration-and-legacy-removal.md`](05-game-connection-hook-migration-and-legacy-removal.md) | replacement-first hook migration과 duplicate owner 제거 | 11 |
| 6 | [`06-react-query-cache-ownership-and-invalidation.md`](06-react-query-cache-ownership-and-invalidation.md) | server state를 stable key·exact invalidation·session eviction cache로 통합 | 10 |
| 7 | [`07-authoritative-snapshot-rendering-and-input.md`](07-authoritative-snapshot-rendering-and-input.md) | server snapshot만 렌더링하고 versioned room-scoped input만 전송 | 10 |
| 8 | [`08-guest-browser-policy-and-transient-results.md`](08-guest-browser-policy-and-transient-results.md) | demo capability, transient result, active-room/reconnect recovery | 12 |

## Thread 사이의 관계

이 문서들은 단일 선형 단계가 아닙니다. 몇 가지 dependency만 존재하고 나머지는 병렬 개발 단위입니다.

```text
Thread 01: runtime / shell / navigation
       ├───────────────┐
       ↓               ↓
Thread 02: HTTP trust  Thread 03: resource screens
       ↓               ↓
       └──────→ Thread 06: canonical server-state cache

Thread 07: first play rendering/input
       ↓
Thread 04: reducer + transport owner
       ↓
Thread 05: hook adoption + legacy removal

Thread 08: guest policy
       ├─ Thread 01 shell/navigation 소비
       ├─ Thread 02 guest API/ticket boundary 소비
       └─ Thread 04 reconnect capability 소비
```

Thread 07의 초기 play implementation이 Thread 04·05보다 역사적으로 앞서지만, 각 문서는 서로 다른 문제를 소유합니다. 따라서 모든 문서를 “앞 Thread의 완성 뒤 다음 Thread 시작”으로 해석하면 안 됩니다.

## 의도적 중복 SHA

`4f5199097284`는 유일한 중복입니다.

- Thread 04: `GameSocketClient`, reducer, hook, play page의 fresh-ticket reconnect와 duplicate match 차단
- Thread 08: `HomePage`, `demoPolicy`의 active-room route recovery와 transient result notice

같은 diff 안에서 서로 다른 browser subsystem이 수정되었으므로 SHA를 두 Thread에 두되, 코드와 설명 범위는 겹치지 않게 분리했습니다.

## 최종 browser ownership 요약

| 관심사 | 최종 owner |
| --- | --- |
| document metadata/global style | `RootLayout` |
| common layout/navigation identity | `AppShell` + stable navigation policy |
| HTTP request/error/runtime parse | `apiFetch` + endpoint helper |
| durable session transport | HttpOnly cookie를 포함하는 fetch |
| server-backed resource data | one `QueryClient` + stable query keys |
| game application state | pure reducer behind `useGameConnection` |
| ticket/socket/reconnect/input sequence | `GameSocketClient` |
| page presentation/user form state | route/page component |
| authoritative game scene | accepted server `GameSnapshot` |
| canvas smoothing | copied snapshot buffer + position-only interpolation |
| guest surface | centralized `demoPolicy` + middleware/page consumers |

## 검사 범위와 제한

지정된 `web/ft_transcendence` branch의 exact SHA diff와 당시 source/test 변경을 GitHub repository에서 정적으로 검사했습니다. 다른 branch의 코드를 참고하지 않았고, final HEAD 구현을 과거 commit 설명에 사용하지 않았습니다.

이 환경에서는 repository를 로컬 checkout할 수 없어 다음 command는 실행하지 않았습니다.

- `pnpm build`, typecheck, Vitest
- Playwright/E2E
- 실제 WebSocket·cookie runtime

따라서 문서에서 “테스트가 검증한다”는 표현은 해당 SHA의 test source와 assertion이 겨냥하는 범위를 뜻하며, 이 작업에서 test pass를 새로 확인했다는 뜻이 아닙니다.
