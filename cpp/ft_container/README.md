# ft_container Development Threads — rewritten

이 디렉터리는 `seungwoo7050/42-archive`의 `cpp/ft_container` 브랜치에 있는 Development Thread 여섯 편을 실제 commit diff와 각 SHA 시점의 source를 기준으로 다시 작성한 결과입니다.

기존 `development-thread-workbook/completed` 문서에서 Thread 경계, 포함 commit, SHA, 제목, importance, tags, source-established role을 출발점으로 사용했습니다. 기존 본문의 학습 골격, 반복 질문, 완료 체크리스트, 서술은 정답으로 간주하지 않았습니다.

## 작성 원칙

- 대상 branch는 `cpp/ft_container` 하나로 제한했습니다.
- 각 commit은 정확한 SHA의 diff와 해당 시점의 source를 기준으로 설명했습니다.
- 후대 representation과 helper를 이전 commit에 소급하지 않았습니다.
- 같은 commit에 섞인 변경 중 Thread와 무관한 부분은 제외했습니다.
- importance와 tags는 기존 연동 metadata를 보존했습니다.
- Thread를 단순한 시간순 commit 묶음이 아니라 하나의 개발 문제와 불변 조건을 설명하는 단위로 정리했습니다.
- S급 commit은 representation, failure 가능성, 결정, ownership/lifetime, 후속 검증까지 깊게 설명했습니다.
- B/C급 commit은 맡은 역할에 필요한 만큼만 설명하고 고정 heading을 반복하지 않았습니다.
- repository source와 test를 검토했지만 이 환경에서는 build, test, sanitizer, CI를 실행하지 않았습니다.

## 문서 구성

| 문서 | 중심 문제 | 핵심 도달점 |
| --- | --- | --- |
| [01 — C++98 generic interface foundation](01-cxx98-generic-interface-foundation.md) | C++98에서 container들이 공유할 compile-time dispatch, pair, range algorithm, iterator vocabulary | fill/range overload와 vector/map iterator가 같은 공용 형식을 사용 |
| [02 — Vector ownership, aliasing, exception-safe mutation](02-vector-ownership-aliasing-exception-safe-mutation.md) | raw storage와 live object를 구분하면서 growth, self-aliasing, insert, exception을 처리 | `_size`가 constructed-object count로 유지되고 실패 시 partial tail/new block이 회수됨 |
| [03 — Map from unbalanced BST to verified red-black core](03-map-unbalanced-bst-to-verified-red-black-core.md) | 공개 정렬 결과뿐 아니라 red-black 구조와 logarithmic complexity까지 보장 | 삽입·삭제 fixup, structural inspector, deterministic random test, 높이/비교 상한 |
| [04 — Stable map iterators through structural mutation](04-stable-map-iterators-through-structural-mutation.md) | copied root를 가진 null-end iterator가 회전·root 삭제·swap에서 stale해지는 문제 | value-free header가 root/min/max/end를 소유하고 element iterator는 node identity만 보유 |
| [05 — Stateful allocators and map failure transactions](05-stateful-allocators-map-failure-transactions.md) | allocator/comparator state와 physical tree ownership을 예외 아래 일관되게 유지 | 비교를 allocation 전에 끝내고 temporary tree와 policy-first commit 순서를 사용 |
| [06 — Header-only public surface and automated acceptance](06-header-only-public-surface-automated-acceptance.md) | monolithic test를 넘어 독립 header, multi-TU consumer, sanitizer, compiler matrix를 검증 | `make check`와 격리된 `sanitize`, compiler/platform CI로 acceptance 범위를 반복 가능하게 만듦 |

## Thread 사이의 관계

아래 관계는 학습 순서일 뿐, 모든 Thread가 하나의 선형 feature chain이라는 뜻은 아닙니다.

```text
01 공용 generic vocabulary
 ├─ 02 vector lifetime/mutation
 └─ 03 map red-black structure
      ├─ 04 map iterator/end representation
      └─ 05 map allocator/comparator failure transaction

06 public acceptance
 └─ 01~05의 공개 결과와 test surface를 compile/link/sanitizer/CI에서 검증
```

Thread 03, 04, 05는 같은 `map` source를 서로 다른 문제 관점에서 다룹니다.

- Thread 03은 ordering, color, black height, complexity를 다룹니다.
- Thread 04는 element identity, `end()` representation, mutation 뒤 traversal을 다룹니다.
- Thread 05는 allocator identity, comparator policy, exception 시 node ownership을 다룹니다.

따라서 일부 commit의 시간적 순서는 서로 얽혀 있지만, 문서에서는 억지로 하나의 “다음 단계”로 합치지 않았습니다.

## 반복해서 등장하는 핵심 불변 조건

### Vector

```text
[0, size)       = constructor가 완료된 live objects
[size, capacity)= raw storage
```

allocation, construction, assignment, destruction을 같은 동작으로 취급하지 않습니다. reallocation path는 새 block을 완성한 뒤 publish하며, spare-capacity insertion은 raw tail construction과 live-slot assignment를 구분합니다.

### Map tree

- map root에서 도달 가능한 node는 정확히 한 map이 소유합니다.
- 새 node는 comparator search가 끝난 뒤 allocate합니다.
- red-black root는 black이고 red node의 child는 black입니다.
- 모든 root-to-null path의 black height가 같습니다.
- header는 current root, minimum, maximum과 `end()`를 표현합니다.

### Failure transaction

- construct가 실패하면 완료된 prefix/node만 destroy하고 allocation을 해제합니다.
- replacement state는 temporary owner에서 먼저 완성합니다.
- 예외를 던질 수 있는 comparator 교환은 tree와 allocator ownership 이동보다 먼저 시도합니다.
- test는 visible output뿐 아니라 live object 수, outstanding blocks, parent/color/black-height, tree height를 별도로 관찰합니다.

### Header-only acceptance

```text
behavior test
≠ independent-header compile
≠ multi-translation-unit link
≠ sanitizer
≠ compiler/platform matrix
```

한 층의 성공이 다른 층을 자동으로 증명하지 않으므로 검증을 서로 다른 target으로 유지합니다.

## 검토 범위와 비검토 범위

확인한 것:

- 여섯 기존 Thread의 metadata
- 각 포함 commit의 exact SHA, subject, changed paths와 diff
- 설명에 필요한 당시 header/test/Makefile/workflow source
- fix 전후 representation, mutation order, cleanup, test assertions
- 각 test가 증명하는 것과 증명하지 않는 것

수행하지 않은 것:

- 다른 branch의 code 참조
- final HEAD를 과거 commit에 소급
- local checkout/build
- test binary 실행
- ASan/UBSan 실행
- GitHub Actions 결과 확인 또는 재실행

문서 안의 “검증한다”는 표현은 test source가 가진 assertion과 관찰 기법을 뜻합니다. 이 작업 자체가 해당 test의 실행 성공을 새로 증명했다는 뜻은 아닙니다.
