# 04 — 도메인 워크플로와 실시간 기능

- Repository: `seungwoo7050/42-archive`
- 조사 브랜치: `web/ft_transcendence`
- 원본 폴더: `development-thread-workbook/04-domain-workflows-and-realtime-features/completed`
- 재작성 범위: README 1개와 Development Thread 7개
- 포함 commit: 총 70개

## 재작성 기준

기존 문서의 Thread 구분, SHA, 제목, importance, tags, 역할은 초기 자료로 유지했습니다. 본문은 워크북의 반복 섹션과 자가 점검표를 답안처럼 보존하지 않고, 각 Thread가 실제로 설명하는 **개발 문제·실패·수정·검증·불변 조건**을 중심으로 다시 구성했습니다.

모든 구현 설명은 지정 브랜치의 해당 SHA diff와 그 시점 source에 근거합니다. 기존 역할 설명과 source가 충돌한 경우 source를 우선했습니다. 예를 들어 Thread 07의 `87b38e2f23c8`은 room이 optional NPC identity를 받을 준비를 하지만, persisted NPC를 실제로 전달하는 최초 caller는 `1122e6a4b901`의 queue fallback입니다. 같은 commit에 섞인 다른 관심사의 변경은 Thread 설명에서 제외했으며, 후대 구현을 과거 commit에 소급하지 않았습니다. repository command와 test runner는 실행하지 않았으므로, 테스트에 대해서는 source상 fixture·통과 경로·assertion·증명 범위만 기록합니다.

## Thread 경계

| 문서 | Commit 수 | 중심 문제 |
| --- | ---: | --- |
| `01-tournament-contract-schema-and-bracket-construction.md` | 10 | 참가자 목록에서 저장된 대진 경기 모델로 이동 |
| `02-tournament-room-start-rollback-and-finalization-handoff.md` | 10 | realtime room과 영속 대진의 부분 성공을 롤백하고 결과 확정을 원자화 |
| `03-profile-friendship-dashboard-and-ranking-journeys.md` | 17 | sample·고정값·추정 지표를 제거하고 읽기 모델과 cache를 사실에 맞춤 |
| `04-lobby-presence-chat-and-live-statistics.md` | 12 | durable chat와 process-local presence/statistics를 한 로비 화면에서 결합 |
| `05-chat-scope-storage-and-room-authorization.md` | 8 | scope-room 저장 불변식과 현재 경기방 audience 권한을 계층별로 강제 |
| `06-pause-resume-and-input-neutralization.md` | 5 | pause를 timer 정지가 아닌 입력 상태 경계로 완성 |
| `07-npc-ai-policy-and-fallback-journey.md` | 8 | NPC identity·queue timer·AI policy·UI 표시를 하나의 fallback 여정으로 연결 |
| **합계** | **70** | |

Thread는 위 순서대로 읽을 수 있지만 서로 다른 기능이 병렬로 개발된 부분은 선형 의존 관계로 꾸미지 않았습니다. 특히 다음 경계는 유지했습니다.

- friendship pair의 canonical persistence와 동시성은 다른 persistence Thread의 주 책임입니다.
- simulation loop·scheduler·connection lifecycle 전체는 core realtime architecture의 주 책임입니다.
- 이 폴더는 해당 하위 시스템을 제품 workflow에서 **호출하고, 표시하고, 실패 시 되돌리고, 다음 owner에게 인계하는 지점**만 다룹니다.

## 읽는 순서

1. [토너먼트 대진은 참가자 목록이 아니라 저장된 경기다](01-tournament-contract-schema-and-bracket-construction.md)
2. [토너먼트 경기방과 저장소를 하나의 전이처럼 다루기](02-tournament-room-start-rollback-and-finalization-handoff.md)
3. [읽기 화면에서 샘플과 추정값을 걷어내기](03-profile-friendship-dashboard-and-ranking-journeys.md)
4. [로비의 저장 이력과 실시간 상태를 합치기](04-lobby-presence-chat-and-live-statistics.md)
5. [채팅의 scope·room·audience를 같은 규칙으로 묶기](05-chat-scope-storage-and-room-authorization.md)
6. [일시정지는 타이머뿐 아니라 입력 의도도 멈춰야 한다](06-pause-resume-and-input-neutralization.md)
7. [NPC를 저장된 사용자로 만들고 대기열 fallback까지 연결하기](07-npc-ai-policy-and-fallback-journey.md)
