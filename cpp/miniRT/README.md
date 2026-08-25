# miniRT Development Threads — 재작성본

이 문서 세트는 `cpp/miniRT`의 개발 이력을 단순한 시간순 커밋 목록이 아니라, 서로 독립적으로
추적할 수 있는 **개발 문제 단위**로 복원합니다. 각 문서는 실제 commit diff와 해당 SHA의
source/test를 근거로 문제, 결정, 구현, 실패 가능성, 수정, 검증 범위를 연결합니다.

> **조사 범위**
>
> 이 문서는 `cpp/miniRT` 브랜치의 표시된 exact SHA와 그 parent diff만을 근거로 작성했습니다.
> 다른 브랜치나 최종 HEAD의 구현을 과거 커밋에 소급하지 않았습니다. GitHub에서 diff와
> 해당 SHA의 source/test를 확인했으며, 로컬 checkout을 확보하지 못해 build·test·benchmark는
> 실행하지 않았습니다. 따라서 아래의 실행 결과는 repository에 들어 있는 assertion과
> expected value의 의미를 설명한 것이며, 이 작업 환경에서 새로 측정한 결과가 아닙니다.

## 문서 구성

| 문서 | 중심 문제 | 핵심 도달점 |
| --- | --- | --- |
| [01-geometric-contracts-to-first-image.md](01-geometric-contracts-to-first-image.md) | 공통 hit 계약에서 parser·camera·shading·output을 거쳐 첫 이미지까지 연결 | 유효한 `.rt`가 결정적인 P3 PPM과 checksum으로 노출되는 최초 vertical slice |
| [02-large-vector-normalization.md](02-large-vector-normalization.md) | 큰 유한 벡터의 길이 계산 overflow | `std::hypot` 기반 정규화와 정확한 회귀 사례 |
| [03-correctness-preserving-bvh.md](03-correctness-preserving-bvh.md) | 선형 탐색의 의미를 바꾸지 않고 BVH로 작업량 감소 | bounds, 결정적 build/traversal, 동치 검증, derived-state lifecycle |
| [04-material-syntax-and-reflection.md](04-material-syntax-and-reflection.md) | diffuse-only tracer에 metal reflection을 추가하면서 기존 문법 보존 | 선택적 material 문법, 깊이 소비, secondary-ray 검증 |
| [05-deterministic-tiled-rendering.md](05-deterministic-tiled-rendering.md) | 직렬 이미지를 병렬화하면서 byte와 work semantics 보존 | 고정 tile, 원자적 분배, disjoint writes, worker failure 전파 |
| [06-image-representation-and-atomic-output.md](06-image-representation-and-atomic-output.md) | 이미지 크기·저장소·checksum·파일 게시의 실패 안전성 | checked arithmetic, exact storage, 표준 FNV-1a, temporary publication |
| [07-reproducible-verification.md](07-reproducible-verification.md) | 직접 빌드와 산발적 검증을 반복 가능한 build/test matrix로 전환 | `raycore`, CTest, ASan/UBSan, Ubuntu/macOS CI |

## 기존 metadata에 대한 repository 보정

기존 `completed/01-geometric-contracts-to-first-image.md`의 commit map은 첫 이미지 경로에서
실제로 필요한 다음 10개 커밋을 생략하고 있었습니다.

- 도형 구현: `dfb5b010ed54`, `d89812d9173a`, `7265686c18ee`, `197dbf694170`
- parser 값 해석: `7bba0af26d17`, `d4a24901051a`
- parser 지시어 구현: `9aef46929554`, `5e21e6900fd9`, `c17a4b5737d2`, `d0cc38dd5762`

특히 `6bff18bf0bac`의 exact source는 줄을 순회하고 token을 만든 뒤 모든 non-empty line을
`unknown directive`로 거부하는 dispatch 골격입니다. 실제 `R`, `A`, `C`, `L`, `sp`, `pl`,
`cy` 처리는 그 뒤 여섯 커밋에서 점진적으로 추가됩니다. `1e1fda47d913`은 이 구현들을
도입하는 커밋이 아니라, 이미 만들어진 지시어 처리 위에 필수 지시어 검증과 text/file loader를
완성하는 커밋입니다.

추가한 10개 커밋의 importance와 tags는 임의로 만든 것이 아닙니다. 같은 브랜치의
`commit/commit-importance.md`에 있는 공식 분류를 그대로 사용했습니다. 나머지 여섯 Thread의
기존 commit map은 실제 diff와 설명하려는 문제 단위가 일치하여 유지했습니다.

## Thread 관계

이 번호는 읽기 편의를 위한 배열이지 단일 개발 순서를 뜻하지 않습니다.

- Thread 1은 최초로 동작하는 직렬 기준선을 만든다.
- Thread 2는 그 기준선에서 발견된 독립적인 수치 안정성 수정이다.
- Thread 3은 교차 탐색을 가속하지만 Thread 1의 hit winner와 image semantics를 보존해야 한다.
- Thread 4는 tracing semantics를 확장한다.
- Thread 5는 pixel production을 병렬화하지만 Thread 1·3·4가 정한 결과를 바꾸면 안 된다.
- Thread 6은 생성된 이미지의 표현과 외부 파일 게시를 안전하게 만든다.
- Thread 7은 앞의 모든 계약을 반복 검증할 build/test 환경을 만든다.

따라서 “이전 Thread가 끝나야 다음 Thread가 시작된다”는 서술은 사용하지 않았습니다. 대신 각
문서 끝에 그 Thread가 보장하는 것과 의도적으로 다루지 않는 범위를 적었습니다.

## 문서 작성 기준

1. SHA마다 실제 diff가 도입한 변화만 설명했습니다.
2. 같은 commit에 섞인 무관한 변경은 해당 Thread에서 제외했습니다.
3. feature가 만든 위험은 후속 fix/test와 연결했습니다.
4. test는 **무엇을 증명하는지**와 **무엇을 증명하지 않는지**를 분리했습니다.
5. 중요한 S/A commit은 자료 표현, 상태 전이, tie rule, failure path까지 설명했습니다.
6. 작은 B commit은 실제 역할만 남기고 반복 템플릿으로 부풀리지 않았습니다.
7. 후대의 private field, BVH, tile renderer, checked output 등을 최초 렌더러 시점에 소급하지
   않았습니다.

## 권장 읽기 방식

프로젝트의 최초 실행 경로가 필요하면 Thread 1부터 읽습니다. 특정 문제만 확인하려면 각
문서는 독립적으로 읽을 수 있습니다. BVH와 병렬 렌더링을 함께 볼 때는 Thread 3의
“동일 결과” 계약을 먼저 확인한 뒤 Thread 5의 tile ownership을 읽는 편이 좋습니다.
출력 파일 손상과 checksum 의미는 Thread 6에서 별도로 다룹니다.
