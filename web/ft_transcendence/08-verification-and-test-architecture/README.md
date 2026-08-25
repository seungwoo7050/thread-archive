# 검증 및 테스트 아키텍처 — Development Thread 재작성본

이 디렉터리는 `web/ft_transcendence` 브랜치의 다음 범위만 조사해 재작성했습니다.

```text
development-thread-workbook/
└── 08-verification-and-test-architecture/
    └── completed/
```

기존 문서의 Thread 구분과 commit metadata는 유지하되, 반복적인 학습 템플릿·빈칸·자가 점검을 제거하고 실제 개발 문제를 중심으로 다시 구성했습니다. 각 문서는 **정확한 SHA의 diff와 그 시점의 source/test code**를 기준으로 하며, 같은 commit에 섞인 다른 관심사는 해당 Thread에서 제외합니다.

## 문서 구성

| 번호 | 문서 | 중심 문제 |
| ---: | --- | --- |
| 01 | [실행 가능한 프로토콜과 HTTP 계약 검증](01-executable-protocol-and-http-contract-verification.md) | 공유 schema가 실제 API route의 신뢰 경계가 되고, 오래된 protocol shape가 거부되는 과정 |
| 02 | [인증 ticket·실패 격리·transport 한계](02-authentication-ticket-failure-containment-and-transport-limits.md) | cookie session에서 일회용 WebSocket ticket으로 넘어가고, 인증 전후의 자원·오류·payload 경계를 제한하는 과정 |
| 03 | [결정적 simulation·시간·snapshot 전달 검증](03-deterministic-simulation-timing-and-snapshot-delivery-verification.md) | authoritative state의 결정성과 전달 압력을 분리하고, callback 지연을 congestion으로 오판한 가정을 교정하는 과정 |
| 04 | [GameHub lifecycle·재연결·매칭·결과 저장 복구](04-gamehub-lifecycle-reconnect-matchmaking-and-finalization-recovery.md) | room·connection·reservation·timer·finalization의 소유권을 실패 뒤에도 끝까지 회수하는 과정 |
| 05 | [PostgreSQL 통합·동시성·migration·실패 주입](05-postgresql-integration-concurrency-migration-and-failure-injection.md) | memory 구현이 증명할 수 없는 transaction·constraint·lock·migration 의미를 실제 PostgreSQL로 검증하는 과정 |
| 06 | [실행 process smoke와 browser E2E](06-process-smoke-and-browser-end-to-end-verification.md) | 초기 bearer/query smoke가 cookie/ticket/v1 계약으로 교정되고 browser 복구 흐름까지 확장되는 과정 |
| 07 | [benchmark·load·fault recovery 검증](07-benchmark-load-and-fault-recovery-verification.md) | microbenchmark, executable SLO, k6/Toxiproxy, production cadence 수정, 복구 report를 서로 다른 증거 층으로 결합하는 과정 |

## 보존한 metadata

- Thread: **7개**
- 고유 commit: **46개**
- 중요도: **S 2개 / A 28개 / B 16개**
- 각 commit의 SHA, subject, importance, tags, Thread 내 역할

importance와 tags는 다른 Development Thread 문서와 연동되는 분류 체계이므로 변경하지 않았습니다.

## 이 재작성본의 읽는 법

문서의 commit map은 history 탐색용 metadata입니다. 본문은 commit 하나마다 같은 heading을 반복하지 않고, 다음과 같은 실제 관계가 있을 때만 묶습니다.

```text
계약 또는 정상 경로 도입
        ↓
숨겨진 실패 가정이나 관찰 공백 발견
        ↓
root-cause 수준의 구현 수정
        ↓
그 실패를 직접 재현하는 regression / integration / load evidence
```

시간상 인접하더라도 같은 개발 문제를 다루지 않으면 연결하지 않습니다. 반대로 feature, fix, test가 여러 달에 걸쳐 흩어져 있어도 같은 불변 조건을 완성한다면 하나의 Thread 안에서 연결합니다.

## 증거 범위

이번 작업에서 확인한 것은 다음 두 종류입니다.

1. 각 commit의 exact diff
2. 해당 SHA에 존재하는 production source, fixture, test harness와 assertion

다음 명령들은 이 작성 환경에서 실행하지 않았습니다.

- `pnpm` unit/integration suite
- Testcontainers PostgreSQL suite
- Playwright browser suite
- k6 load run
- Toxiproxy fault scenario

따라서 문서의 “검증한다”는 표현은 **그 검증 코드와 assertion이 해당 SHA에 존재한다는 뜻**입니다. 실제 실행 통과나 측정 수치를 새로 주장하지 않습니다. benchmark/load/fault 문서에서는 “실행 가능한 계약”과 “실제 실행 결과”를 명시적으로 구분했습니다.
