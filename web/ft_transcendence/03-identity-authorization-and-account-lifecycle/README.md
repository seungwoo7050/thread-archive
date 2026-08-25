# 신원·권한·계정 수명주기

등록 사용자의 session 발급·폐기, 관리자 role provisioning, account suspension, audit atomicity, 이미 열린 realtime connection의 권한 회수까지 다룬다.

- Repository: `seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- 조사 기준 branch head: `67441b68a1b2641f1b9ebfe582909ab5f9c7f5cd`
- 대상 commit: 20개
- Thread 수: 3개

## Thread

| 번호 | 문서 | 핵심 개발 문제 |
| ---: | --- | --- |
| 1 | [server session — logout 폐기와 인증 migration](01-server-session-logout-and-auth-migrations.md) | client cookie 제거와 server credential 폐기의 차이, 인증 전환 시 legacy session 정리 |
| 2 | [명시적 role과 관리자 권한](02-explicit-role-and-administrator-authorization.md) | handle 기반 승격 제거, 별도 provisioning, 정지된 기존 관리자 session의 권한 재평가 |
| 3 | [계정 정지 — audit 원자성과 열린 WebSocket 권한 회수](03-suspension-audit-atomicity-and-live-revocation.md) | status·audit의 transaction, 신규 작업 차단, 이미 열린 realtime authority 회수 |

## 세 Thread가 함께 만드는 최종 상태

```text
[registered identity]
  ordinary login/upsert → role=user
  explicit provisioning → role 변경

[session]
  repository가 token 발급·해석
  logout → 선택된 token 삭제
  auth migration → legacy PostgreSQL sessions 전체 폐기

[administrator request]
  current session user
    → active status 확인
    → admin role 확인

[account suspension]
  users status + admin_actions를 한 transaction으로 commit
    → 보호된 새 HTTP/WS work 거부
    → current GameHub client의 runtime ownership 회수
    → socket close 4003
```

## 핵심 불변 조건

| 불변 조건 | 주된 commit |
| --- | --- |
| logout 성공 후 같은 token은 다시 인증되지 않는다 | `252befef9527`, `bc789124b20b` |
| auth migration은 session만 제거하고 선택된 사용자·경기 history를 유지한다 | `b93910708330`, `0649b63a1ca9` |
| user가 고른 handle은 관리자 role을 만들지 않는다 | `45225adcfcd9`, `dae31d4a223c` |
| 관리자 capability는 current `active + admin` 상태에서만 유효하다 | `c577fe2603e3`, `aa037c5291fe` |
| PostgreSQL ban과 audit action은 함께 commit되거나 rollback된다 | `d0137660cd9f`, `9106abc10d0e` |
| ban은 새 admission뿐 아니라 현재 authoritative WebSocket도 회수한다 | `40e5c520d49c`, `454cbf2c95e0` |

## 조사·작성 기준

- 각 commit은 exact SHA의 diff와 해당 SHA 시점 source/test를 기준으로 설명했다.
- 동일 commit에 섞인 tournament, broad E2E, unrelated UI 변경은 각 Thread 질문에 필요하지 않으면 제외했다.
- commit subject, SHA, importance, tags, Thread 구분은 기존 metadata를 유지했다.
- repository와 subject의 표현이 어긋난 경우 subject를 바꾸지 않고 실제 diff의 동작을 명시했다. 예를 들어 `c577fe2603e3`은 제목과 달리 login 발급이 아니라 `requireAdmin`의 request authorization을 수정한다.
- 세 Thread의 마지막 SHA가 작성 시점 `web/ft_transcendence` head의 조상임을 확인했다.
- runtime test는 실행하지 않았다. 따라서 문서의 test 설명은 source에 구현된 fixture·production path·assertion·비보장 범위를 분석한 결과이며, 실행 성공을 주장하지 않는다.

## 카테고리 경계

다음 항목은 인접하지만 이 카테고리의 세 Thread에서 다루지 않는다.

- one-time WebSocket ticket의 hash 저장·단회 소비와 cookie-only admission
- guest transient identity와 등록 계정의 격리
- role 변경 승인·audit workflow와 multi-factor administrator policy
- multi-instance socket revocation을 위한 pub/sub 또는 outbox
- production migration orchestration, backup/restore, 배포 rollback
