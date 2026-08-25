# Thread 4. 선택적 재질 문법과 깊이 제한 metal reflection

최초 renderer의 모든 surface는 diffuse였습니다. 이 Thread는 hit contract나 Scene traversal을
교체하지 않고, `Material`이 결정하는 tracing branch를 추가합니다. 동시에 기존 `.rt` 파일이
그대로 diffuse로 해석되도록 optional syntax를 설계하고, recursion depth와 secondary-ray
count를 테스트로 고정합니다.

> **조사 범위**
>
> 이 문서는 `cpp/miniRT` 브랜치의 표시된 exact SHA와 그 parent diff만을 근거로 작성했습니다.
> 다른 브랜치나 최종 HEAD의 구현을 과거 커밋에 소급하지 않았습니다. GitHub에서 diff와
> 해당 SHA의 source/test를 확인했으며, 로컬 checkout을 확보하지 못해 build·test·benchmark는
> 실행하지 않았습니다. 따라서 아래의 실행 결과는 repository에 들어 있는 assertion과
> expected value의 의미를 설명한 것이며, 이 작업 환경에서 새로 측정한 결과가 아닙니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `85583e1e9beb` | `feat(material): metal 모델과 깊이 제한 반사 구현` | A | `CORE, MATERIAL, RAY_PIPELINE` | perfect-metal branch와 recursion depth 소비를 tracing에 추가 |
| 2 | `a90130a5b030` | `feat(parser): 선택적 도형 재질 문법 추가` | B | `PARSER, MATERIAL` | 기존 색상-only 문법을 유지하며 optional `diffuse|metal` token 추가 |
| 3 | `9a352ffe8233` | `test(material): 재질 파싱과 반사 깊이 검증` | B | `TEST, MATERIAL, DETERMINISM` | 기본값·unknown material·depth·secondary ray·diffuse checksum을 고정 |
| 4 | `3aa806753cc4` | `feat(cli): 반사 깊이 option과 기본값 추가` | B | `CLI, MATERIAL` | `--max-depth 0..32`와 default 4를 public CLI에 노출 |

## `85583e1e9beb` — 하나의 hit에서 두 tracing 의미로 분기하다

**중요도** `A` · **태그** `CORE, MATERIAL, RAY_PIPELINE`

`Material`은 `Diffuse`와 `Metal` type을 갖게 됩니다. diffuse branch는 기존
ambient/direct/shadow shading을 그대로 사용합니다. metal branch는 perfect reflection 방향을
계산해 한 개의 secondary ray를 재귀적으로 trace합니다.

```cpp
Vec3 reflect(const Vec3& direction, const Vec3& normal) {
    return direction - normal * (2.0 * dot(direction, normal));
}

Color traceRay(const Scene& scene,
               const Ray& ray,
               int max_depth,
               RenderStats* stats) {
    HitRecord hit;
    if (!findNearestHit(scene, ray, hit, kRayTMin,
                        std::numeric_limits<double>::infinity(), stats))
        return scene.background;

    if (hit.material.type == MaterialType::Metal) {
        if (max_depth <= 0)
            return Color();

        const Vec3 reflected =
            normalize(reflect(normalize(ray.direction), hit.normal));
        const Ray secondary(hit.point + hit.normal * kRayTMin,
                            reflected);
        if (stats)
            ++stats->secondaryRays;
        return hit.material.albedo *
               traceRay(scene, secondary, max_depth - 1, stats);
    }

    return shadeHit(scene, hit, ray, stats);
}
```

여기서 depth는 “현재 primary ray를 셀 것인가”가 아니라 **metal bounce를 더 만들 수 있는
예산**입니다.

- diffuse hit는 depth를 소비하지 않고 직접광을 계산합니다.
- metal hit에서 `max_depth <= 0`이면 black을 반환하고 secondary ray를 만들지 않습니다.
- metal hit에서 depth가 남아 있으면 정확히 1을 소비합니다.
- secondary ray origin은 normal 방향으로 `kRayTMin`만큼 이동해 자기 surface를 즉시 다시 맞는
  위험을 줄입니다.
- reflected color에는 metal albedo가 component-wise 적용됩니다.

perfect metal이므로 fuzz, roughness, Fresnel, energy-conserving BRDF는 없습니다. 이 commit의
의도는 재질 시스템 전체가 아니라 결정적인 한 갈래의 반사 semantics를 추가하는 것입니다.

## `a90130a5b030` — 기존 scene 문법을 깨지 않는 확장

**중요도** `B` · **태그** `PARSER, MATERIAL`

기존 primitive directive의 마지막 값은 color였습니다.

```text
sp center diameter color
pl point normal color
cy center axis diameter height color
```

커밋은 color 뒤에 optional material token을 허용합니다.

```text
sp center diameter color [diffuse|metal]
pl point normal color [diffuse|metal]
cy center axis diameter height color [diffuse|metal]
```

arity는 각 directive에서 기존 값 또는 기존+1만 허용합니다. token이 없으면
`MaterialType::Diffuse`, `diffuse`도 diffuse, `metal`은 metal입니다. 다른 문자열은
source-located `ParseError`가 되고 shape는 추가되지 않습니다.

이 결정은 두 가지를 동시에 지킵니다.

1. 기존 `.rt` fixture는 수정하지 않아도 같은 diffuse image를 만듭니다.
2. material type은 color parsing과 shape construction 사이에서 확정되어, 잘못된 token이
   부분적으로 Scene을 변경하지 않습니다.

이 SHA는 CLI depth default를 바꾸지 않습니다. parser는 재질을 표현할 수 있게 되었지만
tracing budget의 public option은 뒤 commit이 담당합니다.

## `9a352ffe8233` — 문법과 tracing state를 같은 테스트에서 연결

**중요도** `B` · **태그** `TEST, MATERIAL, DETERMINISM`

테스트는 다음 서로 다른 계약을 분리합니다.

### Parser compatibility

- material token을 생략하면 diffuse입니다.
- `diffuse`와 `metal`을 명시하면 각각 해당 type입니다.
- 알 수 없는 material name은 parse failure입니다.
- 기존 diffuse fixture의 checksum은 `456dc8d87ebf194f`로 유지됩니다.

마지막 항목은 syntax 확장이 기존 scene의 pixels를 바꾸지 않았다는 강한 회귀 기준입니다.

### Reflection depth

1×1 mirror fixture에서 depth 0은 black이어야 하고, depth 1은 repository가 기대하는
`(0.2, 0.25, 0.1875)` color를 만듭니다. 같은 input을 반복 trace해 같은 결과가 나오는지도
검사합니다.

stats는 depth 1에서 `secondaryRays == 1`이어야 합니다. 이는 단순히 final color가 맞는 것보다
더 좁은 구현 계약을 고정합니다. 예를 들어 불필요한 secondary ray를 두 번 만들고 같은 color를
합성하는 회귀도 work counter로 드러납니다.

이 테스트가 모든 반사 깊이와 복잡한 mirror hall의 안정성을 증명하는 것은 아닙니다. 정확히
parser default, unknown token, 한 번의 depth consumption, diffuse backward compatibility를
검증합니다.

## `3aa806753cc4` — tracing budget을 CLI 계약으로 만들다

**중요도** `B` · **태그** `CLI, MATERIAL`

`RenderSettings::maxDepth` 기본값은 1에서 4로 바뀌고 CLI는 다음 option을 받습니다.

```text
--max-depth N
```

`N`은 정수 `0..32`여야 합니다. 값 누락, 범위 밖, 중복 option은 usage error입니다.
option order는 output path와 충돌하지 않도록 parser가 명시적으로 처리합니다.

- `0`은 metal surface에서 secondary ray를 만들지 않는 유효한 설정입니다.
- `32` upper bound는 무한 recursion을 막기 위한 public resource limit입니다.
- default 4는 parser/renderer 내부 상수가 아니라 `RenderSettings`에 들어가 function API와 CLI가
  같은 값을 공유합니다.

이 commit은 reflection algorithm을 바꾸지 않습니다. 이미 구현된 depth parameter를 사용자에게
노출하고 default policy를 정합니다.

## 최종 tracing 흐름

```text
nearest hit
  ├─ miss
  │    -> background
  ├─ diffuse
  │    -> ambient + visible direct lighting
  └─ metal
       ├─ maxDepth <= 0 -> black
       └─ reflected direction
            -> origin offset
            -> secondaryRays += 1
            -> trace(maxDepth - 1)
            -> multiply by albedo
```

## 보장과 경계

이 Thread가 보장하는 것은 다음입니다.

- material token이 없던 기존 scene은 diffuse로 유지됩니다.
- unknown material은 shape construction 전에 실패합니다.
- metal bounce는 한 단계마다 depth를 정확히 1 소비합니다.
- depth 0은 더 이상 secondary ray를 만들지 않습니다.
- diffuse compatibility와 한 단계 reflection의 pixels/work가 regression으로 고정됩니다.

다음은 다루지 않습니다.

- glossy/fuzzy metal, Fresnel, refraction, texture
- recursion을 iterative path tracing으로 바꾸는 문제
- BVH가 secondary ray에도 같은 결과를 주는지에 대한 전체 증명: BVH 동치는 Thread 3의 mode
  contract를 사용합니다.
- 여러 worker에서 reflection을 수행하는 scheduling: Thread 5
