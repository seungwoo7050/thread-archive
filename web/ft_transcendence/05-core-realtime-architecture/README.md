# Core Realtime Architecture — Development Threads

이 디렉터리는 `web/ft_transcendence` 브랜치의 실시간 게임 핵심부를 여덟 개의 개발 문제로 나누어 복원한 문서입니다. 문서는 커밋을 시간순으로 요약하지 않습니다. 하나의 기능이나 실패 조건이 어떤 가정에서 시작해 어떤 불변 조건으로 정리되었는지를, 해당 SHA의 diff와 그 시점의 소스를 근거로 설명합니다.

## 문서 구성

| 문서 | 다루는 개발 문제 |
| --- | --- |
| [01-authoritative-deterministic-game-mechanics.md](./01-authoritative-deterministic-game-mechanics.md) | 서버 권위 물리, 순수 상태 전이, 중복 규칙 제거, golden replay |
| [02-cookie-identity-websocket-admission.md](./02-cookie-identity-websocket-admission.md) | 쿠키 세션, 단발 WebSocket ticket, handshake 자원 상한, 로그 비밀값 제거 |
| [03-versioned-realtime-protocol-and-monotonic-state.md](./03-versioned-realtime-protocol-and-monotonic-state.md) | 양방향 runtime codec, 프로토콜 버전, 입력·snapshot 단조 증가 |
| [04-atomic-idempotent-match-finalization.md](./04-atomic-idempotent-match-finalization.md) | 결과 멱등성, 참가자 잠금, rating·tournament 원자적 반영, 재시도 |
| [05-room-lifecycle-connection-replacement-and-recovery.md](./05-room-lifecycle-connection-replacement-and-recovery.md) | 방 상태 기계, active connection 교체, 15초 좌석 예약, 몰수패·복구 |
| [06-matchmaking-reservation-ownership-and-rollback.md](./06-matchmaking-reservation-ownership-and-rollback.md) | rating 기반 매칭, `queued`/`matched` 예약, AI fallback, 모든 종료 경로의 해제 |
| [07-guest-mode-as-isolated-transient-trust-domain.md](./07-guest-mode-as-isolated-transient-trust-domain.md) | 게스트 전용 인증·자원 제한·매칭 격리·비영속 결과 |
| [08-runtime-timing-backpressure-drain-and-operational-evidence.md](./08-runtime-timing-backpressure-drain-and-operational-evidence.md) | bounded scheduling, heartbeat, input gate, latest snapshot, drain, shutdown, 부하·장애 증거 |

## 읽는 방법

각 문서는 독립된 Thread입니다. 예를 들어 protocol versioning과 room recovery는 서로 의존하는 지점이 있지만, 하나가 다른 하나의 “다음 단계”라는 뜻은 아닙니다. 필요한 문제부터 읽고, 문서 안에서 명시한 관련 커밋만 함께 추적하면 됩니다.

커밋 표의 SHA·제목·importance·tags는 기존 workbook의 연동 metadata를 유지했습니다. 본문은 기존 골격을 축약한 것이 아니라 exact SHA의 실제 변경을 다시 대조해 작성했습니다. 같은 커밋에 Thread와 무관한 변경이 섞인 경우에는 해당 부분을 제외했습니다.

## 검증 범위

- 조사 대상은 `web/ft_transcendence` 브랜치로 제한했습니다.
- 각 설명은 표시된 SHA 또는 그 parent와의 diff, 해당 SHA 시점의 source/test를 기준으로 합니다.
- 후대 구현을 과거 커밋의 보장으로 소급하지 않았습니다.
- repository의 테스트 코드는 읽어 assertion과 실행 경로를 확인했지만, 이 작업 환경에서는 checkout·build·test·load suite를 직접 실행하지 않았습니다. 따라서 문서에서 “테스트가 고정한다”는 표현은 테스트 코드가 요구하는 계약을 뜻하며, 이 환경에서의 실행 성공을 뜻하지 않습니다.
