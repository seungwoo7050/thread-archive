# 시간 지연 기반 신호 전송에서 순번이 일치하는 ACK까지

> 완성형 해설서가 아닙니다. 아래 확정 사항을 기준으로 각 commit SHA의 실제 코드와 diff를 읽고 기록란을 채웁니다.

## 1. 개발 흐름 목표

고정 지연에 기대던 초기 bit 전송이 signal ACK 기반 stop-and-wait를 거쳐, 출처와 sequence를 검증하는 Unix datagram ACK 방식으로 바뀌는 과정을 복원합니다. 마지막 고정 지연 제거가 단순한 속도 조정이 아니라 더 강한 protocol 보장의 결과임을 해당 SHA의 코드로 설명할 수 있어야 합니다.

### Significance

초기 pacing은 standard signal 중복 병합 가능성을 낮출 뿐 처리 완료를 증명하지 못합니다. signal ACK는 인과적인 흐름 제어를 추가하지만 응답 식별력이 약하고 같은 signal 체계에 응답 책임까지 얹습니다. datagram control channel이 identity와 sequence를 제공한 뒤에는 이전 signal ACK 경로와 timing delay를 제거할 수 있습니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- 초기 client는 byte와 bit를 어떤 순서로 signal에 대응시키며, fixed delay는 무엇을 보장하지 못하는가?
- signal ACK wait 전에 response signal을 block해야 했던 race는 어느 순서에서 발생하는가?
- timeout은 delivery 보장과 어떻게 다르며, 어떤 failure에서 bit cursor가 전진하지 않아야 하는가?
- request/response record의 어떤 field가 session과 개별 bit transition을 식별하는가?
- datagram ACK 도입 뒤에도 남아 있던 구 signal ACK path는 어디였으며 제거 후 success source가 하나로 수렴했는가?
- fixed delay를 제거해도 one-bit-in-flight 불변 조건이 어떤 코드 순서로 유지됩니까?

## 3. 완료 기준

- [x] fixed delay, signal ACK, timeout, sequence datagram ACK의 역할 차이를 코드로 설명합니다.
- [x] 각 단계에서 client가 다음 bit로 넘어가는 조건과 failure branch를 기록합니다.
- [x] wire record, sequence, source endpoint 검증이 결합되는 조건식을 찾았습니다.
- [x] legacy ACK 제거 전후 정상 처리를 비교해 단일 authoritative response path를 확인했습니다.
- [x] 최종 상태에서 sleep 없이 ordering이 유지되는 execution flow를 함수 단위로 복원했습니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `89637d63b56f` | feat(client): 메시지 바이트를 시그널로 전송 | A | CORE, SIGNAL_DATA | message byte를 MSB부터 signal bit로 직렬화하고 provisional fixed delay로 전송 속도를 낮춥니다. |
| 2 | `78de95b3cacb` | feat(protocol): 비트 처리마다 ACK 전송 | A | CORE, SIGNAL_DATA, RISK | bit마다 signal ACK를 기다리는 stop-and-wait를 도입하고 ACK-before-wait race를 막습니다. |
| 3 | `765efe7b75c9` | feat(client): ACK 대기 시간 초과 처리 | A | RISK, PROCESS_LIFECYCLE, PRACTICAL | ACK wait를 alarm 기반 timeout으로 제한하고 timeout과 다른 send failure를 구분합니다. |
| 4 | `342aea9ce9a8` | fix(client): ACK 이후 시그널 전송 간격 안정화 | B | SIGNAL_DATA, PRACTICAL | signal ACK 뒤 short inter-signal gap을 유지하고 acknowledgement deadline을 늘립니다. |
| 5 | `4f17de94e025` | fix(client): 인터럽트 뒤 남은 전송 간격 유지 | B | SIGNAL_DATA, PRACTICAL | `nanosleep`이 `EINTR`로 중단되면 returned remainder로 같은 logical gap을 이어갑니다. |
| 6 | `ebed06775b92` | feat(protocol): 응답 메시지 wire 형식 정의 | A | ARCH, RESPONSE | `ACQUIRE`, `READY`, `ACK`를 표현하는 request/response records와 identity fields를 정의합니다. |
| 7 | `4234233ebd30` | feat(protocol): 비트 ACK를 sequence 응답으로 큐잉 | S | ARCH, RESPONSE, CORE | accepted bit에 sequence를 부여하고 datagram ACK send work를 pipe로 queue해 direct signal response에서 분리합니다. |
| 8 | `d3eacbbfeadc` | feat(client): 비트 ACK를 sequence로 상관 검증 | A | RESPONSE, RISK | client가 expected server source와 exact current sequence의 datagram ACK만 bit success로 수락합니다. |
| 9 | `aeb1b00867f4` | refactor(protocol): 이전 signal ACK 경로 제거 | A | ARCH, RESPONSE, REFACTOR | obsolete ACK/NACK signal machinery를 client/server/shared/test sender에서 제거하고 datagram response만 남깁니다. |
| 10 | `1487a861046e` | perf(protocol): 검증된 ACK 뒤 고정 지연 제거 | A | PERF, RESPONSE | matching sequence ACK 뒤 fixed sleep을 production client와 session sender에서 제거합니다. |

확인 원칙:

- 각 항목은 해당 SHA의 tree를 기준으로 읽었습니다.
- 변경 전 상태는 해당 SHA의 parent 또는 지정된 이전 관련 SHA에서 확인했습니다.
- 같은 commit이 다른 Thread에 다시 등장해도 이 개발 흐름의 질문으로 별도 기록했습니다.
- runtime test는 실행하지 않았으며, 실행 결과처럼 표현하지 않았습니다.

## 5. 커밋별 학습 기록

### 1. `89637d63b56f` — feat(client): 메시지 바이트를 시그널로 전송

- **중요도:** A
- **태그:** CORE, SIGNAL_DATA
- **개발 흐름에서의 역할:** message byte를 MSB부터 signal bit로 직렬화하고 provisional fixed delay로 전송 속도를 낮춥니다.

#### 원문에서 확정된 맥락

client는 target PID를 검증하고 message를 encoding 해석 없이 byte sequence로 취급합니다. zero와 one은 서로 다른 user signal이며 delay는 coalescing 위험을 줄일 뿐 receiver 처리 완료를 증명하지 못합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `mt_parse_pid`와 `kill(pid, 0)`의 target 검증
- [x] `send_byte`의 7→0 bit 순회
- [x] 0/1을 두 user signal로 매핑하는 `send_bit` branch
- [x] `kill` 성공 뒤 150µs 고정 지연
- [x] send failure 시 caller로 즉시 반환하는 경로

#### 비교 기준

`8e5371c7b85e`의 server assembly가 byte를 왼쪽으로 밀어 MSB-first signal을 복원하는 규칙과 맞췄습니다. parent diff에서는 client transport와 PID parser가 새로 연결됩니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: parent에서는 server 쪽 MSB-first assembly는 존재하지만 client가 payload를 signal sequence로 만드는 transport path가 완성되지 않았습니다. `main`은 아직 `send_byte`/`send_bit`를 호출하지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: standard signal은 같은 종류의 여러 pending instance를 counted queue처럼 보존하지 않으므로, 송신자가 receiver 처리 여부와 무관하게 전진하면 bit가 합쳐질 수 있습니다.
- 변경된 decision과 state mutation 순서: `src/client.c`에 byte loop와 bit loop를 만들고 각 bit를 `MT_ZERO_SIGNAL` 또는 `MT_ONE_SIGNAL`로 변환했습니다. `main` → `mt_parse_pid` → `kill(pid, 0)` → payload byte 선택 → bit 7부터 0까지 `send_bit` → `kill` 성공 뒤 `usleep(150)` → 다음 bit입니다. 이 SHA에서는 payload 뒤 NUL terminator를 전송하지 않습니다.
- 정상 경로와 failure 경로가 갈라지는 조건: PID parse/process probe/`kill`/delay 중 하나가 실패하면 오류를 반환하고 현재 함수가 끝납니다. 실패한 `send_bit` 뒤에는 bit index나 다음 byte로 전진하지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `78de95b3cacb`가 elapsed time 대신 server ACK를 다음 bit의 조건으로 바꾸고, `765efe7b75c9`가 그 wait를 제한합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] sender-side bit 표현과 MSB-first 순서
- [x] message를 raw byte sequence로 취급하는 기준

**아직 보장하지 않는 것**

- [x] receiver 처리 완료
- [x] signal multiplicity 보존
- [x] NUL framing
- [x] session 소유권
- [x] acknowledgement

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/parse_pid.c`, `include/minitalk.h`, `Makefile`
- symbol 또는 함수: `main`, `send_byte`, `send_bit`, `mt_parse_pid`
- 확인한 state fields: `bit`, `byte pointer`
- caller → callee: `main` → `send_byte` → `send_bit` → `kill`/`usleep`
- 핵심 branch 또는 mutation 순서: `main` → `mt_parse_pid` → `kill(pid, 0)` → payload byte 선택 → bit 7부터 0까지 `send_bit` → `kill` 성공 뒤 `usleep(150)` → 다음 bit입니다. 이 SHA에서는 payload 뒤 NUL terminator를 전송하지 않습니다.
- parent 또는 이전 관련 SHA와의 diff 요약: client transport 책임과 PID parser가 추가됐고, 아직 ACK·timeout·session state는 없습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`78de95b3cacb`에서 elapsed time 대신 ACK가 next-bit condition이 됩니다.
### 2. `78de95b3cacb` — feat(protocol): 비트 처리마다 ACK 전송

- **중요도:** A
- **태그:** CORE, SIGNAL_DATA, RISK
- **개발 흐름에서의 역할:** bit마다 signal ACK를 기다리는 stop-and-wait를 도입하고 ACK-before-wait race를 막습니다.

#### 원문에서 확정된 맥락

client는 ACK signal을 먼저 block하고 data signal을 보낸 뒤 `sigsuspend`로 기다립니다. server가 한 bit 처리 뒤 ACK를 보내며 client는 ACK 전에는 다음 bit로 진행하지 않습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] ACK handler가 `volatile sig_atomic_t` flag를 변경
- [x] `sigprocmask(SIG_BLOCK)`가 data `kill`보다 먼저 실행
- [x] `sigsuspend` mask에서 ACK만 임시 unblock
- [x] server bit mutation 뒤 sender PID로 ACK 전송
- [x] ACK success 뒤에만 bit cursor 감소

#### 비교 기준

`89637d63b56f`과 비교하면 다음 bit 전진 조건이 fixed delay 완료에서 ACK flag 관측으로 바뀌고 NUL terminator 전송도 추가됩니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `89637d63b56f`의 `send_bit`은 `kill` 뒤 짧은 delay만 기다렸고 server 처리 완료를 관측하는 state가 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: delay는 완료 사실이 아니며, ACK를 block하지 않고 먼저 보내면 server ACK가 `sigsuspend` 진입 전에 도착해 영원히 기다리는 lost-wakeup race도 생깁니다.
- 변경된 decision과 state mutation 순서: client에 `volatile sig_atomic_t` ACK flag와 handler를 두고 ACK signal을 block한 상태에서 data signal을 전송하도록 순서를 고정했습니다. server는 한 bit의 state mutation 뒤 sender PID로 ACK를 보냅니다. ACK block → `g_acknowledged = 0` → data `kill` → ACK만 unblocked한 mask로 `sigsuspend` 반복 → handler flag 관측 → `send_bit` success → `send_byte`가 bit index 감소 순서입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: `sigprocmask` 또는 `kill`이 실패하면 전송을 중단합니다. ACK가 없으면 무기한 wait하며, ACK signal에는 bit/session identity가 없습니다.
- 후속 commit이 강화하거나 교체하는 부분: `765efe7b75c9`가 timeout state를 추가하고, `ebed06775b92` 이후 datagram token이 generic signal ACK를 대체합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | short delay면 server가 이전 signal을 처리했을 것이다. | `89637d63b56f: send_bit`의 `kill` 뒤 `usleep(150)` |
| 실제 failure 또는 위험 | standard signal coalescing과 ACK-before-wait lost wakeup이 남습니다. | 직전에는 receiver state를 확인하는 flag/mask가 없었습니다. |
| root cause | 완료 조건이 elapsed time이고 ACK 도착 전에 block하지 않았습니다. | 새 diff가 ACK block을 `kill` 앞에 둔 이유입니다. |
| 수정 불변 조건/decision | ACK를 먼저 block하고 한 bit를 보낸 뒤 handler flag를 확인해야 다음 bit로 진행합니다. | `src/client.c: send_bit`, server bit handler의 ACK `kill` |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: ACK block → `g_acknowledged = 0` → data `kill` → ACK만 unblocked한 mask로 `sigsuspend` 반복 → handler flag 관측 → `send_bit` success → `send_byte`가 bit index 감소 순서입니다.
- 수정된 불변 조건을 고정하는 후속 회귀 테스트: 이 단계 전용 test commit은 Thread에 없습니다. 후속 datagram 검증은 Thread 6에서 검증됩니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] one-bit stop-and-wait
- [x] block-before-send ordering
- [x] payload 뒤 NUL frame 전송

**아직 보장하지 않는 것**

- [x] strong response correlation
- [x] bounded wait
- [x] final datagram architecture

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/server.c`, `include/minitalk.h`
- symbol 또는 함수: `handle_ack`, `send_bit`, `send_byte`, `handle_bit`
- 확인한 state fields: `g_acknowledged`, `current_byte`, `received_bits`
- caller → callee: client `send_byte` → `send_bit`; server 시그널 처리 함수 → bit mutation → `kill(sender, MT_ACK_SIGNAL)`
- 핵심 branch 또는 mutation 순서: ACK block → `g_acknowledged = 0` → data `kill` → ACK만 unblocked한 mask로 `sigsuspend` 반복 → handler flag 관측 → `send_bit` success → `send_byte`가 bit index 감소 순서입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: fixed delay-only path가 signal ACK stop-and-wait로 교체되고 terminator byte가 전송됩니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`765efe7b75c9`에서 wait에 finite deadline이 추가됩니다.
### 3. `765efe7b75c9` — feat(client): ACK 대기 시간 초과 처리

- **중요도:** A
- **태그:** RISK, PROCESS_LIFECYCLE, PRACTICAL
- **개발 흐름에서의 역할:** ACK wait를 alarm 기반 timeout으로 제한하고 timeout과 다른 send failure를 구분합니다.

#### 원문에서 확정된 맥락

process existence는 protocol participation이 아닙니다. ACK가 없으면 current transmission을 실패로 끝내고 next bit로 진행하지 않습니다. timeout은 retransmission이나 delivery guarantee가 아닙니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] ACK와 timeout handler state 구분
- [x] alarm 시작 → wait → alarm 취소 순서
- [x] `kill` failure와 timeout status 분리
- [x] timeout에서 bit cursor 유지
- [x] `MT_ACK_TIMEOUT_SECONDS` 1초 정의

#### 비교 기준

`78de95b3cacb`의 단일 ACK flag loop에 `g_timed_out`, `SIGALRM`, status 구분, alarm 취소가 추가됩니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `78de95b3cacb`의 client는 target process가 ACK를 보내지 않아도 `sigsuspend` loop를 끝낼 수 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: PID가 존재하거나 `kill`이 성공해도 상대가 minitalk server로 동작한다는 뜻은 아니므로 unbounded suspension이 가능합니다.
- 변경된 decision과 state mutation 순서: ACK와 별도의 timeout flag를 추가하고 `SIGALRM`을 같은 wait loop의 종료 조건으로 사용했습니다. send error와 timeout을 다른 status로 반환합니다. flags reset → data `kill` → `alarm(MT_ACK_TIMEOUT_SECONDS)` → ACK/timeout 중 하나가 될 때까지 `sigsuspend` → `alarm(0)` → ACK면 success, timeout이면 `MT_SEND_TIMEOUT`입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: `kill` failure는 send error, timeout flag는 timeout diagnostic으로 갈라집니다. timeout branch는 `send_byte`에 실패를 반환하므로 현재 bit index가 유지됩니다. `sigsuspend`의 wakeup error는 이 SHA에서 별도 영구 오류로 분류하지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `342aea9ce9a8`이 timeout을 3초로 늘리고 ACK 뒤 pacing을 둡니다. 이후 datagram wait는 monotonic absolute deadline으로 교체됩니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] unresponsive target에 대한 bounded liveness failure
- [x] timeout 뒤 transmission 중단

**아직 보장하지 않는 것**

- [x] lost-bit recovery
- [x] retransmission
- [x] ACK loss deduplication
- [x] monotonic clock deadline

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `include/minitalk.h`
- symbol 또는 함수: `handle_response`, `send_bit`, `send_byte`
- 확인한 state fields: `g_acknowledged`, `g_timed_out`
- caller → callee: `send_byte` → `send_bit` → `kill`/`alarm`/`sigsuspend`
- 핵심 branch 또는 mutation 순서: flags reset → data `kill` → `alarm(MT_ACK_TIMEOUT_SECONDS)` → ACK/timeout 중 하나가 될 때까지 `sigsuspend` → `alarm(0)` → ACK면 success, timeout이면 `MT_SEND_TIMEOUT`입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: unbounded ACK wait가 alarm으로 제한되고 caller가 send error와 timeout을 다른 diagnostic으로 출력할 수 있게 됩니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`342aea9ce9a8`에서 timeout과 post-ACK pacing이 조정됩니다.
### 4. `342aea9ce9a8` — fix(client): ACK 이후 시그널 전송 간격 안정화

- **중요도:** B
- **태그:** SIGNAL_DATA, PRACTICAL
- **개발 흐름에서의 역할:** signal ACK 뒤 short inter-signal gap을 유지하고 acknowledgement deadline을 늘립니다.

#### 원문에서 확정된 맥락

ACK가 serialization 조건이고 delay는 scheduling sensitivity를 낮추는 workaround입니다. timeout과 pacing constant는 서로 다른 역할입니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] ACK success branch 뒤 gap 호출
- [x] timeout 3초와 pacing 500µs의 별도 정의
- [x] delay failure의 send error 전달
- [x] ACK 전이 아니라 뒤에만 pacing 적용

#### 비교 기준

`765efe7b75c9`에서 timeout 값과 ACK-success tail만 바뀌며 protocol identity는 그대로 generic signal ACK입니다.

#### B-level 구현 역할 기록

- Thread 전체에서 이 commit이 연결하는 앞/뒤 단계: alarm 기반 one-bit stop-and-wait는 있었지만 ACK 직후 다음 signal을 즉시 보내는 초기 signal-only 구현이 host scheduling에 민감했습니다. → `4f17de94e025`가 interrupted sleep의 remainder를 보존하고 `1487a861046e`가 datagram ACK 이후 이 workaround 자체를 삭제합니다.
- 실제로 추가·수정된 핵심 symbol과 state: timeout을 1초에서 3초로 늘리고 `MT_SIGNAL_GAP_US` 500µs를 ACK success branch 뒤에 추가했습니다.
- 이 commit만으로 충분하지 않아 후속 commit을 확인해야 하는 부분: signal ACK는 causal condition이지만 당시 handler cycle과 scheduling variation에 대한 구현상 여유가 없었습니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | ACK만 받으면 다음 signal을 즉시 보내도 scheduling variation과 무관합니다. | 직전 success branch는 ACK flag 확인 즉시 반환했습니다. |
| 실제 failure 또는 위험 | early signal-only response path가 handler-cycle timing에 민감할 수 있습니다. | 당시 ACK와 data가 모두 standard 시그널 처리 함수 path를 공유했습니다. |
| root cause | causal ACK와 구현상 pacing의 역할을 분리하지 않았습니다. | 새 상수는 timeout과 별도로 정의되고 ACK 뒤에만 사용됩니다. |
| 수정 불변 조건/decision | ACK를 ordering 조건으로 유지하되 정상 처리에 500µs gap을 둡니다. | `src/client.c: send_bit` success tail |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: matching signal ACK 확인 → `usleep(MT_SIGNAL_GAP_US)` → success 반환 → 다음 bit입니다. timeout timer와 pacing delay는 별도 상수입니다.
- 수정된 invariant를 고정하는 후속 regression test: 전용 test는 없으며 `4f17de94e025`가 EINTR progress를 수정하고 `1487a861046e`가 최종 제거합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] 초기 signal-only protocol의 scheduling 안정화

**아직 보장하지 않는 것**

- [x] sequence identity
- [x] delay 자체의 correctness guarantee

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `include/minitalk.h`
- symbol 또는 함수: `send_bit`
- 확인한 state fields: `g_acknowledged`, `g_timed_out`
- caller → callee: `send_bit` ACK wait → `usleep` → caller success
- 핵심 branch 또는 mutation 순서: matching signal ACK 확인 → `usleep(MT_SIGNAL_GAP_US)` → success 반환 → 다음 bit입니다. timeout timer와 pacing delay는 별도 상수입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: deadline을 늘리고 성공 path에만 explicit inter-signal delay를 추가했습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`4f17de94e025`에서 interrupted sleep의 remainder를 보존합니다.
### 5. `4f17de94e025` — fix(client): 인터럽트 뒤 남은 전송 간격 유지

- **중요도:** B
- **태그:** SIGNAL_DATA, PRACTICAL
- **개발 흐름에서의 역할:** `nanosleep`이 `EINTR`로 중단되면 returned remainder로 같은 logical gap을 이어갑니다.

#### 원문에서 확정된 맥락

partial sleep을 completed interval로 취급하지 않으며 original full duration을 반복 요청하지도 않습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `nanosleep(&remaining, &remaining)` retry loop
- [x] `errno == EINTR`만 반복
- [x] kernel remainder가 다음 request가 되는 alias
- [x] non-EINTR error를 별도 반환하지 않는 실제 한계

#### 비교 기준

`342aea9ce9a8`의 single `usleep`을 remainder-aware `nanosleep` loop로 교체한 부분만 추적했습니다.

#### B-level 구현 역할 기록

- Thread 전체에서 이 commit이 연결하는 앞/뒤 단계: `342aea9ce9a8`은 ACK 뒤 `usleep` 한 번으로 gap을 요청했습니다. → `1487a861046e`에서 correlated datagram ACK가 ordering 근거가 된 뒤 helper와 gap 상수가 제거됩니다.
- 실제로 추가·수정된 핵심 symbol과 state: gap을 5ms `timespec`으로 바꾸고 `nanosleep(&remaining, &remaining)`을 `errno == EINTR`인 동안 반복하는 helper를 추가했습니다.
- 이 commit만으로 충분하지 않아 후속 commit을 확인해야 하는 부분: unrelated signal이 sleep을 중단하면 실제 gap이 짧아지고, 처음 duration을 반복하면 이미 경과한 시간까지 중복 대기합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | 한 번의 sleep 호출이 requested interval 전체를 보장합니다. | 직전 ACK-success branch의 단일 `usleep` |
| 실제 failure 또는 위험 | `EINTR`가 gap을 조기에 끝내 timing workaround를 약화합니다. | signal을 사용하는 process이므로 unrelated delivery가 sleep을 중단할 수 있습니다. |
| root cause | interrupted sleep을 remaining duration을 가진 partial operation으로 다루지 않았습니다. | `nanosleep`의 두 번째 인자를 같은 `remaining` object로 사용합니다. |
| 수정 불변 조건/decision | `EINTR`이면 remainder로 같은 logical interval을 이어갑니다. | `src/client.c: wait_signal_gap` loop |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: ACK success → `wait_signal_gap` → remaining timespec 초기화 → interrupted call마다 kernel이 돌려준 remainder로 재호출 → success 또는 non-EINTR 종료입니다.
- 수정된 invariant를 고정하는 후속 regression test: 전용 regression test는 없고 최종적으로 `1487a861046e`가 이 timing path를 삭제합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] EINTR 아래 provisional pacing interval의 남은 시간 처리
- [x] partial sleep progress 보존

**아직 보장하지 않는 것**

- [x] timing-independent ordering
- [x] final response correlation
- [x] non-EINTR sleep error propagation

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `include/minitalk.h`
- symbol 또는 함수: `wait_signal_gap`, `send_bit`
- 확인한 state fields: `remaining timespec`
- caller → callee: `send_bit` → `wait_signal_gap` → `nanosleep`
- 핵심 branch 또는 mutation 순서: ACK success → `wait_signal_gap` → remaining timespec 초기화 → interrupted call마다 kernel이 돌려준 remainder로 재호출 → success 또는 non-EINTR 종료입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: single `usleep`이 remainder-aware `nanosleep` loop로 바뀌고 gap이 5ms로 조정됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`1487a861046e`에서 이 delay 자체가 제거됩니다.
### 6. `ebed06775b92` — feat(protocol): 응답 메시지 wire 형식 정의

- **중요도:** A
- **태그:** ARCH, RESPONSE
- **개발 흐름에서의 역할:** `ACQUIRE`, `READY`, `ACK`를 표현하는 request/response records와 identity fields를 정의합니다.

#### 원문에서 확정된 맥락

magic, kind, PID, nonce/token, status가 signal data channel과 별도의 control-plane identity를 제공합니다. records는 host-local in-memory ABI이며 portable serialized network protocol이 아닙니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `t_mt_request`/`t_mt_response` field order와 fixed-width types
- [x] magic와 ACQUIRE/READY/ACK kind constants
- [x] client/server PID, nonce/token, status 배치
- [x] `sizeof(record)`를 wire length로 사용할 전제
- [x] byte-order/alignment serialization 부재

#### 비교 기준

`4f17de94e025`까지의 signal ACK는 identity payload가 없었고, 새 records는 acquisition과 bit transition을 구분할 field를 제공합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: generic ACK signal은 sender/transition별 payload를 담지 못해 어느 acquisition 또는 어느 bit에 대한 응답인지 표현할 수 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: stale ACK, 다른 server의 응답, session READY와 bit ACK를 같은 signal만으로 구별할 수 없습니다.
- 변경된 decision과 state mutation 순서: shared header에 fixed-width magic/kind/token/status와 `pid_t` identity를 가진 `t_mt_request`, `t_mt_response`를 정의했습니다. schema만 정의한 commit입니다. 이후 caller가 `sizeof(struct)`를 datagram length로 사용하며 request nonce는 READY token으로, server sequence는 ACK token으로 연결됩니다.
- 정상 경로와 failure 경로가 갈라지는 조건: 이 SHA 자체에는 send/receive acceptance branch가 없습니다. padding, byte order, ABI version을 직렬화하지 않아 같은 host/ABI 범위에 제한됩니다.
- 후속 commit이 강화하거나 교체하는 부분: `4234233ebd30`이 server sequence ACK work를 만들고 `f8e8444c5ded`/`d3eacbbfeadc`가 client acceptance predicate를 구현합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] correlated control message를 표현할 schema
- [x] session response와 bit response 구분 능력

**아직 보장하지 않는 것**

- [x] runtime ACQUIRE validation
- [x] client acceptance predicate
- [x] portable ABI/byte order/versioning

#### 코드 증거 기록

- 파일 경로: `include/minitalk.h`
- symbol 또는 함수: `t_mt_request`, `t_mt_response`
- 확인한 state fields: `magic`, `kind`, `nonce`, `token`, `status`, `client_pid`, `server_pid`
- caller → callee: shared record definition → 후속 client/server send/receive caller
- 핵심 branch 또는 mutation 순서: schema만 정의한 commit입니다. 이후 caller가 `sizeof(struct)`를 datagram length로 사용하며 request nonce는 READY token으로, server sequence는 ACK token으로 연결됩니다.
- parent 또는 이전 관련 SHA와의 diff 요약: signal-only constants 옆에 host-local datagram record schema와 kind/status constants가 추가됐습니다.
- 삽입한 최소 코드 조각과 선택 이유: `ebed06775b92`, `include/minitalk.h`: acquisition identity를 만드는 field와 raw ABI 범위를 보여 주는 최소 조각입니다.

```c
typedef struct s_mt_request
{
	uint32_t magic;
	uint32_t kind;
	uint32_t nonce;
	pid_t   client_pid;
} t_mt_request;
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`4234233ebd30`에서 sequence가 actual ACK work에 연결됩니다.
### 7. `4234233ebd30` — feat(protocol): 비트 ACK를 sequence 응답으로 큐잉

- **중요도:** S
- **태그:** ARCH, RESPONSE, CORE
- **개발 흐름에서의 역할:** accepted bit에 sequence를 부여하고 datagram ACK send work를 pipe로 queue해 direct signal response에서 분리합니다.

#### 원문에서 확정된 맥락

이 SHA에서는 handler가 bit transition을 여전히 수행하지만 `sendto`는 normal 문맥으로 미뤄집니다. queue write failure는 overflow flag로 드러냅니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] server sequence field의 초기화/reset/increment
- [x] ACK work record의 client PID/kind/token/status
- [x] bit mutation 뒤 response work 생성 순서
- [x] handler의 full-size pipe write와 overflow flag
- [x] pipe consumer의 `sendto` 호출
- [x] legacy signal ACK와 datagram ACK의 공존
- [x] NUL reset에서 sequence가 0으로 돌아가는 경로

#### 비교 기준

`ebed06775b92`의 static schema가 `g_sequence`와 response-work pipe에 연결됐습니다. final self-pipe와 달리 이 SHA의 pipe payload는 signal fact가 아니라 이미 처리된 bit의 ACK work입니다.

#### S-level 재구성

- 이 commit 직전의 관련 state와 caller/callee: server는 bit state를 signal handler에서 직접 바꾸고 generic ACK/NACK signal을 보냈습니다. `ebed06775b92`의 record schema는 아직 runtime sequence와 연결되지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: generic ACK에는 bit identity가 없고 `sendto`를 시그널 처리 함수에서 수행할 수 없습니다. 다만 상태 전이 자체도 handler에 남아 있어 완전한 async boundary는 아직 아닙니다.
- 변경된 decision과 state mutation 순서: session에 `g_sequence`를 추가하고 accepted bit마다 current token을 캡처한 `t_response_request`를 pipe에 씁니다. event loop가 pipe record를 읽어 Unix datagram `sendto`를 수행합니다. handler가 sender authorization → current sequence 캡처 → byte shift/bit count → 완성 byte 출력 또는 NUL reset → legacy signal ACK → active session이면 sequence 증가 → response work pipe write를 수행합니다. normal loop는 record를 읽어 `send_response`를 호출합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: pipe write가 full record가 아니면 `g_response_overflow`를 세웁니다. 이때 bit/output state는 이미 바뀌었을 수 있으므로 queue loss를 복구하지 못합니다. 이 SHA의 `respond_to_bit`은 datagram send failure를 authoritative session error로 완전히 연결하지 않았습니다.
- 후속 commit이 강화하거나 교체하는 부분: `d3eacbbfeadc`가 exact sequence ACK를 client success 조건으로 삼고 `aeb1b00867f4`가 legacy signal path를 제거합니다. `22363f83ff25`에서는 bit mutation 자체도 self-pipe consumer로 이동합니다.

| 추적 항목 | 학습자 기록 | 코드 근거 |
| --- | --- | --- |
| 직전 architecture/state | handler가 bit state와 generic signal ACK를 직접 처리 | parent `src/server.c: handle_bit` |
| 해결하려던 핵심 문제 | ACK identity와 async-unsafe socket send 분리 | `g_sequence`, `t_response_request`, response pipe |
| 실패 가능한 interleaving 또는 partial failure | state mutation 뒤 pipe enqueue 실패로 응답 work 손실 | `g_response_overflow` set 시점 |
| 선택한 decision | bit마다 token을 캡처하고 datagram send를 loop로 위임 | `queue_response` → `respond_to_bit` |
| 소유권/lifecycle/상태 전이 | owner session이 sequence를 소유하며 NUL reset 시 함께 초기화 | `reset_session`, `g_sequence` |
| 후속 fix 또는 regression evidence | client correlation, legacy 제거, self-pipe event-loop 전환 | `d3eacbbfeadc`, `aeb1b00867f4`, `22363f83ff25` |

#### 보장 범위

**이 commit이 보장하는 것**

- [x] accepted bit마다 distinct sequence identity 부여
- [x] socket send를 normal 문맥으로 queue
- [x] full-size pipe write 실패 감지

**아직 보장하지 않는 것**

- [x] final async-signal-safe handler
- [x] single authoritative response path
- [x] queue-loss recovery
- [x] output-before-ACK commit

#### 코드 증거 기록

- 파일 경로: `src/server.c`, `src/client.c`, `include/minitalk.h`
- symbol 또는 함수: `handle_bit`, `queue_response`, `respond_to_bit`, `send_response`, `reset_session`
- 확인한 state fields: `g_client_pid`, `g_current_byte`, `g_received_bits`, `g_line_started`, `g_sequence`, `g_response_overflow`
- caller → callee: 시그널 처리 함수 `handle_bit` → `queue_response` → pipe; event loop → `respond_to_bit` → `send_response` → `sendto`
- 핵심 branch 또는 mutation 순서: handler가 sender authorization → current sequence 캡처 → byte shift/bit count → 완성 byte 출력 또는 NUL reset → legacy signal ACK → active session이면 sequence 증가 → response work pipe write를 수행합니다. normal loop는 record를 읽어 `send_response`를 호출합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: schema가 server sequence state와 queued ACK work로 연결됐지만 handler가 authorization/assembly/output/legacy ACK를 계속 담당합니다.
- 삽입한 최소 코드 조각과 선택 이유: `4234233ebd30`, `src/server.c: queue_response` 계열: token을 가진 ACK work와 full-size write 실패 정책을 함께 보여 줍니다.

```c
request.client_pid = g_client_pid;
request.kind = MT_RESPONSE_ACK;
request.token = sequence;
request.status = MT_RESPONSE_OK;
if (write(g_response_pipe[1], &request, sizeof(request)) != sizeof(request))
	g_response_overflow = 1;
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`d3eacbbfeadc`에서 client가 matching sequence response만 수락합니다.
### 8. `d3eacbbfeadc` — feat(client): 비트 ACK를 sequence로 상관 검증

- **중요도:** A
- **태그:** RESPONSE, RISK
- **개발 흐름에서의 역할:** client가 expected server source와 exact current sequence의 datagram ACK만 bit success로 수락합니다.

#### 원문에서 확정된 맥락

signal 전 server endpoint를 검증하고 monotonic deadline을 설정합니다. size, source, PID, kind, token, magic, status가 모두 맞아야 sequence가 증가합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] signal 전 expected server socket validation
- [x] bit별 absolute `CLOCK_MONOTONIC` deadline
- [x] `kill` 뒤 response receive loop
- [x] size/source/PID/magic/kind/token/status conjunction
- [x] invalid candidate가 deadline을 갱신하지 않는 branch
- [x] matching ACK 뒤에만 sequence/cursor 증가

#### 비교 기준

`4234233ebd30`의 server token initial/increment rule과 일치시키고, signal ACK가 client success 판단에서 빠진 diff를 확인했습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: server는 sequence datagram ACK를 만들지만 client send path는 여전히 generic signal ACK를 먼저 기다리고 datagram response를 보조적으로 사용했습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: source나 token이 다른 stale/forged datagram이 도착했다는 사실만으로 current bit를 완료하면 cursor가 잘못 전진합니다.
- 변경된 decision과 state mutation 순서: `send_bit`이 expected server endpoint와 current sequence를 고정하고 signal을 보낸 뒤 one absolute monotonic deadline 안에서 exact ACK predicate를 만족하는 datagram만 기다리도록 했습니다. server endpoint validation → absolute deadline 생성 → data `kill` → response receive loop → exact size/source/magic/server PID/ACK kind/current token/OK status 확인 → success 반환 → caller가 sequence와 bit cursor 증가입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: invalid candidate는 버리고 같은 deadline으로 계속 기다립니다. timeout/send/receive permanent error는 current sequence와 bit cursor를 유지한 채 실패합니다.
- 후속 commit이 강화하거나 교체하는 부분: `aeb1b00867f4`가 남은 signal ACK handler/mask를 제거하고 `b361ef9745ff`/`1ed2acbaa353`가 forged·oversized·flood input을 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] per-bit response correlation
- [x] stale/forged frame이 cursor를 전진시키지 않음
- [x] bit wait의 finite deadline

**아직 보장하지 않는 것**

- [x] output-before-ACK commit
- [x] legacy handler/mask 완전 제거
- [x] invalid-flood regression evidence

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/response_channel.c`, `include/minitalk.h`
- symbol 또는 함수: `send_bit`, `wait_for_response`, `read_response`, `response_matches`
- 확인한 state fields: `sequence`, `deadline`, `expected server path`
- caller → callee: `send_byte` → `send_bit` → `kill` → `wait_for_response`/`recvfrom`
- 핵심 branch 또는 mutation 순서: server endpoint validation → absolute deadline 생성 → data `kill` → response receive loop → exact size/source/magic/server PID/ACK kind/current token/OK status 확인 → success 반환 → caller가 sequence와 bit cursor 증가입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: signal ACK wait를 client progress condition에서 제거하고 sequence datagram response predicate를 authoritative success로 사용합니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`aeb1b00867f4`에서 generic signal response와 implicit first-bit ownership을 삭제합니다.
### 9. `aeb1b00867f4` — refactor(protocol): 이전 signal ACK 경로 제거

- **중요도:** A
- **태그:** ARCH, RESPONSE, REFACTOR
- **개발 흐름에서의 역할:** obsolete ACK/NACK signal machinery를 client/server/shared/test sender에서 제거하고 datagram response만 남깁니다.

#### 원문에서 확정된 맥락

handlers, alarm flags, wait masks, signal responses가 사라지며 server는 explicit acquisition 없는 sender signal로 owner를 지정하지 않습니다. owner에게 response send가 실패하면 session을 reset합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] ACK/NACK constants와 handler registration 삭제
- [x] client signal flags/alarm/mask/`sigsuspend` 제거
- [x] server first-bit owner assignment branch 제거
- [x] unauthorized sender signal ignore
- [x] bit success의 단일 datagram path
- [x] response send failure 뒤 session reset

#### 비교 기준

`d3eacbbfeadc` tree의 ACK/NACK macros, signal handlers, alarm/mask state, server first-bit assignment branch를 모두 찾아 삭제 후 남는 datagram path를 확인했습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `d3eacbbfeadc`에서 datagram ACK가 progress condition이 됐지만 signal ACK/NACK constants, handlers, wait masks와 server first-bit capture branch가 남아 있었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 두 success mechanism은 서로 다른 결과를 낼 수 있고, data signal의 implicit owner capture는 ACQUIRE 검증을 우회해 소유권 authority가 둘이 됩니다.
- 변경된 decision과 state mutation 순서: client와 test sender의 signal response machinery를 삭제하고 server에서 ACK/NACK `kill` 및 `g_client_pid == 0` first-bit assignment를 제거했습니다. ACQUIRE가 owner를 정한 뒤에만 owner PID의 data event를 처리합니다. bit/output 후 sequence datagram ACK send가 유일한 정상 처리가며 owner에게 response send가 실패하면 `reset_session(1)`을 호출합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: owner가 0이거나 sender가 owner와 다르면 data signal을 무시합니다. datagram send failure는 current session reset/오류 처리로 이어지며 generic signal 성공으로 보완하지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `1487a861046e`가 마지막 fixed gap을 제거하고 `db2004556d8b`가 output-before-ACK ordering을 보강합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | signal ACK와 datagram ACK가 함께 있어도 같은 success를 나타냅니다. | 직전 client/server tree에 두 response path가 동시에 존재했습니다. |
| 실제 failure 또는 위험 | 두 path가 다른 결과를 내고 implicit first-bit path가 acquisition을 우회합니다. | server의 `g_client_pid == 0` branch가 unaquired signal sender를 owner로 지정했습니다. |
| root cause | protocol success와 소유권의 authoritative path가 둘 이상이었습니다. | 시그널 처리 함수/mask 및 first-bit assignment가 datagram path와 병존했습니다. |
| 수정 불변 조건/decision | success는 sequence datagram ACK만, owner는 validated ACQUIRE 뒤에만 존재합니다. | signal response code 삭제와 unauthorized sender early return |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: ACQUIRE가 owner를 정한 뒤에만 owner PID의 data event를 처리합니다. bit/output 후 sequence datagram ACK send가 유일한 정상 처리가며 owner에게 response send가 실패하면 `reset_session(1)`을 호출합니다.
- 수정된 invariant를 고정하는 후속 regression test: `b361ef9745ff`가 response identity를, `e56e8cc87315`가 idle reservation exclusivity를 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] single source of truth for ACK
- [x] 소유권은 ACQUIRE 뒤에만 시작
- [x] unauthorized data signal이 state를 바꾸지 않음

**아직 보장하지 않는 것**

- [x] output commit ordering
- [x] fixed delay 제거
- [x] same-UID authentication

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/server.c`, `tests/session_sender.c`, `include/minitalk.h`
- symbol 또는 함수: `send_bit`, `process_bit`, `send_response`, `reset_session`
- 확인한 state fields: `g_client_pid`, `sequence`
- caller → callee: validated ACQUIRE → server owner state; client bit `kill` → server process → datagram ACK → client exact wait
- 핵심 branch 또는 mutation 순서: ACQUIRE가 owner를 정한 뒤에만 owner PID의 data event를 처리합니다. bit/output 후 sequence datagram ACK send가 유일한 정상 처리가며 owner에게 response send가 실패하면 `reset_session(1)`을 호출합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: parallel signal response machinery와 implicit data-based 소유권이 삭제돼 datagram ACK와 ACQUIRE만 authoritative path로 남습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`1487a861046e`에서 remaining timing workaround가 제거됩니다.
### 10. `1487a861046e` — perf(protocol): 검증된 ACK 뒤 고정 지연 제거

- **중요도:** A
- **태그:** PERF, RESPONSE
- **개발 흐름에서의 역할:** matching sequence ACK 뒤 fixed sleep을 production client와 session sender에서 제거합니다.

#### 원문에서 확정된 맥락

client는 여전히 one bit를 보내고 exact ACK를 기다린 뒤 다음 bit로 진행합니다. delay 제거는 ordering이 time이 아니라 correlation에 의해 보장됨을 보여 줍니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] production `wait_signal_gap` helper/constant 삭제
- [x] session sender의 동일 delay 삭제
- [x] send와 matching wait가 한 iteration에서 직렬화
- [x] ACK 전 cursor/sequence 불변
- [x] timeout과 success branch 유지

#### 비교 기준

`4f17de94e025`의 remainder-aware gap implementation과 `aeb1b00867f4`의 datagram-only path를 함께 비교해 sleep만 제거됐음을 확인했습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: datagram ACK가 progress condition이 된 뒤에도 production client와 test session sender는 ACK success 뒤 5ms gap을 유지했습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: validated ACK가 server의 accepted bit를 이미 식별하므로 고정 gap은 correctness에 기여하지 않고 bit마다 latency만 누적합니다.
- 변경된 decision과 state mutation 순서: `MT_SIGNAL_GAP` 상수와 sleep helper/call을 client와 session sender에서 삭제했습니다. current bit signal 전송 → exact current sequence ACK wait → success 반환 → caller가 sequence/bit cursor 증가 → 즉시 다음 bit입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: ACK 전에 cursor나 sequence는 증가하지 않습니다. send/receive/timeout failure는 current bit에서 전체 transmission을 중단합니다.
- 후속 commit이 강화하거나 교체하는 부분: 이 commit이 Thread final state입니다. 후속 output commit fix는 ACK를 보내기 전 server stdout 성공 조건을 강화합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] sleep 없는 one-bit-in-flight ordering
- [x] bit당 unnecessary latency 제거

**아직 보장하지 않는 것**

- [x] multiple bits in flight
- [x] retransmission
- [x] exactly-once transaction
- [x] ACK loss recovery

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `tests/session_sender.c`, `include/minitalk.h`
- symbol 또는 함수: `send_bit`, `send_byte`
- 확인한 state fields: `sequence`, `bit index`
- caller → callee: `send_byte` → `send_bit` → exact response wait → caller cursor advance
- 핵심 branch 또는 mutation 순서: current bit signal 전송 → exact current sequence ACK wait → success 반환 → caller가 sequence/bit cursor 증가 → 즉시 다음 bit입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: pacing helper와 상수만 제거되고 send→validate→advance stop-and-wait 순서는 유지됩니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

이 commit이 Thread final state입니다. elapsed time은 success condition으로 남지 않습니다.

## 6. 불변 조건 ledger

### 원자료에서 확인된 핵심 불변 조건

- 논리적으로 한 번에 하나의 data bit만 전송 중입니다.
- client는 expected server endpoint에서 온 exact sequence ACK를 수락한 뒤에만 다음 bit로 진행합니다.
- stale, forged, malformed, uncorrelated response는 transition을 완료하지 않습니다.
- 무응답은 bounded failure로 끝나며 현재 bit를 성공 처리하지 않습니다.

### 시간에 따른 변화 기록

| Commit | 원자료에서 확인된 변화 | 실제 state/condition | code evidence | 상태: 도입·강화·부족·복구·검증 |
| --- | --- | --- | --- | --- |
| `89637d63b56f` | message byte를 MSB부터 signal bit로 직렬화하고 provisional fixed delay로 전송 속도를 낮춥니다. | bit index는 `send_bit`가 성공한 뒤에만 감소하지만 성공 조건은 `kill`과 fixed delay뿐입니다. | `src/client.c: send_byte`, `send_bit` | 도입·부족 |
| `78de95b3cacb` | bit마다 signal ACK를 기다리는 stop-and-wait를 도입하고 ACK-before-wait race를 막습니다. | `g_acknowledged`가 false인 동안 current bit가 outstanding이며 ACK 뒤에만 bit index가 감소합니다. | `src/client.c: send_bit`, `src/server.c` ACK branch | 강화·부족 |
| `765efe7b75c9` | ACK wait를 alarm 기반 timeout으로 제한하고 timeout과 다른 send failure를 구분합니다. | ACK 또는 timeout 중 하나가 outstanding bit의 종료 상태이며 timeout에는 cursor가 전진하지 않습니다. | `src/client.c: send_bit` alarm/flag branches | 강화 |
| `342aea9ce9a8` | signal ACK 뒤 short inter-signal gap을 유지하고 acknowledgement deadline을 늘립니다. | ACK는 전진 조건, 500µs gap은 ACK 뒤 scheduling workaround입니다. | `src/client.c: send_bit`, timeout/gap constants | 강화·임시 |
| `4f17de94e025` | `nanosleep`이 `EINTR`로 중단되면 returned remainder로 같은 logical gap을 이어갑니다. | pacing request는 EINTR 때 remaining time을 소유하지만 protocol success는 여전히 signal ACK입니다. | `src/client.c: wait_signal_gap` | 복구·임시 |
| `ebed06775b92` | `ACQUIRE`, `READY`, `ACK`를 표현하는 request/response records와 identity fields를 정의합니다. | wire schema가 nonce/token과 PID identity를 표현하지만 아직 runtime state를 전진시키지 않습니다. | `include/minitalk.h: t_mt_request`, `t_mt_response` | 도입 |
| `4234233ebd30` | accepted bit에 sequence를 부여하고 datagram ACK send work를 pipe로 queue해 direct signal response에서 분리합니다. | server가 accepted bit의 pre-increment sequence를 ACK token으로 캡처하고 pipe work로 전달합니다. | `src/server.c: handle_bit`, `queue_response`, `respond_to_bit` | 도입·부족 |
| `d3eacbbfeadc` | client가 expected server source와 exact current sequence의 datagram ACK만 bit success로 수락합니다. | client의 current `sequence`가 outstanding bit identity이며 exact ACK 뒤에만 증가합니다. | `src/client.c: send_bit`, response validation helpers | 강화 |
| `aeb1b00867f4` | obsolete ACK/NACK signal machinery를 client/server/shared/test sender에서 제거하고 datagram response만 남깁니다. | owner와 bit progress의 authoritative source가 ACQUIRE와 sequence datagram ACK 하나로 수렴합니다. | `src/server.c` first-bit branch 삭제, `src/client.c` signal wait 삭제 | 복구·강화 |
| `1487a861046e` | matching sequence ACK 뒤 fixed sleep을 production client와 session sender에서 제거합니다. | one outstanding bit의 progress는 exact ACK 하나로만 결정되며 delay state가 없습니다. | `src/client.c: send_bit`, delay helper removal diff | 강화·완료 |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 또는 상태 | 실제 failure/위험 | Fix 또는 전환 commit | 수정된 decision/불변 조건 | Test 또는 후속 검증 | 학습자 code evidence |
| --- | --- | --- | --- | --- | --- |
| 여러 bit를 시간 간격만 두고 전송 | receiver 완료를 증명하지 못해 signal 병합과 byte 정렬 손상 위험 | `78de95b3cacb` | bit별 signal ACK stop-and-wait | 이 단계 전용 test commit은 Thread에 없음 | `src/client.c: send_bit`; ACK block-before-send |
| generic signal ACK | 어느 bit의 response인지 식별 불가하고 parallel success path가 남음 | `ebed06775b92 → 4234233ebd30 → d3eacbbfeadc → aeb1b00867f4` | wire field와 sequence correlation, legacy path 제거 | `b361ef9745ff`, `1ed2acbaa353` | record schema, server `g_sequence`, client exact predicate |
| ACK 뒤 fixed delay | validated ACK 뒤에도 bit마다 latency 누적 | `1487a861046e` | delay 제거, ACK를 causal progress 근거로 사용 | production client/session sender diff | sleep helper/constant 제거 뒤 send→wait→advance 유지 |

전용 test commit이 없는 연결에는 존재하지 않는 test를 만들어 적지 않았습니다.

## 8. 소유권 / state / responsibility 변화

| 단계 | state 또는 responsibility owner | transition | 당시 한계 또는 다음 변화 | 실제 symbol/field |
| --- | --- | --- | --- | --- |
| 초기 | client bit cursor | elapsed time 뒤 다음 bit | server 처리 완료와 직접 연결되지 않음 | `send_byte` bit index, `send_bit` |
| signal ACK | client ACK flag와 signal mask | handler flag 뒤 다음 bit | transition identity 없음 | `g_acknowledged`, `sigprocmask`, `sigsuspend` |
| datagram 전환 | server sequence와 queued response | accepted bit마다 token 부여 | legacy ACK와 잠시 공존 | `g_sequence`, `t_response_request`, response pipe |
| 최종 | client outstanding sequence | exact ACK 뒤 증가 | fixed delay와 signal ACK 제거 | client `sequence`, response predicate |

## 9. 개발 흐름의 최종 상태

원자료에서 확인된 최종 조건:

- data bit는 계속 `SIGUSR1`/`SIGUSR2`로 전달됩니다.
- 진행 허가는 expected server endpoint의 sequence-correlated datagram ACK 하나로 결정됩니다.
- fixed sleep은 ordering 불변 조건에 포함되지 않습니다.
- timeout은 uncertainty를 bounded failure로 끝내며 retransmission 또는 exactly-once를 제공하지 않습니다.

학습자 기록:

- 최종 state fields와 owner: client의 current `sequence`와 bit index가 outstanding transition을 소유합니다. server는 owner session의 `g_sequence`와 byte/bit state를 소유하고 같은 token으로 ACK를 만듭니다.
- 정상 transition 순서: payload byte 선택 → MSB-first signal 선택 → expected server/deadline 확정 → `kill` → exact datagram ACK 검증 → sequence와 bit cursor 증가입니다.
- 실패 시 중단·reset·cleanup 순서: endpoint validation, signal send, response receive 또는 deadline failure에서 cursor를 유지하고 client가 실패합니다. server response failure는 해당 시점의 session reset/오류 처리로 이어집니다.
- 최종 상태가 보장하지 않는 것: retransmission, duplicate suppression, multiple in-flight bits, output와 ACK의 atomic transaction, same-UID peer의 cryptographic authentication은 제공하지 않습니다.
- 이 개발 흐름을 한 문단으로 설명한 최종 서술: 이 개발 흐름은 “시간이 충분히 지났으니 다음 bit”라는 추정에서 출발해, ACK signal을 block-before-send로 기다리는 stop-and-wait, finite timeout, identity를 가진 datagram record, server sequence와 client acceptance predicate를 차례로 도입합니다. legacy signal path를 제거한 뒤 exact ACK만 progress를 허용하므로 마지막 fixed delay를 삭제해도 one-bit-in-flight ordering이 유지됩니다.

## 10. 최종 architecture 또는 실행 순서 정리

- [x] client가 payload byte와 MSB-first bit 위치를 선택하는 지점
- [x] zero/one을 signal로 전송하는 호출
- [x] current sequence와 absolute deadline을 만드는 지점
- [x] server가 accepted bit에 같은 sequence ACK를 생성하는 경로
- [x] client가 source와 response fields를 검증하는 predicate
- [x] sequence와 bit cursor가 증가하는 유일한 지점
- [x] timeout/send/receive failure에서 중단되는 경로

```text
client main/send_byte
    -> target PID와 server endpoint 검증
    -> current byte의 bit 7..0 선택
    -> current sequence와 absolute deadline 확정
    -> kill(server_pid, zero_or_one_signal)
    -> server accepted bit mutation + same sequence ACK 생성
    -> client recvfrom: exact size/source/magic/PID/kind/token/status 검증
    -> match면 sequence/bit cursor 증가
    -> mismatch면 같은 deadline으로 반복
    -> timeout/error면 cursor 유지 후 실패/cleanup

```

- 실제 함수·파일을 반영한 완성 흐름: `src/client.c`의 `send_byte`/`send_bit`와 response receive helpers가 sender 흐름을, `src/server.c`의 bit processor와 `send_response`가 receiver 흐름을 구성합니다.
- asynchronous boundary: data signal delivery와 server handler/event processing 사이, 그리고 server Unix datagram send와 client `recvfrom` 사이입니다.
- externally visible commit point: client 관점에서는 expected server endpoint에서 exact current sequence ACK를 수락하는 순간입니다. Thread 4에서 server output-before-ACK 조건이 추가됩니다.
- cleanup owner: client endpoint는 client cleanup이, server endpoint와 response/event descriptors는 server cleanup이 소유합니다.

## 11. 학습 완료 자가 점검

- [x] commit map의 10개 SHA를 source 순서대로 모두 설명할 수 있습니다.
- [x] 각 code excerpt에 SHA, path, symbol, 선택 이유가 기록돼 있습니다.
- [x] 최종 HEAD 코드를 historical SHA의 증거로 사용한 곳이 없습니다.
- [x] 정상 경로와 실패 처리를 state mutation 순서로 설명할 수 있습니다.
- [x] source 확정 불변 조건과 직접 확인한 code evidence를 구분했습니다.
- [x] test commit의 불변 조건, failure, technique, production path, proves/not-proves를 기록했습니다.
- [x] Thread final state를 함수와 state field 수준으로 설명할 수 있습니다.
- [ ] 해당 SHA의 test를 로컬에서 직접 실행했습니다. — 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

### 이 Thread와 직접 연결된 Major Engineering Difficulties

- standard signal은 동일 signal의 임의 multiplicity를 reliable counted queue처럼 보존하지 않습니다.
- signal data channel과 datagram completion channel의 state ordering을 맞춰야 합니다.
- invalid response를 무시하는 동안에도 original monotonic deadline을 유지해야 합니다.

---

# 첫 비트 암묵적 점유에서 자원 상태를 고려한 세션 복구까지

> 완성형 해설서가 아닙니다. 아래 확정 사항을 기준으로 각 commit SHA의 실제 코드와 diff를 읽고 기록란을 채웁니다.

## 1. 개발 흐름 목표

첫 data signal의 sender PID를 암묵적으로 owner로 잡던 설계가 명시적인 `ACQUIRE`/`READY` reservation으로 바뀌고, 마지막에는 PID 존재뿐 아니라 usable response endpoint까지 owner availability에 포함되는 과정을 복원합니다.

### Significance

single byte accumulator를 여러 sender가 공유하므로 소유권이 없으면 bit가 섞입니다. first-bit capture는 interleaving을 막지만 data 전 reservation을 표현하지 못합니다. explicit acquisition이 authority를 control channel로 옮긴 뒤에는 zombie처럼 PID는 남았지만 response socket은 사라진 상태가 PID-only liveness 가정을 깨뜨립니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- owner PID, partial byte, received-bit count, sequence, visible-line state는 언제 함께 생성·reset됩니까?
- first-bit 소유권은 competing sender를 어떻게 거부하며 explicit acquisition은 어떤 race를 없애는가?
- dead owner recovery에서 partial output을 newline으로 닫는 순서는 state reset과 어떻게 연결됩니까?
- `ACQUIRE` source path와 claimed PID는 owner assignment 전에 어디에서 함께 검증됩니까?
- data를 전혀 보내지 않은 live reserver도 exclusive하다는 사실을 어떤 test가 고정하는가?
- exited-but-unreaped owner에서 `kill(pid, 0)`과 response endpoint 상태가 왜 다르게 보이는가?

## 3. 완료 기준

- [x] implicit first-bit 소유권과 explicit acquisition의 transition을 code diff로 비교했습니다.
- [x] owner recovery가 reset해야 하는 coupled fields와 partial-line delimiter를 기록했습니다.
- [x] live competitor, dead owner, idle reserver, zombie owner의 server 결과를 설명했습니다.
- [x] PID-only liveness의 root cause와 endpoint-aware fix를 helper/caller로 연결했습니다.
- [x] 각 test commit의 process orchestration과 production path를 분리해 기록했습니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `10a7211969bf` | fix(server): 활성 세션에 다른 송신자 거부 | A | SESSION, RISK | active sender PID를 기록하고 NUL terminator까지 다른 sender의 data signal을 거부합니다. |
| 2 | `bc337552961a` | fix(server): 종료된 송신자의 세션 복구 | A | SESSION, PROCESS_LIFECYCLE, RISK | recorded owner가 사라지면 partial byte, bit count, owner, line state를 함께 reset하고 new sender가 진행할 수 있게 합니다. |
| 3 | `bdccf91f5a44` | test(server): 중단·경쟁 송신자 세션 검증 | A | TEST, SESSION, RISK | dedicated sender로 one-bit abandon, complete-byte-plus-partial abandon, live competition, owner exit recovery를 재현합니다. |
| 4 | `caf2feec4971` | feat(server): 획득 요청을 검증해 세션 소유권 예약 | S | ARCH, SESSION, CORE | exact `ACQUIRE` datagram을 검증한 뒤 data 전에 owner를 예약하고 `READY` 또는 `BUSY`를 반환합니다. |
| 5 | `f8e8444c5ded` | feat(client): READY 응답을 출처와 nonce로 상관 검증 | A | RESPONSE, RISK, INTEGRATION | client가 nonzero nonce ACQUIRE를 보내고 expected source, PID, fields, deadline이 맞는 READY만 수락합니다. |
| 6 | `e56e8cc87315` | test(session): 데이터 없는 활성 예약 경쟁 검증 | A | TEST, SESSION | session helper의 `reserve` mode가 acquisition만 완료하고 data를 보내지 않은 채 live owner를 유지합니다. |
| 7 | `1e3da4580733` | fix(server): 응답 경로가 사라진 세션 소유자 회수 | S | SESSION, PROCESS_LIFECYCLE, DEBUG | owner availability를 process presence와 expected same-UID client response socket usability의 결합으로 정의합니다. |
| 8 | `a481bfabb7b5` | test(session): 종료 송신자 회수 전 새 세션 복구 검증 | A | TEST, SESSION, PROCESS_LIFECYCLE | partial-session child를 exit시킨 뒤 `waitid(..., WNOWAIT)`로 unreaped 상태를 유지해 zombie PID와 vanished endpoint를 동시에 관측합니다. |

확인 원칙:

- 각 항목은 해당 SHA의 tree를 기준으로 읽었습니다.
- 변경 전 상태는 해당 SHA의 parent 또는 지정된 이전 관련 SHA에서 확인했습니다.
- 같은 commit이 다른 Thread에 다시 등장해도 이 개발 흐름의 질문으로 별도 기록했습니다.
- runtime test는 실행하지 않았으며, 실행 결과처럼 표현하지 않았습니다.

## 5. 커밋별 학습 기록

### 1. `10a7211969bf` — fix(server): 활성 세션에 다른 송신자 거부

- **중요도:** A
- **태그:** SESSION, RISK
- **개발 흐름에서의 역할:** active sender PID를 기록하고 NUL terminator까지 다른 sender의 data signal을 거부합니다.

#### 원문에서 확정된 맥락

server에는 하나의 partial-byte accumulator만 있으므로 sender interleaving은 undecodable byte를 만듭니다. 이 단계의 소유권은 first accepted bit에서 시작합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] owner PID field의 0 initial value
- [x] `siginfo_t.si_pid`로 first owner를 지정하는 branch
- [x] non-owner가 accumulator mutation 전에 거부되는 condition
- [x] competitor NACK와 client rejected status
- [x] NUL terminator 뒤 owner/byte/bit-count reset

#### 비교 기준

직전 handler의 shared accumulator 앞에 sender identity check와 first-owner assignment가 추가됐습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: server의 `current_byte`와 `received_bits`는 process-wide 단일 accumulator였고 어느 sender가 만든 partial state인지 기록하지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 서로 다른 PID가 번갈아 signal을 보내면 두 client의 bit가 같은 byte에 합쳐지고 한 client의 NUL이 다른 client의 state를 해제할 수 있습니다.
- 변경된 decision과 state mutation 순서: `g_client_pid`를 추가해 첫 accepted signal의 `si_pid`를 owner로 저장하고 NUL terminator가 완성될 때까지 다른 PID의 signal을 accumulator에 넣지 않습니다. handler entry → `g_client_pid == 0`이면 sender를 owner로 지정 → owner mismatch면 NACK 후 return → owner bit만 shift/count → 8 bits 완성 → NUL이면 byte/count/owner reset입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: non-owner는 state mutation 전에 NACK를 받고 client의 reject failure로 끝납니다. owner가 중간에 종료되면 NUL이 오지 않아 session이 영구 점유되는 한계가 남습니다.
- 후속 commit이 강화하거나 교체하는 부분: `bc337552961a`가 vanished owner recovery를 추가하고 `caf2feec4971`가 ownership start를 ACQUIRE control message로 옮깁니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | 한 client의 signal sequence가 다른 client와 섞이지 않습니다. | 직전 server state에는 sender PID field가 없었습니다. |
| 실제 failure 또는 위험 | 두 sender의 bits가 하나의 `(current_byte, received_bits)`를 공유합니다. | 모든 data signal이 같은 handler globals를 수정했습니다. |
| root cause | shared receive state에 owner identity가 없습니다. | 새 `g_client_pid`와 mismatch early return이 직접 보강점입니다. |
| 수정 불변 조건/decision | 한 owner PID만 NUL frame까지 receive state를 변경합니다. | `src/server.c: handle_bit` owner assignment/check/reset |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: handler entry → `g_client_pid == 0`이면 sender를 owner로 지정 → owner mismatch면 NACK 후 return → owner bit만 shift/count → 8 bits 완성 → NUL이면 byte/count/owner reset입니다.
- 수정된 invariant를 고정하는 후속 regression test: `bdccf91f5a44`의 live competitor scenario가 이 invariant를 process 수준에서 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] single active sender로 bit interleaving 차단

**아직 보장하지 않는 것**

- [x] data-before-reservation
- [x] correlated READY/BUSY
- [x] abandoned-owner recovery

#### 코드 증거 기록

- 파일 경로: `src/server.c`, `src/client.c`, `include/minitalk.h`
- symbol 또는 함수: `handle_bit`, `send_bit`
- 확인한 state fields: `g_client_pid`, `g_current_byte`, `g_received_bits`, `g_rejected`
- caller → callee: server 시그널 처리 함수 → owner check → bit assembly → ACK/NACK `kill`; client handler flag → send status
- 핵심 branch 또는 mutation 순서: handler entry → `g_client_pid == 0`이면 sender를 owner로 지정 → owner mismatch면 NACK 후 return → owner bit만 shift/count → 8 bits 완성 → NUL이면 byte/count/owner reset입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: shared accumulator에 owner identity와 competitor rejection이 추가됐지만 release는 NUL frame뿐입니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`bc337552961a`에서 owner가 terminator 전에 사라진 경우 recovery가 추가됩니다.
### 2. `bc337552961a` — fix(server): 종료된 송신자의 세션 복구

- **중요도:** A
- **태그:** SESSION, PROCESS_LIFECYCLE, RISK
- **개발 흐름에서의 역할:** recorded owner가 사라지면 partial byte, bit count, owner, line state를 함께 reset하고 new sender가 진행할 수 있게 합니다.

#### 원문에서 확정된 맥락

`kill(owner, 0)`과 `ESRCH`로 liveness를 판단합니다. visible payload가 있으면 newline으로 닫고, vanished owner에게 ACK send가 실패해도 recovery합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `kill(owner, 0)`와 `errno == ESRCH` branch
- [x] owner/current-byte/bit-count/line-state coupled reset
- [x] visible partial line에 newline을 쓰는 condition
- [x] ACK send ESRCH에서 recovery
- [x] new sender가 reset된 state에서 시작하는 순서

#### 비교 기준

`10a7211969bf`의 NUL-only release에 PID probe, line-visible state와 recovery reset path가 추가됐습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `10a7211969bf`는 owner가 NUL terminator를 보내야만 `g_client_pid`와 accumulator를 해제했습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: owner가 mid-byte 또는 complete byte 뒤 partial message 상태에서 종료하면 server가 계속 BUSY/NACK만 보내고 다음 sender는 진행할 수 없습니다.
- 변경된 decision과 state mutation 순서: `g_line_started`와 `reset_session`을 추가하고 owner probe가 `ESRCH`이거나 ACK send가 ESRCH로 실패하면 visible line을 newline으로 닫은 뒤 coupled state를 초기화합니다. competitor signal → `kill(owner, 0)` → `ESRCH`면 `reset_session(1)` → newline 필요 시 출력 → byte/bit/owner/line fields reset → new sender가 clean state에서 first owner가 됩니다.
- 정상 경로와 failure 경로가 갈라지는 조건: owner가 존재하면 competitor를 계속 거부합니다. owner ACK `kill`이 ESRCH이면 reset합니다. 이 SHA의 output write 결과는 아직 protocol failure로 충분히 전파되지 않으며 zombie PID는 `kill(pid,0)`에 성공해 회수되지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `bdccf91f5a44`가 abandon/competition을 검증하고 `1e3da4580733`이 PID-only liveness를 endpoint-aware availability로 바꿉니다. Thread 4의 `db2004556d8b`가 delimiter write를 commit condition으로 만듭니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | owner는 NUL terminator로 정상 종료합니다. | 직전 release branch는 completed NUL byte뿐이었습니다. |
| 실제 failure 또는 위험 | mid-message exit가 owner와 partial state를 남겨 server를 점유합니다. | `g_client_pid`, partial byte/count가 process exit와 독립적으로 유지됩니다. |
| root cause | owner 수명 종료 뒤 coupled fields를 회수하는 transition이 없습니다. | 새 `reset_session`이 byte/bit/PID/line을 한곳에서 초기화합니다. |
| 수정 불변 조건/decision | PID absence에서 visible line을 delimit하고 receive state를 함께 reset합니다. | `kill(...,0)`/ACK ESRCH branches → `reset_session(1)` |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: competitor signal → `kill(owner, 0)` → `ESRCH`면 `reset_session(1)` → newline 필요 시 출력 → byte/bit/owner/line fields reset → new sender가 clean state에서 first owner가 됩니다.
- 수정된 invariant를 고정하는 후속 regression test: `bdccf91f5a44`가 one-bit/partial abandon과 live competition을 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] dead PID가 accumulator를 영구 점유하지 않음
- [x] visible partial output의 session delimiting 의도

**아직 보장하지 않는 것**

- [x] zombie owner detection
- [x] endpoint liveness
- [x] all-or-failure delimiter commit

#### 코드 증거 기록

- 파일 경로: `src/server.c`
- symbol 또는 함수: `handle_bit`, `reset_session`, `flush_byte`
- 확인한 state fields: `g_client_pid`, `g_current_byte`, `g_received_bits`, `g_line_started`
- caller → callee: 시그널 처리 함수 → owner probe/ACK send → `reset_session` → output/reset
- 핵심 branch 또는 mutation 순서: competitor signal → `kill(owner, 0)` → `ESRCH`면 `reset_session(1)` → newline 필요 시 출력 → byte/bit/owner/line fields reset → new sender가 clean state에서 first owner가 됩니다.
- parent 또는 이전 관련 SHA와의 diff 요약: NUL-only cleanup이 owner disappearance에서도 실행되며 visible output 경계를 위한 `g_line_started`가 추가됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`bdccf91f5a44`에서 abandoned/live-competition cases를 검증합니다.
### 3. `bdccf91f5a44` — test(server): 중단·경쟁 송신자 세션 검증

- **중요도:** A
- **태그:** TEST, SESSION, RISK
- **개발 흐름에서의 역할:** dedicated sender로 one-bit abandon, complete-byte-plus-partial abandon, live competition, owner exit recovery를 재현합니다.

#### 원문에서 확정된 맥락

normal client가 만들기 어려운 coupled session transitions를 real processes로 검증하고 expected output에 abandoned visible line을 닫는 newline을 포함합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] sender mode별 signal count와 exit/pause 지점
- [x] server/sender/competitor process orchestration
- [x] live owner 유지 중 normal client 경쟁 순서
- [x] owner exit 뒤 replacement synchronization
- [x] partial byte discard와 complete-byte output 기대값
- [x] failure cleanup/terminate/reap logic

#### 비교 기준

`bc337552961a`의 owner check/reset branches를 각 sender mode와 expected output에 대응시켰습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: dead-owner reset과 competitor rejection은 구현됐지만 실제 process exit/interleaving에서 partial byte와 line output이 원하는 대로 분리되는 regression evidence가 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: normal client는 항상 full frame을 보내므로 one-bit abandon, complete-byte 뒤 partial abandon, live holder를 deterministic하게 만들 수 없습니다.
- 변경된 decision과 state mutation 순서: `tests/session_sender.c`에 `bit`, `partial`, `hold` modes를 두고 shell test가 real server/client/sender processes를 순서대로 실행합니다. server start → specialized sender가 지정 bit 수만 전송하거나 pause → competitor client 실행/상태 확인 → owner exit 또는 terminate → replacement client → expected stdout/stderr/status 비교 → 모든 child reap/cleanup입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: one-bit abandon은 partial byte를 폐기하고 출력이 없어야 합니다. complete `X` 뒤 partial abandon은 `X
`을 남깁니다. live holder 동안 competitor는 실패하고, holder exit 뒤 replacement는 성공해야 합니다.
- 후속 commit이 강화하거나 교체하는 부분: `caf2feec4971` 이후 test helper가 explicit acquisition을 사용하도록 변하고 `e56e8cc87315`이 no-data reservation을 별도로 검증합니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** 한 live owner만 receive state를 변경하고 dead owner의 partial state는 다음 sender가 상속하지 않습니다.
- **재현하는 failure 또는 boundary:** one-bit abandon, complete-byte-plus-partial abandon, live competitor, owner exit recovery
- **사용한 test technique:** specialized sender + real server/client processes + exact stdout/status comparison
- **분류:** targeted process-level integration regression
- **failure 주입 또는 process orchestration 시작 지점:** `tests/session_ownership.sh`가 server를 시작하고 `session_sender` mode를 선택합니다.
- **production code에 진입하는 최초 호출:** helper의 real `kill`이 production server 시그널 처리 함수에 진입합니다.
- **핵심 assertion과 관측값:** live competitor failure, owner exit 뒤 replacement success, one-bit no output, partial scenario의 `X
` 경계, child cleanup을 확인합니다.
- **증명하는 것:** live exclusivity<br>dead-owner recovery<br>partial-state discard<br>visible line delimiter
- **증명하지 않는 것:** ACQUIRE/READY semantics<br>zombie owner<br>write failure
- **후속 변경에서 막아야 할 구체적인 회귀:** 다른 sender의 bit가 active owner accumulator에 섞이거나 dead owner state가 replacement에 상속되는 회귀를 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] live owner exclusivity에 대한 process-level evidence
- [x] dead-owner recovery
- [x] partial-state discard
- [x] visible line delimiter

**아직 보장하지 않는 것**

- [x] ACQUIRE/READY semantics
- [x] zombie owner
- [x] write failure

#### 코드 증거 기록

- 파일 경로: `tests/session_sender.c`, `tests/session_ownership.sh`, `Makefile`
- symbol 또는 함수: `main`, `send_bits`, `wait_for_ack`
- 확인한 state fields: `sender mode`, `ready synchronization`, `child PIDs`
- caller → callee: test shell → session sender/client/server executables → production 시그널 처리 함수/session reset
- 핵심 branch 또는 mutation 순서: server start → specialized sender가 지정 bit 수만 전송하거나 pause → competitor client 실행/상태 확인 → owner exit 또는 terminate → replacement client → expected stdout/stderr/status 비교 → 모든 child reap/cleanup입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: specialized sender binary와 multi-process session scenarios가 test target에 추가됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`caf2feec4971`에서 ownership start mechanism이 control channel로 이동합니다.
### 4. `caf2feec4971` — feat(server): 획득 요청을 검증해 세션 소유권 예약

- **중요도:** S
- **태그:** ARCH, SESSION, CORE
- **개발 흐름에서의 역할:** exact `ACQUIRE` datagram을 검증한 뒤 data 전에 owner를 예약하고 `READY` 또는 `BUSY`를 반환합니다.

#### 원문에서 확정된 맥락

request size, magic, kind, client PID, process viability, source path가 모두 맞아야 합니다. prior unavailable owner는 reset하고 new READY send failure는 reservation rollback으로 이어집니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] response socket의 exact request-size check
- [x] magic/kind/client PID/source address validation
- [x] claimed PID에서 expected client path derivation
- [x] live owner BUSY branch
- [x] free state owner assignment와 fields init
- [x] nonce를 echo한 READY send
- [x] READY send failure rollback
- [x] explicit acquisition 없는 data signal 처리 branch의 잔존

#### 비교 기준

`bc337552961a`의 first-bit assignment와 new `handle_session_request` owner assignment를 나란히 확인했습니다. scaffold의 역할은 보존하되 handler bypass가 남은 실제 tree 차이를 기록했습니다.

#### S-level 재구성

- 이 commit 직전의 관련 state와 caller/callee: 소유권은 first data signal의 sender PID를 handler가 암묵적으로 캡처했습니다. control plane에는 `ACQUIRE` schema가 있었지만 server reservation transition이 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: data 전 exclusivity와 correlated READY/BUSY를 표현할 수 없고, first bit와 owner assignment가 하나의 async path에 묶였습니다.
- 변경된 decision과 state mutation 순서: server response socket을 `pselect` loop에 넣고 exact ACQUIRE datagram을 검증한 뒤 free state에서 owner fields를 초기화하고 request nonce를 READY token으로 되돌려줍니다. READY send가 실패하면 방금 만든 reservation을 rollback합니다. `recvfrom` → exact request size → magic/kind/client PID/source path/same-UID socket/process probe → 기존 owner available이면 BUSY → unavailable이면 recovery → free state owner assignment/receive fields init → READY send → 실패 시 just-created owner reset입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: invalid request는 owner state를 바꾸지 않습니다. live owner에는 BUSY를 보냅니다. READY send failure는 phantom owner를 남기지 않습니다. **관찰된 불일치:** 이 SHA의 data handler에는 `g_client_pid == 0`이면 first signal sender를 owner로 지정하는 구 branch가 아직 남아 있어, explicit ACQUIRE 없는 data를 완전히 차단하는 invariant는 이 commit 하나로는 성립하지 않습니다. 해당 bypass는 `aeb1b00867f4`에서 제거됩니다.
- 후속 commit이 강화하거나 교체하는 부분: `f8e8444c5ded`가 conforming client의 ACQUIRE/READY correlation을 구현하고 `aeb1b00867f4`가 남은 implicit first-bit bypass를 제거합니다. `e56e8cc87315`은 idle reservation을 검증합니다.

| 추적 항목 | 학습자 기록 | 코드 근거 |
| --- | --- | --- |
| 직전 architecture/state | first accepted data signal이 owner를 정함 | `bc337552961a: handle_bit` |
| 해결하려던 핵심 문제 | data 전 reservation과 correlated READY/BUSY | response socket request path |
| 실패 가능한 interleaving 또는 partial failure | READY send 실패 뒤 owner만 남는 phantom reservation | new-owner flag와 rollback branch |
| 선택한 decision | exact ACQUIRE validation 후 owner assign, nonce echo READY | `handle_session_request`, `send_response` |
| 소유권/lifecycle/상태 전이 | free→reserved→READY 또는 send failure→free | `g_client_pid` 및 receive fields |
| 후속 fix 또는 regression evidence | implicit first-bit bypass 제거와 idle reservation test | `aeb1b00867f4`, `e56e8cc87315` |

#### 보장 범위

**이 commit이 보장하는 것**

- [x] validated ACQUIRE/READY reservation path
- [x] source/PID binding을 통과한 owner assignment
- [x] READY send failure의 phantom owner rollback

**아직 보장하지 않는 것**

- [x] 이 SHA 단독의 data-before-ACQUIRE 강제
- [x] resource-aware zombie recovery
- [x] lease
- [x] full adversarial validation evidence

#### 코드 증거 기록

- 파일 경로: `src/server.c`, `src/response_channel.c`, `include/minitalk.h`
- symbol 또는 함수: `handle_request`, `handle_session_request`, `send_response`, `reset_session`, `handle_bit`
- 확인한 state fields: `g_client_pid`, `g_current_byte`, `g_received_bits`, `g_sequence`, `g_line_started`
- caller → callee: event loop response socket readiness → request receive/validation → `handle_session_request` → READY/BUSY `send_response`
- 핵심 branch 또는 mutation 순서: `recvfrom` → exact request size → magic/kind/client PID/source path/same-UID socket/process probe → 기존 owner available이면 BUSY → unavailable이면 recovery → free state owner assignment/receive fields init → READY send → 실패 시 just-created owner reset입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: control-plane reservation path가 추가됐지만 기존 시그널 처리 함수의 first-bit owner capture가 아직 병존합니다.
- 삽입한 최소 코드 조각과 선택 이유: `caf2feec4971`, `src/server.c` data handler: scaffold 역할과 달리 explicit acquisition bypass가 아직 남아 있음을 보여 주는 최소 historical evidence입니다.

```c
if (g_client_pid == 0)
	g_client_pid = info->si_pid;
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`f8e8444c5ded`에서 client가 matching READY만 acquisition success로 수락합니다.
### 5. `f8e8444c5ded` — feat(client): READY 응답을 출처와 nonce로 상관 검증

- **중요도:** A
- **태그:** RESPONSE, RISK, INTEGRATION
- **개발 흐름에서의 역할:** client가 nonzero nonce ACQUIRE를 보내고 expected source, PID, fields, deadline이 맞는 READY만 수락합니다.

#### 원문에서 확정된 맥락

`/dev/urandom` nonce와 one absolute `CLOCK_MONOTONIC` deadline을 사용하며 malformed/unrelated datagram은 session을 시작시키지 않습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] nonce generation의 open/read/close와 zero handling
- [x] ACQUIRE client PID와 nonce fields
- [x] server destination endpoint construction/validation
- [x] READY exact size/source/magic/PID/kind/token/status predicate
- [x] invalid candidate 뒤 same absolute deadline 유지
- [x] READY success 뒤에만 data path 진입

#### 비교 기준

`caf2feec4971`의 nonce echo와 expected source-path rule을 client predicate 항목별로 대조했습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: server reservation path는 있었지만 production client가 ACQUIRE를 보내고 READY를 검증하는 lifecycle이 완성되지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: READY kind 하나만 보거나 first datagram을 수락하면 stale/forged response가 session establishment를 거짓 완료할 수 있습니다.
- 변경된 decision과 state mutation 순서: client endpoint를 준비하고 `/dev/urandom`에서 nonzero nonce를 만든 뒤 ACQUIRE를 보내며 exact size/source/magic/server PID/READY/token/status를 모두 검증합니다. client endpoint bind → target server socket validation → nonce open/read/close, zero면 1로 보정 → ACQUIRE send → one absolute monotonic deadline → invalid response discard loop → matching READY 뒤에만 payload send path 진입입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: nonce source/open/read/close, destination validation, send, deadline/receive permanent error 또는 timeout은 session failure입니다. invalid datagram은 state를 바꾸지 않고 같은 deadline을 소비합니다.
- 후속 commit이 강화하거나 교체하는 부분: `e56e8cc87315`이 no-data reservation exclusivity를 검증하고 Thread 6의 `b361ef9745ff`/`1ed2acbaa353`이 forged/flood input을 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] correlated session establishment
- [x] bounded readiness wait
- [x] malformed/unrelated response가 owner progress를 승인하지 않음

**아직 보장하지 않는 것**

- [x] final owner availability
- [x] idle lease
- [x] same-UID authentication

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/response_channel.c`, `include/minitalk.h`
- symbol 또는 함수: `generate_nonce`, `request_session`, `wait_for_response`, `read_response`
- 확인한 state fields: `nonce`, `deadline`, `expected server path`
- caller → callee: `main` → client endpoint setup → `request_session` → `sendto`/`recvfrom` → payload send
- 핵심 branch 또는 mutation 순서: client endpoint bind → target server socket validation → nonce open/read/close, zero면 1로 보정 → ACQUIRE send → one absolute monotonic deadline → invalid response discard loop → matching READY 뒤에만 payload send path 진입입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: production client가 data 전에 correlated ACQUIRE/READY handshake를 수행합니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`e56e8cc87315`에서 data 없는 acquired owner도 exclusive한지 검증합니다.
### 6. `e56e8cc87315` — test(session): 데이터 없는 활성 예약 경쟁 검증

- **중요도:** A
- **태그:** TEST, SESSION
- **개발 흐름에서의 역할:** session helper의 `reserve` mode가 acquisition만 완료하고 data를 보내지 않은 채 live owner를 유지합니다.

#### 원문에서 확정된 맥락

그동안 normal client는 BUSY여야 하고 stdout은 비어 있어야 합니다. reserver exit 뒤 later client는 정상 acquisition과 message completion을 수행합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `reserve` mode의 endpoint setup과 ACQUIRE/READY-only path
- [x] reserver를 live로 유지하는 synchronization
- [x] normal client BUSY assertion
- [x] reservation 중 no-output assertion
- [x] reserver exit 뒤 later client order
- [x] first bit 없이 owner state 유지

#### 비교 기준

`caf2feec4971`의 owner assignment가 request handler 안에서 READY 전에 일어나는지 no-data hold와 연결했습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: 기존 session tests는 owner가 적어도 일부 data signal을 보낸 상태를 만들었기 때문에 소유권 start가 ACQUIRE인지 first bit인지 구분하지 못했습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: server가 READY를 보낸 뒤에도 first bit 전에는 owner가 없다고 처리하는 회귀가 생기면 두 client가 동시에 session을 얻을 수 있습니다.
- 변경된 decision과 state mutation 순서: session sender의 `hold` 경로를 protocol-conformant `reserve` mode로 바꿔 endpoint setup과 ACQUIRE/READY만 완료한 뒤 pause합니다. reserver child ACQUIRE/READY → ready synchronization → data를 보내지 않고 live 유지 → normal client는 BUSY failure → stdout empty 확인 → reserver exit → later client acquisition/message success입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: reservation 동안 output이 생기면 실패입니다. competitor가 성공해도 실패입니다. reserver exit 뒤 later client가 계속 BUSY면 recovery failure입니다.
- 후속 commit이 강화하거나 교체하는 부분: `1e3da4580733`이 반대로 PID는 남지만 endpoint가 사라진 owner를 회수합니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** successful acquisition 자체가 소유권 start이며 payload progress는 조건이 아닙니다.
- **재현하는 failure 또는 boundary:** first bit 전 owner가 없다고 봐 두 client에게 READY를 주는 회귀
- **사용한 test technique:** protocol-conformant helper가 ACQUIRE/READY만 완료하고 live pause
- **분류:** deterministic session integration regression
- **failure 주입 또는 process orchestration 시작 지점:** `tests/session_ownership.sh`가 `session_sender reserve`를 시작하고 ready marker를 기다립니다.
- **production code에 진입하는 최초 호출:** reserve helper의 ACQUIRE datagram이 production server request handler에 진입합니다.
- **핵심 assertion과 관측값:** competitor BUSY, reservation 동안 stdout empty, reserver exit 뒤 later client success를 확인합니다.
- **증명하는 것:** idle live exclusivity<br>reservation 중 no output<br>exit 후 reacquire
- **증명하지 않는 것:** lease policy<br>partial-message recovery<br>zombie recovery
- **후속 변경에서 막아야 할 구체적인 회귀:** owner assignment을 first bit 시점으로 되돌려 idle reservation을 무시하는 회귀를 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] idle live reservation exclusivity
- [x] reservation 중 no output
- [x] owner exit 뒤 reacquire

**아직 보장하지 않는 것**

- [x] lease policy
- [x] partial-message recovery
- [x] zombie recovery

#### 코드 증거 기록

- 파일 경로: `tests/session_sender.c`, `tests/session_ownership.sh`
- symbol 또는 함수: `reserve_session`, `main`
- 확인한 state fields: `reservation nonce`, `bound response endpoint`, `ready marker`
- caller → callee: test shell → reserve helper handshake → production `handle_session_request` → competitor client
- 핵심 branch 또는 mutation 순서: reserver child ACQUIRE/READY → ready synchronization → data를 보내지 않고 live 유지 → normal client는 BUSY failure → stdout empty 확인 → reserver exit → later client acquisition/message success입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: helper가 data bits를 보내는 hold mode 대신 acquisition만 유지하는 reserve mode를 제공합니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`1e3da4580733`에서 PID는 남지만 protocol resource는 사라진 반대 경계를 다룹니다.
### 7. `1e3da4580733` — fix(server): 응답 경로가 사라진 세션 소유자 회수

- **중요도:** S
- **태그:** SESSION, PROCESS_LIFECYCLE, DEBUG
- **개발 흐름에서의 역할:** owner availability를 process presence와 expected same-UID client response socket usability의 결합으로 정의합니다.

#### 원문에서 확정된 맥락

exited-but-unreaped child는 `kill(pid, 0)`에 성공하지만 cleanup으로 socket path를 제거해 ACK를 받을 수 없습니다. PID 또는 endpoint가 unusable하면 recovery할 수 있습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] owner PID/process probing helper
- [x] owner PID에서 expected client path derivation
- [x] same-UID Unix socket validation
- [x] process+endpoint 결합 contract
- [x] competing ACQUIRE의 unavailable-owner recovery
- [x] delimiter 성공 뒤 coupled fields reset
- [x] PID alive+endpoint absent가 recovery가 되는 branch

#### 비교 기준

`bc337552961a`의 ESRCH-only branch를 new helper call graph와 직접 비교했습니다.

#### S-level 재구성

- 이 commit 직전의 관련 state와 caller/callee: server는 owner availability를 `kill(owner, 0)`/`ESRCH`로만 판단했습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: zombie child는 PID table entry가 남아 `kill(pid,0)`이 성공하지만 process cleanup으로 response socket path가 이미 사라져 ACK를 받을 수 없습니다. server는 이를 live owner로 보고 계속 BUSY를 반환합니다.
- 변경된 decision과 state mutation 순서: `session_owner_available`을 도입해 PID probe, owner PID 기반 expected client path derivation, same-UID Unix socket 검증을 모두 통과해야 usable owner로 봅니다. competing ACQUIRE → current owner PID probe → expected client path 생성 → socket type/UID usability 확인 → 하나라도 실패하면 recovery delimiter 시도 → 성공 뒤 owner/byte/bit/sequence/line reset → new request owner assignment/READY입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: PID alive + endpoint absent는 BUSY가 아니라 recovery입니다. visible line delimiter write가 실패하면 reset/reassignment를 완료하지 않고 event loop error로 전파합니다. valid process+endpoint이면 BUSY입니다.
- 후속 commit이 강화하거나 교체하는 부분: `a481bfabb7b5`가 `WNOWAIT` zombie window를 deterministic하게 재현합니다. PID reuse와 same-UID malicious socket은 여전히 인증되지 않습니다.

| 추적 항목 | 학습자 기록 | 코드 근거 |
| --- | --- | --- |
| 직전 architecture/state | `kill(owner,0)` success면 owner available | old owner probe in request/bit paths |
| 해결하려던 핵심 문제 | zombie PID와 vanished response endpoint 분리 | new `session_owner_available` |
| 실패 가능한 interleaving 또는 partial failure | child cleanup이 path를 unlink한 뒤 parent가 reap 전 ACQUIRE | PID alive + path absent |
| 선택한 decision | process presence와 expected same-UID socket을 모두 요구 | PID probe/path derive/socket validate |
| 소유권/lifecycle/상태 전이 | unavailable→delimiter commit→coupled reset→new owner | `reset_session`, owner fields |
| 후속 fix 또는 regression evidence | WNOWAIT zombie regression | `a481bfabb7b5` |

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | `kill(owner, 0)` success면 owner가 ACK를 받을 수 있습니다. | 직전 availability branch는 ESRCH만 dead로 분류했습니다. |
| 실제 failure 또는 위험 | zombie child는 PID entry가 남지만 response socket은 이미 사라집니다. | client cleanup과 parent reap은 별개 lifecycle event입니다. |
| root cause | kernel identity presence와 protocol availability를 동일시했습니다. | 새 helper가 PID와 expected endpoint를 별도 검사합니다. |
| 수정 불변 조건/decision | process와 expected response endpoint가 모두 usable해야 owner가 available합니다. | `session_owner_available` conjunction과 request recovery branch |
| regression | `a481bfabb7b5`가 WNOWAIT로 zombie window를 유지합니다. | test가 `kill -0` success와 path absence를 동시에 assertion합니다. |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: competing ACQUIRE → current owner PID probe → expected client path 생성 → socket type/UID usability 확인 → 하나라도 실패하면 recovery delimiter 시도 → 성공 뒤 owner/byte/bit/sequence/line reset → new request owner assignment/READY입니다.
- 수정된 invariant를 고정하는 후속 regression test: `a481bfabb7b5`

#### 보장 범위

**이 commit이 보장하는 것**

- [x] application-level liveness = process + response resource
- [x] zombie owner가 server를 영구 점유하지 않음
- [x] new session이 prior partial state를 상속하지 않음

**아직 보장하지 않는 것**

- [x] PID reuse authentication
- [x] same-UID adversary blocking
- [x] lease-based recovery

#### 코드 증거 기록

- 파일 경로: `src/server.c`, `src/response_channel.c`
- symbol 또는 함수: `session_owner_available`, `mt_response_path`, `valid_client_socket`, `handle_session_request`, `reset_session`
- 확인한 state fields: `client_pid`, `current_byte`, `received_bits`, `sequence`, `line_started`
- caller → callee: `handle_session_request` → `session_owner_available` → path derivation/socket validation → `reset_session`/BUSY
- 핵심 branch 또는 mutation 순서: competing ACQUIRE → current owner PID probe → expected client path 생성 → socket type/UID usability 확인 → 하나라도 실패하면 recovery delimiter 시도 → 성공 뒤 owner/byte/bit/sequence/line reset → new request owner assignment/READY입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: owner liveness predicate가 kernel PID presence 하나에서 protocol response resource usability와의 conjunction으로 바뀝니다.
- 삽입한 최소 코드 조각과 선택 이유: `1e3da4580733`, `src/server.c: session_owner_available`의 핵심 call graph를 압축한 조각입니다.

```c
if (kill(client_pid, 0) == -1 && errno == ESRCH)
	return (0);
if (mt_response_path(MT_ROLE_CLIENT, client_pid, path, sizeof(path)) == -1)
	return (0);
return (valid_client_socket(path));
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`a481bfabb7b5`가 exited-but-unreaped state를 실제로 고정합니다.
### 8. `a481bfabb7b5` — test(session): 종료 송신자 회수 전 새 세션 복구 검증

- **중요도:** A
- **태그:** TEST, SESSION, PROCESS_LIFECYCLE
- **개발 흐름에서의 역할:** partial-session child를 exit시킨 뒤 `waitid(..., WNOWAIT)`로 unreaped 상태를 유지해 zombie PID와 vanished endpoint를 동시에 관측합니다.

#### 원문에서 확정된 맥락

test는 `kill(pid, 0)` success와 socket-path absence를 확인한 뒤 child reap 전에 new client를 실행합니다. server는 abandoned output을 delimit하고 new message를 완료해야 합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] partial-session child acquisition/data/exit path
- [x] `waitid`의 WEXITED|WNOHANG|WNOWAIT flags
- [x] `kill(child,0)` success assertion
- [x] child response path absence assertion
- [x] reap 전에 new client 실행
- [x] abandoned newline + new message expected output
- [x] final child reap/server cleanup

#### 비교 기준

`1e3da4580733`의 process probe/path validation branch를 test의 두 관측값과 일대일로 연결했습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: endpoint-aware availability fix는 구현됐지만 zombie 상태와 client socket cleanup 순서를 직접 고정하는 root-cause-specific test가 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 일반 `wait`는 child를 즉시 reap해 PID-only bug를 숨기므로 exited-but-unreaped window를 유지해야 합니다.
- 변경된 decision과 state mutation 순서: `unreaped_exec` helper가 partial-session child를 fork하고 `waitid(P_PID, ..., WEXITED|WNOHANG|WNOWAIT)`로 exit를 확인하되 reap하지 않습니다. partial child ACQUIRE/data/exit → WNOWAIT로 zombie 확인 → `kill(child,0)` success assertion → client path `ENOENT` assertion → child reap 전 replacement client → abandoned newline + new message 확인 → final reap/server cleanup입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: PID probe가 성공한다는 사실만으로 BUSY가 되면 replacement가 실패해 test가 잡습니다. endpoint-aware recovery가 성공하면 visible partial line을 newline으로 닫고 new frame을 별도 줄에 출력합니다.
- 후속 commit이 강화하거나 교체하는 부분: 이 commit이 Thread final regression evidence입니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** owner는 PID entry뿐 아니라 usable response endpoint를 유지해야 합니다.
- **재현하는 failure 또는 boundary:** exited-but-unreaped owner가 server를 BUSY로 고정하는 상황
- **사용한 test technique:** `fork` + partial sender + `waitid(..., WNOWAIT)` + real endpoint observation
- **분류:** root-cause-specific deterministic regression
- **failure 주입 또는 process orchestration 시작 지점:** `tests/session_ownership.sh`가 `unreaped_exec`를 실행해 partial child를 만듭니다.
- **production code에 진입하는 최초 호출:** replacement ACQUIRE가 production `handle_session_request`와 `session_owner_available`에 진입합니다.
- **핵심 assertion과 관측값:** `kill(child,0)` success, endpoint path absence, child reap 전 replacement success, abandoned/new output separation을 확인합니다.
- **증명하는 것:** zombie PID/vanished endpoint separation<br>reap-before-recovery 불필요<br>output session separation
- **증명하지 않는 것:** PID reuse authentication<br>same-UID adversary resistance<br>lease behavior
- **후속 변경에서 막아야 할 구체적인 회귀:** ESRCH-only owner availability로 되돌아가 zombie owner가 BUSY를 영구 유지하는 회귀를 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] zombie PID와 vanished endpoint의 분리 검증
- [x] reap 전에 owner recovery 가능
- [x] output session separation

**아직 보장하지 않는 것**

- [x] PID reuse authentication
- [x] same-UID adversary resistance
- [x] lease behavior

#### 코드 증거 기록

- 파일 경로: `tests/unreaped_exec.c`, `tests/session_ownership.sh`, `tests/session_sender.c`, `Makefile`
- symbol 또는 함수: `main`, `waitid`, `waitpid`
- 확인한 state fields: `zombie child PID`, `client socket path`, `partial output`
- caller → callee: test shell → `unreaped_exec` → session sender → production server availability/recovery → replacement client
- 핵심 branch 또는 mutation 순서: partial child ACQUIRE/data/exit → WNOWAIT로 zombie 확인 → `kill(child,0)` success assertion → client path `ENOENT` assertion → child reap 전 replacement client → abandoned newline + new message 확인 → final reap/server cleanup입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: WNOWAIT helper와 zombie-specific scenario가 추가돼 PID-only assumption의 정확한 failure window를 재현합니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

이 commit이 Thread final regression evidence입니다.

## 6. 불변 조건 ledger

### 원자료에서 확인된 핵심 불변 조건

- explicit acquisition을 완료한 하나의 client만 bit, byte, sequence, output state를 변경합니다.
- live owner가 있으면 competing acquisition은 `BUSY`이고 competing data signal은 state를 바꾸지 않습니다.
- owner reassignment 전 already-visible partial line을 성공적으로 delimit해야 합니다.
- owner availability는 retained PID entry뿐 아니라 usable same-UID response socket을 요구합니다.
- `caf2feec4971` 단독 tree에는 implicit first-bit capture가 남았고, 이 bypass는 `aeb1b00867f4`에서 제거됩니다.

### 시간에 따른 변화 기록

| Commit | 원자료에서 확인된 변화 | 실제 state/condition | code evidence | 상태: 도입·강화·부족·복구·검증 |
| --- | --- | --- | --- | --- |
| `10a7211969bf` | active sender PID를 기록하고 NUL terminator까지 다른 sender의 data signal을 거부합니다. | `g_client_pid`가 0이면 first sender가 owner가 되고 NUL frame까지 다른 PID가 byte state를 바꾸지 못합니다. | `src/server.c: handle_bit` owner branches | 도입·부족 |
| `bc337552961a` | recorded owner가 사라지면 partial byte, bit count, owner, line state를 함께 reset하고 new sender가 진행할 수 있게 합니다. | owner PID absence가 session recovery trigger가 되고 `g_line_started`가 delimiter 필요 여부를 결정합니다. | `src/server.c: reset_session`, owner probe branches | 복구·부족 |
| `bdccf91f5a44` | dedicated sender로 one-bit abandon, complete-byte-plus-partial abandon, live competition, owner exit recovery를 재현합니다. | real child lifecycle로 first-bit ownership, abandon reset, live competition 결과를 관측합니다. | `tests/session_sender.c`, `tests/session_ownership.sh` | 검증 |
| `caf2feec4971` | exact `ACQUIRE` datagram을 검증한 뒤 data 전에 owner를 예약하고 `READY` 또는 `BUSY`를 반환합니다. | validated ACQUIRE가 reservation path를 만들지만 data handler의 implicit capture branch가 함께 남아 있습니다. | `src/server.c: handle_session_request`와 `handle_bit` 직접 비교 | 도입·불충분 |
| `f8e8444c5ded` | client가 nonzero nonce ACQUIRE를 보내고 expected source, PID, fields, deadline이 맞는 READY만 수락합니다. | outstanding acquisition은 nonzero nonce와 expected server endpoint/deadline으로 식별됩니다. | `src/client.c: request_session`, response predicate | 강화 |
| `e56e8cc87315` | session helper의 `reserve` mode가 acquisition만 완료하고 data를 보내지 않은 채 live owner를 유지합니다. | READY를 받은 live reserver의 `g_client_pid`는 bit count가 0이어도 exclusive합니다. | `tests/session_sender.c: reserve`, server request owner assignment | 검증 |
| `1e3da4580733` | owner availability를 process presence와 expected same-UID client response socket usability의 결합으로 정의합니다. | owner availability는 PID와 expected same-UID response socket 둘 다 usable인 conjunction입니다. | `src/server.c: session_owner_available`, `handle_session_request` | 복구·강화 |
| `a481bfabb7b5` | partial-session child를 exit시킨 뒤 `waitid(..., WNOWAIT)`로 unreaped 상태를 유지해 zombie PID와 vanished endpoint를 동시에 관측합니다. | test가 PID table retention과 response endpoint cleanup을 동시에 관측해 resource-aware availability를 검증합니다. | `tests/unreaped_exec.c`, session shell assertions | 검증 |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 또는 상태 | 실제 failure/위험 | Fix 또는 전환 commit | 수정된 decision/불변 조건 | Test 또는 후속 검증 | 학습자 code evidence |
| --- | --- | --- | --- | --- | --- |
| shared accumulator에 sender identity 없음 | 서로 다른 client의 bits가 한 byte에 섞임 | `10a7211969bf` | first sender를 active owner로 기록하고 competitor 거부 | `bdccf91f5a44` | `g_client_pid` assignment/mismatch early return |
| first data signal이 ownership 시작점 | data 전 reservation과 correlated BUSY/READY 불가 | `caf2feec4971`, 완전한 bypass 제거는 `aeb1b00867f4` | validated ACQUIRE/READY가 owner authority, unaquired data는 무시 | `e56e8cc87315` | request validation/owner assignment + later first-bit branch deletion |
| `kill(owner, 0)` success면 usable owner | zombie PID는 남고 response endpoint는 사라짐 | `1e3da4580733` | process identity와 response endpoint를 함께 availability로 검사 | `a481bfabb7b5` | `session_owner_available`; WNOWAIT/ENOENT assertions |

전용 test commit이 없는 연결에는 존재하지 않는 test를 만들어 적지 않았습니다.

## 8. 소유권 / state / responsibility 변화

| 단계 | state 또는 responsibility owner | transition | 당시 한계 또는 다음 변화 | 실제 symbol/field |
| --- | --- | --- | --- | --- |
| `10a7211969bf` | first accepted signal sender PID | NUL terminator까지 owner | implicit ownership | `g_client_pid`, handler first-bit branch |
| `bc337552961a` | PID existence | ESRCH owner면 partial state recovery | PID-only liveness | `kill(owner,0)`, `reset_session` |
| `caf2feec4971` | validated ACQUIRE sender | owner assign 후 nonce-correlated READY/BUSY | data handler bypass가 아직 잔존 | `handle_session_request`, `g_client_pid` |
| `aeb1b00867f4` 이후 | ACQUIRE path만 owner authority | unaquired data는 ignore | explicit data-before-signal ownership | first-bit assignment 삭제 |
| `1e3da4580733` | process + usable endpoint | 둘 중 하나가 없으면 recovery | resource-aware liveness | `session_owner_available` |

## 9. 개발 흐름의 최종 상태

원자료에서 확인된 최종 조건:

- 소유권은 first bit가 아니라 validated `ACQUIRE`/`READY` transition에서 시작합니다.
- live and usable owner는 payload progress가 없어도 exclusive합니다.
- unavailable owner의 partial byte는 폐기되고 visible output은 delimit된 뒤 reassignment됩니다.
- PID가 남아도 expected client response socket이 사라졌으면 protocol owner로 유지되지 않습니다.

학습자 기록:

- 최종 state fields와 owner: server event loop가 `client_pid`, `current_byte`, `received_bits`, `sequence`, `line_started`를 하나의 session state로 소유합니다. client endpoint path는 해당 client process가 bind 수명 동안 소유합니다.
- 정상 transition 순서: exact ACQUIRE receive/validation → prior owner availability 확인 및 필요 시 delimiter commit/reset → new owner fields 초기화 → nonce-correlated READY → owner data events만 mutation → NUL frame에서 release입니다.
- 실패 시 중단·reset·cleanup 순서: invalid ACQUIRE는 무시하고 live owner면 BUSY입니다. READY send failure는 새 reservation을 rollback합니다. unavailable owner delimiter 실패는 reassignment를 막고 server error/cleanup으로 전파합니다.
- 최종 상태가 보장하지 않는 것: lease, idle progress timeout, PID reuse 방지, same-UID peer authentication, session handoff의 distributed transaction은 제공하지 않습니다.
- 이 개발 흐름을 한 문단으로 설명한 최종 서술: 이 개발 흐름은 shared accumulator에 first-sender PID를 붙여 interleaving을 막는 단계에서 시작합니다. dead PID recovery와 line delimiter를 추가한 뒤 ACQUIRE/READY control path로 reservation을 옮기지만, 해당 도입 SHA에는 implicit first-bit bypass가 잠시 남습니다. 이후 datagram-only 소유권으로 수렴하고, 마지막에는 PID 존재와 client response socket usability를 함께 검사해 zombie owner까지 회수합니다.

## 10. 최종 architecture 또는 실행 순서 정리

- [x] server datagram receive와 exact ACQUIRE validation
- [x] prior owner availability check와 필요시 recovery delimiter
- [x] owner assignment 및 nonce-correlated READY/BUSY
- [x] READY send failure 시 new reservation rollback
- [x] owner PID의 data signal만 상태 전이에 적용
- [x] NUL frame 완료 시 session release
- [x] process 또는 endpoint 소실 뒤 competing acquisition recovery

```text
server pselect: response socket readable
    -> recvfrom exact ACQUIRE
    -> size/magic/kind/claimed PID/source path/same-UID validation
    -> existing owner: process + expected endpoint availability check
    -> available: BUSY response
    -> unavailable: visible-line delimiter commit -> coupled reset
    -> assign new owner fields -> READY(nonce)
    -> READY send failure: rollback
    -> self-pipe data event from owner only
    -> bit/byte/sequence/output/ACK
    -> NUL frame: delimiter + session release

```

- 실제 함수·파일을 반영한 완성 흐름: `src/server.c`의 request handler, `session_owner_available`, session reset과 bit processor가 control/data lifecycle을 연결합니다.
- asynchronous boundary: ACQUIRE Unix datagram과 data signal은 서로 다른 입력 채널이며 server event loop가 두 채널의 state order를 직렬화합니다.
- externally visible commit point: reservation은 validated ACQUIRE 뒤 owner state와 READY send가 성공한 시점이며, reassignment는 recovery delimiter가 성공한 뒤입니다.
- cleanup owner: server session reset이 coupled receive state를, client/server endpoint cleanup이 각 bound path와 descriptor를 소유합니다.

## 11. 학습 완료 자가 점검

- [x] commit map의 8개 SHA를 source 순서대로 모두 설명할 수 있습니다.
- [x] 각 code excerpt에 SHA, path, symbol, 선택 이유가 기록돼 있습니다.
- [x] 최종 HEAD 코드를 historical SHA의 증거로 사용한 곳이 없습니다.
- [x] 정상 경로와 실패 처리를 state mutation 순서로 설명할 수 있습니다.
- [x] source 확정 불변 조건과 직접 확인한 code evidence를 구분했습니다.
- [x] test commit의 불변 조건, failure, technique, production path, proves/not-proves를 기록했습니다.
- [x] Thread final state를 함수와 state field 수준으로 설명할 수 있습니다.
- [ ] 해당 SHA의 test를 로컬에서 직접 실행했습니다. — 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

### 이 Thread와 직접 연결된 Major Engineering Difficulties

- process exit, PID table retention, socket cleanup이 서로 다른 시점에 일어납니다.
- partial byte와 already-written bytes를 함께 복구하지 않으면 sessions가 섞입니다.
- no-lease design에서는 live but idle owner도 계속 exclusive하므로 availability와 progress를 구분해야 합니다.

---

# 비동기 신호 처리에서 즉시 중단하는 셀프 파이프 이벤트 반복문까지

> 완성형 해설서가 아닙니다. 아래 확정 사항을 기준으로 각 commit SHA의 실제 코드와 diff를 읽고 기록란을 채웁니다.

## 1. 개발 흐름 목표

시그널 처리 함수가 protocol state를 직접 변경하던 구조가 fixed event record와 nonblocking self-pipe를 거쳐, `pselect` event loop 하나가 session·bit·output·ACK state를 전담하는 구조로 바뀌는 과정을 복원합니다.

### Significance

핵심은 pipe 사용 자체가 아니라 authoritative state mutation을 normal execution context 하나로 모은 것입니다. handler는 `errno`를 보존하고 ordered fact만 전달합니다. event 하나의 손실은 bit 하나의 손실이므로 recoverable queue hiccup이 아니라 fail-stop 조건입니다. termination과 inherited mask도 같은 event architecture에 통합됩니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- self-pipe 전 handler가 직접 수행하던 authorization, bit assembly, output, ACK 작업은 무엇입니까?
- event record의 최소 fields와 `PIPE_BUF` compile-time condition은 무엇입니까?
- handler가 보존하는 `errno`와 허용 operation은 실제 코드에서 어떻게 제한됩니까?
- failed 또는 partial pipe write를 왜 fatal overflow로 처리하는가?
- termination signal을 같은 event path로 보내면서 종료 상태와 cleanup을 어떻게 보존하는가?
- `exec`로 상속된 blocked mask를 server가 어떤 initialization order로 정상화하는가?

## 3. 완료 기준

- [x] handler 전후 responsibility를 symbol 단위로 비교했습니다.
- [x] event write/read/validation/bit transition/output/ACK 순서를 trace로 작성했습니다.
- [x] event loss detection부터 server exit와 endpoint cleanup까지 연결했습니다.
- [x] data event와 termination event의 공통 input과 다른 result를 설명했습니다.
- [x] masked-exec test를 disposition과 deliverability production code에 연결했습니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `a7d994b0a4b6` | refactor(server): 비트 상태 전이 로직 추출 | B | REFACTOR, SELF_PIPE | bit processing을 explicit `(sender, signal)` event를 받는 function으로 추출하고 event size가 `PIPE_BUF` 안인지 compile-time으로 확인합니다. |
| 2 | `22363f83ff25` | refactor(server): signal 처리를 self-pipe event loop로 제한 | S | ARCH, SELF_PIPE, RISK | handler를 `errno` preservation과 fixed event self-pipe write로 제한하고 `pselect` loop가 모든 protocol work를 수행하게 합니다. |
| 3 | `4c535ac8657e` | test(server): self-pipe 이벤트 손실 시 fail-stop 검증 | A | TEST, SELF_PIPE, RISK | handler의 first self-pipe event write에 `EAGAIN`을 주입하고 server fail-stop을 검증합니다. |
| 4 | `e304c63bee3e` | fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리 | A | SELF_PIPE, PROCESS_LIFECYCLE | `SIGHUP`, `SIGINT`, `SIGTERM`을 data signal과 같은 self-pipe event로 전달하고 normal cleanup path에서 종료합니다. |
| 5 | `686b0d2a14e3` | fix(server): 상속된 이벤트 시그널 마스크 해제 | A | PROCESS_LIFECYCLE, RISK | handler 설치 후 two data signals와 three supported termination signals를 process mask에서 명시적으로 unblock합니다. |
| 6 | `72424469474c` | test(server): 차단된 시그널 마스크 상속 뒤 메시지 검증 | B | TEST, PROCESS_LIFECYCLE | wrapper가 data와 termination signals를 block한 뒤 server를 `exec`하고 message delivery와 SIGTERM cleanup을 검증합니다. |

확인 원칙:

- 각 항목은 해당 SHA의 tree를 기준으로 읽었습니다.
- 변경 전 상태는 해당 SHA의 parent 또는 지정된 이전 관련 SHA에서 확인했습니다.
- 같은 commit이 다른 Thread에 다시 등장해도 이 개발 흐름의 질문으로 별도 기록했습니다.
- runtime test는 실행하지 않았으며, 실행 결과처럼 표현하지 않았습니다.

## 5. 커밋별 학습 기록

### 1. `a7d994b0a4b6` — refactor(server): 비트 상태 전이 로직 추출

- **중요도:** B
- **태그:** REFACTOR, SELF_PIPE
- **개발 흐름에서의 역할:** bit processing을 explicit `(sender, signal)` event를 받는 function으로 추출하고 event size가 `PIPE_BUF` 안인지 compile-time으로 확인합니다.

#### 원문에서 확정된 맥락

이 intermediate SHA에서는 handler가 extracted function을 여전히 호출합니다. authorization, byte assembly, sequence, output, ACK queueing을 `siginfo_t` 자체와 분리한 preparatory refactor입니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `t_bit_event`의 sender/signal fields
- [x] `sizeof(t_bit_event) <= PIPE_BUF` compile-time assertion
- [x] extracted bit-transition function signature
- [x] session/byte/bit/sequence/line state access
- [x] handler가 event를 만들고 function을 직접 호출하는 중간 상태

#### 비교 기준

parent handler body에서 이동한 authorization/assembly/response logic과 `22363f83ff25`의 caller 이동을 분리해 확인했습니다.

#### B-level 구현 역할 기록

- Thread 전체에서 이 commit이 연결하는 앞/뒤 단계: signal handler body가 `siginfo_t`에서 sender를 읽는 동시에 owner 검사, bit assembly, output, sequence와 ACK queueing까지 직접 수행했습니다. → `22363f83ff25`에서 handler caller가 transition function이 아니라 nonblocking event-pipe write로 바뀌고 normal loop가 `process_bit`을 호출합니다.
- 실제로 추가·수정된 핵심 symbol과 state: `t_bit_event { pid_t sender; int signal; }`을 만들고 `process_bit(const t_bit_event *)` 계열 함수로 transition logic을 옮겼습니다. compile-time assertion으로 record가 `PIPE_BUF` 이하인지 확인했습니다.
- 이 commit만으로 충분하지 않아 후속 commit을 확인해야 하는 부분: handler metadata와 protocol transition이 한 함수에 결합돼 self-pipe consumer로 caller를 옮길 수 없었습니다. 다만 이 commit은 책임 이동을 완료하지 않습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] transition logic을 handler metadata에서 explicit event value로 분리
- [x] atomic self-pipe delivery를 위한 fixed record 조건 준비

**아직 보장하지 않는 것**

- [x] async-safe handler boundary
- [x] normal-loop state 소유권
- [x] event-loss policy

#### 코드 증거 기록

- 파일 경로: `src/server.c`
- symbol 또는 함수: `t_bit_event`, `process_bit`, `handle_signal`
- 확인한 state fields: `sender`, `signal`, `client_pid`, `current_byte`, `received_bits`, `sequence`, `line_started`
- caller → callee: 시그널 처리 함수 → event value construction → `process_bit` (여전히 handler context)
- 핵심 branch 또는 mutation 순서: handler가 sender PID와 signal number를 event value에 복사한 뒤 같은 handler 문맥에서 extracted transition function을 직접 호출합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: handler 내부 logic이 event-value function으로 추출됐지만 실행 문맥은 아직 바뀌지 않았습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`22363f83ff25`에서 handler는 transition function 대신 event pipe write만 수행합니다.
### 2. `22363f83ff25` — refactor(server): signal 처리를 self-pipe event loop로 제한

- **중요도:** S
- **태그:** ARCH, SELF_PIPE, RISK
- **개발 흐름에서의 역할:** handler를 `errno` preservation과 fixed event self-pipe write로 제한하고 `pselect` loop가 모든 protocol work를 수행하게 합니다.

#### 원문에서 확정된 맥락

event loop는 complete records를 읽어 sender authorization, bit/byte state, stdout, sequence ACK를 처리합니다. failed/partial event write는 overflow flag를 세우고 server는 계속 진행하지 않습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] handler entry/exit의 `errno` save/restore
- [x] event에 sender PID와 signal만 복사
- [x] self-pipe descriptors와 nonblocking/CLOEXEC setup
- [x] full event-size write check와 overflow flag
- [x] `pselect`의 response socket/self-pipe fd set
- [x] complete record read와 validation
- [x] authorization→assembly→output→sequence→ACK order
- [x] ordinary loop-owned state로 field type/소유권 이동
- [x] overflow→event-loop error→정리 과정

#### 비교 기준

`a7d994b0a4b6`의 handler → `process_bit` direct call이 handler → event-pipe write, loop → `process_bit`으로 바뀐 diff를 확인했습니다.

#### S-level 재구성

- 이 commit 직전의 관련 state와 caller/callee: `a7d994b0a4b6`에서 event value는 생겼지만 handler가 `process_bit`을 직접 호출해 ordinary state와 stdout/socket work를 async context에서 수행했습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: multi-field session mutation과 output/socket calls를 시그널 처리 함수에서 실행하면 async-signal-safe 범위를 넘고 normal event-loop input과 state ordering도 분산됩니다.
- 변경된 decision과 state mutation 순서: self-pipe read/write descriptors를 만들고 write end를 nonblocking/CLOEXEC로 설정했습니다. handler는 `errno`를 보존한 채 fixed `t_bit_event` 한 건을 pipe에 쓰고, `pselect` loop가 complete record를 읽어 모든 protocol transition을 실행합니다. 시그널 처리 함수: save `errno` → event(sender, signal) 생성 → one `write` → full-size가 아니면 `g_event_overflow=1` → restore `errno`. event loop: response socket/self-pipe readiness → overflow 확인 → complete record read/validation → authorization → bit/byte/output/sequence → datagram ACK → loop입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: handler pipe write의 -1/partial count는 event loss로 간주합니다. loop는 overflow flag나 incomplete/permanent read, output/response error를 만나면 error return하고 main cleanup으로 종료합니다. queue capacity를 늘리거나 lost event를 복원하지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `4c535ac8657e`가 first write `EAGAIN`을 주입해 fail-stop을 검증하고 `e304c63bee3e`가 termination events를 같은 경로에 넣습니다. Thread 4가 output-before-ACK를 보강합니다.

| 추적 항목 | 학습자 기록 | 코드 근거 |
| --- | --- | --- |
| 직전 architecture/state | handler가 event value를 만든 뒤 protocol transition을 직접 호출 | `a7d994b0a4b6` handler |
| 해결하려던 핵심 문제 | async 문맥에서 ordinary state/output/socket work 제거 | new self-pipe producer/consumer |
| 실패 가능한 interleaving 또는 partial failure | nonblocking pipe full/partial write로 ordered bit event 손실 | `g_event_overflow` |
| 선택한 decision | handler는 fixed record enqueue, loop는 모든 mutation/I/O | `handle_signal`, `read_event`, `process_bit` |
| 소유권/lifecycle/상태 전이 | ordinary session fields는 loop stack/state가 소유; handler global은 overflow/fds만 참조 | field type and caller diff |
| 후속 fix 또는 regression evidence | EAGAIN fault injection, termination integration, output commit | `4c535ac8657e`, `e304c63bee3e`, `db2004556d8b` |

#### 보장 범위

**이 commit이 보장하는 것**

- [x] async-signal-safe responsibility boundary
- [x] sequential authoritative 상태 머신
- [x] event-loss fail-stop

**아직 보장하지 않는 것**

- [x] unbounded pipe capacity
- [x] lost-event reconstruction
- [x] output-before-ACK commit 완성

#### 코드 증거 기록

- 파일 경로: `src/server.c`
- symbol 또는 함수: `handle_signal`, `read_event`, `process_bit`, `run_event_loop`, `cleanup_server`
- 확인한 state fields: `g_event_overflow`, `event_pipe`, `client_pid`, `current_byte`, `received_bits`, `sequence`, `line_started`
- caller → callee: kernel signal → `handle_signal` → event pipe; `pselect` loop → `read_event` → `process_bit` → output/`send_response`
- 핵심 branch 또는 mutation 순서: 시그널 처리 함수: save `errno` → event(sender, signal) 생성 → one `write` → full-size가 아니면 `g_event_overflow=1` → restore `errno`. event loop: response socket/self-pipe readiness → overflow 확인 → complete record read/validation → authorization → bit/byte/output/sequence → datagram ACK → loop입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: response-work pipe 중간 architecture를 signal-fact self-pipe로 교체하고 authoritative state를 normal 문맥으로 이동했습니다.
- 삽입한 최소 코드 조각과 선택 이유: `22363f83ff25`, `src/server.c: signal handler`: 최종 handler 책임과 event-loss detection을 동시에 보여 주는 최소 조각입니다.

```c
saved_errno = errno;
event.sender = info->si_pid;
event.signal = signal;
if (write(g_event_pipe[1], &event, sizeof(event)) != sizeof(event))
	g_event_overflow = 1;
errno = saved_errno;
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`4c535ac8657e`에서 first event-write `EAGAIN`을 주입합니다.
### 3. `4c535ac8657e` — test(server): self-pipe 이벤트 손실 시 fail-stop 검증

- **중요도:** A
- **태그:** TEST, SELF_PIPE, RISK
- **개발 흐름에서의 역할:** handler의 first self-pipe event write에 `EAGAIN`을 주입하고 server fail-stop을 검증합니다.

#### 원문에서 확정된 맥락

dropped event는 ordered stream의 한 bit를 잃는 것이므로 subsequent processing을 계속할 수 없습니다. server error exit, endpoint removal, no success ACK, client failure를 확인합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] production event write를 대체하는 compile-time hook
- [x] first call only `EAGAIN` state
- [x] handler overflow flag production branch
- [x] event loop의 flag→error exit
- [x] server endpoint removal assertion
- [x] client no-success-ACK/bounded failure assertion

#### 비교 기준

`22363f83ff25`의 production write macro/seam과 overflow reader를 test env/hook state에 연결했습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: production code는 failed/partial event write를 fatal flag로 표시했지만 실제 handler branch를 deterministic하게 실행하는 test가 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: OS pipe를 실제로 채우는 방식은 timing-dependent하고 first lost event를 정확히 고정하기 어렵습니다.
- 변경된 decision과 state mutation 순서: Make fault build가 production event-write call을 `MT_EVENT_WRITE` hook으로 치환하고 test implementation은 env 설정 시 첫 호출만 `-1/EAGAIN`을 반환합니다. fault server start → client가 첫 data signal 전송 → handler hook first call EAGAIN → production full-size check가 overflow flag set → loop가 flag 관측 후 error return → main diagnostic/cleanup → endpoint removal → client는 success ACK 없이 bounded failure입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: 주입은 first call only라 failure 지점이 deterministic합니다. hook 이후 protocol processing을 계속하거나 ACK를 보내면 test가 client/server status와 output/endpoint assertion에서 실패합니다.
- 후속 commit이 강화하거나 교체하는 부분: `e304c63bee3e`는 expected termination을 error overflow와 구분해 같은 cleanup architecture로 보냅니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** self-pipe에 전달되지 않은 signal event가 있으면 protocol processing을 계속하지 않습니다.
- **재현하는 failure 또는 boundary:** nonblocking pipe write `EAGAIN`으로 one bit event가 사라지는 상황
- **사용한 test technique:** handler event-write hook의 first-call deterministic fault injection
- **분류:** deterministic fault-injection regression
- **failure 주입 또는 process orchestration 시작 지점:** `tests/protocol_regressions.sh`가 fault server에 `MT_TEST_EVENT_EAGAIN=1`을 설정합니다.
- **production code에 진입하는 최초 호출:** client의 first signal이 production handler에 들어가고 handler가 `MT_EVENT_WRITE`를 호출합니다.
- **핵심 assertion과 관측값:** server nonzero/error diagnostic, endpoint removal, client timeout/failure, success ACK 부재를 확인합니다.
- **증명하는 것:** event loss detection<br>no false ACK<br>server fail-stop<br>endpoint cleanup
- **증명하지 않는 것:** OS-level signal completeness<br>overflow recovery<br>burst capacity
- **후속 변경에서 막아야 할 구체적인 회귀:** overflow flag를 무시하고 shifted bit state로 계속 처리하는 회귀를 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] event loss detection regression
- [x] no false ACK
- [x] server fail-stop
- [x] endpoint cleanup

**아직 보장하지 않는 것**

- [x] OS-level signal completeness
- [x] overflow recovery
- [x] burst capacity

#### 코드 증거 기록

- 파일 경로: `src/server.c`, `tests/protocol_regressions.sh`, `Makefile`
- symbol 또는 함수: `MT_EVENT_WRITE`, `mt_test_event_write`, `handle_signal`, `run_event_loop`
- 확인한 state fields: `first-call injection flag`, `g_event_overflow`
- caller → callee: real client signal → production handler → injected write hook → production overflow branch → loop/main cleanup
- 핵심 branch 또는 mutation 순서: fault server start → client가 첫 data signal 전송 → handler hook first call EAGAIN → production full-size check가 overflow flag set → loop가 flag 관측 후 error return → main diagnostic/cleanup → endpoint removal → client는 success ACK 없이 bounded failure입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: fault-only server target와 deterministic event-write seam/scenario가 추가됐으며 production algorithm은 같은 full-size check를 실행합니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`e304c63bee3e`는 expected external termination도 same cleanup architecture로 보냅니다.
### 4. `e304c63bee3e` — fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리

- **중요도:** A
- **태그:** SELF_PIPE, PROCESS_LIFECYCLE
- **개발 흐름에서의 역할:** `SIGHUP`, `SIGINT`, `SIGTERM`을 data signal과 같은 self-pipe event로 전달하고 normal 정리 과정에서 종료합니다.

#### 원문에서 확정된 맥락

event loop가 termination event를 인식해 signal number를 main에 돌려주고 process는 `128 + signal`로 종료합니다. close/unlink bookkeeping은 handler가 아니라 registered cleanup에서 수행합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] termination signals에 same handler registration
- [x] event record의 signal-number representation
- [x] event-loop data/termination branch
- [x] signal number를 main에 반환하는 contract
- [x] `128 + signal` status
- [x] registered cleanup의 close/unlink
- [x] handler에 close/unlink/exit가 없음

#### 비교 기준

`22363f83ff25`의 data-only event branch에 termination classification/return contract와 handler registration이 추가됐습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: data signals는 self-pipe를 사용했지만 termination signal은 별도 direct-exit handler/lifecycle로 처리돼 normal descriptor/path cleanup과 분리될 수 있었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: handler에서 `_exit`하면 registered cleanup을 건너뛰고, close/unlink 같은 bookkeeping을 handler에서 수행하면 async-signal-safe 범위를 벗어납니다.
- 변경된 decision과 state mutation 순서: HUP/INT/TERM에 data와 같은 event handler를 등록하고 event loop가 signal number를 termination event로 분류해 main에 반환하도록 했습니다. termination signal → handler errno save/event pipe write/restore → loop complete record read → termination branch가 signal number return → main이 `128 + signal` status 선택 → registered cleanup이 fd close/path unlink를 수행합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: data signals는 `process_bit`로, supported termination signals는 protocol mutation 없이 shutdown status로 갈라집니다. enqueue failure면 ordinary event loss error path입니다.
- 후속 commit이 강화하거나 교체하는 부분: `686b0d2a14e3`이 inherited blocked mask에서도 이 signals가 deliverable하도록 합니다. `72424469474c`가 SIGTERM status 143과 endpoint cleanup을 검증합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | termination handler에서 직접 종료해도 owned endpoint와 descriptors가 정리됩니다. | 직전 termination lifecycle은 data event loop와 분리됐습니다. |
| 실제 failure 또는 위험 | async 문맥에서 cleanup을 생략하거나 unsafe filesystem/socket bookkeeping을 수행할 수 있습니다. | bound Unix path와 pipe/socket descriptors는 normal teardown state가 필요합니다. |
| root cause | external shutdown을 normal event-loop 소유권과 다른 path로 처리했습니다. | 새 diff가 termination을 같은 event record/loop return으로 옮깁니다. |
| 수정 불변 조건/decision | termination도 self-pipe event로 capture하고 main/atexit path에서 `128 + signal`로 종료합니다. | handler registration, loop termination branch, main status calculation |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: termination signal → handler errno save/event pipe write/restore → loop complete record read → termination branch가 signal number return → main이 `128 + signal` status 선택 → registered cleanup이 fd close/path unlink를 수행합니다.
- 수정된 invariant를 고정하는 후속 regression test: `72424469474c`가 masked-exec 상황에서 SIGTERM status/cleanup까지 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] normal-context asynchronous shutdown
- [x] signal-derived status
- [x] endpoint 정리 과정 재사용

**아직 보장하지 않는 것**

- [x] uncatchable signals
- [x] cleanup success의 transaction 보장

#### 코드 증거 기록

- 파일 경로: `src/server.c`
- symbol 또는 함수: `install_signal_handlers`, `handle_signal`, `run_event_loop`, `main`, `cleanup_server`
- 확인한 state fields: `termination signal number`, `event record`
- caller → callee: termination handler → self-pipe → event loop → main return/status → atexit cleanup
- 핵심 branch 또는 mutation 순서: termination signal → handler errno save/event pipe write/restore → loop complete record read → termination branch가 signal number return → main이 `128 + signal` status 선택 → registered cleanup이 fd close/path unlink를 수행합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: 별도 direct termination 경로가 self-pipe event와 normal cleanup/status path에 통합됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`686b0d2a14e3`에서 inherited mask가 event delivery를 막지 않도록 unblock합니다.
### 5. `686b0d2a14e3` — fix(server): 상속된 이벤트 시그널 마스크 해제

- **중요도:** A
- **태그:** PROCESS_LIFECYCLE, RISK
- **개발 흐름에서의 역할:** handler 설치 후 two data signals와 three supported termination signals를 process mask에서 명시적으로 unblock합니다.

#### 원문에서 확정된 맥락

signal disposition만 설치해도 blocked mask는 `exec` 뒤 남습니다. server는 필요한 signal delivery conditions를 스스로 세우고 unrelated inherited settings는 유지합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] required five-signal set construction
- [x] `sigprocmask(SIG_UNBLOCK)` 위치
- [x] unblock failure→startup error/cleanup
- [x] unrelated masks 보존
- [x] PID publication이 normalization 뒤

#### 비교 기준

`e304c63bee3e` startup sequence와 비교해 handler disposition 다음, readiness publication 전 deliverability normalization이 추가됐습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: server는 handlers를 설치했지만 launcher가 blocked mask를 설정한 뒤 `exec`하면 그 mask를 상속했습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: blocked signal에는 handler가 있어도 delivery되지 않으므로 server가 PID를 출력한 뒤 data/termination events를 영원히 처리하지 않을 수 있습니다.
- 변경된 decision과 state mutation 순서: required two data signals와 HUP/INT/TERM만 담은 set을 만들고 `sigprocmask(SIG_UNBLOCK, ...)`를 handler 설치 뒤 PID publication 전에 호출합니다. endpoint/self-pipe setup → `sigaction` handlers 설치 → required signal set 구성 → `SIG_UNBLOCK` → 성공 뒤 PID publication/event loop입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: unblock가 실패하면 startup error로 cleanup합니다. `SIG_UNBLOCK`은 listed signals만 바꾸므로 unrelated inherited blocked settings는 유지됩니다.
- 후속 commit이 강화하거나 교체하는 부분: `72424469474c`가 exact set을 먼저 block한 wrapper에서 message와 SIGTERM cleanup을 검증합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | handler installation이면 signal delivery가 보장됩니다. | 직전 init은 `sigaction`만 수행했습니다. |
| 실제 failure 또는 위험 | `exec` inherited mask 때문에 PID만 publish하고 events를 받지 못합니다. | POSIX process signal mask는 exec 뒤 유지됩니다. |
| root cause | disposition만 설정하고 deliverability를 초기화하지 않았습니다. | 새 helper가 `SIG_UNBLOCK`을 명시적으로 호출합니다. |
| 수정 불변 조건/decision | required data/termination signals를 PID publication 전에 explicit unblock합니다. | `src/server.c: unblock_event_signals`와 main call order |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: endpoint/self-pipe setup → `sigaction` handlers 설치 → required signal set 구성 → `SIG_UNBLOCK` → 성공 뒤 PID publication/event loop입니다.
- 수정된 invariant를 고정하는 후속 regression test: `72424469474c`의 masked-exec regression입니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] launcher mask와 무관한 required event delivery

**아직 보장하지 않는 것**

- [x] all inherited signal policy reset
- [x] unhandled signal behavior

#### 코드 증거 기록

- 파일 경로: `src/server.c`
- symbol 또는 함수: `unblock_event_signals`, `main`
- 확인한 state fields: `sigset_t event_signals`
- caller → callee: server initialization → handler install → `unblock_event_signals` → PID output/event loop
- 핵심 branch 또는 mutation 순서: endpoint/self-pipe setup → `sigaction` handlers 설치 → required signal set 구성 → `SIG_UNBLOCK` → 성공 뒤 PID publication/event loop입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: required signal disposition뿐 아니라 deliverability를 server가 자체 초기화합니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`72424469474c`에서 masked-exec server의 message와 SIGTERM을 검증합니다.
### 6. `72424469474c` — test(server): 차단된 시그널 마스크 상속 뒤 메시지 검증

- **중요도:** B
- **태그:** TEST, PROCESS_LIFECYCLE
- **개발 흐름에서의 역할:** wrapper가 data와 termination signals를 block한 뒤 server를 `exec`하고 message delivery와 SIGTERM cleanup을 검증합니다.

#### 원문에서 확정된 맥락

PID publication만으로 success로 보지 않습니다. bit events, ACKs, NUL frame, status 143, endpoint removal, no unexpected diagnostics를 확인합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] wrapper의 exact blocked signal set와 `exec`
- [x] PID publication 뒤 real client start
- [x] message+newline expected output
- [x] real SIGTERM과 status 143
- [x] server endpoint removal
- [x] stderr/diagnostic comparison

#### 비교 기준

`686b0d2a14e3`의 unblock set과 wrapper block set을 항목별로 맞추고 data/termination event branches를 모두 연결했습니다.

#### B-level 구현 역할 기록

- Thread 전체에서 이 commit이 연결하는 앞/뒤 단계: required signal unblock fix는 구현됐지만 실제 `exec` inheritance와 data/termination 양쪽을 함께 검증하는 test가 없었습니다. → 이 commit이 Thread final lifecycle regression입니다.
- 실제로 추가·수정된 핵심 symbol과 state: `masked_exec` wrapper가 data와 HUP/INT/TERM을 block한 채 real server를 `exec`하고 shell test가 real client message와 real SIGTERM을 보냅니다.
- 이 commit만으로 충분하지 않아 후속 commit을 확인해야 하는 부분: 단순 startup/PID assertion은 blocked mask가 실제 event processing을 막는 failure를 놓칩니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** server는 launcher mask와 무관하게 required event signals를 받을 수 있어야 합니다.
- **재현하는 failure 또는 boundary:** blocked mask를 상속해 PID는 출력하지만 data/termination을 처리하지 못하는 server
- **사용한 test technique:** masked-exec wrapper + real server/client + real SIGTERM
- **분류:** process-lifecycle integration regression
- **failure 주입 또는 process orchestration 시작 지점:** `tests/inherited_mask.sh`가 `masked_exec` wrapper로 server를 시작합니다.
- **production code에 진입하는 최초 호출:** wrapper가 `exec`한 server의 `unblock_event_signals` initialization입니다.
- **핵심 assertion과 관측값:** message+newline, client completion, SIGTERM status 143, endpoint removal, clean stderr를 확인합니다.
- **증명하는 것:** data unblock<br>termination unblock<br>status 143<br>endpoint cleanup
- **증명하지 않는 것:** all signal policies<br>event loss behavior<br>other launch environments
- **후속 변경에서 막아야 할 구체적인 회귀:** handler만 설치하고 inherited mask를 방치하는 회귀를 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] data signals unblock에 대한 process evidence
- [x] termination signal unblock
- [x] status 143
- [x] endpoint cleanup

**아직 보장하지 않는 것**

- [x] all signal policies
- [x] event loss behavior
- [x] 다른 launcher 환경 전체

#### 코드 증거 기록

- 파일 경로: `tests/masked_exec.c`, `tests/inherited_mask.sh`, `Makefile`
- symbol 또는 함수: `main`, `sigprocmask`, `execv`
- 확인한 state fields: `inherited blocked set`, `server PID/status`
- caller → callee: test shell → `masked_exec` → real server initialization → real client/self-pipe → SIGTERM/cleanup
- 핵심 branch 또는 mutation 순서: wrapper signal block → `exec(server)` → PID publication wait → real client message → exact output/ACK completion → SIGTERM → wait status 143 → endpoint removal/stderr 확인입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: masked-exec wrapper와 end-to-end lifecycle scenario가 test target에 추가됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

이 commit이 Thread final lifecycle regression입니다.

## 6. 불변 조건 ledger

### 원자료에서 확인된 핵심 불변 조건

- handler는 `errno`를 보존하고 async-signal-safe event delivery만 수행합니다.
- session, bit, byte, sequence, stdout, socket work는 event loop가 전담합니다.
- event record는 self-pipe에 atomic하게 기록 가능한 fixed size입니다.
- event loss가 감지되면 silent continuation이 아니라 fail-stop cleanup을 수행합니다.
- data와 termination signals는 inherited mask와 무관하게 deliverable해야 합니다.

### 시간에 따른 변화 기록

| Commit | 원자료에서 확인된 변화 | 실제 state/condition | code evidence | 상태: 도입·강화·부족·복구·검증 |
| --- | --- | --- | --- | --- |
| `a7d994b0a4b6` | bit processing을 explicit `(sender, signal)` event를 받는 function으로 추출하고 event size가 `PIPE_BUF` 안인지 compile-time으로 확인합니다. | protocol transition이 `(sender, signal)` value를 입력으로 받지만 handler가 authoritative caller입니다. | `src/server.c: t_bit_event`, extracted processor, handler caller | 도입·준비 |
| `22363f83ff25` | handler를 `errno` preservation과 fixed event self-pipe write로 제한하고 `pselect` loop가 모든 protocol work를 수행하게 합니다. | handler는 event producer이고 event loop만 session/bit/output/ACK state를 수정합니다. enqueue loss는 fatal flag입니다. | `src/server.c: handle_signal`, event pipe setup, `run_event_loop` | 강화·전환 |
| `4c535ac8657e` | handler의 first self-pipe event write에 `EAGAIN`을 주입하고 server fail-stop을 검증합니다. | first event enqueue 실패가 `g_event_overflow`를 거쳐 server error/cleanup으로 끝나는 production path를 검증합니다. | fault write hook + `tests/protocol_regressions.sh` assertions | 검증 |
| `e304c63bee3e` | `SIGHUP`, `SIGINT`, `SIGTERM`을 data signal과 같은 self-pipe event로 전달하고 normal cleanup path에서 종료합니다. | supported termination signal은 data와 같은 event transport를 쓰되 loop에서 shutdown result로 분기합니다. | `src/server.c` handler registrations, loop return, main status | 강화 |
| `686b0d2a14e3` | handler 설치 후 two data signals와 three supported termination signals를 process mask에서 명시적으로 unblock합니다. | server startup이 required event set의 deliverability를 직접 소유합니다. | `src/server.c: unblock_event_signals`, main call order | 복구 |
| `72424469474c` | wrapper가 data와 termination signals를 block한 뒤 server를 `exec`하고 message delivery와 SIGTERM cleanup을 검증합니다. | blocked mask를 상속한 real process가 data와 termination 모두 처리하고 cleanup하는지 검증합니다. | `tests/masked_exec.c`, `tests/inherited_mask.sh` | 검증 |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 또는 상태 | 실제 failure/위험 | Fix 또는 전환 commit | 수정된 decision/불변 조건 | Test 또는 후속 검증 | 학습자 code evidence |
| --- | --- | --- | --- | --- | --- |
| handler가 state/output/ACK를 직접 처리 | async-safe 범위를 넘고 multi-field state가 handler context에 결합 | `a7d994b0a4b6 → 22363f83ff25` | event value 추출 후 handler는 self-pipe write만 수행 | `4c535ac8657e` | handler full-record write/overflow flag/loop error |
| termination handler에서 직접 cleanup | filesystem/socket bookkeeping을 async context에서 수행하거나 생략할 위험 | `e304c63bee3e` | termination도 event loop와 normal cleanup 사용 | `72424469474c` | termination event branch, main `128+signal`, endpoint assertion |
| launcher가 event signals를 block한 채 exec | PID는 publish하지만 data/shutdown events를 받지 못함 | `686b0d2a14e3` | required signals explicit unblock | `72424469474c` | `unblock_event_signals`; masked-exec message/SIGTERM |

전용 test commit이 없는 연결에는 존재하지 않는 test를 만들어 적지 않았습니다.

## 8. 소유권 / state / responsibility 변화

| 단계 | state 또는 responsibility owner | transition | 당시 한계 또는 다음 변화 | 실제 symbol/field |
| --- | --- | --- | --- | --- |
| pre-refactor | 시그널 처리 함수 | authorization, assembly, output, response | complex async shared state | handler globals and direct calls |
| `a7d994b0a4b6` | extracted transition function | event value를 받지만 handler가 호출 | conceptual separation | `t_bit_event`, `process_bit` |
| `22363f83ff25` | `pselect` event loop | 모든 authoritative mutation과 I/O | handler는 event producer | event pipe, ordinary session fields |
| shutdown integration | event loop + main/atexit | termination signal number→status/cleanup | handler cleanup 없음 | loop return, main, `cleanup_server` |

## 9. 개발 흐름의 최종 상태

원자료에서 확인된 최종 조건:

- handler는 sender PID와 signal number를 fixed event로 복사해 nonblocking self-pipe에 씁니다.
- event loop가 complete event를 읽고 authorization, assembly, stdout, ACK를 순서대로 실행합니다.
- event enqueue loss는 server error exit와 cleanup으로 끝납니다.
- SIGHUP/SIGINT/SIGTERM도 event loop를 거쳐 `128 + signal` status로 처리됩니다.
- required signals는 handler 설치 뒤 명시적으로 unblock됩니다.

학습자 기록:

- 최종 state fields와 owner: ordinary `client_pid`, `current_byte`, `received_bits`, `sequence`, `line_started`는 event loop가 소유합니다. handler와 loop가 공유하는 signal-safe state는 event-pipe descriptors와 `volatile sig_atomic_t g_event_overflow`입니다.
- 정상 transition 순서: signal delivery → handler의 errno 보존/fixed record write → pselect wakeup → complete event read → data/termination 분기 → data면 authorization/assembly/output/ACK, termination이면 signal status return입니다.
- 실패 시 중단·reset·cleanup 순서: event enqueue loss 또는 permanent read/transition/output/response error는 loop error→main failure→registered cleanup입니다. supported termination은 error가 아니라 `128+signal` status로 같은 cleanup을 사용합니다.
- 최종 상태가 보장하지 않는 것: pipe capacity 확장, lost event 복원, standard signal multiplicity 자체의 보존, uncatchable signal cleanup은 제공하지 않습니다.
- 이 개발 흐름을 한 문단으로 설명한 최종 서술: 이 개발 흐름은 handler body를 event value processor로 먼저 분리한 뒤, fixed record를 nonblocking self-pipe에 쓰는 producer로 handler 책임을 축소합니다. event loop 하나가 session과 I/O를 직렬화하고 enqueue loss를 fail-stop으로 처리합니다. termination도 같은 event path에 넣고 required signals를 startup에서 unblock해 launcher의 inherited mask와 무관한 lifecycle을 완성합니다.

## 10. 최종 architecture 또는 실행 순서 정리

- [x] `sigaction`과 signal mask initialization
- [x] handler entry에서 `errno` save
- [x] sender/signal을 event record로 copy
- [x] nonblocking pipe write와 overflow flag
- [x] `pselect` wake-up과 complete record read
- [x] data event/termination event branch
- [x] 상태 전이, output, ACK 또는 종료 상태
- [x] main return과 registered cleanup

```text
server initialization
    -> self-pipe/socket create + flags
    -> sigaction(data, HUP/INT/TERM)
    -> required signals SIG_UNBLOCK
    -> publish PID
asynchronous handler
    -> save errno -> event(sender, signal) -> nonblocking full-record write
    -> failure/partial: g_event_overflow = 1 -> restore errno
pselect event loop
    -> read complete event
    -> overflow/error: fail-stop
    -> termination: return signal number
    -> data: authorize -> assemble -> output -> sequence ACK
    -> main status -> registered cleanup

```

- 실제 함수·파일을 반영한 완성 흐름: `src/server.c`의 signal initialization/handler/event pipe/read loop/processor/main cleanup이 전체 path를 구성합니다.
- asynchronous boundary: handler의 fixed record write가 유일한 async→normal context handoff입니다.
- externally visible commit point: data event가 event loop에서 완전히 처리되고 필요한 output 뒤 ACK가 성공한 시점입니다. event enqueue 자체는 protocol success가 아닙니다.
- cleanup owner: main/registered server cleanup이 response socket, event pipe와 bound server path를 정리합니다.

## 11. 학습 완료 자가 점검

- [x] commit map의 6개 SHA를 source 순서대로 모두 설명할 수 있습니다.
- [x] 각 code excerpt에 SHA, path, symbol, 선택 이유가 기록돼 있습니다.
- [x] 최종 HEAD 코드를 historical SHA의 증거로 사용한 곳이 없습니다.
- [x] 정상 경로와 실패 처리를 state mutation 순서로 설명할 수 있습니다.
- [x] source 확정 불변 조건과 직접 확인한 code evidence를 구분했습니다.
- [x] test commit의 불변 조건, failure, technique, production path, proves/not-proves를 기록했습니다.
- [x] Thread final state를 함수와 state field 수준으로 설명할 수 있습니다.
- [ ] 해당 SHA의 test를 로컬에서 직접 실행했습니다. — 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

### 이 Thread와 직접 연결된 Major Engineering Difficulties

- asynchronous signal delivery와 sequential protocol 상태 머신 사이에 lossless bridge가 필요합니다.
- handler에서 complex work를 제거하면서 sender PID와 signal identity를 보존해야 합니다.
- shutdown과 mask normalization을 별도 unsafe path가 아니라 같은 lifecycle에 통합해야 합니다.

---

# 프로토콜 완료 조건에 포함되는 출력 성공

> 완성형 해설서가 아닙니다. 아래 확정 사항을 기준으로 각 commit SHA의 실제 코드와 diff를 읽고 기록란을 채웁니다.

## 1. 개발 흐름 목표

single `write`를 호출하던 helper가 all-or-failure write contract로 바뀌고, payload·terminator·recovery delimiter·PID output이 성공한 뒤에만 protocol ACK를 보낼 수 있게 되는 commit boundary를 복원합니다.

### Significance

server의 in-memory state와 stdout은 분리된 효과가 아닙니다. 상태 전이 뒤 ACK를 보내고 stdout이 실패하면 client는 실제로 보이지 않은 data를 success로 판단합니다. `mt_write_all`이 local completion contract를 만들고 S-level fix가 output-before-ACK order를 정의하며 fault tests가 normal output과 abandoned-session recovery newline을 함께 고정합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- `write`의 EINTR, short count, zero count, terminal error를 `mt_write_all`은 각각 어떻게 처리합니까?
- completed byte 또는 NUL delimiter에서 state mutation, stdout write, ACK send의 실제 순서는 무엇입니까?
- PID publication failure는 endpoint와 process cleanup에 어떻게 전파됩니까?
- `SIGPIPE` ignore로 closed stdout을 `EPIPE`로 관측하는 이유는 무엇입니까?
- output success 뒤 ACK datagram이 유실되는 경우까지 이 불변 조건이 해결하는가?
- dead-owner recovery newline failure가 owner reset과 replacement READY를 왜 막아야 하는가?

## 3. 완료 기준

- [x] `mt_write_all` loop 불변 조건과 pointer/remaining update를 코드로 설명했습니다.
- [x] PID, payload byte, NUL newline, recovery newline의 output-to-ACK ordering을 기록했습니다.
- [x] output failure가 event loop → main → endpoint cleanup → client failure로 전파되는 path를 작성했습니다.
- [x] fault-injection mode를 production branch와 process result에 연결했습니다.
- [x] local commit guarantee와 non-exactly-once limit를 구분했습니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `826dd34c378f` | fix(io): 중단·부분 쓰기를 끝까지 처리 | A | OUTPUT_COMMIT, PRACTICAL, RISK | `mt_write_all`이 EINTR를 retry하고 short write만큼 offset을 전진하며 zero progress를 `EIO`로 처리합니다. |
| 2 | `db2004556d8b` | fix(server): stdout 실패 뒤 ACK 전송 차단 | S | CORE, OUTPUT_COMMIT, RISK | PID, payload, terminator newline, recovery newline을 all-or-failure로 쓰고 failure 시 triggering ACK를 보내지 않습니다. |
| 3 | `9aa80e047514` | test(server): 부분 쓰기와 출력 실패 검증 | A | TEST, OUTPUT_COMMIT, RISK | low-level write를 deterministic test implementation으로 바꿔 EINTR, one-byte short write, zero write, selected payload/newline EPIPE를 주입합니다. |
| 4 | `081a882d7fa3` | test(server): 회수 줄바꿈 출력 실패 검증 | A | TEST, OUTPUT_COMMIT, SESSION | dead owner가 visible partial line을 남긴 상태에서 replacement acquisition이 recovery newline을 쓰는 순간 failure를 주입합니다. |

확인 원칙:

- 각 항목은 해당 SHA의 tree를 기준으로 읽었습니다.
- 변경 전 상태는 해당 SHA의 parent 또는 지정된 이전 관련 SHA에서 확인했습니다.
- 같은 commit이 다른 Thread에 다시 등장해도 이 개발 흐름의 질문으로 별도 기록했습니다.
- runtime test는 실행하지 않았으며, 실행 결과처럼 표현하지 않았습니다.

## 5. 커밋별 학습 기록

### 1. `826dd34c378f` — fix(io): 중단·부분 쓰기를 끝까지 처리

- **중요도:** A
- **태그:** OUTPUT_COMMIT, PRACTICAL, RISK
- **개발 흐름에서의 역할:** `mt_write_all`이 EINTR를 retry하고 short write만큼 offset을 전진하며 zero progress를 `EIO`로 처리합니다.

#### 원문에서 확정된 맥락

string/number helpers가 이 primitive를 사용합니다. success는 모든 requested bytes가 descriptor interface에 전달됐다는 뜻이며 terminal error는 숨기지 않습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `mt_write_all`의 zero-length 성공 경로
- [x] `bytes + offset`과 `size - offset`을 사용하는 progress loop
- [x] positive short write 뒤 offset 증가
- [x] `EINTR`만 retry하는 branch
- [x] `write == 0`에서 `errno = EIO`와 failure
- [x] `mt_putstr_fd`/`mt_putnbr_fd`가 새 primitive를 호출하는 diff

#### 비교 기준

초기 `bf8163cdd7dd`의 single-write helper와 비교하면, 한 syscall 성공 여부가 아니라 requested byte 전체의 누적 완료가 success contract가 됩니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `src/write_utils.c`의 기존 출력 helper는 단일 `write` 호출의 반환값을 완전한 출력으로 취급했습니다. `mt_putstr_fd`와 `mt_putnbr_fd`는 partial progress를 복구할 공통 primitive가 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: POSIX `write`는 signal에 의해 `EINTR`로 중단되거나 요청 길이보다 적은 양만 쓸 수 있습니다. 0을 반환하면 offset이 전진하지 않아 단순 retry loop도 무한 반복할 수 있습니다.
- 변경된 decision과 state mutation 순서: `mt_write_all(int fd, const void *buffer, size_t size)`를 추가하고 `const unsigned char *bytes`, `size_t offset`, `ssize_t written`으로 progress를 명시적으로 추적했습니다. `offset = 0` → `write(fd, bytes + offset, size - offset)` → `EINTR`이면 같은 offset으로 retry → positive short count면 그 수만큼 offset 증가 → offset이 size에 도달하면 success입니다. `size == 0`은 loop를 건너뛰어 성공합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: `write == -1 && errno == EINTR`만 retry합니다. 다른 `-1`은 그대로 실패하고, `write == 0`은 `errno = EIO`로 바꾼 뒤 실패합니다. 이 SHA의 `mt_putstr_fd`/`mt_putnbr_fd`는 새 primitive를 호출하지만 반환값은 아직 상위 protocol 판단에 연결하지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `db2004556d8b`가 이 helper의 반환값을 PID publication, payload, NUL delimiter, recovery delimiter의 ACK 조건에 연결합니다. `9aa80e047514`가 모든 progress branch를 deterministic hook으로 검증합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | 한 번의 successful `write` call이 requested buffer 전체를 출력합니다. | 직전 helper는 반환 count를 누적하지 않는 single-call 구현이었습니다. |
| 실제 failure 또는 위험 | `EINTR`, short write, zero progress에서 PID·diagnostic·protocol-visible output이 잘리거나 loop가 멈출 수 있습니다. | `write`의 반환 count와 remaining length를 저장하는 state가 없었습니다. |
| root cause | descriptor write를 partial-progress interface가 아니라 atomic completion operation으로 취급했습니다. | 새 구현은 `offset`과 `written`을 분리해 progress를 직접 관리합니다. |
| 수정 불변 조건/decision | success는 모든 bytes를 쓴 경우뿐이며 interruption은 retry하고 short count만큼 전진합니다. | `mt_write_all`의 `while (offset < size)`와 세 반환 branch |
| regression | `9aa80e047514`가 first-call EINTR, one-byte short write, zero write를 주입합니다. | `tests/write_fault.c`, `tests/output_failure.sh`, fault server build seam |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: `offset = 0` → `write(fd, bytes + offset, size - offset)` → `EINTR`이면 같은 offset으로 retry → positive short count면 그 수만큼 offset 증가 → offset이 size에 도달하면 success입니다. `size == 0`은 loop를 건너뛰어 성공합니다.
- 수정된 invariant를 고정하는 후속 regression test: `9aa80e047514`의 deterministic write-fault suite입니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] complete-or-fail descriptor write
- [x] partial progress와 interruption 처리
- [x] zero progress를 무한 loop가 아닌 `EIO`로 종료

**아직 보장하지 않는 것**

- [x] server ACK ordering
- [x] endpoint cleanup
- [x] exactly-once output
- [x] 상위 void helper에서 error 전달

#### 코드 증거 기록

- 파일 경로: `src/write_utils.c`, `include/minitalk.h`
- symbol 또는 함수: `mt_write_all`, `mt_putstr_fd`, `mt_putnbr_fd`
- 확인한 state fields: `offset`, `written`
- caller → callee: `mt_putstr_fd`/`mt_putnbr_fd` → `mt_write_all` → `write`
- 핵심 branch 또는 mutation 순서: `offset = 0` → `write(fd, bytes + offset, size - offset)` → `EINTR`이면 같은 offset으로 retry → positive short count면 그 수만큼 offset 증가 → offset이 size에 도달하면 success입니다. `size == 0`은 loop를 건너뛰어 성공합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: single `write` 사용을 공통 progress loop로 교체했습니다. protocol code가 반환값을 검사하는 단계는 아직 아닙니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `826dd34c378f`, `src/write_utils.c`, `mt_write_all`. partial progress, EINTR, zero-progress failure를 한 번에 보여 주는 최소 loop입니다.

```c
bytes = (const unsigned char *)buffer;
offset = 0;
while (offset < size)
{
    written = write(fd, bytes + offset, size - offset);
    if (written == -1 && errno == EINTR)
        continue ;
    if (written == -1)
        return (-1);
    if (written == 0)
    {
        errno = EIO;
        return (-1);
    }
    offset += (size_t)written;
}
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`db2004556d8b`에서 local write success가 protocol commit condition이 됩니다.
### 2. `db2004556d8b` — fix(server): stdout 실패 뒤 ACK 전송 차단

- **중요도:** S
- **태그:** CORE, OUTPUT_COMMIT, RISK
- **개발 흐름에서의 역할:** PID, payload, terminator newline, recovery newline을 all-or-failure로 쓰고 failure 시 triggering ACK를 보내지 않습니다.

#### 원문에서 확정된 맥락

`SIGPIPE`를 ignore해 closed output을 `EPIPE` return으로 처리합니다. output failure는 event loop 밖으로 전파되고 recovery reset도 delimiter write failure를 반환합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] server startup의 `SIGPIPE` ignore
- [x] PID publication의 all-or-failure write와 startup failure
- [x] payload write 성공 뒤 line state 변경
- [x] NUL newline 성공 뒤 session reset
- [x] recovery newline 성공 뒤 coupled reset
- [x] bit processor → event loop → main error 전파
- [x] output failure branch에서 triggering ACK 생략
- [x] output success 뒤 ACK send failure의 남은 한계

#### 비교 기준

`826dd34c378f`의 local writer contract를 server call sites에 연결합니다. parent와 달리 `reset_session`과 `flush_byte`가 fallible `int`를 반환하고 그 결과가 event loop 종료까지 전달됩니다.

#### S-level 재구성

- 이 commit 직전의 관련 state와 caller/callee: writer primitive는 complete-or-fail이 됐지만 server의 protocol 상태 전이와 ACK 순서가 output success에 종속되지 않았고 일부 출력 helper는 failure를 상위로 전달하지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: state를 완료하고 ACK부터 보내면 stdout이 partial 또는 `EPIPE`여도 client는 bit가 성공했다고 판단합니다. dead-owner recovery에서도 field를 먼저 clear한 뒤 newline이 실패하면 두 session의 visible boundary가 사라집니다.
- 변경된 decision과 state mutation 순서: server의 PID, payload byte, NUL newline, recovery newline을 모두 `mt_write_all`의 반환값으로 검사하고 output error를 bit processor/event loop/main까지 전파했습니다. `SIGPIPE`는 `SIG_IGN`으로 설정해 process-killing signal 대신 error return을 받습니다. startup은 endpoint/handlers 준비 → PID text와 newline complete write → event loop입니다. completed payload는 byte write 성공 뒤 `g_line_started = 1`; NUL은 newline write 성공 뒤 session reset; recovery는 필요한 newline write 성공 뒤 coupled fields reset; 그 후에만 triggering ACK를 보냅니다.
- 정상 경로와 failure 경로가 갈라지는 조건: 어느 required write든 실패하면 processor 또는 reset이 `-1`을 반환하고 event loop/main이 종료하며 ACK는 생략됩니다. output은 성공했지만 subsequent `sendto` ACK가 실패하는 branch는 여전히 가능하고 이미 보인 output을 rollback하지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `9aa80e047514`가 normal startup/payload/NUL failure를, `081a882d7fa3`이 recovery delimiter failure를 실제 production path의 hook으로 고정합니다.

| 추적 항목 | 학습자 기록 | 코드 근거 |
| --- | --- | --- |
| 직전 architecture/state | writer는 all-or-failure지만 server ACK 판단과 output result가 아직 결합되지 않았습니다. | parent server output call sites와 `826dd34c378f: mt_write_all` |
| 해결하려던 핵심 문제 | 보이지 않은 payload 또는 delimiter를 client가 성공으로 오인하는 false ACK입니다. | payload/NUL/recovery write와 response order |
| 실패 가능한 interleaving 또는 partial failure | stdout complete 전 ACK, recovery fields clear 후 newline EPIPE, PID 일부 출력 후 startup 지속입니다. | `flush_byte`, `reset_session`, PID publication branch |
| 선택한 decision | 모든 required output을 fallible commit 단계로 만들고 ACK보다 먼저 완료합니다. | `mt_write_all` return checks와 processor return propagation |
| 소유권/lifecycle/상태 전이 | event loop가 state/output/ACK 순서를 소유하며 failure면 main/cleanup으로 전파합니다. | `process_bit` → loop → main → `cleanup_server` |
| 후속 fix 또는 regression evidence | normal output faults와 recovery newline fault를 별도 deterministic tests로 나눴습니다. | `9aa80e047514`, `081a882d7fa3` |

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | byte state를 완성하면 ACK를 보내도 됩니다. | 직전 code는 stdout completion을 authoritative success predicate로 사용하지 않았습니다. |
| 실제 failure 또는 위험 | stdout partial/`EPIPE`에도 client가 success를 관측할 수 있습니다. | output과 response가 서로 다른 실패 가능한 외부 효과입니다. |
| root cause | in-memory transition과 visible output commit order가 분리됐습니다. | 수정 diff가 write return을 processor/loop return과 ACK condition에 연결합니다. |
| 수정 불변 조건/decision | required output complete 뒤에만 triggering ACK를 보내고, recovery delimiter 성공 전에는 owner를 reset/reassign하지 않습니다. | `flush_byte`, `reset_session`, `process_bit`의 return/order |
| regression | `9aa80e047514`와 `081a882d7fa3`이 각 output site의 false-success를 검증합니다. | write fault hook, client timeout, server exit, endpoint removal assertions |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: startup은 endpoint/handlers 준비 → PID text와 newline complete write → event loop입니다. completed payload는 byte write 성공 뒤 `g_line_started = 1`; NUL은 newline write 성공 뒤 session reset; recovery는 필요한 newline write 성공 뒤 coupled fields reset; 그 후에만 triggering ACK를 보냅니다.
- 수정된 invariant를 고정하는 후속 regression test: `9aa80e047514` normal output suite와 `081a882d7fa3` recovery-newline suite입니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] visible output before success ACK
- [x] PID publication 실패 시 false readiness 방지
- [x] recovery delimiter가 session handoff의 commit 조건
- [x] output failure 뒤 endpoint 정리 과정 진입

**아직 보장하지 않는 것**

- [x] atomic output+ACK transaction
- [x] ACK-loss dedup/retry
- [x] remote persistence
- [x] 이미 성공한 output rollback

#### 코드 증거 기록

- 파일 경로: `src/server.c`, `src/write_utils.c`, `include/minitalk.h`
- symbol 또는 함수: `install_signal_handlers`, `reset_session`, `flush_byte`, `process_bit`, `run_event_loop`, `main`, `mt_write_all`
- 확인한 state fields: `g_current_byte`, `g_received_bits`, `g_client_pid`, `g_line_started`, `g_sequence`
- caller → callee: event loop → `process_bit` → `flush_byte`/`reset_session` → `mt_write_all`; success 뒤 `send_response`
- 핵심 branch 또는 mutation 순서: startup은 endpoint/handlers 준비 → PID text와 newline complete write → event loop입니다. completed payload는 byte write 성공 뒤 `g_line_started = 1`; NUL은 newline write 성공 뒤 session reset; recovery는 필요한 newline write 성공 뒤 coupled fields reset; 그 후에만 triggering ACK를 보냅니다.
- parent 또는 이전 관련 SHA와의 diff 요약: void 또는 unchecked output를 fallible calls로 바꾸고, 모든 관련 caller가 failure를 위로 반환하도록 수정했습니다. `SIGPIPE` 처리도 startup contract에 추가됐습니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `db2004556d8b`, `src/server.c`, `reset_session`. visible delimiter가 성공하기 전에는 owner와 partial state를 지우지 않는 commit order를 보여 줍니다.

```c
static int reset_session(int close_partial_line)
{
    if (close_partial_line && g_line_started
        && mt_write_all(STDOUT_FILENO, "\n", 1) == -1)
        return (-1);
    g_current_byte = 0;
    g_received_bits = 0;
    g_client_pid = 0;
    g_line_started = 0;
    g_sequence = 0;
    return (0);
}
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`9aa80e047514`가 normal output failures를, `081a882d7fa3`이 recovery newline failure를 검증합니다.
### 3. `9aa80e047514` — test(server): 부분 쓰기와 출력 실패 검증

- **중요도:** A
- **태그:** TEST, OUTPUT_COMMIT, RISK
- **개발 흐름에서의 역할:** low-level write를 deterministic test implementation으로 바꿔 EINTR, one-byte short write, zero write, selected payload/newline EPIPE를 주입합니다.

#### 원문에서 확정된 맥락

complete output under partial progress와 startup/message failure의 no-false-success, endpoint cleanup, client timeout을 검증합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] production write를 hook으로 바꾸는 fault build seam
- [x] first-call EINTR mode
- [x] 한 call당 1 byte short-write mode
- [x] selected zero/EPIPE call state
- [x] partial progress 뒤 complete output assertion
- [x] PID zero-write startup failure
- [x] payload/NUL newline EPIPE 뒤 no ACK와 cleanup

#### 비교 기준

`826dd34c378f`의 `mt_write_all` branches와 `db2004556d8b`의 PID/payload/NUL output sites를 test mode별로 일대일 매핑합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: production code에는 all-or-failure loop와 output-before-ACK branch가 있었지만 actual kernel timing에 의존하면 EINTR/short/zero/특정 EPIPE 지점을 반복 재현하기 어려웠습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 일반 smoke test는 write가 항상 한 번에 완료되는 환경에서 정상 처리만 통과할 수 있어 partial-progress 알고리즘과 각 output failure의 ACK 억제를 증명하지 못합니다.
- 변경된 decision과 state mutation 순서: fault build에서 production `write` 호출을 test wrapper로 치환하고 환경/내부 call state로 first EINTR, one-byte maximum, selected zero, selected byte/newline EPIPE를 결정적으로 반환합니다. Makefile fault target → `tests/write_fault.c` hook 선택 → real server algorithm 실행 → shell scenario가 server/client status, stdout/stderr, socket path를 관측합니다. EINTR/short mode는 같은 logical output을 완성하고, zero/EPIPE mode는 server error와 no-success response로 끝납니다.
- 정상 경로와 failure 경로가 갈라지는 조건: startup PID zero-write는 PID publication 전에 실패합니다. payload 또는 terminating newline EPIPE는 bit processor가 실패하고 ACK가 없어 client가 bounded timeout합니다. short/EINTR mode는 offset loop가 계속 진행해 expected bytes를 완성합니다.
- 후속 commit이 강화하거나 교체하는 부분: `081a882d7fa3`이 같은 seam을 dead-owner recovery newline이라는 별도 branch에 적용합니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** stdout effect가 complete되지 않으면 해당 transition을 success ACK하지 않습니다.
- **재현하는 failure 또는 boundary:** EINTR, one-byte short write, zero progress, payload/newline EPIPE
- **사용한 test technique:** compile-time/link-time write hook against the real server algorithm
- **분류:** deterministic production-path fault regression
- **failure 주입 또는 process orchestration 시작 지점:** Makefile의 fault server build가 `tests/write_fault.c`를 연결하고 `tests/output_failure.sh`가 mode별 환경을 설정합니다.
- **production code에 진입하는 최초 호출:** server PID publication 또는 signal event가 `mt_write_all`을 호출하면서 test hook에 진입합니다.
- **핵심 assertion과 관측값:** short/EINTR에서는 expected output 완성, zero/EPIPE에서는 nonzero server/client status, timeout diagnostic, no false payload, bound endpoint 제거를 검사합니다.
- **증명하는 것:** partial progress 보존<br>EINTR retry<br>zero progress failure<br>required output failure 뒤 no false ACK<br>startup/message cleanup
- **증명하지 않는 것:** recovery delimiter branch<br>output success 뒤 datagram loss<br>exactly-once<br>실제 시스템 부하에서 모든 scheduling
- **후속 변경에서 막아야 할 구체적인 회귀:** writer가 short count를 무시하거나 output error 뒤 ACK를 보내는 변경을 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] EINTR/short write에서도 complete output
- [x] zero progress의 controlled startup failure
- [x] payload/NUL failure 뒤 false ACK 방지
- [x] server endpoint cleanup과 client bounded failure

**아직 보장하지 않는 것**

- [x] recovery newline failure
- [x] output success 뒤 ACK loss
- [x] exactly-once transaction
- [x] 모든 OS write failure 조합

#### 코드 증거 기록

- 파일 경로: `Makefile`, `tests/write_fault.c`, `tests/write_fault.h`, `tests/output_failure.sh`, `src/write_utils.c`, `src/server.c`
- symbol 또는 함수: `mt_test_write`, `mt_write_all`, `fault server target`, `output_failure.sh scenarios`
- 확인한 state fields: `fault call count`, `selected mode`, `selected byte/newline ordinal`
- caller → callee: fault server의 `mt_write_all` → test hook → configured return; shell → real client/server process 관측
- 핵심 branch 또는 mutation 순서: Makefile fault target → `tests/write_fault.c` hook 선택 → real server algorithm 실행 → shell scenario가 server/client status, stdout/stderr, socket path를 관측합니다. EINTR/short mode는 같은 logical output을 완성하고, zero/EPIPE mode는 server error와 no-success response로 끝납니다.
- parent 또는 이전 관련 SHA와의 diff 요약: production binary는 유지하고 test-only link seam과 fault server target, process-level assertion script를 추가했습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`081a882d7fa3`에서 recovery delimiter까지 같은 invariant를 검증합니다.
### 4. `081a882d7fa3` — test(server): 회수 줄바꿈 출력 실패 검증

- **중요도:** A
- **태그:** TEST, OUTPUT_COMMIT, SESSION
- **개발 흐름에서의 역할:** dead owner가 visible partial line을 남긴 상태에서 replacement acquisition이 recovery newline을 쓰는 순간 failure를 주입합니다.

#### 원문에서 확정된 맥락

server는 event loop를 종료하고 replacement client는 READY/ACK success가 아니라 timeout해야 합니다. fields만 clear하고 visible boundary를 쓰지 못한 상태를 successful recovery로 보지 않습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] visible partial session setup
- [x] owner exit 뒤 replacement ACQUIRE 순서
- [x] recovery newline ordinal을 선택하는 fault mode
- [x] delimiter EPIPE 뒤 server error exit
- [x] replacement client timeout/no READY
- [x] endpoint cleanup과 new payload 미출력 assertion

#### 비교 기준

`db2004556d8b: reset_session(close_partial_line)`의 write-before-clear order와 `9aa80e047514`의 hook seam을 함께 사용합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `9aa80e047514`는 PID, normal payload, NUL delimiter를 검증했지만 competing ACQUIRE가 dead owner를 회수하면서 쓰는 별도 newline branch는 다루지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: recovery branch가 field reset과 reassignment만 성공으로 처리하면 newline EPIPE 뒤에도 replacement client에게 READY를 보내 두 session의 visible output이 합쳐질 수 있습니다.
- 변경된 decision과 state mutation 순서: specialized partial sender로 complete visible byte와 unfinished state를 남긴 뒤 owner를 종료하고, replacement ACQUIRE 시 두 번째 newline write에 EPIPE를 주입하도록 fault mode를 확장했습니다. server start → partial-session helper가 acquire/visible byte/partial bit 후 exit → replacement client ACQUIRE → owner unavailable 판단 → `reset_session(1)`의 recovery newline write → injected EPIPE → reset/READY 생략 → loop/main failure → endpoint cleanup; replacement client는 timeout합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: delimiter write가 실패하면 coupled fields를 clear하지 않고 `reset_session`이 `-1`을 반환합니다. event loop는 종료되고 replacement READY와 subsequent bit ACK 모두 전송되지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: 이 commit이 Thread final regression입니다. 후속 상태에서도 recovery output은 successful handoff의 선행 조건이어야 합니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** visible abandoned line을 delimit하기 전에는 owner reassignment가 완료되지 않습니다.
- **재현하는 failure 또는 boundary:** new ACQUIRE 중 recovery newline EPIPE
- **사용한 test technique:** partial-session real process + selected write fault + real replacement client
- **분류:** failure-path-specific deterministic regression
- **failure 주입 또는 process orchestration 시작 지점:** session sender가 visible byte와 partial next byte를 남기고 종료한 뒤 replacement client를 실행합니다.
- **production code에 진입하는 최초 호출:** server의 competing ACQUIRE path가 unavailable owner를 확인하고 `reset_session(1)`을 호출합니다.
- **핵심 assertion과 관측값:** server/client nonzero status, client timeout diagnostic, server endpoint removal, replacement message 미출력, abandoned line 이후 성공 boundary 부재를 검사합니다.
- **증명하는 것:** delimiter failure suppresses reassignment<br>READY/ACK 없음<br>server fail-stop<br>cleanup 실행
- **증명하지 않는 것:** recovery retry<br>output rollback<br>delimiter 성공 뒤 ACK loss<br>다중 replacement 경쟁
- **후속 변경에서 막아야 할 구체적인 회귀:** fields를 먼저 clear하거나 delimiter failure 뒤 READY를 보내는 변경을 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] failed recovery delimiter가 READY/ACK를 억제
- [x] owner reassignment false success 방지
- [x] unrecoverable output failure 뒤 fail-stop cleanup

**아직 보장하지 않는 것**

- [x] recovery retry
- [x] 이미 출력된 partial line rollback
- [x] delimiter 성공 뒤 ACK loss
- [x] persistent output transaction

#### 코드 증거 기록

- 파일 경로: `tests/output_failure.sh`, `tests/session_sender.c`, `tests/write_fault.c`, `src/server.c`
- symbol 또는 함수: `reset_session`, `session_sender partial mode`, `recovery-newline fault scenario`
- 확인한 state fields: `g_line_started`, `g_client_pid`, `g_current_byte`, `g_received_bits`, `g_sequence`, `fault newline count`
- caller → callee: replacement ACQUIRE handling → owner availability recovery → `reset_session(1)` → `mt_write_all` → fault hook
- 핵심 branch 또는 mutation 순서: server start → partial-session helper가 acquire/visible byte/partial bit 후 exit → replacement client ACQUIRE → owner unavailable 판단 → `reset_session(1)`의 recovery newline write → injected EPIPE → reset/READY 생략 → loop/main failure → endpoint cleanup; replacement client는 timeout합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: 기존 output fault suite에 dead-owner recovery setup과 selected recovery-newline failure assertion을 추가했습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

이 commit이 Thread final regression입니다.

## 6. 불변 조건 ledger

### 원자료에서 확인된 핵심 불변 조건

- writer success는 requested bytes 전체가 descriptor interface에 전달됐음을 뜻합니다.
- EINTR와 short write는 progress를 보존해 끝까지 처리하며 zero write는 `EIO`입니다.
- payload byte 또는 delimiter ACK는 corresponding stdout write success 뒤에만 전송됩니다.
- recovery delimiter failure는 session reassignment success로 처리되지 않습니다.
- unrecoverable output failure는 정리 과정으로 전파되고 false ACK를 막습니다.

### 시간에 따른 변화 기록

| Commit | 원자료에서 확인된 변화 | 실제 state/condition | code evidence | 상태: 도입·강화·부족·복구·검증 |
| --- | --- | --- | --- | --- |
| `826dd34c378f` | `mt_write_all`이 EINTR를 retry하고 short write만큼 offset을 전진하며 zero progress를 `EIO`로 처리합니다. | `offset`이 requested `size`에 도달해야 success이고, zero progress는 `EIO`입니다. | `src/write_utils.c: mt_write_all` | 도입 |
| `db2004556d8b` | PID, payload, terminator newline, recovery newline을 all-or-failure로 쓰고 failure 시 triggering ACK를 보내지 않습니다. | output success가 ACK 전에 필요한 protocol commit condition이 되고 recovery fields는 delimiter 성공 뒤에만 clear됩니다. | `src/server.c: flush_byte`, `reset_session`, bit processor response order | 강화·수정 |
| `9aa80e047514` | low-level write를 deterministic test implementation으로 바꿔 EINTR, one-byte short write, zero write, selected payload/newline EPIPE를 주입합니다. | fault hook이 production progress/commit branches를 결정적으로 선택하고 process-level outcomes를 확인합니다. | `tests/write_fault.c`, `tests/output_failure.sh`, Makefile fault target | 검증 |
| `081a882d7fa3` | dead owner가 visible partial line을 남긴 상태에서 replacement acquisition이 recovery newline을 쓰는 순간 failure를 주입합니다. | recovery delimiter가 성공하지 않으면 owner/reset/reassignment transition이 commit되지 않습니다. | `tests/output_failure.sh` recovery scenario와 `src/server.c: reset_session` | 검증 |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 또는 상태 | 실제 failure/위험 | Fix 또는 전환 commit | 수정된 decision/불변 조건 | Test 또는 후속 검증 | 학습자 code evidence |
| --- | --- | --- | --- | --- | --- |
| single `write` success를 complete output으로 간주 | EINTR/short write에서 PID·payload truncation 가능 | `826dd34c378f` | `mt_write_all` all-or-failure loop | `9aa80e047514` | writer hook의 EINTR/one-byte/zero branches |
| state transition 후 ACK부터 전송 | stdout failure에도 client가 success 관측 | `db2004556d8b` | output complete 뒤 ACK, failure면 server exit | `9aa80e047514` | payload/newline EPIPE, timeout, cleanup assertions |
| dead-owner fields를 먼저 reset | recovery newline failure에도 clean handoff로 오인 | `db2004556d8b` | delimiter success를 recovery commit 조건에 포함 | `081a882d7fa3` | recovery newline EPIPE 뒤 no READY/ACK |

전용 test commit이 없는 연결에는 존재하지 않는 test를 만들어 적지 않았습니다.

## 8. 소유권 / state / responsibility 변화

| 단계 | state 또는 responsibility owner | transition | 당시 한계 또는 다음 변화 | 실제 symbol/field |
| --- | --- | --- | --- | --- |
| write primitive | `mt_write_all` caller | offset와 remaining을 끝까지 소유 | success 의미 통일 | `bytes`, `offset`, `written` |
| server byte commit | event loop/bit processor | state 판단 → stdout → ACK | write failure 시 ACK 없음 | `flush_byte`, `process_bit`, `g_sequence` |
| recovery commit | session reset caller | delimiter success 뒤 reset/reassign | failure 시 server 종료 | `reset_session(close_partial_line)` |
| startup | main/initialization | PID line complete write 뒤 ready | false readiness 방지 | PID buffer/publication path, `cleanup_server` |

## 9. 개발 흐름의 최종 상태

원자료에서 확인된 최종 조건:

- 모든 protocol-visible server output은 all-or-failure writer를 사용합니다.
- ACK는 triggering bit가 요구한 visible output effect가 성공한 뒤에만 전송됩니다.
- closed stdout pipe는 asynchronous termination 대신 `EPIPE` 오류 처리로 처리됩니다.
- output success 뒤 ACK loss는 남으므로 exactly-once transaction은 아닙니다.

학습자 기록:

- 최종 state fields와 owner: `mt_write_all`의 `offset`이 local write progress를 소유하고, event loop가 `g_current_byte`, `g_received_bits`, `g_client_pid`, `g_line_started`, `g_sequence`와 output/ACK 순서를 소유합니다.
- 정상 transition 순서: bit event가 byte 또는 delimiter completion을 결정한 뒤 `mt_write_all`이 필요한 output을 모두 완료합니다. 성공한 경우에만 line/session state를 commit하고 matching ACK를 보냅니다.
- 실패 시 중단·reset·cleanup 순서: EINTR은 retry하고 short count는 누적합니다. zero/terminal error는 processor→event loop→main failure로 전파되며 triggering ACK를 생략하고 registered cleanup이 descriptor와 bound endpoint를 정리합니다.
- 최종 상태가 보장하지 않는 것: output과 ACK의 원자적 transaction, output rollback, ACK 재전송·deduplication, remote persistence는 제공하지 않습니다.
- 이 개발 흐름을 한 문단으로 설명한 최종 서술: 이 개발 흐름은 descriptor write를 partial-progress interface로 바로잡은 뒤 그 success를 server protocol의 commit 조건으로 끌어올립니다. PID, payload, NUL delimiter와 abandoned-session delimiter는 모두 complete write 뒤에만 다음 state나 ACK로 넘어갑니다. deterministic fault tests는 EINTR·short·zero·EPIPE의 각 branch를 고정하지만 output 성공 후 ACK 손실까지 제거하지는 못합니다.

## 10. 최종 architecture 또는 실행 순서 정리

- [x] bit event로 byte/delimiter completion 결정
- [x] output buffer와 length 선택
- [x] `mt_write_all` retry/advance loop
- [x] write success 뒤 state reset 또는 line state update
- [x] matching ACK send
- [x] write failure에서 ACK 생략과 event-loop error
- [x] main exit와 endpoint cleanup

```text
signal event in normal event loop
    -> authorize and append one bit
    -> if byte incomplete: prepare ACK
    -> if payload complete: mt_write_all(payload)
    -> if NUL complete: mt_write_all("\n") -> reset session
    -> only after required output: send matching ACK
output failure
    -> processor -1 -> event loop -1 -> main failure
    -> no triggering ACK -> cleanup socket/pipe/path
dead-owner recovery
    -> if visible line: mt_write_all("\n")
    -> success only: clear coupled fields and consider reassignment

```

- 실제 함수·파일을 반영한 완성 흐름: `src/server.c`의 bit processor/`flush_byte`/`reset_session`과 `src/write_utils.c: mt_write_all`이 output commit path를 구성합니다.
- asynchronous boundary: 시그널 처리 함수는 event만 enqueue하며 stdout commit과 ACK 판단은 event loop normal 문맥에서 실행됩니다.
- externally visible commit point: 현재 transition이 요구하는 stdout bytes가 모두 쓰이고, 그 뒤 matching ACK를 전송하는 시점입니다. output 성공 자체와 ACK delivery는 하나의 원자적 transaction이 아닙니다.
- cleanup owner: server main/registered cleanup이 output 또는 response failure 뒤 response socket, self-pipe, bound path를 정리합니다.

## 11. 학습 완료 자가 점검

- [x] commit map의 4개 SHA를 source 순서대로 모두 설명할 수 있습니다.
- [x] 각 code excerpt에 SHA, path, symbol, 선택 이유가 기록돼 있습니다.
- [x] 최종 HEAD 코드를 historical SHA의 증거로 사용한 곳이 없습니다.
- [x] 정상 경로와 실패 처리를 state mutation 순서로 설명할 수 있습니다.
- [x] source 확정 불변 조건과 직접 확인한 code evidence를 구분했습니다.
- [x] test commit의 불변 조건, failure, technique, production path, proves/not-proves를 기록했습니다.
- [x] Thread final state를 함수와 state field 수준으로 설명할 수 있습니다.
- [ ] 해당 SHA의 test를 로컬에서 직접 실행했습니다. — 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

### 이 Thread와 직접 연결된 Major Engineering Difficulties

- partial descriptor progress와 protocol state progress를 같은 commit order로 묶어야 합니다.
- output success 뒤 ACK loss는 exactly-once transaction 없이 완전히 제거할 수 없습니다.
- PID, payload, NUL newline, recovery newline의 모든 output sites를 같은 contract로 바꿔야 합니다.

---

# 엔드포인트 소유 관계와 제한된 폴링

> 완성형 해설서가 아닙니다. 아래 확정 사항을 기준으로 각 commit SHA의 실제 코드와 diff를 읽고 기록란을 채웁니다.

## 1. 개발 흐름 목표

per-UID Unix socket namespace와 client/server endpoint 수명을 복원하고, path를 계산한 사실과 actual bind 소유권을 구분한 fix, `fd_set`이 표현할 수 없는 descriptor를 startup에서 거부하는 runtime boundary를 확인합니다.

### Significance

predictable socket path는 naming convenience가 아니라 filesystem authority와 cleanup responsibility를 만듭니다. same-UID stale socket은 교체할 수 있지만 regular file이나 자신이 bind하지 않은 entry를 삭제해서는 안 됩니다. valid descriptor라도 `FD_SETSIZE` 이상이면 `FD_SET`이 object 밖을 쓸 수 있으므로 polling representation의 한계를 controlled failure로 바꿔야 합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- runtime directory의 UID와 permissions를 어떤 checks로 보장하는가?
- role/PID path에서 invalid role, nonpositive PID, `sun_path` overflow를 어디서 거부하는가?
- client/server socket descriptor, bound path, cleanup registration의 acquisition order는 무엇입니까?
- same-UID stale socket과 protected regular file을 어떤 code path로 구분하는가?
- bind failure 뒤 cleanup이 unowned path를 삭제할 수 있었던 state mistake는 무엇입니까?
- 어떤 descriptors가 `fd_set`에 들어가며 `FD_SETSIZE` guard가 각 `FD_SET` 전에 실행됩니까?
- high-descriptor test는 mock integer가 아니라 real inherited table을 어떻게 만드는가?

## 3. 완료 기준

- [x] runtime directory와 path helper의 validation/permission rules를 코드로 기록했습니다.
- [x] client/server의 socket create → flags → stale handling → bind → cleanup 순서를 복원했습니다.
- [x] computed path, stale replacement authority, successful bind 소유권을 구분했습니다.
- [x] regular-file preservation과 clean startup failure를 test assertion에 연결했습니다.
- [x] client socket, server socket, self-pipe read fd guards와 real high-fd regression을 확인했습니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `2c37cb592d05` | feat(runtime): 안전한 응답 endpoint 경로 생성 | A | ARCH, ENDPOINT, RISK | per-UID private runtime directory와 role/PID-derived Unix socket path helper를 정의합니다. |
| 2 | `25780b881ee8` | feat(client): datagram 응답 endpoint 수명주기 관리 | B | ENDPOINT, PROCESS_LIFECYCLE | client가 nonblocking, close-on-exec datagram socket을 PID-derived path에 bind하고 invocation lifetime에 맞춰 cleanup합니다. |
| 3 | `32390dcdfc1b` | feat(server): datagram 응답 endpoint 수명주기 관리 | B | ENDPOINT, PROCESS_LIFECYCLE | server가 long-lived nonblocking, close-on-exec datagram endpoint를 server path에 bind하고 normal exit와 rollback에서 cleanup합니다. |
| 4 | `622d80020fb2` | fix(client): bind한 응답 경로만 정리 | A | ENDPOINT, RISK | client cleanup이 response path를 실제 `bind`한 경우에만 unlink하도록 bound ownership flag를 사용합니다. |
| 5 | `ffd3647a1518` | test(runtime): stale 응답 endpoint 처리 검증 | A | TEST, ENDPOINT, RISK | real PID-derived paths에 stale client/server sockets, regular files, unrelated live processes를 만들어 endpoint trust와 cleanup policy를 검증합니다. |
| 6 | `4e1c84bfacfc` | fix(runtime): select 범위를 벗어난 descriptor 거부 | A | EDGE, PRACTICAL, RISK | client response socket, server response socket, self-pipe read fd가 `FD_SETSIZE` 이상이면 initialization에서 거부합니다. |
| 7 | `1de95310195d` | test(runtime): 높은 descriptor 번호의 안전한 실패 검증 | A | TEST, EDGE, RISK | wrapper가 `/dev/null`을 반복 open해 inherited descriptor table을 `FD_SETSIZE` boundary까지 높인 뒤 real client/server를 `exec`합니다. |

확인 원칙:

- 각 항목은 해당 SHA의 tree를 기준으로 읽었습니다.
- 변경 전 상태는 해당 SHA의 parent 또는 지정된 이전 관련 SHA에서 확인했습니다.
- 같은 commit이 다른 Thread에 다시 등장해도 이 개발 흐름의 질문으로 별도 기록했습니다.
- runtime test는 실행하지 않았으며, 실행 결과처럼 표현하지 않았습니다.

## 5. 커밋별 학습 기록

### 1. `2c37cb592d05` — feat(runtime): 안전한 응답 endpoint 경로 생성

- **중요도:** A
- **태그:** ARCH, ENDPOINT, RISK
- **개발 흐름에서의 역할:** per-UID private runtime directory와 role/PID-derived Unix socket path helper를 정의합니다.

#### 원문에서 확정된 맥락

directory owner와 permissions를 검사해 group/other accessible state를 거부하고 helper는 role, positive PID, Unix path length를 검증합니다. cooperative local integrity boundary이며 same-UID authentication은 아닙니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] UID를 runtime directory name에 포함
- [x] `mkdir(..., 0700)`과 existing directory `lstat`
- [x] directory type/current UID/`(mode & 077) == 0` 검사
- [x] role whitelist와 positive PID check
- [x] formatted path truncation refusal
- [x] host-local same-UID boundary의 한계

#### 비교 기준

parent에는 shared response endpoint namespace가 없습니다. 이 commit은 path를 계산할 authority만 정의하며 successful bind 소유권은 아직 caller가 관리해야 합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: control datagram schema는 정의됐지만 client/server가 사용할 filesystem namespace, path validation, stale-entry policy의 공통 기준이 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: predictable AF_UNIX path를 임의로 만들면 다른 UID가 접근 가능한 directory, 잘린 `sun_path`, invalid PID/role, object type을 확인하지 않은 destructive cleanup으로 이어질 수 있습니다.
- 변경된 decision과 state mutation 순서: `src/response_channel.c`에 current UID를 포함한 private runtime directory helper와 role/PID path helper를 추가했습니다. existing directory는 type, owner, permission bits를 검증합니다. `mt_response_path`가 buffer/role/PID를 검사 → `mt_runtime_dir`가 `/tmp/signal-message-bus-<uid>`를 format → `mkdir(...,0700)` 또는 existing dir 검사 → role whitelist → `<dir>/<role>-<pid>.sock` format과 truncation 검사입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: NULL/zero buffer, `pid <= 1`, unknown role는 failure입니다. directory가 current UID 소유 directory가 아니거나 mode의 `077` 중 하나라도 켜져 있으면 `EACCES`; formatted length가 capacity 이상이면 `ENAMETOOLONG`입니다.
- 후속 commit이 강화하거나 교체하는 부분: `25780b881ee8`과 `32390dcdfc1b`가 client/server socket lifetime에 helper를 적용하고 `ffd3647a1518`이 actual filesystem objects로 policy를 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] authoritative endpoint naming
- [x] private per-UID namespace validation
- [x] role/PID/path-length rejection

**아직 보장하지 않는 것**

- [x] same-UID authentication
- [x] actual bind 소유권
- [x] stale socket replacement의 전체 lifecycle
- [x] polling fd bounds

#### 코드 증거 기록

- 파일 경로: `src/response_channel.c`, `include/minitalk.h`, `Makefile`
- symbol 또는 함수: `validate_runtime_dir`, `mt_runtime_dir`, `mt_response_path`
- 확인한 state fields: `directory path buffer`, `formatted length`
- caller → callee: client/server endpoint setup → `mt_response_path` → `mt_runtime_dir` → `validate_runtime_dir`
- 핵심 branch 또는 mutation 순서: `mt_response_path`가 buffer/role/PID를 검사 → `mt_runtime_dir`가 `/tmp/signal-message-bus-<uid>`를 format → `mkdir(...,0700)` 또는 existing dir 검사 → role whitelist → `<dir>/<role>-<pid>.sock` format과 truncation 검사입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: 새 source file과 header declarations/build object가 추가됐고 아직 socket creation/bind는 없습니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `2c37cb592d05`, `src/response_channel.c`, `validate_runtime_dir`. namespace가 directory type, owner, group/other access를 모두 요구함을 보여 줍니다.

```c
if (!S_ISDIR(info.st_mode) || info.st_uid != getuid()
    || (info.st_mode & 077) != 0)
{
    errno = EACCES;
    return (-1);
}
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`25780b881ee8`과 `32390dcdfc1b`가 client/server lifetime에 이 policy를 적용합니다.
### 2. `25780b881ee8` — feat(client): datagram 응답 endpoint 수명주기 관리

- **중요도:** B
- **태그:** ENDPOINT, PROCESS_LIFECYCLE
- **개발 흐름에서의 역할:** client가 nonblocking, close-on-exec datagram socket을 PID-derived path에 bind하고 invocation 수명에 맞춰 cleanup합니다.

#### 원문에서 확정된 맥락

existing entry는 same-UID socket일 때만 remove하며 initialization failure는 acquired resources를 unwind합니다. later fix가 bind 소유권 condition을 더 좁힙니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] datagram socket create
- [x] O_NONBLOCK와 FD_CLOEXEC setup
- [x] client PID-derived path 계산
- [x] same-UID socket만 stale unlink
- [x] AF_UNIX bind와 cleanup registration
- [x] partial initialization unwind
- [x] 당시 path-nonempty cleanup condition

#### 비교 기준

`2c37cb592d05`의 naming policy를 concrete socket lifetime에 적용합니다. server처럼 long-lived endpoint가 아니라 한 client invocation에만 존재합니다.

#### B-level 구현 역할 기록

- Thread 전체에서 이 commit이 연결하는 앞/뒤 단계: path helper만 존재했고 client가 READY/ACK datagram을 받을 concrete descriptor와 bound destination을 소유하지 않았습니다. → `32390dcdfc1b`가 server counterpart를 추가하고 `622d80020fb2`가 client cleanup authority를 actual bind success로 좁힙니다.
- 실제로 추가·수정된 핵심 symbol과 state: client에 global response socket/path state, nonblocking+CLOEXEC flag helper, same-UID stale-socket removal, AF_UNIX bind, `atexit` cleanup을 추가했습니다.
- 이 commit만으로 충분하지 않아 후속 commit을 확인해야 하는 부분: client별 response endpoint가 없으면 server가 reply destination을 식별할 수 없습니다. 동시에 predictable path의 stale object와 partial initialization을 안전하게 정리해야 합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] concrete client reply destination 수명
- [x] nonblocking/CLOEXEC setup
- [x] stale same-UID socket replacement과 protected object rejection

**아직 보장하지 않는 것**

- [x] bind 소유권과 unlink의 완전한 대칭
- [x] server endpoint
- [x] same-UID authentication
- [x] FD_SETSIZE guard

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/response_channel.c`, `include/minitalk.h`
- symbol 또는 함수: `bind_client_socket`, `cleanup_response_socket`, `set_nonblocking_close_on_exec`, `remove_stale_socket`
- 확인한 state fields: `g_response_socket`, `g_client_path`
- caller → callee: client `main`/initialization → `bind_client_socket` → path/stale/socket/flags/bind; exit → `cleanup_response_socket`
- 핵심 branch 또는 mutation 순서: path 계산 → existing path `lstat`/same-UID socket이면 unlink → socket create → O_NONBLOCK → FD_CLOEXEC → sockaddr 작성/length check → bind → cleanup registration/사용입니다. 이 SHA의 cleanup은 path가 계산돼 nonempty이면 unlink해 bind 성공 여부와 완전히 대칭적이지 않습니다.
- parent 또는 이전 관련 SHA와의 diff 요약: client에 response resource acquisition과 cleanup이 추가됐지만 actual bind 소유권을 나타내는 boolean은 아직 없습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`32390dcdfc1b`에서 server counterpart가 추가되고 `622d80020fb2`에서 client cleanup을 수정합니다.
### 3. `32390dcdfc1b` — feat(server): datagram 응답 endpoint 수명주기 관리

- **중요도:** B
- **태그:** ENDPOINT, PROCESS_LIFECYCLE
- **개발 흐름에서의 역할:** server가 long-lived nonblocking, close-on-exec datagram endpoint를 server path에 bind하고 normal exit와 rollback에서 cleanup합니다.

#### 원문에서 확정된 맥락

startup은 stale same-UID socket만 제거하고 bind success state를 기록합니다. filesystem path가 crash 뒤 지속될 수 있어 lifecycle bookkeeping이 필요합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] server datagram socket create/flags
- [x] server PID path와 same-UID stale policy
- [x] `g_server_bound` set timing
- [x] response socket event-loop registration
- [x] startup rollback close/unlink condition
- [x] normal exit cleanup

#### 비교 기준

client counterpart와 acquisition 단계는 유사하지만 server는 event pipe와 long-lived `pselect` registration을 함께 소유하고 이미 explicit bound flag를 사용합니다.

#### B-level 구현 역할 기록

- Thread 전체에서 이 commit이 연결하는 앞/뒤 단계: client endpoint는 존재하지만 server가 ACQUIRE를 받거나 READY/ACK을 보내는 long-lived socket과 event-loop registration은 없었습니다. → `ffd3647a1518`이 stale/protected objects를 검증하고 `4e1c84bfacfc`가 response socket과 self-pipe fd의 polling range를 제한합니다.
- 실제로 추가·수정된 핵심 symbol과 state: server에 event/response channel preparation, response socket/path와 `g_server_bound`, stale validation, nonblocking+CLOEXEC, bind, event-loop `FD_SET`, registered cleanup을 추가했습니다.
- 이 commit만으로 충분하지 않아 후속 commit을 확인해야 하는 부분: server path는 process 종료 뒤 filesystem에 남을 수 있고 startup 중 partial resource acquisition도 가능하므로 descriptor와 bound name의 소유권을 별도로 기록해야 합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] long-lived server endpoint lifecycle
- [x] bind-aware startup rollback와 cleanup
- [x] response socket의 event-loop integration

**아직 보장하지 않는 것**

- [x] client 소유권 bug fix
- [x] fd range safety
- [x] shutdown signal integration
- [x] same-UID authentication

#### 코드 증거 기록

- 파일 경로: `src/server.c`, `src/response_channel.c`, `include/minitalk.h`
- symbol 또는 함수: `prepare_response_channel`, `remove_stale_socket`, `cleanup_server`, `run_event_loop`
- 확인한 state fields: `g_response_socket`, `g_server_path`, `g_server_bound`, `g_event_pipe`
- caller → callee: server `main` → `prepare_response_channel` → path/stale/socket/flags/bind; loop → `FD_SET`; exit → `cleanup_server`
- 핵심 branch 또는 mutation 순서: event pipe setup → server path 계산/stale check → datagram socket create/flags → bind → `g_server_bound = 1` → event loop가 socket을 read set에 등록 → normal/error exit에서 close 후 bound path만 unlink합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: server response endpoint, bound flag, event-loop input와 cleanup이 추가됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`622d80020fb2`와 `ffd3647a1518`이 cleanup authority와 stale policy를 강화합니다.
### 4. `622d80020fb2` — fix(client): bind한 응답 경로만 정리

- **중요도:** A
- **태그:** ENDPOINT, RISK
- **개발 흐름에서의 역할:** client cleanup이 response path를 실제 `bind`한 경우에만 unlink하도록 bound 소유권 flag를 사용합니다.

#### 원문에서 확정된 맥락

path 계산은 namespace object 생성 증거가 아닙니다. existing endpoint 때문에 bind가 실패했는데 cleanup이 unconditional unlink하면 다른 process entry를 삭제할 수 있습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] client endpoint state의 `g_client_bound`
- [x] bind success 직후 flag set
- [x] bind 전/실패 시 false state
- [x] descriptor close와 path unlink의 separate conditions
- [x] successful exit의 close/unlink symmetry

#### 비교 기준

`25780b881ee8`의 cleanup condition을 direct diff하면 path nonempty check에 `g_client_bound`가 추가되고 bind success 뒤 flag assignment가 생깁니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `25780b881ee8`의 client는 `g_client_path[0] != 0`이면 cleanup에서 unlink했습니다. path는 bind보다 먼저 계산되므로 bind failure에서도 nonempty였습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 동일 PID-derived name에 protected/existing object가 있어 bind가 실패한 경우, 실패 cleanup이 자신이 생성하지 않은 object를 지울 수 있습니다. descriptor 소유권과 path 소유권이 다른 acquisition 단계입니다.
- 변경된 decision과 state mutation 순서: `g_client_bound` boolean을 추가하고 successful `bind` 직후에만 1로 설정했습니다. cleanup은 descriptor close와 path unlink를 separate conditions로 처리합니다. path 계산/possible stale handling → socket create/configure → bind → success 직후 `g_client_bound = 1`; cleanup은 fd가 있으면 close하고, bound flag와 nonempty path가 모두 true일 때만 unlink한 뒤 flag/path를 reset합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: bind 전 또는 bind 실패에는 bound flag가 0이므로 computed path를 삭제하지 않습니다. successful invocation/후속 failure에는 flag가 1이어서 자신이 bind한 path를 제거합니다.
- 후속 commit이 강화하거나 교체하는 부분: `ffd3647a1518`이 stale socket은 replace하고 regular file/unowned entry는 보존하는 regression으로 이 authority rule을 고정합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | path를 계산했으면 cleanup에서 삭제해도 됩니다. | `25780b881ee8` cleanup은 nonempty path만 확인했습니다. |
| 실제 failure 또는 위험 | bind failure 원인인 existing endpoint 또는 unowned object를 삭제할 수 있습니다. | path 계산은 bind 전이고 cleanup은 init failure에도 실행됩니다. |
| root cause | name knowledge와 resource 소유권을 동일시했습니다. | descriptor와 path의 acquisition 시점이 다른데 하나의 path-nonempty state만 사용했습니다. |
| 수정 불변 조건/decision | successful bind가 있었을 때만 path를 unlink합니다. | `g_client_bound` set/reset과 cleanup condition |
| regression | `ffd3647a1518`이 regular file preservation과 stale socket replacement를 검증합니다. | `tests/stale_exec.c`, `tests/protocol_regressions.sh`의 real path scenarios |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: path 계산/possible stale handling → socket create/configure → bind → success 직후 `g_client_bound = 1`; cleanup은 fd가 있으면 close하고, bound flag와 nonempty path가 모두 true일 때만 unlink한 뒤 flag/path를 reset합니다.
- 수정된 invariant를 고정하는 후속 regression test: `ffd3647a1518` stale/protected endpoint integration test입니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] unlink authority와 successful bind의 대칭
- [x] bind하지 않은 path 보존
- [x] descriptor와 filesystem name의 독립 cleanup

**아직 보장하지 않는 것**

- [x] full stale policy의 adversarial authentication
- [x] server cleanup 변경
- [x] same-UID malicious object
- [x] TOCTOU 제거

#### 코드 증거 기록

- 파일 경로: `src/client.c`
- symbol 또는 함수: `bind_client_socket`, `cleanup_response_socket`
- 확인한 state fields: `g_response_socket`, `g_client_path`, `g_client_bound`
- caller → callee: client endpoint initialization → `bind` → set bound; any exit/error → `cleanup_response_socket`
- 핵심 branch 또는 mutation 순서: path 계산/possible stale handling → socket create/configure → bind → success 직후 `g_client_bound = 1`; cleanup은 fd가 있으면 close하고, bound flag와 nonempty path가 모두 true일 때만 unlink한 뒤 flag/path를 reset합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: global bound flag와 conditional unlink가 추가됐고 나머지 socket setup은 유지됩니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `622d80020fb2`, `src/client.c`, `bind_client_socket`. unlink authority가 실제 bind 성공 뒤에만 생기는 정확한 지점입니다.

```c
if (bind(g_response_socket, (struct sockaddr *)&address,
        sizeof(address)) == -1)
    return (-1);
g_client_bound = 1;
return (0);
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`ffd3647a1518`에서 stale socket replacement와 protected file preservation을 검증합니다.
### 5. `ffd3647a1518` — test(runtime): stale 응답 endpoint 처리 검증

- **중요도:** A
- **태그:** TEST, ENDPOINT, RISK
- **개발 흐름에서의 역할:** real PID-derived paths에 stale client/server sockets, regular files, unrelated live processes를 만들어 endpoint trust와 cleanup policy를 검증합니다.

#### 원문에서 확정된 맥락

same-UID stale socket은 remove/replace되지만 non-socket은 preserved되고 startup은 실패합니다. private directory permissions와 valid server endpoint 없는 PID rejection도 확인합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] real child PID와 path synchronization
- [x] stale Unix socket creation
- [x] expected path의 regular file preservation
- [x] client/server stale replacement
- [x] runtime directory owner/mode assertion
- [x] unrelated live PID without server endpoint rejection
- [x] 모든 child/socket/file cleanup

#### 비교 기준

`2c37cb592d05`, `25780b881ee8`, `32390dcdfc1b`, `622d80020fb2`의 runtime dir/path/stale/bind/conditional cleanup branches를 scenario별로 연결합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: path, stale policy, client/server 수명, client bound flag는 code에 있었지만 real filesystem type/owner/PID combinations에서 destructive cleanup이 없는지 검증되지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: mock path 문자열만으로는 Unix socket inode, regular file, child PID timing, stale entry replacement, runtime directory permission과 cleanup side effect를 함께 재현하지 못합니다.
- 변경된 decision과 state mutation 순서: helper child/exec와 real AF_UNIX sockets/files를 사용해 target PID-derived paths를 만들고 client/server startup과 cleanup 결과를 shell에서 검사했습니다. private runtime dir 준비/검사 → child PID 확보 → 해당 role path에 stale socket 또는 regular file 생성 → real client/server 실행 → status/output/object existence 검사 → child/socket/file cleanup입니다. unrelated live PID에는 server path가 없어 client가 protocol 시작 전에 실패합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: same-UID socket entry는 unlink 후 bind가 성공하고 exit에서 새 path도 제거됩니다. regular file은 non-socket이라 보존되고 process는 clean failure합니다. valid PID만 있고 expected server endpoint가 없으면 client가 target을 protocol server로 인정하지 않습니다.
- 후속 commit이 강화하거나 교체하는 부분: `4e1c84bfacfc`가 path/object authority와 별개인 descriptor representation boundary를 추가합니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** replaceable same-UID stale socket만 제거하고 non-socket/unowned entry는 보존합니다.
- **재현하는 failure 또는 boundary:** predictable path를 근거로 regular file 또는 다른 endpoint를 destructive cleanup하는 상황
- **사용한 test technique:** real filesystem objects, PID-derived names, child exec, live processes
- **분류:** runtime/filesystem integration regression
- **failure 주입 또는 process orchestration 시작 지점:** helper가 실제 PID를 유지한 채 target role path에 socket 또는 regular file을 배치합니다.
- **production code에 진입하는 최초 호출:** client/server initialization의 `mt_response_path`와 `remove_stale_socket`입니다.
- **핵심 assertion과 관측값:** stale socket replacement success, regular file existence 유지와 startup failure, runtime dir 0700/current UID, missing server endpoint rejection, cleanup 후 path 부재를 검사합니다.
- **증명하는 것:** stale replacement<br>regular-file preservation<br>private permissions<br>missing endpoint rejection<br>cleanup symmetry
- **증명하지 않는 것:** same-UID cryptographic authentication<br>high descriptor behavior<br>모든 crash/rename race<br>cross-host portability
- **후속 변경에서 막아야 할 구체적인 회귀:** computed name만으로 object를 삭제하거나 non-server PID를 valid peer로 인정하는 변경을 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] same-UID stale socket replacement
- [x] regular-file/unowned object preservation
- [x] private runtime directory mode/owner
- [x] PID만 존재하는 non-server rejection
- [x] normal cleanup symmetry

**아직 보장하지 않는 것**

- [x] same-UID authentication
- [x] high-fd guard
- [x] 모든 TOCTOU/crash window
- [x] 다른 UID namespace scenario 전체

#### 코드 증거 기록

- 파일 경로: `tests/protocol_regressions.sh`, `tests/stale_exec.c`, `src/response_channel.c`, `src/client.c`, `src/server.c`
- symbol 또는 함수: `stale_exec helper`, `remove_stale_socket`, `bind_client_socket`, `prepare_response_channel`
- 확인한 state fields: `runtime path object type/uid/mode`, `child PID`, `bound flags`
- caller → callee: test helper/filesystem setup → real client/server endpoint setup → production stale/type/bind/cleanup branches
- 핵심 branch 또는 mutation 순서: private runtime dir 준비/검사 → child PID 확보 → 해당 role path에 stale socket 또는 regular file 생성 → real client/server 실행 → status/output/object existence 검사 → child/socket/file cleanup입니다. unrelated live PID에는 server path가 없어 client가 protocol 시작 전에 실패합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: runtime integration helper와 stale/protected object scenarios가 test target에 추가됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`4e1c84bfacfc`에서 endpoint descriptor의 polling representability를 추가합니다.
### 6. `4e1c84bfacfc` — fix(runtime): select 범위를 벗어난 descriptor 거부

- **중요도:** A
- **태그:** EDGE, PRACTICAL, RISK
- **개발 흐름에서의 역할:** client response socket, server response socket, self-pipe read fd가 `FD_SETSIZE` 이상이면 initialization에서 거부합니다.

#### 원문에서 확정된 맥락

`fd_set`은 fixed-size bit representation이므로 large descriptor의 `FD_SET`은 undefined 동작을 일으킬 수 있습니다. valid fd와 selected polling API representability를 구분합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] client response socket guard
- [x] server response socket guard
- [x] self-pipe read fd guard
- [x] 모든 `FD_SET`/`pselect` 이전 실행
- [x] guard failure의 existing 정리 과정
- [x] write-only pipe end가 guard 대상이 아닌 이유

#### 비교 기준

endpoint setup commits의 `FD_SET` 호출 지점을 역추적해 실제 read set에 들어가는 세 descriptor만 creation 단계에서 guard합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: socket/pipe open 성공만 확인했고 later `FD_SET`이 그 integer를 표현할 수 있는지는 검사하지 않았습니다. inherited descriptor pressure가 새 fd를 높은 번호로 밀 수 있습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: `fd >= FD_SETSIZE`인 valid descriptor를 `FD_SET`에 전달하면 fixed bitset 범위 밖 memory를 접근할 수 있습니다. protocol error가 아니라 local undefined behavior입니다.
- 변경된 decision과 state mutation 순서: resource 생성 직후, flags/bind/`FD_SET`보다 먼저 client response socket, server event-pipe read end, server response socket을 각각 `FD_SETSIZE`와 비교해 실패하도록 했습니다. client: `socket` → `fd < FD_SETSIZE` → flags → bind. server: `pipe` → read fd range check → flags; response `socket` → range check → flags/bind. failure는 기존 정리 과정이 열린 fd와 bound path state를 unwind합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: fd가 음수이면 ordinary allocation failure, 범위 이상이면 controlled initialization failure입니다. write-only self-pipe end는 `FD_SET`에 들어가지 않으므로 range guard 대상이 아닙니다.
- 후속 commit이 강화하거나 교체하는 부분: `1de95310195d`가 `/dev/null`을 실제로 반복 open한 뒤 exec하여 client/server가 polling 전에 안전하게 실패하는지 검증합니다.

#### Fix 연결 기록

| 단계 | 원자료에서 확인된 내용 | 해당 SHA의 코드 근거 |
| --- | --- | --- |
| 기존 가정 | open에 성공한 descriptor는 `FD_SET`에 넣을 수 있습니다. | 직전 setup은 `fd == -1`만 확인한 뒤 later `FD_SET`을 수행했습니다. |
| 실제 failure 또는 위험 | `fd >= FD_SETSIZE`면 fixed object 밖에 bit를 쓸 수 있습니다. | client/server event loops가 `fd_set`과 `pselect`를 사용합니다. |
| root cause | OS fd validity와 polling representation range를 혼동했습니다. | 새 branch는 allocation success와 별개로 numeric upper bound를 검사합니다. |
| 수정 불변 조건/decision | polling descriptor는 resource creation 단계에서 range-check합니다. | client socket, server pipe read end, server socket의 three guards |
| regression | `1de95310195d`가 real inherited descriptor table로 경계를 만듭니다. | `tests/high_fd_exec.c`, `tests/high_fd.sh` |

- 변경 전 failure를 재현하거나 추론할 수 있는 입력/상태: 직전 상태와 해당 분기를 직접 비교했습니다.
- root cause가 드러나는 field 또는 call order: client: `socket` → `fd < FD_SETSIZE` → flags → bind. server: `pipe` → read fd range check → flags; response `socket` → range check → flags/bind. failure는 기존 정리 과정이 열린 fd와 bound path state를 unwind합니다.
- 수정된 invariant를 고정하는 후속 regression test: `1de95310195d`의 environmental boundary integration regression입니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] out-of-range fd controlled startup failure
- [x] out-of-bounds `FD_SET` prevention
- [x] valid fd와 polling-representable fd 구분

**아직 보장하지 않는 것**

- [x] dynamic polling support
- [x] descriptor pressure 해소
- [x] range 내부 exhaustion
- [x] poll/epoll portability

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/server.c`
- symbol 또는 함수: `bind_client_socket`, `prepare_response_channel`
- 확인한 state fields: `g_response_socket`, `g_event_pipe[0]`
- caller → callee: resource allocation → immediate range guard → flags/bind → later `FD_SET`/`pselect`
- 핵심 branch 또는 mutation 순서: client: `socket` → `fd < FD_SETSIZE` → flags → bind. server: `pipe` → read fd range check → flags; response `socket` → range check → flags/bind. failure는 기존 정리 과정이 열린 fd와 bound path state를 unwind합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: 세 allocation condition에 `fd >= FD_SETSIZE` branch가 추가됐습니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `4e1c84bfacfc`, client/server socket setup에 동일하게 추가된 guard입니다. open success와 `fd_set` representability가 별도 조건임을 보여 줍니다.

```c
g_response_socket = socket(AF_UNIX, SOCK_DGRAM, 0);
if (g_response_socket == -1
    || g_response_socket >= FD_SETSIZE
    || set_nonblocking_close_on_exec(g_response_socket) == -1)
    return (-1);
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`1de95310195d`에서 real descriptors로 client/server guards를 실행합니다.
### 7. `1de95310195d` — test(runtime): 높은 descriptor 번호의 안전한 실패 검증

- **중요도:** A
- **태그:** TEST, EDGE, RISK
- **개발 흐름에서의 역할:** wrapper가 `/dev/null`을 반복 open해 inherited descriptor table을 `FD_SETSIZE` boundary까지 높인 뒤 real client/server를 `exec`합니다.

#### 원문에서 확정된 맥락

두 executable은 `pselect`나 protocol traffic 전에 normal diagnostic으로 실패해야 합니다. high-fd client failure는 independently running server에 영향을 주지 않아야 합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `/dev/null` open loop와 target boundary
- [x] descriptors를 close하지 않고 `exec`
- [x] real client/server guard cases
- [x] server no-PID-publication assertion
- [x] client no protocol traffic/normal failure
- [x] independent server unaffected assertion
- [x] wrapper/child cleanup

#### 비교 기준

`4e1c84bfacfc`의 three guards 중 allocation order에 따라 client socket 또는 server self-pipe read/response socket이 boundary를 넘는 case를 매핑합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: numeric guards는 code에 있었지만 synthetic integer가 아니라 kernel이 실제 할당한 high fd에서 client/server cleanup과 peer isolation까지 검증되지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 직접 fd 값을 주입하는 단위 테스트는 open table inheritance, exec, new pipe/socket allocation order, real cleanup 동작을 재현하지 못합니다.
- 변경된 decision과 state mutation 순서: `tests/high_fd_exec.c`가 `/dev/null`을 반복 open해 descriptors를 유지한 채 target executable을 `exec`하고, shell이 client와 server cases를 별도로 관측합니다. wrapper starts → low descriptors부터 `/dev/null`을 open해 next allocation을 boundary로 이동 → target client/server exec → production socket/pipe creation → range guard failure → normal diagnostic/status/cleanup. client case에서는 별도 정상 server를 유지해 그 process가 영향받지 않았는지 확인합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: server는 response pipe/socket 중 guard가 실패해 PID를 publish하지 않고 종료합니다. client는 response socket guard에서 protocol datagram/signal을 보내기 전에 실패합니다. wrapper 또는 target setup failure도 nonzero status로 수집됩니다.
- 후속 commit이 강화하거나 교체하는 부분: 이 commit이 Thread final regression입니다. polling API를 바꾸지 않는 한 range guard와 test가 함께 유지돼야 합니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** `fd_set`에 들어가는 descriptors는 모두 `FD_SETSIZE` 미만입니다.
- **재현하는 failure 또는 boundary:** inherited descriptor pressure로 new socket/pipe가 unrepresentable range에 할당되는 상황
- **사용한 test technique:** real `/dev/null` allocations retained across `exec`
- **분류:** environmental boundary integration regression
- **failure 주입 또는 process orchestration 시작 지점:** wrapper가 `FD_SETSIZE` 직전까지 real descriptors를 연속 open합니다.
- **production code에 진입하는 최초 호출:** exec된 client/server가 response socket 또는 event pipe를 생성하면서 production guard에 진입합니다.
- **핵심 assertion과 관측값:** nonzero normal failure, expected diagnostics, server PID 미출력, client protocol traffic 부재, independent server 생존과 cleanup을 검사합니다.
- **증명하는 것:** real high-fd allocation<br>pre-`FD_SET` failure<br>client/server guards<br>unrelated server unaffected
- **증명하지 않는 것:** 모든 descriptor leak 부재<br>poll/epoll behavior<br>range-inside resource exhaustion<br>모든 launcher fd layout
- **후속 변경에서 막아야 할 구체적인 회귀:** guard를 `FD_SET` 뒤로 옮기거나 일부 polling fd를 빠뜨리는 변경을 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] real high-fd condition 생성
- [x] polling 전에 controlled failure
- [x] client/server range guards
- [x] high-fd client failure의 peer isolation

**아직 보장하지 않는 것**

- [x] descriptor leak 전수 검증
- [x] dynamic polling support
- [x] range 안에서의 exhaustion
- [x] 다른 OS의 `FD_SETSIZE` 설정 전체

#### 코드 증거 기록

- 파일 경로: `tests/high_fd_exec.c`, `tests/high_fd.sh`, `Makefile`, `src/client.c`, `src/server.c`
- symbol 또는 함수: `high_fd_exec main`, `bind_client_socket`, `prepare_response_channel`
- 확인한 state fields: `inherited open descriptor table`, `target mode`, `child/server PID`
- caller → callee: shell → `high_fd_exec` open loop → `exec` real binary → production allocation/range guard
- 핵심 branch 또는 mutation 순서: wrapper starts → low descriptors부터 `/dev/null`을 open해 next allocation을 boundary로 이동 → target client/server exec → production socket/pipe creation → range guard failure → normal diagnostic/status/cleanup. client case에서는 별도 정상 server를 유지해 그 process가 영향받지 않았는지 확인합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: high-fd wrapper binary, shell orchestration와 test target이 추가됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

이 commit이 Thread final regression입니다.

## 6. 불변 조건 ledger

### 원자료에서 확인된 핵심 불변 조건

- runtime directory는 current UID 소유이며 group/other access가 없는 private namespace입니다.
- role/PID path는 검증과 length check를 통과한 경우에만 사용합니다.
- process는 실제 bind한 path 또는 policy상 replaceable same-UID stale socket만 unlink합니다.
- regular files와 unowned entries는 보존하고 startup은 cleanly 실패합니다.
- `fd_set`에 넣는 descriptor는 모두 `FD_SETSIZE` 미만입니다.

### 시간에 따른 변화 기록

| Commit | 원자료에서 확인된 변화 | 실제 state/condition | code evidence | 상태: 도입·강화·부족·복구·검증 |
| --- | --- | --- | --- | --- |
| `2c37cb592d05` | per-UID private runtime directory와 role/PID-derived Unix socket path helper를 정의합니다. | validated per-UID directory와 role/PID-derived path가 생기지만 path knowledge는 resource ownership이 아닙니다. | `src/response_channel.c: mt_runtime_dir`, `mt_response_path` | 도입 |
| `25780b881ee8` | client가 nonblocking, close-on-exec datagram socket을 PID-derived path에 bind하고 invocation lifetime에 맞춰 cleanup합니다. | descriptor와 computed path를 client lifetime state로 보관하지만 unlink authority가 bind success와 아직 분리되지 않았습니다. | `src/client.c: bind_client_socket`, `cleanup_response_socket` | 도입·부족 |
| `32390dcdfc1b` | server가 long-lived nonblocking, close-on-exec datagram endpoint를 server path에 bind하고 normal exit와 rollback에서 cleanup합니다. | server는 successful bind를 `g_server_bound`로 기록해 close/unlink를 acquisition과 대칭시킵니다. | `src/server.c: prepare_response_channel`, `cleanup_server` | 도입 |
| `622d80020fb2` | client cleanup이 response path를 실제 `bind`한 경우에만 unlink하도록 bound ownership flag를 사용합니다. | computed path와 actual bound ownership을 `g_client_bound`로 분리해 conditional unlink합니다. | `src/client.c: g_client_bound`, bind/cleanup branches | 수정 |
| `ffd3647a1518` | real PID-derived paths에 stale client/server sockets, regular files, unrelated live processes를 만들어 endpoint trust와 cleanup policy를 검증합니다. | 실제 filesystem object와 process identity로 stale replacement와 protected-entry preservation을 검증합니다. | `tests/stale_exec.c`, protocol regression shell scenarios | 검증 |
| `4e1c84bfacfc` | client response socket, server response socket, self-pipe read fd가 `FD_SETSIZE` 이상이면 initialization에서 거부합니다. | `fd_set`에 등록될 resource는 allocation 직후 `fd < FD_SETSIZE`를 만족해야 합니다. | `src/client.c: bind_client_socket`, `src/server.c: prepare_response_channel` | 수정 |
| `1de95310195d` | wrapper가 `/dev/null`을 반복 open해 inherited descriptor table을 `FD_SETSIZE` boundary까지 높인 뒤 real client/server를 `exec`합니다. | real inherited descriptor pressure에서 client/server가 `FD_SET` 전에 controlled failure하는지 고정합니다. | `tests/high_fd_exec.c`, `tests/high_fd.sh` | 검증 |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 또는 상태 | 실제 failure/위험 | Fix 또는 전환 commit | 수정된 decision/불변 조건 | Test 또는 후속 검증 | 학습자 code evidence |
| --- | --- | --- | --- | --- | --- |
| PID-derived path를 계산하면 cleanup authority가 있다고 간주 | bind failure 뒤 existing/unowned entry를 unlink할 수 있음 | `622d80020fb2` | successful bind를 ownership flag로 기록하고 그때만 unlink | `ffd3647a1518` | regular file preservation/stale socket replacement |
| descriptor open success면 `FD_SET` 가능 | `fd >= FD_SETSIZE`에서 fixed bitset 밖 memory write 가능 | `4e1c84bfacfc` | polling descriptor를 creation 단계에서 range-check | `1de95310195d` | real inherited high-fd wrapper와 pre-poll failure |

전용 test commit이 없는 연결에는 존재하지 않는 test를 만들어 적지 않았습니다.

## 8. 소유권 / state / responsibility 변화

| 단계 | state 또는 responsibility owner | transition | 당시 한계 또는 다음 변화 | 실제 symbol/field |
| --- | --- | --- | --- | --- |
| path helper | runtime namespace policy | role/PID → validated path | 계산만으로 소유권 없음 | `mt_runtime_dir`, `mt_response_path` |
| client/server setup | descriptor + bind state | open/configure/bind 단계별 acquisition | partial unwind 필요 | response fd/path/bound flags |
| client fix | `g_client_bound` | bind success 때 unlink authority 획득 | computed path와 소유권 분리 | bind success assignment, conditional cleanup |
| polling setup | `fd_set` representation | range 안 descriptor만 registration | 범위 밖 startup failure | client/server response fd, self-pipe read fd |

## 9. 개발 흐름의 최종 상태

원자료에서 확인된 최종 조건:

- response endpoints는 private per-UID runtime directory의 validated PID-derived paths를 사용합니다.
- same-UID stale socket은 policy에 따라 replace하지만 non-socket과 unowned entry는 삭제하지 않습니다.
- cleanup은 actual resource-acquisition state와 대칭입니다.
- response sockets와 self-pipe read fd가 `FD_SETSIZE` 범위를 넘으면 polling 전에 실패합니다.

학습자 기록:

- 최종 state fields와 owner: client는 response descriptor/path/`g_client_bound`, server는 response descriptor/path/`g_server_bound`와 event pipe를 소유합니다. runtime helpers는 path만 계산하며 소유권을 부여하지 않습니다.
- 정상 transition 순서: private runtime directory 검증 → role/PID path 생성 → stale object type/UID 판정 → socket/pipe 생성과 flags → polling fd range check → bind 성공과 bound flag → event loop → close와 conditional unlink입니다.
- 실패 시 중단·reset·cleanup 순서: validation/stale/flags/range/bind 단계의 failure는 이미 획득한 descriptors를 닫고, successful bind를 기록한 path만 unlink합니다. protected/unowned object는 보존합니다.
- 최종 상태가 보장하지 않는 것: same-UID malicious peer authentication, TOCTOU 완전 제거, dynamic polling, descriptor pressure 자체의 해소는 제공하지 않습니다.
- 이 개발 흐름을 한 문단으로 설명한 최종 서술: 이 개발 흐름은 Unix socket path를 단순 문자열이 아니라 filesystem resource로 취급합니다. private per-UID directory와 role/PID 검증을 정의하고, client/server의 socket·path 수명을 단계별로 관리합니다. client fix는 computed name과 successful bind 소유권을 분리하며, 마지막 guard는 valid descriptor도 `fd_set` 범위 밖이면 사용할 수 없음을 controlled startup failure로 바꿉니다.

## 10. 최종 architecture 또는 실행 순서 정리

- [x] runtime directory create 또는 existing directory validation
- [x] role/PID path derivation과 length check
- [x] socket create와 nonblocking/CLOEXEC setup
- [x] existing path type/owner/staleness 판정
- [x] bind success와 소유권 flag record
- [x] `FD_SETSIZE` check 뒤 `FD_SET`/`pselect`
- [x] normal exit 또는 rollback에서 close와 conditional unlink

```text
endpoint setup
    -> mt_runtime_dir: mkdir 0700 / validate dir+uid+mode
    -> mt_response_path: role+positive PID+length
    -> inspect existing path
        -> same-UID socket: remove as stale
        -> non-socket/other owner: fail and preserve
    -> socket/pipe allocation
    -> fd < FD_SETSIZE for every later fd_set member
    -> O_NONBLOCK / FD_CLOEXEC
    -> bind -> bound flag = 1
    -> pselect registration/use
cleanup/rollback
    -> close owned descriptors
    -> unlink only if successful bind ownership is recorded

```

- 실제 함수·파일을 반영한 완성 흐름: `src/response_channel.c`의 namespace helpers, `src/client.c: bind_client_socket/cleanup_response_socket`, `src/server.c: prepare_response_channel/cleanup_server`가 전체 lifecycle을 구성합니다.
- asynchronous boundary: endpoint acquisition/cleanup은 normal 문맥에서만 일어나며 시그널 처리 함수는 path·socket cleanup을 수행하지 않습니다.
- externally visible commit point: filesystem endpoint 소유권은 successful `bind` 직후 bound flag를 설정할 때 생깁니다. path 계산이나 socket descriptor creation만으로는 unlink authority가 없습니다.
- cleanup owner: 각 process의 registered cleanup이 자신이 연 descriptor를 닫고 실제 bind한 endpoint만 unlink합니다.

## 11. 학습 완료 자가 점검

- [x] commit map의 7개 SHA를 source 순서대로 모두 설명할 수 있습니다.
- [x] 각 code excerpt에 SHA, path, symbol, 선택 이유가 기록돼 있습니다.
- [x] 최종 HEAD 코드를 historical SHA의 증거로 사용한 곳이 없습니다.
- [x] 정상 경로와 실패 처리를 state mutation 순서로 설명할 수 있습니다.
- [x] source 확정 불변 조건과 직접 확인한 code evidence를 구분했습니다.
- [x] test commit의 불변 조건, failure, technique, production path, proves/not-proves를 기록했습니다.
- [x] Thread final state를 함수와 state field 수준으로 설명할 수 있습니다.
- [ ] 해당 SHA의 test를 로컬에서 직접 실행했습니다. — 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

### 이 Thread와 직접 연결된 Major Engineering Difficulties

- socket path가 crash 뒤 남을 수 있어 stale recovery와 destructive cleanup을 구분해야 합니다.
- same-UID integrity boundary는 same-UID peer authentication이 아닙니다.
- inherited descriptor pressure가 ordinary socket/pipe allocation을 polling range 밖으로 밀 수 있습니다.

---

# 악의적이거나 오래된 입력에서도 제한을 지키는 응답 대조

> 완성형 해설서가 아닙니다. 아래 확정 사항을 기준으로 각 commit SHA의 실제 코드와 diff를 읽고 기록란을 채웁니다.

## 1. 개발 흐름 목표

Unix datagram response가 현재 client와 현재 transition에 속한다는 판단 규칙을 복원합니다. exact frame size, source endpoint, PID, magic, kind, nonce 또는 sequence, status를 함께 검증하고, stale·forged·oversized·uncorrelated traffic을 무시하면서도 원래의 monotonic deadline을 늘리지 않는 bounded wait를 실제 코드와 adversarial tests로 확인합니다.

### Significance

control channel의 응답은 단순히 도착했다는 이유로 state를 전진시킬 수 없습니다. 한 field만 맞는 datagram도 현재 `ACQUIRE` 또는 bit transition을 증명하지 못합니다. production 검증은 모든 identity field와 source를 결합하고, ignored traffic은 기존 deadline budget을 소비할 뿐 새 budget을 만들지 않아야 합니다. 이 개발 흐름은 wire representation에서 시작해 READY와 sequence ACK validation, forged response, oversized frame, invalid flood 검증으로 그 조건을 고정합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문

- `t_mt_request`와 `t_mt_response`의 어떤 fields가 peer와 transition identity를 표현하는가?
- READY와 bit ACK는 token을 각각 nonce와 sequence로 어떻게 해석하는가?
- response source path와 record 내부 server PID를 왜 동시에 검증하는가?
- exact datagram size check는 valid prefix 뒤 trailing byte를 어디서 거부하는가?
- invalid frame을 받은 뒤 어떤 state가 그대로 유지되어야 하는가?
- `CLOCK_MONOTONIC` absolute deadline은 언제 한 번 계산되고 반복 receive에서 어떻게 재사용됩니까?
- forged-source test와 invalid-flood test는 서로 다른 어떤 failure를 고정하는가?

## 3. 완료 기준

- [x] request/response record의 field, 의미, READY/ACK 사용 위치를 표로 정리했습니다.
- [x] READY acceptance predicate를 source address부터 status까지 실제 condition 순서로 복원했습니다.
- [x] sequence ACK acceptance predicate와 sequence advance 지점을 실제 코드로 확인했습니다.
- [x] discarded frame 뒤 nonce, sequence, bit cursor, absolute deadline이 변하지 않음을 확인했습니다.
- [x] forged-source, wrong-token, bad-magic, wrong-PID, oversized, invalid-flood cases를 production branch에 연결했습니다.
- [x] 같은 UID의 predictable path 검증이 cryptographic peer authentication을 뜻하지 않음을 구분했습니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `ebed06775b92` | feat(protocol): 응답 메시지 wire 형식 정의 | A | ARCH, RESPONSE | request와 특정 bit transition을 식별할 수 있는 request/response fields를 정의합니다. |
| 2 | `f8e8444c5ded` | feat(client): READY 응답을 출처와 nonce로 상관 검증 | A | RESPONSE, RISK, INTEGRATION | READY에 exact size, source, PID, kind, nonce, status와 absolute readiness deadline을 적용합니다. |
| 3 | `d3eacbbfeadc` | feat(client): 비트 ACK를 sequence로 상관 검증 | A | RESPONSE, RISK | 현재 sequence와 정확히 일치하는 datagram ACK만 bit success로 인정합니다. |
| 4 | `b361ef9745ff` | test(protocol): 응답 출처와 token 검증 | A | TEST, RESPONSE, RISK | valid READY와 first bit ACK 전에 forged-source와 mismatched-field responses를 주입해 conjunctive validation을 검증합니다. |
| 5 | `1ed2acbaa353` | test(response): oversized 응답과 invalid flood 검증 | A | TEST, RESPONSE, RISK | response record보다 한 byte 큰 datagram과 sustained wrong-token traffic을 이용해 exact framing과 absolute deadline을 검증합니다. |

확인 원칙:

- 각 항목은 해당 SHA의 tree를 기준으로 읽었습니다.
- 변경 전 상태는 해당 SHA의 parent 또는 지정된 이전 관련 SHA에서 확인했습니다.
- 같은 commit이 다른 Thread에 다시 등장해도 이 개발 흐름의 질문으로 별도 기록했습니다.
- runtime test는 실행하지 않았으며, 실행 결과처럼 표현하지 않았습니다.

## 5. 커밋별 학습 기록

### 1. `ebed06775b92` — feat(protocol): 응답 메시지 wire 형식 정의

- **중요도:** A
- **태그:** ARCH, RESPONSE
- **개발 흐름에서의 역할:** request와 특정 bit transition을 식별할 수 있는 request/response fields를 정의합니다.

#### 원문에서 확정된 맥락

`ACQUIRE`, `READY`, acknowledgement traffic을 구분하는 magic과 kind, client/server PID, nonce 또는 token, status를 포함하는 fixed records를 shared header에 추가합니다. records는 host-local in-memory ABI이며 portable serialized protocol이 아닙니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `t_mt_request` field order/type
- [x] `t_mt_response` field order/type
- [x] request/response magic and kind constants
- [x] client/server PID field 의미
- [x] nonce와 token 대응
- [x] OK/BUSY status
- [x] raw structure size가 framing 기준
- [x] serialization/version/byte-order conversion 부재

#### 비교 기준

직전 signal-only ACK에는 kind/token/status/PID field가 없습니다. 이 개발 흐름에서는 각 field가 후속 stale/forged/oversized rejection의 어느 조건에 쓰이는지 연결합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: generic ACK/NACK signal은 sender/transition identity를 payload로 전달하지 못했고, request와 response의 exact frame size를 정의할 shared record도 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 도착한 signal 하나만으로는 어느 acquisition 또는 어느 bit에 대한 응답인지 구분할 수 없습니다. stale/unrelated response를 배제할 field와 framing 기준이 필요합니다.
- 변경된 decision과 state mutation 순서: shared header에 `t_mt_request {magic, kind, nonce, client_pid}`와 `t_mt_response {magic, kind, token, status, server_pid}`를 추가하고 request/response magic, kind, status constants를 정의했습니다. 이 commit은 representation만 정의합니다. 후속 sender/receiver가 raw `sizeof(struct)` datagram을 만들고 exact size, source와 fields를 검사합니다. READY는 request nonce를 response token으로 echo하고 ACK는 current sequence를 token으로 사용합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: schema 자체에는 runtime reject branch가 없습니다. fixed-width integers를 일부 사용하지만 `pid_t`, padding, native byte order를 그대로 포함하므로 다른 ABI/host 간 portable serialization은 아닙니다.
- 후속 commit이 강화하거나 교체하는 부분: `f8e8444c5ded`가 READY predicate와 absolute deadline을, `d3eacbbfeadc`가 per-bit sequence ACK predicate를 구현합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] strong correlation을 표현할 wire schema
- [x] exact-frame 검증의 기준 record size
- [x] READY와 ACK kind 구분 능력

**아직 보장하지 않는 것**

- [x] runtime acceptance predicate
- [x] source endpoint validation
- [x] absolute deadline behavior
- [x] portable ABI 또는 cryptographic authentication

#### 코드 증거 기록

- 파일 경로: `include/minitalk.h`
- symbol 또는 함수: `t_mt_request`, `t_mt_response`, `MT_RESPONSE_MAGIC`, `MT_REQUEST_ACQUIRE`, `MT_RESPONSE_READY`, `MT_RESPONSE_ACK`
- 확인한 state fields: `request nonce`, `response token`, `status`, `client_pid`, `server_pid`
- caller → callee: shared definitions → 후속 client/server `sendto`/`recvfrom` callers
- 핵심 branch 또는 mutation 순서: 이 commit은 representation만 정의합니다. 후속 sender/receiver가 raw `sizeof(struct)` datagram을 만들고 exact size, source와 fields를 검사합니다. READY는 request nonce를 response token으로 echo하고 ACK는 current sequence를 token으로 사용합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: shared constants와 두 raw record type만 추가됐고 actual socket 검증은 아직 없습니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `ebed06775b92`, `include/minitalk.h`. acquisition identity와 response identity를 표현하는 전체 field set을 보여 주는 최소 정의입니다.

```c
typedef struct s_mt_request
{
    uint32_t magic;
    uint32_t kind;
    uint32_t nonce;
    pid_t    client_pid;
} t_mt_request;

typedef struct s_mt_response
{
    uint32_t magic;
    uint32_t kind;
    uint32_t token;
    int32_t  status;
    pid_t    server_pid;
} t_mt_response;
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`f8e8444c5ded`가 READY validation에 이 fields를 실제로 적용합니다.
### 2. `f8e8444c5ded` — feat(client): READY 응답을 출처와 nonce로 상관 검증

- **중요도:** A
- **태그:** RESPONSE, RISK, INTEGRATION
- **개발 흐름에서의 역할:** READY에 exact size, source, PID, kind, nonce, status와 absolute readiness deadline을 적용합니다.

#### 원문에서 확정된 맥락

client는 nonzero nonce를 생성해 `ACQUIRE`를 보내고 expected server endpoint에서 current request와 일치하는 `READY`만 수락합니다. unrelated 또는 malformed datagrams는 무시하며 one absolute `CLOCK_MONOTONIC` deadline을 유지합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `/dev/urandom` partial read/EINTR/close와 zero nonce 처리
- [x] ACQUIRE fields와 expected destination
- [x] absolute `CLOCK_MONOTONIC` deadline one-time calculation
- [x] `sizeof(response)+1` receive buffer와 exact size check
- [x] expected source path check
- [x] magic/server PID/READY/token/status conjunction
- [x] invalid frame 뒤 same deadline 유지
- [x] valid READY 뒤에만 payload path 진입

#### 비교 기준

`ebed06775b92`의 fields를 runtime predicate에 매핑합니다. Session Thread에서는 acquisition ownership으로, 여기서는 client acceptance/deadline state로 읽습니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: server ACQUIRE/READY path는 생겼지만 production client는 session request를 보내고 matching READY를 기다리는 correlated establishment path가 없거나 완전하지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: 같은 local datagram socket에 stale, malformed, wrong peer, wrong request의 frame이 도착할 수 있습니다. invalid frame마다 relative timeout을 다시 시작하면 wait가 무기한 연장됩니다.
- 변경된 decision과 state mutation 순서: `/dev/urandom`에서 nonzero nonce를 만들고 expected server path와 하나의 monotonic absolute deadline을 확정한 뒤 exact-size/source/magic/PID/kind/token/status conjunction만 READY success로 수락했습니다. client endpoint bind → target server path derivation/validation → nonce generation → `ACQUIRE{magic,kind,nonce,client_pid}` send → `clock_gettime(CLOCK_MONOTONIC)`으로 deadline 한 번 계산 → remaining budget으로 `pselect`/`recvfrom` 반복 → exact READY match 후 payload send path입니다.
- 정상 경로와 failure 경로가 갈라지는 조건: random open/read/close, path, send, clock, pselect/recv fatal error는 send error입니다. EAGAIN/EINTR 또는 invalid candidate는 같은 deadline으로 계속합니다. deadline 도달은 timeout, status BUSY는 rejected, full valid READY만 established입니다.
- 후속 commit이 강화하거나 교체하는 부분: `d3eacbbfeadc`가 같은 structure를 per-bit sequence ACK에 적용하고 `b361ef9745ff`/`1ed2acbaa353`이 forged/oversized/flood inputs를 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] nonce-correlated READY acceptance
- [x] malformed/unrelated response rejection
- [x] invalid traffic 아래 bounded readiness wait
- [x] BUSY와 transport timeout 구분

**아직 보장하지 않는 것**

- [x] per-bit sequence ACK correlation
- [x] adversarial test evidence at this SHA
- [x] same-UID cryptographic authentication
- [x] retransmission

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/response_channel.c`, `include/minitalk.h`
- symbol 또는 함수: `generate_nonce`, `time_until`, `valid_source`, `read_response`, `wait_for_response`, `acquire_session`
- 확인한 state fields: `nonce`, `deadline`, `server_path`, `g_response_socket`
- caller → callee: client `main` → endpoint bind → `acquire_session` → `sendto` → response wait/read/validate
- 핵심 branch 또는 mutation 순서: client endpoint bind → target server path derivation/validation → nonce generation → `ACQUIRE{magic,kind,nonce,client_pid}` send → `clock_gettime(CLOCK_MONOTONIC)`으로 deadline 한 번 계산 → remaining budget으로 `pselect`/`recvfrom` 반복 → exact READY match 후 payload send path입니다.
- parent 또는 이전 관련 SHA와의 diff 요약: client에 nonce generation, ACQUIRE send, READY receive validation, absolute-deadline wait와 status-specific return이 추가됐습니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `f8e8444c5ded`, `src/client.c`, response candidate validation. exact size, source와 모든 identity fields가 conjunction임을 보여 줍니다.

```c
if (size != (ssize_t)sizeof(response))
    return (0);
memcpy(&response, payload, sizeof(response));
if (!valid_source(&source, server_path)
    || response.magic != MT_RESPONSE_MAGIC
    || response.server_pid != server_pid || response.kind != kind
    || response.token != token || response.status != MT_RESPONSE_OK)
    return (0);
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`d3eacbbfeadc`가 같은 규칙을 current bit sequence에 적용합니다.
### 3. `d3eacbbfeadc` — feat(client): 비트 ACK를 sequence로 상관 검증

- **중요도:** A
- **태그:** RESPONSE, RISK
- **개발 흐름에서의 역할:** 현재 sequence와 정확히 일치하는 datagram ACK만 bit success로 인정합니다.

#### 원문에서 확정된 맥락

client는 signal을 보내기 전에 expected server endpoint와 bit-specific monotonic deadline을 확정하고, source path, server PID, ACK kind, current sequence, magic, exact size, success status가 모두 맞는 response를 기다립니다. sequence는 그 ACK 뒤에만 증가합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] sequence initial value 0
- [x] signal send 전 current sequence expectation
- [x] server endpoint 검증과 `kill` order
- [x] per-bit absolute monotonic deadline
- [x] exact size/source/field ACK predicate
- [x] wrong sequence discard
- [x] matching ACK 뒤 cursor/sequence advance
- [x] timeout/failure에서 current bit 유지

#### 비교 기준

READY predicate와 exact fields를 공유하지만 token 의미가 acquisition nonce에서 outstanding bit sequence로 바뀝니다. `4234233ebd30` server sequence initial/increment와 맞춥니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: READY는 nonce로 correlated됐지만 data bits는 아직 legacy signal ACK에 의존하거나 datagram ACK acceptance가 current bit cursor에 완전히 연결되지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: generic 또는 stale ACK가 다음 bit를 승인하면 one-bit-in-flight state가 틀어집니다. current sequence와 무관한 frame은 source가 맞아도 transition completion 근거가 아닙니다.
- 변경된 decision과 state mutation 순서: session sequence를 0부터 시작하고 각 bit의 signal 전 current sequence와 one absolute deadline을 response expectation으로 고정했습니다. READY와 공통 source/field checks에 kind ACK와 exact sequence token을 적용했습니다. server endpoint validate → current sequence deadline 생성 → current bit `kill` → exact ACK candidate loop → matching response 후에만 caller가 sequence와 bit cursor를 증가시킵니다. invalid/stale frames는 state를 바꾸지 않고 same deadline budget을 소비합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: send/path/clock/receive permanent error는 failure, deadline 만료는 timeout입니다. wrong size/source/magic/PID/kind/token/status는 discard입니다. matching ACK 전에는 bit index와 sequence가 유지됩니다.
- 후속 commit이 강화하거나 교체하는 부분: `aeb1b00867f4`가 parallel signal ACK path를 제거합니다. `b361ef9745ff`와 `1ed2acbaa353`이 rejection conjunction과 deadline을 adversarial input으로 검증합니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] specific in-flight bit에 대한 correlated completion
- [x] stale ACK가 next bit를 승인하지 못함
- [x] one transition deadline 안의 bounded wait

**아직 보장하지 않는 것**

- [x] oversized/flood regression evidence at this SHA
- [x] retransmission 또는 deduplication
- [x] same-UID malicious peer authentication
- [x] multiple bits in flight

#### 코드 증거 기록

- 파일 경로: `src/client.c`, `src/server.c`, `include/minitalk.h`
- symbol 또는 함수: `send_bit`, `wait_for_response`, `read_response`, `send_response`
- 확인한 state fields: `sequence`, `bit index`, `deadline`, `server_path`
- caller → callee: `send_byte` → `send_bit` → signal `kill` → datagram response wait/validation → cursor/sequence advance
- 핵심 branch 또는 mutation 순서: server endpoint validate → current sequence deadline 생성 → current bit `kill` → exact ACK candidate loop → matching response 후에만 caller가 sequence와 bit cursor를 증가시킵니다. invalid/stale frames는 state를 바꾸지 않고 same deadline budget을 소비합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: per-bit send success가 generic ACK flag가 아니라 exact datagram token과 source/field predicate에 연결됩니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`b361ef9745ff`가 READY와 first ACK 모두에 forged/mismatched candidates를 주입합니다.
### 4. `b361ef9745ff` — test(protocol): 응답 출처와 token 검증

- **중요도:** A
- **태그:** TEST, RESPONSE, RISK
- **개발 흐름에서의 역할:** valid READY와 first bit ACK 전에 forged-source와 mismatched-field responses를 주입해 conjunctive 검증을 검증합니다.

#### 원문에서 확정된 맥락

purpose-built response server는 forged socket, wrong token, invalid magic, incorrect server PID를 가진 frames를 먼저 보내고 마지막에 correctly correlated frame을 보냅니다. real client는 모든 invalid candidate를 무시한 뒤 valid frame에서만 진행해야 합니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] purpose-built expected server/forger sockets
- [x] forged source bind
- [x] wrong token frame
- [x] bad magic frame
- [x] wrong server PID frame
- [x] invalid→valid send order for READY
- [x] first signal event 뒤 same invalid→valid ACK order
- [x] real client success와 cleanup assertions

#### 비교 기준

각 injected frame을 `f8e8444c5ded` READY predicate와 `d3eacbbfeadc` ACK predicate의 한 rejection branch에 대응시킵니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: production predicates는 source와 fields를 함께 검사했지만 각 조건 하나만 틀린 frame이 READY/first ACK를 잘못 완료하지 않는지 end-to-end evidence가 없었습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: normal server는 invalid response를 만들지 않으므로 ordinary 통합 테스트는 reject branches를 통과하지 않습니다. source-only 또는 token-only 검사로 퇴행해도 success test는 통과할 수 있습니다.
- 변경된 decision과 state mutation 순서: `tests/response_server.c`가 expected server path와 별도 forger path를 bind하고, READY와 first ACK마다 forged source → wrong token → bad magic → wrong server PID → valid frame 순서로 전송합니다. response server PID/path 준비 → real client ACQUIRE 수신 → invalid READY sequence와 valid READY → first data signal event 관측 → invalid ACK sequence와 valid ACK → remaining ACKs 정상 응답 → NUL frame/cleanup → shell이 client/helper status와 output/stderr를 검사합니다.
- 정상 경로와 failure 경로가 갈라지는 조건: forger socket frame은 source-path check, token+1은 token check, magic 0은 magic check, PID+1은 server PID check에서 discard됩니다. final valid frame만 acquisition 또는 first bit를 완료합니다.
- 후속 commit이 강화하거나 교체하는 부분: `1ed2acbaa353`가 exact-size trailing byte와 sustained wrong-token traffic 아래 deadline을 별도로 검증합니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** source endpoint와 모든 response identity fields가 맞아야 현재 READY 또는 ACK transition이 완료됩니다.
- **재현하는 failure 또는 boundary:** forged source 또는 token, magic, server PID 하나만 다른 stale/uncorrelated response가 state를 전진시키는 상황
- **사용한 test technique:** purpose-built datagram peer가 invalid candidates를 순서대로 주입한 뒤 valid frame을 전송
- **분류:** adversarial protocol integration regression
- **failure 주입 또는 process orchestration 시작 지점:** `tests/response_server`가 server와 forger paths를 둘 다 bind하고 real client의 ACQUIRE를 받습니다.
- **production code에 진입하는 최초 호출:** client READY wait의 `recvfrom`/predicate와 first bit ACK wait의 같은 predicate입니다.
- **핵심 assertion과 관측값:** client success, helper success, expected payload/NUL output, empty diagnostics, all endpoint cleanup을 검사합니다. Progress가 invalid frames에서 일어났다면 subsequent sequence/output assertion이 깨집니다.
- **증명하는 것:** forged source rejection<br>wrong token rejection<br>bad magic rejection<br>wrong server PID rejection<br>valid correlated frame에서만 progress
- **증명하지 않는 것:** oversized rejection<br>continuous invalid traffic의 timeout bound<br>same-UID authentication<br>CPU/rate bound
- **후속 변경에서 막아야 할 구체적인 회귀:** acceptance predicate를 일부 field만 검사하도록 약화하거나 invalid frame에 sequence/cursor를 전진시키는 변경을 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] forged source rejection
- [x] wrong token rejection
- [x] bad magic rejection
- [x] wrong server PID rejection
- [x] valid correlated frame 이후에만 progress

**아직 보장하지 않는 것**

- [x] oversized frame rejection
- [x] continuous invalid traffic timeout bound
- [x] same-UID cryptographic authentication
- [x] rate limiting

#### 코드 증거 기록

- 파일 경로: `tests/response_server.c`, `tests/response_validation.sh`, `Makefile`, `src/client.c`
- symbol 또는 함수: `reply_with_invalid_events`, `send_response`, `receive_session_request`, `response_server main`
- 확인한 state fields: `g_server_socket`, `g_forger_socket`, `expected token`, `sequence`
- caller → callee: test response server → datagram candidates → production client `read_response` predicate → valid-frame progress
- 핵심 branch 또는 mutation 순서: response server PID/path 준비 → real client ACQUIRE 수신 → invalid READY sequence와 valid READY → first data signal event 관측 → invalid ACK sequence와 valid ACK → remaining ACKs 정상 응답 → NUL frame/cleanup → shell이 client/helper status와 output/stderr를 검사합니다.
- parent 또는 이전 관련 SHA와의 diff 요약: purpose-built response server binary와 validation shell test가 Makefile test target에 추가됐습니다.
- 삽입할 최소 코드 조각과 선택 이유: 표와 호출 순서만으로 불변 조건이 충분히 드러나므로 코드 덤프를 추가하지 않았습니다.
- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

`1ed2acbaa353`가 exact size와 continuous invalid traffic liveness를 별도로 검증합니다.
### 5. `1ed2acbaa353` — test(response): oversized 응답과 invalid flood 검증

- **중요도:** A
- **태그:** TEST, RESPONSE, RISK
- **개발 흐름에서의 역할:** response record보다 한 byte 큰 datagram과 sustained wrong-token traffic을 이용해 exact framing과 absolute deadline을 검증합니다.

#### 원문에서 확정된 맥락

client는 valid prefix 뒤 trailing byte가 있는 frame을 거부하고, otherwise well-formed wrong-token responses가 계속 도착해도 original transition interval 안에서 timeout해야 합니다. invalid input은 processing을 유발할 수 있지만 wait budget을 재설정하지 않습니다.

> 아래 기록은 이 SHA의 tree와 diff에서 확인했습니다. 후속 HEAD의 구현을 이 시점의 보장으로 소급하지 않았습니다.

#### 해당 SHA에서 확인한 코드

- [x] `sizeof(response)+1` payload와 trailing `0xa5`
- [x] valid-looking prefix를 expected server socket에서 send
- [x] received exact-size rejection branch
- [x] wrong-token otherwise-valid response construction
- [x] 100µs sustained send loop와 client-path stop condition
- [x] elapsed 2~6 second assertion
- [x] exact timeout diagnostic와 nonzero status
- [x] invalid receive마다 deadline이 재계산되지 않는 production code

#### 비교 기준

`f8e8444c5ded`/`d3eacbbfeadc`에서 deadline을 한 번 만드는 지점과 receive loop가 `time_until(deadline)`을 반복 사용하는 지점을 test elapsed bound와 연결합니다.

#### A-level 설계 및 failure 기록

- 이 commit 직전의 관련 state와 caller/callee: `b361ef9745ff`는 finite invalid frames 뒤 valid response를 보내 conjunction을 검증했지만 valid prefix+trailing data와 끊임없는 invalid traffic의 liveness는 검증하지 않았습니다.
- 기존 설계가 충분하지 않았던 구체적인 이유: received size를 최소 크기로만 검사하면 oversized frame의 valid prefix가 수락될 수 있습니다. invalid datagram마다 relative timeout을 새로 만들면 attacker가 wait를 무한 연장할 수 있습니다.
- 변경된 decision과 state mutation 순서: response server에 `sizeof(t_mt_response)+1` frame 전송과 wrong-token READY flood mode를 추가했습니다. shell은 flood client가 expected diagnostic으로 2~6초 안에 실패하는지 측정합니다. normal validation scenario에서 forger frame 뒤 legitimate server path가 oversized frame을 보내고 이후 other invalid/valid frames를 보냅니다. flood scenario는 ACQUIRE 수신 후 same server path에서 token+1 READY를 최대 100000회, 100µs 간격으로 client endpoint가 사라질 때까지 보냅니다.
- 정상 경로와 failure 경로가 갈라지는 조건: oversized `recvfrom` count는 `sizeof(response)`와 다르므로 field copy/validation 전에 discard됩니다. wrong token은 exact-size/source/other fields가 맞아도 token branch에서 discard되고 original monotonic deadline이 만료되면 client가 timeout/cleanup합니다.
- 후속 commit이 강화하거나 교체하는 부분: 이 commit이 Thread final adversarial regression입니다. production code는 invalid traffic에 rate limit을 두지 않지만 wait budget은 늘리지 않습니다.

#### 테스트 커밋 분석 기록

- **대상 production 불변 조건:** response는 exact record size여야 하고 ignored traffic은 original monotonic deadline을 연장하지 않습니다.
- **재현하는 failure 또는 boundary:** valid prefix를 가진 oversized frame 수락 또는 invalid response마다 timeout을 재시작해 wait가 무한 연장되는 상황
- **사용한 test technique:** oversized datagram injection + sustained wrong-token response flood
- **분류:** adversarial framing and liveness regression
- **failure 주입 또는 process orchestration 시작 지점:** response server가 legitimate source path에서 oversized frame 또는 `MT_TEST_INVALID_FLOOD` mode를 시작합니다.
- **production code에 진입하는 최초 호출:** client `recvfrom` byte-count check와 token predicate, 이어서 remaining-time calculation입니다.
- **핵심 assertion과 관측값:** oversized frame 이후에도 invalid sequence를 계속 무시하고 valid response에서 정상 progress; flood case는 nonzero client status, exact timeout diagnostic, elapsed 2~6초, helper clean exit입니다.
- **증명하는 것:** exact-size framing<br>trailing byte rejection<br>wrong-token no progress<br>invalid flood가 deadline을 reset하지 않음
- **증명하지 않는 것:** CPU/rate limit<br>peer authentication<br>packet loss recovery<br>deduplication
- **후속 변경에서 막아야 할 구체적인 회귀:** `size >= sizeof` 같은 prefix acceptance나 receive마다 fresh relative timeout을 만드는 변경을 막습니다.

#### 보장 범위

**이 commit이 보장하는 것**

- [x] exact-size acceptance rule
- [x] valid prefix plus trailing byte rejection
- [x] wrong-token traffic이 state를 전진시키지 않음
- [x] continuous invalid input이 transition deadline을 reset하지 못함

**아직 보장하지 않는 것**

- [x] CPU consumption bound under flood
- [x] rate limiting 또는 peer authentication
- [x] packet loss retransmission/deduplication
- [x] cryptographic integrity

#### 코드 증거 기록

- 파일 경로: `tests/response_server.c`, `tests/protocol_regressions.sh`, `src/client.c`
- symbol 또는 함수: `send_oversized_response`, `flood_invalid_responses`, `read_response`, `time_until`
- 확인한 state fields: `oversized payload length`, `wrong token`, `absolute deadline`, `flood tries`, `client endpoint existence`
- caller → callee: test response server → oversized/flood datagrams → client exact-size/token discard loop → original deadline timeout
- 핵심 branch 또는 mutation 순서: normal validation scenario에서 forger frame 뒤 legitimate server path가 oversized frame을 보내고 이후 other invalid/valid frames를 보냅니다. flood scenario는 ACQUIRE 수신 후 same server path에서 token+1 READY를 최대 100000회, 100µs 간격으로 client endpoint가 사라질 때까지 보냅니다.
- parent 또는 이전 관련 SHA와의 diff 요약: existing response server에 oversized sender와 flood mode를 추가하고 protocol regression script에 elapsed-time scenario를 추가했습니다.
- 삽입한 최소 코드 조각과 선택 이유: SHA `1ed2acbaa353`, `tests/response_server.c`, `send_oversized_response`. valid struct prefix 뒤 한 byte를 붙여 exact framing branch를 겨냥합니다.

```c
unsigned char payload[sizeof(*response) + 1];
memcpy(payload, response, sizeof(*response));
payload[sizeof(*response)] = 0xa5;
sendto(socket_fd, payload, sizeof(payload), 0, ...);
```

- 직접 실행한 command 또는 test와 결과: 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

#### 다음 연결

이 commit이 Thread final adversarial regression입니다.

## 6. 불변 조건 ledger

### 원자료에서 확인된 핵심 불변 조건

- response datagram은 protocol record와 정확히 같은 크기여야 하며 valid prefix만으로 수락하지 않습니다.
- expected source path와 record의 magic, server PID, kind, token, status가 모두 일치해야 transition이 완료됩니다.
- READY token은 outstanding acquisition nonce와, ACK token은 현재 outstanding bit sequence와 일치해야 합니다.
- invalid, forged, stale, oversized, uncorrelated frame은 session 또는 bit state를 전진시키지 않습니다.
- ignored traffic은 한 번 설정한 monotonic absolute deadline을 갱신하지 않습니다.

### 시간에 따른 변화 기록

| Commit | 원자료에서 확인된 변화 | 실제 state/condition | code evidence | 상태: 도입·강화·부족·복구·검증 |
| --- | --- | --- | --- | --- |
| `ebed06775b92` | request와 특정 bit transition을 식별할 수 있는 request/response fields를 정의합니다. | nonce와 token을 포함한 fixed request/response record가 생기지만 acceptance behavior는 아직 caller에 없습니다. | `include/minitalk.h: t_mt_request`, `t_mt_response` | 도입·부족 |
| `f8e8444c5ded` | READY에 exact size, source, PID, kind, nonce, status와 absolute readiness deadline을 적용합니다. | outstanding acquisition은 nonce, expected server path/PID, one absolute deadline으로 식별되며 exact READY만 완료합니다. | `src/client.c: generate_nonce`, READY response wait/validation | 강화 |
| `d3eacbbfeadc` | 현재 sequence와 정확히 일치하는 datagram ACK만 bit success로 인정합니다. | current bit는 exact sequence token ACK 전까지 outstanding이며 invalid response에 cursor/sequence가 변하지 않습니다. | `src/client.c: send_bit`, response validation loop | 강화 |
| `b361ef9745ff` | valid READY와 first bit ACK 전에 forged-source와 mismatched-field responses를 주입해 conjunctive validation을 검증합니다. | READY와 first ACK에서 source/field 하나씩 틀린 candidates가 state를 전진시키지 않고 final valid frame만 완료합니다. | `tests/response_server.c: reply_with_invalid_events`, `tests/response_validation.sh` | 검증 |
| `1ed2acbaa353` | response record보다 한 byte 큰 datagram과 sustained wrong-token traffic을 이용해 exact framing과 absolute deadline을 검증합니다. | exact frame length과 original absolute deadline이 adversarial oversized/flood traffic에서도 유지됩니다. | `tests/response_server.c`, `tests/protocol_regressions.sh`, client response loop | 검증 |

## 7. 실패 → 수정 → 검증 연결

| 기존 가정 또는 상태 | 실제 failure/위험 | Fix 또는 전환 commit | 수정된 decision/불변 조건 | Test 또는 후속 검증 | 학습자 code evidence |
| --- | --- | --- | --- | --- | --- |
| 도착 순서나 kind 하나만 맞으면 current response로 간주 | stale/forged datagram이 READY/ACK를 거짓 완료 | `f8e8444c5ded → d3eacbbfeadc` | exact source와 모든 identity fields conjunction | `b361ef9745ff` | forger source, token, magic, PID invalid frames 뒤 valid frame |
| record prefix가 valid하면 frame 전체도 valid | trailing data를 가진 oversized frame 수락 가능 | `f8e8444c5ded` | received size가 record와 정확히 같을 때만 검증 | `1ed2acbaa353` | `sizeof(response)+1` legitimate-source datagram |
| invalid response마다 relative timeout 재시작 | wrong-token flood가 wait를 무기한 연장 | `f8e8444c5ded → d3eacbbfeadc` | transition 시작 때 만든 하나의 absolute monotonic deadline | `1ed2acbaa353` | 100µs wrong-token flood와 elapsed 2~6초 timeout |

전용 test commit이 없는 연결에는 존재하지 않는 test를 만들어 적지 않았습니다.

## 8. 소유권 / state / responsibility 변화

| 단계 | state 또는 responsibility owner | transition | 당시 한계 또는 다음 변화 | 실제 symbol/field |
| --- | --- | --- | --- | --- |
| wire record definition | shared protocol header | peer와 request/bit identity fields 제공 | validation 동작은 caller에 있음 | `t_mt_request`, `t_mt_response` |
| READY wait | client outstanding acquisition | nonce+expected server endpoint exact frame만 완료 | same-UID authentication은 아님 | nonce, server path/PID, deadline |
| ACK wait | client current bit/sequence | matching ACK 뒤에만 cursor/sequence advance | retransmission/dedup 없음 | sequence token, bit index |
| invalid traffic handling | receive loop + original deadline | discard하고 같은 budget으로 계속 wait | processing rate limit 없음 | `read_response`, `time_until` |

## 9. 개발 흐름의 최종 상태

원자료에서 확인된 최종 조건:

- READY는 exact size, expected server source path, magic, server PID, READY kind, nonce token, success status가 모두 맞아야 수락됩니다.
- bit ACK는 같은 검증에 현재 sequence token을 적용하며 exact match 뒤에만 다음 bit로 전진합니다.
- forged, stale, malformed, oversized, wrong-token responses는 transition을 완료하지 않습니다.
- discarded traffic은 original `CLOCK_MONOTONIC` absolute deadline을 다시 시작하지 않으므로 wait는 bounded입니다.

학습자 기록:

- 최종 state fields와 owner: client가 outstanding nonce 또는 sequence, expected server path/PID, response kind와 one absolute deadline을 소유합니다. receive candidate는 검증 완료 전 session/bit state에 반영되지 않습니다.
- 정상 transition 순서: transition identity와 absolute deadline 확정 → request/signal send → remaining budget으로 receive → exact byte count/source 검사 → magic/PID/kind/token/status 검사 → invalid면 state 그대로 반복, valid면 acquisition 또는 bit cursor/sequence advance입니다.
- 실패 시 중단·reset·cleanup 순서: permanent send/receive/clock error는 즉시 실패하고, invalid candidate는 같은 deadline budget 안에서 discard합니다. deadline 만료는 timeout으로 전송 전체를 중단하며 current bit를 성공 처리하지 않습니다.
- 최종 상태가 보장하지 않는 것: same-UID malicious peer에 대한 authentication, invalid traffic rate limiting/CPU bound, retransmission, deduplication, exactly-once delivery는 제공하지 않습니다.
- 이 개발 흐름을 한 문단으로 설명한 최종 서술: 이 개발 흐름은 raw response record의 field set을 current transition의 acceptance predicate로 바꿉니다. READY는 acquisition nonce를, ACK는 bit sequence를 token으로 사용하며 exact size와 expected source path, internal PID, magic, kind, status가 모두 맞아야 progress합니다. invalid datagram은 state를 바꾸지 않고 최초 absolute deadline만 소비하므로 forged·oversized·flood traffic 아래에서도 wait가 bounded됩니다.

## 10. 최종 architecture 또는 실행 순서 정리

- [x] outstanding nonce 또는 sequence와 expected server endpoint 설정
- [x] transition 시작 시 monotonic absolute deadline 계산
- [x] remaining budget으로 response receive 대기
- [x] received datagram의 exact byte count와 source path 검증
- [x] magic, server PID, kind, token, status 검증
- [x] invalid frame이면 state advance 없이 같은 absolute deadline으로 반복
- [x] valid frame이면 acquisition 또는 bit transition 완료
- [x] deadline 도달이면 timeout failure 반환

```text
begin transition
    -> identity = acquisition nonce OR current bit sequence
    -> expected source path/server PID/kind/status
    -> deadline = CLOCK_MONOTONIC now + fixed interval
    -> send ACQUIRE or data signal
receive loop
    -> remaining = deadline - monotonic now
    -> pselect/recvfrom
    -> require exact sizeof(t_mt_response)
    -> require expected source path
    -> require magic + server_pid + kind + token + OK status
    -> invalid: no state advance, repeat with same deadline
    -> valid: commit acquisition or advance bit cursor/sequence
    -> deadline zero: timeout failure

```

- 실제 함수·파일을 반영한 완성 흐름: `include/minitalk.h`의 records, `src/client.c`의 nonce/deadline/response validation, `tests/response_server.c`의 adversarial frames가 complete path를 구성합니다.
- asynchronous boundary: data signal delivery는 asynchronous지만 response acceptance와 all state advance는 client normal-context datagram wait loop에서 수행됩니다.
- externally visible commit point: 모든 source/frame/identity/status 조건을 만족한 READY 또는 ACK를 받은 뒤 acquisition state 또는 bit cursor/sequence를 전진시키는 지점입니다.
- cleanup owner: client cleanup이 timeout/error/success 뒤 response descriptor와 실제 bound client path를 정리합니다. test response server는 server/forger paths를 자체 cleanup합니다.

## 11. 학습 완료 자가 점검

- [x] commit map의 5개 SHA를 source 순서대로 모두 설명할 수 있습니다.
- [x] 각 code excerpt에 SHA, path, symbol, 선택 이유가 기록돼 있습니다.
- [x] 최종 HEAD 코드를 historical SHA의 증거로 사용한 곳이 없습니다.
- [x] 정상 경로와 실패 처리를 state mutation 순서로 설명할 수 있습니다.
- [x] source 확정 불변 조건과 직접 확인한 code evidence를 구분했습니다.
- [x] test commit의 불변 조건, failure, technique, production path, proves/not-proves를 기록했습니다.
- [x] Thread final state를 함수와 state field 수준으로 설명할 수 있습니다.
- [ ] 해당 SHA의 test를 로컬에서 직접 실행했습니다. — 실행하지 않음. 이 환경에서는 GitHub 저장소를 로컬 checkout할 수 없어 해당 SHA의 tree와 diff를 GitHub connector로 검토했습니다. 따라서 아래의 test 결과는 test code가 요구하는 관측값이며, 이 세션에서 실제 실행해 얻은 결과가 아닙니다.

### 이 Thread와 직접 연결된 Major Engineering Difficulties

- predictable local endpoint와 record field 하나만으로는 stale 또는 unrelated traffic을 배제할 수 없습니다.
- acquisition nonce와 per-bit sequence는 범위가 달라 공통 검증과 transition-specific 검증을 함께 유지해야 합니다.
- continuous invalid traffic이 receive loop를 계속 깨워도 relative timeout을 반복 시작하지 않아야 합니다.

---

\
# minitalk 개발 흐름 학습 기록

## 1. 목적

이 문서 세트는 `commit-importance.md`가 확정한 개발 흐름s와 commit 평가를 유지하고, `commit-bodies.md`의 구현 의도와 실패 처리 정보를 이용해 실제 commit history를 복원하기 위한 기록 틀입니다.

완성형 프로젝트 해설서가 아닙니다. 각 SHA의 코드와 diff를 직접 읽고 설계, 구현, 실패, 수정, 검증의 연결을 채웁니다.

## 2. 권장 학습 순서

1. [`01-timing-to-correlated-sequence-acks.md`](01-timing-to-correlated-sequence-acks.md)
2. [`02-session-ownership-and-recovery.md`](02-session-ownership-and-recovery.md)
3. [`03-self-pipe-event-loop.md`](03-self-pipe-event-loop.md)
4. [`04-output-commit-boundary.md`](04-output-commit-boundary.md)
5. [`05-endpoint-ownership-and-bounded-polling.md`](05-endpoint-ownership-and-bounded-polling.md)
6. [`06-bounded-response-correlation.md`](06-bounded-response-correlation.md)

source의 개발 흐름s 순서와 같습니다. 동일 commit은 각 Thread의 학습 관점으로 중복 확인합니다.

## 3. Thread 문서 사용법

1. 해당 SHA의 commit diff와 tree를 엽니다.
2. parent 또는 지정된 이전 관련 SHA와 비교합니다.
3. source 확정 역할과 실제 파일, 함수, state field를 연결합니다.
4. 정상 경로와 failure branch를 caller → callee와 mutation 순서로 추적합니다.
5. 필요한 최소 코드만 path, symbol, SHA와 함께 삽입합니다.
6. test commit이면 production 불변 조건과 test technique을 production path에 연결합니다.
7. commit 기록을 불변 조건 ledger와 실패 → 수정 → 검증 표에 반영합니다.
8. 마지막에는 Thread execution flow를 자신의 설명으로 완성합니다.

```sh
git show --stat <sha>
git show <sha>
git diff <previous-related-sha> <sha> -- <path>
git show <sha>:<path>
```

## 4. 해당 SHA 코드 확인 원칙

- 모든 해석은 해당 commit SHA의 tree를 기준으로 합니다.
- 변경 전 코드는 parent 또는 지정된 이전 관련 SHA에서 확인합니다.
- commit body만 옮기지 않고 path, symbol, condition, state mutation으로 확인합니다.
- source에 없는 불변 조건은 확정 사실로 추가하지 않고 학습자 관찰로 표시합니다.

## 5. 최종 HEAD 소급 사용 금지

최종 HEAD의 함수, 테스트, 주석, state layout을 과거 commit에 소급하지 않습니다.

최종 server가 self-pipe를 사용하더라도 `4234233ebd30`에서는 그 SHA에 실제로 존재하는 중간 response queue와 handler responsibility를 확인해야 합니다. 후속 fix의 helper나 validation을 이전 commit의 보장으로 적지 않습니다.

## 6. Importance별 학습 깊이

| Importance | 요구 깊이 |
| --- | --- |
| S | 문제, 직전 architecture, failure 가능성, 핵심 decision, 실제 핵심 코드, 소유권/lifecycle/상태 전이, 후속 fix/test까지 추적합니다. |
| A | 주요 subsystem, integration point, validation, 실패 처리와 설계 판단을 실제 코드로 확인합니다. |
| B | Thread 전개에서 맡는 구현 역할, 필요한 state 변화, 앞뒤 commit 연결을 확인합니다. |
| C | Thread 이해에 필요한 최소 맥락만 기록합니다. |

## 7. 실제 코드 삽입 기준

각 코드 조각에 commit SHA, 파일 경로, symbol, caller/callee, 보여 주는 불변 조건 또는 failure branch, 이전 관련 SHA와의 차이를 기록합니다.

우선 삽입할 대상:

- 핵심 state field와 초기화/reset
- 소유권 획득·이전·해제 코드
- event registration/update/remove
- send/wait/validate 순서
- error, timeout, partial-operation branch
- cleanup과 rollback
- 회귀 테스트의 failure 주입 지점

## 8. Test commit 학습 방법

각 test commit에서 다음을 구분합니다.

- 대상 production 불변 조건
- 재현 failure 또는 boundary
- test technique
- 실제 production code path
- 증명하는 것과 증명하지 않는 것
- broad integration인지 deterministic regression인지
- 후속 변경에서 막는 회귀

테스트 통과만 적지 않고 child process, signal, socket path, fault hook, inherited mask, descriptor allocation이 어떤 production branch를 만들기 위한 것인지 기록합니다.

## 9. 문서 완료 기준

- 6개 개발 흐름 문서를 source 순서대로 완성했습니다.
- SHA, subject, importance, tags, commit 순서를 변경하지 않았습니다.
- 중복 commit은 각 Thread 관점으로 별도 기록했습니다.
- 중요한 commit마다 해당 SHA의 path, symbol, state, failure evidence가 있습니다.
- S/A/B/C별 깊이가 구분됩니다.
- fix는 기존 가정 → failure → root cause → 수정 불변 조건 → code → regression으로 연결됩니다.
- test는 불변 조건, technique, production path, proves/not-proves를 구분합니다.
- 최종 HEAD를 과거 증거로 사용하지 않았습니다.
- 각 Thread의 설계 → 구현 → 실패 → 수정 → 검증을 commit history로 설명할 수 있습니다.
