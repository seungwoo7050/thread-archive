# push_swap Development Thread 문서

이 디렉터리는 `c/push_swap` 브랜치의 개발 이력을 여섯 개의 문제 단위로 재구성한 문서 세트입니다. 각 문서는 commit 시간순 요약이 아니라 하나의 불변 조건, 책임, 실패와 수정, 또는 검증 기법을 설명합니다.

## 문서 목록

1. [`01-parallel-stack-state-and-operation-invariants.md`](01-parallel-stack-state-and-operation-invariants.md) — 값과 순위를 하나의 원소로 보존하는 stack representation과 11개 operation
2. [`02-input-grammar-coordinate-compression-and-size-safety.md`](02-input-grammar-coordinate-compression-and-size-safety.md) — argv tokenization, strict integer grammar, duplicate rejection, dense rank, 크기 계산 방어
3. [`03-building-the-sorting-engine.md`](03-building-the-sorting-engine.md) — tiny case analysis와 stable LSD radix strategy
4. [`04-independent-correctness-and-cost-evidence.md`](04-independent-correctness-and-cost-evidence.md) — 독립 replay, deterministic fixture, command/movement/allocation metric, sanitizer build
5. [`05-checker-protocol-and-verdict-hardening.md`](05-checker-protocol-and-verdict-hardening.md) — command framing, silent dispatch, `OK`/`KO`/`Error`, read boundary 강화
6. [`06-runtime-fault-injection-and-output-failure-propagation.md`](06-runtime-fault-injection-and-output-failure-propagation.md) — allocation/read/write failure seam, cleanup evidence, output status propagation
7. [`STYLE-GUIDE-AUDIT.md`](STYLE-GUIDE-AUDIT.md) — 최초 위반 사항, 교정 내역, 최종 QC 결과

## 문서 간 관계

입력 문서는 원본 정수와 dense rank를 만들고, stack operation 문서는 두 표현이 분리되지 않도록 이동 규칙을 정합니다. 정렬 엔진은 그 operation만 조합해 command stream을 생성합니다. 독립 검증 문서는 생성된 stream과 비용을 외부에서 측정하고, checker 문서는 stdin의 command protocol과 verdict를 정의합니다. runtime 문서는 allocation·read·write 실패가 helper 안에서 사라지지 않도록 전체 호출 경로를 닫습니다.

## 검토 원칙

각 커밋 설명은 표에 적힌 exact SHA의 diff와 그 시점 source/test에 근거합니다. 같은 commit에 섞인 변경 중 해당 Thread와 무관한 hunk는 제외했습니다. 현재 환경에서는 repository checkout, build, functional test, fault suite, resource suite, sanitizer target을 실행하지 않았으므로 문서는 source에 존재하는 동작과 assertion만 설명하며 실행 성공을 주장하지 않습니다.
