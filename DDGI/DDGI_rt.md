# DDGI Raytracing 문제 분석

## 현재 문제 상황

두 지오메트리가 약간 겹쳐 있을 때, 빛이 닿을 수 없는 위치에 간접광이 보임.

![alt text](image-2.png)

- **왼쪽 구 2개**: 구의 그림자가 바닥 메시에 없음 + DDGI 간접광 = 빛을 직접 받는 바닥 메시보다 더 밝아지는 결과
- **오른쪽 구 1개**: 빛이 생기면 안되는 위치 (구, 바닥 메시로 가려지는 부분)에 간접광 발생

---

## 추가 분석: Geometry 내부 Probe 이상 현상

<https://github.com/user-attachments/assets/1858a026-f6c8-4f0a-bead-4a481cd0f258>

Probe 시각화를 통해 구 geometry 내부의 probe들이 간접광을 받고 있음을 발견.
특히 **빛과 가까운 쪽 probe보다 반대쪽(그늘진) probe들이 더 밝은** 역전 현상이 관찰됨.

![alt text](image-3.png)

Wicked Engine과 비교하면 결정적인 차이가 있음. Wicked Engine에서는 geometry 내부 probe들이 검은색으로 정상 처리됨.

---

## 결정적 실험: Cast Shadow OFF

Wicked Engine은 point light shadow가 구현되어 있으며 geometry별로 cast shadow 옵션을 제어할 수 있다.
**geometry의 cast shadow를 끄자 VizMotive에서 발생한 문제가 동일하게 재현되었다.**

![alt text](image-4.png)

- 구 내부 probe들이 밝아짐
- 빛을 향하는 방향보다 그늘진 방향의 probe들이 더 밝은 역전 현상도 동일하게 발생
- 마치 내부 표면에서 빛이 반사되어 probe가 샘플링하는 것처럼 보임
- (시각화를 위해 cube geometry의 double side 옵션이 켜져 있으나, 끄더라도 probe 색상은 동일)

이것으로 **point light shadow 미구현이 근본 원인**임이 확인되었다.

---

## 근본 원인: Shadow Ray 없이 내벽 Back Face에서 직접광 계산

### 동작 확인

- `RAY_FLAG_CULL_BACK_FACING_TRIANGLES` 주석 처리됨 → **DDGI probe ray는 front/back face 모두 hit 가능**
- Back face hit 시 normal을 반전: `N = -N` (`surfaceHF.hlsli:390-393`)
- Hit surface에서 직접광(point light)을 계산하는 단계가 존재

### Shadow 있을 때 (Wicked Engine 기본 동작)

```
probe ray → 구 내벽 hit (back face)
  → hit point에서 shadow ray를 light 방향으로 발사
  → shadow ray가 구 표면에 막힘 (back face culling 없음 → 내부에서 쏜 ray도 hit)
  → in shadow → direct light = 0
  → probe에 0 radiance 축적 → 검은색 ✓
```

### Shadow 없을 때 (VizMotive 현재 상태)

```
probe ray → 구 내벽 hit (back face)
  → shadow ray 자체를 발사하지 않음 (Phase 2에 point light shadow 코드 없음)
  → back face normal flip: N이 light 방향을 가리킴
  → NdotL > 0 → direct light 계산됨
  → probe에 잘못된 radiance 축적 → 밝아짐 ✗
```

### Cast Shadow OFF가 Shadow Ray를 우회하는 방식

"Cast shadow OFF"는 geometry를 모든 ray에서 보이지 않게 만드는 것이 아니다.
DDGI probe ray와 shadow ray는 서로 다른 **instance inclusion mask**를 사용하며, cast shadow는 geometry의 TLAS instance mask 레벨에서 제어된다.

```hlsl
// ddgi_raytraceCS.hlsl
// DDGI 메인 probe ray (geometry hit용)
q.TraceRayInline(scene_acceleration_structure,
    RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES | RAY_FLAG_FORCE_OPAQUE,
    push.instanceInclusionMask,   // C++에서 넘어온 general geometry mask
    ray);

// DDGI shadow ray (직접광 visibility 체크용)
q.TraceRayInline(scene_acceleration_structure,
    RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES | RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH,
    0xFF,                         // 하드코딩 0xFF
    newRay);
```

- Cast shadow **ON**: geometry의 TLAS instance mask에 shadow bit 포함 → `0xFF & shadow_bit ≠ 0` → shadow ray에 hit됨
- Cast shadow **OFF**: shadow bit 제거 → `0xFF & 0x00 = 0` → shadow ray가 통과함
- DDGI probe ray는 `push.instanceInclusionMask`를 사용하므로 cast shadow OFF geometry도 여전히 hit 가능 → DDGI 자체는 정상 동작

따라서 cast shadow OFF 상태에서 shadow ray가 구 표면을 통과하는 것은, VizMotive에서 shadow ray 자체가 없는 것과 동일한 결과를 낳는다.

### 그늘쪽 probe가 더 밝아지는 역전 현상의 원인

```
구 geometry, point light +X 방향 외부
probe A: 구 내부 그늘쪽 (-X)  /  probe B: 구 내부 밝은쪽 (+X)
```

**Probe A (그늘쪽):**
```
1. Probe A에서 -X 방향으로 ray 발사
2. 구 그늘쪽 내벽 hit (back face, 기하 outward normal = (-1,0,0))
3. back face flip → surface.N = (+1,0,0) ← light 방향을 가리킴
4. NdotL = dot((+1,0,0), light_dir) > 0 → 직접광 계산됨 → 밝음
```

**Probe B (밝은쪽):**
```
1. Probe B에서 +X 방향으로 ray 발사
2. 구 밝은쪽 내벽 hit (back face, 기하 outward normal = (+1,0,0))
3. back face flip → surface.N = (-1,0,0) ← light 반대 방향
4. NdotL = dot((-1,0,0), light_dir) < 0 → 직접광 = 0 → 어두움
```

**결과:** 그늘쪽 내부 probe가 밝은쪽 내부 probe보다 더 밝아지는 역전 현상.
Back face normal flip이 빛 계산 방향을 물리적으로 반전시키기 때문.

---

## Probe Relocation의 한계

`ddgi_updateCS.hlsl`의 relocation 로직:

```hlsl
// 각 ray가 너무 가까이 hit → 반대 방향으로 probe를 밀기
if (depth < probeOffsetDistance)
{
    probeOffsetNew -= ray.direction * (probeOffsetDistance - depth);
}
// 최대 이동량: ddgi_cellsize() * 0.5 (그리드 셀 절반)
probeOffset = clamp(probeOffset, -probe_limit, probe_limit);
```

**닫힌 geometry 내부에 완전히 갇힌 probe의 경우:**
- 모든 방향의 ray가 내벽에 hit → 각 ray의 반발력이 서로 상쇄 → 합력 ≈ 0
- 최대 이동량 제한으로 인해 탈출 불가
- voxelgrid fallback이 있으나 voxelgrid가 활성화된 경우에만 동작

**결론:** Relocation은 벽 근처 probe를 표면에서 띄우는 용도이며, 닫힌 solid geometry 내부에 갇힌 probe를 탈출시키지 못한다.

---

## 해결 방향

### 1. Point Light Shadow 구현 ★ 근본 해결

Shadow ray가 내벽 back face hit에서 빛을 차단하면, 내부 probe에 잘못된 radiance가 축적되지 않는다.
이것이 Wicked Engine에서 cast shadow ON일 때 내부 probe가 자동으로 검은색이 되는 이유다.

- 겹친 geometry에서 간접광 누수 문제도 동시에 해결됨
- 가장 물리적으로 올바른 해결책

### 2. Solid Geometry 내부 Probe Disable ★ 업계 표준 보조 수단

**DDGI 논문(Majercik et al., 2019)과 주요 게임 엔진(UE5 Lumen, Unity HDRP 등)에서 공통적으로 채택하는 방법이다.**

Shadow 구현 이후에도 완전히 갇힌 probe가 남아있는 경우의 보완책으로 사용.
Solid geometry 내부 probe는 신뢰할 수 없는 irradiance를 축적하므로 비활성화하고, 샘플링 시 유효한 주변 probe들의 irradiance만 사용한다.

**판정 기준:**
- Ray hit distance가 모든 방향에서 일정 기준 이하 (사방이 막힘) → 내부 probe로 판정
- 또는 voxelgrid로 probe 위치가 solid voxel 내부임을 직접 확인

**비활성화 probe 처리:**
- 해당 probe의 weight를 0으로 설정 → `ddgi_sample_irradiance`의 trilinear 보간에서 자동으로 제외
- 주변 유효 probe들의 irradiance로 대체

현재 구현에서는 voxelgrid fallback이 일부 역할을 하고 있으나, voxelgrid 비활성화 시 동작하지 않아 불완전하다.

---

## 수정 구현

### Fix 1: Point Light Shadow Map

Point light shadow map이 아무것도 렌더링되지 않던 문제를 두 가지 버그 수정으로 해결했다.

**버그 1 — frustum culling 오작동 (`ShadowMap_Detail.cpp`)**

```cpp
// 수정 전: RHS/LHS BoundingFrustum 불일치로 항상 false → camera_count = 0 → 빈 shadow atlas
if (cam_frustum.Intersects(cameras[shcam].boundingfrustum))
{
    // 이 블록이 절대 실행되지 않음
}

// 수정 후: frustum check 제거, 항상 6개 face 전부 포함
XMStoreFloat4x4(&cb.cameras[camera_count].view_projection, cameras[shcam].view_projection);
cb.cameras[camera_count].output_index = shcam;
// ...
camera_count++;
```

**버그 2 — 큐브맵 카메라 방향/UV 불일치 (`RenderPath3D_Detail.cpp`, `shadowHF.hlsli`)**

`cubemap_to_uv(-L)` 에서 face 선택 규칙:
- `L = light.position - surface.P` (surface→light)
- `-L = surface - light` (light→surface)
- face 0: `(-L).x > 0` → surface가 light 기준 **+X**에 있음 → 카메라는 **+X**를 바라봐야 함

기존 쿼터니언은 방향이 전부 반대였으며, `XMMatrixLookToRH` 특성상 U축이 항상 뒤집힌다.

```cpp
// 수정 후: 각 face가 올바른 방향을 바라보도록 쿼터니언 교체
shcams[0].init(position, XMFLOAT4(0,-0.707f,0,0.707f), ...); // face0: +X 방향, up=(0,1,0)
shcams[1].init(position, XMFLOAT4(0, 0.707f,0,0.707f), ...); // face1: -X 방향, up=(0,1,0)
shcams[2].init(position, XMFLOAT4(0, 0.707f,-0.707f,0), ...); // face2: +Y 방향, up=(0,0,-1)
shcams[3].init(position, XMFLOAT4(0, 0.707f, 0.707f,0), ...); // face3: -Y 방향, up=(0,0,+1)
shcams[4].init(position, XMFLOAT4(0, 1, 0, 0),          ...); // face4: +Z 방향, up=(0,1,0)
shcams[5].init(position, XMFLOAT4(0, 0, 0, 1),          ...); // face5: -Z 방향, up=(0,1,0)
```

`XMMatrixLookToRH`는 right = `cross(up, -fwd)` 방향으로 계산되어 V는 맞지만 U가 항상 뒤집힘.
셰이더에서 face 내부 U를 반전해 보정:

```hlsl
// shadowHF.hlsli — shadow_cube() 두 variant 모두 적용
const float3 uv_slice = cubemap_to_uv(-Lunnormalized);
float2 shadow_uv = uv_slice.xy;
shadow_uv.x = 1.0 - shadow_uv.x; // RHS shadow camera의 U 반전 보정
shadow_uv.x += uv_slice.z;
```

---

### Fix 2: Geometry 내부 Probe 빛 차단 (`ddgi_raytraceCS.hlsl`)

DDGI ray trace는 두 단계로 동작한다:

1. **1단계**: probe 위치 자체에서 직접광 샘플링 + shadow ray → `radiance`에 누적
   (내부 probe의 경우 shadow ray가 메시에 막혀 `radiance ≈ 0`)
2. **2단계**: scene ray 추적 → hit이 front face면 표면 lighting 계산

문제는 2단계에서 back face hit일 때도 lighting이 그대로 실행된다는 것이었다. `TMin = 0.001` 오프셋이 thin geometry(두께 0.3인 floor 등)의 front face를 건너뛰게 만들어 shadow ray가 차단을 실패하고, `ddgi_sample_irradiance`가 외부 probe의 밝은 irradiance를 샘플링해 피드백 루프를 형성했다.

```hlsl
// 수정 전: back face hit 시 depth만 조정하고 lighting은 그대로 실행
if (surface.IsBackface())
{
    hit_depth *= 0.9;
}
// 이 아래에서 direct light + ddgi_sample_irradiance 계산 → 잘못된 radiance 누적

// 수정 후: back face hit 시 depth만 기록하고 즉시 return
if (surface.IsBackface())
{
    hit_depth *= 0.9;
    DDGIRayData rayData;
    rayData.direction = ray.Direction;
    rayData.depth = hit_depth;       // probe visibility 유지
    rayData.radiance = float4(radiance, 1); // 1단계 결과만 (≈ 0)
    rayBuffer[probeIndex * DDGI_MAX_RAYCOUNT + rayIndex].store(rayData);
    return;
}
// 이 아래는 front face hit만 도달 → 올바른 surface lighting 계산
```

depth는 그대로 기록해 probe visibility(주변 geometry와의 거리 인식)는 유지된다.

---

## 최종 결과

### Point Light Shadow + 간접광 정상화

수정 전(`image-2.png`)과 달리 그림자가 올바르게 드리워지며 간접광 누수가 사라졌다.

![alt text](image-5.png)

### Geometry 내부 Probe 검은색 확인

<https://github.com/user-attachments/assets/e80a0319-2479-466c-8ae4-5ec1bc5a3331>

수정 전 밝게 빛나던 내부 probe들이 검은색으로 정상 처리됨. 역전 현상도 해소됨.
