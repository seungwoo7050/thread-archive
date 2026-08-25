# small-shell Development Thread 재작성본

대상 repository branch: `seungwoo7050/42-archive`의 `c/minishell`

이 디렉터리는 `development-thread-workbook/completed`의 5개 Development Thread를 실제 commit diff와 해당 SHA 시점 source에 맞춰 다시 구성한 결과다.

## 문서 구성

| 번호 | 파일 | 개발 문제 |
| ---: | --- | --- |
| 01 | `01-parsed-representation-to-conditional-execution.md` | quote 의미를 보존한 token에서 command/pipeline/connector 표현과 지연 expansion 실행까지 |
| 02 | `02-heredoc-cross-stage-semantics.md` | delimiter provenance, body identity·수집·확장, temporary stream, 입력 경계 복구 |
| 03 | `03-pipeline-process-and-descriptor-ownership.md` | fork/pipe/wait와 parent stdio를 정상·부분 실패 모두에서 회수하는 PID/FD ownership |
| 04 | `04-transactional-allocation-failure.md` | fatal allocator에서 complete-result publication과 command-level failure propagation으로의 전환 |
| 05 | `05-asymptotically-safe-text-construction.md` | 반복 전체 복사를 growable builder로 바꾸고 performance·sanitizer 경로로 관찰 |

## 재작성 기준

- 기존 문서의 Thread 구분, commit SHA, 제목, importance, tags는 초기 metadata로 사용했다.
- 각 commit 설명은 해당 SHA의 실제 diff와 그 시점 source를 기준으로 작성했다.
- 같은 commit에 섞인 다른 주제의 변경은 해당 Thread에서 제외했다.
- AFTER의 heading을 복제하지 않고 정보량과 commit 성격에 따라 구조와 깊이를 조절했다.
- Feature는 도입한 상태와 아직 남은 한계를, fix는 기존 가정 → failure → root cause → decision → code → regression을, test는 대상 invariant·기법·증명 범위를 구분했다.
- Thread 사이를 시간순 단계처럼 강제로 연결하지 않고, 같은 commit이 서로 다른 문제에 기여하는 경우 각 문서에서 관점을 분리했다.

## Repository 대조로 조정한 사항

### Thread 02의 누락 commit 추가

기존 map은 `c30b39c0bcf8`의 입력 경계 복구와 `7e2fdea3affd`의 read-failure test 사이에서 실제 production input contract를 도입한 `2d3791748571`을 포함하지 않았다.

Repository history와 `commit/commit-importance.md`를 대조해 다음 metadata로 Thread 02에 추가했다.

```text
2d3791748571
fix(input): EOF와 입력 실패를 구분
Importance: A
Tags: FAILURE, SHELL_STATE, EDGE
```

이 commit이 `shell_read_line(..., int *failed)`와 runtime-wrapped `read`를 도입하고, normal EOF·recoverable read error·unrecoverable recovery error를 구분하므로 후속 test를 설명하는 데 필수다.

### Thread 03의 timeout-harness 범위 확정

기존 AFTER 문서에 남아 있던 미확정 표기를 `tests/timeout_runner.c`와 `tests/lifecycle.sh`의 exact source로 해소했다. External-signal fixture는 small-shell executor를 직접 시험하는 것이 아니라, timeout runner가 child process group을 종료·회수한다는 검증 기반이다. 이 harness가 있어야 long-running small-shell pipeline timeout test가 descendant를 남기지 않았다고 신뢰할 수 있다.

## 검증 범위

GitHub에서 exact SHA의 diff, 변경 파일, 필요한 source와 test script를 확인했다. 작업 환경에서는 branch를 로컬 checkout할 수 없어 build, behavior suite, fault injection, lifecycle stress, performance, ASan·UBSan target을 다시 실행하지 않았다. 문서에는 새 실행 결과를 주장하지 않고, source가 구현하거나 test script가 요구하는 동작만 기록했다.
