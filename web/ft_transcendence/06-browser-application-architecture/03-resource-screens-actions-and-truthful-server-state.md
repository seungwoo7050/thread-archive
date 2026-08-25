# Thread: 서버 기반 화면과 진실한 상태 표현

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread는 lobby, dashboard, leaderboard, tournament, profile, admin route를 실제 API와 action에 연결하는 과정과, 그 초기 구현이 request failure를 sample success로 위장하던 문제를 교정하는 과정을 다룹니다.

여섯 화면은 서로 다른 도메인 데이터를 표시하지만 같은 설계 패턴으로 시작했습니다.

```text
sample value를 initial state에 배치
    -> mount에서 API 요청
    -> 성공하면 sample을 server data로 교체
    -> 실패하면 catch를 비우거나 sample을 그대로 유지
```

이 패턴은 demo를 빠르게 보이게 하지만 사용자는 현재 보고 있는 값이 서버 결과인지 fallback인지 알 수 없습니다. admin 화면은 권한 없는 실패에도 sample 사용자 목록을 표시했고, dashboard의 점수 graph는 server history가 아니라 고정 polyline이었습니다. profile은 존재하지 않는 handle도 sample user를 복제해 유효한 사용자처럼 보여 줬습니다.

최종 fix는 모든 화면을 동일한 시각 형태로 만드는 대신, 각 resource가 **미확정·실패·비어 있음·성공** 중 어디에 있는지 명시적으로 드러내도록 바꿉니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ea1f1b7ba543` | `feat(web): 로그인 사용자 로비 화면 구성` | B | REALTIME, WEB, OPERATIONS | authenticated home route를 lobby read model과 action을 갖는 화면으로 만듭니다. |
| 2 | `cbe876359d31` | `feat(web): 플레이어 대시보드 구현` | B | WEB | dashboard read model을 표시하는 route를 만듭니다. |
| 3 | `cb295396771f` | `feat(web): 순위표 화면 추가` | B | WEB | leaderboard projection을 표시합니다. |
| 4 | `4370ac3162b2` | `feat(web): 토너먼트 대진표 화면 추가` | B | TOURNAMENT, WEB | tournament list/create를 연결한 초기 bracket screen을 만듭니다. |
| 5 | `0afc0a0694bd` | `feat(web): 공개 프로필 화면 추가` | B | WEB | handle-scoped profile route를 만듭니다. |
| 6 | `5e11e944244d` | `feat(web): 관리자 화면 추가` | B | WEB | user/status를 표시하는 protected admin screen을 만듭니다. |
| 7 | `051eac1b4aee` | `feat(profile): 친구 요청 동작 연결` | B | AUTH, WEB | profile target을 friendship mutation에 연결합니다. |
| 8 | `bfea82733512` | `feat(admin): 사용자 상태 변경 동작 연결` | B | AUTH, WEB | admin status control을 authenticated mutation에 연결합니다. |
| 9 | `a4f665fd2999` | `feat(tournament): 생성과 참가 동작 연결` | B | AUTH, TOURNAMENT | selected tournament state와 create/join action을 연결합니다. |
| 10 | `8b2679d9e190` | `test(e2e): 화면 action의 실제 API 연결 검증` | B | REALTIME, TOURNAMENT, WEB | AI 시작, match chat, friendship, tournament, admin action을 browser에서 검증합니다. |
| 11 | `e0ef3fec89a6` | `feat(chat): 로비 채팅 입력 화면 추가` | B | WEB | 로비 채팅 writer와 bounded local history를 HomePage에 연결하고 lobby read fallback 일부를 제거합니다. |
| 12 | `4f9b3b312d0e` | `fix(lobby): 로비 상태 표현 개선` | B | REALTIME | 고정 wait/주간 증가 문구를 제거하고 server `LobbyStats`를 화면 지표의 source로 사용합니다. |
| 13 | `cd3787eefd6a` | `feat(chat): 로비 채팅과 접속 상태 실시간 반영` | B | AUTH, REALTIME, WEB | HomePage가 lobby WebSocket을 직접 소유해 chat fan-out과 presence-triggered reload를 local state에 반영합니다. |
| 14 | `be31566ac0fd` | `fix(web): 로그인 화면의 sample fallback 제거` | A | AUTH, TOURNAMENT, WEB | authenticated/server-backed screen이 failure를 sample success data로 대체하지 않게 합니다. |

## 1. 화면 표면을 먼저 만든 단계

### 로비 — `ea1f1b7ba543`

authenticated home content가 `AppShell` 안으로 들어가고, hero, score/online/wait cards, player list, lobby chat, queue/AI links를 제공합니다. 하지만 실제 server field와 무관한 presentation도 함께 들어갑니다.

```tsx
<StatCard label="승리" value={String(me.wins)} hint="이번 주 +2" />
<StatCard label="온라인" value={String(players.length)} />
<StatCard label="대기" value="30초" hint="평균 예상 시간" />
```

players/chat는 이전 commit의 sample initial state를 계속 사용합니다. 즉 이 화면은 layout과 domain surface를 만들었지만 모든 숫자가 authoritative하다는 보장은 없습니다.

### 대시보드 — `cbe876359d31`

`DashboardPage`는 `sampleDashboard`로 시작한 뒤 `getDashboard().then(setDashboard)`를 호출합니다. win/loss/winRate/recent matches는 response로 교체되지만, “점수 흐름” SVG는 고정 좌표입니다.

```tsx
<polyline points="0,180 80,164 ... 640,78" ... />
```

request rejection을 처리하지 않으므로 sample dashboard가 그대로 남습니다. 사용자는 네트워크 실패와 실제 전적을 구분할 수 없습니다.

### 순위표 — `cb295396771f`

`sampleLeaderboard`를 initial state로 두고 mount에서 `getLeaderboard` 결과로 교체합니다. row identity는 `entry.user.id`, 표시 값은 rank/rating/winRate입니다. 빈 순위표나 request error branch는 없습니다.

### 토너먼트 — `4370ac3162b2`

`sampleTournaments`를 표시하면서 `createTournament("새로운 퐁퐁 컵")` 성공 결과를 앞에 삽입합니다. 선택된 tournament를 별도 state로 소유하지 않은 초기 구현이라 bracket은 `items[0]`을 암묵적으로 사용합니다. list button도 아직 선택 동작과 연결되지 않습니다.

### 공개 프로필 — `0afc0a0694bd`

route params가 resolve되면 sample user를 handle로 찾고, 없으면 첫 sample을 복제해 handle/display name만 바꿉니다.

```tsx
setUser(
  sampleUsers.find((item) => item.handle === resolved)
  ?? { ...sampleUsers[0], handle: resolved, displayName: "퐁마스터" }
);
```

따라서 존재하지 않는 handle도 정상 profile처럼 보입니다. “선수 번호”도 `handle.length`이고 플레이 스타일 설명은 고정 문장입니다. 이 커밋은 profile page surface를 만들지만 resource 진실성을 보장하지 않습니다.

### 관리자 화면 — `5e11e944244d`

admin users request가 성공하면 server list를 쓰고, 실패는 무시합니다.

```tsx
const [users, setUsers] = useState<PublicUser[]>(sampleUsers);
apiFetch<{ users: PublicUser[] }>("/admin/users")
  .then((result) => setUsers(result.users))
  .catch(() => undefined);
```

권한 부족이나 서버 장애가 sample 운영 데이터로 보이는 가장 위험한 사례입니다. “검토” button도 inert합니다.

## 2. user intent를 실제 mutation에 연결

세 action commit은 화면이 보이는 것에서 실제 상태 변경을 요청하는 단계로 이동합니다.

### `051eac1b4aee` — 친구 요청

profile params에서 얻은 handle을 `requestFriend(handle)`에 전달합니다. 성공하면 response user의 display name으로 message를 만들고, 실패하면 로그인/대상 확인 안내를 표시합니다. 같은 커밋에서 `getProfile(resolved)`도 호출하지만 실패를 무시하므로 sample profile은 여전히 남을 수 있습니다. share button은 실제로 연결하지 않고 명시적으로 disabled 상태로 바뀝니다.

### `bfea82733512` — 관리자 상태 변경

active user는 banned로, banned user는 active로 바꾸는 target status를 계산하고 `setUserStatus` 결과로 해당 row를 교체합니다.

```tsx
const updated = await setUserStatus(
  user.id,
  user.status === "active" ? "banned" : "active"
);
setUsers((current) => current.map((item) =>
  item.id === updated.id ? updated : item
));
```

실패 message는 표시하지만 초기 users request 실패 시에는 “샘플 목록”을 보여 준다고 명시합니다. 즉 mutation 연결과 read truthfulness는 아직 분리된 문제입니다.

### `a4f665fd2999` — tournament 선택·생성·참가

화면이 선택 tournament state를 갖고, create 성공 시 list와 selection을 갱신하며 join action은 선택된 tournament ID를 endpoint에 전달합니다. response를 바탕으로 UI를 갱신하지만 request pending의 중복 submit, cache invalidation, 다른 screen과의 동기화는 아직 component 책임입니다.

### `8b2679d9e190` — action E2E의 범위

Playwright가 AI 시작, match chat, friendship, tournament create/join, admin status action 같은 실제 controls를 클릭하고 결과 text를 확인합니다. 이 검증은 inert control을 방지하고 endpoint wiring을 통과하지만, 초기 sample value가 request failure 뒤 남지 않는다는 것은 증명하지 않습니다. 화면의 heading이나 success message가 보여도 read model의 출처가 sample일 수 있기 때문입니다.

## 3. 로비를 write·realtime 화면으로 확장

### `e0ef3fec89a6` — HTTP chat writer와 fallback 일부 제거

HomePage에 `chatInput`, `notice`, submit form이 추가됩니다. 입력은 trim 후 빈 값이면 중단하고, `sendLobbyChat` 성공 message를 최대 20개 local history에 넣습니다.

동시에 `getLobby` helper의 catch/sample 반환이 제거되어 API failure가 caller까지 올라옵니다. 그러나 HomePage의 players/chat initial state는 여전히 sample입니다. catch message도 “샘플 화면을 표시합니다”라고 명시하므로 adapter fallback은 없어졌지만 presentation fallback은 남아 있습니다.

### `4f9b3b312d0e` — 고정 지표를 `LobbyStats`로 교체

로비 response의 `stats`를 state에 저장하고, online/playing/queue/room/average wait를 표시합니다.

```tsx
<StatCard
  label="대기"
  value={stats?.averageWaitSeconds == null
    ? "대기 없음"
    : `${stats.averageWaitSeconds}초`}
  hint={`큐 ${stats?.queuedPlayers ?? 0}명 · 방 ${stats?.activeRooms ?? 0}개`}
/>
```

“이번 주 +2”, 고정 “30초” 같은 문구가 제거됩니다. `stats === null`일 때는 “확인 중” 또는 0을 사용하므로, 아직 명시적 error model은 아니지만 최소한 fabricated metric을 실제 수치처럼 표시하지 않습니다.

### `cd3787eefd6a` — lobby socket owner

HomePage가 socket ref와 `loadLobby` callback을 소유합니다. 당시 credential 방식인 browser token으로 socket을 열고 다음 event를 처리합니다.

- `chat.message` + `scope === "lobby"`: 같은 ID를 제거한 뒤 최근 20개로 제한
- `presence.changed`: lobby HTTP read를 다시 실행
- `error`: notice 표시

chat send는 socket이 OPEN이면 realtime command를 보내고, 아니면 HTTP helper로 fallback합니다. cleanup은 handler를 null로 만들고 OPEN/CONNECTING socket을 닫은 뒤 ref identity를 확인해 비웁니다.

이 커밋은 실시간 반영을 만들지만 raw JSON cast, browser token, page-owned socket이라는 한계를 그대로 갖습니다. credential과 transport lifecycle의 후속 교정은 Thread 02·04에 속합니다.

## 4. `be31566ac0fd` — 실패를 성공 데이터처럼 보이지 않게 하기

이 A급 fix의 핵심은 sample 파일을 단순히 삭제한 것이 아니라 **resource state의 표현을 바꾼 것**입니다.

### 변경 전 공통 상태

```text
state = sample data
request success -> real data로 교체
request failure -> sample이 남거나 catch 무시
```

### 변경 후 공통 상태

```text
state = null 또는 []
request 시작 -> loading / 미확정
request success + data -> success
request success + empty -> empty
request failure -> error
```

화면별 결과는 다음과 같습니다.

| 화면 | 이전 오해 | 수정 뒤 표현 |
| --- | --- | --- |
| lobby | sample players/chat가 실제 접속자처럼 남음 | empty arrays와 error notice, server data만 표시 |
| dashboard | sample 전적과 고정 성공 화면 | nullable data, loading/error branch, 실제 recent matches만 표시 |
| leaderboard | sample ranking 유지 | loading/error/empty를 분리 |
| tournament | sample 대회와 bracket 유지 | server list가 없으면 empty, action 실패 message |
| profile | 없는 handle도 sample user로 합성 | nullable profile, load failure/없음 표시 |
| admin | 401/403도 sample users 표시 | empty/error 상태, 권한 실패를 운영 데이터처럼 보이지 않음 |

특히 admin과 profile은 “아무것도 모름”을 fake object로 채우지 않습니다. `null`은 단순한 빈 값이 아니라 resource가 아직 확정되지 않았다는 상태가 됩니다.

이 fix가 보장하지 않는 것도 있습니다.

- component별 `useEffect`와 local state 중복은 남아 있습니다.
- 동일 resource를 여러 화면에서 공유하거나 stale time을 관리하지 않습니다.
- mutation 뒤 다른 화면의 data를 자동 갱신하지 않습니다.
- session expiry에서 private state를 일괄 제거하지 않습니다.

이 문제들은 Thread 06의 React Query migration이 해결합니다.

## 최종 화면 계약

| 사용자에게 보이는 상태 | 의미 | 허용되는 source |
| --- | --- | --- |
| loading / 확인 중 | request가 끝나지 않음 | local request state |
| error / 권한 없음 | request 실패 또는 거부 | 실제 error branch |
| empty | request는 성공했지만 항목 없음 | server response의 빈 collection/null |
| success data | 표시 가능한 resource 존재 | server response 또는 실제 mutation result |
| realtime update | socket event가 현재 resource에 적용됨 | scope와 identity가 맞는 event |

sample fixture는 개발·test data일 수 있지만, authenticated production screen의 정상 성공값으로 사용되지 않습니다.

## Thread 경계

- HTTP response schema·cookie credential·structured error는 Thread 02가 소유합니다.
- lobby/game socket을 reusable transport로 분리하는 작업은 Thread 04·05가 소유합니다.
- component local fetch를 canonical cache로 옮기는 작업은 Thread 06이 소유합니다.
- guest mode에서 일부 화면·interaction을 의도적으로 숨기는 정책은 Thread 08이 소유합니다.

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. 이 환경에서는 E2E나 project command를 실행하지 않았습니다.
