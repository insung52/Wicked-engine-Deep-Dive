# DDGI Raytracing 전체 과정 상세 설명

## 목차
1. [Scene Acceleration Structure 생성](#1-scene-acceleration-structure-생성)
2. [DDGI Raytracing 실행](#2-ddgi-raytracing-실행)
3. [Ray Hit 처리 및 Surface Data 로드](#3-ray-hit-처리-및-surface-data-로드)
4. [Lighting 계산 및 Radiance 저장](#4-lighting-계산-및-radiance-저장)

---

## 1. Scene Acceleration Structure 생성

### 1.1 BLAS (Bottom-Level Acceleration Structure) 생성

**목적:** 각 geometry의 삼각형 데이터를 GPU raytracing을 위한 가속 구조로 변환

**위치:** `GeometryComponent.cpp:1680-1712`

```cpp
// BLAS descriptor 설정
RaytracingAccelerationStructureDesc desc;
desc.type = RaytracingAccelerationStructureDesc::Type::BOTTOMLEVEL;

// Geometry 데이터 설정
desc.bottom_level.geometries.emplace_back();
auto& geometry = desc.bottom_level.geometries.back();
geometry.type = RaytracingAccelerationStructureDesc::BottomLevel::Geometry::Type::TRIANGLES;

// Vertex buffer 정보
geometry.triangles.vertex_buffer = part_buffers.generalBuffer;     // Vertex buffer
geometry.triangles.vertex_byte_offset = part_buffers.vbPosW.offset; // Buffer 내 offset
geometry.triangles.vertex_count = primitive.GetNumVertices();       // Vertex 개수
geometry.triangles.vertex_format = Format::R32G32B32_FLOAT;        // Position format
geometry.triangles.vertex_stride = sizeof(Vertex_POS32W);          // Vertex stride

// Index buffer 정보
geometry.triangles.index_buffer = part_buffers.generalBuffer;      // Index buffer
geometry.triangles.index_format = GetIndexFormat(part_index);      // UINT16 or UINT32
geometry.triangles.index_count = subset.indexCount;                // Index 개수
geometry.triangles.index_offset = part_buffers.ib.offset / stride; // Buffer 내 offset

// BLAS 생성
device->CreateRaytracingAccelerationStructure(&desc, &BLASes[lod]);
```

**핵심:**
- BLAS는 **winding order를 저장하지 않음** (vertex/index 데이터만)
- Object space의 geometry 정보만 포함
- LOD별로 별도의 BLAS 생성

### 1.2 BLAS Build (GPU에서 실행)

**위치:** `SceneUpdate_Detail.cpp:1548-1552`

```cpp
switch (geometry.stateBLAS) {
    case BLAS_STATE_NEEDS_REBUILD:
        device->BuildRaytracingAccelerationStructure(&BLAS, cmd, nullptr);
        break;
    case BLAS_STATE_NEEDS_REFIT:
        // Update만 수행 (topology 변경 없음)
        device->BuildRaytracingAccelerationStructure(&BLAS, cmd, &BLAS);
        break;
}
```

**BuildRaytracingAccelerationStructure 실제 구현:**
**위치:** `GraphicsDevice_DX12.cpp:7320-7390`

- DX12의 `BuildRaytracingAccelerationStructure` 명령 생성
- GPU에서 BVH (Bounding Volume Hierarchy) 구축

---

### 1.3 TLAS (Top-Level Acceleration Structure) Instance 설정

**목적:** 각 object의 transform과 BLAS 참조를 설정, **winding order 플래그 설정**

**위치:** `SceneUpdate_Detail.cpp:674-716`

```cpp
// TLAS instance descriptor 설정
RaytracingAccelerationStructureDesc::TopLevel::Instance instance;

// Transform matrix 설정 (4x3 transpose)
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 4; j++) {
        instance.transform[i][j] = world_matrix.m[j][i];  // Transpose!
    }
}

// Instance 정보
instance.instance_id = args.jobIndex;                    // Instance ID
instance.instance_mask = layermask == 0 ? 0 : 0xFF;     // Visibility mask
instance.bottom_level = &geometry.BLASes[renderable.GetLOD()]; // BLAS 참조
instance.instance_contribution_to_hit_group_index = 0;   // Hit group offset
instance.flags = 0;

// Double-sided material 처리
if (renderable.HasDoubleSideMaterial()) {
    instance.flags |= FLAG_TRIANGLE_CULL_DISABLE;
}

// ★★★ Winding order 플래그 설정 (RHS 대응) ★★★
float det = XMVectorGetX(XMMatrixDeterminant(W));
if (det < 0) {
    // VizMotive uses RHS (Right-Handed System), but DXR operates in LHS
    // RHS CCW geometry is interpreted as CW by DXR
    // Only mirrored transforms (det < 0) need CCW flag
    instance.flags |= FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE;
}

// TLAS instance buffer에 쓰기
void* dest = (void*)((size_t)TLAS_instancesMapped +
                     (size_t)args.jobIndex * device->GetTopLevelAccelerationStructureInstanceSize());
device->WriteTopLevelAccelerationStructureInstance(&instance, dest);
```

**핵심:**
- **Transform determinant < 0 (미러 변환)일 때만** CCW 플래그 설정
- RHS CCW geometry는 DXR(LHS)에서 CW로 해석되므로, 일반 변환(det > 0)에서는 플래그 불필요
- Instance mask로 visibility 제어 가능

### 1.4 TLAS Build

**위치:** `SceneUpdate_Detail.cpp:1602`

```cpp
// TLAS 빌드 (모든 instance를 포함하는 top-level BVH 생성)
device->BuildRaytracingAccelerationStructure(&TLAS, cmd, nullptr);

// Barrier: TLAS 빌드 완료 대기
GPUBarrier barriers[] = { GPUBarrier::Memory(&TLAS) };
device->Barrier(barriers, arraysize(barriers), cmd);
```

**결과:** `scene_acceleration_structure` = TLAS가 shader에서 사용 가능

---

## 2. DDGI Raytracing 실행

### 2.1 DDGI Compute Shader 호출

**위치:** DDGI system이 매 프레임 `ddgi_raytraceCS.hlsl` 실행

**Shader 입력:**
- `scene_acceleration_structure`: TLAS (Line 207에서 사용)
- `rayallocationBuffer`: 어떤 probe의 어떤 ray를 trace할지 정보
- `raycountBuffer`: Probe당 ray 개수

**Thread 구조:**
```hlsl
[numthreads(THREADCOUNT, 1, 1)]  // THREADCOUNT = 32
void main(uint3 DTid : SV_DispatchThreadID, ...)
```

### 2.2 Probe 및 Ray 정보 가져오기

**위치:** `ddgi_raytraceCS.hlsl:36-44`

```hlsl
// Ray allocation buffer에서 정보 읽기
const uint allocCount = rayallocationBuffer[3];           // 총 ray 개수
if(DTid.x >= allocCount) return;                          // 범위 체크

const uint rayAlloc = rayallocationBuffer[4 + DTid.x];   // Ray allocation data
const uint probeIndex = rayAlloc & 0xFFFFF;               // 하위 20bit: probe index
const uint rayIndex = rayAlloc >> 20u;                    // 상위 12bit: ray index
const uint rayCount = raycountBuffer[probeIndex] * DDGI_RAY_BUCKET_COUNT;

// Probe 위치 계산
const uint3 probeCoord = ddgi_probe_coord(probeIndex);   // 3D grid 좌표
const float3 probePos = ddgi_probe_position(probeCoord); // World space 위치
```

**Probe position 계산 상세:**
```hlsl
// ShaderInterop_DDGI.h:120-130
inline float3 ddgi_probe_position(min16uint3 probeCoord) {
    float3 pos = GetScene().ddgi.grid_min + probeCoord * ddgi_cellsize();

    // Dynamic offset 추가 (probe가 geometry 안쪽으로 들어가지 않도록)
    uint probeIndex = ddgi_probe_index(probeCoord);
    StructuredBuffer<DDGIProbe> probe_buffer = ...;
    DDGIProbe probe = probe_buffer[probeIndex];
    float3 offset = unpack_half3(probe.offset);
    offset = offset * ddgi_cellsize() * 0.5;
    pos += offset;

    return pos;
}
```

### 2.3 Ray 방향 계산

**위치:** `ddgi_raytraceCS.hlsl:46-52`

```hlsl
RNG rng;
rng.init(DTid.xx, GetFrame().frame_count);  // Random seed

// Spherical Fibonacci 분포로 균일하게 분산된 ray 방향 생성
const float3x3 random_orientation = (float3x3)g_xTransform;
const float3 raydir = normalize(mul(random_orientation,
                                    spherical_fibonacci(rayIndex, rayCount)));

// Spherical Fibonacci 함수 (Line 25-31)
float3 spherical_fibonacci(float i, float n) {
    float phi = 2.0 * PI * madfrac(i, PHI - 1);        // Golden angle
    float cos_theta = 1.0 - (2.0 * i + 1.0) * (1.0 / n);
    float sin_theta = sqrt(clamp(1.0 - cos_theta * cos_theta, 0.0f, 1.0f));
    return float3(cos(phi) * sin_theta, sin(phi) * sin_theta, cos_theta);
}
```

**핵심:**
- Spherical Fibonacci: 구면 상에 균일하게 분포된 방향 생성
- `random_orientation`: 매 프레임 회전하여 temporal aliasing 감소

### 2.4 Ray 설정

**위치:** `ddgi_raytraceCS.hlsl:197-201`

```hlsl
RayDesc ray;
ray.Origin = probePos;      // Probe 위치에서 시작
ray.TMin = 0;               // 최소 거리 (surface에서 시작하는게 아니므로 0)
ray.TMax = FLT_MAX;         // 최대 거리 (무한)
ray.Direction = normalize(raydir);  // Ray 방향 (정규화)
```

---

## 3. Ray Hit 처리 및 Surface Data 로드

### 3.1 Inline Raytracing 실행

**위치:** `ddgi_raytraceCS.hlsl:203-219`

```hlsl
#ifdef RTAPI  // DXR 1.1 Inline Raytracing 사용
vzRayQuery q;
q.TraceRayInline(
    scene_acceleration_structure,    // ★ TLAS (Section 1.4에서 생성)
    //RAY_FLAG_CULL_BACK_FACING_TRIANGLES |  // 주석: backface도 hit 가능
    RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES |    // Procedural geometry 무시
    RAY_FLAG_FORCE_OPAQUE,                   // 모든 geometry를 opaque로 처리
    push.instanceInclusionMask,              // Instance visibility mask
    ray                                       // Ray descriptor
);

// Ray query 실행 (GPU에서 BVH traverse)
while (q.Proceed());

// Hit 결과 확인
if (q.CommittedStatus() != COMMITTED_TRIANGLE_HIT) {
    // Miss: Environment lighting 처리 (Line 221-240)
    ...
}
```

**TraceRayInline 내부 동작 (GPU):**
1. **BVH Traversal:** TLAS → BLAS 순회하며 ray-triangle intersection 계산
2. **Instance Transform 적용:** TLAS의 transform을 사용하여 ray를 object space로 변환
3. **Closest Hit 찾기:** 가장 가까운 intersection 반환
4. **Winding Order 판정:**
   - Instance의 `FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE` 확인
   - Ray 방향과 삼각형 정점 순서로 front/back face 판정

### 3.2 Hit 정보 추출

**위치:** `ddgi_raytraceCS.hlsl:252-265`

```hlsl
// Hit position 계산
ray.Origin = q.WorldRayOrigin() + q.WorldRayDirection() * q.CommittedRayT();
hit_depth = q.CommittedRayT();  // Hit 거리

// Hit한 primitive (삼각형) 정보
PrimitiveID prim;
prim.init();
prim.primitiveIndex = q.CommittedPrimitiveIndex();  // BLAS 내 삼각형 index
prim.instanceIndex = q.CommittedInstanceID();        // TLAS instance ID
prim.subsetIndex = q.CommittedGeometryIndex();      // Geometry subset index

// ★★★ Front/Back face 판정 ★★★
surface.SetBackface(!q.CommittedTriangleFrontFace());
// q.CommittedTriangleFrontFace() = DXR이 판정한 frontface 여부
// !를 붙이는 이유: DXR의 frontface = shader의 backface 개념과 반대
```

**CommittedTriangleFrontFace() 동작:**
- DXR이 instance의 CCW 플래그와 ray 방향으로 판정
- TRUE: Ray가 삼각형 front side를 hit (vertices가 CCW로 보임)
- FALSE: Ray가 삼각형 back side를 hit (vertices가 CW로 보임)

### 3.3 Surface Data 로드

**위치:** `ddgi_raytraceCS.hlsl:264-287`

```hlsl
// Barycentric coordinates로 surface 정보 보간
if (!surface.load(prim, q.CommittedTriangleBarycentrics()))
    return;  // Load 실패 시 종료

// Backface인 경우 depth 조정
if (surface.IsBackface()) {
    hit_depth *= 0.9;  // 내부로 밀어넣어 shadow leak 방지
}

// Surface 위치 및 View 방향 설정
surface.P = ray.Origin;       // Hit position
surface.V = -ray.Direction;   // View direction (ray 반대 방향)
surface.update();             // Normal, tangent 등 계산
```

**surface.load() 내부 동작 (surfaceHF.hlsli):**
```hlsl
// 1. Vertex buffer에서 3개 정점 데이터 읽기
// 2. Barycentric interpolation으로 속성 보간:
//    - Position, Normal, Tangent, UV, Vertex color 등
// 3. Normal mapping 적용
// 4. ★ Backface인 경우 normal flip (surfaceHF.hlsli:390-393)
if (is_backface && !is_emittedparticle) {
    N = -N;  // Normal 반전
}
```

---

## 4. Lighting 계산 및 Radiance 저장

### 4.1 Direct Lighting (Light Sampling)

**위치:** `ddgi_raytraceCS.hlsl:290-418`

```hlsl
// Random하게 light 하나 선택
const uint light_count = lights().item_count();
const uint light_index = lights().first_item() + rng.next_uint(light_count);
ShaderEntity light = load_entity(light_index);

// Lighting 구조체 초기화
Lighting lighting;
lighting.create(0, 0, 0, 0);

float3 L = 0;      // Light direction
float dist = 0;    // Light distance
float NdotL = 0;   // N dot L

// Light type별 처리
switch (light.GetType()) {
case ENTITY_TYPE_DIRECTIONALLIGHT: {
    dist = FLT_MAX;
    L = light.GetDirection().xyz;  // Light direction
    L += sample_hemisphere_cos(L, rng) * light.GetRadius();  // Soft shadow
    NdotL = saturate(dot(L, surface.N));

    if (NdotL > 0) {
        float3 lightColor = light.GetColor().rgb;

        // Atmospheric scattering (optional)
        if (GetFrame().options & OPTION_BIT_REALISTIC_SKY) {
            lightColor *= GetAtmosphericLightTransmittance(...);
        }

        lighting.direct.diffuse = (half3)lightColor;
    }
    break;
}
case ENTITY_TYPE_POINTLIGHT: {
    // Point light 처리 (Line 328-352)
    L = light.position - surface.P;
    const float dist2 = dot(L, L);
    dist = sqrt(dist2);
    L /= dist;
    NdotL = saturate(dot(L, surface.N));

    if (NdotL > 0 && dist2 < range2) {
        lighting.direct.diffuse = lightColor * attenuation_pointlight(...);
    }
    break;
}
// ... SPOTLIGHT 등
}
```

### 4.2 Shadow Ray Tracing

**위치:** `ddgi_raytraceCS.hlsl:389-417`

```hlsl
if (NdotL > 0 && dist > 0) {
    float3 shadow = 1;

    // Shadow ray 설정
    RayDesc newRay;
    newRay.Origin = surface.P;
    newRay.TMin = 0.001;              // Self-intersection 방지
    newRay.TMax = dist;               // Light까지 거리
    newRay.Direction = normalize(L + max3(surface.sss));  // SSS offset

    // Shadow ray trace
    vzRayQuery q;
    q.TraceRayInline(
        scene_acceleration_structure,
        RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES |
        RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH,  // Any hit으로 충분
        0xFF,
        newRay
    );
    while (q.Proceed());

    // Occluded인지 확인
    shadow = q.CommittedStatus() == COMMITTED_TRIANGLE_HIT ? 0 : shadow;

    if (any(shadow)) {
        // 그림자 없음: lighting 적용
        hit_result += light_count * max(0, shadow * lighting.direct.diffuse * NdotL / PI);
    }
}
```

### 4.3 Indirect Lighting (Previous Frame DDGI)

**위치:** `ddgi_raytraceCS.hlsl:421-429`

```hlsl
// Infinite bounces: 이전 프레임 DDGI probe 샘플링
if (push.frameIndex > 0) {
    float energy_conservation = 0.95;
    energy_conservation /= PI;

    // 이전 프레임 DDGI irradiance 샘플링
    float3 ddgi = ddgi_sample_irradiance(surface.P,
                                         surface.facenormal,
                                         surface.dominant_lightdir,
                                         surface.dominant_lightcolor);
    ddgi *= energy_conservation;
    hit_result += ddgi;  // Indirect lighting 추가
}
```

### 4.4 Final Radiance 계산 및 저장

**위치:** `ddgi_raytraceCS.hlsl:430-438`

```hlsl
// Albedo 곱하기
hit_result *= surface.albedo;

// Emissive 추가
hit_result += surface.emissiveColor;

// Total radiance
radiance += hit_result;

// Ray data buffer에 저장
DDGIRayData rayData;
rayData.direction = ray.Direction;  // Ray 방향
rayData.depth = hit_depth;          // Hit 거리
rayData.radiance = float4(radiance, 1);  // Radiance (RGB + weight)

// Buffer에 쓰기
rayBuffer[probeIndex * DDGI_MAX_RAYCOUNT + rayIndex].store(rayData);
```

---

## 전체 흐름 요약

```
[CPU/Scene Update]
1. BLAS 생성 (GeometryComponent.cpp:1687-1712)
   └─> Vertex/Index buffer → BLAS descriptor

2. BLAS Build (SceneUpdate_Detail.cpp:1548)
   └─> GPU에서 BVH 구축

3. TLAS Instance 설정 (SceneUpdate_Detail.cpp:674-716)
   ├─> Transform matrix
   ├─> BLAS 참조
   └─> ★ Winding order 플래그 (det < 0일 때 CCW)

4. TLAS Build (SceneUpdate_Detail.cpp:1602)
   └─> 모든 instance의 top-level BVH 구축

[GPU/DDGI Raytracing]
5. Ray 설정 (ddgi_raytraceCS.hlsl:36-201)
   ├─> Probe 위치 계산
   ├─> Ray 방향 생성 (Spherical Fibonacci)
   └─> RayDesc 구성

6. TraceRayInline (ddgi_raytraceCS.hlsl:203-219)
   ├─> ★ TLAS traverse (scene_acceleration_structure)
   ├─> BVH intersection test
   └─> ★ Front/back face 판정 (CCW 플래그 사용)

7. Surface Data 로드 (ddgi_raytraceCS.hlsl:252-287)
   ├─> Hit 정보 추출 (position, primitive ID, barycentrics)
   ├─> ★ SetBackface(!CommittedTriangleFrontFace())
   ├─> Vertex 보간 (surface.load)
   └─> ★ Backface면 normal flip

8. Lighting 계산 (ddgi_raytraceCS.hlsl:290-432)
   ├─> Direct lighting (Light sampling)
   ├─> Shadow ray tracing
   ├─> Indirect lighting (이전 frame DDGI)
   └─> Final radiance = (direct + indirect) * albedo + emissive

9. 결과 저장 (ddgi_raytraceCS.hlsl:434-438)
   └─> rayBuffer에 radiance 저장

[Post-processing]
10. DDGI Update (ddgi_updateCS.hlsl)
    ├─> rayBuffer 읽기
    ├─> Spherical Harmonics 계산
    └─> Probe irradiance/depth 업데이트
```

---

## 핵심 포인트

### Winding Order 처리 (RHS ↔ DXR LHS)

1. **TLAS 생성 시 (SceneUpdate_Detail.cpp:706-713)**
   ```cpp
   if (det < 0) {  // 미러 변환만
       instance.flags |= FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE;
   }
   ```
   - RHS CCW geometry는 DXR에서 CW로 해석됨
   - det > 0 (정상): CCW 플래그 불필요 (이미 CW로 보임)
   - det < 0 (미러): CCW 플래그 필요 (CW → CCW 반전)

2. **Raytracing 시 (ddgi_raytraceCS.hlsl:262)**
   ```hlsl
   surface.SetBackface(!q.CommittedTriangleFrontFace());
   ```
   - DXR의 frontface 판정 결과를 반전하여 사용

3. **Surface 로드 시 (surfaceHF.hlsli:390-393)**
   ```hlsl
   if (is_backface) {
       N = -N;  // Backface면 normal 반전
   }
   ```
   - Backface에서도 올바른 lighting 계산 가능

### Coordinate System 요약

- **VizMotive:** RHS, CCW winding
- **DXR:** LHS 기준, CW = frontface (기본)
- **해결:** TLAS instance flag로 winding order 조정 (det < 0일 때만)
