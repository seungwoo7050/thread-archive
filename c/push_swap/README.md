# push_swap Development Threads — rewritten

대상은 `42-archive` 저장소의 `c/push_swap` 브랜치입니다. 기존 `development-thread-workbook/completed`에 있던 여섯 Thread의 commit grouping, SHA, 제목, importance, tags를 유지하고, 각 commit의 실제 diff와 해당 SHA source를 기준으로 본문을 다시 작성했습니다.

## 문서 목록

| 번호 | 문서 | 중심 문제 |
| ---: | --- | --- |
| 01 | `01-parallel-stack-state-and-operation-invariants.md` | 병렬 `values`/`ranks` 표현과 11개 연산의 pairing·보존·no-op 불변 조건 |
| 02 | `02-input-grammar-coordinate-compression-and-size-safety.md` | argv token 문법, int 경계, duplicate 거절, dense rank, count·allocation 크기 안전성 |
| 03 | `03-building-the-sorting-engine.md` | 2~5개 tiny sort와 6개 이상 안정적 LSD radix sort |
| 04 | `04-independent-correctness-and-cost-evidence.md` | 독립 Python model, exhaustive/multi-seed 검증, 명령·이동·allocation budget, sanitizer path |
| 05 | `05-checker-protocol-and-verdict-hardening.md` | 최대 3-byte 명령 frame, `EINTR`, NUL·overlength 거절, `OK`/`KO`/`Error` 의미 |
| 06 | `06-runtime-fault-injection-and-output-failure-propagation.md` | N번째 allocation/read/write 실패, short write, SIGPIPE, cleanup과 status 전파 |

## Thread 관계

Thread 번호는 선형 구현 순서를 뜻하지 않습니다.

```text
01 상태·연산 의미 ─────┬─ 03 정렬기가 같은 연산으로 명령 생성
                       ├─ 05 checker가 같은 연산을 silent 재생
02 입력·dense rank ────┘

04는 03의 결과를 독립 model·checker·resource metric으로 검증
06은 01~05 전반의 allocation/read/write 실패 경로를 가로질러 보강
```

## 작성 범위

- 지정 브랜치의 최초 Thread commit `96b5324448e4`와 대상 이력에서 가장 뒤의 `5505adf3e469`가 모두 `c/push_swap`의 조상임을 확인했습니다.
- 각 설명은 exact SHA diff와 그 시점 source를 사용했습니다. final HEAD의 후속 반환형·wrapper·테스트를 이전 commit 설명에 소급하지 않았습니다.
- 같은 commit에 여러 변경이 포함된 경우 해당 Thread와 직접 연결되는 diff만 설명했습니다.
- 기존 workbook의 반복적인 완료란·자가 점검·고정 heading은 제거하고, 문제 크기에 따라 코드·원인·결정·보장 범위를 다르게 배치했습니다.
- 현재 작업 환경에서는 repository를 checkout해 build/test를 실행하지 않았습니다. 따라서 문서는 test source가 무엇을 assertion하는지 설명하며, 이번 세션에서의 통과 결과를 주장하지 않습니다.
