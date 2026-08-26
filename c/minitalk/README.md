# minitalk Development Threads

`c/minitalk` branch의 `development-thread-workbook/completed/`에 있던 6개 Development Thread를 exact commit diff와 해당 SHA의 source를 기준으로 재작성한 결과입니다.

기존 Thread별 commit 순서, SHA, 제목, importance, tags는 유지했습니다. 본문은 `thread-doc-style-guide.md`에 맞춰 학습 체크리스트와 고정 워크북 구조를 제거하고, 커밋 성격에 따라 feat·refactor·fix·test의 설명 구조를 다르게 적용했습니다.

## 문서

1. [다음 bit는 현재 sequence의 검증된 ACK 뒤에만 전송된다](./01-timing-to-correlated-sequence-acks.md)
2. [세션 소유권은 응답 가능한 하나의 sender에게만 유지된다](./02-session-ownership-and-recovery.md)
3. [signal handler는 사실만 기록하고 event loop가 protocol state를 소유한다](./03-self-pipe-event-loop.md)
4. [stdout 반영 성공이 ACK 전송의 commit 조건이다](./04-output-commit-boundary.md)
5. [endpoint 경로는 bind한 process만 정리하고 polling 표현 범위를 넘지 않는다](./05-endpoint-ownership-and-bounded-polling.md)
6. [응답은 현재 operation과 일치할 때만 원래 deadline 안에서 수락된다](./06-bounded-response-correlation.md)

## Thread 관계

이 문서들은 시간순 chapter가 아니라 서로 다른 개발 책임을 설명합니다. 일부 commit은 둘 이상의 Thread에서 서로 다른 문제의 증거로 다시 등장합니다.

```text
signal bit transport ── sequence ACK ───────────────┐
                                                    │
ACQUIRE/READY ── session ownership ────────────────┼─ bounded response correlation
                                                    │
self-pipe ── normal-context state ownership ───────┤
                                                    │
stdout output ── ACK commit boundary ──────────────┤
                                                    │
Unix endpoint path / descriptor lifetime ──────────┘
```

- Thread 1과 6은 `ebed06775b92`, `d3eacbbfeadc`를 공유하지만, 전자는 timing과 one-bit-in-flight의 전환을, 후자는 acceptance predicate와 bounded deadline을 설명합니다.
- Thread 2와 6은 `f8e8444c5ded`를 공유하지만, 전자는 reservation의 시작점을, 후자는 READY correlation을 설명합니다.
- Thread 2의 stale-owner recovery와 Thread 4의 recovery newline은 연결되지만, ownership과 output commit이라는 별개의 책임으로 유지했습니다.
- Thread 1의 response pipe는 ACK send work만 normal context로 넘기는 과도기 구조이고, Thread 3의 self-pipe는 signal event와 authoritative state mutation의 실행 위치를 분리합니다.

## 검수 결과

- 6개 문서, 40개 Thread 배치, 37개 고유 SHA를 확인했습니다.
- commit table의 SHA, 제목, importance, tags가 기존 metadata와 일치합니다.
- 각 commit section에 성격별 설명 구조와 관계가 명시된 `관련 커밋` 절이 있습니다.
- 기본 code evidence는 diff이고, plain C는 line-level 주석이 실제로 필요한 `mt_write_all` 한 곳에만 사용했습니다.
- 중복 ledger·자가 점검·완료 기준·학습자 기록 라벨을 제거했습니다.
- 각 문서는 `이 Thread의 경계`로 닫고, 마지막 줄에 exact SHA 검토 범위와 미실행 항목을 기록합니다.
- branch checkout, build와 test suite 실행은 이 환경에서 수행하지 않았으므로 runtime 통과를 주장하지 않습니다.
