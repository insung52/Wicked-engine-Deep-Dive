# Capsule Shadow

Wicked Engine의 Capsule Shadow는 Shadow Map 없이 **수학적 계산만으로** 캐릭터의 부드러운 그림자를 생성하는 기법이다.

## 목차
1. [개요](#1-개요)
2. [Shadow Map과의 차이](#2-shadow-map과의-차이)
3. [핵심 원리: Spherical Cap Intersection](#3-핵심-원리-spherical-cap-intersection)
4. [전체 파이프라인](#4-전체-파이프라인)
5. [상세 구현](#5-상세-구현)
6. [에디터 파라미터](#6-에디터-파라미터)

---

## 1. 개요

### 문제 상황
캐릭터가 바닥에 서 있을 때, 발 밑에 자연스러운 그림자가 필요하다. Shadow Map으로도 가능하지만:
- 해상도 제한으로 계단 현상 발생
- 부드러운 접지 그림자를 위해 높은 비용 필요

### 해결책: Capsule Shadow
캐릭터의 신체 부위를 **캡슐(Capsule)** 로 근사하고, 각 픽셀에서 "이 캡슐이 빛을 얼마나 가리는가"를 **기하학적으로 계산**한다.

```
        ☀️ 태양 (광원)
         \
          \  빛의 방향
           \
      ┌──────────┐
      │  캡슐     │  ← 캐릭터 다리 (A-B 선분 + 반지름)
      │(occluder)│
      └──────────┘
           │
           ▼
    ═══════════════════  ← 바닥 (셰이딩 중인 픽셀 P)
       어두운 영역
```

---

## 2. Shadow Map과의 차이

| 구분 | Shadow Map | Capsule Shadow |
|------|------------|----------------|
| **원리** | 광원 시점에서 depth 렌더링 후 비교 | 수학 공식으로 차폐량 직접 계산 |
| **추가 패스** | 필요 (shadow map 렌더링) | 불필요 |
| **정확도** | 정확한 실루엣 | 캡슐로 근사 (대략적) |
| **품질** | 해상도 의존 (계단 현상) | 항상 부드러움 |
| **비용** | 해상도에 비례 | 캡슐 개수에 비례 |
| **용도** | 모든 물체의 그림자 | 캐릭터 접지감, Ambient Occlusion |

### 핵심 차이
Shadow Map은 **"이 픽셀이 그림자 안에 있는가? (Yes/No)"** 를 판단한다.
Capsule Shadow는 **"이 픽셀이 캡슐에 의해 얼마나(0~1) 가려지는가?"** 를 계산한다.

---

## 3. 핵심 원리: Spherical Cap Intersection

Capsule Shadow의 수학적 기반은 **Oat & Sander 2007, "Ambient Aperture Lighting"** 논문이다.

### 3.1 문제 정의: 캡슐이 빛을 얼마나 가리는가?

바닥의 한 점 P를 셰이딩할 때, 위에 있는 캡슐이 빛을 얼마나 가리는지 계산해야 한다.

```
                    ☀️ 태양 (무한히 먼 광원)
                     ↓ 빛 방향
                     ↓
              A ●━━━━━━━━━● B      ← 캡슐 (캐릭터 다리)
                 ╲       ╱
                   ╲   ╱  반지름 r
                     ╳
                     │
                     │
    ─────────────────●─────────────────  바닥
                     P (셰이딩할 픽셀)
```

**핵심 질문**: P에서 봤을 때, 캡슐이 "빛이 오는 방향"을 얼마나 가리고 있는가?

---

### 3.2 단계 1: 캡슐을 구체로 단순화

캡슐은 선분(A→B) + 반지름으로 정의된다. 하지만 선분 전체를 계산하기는 복잡하므로, **캡슐 축 위에서 가장 영향을 주는 한 점**을 찾아 그 점에서의 **구체(sphere)** 로 근사한다.

```
빛 방향 L (태양 방향)
    ↘
      A ●━━━━━●━━━━━━━━● B   ← 캡슐 축
              ↑
         여기가 "가장 영향 주는 점"
         이 점을 중심으로 한 구체로 계산
              │
              │
              P
```

#### 왜 한 점만 찾으면 되는가?

캡슐 전체를 적분하는 대신, **빛 방향과 표면 위치를 고려했을 때 가장 그림자에 기여하는 점** 하나만 찾는다. 이 점에서의 구체 차폐가 전체 캡슐 차폐의 좋은 근사가 된다.

#### 최적점 t 계산 공식

```hlsl
float3 Ld = capsuleB - capsuleA;     // 캡슐 축 벡터
float3 L0 = capsuleA - pos;          // A에서 P로의 벡터

float a = dot(cone.xyz, Ld);         // 빛 방향과 캡슐 축의 정렬도
float t = saturate(dot(L0, a * cone.xyz - Ld) / (dot(Ld, Ld) - a * a));

float3 optimalPoint = capsuleA + t * Ld;  // 캡슐 축 위의 최적점
```

**t의 의미**:
- t = 0 → A점이 가장 영향을 줌
- t = 1 → B점이 가장 영향을 줌
- t = 0.5 → 중간점이 가장 영향을 줌

**기하학적 해석**:
이 공식은 "빛 방향 cone의 축"과 "P에서 캡슐을 바라보는 방향"이 가장 잘 정렬되는 캡슐 축 위의 점을 찾는다.

```
빛 방향 (cone.xyz)
      ↘
        A ●━━━━●━━━━━━● B
           t=0  t=0.4  t=1
                 ↑
            이 점에서 빛 방향과
            P→점 방향이 가장 비슷함
                 │
                 P

t=0.4인 이유: 빛이 비스듬히 오므로
중간 쪽 보다는 A쪽이 P에 더 많은 그림자를 드리움
```

---

### 3.3 단계 2: 구체가 차지하는 "각도" 계산

이제 최적점에서의 구체가 P의 시점에서 **얼마나 큰 각도**를 차지하는지 계산한다.

```
                     구체 (반지름 r)
                    ┌───○───┐
                    │   │   │
                    └───┼───┘
                        │
             거리 d      │
         ──────────────►│
                        │
                        P

P에서 봤을 때 구체가 차지하는 반각 θ:
tan(θ) = r / d
θ = arctan(r / d)
```

그런데 코드에서는 `arctan` 대신 다른 방식을 쓴다:

```hlsl
float occluderLength2 = dot(occluder, occluder);  // d² (거리의 제곱)
float cosTheta = sqrt(occluderLength2 / (sqr(sphere.w) + occluderLength2));
```

이것은 다음과 같다:
```
cosTheta = sqrt(d² / (r² + d²))
         = d / sqrt(r² + d²)
         = cos(arctan(r/d))
```

**왜 cosTheta를 직접 계산하는가?**
- `arctan`을 계산한 후 다시 `cos`를 취하는 것보다
- 직접 `cosTheta`를 계산하는 것이 더 빠르다
- 피타고라스 정리를 이용한 항등식

```
         r (반지름)
         │
         │
         └───────── d (거리)
              ╲
               ╲ sqrt(r² + d²)
                ╲
                 θ

cos(θ) = d / sqrt(r² + d²)
```

---

### 3.4 단계 3: 왜 acos를 쓰는가? - Spherical Cap의 각도 표현

**Spherical Cap**이란 구면 위의 "모자" 형태 영역이다.

```
              구면 (단위 구)
            ╱─────────────╲
          ╱   ┌─────────┐   ╲
         │    │Spherical│    │
         │    │   Cap   │    │    ← 반각 r인 영역
         │    └─────────┘    │
         │         ↑         │
         │      중심 방향     │
          ╲                 ╱
            ╲─────────────╱
                  P
```

Cap의 크기는 **반각(half-angle) r** 로 정의된다. 그런데 우리가 가진 것은:
- `cosTheta` (구체가 차지하는 각도의 코사인)
- `cosPhi` (빛 방향과 차폐물 방향 사이 각도의 코사인)

**교집합 계산에는 실제 각도(라디안)가 필요하다!**

```hlsl
float r1 = acosFastPositive(cosCap1);  // cos(θ) → θ (라디안)
float r2 = cap2;                        // 이미 라디안
float d  = acosFast(cosDistance);       // cos(φ) → φ (라디안)
```

**acos가 필요한 이유**:
- 입력: cos 값들 (내적으로 쉽게 계산됨)
- 필요: 실제 각도 (교집합 조건 판단에 필요)
- 변환: `acos(cos값) = 각도`

```
두 Cap의 관계를 판단하려면 각도가 필요:

  r1 + r2 > d  →  겹침
  r1 + r2 = d  →  맞닿음
  r1 + r2 < d  →  분리됨

여기서 r1, r2, d는 모두 각도(라디안)
```

#### acosFast 최적화

표준 `acos`는 GPU에서 비싸므로, Lagarde 2014의 근사 공식을 사용:

```hlsl
float acosFast(float x) {
    // 최대 오차 9.0×10⁻³
    float y = abs(x);
    float p = -0.1565827 * y + 1.570796;  // 1차 근사
    p *= sqrt(1.0 - y);
    return x >= 0.0 ? p : PI - p;
}
```

---

### 3.5 단계 4: Spherical Cap 교집합 계산

이제 두 개의 Spherical Cap이 있다:

**1) Occluder Cap (차폐물)**
- 중심 방향: P에서 구체를 향하는 방향
- 반각: θ (구체가 차지하는 각도)

**2) Light Cone (빛 영역)**
- 중심 방향: 태양 반대 방향 (빛이 "들어오는" 방향, -L)
- 반각: 설정값 (기본 45°)

```
        P에서 본 상반구 (방향 공간)

              ╱───────────────╲
            ╱                   ╲
           │      ┌─────┐        │
           │      │차폐물│        │ ← Occluder Cap (반각 θ)
           │      │ Cap │        │
           │      └──┬──┘        │
           │         │           │
           │    d(각거리)        │
           │         │           │
           │      ┌──┴──┐        │
           │      │Light│        │ ← Light Cone (반각 45°)
           │      │Cone │        │
            ╲     └─────┘       ╱
              ╲───────────────╱

두 Cap 사이의 각거리 = d = acos(cosPhi)
cosPhi = dot(occluderDir, lightDir)
```

#### 교집합 케이스 분류

```hlsl
float sphericalCapsIntersection(float cosCap1, float cosCap2, float cap2, float cosDistance) {
    float r1 = acosFastPositive(cosCap1);  // 차폐물 cap 반각
    float r2 = cap2;                        // 빛 cone 반각
    float d  = acosFast(cosDistance);       // 두 중심 사이 각거리

    // ─────────────────────────────────────────────
    // Case 1: 한 Cap이 다른 Cap에 완전히 포함됨
    // ─────────────────────────────────────────────
    //
    //     ┌─────────────────┐
    //     │  Light Cone     │
    //     │    ┌─────┐      │
    //     │    │Occl.│      │  ← 차폐물이 빛 cone 안에 완전히 있음
    //     │    └─────┘      │     = 차폐물 전체가 빛을 가림
    //     └─────────────────┘
    //
    //     조건: min(r1,r2) <= max(r1,r2) - d
    //
    if (min(r1, r2) <= max(r1, r2) - d) {
        return 1.0 - max(cosCap1, cosCap2);  // 작은 cap의 전체 면적
    }

    // ─────────────────────────────────────────────
    // Case 2: 두 Cap이 완전히 분리됨
    // ─────────────────────────────────────────────
    //
    //     ┌─────┐           ┌─────┐
    //     │Occl.│           │Light│  ← 서로 안 겹침
    //     └─────┘           └─────┘     = 차폐 없음
    //
    //     조건: r1 + r2 <= d
    //
    else if (r1 + r2 <= d) {
        return 0.0;  // 교집합 없음
    }

    // ─────────────────────────────────────────────
    // Case 3: 부분적으로 겹침
    // ─────────────────────────────────────────────
    //
    //           ┌─────┐
    //           │     │
    //        ┌──┼──┐  │  ← 일부만 겹침
    //        │  │▓▓│  │     = 겹치는 부분만큼 가림
    //        │  └──┼──┘
    //        │     │
    //        └─────┘
    //
    float delta = abs(r1 - r2);
    float x = 1.0 - saturate((d - delta) / max(r1 + r2 - delta, 0.0001));
    float area = sqr(x) * (-2.0 * x + 3.0);  // smoothstep 근사
    return area * (1.0 - max(cosCap1, cosCap2));
}
```

---

### 3.6 단계 5: 최종 Occlusion 계산

```hlsl
float directionalOcclusionSphere(float3 pos, float4 sphere, float4 cone) {
    // 1. 차폐물 방향 계산
    float3 occluder = sphere.xyz - pos;
    float occluderLength2 = dot(occluder, occluder);
    float3 occluderDir = occluder * rsqrt(occluderLength2);

    // 2. 빛 방향과 차폐물 방향 사이 각도 (코사인)
    float cosPhi = dot(occluderDir, cone.xyz);

    // 3. 차폐물이 차지하는 각도 (코사인)
    float cosTheta = sqrt(occluderLength2 / (sqr(sphere.w) + occluderLength2));

    // 4. 빛 cone의 반각 (코사인)
    float cosCone = cos(cone.w);

    // 5. 교집합 계산 → 가려지는 비율
    float intersection = sphericalCapsIntersection(cosTheta, cosCone, cone.w, cosPhi);

    // 6. 빛 cone 전체 면적 대비 가려지는 비율 = occlusion
    //    (1 - cosCone)은 빛 cone의 입체각에 비례
    float occlusion = intersection / (1.0 - cosCone);

    // 7. 반환: 1 = 완전히 밝음, 0 = 완전히 가림
    return 1.0 - occlusion;
}
```

---

### 3.7 전체 과정 요약

```
┌─────────────────────────────────────────────────────────────────┐
│ 입력: 픽셀 위치 P, 캡슐 (A, B, 반지름), 빛 방향                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 캡슐 축에서 최적점 찾기                                        │
│    t = (빛 방향과 P 위치를 고려한 공식)                            │
│    최적점 = A + t × (B - A)                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 구체가 P에서 차지하는 각도 계산                                 │
│    cosθ = 거리 / sqrt(거리² + 반지름²)                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 빛 방향과 차폐물 방향 사이 각도                                 │
│    cosφ = dot(차폐물방향, 빛방향)                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 각도로 변환 (acos)                                            │
│    θ = acos(cosθ)  ← 차폐물 cap 반각                             │
│    φ = acos(cosφ)  ← 두 cap 중심 사이 거리                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Spherical Cap 교집합                                          │
│    - 완전 포함 / 완전 분리 / 부분 겹침 판단                         │
│    - 겹치는 면적 계산                                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 최종 Occlusion                                                │
│    occlusion = 교집합 / 빛cone면적                                │
│    return 1 - occlusion  (1=밝음, 0=어두움)                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 전체 파이프라인

### 4.1 흐름도

```
┌─────────────────────────────────────────────────────────────────┐
│                    CPU: Visibility Update                        │
├─────────────────────────────────────────────────────────────────┤
│  1. Humanoid의 래그돌 바디파트(캡슐들) 수집                        │
│  2. Frustum Culling으로 보이는 캡슐만 필터링                      │
│  3. 카메라 거리순 정렬 (가까운 것 우선)                            │
│  4. ShaderEntity 배열에 기록                                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  GPU: Light Culling (Compute)                    │
├─────────────────────────────────────────────────────────────────┤
│  5. 타일별로 영향 주는 캡슐 목록 생성                              │
│  6. Frustum + AABB + Depth Mask 테스트                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   GPU: Shading (Pixel Shader)                    │
├─────────────────────────────────────────────────────────────────┤
│  7. 각 픽셀에서 타일의 캡슐들 순회                                 │
│  8. 각 캡슐에 대해 directionalOcclusionCapsule() 계산             │
│  9. 모든 occlusion 값 누적 곱                                     │
│  10. surface.occlusion에 최종 적용                                │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 데이터 흐름

```
HumanoidComponent                ShaderEntity              Pixel Shader
┌──────────────┐               ┌──────────────┐          ┌──────────────┐
│ ragdoll_     │               │ position (A) │          │ surface.P    │
│ bodyparts[]  │ ──collect──▶  │ tip (B)      │ ──tile─▶ │ light cone   │
│  - capsule   │               │ radius       │  cull    │ occlusion    │
└──────────────┘               │ flags        │          └──────────────┘
                               └──────────────┘
```

---

## 5. 상세 구현

### 5.1 CPU: 캡슐 수집 (wiRenderer.cpp)

Humanoid 컴포넌트에서 래그돌 바디파트의 캡슐들을 수집한다.

```cpp
// wiRenderer.cpp - UpdateVisibility()

// Ragdoll GPU colliders 수집
for (size_t i = 0; i < vis.scene->humanoids.GetCount(); ++i)
{
    const HumanoidComponent& humanoid = vis.scene->humanoids[i];
    const bool capsule_shadow = !humanoid.IsCapsuleShadowDisabled();

    // 래그돌의 각 바디파트 (몸통, 팔, 다리 등)
    for (auto& bodypart : humanoid.ragdoll_bodyparts)
    {
        // Frustum Culling
        Sphere sphere = bodypart.capsule.getSphere();
        sphere.radius += CAPSULE_SHADOW_AFFECTION_RANGE;  // 2.0 추가
        if (!vis.camera->frustum.CheckSphere(sphere.center, sphere.radius))
            continue;

        // ColliderComponent로 변환
        ColliderComponent collider;
        collider.shape = ColliderComponent::Shape::Capsule;
        collider.capsule = bodypart.capsule;
        collider.SetCapsuleShadowEnabled(capsule_shadow);
        collider.dist = DistanceSquared(sphere.center, vis.camera->Eye);

        vis.visibleColliders.push_back(collider);
    }

    // 발 캡슐 추가 (래그돌에 미포함)
    if (capsule_shadow)
    {
        // RightFoot → RightToes
        // LeftFoot → LeftToes
        // 발-발가락 사이를 캡슐로 생성
    }
}

// 카메라 거리순 정렬 (가까운 것 우선 처리)
std::sort(vis.visibleColliders.begin(), vis.visibleColliders.end(),
    [](const ColliderComponent& a, const ColliderComponent& b) {
        return a.dist < b.dist;
    });
```

### 5.2 CPU: ShaderEntity 기록 (wiRenderer.cpp)

수집된 캡슐들을 GPU가 읽을 수 있는 ShaderEntity 배열에 기록한다.

```cpp
// wiRenderer.cpp - UpdatePerFrameData()

for (size_t i = 0; i < vis.visibleColliders.size(); ++i)
{
    const ColliderComponent& collider = vis.visibleColliders[i];
    ShaderEntity shaderentity = {};

    // 캡슐 그림자용 플래그 설정
    if (collider.IsCapsuleShadowEnabled())
    {
        shaderentity.SetFlags(ENTITY_FLAG_CAPSULE_SHADOW_COLLIDER);
    }

    switch (collider.shape)
    {
    case ColliderComponent::Shape::Capsule:
        shaderentity.SetType(ENTITY_TYPE_COLLIDER_CAPSULE);
        shaderentity.position = collider.capsule.base;           // A점
        shaderentity.SetColliderTip(collider.capsule.tip);       // B점
        shaderentity.SetRange(collider.capsule.radius);          // 반지름
        break;
    // ... 다른 shape 처리
    }

    std::memcpy(entityArray + entityCounter, &shaderentity, sizeof(ShaderEntity));
    entityCounter++;
}
```

### 5.3 GPU: Light Culling (lightCullingCS.hlsl)

타일별로 영향 주는 캡슐들을 걸러낸다.

```hlsl
// lightCullingCS.hlsl

// Capsule shadows culling
for (uint i = forces().first_item() + groupIndex; i < forces().end_item();
     i += TILED_CULLING_THREADSIZE * TILED_CULLING_THREADSIZE)
{
    ShaderEntity entity = load_entity(i);

    // 캡슐 그림자 콜라이더만 처리
    if ((entity.GetFlags() & ENTITY_FLAG_CAPSULE_SHADOW_COLLIDER) == 0)
        continue;

    float3 A = entity.position;
    float3 B = entity.GetColliderTip();
    half radius = entity.GetRange() * CAPSULE_SHADOW_BOLDEN;  // 1.2배 확대

    // 캡슐을 감싸는 구체로 컬링
    float3 center = lerp(A, B, 0.5);
    half range = distance(center, A) + radius + CAPSULE_SHADOW_AFFECTION_RANGE;

    float3 positionVS = mul(GetCamera().view, float4(center, 1)).xyz;
    Sphere sphere = { positionVS.xyz, range };

    // Frustum 테스트
    if (SphereInsideFrustum(sphere, GroupFrustum, nearClipVS, maxDepthVS))
    {
        AppendEntity_Transparent(i);

        // AABB 테스트 (더 정밀)
        if (SphereIntersectsAABB(sphere, GroupAABB))
        {
#ifdef ADVANCED_CULLING
            // Depth Mask 테스트
            if (depth_mask & ConstructEntityMask(minDepthVS, __depthRangeRecip, sphere))
#endif
            {
                AppendEntity_Opaque(i);
            }
        }
    }
}
```

### 5.4 GPU: 캡슐 → 구체 차폐 계산 (capsuleShadowHF.hlsli)

캡슐의 축에서 가장 영향을 주는 점을 찾고, 그 점에서 구체 차폐를 계산한다.

```hlsl
// capsuleShadowHF.hlsli

// 구체에 의한 방향성 차폐
float directionalOcclusionSphere(float3 pos, float4 sphere, float4 cone) {
    // pos: 셰이딩 중인 픽셀 위치
    // sphere.xyz: 구체 중심, sphere.w: 반지름
    // cone.xyz: 빛 방향, cone.w: cone 반각

    float3 occluder = sphere.xyz - pos;
    float occluderLength2 = dot(occluder, occluder);
    float3 occluderDir = occluder * rsqrt(occluderLength2);

    // cosPhi: 빛 방향과 차폐물 방향 사이 각도의 코사인
    float cosPhi = dot(occluderDir, cone.xyz);

    // cosTheta: 차폐물이 차지하는 입체각 (가깝고 클수록 큼)
    float cosTheta = sqrt(occluderLength2 / (sqr(sphere.w) + occluderLength2));

    float cosCone = cos(cone.w);

    // 구면 캡 교집합으로 차폐량 계산
    return 1.0 - sphericalCapsIntersection(cosTheta, cosCone, cone.w, cosPhi)
               / (1.0 - cosCone);
}

// 캡슐에 의한 방향성 차폐
float directionalOcclusionCapsule(float3 pos, float3 capsuleA, float3 capsuleB,
                                   float capsuleRadius, float4 cone) {
    // 캡슐 축 벡터
    float3 Ld = capsuleB - capsuleA;
    float3 L0 = capsuleA - pos;

    // 빛 방향을 고려하여 캡슐 축 위에서 가장 영향 주는 점 찾기
    float a = dot(cone.xyz, Ld);
    float t = saturate(dot(L0, a * cone.xyz - Ld) / (dot(Ld, Ld) - a * a));
    float3 posToRay = capsuleA + t * Ld;

    // 해당 점에서 구체 차폐 계산
    return directionalOcclusionSphere(pos, float4(posToRay, capsuleRadius), cone);
}
```

**캡슐 축 위 최적점 찾기 시각화:**
```
빛 방향 (cone.xyz)
    ↘
      A ●━━━━━━●━━━━━━● B   ← 캡슐 축
              ↑
         t = 0.4 (예시)
         이 점이 P에 가장 영향을 줌

              ↓
              P  ← 셰이딩 픽셀
```

### 5.5 GPU: 최종 셰이딩 통합 (shadingHF.hlsli)

TiledLighting 함수에서 캡슐 그림자를 적용한다.

```hlsl
// shadingHF.hlsli - TiledLighting()

// Capsule shadows:
[branch]
if ((GetFrame().options & OPTION_BIT_CAPSULE_SHADOW_ENABLED) &&
    !surface.IsCapsuleShadowDisabled() &&
    !forces().empty())
{
    // Light Cone 설정: 태양 반대 방향 + 확산 각도
    half4 cone = half4(
        GetSunDirection() * half3(-1, 1, -1),  // 수평 반전
        GetCapsuleShadowAngle()                 // 기본 45도
    );
    half capsuleshadow = 1;

    // 타일의 캡슐들 순회
    ShaderEntityIterator iterator = forces();
    for (uint bucket = iterator.first_bucket();
         bucket <= iterator.last_bucket() && capsuleshadow > 0;
         ++bucket)
    {
        uint bucket_bits = load_entitytile(flatTileIndex + bucket);
        bucket_bits = iterator.mask_entity(bucket, bucket_bits);

#ifndef ENTITY_TILE_UNIFORM
        // Bucket Scalarizer (Siggraph 2017 - Improved Culling)
        // 웨이브 내 모든 스레드가 같은 엔티티 처리하도록 최적화
        bucket_bits = WaveReadLaneFirst(WaveActiveBitOr(bucket_bits));
#endif

        [loop]
        while (bucket_bits != 0 && capsuleshadow > 0)
        {
            // 다음 처리할 엔티티 인덱스 추출
            const uint bucket_bit_index = firstbitlow(bucket_bits);
            const uint entity_index = bucket * 32 + bucket_bit_index;
            bucket_bits ^= 1u << bucket_bit_index;

            ShaderEntity entity = load_entity(entity_index);

            // 캡슐 그림자 콜라이더만 처리
            if ((entity.GetFlags() & ENTITY_FLAG_CAPSULE_SHADOW_COLLIDER) == 0)
                continue;

            // 캡슐 파라미터 추출
            float3 A = entity.position;
            float3 B = entity.GetColliderTip();
            half radius = entity.GetRange() * CAPSULE_SHADOW_BOLDEN;  // 1.2배

            // ★ 핵심: 캡슐 차폐 계산
            half occ = directionalOcclusionCapsule(surface.P, A, B, radius, cone);

            // 거리 기반 감쇠 (캡슐에서 멀어지면 영향 감소)
            float3 center = lerp(A, B, 0.5);
            half range = distance(center, A) + radius + CAPSULE_SHADOW_AFFECTION_RANGE;
            half range2 = range * range;
            half dist2 = distance_squared(surface.P, center);
            occ = 1 - saturate((1 - occ) * saturate(attenuation_pointlight(dist2, range, range2)));

            // 누적 곱 (여러 캡슐이 겹치면 더 어두워짐)
            capsuleshadow *= occ;
        }
    }

    // Fade 적용 (완전히 검게 되는 것 방지)
    capsuleshadow = lerp(capsuleshadow, 1, GetCapsuleShadowFade());
    capsuleshadow = saturate(capsuleshadow);

    // ★ 최종 적용: surface occlusion에 곱함
    surface.occlusion *= capsuleshadow;
}
```

---

## 6. 에디터 파라미터

### 6.1 전역 설정 (Graphics Window)

| 파라미터 | 범위 | 기본값 | 설명 |
|----------|------|--------|------|
| Capsule Shadows | On/Off | Off | 기능 활성화 |
| Fade | 0~1 | 0.2 | 그림자 최소 밝기 (0=완전 검정 허용) |
| Angle | 0~90° | 45° | 빛 cone의 확산 각도 |

```cpp
// GraphicsWindow.cpp

capsuleshadowCheckbox.Create("Capsule Shadows: ");
capsuleshadowFadeSlider.Create(0, 1, 0.2f, 100, "CapsuleShadow.Fade: ");
capsuleshadowAngleSlider.Create(0, 90, 45, 90, "CapsuleShadow.Angle: ");
```

### 6.2 Humanoid별 설정 (Humanoid Window)

```cpp
// HumanoidWindow.cpp
// 개별 캐릭터의 캡슐 그림자 비활성화 가능
humanoid.SetCapsuleShadowDisabled(true);
```

### 6.3 Material별 설정 (Material Window)

```cpp
// MaterialWindow.cpp
// 특정 머티리얼에서 캡슐 그림자 수신 비활성화
material.SetCapsuleShadowDisabled(true);
```

---

## 7. 상수 및 설정값

```hlsl
// ShaderInterop_Renderer.h

// 캡슐 그림자가 영향을 미치는 추가 범위
static const float CAPSULE_SHADOW_AFFECTION_RANGE = 2.0;

// 캡슐 반지름 확대 비율 (더 넓은 그림자)
static const float CAPSULE_SHADOW_BOLDEN = 1.2;
```

```cpp
// wiRenderer.cpp

bool CAPSULE_SHADOW_ENABLED = false;
float CAPSULE_SHADOW_ANGLE = XM_PIDIV4;  // 45도
float CAPSULE_SHADOW_FADE = 0.2f;
```

---

## 8. 장단점 정리

### 장점
- **부드러운 그림자**: 수학적 계산이라 항상 soft shadow
- **추가 패스 불필요**: 셰이딩 중에 계산 완료
- **해상도 독립적**: Shadow Map처럼 계단 현상 없음
- **자연스러운 접지감**: 캐릭터 발 밑 그림자에 효과적

### 단점
- **근사적**: 정확한 실루엣이 아닌 캡슐 형태
- **캡슐 수 제한**: 많은 캐릭터 시 성능 영향
- **단일 광원**: 현재 태양 방향만 고려

---

## 참고 자료

- Oat, C., & Sander, P. V. (2007). **"Ambient Aperture Lighting"**
- Lagarde, S. (2014). **"Inverse trigonometric functions GPU optimization for AMD GCN architecture"**
- Shadertoy: https://www.shadertoy.com/view/3stcD4
- Siggraph 2017: **"Improved Culling for Tiled and Clustered Rendering"** (Bucket Scalarizer)
