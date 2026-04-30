# Surfel GI (Wicked Engine)

> Voxel GI 분석: [voxelGI.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/voxelGI/voxelGI.md)

Wicked Engine 9.2 기준으로 분석한 Surfel GI 구현입니다.

---

## 0. 이론

### Surfel이란?

**Surfel(Surface Element)** 은 3D 씬의 표면을 나타내는 **구체 형태의 작은 원반 요소**다.

- 각 surfel은 **위치(position)**, **법선(normal)**, **반경(radius)** 을 가짐
- 표면 위에 동적으로 배치되어 그 지점의 GI(간접광) 정보를 저장
- 고정된 격자가 아니라 **표면에 달라붙는** 방식이므로, 실제 지오메트리에 적응적으로 분포됨

### VXGI와의 차이

| 항목 | Voxel GI | Surfel GI |
|------|----------|-----------|
| GI 저장 단위 | 3D 격자 복셀 | 표면에 배치된 구체(surfel) |
| 씬 표현 | 공간 전체를 격자로 분할 | 실제 표면 위에만 존재 |
| 간접광 계산 | Cone tracing (복셀 샘플링) | Ray tracing (TLAS 기반) |
| 빛 정보 형식 | Anisotropic radiance (6면) | SH(Spherical Harmonics) L1 | 반구 형태에는 SH 에서 음수 로브 (light leak) 현상이 발생할 수 있음, H-basis 로 더 정확한 저장 가능(메모리는 동일)
| 적응성 | 고정 해상도 클립맵 | 커버리지에 따라 동적 스폰/소멸 |
| 공간 범위 | 클립맵 반경 내 | 무한 (hashing 기반 grid) |

### 핵심 아이디어

1. 씬의 표면에 최대 **100,000개**의 surfel을 배치
2. 매 프레임 각 surfel에서 **랜덤 방향으로 ray를 발사**해 조명 계산
3. ray 결과를 **SH(Spherical Harmonics)** 로 압축 저장
4. GI가 필요한 픽셀에서 근처 surfel의 SH를 가중합산해 간접광 계산

---

## 1. 데이터 구조

### 1.1 Surfel (per-surfel, 16바이트 정렬)

```cpp
struct Surfel
{
    SH::L1_RGB::Packed radiance;  // SH L1 radiance (GI lookup에 사용)
    uint2 normal;                  // packed half3 법선
    float3 position;               // 월드 공간 위치
    uint padding1;

    float GetRadius() { return SURFEL_MAX_RADIUS; } // 상수 2.0m
};
```

GI lookup 시 빠른 접근이 필요해서 최소한의 정보만 담음.
반경은 모든 surfel이 동일하게 **2.0m** 고정.

### 1.2 SurfelData (per-surfel, 추가 지속 데이터)

```cpp
struct SurfelData
{
    uint64_t uid;        // instance UID (유효성 검사용)
    uint2 primitiveID;   // 부착된 삼각형 ID
    uint bary;           // barycentric 좌표 (half2 packed)
    uint raydata;        // 24bit rayOffset + 8bit rayCount
    uint properties;     // 8bit life + 8bit recycle + 1bit backface
    float max_inconsistency; // 최대 분산 (ray 수 조절에 사용)
};
```

surfel이 고정된 world position이 아닌 **메시의 삼각형 위**에 붙어 있음.
매 프레임 `primitiveID + bary`로 표면 위치를 다시 계산 → 메시가 움직여도 surfel이 따라감.

### 1.3 Grid 시스템

```
그리드 크기: 128 × 64 × 128 셀
셀 하나의 크기: SURFEL_MAX_RADIUS = 2.0m
총 셀 수: 1,048,576

hashing 모드 (기본 활성화):
  cell = floor(position / 2.0)
  cellindex = (73856093*x ^ 19349663*y ^ 83492791*z) % TABLE_SIZE
  → 무한 월드 지원, 해시 충돌 가능성 있음

비-hashing 모드:
  카메라 중심의 유한 그리드
  cell = floor((pos - cameraPos) / 2.0) + GRID_DIMENSIONS / 2
```

한 surfel은 자신의 반경에 걸치는 최대 **27개 인접 셀**에 등록됨.

### 1.4 Moments 텍스처

```
포맷: R16G16_FLOAT (mean depth, mean depth²)
크기: SQRT(100000) × 4  per surfel (각 surfel당 4×4 texel)
용도: Chebyshev 가중치 계산 (빛이 가려진 방향의 기여도 억제)
```

각 surfel의 **방향별 평균 hit depth**를 저장.
direction → hemioct encoding → 텍스처 UV로 매핑해 샘플링.

### 1.5 버퍼 목록

| 버퍼 | 내용 |
|------|------|
| `surfelBuffer` | `Surfel` 배열 (최대 100,000개) |
| `surfelDataBuffer` | `SurfelData` 배열 |
| `surfelAliveBuffer[2]` | 현재 프레임 / 다음 프레임 alive 인덱스 목록 (ping-pong) |
| `surfelDeadBuffer` | 재사용 가능한 dead surfel 인덱스 풀 |
| `surfelGridBuffer` | `SurfelGridCell` 배열 (count + offset) |
| `surfelCellBuffer` | 각 셀에 속한 surfel 인덱스 목록 |
| `surfelRayBuffer` | `SurfelRayDataPacked` (direction, depth, radiance, surfelIndex) |
| `surfelStatsBuffer` | 전역 통계 (count, deadCount, rayCount, shortage 등) |
| `surfelIndirectBuffer` | `DispatchIndirect` 인자 |
| `surfelMomentsTexture` | depth moments 아틀라스 (2D 텍스처) |
| `surfelVarianceBuffer` | `SurfelVarianceData` per texel (분산 추정) |

---

## 2. 렌더 파이프라인

매 프레임 아래 순서로 실행:

| 순서 | 셰이더 | 설명 |
|------|--------|------|
| A | `surfel_coverageCS` | G-buffer에서 픽셀별 커버리지 측정 + 부족하면 새 surfel 스폰. GI 결과도 함께 출력 |
| B | `surfel_indirectprepareCS` | stats → indirect dispatch args 계산 (count, rayCount → ThreadGroupCount) |
| C | `surfel_updateCS` | alive surfel 위치/법선 갱신, ray 수 결정, dead 처리, grid count 누적 |
| D | `surfel_gridoffsetsCS` | 각 grid cell의 count를 prefix sum으로 offset 계산 |
| E | `surfel_binningCS` | surfel을 해당 grid cell에 실제로 배치 (cellBuffer 채우기) |
| F | `surfel_raytraceCS` | surfel당 N개 ray 발사, 직접광 + 간접광(멀티 바운스) 계산 |
| G | `surfel_integrateCS` | ray 결과를 SH radiance + moments로 통합 |

A 단계에서 GI 결과를 쓰고, 다음 프레임의 surfel 데이터는 C~G에서 갱신됨.
즉 **이번 프레임 GI는 전 프레임에서 계산된 surfel 데이터**를 사용함.

---

## 3. 각 셰이더 상세 분석

### 3.1 `surfel_coverageCS.hlsl` — 커버리지 측정 + GI 출력 (A)

**역할**: 스크린 공간 절반 해상도로 실행 (각 thread → 2×2 픽셀 담당).
픽셀 위치에서 G-buffer를 읽어 월드 좌표/법선을 복원하고, 해당 셀의 surfel들로 GI를 계산.

**GI 계산:**
```hlsl
// 해당 픽셀 위치의 grid cell 조회
int3 gridpos = surfel_cell(surface.P);
SurfelGridCell cell = surfelGridBuffer[cellindex];

for (uint i = 0; i < cell.count; ++i)
{
    Surfel surfel = surfelBuffer[surfelCellBuffer[cell.offset + i]];
    float3 L = surface.P - surfel.position;
    float dist2 = dot(L, L);

    if (dist2 < sqr(surfel.GetRadius()))  // 반경 내
    {
        float contribution = 1;
        contribution *= saturate(dot(N, surfelNormal));       // 법선 유사도
        contribution *= saturate(1 - dist / surfel.GetRadius()); // 거리 감쇠
        contribution = smoothstep(0, 1, contribution);
        coverage += contribution;

        // Chebyshev 가중치 (moments texture 이용)
        float2 moments = surfelMomentsTexture.SampleLevel(...);
        contribution *= surfel_moment_weight(moments, dist);

        // life 기반 가중치 (새 surfel의 검은 pop-in 방지)
        contribution = lerp(0, contribution, saturate(surfel_data.GetLife() / 2.0));

        // SH irradiance 계산
        color += SH::CalculateIrradiance(surfel.radiance.Unpack(), N) * contribution;
    }
}
color.rgb /= color.a;   // 가중 평균
color.rgb /= PI;         // irradiance 정규화
```

**새 surfel 스폰:**
```hlsl
// 16×16 thread group 내에서 커버리지가 가장 낮은 픽셀을 선택
InterlockedMin(GroupMinSurfelCount, encoded_coverage);

// 선택된 픽셀에서 dead list에서 인덱스 pop
InterlockedAdd(surfelStatsBuffer[0].deadCount, -1, deadCount);
uint newSurfelIndex = surfelDeadBuffer[deadCount - 1];

// alive list에 push
surfelDataBuffer[newSurfelIndex] = { primitiveID, bary, uid, ... };
```

커버리지가 `SURFEL_TARGET_COVERAGE(0.8)` 미만인 픽셀에서만 스폰 시도.
가까운 표면일수록 스폰 확률이 낮아져(chance 공식) 과도한 밀집 방지.

---

### 3.2 `surfel_indirectprepareCS.hlsl` — Indirect 인자 준비 (B)

```hlsl
// stats에서 다음 프레임 디스패치 크기 계산
surfelStatsBuffer[0].count = surfel_count;
surfelStatsBuffer[0].nextCount = 0;   // 초기화
surfelStatsBuffer[0].deadCount = dead_count;
surfelStatsBuffer[0].cellAllocator = 0;
surfelStatsBuffer[0].rayCount = 0;

// iterate: surfel당 1 thread
args.iterate.ThreadGroupCountX = (surfel_count + 31) / 32;
// raytrace: ray당 1 thread
args.raytrace.ThreadGroupCountX = (ray_count + 31) / 32;
// integrate: surfel당 1 thread group (8×8)
args.integrate.ThreadGroupCountX = surfel_count;
```

surfel 수와 ray 수에 따라 동적으로 디스패치 크기를 결정해 불필요한 thread 낭비를 없앰.

---

### 3.3 `surfel_updateCS.hlsl` — Surfel 위치 갱신 + Ray 수 결정 (C)

```hlsl
// 삼각형 위의 실제 현재 위치 계산
surface.load(prim, unpack_half2(surfel_data.bary));
surfel.position = surface.P;
surfel.normal = pack_half3(surface.facenormal);

// 27개 인접 셀에 count 누적 (D의 prefix sum 입력)
for (uint i = 0; i < 27; ++i)
    InterlockedAdd(surfelGridBuffer[cellindex].count, 1);

// ray 수 결정: 불일치(inconsistency)가 클수록 많은 ray 할당
uint rayCountRequest = saturate(surfel_data.max_inconsistency) * SURFEL_RAY_BOOST_MAX; // 최대 64
if (recycle > 10) rayCountRequest = 1;  // 재활용 대기 중이면 ray 줄임
if (recycle > 60) rayCountRequest = 0;  // 곧 소멸할 surfel은 ray 없음

// 글로벌 ray 예산(500,000) 내에서 offset 확보
InterlockedAdd(surfelStatsBuffer[0].rayCount, rayCountRequest, rayOffset);
```

**Recycling 처리:**
카메라에서 멀고 프러스텀 밖에 있는 surfel은 `recycle` 카운터 증가.
`SURFEL_RECYCLE_TIME(60프레임)` 이상 + 부족한 surfel이 있으면 반경을 0으로 설정 → dead 처리.

---

### 3.4 `surfel_gridoffsetsCS.hlsl` — Grid Offset Prefix Sum (D)

```hlsl
// 각 셀이 cellBuffer에서 자신의 공간 확보 (prefix sum)
if (surfelGridBuffer[DTid.x].count == 0)
    return;
InterlockedAdd(surfelStatsBuffer[0].cellAllocator,
               surfelGridBuffer[DTid.x].count,
               surfelGridBuffer[DTid.x].offset);
surfelGridBuffer[DTid.x].count = 0;  // count를 0으로 초기화 (binning에서 재사용)
```

`cellAllocator`에 각 셀의 count를 원자적으로 더해 offset을 결정 → E단계에서 surfel 인덱스 삽입 위치로 사용.

---

### 3.5 `surfel_binningCS.hlsl` — Grid Cell에 Surfel 배치 (E)

```hlsl
// 27개 인접 셀에 실제로 surfel 인덱스 기록
for (uint i = 0; i < 27; ++i)
{
    if (surfel_cellintersects(surfel, gridpos))
    {
        uint prevCount;
        InterlockedAdd(surfelGridBuffer[cellindex].count, 1, prevCount);
        surfelCellBuffer[surfelGridBuffer[cellindex].offset + prevCount] = surfel_index;
    }
}
```

D에서 확보한 offset + 재누적된 count를 이용해 cellBuffer에 인덱스를 씀.

---

### 3.6 `surfel_raytraceCS.hlsl` — Ray Tracing (F)

**surfel당 최대 64개 ray를 발사.** 각 ray는:

1. **직접광 (static light 샘플링)**:
```hlsl
// 정적 광원 중 무작위로 하나 선택
const uint light_index = lights().first_item() + rng.next_uint(light_count);
// 그림자 ray 발사 → 가시면 직접광 기여
shadow = TraceRay_Any(shadowRay, ...) ? 0 : 1;
radiance += light_count * (shadow * lightColor * NdotL / PI);
```

2. **반사 ray (hemisphere sampling)**:
```hlsl
ray.Direction = normalize(sample_hemisphere_cos(N, rng));
// TLAS를 통한 교차 검사
TraceRayInline(scene_acceleration_structure, ...);
```

3. **Hit point에서 조명 + 멀티 바운스**:
```hlsl
// hit surface에서 다시 광원 샘플링
hit_result += lightColor * NdotL / PI;

// 이전 프레임 surfel cache로 멀티 바운스 (SURFEL_ENABLE_INFINITE_BOUNCES)
surfel_gi += SH::CalculateIrradiance(surfel.radiance.Unpack(), surface.N) * contribution;
hit_result += surfel_gi.rgb / surfel_gi.a / PI;

hit_result *= surface.albedo;
hit_result += surface.emissiveColor;
radiance += hit_result;
```

결과를 `surfelRayBuffer`에 저장 (direction, depth, radiance).

---

### 3.7 `surfel_integrateCS.hlsl` — SH + Moments 통합 (G)

**Thread group: 8×8 (64 threads) = 한 surfel 담당**

```
SURFEL_MOMENT_RESOLUTION = 4
→ 4×4 = 16 texel, 각 texel이 반구 방향 하나를 담당
```

**1단계: texel별 ray 가중합산 (8×8 thread, 각자 다른 texel 방향 담당)**
```hlsl
float3 texel_direction = decode_hemioct(((GTid.xy + 0.5) / 4.0) * 2 - 1);
texel_direction = mul(texel_direction, get_tangentspace(N));

for (each ray in surfelRayBuffer) {
    float weight = saturate(dot(texel_direction, ray.direction) + 0.01);
    result += ray.radiance * weight;
    total_weight += weight;
}
result /= total_weight;
```

**2단계: Multiscale Mean Estimator (분산 안정화)**
```hlsl
// Firefly 억제 + 분산 기반 blend rate 조절
MultiscaleMeanEstimator(result, varianceData, 0.1);
result = varianceData.mean;
inconsistency = varianceData.inconsistency;  // 불일치도 → 다음 프레임 ray 수 결정
```

**3단계: Depth moments 업데이트**
```hlsl
// 이전 프레임과 lerp (0.02 blend)
result_depth = lerp(prev_moment, result_depth, 0.02);
surfelMomentsTexture[moments_pixel] = result_depth; // mean depth, mean depth²
```

**4단계: SH 계산 (groupIndex == 0 thread만)**
```hlsl
// 4×4 texel 결과를 SH L1으로 project
SH::L1_RGB radiance = SH::L1_RGB::Zero();
for (4×4 texels)
    radiance = SH::Add(radiance, SH::ProjectOntoL1_RGB(direction, value));
radiance = SH::Multiply(radiance, rcp(16 * HEMISPHERE_SAMPLING_PDF));
surfelBuffer[surfel_index].radiance = radiance.Pack();
```

---

## 4. GI 적용

GI가 필요한 렌더 패스에서 `surfel_coverageCS`의 결과 텍스처(`result_halfres`)를 샘플링:

```hlsl
// shadingHF.hlsli / objectHF.hlsli
if (GetFrame().options & OPTION_BIT_SURFELGI)
{
    Texture2D<float3> surfelGI = bindless_textures[GetCamera().texture_surfelgi_index];
    float3 surfelColor = surfelGI.SampleLevel(sampler_linear_clamp, uv, 0);
    lighting.indirect.diffuse = max(lighting.indirect.diffuse, surfelColor);
}
```

별도 cone tracing 없이 **스크린 공간 텍스처를 단순 샘플링**하므로 적용 비용이 낮음.

---

## 5. Surfel 생애 주기

```
[초기화]
  surfelDeadBuffer = {0, 1, 2, ..., 99999}  // 모두 dead
  surfelStatsBuffer.deadCount = SURFEL_CAPACITY

[스폰] (coverageCS)
  coverage < TARGET_COVERAGE인 픽셀에서
  deadBuffer.pop() → primitiveID + bary 기록 → aliveBuffer에 push

[갱신] (updateCS)
  매 프레임 표면 위치 재계산 (메시 이동 추적)
  ray count 결정 (inconsistency 기반)

[재활용] (updateCS)
  카메라 원거리 + 프러스텀 밖 → recycle++ 
  recycle > 60 + shortage > 0 → radius = 0 → dead 처리
  deadBuffer에 인덱스 반환

[소멸]
  deadBuffer에 push → 나중에 새 surfel로 재사용
```

---

## 6. 주요 파라미터

| 파라미터 | 값 | 설명 |
|----------|----|------|
| `SURFEL_CAPACITY` | 100,000 | 최대 surfel 수 |
| `SURFEL_MAX_RADIUS` | 2.0m | surfel 반경 (영향 범위) |
| `SURFEL_TARGET_COVERAGE` | 0.8 | 목표 커버리지 (픽셀당 surfel 기여도 합) |
| `SURFEL_RAY_BUDGET` | 500,000 | 프레임당 최대 ray 수 |
| `SURFEL_RAY_BOOST_MAX` | 64 | surfel당 최대 ray 수 |
| `SURFEL_RECYCLE_TIME` | 60 프레임 | 재활용 대기 기간 |
| `SURFEL_MOMENT_RESOLUTION` | 4×4 | surfel당 moment texel 수 |
| `SURFEL_GRID_DIMENSIONS` | 128×64×128 | 그리드 셀 수 |
| `BLEND_SPEED (moments)` | 0.02 | depth moments 업데이트 속도 |
| `shortWindowBlend` | 0.1 | MultiscaleMeanEstimator 단기 평균 blend |

---

## 7. 알려진 특성 및 제한

| 특성 | 내용 |
|------|------|
| Ray tracing 필수 | TLAS(Acceleration Structure) 기반이므로 RT 지원 GPU 필요 |
| 초기 수렴 지연 | surfel 스폰 직후 life가 낮으면 기여도가 제한됨 (pop-in 완화) |
| 반경 고정 | 모든 surfel이 2.0m 반경 → 세밀한 GI 표현에 한계 |
| 멀티 바운스 | `SURFEL_ENABLE_INFINITE_BOUNCES`로 이전 프레임 surfel 재활용 |
| Hashing 충돌 | `SURFEL_USE_HASHING` 모드에서 해시 충돌 시 잘못된 surfel 참조 가능 |
