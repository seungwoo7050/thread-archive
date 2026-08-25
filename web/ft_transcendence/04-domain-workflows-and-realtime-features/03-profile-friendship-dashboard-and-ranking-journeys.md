# 읽기 화면에서 샘플과 추정값을 걷어내기

원문 Thread: `Profile, friendship, dashboard, and ranking journeys`

## 이 Thread가 다루는 문제

프로필·순위표·대시보드는 대부분 읽기 화면이지만, “무엇을 사실로 보여 줄 것인가”를 결정합니다. 초기 구현은 서버 read model이 완성되기 전 sample 사용자, 고정 rating 그래프, 공식으로 만든 최고 연승, 임의의 플레이 스타일을 화면에 사용했습니다. 더 큰 문제는 서버 요청이 실패한 뒤에도 이 값들이 fallback으로 남아 **실제 사용자 상태처럼 보였다는 점**입니다.

이 Thread의 변화는 단순한 UI 연결이 아닙니다.

- repository가 profile·leaderboard·recent-match·dashboard projection을 소유합니다.
- 브라우저는 요청 실패와 빈 결과를 sample 사실로 덮지 않습니다.
- bounded 최근 경기에서 계산한 지표는 그 범위를 UI에 명시합니다.
- profile mutation과 session 종료는 영향을 받는 cache를 명시적으로 무효화·제거합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `c5b96a06925c` | `feat(db): 프로필 조회와 변경 저장 구현` | B | PERSISTENCE | 공개 profile 조회·인증 profile 변경·active user 목록을 repository에 추가한다. |
| 2 | `0364c42f776b` | `feat(db): 순위 조회 구현` | B | PERSISTENCE | rating·wins 정렬과 rank·win rate를 가진 leaderboard projection을 만든다. |
| 3 | `c7ea1ff241c8` | `feat(db): 최근 경기와 대시보드 조회 구현` | B | PERSISTENCE | 최근 경기와 dashboard read model을 추가하지만 최고 연승은 아직 추정값이다. |
| 4 | `0bcc487d949f` | `feat(api): 프로필과 친구 리소스 라우트 추가` | B | PERSISTENCE | public read와 identity-bound mutation을 HTTP resource로 노출한다. |
| 5 | `cbe876359d31` | `feat(web): 플레이어 대시보드 구현` | B | WEB | DashboardSummary를 소비하는 화면을 만들지만 sample과 고정 그래프가 남아 있다. |
| 6 | `cb295396771f` | `feat(web): 순위표 화면 추가` | B | WEB | shared leaderboard contract를 화면에 렌더링한다. |
| 7 | `0afc0a0694bd` | `feat(web): 공개 프로필 화면 추가` | B | WEB | handle 기반 공개 profile route를 만들지만 존재하지 않는 사용자도 sample로 꾸민다. |
| 8 | `051eac1b4aee` | `feat(profile): 친구 요청 동작 연결` | B | AUTH, WEB | 공개 profile 조회와 인증 friend-request mutation을 연결한다. |
| 9 | `51e66cf1df80` | `fix(profile): 공개 프로필 상태 표현 개선` | B | PERSISTENCE, WEB | 임의 플레이 스타일 대신 서버가 준 최근 경기를 표시한다. |
| 10 | `8d79139a32da` | `fix(dashboard): 경기 상태 표현 개선` | B | WEB | 고정 그래프를 현재 rating과 최근 match delta에서 복원한 점들로 교체한다. |
| 11 | `be31566ac0fd` | `fix(web): 로그인 화면의 sample fallback 제거` | A | AUTH, TOURNAMENT, WEB | 인증·서버 기반 화면이 요청 실패를 sample 사실로 대체하지 못하게 한다. |
| 12 | `035b97ca7c58` | `fix(db): 최근 경기에서 최고 연승 계산` | B | PERSISTENCE | 공식·상수를 제거하고 최근 경기의 실제 win/loss 순서에서 최고 연승을 계산한다. |
| 13 | `6b661420e060` | `test(db): 최고 연승 계산 검증` | B | PERSISTENCE, TEST | newest-first 입력의 역순 처리와 loss reset 규칙을 고정한다. |
| 14 | `7fe29f991a9b` | `fix(dashboard): 연승 지표 설명 정정` | C | - | ‘이번 시즌’을 ‘최근 경기’로 바꿔 bounded read 범위를 숨기지 않는다. |
| 15 | `3c6c9134ee94` | `fix(dashboard): 빈 rating history를 정확히 표시` | B | PERSISTENCE, WEB | 경기 없는 사용자에게 가짜 두 점 그래프를 만들지 않는다. |
| 16 | `c17e7ad0fd84` | `feat(web): profile과 friend 조회 query 추가` | B | WEB | schema 검증 browser helper와 scoped React Query key/invalidation을 추가한다. |
| 17 | `8bc4d0cc32bd` | `test(web): profile과 friend 조회 규칙 검증` | B | AUTH, WEB, TEST | endpoint·payload·cache key·invalidation·logout cleanup을 검증한다. |

## 1. 서버가 제공할 읽기 모델 만들기

### `c5b96a06925c` — profile read와 mutation

PostgreSQL과 memory repository에 ID/normalized handle 조회, optional displayName/avatar update, active user 목록이 추가됩니다. profile 변경은 입력하지 않은 필드를 기존 row에서 유지하고, 인증된 caller에는 `SessionUser`, 공개 조회에는 `PublicUser`를 반환합니다.

이 시점의 `listOnlineUsers`는 이름과 달리 `status = 'active'` 계정을 rating 순으로 읽고 `online: true`로 projection합니다. 실제 WebSocket 연결 여부는 알지 못합니다. 이 부정확성은 로비 Thread의 `8debb1ea3ad3`에서 GameHub client map으로 교정됩니다.

### `0364c42f776b` — leaderboard projection

repository가 rating 내림차순, 동률일 때 wins 기준으로 사용자들을 정렬하고 배열 위치에서 rank를 계산합니다. win rate는 승패 합계가 0일 때 0으로 처리하고, 그렇지 않으면 승리 비율을 반올림합니다.

화면이 같은 사용자 목록을 다시 정렬하거나 순위를 추정하지 않게 했다는 점이 중요합니다. rank는 repository read model의 일부가 됩니다.

### `c7ea1ff241c8` — 최근 8경기와 대시보드, 그리고 첫 번째 거짓 지표

최근 경기는 최신 순으로 최대 8개를 읽고 현재 사용자 관점에서 win/loss, 상대, 점수, rating delta를 구성합니다. 그러나 `bestStreak`는 실제 경기 순서를 계산하지 않았습니다.

- PostgreSQL 구현은 누적 wins/losses에서 만든 공식에 `+ 3`을 더했습니다.
- memory 구현은 상수 `3`을 반환했습니다.
- 일부 memory recent match의 상대는 실제 row와 무관하게 `"AI"`였습니다.

즉 API가 값을 반환한다는 사실과 그 값이 저장된 사건에서 계산됐다는 사실은 달랐습니다. 이 commit은 read model의 뼈대를 만들지만, 최고 연승의 사실성은 후속 fix가 담당합니다.

### `0bcc487d949f` — HTTP resource

공개 profile 조회, 인증 dashboard 조회, friends 조회와 friend request 같은 route가 추가됩니다. public read와 identity-bound mutation을 분리하고 repository가 반환한 projection을 HTTP 계약으로 전달합니다.

친구 관계의 canonical pair·동시성 불변식 자체는 이 Thread의 주제가 아닙니다. 여기서는 profile journey가 그 mutation을 어떻게 호출하고 결과를 다시 읽는지만 다룹니다.

## 2. 첫 화면들은 서버 값과 sample 값을 섞었다

### `cbe876359d31` — 대시보드와 고정 SVG

화면은 `DashboardSummary`를 받을 수 있게 됐지만 초기 state에는 sample dashboard가 있었고 rating 그래프는 데이터와 무관한 고정 path였습니다. 서버 요청이 실패해도 사용자는 실제 대시보드를 본 것처럼 느낄 수 있었습니다.

### `cb295396771f` — 순위표

rank·wins·rating·win rate를 표시하는 화면이 추가됐습니다. 그러나 서버 결과를 읽기 전 또는 실패 시 sample leaderboard가 남는 구조였습니다. UI 구조 자체는 유효하지만 실패 상태가 “가짜 정상 상태”로 변환됐습니다.

### `0afc0a0694bd` — 존재하지 않는 handle에도 사용자를 만들어 주는 공개 profile

초기 route는 요청한 handle을 sample 목록에서 찾고, 없으면 첫 sample user를 복사해 handle과 displayName만 바꿨습니다.

```ts
setUser(
  sampleUsers.find((item) => item.handle === resolved)
    ?? { ...sampleUsers[0], handle: resolved, displayName: "퐁마스터" }
);
```

따라서 어떤 문자열로 접근해도 rating·승패·아바타를 가진 사용자가 나타났습니다. “not found”가 표현될 수 없는 모델입니다. 플레이 스타일과 “최근 30일” 같은 문구도 저장된 근거가 없었습니다.

### `051eac1b4aee` — 실제 profile/friend API 연결, 그러나 fallback은 유지

동적 handle로 profile을 조회하고 친구 요청 버튼이 인증 mutation을 호출합니다. 다만 조회 실패를 무시하거나 sample user를 유지하는 경로가 남아 있어, API를 연결했어도 화면의 사실 원천은 아직 단일하지 않았습니다.

### `51e66cf1df80` — 임의 플레이 스타일 대신 최근 경기

public profile 응답의 recent matches를 state에 저장하고 화면에 표시합니다. 이로써 근거 없는 플레이 스타일 문단은 사라집니다. 그러나 요청 실패 시 sample 화면을 표시한다는 정책은 여전히 남아 있었고, 다음 A급 fix가 이를 제거합니다.

## 3. 보이는 값과 실제 근거를 일치시키기

### `8d79139a32da` — rating graph를 최근 경기 delta에서 역산

고정 SVG 대신 현재 rating에서 최근 match의 rating delta를 거꾸로 빼며 과거 점수를 복원한 뒤, 시간 순서로 그래프 점을 만듭니다. 이 계산은 저장된 전체 rating history가 아니라 **최대 8개 recent match**에 대해서만 유효합니다.

초기 수정에는 경기 이력이 없을 때 현재 rating 앞에 임의의 한 점을 더 만드는 fallback이 남았습니다. 따라서 “고정 그래프 제거”와 “빈 이력 정직성”은 같은 commit에서 완전히 끝나지 않습니다.

### `be31566ac0fd` — sample fallback 제거

인증·서버 기반 화면 전반에서 sample state와 “실패하면 sample을 표시”하는 catch를 제거합니다. Thread와 직접 관련된 profile·dashboard·leaderboard는 loading, request failure, not-found, empty collection을 구분해 표시합니다.

이 commit의 중요도 A는 변경량보다 의미에 있습니다.

> 서버가 대답하지 못한 상태를, 서버가 확인한 정상 상태처럼 꾸미지 않는다.

같은 cross-cutting commit에 토너먼트·로비 변경도 섞여 있지만 이 문서에서는 profile/dashboard/ranking의 read-model 정직성만 다룹니다.

### `035b97ca7c58` — 최고 연승을 실제 결과 순서로 계산

최근 경기는 newest-first로 반환되므로 계산 함수는 이를 reverse해 오래된 경기부터 순회합니다. win이면 current streak를 늘리고 best를 갱신하며, loss이면 current를 0으로 되돌립니다. PostgreSQL 공식과 memory 상수는 모두 이 함수 결과로 교체됩니다.

```text
newest-first API: [win, win, loss, win]
chronological:    [win, loss, win, win]
current:           1 → 0 → 1 → 2
best:              1 → 1 → 1 → 2
```

결과 `2`는 이 사용자의 전체 생애 최고 연승이 아니라 최근 read window 안의 최고 연승입니다.

### `6b661420e060` — ordering과 reset regression

테스트는 newest-first 형태의 win/win/loss/win 입력에서 결과 2를 기대합니다. 단순히 win 개수를 세거나 reverse를 빠뜨리거나 loss 뒤 streak를 유지하는 구현을 구별합니다. source상 production helper를 통과하는 것을 확인했지만 테스트는 실행하지 않았습니다.

### `7fe29f991a9b` — 한 줄짜리 C commit이 닫는 의미 차이

대시보드 hint를 “이번 시즌”에서 “최근 경기”로 바꿉니다. 계산 코드는 이미 올바르더라도 UI가 더 넓은 범위를 주장하면 사용자에게는 여전히 잘못된 지표입니다. 작은 변경이므로 C이지만, bounded read model의 계약을 화면에 드러냅니다.

### `3c6c9134ee94` — 빈 history는 빈 evidence다

경기가 없을 때 현재 rating 주변의 가짜 두 점을 만들던 fallback을 제거합니다. chart는 점이 없음을 명시적으로 표시하고, 현재 rating 자체와 rating 변화 이력의 존재를 혼동하지 않습니다.

## 4. cache도 identity와 같은 사실 범위를 따라야 한다

### `c17e7ad0fd84` — query key와 invalidation ownership

own profile, public profile, friends를 schema-validated browser helper와 React Query options로 옮깁니다. public profile은 handle을 key에 포함하고, own profile과 friends는 인증 사용자에게 묶인 별도 key를 가집니다.

profile 변경 뒤에는 단순히 현재 profile 카드만 갱신하면 안 됩니다. displayName/avatar가 다음 read model에도 들어갈 수 있기 때문에 관련 query를 무효화합니다.

- `me`, own/public profile
- lobby
- dashboard
- friends
- leaderboard
- tournaments
- admin user projection

session expiry 또는 logout에서는 private cache를 제거해 다음 사용자가 이전 사용자의 profile/friends/dashboard를 볼 수 없게 합니다.

### `8bc4d0cc32bd` — HTTP와 cache 규칙을 함께 고정

테스트는 own profile GET과 PATCH body, friend request endpoint를 확인하는 데서 끝나지 않습니다. handle별 profile key, mutation 뒤 정확한 invalidation 집합, session 종료 뒤 private query 제거까지 검증합니다.

이 테스트가 증명하지 않는 것은 서버 측 friendship 동시성이나 여러 브라우저 간 즉시 동기화입니다. 브라우저 한 query client 안에서 읽기 소유권과 무효화 규칙을 고정합니다.

## 최종 사실 원천

| 화면 값 | 최종 원천 | 명시된 한계 |
| --- | --- | --- |
| profile identity·rating·record | repository public/session projection | 요청 실패를 sample로 대체하지 않음 |
| leaderboard rank·win rate | repository 정렬·계산 | 응답 시점 snapshot |
| recent matches | 최신 최대 8개 persisted match | 전체 이력 아님 |
| rating graph | 현재 rating + 최근 match delta 역산 | history가 없으면 빈 상태 |
| 최고 연승 | 최근 경기 chronological win/loss scan | “이번 시즌/전체”로 주장하지 않음 |
| friend/profile 화면 cache | scoped React Query key | mutation/logout 때 명시적 invalidation/removal |

이 Thread의 핵심은 “화면을 API에 연결했다”가 아닙니다. **서버가 확인하지 못한 값은 사실처럼 표시하지 않고, 서버가 제공한 값도 그 데이터 범위 이상으로 해석하지 않는 것**입니다.

## 조사 범위

각 설명은 `web/ft_transcendence`의 표시 SHA diff와 해당 시점 source를 기준으로 작성했습니다. test runner는 실행하지 않았습니다.
