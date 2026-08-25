# Thread 1. 기하 계약에서 첫 번째 렌더링 이미지까지

이 Thread는 흩어진 수학 타입을 하나의 실행 가능한 renderer로 결합하는 최초의 vertical slice를
복원합니다. 출발점은 모든 도형이 같은 방식으로 hit를 반환하는 계약이고, 도착점은 `.rt` 파일을
읽어 pixel을 계산하고 P3 PPM 파일과 checksum을 만드는 CLI입니다.

이 경로는 이후 BVH, metal reflection, tile concurrency, checked image storage가 보존해야 할
**직렬 의미 기준선**입니다. 다만 이 시점의 구현을 후대의 안전한 최종 구현으로 소급하지
않습니다. 초기 `Image` 크기 계산, PPM pixel index, checksum 기준값, 파일 교체에는 후속 Thread가
고치는 한계가 남아 있습니다.

> **조사 범위**
>
> 이 문서는 `cpp/miniRT` 브랜치의 표시된 exact SHA와 그 parent diff만을 근거로 작성했습니다.
> 다른 브랜치나 최종 HEAD의 구현을 과거 커밋에 소급하지 않았습니다. GitHub에서 diff와
> 해당 SHA의 source/test를 확인했으며, 로컬 checkout을 확보하지 못해 build·test·benchmark는
> 실행하지 않았습니다. 따라서 아래의 실행 결과는 repository에 들어 있는 assertion과
> expected value의 의미를 설명한 것이며, 이 작업 환경에서 새로 측정한 결과가 아닙니다.

## Commit map

기존 completed 문서의 12개 commit은 유지했습니다. 실제 repository history에서 그 사이에
존재하며 첫 이미지 경로를 구성하는 도형 구현 4개와 parser 구현 6개를 추가했습니다. 추가
커밋의 importance/tags는 `commit/commit-importance.md`의 분류를 그대로 사용했습니다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f3f1d04cc836` | `feat(geometry): hit와 도형 교차 계약 정의` | S | `ARCH, GEOMETRY, SCENE` | 모든 도형·장면 탐색·shading이 공유할 `Shape`/`HitRecord` 계약 정의 |
| 2 | `dfb5b010ed54` | `feat(geometry): 구 교차 계산 구현` | B | `GEOMETRY` | 두 근 중 `[t_min,t_max]` 안의 가까운 구 hit를 공통 record로 게시 |
| 3 | `d89812d9173a` | `feat(geometry): 평면 교차 계산 구현` | B | `GEOMETRY` | 평행 ray를 거부하고 유효 구간의 평면 hit를 공통 record로 게시 |
| 4 | `7265686c18ee` | `feat(geometry): 유한 원기둥 옆면 교차 구현` | A | `GEOMETRY, HARD` | 임의 축 원기둥의 side quadratic과 높이 clipping 구현 |
| 5 | `197dbf694170` | `feat(geometry): 원기둥 cap과 최근접 hit 선택 완성` | A | `GEOMETRY, HARD` | side와 두 cap 후보를 하나의 closest-hit 갱신 규칙으로 통합 |
| 6 | `2a01cb406d9d` | `feat(scene): 카메라·조명과 장면 aggregate 구성` | A | `ARCH, SCENE` | resolution·ambient·camera·lights·shapes를 한 장면 상태로 결합 |
| 7 | `41a1d6bbe5ef` | `feat(scene): 선형 최근접 교차 탐색 구현` | A | `CORE, SCENE` | 현재 closest를 다음 도형의 `t_max`로 전달하는 선형 기준 경로 확립 |
| 8 | `3545eb1e82df` | `feat(parser): 소스 위치 오류와 line tokenization 구성` | B | `PARSER, PRACTICAL` | source/line을 보존하는 오류와 trim/tokenization 기반 마련 |
| 9 | `7bba0af26d17` | `feat(parser): 유한 수와 범위 값 해석 구현` | B | `PARSER, EDGE` | finite double·positive integer·ratio·positive scalar 검증 추가 |
| 10 | `d4a24901051a` | `feat(parser): 벡터와 색상 token 해석 구현` | B | `PARSER` | comma vector와 0..255 RGB를 내부 `Vec3`/`Color`로 변환 |
| 11 | `6bff18bf0bac` | `feat(parser): 줄 단위 지시어 dispatch 기반 구성` | A | `ARCH, PARSER` | 주석 제거·줄 순회·token dispatch와 unknown-directive 실패 경계 활성화 |
| 12 | `9aef46929554` | `feat(parser): 해상도와 환경광 지시어 지원` | B | `PARSER` | `R`·`A`의 arity, duplicate, range 검증과 Scene 갱신 |
| 13 | `5e21e6900fd9` | `feat(parser): 카메라와 광원 지시어 지원` | B | `PARSER` | `C`·`L`의 방향/FOV/brightness 검증과 Scene 갱신 |
| 14 | `c17a4b5737d2` | `feat(parser): 구와 평면 지시어 지원` | B | `PARSER, GEOMETRY` | `sp`·`pl`에서 검증된 값을 실제 도형으로 생성 |
| 15 | `d0cc38dd5762` | `feat(parser): 원기둥 지시어 지원` | B | `PARSER, GEOMETRY` | `cy`의 axis·diameter·height를 검증하고 원기둥 생성 |
| 16 | `1e1fda47d913` | `feat(parser): 필수 지시어 검증과 입력 loader 완성` | A | `PARSER, INTEGRATION` | `R/A/C` 필수성 및 text/file loader와 fixtures 완성 |
| 17 | `e6da5f987b97` | `feat(camera): 화면 좌표를 카메라 광선으로 변환` | A | `CORE, RAY_PIPELINE` | 카메라 직교 frame과 pixel-center ray 생성 |
| 18 | `e8b7dc42a52c` | `feat(render): 직접광과 그림자 추적 구현` | S | `CORE, RAY_PIPELINE, RISK` | ambient·Lambertian direct light·shadow visibility와 miss background 정의 |
| 19 | `c742b2401e52` | `feat(renderer): 직렬 이미지 렌더링 구현` | S | `ARCH, CORE, RAY_PIPELINE` | 모든 pixel을 고정 순서로 trace·clamp·quantize하는 최초 image loop |
| 20 | `1bc7cacd30aa` | `feat(output): PPM 직렬화와 이미지 체크섬 구현` | A | `OUTPUT, DETERMINISM` | P3 text representation과 image bytes 기반 checksum 노출 |
| 21 | `b983f0ea2744` | `feat(cli): 장면 렌더링 명령 연결` | B | `CLI, INTEGRATION` | load→render→write와 선택적 checksum을 command line에 연결 |
| 22 | `d05a6ab48bb1` | `test(render): 장면 렌더링 smoke 검사 추가` | B | `TEST, INTEGRATION` | valid/invalid CLI path와 repeat-byte 결정성을 smoke 수준에서 검증 |

## 1. 공통 hit 계약이 구체 도형을 같은 방식으로 보이게 한다

### `f3f1d04cc836` — 도형의 결과 표현을 먼저 고정하다

**중요도** `S` · **태그** `ARCH, GEOMETRY, SCENE`

이전에는 각 도형 계산을 만들더라도 renderer가 결과를 공통 방식으로 소비할 표현이 없었습니다.
커밋은 `Shape::intersect`와 `HitRecord`를 동시에 도입합니다.

```cpp
struct HitRecord {
    double t;
    Vec3 point;
    Vec3 normal;
    Material material;
    const Shape* shape;
    bool frontFace;

    void setFaceNormal(const Ray& ray, const Vec3& outward_normal) {
        frontFace = dot(ray.direction, outward_normal) < 0.0;
        normal = frontFace ? outward_normal : -outward_normal;
    }
};

class Shape {
public:
    virtual bool intersect(const Ray& ray,
                           double t_min,
                           double t_max,
                           HitRecord& hit) const = 0;
    virtual std::string typeName() const = 0;
    const Material& material() const;

protected:
    Material material_;
};
```

핵심 결정은 세 가지입니다.

1. **후보 구간을 caller가 준다.** 각 도형은 임의의 양의 hit가 아니라 `[t_min, t_max]` 안의
   후보만 반환합니다. Scene은 이 상한을 줄여 가까운 후보만 남길 수 있습니다.
2. **normal 방향을 record가 통일한다.** 도형은 기하학적 outward normal을 계산하고,
   `setFaceNormal`이 incoming ray를 기준으로 관찰자 쪽 normal과 `frontFace`를 결정합니다.
3. **hit가 shading에 필요한 값을 함께 운반한다.** point, oriented normal, material value,
   non-owning shape pointer가 하나의 record에 담깁니다.

`shape`는 소유권을 이전하지 않습니다. 이 SHA의 도형 객체 수명은 Scene 쪽에서 유지되어야 하고,
record는 교차 결과를 사용하는 동안만 해당 객체를 가리킵니다.

### `dfb5b010ed54` — 구: 가까운 근을 우선하되 유효 구간을 넘지 않는다

**중요도** `B` · **태그** `GEOMETRY`

구 교차는 `half_b` 형태의 quadratic을 사용합니다. 먼저 작은 근을 검사하고 범위 밖이면 큰 근을
검사합니다.

```cpp
double root = (-half_b - sqrt_discriminant) / a;
if (root < t_min || root > t_max) {
    root = (-half_b + sqrt_discriminant) / a;
    if (root < t_min || root > t_max)
        return false;
}

hit.t = root;
hit.point = ray.at(root);
hit.material = material_;
hit.shape = this;
hit.setFaceNormal(ray, (hit.point - center) / radius);
```

반지름과 ray direction의 길이가 `kEpsilon` 이하이면 계산을 거부합니다. 이 커밋은 Scene의
최근접 선택을 구현하지 않습니다. 대신 도형 자체가 주어진 window 안의 가장 가까운 유효 근을
반환한다는 책임을 맡습니다.

### `d89812d9173a` — 평면: 평행성 판단과 유효 구간

**중요도** `B` · **태그** `GEOMETRY`

생성자에서 normal을 정규화하고, 교차 시 denominator가 `kEpsilon` 이하이면 평행으로 봅니다.

```cpp
const double denominator = dot(normal, ray.direction);
if (std::fabs(denominator) <= kEpsilon)
    return false;

const double t = dot(point - ray.origin, normal) / denominator;
if (t < t_min || t > t_max)
    return false;
```

평면은 무한하지만 hit 결과는 여전히 공통 `[t_min,t_max]` 계약을 따릅니다. 후대 BVH Thread에서
평면을 finite box에 억지로 넣지 않고 별도 unbounded path에 남길 수 있는 이유도 이 기하적
성격에서 시작합니다. 그 후대 설계는 여기로 소급하지 않습니다.

### `7265686c18ee` — 임의 축 유한 원기둥의 옆면

**중요도** `A` · **태그** `GEOMETRY, HARD`

원기둥의 축을 정규화한 뒤 ray와 원점 차이를 축 성분과 수직 성분으로 분해합니다.

```cpp
const double direction_axis = dot(ray.direction, axis);
const double origin_axis = dot(oc, axis);
const Vec3 direction_perp = ray.direction - axis * direction_axis;
const Vec3 origin_perp = oc - axis * origin_axis;

const double a = dot(direction_perp, direction_perp);
const double half_b = dot(direction_perp, origin_perp);
const double c = dot(origin_perp, origin_perp) - radius * radius;
```

quadratic root가 유효하더라도 hit point의 축 방향 거리가 `[-height/2, height/2]` 바깥이면
무한 원기둥의 hit일 뿐 유한 원기둥의 옆면 hit가 아니므로 버립니다. normal은 중심축으로 내린
성분을 뺀 수직 벡터입니다.

```cpp
const double axial_distance = dot(point - center, axis);
if (axial_distance < -half_height - kEpsilon ||
    axial_distance >  half_height + kEpsilon)
    continue;

const Vec3 outward_normal =
    normalize((point - center) - axis * axial_distance);
```

이 SHA에는 cap이 없습니다. 따라서 축 방향으로 들어오는 ray나 cap만 맞는 ray는 아직 원기둥을
놓칠 수 있습니다.

### `197dbf694170` — side와 두 cap을 하나의 closest 경쟁으로 합치다

**중요도** `A` · **태그** `GEOMETRY, HARD`

cap은 평면과 교차한 뒤 원판 반경 안인지 확인합니다. 중요한 변화는 side와 cap마다 record를
제각각 덮어쓰지 않고 공통 helper를 통해 `closest`를 갱신한다는 점입니다.

```cpp
bool update_hit_if_closer(...,
                          double t,
                          const Vec3& outward_normal,
                          double t_min,
                          double& closest,
                          HitRecord& hit) {
    if (t < t_min || t > closest)
        return false;
    hit.t = t;
    hit.point = ray.at(t);
    hit.material = material;
    hit.shape = shape;
    hit.setFaceNormal(ray, outward_normal);
    closest = t;
    return true;
}
```

top cap은 `axis`, bottom cap은 `-axis`를 outward normal로 사용합니다. side 두 근과 cap 두 개가
모두 같은 `closest`를 공유하므로 최종 record는 실제로 가장 가까운 유효 surface를 나타냅니다.
seam에서 side/cap이 같은 `t`를 만들 수 있는 tolerance 정책까지 완전히 해결했다는 뜻은 아니지만,
원기둥을 닫힌 finite primitive로 만드는 첫 완성점입니다.

## 2. Scene이 이질적인 도형을 하나의 최근접 질의로 합친다

### `2a01cb406d9d` — 렌더링 입력 상태의 aggregate

**중요도** `A` · **태그** `ARCH, SCENE`

Scene은 이 시점에 resolution과 필수 지시어 존재 여부, ambient, background, camera, lights,
그리고 `std::shared_ptr<Shape>` 목록을 보유합니다. 후대에 shape ownership이
`unique_ptr`/private storage로 바뀌지만 이 SHA에서는 아직 public shared storage입니다.

```cpp
class Scene {
public:
    int width;
    int height;
    bool hasResolution;
    bool hasAmbient;
    bool hasCamera;
    double ambientRatio;
    Color ambientColor;
    Color background;
    Camera camera;
    std::vector<Light> lights;
    std::vector<std::shared_ptr<Shape>> shapes;

    void addShape(std::shared_ptr<Shape> shape);
    void addLight(const Light& light);
};
```

이 aggregate는 parser가 채우고 renderer가 읽는 첫 통합 경계입니다. 아직 BVH나 acceleration
ready state는 없습니다.

### `41a1d6bbe5ef` — 선형 최근접 탐색이 의미 기준선이 되다

**중요도** `A` · **태그** `CORE, SCENE`

```cpp
bool Scene::intersect(const Ray& ray,
                      double t_min,
                      double t_max,
                      HitRecord& hit) const {
    bool found = false;
    double closest = t_max;
    HitRecord candidate;

    for (const std::shared_ptr<Shape>& shape : shapes) {
        if (!shape)
            continue;
        if (shape->intersect(ray, t_min, closest, candidate)) {
            found = true;
            closest = candidate.t;
            hit = candidate;
        }
    }
    return found;
}
```

`closest`를 다음 도형의 `t_max`로 전달하므로 뒤의 도형은 이미 발견한 hit보다 먼 결과를
게시할 수 없습니다. 이 구현은 단순하지만 이후 BVH가 결과를 재배열하면서도 보존해야 할 reference
semantics입니다.

이 시점의 equal-`t` 결과는 각 도형의 `t > t_max` 검사와 순회 순서에 의해 뒤 도형이 같은
`t`로 record를 갱신할 수 있습니다. 후대 `AccelMode`가 이 동작을 원래 shape index로 명시화합니다.

## 3. Parser는 한 번에 완성되지 않았다

### `3545eb1e82df` — 오류 위치와 line token

**중요도** `B` · **태그** `PARSER, PRACTICAL`

`ParseError`는 source name과 line number를 저장하고 message를 `source:line: message` 형태로
만듭니다. `trim`과 whitespace tokenization도 추가됩니다. 아직 line loop와 directive 처리는
활성화되지 않은 기반 단계입니다.

### `7bba0af26d17` — 숫자를 단순 변환하지 않고 의미 범위까지 검증

**중요도** `B` · **태그** `PARSER, EDGE`

- `std::stod`가 token 전체를 소비하고 `std::isfinite(value)`여야 합니다.
- resolution integer는 `1..INT_MAX`입니다.
- ratio는 `[0,1]`입니다.
- diameter/height 같은 scalar는 양수여야 합니다.
- arity 오류는 기대 형식과 실제 argument 수를 source-located error로 만듭니다.

```cpp
const double value = std::stod(token, &parsed);
if (parsed != token.size() || !std::isfinite(value))
    throw std::invalid_argument("not a finite number");
```

이 commit은 helper를 준비할 뿐 directive에서 아직 호출하지 않습니다.

### `d4a24901051a` — comma token을 내부 값으로 변환

**중요도** `B` · **태그** `PARSER`

`x,y,z`는 정확히 세 부분이어야 하고 각 부분은 앞의 finite parser를 통과합니다. RGB는 각 channel이
정수 `0..255`인지 검사한 뒤 `[0,1]` double color로 변환합니다. 빈 성분이나 추가 comma는
명시적으로 실패합니다.

### `6bff18bf0bac` — dispatch 골격의 정확한 의미

**중요도** `A` · **태그** `ARCH, PARSER`

기존 문서의 표현과 달리 이 SHA는 실제 directive를 처리하지 않습니다. exact source는 다음
순서를 활성화합니다.

```cpp
while (std::getline(input, line)) {
    ++line_number;
    const std::size_t comment = line.find('#');
    if (comment != std::string::npos)
        line.erase(comment);
    line = trim(line);
    if (line.empty())
        continue;

    const std::vector<std::string> tokens = splitTokens(line);
    const std::string& id = tokens[0];

    throw ParseError(source_name,
                     line_number,
                     "unknown directive '" + id + "'");
}
```

즉 이 commit의 역할은 “모든 지시어가 이미 dispatch된다”가 아니라, line lifecycle과
unknown-directive failure를 고정하고 다음 commit들이 `if/else if` handler를 끼워 넣을 grammar
경계를 만드는 것입니다.

### `9aef46929554` — `R`과 `A`

**중요도** `B` · **태그** `PARSER`

`R width height`는 positive integer 두 개와 singleton duplicate rule을 사용합니다.
`A ratio r,g,b`는 ratio `[0,1]`, RGB channel 범위, duplicate rule을 적용한 뒤
`scene.hasResolution`/`scene.hasAmbient`를 publish합니다.

### `5e21e6900fd9` — `C`와 `L`

**중요도** `B` · **태그** `PARSER`

카메라는 position, nonzero direction, `0 < fov < 180`을 요구하고 direction을 정규화합니다.
`C`는 singleton입니다. `L`은 position, brightness ratio, color를 받아 여러 개 추가할 수 있습니다.

### `c17a4b5737d2` — `sp`와 `pl`

**중요도** `B` · **태그** `PARSER, GEOMETRY`

sphere는 positive diameter를 radius로 나누어 생성합니다. plane은 nonzero normal을 요구합니다.
둘 모두 diffuse material color를 만들고, 검증이 끝난 뒤에만 `scene.addShape`를 호출합니다.

### `d0cc38dd5762` — `cy`

**중요도** `B` · **태그** `PARSER, GEOMETRY`

원기둥은 center, nonzero axis, positive diameter, positive height, color를 요구합니다.
parser가 diameter를 `* 0.5`하여 geometry constructor의 radius 단위로 변환합니다.

### `1e1fda47d913` — 부분 문법을 usable loader로 닫다

**중요도** `A` · **태그** `PARSER, INTEGRATION`

line parsing이 끝난 뒤 `R`, `A`, `C`가 없으면 line `0`의 source-level error로 거부합니다.

```cpp
if (!scene.hasResolution)
    throw ParseError(source_name, 0, "missing R width height directive");
if (!scene.hasAmbient)
    throw ParseError(source_name, 0, "missing A ratio r,g,b directive");
if (!scene.hasCamera)
    throw ParseError(source_name, 0, "missing C pos dir fov directive");
```

또한 `parseSceneText`, `parseSceneFile`, `loadScene`을 추가하고 valid `basic.rt`와 invalid ambient
fixture를 제공합니다. 이 commit이 처음으로 지시어 자체를 구현한 것은 아닙니다. 앞선 여섯 단계가
만든 parser를 파일·문자열 입력에서 사용할 수 있는 완결된 boundary로 만든 것입니다.

## 4. Scene에서 pixel color까지

### `e6da5f987b97` — pixel center를 world-space ray로 바꾸다

**중요도** `A` · **태그** `CORE, RAY_PIPELINE`

카메라 방향을 forward로 만들고, world-up과 거의 평행하면 다른 up 후보를 선택합니다.
`right = normalize(cross(up, forward))`, `true_up = cross(forward, right)`로 직교 frame을 만들고
FOV와 aspect ratio로 viewport 크기를 계산합니다.

pixel `(x,y)`는 corner가 아니라 `(x+0.5,y+0.5)` 중심에서 sample됩니다. image y가 아래로
증가하는 convention을 camera-space y에 반전해 적용합니다. 이 commit은 한 pixel에 한 ray만
생성하며 anti-aliasing은 없습니다.

### `e8b7dc42a52c` — visible light만 diffuse contribution을 더한다

**중요도** `S` · **태그** `CORE, RAY_PIPELINE, RISK`

miss는 `scene.background`를 반환합니다. hit는 ambient부터 시작하고 각 point light에 대해 다음을
수행합니다.

1. light 방향과 거리를 계산한다.
2. `max(0, dot(normal, light_direction))`이 0이면 건너뛴다.
3. self-intersection을 피하려고 shadow origin을 `hit.point + hit.normal * kRayTMin`으로 옮긴다.
4. light 직전까지만 shadow ray를 검사한다.
5. occluded가 아니면 albedo × light color × brightness × cosine을 더한다.

```cpp
const Vec3 shadow_origin = hit.point + hit.normal * kRayTMin;
if (isOccluded(scene,
               Ray(shadow_origin, light_direction),
               distance_to_light))
    continue;
```

`isOccluded`는 최대 거리를 `distance - kRayTMin`으로 줄여 light 뒤의 도형이 그림자를 만들지
않게 합니다. 결과는 `clampColor`로 제한됩니다. 이 SHA에는 metal recursion이 없으며 모든 surface는
diffuse입니다.

### `c742b2401e52` — 최초의 완전한 serial image loop

**중요도** `S` · **태그** `ARCH, CORE, RAY_PIPELINE`

```cpp
for (int y = 0; y < scene.height; ++y) {
    for (int x = 0; x < scene.width; ++x) {
        const Ray ray = makeCameraRay(scene.camera,
                                      scene.width,
                                      scene.height,
                                      x + 0.5,
                                      y + 0.5);
        const Color clamped = clampColor(traceRay(scene, ray));
        image.pixels[offset++] =
            static_cast<unsigned char>(std::lround(clamped.x * 255.0));
        image.pixels[offset++] =
            static_cast<unsigned char>(std::lround(clamped.y * 255.0));
        image.pixels[offset++] =
            static_cast<unsigned char>(std::lround(clamped.z * 255.0));
    }
}
```

순회는 row-major이고, 한 thread가 한 offset을 단조 증가시키므로 pixel write order가
결정적입니다. 이후 tile renderer와 worker pool은 이 pixel kernel의 결과를 바꾸지 않아야 합니다.

역사적 한계도 남아 있습니다. 이 SHA의 constructor는
`static_cast<std::size_t>(width_value * height_value * 3)`처럼 **int 곱셈 뒤** 변환합니다.
양수/overflow 검사는 Thread 6의 `71096cd311d5`에서 추가됩니다.

## 5. 내부 image를 외부 artifact로 노출하다

### `1bc7cacd30aa` — P3 PPM과 최초 checksum

**중요도** `A` · **태그** `OUTPUT, DETERMINISM`

writer는 `P3`, width/height, max value `255`, 각 RGB triplet을 text로 출력합니다. checksum은
image dimensions와 pixel bytes를 순서대로 hash해 렌더 결과 비교 수단을 제공합니다.

하지만 exact SHA에는 두 후속 수정 대상이 있습니다.

- PPM index가 `static_cast<size_t>((y * width + x) * 3)`처럼 int 연산 뒤 변환됩니다.
- FNV-1a offset basis가 표준 64-bit 값보다 한 자리 짧습니다.

따라서 이 commit의 올바른 설명은 “최초의 deterministic representation을 노출했다”이지,
“최종적으로 overflow-safe하고 표준 FNV-1a인 output을 완성했다”가 아닙니다.

### `b983f0ea2744` — CLI orchestration

**중요도** `B` · **태그** `CLI, INTEGRATION`

CLI는 입력 `.rt`, 출력 `.ppm`, 선택적 `--checksum`만 허용합니다. 정상 경로는
`loadScene → renderScene → writePpmFile → checksum 출력`입니다. usage 오류는 status 2,
parse/render/output exception은 diagnostic과 status 1입니다.

### `d05a6ab48bb1` — 첫 end-to-end smoke의 범위

**중요도** `B` · **태그** `TEST, INTEGRATION`

shell test는 다음을 확인합니다.

- unknown directive scene은 실패하고 output file을 만들지 않는다.
- 고정 64×32 scene을 두 번 렌더한다.
- PPM header가 `P3`, `64 32`, `255`인지 확인한다.
- checksum이 16자리 hex 형태이고 두 실행에서 같다.
- 두 PPM 파일이 byte-identical이다.

이 테스트는 vertical slice가 실제 CLI에서 이어지고 반복 실행이 같은 artifact를 만든다는
integration evidence입니다. 그러나 checksum의 **정확한 기준값**, 모든 pixel의 물리적 정확성,
large dimension safety, output failure 시 기존 파일 보존은 증명하지 않습니다. 그 항목은 후속
Thread 6에서 별도 테스트로 고정됩니다.

## 최종 실행 흐름

```text
.rt text
  -> line/comment/token 처리
  -> finite/range/vector/color 검증
  -> R/A/C/L/sp/pl/cy dispatch
  -> 필수 R/A/C 확인
  -> Scene(camera, lights, shared shapes)
  -> pixel-center camera ray
  -> Scene::intersect linear closest hit
  -> miss: background
     hit: ambient + visible Lambertian direct light
  -> clamp + RGB byte quantization
  -> row-major Image
  -> P3 serialization + 초기 checksum
  -> CLI output
```

## 이 Thread가 보장하는 것

- 모든 primitive가 동일한 hit window와 oriented-normal contract를 사용합니다.
- parser가 source-located error로 잘못된 값과 문법을 거부하고 유효한 Scene을 만듭니다.
- 카메라, 교차, shadow, direct lighting, quantization이 한 개의 serial image path로 연결됩니다.
- 같은 smoke fixture의 반복 CLI 실행은 같은 checksum 형태와 같은 PPM bytes를 기대합니다.

## 이 Thread가 의도적으로 다루지 않는 것

- 큰 유한 벡터의 overflow-safe normalization: Thread 2
- BVH build/traversal과 linear equivalence: Thread 3
- metal reflection과 depth syntax: Thread 4
- tile concurrency와 worker failure: Thread 5
- checked image arithmetic, 표준 FNV-1a, malformed storage, atomic file publication: Thread 6
- CMake/CTest/sanitizer/CI verification matrix: Thread 7

이 항목들은 첫 renderer의 후속 개선이지, 최초 SHA에 이미 존재했던 기능이 아닙니다.
